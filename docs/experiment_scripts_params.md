# 四个复现脚本的逐配置·逐参数详解

> 针对 `push_button.sh` 里这四步，把**每一个配置变量、每一次实验调用的每一个位置参数**都拆到底：
> - `./scripts/run_tpcc.sh $RESULT_ROOT_DIR`
> - `./scripts/run_ycsb.sh $RESULT_ROOT_DIR`
> - `./scripts/run_hwcc_budget.sh $RESULT_ROOT_DIR`
> - `./scripts/parse/parse_swcc.py $RESULT_ROOT_DIR`
>
> 三个 run 脚本都经 `scripts/common.sh` 的包装函数→`scripts/run.sh`→各 VM 上的 `bench_tpcc/bench_ycsb`。要理解参数，必须先理解中间这层「参数转发链」。

---

## 0. 参数转发链（务必先读）

```
run_tpcc.sh / run_ycsb.sh / run_hwcc_budget.sh
   │  顶部 typeset 一批配置常量（HOST_NUM、预算、运行时长…）
   ▼  调用包装函数（每次只变 1 个维度）
common.sh: run_remote_txn_overhead_tpcc (17 参)  /  _ycsb (20 参)
   │  把固定项写死（QUERY=mixed、ENABLE_MIG_OPT=1、GATHER=0、KEYS=300000…）
   │  内部对「远程比例」循环 7 次(TPCC) / 11 次(YCSB)，结果都 >> 同一个 .txt
   ▼  每次循环调用一次
run.sh TPCC <21参> / YCSB <23参>
   │  按协议选 CXL 环形缓冲尺寸；按 LOGGING_TYPE 展开 3 个日志 flag
   ▼
run_exp_tpcc/ycsb → ssh 到 8 个 VM 跑 bench_tpcc/bench_ycsb（host0 前台、其余后台）
```

### `run_remote_txn_overhead_tpcc` 的 17 个形参（`common.sh:7-42`）

| 位 | 形参 | 传给 run.sh 的去向 |
|---|---|---|
| 1 | `RESULT_DIR` | 结果落盘目录（不进 binary） |
| 2 | `PROTOCOL` | `run.sh` 第 1 参 → `--protocol` |
| 3 | `HOST_NUM` | host 数 |
| 4 | `WORKER_NUM` | `--threads` |
| 5 | `USE_CXL_TRANS` | `--use_cxl_transport` |
| 6 | `USE_OUTPUT_THREAD` | `--use_output_thread` |
| 7 | `MIGRATION_POLICY` | `--migration_policy` |
| 8 | `WHEN_TO_MOVE_OUT` | `--when_to_move_out` |
| 9 | `MAX_MIGRATED_ROWS_SIZE` | = `HW_CC_BUDGET` → `--hw_cc_budget`（字节） |
| 10 | `ENABLE_SCC` | `--enable_scc` |
| 11 | `SCC_MECHANISM` | `--scc_mechanism` |
| 12 | `PRE_MIGRATE` | `--pre_migrate`（**注意：第 1 档 0/0 强制 None**，见下） |
| 13 | `LOGGING_TYPE` | 展开成 lotus_checkpoint/wal_* |
| 14 | `WAL_GROUP_COMMIT_TIME` | = `EPOCH_LEN` → `--wal_group_commit_time`（微秒） |
| 15 | `MODEL_CXL_SEARCH_OVERHEAD` | `--model_cxl_search_overhead` |
| 16 | `TIME_TO_RUN` | `--time_to_run`（秒，含 warmup） |
| 17 | `TIME_TO_WARMUP` | `--time_to_warmup`（秒） |

函数内部 7 次调用 `run.sh`（`common.sh:35-41`），唯一变化的是 TPC-C 的「远程 NewOrder%/Payment%」：

```
mixed 0  0    ← 第1档：纯本地，且 PRE_MIGRATE 写死为 None（无远程访问，不必预迁移）
mixed 10 15   ← 之后各档用传入的 $PRE_MIGRATE
mixed 20 30
mixed 30 45
mixed 40 60
mixed 50 75
mixed 60 90   ← 最后一档：60% NewOrder + 90% Payment 为远程
```

被 `common.sh` 写死、不暴露给上层的项：`QUERY_TYPE=mixed`、`ENABLE_MIGRATION_OPTIMIZATION=1`、`GATHER_OUTPUTS=0`。

### `run_remote_txn_overhead_ycsb` 的 20 个形参（`common.sh:44-88`）

与 TPCC 版差异：第 5-7 位插入 YCSB 专属的 `WORKLOAD`(查询类型)、`RW_RATIO`(读%)、`ZIPF_THETA`(偏斜)，其余顺延。内部 `KEYS` 写死为 **300000**（`common.sh:71`）。11 次调用唯一变化的是跨分区比例 `CROSS_RATIO`：`0,10,20,...,100`（`common.sh:77-87`），同样第 1 档（CROSS_RATIO=0）强制 `PRE_MIGRATE=None`。

### 结果文件命名规则（解析脚本据此识别）

