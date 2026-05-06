# 002 — Making GDB Production-Ready: 20 Bugs, 3 New Features, and a Full Audit

> Part of the GDB build-in-public series · v1.5.1 → v1.7.0

---

## Where We Left Off

After the initial benchmark win (v0.2.1 was 25× faster than MySQL at writes), the next challenge was clear: *fast means nothing if it's not safe.*

GDB was quick, but it hadn't been battle-tested. I needed to answer one honest question:

**"Would I trust this with real data?"**

The answer was: not yet.

---

## The Audit: What I Actually Found

I ran a full security and correctness audit of the codebase. Here's what came up:

### 🔴 Critical — Data Could Silently Corrupt

**Bug 1: rebuildTypedCaches() overwrote delta-replayed state**

After a crash, GDB replays the WAL to rebuild state. The WAL replay correctly rebuilt the typed caches (ZSet, Hash, List, Set). Then `rebuildTypedCaches()` ran *after* — and overwrote everything with stale data from memTable.

```
WAL replay → zcache correct (player = 9500 score) ✅
rebuildTypedCaches() → zcache overwritten (player = 9000) ❌ stale!
```

The last N writes after any crash would be silently lost. Every restart.

**Fix:** Track which keys were delta-replayed. Pass that set to `rebuildTypedCaches()` and skip those keys.

---

**Bug 2: Compaction lost all typed data**

WAL compaction writes a snapshot of all data, then truncates the log. The snapshot code only read from `memTable`.

The problem: typed cache data (Hash, ZSet, List, Set) wasn't *in* memTable when using delta writes. It lived in separate in-memory caches.

```
Compaction runs → Writes snapshot from memTable only
memTable: {"user:1": "Budi"} only (strings)
zcache:   {"leaderboard": [...1000 members...]} ← NEVER WRITTEN
WAL truncated → leaderboard gone forever
```

**Fix:** Before writing the snapshot, `flushTypedCachesToMemTable()` serializes all typed caches into memTable first.

---

**Bug 3: Delta writes still called the expensive serializer**

I had added delta WAL writes — tiny 50-byte records instead of full JSON blobs. But I left a call to `zSerialize()` inside `walDeltaZAdd()`:

```go
func (e *Engine) walDeltaZAdd(key string, score float64, member string) {
    e.appendDelta(...)  // ✅ 50 bytes — good
    e.zSerialize(key)   // ❌ still serializing full ZSet to memTable!
}
```

This means the delta optimization was mostly nullified. And it introduced stale data into memTable (see Bug 1 above).

**Fix:** Remove `zSerialize()` from delta write functions. memTable is only updated during compaction now.

---

### 🟠 Serious — Production Impact

**Bug 4: Stale lock file after crash**

GDB creates a `gdb.lock` file to prevent two processes from corrupting the same data directory. But if the process is killed (SIGKILL, power cut), the lock file remains. Next startup: "database already in use" — permanently.

**Fix:** On startup, if lock file exists, try to open it exclusively. If successful, the original process is dead — take the lock. If not, the process is alive, report PIDs clearly.

**Bug 5: evictIfNeeded() didn't clear typed caches**

The LRU eviction removed keys from `memTable`, `expires`, and the sorted index. But the typed caches (hcache, zcache etc.) kept the data in memory.

With a memory limit set, GDB would evict keys from the index — but the actual data would remain in RAM. The limit was effectively ignored.

**Fix:** Added `evictTypedCache(key)` helper, called on each eviction.

---

### 🟡 Security

- WAL file created with `0644` permissions — readable by any user on the system. Changed to `0600`.
- Lock file same issue. Fixed.

---

## The Optimizations That Mattered

### WAL Delta Writes

Before: every `ZADD` serialized the *entire ZSet* to JSON and wrote it to WAL.

```
ZSet with 1000 members → ~30KB JSON written to WAL per operation
```

After: write a compact delta record instead.

```json
{"d":1,"op":1,"k":"leaderboard","s":"9500","m":"player1"}
```

That's ~55 bytes. **545× smaller per write.**

Recovery plays these delta records back directly into the typed caches in O(1) per record — no full re-parse of MB of JSON.

