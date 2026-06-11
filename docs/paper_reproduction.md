# Tigon 论文数据复现指南：命令、流程与全参数语义

> 本文整理复现 OSDI '25 Tigon 论文数据所需的**全部命令、执行流程、每条命令的含义及所有传参的含义**，并给出与论文图表的对应关系。所有结论均带 `file:line` 源码引用（脚本路径相对仓库根目录）。
>
> 官方权威说明见 `README.md`（硬件要求 :36-45，环境搭建 :47-77，参数解释 :162-209，图表对照 :103-160）。

---

## 0. 总览：四阶段流程

```
阶段 I   环境准备（裸机 → VM 集群模拟 CXL pod）
  ./scripts/setup.sh HOST                                        # 主机装工具链/库/SSH
  ./emulation/image/make_vm_img.sh                               # mkosi 构建 Ubuntu 22.04 VM 镜像
  sudo daxctl reconfigure-device --mode=system-ram dax0.0 --force  # 仅 Option A：CXL 内存转 CPU-less NUMA 节点
  sudo ./emulation/start_vms.sh --using-old-img --cxl 0 5 8 0 2  # Option A 真实 CXL（最后一个参数=CXL 的 NUMA 节点号）
  sudo ./emulation/start_vms.sh --using-old-img --cxl 0 5 8 1 1  # Option B 双 socket 远程 NUMA 模拟
  ./scripts/setup.sh VMS 8                                       # 向 8 个 VM 同步并加载 CXL 内核模块

阶段 II  编译与分发
  ./scripts/run.sh COMPILE_SYNC 8                                # 编译 bench_* 并分发到 8 个 VM

阶段 III 跑实验
  ./scripts/push_button.sh results/test1                         # 一键复现全部论文图表（~7.5h）
  # 或分图复现：run_tpcc.sh / run_ycsb.sh / run_hwcc_budget.sh / run_swcc.sh [+ run_misc.sh]
  # 或单点实验：./scripts/run.sh TPCC|YCSB <21~23 个参数>

阶段 IV  解析与出图（push_button.sh 已自动包含）
  ./scripts/parse/parse_*.py results/test1   →  CSV
  ./scripts/plot/plot_*.py  results/test1    →  PDF（论文 Figure 4/5/7/8）
```

**硬件前提**（`README.md:36-45`）：
- **Option A**：单 socket ≥40 核 + 接在第一个 socket 的 CXL 内存，Ubuntu 22.04
- **Option B**（无 CXL 硬件）：双 socket 各 ≥40 核，用远程 NUMA 节点模拟 CXL，Ubuntu 22.04
- 普通开发机只能编译（`cmake -B build && cmake --build build`），无法真跑分布式实验

---

## 1. 阶段 I：环境准备命令详解

### 1.1 `./scripts/setup.sh HOST`（`scripts/setup.sh:24-49`）

主机一次性初始化：
- 工具链：cmake、gcc-12/g++-12、clang-15/clang++-15、lld-15、cargo（:31）
- 库：libboost-all-dev、libjemalloc-dev、libgoogle-glog-dev、libgtest-dev（:34）
- VM 模拟依赖：python3/pip、mkosi（造 VM 镜像）、OVMF（QEMU UEFI 固件）、numactl、pyroute2（:37）
- 解析/绘图：pandas、matplotlib、msttcorefonts（:40-42）
- SSH：生成 RSA 密钥对，配置免密登录（:45-47）

### 1.2 `./emulation/image/make_vm_img.sh`

用 mkosi 生成 VM 根镜像 root.img（Ubuntu 22.04，预置 SSH 密钥、bash 配置，启用 sshd / systemd-networkd / chrony）。

### 1.3 `sudo ./emulation/start_vms.sh <7 个参数>`（`emulation/start_vms.sh:7-53`）

启动 VM 集群模拟 CXL pod。**7 个位置参数**（:12-18）：

