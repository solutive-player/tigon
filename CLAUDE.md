# CLAUDE.md

本文件为 Claude Code 在本仓库工作时提供指引。

## 项目概述

Tigon 是 OSDI '25 论文《Tigon: A Distributed Database for a CXL Pod》(Huang et al.) 对应的
研究型分布式内存事务数据库。它采用 **Pasha 架构**(CIDR '25),通过共享 CXL 内存同步跨主机的
并发数据访问。

- 代码 fork 自 `lotus` 代码库,因此**根命名空间是 `star::`**(不是 `tigon`),不要因此困惑。
- 这是为复现论文图表而写的研究代码,正确性以"能复现论文数字"为准。
- 环境搭建、VM/CXL pod 模拟、图表复现步骤见 `README.md`,本文件不重复这些内容。

## 论文概念与代码的对应关系(最重要)

- **Pasha 架构**:观察到虽然数据库很大,但被多主机并发读写的元组集合很小。因此只把这些
  跨主机热点元组迁移进共享 CXL 内存。
- **HWcc / SWcc 拆分**:一行被迁移到 CXL 后,会拆成两部分——*HWcc record*(跨主机同步所需
  的元数据,放在小而稀缺的硬件缓存一致区)和 *SWcc row*(实际数据,放在非硬件一致的 CXL 区)。
  `HW_CC_BUDGET` 参数限定 HWcc 区大小。
- **软件缓存一致性 (SCC)**:让 SWcc row 可被缓存读取的协议。可选 `WriteThrough`(默认)、
  `WriteThroughNoSharedRead`、`NonTemporal`、`NoOP`。
  代码:`protocol/Pasha/SCCManager.{h,cpp}`、`protocol/Pasha/SCCManagerFacrtory.h`
  (注意文件名拼写为 `Facrtory`)、`protocol/Pasha/SCC{NoOP,NonTemporal,WriteThrough}.h`,
  以及 `protocol/TwoPLPasha/TwoPLPashaSCC*.h` 各变体。
- **数据迁移**:将行移入/移出 CXL,策略有 `Clock` / `LRU` / `FIFO` / `Eagerly` / `NoMoveOut`。
  代码:`protocol/Pasha/MigrationManager.{h,cpp}`、`MigrationManagerFactory.h`、
  `protocol/Pasha/Policy*.h`。
- **并发控制协议**:`TwoPLPasha` = Tigon(主系统);`SundialPasha` = 跑在 Pasha 上的 Sundial+;
  `TwoPL` / `Sundial` = 基线;`TwoPLPashaPhantom` = 关闭幻读避免的 Tigon。

## 仓库结构

- `core/` — 核心抽象:`Coordinator.h`、`Worker.h`、`Executor.h`、`Dispatcher.h`、`Table.h`、
  `CXLTable.h`、`Context.h`、`Partitioner.h`,以及 `group_commit/`(epoch 组提交)。
- `protocol/` — 每个并发控制协议一个子目录;`Pasha/` 存放共享的迁移与 SCC 机制。
  **注意**:`CMakeLists.txt` 只编译 `Pasha/`、`SundialPasha/`、`TwoPLPasha/`、`common/`、
  `core/` 下的 `.cpp`;其余协议目录(Aria、Calvin、Silo、Star 等)是 header-only / 遗留代码。
- `common/` — CXL 内存与传输(`CXLMemory.*`、`CXLTransport.*`、`CXL_EBR.*`、
  `MPSCRingBuffer.h`)、数据结构(`btree_olc/`、`btree_olc_cxl/`、`CCHashTable.h`、`CCSet.h`)、
  `WALLogger.h`、消息/序列化、同步原语。
- `benchmark/` — `tpcc/`、`ycsb/`、`smallbank/`、`tatp/`,每个含
  `Database/Schema/Context/Transaction/Workload/Query/Storage/Random` 头文件。
- `bench_*.cpp` — 可执行入口。**仅 `bench_tpcc` 和 `bench_ycsb` 会被构建**;
  smallbank/tatp 在 `CMakeLists.txt` 中已注释。
- `scripts/` — `run.sh`(构建/运行/kill)、`setup.sh`、`push_button.sh`、各图对应的
  `run_*.sh`、`parse/`、`plot/`。
- `emulation/` — 基于 QEMU/ivshmem 的 VM 版 CXL pod 模拟。
- `dependencies/` — `cxlalloc/`(预编译的 `libcxlalloc_static.a` CXL 分配器)与 `kernel_module/`。

## 构建与运行

- 工具链:**clang-15 / clang++-15**、**C++14**、`lld-15`,`-O3 -march=native -DNDEBUG`
  (见 `CMakeLists.txt`)。链接 `cxlalloc`、`jemalloc`、`glog`、`gflags`。
- 构建:`./scripts/run.sh COMPILE`(在 `build/` 中执行 `cmake .. && make -j`);
  或手动 `mkdir -p build && cd build && cmake .. && make -j`。
  产物:`build/bench_tpcc`、`build/bench_ycsb`。
- 运行 / 冒烟测试:`./scripts/run.sh TPCC ...` 或 `YCSB ...`(完整参数列表见 `README.md`
  和 `run.sh` 的 `print_usage`);`./scripts/run.sh CI HOST_NUM WORKER_NUM` 是轻量正确性检查;
  `./scripts/run.sh KILL` 停止运行中的实验。
- 跑真实实验需要 `README.md` 里的 VM/CXL pod 环境;**仅构建不需要 CXL 硬件**。
- 注意:当前开发环境可能未安装 clang-15,届时无法本地验证构建——如遇此情况请如实说明,
  不要谎称构建成功。

## 代码约定

- 格式化:仓库带 `.clang-format`——改动 C++ 后请运行 clang-format;制表符缩进(宽度 8)、
  160 列上限、函数后换行的大括号风格。
- 头文件用 `#pragma once`;include 路径相对仓库根(`core/...`、`common/...`)。
- 命名空间:根 `star::`,各 benchmark 用 `star::tpcc::` / `star::ycsb::`。
- 命名:类型 `PascalCase`,函数/变量 `camelCase`,宏/枚举 `UPPER_CASE`。
- 日志/断言:用 glog——`LOG(INFO)`、`CHECK`、`DCHECK`。
- `core/Context.h` 是贯穿全局的中心配置结构;新增可调项要加在这里。

## 修改时的注意事项

- 这是复现特定论文图表的研究代码——务必保持 `TwoPLPasha`(Tigon)及各基线的行为,
  不要随意"清理"协议代码。
- 在协议目录下新增源文件时,确认它落在 `CMakeLists.txt` 的 `GLOB_RECURSE` 范围内
  (仅 `Pasha`、`SundialPasha`、`TwoPLPasha`、`common`、`core` 被 glob)。
- 新增一个配置开关通常要改三处:`core/Context.h`、对应的 `bench_*.cpp` gflag、
  以及 `scripts/run.sh` 的参数解析。
- 不要修改 `dependencies/cxlalloc/`(预编译静态库)和 `results/`(论文数据)。
