# Tigon 核心类与实例关系整理

> 本文整理 Tigon 运行时涉及的核心类（`Coordinator` / `Worker` / `Executor` / `Manager` / `Database` / `Table` / `CXLTable` / `CXLTableBTreeOLC` / `BPlusTree` 等），及其**实例的创建、调用、交互、数量对应关系**。所有结论均带 `file:line` 源码引用。
>
> 关键词约定：`H` = host/进程数（= `coordinator_num` = `context.peers.size()`）；`W` = 每节点 worker 线程数（= `context.worker_num` = `--threads`）；`P` = 全局分区数（= `context.partition_num`）；`IO` = IO 线程数（= `context.io_thread_num`）。

---

## 0. 总览：一张图看懂数量关系

```
                          整个系统（一个 CXL pod）
                                   │
          ┌────────────────────────┼────────────────────────┐
          │ Host 0 (进程)          │ Host 1 (进程)   ...      │ Host H-1
          │                        │                          │
   ┌──────┴───────┐         ┌──────┴───────┐
   │ 1×Coordinator│         │ 1×Coordinator│   ← 每进程恰好 1 个
   │ 1×Database   │         │ 1×Database   │   ← 每进程恰好 1 个（私有内存侧）
   │ 1×Context    │         │ 1×Context    │
   └──────┬───────┘         └──────────────┘
          │
          │ Coordinator.workers (vector, 共 W+1 个元素)
          ├── Executor[0]   ┐
          ├── Executor[1]   │  W 个 Executor 线程（实际执行事务）
          ├── ...           │  每个 Executor 独占 1 个：protocol、workload、partitioner
          ├── Executor[W-1] ┘
          └── Manager       ← 1 个 Manager（协调/统计，不跑事务）
          │
          ├── iDispatchers  ← IO 个 IncomingDispatcher 线程
          └── oDispatchers  ← 0 或 IO 个 OutgoingDispatcher 线程

   全局单例（每进程一份，跨 worker 共享）：
     migration_manager / scc_manager / twopl_pasha_global_helper
     cxl_memory / cxl_transport / global_ebr_meta

   CXL 共享内存侧（所有 Host 通过 offset_ptr 映射到同一物理内存）：
     cxl_tbl_vecs[table_id][partition_id] → CXLTableBTreeOLC → btreeolc_cxl::BPlusTree
```

**核心数量公式：**

| 对象 | 数量 | 作用域 |
|---|---|---|
| Host / 进程 | `H` | 系统级 |
| Coordinator | `1` 每进程，全系统 `H` | 进程级（单例） |
| Database（私有内存侧） | `1` 每进程，全系统 `H` | 进程级（单例） |
| Context | `1` 每进程 | 进程级（单例） |
| Worker（= Executor + Manager） | `W+1` 每进程 | 线程级 |
| Executor | `W` 每进程，全系统 `H×W` | 线程级 |
| Manager | `1` 每进程 | 线程级 |
| 协议实例（如 `TwoPLPasha`） | `W` 每进程（每 Executor 1 个） | 线程级 |
| Partitioner | `W` 每进程（每 Executor 1 个，逻辑相同） | 线程级 |
| IncomingDispatcher | `IO` 每进程 | 线程级 |
| OutgoingDispatcher | `0` 或 `IO` 每进程 | 线程级 |
| `migration_manager` / `scc_manager` / `twopl_pasha_global_helper` | `1` 每进程（全局单例） | 进程级 |
| Table 实例（私有侧，TPC-C） | `11 × P` 每进程 | Database 内 |
| BPlusTree（私有侧） | 每个 `TableBTreeOLC` 内含 `1` 个 | Table 内 |
| CXLTableBTreeOLC（CXL 侧） | 每张需迁移的表 `× P` | CXL 共享内存 |
| btreeolc_cxl::BPlusTree（CXL 侧） | 每个 `CXLTableBTreeOLC` 内含 `1` 个指针 | CXL 共享内存（全 Host 共享同一份） |

---

## 1. 进程入口与顶层对象（1:1:1）

