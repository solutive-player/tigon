# Tigon 数据迁移替换算法详解（Clock / LRU / FIFO / NoMoveOut / Eagerly）

> 本文逐一剖析 Pasha 数据迁移的五种替换（淘汰）策略的实现细节：数据结构、迁入/迁出/访问更新逻辑、淘汰算法、追踪结构所在内存、锁、触发时机，以及它们之间的关键设计取舍。
>
> 代码位置：`protocol/Pasha/MigrationManager.h`（基类）+ `Policy{Clock,LRU,FIFO,NoMoveOut,Eagerly}.h`（策略）+ `MigrationManagerFactory.h`（工厂）。通过 `--migration_policy` 选择（见 `docs/cli_flags_reference.md` §D）。

---

## 0. 背景：替换算法在 Tigon 里解决什么问题

CXL 1.1 的**硬件缓存一致（HW-cc）区预算有限**（`--hw_cc_budget`，默认 200MB）。被跨 host 访问的热点行会被「迁入」该区以享受硬件一致；当区将满时，必须「迁出」冷行腾空间。**选谁迁出 = 经典的缓存替换问题**。五种策略就是五种 victim 选择算法。

五种策略一句话对比：

| 策略 | 淘汰依据 | 追踪结构 | 访问更新开销 | 何时迁出 |
|---|---|---|---|---|
| **NoMoveOut** | 不淘汰 | 无 | 无 | 永不迁出 |
| **Eagerly** | 全部迁出 | 本地 DRAM list | 无 | 一触发就清空 |
| **FIFO** | 迁入顺序（最早先出） | 本地 DRAM list（全局单链） | 无 | 满了逐个迁出 |
| **Clock** | 第二次机会位（近似 LRU） | 本地 DRAM 链表 + CXL 内 1B 位 | 极小（置 1 字节） | 满了环形扫描 |
| **LRU** | 精确最近最少使用 | **CXL 共享** offset_ptr 链表 | 大（加锁+链表搬移） | 满了从头淘汰 |

---

## 1. 公共基础设施：`MigrationManager` 基类（`MigrationManager.h`）

所有策略继承 `MigrationManager`，共用一套接口与回调。

### 1.1 三个「实际搬数据」的回调（绑定到 Helper，工厂注入）

`MigrationManager.h:30-36, 80-82`，由 `MigrationManagerFactory` 用 `std::bind` 绑到 `TwoPLPashaHelper`/`SundialPashaHelper` 的方法：

| 回调 | 作用 |
|---|---|
| `move_from_partition_to_shared_region(table, key, row, inc_ref_cnt, &meta)` | **真正迁入**：把行从本地分区拷进 CXL 共享区，分配 CXL 内存，返回策略元数据指针。返回 `migration_result` |
| `move_from_shared_region_to_partition(table, key, row)` | **真正迁出**：把行从 CXL 区搬回本地分区，释放（EBR 退休）CXL 内存 |
| `delete_and_update_next_key_info(table, key, is_local, &need_move_out, &meta)` | 删除行 + 更新幻读防护的 next-key 信息 |

> 策略类本身**不碰数据**，只负责「记录哪些行被迁入、决定淘汰谁」；真正的内存搬运交给这三个回调。这是策略与机制分离的设计。

### 1.2 五个虚接口（策略各自实现）

| 接口 | 默认 | 语义 |
|---|---|---|
| `init_migration_policy_metadata(meta, ...)` | 空 | 行迁入时初始化该行的策略元数据（内嵌在 CXL SCC data 的 24 字节里） |
| `access_row(meta, partition_id)` | 空 | **每次访问迁移行时**更新其"热度/recency"（Clock 置位 / LRU 提升） |
| `move_row_in(...)` | 纯虚 | 迁入 + 把行登记进追踪结构 |
| `move_row_out(partition_id)` | 纯虚 | 按策略选 victim 迁出，直到 `usage < budget` |
| `delete_specific_row_and_move_out(...)` | 纯虚 | 删除行并从追踪结构摘除 |

### 1.3 关键常量与数据结构