`common.sh:31`（TPCC）/`:73`（YCSB）把所有配置拼进文件名：
```
tpcc-<PROTOCOL>-<HOST>-<WORKER>-<CXL>-<OUTPUT>-<MIGPOL>-<WHEN>-<BUDGET>-<SCC>-<SCCMECH>-<PREMIG>-<LOG>-<EPOCH>-<MODEL>.txt
ycsb-<PROTOCOL>-<WORKLOAD>-<HOST>-<WORKER>-<RW>-<ZIPF>-<CXL>-<OUTPUT>-<MIGPOL>-<WHEN>-<BUDGET>-<SCC>-<SCCMECH>-<PREMIG>-<LOG>-<EPOCH>-<MODEL>.txt
```
**这套命名是 run 脚本与 parse 脚本之间的"契约"**——parse_swcc.py 正是靠拼出同样的名字来读对文件（见 §5）。

---

## 1. `./scripts/run_tpcc.sh $RESULT_ROOT_DIR`（生成 Fig 4 数据）

### 1.1 顶部配置常量（run_tpcc.sh:24-35）

| 变量 | 值 | 含义 |
|---|---|---|
| `HOST_NUM` | `8` | 8 个 host（VM） |
| `WORKER_NUM` | `3` | Tigon / 改进基线每 host 3 个事务 worker 线程 |
| `OLD_WORKER_NUM` | `2` | 旧基线（带专职输出线程）每 host 2 worker——留 1 核给输出线程 |
| `DEFAULT_WAL_GROUP_COMMIT_TIME` | `10000` | 组提交 epoch = 10000µs = **10ms** |
| `DEFAULT_HCC_SIZE_LIMIT` | `1024*1024*200` = `209715200` | HW-cc 区预算 = **200MB**（字节） |
| `TPCC_RUN_TIME` | `30` | 单点运行 30 秒 |
| `TPCC_WARMUP_TIME` | `10` | 预热 10 秒（不计入统计） |
| `RESULT_DIR` | `$RESULT_ROOT_DIR/tpcc` | 结果落 `<root>/tpcc/` |

### 1.2 八次实验调用（run_tpcc.sh:44-57）逐参数

每行 = 一个系统配置 × 7 档远程比例。下表把 17 个参数全部展开（`R`=RESULT_DIR 省略）：

| # | 论文标签 | 协议 | host | worker | CXL | 输出线程 | 迁移策略 | 迁出时机 | 预算(B) | SCC | SCC机制 | 预迁移 | 日志 | epoch | model | run | warmup |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 44 | TwoPL-CXL-improved | TwoPL | 8 | 3 | 1 | 0 | NoMoveOut | OnDemand | 0 | 0 | NoOP | None | GROUP_WAL | 10000 | 0 | 30 | 10 |
| 45 | Sundial-CXL-improved | Sundial | 8 | 3 | 1 | 0 | NoMoveOut | OnDemand | 0 | 0 | NoOP | None | GROUP_WAL | 10000 | 0 | 30 | 10 |
| 46 | TwoPL-CXL | TwoPL | 8 | **2** | 1 | **1** | NoMoveOut | OnDemand | 0 | 0 | NoOP | None | GROUP_WAL | 10000 | 0 | 30 | 10 |
| 47 | Sundial-CXL | Sundial | 8 | **2** | 1 | **1** | NoMoveOut | OnDemand | 0 | 0 | NoOP | None | GROUP_WAL | 10000 | 0 | 30 | 10 |
| 48 | TwoPL-NET | TwoPL | 8 | 2 | **0** | 1 | NoMoveOut | OnDemand | 0 | 0 | NoOP | None | GROUP_WAL | 10000 | 0 | 30 | 10 |
| 49 | Sundial-NET | Sundial | 8 | 2 | **0** | 1 | NoMoveOut | OnDemand | 0 | 0 | NoOP | None | GROUP_WAL | 10000 | 0 | 30 | 10 |
| 56 | **Tigon** | TwoPLPasha | 8 | 3 | 1 | 0 | **Clock** | OnDemand | **209715200** | **1** | **WriteThrough** | **NonPart** | GROUP_WAL | 10000 | 0 | 30 | 10 |
| 57 | Tigon 关幻读 | TwoPLPashaPhantom | 8 | 3 | 1 | 0 | Clock | OnDemand | 209715200 | 1 | WriteThrough | NonPart | GROUP_WAL | 10000 | 0 | 30 | 10 |

**逐参数语义解读**：

- **协议（第2参）**：`TwoPL`/`Sundial` 是基线；`TwoPLPasha` 是 Tigon；`TwoPLPashaPhantom` 实际仍传 `--protocol=TwoPLPasha` 但加 `--enable_phantom_detection=false`（run.sh:224），用来量化幻读防护（next-key locking）的开销。
- **worker（第4参）3 vs 2**：CXL-improved 用满 3 worker；CXL/NET 版用 2 worker 是因为还要 `USE_OUTPUT_THREAD=1` 占用 1 个核做专职消息发送线程，保证「核数公平」对比。
- **CXL（第5参）+ 输出线程（第6参）的三种组合**正是论文区分 NET / CXL / CXL-improved 三种基线传输方式的关键：
  - `1,0` = 走 CXL 共享内存传输、无专职输出线程（improved）
  - `1,1` = 走 CXL + 专职输出线程（CXL）
  - `0,1` = 走 TCP 网络（NET）