| 位置 | 参数 | 含义 |
|---|---|---|
| $1 | `--using-new-img` / `--using-old-img` | 新镜像会先删除 `emulation/vms/` 重建；旧镜像复用现有磁盘（:32-37） |
| $2 | `--cxl` / `--sriov` | 网络模式；复现用 `--cxl`（virtio 网络）；`--sriov` 走 Mellanox 直通（:41-52） |
| $3 | `host_id` | 物理主机编号（单机模拟固定 0） |
| $4 | `num_cpus` | **每个 VM** 的 vCPU 数（论文配置 5） |
| $5 | `num_vms` | VM 数量 = 模拟的 host 数（论文配置 8） |
| $6 | `configure_uncore_freq` | =1 时启用超线程 + 把 socket0 uncore 频率锁 2400MHz（`host_setup/uncore_freq.py`，:22-29）；Option A 传 0，Option B 传 1 |
| $7 | `shmem_dir_numa` | 共享内存（模拟 CXL）绑定的 NUMA 节点号：Option A 传 CXL 内存的节点号（如 2），Option B 传远程 socket 节点号（如 1） |

内部固定配置（:48-52）：每 VM 内存 10240MB；共享内存 `/mnt/cxl_mem` 共 **65536MB（64GB）**（tmpfs 挂载 + `numactl` 绑定到 $7 指定节点，`emulation/vm_lib/ivshmem.py:11-18`）；启动 Rust 版 ivshmem-server（:29-64）+ QEMU ivshmem-doorbell 设备，把同一段共享内存映射进所有 VM——这就是「CXL pod」的模拟基础。

清理命令：`sudo ./emulation/kill_vms.sh`（pkill qemu + 清共享内存）。

### 1.4 `./scripts/setup.sh VMS <N>`（`scripts/setup.sh:50-79`）

对 N 个 VM 逐一执行：
- 同步内核模块三件套（:58-62）：`cxl_init`、`cxl_recover_meta`、`cxl_ivpci.ko`（来自 `dependencies/kernel_module/`）
- 同步运行库（:64-69）：`libjemalloc.so.2`、`libglog.so.0`、`libgflags.so.2.2`
- 加载模块（:71-77）：`rmmod cxl_ivpci`（清旧）→ `insmod ./cxl_ivpci.ko`

**三件套作用**：
- `cxl_ivpci.ko`：把 QEMU ivshmem 共享内存暴露为 CXL 内存设备（虚拟 PCI 驱动）
- `cxl_init --machine-count 16 --size $((2**30*64)) -z`：在 **VM0** 上初始化 64GB CXL 内存全局元数据（`scripts/utilities.sh:50`）
- `cxl_recover_meta --tot_machines 16`：其余 VM 恢复/映射同一份元数据（`utilities.sh:53`）

> 注意：`cxl_init`/`cxl_recover_meta` 不需要手工执行——`run.sh` 每次跑实验前会通过 `init_cxl_for_vms` 自动调用（`utilities.sh:44-55`，见 §3.1）。

VM 访问方式（`utilities.sh:7-14`）：宿主机 `ssh -p $((10022+i)) root@127.0.0.1` 进入第 i 个 VM；VM 间互联网段 `192.168.100.(2+i)`。

---

## 2. 阶段 II：编译与分发

`./scripts/run.sh` 的辅助模式（模式列表 `scripts/run.sh:12`）：

| 命令 | 含义 |
|---|---|
| `./scripts/run.sh COMPILE` | 本机 `cd build && cmake .. && make -j`，产出 `bench_tpcc` / `bench_ycsb` / `bench_smallbank` / `bench_tatp`（run.sh:1070-1083） |
| `./scripts/run.sh COMPILE_SYNC <vm_num>` | 编译 + `sync_binaries` 把二进制 scp 到所有 VM 的 `~/pasha/` 目录（run.sh:1084-1102） |
| `./scripts/run.sh KILL <host_num>` | 杀掉所有 VM 上残留的 bench 进程（run.sh:1059-1069） |
| `./scripts/run.sh COLLECT_OUTPUTS <host_num>` | 收集各 VM 的 output.txt 到本地（run.sh:1115-1125） |

---

## 3. 阶段 III：单点实验 `run.sh` 全参数详解

### 3.1 每次实验的公共前置动作

任一 TPCC/YCSB/... 实验启动前，`run_exp_*` 都会自动执行（`run.sh:131-133` / `:311-313`）：
1. `kill_prev_exps`：杀掉上次残留进程
2. `delete_log_files`：清理旧 WAL 日志
3. `init_cxl_for_vms`：VM0 跑 `cxl_init` 重新初始化 CXL 内存，其余 VM 跑 `cxl_recover_meta`（`utilities.sh:44-55`）

