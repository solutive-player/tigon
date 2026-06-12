# Tigon 全部命令行参数（`--flag`）详解

> 本文穷举 Tigon 所有可通过 `--xxx` 传入的 gflags 参数：**类型、默认值、语义、合法取值、实际消费点（file:line）、是否被 `scripts/run.sh` 固定或暴露**。
>
> - 通用 flag 定义在 `core/Macros.h:11-93`，经 `SETUP_CONTEXT` 宏（`Macros.h:95-162`）映射到 `context.*` 字段，被所有 `bench_*` 共享。
> - benchmark 专属 flag 定义在各自 `bench_*.cpp` 顶部。
> - **CLI 默认值 = `DEFINE_*` 里的默认值**（`SETUP_CONTEXT` 总会执行，覆盖 `Context.h` 里的成员初值）。
> - 标注 ⚙️ = `run.sh` 写死不可调；🎛️ = `run.sh` 经位置参数暴露；🚫 = 定义了但代码中未实际消费（遗留/预留）。

---

## A. 集群拓扑与线程

| flag | 类型 | 默认 | 语义 / 取值 | 消费点 | run.sh |
|---|---|---|---|---|---|
| `servers` | string | `127.0.0.1:10010` | 分号分隔的所有 coordinator 地址 `ip:port;ip:port…`。`peers.size()` 即 host 数 | `SETUP_CONTEXT` split → `context.peers`（Macros.h:96-97） | 🎛️ 由 `print_server_string` 按 HOST_NUM 生成 `192.168.100.2:1234;…` |
| `id` | int32 | `0` | 本节点 coordinator ID（0 .. coordinator_num-1） | `context.coordinator_id`（Macros.h:98）；启动 host0 前台、其余后台 | 🎛️ 循环变量 i |
| `threads` | int32 | `1` | 每节点 worker（Executor）线程数 | `context.worker_num`（Macros.h:99）→ WorkerFactory 建 W 个 Executor | 🎛️ = WORKER_NUM |
| `io` | int32 | `1` | IO dispatcher 线程数（每个起一对 In/OutgoingDispatcher） | `context.io_thread_num`（Macros.h:100）；Coordinator.h:105-110/181-182 分配 socket 组与 dispatcher | ⚙️ 默认 1（**CXL 传输要求 io==1**，bench_ycsb.cpp:28） |
| `partition_num` | int32 | `1` | 全局分区总数 | `context.partition_num`（Macros.h:101）；Partitioner 据此映射 | 🎛️ TPCC=HOST×WORKER，YCSB=HOST |
| `partitioner` | string | `hash` | 分区策略：`hash`=`HashPartitioner`(partition_id%coord_num=主，无副本)；`hash2`=`HashReplicatedPartitioner<2>`(2 副本)；`pb`=`PrimaryBackupPartitioner`(需 2 节点严格主备) | `PartitionerFactory::create_partitioner`（Partitioner.h:474）；`set_star_partitioner()` 对 Star 协议改写为 StarS/StarC（Context.h:18-28） | ⚙️ 固定 hash |
| `cpu_affinity` | bool | `false` | 线程绑核开关 | `context.cpu_affinity`（Macros.h:126）🚫 主路径未消费 | ⚙️ 默认 |
| `cpu_core_id` | int32 | `0` | 绑核目标核 ID | `context.cpu_core_id`（Macros.h:128）🚫 主路径未消费 | ⚙️ 默认 |

> 数量关系：H 个 host × 各 1 Coordinator/Database；每 Coordinator W 个 Executor + 1 Manager + IO 对 dispatcher（详见 `docs/class_relationships.md`）。

---

## B. 协议选择与通用事务控制

