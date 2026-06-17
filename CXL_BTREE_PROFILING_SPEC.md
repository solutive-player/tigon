# CXL B+ 树增删改查细粒度时延打点 — 源码修改方案 SPEC

> 版本：v1.0　|　日期：2026-06-10　|　对象：`common/btree_olc_cxl/BTreeOLC_CXL.h` + `core/CXLTable.h`
>
> 本文是 [`PERF_MONITORING_SPEC.md`](./PERF_MONITORING_SPEC.md) 第 5.7 节的**落地实现细化**：
> 给出对 **CXL B+ 树增删改查（增 insert / 删 remove / 改 update / 查 search+scan）** 做细粒度
> 时延打点、并在**程序结束时通过统一数据收集接口汇总输出**所需的**全部源码修改**。
> 所有改动均给出 `文件:行号` 与可直接套用的代码片段。

---

## 目录
1. [设计决策](#1-设计决策)
2. [增删改查 → 代码入口映射](#2-增删改查--代码入口映射)
3. [新增文件①：数据收集接口 `common/CXLBTreeProfiler.h`](#3-新增文件common-cxlbtreeprofilerh)
4. [新增文件②：单例定义 `common/CXLBTreeProfiler.cpp`](#4-新增文件common-cxlbtreeprofilercpp)
5. [修改①：在 `core/CXLTable.h` 打点（推荐主方案）](#5-修改core-cxltableh-打点推荐主方案)
6. [修改②（可选）：在 `BTreeOLC_CXL.h` 内统计 OLC 重试](#6-修改可选btreeolc_cxlh-内统计-olc-重试)
7. [修改③：程序结束时的数据收集 Hook（`core/Coordinator.h`）](#7-修改程序结束时的数据收集-hookcorecoordinatorh)
8. [修改④：构建集成（CMake）](#8-修改构建集成cmake)
9. [线程安全、开销与采样](#9-线程安全开销与采样)
10. [示例输出与验证](#10-示例输出与验证)
11. [改动清单速查](#11-改动清单速查)

---

## 1. 设计决策

| 决策 | 选择 | 理由（对应源码） |
|---|---|---|
| **打点层次** | 主方案在 **wrapper 层 `CXLTableBTreeOLC`**（`core/CXLTable.h:83-203`），不在共享 B+ 树内部 | wrapper 是 增删改查 的天然边界（`search/scan/insert/remove`，`:132/147/164/179`）；**它驻留在 DRAM、每个 host 进程各有一份**，而底层 `btreeolc_cxl::BPlusTree` 位于 **CXL 共享内存、跨进程共享**（`CXLTable.h:200` + `Database.h` 中由 `id==0` 创建）。**给共享 B+ 树节点加字段会破坏跨 host 内存布局**，必须避免。 |
| **时间精度** | **纳秒**（`std::chrono::steady_clock` 或 `rdtsc`），不复用 `ScopedTimer` | `ScopedTimer` 回调单位是**微秒**（`common/Time.h:39`），B+ 树单次操作常 < 1µs，会被截断为 0。 |
| **聚合方式** | 全局单例 `CXLBTreeProfiler`，**每线程槽位**累积，退出时合并 | 与 `SCCManager`（全局 `atomic` 计数 + `print_stats()`，`SCCManager.h:38-88`）/ `MigrationManager`（`MigrationManager.cpp`）一致；热路径**无锁**。 |
| **指标内容** | 每种操作记录：`count / sum / max / 分位数(p50/p75/p95/p99) / OLC 重试次数` | `count+sum+max` 用 `atomic` 永远开启；分位数用每线程样本缓冲，退出时灌入 `Percentile`（`common/Percentile.h`）。 |
| **收集时机** | 程序结束、worker join 之后，在 `Coordinator` 退出流程打印 | 紧挨现有 `scc_manager->print_stats()`（`core/Coordinator.h:371`）。 |
| **零侵入开关** | 宏 `TIGON_BTREE_PROFILING` 包裹所有打点，默认关闭 | benchmark 跑分不受影响；诊断时编译期开启。 |

---

## 2. 增删改查 → 代码入口映射

| 语义 | wrapper 方法（`core/CXLTable.h`） | 底层 CXL B+ 树方法（`BTreeOLC_CXL.h`） | 事务侧调用点（举例） |
|---|---|---|---|
| **查 (point)** | `search()` `:132` | `lookup(k, value)` `:2648` | `TwoPLPashaHelper.h:get_migrated_row()`（远端读命中） |
| **查 (range)** | `scan()` `:147` | `scanForUpdate(min_k, proc)` `:2292` | `TwoPLPashaExecutor.h:357-358`（远端扫描） |
| **增** | `insert()` `:164` | `insert(k, value)` `:1668` | `TwoPLPashaHelper.h` 迁入 / `move_from_*_to_shared_region`（`:1343/1494`） |
| **删** | `remove()` `:179` | `remove(k)` `:2180` | `TwoPLPashaHelper.h` 迁出 / 删除（`:1656/1778`） |
| **改** | *（当前 wrapper 未暴露）* | `lookupForUpdate(k, proc)` `:2657` | 现状：值原地更新走 `search()` + SCC `do_write`（见 PERF_SPEC §5.6）；如需 B+ 树级原子改值，建议新增 wrapper `update()` 包装 `lookupForUpdate`。 |

> **说明**：`scan()` wrapper 实际调用的是 `scanForUpdate`（`CXLTable.h:161`），会对叶子加写锁，因此"扫"本身具备"改"能力。点更新（改单值）在 Tigon 当前路径是 `search()` 命中后对行数据原地写，不经过 B+ 树结构更新——本文为完整起见，给出可选的 `update()` wrapper 打点。

---

## 3. 新增文件①：`common/CXLBTreeProfiler.h`

数据收集接口的核心。提供 `record()`（热路径打点）与 `report()`（退出汇总）。

```cpp
// common/CXLBTreeProfiler.h
#pragma once

#include <atomic>
#include <cstdint>
#include <mutex>
#include <vector>
#include <array>
#include <x86intrin.h>     // __rdtsc
#include <glog/logging.h>
#include "common/Percentile.h"

namespace star
{

// 增删改查 + 扫描 的操作类型
enum class BTreeOp : int { Search = 0, Scan = 1, Insert = 2, Remove = 3, Update = 4, COUNT = 5 };

static inline const char *btree_op_name(BTreeOp op)
{
        switch (op) {
        case BTreeOp::Search: return "SEARCH(查)";
        case BTreeOp::Scan:   return "SCAN(扫)";
        case BTreeOp::Insert: return "INSERT(增)";
        case BTreeOp::Remove: return "REMOVE(删)";
        case BTreeOp::Update: return "UPDATE(改)";
        default:              return "UNKNOWN";
        }
}

class CXLBTreeProfiler {
    public:
        static constexpr int kNumOps = static_cast<int>(BTreeOp::COUNT);

        // 每线程槽位：热路径只写本线程数据，天然无锁
        struct ThreadSlot {
                struct PerOp {
                        uint64_t count = 0;
                        uint64_t sum_ns = 0;
                        uint64_t max_ns = 0;
                        uint64_t restarts = 0;       // OLC 重试累计（可选）
                        std::vector<uint64_t> samples;  // 供分位数；按采样率写入
                };
                std::array<PerOp, kNumOps> ops;
        };

        // 在每个线程首次打点时调用一次，登记本线程槽位
        ThreadSlot *register_thread()
        {
                std::lock_guard<std::mutex> g(reg_mutex_);
                slots_.push_back(new ThreadSlot());
                return slots_.back();
        }

        // 热路径打点：ns 为本次操作纯耗时，n_restart 为本次 OLC 重试次数
        static inline void record(ThreadSlot *slot, BTreeOp op, uint64_t ns, uint64_t n_restart = 0)
        {
                auto &o = slot->ops[static_cast<int>(op)];
                o.count++;
                o.sum_ns += ns;
                if (ns > o.max_ns) o.max_ns = ns;
                o.restarts += n_restart;
                // 采样：每 8 次取 1 次进入分位数缓冲，控制内存与开销
                if ((o.count & 0x7) == 0)
                        o.samples.push_back(ns);
        }

        // 程序结束时统一收集 + 打印（每个 host 进程一份）
        void report(std::size_t coordinator_id)
        {
                std::lock_guard<std::mutex> g(reg_mutex_);
                for (int i = 0; i < kNumOps; i++) {
                        uint64_t total_cnt = 0, total_sum = 0, total_max = 0, total_restart = 0;
                        Percentile<uint64_t> pct;
                        for (auto *s : slots_) {
                                auto &o = s->ops[i];
                                total_cnt += o.count;
                                total_sum += o.sum_ns;
                                total_restart += o.restarts;
                                if (o.max_ns > total_max) total_max = o.max_ns;
                                pct.add(o.samples);     // 合并样本（Percentile.h:35）
                        }
                        if (total_cnt == 0) continue;
                        LOG(INFO) << "[CXL-BTree Profiler] host " << coordinator_id
                                  << " op=" << btree_op_name(static_cast<BTreeOp>(i))
                                  << " cnt=" << total_cnt
                                  << " avg_ns=" << (total_sum / total_cnt)
                                  << " p50=" << pct.nth(50) << " p75=" << pct.nth(75)
                                  << " p95=" << pct.nth(95) << " p99=" << pct.nth(99)
                                  << " max_ns=" << total_max
                                  << " olc_restarts=" << total_restart
                                  << " restart_ratio=" << (100.0 * total_restart / total_cnt) << "%";
                }
        }

    private:
        std::mutex reg_mutex_;
        std::vector<ThreadSlot *> slots_;
};

// 全局单例（与 scc_manager / migration_manager 同风格）
extern CXLBTreeProfiler *cxl_btree_profiler;

// 每线程槽位指针（首次使用时登记）
inline CXLBTreeProfiler::ThreadSlot *get_btree_profiler_slot()
{
        static thread_local CXLBTreeProfiler::ThreadSlot *slot = nullptr;
        if (__builtin_expect(slot == nullptr, 0)) {
                if (cxl_btree_profiler != nullptr)
                        slot = cxl_btree_profiler->register_thread();
        }
        return slot;
}

// RAII 计时器（纳秒），析构时自动 record。配合 n_restart 引用回填。
class BTreeScopedNs {
    public:
        BTreeScopedNs(BTreeOp op, uint64_t *restart_ref = nullptr)
                : op_(op), restart_ref_(restart_ref)
        {
                start_ = std::chrono::steady_clock::now();
        }
        ~BTreeScopedNs()
        {
                auto slot = get_btree_profiler_slot();
                if (slot == nullptr) return;
                uint64_t ns = std::chrono::duration_cast<std::chrono::nanoseconds>(
                                      std::chrono::steady_clock::now() - start_).count();
                CXLBTreeProfiler::record(slot, op_, ns, restart_ref_ ? *restart_ref_ : 0);
        }
    private:
        BTreeOp op_;
        uint64_t *restart_ref_;
        std::chrono::steady_clock::time_point start_;
};

} // namespace star
```

---

## 4. 新增文件②：`common/CXLBTreeProfiler.cpp`

仅定义全局单例（与 `protocol/Pasha/SCCManager.cpp` 完全同构）：

```cpp
// common/CXLBTreeProfiler.cpp
#include "common/CXLBTreeProfiler.h"

namespace star {

CXLBTreeProfiler *cxl_btree_profiler = nullptr;

}
```

初始化在 `TwoPLPashaExecutor` 构造里、与 `scc_manager` 一起完成（`TwoPLPashaExecutor.h:57` 附近）：

```cpp
// protocol/TwoPLPasha/TwoPLPashaExecutor.h，在 if (id == 0) { ... } 内，scc_manager 初始化后追加：
#ifdef TIGON_BTREE_PROFILING
        cxl_btree_profiler = new CXLBTreeProfiler();
#endif
```
> 并在该文件顶部 `#include "common/CXLBTreeProfiler.h"`。

---

## 5. 修改①：在 `core/CXLTable.h` 打点（推荐主方案）

在 `CXLTableBTreeOLC` 的 4 个方法外层套 `BTreeScopedNs`。**纯增量改动**，不动函数逻辑。

**文件顶部**（`core/CXLTable.h:8` 后）追加：
```cpp
#include "common/CXLBTreeProfiler.h"
```

**查 — `search()`（`CXLTable.h:132`）：**
```cpp
virtual void *search(const void *key) override
{
#ifdef TIGON_BTREE_PROFILING
        BTreeScopedNs _t(BTreeOp::Search);
#endif
        const auto &k = *static_cast<const KeyType *>(key);
        BTreeOLCValue value;
        bool success = cxl_btree_->lookup(k, value);
        ...
}
```

**扫 — `scan()`（`CXLTable.h:147`）：**
```cpp
virtual void scan(const void *min_key, std::function<bool(const void *, void *, bool)> scan_processor) override
{
#ifdef TIGON_BTREE_PROFILING
        BTreeScopedNs _t(BTreeOp::Scan);
#endif
        const auto &min_k = *static_cast<const KeyType *>(min_key);
        ...
        cxl_btree_->scanForUpdate(min_k, processor);
}
```
> 注意：scan 的处理回调内含用户逻辑，若想只测 B+ 树遍历本身而排除回调耗时，应改为在 `scanForUpdate` 内部打点（见 §6 思路）。本层测的是"含回调的整段扫描时延"。

**增 — `insert()`（`CXLTable.h:164`）：**
```cpp
virtual bool insert(const void *key, void *row, bool is_placeholder = false) override
{
#ifdef TIGON_BTREE_PROFILING
        BTreeScopedNs _t(BTreeOp::Insert);
#endif
        ...
        bool success = cxl_btree_->insert(k, value);
        return success;
}
```

**删 — `remove()`（`CXLTable.h:179`）：**
```cpp
virtual bool remove(const void *key, void *row) override
{
#ifdef TIGON_BTREE_PROFILING
        BTreeScopedNs _t(BTreeOp::Remove);
#endif
        ...
        bool success = cxl_btree_->remove(k);
        ...
}
```

**改 —（可选）新增 `update()` wrapper**，包装底层 `lookupForUpdate`（`BTreeOLC_CXL.h:2657`），并在 `CXLTableBase`（`CXLTable.h:15-30`）加纯虚声明：
```cpp
// CXLTableBase 接口追加：
virtual bool update(const void *key, std::function<void(void *row)> updater) = 0;

// CXLTableBTreeOLC 实现：
virtual bool update(const void *key, std::function<void(void *row)> updater) override
{
#ifdef TIGON_BTREE_PROFILING
        BTreeScopedNs _t(BTreeOp::Update);
#endif
        const auto &k = *static_cast<const KeyType *>(key);
        return cxl_btree_->lookupForUpdate(k, [&](const KeyType &, BTreeOLCValue &v) {
                updater(v.row.get());
        });
}
// CXLTableHashMap 也需补一个 CHECK(0) 的同名实现以满足接口。
```

---

## 6. 修改②（可选）：在 `BTreeOLC_CXL.h` 内统计 OLC 重试

wrapper 层测的是"含重试的端到端时延"。若要**单独量化 OLC 乐观锁重试放大**（读写争用下 `goto restart` 次数），在各公有方法的 `restart:` 循环里加一个**栈上局部计数器**（**不改共享内存布局，安全**），并把它通过 `BTreeScopedNs` 的 `restart_ref` 回填。

以 `lookup`（`BTreeOLC_CXL.h:2648` → 重试标签 `:3251`）为例：

```cpp
// _lookup(...) 内，restart: 标签处
bool _lookup(const KeyType &key, ValueType &result)
{
        uint64_t n_restart = 0;          // 新增：栈上局部，零共享开销
restart:
        // ... 原逻辑 ...
        if (needRestart) { ++n_restart; goto restart; }   // 在每个 goto restart 前自增
        // ... 成功返回前，把 n_restart 暴露出去（见下）
}
```

为把 `n_restart` 传到 profiler，最简方案是在 wrapper 调用前后比较一个 thread_local 计数，或给底层方法增加一个可选 `uint64_t *out_restart = nullptr` 出参。推荐后者（改动可控）：

```cpp
// 公有方法签名增可选出参，默认 nullptr，老调用方不受影响：
bool lookup(const KeyType &key, ValueType &result, uint64_t *out_restart = nullptr);
// 返回前： if (out_restart) *out_restart = n_restart;
```

wrapper 端：
```cpp
virtual void *search(const void *key) override
{
        uint64_t n_restart = 0;
#ifdef TIGON_BTREE_PROFILING
        BTreeScopedNs _t(BTreeOp::Search, &n_restart);
#endif
        ...
        bool success = cxl_btree_->lookup(k, value, &n_restart);
        ...
}
```

> 需要同样处理的公有方法及其重试标签行号：`insert`(1668/1673)、`insert(...,result*)`(1832/1837)、`remove`(2180→`_remove`2961/2968)、`scan`(2206/2211)、`scanForUpdate`(2292/2300)、`lookup`(2648→`_lookup`3248/3251)、`lookupForUpdate`(2657→3101/3106)。

---

## 7. 修改③：程序结束时的数据收集 Hook（`core/Coordinator.h`）

在 worker 全部 `onExit()`/join 之后、与现有 `scc_manager->print_stats()` 并列处（`core/Coordinator.h:369-371`）追加：

```cpp
                // print software cache-coherence stats
                if (scc_manager != nullptr)
                        scc_manager->print_stats();

#ifdef TIGON_BTREE_PROFILING
                // print CXL B+tree CRUD latency stats（新增：统一数据收集接口）
                if (cxl_btree_profiler != nullptr)
                        cxl_btree_profiler->report(this->id);
#endif
```
> 文件顶部 `#include "common/CXLBTreeProfiler.h"`（`Coordinator.h` 已 include 多个 common 头，追加一行即可）。
> 该位置保证所有 worker 线程已停止，槽位数据稳定，可安全合并（`report()` 内持 `reg_mutex_`）。

可选：若想导出 CDF，`report()` 内对每个 `Percentile` 调 `save_cdf(path)`（`Percentile.h:70`），路径用 `context.cdf_path` 派生。

---

## 8. 修改④：构建集成（CMake）

`CMakeLists.txt` 将新增 `.cpp` 纳入编译，并提供开关选项：

```cmake
# 源文件列表中追加：
#   common/CXLBTreeProfiler.cpp

# 新增编译开关（默认 OFF）：
option(TIGON_BTREE_PROFILING "Enable CXL B+tree CRUD latency profiling" OFF)
if (TIGON_BTREE_PROFILING)
    add_compile_definitions(TIGON_BTREE_PROFILING)
endif()
```
> 现有 `CMakeLists.txt` 若用 `file(GLOB ... *.cpp)` 收集源文件，则 `.cpp` 自动纳入，仅需加 `option`/`add_compile_definitions`。

开启诊断构建：`cmake -DTIGON_BTREE_PROFILING=ON ..`

---

## 9. 线程安全、开销与采样

- **热路径无锁**：`record()` 只写**本线程**槽位（`thread_local` 指针，`get_btree_profiler_slot()`），无原子、无锁。
- **登记一次性加锁**：`register_thread()` 仅每线程首次调用时持锁，之后不再。
- **退出合并持锁**：`report()` 持 `reg_mutex_` 串行合并，此时 worker 已停。
- **内存有界**：分位数样本按 1/8 采样（`(count & 0x7)==0`），可调；`count/sum/max` 全量但仅 3 个标量。
- **计时开销**：`steady_clock::now()` 约 ~20ns。查（search）是最热操作（每次远端读都走），若开销敏感，可：
  - 把 `BTreeScopedNs` 内部换成 `__rdtsc()` 差值（再除以 CPU 频率换算 ns），更轻量；或
  - 仅在 `warmed_up`（`Percentile.h:18` 的 extern）为真时打点。
- **默认零开销**：未定义 `TIGON_BTREE_PROFILING` 时所有打点被预处理器删除，release/benchmark 不受影响。

---

## 10. 示例输出与验证

程序结束时（每个 host）预期日志：
```
[CXL-BTree Profiler] host 0 op=SEARCH(查) cnt=128450 avg_ns=312 p50=270 p75=340 p95=560 p99=910 max_ns=8200 olc_restarts=1875 restart_ratio=1.46%
[CXL-BTree Profiler] host 0 op=SCAN(扫)   cnt=3120   avg_ns=1850 p50=1600 p75=2100 p95=3900 p99=6100 max_ns=21000 olc_restarts=410 restart_ratio=13.14%
[CXL-BTree Profiler] host 0 op=INSERT(增) cnt=940    avg_ns=2100 p50=1900 ... 
[CXL-BTree Profiler] host 0 op=REMOVE(删) cnt=512    avg_ns=2450 ...
```

**验证步骤：**
1. `cmake -DTIGON_BTREE_PROFILING=ON ..` 编译通过（确认新 `.cpp` 链接、单例符号唯一）。
2. 用 README 的 hello-world 命令跑 TPC-C：
   `./scripts/run.sh TPCC TwoPLPasha 8 3 mixed 10 15 1 0 1 Clock OnDemand 200000000 1 WriteThrough None 15 5 GROUP_WAL 20000 0 0`
3. 在退出日志中确认每个 host 打印 `[CXL-BTree Profiler]` 各操作行，`cnt` 与负载相符（mixed/多分区时 search/scan 计数应明显上升）。
4. 关掉宏重新编译，确认日志消失且吞吐与基线一致（验证零开销）。
5. 与 `PERF_MONITORING_SPEC.md` §5.7 的预期指标对照，必要时用 `save_cdf` 出图（`scripts/plot`）。

---

## 11. 改动清单速查

| 类型 | 文件 | 位置 | 改动 |
|---|---|---|---|
| 新增 | `common/CXLBTreeProfiler.h` | — | 收集接口：`CXLBTreeProfiler` + `BTreeScopedNs` + 单例声明 |
| 新增 | `common/CXLBTreeProfiler.cpp` | — | 定义 `cxl_btree_profiler` 单例 |
| 改 | `core/CXLTable.h` | `:8` / `:132,147,164,179` | include + 4 方法打点（+可选 `update()`） |
| 改 | `core/CXLTable.h` | `:15-30` | （可选）`CXLTableBase` 增 `update()` 纯虚 + HashMap 占位实现 |
| 改 | `protocol/TwoPLPasha/TwoPLPashaExecutor.h` | `:57` 附近 | include + `id==0` 时 `new CXLBTreeProfiler()` |
| 改（可选） | `common/btree_olc_cxl/BTreeOLC_CXL.h` | 各公有方法 `restart:` | 栈上 `n_restart` 计数 + 可选出参 |
| 改 | `core/Coordinator.h` | `:371` 后 | include + `cxl_btree_profiler->report(id)` |
| 改 | `CMakeLists.txt` | — | 纳入 `.cpp` + `option(TIGON_BTREE_PROFILING)` |

---

*本文档所有 `文件:行号` 基于分支 `claude/lucid-euler-SKPMw` 源码快照，可逐条核对。配套总览见 `PERF_MONITORING_SPEC.md`。*