入口 `main()`（以 TPC-C 为例）按固定顺序创建三个进程级单例对象：

- `bench_tpcc.cpp:29` — `tpcc::Context context;`（1 个）
- `bench_tpcc.cpp:30` — `SETUP_CONTEXT(context)` 把命令行 flags 灌入 Context（宏定义 `core/Macros.h:95-162`）
- `bench_tpcc.cpp:55` — `tpcc::Database db;`（1 个）
- `bench_tpcc.cpp:56` — `db.initialize(context)` 建表（按 partition × table 创建所有 Table 实例）
- `bench_tpcc.cpp:61` — `star::Coordinator c(FLAGS_id, db, context);`（1 个）
- `bench_tpcc.cpp:62-63` — `c.connectToPeers(); c.start();`

> **结论**：一个进程 = 1 Coordinator + 1 Database + 1 Context。`db` 与 `context` 以**引用**形式贯穿后续所有对象，从不复制。

### Context 关键数量字段（`core/Context.h:16-126`）

| 字段 | 含义 |
|---|---|
| `coordinator_id` | 本节点 ID（0 .. H-1） |
| `coordinator_num` (`H`) | 节点/进程总数 = `peers.size()` |
| `worker_num` (`W`) | 本节点 worker 线程数（`--threads`） |
| `partition_num` (`P`) | 全局分区数 |
| `io_thread_num` (`IO`) | IO 线程数 |
| `peers` | 所有节点 `host:port` 列表 |
| `migration_policy` / `when_to_move_out` / `enable_scc` / `scc_mechanism` / `hw_cc_budget` | Pasha 迁移与软件缓存一致配置（`Context.h:106-125`） |

---

## 2. Coordinator（每进程 1 个）

- 定义：`core/Coordinator.h:31-671`
- 构造：`Coordinator.h:34`，`coordinator_num` 由 `context.peers.size()` 初始化（`Coordinator.h:36`）

**核心成员（`Coordinator.h:648-670`）：**

| 成员 | 类型 | 数量 |
|---|---|---|
| `id` / `coordinator_num` | `size_t` | 本节点 ID / 节点总数 |
| `workers` | `vector<shared_ptr<Worker>>` | `W+1`（W 个 Executor + 1 Manager） |
| `iDispatchers` | `vector<unique_ptr<IncomingDispatcher>>` | `IO` |
| `oDispatchers` | `vector<unique_ptr<OutgoingDispatcher>>` | `0` 或 `IO` |
| `in_queue` / `out_queue` / `out_to_in_queue` | `LockfreeQueue<Message*>` | 消息通道 |
| `inSockets` / `outSockets` | `vector<vector<Socket>>` | 维度 `[IO][H]` |
| `cxl_ringbuffers` | `MPSCRingBuffer*` | 启用 CXL 传输时 `H` 个（`Coordinator.h:416-424`） |

**创建 Worker（`Coordinator.h:101-102`）：**
```cpp
LOG(INFO) << "Coordinator initializes " << context.worker_num << " workers.";
workers = WorkerFactory::create_workers(id, db, context, workerStopFlag);
```

**线程模型（`Coordinator.h:178-224`）：**
- `W+1` 个 Worker 线程（每个 Executor 一个 + Manager 一个）
- `IO` 个 IncomingDispatcher 线程（`Coordinator.h:190-191`）；消息按 `workerId % io_thread_num == group_id` 路由（`Coordinator.h:82`）
- `0`/`IO` 个 OutgoingDispatcher 线程（取决于 `use_output_thread`，`Coordinator.h:192-205`）
- `0`/`1` 个 Logger 线程（取决于 WAL 配置，`Coordinator.h:208-212`）

---

## 3. Worker 体系（基类 + Executor + Manager）

### 继承关系

