# 为 MPSCRingBuffer 增加时延统计（send/recv/spin）的设计分析

> 需求：在 `MPSCRingBuffer`（`common/MPSCRingBuffer.h`）中加入
> ```cpp
> Percentile<uint64_t> ring_spin_wait;   // dequeue 中 while 等待 is_ready 的自旋时延
> Percentile<uint64_t> send_lat;         // 入队时延
> Percentile<uint64_t> recv_lat;         // 出队时延
> ```
> 来测量入队/出队/自旋等待时延。**难点**：`MPSCRingBuffer` 位于 CXL 共享内存，被多节点/多进程映射，物理地址与虚拟地址都不一致，只有 offset 偏移一致。本文分析为什么不能直接加、应该怎么实现。

---

## 1. 结论速览

1. **现成的 `Percentile<uint64_t>` 绝对不能**直接作为成员放进 `MPSCRingBuffer`——它内部含 `std::vector`（裸堆指针，指向**进程私有 DRAM**），既非位置无关、其数据也不在 CXL，跨进程访问必然崩溃/UB。
2. 有两条正确路线：
   - **路线一（推荐，方案 A/B）**：这三个时延**本质是 per-process / per-thread 的局部量**，把统计放**本地 DRAM**、按**线程隔离**（仿 `CXL_EBR` 的 `thread_local`）。无需进共享结构。
   - **路线二（按需共享，方案 D）**：若确实想让统计**驻留 CXL、跨进程共享**（例如 host0 不收消息就能读到全 pod 合并分布），则必须**抛弃 `std::vector`，用 `boost::interprocess::offset_ptr` + CXL 固定容量数组 + 原子计数**自己重建一个位置无关的 `CXLPercentile`——这正是 `MPSCRingBuffer::entries_buffer`（`offset_ptr<char>`）的同款手法。
3. 同进程内的「自旋时延、单次 enqueue/dequeue 耗时」是**时长（duration）**，无论本地还是共享方案，其数值跨进程都可比（见 §3 时钟说明）。
4. 若想测**跨节点端到端传输时延**（入队→出队的绝对时间差），需在 **Entry 内嵌 `uint64_t` 时间戳**（位置无关）+ **跨进程同步时钟**（`Time::now()` 的 steady_clock 跨进程不可减，是真正难点，见 §6）。
5. **把它们改成 `static` 可以解决崩溃**——因为 `static` 类成员不在对象/CXL 里、而在进程本地静态区；但 `static` 解决不了线程安全（要配 `thread_local`）、也换不来跨进程共享。推荐 **`static thread_local`**（等价方案 A），详见 §8。

---

## 2. 问题本质：为什么 Percentile 不能进 CXL 结构

### 2.1 MPSCRingBuffer 为什么是「位置无关」的

`MPSCRingBuffer` 整体分配在 CXL 共享内存（`core/Coordinator.h:416-427`）：
```cpp
// host0：在 CXL 分配 coordinator_num 个 ringbuffer，placement-new，发布
cxl_ringbuffers = cxlalloc_malloc_wrapper(sizeof(MPSCRingBuffer) * coordinator_num, ...);
new(&cxl_ringbuffers[i]) MPSCRingBuffer(...);
commit_shared_data_initialization(cxl_transport_root_index, cxl_ringbuffers);
// 其余 host：映射同一份物理 CXL（虚拟地址可能不同）
wait_and_retrieve_cxl_shared_data(cxl_transport_root_index, &tmp);
```
**同一块物理 CXL 内存在每个进程被映射到不同的虚拟基址**。因此结构里**任何指针都必须是 offset_ptr**（相对偏移），不能是裸虚拟地址。现有成员全部满足：
```cpp
uint64_t entry_struct_size, entry_data_size, entry_num;   // POD
std::atomic<uint64_t> head, tail, count;                   // POD 原子
boost::interprocess::offset_ptr<char> entries_buffer;      // 位置无关
struct Entry { uint32_t ...; std::atomic<uint8_t> is_ready; uint8_t data[]; };  // 全 POD + flexible array
```
这是它能跨进程工作的根本原因。

### 2.2 Percentile 的内部结构（`common/Percentile.h`）