- **迁移策略（第7参）`NoMoveOut` vs `Clock`**：基线 `NoMoveOut` = 不迁移数据（关掉 Pasha 迁移）；Tigon `Clock` = 时钟算法选迁出 victim。
- **预算（第9参）0 vs 209715200**：基线给 0（无 HW-cc 迁移区）；Tigon 给 200MB。
- **SCC（第10/11参）0/NoOP vs 1/WriteThrough**：基线关软件缓存一致；Tigon 开 WriteThrough。
- **预迁移（第12参）None vs NonPart**：Tigon 对「不可分区数据」（TPC-C 里如 item、跨 warehouse 共享的部分）预迁移（NonPart），减少运行时迁移抖动；但注意每个配置的**第 1 档（0/0）仍被 common.sh 强制 None**。
- 其余对所有行一致：`OnDemand`（CXL 满才迁出）、`GROUP_WAL`+`10000`（10ms 组提交）、`model=0`（开 shortcut 优化）、`30/10`（运行/预热秒数）。

> 总实验数：8 配置 × 7 远程比例 = **56 次** `bench_tpcc` 启动，约 56×(30+10)s ≈ 37 分钟纯跑，加上每次重置 CXL/起停开销，约 1 小时（README:121）。
>
> 基线行虽然也传了迁移/SCC/预算参数，但 `run.sh` 的 `Sundial`/`TwoPL` 分支根本不把这些 flag 转发给 binary（run.sh:163-261），所以对基线而言这些值是"形式占位、实际忽略"。

---

## 2. `./scripts/run_ycsb.sh $RESULT_ROOT_DIR`（生成 Fig 5 数据）

### 2.1 顶部配置常量（run_ycsb.sh:24-38）

在 run_tpcc.sh 那套基础上多两个读写比常量：

| 变量 | 值 | 含义 |
|---|---|---|
| `HOST_NUM` / `WORKER_NUM` / `OLD_WORKER_NUM` | 8 / 3 / 2 | 同 TPCC |
| `DEFAULT_WAL_GROUP_COMMIT_TIME` | 10000 | 10ms 组提交 |
| `DEFAULT_HCC_SIZE_LIMIT` | 209715200 | 200MB 预算 |
| `YCSB_RUN_TIME` / `YCSB_WARMUP_TIME` | 30 / 10 | 运行/预热秒数 |
| `READ_INTENSIVE_RW_RATIO` | `95` | 读密集 = 95% 读 |
| `WRITE_INTENSIVE_RW_RATIO` | `50` | 写密集 = 50% 读（半读半写） |
| `RESULT_DIR` | `$RESULT_ROOT_DIR/ycsb` | 结果落 `<root>/ycsb/` |

### 2.2 十二次实验调用（run_ycsb.sh:48-65）逐参数

4 种负载 × 3 个系统 = 12 行。下表展开 20 个参数中的关键列（host/worker=8/3、CXL/输出=1/0、迁出=OnDemand、日志=GROUP_WAL/10000、model=0、run/warmup=30/10 全部相同，略去）：

| # | 负载 | 协议 | WORKLOAD | RW_RATIO | ZIPF | 迁移策略 | 预算 | SCC | SCC机制 | 预迁移 |
|---|---|---|---|---|---|---|---|---|---|---|
| 48 | 只读 | TwoPL | rmw | **100** | 0.7 | NoMoveOut | 0 | 0 | NoOP | None |
| 49 | 只读 | Sundial | rmw | 100 | 0.7 | NoMoveOut | 0 | 0 | NoOP | None |
| 50 | 只读 | **TwoPLPasha** | rmw | 100 | 0.7 | Clock | 209715200 | 1 | WriteThrough | NonPart |
| 53 | 只写 | TwoPL | rmw | **0** | 0.7 | NoMoveOut | 0 | 0 | NoOP | None |
| 54 | 只写 | Sundial | rmw | 0 | 0.7 | NoMoveOut | 0 | 0 | NoOP | None |
| 55 | 只写 | TwoPLPasha | rmw | 0 | 0.7 | Clock | 209715200 | 1 | WriteThrough | NonPart |
| 58 | 读密集 | TwoPL | rmw | **95** | 0.7 | NoMoveOut | 0 | 0 | NoOP | None |
| 59 | 读密集 | Sundial | rmw | 95 | 0.7 | NoMoveOut | 0 | 0 | NoOP | None |
| 60 | 读密集 | TwoPLPasha | rmw | 95 | 0.7 | Clock | 209715200 | 1 | WriteThrough | NonPart |
| 63 | 写密集 | TwoPL | rmw | **50** | 0.7 | NoMoveOut | 0 | 0 | NoOP | None |
| 64 | 写密集 | Sundial | rmw | 50 | 0.7 | NoMoveOut | 0 | 0 | NoOP | None |
| 65 | 写密集 | TwoPLPasha | rmw | 50 | 0.7 | Clock | 209715200 | 1 | WriteThrough | NonPart |

**逐参数语义解读**：

