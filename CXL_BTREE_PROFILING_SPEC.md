# CXL B+ 树增删改查细粒度时延打点 — 源码修改方案 SPEC

> 版本：v4.0　|　日期：2026-06-10　|　对象：`core/CXLTable.h` + `core/Table.cpp` + `core/Coordinator.h`
>
> 本文是 [`PERF_MONITORING_SPEC.md`](./PERF_MONITORING_SPEC.md) §5.7 的落地实现。
> **约束**：**不新增文件、不新增类/结构体/枚举**；计时用现有 **`star::Time`**；**不使用编译开关宏**；
> **所有时延变量统一用现有的 `star::Percentile` 收集**（`common/Percentile.h`）。

---

## 0. 关于 `star::Time` 的可行性判断（结论：可行 ✅）

| 检查项 | 源码 | 结论 |
|---|---|---|
| 精度 | `common/Time.h:13-18`：`Time::now()` 返回 **纳秒** | <1µs 的 B+ 树操作不会被截断（区别于 `ScopedTimer` 微秒回调 `Time.h:39`） |
| 起点 | `common/Time.cpp:9`：`startTime` 静态初始化 | 程序加载即有效 |
| 用法 | `uint64_t t0=Time::now(); ...; uint64_t ns=Time::now()-t0;` | 仅需 `#include "common/Time.h"` |

---

## 1. 设计决策

| 决策 | 选择 | 理由 |
|---|---|---|
| **统一用 `Percentile` 收集** | 每种操作一个全局 `star::Percentile<uint64_t>`（既有类，`common/Percentile.h`），存纳秒时延 | 满足"所有相关变量用 Percentile"；可直接出 `avg()/nth(50/75/95/99)` 与 CDF（`save_cdf`） |
| **线程安全** | 全局 `std::mutex` 短临界区保护 `Percentile::add` | `Percentile` 非线程安全（`add` 内 `data_.push_back` `Percentile.h:31`）；wrapper 被一个 host 的多 worker 共享 |
| **可接受锁开销** | CXL B+ 树只服务**迁移/远端**数据（非本地 DRAM 热路径），且 `Percentile::add` 仅在 `warmed_up` 时按 ~10% 采样（`Percentile.h:28`） | 实际进入临界区的频率很低，锁竞争可忽略；如仍敏感见 §6 无锁变体 |
| **不新增文件/类/宏** | 复用 `extern Percentile` 全局变量 + `extern std::mutex` + `LOG` | 与 `MigrationManager.cpp` 全局变量风格一致 |
| **计时** | `star::Time::now()`（纳秒） | 见 §0 |
| **打点层次** | wrapper 层 `CXLTableBTreeOLC`（`core/CXLTable.h:83-203`） | 增删改查天然边界（`:132/147/164/179`）；驻 DRAM、每 host 一份；底层 `BPlusTree` 在 CXL 共享内存跨进程共享（`:200`），**绝不能加字段** |
| **收集时机** | `Coordinator` 退出、worker join 后，紧挨 `scc_manager->print_stats()`（`Coordinator.h:371`） | 数据稳定，`nth()` 排序安全 |

---

## 2. 增删改查 → 代码入口映射

| 语义 | wrapper 方法（`core/CXLTable.h`） | 底层 CXL B+ 树（`BTreeOLC_CXL.h`） | 对应 Percentile |
|---|---|---|---|
| **查 (point)** | `search()` `:132` | `lookup` `:2648` | `cxl_btree_search_pct` |
| **查 (range)/扫** | `scan()` `:147` | `scanForUpdate` `:2292` | `cxl_btree_scan_pct` |
| **增** | `insert()` `:164` | `insert` `:1668` | `cxl_btree_insert_pct` |
| **删** | `remove()` `:179` | `remove` `:2180` | `cxl_btree_remove_pct` |
| **改** | *(未暴露)* | `lookupForUpdate` `:2657` | （需要时加 `cxl_btree_update_pct`） |

---

## 3. 修改①：全局 `Percentile` 变量（声明 + 定义）

**声明** —— `core/CXLTable.h` 顶部追加 include；`namespace star {` 内（`:13` 后）追加全局变量：

```cpp
// core/CXLTable.h —— 顶部 include 区
#include "common/Time.h"
#include "common/Percentile.h"
#include <mutex>

// core/CXLTable.h —— namespace star 内
// 所有时延统一用 Percentile 收集（纳秒）；一个全局 mutex 保护并发 add
extern std::mutex                cxl_btree_pct_mutex;
extern star::Percentile<uint64_t> cxl_btree_search_pct;
extern star::Percentile<uint64_t> cxl_btree_scan_pct;
extern star::Percentile<uint64_t> cxl_btree_insert_pct;
extern star::Percentile<uint64_t> cxl_btree_remove_pct;
```

**定义** —— 既有文件 `core/Table.cpp`（已被 `CMakeLists.txt:27` 的 `core/*.cpp` GLOB 自动编译）追加：