```cpp
template <class T> class Percentile {
    Random rand;                       // 含随机数状态（POD，但 seed 用了 (uint64_t)this）
    bool isSorted_ = true;
    std::vector<element_type> data_;   // ★ 内部是 3 个裸指针(_M_start/_M_finish/_M_end)，指向进程私有堆
    element_type sum = 0;
    // add(): data_.push_back(value)  → 向【本进程私有堆】动态扩容
    // nth(): std::sort(data_) ...
};
```

把它放进 CXL 结构会同时踩三个雷：

| 雷点 | 后果 |
|---|---|
| `std::vector` 内部存的是**裸虚拟地址** | host A 写进去的指针，host B 按自己的地址空间解读 = 野指针 |
| vector 指向的**堆内存在进程私有 DRAM**，不在 CXL | 其它进程根本看不到那块内存，即使指针"对"也无数据 |
| `push_back` 会调用 `operator new` 从**本进程堆**扩容 | 多进程并发 add 各自往私有堆分配，写进共享 vector 头 → 互相覆盖、double free |

> 结论：`offset_ptr` 只能解决「指向 CXL 内部的指针」的偏移问题；而 `std::vector` 指向的是**进程私有堆**、且其指针类型是裸指针——**无论如何都不能驻留共享内存**。这对所有 STL 容器（`vector/string/map/list`…）同理。

### 2.3 更关键的：这三个指标本来就不该共享

- `send_lat`：**生产者**测自己 enqueue 的耗时。一个 ringbuffer（如 `cxl_ringbuffers[B]` 是 B 的收件箱）会被**多个生产者进程**（任意 host A→B）并发入队，各自测各自的。
- `recv_lat` / `ring_spin_wait`：**消费者** B（该 ringbuffer 的唯一拥有者）测出队/自旋。

即每个数值都由**某一个进程的某一个线程**产生，天然是局部统计。强行塞进共享结构既错误又无意义——把它们留在**本地 DRAM** 才对。

---

## 3. 测量点与并发分析（先搞清"谁在哪测"）

| 指标 | 测量位置 | 测量者 | 并发情况 |
|---|---|---|---|
| `send_lat` | `send()`（`MPSCRingBuffer.h:155-159`）整个入队（含重试） | 生产者线程 | **多生产者**（MPSC）：同进程内多个 worker/dispatcher 线程可并发入队同一 ringbuffer |
| 入队重试自旋 | `send()` 的 `while(enqueue()!=true)`（队列满 backoff） | 生产者线程 | 同上 |
| `recv_lat` | `recv()`/`dequeue()`（`:101-172`）整个出队 | 消费者线程 | **单消费者**：每 ringbuffer 仅其拥有者 host 的 IncomingDispatcher 出队 |
| `ring_spin_wait` | `dequeue()` 的 `while(entry->is_ready.load()!=1)`（`:121`） | 消费者线程 | 单消费者 |

**两个并发结论**：
1. `Percentile::add` 不是线程安全的（`push_back` + 改 `sum`）。`send_lat` 被同进程多生产者线程并发命中 → **必须按线程隔离**（每线程一份 Percentile），否则数据竞争。
2. 跨进程天然隔离（各进程各自的本地统计），不需要也不能共享。

**时钟问题**：`Time::now()`（`common/Time.h:13-17`）= 相对**本进程** `startTime` 的 steady_clock 纳秒数。同进程内做差有效；**跨进程做差无意义**（startTime 不同 + steady_clock 互相独立）。所以同进程内的「耗时/自旋」可直接测；跨节点端到端时延要特殊处理（见 §6）。

---

## 4. 方案 A（推荐）：thread_local 本地 DRAM 统计，仿 CXL_EBR

完全不动 `MPSCRingBuffer` 的 CXL 内存布局，统计放 `thread_local`（本地 DRAM、自动按线程隔离），与代码库既有的 `CXL_EBR::get_local_ebr_meta()`（`common/CXL_EBR.h:192-196`）一致。

### 4.1 统计结构与访问器

