# CLAUDE.md

本文件为 Claude Code（claude.ai/code）等基于 LLM 的编码助手在本仓库工作时提供导航。
This file guides Claude Code (claude.ai/code) and other LLM-based coding assistants working in this repository.

> 双语说明 / Bilingual Note：每节先给出中文，再给出英文要点。命令、路径、参数名一律保留英文原样。
> Each section is presented in Chinese first, then with English bullet points. Commands, paths, and parameter names are kept verbatim in English.

---

## 1. 项目概述 / Project Overview

**Tigon** 是 OSDI '25 论文 *“Tigon: A Distributed Database for a CXL Pod”* 的配套实现。它是一个面向 **CXL Pod** 的研究型分布式事务内存数据库，通过共享 CXL 内存同步跨主机的并发访问，并采用 CIDR '25 论文中的 **Pasha** 架构。代码库基于 [lotus](https://github.com/DBOS-project/lotus) 改造而来；内存 B+Tree 来自 btreeolc；CXL transport 使用的无锁 MPSC ringbuffer 来自 waitfree-mpsc-queue。

仓库包含：
- Tigon 主系统（`protocol/TwoPLPasha/`）
- Sundial-CXL / Sundial+（`protocol/Sundial*/`、`protocol/SundialPasha/`）
- DS2PL-CXL / DS2PL+（`protocol/TwoPL*/`）
- TPC-C / YCSB / SmallBank / TATP benchmark 框架（`benchmark/`、`bench_*.cpp`）
- 在单台物理机上模拟 CXL Pod 的脚本（`emulation/`）
- 编译、运行、复现论文图表的脚本（`scripts/`）

**English:**
- Research distributed transactional in-memory DB for CXL Pods (OSDI '25 artifact)
- Implements Tigon plus baselines (Sundial-CXL/+, DS2PL-CXL/+) under one harness
- Built on lotus; uses btreeolc and a wait-free MPSC ring buffer
- Workloads: TPC-C, YCSB, SmallBank, TATP via `bench_*.cpp` entry points
- Includes a VM-based CXL Pod emulator and reproduction scripts

详见 `README.md`。论文与引用信息见 README 末尾。

---

## 2. 硬件组网与特性 / Hardware Setup & Networking

### 2.1 硬件需求 / Hardware Requirements

- **Option A**：≥ 40 核单 socket 机器 + 连接到第一个 socket 的 CXL 内存 + Ubuntu 22.04
- **Option B**（无 CXL 设备时）：双 socket 机器（每个 socket ≥ 40 核），用远端 NUMA 节点模拟 CXL + Ubuntu 22.04

### 2.2 CXL Pod 模拟拓扑 / Emulation Topology

由于尚无支持细粒度共享与硬件 cache coherence 的商用 CXL 设备，本项目用 VM 模拟一个 CXL Pod：

- 每个**物理主机** ≙ 一台运行在同一物理机上的 **VM**
- 多 VM 之间通过 **ivshmem** 共享同一块 **CXL 1.1** 内存模块（或远端 NUMA 节点）
- 该共享内存对挂接的物理机硬件 cache-coherent，从而在 VM 间天然维持一致性
- 拓扑示意见 `emulation.png`

关键实现路径：
- `emulation/start_vms.sh` —— 启动 VM 集群
- `emulation/image/` —— VM 镜像构建
- `emulation/ivshmem/` —— 跨 VM 共享内存通道
- `emulation/vm_lib/`、`emulation/host_setup/` —— host 与 VM 端配套脚本
- `scripts/setup.sh` —— host / VM 环境初始化

### 2.3 系统特性 / Key Features

- **硬件 cache-coherent CXL 共享区域** + **软件 cache-coherence 协议**（`WriteThrough` / `WriteThroughNoSharedRead` / `NonTemporal` / `NoOP`），可通过 `SCC_MECH` 选择
- **HW_CC_BUDGET**：限制硬件 cache-coherent 区域大小（字节）
- **数据迁移**：`Clock` / `LRU` / `FIFO` / `NoMoveOut` 多种策略，由 `MigrationManager` 与 `Policy*` 实现（`protocol/Pasha/`）
- **Epoch-based group commit WAL**（`common/WALLogger.h`、`core/group_commit/`）；`LOGGING_TYPE` 可选 `GROUP_WAL` / `BLACKHOLE`
- **CXL transport** 替代普通 socket（`USE_CXL_TRANS`），底层为无锁 MPSC ringbuffer（`common/MPSCRingBuffer.h`）

### 2.4 从零搭建模拟环境 / Bootstrap Emulation

> 所有命令均在仓库根目录执行 / Run everything from the repository root.

```bash
# 1. Host 端初始化 / Host setup
./scripts/setup.sh HOST

# 2. 构建 VM 镜像 / Build VM image
./emulation/image/make_vm_img.sh

# 3. 启动 VM 集群 / Launch VMs (此处启动 8 台 VM, 每台 5 核)
# Option A: 真实 CXL 内存
sudo daxctl reconfigure-device --mode=system-ram dax0.0 --force
sudo ./emulation/start_vms.sh --using-old-img --cxl 0 5 8 0 2  # 末位为 CXL 内存所在 NUMA 号
# Option B: 用远端 NUMA 模拟
sudo ./emulation/start_vms.sh --using-old-img --cxl 0 5 8 1 1

# 4. VM 端初始化 / VM setup（参数 8 = VM 数量）
./scripts/setup.sh VMS 8

# 5. 编译并同步二进制到所有 VM / Compile & sync binaries
./scripts/run.sh COMPILE_SYNC 8
```

---

## 3. 构建与运行 / Build & Run

### 3.1 本地构建 / Local Build

```bash
mkdir build && cd build
cmake ..
make -j
```
产物 / Artifacts：`build/bench_tpcc`、`build/bench_ycsb`（`bench_smallbank` / `bench_tatp` 默认未启用，见 `CMakeLists.txt`）。

### 3.2 单实验示例 / Single-Run Examples

```bash
# TPC-C, Tigon (TwoPLPasha), 8 hosts, 3 workers/host
./scripts/run.sh TPCC TwoPLPasha 8 3 mixed 10 15 1 0 1 Clock OnDemand 200000000 1 WriteThrough None 15 5 GROUP_WAL 20000 0 0

# YCSB, Tigon, 300k keys, 50/50 RW, theta=0.7, 10% cross-host
./scripts/run.sh YCSB TwoPLPasha 8 3 rmw 300000 50 0.7 10 1 0 1 Clock OnDemand 200000000 1 WriteThrough None 30 10 BLACKHOLE 20000 0 0
```

参数全集说明见 `README.md` 中 *Test Tigon in Various Configurations* 一节。

### 3.3 论文结果复现 / Reproducing Paper Results

- 一键全量（约 7.5 小时）：`./scripts/push_button.sh results/test1`
- 单图：`./scripts/run_tpcc.sh`、`./scripts/run_ycsb.sh`、`./scripts/run_hwcc_budget.sh`、`./scripts/run_swcc.sh`
- 解析 / 画图：`scripts/parse/parse_*.py`、`scripts/plot/plot_*.py`，输出 PDF 到 `results/`

### 3.4 重要约定 / Conventions

- **始终在项目根目录运行命令**（README 强调）
- 修改源码后，需要重新 `./scripts/run.sh COMPILE_SYNC <N>` 才能在 VM 中生效
- 项目无单元测试套件，正确性由 benchmark 末尾的 `check_consistency()` 校验（参见 `benchmark/tpcc/Database.h` 内 `TPC-C consistency check passed!` 输出）

---

## 4. 架构概述 / Architecture Overview

### 4.1 核心引擎 `core/`

| 组件 | 作用 |
|---|---|
| `Coordinator.h` | 主控：初始化 CXL 内存、线程池、日志，编排 worker / dispatcher / executor，输出全局统计 |
| `Dispatcher.h` | 节点间消息路由（Incoming/Outgoing） |
| `Executor.h` | 按协议执行事务的 worker 主循环 |
| `Manager.h` | 资源与分区管理 |
| `CXLTable.h` | CXL / 本地内存的统一表抽象 |
| `Worker.h` | 事务工作线程 |
| `group_commit/` | epoch 组提交 WAL 实现 |
| `factory/` | 协议 / workload 工厂 |
| `Macros.h` | 全大写宏（如 `SETUP_CONTEXT`） |

### 4.2 协议层 `protocol/`

每个协议是独立子目录，统一遵循文件骨架：
```
<Protocol>/
  <Protocol>.h           # 协议主类
  <Protocol>Transaction.h
  <Protocol>Executor.h
  <Protocol>Helper.h
  <Protocol>Message.h
  <Protocol>RWKey.h      # 读写集追踪
```
Pasha 系列额外含 `MigrationManager`、`Policy{Clock,LRU,FIFO,NoMoveOut}`、`SCCManager` 与对应 SCC 实现（`SCC{WriteThrough,NonTemporal,NoOP,...}`）。

实现的协议：`TwoPLPasha`（即 Tigon）、`SundialPasha`、`Pasha`、`TwoPL`、`Sundial`、`Calvin`、`Aria`、`Silo`、`HStore`、`Star` 等。

### 4.3 公共组件 `common/`

`CXLMemory.h`、`CXLTransport*`、`WALLogger.h`、`Message.h`、`ThreadPool.h`、`Socket.h`、`MPSCRingBuffer.h`、`btree_olc/`、`Zipf.h`、原子操作工具等。**新代码请优先复用这里的工具，不要造轮子。**

### 4.4 入口文件 / Entry Points

- `bench_tpcc.cpp`、`bench_ycsb.cpp`：构造 Database → Coordinator → start → consistency check
- `bench_smallbank.cpp`、`bench_tatp.cpp`：当前未在 `CMakeLists.txt` 启用，使用前需取消注释

---

## 5. 目录结构 / Directory Layout

| 目录 / Directory | 用途 / Purpose |
|---|---|
| `benchmark/` | TPC-C / YCSB / SmallBank / TATP 实现，每个含 `Database.h` / `Storage.h` / `Query.h` / `Transaction.h` |
| `core/` | 协议无关的执行引擎与控制平面 |
| `protocol/` | 各并发控制协议实现 |
| `common/` | CXL 内存 / 传输、WAL、线程池、B+Tree、消息、分布生成器 |
| `dependencies/` | 第三方组件：`cxlalloc/`（CXL 内存分配器）、`kernel_module/`（CXL 内核模块） |
| `emulation/` | VM 镜像构建与启动、ivshmem、host / VM 配置脚本 |
| `scripts/` | 编排脚本（`run.sh` / `setup.sh` / `push_button.sh` / `run_*.sh`）+ `parse/` + `plot/` |
| `results/` | 实验输出（含预先采集的 `motor` 基线数据） |
| `bench_*.cpp` | 各 workload 的 `main` 入口 |
| `CMakeLists.txt` | 构建配置 |
| `.clang-format` | 代码格式化规则 |
| `LICENSE` | Apache 2.0 |

---

## 6. 代码风格与约定 / Code Style & Conventions

### 6.1 语言与工具链 / Language & Toolchain

- **C++14**（`CMakeLists.txt` 第 8 行 `CMAKE_CXX_STANDARD 14`）
- **Clang-15 / LLD-15**（`CMakeLists.txt` 第 4–5 行）
- 编译选项：`-O3 -march=native -pthread -pedantic -g3`（保留帧指针，便于 perf）
- 全局 allocator：**jemalloc**；CXL 共享数据走 `dependencies/cxlalloc/`，**不要用 `malloc` / `new` 分配跨 host 共享数据**
- 依赖：`glog`、`gflags`、`jemalloc`、`cxlalloc`

### 6.2 格式化 / Formatting

仓库根 `.clang-format` 已配置：
- 4 空格缩进，`AccessModifierOffset: -4`
- 函数 / 命名空间后强制换行（brace wrapping）
- 空函数体 / 类 / 命名空间不折行
- 短函数不合并到单行

提交前请运行 `clang-format -i <file>`。

### 6.3 命名约定 / Naming

- 类 / 类型：`PascalCase`
- 函数 / 变量：`camelCase`
- 宏：`UPPER_SNAKE_CASE`（见 `core/Macros.h`）
- 协议文件遵循 `<Protocol>{Protocol,Transaction,Executor,Helper,Message,RWKey}.h` 命名

### 6.4 注释与日志 / Comments & Logging

- 注释默认精简，必要时用英文
- 日志用 **glog**：`LOG(INFO)` / `VLOG(k)` / `DLOG(...)`，输出形如 `I0426 06:22:58.143128 204381 Coordinator.h:610]`
- 性能与统计在 `Coordinator.h` 末尾汇总（throughput / abort / round_trip_latency / WAL group commit 分位等）

### 6.5 测试 / Testing

- 无单元测试框架；正确性靠 benchmark 末尾的 `check_consistency()`
- CI：`.github/workflows/claude.yml`（占位，实际为空）

---

## 7. 给 Claude 的工作提示 / Tips for Claude

- 修改协议时，通常需要同步改动三处：`<Protocol>Executor.h`、`<Protocol>Message.h`、`<Protocol>RWKey.h`
- 新增 CXL 共享数据结构必须用 `cxlalloc`；查看 `common/CXLMemory.h` 的现有抽象再决定是否新增
- 修改后要在 VM 中验证：`./scripts/run.sh COMPILE_SYNC <N>` 然后跑一条 `run.sh TPCC ...` 或 `run.sh YCSB ...`，关注末尾 `consistency check passed!` 与 `Coordinator exits.`
- `scripts/run.sh` 的位置参数较长，不要凭记忆拼参数，参考 README *Test Tigon in Various Configurations* 一节，或先 `head -100 scripts/run.sh` 对照
- 解析输出请复用 `scripts/parse/`，画图复用 `scripts/plot/`，避免重复实现
- 论文复现脚本耗时极长（一键 ~7.5h），调试时先用单条 `run.sh` 命令快速验证
