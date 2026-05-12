# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## About

Tigon is a research distributed transactional in-memory database that synchronizes cross-host concurrent data accesses over shared CXL memory. It implements the Pasha architecture (CIDR '25) and is the subject of the Tigon paper (OSDI '25). The repo also contains CXL-optimized variants of Sundial and DS2PL as baselines.

## Build

```bash
mkdir -p build && cd build
cmake ..
make -j$(nproc)
```

The build produces `bench_tpcc` and `bench_ycsb` binaries in `build/`. Compiler is clang-15, C++14, `-O3 -march=native`. SmallBank and TATP targets exist but are commented out.

To compile and sync binaries to VMs:
```bash
./scripts/run.sh COMPILE_SYNC <HOST_NUM>   # compile locally + rsync to VMs
```

Dependencies: clang-15, lld-15, jemalloc, glog, gflags, boost. The `dependencies/cxlalloc/` directory contains the CXL allocator as a prebuilt static library.

## Running Experiments

All commands must be run from the project root. Experiments run across VMs via SSH; the VM network uses `192.168.100.{2+i}:1234` for host `i`.

```bash
# TPC-C (Tigon = TwoPLPasha)
./scripts/run.sh TPCC TwoPLPasha 8 3 mixed 10 15 1 0 1 Clock OnDemand 200000000 1 WriteThrough None 30 10 BLACKHOLE 20000 0 0

# YCSB
./scripts/run.sh YCSB TwoPLPasha 8 3 rmw 300000 50 0.7 10 1 0 1 Clock OnDemand 200000000 1 WriteThrough None 30 10 BLACKHOLE 20000 0 0
```

**System name mapping:**
- `TwoPLPasha` → Tigon (the main system)
- `SundialPasha` → Sundial+
- `TwoPL` → DS2PL / DS2PL-CXL
- `Sundial` → Sundial / Sundial-CXL
- `TwoPLPashaPhantom` → Tigon with phantom avoidance disabled

**Key configuration knobs:**
- `HW_CC_BUDGET`: bytes of hardware cache-coherent CXL region
- `SCC_MECH`: `WriteThrough` (Tigon default), `WriteThroughNoSharedRead`, `NonTemporal`, `NoOP`
- `MIGRATION_POLICY`: `Clock`, `LRU`, `FIFO`, `NoMoveOut`
- `LOGGING_TYPE`: `BLACKHOLE` (no logging), `GROUP_WAL` (epoch-based group commit)

## Architecture

### Layer Overview

```
bench_tpcc.cpp / bench_ycsb.cpp        ← entry points (gflags parsing, Coordinator launch)
  └── core/Coordinator.h               ← orchestrates all threads on one host
        ├── core/Dispatcher.h          ← I/O threads: socket and CXL transport receive
        └── core/Executor<W,P>         ← worker threads: run transactions
              ├── Workload (benchmark/) ← generates transactions
              └── Protocol (protocol/) ← concurrency control logic
```

### core/

- **`Coordinator`**: Top-level per-host object. Initializes CXL memory (`CXLMemory`), CXL transport, WAL logger, then spawns Executor worker threads and Dispatcher I/O threads. Aggregates stats and prints them periodically.
- **`Executor<Workload, Protocol>`**: Base worker class that drives the transaction execution loop — generate txn from workload, execute via protocol, handle messages from remote peers.
- **`Worker`**: Abstract base for all threads; holds commit/abort counters and message queues.
- **`Dispatcher`**: Dedicated I/O thread that reads incoming socket or CXL transport messages and routes them to the correct worker's queue.
- **`Table` / `ITable`**: Local in-memory table backed by either a `HashMap` or `BTreeOLC` (optimistic lock coupling B+Tree). Row layout is `(MetaDataType*, void*)` = `(atomic<uint64_t>*, data)`.
- **`CXLTableBase` / `CXLTableHashMap` / `CXLTableBTree`**: Shared CXL-resident table counterparts backed by `CCHashTable` (concurrent cuckoo hash) or `BTreeOLC_CXL`. Accessed by all hosts via CXL shared memory.

### protocol/

Each protocol directory follows the same pattern:
- `<Proto>.h` — lock/version metadata accessors on row metadata word
- `<Proto>Transaction.h` — transaction state, read/write set
- `<Proto>RWKey.h` — per-key metadata in the read/write set
- `<Proto>Executor.h` — extends `core/Executor`, overrides `setupHandlers()` and the execution loop
- `<Proto>Message.h` / `MessageFactory` / `MessageHandler` — network message types and dispatch

**Active protocols (compiled):**
- `TwoPLPasha/` — Tigon: 2PL extended with Pasha CXL data movement and SCC
- `SundialPasha/` — Sundial adapted to Pasha architecture
- `TwoPL/`, `Sundial/` — baseline distributed protocols

**Legacy protocols (not compiled in CMakeLists but present):** Aria, Calvin, Silo, SiloGC, Star, H-Store, TwoPLGC.

### protocol/Pasha/

Shared infrastructure for any Pasha-architecture protocol:

- **`MigrationManager`**: Tracks which rows live in local partition memory vs. the shared CXL region. On a cross-partition access, it calls `move_row_in` to migrate a row into CXL. Eviction is driven by a policy object.
- **`Policy{Clock,LRU,FIFO,NoMoveOut}`**: Eviction policies for the CXL hardware-cache-coherent budget. When the budget is exceeded, a victim row is selected and moved back to local memory.
- **`SCCManager`** and implementations (`SCCWriteThrough`, `SCCNonTemporal`, `SCCNoOP`): Software cache-coherence layer. Before reading or writing a CXL row, these determine whether to issue `clflush`/`clwb` to maintain coherence across hosts that lack hardware cache coherence for that region.

### benchmark/

Each benchmark (`tpcc/`, `ycsb/`, `smallbank/`, `tatp/`) exposes:
- `Database` — loads/initializes all tables, exposes `create_or_retrieve_cxl_tables()` for Pasha-aware setup
- `Workload` — generates `Transaction` objects
- `Transaction` — implements workload-specific txn logic against `ITable`
- `Context` — workload parameters (parsed from gflags in `bench_*.cpp`)
- `Schema` / `Storage` — struct definitions for rows

### common/

- `CXLMemory`: Manages `cxlalloc` initialization and the CXL shared memory region. Provides `commit_shared_data_initialization` / `wait_and_retrieve_cxl_shared_data` for cross-host pointer sharing via fixed root slots.
- `CXLTransport` / `MPSCRingBuffer`: Lock-free MPSC ring buffer in CXL memory used as a low-latency inter-host transport (alternative to TCP sockets).
- `CXL_EBR`: Epoch-based reclamation for safe concurrent CXL memory reclaim.
- `WALLogger` / `GroupCommitLogger`: Epoch-based group-commit WAL; the epoch counter is stored in CXL memory so all hosts advance together.
- `CCHashTable`: Concurrent cuckoo hash table designed for CXL-resident data.
