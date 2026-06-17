# 实验图表中的「时延」与「吞吐率」分别来自哪条打印

> 本文回答：Tigon 实验图表里的**吞吐率（throughput）**和**时延（latency）**两类指标，分别由代码里**哪一条 `LOG(INFO)` 打印**产生、解析脚本取该行的**第几个 token**、这些数值的真实含义与计算链。
>
> 一句话结论：**吞吐率看 `Coordinator.h:610`（Global Stats），时延看 `WALLogger.h` 的 `Group Commit Stats`（print_sync_stats）**。

---

## 0. 结论速览

| 指标 | 来源打印 | glog 行标记 | 取值 token | 字段含义 | 消费的解析脚本 |
|---|---|---|---|---|---|
| **吞吐率** (txns/sec) | `Coordinator.h:610` `LOG(INFO) << "Global Stats:"` | `Coordinator.h:610]` | `tokens[7]` | `total_commit`（全 host 每秒平均提交数） | `parse_tpcc/ycsb/hwcc_budget/swcc.py`、`parse_all.py`（common.py `get_row`） |
| **时延 P50** (µs) | `WALLogger.h` `print_sync_stats()` 的 `LOG(INFO) << "Group Commit Stats:"` | `WALLogger.h:539]`（旧）/ 当前在 `:549` | `tokens[7]` | `txn_latency` 第 50 百分位（端到端提交时延） | `parse_all.py` `get_latency_p50` |
| **时延 P99** (µs) | 同上一行 | 同上 | `tokens[16]` | `txn_latency` 第 99 百分位 | `parse_all.py` `get_latency_p99` |

> 主吞吐图（论文 Fig 4/5/7/8）只用吞吐率；**时延仅出现在 `run_misc.sh` 的 Logging 实验**（日志 epoch 长度 ↔ 吞吐/时延权衡），由 `parse_all.py` 生成 `*-logging-latency-*.csv`。

---

## 1. 吞吐率链路：`Coordinator.h:610`

### 1.1 打印语句

`core/Coordinator.h:609-619`（仅 host0 打印）：
```cpp
if (id == 0) {
    LOG(INFO) << "Global Stats:"
              << " total_commit: " << commit            // ← 解析取这个值
              << " total_size_index_usage: " << ...
              ...
}
```

### 1.2 `total_commit` 名为"总数"实为"每秒吞吐率"

计算链（`core/Coordinator.h`）：
1. **每秒采样循环**（`:236-341`）：每轮 `sleep_for(1s)`（`:237`），累加所有 worker 的 `n_commit` 为该秒提交数（`:262`）。
2. **只统计预热后的秒**（`:324-326`）：`if (count > warmup && count <= timeToRun - cooldown)` 才把 `n_commit` 累进 `total_commit`。
3. **求平均**（`:343`）：`count = timeToRun - warmup - cooldown` = 有效测量秒数（**这就是"时间"在吞吐里的角色——做分母**）。
4. **传入 gather**（`:374`）：`gather_and_print(1.0 * total_commit / count, ...)` → 单 host 每秒平均提交数。
5. **跨 host 汇总**（`:556-582`）：host0 收集其余 host 并 `commit += r_commit`（`:581`）→ 全系统每秒吞吐。
6. **打印**（`:610`）：`total_commit:` 后即全系统 txns/sec。

所以 README 里 `total_commit: 360162` ≈ 8 host 合计约 36 万 txns/sec。

### 1.3 token 拆解

日志行（README:92）：
```
I0426 06:22:58.143128 204381 Coordinator.h:610] Global Stats: total_commit: 360162 total_size_index_usage: ...
 [0]      [1]          [2]     [3]                 [4]    [5]    [6]           [7]
```
解析脚本 `scripts/parse/common.py:14-17`：匹配 `tokens[3] == "Coordinator.h:610]"`，取 `tokens[7]`。

---

## 2. 时延链路：`WALLogger.h` 的 `Group Commit Stats`

### 2.1 打印语句