- `migration_policy_meta_size = 24`（字节，`MigrationManager.h:23`）：**每个迁移行内嵌的策略元数据上限**，存放在该行的 `TwoPLPashaSharedDataSCC.migration_policy_meta`（CXL 共享）。LRU 的 `LRUMeta`（3×offset_ptr=24B）正好占满，故 `PolicyLRU` 构造时 `CHECK(24 >= sizeof(LRUMeta))`（`PolicyLRU.h:165`）。
- `migrated_row_entity`（`MigrationManager.h:47-65`）：描述一个迁移行——`table` / `key[64]` / `local_row`（本地行的 `<metadata*, value*>`）/ `migration_manager_meta`（指向策略元数据）。
- `migration_result`（`:13-17`）：`SUCCESS` / `FAIL_ALREADY_IN_CXL` / `FAIL_OOM`。
- `when_to_move_out`：`OnDemand` / `Reactive`（`:25-28, 38-44`）。

### 1.4 触发时机：迁入/迁出/访问分别在哪被调

| 接口 | 触发点 | 说明 |
|---|---|---|
| `move_row_in` | `TwoPLPashaMessage.h:187`（`data_migration_request_handler`）、`:360/579`（扫描/插入） | 远程访问该行时，**该行所属分区的 master host** 收到 `DATA_MIGRATION_REQUEST` 后迁入 |
| `move_row_out`（**OnDemand**） | `TwoPLPashaMessage.h:204-206, 373-375` | **迁入后立即**在同一 handler 检查预算并迁出（"满了才腾"） |
| `move_row_out`（**Reactive**） | `TwoPLPasha.h:333/540`、`TwoPLPashaExecutor.h:129` | **事务提交后**经 `DATA_MOVEOUT_HINT` 异步迁出 |
| `access_row` | `TwoPLPashaHelper.h:1228` | **任一 host** 访问迁移行时更新热度（关键：跨 host 访问也会更新） |

### 1.5 预算判定（除 NoMoveOut/Eagerly 外通用）

迁出循环的终止条件统一是：
```cpp
if (cxl_memory.get_stats(CXLMemory::TOTAL_HW_CC_USAGE) < hw_cc_budget) { ... return; }
```
即只要 HW-cc 用量降到预算以下就停（`PolicyFIFO.h:65/74`、`PolicyClock.h:181/203`、`PolicyLRU.h:229/246`）。`TOTAL_HW_CC_USAGE` 的记账见 `docs/...`（CXLMemory 在 alloc/free 时 `fetch_add/sub`）。

---

## 2. NoMoveOut —— 永不迁出（`PolicyNoMoveOut.h`）

最简单的"非策略"：

```cpp
migration_result move_row_in(...) {
    return move_from_partition_to_shared_region(table, key, row, inc_ref_cnt, meta);  // 直接迁入，不登记
}
bool move_row_out(uint64_t partition_id) {
    return true;        // ← 什么都不做，永不迁出
}
```
- **无追踪结构**：迁入的行从不登记、从不淘汰。
- **行为**：数据一旦迁入 HW-cc 区就永久驻留，区只增不减。当 CXL 区满，下一次 `move_row_in` 的 `move_from_partition_to_shared_region` 返回 `FAIL_OOM` → 迁移失败 → 事务退回普通远程访问（不迁移）。
- **用途**：① 基线协议（TwoPL/Sundial）配 `NoMoveOut` + `budget=0`，等价于「关闭迁移」；② 预算充裕、无需淘汰的场景。

---

## 3. Eagerly —— 一触发就清空（`PolicyEagerly.h`）

```cpp
std::list<migrated_row_entity> fifo_queue;   // 本地 DRAM
std::mutex queue_mutex;

bool move_row_out(uint64_t partition_id) {           // 不看 budget！
    for (it = fifo_queue.begin(); it != end; ) {
        if (move_from_shared_region_to_partition(it->table, it->key, it->local_row))
            it = fifo_queue.erase(it);                // 能迁就迁，全部清空
        else it++;
    }
    return true;
}
```
- **追踪结构**：本地 DRAM 的 `std::list`（迁入时 push_back，`:49`）。
- **淘汰算法**：`move_row_out` **不检查预算**，遍历整个队列把**所有**能迁出的行一次性搬回（清空 HW-cc 区）。
- `access_row`：空操作（不记热度）。
- **行为**：最激进——每次触发迁出就尽量把迁入的数据全搬回去。HW-cc 占用最低，但迁移流量最大。通常配 `Reactive`（每事务提交后清空）。
- **名字注意**：内部结构体仍叫 `FIFOMeta`（与 FIFO 共用），但迁出逻辑完全不同（FIFO 是逐个、Eagerly 是全清）。

