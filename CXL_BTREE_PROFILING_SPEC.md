# CXL B+ 树增删改查细粒度时延打点 — 源码修改方案 SPEC

> 版本：v2.0　|　日期：2026-06-10　|　对象：`core/CXLTable.h` + `core/Table.cpp` + `core/Coordinator.h`
>
> 本文是 [`PERF_MONITORING_SPEC.md`](./PERF_MONITORING_SPEC.md) §5.7 的落地实现。
> **约束（v2 更新）**：**不新增任何文件、不新增任何类/结构体/枚举**；计时统一用现有的
> **`star::Time`**；只复用现有设施（`extern std::atomic` 全局计数 + `LOG`），与
> `MigrationManager` 的 `num_data_move_in/out`（`protocol/Pasha/MigrationManager.cpp`）同风格。

---

## 0. 关于 `star::Time` 的可行性判断（结论：可行 ✅）

| 检查项 | 源码 | 结论 |
|---|---|---|
| 精度 | `common/Time.h:13-18`：`Time::now()` 用 `duration_cast<std::chrono::nanoseconds>` 返回 **纳秒** | B+ 树单次操作常 < 1µs，**纳秒精度足够**，不会像 `ScopedTimer`（回调单位为微秒，`Time.h:39`）那样被截断为 0 |
| 起点初始化 | `common/Time.cpp:9`：`Time::startTime = steady_clock::now()`（静态初始化） | 程序加载即有效，打点处直接 `Time::now()` 取差值即可 |
| 时钟单调性 | `steady_clock` | 单调，适合测时延 |
| 可用性 | 已被 `Dispatcher.h`/`ControlMessage.h`/`TwoPLPashaMessage.h` 等广泛使用 | 仅需在 `core/CXLTable.h` 顶部 `#include "common/Time.h"` |

**用法**：
```cpp
uint64_t t0 = star::Time::now();   // ns
/* ... 被测操作 ... */
uint64_t ns = star::Time::now() - t0;
```

---