```
Worker (core/Worker.h:16-77)  ← 抽象基类，持 coordinator_id / id / 统计计数器
  ├── Executor<Workload, Protocol>  (core/Executor.h:24-430)  ← 跑事务
  │     └── 协议特化子类，如 TwoPLPashaExecutor<Workload> (protocol/TwoPLPasha/TwoPLPashaExecutor.h:19)
  │                          SiloExecutor / SundialExecutor / ...
  └── Manager (core/Manager.h:20-335)  ← 协调/同步/统计，不跑事务
        └── 协议特化，如 CalvinManager / StarManager / AriaManager
```

### WorkerFactory 创建逻辑（以 TwoPLPasha 为例，`core/factory/WorkerFactory.h:180-191`）

```cpp
auto manager = std::make_shared<Manager>(coordinator_id, context.worker_num, context, stop_flag);
for (auto i = 0u; i < context.worker_num; i++) {
    workers.push_back(std::make_shared<TwoPLPashaExecutor<WorkloadType>>(
        coordinator_id, i, db, context,
        manager->worker_status, manager->n_completed_workers, manager->n_started_workers));
}
workers.push_back(manager);   // 末尾追加 1 个 Manager
```

> **结论**：`workers` 共 `W+1` 个元素 —— 前 `W` 个是 Executor，最后 1 个是 Manager。所有 Executor 与该 Manager **共享同一组同步变量**（`worker_status` / `n_completed_workers` / `n_started_workers`），这是 Manager 协调各 Executor 的方式。
>
> 参数来源（`core/Macros.h:95-162`）：`worker_num = FLAGS_threads`，`coordinator_num = peers.size()`。

### Executor 持有的对象（每 Executor 各 1 份）

`core/Executor.h:37-55` 构造函数中：
- `ProtocolType protocol(db, context, *partitioner)`（`Executor.h:47`）— **每 Executor 独占 1 个协议实例**
- `WorkloadType workload`（每 Executor 1 个）
- `partitioner = PartitionerFactory::create_partitioner(...)`（`Executor.h:45`）— 每 Executor 1 个（逻辑相同，可视为可共享）
- `in_queue` / `out_queue`（每 Executor 私有消息队列）
- `db` / `context`（引用，全进程共享）

---

## 4. 协议层实例（TwoPLPasha 等）

- `TwoPLPasha` 定义：`protocol/TwoPLPasha/TwoPLPasha.h:21-37`
- 构造仅持引用：`TwoPLPasha(DatabaseType &db, const ContextType &context, Partitioner &partitioner)`（`TwoPLPasha.h:32-37`）

**持有关系链：**
```
Coordinator → workers[i] = TwoPLPashaExecutor[i] → protocol = TwoPLPasha<Database>[i]
                                                       ├── db        (共享引用)
                                                       ├── context   (共享引用)
                                                       └── partitioner(本 Executor 私有)
```

> **数量**：协议实例数 = Executor 数 = `W` 每进程，全系统 `H×W`。各协议实例**独立维护事务状态**（TID 生成、max_tid 等），但共享同一 Database/Context。

### Pasha 全局单例（每进程 1 个，仅 Executor 0 初始化）

`protocol/TwoPLPasha/TwoPLPashaExecutor.h:42-74`：仅当 `id == 0` 的 Executor 负责初始化全局指针，其余 Executor 调用 `wait_for_pasha_metadata_init()` 等待。

| 全局单例 | 声明 | 定义 |
|---|---|---|
| `migration_manager` | `protocol/Pasha/MigrationManager.h:90` | `MigrationManager.cpp:7` |
| `scc_manager` | `protocol/Pasha/SCCManager.h:91` | `SCCManager.cpp:7` |
| `twopl_pasha_global_helper` | `protocol/TwoPLPasha/TwoPLPashaHelper.h:2040` | `TwoPLPashaHelper.cpp:12`（初始化 `TwoPLPashaExecutor.h:46-48`） |

> **结论**：这三者是**进程级全局单例**（不是每 worker 一个），被本进程所有 Executor 共享。多个 Host 各有自己独立的一套。

### 其它进程级全局对象

| 对象 | 定义 |
|---|---|
| `CXLMemory cxl_memory` | `common/CXLMemory.cpp:10` |
| `CXLTransport *cxl_transport` | `common/CXLTransport.cpp:1` |
| `CXL_EBR *global_ebr_meta` | `common/CXL_EBR.cpp:9` |