随后：host 1..N-1 以 `nohup ... &> output.txt &` 后台启动，**host 0 前台启动**并实时打印统计（`run.sh:135-161`）；server 列表自动生成为 `192.168.100.2:1234;192.168.100.3:1234;...`（`print_server_string`，`run.sh:80-94`）。

### 3.2 TPCC 模式：`./scripts/run.sh TPCC <21 个参数>`

参数解析见 `run.sh:826-851`（含模式字共 22 个 token）。逐位语义（官方解释 `README.md:179-209`）：

| # | 参数 | 取值与含义 | 映射到的 gflags |
|---|---|---|---|
| 1 | `SYSTEM` | 协议：`TwoPLPasha`（**Tigon 本体**）/ `TwoPLPashaPhantom`（Tigon 关幻读防护，实际传 `--protocol=TwoPLPasha --enable_phantom_detection=false`，run.sh:224）/ `SundialPasha` / `Sundial` / `TwoPL` | `--protocol` |
| 2 | `HOST_NUM` | 参与实验的 host（VM）数 | 决定启动几个进程、`--servers` 列表长度 |
| 3 | `WORKER_NUM` | 每 host 事务 worker 线程数 | `--threads` |
| 4 | `QUERY_TYPE` | `mixed`=TPC-C 全部 5 种事务；`first_two`=只跑 NewOrder+Payment | `--query` |
| 5 | `REMOTE_NEWORDER_PERC` | 远程（跨分区）NewOrder 事务百分比 0-100 | `--neworder_dist` |
| 6 | `REMOTE_PAYMENT_PERC` | 远程 Payment 事务百分比 0-100 | `--payment_dist` |
| 7 | `USE_CXL_TRANS` | 1=消息走 CXL 共享内存环形缓冲，0=走 TCP | `--use_cxl_transport` |
| 8 | `USE_OUTPUT_THREAD` | 1=把输出线程改造为专职发送线程；**要求 #7=1**（README:196） | `--use_output_thread` |
| 9 | `ENABLE_MIGRATION_OPTIMIZATION` | 1=启用 Pasha 数据迁移优化 | `--enable_migration_optimization` |
| 10 | `MIGRATION_POLICY` | 迁出victim选择策略：`Clock` / `LRU` / `FIFO` / `NoMoveOut` | `--migration_policy` |
| 11 | `WHEN_TO_MOVE_OUT` | 何时迁出：`OnDemand`=CXL 满了才迁出；`Reactive`=提交后即发迁出提示 | `--when_to_move_out` |
| 12 | `HW_CC_BUDGET` | 硬件缓存一致（HW-cc）CXL 区大小上限，**单位字节**（论文默认 200000000≈200MB） | `--hw_cc_budget` |
| 13 | `ENABLE_SCC` | 1=启用软件缓存一致 | `--enable_scc` |
| 14 | `SCC_MECH` | SCC 机制：`WriteThrough`（Tigon 默认）/ `WriteThroughNoSharedRead`（禁共享读者）/ `NonTemporal`（始终非时序访问）/ `NoOP`（假设全硬件一致，不做任何事） | `--scc_mechanism` |
| 15 | `PRE_MIGRATE` | 实验前预迁移：`None` / `NonPart`（仅不可分区数据）/ `All` | `--pre_migrate` |
| 16 | `TIME_TO_RUN` | 总运行秒数（**含** warmup） | `--time_to_run` |
| 17 | `TIME_TO_WARMUP` | 预热秒数（不计入统计） | `--time_to_warmup` |
| 18 | `LOGGING_TYPE` | `BLACKHOLE`=不落盘 / `GROUP_WAL`=epoch 组提交 / `WAL`=逐条提交（派生逻辑见下） | 派生出 3 个 flags |
| 19 | `EPOCH_LEN` | 组提交 epoch 长度，仅 `GROUP_WAL` 生效。**单位微秒**（gflag 注释 "in us"，`core/Macros.h:46`；README:207 写 ms 系笔误——示例 20000=20ms，run.sh:869 注释 "SiloR uses 40ms" 对应 40000） | `--wal_group_commit_time` |
| 20 | `MODEL_CXL_SEARCH` | 1=建模「本地操作也总查 CXL 索引」的开销，即关闭 shortcut 指针优化的对照（仅 TwoPLPasha 分支传入，run.sh:196） | `--model_cxl_search_overhead` |
| 21 | `GATHER_OUTPUTS` | 1=实验后收集所有 host 的 output.txt；0=只看 host0（run.sh:267-269） | （脚本行为，非 flag） |

