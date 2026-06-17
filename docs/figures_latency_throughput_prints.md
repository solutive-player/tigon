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
- `txn_start_times[i]`：事务的 commit record 进入日志流水线的时刻（slave logger 记录）。
- `now`：该事务所在 log buffer 被 master **持久化 sync 完成**的时刻。
- 二者之差 = **端到端提交时延**（从提交记录排队到落盘可见），单位 **微秒（us）**。

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

## 5. 坑点与注意事项

1. **行号偏移（重要）**：`parse_all.py` 与 README 硬编码行标记 `WALLogger.h:539]`，但**当前源码 `PashaGroupCommitLogger::print_sync_stats` 的首条 LOG 已在 `WALLogger.h:549`**。glog 打印的是 `LOG()` 语句所在行号，代码上移/下移会改变它。若用当前源码构建却用旧 `parse_all.py` 解析，**时延列会全空**（匹配不到）。修法：跑前 `grep -n "Group Commit Stats" common/WALLogger.h` 确认实际行号，同步改 `parse_all.py:45/57` 的 `"WALLogger.h:XXX]"`。吞吐侧 `Coordinator.h:610` 同理需对齐。
2. **BLACKHOLE 无时延数据**：`BlackholeLogger` 不做组提交、`print_sync_stats` 为空，不产生 `Group Commit Stats` 行 → `Tigon-no-logging` 那列时延会缺失（只有吞吐）。
3. **单位**：时延是**微秒（us）**，且代码里 `latency / 1000`（`WALLogger.h:523`）是从 ns 转 us。吞吐是 **txns/sec**。
4. **时延 CSV 未转置**：`parse_latency` 的 `zip(*rows)` 被注释（`:80-81`），与吞吐 CSV 布局不一致——这是 `parse_all.py` 与 `common.py` 的一处结构差异。
5. **时延依赖 GROUP_WAL**：只有 `GROUP_WAL` 模式才有 `PashaGroupCommitLogger` 打印 `Group Commit Stats`；`WAL`（SimpleWALLogger）/`BLACKHOLE` 不产此行。主四图都用 GROUP_WAL，但它们的 parse 脚本（parse_tpcc/ycsb/...）只取吞吐、不取时延。
6. **per-second 同名字段易混**：`Coordinator.h:307` 的 `txn latency` ≠ `WALLogger.h` 的 `txn_latency`；前者是 worker 窗口均值、后者是端到端提交时延分位数。

---

## 6. 数据流总图

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