```cpp
// 新增：common/MPSCRingBufferStats.h
namespace star {
struct MPSCRingBufferStats {
    Percentile<uint64_t> ring_spin_wait;   // dequeue 自旋等待 (us)
    Percentile<uint64_t> send_lat;         // 入队耗时 (us)
    Percentile<uint64_t> recv_lat;         // 出队耗时 (us)
};

// 每线程一份，位于本地 DRAM，天然隔离、无锁、无跨进程问题
inline MPSCRingBufferStats &get_local_ringbuffer_stats() {
    static thread_local MPSCRingBufferStats stats;
    return stats;
}
}
```
> 若需区分「不同 ringbuffer」的统计，可改为 `static thread_local std::unordered_map<uint64_t, MPSCRingBufferStats>`，或在 `send/recv` 增加 `ringbuffer_id` 参数索引。通常合并即可。

### 4.2 在 MPSCRingBuffer 里埋点（不增任何 CXL 成员）

```cpp
uint64_t send(char *data, uint64_t data_size) {
    auto t0 = std::chrono::steady_clock::now();              // 本进程 steady_clock
    while (enqueue(data, data_size) != true);                // 含队列满重试自旋
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(
                  std::chrono::steady_clock::now() - t0).count();
    get_local_ringbuffer_stats().send_lat.add(us);           // 写本线程本地统计
    return data_size;
}

uint64_t dequeue(char *data_buffer, uint64_t buffer_size) {
    ...
    /* 自旋等待 is_ready：start/end 各取一次时间，避免每轮取时间 */
    auto s0 = std::chrono::steady_clock::now();
    while (entry->is_ready.load(std::memory_order_acquire) != 1);
    auto spin_us = std::chrono::duration_cast<std::chrono::microseconds>(
                       std::chrono::steady_clock::now() - s0).count();
    get_local_ringbuffer_stats().ring_spin_wait.add(spin_us);
    ...
}

uint64_t recv(char *buffer, uint64_t buffer_size) {
    if (size() == 0) return 0;
    auto t0 = std::chrono::steady_clock::now();
    auto n = dequeue(buffer, buffer_size);
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(
                  std::chrono::steady_clock::now() - t0).count();
    get_local_ringbuffer_stats().recv_lat.add(us);
    return n;
}
```
- 全部用**本进程 steady_clock 做差**，同进程内有效。
- `Percentile::add` 已自带「预热后 + ~10% 抽样」门控（`Percentile.h:28`），内存有界。
- `thread_local` 保证每个生产者/消费者线程各写各的，无数据竞争、无锁。

### 4.3 关停时聚合打印（仿 EBR/WALLogger）

每线程的 `thread_local` 在自己线程内打印，或在 Worker 退出钩子里调用：
```cpp
void print_ringbuffer_stats() {
    auto &s = get_local_ringbuffer_stats();
    LOG(INFO) << "RingBuffer Stats: send_lat "
              << s.send_lat.nth(50) << "/" << s.send_lat.nth(99) << " us (p50/p99)"
              << ", recv_lat " << s.recv_lat.nth(50) << "/" << s.recv_lat.nth(99) << " us"
              << ", spin_wait " << s.ring_spin_wait.nth(50) << "/" << s.ring_spin_wait.nth(99) << " us";
}
```
在 `Worker::onExit()` 或 dispatcher 线程结束处调用（参考 `CXL_EBR::print_statistics` 的调用方式）。解析脚本可像 `parse_all.py` 抓 `WALLogger.h:539]` 那样匹配 `"...RingBuffer Stats:"` 行的固定 token 位。

**方案 A 优点**：零改动 CXL 布局、自动线程隔离、与 EBR 既有模式一致、无锁。**这是首选。**

---

## 5. 方案 B：把统计挂到 per-process 的 CXLTransport（本地 DRAM）

`CXLTransport`（`common/CXLTransport.h`）是**每进程一个**的普通堆对象（`cxl_transport = new CXLTransport(...)`，`Coordinator.h:420/428`），不在 CXL。可让它持有本地统计数组，`send/recv` 时记录：

```cpp
class CXLTransport {
    MPSCRingBuffer *cxl_ringbuffers;                       // 共享(CXL)
    std::vector<MPSCRingBufferStats> stats_per_ringbuffer; // 本地 DRAM，每 ringbuffer 一份
    ...
    void send(Message *m) {
        auto dst = m->get_dest_node_id();
        auto t0 = steady_clock::now();
        cxl_ringbuffers[dst].send(m->get_raw_ptr(), m->get_message_length());
        stats_per_ringbuffer[dst].send_lat.add(elapsed_us(t0));  // 本进程本地
    }
};
```
- 优点：统计与共享结构彻底分离、可按 ringbuffer 维度区分。
- **缺点/注意**：`stats_per_ringbuffer[dst]` 仍会被**同进程多线程**并发写 → 需要 `thread_local` 化或每元素加锁。所以实际仍要结合 §4 的 thread_local（如 `std::vector<thread_local-ish>` 或二维 `[thread][ringbuffer]`）。