**LOGGING_TYPE 派生逻辑**（`run.sh:862-879`）：

| LOGGING_TYPE | `--lotus_checkpoint` | `--wal_group_commit_time` | `--wal_group_commit_size` |
|---|---|---|---|
| `WAL` | 1（开日志） | 0 | 0 |
| `GROUP_WAL` | 1 | `EPOCH_LEN` | 10 |
| `BLACKHOLE` | 0（关日志） | 0 | 0 |

**其它派生/固定值**：
- `partition_num = HOST_NUM × WORKER_NUM`（TPC-C 每 worker 一个分区=一个 warehouse 组，`run.sh:127`）
- CXL 环形缓冲尺寸按协议自动选（`run.sh:816-823, 853-859`）：Pasha 系（TwoPLPasha/SundialPasha）entry 2048B × 8192 个；基线协议 entry 65536B × 8192 个 → `--cxl_trans_entry_struct_size / --cxl_trans_entry_num`
- 脚本写死、不可经位置参数调整的 flags（`run.sh:139-148`）：`--granule_count=2000`、`--partitioner=hash`、`--persist_latency=0`、`--replica_group=1`、`--lock_manager=0`、`--batch_flush=1`、`--lotus_async_repl=true`、`--batch_size=0`、`--hstore_command_logging=false`、`--log_path=/root/pasha_log`、`--logtostderr=1`
- 基线协议（Sundial/TwoPL）分支不传迁移/SCC 相关 flags（`run.sh:163-261`），这些 flags 仅 Pasha 系有效

**Hello-World 示例逐参数注解**（`README.md:87`）：

```
./scripts/run.sh TPCC TwoPLPasha 8 3 mixed 10 15 1 0 1 Clock OnDemand 200000000 1 WriteThrough None 15 5 GROUP_WAL 20000 0 0
                 │    │          │ │ │     │  │  │ │ │ │     │        │         │ │            │    │  │ │         │     │ │
                 模式 │          │ │ │     │  │  │ │ │ │     │        │         │ │            │    │  │ │         │     │ └ GATHER_OUTPUTS=0 只看host0
                      Tigon协议  │ │ │     │  │  │ │ │ │     │        │         │ │            │    │  │ │         │     └ MODEL_CXL_SEARCH=0 开启shortcut优化
                                 8 hosts   │  │  │ │ │ │     │        │         │ │            │    │  │ │         └ epoch=20000µs=20ms
                                   3 worker/host │ │ │ │     │        │         │ │            │    │  │ └ GROUP_WAL 组提交日志
                                     mixed 全5种事务 │ │     │        │         │ │            │    │  └ warmup 5s
                                           10% 远程NewOrder  │        │         │ │            │    └ 总共跑 15s
                                              15% 远程Payment│        │         │ │            └ 不预迁移
                                                 CXL传输=开  │        │         │ └ SCC机制=WriteThrough
                                                   输出线程=关        │         └ SCC=开
                                                     迁移优化=开      └ HW-cc 预算 200MB
                                                       Clock 迁出策略 + OnDemand 满了才迁出
```

### 3.3 YCSB 模式：`./scripts/run.sh YCSB <23 个参数>`

参数解析见 `run.sh:889-911`（含模式字共 24 个 token）。与 TPCC 的差异：第 4-8 位换成 YCSB 专属参数，其余完全同序：

| # | 参数 | 含义 | gflags |
|---|---|---|---|
| 4 | `QUERY_TYPE` | `rmw`=标准读改写 / `scan`=范围查询 / `custom`=混合插入删除（README:185） | `--query` |
| 5 | `KEYS` | **每 host** 的 KV 对数量（论文常用 300000） | `--keys` |
| 6 | `RW_RATIO` | 读操作百分比（100=纯读，50=半读半写，0=纯写） | `--read_write_ratio` |
| 7 | `ZIPF_THETA` | Zipfian 偏斜因子（0=均匀，论文常用 0.7，高偏斜 0.99） | `--zipf` |
| 8 | `CROSS_RATIO` | 事务内远程（跨分区）操作百分比 0-100 | `--cross_ratio` |
| 9-23 | 同 TPCC 的 7-21 位 | USE_CXL_TRANS … GATHER_OUTPUTS，语义完全相同 | 同上 |