| flag | 类型 | 默认 | 语义 / 取值 | 消费点 | run.sh |
|---|---|---|---|---|---|
| `protocol` | string | `Scar` | 并发控制协议名，`WorkerFactory.h` 编译期按字符串实例化。复现常用：`TwoPLPasha`(Tigon)/`TwoPLPashaPhantom`/`SundialPasha`/`Sundial`/`TwoPL`；另有 Silo/Star/Calvin/Aria/H-Store 等 | `context.protocol`（Macros.h:108）→ WorkerFactory 分支 | 🎛️ = SYSTEM（Phantom 走 TwoPLPasha+关 phantom） |
| `sleep_on_retry` | bool | `true` | 事务 abort 后重试前是否 sleep | `context.sleep_on_retry`（Macros.h:103） | ⚙️ 默认 |
| `sleep_time` | int32 | `100` | abort 重试 sleep 时长（ms） | `context.sleep_time`（Macros.h:107） | ⚙️ 默认 |
| `batch_size` | int32 | `100` | Star/Calvin/Aria 每相位批量事务数 | `context.batch_size`（Macros.h:104）；Star 动态调整 | ⚙️ 固定 0（非批量协议忽略） |
| `batch_flush` | int32 | `50` | 执行中每处理多少条事务 flush 一次消息 | `context.batch_flush`（Macros.h:106）；group_commit/Executor.h:157、StarExecutor.h:232、AriaExecutor 多处 | ⚙️ 固定 1 |
| `group_time` | int32 | `10` | Star/Calvin 批量分组定时器（**毫秒**），与 WAL 的 wal_group_commit_time 无关 | `context.group_time`（Macros.h:105）；group_commit/Manager.h:33 sleep、StarManager.h:46 动态批大小 | ⚙️ 默认 |
| `replica_group` | string | `1,3` | Calvin 各副本组大小（逗号分隔整数），`CalvinHelper::string_to_vint` 解析 | Macros.h:109；CalvinExecutor.h:64 建 CalvinPartitioner | ⚙️ 固定 1 |
| `lock_manager` | string | `1,1` | Calvin 各副本组的 lock manager 数 | Macros.h:110；CalvinExecutor.h:66 `n_lock_manager` | ⚙️ 固定 0 |
| `calvin_same_batch` | bool | `false` | Calvin 始终重跑同一批事务（测试用） | `context.calvin_same_batch`（Macros.h:117） | ⚙️ 默认 |
| `read_on_replica` | bool | `false` | 2PL/Silo 允许从副本读 | `context.read_on_replica`（Macros.h:111）；SiloExecutor.h:50 配合 `is_partition_replicated_on` | ⚙️ 默认 |
| `local_validation` | bool | `false` | 本地验证 | Macros.h:112 🚫 未消费（遗留） | ⚙️ 默认 |
| `rts_sync` | bool | `false` | RTS 同步 | Macros.h:113 🚫 未消费（遗留） | ⚙️ 默认 |
| `plv` | bool | `true` | 并行加锁与验证 | `context.parallel_locking_and_validation`（Macros.h:116）🚫 未见显式消费 | ⚙️ 默认 |

---

## C. CXL 传输（Tigon 核心）

| flag | 类型 | 默认 | 语义 | 消费点 | run.sh |
|---|---|---|---|---|---|
| `use_cxl_transport` | bool | `false` | true=消息走 CXL 共享内存 MPSC 环形缓冲；false=走 TCP socket | `context.use_cxl_transport`（Macros.h:147）；Coordinator 据此建 ringbuffer 或 socket | 🎛️ = USE_CXL_TRANS |
| `use_output_thread` | bool | `false` | true=把输出线程改造成专职消息发送线程（**必须配 use_cxl_transport=1**） | `context.use_output_thread`（Macros.h:148） | 🎛️ = USE_OUTPUT_THREAD |
| `cxl_trans_entry_struct_size` | uint64 | `8192` | 单个环形缓冲 entry 字节数 | `context.cxl_trans_entry_struct_size`（Macros.h:149）；MPSCRingBuffer 分配 | ⚙️ Pasha=2048 / 基线=65536 |
| `cxl_trans_entry_num` | uint64 | `4096` | 每环形缓冲 entry 个数 | `context.cxl_trans_entry_num`（Macros.h:150） | ⚙️ 固定 8192 |