> 方案 B 适合「想在 transport 层而非 ringbuffer 层埋点」的场景；隔离手段还是回到 thread_local。

---

## 6. 方案 C：跨节点端到端传输时延（生产者入队→消费者出队）

如果目标是测**一条消息从生产者入队到消费者出队的真实跨节点时延**（不是单侧耗时），那必须把时间戳**随消息走**，并解决跨进程时钟。

### 6.1 在 Entry 内嵌时间戳（位置无关，可放 CXL）

`Entry` 是纯 POD，加一个 `uint64_t` 字段**完全合法**（数字没有地址问题）：
```cpp
struct Entry {
    uint32_t remaining_size;
    uint32_t dequeue_offset;
    std::atomic<uint8_t> is_ready;
    uint64_t enqueue_ts;     // ★ 新增：入队时间戳（共享时钟），位置无关
    uint8_t data[];
};
```
```cpp
// enqueue：写入共享时钟时间戳，并 clwb 刷到 CXL
entry->enqueue_ts = shared_now();
clwb(&entry->enqueue_ts, sizeof(uint64_t));
// dequeue：消费者算端到端时延
get_local_ringbuffer_stats().transport_lat.add((shared_now() - entry->enqueue_ts) / 1000);
```
注意 `entry_data_size = entry_struct_size - 9`（`:29`）是按现有 9 字节头算的；加 8 字节时间戳后要改成 `- 17`（或重新 `offsetof(Entry, data)`），否则 data 区大小算错。

### 6.2 真正的难点：跨进程时钟

`shared_now()` **不能**用现有 `Time::now()`（steady_clock 相对各进程 startTime，跨进程不可比）。可选：

| 方案 | 说明 | 误差/代价 |
|---|---|---|
| `CLOCK_REALTIME`（wall clock） | Tigon 的 VM 已用 `ptp_kvm + chrony` 同步壁钟（`emulation/image/make_vm_img.sh:60-62`） | 受 chrony 同步精度限制（µs~ms 级抖动），亚微秒时延不可信 |
| CXL 共享单调计数器 | 在 CXL 放一个 `std::atomic<uint64_t>` 计数器，各 host 读同一个 | 读 CXL 原子有开销，且只是逻辑序、非真实时间 |
| 复用 `cxl_global_epoch` | 已有的 CXL 全局 epoch（`WALLogger`/EBR 用） | 粒度太粗（epoch 级，10ms），不适合细粒度时延 |

> **强烈提示**：跨节点端到端时延受时钟同步精度支配。若 chrony 同步误差是 50µs，而传输本身只有几 µs，则测出来的数几乎全是时钟噪声。**建议优先用方案 A 测同进程的「入队耗时/出队耗时/自旋等待」这些可靠量；端到端时延仅在时钟同步足够好（如同物理机 ptp）时才有意义，并在文档/输出里标注其精度边界。**

---

## 7. 方案 D：用 `offset_ptr` 重建 CXL 常驻 Percentile（按需共享）

如果需求**就是**要让 `Percentile` 跟着 `MPSCRingBuffer` 一起驻留 CXL、跨进程共享（而非各进程各算），那么唯一正确的做法是：**不要用 `std::vector`，照搬 `entries_buffer` 的 `offset_ptr<char>` 手法，自己写一个位置无关的 `CXLPercentile`**。

### 7.1 为什么 `offset_ptr` 能救「数组」但救不了「`std::vector`」

- `boost::interprocess::offset_ptr<T>` 存的是「相对自身地址的偏移」，解引用时 `this + offset` → 在任何进程的任何映射基址下都指向**同一块 CXL 物理内存**。这正是 `MPSCRingBuffer::entries_buffer`（`:212`）能跨进程工作的原因。
- 但 `std::vector` 内部是**裸指针**且它管理的存储来自**进程私有堆**——你无法让 `std::vector` 改用 offset_ptr，也无法让它从 CXL 分配（除非换 `boost::interprocess` 的分配器，过重）。
- 所以**正解 = 丢掉 `std::vector`，把样本数组换成「`offset_ptr<T>` 指向的 CXL 固定容量数组」+ 原子游标**，结构里只剩位置无关成员。