YCSB 特有派生：`partition_num = HOST_NUM`（每 host 一个分区，`run.sh:307`）；跨分区事务固定访问 2 个分区 `--cross_part_num=2`（`run.sh:328`）。

**官方示例**（`README.md:177`）：
```
./scripts/run.sh YCSB TwoPLPasha 8 3 rmw 300000 50 0.7 10 1 0 1 Clock OnDemand 200000000 1 WriteThrough None 30 10 BLACKHOLE 20000 0 0
# = Tigon, 8 host × 3 worker, rmw 负载, 每host 30万key, 50%读, zipf 0.7, 10%跨分区,
#   CXL传输开/输出线程关, 迁移优化开(Clock+OnDemand, 200MB预算), SCC开(WriteThrough), 不预迁移,
#   跑30s(含10s warmup), 不落盘(BLACKHOLE, 此时EPOCH_LEN=20000被忽略), 开shortcut优化, 只看host0
```

### 3.4 SmallBank / TATP 模式

`./scripts/run.sh SmallBank|TATP <21 个参数>`（各含模式字 22 个 token，`run.sh:943-1058`）：结构同 YCSB 但去掉 RW_RATIO（负载混合比例由 benchmark 内置），参数序为 `SYSTEM HOST_NUM WORKER_NUM QUERY_TYPE KEYS ZIPF_THETA CROSS_RATIO USE_CXL_TRANS ...`（后段同 TPCC 7-21 位）。`--keys` 默认值为每分区 1000 万账户（`bench_smallbank.cpp:8`）。

### 3.5 实验输出怎么读

host0 前台每秒打印吞吐/abort 率/迁移频率，结束时打印汇总（`README.md:90-99`）：
- `Coordinator.h:610] Global Stats: total_commit ... total_size_index/metadata/data/transport/misc_usage total_hw_cc_usage total_usage`——**第 8 个字段 total_commit 即吞吐数据来源**（解析脚本依赖此行，见 §6）
- `WALLogger.h:539/545/550]`：组提交延迟 / 排队延迟 / 落盘延迟分位数
- `Database.h:736] TPC-C consistency check passed!`：实验末自动一致性校验

---

## 4. 阶段 III：批量复现脚本 ↔ 论文图表

### 4.1 一键复现：`./scripts/push_button.sh results/test1`（~7.5h）

`scripts/push_button.sh:1-49`，唯一参数 = 结果根目录。内部顺序执行五组「run → parse → plot」：

| 顺序 | 实验脚本 | 实验矩阵 | 解析 | 绘图 | 论文图 | 输出 PDF | 耗时 |
|---|---|---|---|---|---|---|---|
| 1 | `run_tpcc.sh` | 6 个基线（Sundial/TwoPL × NET/CXL/CXL-improved）+ 2 个 Tigon（TwoPLPasha / TwoPLPashaPhantom），各 × 7 档远程比例（0/0 → 60/90，NewOrder%/Payment% 成对递增） | `parse_tpcc.py` | `plot_tpcc_sundial.py` `plot_tpcc_twopl.py` `plot_tpcc.py` | **Fig 4(a)(b)(c)** | `tpcc/tpcc-sundial.pdf` `tpcc/tpcc-twopl.pdf` `tpcc/tpcc.pdf` | ~1h |
| 2 | `run_ycsb.sh` | 3 系统 × 4 档读写比（100/95/50/0，θ=0.7）× 11 档跨分区比例（0→100%） | `parse_ycsb.py` | `plot_ycsb.py` | **Fig 5** | `ycsb/ycsb.pdf` | ~2.5h |
| 3 | `run_hwcc_budget.sh` | Tigon 在 5 档 HW-cc 预算（200/150/100/50/10 MB）下跑 TPC-C 最大远程比 + YCSB 读/写密集（单次 120s 长跑，warmup 60s） | `parse_hwcc_budget.py` | `plot_hwcc_budget.py` | **Fig 7** | `hwcc_budget/hwcc_budget.pdf` | ~3h |
| 4 | `run_swcc.sh` | 4 种 SCC 机制（WriteThrough / WriteThroughNoSharedRead / NonTemporal / NoOP）× TPC-C + YCSB | `parse_swcc.py` | `plot_swcc.py` | **Fig 8** | `swcc/swcc.pdf` | ~1h |
| 5 | `run_misc.sh` | 补充实验：shortcut 优化对照、logging epoch 扫描（BLACKHOLE + GROUP_WAL 1~50ms）、LRU vs Clock 迁移策略、高偏斜 θ=0.99、扩展性 1/2/4/6/8 host（`run_misc.sh:1-192`） | — | — | 其余章节数据 | — | 变长 |