`PashaGroupCommitLogger::print_sync_stats()`（GROUP_WAL 模式下生效的日志器），`common/WALLogger.h:547-553`：
```cpp
void print_sync_stats() override {
    LOG(INFO) << "Group Commit Stats: "
              << txn_latency.nth(50) << " us (50%) " << txn_latency.nth(75) << " us (75%) "
              << txn_latency.nth(95) << " us (95%) " << txn_latency.nth(99) << " us (99%) "
              << txn_latency.avg()   << " us (avg)"
              << " committed_txn_cnt " << committed_txn_cnt;
    LOG(INFO) << "Queuing Stats: " << ...;     // 另两条，未进图
    LOG(INFO) << "Disk Sync Stats: " << ...;
}
```

### 2.2 `txn_latency` 测的是什么——端到端提交时延

计算在 master 落盘循环里（`WALLogger.h:518-523`）：
```cpp
committed_txn_cnt += log_buffer->txn_start_times.size();
auto now = Time::now();
for (auto i = 0; i < log_buffer->txn_start_times.size(); i++) {
    auto latency = now - log_buffer->txn_start_times[i];   // 提交记录入队 → 落盘完成
    txn_latency.add(latency / 1000);                       // ns → us（除以 1000）
}
```
- `txn_start_times[i]`：slave logger 在写 commit record 时记下的时间戳，**意图回推到事务开始时刻**（`WALLogger.h:413`，见 §5.2）。
- `now`：该事务所在 log buffer 被 master **持久化 sync 完成**的时刻。
- 二者之差 = 时延，单位 **微秒（us）**。
- **设计意图 vs 实际值**：变量命名/回推逻辑的意图是「事务开始 → 持久化」的端到端时延；但因 `WALLogger.h:413` 处存入的是 `Time::now()`(ns) `−` `latency`(us) 存在 **ns/µs 单位不一致**（见 §6 坑点），执行阶段那段时延被额外缩小 1000×而几乎抵消，**实际得到的数值由「commit record 落盘延迟」主导**（即 record 记录 → durable sync 的时间，µs）。源码级推导见 §5.2。

### 2.3 三条延迟统计的区别（只有第一条进图）

| 打印行 | 指标 | 含义（`WALLogger.h`） |
|---|---|---|
| `Group Commit Stats:` | `txn_latency` | **端到端**：提交记录入队 → 落盘完成（:521-523）✅ 进图 |
| `Queuing Stats:` | `queuing_latency` | 排队：buffer 入队 → 开始写盘（:509-510）❌ 不进图 |
| `Disk Sync Stats:` | `disk_sync_latency` | 落盘：`write`+`fdatasync` 耗时（:513-514）❌ 不进图 |

近似关系：`txn_latency ≈ grouping(epoch 等待) + queuing + disk_sync`。图表里的"时延"专指 `txn_latency`。

### 2.4 token 拆解

日志行（README:94）：
```
I0426 06:22:59.143455 204381 WALLogger.h:539] Group Commit Stats: 45391 us (50%) 62229 us (75%) 91695 us (95%) 111490 us (99%) 47239 us (avg) committed_txn_cnt 620218
 [0]      [1]          [2]     [3]               [4]    [5]    [6]    [7]  [8] [9]  [10] ...        ...           [16]
```
| token | 值 | 含义 |
|---|---|---|
| `tokens[3]` | `WALLogger.h:539]` | 行标记（解析匹配键） |
| `tokens[7]` | `45391` | **P50** 时延（us）→ `get_latency_p50` |
| `tokens[10]` | `62229` | P75 |
| `tokens[13]` | `91695` | P95 |
| `tokens[16]` | `111490` | **P99** 时延（us）→ `get_latency_p99` |
| `tokens[19]` | `47239` | 平均 |

`scripts/parse/parse_all.py:38-60`：
```python
def get_latency_p50(input):
    ... if tokens[3] == "WALLogger.h:539]": lat.append(float(tokens[7]))   # P50
def get_latency_p99(input):
    ... if tokens[3] == "WALLogger.h:539]": lat.append(float(tokens[16]))  # P99
```

---

## 3. 时延实验：`run_misc.sh` Logging 段 + `parse_all.py`