---

## 5. Partitioner（每 Executor 1 个）

- 定义：`core/Partitioner.h:15-71`（基类）及多个实现（`Partitioner.h:80-470`）
- 创建：`core/Executor.h:45`，`PartitionerFactory::create_partitioner(context.partitioner, coordinator_id, coordinator_num)`

**职责**：决定分区→Coordinator 映射、主副本位置、数据本地/远程判定。
- `master_coordinator(partition_id)` / `has_master_partition(partition_id)` / `is_partition_replicated_on(partition_id, coordinator_id)`
- 哈希分区：主副本位置 = `partition_id % coordinator_num`，每节点约持 `P/H` 个主分区。

> **数量**：每 Executor 各建 1 个 Partitioner 实例（共 `W` 每进程），逻辑无状态、计算结果一致。

---

## 6. 数据存储层：Database → Table → BPlusTree

### 6.1 Database（每进程 1 个，按 benchmark 各有定义）

| Benchmark | 定义 | 每分区表数 |
|---|---|---|
| TPC-C | `benchmark/tpcc/Database.h:35-1725` | 11 |
| YCSB | `benchmark/ycsb/Database.h:30-345` | 1 |
| SmallBank | `benchmark/smallbank/Database.h:30-326` | 2 |
| TATP | `benchmark/tatp/Database.h` | （同模式） |

**Table 组织（TPC-C，`benchmark/tpcc/Database.h:457,1708`）：**
```cpp
std::vector<std::vector<ITable*>> tbl_vecs;   // [table_id][partition_id]
tbl_vecs.resize(11);                          // 11 种表
```
- 访问：`find_table(table_id, partition_id) → tbl_vecs[table_id][partition_id]`（`Database.h:56-61`）
- 11 种表索引：0 warehouse / 1 district / 2 customer / 3 customer_name_idx / 4 history / 5 new_order / 6 order / 7 order_cust / 8 order_line / 9 item / 10 stock
- 建表循环 `partitionID = 0 .. P-1`（`Database.h:171-434`），每分区各建一套表
- **特例**：item 表只读，仅建 1 个实例（partition 0）共享（`Database.h:436-454`）

> **数量**：Table 实例数 = `每分区表数 × P`。TPC-C 即 `11 × P`（item 表除外，全局 1 个）。

### 6.2 Table 抽象基类与实现（`core/Table.h`）

```
ITable (core/Table.h:23-126)  ← 抽象基类
  ├── TableHashMap   (Table.h:172-366)  无序哈希表；底层 HashMap map_ (Table.h:363)
  ├── TableBTreeOLC  (Table.h:368-785)  有序 B+树，支持 scan / next-key locking；底层 BTree btree (Table.h:782)
  ├── HStoreTable    (Table.h:787-953)  HStore 协议专用
  └── HStoreCOWTable (Table.h:955-1178) Copy-On-Write / checkpoint
```

接口（virtual，`Table.h:52-100`）：`search* / scan / insert* / remove* / update*`。

### 6.3 BPlusTree（每个 TableBTreeOLC 内含 1 个）

- 标准内存版：`btreeolc::BPlusTree`（`common/btree_olc/BTreeOLC.h:165-167`）—— 乐观锁（版本号）内部节点 / 叶子节点，页大小 4096。
- `TableBTreeOLC` 内：`using BTree = btreeolc::BPlusTree<KeyType, BTreeOLCValue, ...>`（`Table.h:410`），成员 `BTree btree;`（`Table.h:782`）。

> **数量**：1 个 `TableBTreeOLC` 实例 = 1 个 `BPlusTree` 实例（值类型成员，非指针）。

### 6.4 CXL 共享内存侧：CXLTable → CXLTableBTreeOLC → btreeolc_cxl::BPlusTree