各 run_*.sh 都只接 1 个参数（结果目录），内部经 `scripts/common.sh` 的 `run_remote_txn_overhead_tpcc/ycsb` 包装函数循环调用 §3 的 `run.sh TPCC/YCSB ...`（`common.sh:7-88`），公共配置：8 host × 3 worker、GROUP_WAL epoch 10000µs（10ms）、HW-cc 预算 200MB、单点 30s 跑 + 10s warmup（`run_tpcc.sh:22-37`）。

分图单独复现命令与耗时（`README.md:120-160`）：
```bash
# Figure 4 (~1h)
./scripts/run_tpcc.sh ./results/test1 && ./scripts/parse/parse_tpcc.py ./results/test1
./scripts/plot/plot_tpcc_sundial.py ./results/test1   # Fig 4(a)
./scripts/plot/plot_tpcc_twopl.py   ./results/test1   # Fig 4(b)
./scripts/plot/plot_tpcc.py         ./results/test1   # Fig 4(c)
# Figure 5 (~2.5h)
./scripts/run_ycsb.sh ./results/test1 && ./scripts/parse/parse_ycsb.py ./results/test1 && ./scripts/plot/plot_ycsb.py ./results/test1
# Figure 7 (~3h)
./scripts/run_hwcc_budget.sh ./results/test1 && ./scripts/parse/parse_hwcc_budget.py ./results/test1 && ./scripts/plot/plot_hwcc_budget.py ./results/test1
# Figure 8 (~1h)
./scripts/run_swcc.sh ./results/test1 && ./scripts/parse/parse_swcc.py ./results/test1 && ./scripts/plot/plot_swcc.py ./results/test1
```

> 论文系统名 ↔ 脚本配置对应（`README.md:9-10` + `run_tpcc.sh:42-59`）：
> - **Tigon** = `TwoPLPasha`（CXL 传输 + 迁移 + SCC 全开）
> - **Sundial+ / DS2PL+**（CXL-improved）= `Sundial`/`TwoPL` + `USE_CXL_TRANS=1, USE_OUTPUT_THREAD=0`（3 worker）
> - **Sundial-CXL / DS2PL-CXL** = `USE_CXL_TRANS=1, USE_OUTPUT_THREAD=1`（2 worker，留 1 核给输出线程）
> - **Sundial / DS2PL**（NET）= `USE_CXL_TRANS=0`（走 TCP）
> - **Motor** 基线需 4 台 RDMA 机器，仓库直接提供预测数据 `results/motor/*.csv`（`README.md:31`）

---

## 5. 底层 gflags 完整对照（`core/Macros.h:11-93` + `bench_*.cpp`）

`run.sh` 最终在每个 VM 上执行的真实命令形如（`run.sh:139-148`）：

```
./bench_tpcc --logtostderr=1 --id=<i> --servers="192.168.100.2:1234;..." \
  --threads=W --partition_num=P --granule_count=2000 \
  --log_path=/root/pasha_log --lotus_checkpoint=.. --persist_latency=0 \
  --wal_group_commit_time=.. --wal_group_commit_size=.. \
  --partitioner=hash --hstore_command_logging=false --replica_group=1 --lock_manager=0 \
  --batch_flush=1 --lotus_async_repl=true --batch_size=0 \
  --time_to_run=.. --time_to_warmup=.. \
  --use_cxl_transport=.. --use_output_thread=.. \
  --cxl_trans_entry_struct_size=.. --cxl_trans_entry_num=.. \
  --enable_migration_optimization=.. --migration_policy=.. --when_to_move_out=.. --hw_cc_budget=.. \
  --enable_scc=.. --scc_mechanism=.. [--model_cxl_search_overhead=..] [--enable_phantom_detection=false] \
  --pre_migrate=.. --protocol=.. --query=.. --neworder_dist=.. --payment_dist=..
```

### 5.1 网络与系统