时延唯一被画进图表的场景 = **日志机制的吞吐/时延权衡实验**（不在 push_button 主四图里，属 misc）。

### 3.1 跑实验（`scripts/run_misc.sh:114-132`）

固定 Tigon 配置，**扫 GROUP_WAL 的 epoch 长度**（`EPOCH_LEN`，单位 µs）：
```
BLACKHOLE（不落盘）→ GROUP_WAL 50000/40000/30000/20000/10000/5000/1000
```
epoch 越大 → 攒批越久 → 吞吐越高但时延越高；epoch 越小 → 时延越低但吞吐降。这正是要画的权衡曲线。

### 3.2 解析（`scripts/parse/parse_all.py:249-278`）

每组同时产出**两个** CSV：
```python
def parse_tpcc_logging(dir):
    ...
    parse_results(input_file_list, dir+"/tpcc-logging.csv", header)         # 吞吐（Coordinator.h:610）
    parse_latency(input_file_list, dir+"/tpcc-logging-latency.csv", header) # 时延（WALLogger.h:539）
```
- 输入文件按命名契约拼出（`Tigon-no-logging`=BLACKHOLE、`Tigon-1ms`=GROUP_WAL-1000、…`Tigon-50ms`=GROUP_WAL-50000）。
- 吞吐 CSV 走 `parse_results`（转置：行=远程比例、列=系统）。
- 时延 CSV 走 `parse_latency`：**P50 块 + P99 块上下拼接**，且 **`zip` 转置被注释掉**（`parse_all.py:80-81`）→ 时延 CSV 结构是「**行=系统**」，与吞吐 CSV 的转置布局**不同**，绘图时需相应处理。

---

## 4. 其它「实时延迟」打印（不进图表，仅供观察）

| 打印 | 行 | 内容 | 用途 |
|---|---|---|---|
| 每秒实时 | `Coordinator.h:307-311` | `commit: <n> abort: ... persistence latency <..> txn latency <..> queued lock latency <..> lock latency <..>` | 每秒一行的实时吞吐+各类延迟（worker 上报窗口值），README:211「每秒打印」即此 |
| 往返延迟 | `Coordinator.h:155-156` | `round_trip_latency <p50> (50th) ...` | 网络/CXL 往返延迟分位数 |

这些是调试/观察用，**解析脚本不读它们**（它们的行标记不是 `Coordinator.h:610]` 也不是 `WALLogger.h:539]`）。注意 `Coordinator.h:307` 行内的 `txn latency` 是 worker 窗口平均，与 WAL 的 `txn_latency`（端到端提交时延）**不是同一指标**，别混淆。

---

## 5. 源码级统计采集与计算原理

前面讲了「哪条打印、取第几个 token」。本节深入源码，讲清这两个数值在底层是**怎么被一条条采集、又怎么被聚合/分位计算**出来的——这决定了它们的统计性质（吞吐是**精确全量计数**，时延是**抽样分位**）。

### 5.1 吞吐率：每事务原子计数 → 每秒清零 → 跨 host 累加

**(1) 每事务 +1（全量，不抽样）。** 每个 Executor 提交成功一笔事务就把自己的原子计数器 `+1`：

`core/Executor.h:146-157`：
```cpp
commit = protocol.commit(*transaction, messages);   // 协议提交
...
if (commit) {
    db.global_total_commit.fetch_add(1);            // 一致性校验用的全局计数
    n_commit.fetch_add(1);                           // ← 本 worker 的吞吐计数器 +1
    ...
}
```
（group-commit 类协议如 H-Store 在 `core/group_commit/Executor.h:119` 同样 `n_commit.fetch_add(1)`。）

计数器定义在 `core/Worker.h:65`，每个 worker 一个、原子类型：
```cpp
std::atomic<uint64_t> n_commit, n_abort_no_retry, n_abort_lock, ... ;
```
> **要点**：吞吐是**每笔提交都计**的精确计数，无抽样、无丢弃。