### 7.2 `CXLPercentile` 骨架（位置无关、可嵌入 MPSCRingBuffer）

```cpp
// common/CXLPercentile.h —— 位置无关版 Percentile，可作为 MPSCRingBuffer 成员
template <class T>
class CXLPercentile {
    public:
        static constexpr uint64_t max_samples = 8192;   // 固定容量：CXL 无法跨进程 realloc

        // host0 在构造 MPSCRingBuffer 时调用：在 CXL 分配样本数组
        void init_in_cxl() {
                data_ = reinterpret_cast<T *>(
                        cxl_memory.cxlalloc_malloc_wrapper(sizeof(T) * max_samples, CXLMemory::MISC_ALLOCATION));
                count_.store(0);
                sum_.store(0);
                pthread_spin_init(&lock_, PTHREAD_PROCESS_SHARED);   // 跨进程自旋锁（仿 Clock/LRU）
        }

        // 多生产者并发安全：原子预留槽位；满了直接丢弃（也可改环形覆盖）
        void add(T value) {
                if (warmed_up == false /* || 抽样门控 */) return;
                uint64_t idx = count_.fetch_add(1, std::memory_order_relaxed);
                if (idx >= max_samples) { count_.store(max_samples); return; }  // 溢出保护
                data_.get()[idx] = value;
                clwb(&data_.get()[idx], sizeof(T));          // 刷到 CXL，保证跨 host 可见
                sum_.fetch_add(value, std::memory_order_relaxed);
        }

        // 读侧（通常只有 ringbuffer 拥有者 host 打印）：快照到本地排序，绝不在 CXL 上原地 sort
        T nth(double n) {
                uint64_t cnt = std::min<uint64_t>(count_.load(std::memory_order_acquire), max_samples);
                if (cnt == 0) return 0;
                clflush(data_.get(), sizeof(T) * cnt);       // 让本 host 读到最新写入
                std::vector<T> local(data_.get(), data_.get() + cnt);   // ← 本地副本（local DRAM）
                std::sort(local.begin(), local.end());
                uint64_t i = static_cast<uint64_t>(ceil(n / 100.0 * cnt)) - 1;
                return local[i];
        }

        double avg() {
                uint64_t cnt = std::min<uint64_t>(count_.load(), max_samples);
                return cnt ? (double)sum_.load() / cnt : 0;
        }

    private:
        boost::interprocess::offset_ptr<T> data_{ nullptr };   // ★ 位置无关，指向 CXL 样本数组
        std::atomic<uint64_t> count_{ 0 };                     // 已写样本数（原子游标）
        std::atomic<uint64_t> sum_{ 0 };
        pthread_spinlock_t lock_;                              // 可选：保护更复杂的更新
};
```

嵌入 `MPSCRingBuffer`，并在构造函数里初始化（host0 路径）：
```cpp
class MPSCRingBuffer {
    ...
    CXLPercentile<uint64_t> ring_spin_wait;   // 全部位置无关，可安全驻留 CXL
    CXLPercentile<uint64_t> send_lat;
    CXLPercentile<uint64_t> recv_lat;

    MPSCRingBuffer(uint64_t entry_struct_size, uint64_t entry_num) : ... {
        entries_buffer = cxl_memory.cxlalloc_malloc_wrapper(...);   // 既有
        ring_spin_wait.init_in_cxl();   // ← 各 Percentile 的样本数组也在 CXL 分配
        send_lat.init_in_cxl();
        recv_lat.init_in_cxl();
        ...
    }
};
```
> 因为 `MPSCRingBuffer` 本身由 host0 `placement-new` 在 CXL 中构造、再 `commit_shared_data_initialization` 发布（`Coordinator.h:416-421`），内部这几个 `offset_ptr data_` 也都是 host0 写入的偏移值，对所有 host 自动有效——和 `entries_buffer` 完全同源。

### 7.3 必须处理的四个要点