| flag | 默认 | 含义 | run.sh 中 |
|---|---|---|---|
| `--servers` | 127.0.0.1:10010 | 分号分隔的所有 coordinator 地址（Macros.h:11） | 自动生成 |
| `--id` | 0 | 本节点 coordinator ID（:12） | 循环变量 i |
| `--threads` | 1 | worker 线程数（:13） | =WORKER_NUM |
| `--io` | 1 | IO 线程数（:14） | 默认 |
| `--partition_num` | 1 | 全局分区数（:15） | TPCC=H×W，YCSB=H |
| `--partitioner` | hash | 分区器 hash/hash2/pb（:16） | 固定 hash |
| `--granule_count` | 1 | 每分区 granule 数（:71） | 固定 2000 |
| `--cpu_affinity` / `--cpu_core_id` | false / 0 | 线程绑核（:41,44） | 默认 |
| `--tcp_no_delay` / `--tcp_quick_ack` | true / false | TCP 选项（:38-39） | 默认 |

### 5.2 协议与事务

| flag | 默认 | 含义 | run.sh 中 |
|---|---|---|---|
| `--protocol` | Scar | 并发控制协议名，经 `WorkerFactory.h` 编译期分派（:22） | =SYSTEM |
| `--query` | neworder/rmw | 负载查询类型（bench_tpcc.cpp:8 / bench_ycsb.cpp:7） | =QUERY_TYPE |
| `--batch_size` / `--batch_flush` | 100 / 50 | Calvin/Star 批大小 / 批刷新（:18,20） | 固定 0 / 1 |
| `--sleep_on_retry` / `--sleep_time` | true / 100 | abort 重试退避（:17,21） | 默认 |
| `--replica_group` / `--lock_manager` | 1,3 / 1,1 | Calvin 副本组/锁管理器（:23-24） | 固定 1 / 0 |
| `--enable_phantom_detection` | true | TwoPLPasha 幻读防护（next-key locking，:84） | Phantom 变体置 false |

### 5.3 持久化与日志

| flag | 默认 | 含义 | run.sh 中 |
|---|---|---|---|
| `--log_path` | "" | WAL 目录（:37） | 固定 /root/pasha_log |
| `--lotus_checkpoint` | 0 | 0=关日志 1=只日志 2-4=COW checkpoint 组合（:55-67） | 由 LOGGING_TYPE 派生 0/1 |
| `--wal_group_commit_time` | 10 | 组提交 epoch，**微秒**（:46） | =EPOCH_LEN（GROUP_WAL 时） |
| `--wal_group_commit_size` | 7 | 组提交批大小（:47） | 派生 0/10 |
| `--persist_latency` | 110 | 模拟持久化延迟 µs（:45） | 固定 0 |
| `--lotus_async_repl` | false | 异步复制（:54）；注意 replication 实际未实现 | 固定 true |
| `--hstore_command_logging` | true | H-Store 命令日志（:42） | 固定 false |

### 5.4 CXL 传输

| flag | 默认 | 含义 | run.sh 中 |
|---|---|---|---|
| `--use_cxl_transport` | false | 消息走 CXL 共享内存 MPSC 环形缓冲而非 TCP（:74） | =USE_CXL_TRANS |
| `--use_output_thread` | false | 专职输出线程（依赖上者，:75） | =USE_OUTPUT_THREAD |
| `--cxl_trans_entry_struct_size` | 8192 | 环形缓冲单 entry 字节数（:76） | Pasha=2048，基线=65536 |
| `--cxl_trans_entry_num` | 4096 | 每环形缓冲 entry 数（:77） | 固定 8192 |

### 5.5 Pasha 数据迁移与 SCC

| flag | 默认 | 含义 | run.sh 中 |
|---|---|---|---|
| `--enable_migration_optimization` | true | 数据迁移优化总开关（:79） | =参数⑨ |
| `--migration_policy` | Eagerly | Clock/LRU/FIFO/NoMoveOut/Eagerly（:80） | =MIGRATION_POLICY |
| `--when_to_move_out` | Reactive | OnDemand/Reactive（:81） | =WHEN_TO_MOVE_OUT |
| `--hw_cc_budget` | 200MB | HW-cc CXL 区字节上限（:82） | =HW_CC_BUDGET |
| `--enable_scc` | true | 软件缓存一致开关（:87） | =ENABLE_SCC |
| `--scc_mechanism` | NoOP | WriteThrough/WriteThroughNoSharedRead/NonTemporal/NoOP（:88） | =SCC_MECH |
| `--pre_migrate` | None | None/NonPart/All 预迁移（:93） | =PRE_MIGRATE |
| `--model_cxl_search_overhead` | false | 关闭 shortcut 指针优化的对照建模（:85） | =MODEL_CXL_SEARCH |