- **WORKLOAD=`rmw`**（read-modify-write）：YCSB 标准事务——读一批 key、改、写回。所有行一致。
- **RW_RATIO** 是这个脚本扫的核心维度，对应 4 种负载：`100`(纯读) / `0`(纯写) / `95`(读密集) / `50`(写密集)。论文 Fig 5 的 4 个子图就是这 4 个值。语义：读操作占比%（100=全读，0=全写）。
- **ZIPF=0.7**：Zipfian 偏斜因子，固定 0.7（中等偏斜，制造热点 key 让跨 host 竞争有意义）。
- 系统三件套与 TPCC 完全同理：基线 `NoMoveOut`+预算0+`NoOP`（关 Pasha），Tigon `Clock`+200MB+`WriteThrough`+`NonPart`。
- **每行内部扫 11 档 `CROSS_RATIO`**（0→100，common.sh:77-87）：事务内跨分区（远程）操作百分比。这是 Fig 5 每个子图的 X 轴。YCSB 每 host 一个分区，跨分区即跨 host。

> 与 TPCC 的两点结构差异：① 这里只有 3 个系统（TwoPL/Sundial/Tigon），没有 CXL/NET/improved 三分变体（统一用 CXL-improved 的 `1,0` 配置）；② 远程维度是单值 `CROSS_RATIO`（0-100，11 档），不像 TPCC 是 NewOrder%/Payment% 成对（7 档）。
>
> 总实验数：12 配置 × 11 跨分区比例 = **132 次** `bench_ycsb` 启动 ≈ 132×40s ≈ 88 分钟纯跑 + 开销 ≈ 2.5 小时（README:135）。

---

## 3. `./scripts/run_hwcc_budget.sh $RESULT_ROOT_DIR`（生成 Fig 7 数据）

研究「HW-cc 预算缩小时 Tigon 性能如何变化」，所以**只跑 Tigon（TwoPLPasha），扫预算这一维**。

### 3.1 顶部配置常量（run_hwcc_budget.sh:24-48）

| 变量 | 值 | 含义 |
|---|---|---|
| `HOST_NUM` / `WORKER_NUM` / `OLD_WORKER_NUM` | 8 / 3 / 2 | 同前 |
| `DEFAULT_WAL_GROUP_COMMIT_TIME` | 10000 | 10ms |
| `DEFAULT_HCC_SIZE_LIMIT` | 209715200 | 200MB（本脚本未直接用，用下面 5 档） |
| `DATA_MOVEMENT_EXP_HCC_SIZE_LIMIT_1` | `1024*1024*200`=209715200 | **200MB**（预算档 1） |
| `..._2` | `1024*1024*150`=157286400 | **150MB**（档 2） |
| `..._3` | `1024*1024*100`=104857600 | **100MB**（档 3） |
| `..._4` | `1024*1024*50`=52428800 | **50MB**（档 4） |
| `..._5` | `1024*1024*10`=10485760 | **10MB**（档 5，极限压力） |
| `TPCC_RUN_TIME` / `TPCC_WARMUP_TIME` | **120** / **60** | TPC-C 跑 120s / 预热 60s（比主实验长得多） |
| `YCSB_RUN_TIME` / `YCSB_WARMUP_TIME` | **60** / **30** | YCSB 跑 60s / 预热 30s |
| `READ_INTENSIVE_RW_RATIO` | 95 | 读密集 |
| `WRITE_INTENSIVE_RW_RATIO` | 50 | （本脚本未用写密集） |
| `RESULT_DIR` | `$RESULT_ROOT_DIR/hwcc_budget` | 落 `<root>/hwcc_budget/` |

> **为什么 run/warmup 比主实验长？** 预算缩小后系统要经历更频繁的数据迁入/迁出，达到稳态更慢；拉长预热（TPCC 60s、YCSB 30s）+ 测量窗口（120s/60s）才能测到稳定吞吐，避免冷启动假象。

### 3.2 十次实验调用（run_hwcc_budget.sh:58-69）逐参数

5 档预算 × {TPC-C, YCSB 读密集} = 10 行。除预算外，所有 Tigon 参数一致：

**TPC-C 部分（58-62）**，每行 = `run_remote_txn_overhead_tpcc`：

| # | 协议 | host/worker | CXL/输出 | 迁移策略 | 迁出 | **预算** | SCC | SCC机制 | 预迁移 | 日志/epoch | model | run/warmup |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 58 | TwoPLPasha | 8/3 | 1/0 | Clock | OnDemand | **200MB** | 1 | WriteThrough | **None** | GROUP_WAL/10000 | 0 | 120/60 |
| 59 | TwoPLPasha | 8/3 | 1/0 | Clock | OnDemand | **150MB** | 1 | WriteThrough | None | GROUP_WAL/10000 | 0 | 120/60 |
| 60 | TwoPLPasha | 8/3 | 1/0 | Clock | OnDemand | **100MB** | 1 | WriteThrough | None | GROUP_WAL/10000 | 0 | 120/60 |
| 61 | TwoPLPasha | 8/3 | 1/0 | Clock | OnDemand | **50MB** | 1 | WriteThrough | None | GROUP_WAL/10000 | 0 | 120/60 |
| 62 | TwoPLPasha | 8/3 | 1/0 | Clock | OnDemand | **10MB** | 1 | WriteThrough | None | GROUP_WAL/10000 | 0 | 120/60 |

**YCSB 读密集部分（65-69）**，每行 = `run_remote_txn_overhead_ycsb`，WORKLOAD=`rmw`、RW_RATIO=`95`、ZIPF=`0.7`，预算同样 200→10MB 五档，run/warmup=60/30。