```cpp
// core/Table.cpp，namespace star 内
#include "core/CXLTable.h"   // 若尚未包含

namespace star {
std::mutex                cxl_btree_pct_mutex;
star::Percentile<uint64_t> cxl_btree_search_pct;
star::Percentile<uint64_t> cxl_btree_scan_pct;
star::Percentile<uint64_t> cxl_btree_insert_pct;
star::Percentile<uint64_t> cxl_btree_remove_pct;
}
```

---

## 4. 修改②：在 `core/CXLTable.h` wrapper 打点（`star::Time` 计时 + `Percentile` 收集）

每个方法首尾用 `star::Time::now()` 计时，临界区内 `Percentile::add(ns)`。有提前返回的方法收敛到单出口。

**查 — `search()`（`CXLTable.h:132-145`）：**
```cpp
virtual void *search(const void *key) override
{
        uint64_t _t0 = star::Time::now();
        const auto &k = *static_cast<const KeyType *>(key);
        BTreeOLCValue value;
        bool success = cxl_btree_->lookup(k, value);
        void *ret = nullptr;
        if (success == true) {
                CHECK(value.is_valid == true);
                ret = value.row.get();
        }
        uint64_t _ns = star::Time::now() - _t0;
        { std::lock_guard<std::mutex> _g(cxl_btree_pct_mutex); cxl_btree_search_pct.add(_ns); }
        return ret;
}
```

**扫 — `scan()`（`CXLTable.h:147-162`）：**
```cpp
virtual void scan(const void *min_key, std::function<bool(const void *, void *, bool)> scan_processor) override
{
        uint64_t _t0 = star::Time::now();
        const auto &min_k = *static_cast<const KeyType *>(min_key);
        auto processor = [&](const KeyType &key, BTreeOLCValue &value, bool is_last_tuple) -> bool {
                return scan_processor(&key, value.row.get(), is_last_tuple);
        };
        cxl_btree_->scanForUpdate(min_k, processor);
        uint64_t _ns = star::Time::now() - _t0;
        { std::lock_guard<std::mutex> _g(cxl_btree_pct_mutex); cxl_btree_scan_pct.add(_ns); }
}
```
> scan 计的是"含事务侧回调"的整段扫描时延（回调做下一键加锁等）。

**增 — `insert()`（`CXLTable.h:164-177`）：**
```cpp
virtual bool insert(const void *key, void *row, bool is_placeholder = false) override
{
        uint64_t _t0 = star::Time::now();
        const auto &k = *static_cast<const KeyType *>(key);
        BTreeOLCValue value;
        value.row = row;
        value.is_valid.store(is_placeholder == true ? false : true);
        bool success = cxl_btree_->insert(k, value);
        uint64_t _ns = star::Time::now() - _t0;
        { std::lock_guard<std::mutex> _g(cxl_btree_pct_mutex); cxl_btree_insert_pct.add(_ns); }
        return success;
}
```

**删 — `remove()`（`CXLTable.h:179-187`）：**
```cpp
virtual bool remove(const void *key, void *row) override
{
        uint64_t _t0 = star::Time::now();
        const auto &k = *static_cast<const KeyType *>(key);
        bool success = cxl_btree_->remove(k);
        CHECK(success == true);
        uint64_t _ns = star::Time::now() - _t0;
        { std::lock_guard<std::mutex> _g(cxl_btree_pct_mutex); cxl_btree_remove_pct.add(_ns); }
        return success;
}
```

> **改**：点更新当前走 `search()`+SCC 写（已被 `search_pct` 覆盖前半段）。若要 B+ 树级原子改值，仿照上面包装 `lookupForUpdate`（`BTreeOLC_CXL.h:2657`）并新增 `cxl_btree_update_pct`——仍无需新类/文件/宏。

---

## 5. 修改③：程序结束时的数据收集（`core/Coordinator.h`）

在 `scc_manager->print_stats()` 之后（`core/Coordinator.h:371`）追加，从各 `Percentile` 取分位数与均值：

```cpp
                if (scc_manager != nullptr)
                        scc_manager->print_stats();

                // ===== CXL B+tree 增删改查时延收集（每 host 一份，统一用 Percentile） =====
                {
                        std::lock_guard<std::mutex> _g(cxl_btree_pct_mutex);
                        auto _dump = [&](const char *name, star::Percentile<uint64_t> &p) {
                                LOG(INFO) << "[CXL-BTree CRUD] host " << id << " " << name
                                          << " samples=" << p.size()        // 采样计数(~10%)
                                          << " avg_ns=" << p.avg()
                                          << " p50=" << p.nth(50) << " p75=" << p.nth(75)
                                          << " p95=" << p.nth(95) << " p99=" << p.nth(99);
                        };
                        _dump("SEARCH(查)", cxl_btree_search_pct);
                        _dump("SCAN(扫)",   cxl_btree_scan_pct);
                        _dump("INSERT(增)", cxl_btree_insert_pct);
                        _dump("REMOVE(删)", cxl_btree_remove_pct);
                }
```
> - `Coordinator.h` 通过现有 include 链可见这些 extern；如不可见，顶部补 `#include "core/CXLTable.h"`。
> - 此处 worker 已全部 join，`Percentile::nth()` 内部排序（`Percentile.h:103`）安全。
> - 如需 CDF，可对每个 `Percentile` 调 `save_cdf(path)`（`Percentile.h:70`），路径由 `context.cdf_path` 派生。