### 5.6 运行控制与 benchmark 专用

| flag | 默认 | 含义 |
|---|---|---|
| `--time_to_run` / `--time_to_warmup` | 30 / 10 | 总时长（含预热）/ 预热秒数（:90-91） |
| `--neworder_dist` / `--payment_dist` | 10 / 15 | TPC-C 远程 NewOrder/Payment %（bench_tpcc.cpp:9-10） |
| `--keys` | 200000 | YCSB 每分区 key 数（bench_ycsb.cpp:12）；SmallBank/TATP 默认 1000 万（bench_smallbank.cpp:8） |
| `--read_write_ratio` / `--read_only_ratio` | 80 / 0 | YCSB 读比例 / 只读事务比例（bench_ycsb.cpp:9-10） |
| `--cross_ratio` / `--cross_part_num` | 0 / 2 | 跨分区操作% / 跨分区事务涉及分区数（bench_ycsb.cpp:11,14） |
| `--zipf` | 0 | Zipfian 偏斜（bench_ycsb.cpp:13） |
| `--operation_replication` | false | TPC-C 操作级复制（bench_tpcc.cpp:7） |

（其余 Star/Calvin/Aria/Kiva 专属优化 flags 见 `Macros.h:25-53`，复现论文主图时均用默认值。）

---

## 6. 阶段 IV：解析与绘图机制

数据流：

```
VM 上 output.txt（host0 stdout / 各 host 收集）
  │  关键行: "... Coordinator.h:610] Global Stats: total_commit <吞吐> ..."
  ▼
scripts/parse/common.py:get_row()（:9-19）→ 匹配 tokens[3]=="Coordinator.h:610]"，取 tokens[7] 为吞吐
scripts/parse/parse_{tpcc,ycsb,hwcc_budget,swcc}.py → parse_results() 按(远程比例 × 系统)组表
  + append_motor_numbers() 合并 results/motor/{tpcc,ycsb-*}.csv 预存 Motor 基线
  ▼
results/<dir>/{tpcc,ycsb,hwcc_budget,swcc}/*.csv
  ▼
scripts/plot/plot_*.py（matplotlib，pdf.fonttype=42 可编辑文字）
  ▼
论文 Figure 4(a)(b)(c) / 5 / 7 / 8 的 PDF
```

结果目录结构（以 `results/test1` 为例）：每组实验一个子目录（tpcc/ycsb/hwcc_budget/swcc），内含原始日志 `*.txt`、解析后 `*.csv`、最终 `*.pdf`。

---

## 7. 注意事项与常见坑

1. **硬件依赖**：完整复现必须有真实 CXL 1.1 硬件（Option A）或双 socket 机器（Option B）；普通环境只能 `COMPILE`，跑不了分布式实验（CLAUDE.md「构建」节）。
2. **Option A 需先把 CXL 设备转成 NUMA 节点**：`sudo daxctl reconfigure-device --mode=system-ram dax0.0 --force`（README:67），并把节点号传给 `start_vms.sh` 第 7 参。
3. **`USE_OUTPUT_THREAD=1` 必须配 `USE_CXL_TRANS=1`**（README:196；CLAUDE.md 雷区 `USE_OUTPUT_THREAD` 依赖 `USE_CXL_TRANS`）。
4. **EPOCH_LEN 单位是微秒**（gflag 定义 `Macros.h:46`），README:207 的 "(ms)" 是笔误；论文配置 10000=10ms。
5. **多次复现换结果目录**（results/test2、test3 …）避免覆盖（README:105）；push_button 建议 tmux 过夜跑。
6. **每次实验自动重置 CXL 内存**（`init_cxl_for_vms`），无须手工清理；残留进程可用 `./scripts/run.sh KILL <N>` 强杀。
7. **正确性自检**：TPC-C 结束打印 `consistency check passed!`（README:99，`benchmark/tpcc/Database.h` 的 check_consistency）；没看到说明实验异常。
8. **Motor 基线不用跑**：预存于 `results/motor/`；想真跑需 4 台 RDMA 机器（README:31）。
9. **host 数上限**：EBR ≤8 host / SCC 位 ≤16 host / EBR ≤5 worker（CLAUDE.md 雷区）——论文配置 8 host × 3 worker 恰在范围内，勿随意加大。