**关键差异点（与 run_tpcc.sh / run_swcc.sh 对比）**：

- **`PRE_MIGRATE=None`**（而非 NonPart）：研究的是「运行时按需迁移」在预算受限下的表现，若预迁移会掩盖效果，故全程不预迁移。
- **预算是唯一变量**：5 档从 200MB（宽裕）到 10MB（极限），观察吞吐随预算下降的曲线——这正是 Fig 7 的 Y 轴随 X（预算）变化。
- 每行内部仍扫远程比例（TPCC 7 档 / YCSB 11 档），但 Fig 7 通常取**最高远程比例那一点**（迁移压力最大）来画预算敏感性。

> 总实验数：(5 预算 × 7 远程比例) [TPCC] + (5 × 11) [YCSB] = 35 + 55 = **90 次**启动；但每次时长翻倍（120/60s），所以总耗时最长，约 3 小时（README:144）。

---

## 4. `./scripts/run_swcc.sh $RESULT_ROOT_DIR`（生成 Fig 8 数据）

研究「不同**软件缓存一致（SCC）机制**对性能的影响」，所以**只跑 Tigon（TwoPLPasha），扫 SCC 机制这一维**——其余配置全部锁定为 Tigon 默认值，只切换 `<ENABLE_SCC>-<SCC_MECHANISM>-<PRE_MIGRATE>` 三段。它产出的 8 个 `.txt` 正是 §5 的 `parse_swcc.py` 要读的输入。

### 4.1 顶部配置常量（run_swcc.sh:24-48）

| 变量 | 值 | 含义 |
|---|---|---|
| `HOST_NUM` / `WORKER_NUM` / `OLD_WORKER_NUM` | 8 / 3 / 2 | 同其它脚本（OLD_WORKER_NUM 本脚本未用到） |
| `DEFAULT_WAL_GROUP_COMMIT_TIME` | `10000` | 组提交 epoch = 10000µs = **10ms** |
| `DEFAULT_HCC_SIZE_LIMIT` | `1024*1024*200` = `209715200` | HW-cc 区预算 = **200MB**（本脚本实际用的预算） |
| `DATA_MOVEMENT_EXP_HCC_SIZE_LIMIT_1..5` | 200/150/100/50/10 MB | 5 档预算常量（从 run_hwcc_budget.sh 模板复制残留，**本脚本未使用**） |
| `TPCC_RUN_TIME` / `TPCC_WARMUP_TIME` | `30` / `10` | TPC-C 运行 30s / 预热 10s |
| `YCSB_RUN_TIME` / `YCSB_WARMUP_TIME` | `30` / `10` | YCSB 运行 30s / 预热 10s |
| `READ_INTENSIVE_RW_RATIO` | `95` | YCSB 读密集 = 95% 读 |
| `WRITE_INTENSIVE_RW_RATIO` | `50` | 写密集（**本脚本未用**，只跑读密集） |
| `RESULT_DIR` | `$RESULT_ROOT_DIR/swcc` | 结果落 `<root>/swcc/` |

> `DATA_MOVEMENT_EXP_HCC_SIZE_LIMIT_*` 与 `WRITE_INTENSIVE_RW_RATIO` 是模板复制带来的「声明但未引用」常量——SWcc 实验固定用 200MB 预算、只测读密集，不扫预算、不测写密集。

### 4.2 八次实验调用（run_swcc.sh:58-68）逐参数

4 种 SCC 机制 × {TPC-C, YCSB 读密集} = 8 行。除 `<ENABLE_SCC>-<SCC_MECHANISM>-<PRE_MIGRATE>` 三段外，所有参数对 8 行完全一致（host/worker=8/3、CXL/输出=1/0、迁移=Clock、迁出=OnDemand、预算=200MB、日志=GROUP_WAL/10000、model=0、run/warmup=30/10）。

**TPC-C 部分（58-61）**，每行 = `run_remote_txn_overhead_tpcc`（17 参）：

| # | 论文含义 | 协议 | **ENABLE_SCC** | **SCC_MECHANISM** | **PRE_MIGRATE** | 其余 |
|---|---|---|---|---|---|---|
| 58 | Tigon（默认） | TwoPLPasha | `1` | `WriteThrough` | `NonPart` | 8/3·1/0·Clock·OnDemand·200MB·GROUP_WAL/10000·model0·30/10 |
| 59 | Tigon + 无共享读者 | TwoPLPasha | `1` | `WriteThroughNoSharedRead` | `NonPart` | 同上 |
| 60 | Tigon + 非时序 | TwoPLPasha | `1` | `NonTemporal` | `NonPart` | 同上 |
| 61 | Tigon + 关 SCC | TwoPLPasha | **`0`** | `NoOP` | **`None`** | 同上 |

**YCSB 读密集部分（65-68）**，每行 = `run_remote_txn_overhead_ycsb`（20 参），固定 `WORKLOAD=rmw`、`RW_RATIO=95`、`ZIPF=0.7`，SCC 三段同样切 4 种，run/warmup=30/10。

**SCC 机制（第 10/11 参 = ENABLE_SCC / SCC_MECHANISM）四种取值的语义**（对照 `--scc_mechanism`，README:202、`protocol/Pasha/SCCManager`）：

