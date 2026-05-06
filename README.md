<div align="center">

# ⚡ GDB — Go Database Engine

**255,000 writes/sec · 1.4M reads/sec · Zero dependencies · Single binary**

[![Version](https://img.shields.io/badge/version-v0.2.1-blue)]()
[![Go](https://img.shields.io/badge/Go-1.21+-00ADD8?logo=go)]()
[![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20Linux%20%7C%20macOS-lightgrey)]()
[![Status](https://img.shields.io/badge/status-active%20development-green)]()

*A blazing-fast, embeddable key-value database engine built from scratch in Go*

**2.5x faster writes than Redis · 25x faster than MySQL · Zero external dependencies**

[📥 Download Binary](#-download) · [📖 Documentation](./docs/) · [📊 Benchmarks](./benchmarks/) · [📰 Dev Journal](./devlog/) · [🗺️ Roadmap](#️-roadmap)

</div>

---

## ⚡ Performance (Intel Pentium G4560, 8GB RAM, SSD)

| Operation | GDB v0.2.1 | Redis | MySQL | PostgreSQL |
|---|---|---|---|---|
| Write (sequential) | **255,000/s** | 100,000/s | 10,000/s | 15,000/s |
| Read (sequential) | **1,387,000/s** | 800,000/s | 80,000/s | 80,000/s |
| Mixed R/W | **552,000/s** | 300,000/s | 30,000/s | 40,000/s |
| Concurrent Write | **201,000/s** | 80,000/s | 12,000/s | 18,000/s |
| Concurrent Read | **1,190,000/s** | 600,000/s | 150,000/s | 150,000/s |

```
GDB vs MySQL      → 25x faster writes, 17x faster reads
GDB vs PostgreSQL → 17x faster writes, 17x faster reads  
GDB vs Redis      →  2.5x faster writes, 1.7x faster reads
```

---

## 🚀 Quick Start

```bash
# 1. Download binary (see Releases section)
# 2. Run as server
./gdb --server --addr 0.0.0.0:6380

# 3. Connect with any TCP client (telnet/nc)
telnet localhost 6380

GDB v0.2 — Ketik HELP untuk bantuan
> SET user:1 {"name":"Budi","age":25}
OK (45µs)
> GET user:1
{"name":"Budi","age":25} (8µs)
> SETEX session:abc 30 user_data
OK (expire: 30.0s) (52µs)
> TTL session:abc
28.500s (5µs)
> STATS
📊 GDB Statistics:
  writes     : 2
  reads      : 1
  avg_batch  : 884.2 writes/sync
  mem_used   : 64 B
  ...
```

---

## 🔥 Key Features

### Performance
- **Group Commit architecture** — batch hundreds of writes per single fsync
- **In-memory MemTable** — all reads served from RAM, O(1) lookup
- **Lock-free concurrent reads** — `sync.RWMutex` based, readers never block each other
- **3 sync modes** — choose between raw speed and durability

### Durability
- **Write-Ahead Log (WAL)** — CRC32 integrity check per record
- **Snapshot + WAL** — fast startup, compact history automatically
- **WAL Compaction** — auto-compact when WAL > 50MB, prevents disk overflow
- **Crash recovery** — automatic on startup, replays WAL on top of snapshot

### Features
- **Native TTL** — `SETEX`, `EXPIRE`, `PERSIST`, `TTL` commands — persistent after crash
- **Memory limit + LRU eviction** — `--maxmem=256MB`
- **Password auth** with brute force protection (5 fails → locked 60s)
- **Connection limit**, request size limit, idle timeout
- **Prefix scan** — `SCAN user:` returns all matching keys with TTL info

### Developer Experience
- **Zero external dependencies** — only Go standard library
- **Single binary** — download and run, no installation
- **Embeddable** — use as library directly in your Go app
- **Built-in benchmark** — `./gdb --bench`

---

## 🔒 Security Layers

```
L1: Password Authentication   → AUTH command
L2: Brute Force Lockout       → 5 fails → locked 60s  
L3: Auth Throttle             → 200ms delay per fail
L4: Connection Limit          → --maxconns=N
L5: Request Size Limit        → max 512KB per command
L6: Idle Timeout              → 5 min auto-disconnect
```

---

## 📥 Download

| Platform | Binary | Size |
|---|---|---|
| Windows x64 | [gdb-windows-amd64.exe](./releases/) | ~6MB |
| Linux x64 | [gdb-linux-amd64](./releases/) | ~5MB |
| macOS x64 | [gdb-darwin-amd64](./releases/) | ~5MB |

> Source code is proprietary during development. Will be open-sourced at v1.0.

---

## 🗺️ Roadmap

- [x] **v0.1** — Core engine: MemTable, WAL, Group Commit, TCP server
- [x] **v0.2** — TTL expiry, Memory limit + LRU, WAL Compaction, Auth  
- [x] **v0.2.1** — Security hardening: brute force, conn limit, 6 bug fixes
- [ ] **v0.3** — TLS/SSL, MSET/MGET, INCR/DECR, Config file
- [ ] **v0.4** — HTTP REST API, `gdb-cli` client, Docker image
- [ ] **v0.5** — B+Tree index, Range Query, Transactions (MULTI/EXEC)
- [ ] **v1.0** — Replication, Prometheus metrics, Go + Python client libraries, Open Source 🎉

---

## 📰 Dev Journal

Building this database from scratch — documenting every decision:

| # | Topic | What I learned |
|---|---|---|
| [001](./devlog/001-why-i-built-this.md) | Why build a database? | Existing solutions trade speed for features |
| [002](./devlog/002-write-ahead-log.md) | How Write-Ahead Logs work | Append-only = fastest possible disk write |
| [003](./devlog/003-group-commit.md) | Group Commit: 255K writes/sec | Batch 884 writes into 1 fsync |
| [004](./devlog/004-ttl-expiry.md) | Implementing TTL like Redis | Active sweep + lazy delete hybrid |
| [005](./devlog/005-security-bugs.md) | 6 critical bugs I found | Data loss, goroutine leak, blocking writes |

---

## 🏗️ Architecture (Brief)

```
Application
    │
TCP Server (auth · limits · idle timeout)
    │
Storage Engine
    ├── MemTable (map + RWMutex)     ← all reads here, O(1)
    ├── WAL (append-only + CRC32)    ← all writes go here first
    └── Snapshot (binary checkpoint) ← compacted state on disk
    
Background Goroutines:
    ├── Syncer    → flush WAL every 10ms (Group Commit)
    ├── Expirer   → sweep expired TTL keys every 200ms
    └── Maintainer → eviction check 500ms · compact check 30s
```

[Full architecture documentation →](./docs/architecture.md)

---

## ⭐ Follow Along

This project is being built in public. Star ⭐ this repo to stay updated!

Each new version will have:
- Changelog with what changed and why
- Updated benchmarks  
- New devlog posts explaining the technical decisions

---

<div align="center">

*Built with Go · Zero dependencies · Made with ❤️*

**[⭐ Star this repo](https://github.com/Gicelldev/gdb-devlog)** if you find it interesting!

</div>