| 要点 | 说明 |
|---|---|
| **固定容量** | CXL 不能跨进程 `realloc`，必须预分配 `max_samples`；溢出要么丢弃要么环形覆盖。`std::vector` 的「自动增长」在这里无法实现。 |
| **多生产者并发** | `send_lat` 被多 host、多线程并发 `add` → 用 `count_.fetch_add` 原子预留槽位写入。不能用 `std::vector::push_back`。 |
| **缓存一致（clwb/clflush）** | 写样本后 `clwb`（仿 `enqueue` 对 `entry->data`，`:90`），读前 `clflush`（仿 `dequeue`，`:133`），否则跨 host 读到旧值。这是 CXL 1.1 软件维护一致的固有开销。 |
| **读侧快照排序** | `nth()` 要把 CXL 数据**拷到本地 `std::vector` 再 `std::sort`**——不能在共享数组上原地排序（会与并发写冲突、且改动共享数据）。 |

### 7.4 数值语义：时长可比，绝对时间不可比

- `send_lat/recv_lat/ring_spin_wait` 都是**同进程 steady_clock 的时长差**（duration），把不同 host 的 duration 混进同一个 CXLPercentile 是**有意义的**——µs 时长本身与时钟基准无关，可直接合并成全 pod 分布。
- 但若要存**绝对时间戳**做跨节点端到端时延，仍受 §6 的时钟同步限制——那是另一回事，与 offset_ptr 无关。

### 7.5 方案 A（本地 thread_local）vs 方案 D（offset_ptr 共享），怎么选

| 维度 | 方案 A：本地 thread_local | 方案 D：offset_ptr CXL 常驻 |
|---|---|---|
| 改动 | 不动 CXL 布局，最小 | 改 MPSCRingBuffer 成员 + 写 CXLPercentile |
| 并发开销 | 无锁、无跨核 | 多 host 抢同一原子游标 + clwb/clflush，**CXL 上 cache-line bouncing**（同 LRU 共享链之痛） |
| 容量 | `std::vector` 自动增长 | 固定 `max_samples`，会丢样本 |
| 聚合 | 需各进程各自打印 / 事后汇总 | host0 直接读到全 pod 合并分布，无需 STATISTICS 消息 |
| 正确性风险 | 低 | 高（容量/并发/一致性都要手写对） |
| 适用 | **绝大多数情况（推荐）** | 确需「单一共享统计对象、免消息聚合」时 |

> **建议**：除非明确需要「不发消息就能在 host0 看到全 pod 合并分位」，否则优先方案 A。方案 D 的共享代价（CXL 原子竞争 + 刷缓存 + 固定容量）通常不值得，且会反过来污染你想测的传输路径性能。

---

## 8. 能不能直接把这几个变量改成 `static`？（结论：能解决崩溃，但要看清原因与代价）

这是一个很自然的想法，答案是 **能解决崩溃，但要理解它"为什么能"以及它"换不来什么"**。

### 8.1 关键认知：`static` 类成员根本不在对象里、不在 CXL 里

C++ 的 `static` 数据成员**不属于对象的内存布局**：
- 它不计入 `sizeof(MPSCRingBuffer)`，**不会随 `placement-new` 进入 CXL**；
- 它是**每进程一份**、位于该进程**静态存储区（本地 DRAM）**的全局变量，只是名字写在类作用域里而已。

所以 `static Percentile<uint64_t> send_lat;` 天然就在**本地 DRAM**：内部 `std::vector` 从本进程堆分配、裸指针只在本进程内解引用 → **跨进程地址不一致 / offset_ptr 的问题根本不会发生** ✓。本质上，`static` 把变量从「CXL 对象内」挪到了「进程本地静态区」——和方案 A 殊途同归。

> ⚠️ **最容易误解的点**：`static` 类成员 **≠ 共享内存**。它是「每进程各一份的全局」，不是「多进程共享的同一份」。想让多进程看到同一份统计，只有放 CXL（方案 D 的 `offset_ptr`）。**如果你以为 `static` 能让各节点共享统计，那是错的。**

### 8.2 直接用 `static` 的三个后果