- **`1` + `WriteThrough`**（Tigon 默认）：软件缓存一致开启，写穿透机制——读 miss 时 `clflush` 后置 per-host 有效位，写时清所有 host 的有效位并 `clwb` 刷回 CXL，且**允许共享读者**（多 host 同时持有效副本）。这是 Tigon 兜底 CXL 1.1 硬件缓存一致预算有限的核心机制。
- **`1` + `WriteThroughNoSharedRead`**：同样写穿透，但**禁用共享读者优化**——用来量化「允许多 host 同时缓存只读副本」带来的收益（关掉它性能应下降）。
- **`1` + `NonTemporal`**：始终用非时序（non-temporal）访问 CXL，绕过 CPU cache 直接读写——避免维护一致性位的开销，但每次都走内存、无 cache 局部性收益。对照实验。
- **`0` + `NoOP`**：完全关闭软件缓存一致（`enable_scc=0`），`NoOP` 机制不做任何一致性维护——**假设底层硬件已提供完整缓存一致**。这是「无 SWcc」对照基准，衡量 SCC 本身的开销。

**为什么 NoOP 那行配 `PRE_MIGRATE=None` 而非 NonPart？**（59-61 是 NonPart，61 是 None）

- 前三种（WriteThrough 系 + NonTemporal）都开 SCC，配合 `NonPart` 预迁移「不可分区数据」，与 Tigon 主实验一致。
- `NoOP` 关闭了缓存一致，迁移行的一致性维护机制也随之失效，故配套关闭预迁移（`None`），避免迁移与无一致性机制冲突——这也是 `parse_swcc.py` 里 NoOP 文件名后缀是 `0-NoOP-None`、其余是 `1-<机制>-NonPart` 的原因（见 §5.2）。

> 每行内部仍扫远程比例（TPC-C 7 档、YCSB 11 档，由 `common.sh` 循环）。Fig 8 横轴即远程比例，4 条曲线 = 4 种 SCC 机制。
>
> 总实验数：(4 机制 × 7 远程比例) [TPCC] + (4 × 11) [YCSB] = 28 + 44 = **72 次**启动 ≈ 72×40s ≈ 48 分钟 + 开销 ≈ 1 小时（README:153）。

---

## 5. `./scripts/parse/parse_swcc.py $RESULT_ROOT_DIR`（把 SWcc 实验 .txt 解析成 Fig 8 的 .csv）

> 前提：`parse_swcc.py` 解析的是 `run_swcc.sh` 产出的文件。`run_swcc.sh`（结构同上述 run 脚本）对 **4 种软件缓存一致机制**各跑一遍 TPC-C + YCSB 读密集（`run_swcc.sh:58-68`）：
> - `WriteThrough`（Tigon 默认，SCC 开）
> - `WriteThroughNoSharedRead`（关共享读者优化）
> - `NonTemporal`（始终非时序访问）
> - `NoOP`（SCC 关，`enable_scc=0`，假设全硬件一致）
>
> 这 4 个机制对应文件名里 `<SCC>-<SCCMECH>-<PREMIG>` 三段的不同取值。

### 5.1 顶层执行（parse_swcc.py:40-48）

```python
if len(sys.argv) != 2:                                  # 必须恰好 1 个命令行参数
        print("Usage: " + sys.argv[0] + " RESULT_ROOT_DIR")
        sys.exit(-1)
res_root_dir = sys.argv[1]                              # 结果根目录(如 results/test1)
swcc_res_dir = res_root_dir + "/swcc"                   # SWcc 子目录(run_swcc.sh 的输出地)
parse_tpcc_swcc(swcc_res_dir)                           # 解析 TPC-C 的 4 机制 → tpcc-swcc.csv
parse_ycsb_swcc(swcc_res_dir, "95", "0.7")             # 解析 YCSB 读密集(95% 读, zipf 0.7) → ycsb-swcc-95-0.7.csv
```

- `sys.argv[1]` = 跟 run 脚本同一个 `RESULT_ROOT_DIR`。
- `parse_ycsb_swcc` 的两个字面量参数 `"95"`、`"0.7"`：**必须和 `run_swcc.sh` 里 YCSB 跑的 `READ_INTENSIVE_RW_RATIO=95`、`ZIPF=0.7` 完全一致**，否则拼出的文件名对不上、读不到数据。

### 5.2 构造 TPC-C 输入文件清单（parse_swcc.py:25-31）

```python
def construct_input_list_tpcc_swcc(swcc_res_dir):
        input_file_list = list()
        input_file_list.append(("Tigon",
                swcc_res_dir + "/tpcc-TwoPLPasha-8-3-1-0-Clock-OnDemand-209715200-1-WriteThrough-NonPart-GROUP_WAL-10000-0.txt"))
        input_file_list.append(("Tigon (NoSharedReader)",
                swcc_res_dir + "/tpcc-TwoPLPasha-8-3-1-0-Clock-OnDemand-209715200-1-WriteThroughNoSharedRead-NonPart-GROUP_WAL-10000-0.txt"))
        input_file_list.append(("Tigon (NonTemporal)",
                swcc_res_dir + "/tpcc-TwoPLPasha-8-3-1-0-Clock-OnDemand-209715200-1-NonTemporal-NonPart-GROUP_WAL-10000-0.txt"))
        input_file_list.append(("Tigon (NoSWcc)",
                swcc_res_dir + "/tpcc-TwoPLPasha-8-3-1-0-Clock-OnDemand-209715200-0-NoOP-None-GROUP_WAL-10000-0.txt"))
        return input_file_list
```