---

## D. Pasha 数据迁移

| flag | 类型 | 默认 | 语义 / 取值 | 消费点 | run.sh |
|---|---|---|---|---|---|
| `enable_migration_optimization` | bool | `true` | 迁移数据**写回优化**：true=仅当行被改过才回拷（`is_data_modified_since_moved_out`）；false=总是回拷（保守）。**与是否迁移无关**，只控制回拷效率 | TwoPLPashaHelper.h:1330/1467/1645/1751 | 🎛️ = ENABLE_MIGRATION_OPTIMIZATION（run.sh 固定 1） |
| `migration_policy` | string | `Eagerly` | 迁出 victim 选择 / 迁移时机策略：`Clock`/`LRU`/`FIFO`/`NoMoveOut`(不迁出)/`Eagerly`(立即) | `context.migration_policy`（Macros.h:152）；MigrationManagerFactory 选策略类 | 🎛️ Tigon=Clock，基线=NoMoveOut |
| `when_to_move_out` | string | `Reactive` | 何时迁出：`OnDemand`(CXL 满才迁出)/`Reactive`(提交后发迁出提示)/`Proactive` | `context.when_to_move_out`（Macros.h:153）；MigrationManager.h:38-44 | 🎛️ = WHEN_TO_MOVE_OUT（固定 OnDemand） |
| `hw_cc_budget` | uint64 | `200MB`(1024*1024*200) | 硬件缓存一致（HW-cc）CXL 区字节上限 | `context.hw_cc_budget`（Macros.h:154） | 🎛️ = HW_CC_BUDGET（基线 0 / Tigon 200MB / Fig7 扫 5 档） |
| `pre_migrate` | string | `None` | 实验前预迁移：`None`(不迁)/`NonPart`(迁不可分区数据)/`All`(全迁) | `context.pre_migrate`（Macros.h:161） | 🎛️ = PRE_MIGRATE（Tigon NonPart，第 1 档强制 None） |

> **`enable_migration_optimization=false` ≠ `migration_policy=NoMoveOut`**：前者仍迁移、只是回拷更保守；后者根本不把数据迁出 HW-cc 区。二者独立。

---

## E. Pasha 软件缓存一致（SCC）

| flag | 类型 | 默认 | 语义 / 取值 | 消费点 | run.sh |
|---|---|---|---|---|---|
| `enable_scc` | bool | `true` | 软件缓存一致总开关 | `context.enable_scc`（Macros.h:157）；SCCManagerFactory | 🎛️ = ENABLE_SCC |
| `scc_mechanism` | string | `NoOP` | SCC 机制：`WriteThrough`(默认,写穿透+per-host有效位+允许共享读者)/`WriteThroughNoSharedRead`(禁共享读者)/`NonTemporal`(始终非时序访问)/`NoOP`(不维护,假设硬件全一致) | `context.scc_mechanism`（Macros.h:158） | 🎛️ = SCC_MECH |
| `enable_phantom_detection` | bool | `true` | TwoPLPasha 幻读防护（next-key locking + prev/next real 位） | `context.enable_phantom_detection`（Macros.h:156） | 🎛️ TwoPLPashaPhantom 时置 false |
| `model_cxl_search_overhead` | bool | `false` | true=本地操作也走一遍 CXL 索引 `search`，模拟「总查共享索引」的开销（**真实搜索，非 sleep**）；用作 shortcut 指针优化的对照 | TwoPLPashaExecutor.h:103-104 → Helper.h:1189-1199 `cxl_table->search(key)` | 🎛️ = MODEL_CXL_SEARCH（固定 0=开优化） |

---

## F. 持久化 / WAL / Checkpoint