## 目录
1. [设计决策（v2）](#1-设计决策v2)
2. [增删改查 → 代码入口映射](#2-增删改查--代码入口映射)
3. [修改①：全局计数器（`core/CXLTable.h` 声明 + `core/Table.cpp` 定义）](#3-修改全局计数器声明--定义)
4. [修改②：在 `core/CXLTable.h` wrapper 打点（用 `star::Time`）](#4-修改在-corecxltableh-wrapper-打点用-startime)
5. [修改③：程序结束时的数据收集（`core/Coordinator.h`）](#5-修改程序结束时的数据收集corecoordinatorh)
6. [修改④：构建开关（CMake）](#6-修改构建开关cmake)
7. [可选增强：要分位数时复用现有 `Executor`+`Percentile`](#7-可选增强要分位数时复用现有-executorpercentile)
8. [线程安全与开销](#8-线程安全与开销)
9. [示例输出与验证](#9-示例输出与验证)
10. [改动清单速查](#10-改动清单速查)

---

## 1. 设计决策（v2）

| 决策 | 选择 | 理由 |
|---|---|---|
| **不新增文件/类** | 复用 `extern std::atomic<uint64_t>` 全局变量（**自由变量，非类**）+ 自由 `inline` 函数 + `LOG` | 满足约束；与 `MigrationManager.cpp` 的 `num_data_move_in/out` 完全同构 |
| **计时** | `star::Time::now()`（纳秒） | 见 §0，可行且精度足够 |
| **打点层次** | wrapper 层 `CXLTableBTreeOLC`（`core/CXLTable.h:83-203`） | 增删改查天然边界（`search/scan/insert/remove` `:132/147/164/179`）；**驻 DRAM、每 host 一份**；底层 `btreeolc_cxl::BPlusTree` 在 **CXL 共享内存跨进程共享**（`:200`），**绝不能加字段** |
| **聚合粒度** | 每 host 进程一组全局原子计数（被该 host 所有 worker 线程共享累加） | 与 `scc_manager`/`migration_manager` 的 per-host 统计一致 |
| **指标** | 每操作：`cnt` / `sum_ns`(→avg) / `max_ns` | 原子、无锁、永远可开启；分位数见 §7 可选方案 |
| **收集时机** | `Coordinator` 退出、worker join 后，紧挨 `scc_manager->print_stats()`（`Coordinator.h:371`） | 数据已稳定 |
| **零开销开关** | 宏 `TIGON_BTREE_PROFILING`，默认 OFF | benchmark 不受影响 |

> **约束解读**：不新增 `.h/.cpp` 文件、不新增 `class/struct/enum`。允许：在**既有文件**中新增
> `extern` 全局变量、`inline` 自由函数、`LOG` 语句，以及给**既有类**加成员（§7 用到）。

---

## 2. 增删改查 → 代码入口映射

| 语义 | wrapper 方法（`core/CXLTable.h`） | 底层 CXL B+ 树（`BTreeOLC_CXL.h`） |
|---|---|---|
| **查 (point)** | `search()` `:132` | `lookup(k,value)` `:2648` |
| **查 (range)/扫** | `scan()` `:147` | `scanForUpdate(min_k,proc)` `:2292` |
| **增** | `insert()` `:164` | `insert(k,value)` `:1668` |
| **删** | `remove()` `:179` | `remove(k)` `:2180` |
| **改** | *(wrapper 未暴露)* | `lookupForUpdate(k,proc)` `:2657`；当前点更新走 `search()`+SCC `do_write`（PERF_SPEC §5.6） |

---

## 3. 修改①：全局计数器（声明 + 定义）

**声明** —— 在 `core/CXLTable.h` 的 `namespace star {` 内（例如紧跟 `:13` 之后）追加；同时在文件顶部
`#include "common/Time.h"` 与 `#include <atomic>`：

```cpp
// core/CXLTable.h —— 顶部 include 区
#include "common/Time.h"
#include <atomic>

// core/CXLTable.h —— namespace star 内，全局计数器（增删改查 + 扫）
// 命名沿用 MigrationManager.cpp 中 num_data_move_in/out 的全局原子风格
extern std::atomic<uint64_t> cxl_btree_search_cnt, cxl_btree_search_ns, cxl_btree_search_max_ns;
extern std::atomic<uint64_t> cxl_btree_scan_cnt,   cxl_btree_scan_ns,   cxl_btree_scan_max_ns;
extern std::atomic<uint64_t> cxl_btree_insert_cnt, cxl_btree_insert_ns, cxl_btree_insert_max_ns;
extern std::atomic<uint64_t> cxl_btree_remove_cnt, cxl_btree_remove_ns, cxl_btree_remove_max_ns;

// 无锁 max 更新（自由 inline 函数，非类）；C++17 无 atomic::fetch_max，用 CAS 循环
static inline void cxl_btree_atomic_max(std::atomic<uint64_t> &m, uint64_t v)
{
        uint64_t cur = m.load(std::memory_order_relaxed);
        while (v > cur && !m.compare_exchange_weak(cur, v, std::memory_order_relaxed)) { }
}
```

**定义** —— 在**既有文件** `core/Table.cpp`（属于 `CMakeLists.txt:27` 的 `core/*.cpp` GLOB，自动编译）中追加：

```cpp
// core/Table.cpp 末尾，namespace star 内
#include "core/CXLTable.h"   // 若尚未包含

namespace star {
std::atomic<uint64_t> cxl_btree_search_cnt{0}, cxl_btree_search_ns{0}, cxl_btree_search_max_ns{0};
std::atomic<uint64_t> cxl_btree_scan_cnt{0},   cxl_btree_scan_ns{0},   cxl_btree_scan_max_ns{0};
std::atomic<uint64_t> cxl_btree_insert_cnt{0}, cxl_btree_insert_ns{0}, cxl_btree_insert_max_ns{0};
std::atomic<uint64_t> cxl_btree_remove_cnt{0}, cxl_btree_remove_ns{0}, cxl_btree_remove_max_ns{0};
}
```

---

## 4. 修改②：在 `core/CXLTable.h` wrapper 打点（用 `star::Time`）

只在每个方法首尾加计时与原子累加；用宏包裹保证默认零开销。为避免多处 `return` 漏记，
对有提前返回的方法把返回值收敛到一个局部变量后统一记录。

**查 — `search()`（`CXLTable.h:132-145`）改为单出口：**
```cpp
virtual void *search(const void *key) override
{
#ifdef TIGON_BTREE_PROFILING
        uint64_t _t0 = star::Time::now();
#endif
        const auto &k = *static_cast<const KeyType *>(key);
        BTreeOLCValue value;
        bool success = cxl_btree_->lookup(k, value);
        void *ret = nullptr;
        if (success == true) {
                CHECK(value.is_valid == true);
                ret = value.row.get();
        }
#ifdef TIGON_BTREE_PROFILING
        uint64_t _ns = star::Time::now() - _t0;
        cxl_btree_search_cnt.fetch_add(1, std::memory_order_relaxed);
        cxl_btree_search_ns.fetch_add(_ns, std::memory_order_relaxed);
        cxl_btree_atomic_max(cxl_btree_search_max_ns, _ns);
#endif
        return ret;
}
```

**扫 — `scan()`（`CXLTable.h:147-162`，单出口，最简单）：**
```cpp
virtual void scan(const void *min_key, std::function<bool(const void *, void *, bool)> scan_processor) override
{
#ifdef TIGON_BTREE_PROFILING
        uint64_t _t0 = star::Time::now();
#endif
        const auto &min_k = *static_cast<const KeyType *>(min_key);
        auto processor = [&](const KeyType &key, BTreeOLCValue &value, bool is_last_tuple) -> bool {
                return scan_processor(&key, value.row.get(), is_last_tuple);
        };
        cxl_btree_->scanForUpdate(min_k, processor);
#ifdef TIGON_BTREE_PROFILING
        uint64_t _ns = star::Time::now() - _t0;
        cxl_btree_scan_cnt.fetch_add(1, std::memory_order_relaxed);
        cxl_btree_scan_ns.fetch_add(_ns, std::memory_order_relaxed);
        cxl_btree_atomic_max(cxl_btree_scan_max_ns, _ns);
#endif
}
```
> 注：scan 计的是"含用户回调"的整段扫描时延（回调在事务侧做下一键加锁等逻辑）。若要剔除回调、只测纯遍历，需深入 `scanForUpdate` 内部打点——但那在共享内存代码里，权衡后本方案测整段，语义更贴近"事务看到的扫描时延"。

**增 — `insert()`（`CXLTable.h:164-177`，已有 `success`）：**
```cpp
virtual bool insert(const void *key, void *row, bool is_placeholder = false) override
{
#ifdef TIGON_BTREE_PROFILING
        uint64_t _t0 = star::Time::now();
#endif
        const auto &k = *static_cast<const KeyType *>(key);
        BTreeOLCValue value;
        value.row = row;
        value.is_valid.store(is_placeholder == true ? false : true);
        bool success = cxl_btree_->insert(k, value);
#ifdef TIGON_BTREE_PROFILING
        uint64_t _ns = star::Time::now() - _t0;
        cxl_btree_insert_cnt.fetch_add(1, std::memory_order_relaxed);
        cxl_btree_insert_ns.fetch_add(_ns, std::memory_order_relaxed);
        cxl_btree_atomic_max(cxl_btree_insert_max_ns, _ns);
#endif
        return success;
}
```

**删 — `remove()`（`CXLTable.h:179-187`）：**
```cpp
virtual bool remove(const void *key, void *row) override
{
#ifdef TIGON_BTREE_PROFILING
        uint64_t _t0 = star::Time::now();
#endif
        const auto &k = *static_cast<const KeyType *>(key);
        bool success = cxl_btree_->remove(k);
        CHECK(success == true);
#ifdef TIGON_BTREE_PROFILING
        uint64_t _ns = star::Time::now() - _t0;
        cxl_btree_remove_cnt.fetch_add(1, std::memory_order_relaxed);
        cxl_btree_remove_ns.fetch_add(_ns, std::memory_order_relaxed);
        cxl_btree_atomic_max(cxl_btree_remove_max_ns, _ns);
#endif
        return success;
}
```

> **改（改值）**：当前 wrapper 不暴露 update（点更新走 `search()`+SCC 写，已被 `search` 计时覆盖一半）。若日后需要 B+ 树级原子改值，可仿照上面包装 `cxl_btree_->lookupForUpdate`（`BTreeOLC_CXL.h:2657`）并复用同样的打点四件套——**仍无需新类/新文件**。

---

## 5. 修改③：程序结束时的数据收集（`core/Coordinator.h`）

在 `scc_manager->print_stats()` 之后（`core/Coordinator.h:371`）追加几行 `LOG`（用一个**局部 lambda**算 avg，非类）：

```cpp
                // print software cache-coherence stats
                if (scc_manager != nullptr)
                        scc_manager->print_stats();

#ifdef TIGON_BTREE_PROFILING
                // ===== CXL B+tree 增删改查时延收集（每 host 一份） =====
                auto _avg = [](std::atomic<uint64_t> &sum, std::atomic<uint64_t> &cnt) -> uint64_t {
                        uint64_t c = cnt.load();
                        return c ? sum.load() / c : 0;
                };
                LOG(INFO) << "[CXL-BTree CRUD] host " << id
                          << " | SEARCH cnt=" << cxl_btree_search_cnt.load()
                          << " avg_ns=" << _avg(cxl_btree_search_ns, cxl_btree_search_cnt)
                          << " max_ns=" << cxl_btree_search_max_ns.load()
                          << " | SCAN cnt=" << cxl_btree_scan_cnt.load()
                          << " avg_ns=" << _avg(cxl_btree_scan_ns, cxl_btree_scan_cnt)
                          << " max_ns=" << cxl_btree_scan_max_ns.load()
                          << " | INSERT cnt=" << cxl_btree_insert_cnt.load()
                          << " avg_ns=" << _avg(cxl_btree_insert_ns, cxl_btree_insert_cnt)
                          << " max_ns=" << cxl_btree_insert_max_ns.load()
                          << " | REMOVE cnt=" << cxl_btree_remove_cnt.load()
                          << " avg_ns=" << _avg(cxl_btree_remove_ns, cxl_btree_remove_cnt)
                          << " max_ns=" << cxl_btree_remove_max_ns.load();
#endif
```
> `core/Coordinator.h` 已 include 多个 common 头；`core/CXLTable.h` 中声明的 extern 通过现有
> include 链可见（Coordinator → Executor → ... → Table/CXLTable）。如不可见，在 `Coordinator.h`
> 顶部补 `#include "core/CXLTable.h"` 即可。该处 worker 已全部 join，原子计数已稳定。

---

## 6. 修改④：构建开关（CMake）

`core/Table.cpp` 已被 `CMakeLists.txt:27` 的 `file(GLOB_RECURSE ... core/*.cpp)` 收集，**无需新增源文件**。仅加开关：

```cmake
option(TIGON_BTREE_PROFILING "Enable CXL B+tree CRUD latency profiling" OFF)
if (TIGON_BTREE_PROFILING)
    add_compile_definitions(TIGON_BTREE_PROFILING)
endif()
```
开启：`cmake -DTIGON_BTREE_PROFILING=ON ..`

---

## 7. 可选增强：要分位数时复用现有 `Executor`+`Percentile`

§3-§5 的全局原子方案给出 `cnt/avg/max`。若还要 **p50/p95/p99 分位数**，**仍不新增类/文件**——
给**既有类** `Executor`（`core/Executor.h`）加 `Percentile<uint64_t>` 成员（`Percentile` 是既有类），
在既有 `onExit()` 里打印（与现有 `commit_latency` 等一致）：

- 加成员（`Executor.h:389` 附近）：
  ```cpp
  Percentile<uint64_t> cxl_btree_search_pct, cxl_btree_scan_pct, cxl_btree_insert_pct, cxl_btree_remove_pct;
  ```
- 打点改为记录到"当前 worker 线程的 Executor"。由于 wrapper 被多线程共享、`Percentile` 非线程安全，
  必须记录到**每线程**对象。两种接法：
  - **(推荐)** 在 worker 线程可见 `this`（Executor）的 **CXL B+ 树调用点** 处用 `Time::now()` 计时并
    `this->cxl_btree_*_pct.add(ns)`：
    - 查：`TwoPLPashaExecutor.h:136` 包住 `twopl_pasha_global_helper->get_migrated_row(...)`
    - 扫：`TwoPLPashaExecutor.h:358` 包住 `target_cxl_table->scan(...)`
    - 增/删：迁移路径在 worker 线程的 `process_request()`/`commit()` 内执行，于对应调用点记录。
  - 在既有 `onExit()`（`Executor.h:231`）的 LOG 末尾追加这几个 `nth(50/95/99)`。
- `Percentile::add` 受 `warmed_up` 与 10% 采样约束（`Percentile.h:28`），开销可控。

> 该增强保持"零新文件/新类"：只是给既有 `Executor` 加成员、复用既有 `Percentile`/`Time`。
> 默认主方案（§3-§5 全局原子）已足够定位时延；分位数按需开启。

---

## 8. 线程安全与开销

- **线程安全**：wrapper 实例被一个 host 的多 worker 线程共享 → 用 `std::atomic` 累加，**无数据竞争、无锁**。
- **`max` 更新**：`cxl_btree_atomic_max` 用 `compare_exchange_weak` 自旋，仅在刷新最大值时偶发循环。
- **计时开销**：`star::Time::now()` ≈ 一次 `steady_clock::now()` + 一次减法，约 ~20ns；查（search）最热（每次远端读都过），可只在 `warmed_up`（`Percentile.h:18` 的 extern）为真时打点进一步降噪。
- **默认零开销**：未定义 `TIGON_BTREE_PROFILING` 时，所有打点被预处理器移除。
- **内存**：仅 12 个 `uint64_t` 原子，常数级。

---

## 9. 示例输出与验证

退出日志（每 host 一行）：
```
[CXL-BTree CRUD] host 0 | SEARCH cnt=128450 avg_ns=312 max_ns=8200 | SCAN cnt=3120 avg_ns=1850 max_ns=21000 | INSERT cnt=940 avg_ns=2100 max_ns=15300 | REMOVE cnt=512 avg_ns=2450 max_ns=17800
```

**验证：**
1. `cmake -DTIGON_BTREE_PROFILING=ON .. && make` 通过（确认 extern 链接唯一、无重定义）。
2. 跑 README hello-world（TPC-C / TwoPLPasha）。
3. 退出日志出现 `[CXL-BTree CRUD]`；提高多分区比例时 SEARCH/SCAN 的 `cnt` 应明显上升。
4. 关宏重编，确认日志消失且吞吐与基线一致（零开销）。
5. 如需分布，按 §7 开启分位数变体并核对。

---

## 10. 改动清单速查

| 类型 | 文件（均为**既有**） | 位置 | 改动 |
|---|---|---|---|
| 改 | `core/CXLTable.h` | 顶部 | `#include "common/Time.h"`、`<atomic>` |
| 改 | `core/CXLTable.h` | `namespace star` 内 | `extern` 12 个原子计数 + `inline cxl_btree_atomic_max` |
| 改 | `core/CXLTable.h` | `:132/147/164/179` | 4 方法用 `star::Time::now()` 打点（search 改单出口） |
| 改 | `core/Table.cpp` | 末尾 | 定义 12 个原子计数（含 include CXLTable.h） |
| 改 | `core/Coordinator.h` | `:371` 后 | `#ifdef` 内 `LOG` 收集输出（局部 lambda 算 avg） |
| 改 | `CMakeLists.txt` | — | `option(TIGON_BTREE_PROFILING)` + `add_compile_definitions` |
| 改（可选） | `core/Executor.h` + `protocol/TwoPLPasha/TwoPLPashaExecutor.h` | `:389` / `:136,358` | 加 `Percentile` 成员 + 调用点打点 + `onExit()` 打印（要分位数时） |

> **无新增文件、无新增类/结构体/枚举**；计时统一用 `star::Time`。

---

*基于分支 `claude/lucid-euler-SKPMw` 源码快照，逐条可核对。配套总览见 `PERF_MONITORING_SPEC.md`。*