All operations now use deltas:
- `ZADD` → deltaZAdd
- `ZREM` → deltaZRem
- `ZINCRBY` → deltaZAdd (same as update)
- `ZPOPMIN/MAX` → deltaZRem
- `HSET` → deltaHSet
- `HDEL` → deltaHDel
- `LPUSH/RPUSH` → deltaLPush/deltaRPush
- `LPOP/RPOP` → deltaLPop/deltaRPop

### Binary Insertion for ZSet

ZSet (sorted set) operations used to call `sort.Slice()` on every `ZADD` — O(n log n) for every single member update.

Since the slice is *already sorted*, inserting one element is actually O(n) with a binary search for position + slice copy. A lot faster in practice because of cache locality.

Benchmark result: `ZIncrBy` became **2.4× faster**.

---

## New Features Added

### Streams (XADD / XREAD / XRANGE)

Redis Streams are an append-only log structure. Each entry has:
- A unique, monotonically increasing ID (`milliseconds-sequence`)
- A set of field-value pairs

```
XADD events * action "login" user "gilang"   → "1746600000000-0"
XADD events * action "purchase" item "book"  → "1746600000001-0"
XREAD events 0                               → [both entries]
```

Use cases:
- **Event sourcing** — append user actions, replay history
- **Message queue** — XADD producer, XREAD consumer with last-ID tracking
- **Activity feed** — append-only, trim with XTRIM

Implemented: `XADD`, `XREAD`, `XRANGE`, `XREVRANGE`, `XLEN`, `XTRIM`, `XDEL`, `XINFO`

### ACL — Per-User Permissions

The original design had a single global password. With ACL, you can create multiple users with specific permissions:

```
ACLSETUSER analyst password123 r    ← read only
ACLSETUSER writer  password456 rw   ← read + write
ACLSETUSER admin   password789 *    ← everything
```

Permissions: `Read | Write | Admin | PubSub | All`

Backward compatible: if you just set a global password, it maps to a `default` user with `ACLAll` — no behavior change.

---

## Where GDB Stands Now (v1.7.0)

**Benchmark on Intel Pentium G4560 (2017 budget CPU):**

```
HGET  (cache read)   :  304 ns/op  = 3,289,000 reads/sec
ZSCORE (ZSet lookup) :  260 ns/op  = 3,846,000 reads/sec
SIsMember (Set check):  263 ns/op  = 3,802,000 reads/sec
PUT (WAL write)      :   47 µs/op  =    21,000 writes/sec
```

**VS the competition (DB-Engines May 2026 ranking):**

| Database | Score | Throughput |
|---|---:|---:|
| Redis (#1 globally) | 147.63 | ~250K ops/sec |
| DragonflyDB (#34) | 0.56 | ~4.3M ops/sec |
| **GDB embedded reads** | — | **~3.8M ops/sec** |
| BoltDB (#35) | 0.55 | ~500K ops/sec |
| BadgerDB (#50) | 0.08 | ~333K ops/sec |

GDB reads outperform Redis because there's no network overhead — everything is in-process. For writes, WAL durability puts a ceiling, but that's the correct tradeoff: correctness over raw speed.

**What GDB has that nothing else does:**
- REST HTTP API built-in (no extra server needed)
- SSE Pub/Sub (realtime without WebSocket library)
- Embedded mode + server mode in same binary
- All 5 Redis data types + Streams + ACL
- Zero dependencies, 7MB binary — runs anywhere Go runs

---

## Lessons From This Sprint

**1. Optimization and correctness have hidden interactions.**
Adding delta writes was a performance win — but it broke recovery in a subtle way that only shows up after a crash and compaction. The fix required understanding the full data flow: WAL → recover() → rebuildTypedCaches() → serve.

**2. Audit before shipping.**
Every one of those critical bugs would have been invisible in normal operation. They only surface under: crash during write burst, compaction after delta writes, memory pressure with typed data. Classic "works on my machine" situations.

**3. File permissions are security.**
The WAL contains every key and value ever written. 0644 means any user on the system can read it. 0600 is the correct default for database files. Small change, big impact.

---

## Next Up

- **Replication**: stream WAL to replica nodes for HA
- **Lua EVAL**: atomic server-side scripting
- **Client SDKs**: official Python and Node.js packages
- **Write parallelism**: sharded mutexes for multi-core scaling

---

**[← 001 — Why I Built This](./001-why-i-built-this.md)**

*GDB v1.7.0 · github.com/Gicelldev/gdb-engine · by Gilang Raja*