| 后果 | 说明 |
|---|---|
| ① **不跨进程共享** | 每个进程一份副本。要 host0 看到全 pod 合并分布，仍需方案 D（offset_ptr 入 CXL）或事后用 `STATISTICS` 消息聚合（见 `figures_latency_throughput_prints.md` §6）。 |
| ② **进程内所有 ringbuffer 共用一份** | `static` 是 per-class-per-process，本进程 `coordinator_num` 个 ringbuffer 全写同一个 `send_lat` → 丢失「按目标节点区分」的粒度，变成本进程总体 transport 时延。 |
| ③ **仍然不是线程安全的** | `static` 不解决并发！多个生产者线程并发 `add` 同一个 static `Percentile` → `std::vector::push_back` 数据竞争。反而比 thread_local 更糟（全进程线程抢一个）。**必须再加锁。** |

此外：非 inline 的 static 数据成员需要**类外定义**（或 C++17 `inline static`），否则链接报错。

### 8.3 推荐写法：`static thread_local`（一举解决崩溃 + 线程安全）

把它们声明为 **`static thread_local`**，同时拿下「不在 CXL」和「无数据竞争」，且语法上仍写在 `class MPSCRingBuffer` 内、满足"作为成员"的诉求：

```cpp
class MPSCRingBuffer {
    ...
    static thread_local Percentile<uint64_t> ring_spin_wait;
    static thread_local Percentile<uint64_t> send_lat;
    static thread_local Percentile<uint64_t> recv_lat;
};
// 类外定义（每线程各一份，位于线程本地存储）
thread_local Percentile<uint64_t> MPSCRingBuffer::ring_spin_wait;
thread_local Percentile<uint64_t> MPSCRingBuffer::send_lat;
thread_local Percentile<uint64_t> MPSCRingBuffer::recv_lat;
```

- `static thread_local` = **每线程一份、位于线程本地存储（本地 DRAM）**——这正是代码库 `CXL_EBR` 用的同款模式（`common/CXL_EBR.h:194` 的 `static thread_local EBRMetaLocal`）。
- 优点：① 不在 CXL，无跨进程指针问题；② 每线程独立，`add` 无需锁、无竞争；③ 仍写在类里，满足"成员"诉求。
- 仍是 per-process（不跨进程聚合）、且同一线程内所有 ringbuffer 合并。若要**按 ringbuffer 区分**，改成 `static thread_local std::unordered_map<uint64_t, Stats>` 用 ringbuffer id 索引。
- 埋点代码同 §4.2，直接 `send_lat.add(us)` 即写当前线程的实例；打印在各线程退出处（仿 `CXL_EBR::print_statistics`）。

### 8.4 四种"放哪"对比

| 写法 | 在 CXL? | 跨进程共享? | 线程安全? | 区分 ringbuffer? | 评价 |
|---|---|---|---|---|---|
| 普通成员 `Percentile x;` | 是（随对象进 CXL） | — | — | 是（每对象） | ❌ 崩溃（std::vector 裸指针） |
| `static Percentile x;` | 否（进程静态区/本地 DRAM） | 否 | ❌ 需自己加锁 | 否（进程内全合并） | ⚠️ 能跑，但要加锁、粒度粗 |
| **`static thread_local Percentile x;`** | 否（线程本地/本地 DRAM） | 否 | ✅ | 否（线程内合并） | ✅ **推荐（= 方案 A）** |
| `CXLPercentile`（offset_ptr，§7） | 是 | ✅ | 需原子/锁 | 看设计 | 仅当**确需跨进程共享**时（方案 D） |

### 8.5 一句话总结

**`static` 能解决崩溃——因为它把变量从 CXL 对象里挪到了进程本地静态区，根本不参与跨进程映射；但它解决不了线程安全（要配 `thread_local`），也换不来"跨进程共享"（那只能靠方案 D 的 `offset_ptr` 入 CXL）。** 因此推荐 **`static thread_local`**（等价方案 A）；只有当你明确需要"多节点共享同一份统计、host0 免消息直读"时，才用方案 D。

---

## 9. 反模式（明确不要这样做）