---

## 4. FIFO —— 先进先出（`PolicyFIFO.h`）

```cpp
std::list<migrated_row_entity> fifo_queue;   // 本地 DRAM，全局单链（不分 partition）
uint64_t hw_cc_budget;

bool move_row_out(uint64_t partition_id) {              // partition_id 被忽略
    if (usage < hw_cc_budget) return;                    // 没超预算不动
    for (it = fifo_queue.begin(); it != end; ) {         // 从队头（最早迁入）开始
        if (move_from_shared_region_to_partition(...)) {
            it = fifo_queue.erase(it);
            if (usage < hw_cc_budget) break;             // 降到预算下就停
        } else it++;
    }
}
```
- **追踪结构**：本地 DRAM `std::list`，**全局一条**（所有 partition 混在一起，`move_row_out` 忽略 `partition_id`）。
- **淘汰算法**：超预算时从**队头**（最早迁入的行）逐个迁出，直到降到预算以下。
- `access_row`：空操作——**不考虑访问热度**，纯按迁入先后。
- **行为**：经典 FIFO，实现简单。缺点：可能淘汰仍频繁访问的"老"行（无 recency 感知）。

---

## 5. Clock —— 第二次机会（`PolicyClock.h`）★ Tigon 默认

近似 LRU、开销远小于 LRU，是 Tigon 主实验默认策略。

### 5.1 数据结构

```cpp
struct ClockMeta { uint8_t second_chance = 0; };        // 每行 1 字节，内嵌在 CXL SCC data
struct ClockTrackerNode { migrated_row_entity row_entity; ClockTrackerNode *next, *prev; };  // 本地 DRAM
class ClockTracker {                                     // 每 partition 一个
    ClockTrackerNode *head, *tail, *cursor;              // 双向链表 + 环形游标
    pthread_spinlock_t clock_tracker_lock;               // PROCESS_SHARED 自旋锁
};
ClockTracker *clock_trackers;   // new ClockTracker[partition_num] —— 本地 DRAM（:139）
```
**关键**：链表节点（`ClockTrackerNode`）在**本地 DRAM**（`new`），而 `second_chance` 位（`ClockMeta`）在**每行的 CXL 共享元数据**里。

### 5.2 三段逻辑

- **`access_row`（置位，极廉价）**（`:151-155`）：
  ```cpp
  clock_meta->second_chance = 1;     // 仅写 1 字节，无锁
  ```
  任一 host 访问迁移行 → 把它在 CXL 共享 data 里的 `second_chance` 置 1。
- **`move_row_in`（登记）**（`:157-173`）：迁入成功后 `new ClockTrackerNode`，`track` 到链表尾（加 spinlock）。
- **`move_row_out`（环形扫描淘汰）**（`:175-214`）：
  ```cpp
  if (usage < budget) return;
  while (true) {
      victim = clock_tracker.move_forward_and_get_cursor();  // 游标环形前进（cursor=cursor->next，到尾回到 head）
      if (victim == nullptr) break;
      if (clock_meta->second_chance == 1) { clock_meta->second_chance = 0; continue; }  // 给第二次机会：清位、跳过
      if (move_from_shared_region_to_partition(...)) {       // second_chance==0 → 迁出
          move_forward_and_get_cursor(); untrack(victim);
          if (usage < budget) break;
      }
  }
  ```

### 5.3 算法本质

经典 **Second-Chance / Clock**：游标像时钟指针环形扫描，遇到 `second_chance==1` 的行清位放过（这一轮不淘汰），遇到 `==0` 的行淘汰。一个行被访问后能"逃过一劫"，近似 LRU 但只需读写 1 字节。**Tigon 默认选 Clock = 近似精度 + 极低更新开销**的折中。