```
CXLTableBase (core/CXLTable.h:15-30)  ← CXL 表抽象基类
  ├── CXLTableHashMap   (CXLTable.h:32-81)  底层 CCHashTable* cxl_hashtable_ (CXLTable.h:78)
  └── CXLTableBTreeOLC  (CXLTable.h:83-203) 底层 CXLBTree* cxl_btree_ (CXLTable.h:200)
```

`CXLTableBTreeOLC<KeyType, KeyComparator>`（`CXLTable.h:83-203`）核心：
- 值类型 `BTreeOLCValue { offset_ptr<void> row; atomic<bool> is_valid; }`（`CXLTable.h:91-109`）—— 用 `boost::interprocess::offset_ptr` 实现**位置无关**，跨 Host 映射有效。
- `using CXLBTree = btreeolc_cxl::BPlusTree<...>`（`CXLTable.h:121`）
- 成员 `CXLBTree *cxl_btree_;`（`CXLTable.h:200`）—— **指针**，指向 CXL 共享内存中的 B+树
- 构造接受现成指针 `CXLTableBTreeOLC(CXLBTree *cxl_btree, tableID, partitionID)`（`CXLTable.h:125-130`）
- `search/scan/insert/remove` 全部转发到 `cxl_btree_`（`CXLTable.h:132-187`）

CXL B+树定义：`btreeolc_cxl::BPlusTree`（`common/btree_olc_cxl/BTreeOLC_CXL.h:166-177`）。

**CXL 表的创建与共享（`benchmark/tpcc/Database.h:897-942`）：**
- Coordinator 0 在 CXL 内存分配 `sizeof(CXLBTree) × P`（`Database.h:935-936`），其余 Host 通过 `offset_ptr` 映射到**同一份**物理 B+树（这正是 CLAUDE.md 所述 host0 `commit_shared_data_initialization` / 其余 host `wait_and_retrieve` 协议）。
- Database 另持 `cxl_tbl_vecs[table_id][partition_id]`（CXLTableBase*），结构与私有侧 `tbl_vecs` 对称。

> **数量**：1 个 `CXLTableBTreeOLC` = 1 个 `btreeolc_cxl::BPlusTree`（指针引用）。但与私有侧不同，**全系统 H 个 Host 共享同一份 CXL B+树物理实例**（每 partition 一份，存于 CXL 共享内存），而私有侧 `BPlusTree` 是每进程各一份。

---

## 7. 端到端数量对应关系（公式汇总）

```
Host / 进程数                = H  (= coordinator_num = peers.size())
Coordinator                  = H            (每进程 1)
Database (私有内存侧)        = H            (每进程 1)
Context                      = H            (每进程 1)

Worker 线程 (Executor+Manager)= H × (W+1)
  ├ Executor                 = H × W
  └ Manager                  = H × 1
协议实例 (TwoPLPasha 等)     = H × W        (每 Executor 1)
Partitioner                  = H × W        (每 Executor 1，逻辑相同)
IncomingDispatcher 线程      = H × IO
OutgoingDispatcher 线程      = H × {0 | IO}

进程级全局单例 (每项)        = H            (每进程 1：migration/scc/helper/cxl_memory/...)

Table 实例 (私有侧, TPC-C)   = H × (11 × P) (item 表除外，全局 1)
BPlusTree (私有侧)           = 每个 TableBTreeOLC 内 1 个 (值成员)

CXLTableBTreeOLC (CXL 侧)    = 每张迁移表 × P  (存于 CXL 共享内存)
btreeolc_cxl::BPlusTree      = 每个 CXLTableBTreeOLC 内 1 个指针；
                               全 H 个 Host 共享同一物理实例 (每 partition 一份)
```

**典型配置示例** `--servers="n0:10010;n1:10010;n2:10010" --threads=8 --io=2 --partition_num=12`（TwoPLPasha）：
- `H=3, W=8, IO=2, P=12`
- 每进程：1 Coordinator + 1 Database + 9 Worker（8 Executor + 1 Manager）+ 8 协议实例 + 8 Partitioner + 2 入 Dispatcher（+ 可选 2 出 Dispatcher）
- 每进程 Table 实例：`11 × 12 = 132`（item 表全局 1 个）；每个 TableBTreeOLC 各含 1 个 BPlusTree
- 全局单例：每进程各 1 个 migration_manager / scc_manager / twopl_pasha_global_helper
- CXL 侧：每张迁移表 × 12 个 partition 的 CXLTableBTreeOLC + 对应 btreeolc_cxl::BPlusTree，3 个 Host 共享同一份物理 B+树