1. ❌ `Percentile<uint64_t> send_lat;` 作为 `MPSCRingBuffer` 成员——含 `std::vector`，跨进程必崩（§2）。
2. ❌ 直接用 `offset_ptr` 包**现成的 `Percentile`/`std::vector`**——offset_ptr 只解决「指向 CXL 内部」的偏移；`std::vector` 的存储在进程私有堆、内部又是裸指针，套一层 offset_ptr 也没用。**正确的 offset_ptr 用法是把 `std::vector` 整体换成「`offset_ptr<T>` + CXL 固定数组 + 原子游标」自己重建（见 §7），而不是去包 `std::vector`。**
3. ❌ 在共享结构里存「指向本地统计的裸指针」（如 `MPSCRingBufferStats *local_stats;`）——这是**单个共享对象**，多个进程会把各自的本地地址写进同一字段、互相覆盖；且对其它进程是野指针。
4. ❌ 在共享结构里放 `std::mutex`/`std::atomic<T*>` 指向本地对象——同上，本地地址不跨进程。
5. ❌ 用 `Time::now()` 做跨进程时间戳相减（§3 时钟问题）。

---

## 10. 推荐落地步骤

1. 新增 `common/MPSCRingBufferStats.h`：`MPSCRingBufferStats` 结构 + `get_local_ringbuffer_stats()`（`static thread_local`）。
2. 在 `MPSCRingBuffer::send/recv/dequeue` 用本进程 `steady_clock` 埋点，写入 `get_local_ringbuffer_stats()`（§4.2）。**不给 `MPSCRingBuffer` 增加任何成员**，CXL 布局零变化。
3. 在 dispatcher/worker 线程退出处调用 `print_ringbuffer_stats()`（§4.3），输出固定格式行，便于解析。
4. （可选，谨慎）若需跨节点端到端时延：给 `Entry` 加 `uint64_t enqueue_ts`，同步修正 `entry_data_size`（`-9 → -17`）与 `memset/clwb` 范围，并实现一个**明确标注精度**的 `shared_now()`（优先 CLOCK_REALTIME，注明依赖 chrony/ptp 同步）。
5. （可选，按需共享）若确需统计驻留 CXL、host0 免消息直读全 pod 合并分布：按 §7 写 `CXLPercentile`（`offset_ptr<T>` + CXL 固定数组 + 原子游标 + clwb/clflush），把它作为 `MPSCRingBuffer` 成员，并在构造函数 host0 路径 `init_in_cxl()`；读侧 `nth()` 必须快照到本地再排序。
6. 验证：单机多 VM 跑 `./scripts/run.sh TPCC TwoPLPasha ... 1 ...`（`USE_CXL_TRANS=1`），确认无崩溃、统计行正常输出；对照 `Coordinator.h:307` 的 per-second 输出交叉验证量级。

---

## 11. 一图总结

```
            CXL 共享内存（多进程不同虚拟基址，靠 offset 一致）
   ┌────────────────────────────────────────────────────────────────┐
   │ MPSCRingBuffer  [head/tail/count: atomic][entries: offset_ptr<char>]│  ← 只放位置无关数据
   │ Entry { remaining_size, is_ready, (可选)enqueue_ts:uint64, data[] } │  ← 数字时间戳 OK
   │                                                                  │
   │ 方案D(按需): CXLPercentile {                                      │  ← 用 offset_ptr 重建
   │     offset_ptr<uint64_t> data_;  atomic count_/sum_;  spinlock   │     的位置无关 Percentile
   │ }  ← 固定容量 + 原子游标 + clwb/clflush；nth() 快照到本地排序     │
   └────────────────────────────────────────────────────────────────┘
                         ▲ 入队/出队（steady_clock 同进程做差，时长可比）
   ┌─────────────────────┴──────────────────────────────────────────┐
   │  本地 DRAM（每进程私有，按线程隔离）—— 方案A(推荐)              │
   │  thread_local MPSCRingBufferStats {                             │  ← 现成 Percentile 放这里
   │      Percentile send_lat / recv_lat / ring_spin_wait            │
   │  }   ← 含 std::vector，绝不能进 CXL                             │
   └────────────────────────────────────────────────────────────────┘

  跨节点端到端时延 = 消费者 shared_now() − Entry.enqueue_ts（需同步时钟，精度受限）
  关键区分：std::vector 的 Percentile→本地DRAM(方案A)；想进CXL→须换 offset_ptr 重建(方案D)
```

> 关联阅读：CXL 位置无关分配/映射见 `architecture.md` §2 与 `common/CXLMemory.h`；thread_local 统计先例见 `common/CXL_EBR.h`；Percentile 抽样/分位机制见 `docs/figures_latency_throughput_prints.md` §5.2。