| flag | 类型 | 默认 | 语义 / 取值 | 消费点 | run.sh |
|---|---|---|---|---|---|
| `log_path` | string | `""` | WAL 落盘目录 | `context.log_path`（Macros.h:122） | ⚙️ 固定 /root/pasha_log |
| `lotus_checkpoint` | int32 | `0` | COW/checkpoint/logging 组合方案：`0`=全关 / `1`=仅日志 / `2`=COW+日志 / `3`=COW+checkpoint(日志关) / `4`=三者全开（Macros.h:55-67 枚举） | `context.lotus_checkpoint`（Macros.h:144）；Coordinator.h:55-97 选 logger | 🎛️ 由 LOGGING_TYPE 派生 0/1 |
| `lotus_checkpoint_location` | string | `""` | checkpoint 文件目录 | `context.lotus_checkpoint_location`（Macros.h:145） | ⚙️ 默认空 |
| `lotus_async_repl` | bool | `false` | Lotus 异步复制（replication 实际未实现） | `context.lotus_async_repl`（Macros.h:143） | ⚙️ 固定 true（形式占位） |
| `wal_group_commit_time` | int32 | `10` | WAL 组提交 epoch（**微秒**），即 master 推进 epoch 的间隔 | `context.wal_group_commit_time`（Macros.h:131）；WALLogger.h:256/262/269/354/468 | 🎛️ = EPOCH_LEN（GROUP_WAL 时） |
| `wal_group_commit_size` | int32 | `7` | WAL 组提交批大小 | `context.group_commit_batch_size`（Macros.h:133） | ⚙️ 由 LOGGING_TYPE 派生 0/10 |
| `persist_latency` | int32 | `110` | 模拟磁盘持久化延迟（微秒） | `context.emulated_persist_latency`（Macros.h:130） | ⚙️ 固定 0（master 用 DirectFileWriter，须 0） |
| `group_commit_batch_size` | — | — | （= wal_group_commit_size 的 context 字段名，无独立 flag） | — | — |

---

## G. 运行时长

| flag | 类型 | 默认 | 语义 | 消费点 | run.sh |
|---|---|---|---|---|---|
| `time_to_run` | int32 | `30` | 总运行秒数（**含**预热） | `context.time_to_run`（Macros.h:159） | 🎛️ = TIME_TO_RUN |
| `time_to_warmup` | int32 | `10` | 预热秒数（不计入统计） | `context.time_to_warmup`（Macros.h:160） | 🎛️ = TIME_TO_WARMUP |

---

## H. TCP 网络选项

| flag | 类型 | 默认 | 语义 | 消费点 | run.sh |
|---|---|---|---|---|---|
| `tcp_no_delay` | bool | `true` | true=禁用 Nagle 算法（降延迟） | `context.tcp_no_delay`（Macros.h:124）；Coordinator.h:486-488 `disable_nagle_algorithm()` | ⚙️ 默认 |
| `tcp_quick_ack` | bool | `false` | true=启用 TCP quick ACK | `context.tcp_quick_ack`（Macros.h:125）；Coordinator.h:471/489 `set_quick_ack_flag` | ⚙️ 默认 |
| `sender_group_nop_count` | int32 | `40000` | TCP 发送端消息分组时空转的 `nop` 指令数（人为引入聚合延迟） | `context.sender_group_nop_count`（Macros.h:139）；Dispatcher.h:247-248 `asm("nop")` 循环 | ⚙️ 默认 |

> CXL 传输模式（`use_cxl_transport=1`）下 TCP 选项基本不生效。

---

## I. 协议专属优化开关

### Star（`protocol/Star`）
| flag | 默认 | 语义 / 消费点 |
|---|---|---|
| `star_sync` | `false` | →`context.star_sync_in_single_master_phase`（Macros.h:114）。true=单主相位同步复制并等响应；false=异步。Star.h:271/283-285 |
| `star_dynamic_batch_size` | `true` | 据 running_time vs group_time 动态调 batch_size。StarManager.h:122-124 |

