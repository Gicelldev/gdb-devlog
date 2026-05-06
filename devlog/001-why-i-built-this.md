# 001 — Why I Built My Own Database Engine

> Part of the GDB build-in-public series

---

## The Starting Point

Every developer has that moment where they think: *"Could I build this?"*

For me it was databases. I'd been using MySQL, PostgreSQL, Redis for years — treating them as magic black boxes. One day I decided to stop doing that.

**The challenge I set:** Build a database engine that beats all of them on raw performance.

## What I Found When I Looked Inside

Most databases are slow for the same reason: **they're designed for correctness and features first, speed second.**

MySQL with full durability: ~1,000-15,000 writes/second  
PostgreSQL with full durability: ~15,000-20,000 writes/second  
Redis (in-memory, no fsync): ~100,000 writes/second

All of them trade performance for features: SQL queries, ACID transactions, replication, indexing...

## The Question

What if I throw away everything except the essentials?

- Keep data safe on disk (crash recovery)
- Make reads as fast as possible (from RAM)
- Make writes as fast as possible (batch disk I/O)
- Start simple, add complexity only when needed

## The Result So Far

```
GDB v0.2.1 — Intel Pentium G4560, 8GB RAM, SSD

Sequential Write : 255,000 ops/sec  (vs MySQL ~10,000)
Sequential Read  : 1,387,000 ops/sec (vs MySQL ~80,000)
Mixed R/W        : 552,000 ops/sec

GDB is 25x faster at writes than MySQL.
GDB is 2.5x faster at writes than Redis.
```

On a basic budget CPU from 2017. Not a server. Not a cloud instance.

## How?

Three key architectural decisions:

**1. Everything lives in RAM**  
No disk seeks for reads. Hash map lookups are O(1) — nanoseconds, not milliseconds.

**2. Writes go to an append-only log first**  
Sequential disk writes are 10-100x faster than random writes. The WAL (Write-Ahead Log) captures everything sequentially.

**3. Group Commit — the secret weapon**  
Instead of calling `fsync()` (expensive!) for every write, I batch hundreds of writes together and call `fsync()` once every 10ms. 

Result: 255,000 writes all sharing that one expensive syscall.

## What's Next

In the [next devlog](./002-write-ahead-log.md), I'll go deep on how the Write-Ahead Log works — including the binary format, CRC32 checksums, and crash recovery.

---

**[← Back to main](../README.md)** · **[002 — How WAL works →](./002-write-ahead-log.md)**

*GDB v0.2.1 | Source code proprietary during development · Open source at v1.0*