**(2) 每秒采样并清零。** Coordinator 主循环每秒把所有 worker 的 `n_commit` 求和成该秒的 `n_commit`，随即**清零**让其重新计下一秒（`core/Coordinator.h:262-263`）：
```cpp
n_commit += workers[i]->n_commit.load();
workers[i]->n_commit.store(0);          // 清零 → 下一秒重新累计
```
该秒值打印在 `:307`（实时），并在预热后累加进 `total_commit`（`:326`）。

**(3) 求平均 + 跨 host 累加。** 测量结束后 `1.0*total_commit/count`（`:374`）得本 host 每秒吞吐，再经 `STATISTICS` 控制消息汇总到 host0（`:556-582`）：非 host0 用 `ControlMessageFactory::new_statistics_message` 把自己的 commit 值编码发出（`:592-594`），host0 解码后 `commit += r_commit`（`:573/581`）。最终 `:610` 打印的就是**全系统每秒吞吐**。

**计算式**：
```
吞吐(txns/sec) = Σ_host ( Σ_{t∈[warmup, timeToRun)} Σ_worker n_commit[worker,t] ) / (timeToRun - warmup - cooldown)
```
分母 `count` = 有效测量秒数（**时间在这里只当除数**）。

### 5.2 时延：commit record 打点 → 落盘时算差 → Percentile 抽样分位

**(1) 在哪打点（仅 commit record，persist=true）。** 事务的 redo/commit 记录写进日志流水线时，slave logger 的 `write()` 被调用；只有 commit record 传 `persist=true`，此时记一条时间戳到 buffer 的 `txn_start_times`：

`common/WALLogger.h:411-414`（`PashaGroupCommitLoggerSlave::write`）：
```cpp
if (persist == true) {
    auto latency = duration_cast<microseconds>(steady_clock::now() - txn_start_time).count();  // 执行已耗时(us)
    cur_log_buffer->txn_start_times.push_back(Time::now() - latency);    // 回推到"事务开始"的绝对时刻
}
```
- `txn_start_time` = 该事务 steady_clock 起始点（由 Executor 传入）。
- 意图：`Time::now() - latency` 把时间戳**回推到事务开始的那一刻**，使后面算出的是「事务开始 → 持久化」的端到端时延。
- `Time::now()`（`common/Time.h:13-17`）= 相对 `startTime` 的**纳秒**单调计数。

**(2) 落盘时算差并入样。** master 线程把 buffer 写盘 `fdatasync` 后，对 buffer 内每笔事务算 `now - 起始时刻`，转微秒后入 `txn_latency`：

`common/WALLogger.h:518-523`：
```cpp
auto now = Time::now();                                    // ns
for (i in txn_start_times) {
    auto latency = now - log_buffer->txn_start_times[i];   // ns
    txn_latency.add(latency / 1000);                        // → us，入分位统计
}
```
同循环还采集 `queuing_latency`（buffer 排队，`:509-510`）与 `disk_sync_latency`（`write+fdatasync`，`:513-514`）。

**(3) Percentile 的采集与分位计算（关键：抽样 + 预热门控）。** `txn_latency` 是 `Percentile<uint64_t>`（`WALLogger.h:582`）。其 `add`/`nth`/`avg` 实现（`common/Percentile.h`）：

```cpp
void add(const element_type &value) {
    if (warmed_up == false || rand.uniform_dist(0, 100) > 10)  // 预热前 或 ~90% 概率 → 丢弃
        return;                                                // 注释写"record 2%"，实际约 11/101≈10.9%
    data_.push_back(value);  sum += value;
}
element_type nth(double n) {                 // 最近秩法(nearest-rank)
    checkSort();                             // 惰性排序 data_
    auto i = ceil(n/100 * size()) - 1;       // 第 n 百分位的下标
    return data_[i];
}
element_type avg() { return sum / (size() + 0.1); }   // 仅对抽样值求均值
```
- **预热门控**：`warmed_up`（全局，`Coordinator.h:30` 定义、`:325` 在预热结束后置 true）——预热期的时延全部丢弃。这与吞吐「预热后才累加」的窗口一致。
- **抽样**：仅约 **10.9%** 的事务被记录（`uniform_dist(0,100) ∈ [0,10]` 命中）。代码注释说 2%，与实现不符（见 §6）。抽样是为限制 `data_` 内存。
- **分位**：`nth(50/75/95/99)` 用最近秩法——先排序，取 `ceil(p/100 × N)-1` 下标。