---

## 6. LRU —— 精确最近最少使用（`PolicyLRU.h`）

### 6.1 数据结构（关键：链表在 CXL 共享内存）

```cpp
struct LRUMeta {                                         // 24 字节，就是链表节点，内嵌在 CXL SCC data
    migrated_row_entity *row_entity_ptr;                 // 8B（指向本地 DRAM 的 entity）
    boost::interprocess::offset_ptr<LRUMeta> prev, next; // 8B + 8B，位置无关指针
};
class LRUTracker {
    offset_ptr<LRUMeta> head, tail;                      // CXL 内偏移指针
    LRUMeta *cur_victim;
    pthread_spinlock_t lru_tracker_lock;                 // PROCESS_SHARED
};
LRUTracker *lru_trackers;   // 每 partition 一个，分配在 CXL 共享内存
```
构造时（`:167-177`）：**host0** `cxlalloc_malloc_wrapper` 分配 `lru_trackers` 并 `commit_shared_data_initialization`；**其余 host** `wait_and_retrieve_cxl_shared_data` 映射到同一份。→ **整条 LRU 链在 CXL 共享区，所有 host 共享、用 offset_ptr 跨进程寻址**。链表节点 `LRUMeta` 就是每行的 24B CXL 策略元数据本身（prev/next 嵌在里面）。

### 6.2 三段逻辑

- **`access_row`（提升到尾，较重）**（`:188-196`）：
  ```cpp
  lru_tracker.lock();
  lru_tracker.promote(lru_meta);   // = untrack + track 到尾部
  lru_tracker.unlock();
  ```
  每次访问都加 spinlock + 把节点从链中摘下再挂到尾 → **精确 recency**，但开销大；且链在 CXL 共享，**跨 host 访问同一 partition 会争用同一把 spinlock**。
- **`move_row_in`**（`:198-221`）：迁入后 `track` 到尾。
- **`move_row_out`（从头淘汰）**（`:223-257`）：
  ```cpp
  if (usage < budget) return;
  while (true) {
      victim = get_next_victim();   // 从 head（最久未访问）开始
      if (move_from_shared_region_to_partition(...)) {
          untrack(victim); reset_cur_victim(); delete victim->row_entity_ptr;
          if (usage < budget) break;
      }
  }
  ```

### 6.3 算法本质

**精确 LRU**：head=最久未用、tail=最近用；访问即提升到 tail，淘汰从 head 取。比 Clock 精确，但代价：① 每次访问加锁+链表搬移；② 链表必须放 CXL 共享（因为 `access_row` 可能由远程 host 触发，需修改共享链）。

---

## 7. 横向对比

| 维度 | NoMoveOut | Eagerly | FIFO | **Clock** | LRU |
|---|---|---|---|---|---|
| 淘汰精度 | 不淘汰 | 全清 | 迁入顺序 | 近似 LRU | 精确 LRU |
| 追踪结构 | 无 | 本地 list | 本地 list（全局） | 本地链表（每 partition）+ CXL 1B 位 | **CXL 共享链表（每 partition）** |
| 每行策略元数据 | 无 | 8B（`FIFOMeta`） | 8B | 1B（`ClockMeta`） | 24B（`LRUMeta`=链表节点） |
| `access_row` 开销 | 无 | 无 | 无 | **极小**（写 1B，无锁） | **大**（加锁+链表搬移） |
| 是否分 partition | — | 否（全局 list） | 否（全局 list） | 是（per-partition tracker） | 是（per-partition tracker） |
| 迁出粒度 | — | 全部 | 逐个到预算 | 逐个到预算 | 逐个到预算 |
| 锁 | — | `std::mutex` | `std::mutex` | `pthread_spinlock`(PROCESS_SHARED) | `pthread_spinlock`(PROCESS_SHARED) |
| 跨 host recency | — | — | — | ✅ 通过共享 1B 位 | ✅ 通过共享链表 |

---

## 8. 关键设计洞察：Clock vs LRU 的共享内存权衡

这是 Tigon 默认选 Clock 而非 LRU 的核心原因：

