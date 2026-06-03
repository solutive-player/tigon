# CLAUDE.md

本文件为 Claude Code / 开发者提供 Tigon 仓库的快速导航与约定。**深入的架构、数据流、设计动机请看 [`architecture.md`](./architecture.md)。**

## 这是什么

**Tigon**：面向 CXL pod 的研究型分布式内存事务数据库（OSDI '25），采用 Pasha 架构（CIDR '25）。主协议 `TwoPLPasha`（2PL + 数据迁移 + 软件缓存一致）即 Tigon 本体；另含 `SundialPasha` 及一批基线协议。负载支持 TPC-C / YCSB / SmallBank / TATP。基于 [lotus](https://github.com/DBOS-project/lotus) 代码库。

## 目录速览

| 路径 | 内容 |
|---|---|
| `bench_*.cpp` | 各 benchmark 入口 `main()`（tpcc/ycsb/smallbank/tatp） |
| `core/` | 运行时：`Coordinator`/`Worker`/`Executor`/`Manager`/`Dispatcher`/`Partitioner`/`Table`/`CXLTable`/`Context`；`factory/WorkerFactory.h`；`group_commit/` |
| `protocol/TwoPLPasha/` | **Tigon 主协议**：`TwoPLPasha`/`...Transaction`/`...Helper`/`...RWKey`/`...Message`/`...Executor` + SCC 变体 |
| `protocol/Pasha/` | 共享基础设施：`MigrationManager` + 迁移策略（Clock/LRU/FIFO/NoMoveOut/Eagerly）；`SCCManager` + 机制（WriteThrough/NonTemporal/NoOP） |
| `protocol/{Sundial,TwoPL,SundialPasha,Silo,Star,Calvin,Aria,H-Store,...}/` | 基线/对照协议 |
| `common/` | CXL 基础设施：`CXLMemory`/`CXLTransport`/`MPSCRingBuffer`/`AtomicOffsetPtr`/`CXL_EBR`/`WALLogger`；`btree_olc_cxl/`、`CCHashTable`、无锁队列等 |
| `benchmark/<bench>/` | 每负载的 `Database/Workload/Transaction/Query/Schema/Storage/Context` |
| `dependencies/` | `cxlalloc`（共享内存分配器，预编译 `.a`）、`kernel_module`（CXL ivshmem 内核模块） |
| `emulation/` | 在单机用 VM + ivshmem 模拟 CXL pod 的脚本 |
| `scripts/` | `setup.sh`/`run.sh`/`push_button.sh` + `parse/` + `plot/` 复现论文图表 |
| `results/` | 预存结果（含 Motor 基线原始数据） |

## 构建

仓库使用 CMake（见 `CMakeLists.txt`），产物为 `bench_tpcc` / `bench_ycsb` / `bench_smallbank` / `bench_tatp`。

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build -j
```

> 注意：完整运行依赖真实 CXL 1.1 硬件（或双 socket 远程 NUMA 模拟）+ `dependencies/kernel_module` + ivshmem VM，普通环境只能编译、无法真跑分布式。VM 化构建分发：`./scripts/run.sh COMPILE_SYNC <vm_num>`。

## 运行

```bash
# TPCC / Tigon 示例（参数语义见 README「Test Tigon in Various Configurations」）
./scripts/run.sh TPCC TwoPLPasha 8 3 mixed 10 15 1 0 1 Clock OnDemand 200000000 1 WriteThrough None 30 10 GROUP_WAL 20000 0 0
# 复现全部论文图表（~7.5h）
./scripts/push_button.sh results/test1
```

`SYSTEM` 取值：`TwoPLPasha`(Tigon) / `TwoPLPashaPhantom` / `SundialPasha` / `Sundial` / `TwoPL`。

## 关键约定与注意点

- **header-only 模板协议**：协议与表大量用 C++ 模板，按 `context.protocol` 字符串在 `core/factory/WorkerFactory.h` 编译期实例化；新增协议从这里加分支。
- **`SETUP_CONTEXT` 宏**：`bench_*.cpp` 用它把命令行 flags 灌进 `Context`。
- **命名空间** `star`；代码风格见根目录 `.clang-format`（改动请保持一致）。
- **CXL 初始化协议**：共享对象一律 host0 `commit_shared_data_initialization`、其余 host `wait_and_retrieve_cxl_shared_data`；新增共享结构沿用该模式并申请新的 root index（见 `common/CXLMemory.h`）。
- **改动雷区**（详见 `architecture.md` §7）：
  - replication / recovery **未实现**；不要假设它们可用。
  - 硬编码上限：≤8 host（EBR）、≤16 host（SCC 位）、≤5 worker（EBR）；RWKey 64bit 位域。
  - 幻读防护不支持单事务内重复/重叠查询、连续 insert 只锁最后一个 key。
  - `USE_OUTPUT_THREAD` 依赖 `USE_CXL_TRANS`。

## 修改后的验证

- 优先跑对应 benchmark 的一致性检查（如 TPC-C 结束时 `Database.h` 的 `check_consistency` / `consistency check passed!`）。
- 性能/正确性回归用 `scripts/run_*.sh` + `scripts/parse/` 对比 `results/`。