### Aria（`protocol/Aria`，主路径有效）
| flag | 默认 | 语义 / 消费点 |
|---|---|---|
| `aria_read_only` | `true` | 只读事务跳过 reserve/analyze。AriaExecutor.h:259/307/405 |
| `aria_reordering` | `true` | 无 WAR/RAW 依赖时提前提交。AriaExecutor.h:437-445 |
| `aria_si` | `false` | Aria 快照隔离。🚫 定义未消费（AriaExecutor.h:84） |

### Kiva（基础设施已移除，🚫 全部未实际消费）
| flag | 默认 | 语义 |
|---|---|---|
| `kiva_read_only` | `true` | Kiva 只读优化（Macros.h:118） |
| `kiva_reordering` | `true` | Kiva 重排序（Macros.h:119） |
| `kiva_si` | `false` | Kiva 快照隔离（Macros.h:120） |

### H-Store（`protocol/H-Store`）
| flag | 默认 | 语义 / 消费点 |
|---|---|---|
| `enable_hstore_master` | `true` | 启用 H-Store master 做锁调度，>worker_num 的消息路由给 master worker。Dispatcher.h:71/123/128 |
| `hstore_active_active` | `false` | active-active 复制：每 worker 单事务批、全 worker 跑全部事务。HStoreExecutor.h:627 等 |
| `hstore_command_logging` | `true` | H-Store 命令日志模式。HStoreExecutor.h:851 提交路径 `write_command()` | 
| `enable_hstore_master` 等 | | run.sh 中 `hstore_command_logging` ⚙️固定 false |

---

## J. Straggler（落后者）注入实验参数

| flag | 类型 | 默认 | 语义 / 消费点 |
|---|---|---|---|
| `stragglers_per_batch` | int32 | `0` | 每批注入的 straggler 事务数。CalvinExecutor.h:372-377 随机命中则设 straggler_wait_time |
| `stragglers_num_txn_len` | int32 | `10` | straggler 事务长度类型数。`context.straggler_num_txn_len`（Macros.h:141） |
| `stragglers_partition` | int32 | `-1` | 注入 straggler 的分区 ID，-1=所有分区。ycsb/Transaction.h:178-186 |
| `stragglers_zipf_factor` | double | `0` | straggler 长度的 Zipf 因子（0=均匀）。CalvinExecutor.h:379-383 |

---

## K. Benchmark 专属参数

### 通用（多个 bench 都有）
| flag | bench | 默认 | 语义 |
|---|---|---|---|
| `query` | tpcc/ycsb | tpcc:`neworder` / ycsb:`rmw` | 负载查询类型。TPCC:`mixed`(全5种)/`neworder`/`payment`/`test`；YCSB:`rmw`/`mixed`/`scan` |
| `keys` | ycsb/smallbank/tatp | ycsb:`200000` / sb,tatp:`10000000` | 每分区 key/账户数 |
| `zipf` | ycsb/smallbank/tatp | `0` | Zipfian 偏斜（0=均匀） |
| `cross_ratio` | ycsb/smallbank/tatp | `0` | 跨分区事务/操作百分比 0-100 → `context.crossPartitionProbability`（bench_ycsb.cpp:56） |

### TPC-C（`bench_tpcc.cpp`）
| flag | 默认 | 语义 |
|---|---|---|
| `query` | `neworder` | 见上（run.sh 用 mixed） |
| `neworder_dist` | `10` | 远程 NewOrder 事务百分比 0-100 🎛️ = REMOTE_NEWORDER_PERC |
| `payment_dist` | `15` | 远程 Payment 事务百分比 0-100 🎛️ = REMOTE_PAYMENT_PERC |
| `operation_replication` | `false` | 操作级复制开关 |