> **要点对比**：吞吐 = **全量精确计数**；时延 = **预热后 ~11% 抽样**的最近秩分位。两者统计性质不同。

### 5.3 一张表对照两者的源码采集方式

| 维度 | 吞吐率 | 时延 (txn_latency) |
|---|---|---|
| 采集粒度 | 每笔提交 `n_commit.fetch_add(1)`（Executor.h:157） | 每笔 commit record 打点 + 落盘算差（WALLogger.h:413/522） |
| 抽样 | 无（全量计数） | 约 11% 抽样（Percentile.h:28） |
| 预热处理 | 预热秒不累加（Coordinator.h:324） | 预热前 `add` 直接丢弃（Percentile.h:28 `warmed_up`） |
| 聚合方式 | 每秒求和→清零→窗口累加→/count→跨host相加 | 入 `Percentile`，排序后取最近秩分位 |
| 时间的角色 | 做除数（count=测量秒数） | 做被测量（now − 起始时刻） |
| 单位 | txns/sec | µs（`/1000` 从 ns 转） |
| 统计性质 | 精确总量平均 | 抽样经验分位 |

---

## 6. 多节点数据汇总与各节点统计范围

实验是 `coordinator_num` 个 host（进程）并行跑。本节讲清：**每个 host 在本地统计的数据范围是什么、最终如何汇总成图表里的一个数**。核心结论先行——

> 吞吐和 CXL 用量是**加性指标**，由 `STATISTICS` 消息汇总到 host0 **求和**；时延**不进汇总消息**，各 host 各自打印、图表只取 **host0 那一份**。

### 6.1 三类指标的统计范围与汇总方式

| 指标 | 每个 host 的统计范围 | 跨 host 汇总 | 最终落到哪条打印 |
|---|---|---|---|
| 吞吐 `commit` | 本 host 所有 worker 提交的事务数（= 本 host **发起/拥有**的事务，`n_commit` 之和 / count） | `STATISTICS` 消息 → host0 **相加** | host0 的 `Global Stats`（Coordinator.h:610） |
| CXL 6 类用量 | 本 host 进程向 CXL 共享区的分配量（index/metadata/data/transport/misc/hw_cc） | 同上 **相加** | host0 的 `Global Stats` |
| 时延 `txn_latency` | 本 host 自己的 logger 测的、流经**本 host** 的事务（抽样分位） | **不汇总**，各 host 各打各的 | 各 host 的 `Group Commit Stats`；图表取 host0 那份 |

### 6.2 `STATISTICS` 消息：只携带吞吐 + CXL 用量，没有时延

汇总用的控制消息字段固定（`core/ControlMessage.h:18-37` `new_statistics_message`）：
```cpp
encoder << coordinator_id << commit                       // int + double
        << size_index_usage << size_metadata_usage << size_data_usgae
        << size_transport_usage << size_misc_usage << size_hwcc_usage;   // 6 × uint64
```
即 `coordinator_id(int) + commit(double) + 6×uint64 CXL 用量`。host0 解码时还会 `CHECK` 长度正好是这些字段之和（`Coordinator.h:568`）。**没有任何 latency 字段** → 时延天然无法跨节点聚合。

### 6.3 汇总流程：`gather_and_print`（所有 host 都调用）

测量结束、所有 worker 停止并 join 后（`Coordinator.h:359-364`），**每个 host** 都调用 `gather_and_print`（`:374`），传入自己的 `1.0*total_commit/count` 与 6 类 CXL 用量：

- **非 host0**（`else` 分支，`Coordinator.h:591-603`）：用自己的 commit + 用量构造 `STATISTICS` 消息，`out_queue.push` 或 `cxl_transport->send` 发给 host0。
- **host0**（`id==0` 分支，`:556-589`）：循环 `coordinator_num - 1` 次，`in_queue.wait_till_non_empty` 收消息、解码，逐项累加：
  ```cpp
  commit += r_commit;              // :581（hash 分区下所有 host 都加）
  size_index_usage += r_size_index_usage;  // :583 等 6 行
  ```
  host0 传入的初值是**它自己**的 `total_commit/count`（`:374`），所以最终 `Global Stats` = **host0 自己 + Σ 其余 host**，再于 `:610` 打印。