- **recency 信息必须跨 host 可见**：迁移行被任意 host 访问（`access_row` 由访问方调用），淘汰决策（在 partition master 上）要能看到所有 host 的访问。
- **LRU 的做法**：把整条链表放 CXL 共享内存（offset_ptr 寻址），任一 host 访问都 `promote`（加锁改共享链）。→ **每次访问一把跨进程自旋锁 + 链表写**，热点 partition 上多 host 争用严重。
- **Clock 的做法**：链表留在本地 DRAM（只有 master 用、淘汰时遍历），跨 host 共享的只是**每行 1 字节的 `second_chance` 位**；访问只需无锁写 1 字节。→ 跨 host 开销最小化。
- **代价**：Clock 是近似 LRU（second-chance），精度略低；但换来访问路径几乎零开销，对高并发跨 host 负载更友好。

> 一句话：**LRU 用「全链入 CXL + 每访问加锁」换精确；Clock 用「每行 1 位入 CXL + 无锁置位」换低开销**。Tigon 选后者。

---

## 9. 与 OnDemand / Reactive 的配合

替换算法（选谁迁出）与"何时迁出"（`--when_to_move_out`）正交：

- **OnDemand**：`move_row_in` 后立即在同一 handler 调 `move_row_out`（`TwoPLPashaMessage.h:204-206`）——只有 HW-cc 区将满才触发淘汰，**惰性、按需**。Tigon 主实验用 `Clock + OnDemand`。
- **Reactive**：事务提交后对涉及的远程 host 发 `DATA_MOVEOUT_HINT`，异步迁出（`TwoPLPasha.h:333`）。
- **Eagerly 策略**通常配 Reactive：每次触发就清空，把 HW-cc 占用压到最低。

---

## 10. 实验中如何选用

- `--migration_policy` 取值：`Clock` / `LRU` / `FIFO` / `NoMoveOut` / `Eagerly`（`Macros.h:80`，经 run.sh 第 ⑩ 位参数）。
- **Tigon 主实验**：`Clock`（`run_tpcc.sh`/`run_ycsb.sh` 的 TwoPLPasha 行）。
- **基线**：`NoMoveOut`（+ budget=0，等价关迁移）。
- **对照实验**：`run_misc.sh:72-73` 用 `LRU` 跑迁移策略对比（LRU vs Clock，验证 Clock 的低开销优势）。
- 工厂分派：`MigrationManagerFactory::create_migration_manager` 按 `protocol`（TwoPLPasha/SundialPasha）+ `migration_policy` 字符串 `new` 对应策略（`MigrationManagerFactory.h:23-115`）。

---

## 11. 坑点与注意事项

1. **NoMoveOut 满了即 OOM 回退**：不淘汰，CXL 满后 `move_row_in` 返回 `FAIL_OOM`，迁移失败、退回远程访问——不是报错，是降级。
2. **FIFO/Eagerly 是全局单链、不分 partition**：`move_row_out(partition_id)` 忽略 `partition_id`，淘汰跨所有 partition 的最早行；Clock/LRU 是 per-partition tracker，只在该 partition 内淘汰。
3. **LRUMeta 必须 ≤ 24B**：`migration_policy_meta_size=24` 是每行 CXL 元数据预算，`LRUMeta`（24B）正好占满，构造时 `CHECK`。新增更大的策略元数据需同步调大该常量。
4. **Clock 的 PROCESS_SHARED 锁略显多余**：`clock_trackers` 在本地 DRAM（`new`），其 `pthread_spinlock` 声明为 `PTHREAD_PROCESS_SHARED`，但链表本身不跨进程（只有 master 用）——这是与 LRU 对称写法的产物，功能上本进程内即可。
5. **Eagerly 不看预算**：`move_row_out` 无视 `hw_cc_budget` 直接清空，配 OnDemand 会导致迁入立刻被清、抖动剧烈，一般配 Reactive。
6. **access_row 仅对有 recency 概念的策略有意义**：FIFO/Eagerly/NoMoveOut 的 `access_row` 是空操作；只有 Clock（置位）/LRU（提升）真正更新热度。