返回 4 个 `(图例标签, 文件路径)` 元组。把文件名按 §0 的命名规则逐段拆开（`tpcc-<协议>-<host>-<worker>-<CXL>-<输出>-<迁移>-<迁出>-<预算>-<SCC>-<SCC机制>-<预迁移>-<日志>-<epoch>-<model>`）：

| 标签 | 协议 | host | worker | CXL | 输出 | 迁移 | 迁出 | 预算 | **SCC** | **SCC机制** | **预迁移** | 日志 | epoch | model |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Tigon | TwoPLPasha | 8 | 3 | 1 | 0 | Clock | OnDemand | 209715200 | **1** | **WriteThrough** | **NonPart** | GROUP_WAL | 10000 | 0 |
| Tigon (NoSharedReader) | …同… | | | | | | | | 1 | **WriteThroughNoSharedRead** | NonPart | | | |
| Tigon (NonTemporal) | …同… | | | | | | | | 1 | **NonTemporal** | NonPart | | | |
| Tigon (NoSWcc) | …同… | | | | | | | | **0** | **NoOP** | **None** | | | |

可见 4 行只有 `<SCC>-<SCC机制>-<预迁移>` 三段不同，**精确对应 `run_swcc.sh:58-61` 跑出的 4 个文件**（前三者 SCC 开+NonPart，最后 NoOP 时 SCC 关+预迁移 None——因为关了缓存一致，配套也不预迁移）。`209715200`=200MB、`10000`=10ms epoch、`8-3`=8host×3worker 全部硬编码进路径，必须和 run_swcc.sh 的常量一致。

### 5.3 解析 TPC-C 并写 CSV（parse_swcc.py:33-37）

```python
def parse_tpcc_swcc(swcc_res_dir):
        input_file_list = construct_input_list_tpcc_swcc(swcc_res_dir)   # 上面 4 个文件
        output_file_name = swcc_res_dir + "/tpcc-swcc.csv"              # 输出 CSV
        header_row = ["Remote_Ratio", "0/0", "10/15", "20/30", "30/45", "40/60", "50/75", "60/90"]  # 7 档远程比例
        parse_results(input_file_list, output_file_name, header_row)
```

- `header_row` 第 1 列是行标签列名 `Remote_Ratio`，后 7 列正是 TPC-C 的 7 档 NewOrder/Payment 远程比例——**顺序必须和 `common.sh` 里 7 次调用的顺序（0/0,10/15,…,60/90）一致**，因为吞吐值就是按这个顺序追加进 .txt 的。

### 5.4 YCSB 版（parse_swcc.py:11-23）

```python
def construct_input_list_ycsb_swcc(swcc_res_dir, rw_ratio, zipf_theta):
        # 文件名比 TPCC 多了 <WORKLOAD=rmw>-<RW>-<ZIPF> 三段
        input_file_list.append(("Tigon", swcc_res_dir +
                "/ycsb-TwoPLPasha-rmw-8-3-" + rw_ratio + "-" + zipf_theta +
                "-1-0-Clock-OnDemand-209715200-1-WriteThrough-NonPart-GROUP_WAL-10000-0.txt"))
        ... (NoSharedReader / NonTemporal / NoSWcc 三行，同 TPCC 那样只变 SCC 三段)

def parse_ycsb_swcc(swcc_res_dir, rw_ratio, zipf_theta):
        input_file_list = construct_input_list_ycsb_swcc(swcc_res_dir, rw_ratio, zipf_theta)
        output_file_name = swcc_res_dir + "/ycsb-swcc-" + rw_ratio + "-" + zipf_theta + ".csv"   # ycsb-swcc-95-0.7.csv
        header_row = ["Remote_Ratio", "0","10","20","30","40","50","60","70","80","90","100"]   # 11 档跨分区比例
        parse_results(input_file_list, output_file_name, header_row)
```

`rw_ratio`/`zipf_theta` 由主程序传入 `"95"`/`"0.7"`，既用于**拼输入文件名**（要对上 run_swcc.sh），又用于**命名输出 CSV**。header 是 YCSB 的 11 档跨分区比例 0→100。

### 5.5 解析引擎（parse/common.py）—— 数值是怎么抓出来的

```python
def get_row(input):                         # input = (标签, 文件路径)
        tputs = list()
        tputs.append(input[0])              # 行首放图例标签，如 "Tigon"
        for line in fileinput.FileInput(input[1]):   # 逐行读该 .txt
                tokens = line.strip().split()         # 按空白切词
                if len(tokens) > 7 and tokens[3] == "Coordinator.h:610]":  # 命中"全局统计"行
                        tputs.append(float(tokens[7]))                      # 取第 8 个词为吞吐
        return tputs                        # 返回 [标签, 值0, 值10, ...]
```