### 6.4 为什么吞吐「相加」、时延「不汇总」

- **吞吐 / CXL 用量是加性的**：每个 host 提交的是**不同的**事务（事务按发起 host 划分，无重复计数）；CXL 各 host 分配的是共享区里**各自的**字节。相加即得全 pod 总量，语义正确。
- **时延是分布、不可简单相加**：要得到全 pod 的 P50/P99，必须把各 host 的**样本集合并**后重新排序取分位，代价大且消息要带样本。代码选择最省事的做法——**只让各 host 打印自己本地的分位**，图表取 host0 的 `Group Commit Stats`。因此严格说，图里的时延是 **host0 视角的代表值**，不是全 pod 聚合分位（各 host 负载对称时近似可代表全局）。

### 6.5 「事务范围」细节：为什么相加不重不漏

- 每个 host 的 worker 只驱动**它自己生成**的事务（`workload.next_transaction` 按本 host 的 partition 生成）。远程访问通过 Pasha 迁移读到远端数据，但**事务由发起 host 提交**，`n_commit` 计在发起 host 上。
- 因此 hash 分区下 `Σ_host total_commit` 对全 pod 事务**无重复、无遗漏**。
- **`hpb` 副本分区特例**（`Coordinator.h:574-579`）：副本 coordinator 的 commit 被单独计入 `replica_sum`（`:578`、`:605` 另行打印），**不计入吞吐**，避免主/副本重复计数。默认 `hash` 分区无此问题，全部相加。

### 6.6 时序：先出全局吞吐，再出各 host 时延

```
跑完(:341) → 算 total_commit/count → 停 worker 并 join(:359-364)
   → 各 host 打印自己的 CXL/SCC stats(:367/:370)
   → gather_and_print(:374)：host0 收齐并打印  ★Global Stats（吞吐+CXL，全局汇总）
   → (停 IO 线程后) 各 host master_logger->print_sync_stats(:400-401)：★Group Commit Stats（时延，各自本地）
```
所以 output.txt 里 **`Global Stats`（吞吐）在前、`Group Commit Stats`（时延）在后**（对应 README:92 然后 :94）。host0 的 output.txt 同时含两者；其余 host 的 output.txt 只有自己的本地 `Group Commit Stats`（除非 `GATHER_OUTPUTS=1` 才被收集，但图表不用）。

### 6.7 一图看懂多节点汇总

```
 host1 ┐ (commit_1 + CXL用量_1)  ─STATISTICS消息→┐
 host2 ┤ (commit_2 + CXL用量_2)  ─STATISTICS消息→┤
  ...  ┤                                          ├─► host0 求和
 hostN ┘ (commit_N + CXL用量_N)  ─STATISTICS消息→┘     commit = Σ commit_i
                                                       size_* = Σ size_*_i
                                                       └─► Coordinator.h:610 打印 Global Stats（吞吐/CXL）

 各 host 独立：master_logger->print_sync_stats() → 各自的 Group Commit Stats（时延，本地抽样分位，不汇总）
              图表只取 host0 这一份
```

---

## 7. 坑点与注意事项