> **采样说明**：`Percentile::add` 仅在 `warmed_up==true` 时按 ~10% 采样（`Percentile.h:28`），故 `size()`
> 是采样计数；分位数与均值是该 ~10% 样本的统计，与现有 `commit_latency` 等指标口径一致。

---

## 6. 可选：无锁变体（要消除全局锁时）

若担心全局 `mutex` 在高频场景的竞争，可改为**每 worker 线程各一份 `Percentile`**（仍是既有类，无新类/文件）：
- 给既有类 `Executor`（`core/Executor.h:389` 附近）加成员：
  ```cpp
  Percentile<uint64_t> cxl_btree_search_pct, cxl_btree_scan_pct, cxl_btree_insert_pct, cxl_btree_remove_pct;
  ```
- 在 worker 线程可见 `this`（Executor）的 CXL B+ 树调用点用 `Time::now()` 计时并 `this->..._pct.add(ns)`：
  查 `TwoPLPashaExecutor.h:136`（`get_migrated_row`）、扫 `:358`（`target_cxl_table->scan`）、
  增/删在 `process_request()`/`commit()` 的迁移调用点。
- 在既有 `onExit()`（`Executor.h:231`）按现有风格打印各 worker 的 `nth()`（每 worker 一行，天然无锁）。

> 取舍：全局锁方案（§3-§5）**单点打点、覆盖所有调用方、输出一行聚合**，但有短临界区；
> 无锁方案输出每 worker 一行、需在多个调用点插桩。鉴于 CXL B+ 树仅服务迁移/远端数据、且 10% 采样，
> **默认推荐 §3-§5 全局锁方案**。

---

## 7. 线程安全与开销

- **正确性**：全局 `Percentile` 的并发 `add` 由 `cxl_btree_pct_mutex` 保护，无数据竞争。
- **计时**：每次操作 2 次 `star::Time::now()`（各 ≈ `steady_clock::now()`）。
- **锁**：仅在 `warmed_up` 且命中 10% 采样时进入临界区，且 `add` 仅一次 `push_back`，临界区极短。
- **内存**：每个 `Percentile` 仅存采样点（`std::vector`），随运行增长但受 10% 采样约束。
- 始终生效（无编译宏）；预热期 `warmed_up==false` 时 `add` 直接返回（`Percentile.h:28`），开销近零。

---

## 8. 示例输出与验证

退出日志（每 host 4 行）：
```
[CXL-BTree CRUD] host 0 SEARCH(查) samples=12840 avg_ns=315 p50=270 p75=340 p95=560 p99=910
[CXL-BTree CRUD] host 0 SCAN(扫)   samples=312   avg_ns=1850 p50=1600 p75=2100 p95=3900 p99=6100
[CXL-BTree CRUD] host 0 INSERT(增) samples=94    avg_ns=2100 p50=1900 p75=2300 p95=4100 p99=15200
[CXL-BTree CRUD] host 0 REMOVE(删) samples=51    avg_ns=2450 ...
```

**验证：**
1. `cmake .. && make` 通过（extern 符号唯一、无重定义）。
2. 跑 README hello-world（TPC-C / TwoPLPasha）：
   `./scripts/run.sh TPCC TwoPLPasha 8 3 mixed 10 15 1 0 1 Clock OnDemand 200000000 1 WriteThrough None 15 5 GROUP_WAL 20000 0 0`
3. 退出日志出现 4 行 `[CXL-BTree CRUD]`；提高多分区比例时 SEARCH/SCAN 的 `samples` 明显上升。
4. 对照 `PERF_MONITORING_SPEC.md` §5.7 预期；需要分布图时用 `save_cdf` + `scripts/plot`。

---

## 9. 改动清单速查

| 类型 | 文件（均为**既有**） | 位置 | 改动 |
|---|---|---|---|
| 改 | `core/CXLTable.h` | 顶部 | `#include` Time.h / Percentile.h / `<mutex>` |
| 改 | `core/CXLTable.h` | `namespace star` 内 | `extern` 4 个 `Percentile<uint64_t>` + 1 个 `std::mutex` |
| 改 | `core/CXLTable.h` | `:132/147/164/179` | 4 方法 `star::Time` 计时 + 锁内 `Percentile::add`（search 单出口） |
| 改 | `core/Table.cpp` | 末尾 | 定义 4 个 Percentile + mutex |
| 改 | `core/Coordinator.h` | `:371` 后 | 锁内 `LOG` 打印各 Percentile 的 `avg/nth` |
| 改（可选） | `core/Executor.h` + `TwoPLPashaExecutor.h` | `:389` / `:136,358` | 无锁变体：per-worker Percentile 成员 + 调用点打点 + `onExit()` 打印 |

> **无新增文件、无新增类/结构体/枚举、无编译宏**；计时用 `star::Time`，**所有时延变量统一用 `star::Percentile` 收集**。

---

*基于分支 `claude/lucid-euler-SKPMw` 源码快照，逐条可核对。配套总览见 `PERF_MONITORING_SPEC.md`。*