---

## 8. 交互关系（调用链）

1. **启动**：`main` → `Coordinator::start()` → `WorkerFactory::create_workers()` 建 `W` Executor + 1 Manager → 各起线程跑 `Worker::start()`。
2. **事务执行**：`Executor` 调用其私有 `protocol`（如 `TwoPLPasha`）→ 经 `partitioner` 判定本地/远程 → 读写经 `db.find_table(table_id, partition_id)` 定位 `ITable` → `TableBTreeOLC::search/insert/...` → 内部 `btreeolc::BPlusTree`。
3. **CXL 迁移**：数据迁移由进程级 `migration_manager` 调度，跨 Host 数据落在 CXL 侧 `cxl_tbl_vecs[...]` → `CXLTableBTreeOLC` → `btreeolc_cxl::BPlusTree`（offset_ptr 寻址）；缓存一致由 `scc_manager` 维护。
4. **跨节点消息**：Executor 把 `Message` 投入自身 `out_queue` → `OutgoingDispatcher`（或同步发送）→ 网络/CXL 环形缓冲 → 对端 `IncomingDispatcher` → 按 `workerId % io_thread_num` 路由进目标 Executor 的 `in_queue`。
5. **同步/统计**：`Manager` 通过与所有 Executor 共享的 `worker_status` / `n_completed_workers` / `n_started_workers` 协调各阶段，并跨 Host 同步运行进度。

---

## 9. 关键源码索引

| 组件 | 文件 | 行号 |
|---|---|---|
| main 入口 (TPC-C) | `bench_tpcc.cpp` | 23-67 |
| SETUP_CONTEXT 宏 | `core/Macros.h` | 95-162 |
| Context | `core/Context.h` | 16-126 |
| Coordinator | `core/Coordinator.h` | 31-671（构造 34-112，启动 178-408，成员 648-670） |
| WorkerFactory（TwoPLPasha 分支） | `core/factory/WorkerFactory.h` | 180-191 |
| Worker 基类 | `core/Worker.h` | 16-77 |
| Executor | `core/Executor.h` | 24-430（协议/分区器创建 37-55） |
| Manager | `core/Manager.h` | 20-335 |
| Partitioner | `core/Partitioner.h` | 15-71 |
| TwoPLPasha | `protocol/TwoPLPasha/TwoPLPasha.h` | 21-37 |
| TwoPLPashaExecutor（全局单例初始化） | `protocol/TwoPLPasha/TwoPLPashaExecutor.h` | 19-75（42-74） |
| migration_manager | `protocol/Pasha/MigrationManager.h:90` / `.cpp:7` | — |
| scc_manager | `protocol/Pasha/SCCManager.h:91` / `.cpp:7` | — |
| twopl_pasha_global_helper | `protocol/TwoPLPasha/TwoPLPashaHelper.h:2040` / `.cpp:12` | — |
| ITable + 实现 | `core/Table.h` | 23-126；TableHashMap 172-366；TableBTreeOLC 368-785 |
| BPlusTree（私有） | `common/btree_olc/BTreeOLC.h` | 165-167 |
| CXLTableBase + 实现 | `core/CXLTable.h` | 15-30；HashMap 32-81；BTreeOLC 83-203 |
| btreeolc_cxl::BPlusTree（CXL） | `common/btree_olc_cxl/BTreeOLC_CXL.h` | 166-177 |
| Database (TPC-C) | `benchmark/tpcc/Database.h` | 35-1725（tbl_vecs 457/1708，建表 171-454，CXL 表 897-942） |
| Database (YCSB / SmallBank) | `benchmark/ycsb/Database.h` / `benchmark/smallbank/Database.h` | 30-345 / 30-326 |