1. **行号偏移（重要）**：`parse_all.py` 与 README 硬编码行标记 `WALLogger.h:539]`，但**当前源码 `PashaGroupCommitLogger::print_sync_stats` 的首条 LOG 已在 `WALLogger.h:549`**。glog 打印的是 `LOG()` 语句所在行号，代码上移/下移会改变它。若用当前源码构建却用旧 `parse_all.py` 解析，**时延列会全空**（匹配不到）。修法：跑前 `grep -n "Group Commit Stats" common/WALLogger.h` 确认实际行号，同步改 `parse_all.py:45/57` 的 `"WALLogger.h:XXX]"`。吞吐侧 `Coordinator.h:610` 同理需对齐。
2. **BLACKHOLE 无时延数据**：`BlackholeLogger` 不做组提交、`print_sync_stats` 为空，不产生 `Group Commit Stats` 行 → `Tigon-no-logging` 那列时延会缺失（只有吞吐）。
3. **单位**：时延是**微秒（us）**，且代码里 `latency / 1000`（`WALLogger.h:523`）是从 ns 转 us。吞吐是 **txns/sec**。
4. **时延 CSV 未转置**：`parse_latency` 的 `zip(*rows)` 被注释（`:80-81`），与吞吐 CSV 布局不一致——这是 `parse_all.py` 与 `common.py` 的一处结构差异。
5. **时延依赖 GROUP_WAL**：只有 `GROUP_WAL` 模式才有 `PashaGroupCommitLogger` 打印 `Group Commit Stats`；`WAL`（SimpleWALLogger）/`BLACKHOLE` 不产此行。主四图都用 GROUP_WAL，但它们的 parse 脚本（parse_tpcc/ycsb/...）只取吞吐、不取时延。
6. **per-second 同名字段易混**：`Coordinator.h:307` 的 `txn latency` ≠ `WALLogger.h` 的 `txn_latency`；前者是 worker 窗口均值、后者是端到端提交时延分位数。
7. **ns/µs 单位不一致（影响时延语义）**：`WALLogger.h:412-413` 存入 `txn_start_times` 时用 `Time::now()`（**ns**）减 `latency`（**µs**），而 `:520-523` 又用 `Time::now()`（ns）算差并 `/1000`。净效果是事务执行段时延被多除 1000×而几近消失，**实际 `txn_latency` 由 commit-record 落盘延迟主导**（见 §2.2/§5.2）。即「端到端」是设计意图，落到数字上更接近「记录→durable」。
8. **Percentile 抽样率注释与实现不符**：`Percentile.h:28` 注释写 "record 2% of the data"，但实现 `uniform_dist(0,100) > 10 → return` 实际记录约 `11/101 ≈ 10.9%`。分析时延样本量时以实现为准。
9. **时延是抽样、吞吐是全量**：`txn_latency`/`commit_latency` 等都经 `Percentile` 约 11% 抽样且仅预热后记录；`n_commit` 是全量精确计数。比较「样本数」时别把两者当同口径。
10. **时延是 host0 视角、吞吐是全 pod 求和**：`STATISTICS` 消息只汇总吞吐+CXL 用量（host0 求和，§6.3），时延不进汇总——图表里的时延只反映 host0 本地的 `Group Commit Stats`（§6.4）。负载非对称时，它不代表全 pod 分位。

---

## 8. 数据流总图

```
                       bench_* 进程（host0 前台）
                              │
        ┌─────────────────────┴──────────────────────┐
        │ 每秒采样循环 Coordinator.h:236-341          │
        │   每秒打印 :307（实时吞吐+延迟，不进图）     │
        │   预热后累加 total_commit                    │
        └─────────────────────┬──────────────────────┘
                              │ 结束：total_commit/count，跨 host 汇总
                              ▼
   ┌──────────────────────────┐        ┌─────────────────────────────────────┐
   │ Coordinator.h:610        │        │ WALLogger.h:549(标记539)            │
   │ "Global Stats:           │        │ "Group Commit Stats:                │
   │  total_commit: <吞吐>"   │        │  <txn_latency p50/p75/p95/p99/avg>" │
   └───────────┬──────────────┘        └──────────────────┬──────────────────┘
               │ tokens[7]                                 │ p50=tokens[7]  p99=tokens[16]
               ▼                                           ▼
   parse_*.py get_row                          parse_all.py get_latency_p50/p99
   (Fig 4/5/7/8 + logging 吞吐)                (logging-latency CSV)
               │                                           │
               ▼                                           ▼
        *.pdf（吞吐曲线）                       *-logging-latency-*.csv（时延）
```

> 关联阅读：吞吐解析细节见 `docs/tpcc_parse_plot_analysis.md`；实验脚本参数见 `docs/experiment_scripts_params.md`；WALLogger 组提交机制见 `architecture.md` §5.14。