对一行日志（README:92）：
```
I0426 06:22:58.143128 204381 Coordinator.h:610] Global Stats: total_commit: 360162 total_size_index_usage: ...
  [0]      [1]          [2]     [3]                [4]    [5]    [6]           [7]
```
- `tokens[3] == "Coordinator.h:610]"`：glog 行尾的「文件:行号]」标记，用它定位汇总统计行。
- `tokens[7]` = `total_commit` 的数值（**测量窗口内的总提交事务数，作为吞吐指标**）。
- 一个 .txt 里有 N 次实验（N 档远程比例）就有 N 行这种统计 → `get_row` 返回 `[标签, t0, t1, …, t(N-1)]`，长度与 header 列数对齐。

```python
def parse_results(input_list, output_file_name, header_row):
        rows = [header_row]                 # 第 1 行：表头
        for input in input_list:            # 每个系统配置一行
                rows.append(get_row(input)) # [标签, 各远程比例的吞吐...]
        rows = zip(*rows)                   # 【转置】行列互换：远程比例变成行，系统变成列
        with open(output_file_name, "w") as f:
                csv.writer(f).writerows(rows)
```

- 转置前：每行一个系统（Tigon / NoSharedReader / NonTemporal / NoSWcc），每列一个远程比例。
- `zip(*rows)` 转置后：每行一个远程比例，每列一个系统——这才是 `plot_swcc.py` 期望的「X=远程比例、多条曲线=各机制」格式。
- 最终 CSV 形如：
  ```
  Remote_Ratio, Tigon, Tigon (NoSharedReader), Tigon (NonTemporal), Tigon (NoSWcc)
  0/0,          <tput>, <tput>,                 <tput>,               <tput>
  10/15,        ...
  ...
  60/90,        ...
  ```

```python
def append_motor_numbers(output_file_name, motor_csv_name):   # parse_swcc.py 不调用
        # 仅 parse_tpcc.py / parse_ycsb.py 用：把 results/motor/*.csv 的 Motor 列拼到右边
```
> 注意：**SWcc 实验不含 Motor 基线**（Motor 没有软件缓存一致这个维度），所以 `parse_swcc.py` 不调用 `append_motor_numbers`——这也是它和 `parse_tpcc.py`/`parse_ycsb.py` 的唯一流程差异。

### 5.6 parse_swcc.py 端到端数据流

```
run_swcc.sh 跑出 8 个 .txt(TPCC 4机制 + YCSB 4机制)，每个 .txt 含多档远程比例的 Global Stats 行
        │
parse_swcc.py: 按硬编码文件名规则拼出路径(必须与 run_swcc.sh 常量一致)
        │  get_row: 抓每个 .txt 里所有 "Coordinator.h:610]" 行的 tokens[7]=total_commit
        │  parse_results: 4 机制拼成表 → zip 转置(远程比例为行、机制为列)
        ▼
tpcc-swcc.csv (表头 0/0..60/90)  +  ycsb-swcc-95-0.7.csv (表头 0..100)
        ▼
plot_swcc.py → swcc.pdf (论文 Figure 8：4 种 SCC 机制吞吐对比)
```

---

## 6. 五脚本横向对比速览

| 维度 | run_tpcc.sh | run_ycsb.sh | run_hwcc_budget.sh | run_swcc.sh | parse_swcc.py |
|---|---|---|---|---|---|
| 目标图 | Fig 4(a/b/c) | Fig 5 | Fig 7 | Fig 8 | （解析 Fig 8 数据） |
| 扫描维度 | 协议×传输方式(8 配置) | 读写比(4)×系统(3) | **HW-cc 预算(5 档)** | **SCC 机制(4)** | — |
| 协议 | TwoPL/Sundial/TwoPLPasha(+Phantom) | TwoPL/Sundial/TwoPLPasha | 仅 TwoPLPasha | 仅 TwoPLPasha | — |
| 预算 | 基线0 / Tigon 200MB | 基线0 / Tigon 200MB | **200/150/100/50/10MB** | 200MB | — |
| SCC | 基线NoOP / Tigon WriteThrough | 同左 | WriteThrough | **WriteThrough/NoSharedRead/NonTemporal/NoOP** | — |
| 预迁移 | Tigon NonPart | Tigon NonPart | **None** | NonPart（NoOP 时 None） | — |
| run/warmup | 30/10 | 30/10 | **120/60(TPCC), 60/30(YCSB)** | 30/10 | — |
| 负载 | TPC-C mixed | YCSB rmw(4 读写比) | TPC-C + YCSB 读密集 | TPC-C + YCSB 读密集 | — |
| 远程维度 | NewOrder/Payment 7 档 | CROSS_RATIO 11 档 | 同各自 benchmark | 同各自 benchmark | — |
| 单点数 | 56 | 132 | 90 | **72** | 读 8 个 .txt |

**贯穿五者的不变量**：`HOST_NUM=8`、`WORKER_NUM=3`、`EPOCH=10000µs`、`LOGGING=GROUP_WAL`、`WHEN_TO_MOVE_OUT=OnDemand`、`MIGRATION_POLICY=Clock`（Tigon）、`MODEL_CXL_SEARCH=0`、`GATHER_OUTPUTS=0`、`QUERY=mixed`(TPCC)/`rmw`(YCSB)、`KEYS=300000`(YCSB)。每个实验「只动一个旋钮」正是论文做受控变量对比的方法论——`run_swcc.sh` 锁死预算/迁移/负载，唯独切换 SCC 机制，正是这一方法论最纯粹的体现。