### YCSB（`bench_ycsb.cpp`）
| flag | 默认 | 语义 / 消费点 |
|---|---|---|
| `read_write_ratio` | `80` | 读操作百分比 0-100 🎛️ = RW_RATIO |
| `read_only_ratio` | `0` | 只读事务百分比 → `context.readOnlyTransaction`（bench_ycsb.cpp:55） |
| `cross_ratio` | `0` | 跨分区操作% 🎛️ = CROSS_RATIO |
| `keys` | `200000` | 每分区 KV 数 🎛️ = KEYS（run.sh 用 300000） |
| `zipf` | `0` | 偏斜 🎛️ = ZIPF_THETA（常用 0.7） |
| `cross_part_num` | `2` | 跨分区事务涉及的分区数 ⚙️ 固定 2 |
| `nop_prob` | `0` | 事务带 CPU 空转的概率（万分比）。ycsb/Transaction.h:167-173 |
| `n_nop` | `0` | 命中 nop_prob 时执行的 `asm("nop")` 条数。Transaction.h:170 |
| `lotus_sp_parallel_exec_commit` | `false` | Lotus/H-Store 并行执行+提交优化 → `context.lotus_sp_parallel_exec_commit`（bench_ycsb.cpp:58） |

### SmallBank / TATP（`bench_smallbank.cpp` / `bench_tatp.cpp`）
仅 `cross_ratio`(0) / `keys`(10000000) / `zipf`(0) 三个专属 flag，语义同上。

---

## L. 杂项 & 延迟注入

| flag | 类型 | 默认 | 语义 / 消费点 |
|---|---|---|---|
| `delay` | int32 | `0` | 注入的统一网络延迟（微秒）→ `context.delay_time`（Macros.h:121）；Delay.h:34-57 `SameDelay`、CalvinExecutor.h:73 |
| `cdf_path` | string | `""` | CDF 分布延迟文件路径 → `context.cdf_path`（Macros.h:123）🚫 未实际使用 |
| `cross_txn_workers` | int32 | `0` | 生成跨分区事务的 worker 数 → `context.cross_txn_workers`（Macros.h:129）🚫 未实际消费（YCSB 实际用 cross_ratio） |
| `granule_count` | int32 | `1` | 每分区 granule 数（细粒度锁/迁移单元） → `context.granules_per_partition`（Macros.h:142） | ⚙️ 固定 2000 |

---

## M. 遗留 / 未消费 flag 速查（定义了但代码无实际效果）

| flag | 说明 |
|---|---|
| `local_validation` / `rts_sync` / `plv` | 验证相关，主路径未消费 |
| `kiva_read_only` / `kiva_reordering` / `kiva_si` | Kiva 协议基础设施已移除 |
| `aria_si` | Aria 快照隔离，定义未消费 |
| `cross_txn_workers` | YCSB 实际走 `cross_ratio` |
| `cdf_path` | CDF 延迟分布未启用 |
| `cpu_affinity` / `cpu_core_id` | 主协议路径未绑核 |

> 这些 flag 传了也无效果，是 lotus 代码库继承下来的历史残留或预留接口。

---

## N. 复现实验的「核心可调旋钮」总览

`run.sh` 真正暴露给实验者调节的（🎛️）就这 21（TPCC）/23（YCSB）个：
`protocol, HOST_NUM(servers/partition), threads, query, neworder_dist/payment_dist 或 keys/read_write_ratio/zipf/cross_ratio, use_cxl_transport, use_output_thread, enable_migration_optimization, migration_policy, when_to_move_out, hw_cc_budget, enable_scc, scc_mechanism, pre_migrate, time_to_run, time_to_warmup, LOGGING_TYPE(→lotus_checkpoint/wal_*), wal_group_commit_time, model_cxl_search_overhead, GATHER_OUTPUTS`。

其余几十个 flag 要么被 `run.sh` ⚙️ 写死（如 partitioner=hash、granule_count=2000、persist_latency=0），要么 🚫 遗留无效——直接用 `bench_*` 二进制 + `--flag` 才能触及，常用于协议开发/调试而非论文复现。

> 参数到实验脚本的映射详见 `docs/paper_reproduction.md` §3/§5、`docs/experiment_scripts_params.md`；运行时如何消费这些 context 字段详见 `docs/class_relationships.md` 与 `docs/scripts_walkthrough.md`。
