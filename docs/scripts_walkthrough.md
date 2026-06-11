# Tigon 复现脚本逐行解读

> 本文把复现流程里执行的每个 `sh` 脚本**逐行**拆开解释：每行代码做什么、每个配置参数/变量的含义。是 `docs/paper_reproduction.md`（流程与参数语义）的底层补充。
>
> 涉及脚本（按调用顺序）：
> 1. `scripts/setup.sh` — 环境准备
> 2. `scripts/utilities.sh` — 公共函数库（被 setup/run 复用）
> 3. `emulation/image/make_vm_img.sh` — 构建 VM 镜像
> 4. `emulation/start_vms.sh` + `emulation/host_setup/cxl_global.sh` + `emulation/vm_lib/ivshmem.py` — 启动 VM 集群
> 5. `scripts/run.sh` — 编译/分发/跑单点实验（主调度器）
> 6. `scripts/common.sh` — 批量实验包装函数
> 7. `scripts/run_tpcc.sh`（及同类 run_*.sh）— 成组实验
> 8. `scripts/push_button.sh` — 一键全流程

---

## 0. 所有脚本共有的「文件头」约定

几乎每个 bash 脚本开头都有这几行，含义统一，后文不再重复：

```bash
#! /bin/bash                                                        # shebang：用 bash 解释执行
set -uo pipefail                                                    # u=引用未定义变量即报错；o pipefail=管道中任一命令失败则整条失败
# set -x                                                            # （注释掉的）调试开关：打开会回显每条执行的命令
typeset SCRIPT_DIR=$( cd -- "$( dirname -- "${BASH_SOURCE[0]}" )" &> /dev/null && pwd )  # 脚本自身所在的绝对目录
typeset current_date_time="`date +%Y%m%d%H%M`"                     # 当前时间戳（部分脚本留作命名用，多数没真正使用）
source $SCRIPT_DIR/utilities.sh                                     # 引入公共函数库（见 §2）
```

- `set -u`：防止变量名拼错被当空串静默跑过。
- `set -o pipefail`：例如 `cmd1 | cmd2`，cmd1 挂了也能让脚本感知。
- `SCRIPT_DIR` 用 `BASH_SOURCE[0]` 而非 `$0`，保证无论从哪个目录调用都能定位脚本自身位置——所有相对路径（`../build`、`utilities.sh`）都以它为基准。
- `typeset`：ksh/bash 里等价于 `local`/声明变量（这套脚本沿用了 ksh 风格）。
- 注意 `emulation/` 下的脚本用的是 `set -euo pipefail`（多了 `e`：任一命令返回非 0 立即退出，更严格），并常配 `set -x`（默认开启命令回显，便于排障）。

---

## 1. `scripts/setup.sh` — 主机 / VM 环境准备

```bash
function print_usage {                          # 用法提示函数
        echo "[usage] ./setup.sh [HOST/VMS] EXP-SPECIFIC"
        echo "HOST: None"                        # HOST 模式无额外参数
        echo "VMS: HOST_NUM"                      # VMS 模式需 1 个参数：VM 数
}
if [ $# -lt 1 ]; then                            # $#=参数个数；少于 1 个就报用法并退出
        print_usage
        exit -1
fi
typeset TASK_TYPE=$1                             # 第 1 个参数=任务类型（HOST 或 VMS）
source $SCRIPT_DIR/utilities.sh                  # 引入 ssh_command / sync_files 等公共函数
```

### 1.1 `HOST` 分支（一次性配置开发主机，setup.sh:24-49）

```bash
if [ $TASK_TYPE = "HOST" ]; then
        if [ $# != 1 ]; then print_usage; exit -1; fi          # HOST 模式必须恰好 1 个参数
        # tool chains（编译工具链）
        sudo apt-get install -y cmake gcc-12 g++-12 clang-15 clang++-15 lld-15 cargo
        #   cmake=构建系统; gcc/g++-12=C++ 编译器; clang-15/lld-15=备选编译器/链接器; cargo=Rust(给 ivshmem-server 用)
        # libraries（C++ 依赖库）
        sudo apt-get install -y libboost-all-dev libjemalloc-dev libgoogle-glog-dev libgtest-dev
        #   boost=通用库(含 interprocess offset_ptr); jemalloc=高性能分配器; glog=日志; gtest=单测
        # required by VM-based emulation（VM 模拟所需）
        sudo apt-get install -y python3 python3-pip mkosi ovmf numactl
        #   mkosi=造 VM 镜像; ovmf=QEMU 的 UEFI 固件; numactl=NUMA 内存绑定(模拟 CXL 关键)
        sudo pip3 install pyroute2                              # Python 操作网络/路由(配 VM 网桥); 用 sudo 因 root 也要用
        # required by parsing and plotting（解析与绘图）
        pip3 install pandas matplotlib                          # pandas=读 CSV; matplotlib=画论文图
        sudo apt-get install -y msttcorefonts -qq              # 微软字体(论文图字体一致); -qq=安静模式
        rm ~/.cache/matplotlib -rf                             # 清 matplotlib 字体缓存(让新字体生效)
        # setup ssh key（免密 SSH，宿主机要免密登进各 VM）
        [ -f $HOME/.ssh/id_rsa ] || ssh-keygen -t rsa -N "" -f $HOME/.ssh/id_rsa   # 没有密钥就生成(-N ""=空口令)
        cat $HOME/.ssh/id_rsa.pub >> $HOME/.ssh/authorized_keys                     # 把公钥加入自身授权(VM 镜像会复制它)
        exit 0
```

### 1.2 `VMS` 分支（配置已启动的 VM，setup.sh:50-80）

```bash
elif [ $TASK_TYPE = "VMS" ]; then
        if [ $# != 2 ]; then print_usage; exit -1; fi          # VMS 模式必须 2 个参数
        typeset HOST_NUM=$2                                     # 第 2 参=VM 数量
        # sync kernel module（同步 CXL 内核模块三件套到每个 VM 的 /root/）
        sync_files $SCRIPT_DIR/../dependencies/kernel_module/cxl_init        /root/cxl_init        $HOST_NUM
        sync_files $SCRIPT_DIR/../dependencies/kernel_module/cxl_recover_meta /root/cxl_recover_meta $HOST_NUM
        sync_files $SCRIPT_DIR/../dependencies/kernel_module/cxl_ivpci.ko    /root/cxl_ivpci.ko    $HOST_NUM
        #   cxl_init=初始化 CXL 元数据的用户态程序; cxl_recover_meta=从节点恢复元数据; cxl_ivpci.ko=把共享内存暴露成 CXL 设备的内核模块
        # sync dependencies（同步运行库，VM 里没装 apt 包，靠拷 .so）
        sync_files /lib/x86_64-linux-gnu/libjemalloc.so.2 /lib/x86_64-linux-gnu/libjemalloc.so.2 $HOST_NUM  # 放系统库路径
        sync_files /lib/x86_64-linux-gnu/libjemalloc.so.2 /root/libjemalloc.so.2                 $HOST_NUM  # 再放 /root 一份(LD_PRELOAD 备用)
        sync_files /lib/x86_64-linux-gnu/libglog.so.0     /lib/x86_64-linux-gnu/libglog.so.0     $HOST_NUM
        sync_files /lib/x86_64-linux-gnu/libgflags.so.2.2 /lib/x86_64-linux-gnu/libgflags.so.2.2 $HOST_NUM
        # setup the VM(s)：逐 VM 重新加载内核模块
        for (( i=0; i < $HOST_NUM; ++i )); do
                ssh_command "rmmod cxl_ivpci 2>/dev/null" $i   # 卸载旧模块(忽略报错)
                ssh_command "insmod ./cxl_ivpci.ko 2>/dev/null" $i  # 装新模块
        done
        exit 0
else
        print_usage; exit -1                                   # 任务类型既非 HOST 也非 VMS
fi
```

---

## 2. `scripts/utilities.sh` — 公共函数库

被 setup.sh / run.sh 复用的工具函数。

```bash
function ssh_command {                           # 在某个 VM 上执行一条命令
        typeset command=$1                       # 第 1 参：要执行的命令串
        typeset vm_id=$2                         # 第 2 参：VM 编号
        typeset base_port=10022                  # VM 的 SSH 端口基址
        typeset port=$(expr $base_port + $vm_id) # 第 i 个 VM 的 SSH 端口 = 10022+i
        ssh -o StrictHostKeyChecking=accept-new -o UserKnownHostsFile=/dev/null -o LogLevel=ERROR -p $port root@127.0.0.1 ""$command""
        #   所有 VM 都在宿主机 127.0.0.1，靠端口区分; 关掉 host key 校验/已知主机记录(VM 反复重建)，只报 ERROR
}

function sync_files {                            # scp 一个文件/目录到所有 VM
        typeset src=$1                           # 源路径(宿主机)
        typeset dst=$2                           # 目标路径(VM 内)
        typeset vm_num=$3                        # VM 数量
        typeset base_port=10022
        for (( i=0; i < $vm_num; ++i )); do
                port=$(expr $base_port + $i)
                scp ... -r -P $port $src root@127.0.0.1:$dst > /dev/null   # -r=递归; -P=端口; 静默
        done
}

function setup_hostnames {                       # 给每个 VM 设主机名(192.168.100.X)，本流程未直接调用
        typeset vm_num=$1; typeset base=2
        for (( i=0; i < $vm_num; ++i )); do
                ip=$(expr $base + $i)            # VM i 的 IP 末段 = 2+i
                ssh_command "hostnamectl set-hostname 192.168.100.$ip" $i
        done
}

function init_cxl_for_vms {                      # 【关键】每次实验前重置 CXL 共享内存
        typeset vm_num=$1
        typeset cxl_init=./cxl_init
        ssh_command "$cxl_init --machine-count 16 --size $((2 ** 30 * 64)) -z" 0
        #   仅在 VM0 上跑 cxl_init：--machine-count 16=最多支持 16 台机器的元数据槽; --size=64GB(2^30*64); -z=清零初始化
        for (( i=1; i < $vm_num; ++i )); do
                ssh_command "./cxl_recover_meta --tot_machines 16" $i > /dev/null
                #   其余 VM 跑 cxl_recover_meta 映射/恢复同一份元数据(--tot_machines 须和上面 16 一致)
        done
}

function rm_files_for_vms {                      # 删除所有 VM 上的某文件
        typeset file=$1; typeset vm_num=$2
        for (( i=0; i < $vm_num; ++i )); do
                ssh_command "[ -e $file ] && rm -rf $file" $i
        done
}
```

> 关键认知：**实验拓扑通过约定固化**——VM i 的 SSH 端口=10022+i，内网 IP=192.168.100.(2+i)。`init_cxl_for_vms` 是「VM0 建、其余 VM 取」模式（对应 CLAUDE.md 的 CXL 初始化协议），每次实验前重新执行以清空上次的 CXL 状态。

---

## 3. `emulation/image/make_vm_img.sh` — 构建 VM 镜像

用 mkosi 造一个 Ubuntu 22.04 (jammy) 根镜像 `root.img`。

```bash
set -x; set -euo pipefail                        # 回显命令 + 严格模式
typeset SCRIPT_DIR=...; typeset BUILD_DIR=${SCRIPT_DIR}   # 在脚本所在目录构建
if [ $# != 0 ]; then print_usage; exit -1; fi    # 不接任何参数
typeset mkosi_bin="python3 -m mkosi"             # mkosi 调用方式
typeset mkosi_opts=("-f")                         # -f=强制覆盖已有镜像

enable_systemd_service() {                        # 在镜像里启用某 systemd 服务(做开机自启软链)
        servicename="$1"
        mkdir -p mkosi.extra/etc/systemd/system/multi-user.target.wants
        ln -sf "/usr/lib/systemd/system/${servicename}.service" \
               "mkosi.extra/etc/systemd/system/multi-user.target.wants/${servicename}.service"
}

mkdir -p ${BUILD_DIR}/mkosi.extra                 # mkosi.extra=会原样拷进镜像根目录的覆盖层
mkdir -p ${BUILD_DIR}/mkosi.cache ${BUILD_DIR}/mkosi.builddir   # mkosi 缓存/构建目录
cd ${BUILD_DIR}

# 把宿主机 SSH 密钥塞进镜像 /root/.ssh，使宿主机能免密登 VM
mkdir -p mkosi.extra/root/.ssh
cp -L ~/.ssh/id_rsa.pub mkosi.extra/root/.ssh/authorized_keys   # -L=跟随符号链接拷真实文件
cp ~/.ssh/id_rsa.pub mkosi.extra/root/.ssh
cp ~/.ssh/id_rsa     mkosi.extra/root/.ssh        # 私钥也放进去(VM 间也要互相 ssh)
touch .../root/.ssh/config
echo -e "Host *\n    StrictHostKeyChecking no" >> .../root/.ssh/config   # VM 内 ssh 一律不校验 host key
chmod 400 .../root/.ssh/config                    # SSH 要求 config/key 权限收紧
chmod -R go-rwx mkosi.extra/root                  # 去掉 group/other 权限

source $SCRIPT_DIR/ubuntu_rootfs.sh mkosi.extra   # 引入 rootfs 定制(装包列表等，定义 network_device 等变量)
cp -Lr ~/.bash* ${BUILD_DIR}/mkosi.extra/root/    # 把宿主机 .bashrc 等带进 VM

if [ -f /etc/localtime ]; then ... cp -P /etc/localtime mkosi.extra/etc/; fi   # 同步时区

# 时钟同步(VM 用宿主机 PTP 时钟，保证多 VM 时间一致，影响 epoch/延迟测量)
echo ptp_kvm > .../etc/modules-load.d/ptp_kvm.conf
echo "refclock PHC /dev/ptp0 poll 2" >> .../etc/chrony/chrony.conf

# 写入 Ubuntu jammy 的 apt 源
echo "deb http://us.archive.ubuntu.com/ubuntu/ jammy main ..." > .../etc/apt/sources.list
...(jammy-updates / jammy-backports / jammy-security 三条)

if echo ${network_device} | grep -i "Ethernet Controller E810-C" ...; then
        wget .../iavf-4.8.2.tar.gz -O .../root/iavf-4.8.2.tar.gz   # 特定 Intel 网卡(E810)才下驱动
fi

enable_systemd_service sshd                       # 开机自启 sshd(否则宿主机连不进)
enable_systemd_service systemd-networkd           # 网络
enable_systemd_service systemd-resolved           # DNS
enable_systemd_service chrony                      # 时钟同步

# 开启网桥转发(VM 间二层互通需要)
mkdir -p mkosi.extra/etc/sysctl.d/
tee mkosi.extra/etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

mkosi_ver="$($mkosi_bin --version | awk '/mkosi/{ print $2 }')"   # 取 mkosi 版本号
if (( mkosi_ver >= 9 )); then mkosi_opts+=("--autologin"); fi      # 新版支持自动登录(调试方便)
mkosi_opts+=("build")                             # 子命令 build
sudo -E env "PATH=$PATH" $mkosi_bin ${mkosi_opts[@]}   # 真正构建镜像(-E 保留环境变量)
sudo chmod go+rw root.img                          # 放开镜像权限(QEMU 以普通用户读写)
cd -
```

> 产物：`emulation/image/root.img`，被后续 `start_vms.sh` 当作所有 VM 的（写时复制）磁盘。

---

## 4. `emulation/start_vms.sh` — 启动 VM 集群

```bash
set -euo pipefail; set -x
typeset SCRIPT_DIR=...
if [ $# != 7 ]; then echo "[ERROR] Wrong arguments number. Abort."; exit 0; fi   # 必须 7 个参数

# $1 = --using-new-img / --using-old-img
# $2 = --cxl / --sriov
typeset host_id=$3                  # 物理主机编号(单机模拟固定 0)
typeset num_cpus=$4                 # 每个 VM 的 vCPU 数(论文 5)
typeset num_vms=$5                  # VM 数量 = 模拟的 host 数(论文 8)
typeset configure_uncore_freq=$6   # 是否调 uncore 频率(1=调)
typeset shmem_dir_numa=$7          # 共享内存绑定的 NUMA 节点号

source $SCRIPT_DIR/host_setup/cxl_global.sh        # 引入频率/超线程/NUMA 调优函数(见 §5)
if [ $configure_uncore_freq -eq 1 ]; then          # Option B(NUMA 模拟)才进来
        enable_ht                                  # 开超线程
        sudo modprobe msr                          # 加载 MSR 模块(读写 CPU 寄存器)
        sudo python3 $SCRIPT_DIR/host_setup/uncore_freq.py set --freq 2400 --socket 0   # socket0 uncore 锁 2400MHz
        sudo python3 $SCRIPT_DIR/host_setup/uncore_freq.py set --freq 0    --socket 1   # socket1 不限(0)
        sudo python3 $SCRIPT_DIR/host_setup/uncore_freq.py print
        cd $SCRIPT_DIR
fi
check_comm_conf_no_tune_freq                       # 通用调优：关 NMI 看门狗/ASLR/KSM/NUMA 均衡/透明大页，关超线程+睿频，设 performance 调速器

if [ $1 = "--using-new-img" ]; then                # 新镜像：先删旧 vms 目录
        test -e $SCRIPT_DIR/vms && rm -r $SCRIPT_DIR/vms
else
        echo "starting vm using existing img..."   # 旧镜像：复用现有磁盘
fi
mkdir -p $SCRIPT_DIR/vms                            # 每个 VM 的运行态目录

if [ $2 = "--sriov" ]; then                        # SR-IOV 网络(走 Mellanox 物理网卡直通)
        sudo python3 $SCRIPT_DIR/vm_lib/start_vm.py start_vm --num_vms $num_vms \
        --num_cpus $num_cpus --mem_size_mb 10240 \                          # 每 VM 10GB 内存
        --shmem_dir /mnt/cxl_mem --shmem_size_mb 65536 --vmdir $SCRIPT_DIR/vms \   # 共享内存 64GB 在 /mnt/cxl_mem
        --use_mlnx_ether --num_ether_per_vm 1 --num_ib_per_vm 0 --add_user_ssh --use-ivshmem-doorbell --host_id $host_id \
        --shmem_dir_numa $shmem_dir_numa
else                                               # 默认 --cxl：用 virtio 网络
        sudo python3 $SCRIPT_DIR/vm_lib/start_vm.py start_vm --num_vms $num_vms \
        --num_cpus $num_cpus --mem_size_mb 10240 \
        --shmem_dir /mnt/cxl_mem --shmem_size_mb 65536 --vmdir $SCRIPT_DIR/vms \
        --add_user_ssh --use-ivshmem-doorbell --host_id $host_id \           # --use-ivshmem-doorbell=带中断的共享内存设备
        --shmem_dir_numa $shmem_dir_numa
fi
```

关键固定值：每 VM **10240MB 内存**；共享内存（=模拟的 CXL 内存）**65536MB（64GB）** 挂在 `/mnt/cxl_mem`，并通过 `--shmem_dir_numa` 绑到指定 NUMA 节点；`--use-ivshmem-doorbell` 表示用「带门铃中断」的 ivshmem 设备（支持跨 VM 通知，CXL transport 需要）。

### 4.1 `start_vms.sh` 的 7 个参数速查

| 位置 | 参数 | 含义 | Option A 示例 | Option B 示例 |
|---|---|---|---|---|
| $1 | 镜像模式 | `--using-old-img` 复用 / `--using-new-img` 重建 | --using-old-img | --using-old-img |
| $2 | 网络模式 | `--cxl`(virtio) / `--sriov`(物理网卡直通) | --cxl | --cxl |
| $3 | host_id | 物理主机编号 | 0 | 0 |
| $4 | num_cpus | 每 VM vCPU 数 | 5 | 5 |
| $5 | num_vms | VM 数 = host 数 | 8 | 8 |
| $6 | configure_uncore_freq | 是否锁 uncore 频率 | 0 | 1 |
| $7 | shmem_dir_numa | 共享内存绑定的 NUMA 节点 | CXL 节点号(如 2) | 远程 socket 节点(如 1) |

---

## 5. `emulation/host_setup/cxl_global.sh` — 宿主机调优函数库

只被 `source` 引用，提供一批关掉「测量噪声源」的函数。复现实验里实际被调用的是 `check_comm_conf_no_tune_freq`（start_vms.sh:30），它组合了：

```bash
check_comm_conf_no_tune_freq() {
        disable_nmi_watchdog       # 关 NMI 看门狗(echo 0 > /proc/sys/kernel/nmi_watchdog)：减少周期性中断
        disable_va_aslr            # 关地址随机化(randomize_va_space=0)：结果可复现 + offset_ptr 跨进程一致
        disable_ksm                # 关内核同页合并(ksm/run=0)：避免后台扫描抢 CPU
        disable_numa_balancing     # 关自动 NUMA 均衡(numa_balancing=0)：避免页迁移扰动
        disable_thp                # 关透明大页(transparent_hugepage=never)：避免 khugepaged 抖动
        disable_ht                 # 关超线程(smt/control=off)：减少核间干扰
        disable_turbo              # 关睿频(no_turbo=1 / boost=0)：频率稳定，延迟测量准
        set_performance_mode       # CPU 调速器设 performance：始终跑最高频
}
```

其它函数（`enable_ht`/`configure_cxl_exp_cores`/`check_cxl_conf`/`run_emon_*` 等）用于真实 CXL 硬件实验或 Intel EMON 性能剖析，复现主流程不走，不展开。每个 `disable_*` 本质都是 `echo <值> | sudo tee <某个 /proc 或 /sys 文件>`——直接写内核可调参数。

---

## 6. `emulation/vm_lib/ivshmem.py` — 共享内存服务端（被 start_vm.py 调用）

虽是 Python，但它是「CXL pod」模拟的核心，逐函数说明：

```python
def setup_cxl_host(cxl_host_dir, cxl_host_dir_size_mb, shmem_dir_numa):
    run_local_command(f"sudo mkdir -p {cxl_host_dir}".split())          # 建挂载点(如 /mnt/cxl_mem)
    numa = ",".join(map(str, shmem_dir_numa))                            # NUMA 节点列表转 "1" 或 "1,2"
    if os.path.ismount(cxl_host_dir):                                    # 已挂载就先卸载(清状态)
        run_local_command(f"sudo umount {cxl_host_dir}".split())
    if not os.path.ismount(cxl_host_dir):
        run_local_command(f"sudo mount -t tmpfs -o size={cxl_host_dir_size_mb}M "
                          f"-o mpol=bind:{numa} -o rw,nosuid,nodev tmpfs {cxl_host_dir}".split())
        #  把一块 tmpfs(内存文件系统)挂上来当"CXL 内存"：size 限大小，mpol=bind:NUMA 把这块内存死绑到指定 NUMA 节点
        #  —— 这就是"用远程 NUMA 节点模拟 CXL 延迟"的实现核心
```

```python
def start_ivshmem(vmdir, num_vms, shmem_dir, shmem_size_mb, msi_vectors):
    rust_ivshmem = .../ivshmem/ivshmem-host                              # Rust 版 ivshmem-server 源码目录
    ivshmem_server_path = .../target/release/ivshmem-server
    if not os.path.exists(ivshmem_server_path):
        run_local_shell_command(f"cd {rust_ivshmem} && cargo build --release")   # 没编译就现编(需要 cargo)
    ivshmem_server_sock = os.path.join(vmdir, "ivshmem_sock")            # QEMU 连服务端用的 UNIX socket
    cxl_path = os.path.join(shmem_dir, "mem_1")                          # 真正的共享内存后备文件
    if not os.path.exists(cxl_path):
        with open(cxl_path, "a") as out:
            out.truncate(shmem_size_mb * 1024)                          # 预分配文件大小
    sp.Popen([ ivshmem_server_path,
               "--socket-path", ivshmem_server_sock,                    # QEMU 连接点
               "--memory-path",  cxl_path,                              # 共享内存文件
               "--memory-size", f"{shmem_size_mb}M",                    # 大小(64GB)
               "--vector-count", str(msi_vectors),                      # MSI 中断向量数(门铃中断用)
               "--vm-count",     str(num_vms),                          # 接入的 VM 数
               "--vm-offset" ], ...)                                    # VM 偏移分配
    return ivshmem_server_sock
```

```python
def cleanup_ivshmem_setup(cxl_host_dir):       # kill_vms.sh 间接用：清理
    run_local_command_allow_fail("pkill -9 qemu".split())   # 杀所有 QEMU
    run_local_command_allow_fail("pkill -9 ivshmem".split())# 杀 ivshmem-server
    time.sleep(3)
    if os.path.ismount(cxl_host_dir):
        run_local_command(f"sudo umount {cxl_host_dir}".split())  # 卸载共享内存
```

> 链路：`tmpfs(绑 NUMA) → mem_1 文件 → ivshmem-server → 每个 QEMU 的 ivshmem-doorbell PCI 设备 → VM 内 cxl_ivpci.ko → 暴露成 CXL 内存`。所有 VM 映射同一个 `mem_1`，于是它们「看到同一块带硬件缓存一致性的内存」=CXL pod。

---

## 7. `scripts/run.sh` — 主调度器（编译/分发/跑实验）

文件结构：开头是一批**函数定义**（print_usage / kill_prev_exps / sync_binaries / run_exp_tpcc / run_exp_ycsb / ...），文件末尾才是**参数分发**（按 `$1` 即 RUN_TYPE 选分支）。

### 7.1 顶部辅助函数（run.sh:11-94）

```bash
function kill_prev_exps {                         # 杀掉所有 VM 上残留的 bench 进程
        typeset HOST_NUM=$1
        for (( i=0; i < $HOST_NUM; ++i )); do
                ssh_command "pkill bench_tpcc" $i
                ssh_command "pkill bench_ycsb" $i
                # bench_smallbank / bench_tatp 被注释(默认不编译这俩)
        done
}

function delete_log_files {                       # 清 WAL 日志(函数体大部分被注释，现为空操作)
        typeset MAX_HOST_NUM=8
        typeset LOG_FILE_NAME=pasha_log_non_group_commit.txt
        # 原本会逐 VM 删日志 + drop_caches + sync，现已注释停用
}

function gather_other_output {                    # 把 host1..N-1 的 output.txt 打印出来
        typeset HOST_NUM=$1
        for (( i=1; i < $HOST_NUM; ++i )); do     # 从 1 开始(host0 是前台直接可见)
                ssh_command "cat pasha/output.txt" $i
        done
}

function sync_binaries {                          # 把编译好的二进制 scp 到各 VM
        typeset HOST_NUM=$1
        cd $SCRIPT_DIR
        for (( i=0; i < $HOST_NUM; ++i )); do
                ssh_command "mkdir -p pasha" $i   # VM 内建 ~/pasha 工作目录
        done
        sync_files $SCRIPT_DIR/../build/bench_tpcc /root/pasha/ $HOST_NUM
        sync_files $SCRIPT_DIR/../build/bench_ycsb /root/pasha/ $HOST_NUM
        # smallbank/tatp 注释掉
        exit -1                                   # 注：这里 exit -1 是该函数的固有行为(同步完即整体退出)
}

function print_server_string {                    # 生成 --servers 字符串
        typeset HOST_NUM=$1
        for (( ip=2; ... )); do
                # 拼成 "192.168.100.2:1234;192.168.100.3:1234;..."(IP 从 .2 起，端口 1234)
        done
}
```

### 7.2 `run_exp_tpcc`（run.sh:96-272）—— 真正拉起一次 TPC-C 实验

```bash
function run_exp_tpcc {
        if [ $# != 25 ]; then print_usage; exit -1; fi   # 内部函数收 25 个参数(比 CLI 多了已展开的派生值)
        typeset PROTOCOL=$1 ... GATHER_OUTPUT=${25}        # 25 个位置参数逐一 typeset(见 paper_reproduction.md §3.2)
        typeset PARTITION_NUM=$(expr $HOST_NUM \* $WORKER_NUM)  # 【TPC-C 派生】分区数 = host × worker
        typeset SERVER_STRING=$(print_server_string $HOST_NUM) # 生成 --servers
        kill_prev_exps $HOST_NUM                            # 清残留
        delete_log_files                                   # 清日志(当前空操作)
        init_cxl_for_vms $HOST_NUM                          # 【关键】重置 CXL 共享内存(VM0 init、其余 recover)

        if [ $PROTOCOL = "SundialPasha" ]; then
                for (( i=1; i < $HOST_NUM; ++i )); do      # 先后台启动 host 1..N-1
                        ssh_command "cd pasha; nohup ./bench_tpcc --logtostderr=1 --id=$i --servers=\"...\"
                                --threads=$WORKER_NUM --partition_num=$PARTITION_NUM --granule_count=2000
                                ... &> output.txt < /dev/null &" $i    # nohup+&=后台; 输出重定向到 output.txt
                done
                ssh_command "cd pasha; ./bench_tpcc --logtostderr=1 --id=0 ..." 0   # host0 前台启动(实时打印统计)
        elif [ $PROTOCOL = "Sundial" ]; then ...           # 基线分支：不传 migration/scc/pre_migrate flags
        elif [ $PROTOCOL = "TwoPLPasha" ]; then ...        # Tigon：额外传 --model_cxl_search_overhead
        elif [ $PROTOCOL = "TwoPLPashaPhantom" ]; then ... # 传 --enable_phantom_detection=false，但 --protocol 仍是 TwoPLPasha
        elif [ $PROTOCOL = "TwoPL" ]; then ...             # 基线 2PL
        else echo "Protocol not supported!"; exit -1; fi

        if [ $GATHER_OUTPUT == 1 ]; then gather_other_output $HOST_NUM; fi   # 需要就收集各 host 输出
        kill_prev_exps $HOST_NUM                            # 实验结束再清一遍
}
```

5 个协议分支的命令体几乎一致，差异只在传给 `bench_tpcc` 的 flag 集合（已在 `docs/paper_reproduction.md` §3.2 / §5 详列）。每个 host 的真实命令形如 run.sh:139-148。`run_exp_ycsb`（run.sh:274-808）结构同理，差异：`PARTITION_NUM=$(expr $HOST_NUM)`（每 host 一分区，:307），且传 YCSB 专属 flags（`--keys/--read_write_ratio/--zipf/--cross_ratio/--cross_part_num=2`）。

### 7.3 文件末尾的参数分发（run.sh:810-1129）

```bash
if [ $# -lt 1 ]; then print_usage; exit -1; fi             # 至少 1 个参数(RUN_TYPE)
typeset RUN_TYPE=$1                                         # 第 1 参=模式

# 全局：CXL 环形缓冲尺寸(按协议二选一)
typeset PASHA_CXL_TRANS_ENTRY_STRUCT_SIZE=2048             # Pasha 系：单 entry 2KB
typeset PASHA_CXL_TRANS_ENTRY_NUM=8192                     # entry 数 8192
typeset BASELINE_CXL_TRANS_ENTRY_STRUCT_SIZE=65536        # 基线：单 entry 64KB(无迁移，消息更大)
typeset BASELINE_CXL_TRANS_ENTRY_NUM=8192

if [ $RUN_TYPE = "TPCC" ]; then
        if [ $# != 22 ]; then print_usage; exit -1; fi    # TPCC 须 22 token(模式+21参)
        typeset PROTOCOL=$2 ... GATHER_OUTPUT=${22}        # 解析 21 个 CLI 参数
        # 按协议选环形缓冲尺寸
        if [ $PROTOCOL = "SundialPasha" ] || [ $PROTOCOL = "TwoPLPasha" ]; then
                CXL_TRANS_ENTRY_STRUCT_SIZE=$PASHA_...; CXL_TRANS_ENTRY_NUM=$PASHA_...
        else
                CXL_TRANS_ENTRY_STRUCT_SIZE=$BASELINE_...; ...
        fi
        typeset LOG_PATH="/root/pasha_log"                 # WAL 落盘目录(固定)
        # 【LOGGING_TYPE 派生】把日志类型展开成 3 个 flag(详见 paper_reproduction.md §3.2 表)
        if   [ $LOGGING_TYPE = "WAL" ];       then LOTUS_CHECKPOINT=1; WAL_GROUP_COMMIT_TIME=0;          WAL_GROUP_COMMIT_BATCH_SIZE=0
        elif [ $LOGGING_TYPE = "GROUP_WAL" ]; then LOTUS_CHECKPOINT=1; WAL_GROUP_COMMIT_TIME=$EPOCH_LEN; WAL_GROUP_COMMIT_BATCH_SIZE=10
        elif [ $LOGGING_TYPE = "BLACKHOLE" ]; then LOTUS_CHECKPOINT=0; WAL_GROUP_COMMIT_TIME=0;          WAL_GROUP_COMMIT_BATCH_SIZE=0
        else print_usage; exit -1; fi
        run_exp_tpcc $PROTOCOL ... $GATHER_OUTPUT          # 调上面的函数，传入展开后的 25 个参数
        exit 0
elif [ $RUN_TYPE = "YCSB" ]; then
        if [ $# != 24 ]; then ...; fi                      # YCSB 须 24 token(模式+23参)
        ...(同样解析 + 选缓冲尺寸 + LOGGING_TYPE 派生)... 
        run_exp_ycsb ...; exit 0
elif [ $RUN_TYPE = "SmallBank" ]; then ...                 # 22 token
elif [ $RUN_TYPE = "TATP" ]; then ...                      # 22 token
elif [ $RUN_TYPE = "KILL" ]; then                          # 杀进程
        typeset HOST_NUM=$2; kill_prev_exps $HOST_NUM; exit 0
elif [ $RUN_TYPE = "COMPILE" ]; then                       # 仅编译
        cd $SCRIPT_DIR/../; mkdir -p build; cd build; cmake ..; make -j; exit 0
elif [ $RUN_TYPE = "COMPILE_SYNC" ]; then                  # 编译 + 分发
        typeset HOST_NUM=$2
        cd $SCRIPT_DIR/../; mkdir -p build; cd build; cmake ..; make -j   # 先编译
        sync_binaries $HOST_NUM; exit 0                    # 再 scp 到各 VM
elif [ $RUN_TYPE = "CI" ]; then echo "CI under construction!"; exit -1    # 占位未实现
elif [ $RUN_TYPE = "COLLECT_OUTPUTS" ]; then               # 收集输出
        typeset HOST_NUM=$2; gather_other_output $HOST_NUM; exit 0
else print_usage; exit -1; fi
```

> 一次 `run.sh TPCC ...` 的完整动线：**解析 21 参 → 按协议定环形缓冲 → 按 LOGGING_TYPE 展开日志 flag → run_exp_tpcc(清残留→重置 CXL→后台起 host1..N-1→前台起 host0→可选收集→清残留)**。

---

## 8. `scripts/common.sh` — 批量实验包装函数

提供「固定其它参数、只扫某一维」的封装，供 run_*.sh 调用。核心函数 `run_remote_txn_overhead_tpcc`（扫远程比例）：

```bash
function run_remote_txn_overhead_tpcc {
        if [ $# != 17 ]; then print_usage; exit -1; fi    # 收 17 个参数
        typeset RESULT_DIR=$1 PROTOCOL=$2 HOST_NUM=$3 WORKER_NUM=$4 \
                USE_CXL_TRANS=$5 USE_OUTPUT_THREAD=$6 MIGRATION_POLICY=$7 WHEN_TO_MOVE_OUT=$8 \
                MAX_MIGRATED_ROWS_SIZE=$9 ENABLE_SCC=${10} SCC_MECHANISM=${11} PRE_MIGRATE=${12} \
                LOGGING_TYPE=${13} WAL_GROUP_COMMIT_TIME=${14} MODEL_CXL_SEARCH_OVERHEAD=${15} \
                TIME_TO_RUN=${16} TIME_TO_WARMUP=${17}
        # 结果文件名 = 把所有配置拼进去(便于 parse 脚本按名识别)
        typeset RESULT_FILE=$RESULT_DIR/tpcc-$PROTOCOL-$HOST_NUM-$WORKER_NUM-$USE_CXL_TRANS-...-$MODEL_CXL_SEARCH_OVERHEAD.txt
        mkdir -p $RESULT_DIR
        # 对 7 档远程比例(NewOrder%/Payment%)各跑一次，输出全部追加(>>)到同一个结果文件
        $SCRIPT_DIR/run.sh TPCC $PROTOCOL $HOST_NUM $WORKER_NUM mixed 0  0  $USE_CXL_TRANS ... 0 >> $RESULT_FILE 2>&1
        $SCRIPT_DIR/run.sh TPCC $PROTOCOL $HOST_NUM $WORKER_NUM mixed 10 15 ...              0 >> $RESULT_FILE 2>&1
        $SCRIPT_DIR/run.sh TPCC ... mixed 20 30 ... ; ... mixed 30 45 ...; ... mixed 40 60 ...
        $SCRIPT_DIR/run.sh TPCC ... mixed 50 75 ... ; ... mixed 60 90 ...
        #   注意：第 1 档(0/0)用 PRE_MIGRATE=None；其余档用传入的 $PRE_MIGRATE —— 纯本地负载无需预迁移
}
```

> 关键点：
> - 第 9 参 `MAX_MIGRATED_ROWS_SIZE` 即传给 `run.sh` 的 `HW_CC_BUDGET`（HW-cc 区上限）。
> - 这里把 `run.sh` 的 `QUERY_TYPE` 固定为 `mixed`、`GATHER_OUTPUTS` 固定为 `0`、`ENABLE_MIGRATION_OPTIMIZATION` 固定为 `1`。
> - 7 次调用唯一变化的就是 `REMOTE_NEWORDER_PERC / REMOTE_PAYMENT_PERC`（0/0 → 60/90），构成论文图 4 的 X 轴。
> - `>> $RESULT_FILE 2>&1`：标准输出+错误都追加进同一文件，供后续 `parse_tpcc.py` 解析。

`run_remote_txn_overhead_ycsb`（common.sh:44-88）同理，固定 `KEYS=300000`，扫 11 档 `CROSS_RATIO`（0→100），构成图 5 的 X 轴。`run_tpcc_single` / `run_ycsb_single`（common.sh:90-158）则是「只跑一个点」的版本，供 misc/扩展性实验逐点调用。

---

## 9. `scripts/run_tpcc.sh` — 成组 TPC-C 实验（代表 run_*.sh）

```bash
source $SCRIPT_DIR/common.sh                       # 引入 run_remote_txn_overhead_tpcc
if [ $# != 1 ]; then print_usage; exit -1; fi      # 1 个参数：结果根目录
typeset RESULT_ROOT_DIR=$1

########### General Configuration ###########
typeset HOST_NUM=8                                  # 8 个 host
typeset WORKER_NUM=3                                # Tigon/改进基线：3 worker/host
typeset OLD_WORKER_NUM=2                            # 旧基线(带输出线程)：2 worker(留 1 核给输出线程)
typeset DEFAULT_WAL_GROUP_COMMIT_TIME=10000         # 组提交 epoch=10000µs=10ms
typeset DEFAULT_HCC_SIZE_LIMIT=$(( 1024*1024*200 )) # HW-cc 预算=200MB
typeset TPCC_RUN_TIME=30                            # 单点跑 30s
typeset TPCC_WARMUP_TIME=10                         # 预热 10s

typeset RESULT_DIR=$RESULT_ROOT_DIR/tpcc            # 结果落到 <root>/tpcc/
mkdir -p $RESULT_DIR

########### Baselines（6 条基线配置）###########
# 参数序: RESULT_DIR PROTOCOL HOST WORKER CXL OUTPUT MIG_POLICY WHEN BUDGET SCC SCC_MECH PRE_MIG LOG EPOCH MODEL RUN WARMUP
run_remote_txn_overhead_tpcc $RESULT_DIR TwoPL   8 3 1 0 NoMoveOut OnDemand 0 0 NoOP None GROUP_WAL 10000 0 30 10   # TwoPL-CXL-improved(CXL传输,无输出线程,3worker)
run_remote_txn_overhead_tpcc $RESULT_DIR Sundial 8 3 1 0 NoMoveOut OnDemand 0 0 NoOP None GROUP_WAL 10000 0 30 10   # Sundial-CXL-improved
run_remote_txn_overhead_tpcc $RESULT_DIR TwoPL   8 2 1 1 NoMoveOut OnDemand 0 0 NoOP None GROUP_WAL 10000 0 30 10   # TwoPL-CXL(CXL传输+输出线程,2worker)
run_remote_txn_overhead_tpcc $RESULT_DIR Sundial 8 2 1 1 NoMoveOut OnDemand 0 0 NoOP None GROUP_WAL 10000 0 30 10   # Sundial-CXL
run_remote_txn_overhead_tpcc $RESULT_DIR TwoPL   8 2 0 1 NoMoveOut OnDemand 0 0 NoOP None GROUP_WAL 10000 0 30 10   # TwoPL-NET(走TCP)
run_remote_txn_overhead_tpcc $RESULT_DIR Sundial 8 2 0 1 NoMoveOut OnDemand 0 0 NoOP None GROUP_WAL 10000 0 30 10   # Sundial-NET

########### Tigon（2 条）###########
run_remote_txn_overhead_tpcc $RESULT_DIR TwoPLPasha        8 3 1 0 Clock OnDemand 209715200 1 WriteThrough NonPart GROUP_WAL 10000 0 30 10   # Tigon(迁移Clock+SCC WriteThrough)
run_remote_txn_overhead_tpcc $RESULT_DIR TwoPLPashaPhantom 8 3 1 0 Clock OnDemand 209715200 1 WriteThrough NonPart GROUP_WAL 10000 0 30 10   # Tigon 关幻读防护
```

逐配置含义对照（基线 vs Tigon 的开关差异）：

| 配置 | 协议 | worker | CXL | 输出线程 | 迁移策略 | SCC | 含义 |
|---|---|---|---|---|---|---|---|
| TwoPL/Sundial-CXL-improved | TwoPL/Sundial | 3 | 1 | 0 | NoMoveOut | NoOP | CXL 传输但无迁移/SCC，3 worker |
| TwoPL/Sundial-CXL | 同上 | 2 | 1 | 1 | NoMoveOut | NoOP | 多占 1 核做输出线程 |
| TwoPL/Sundial-NET | 同上 | 2 | 0 | 1 | NoMoveOut | NoOP | 退回 TCP 网络 |
| **Tigon** | TwoPLPasha | 3 | 1 | 0 | **Clock** | **WriteThrough** | 全功能：迁移+软件一致 |

> `NoMoveOut`+预算 0+`NoOP`：基线不做数据迁移、不做软件缓存一致——这正是用同一套代码跑出「无 Pasha 优化」对照组的方法。

`run_ycsb.sh` / `run_hwcc_budget.sh` / `run_swcc.sh` 结构完全相同：顶部设公共配置 → 调 `run_remote_txn_overhead_*`，区别只在扫的维度（YCSB 扫读写比、hwcc 扫预算、swcc 扫 SCC 机制）。

---

## 10. `scripts/push_button.sh` — 一键全流程

```bash
set -uo pipefail
if [ $# != 1 ]; then print_usage; exit -1; fi      # 1 个参数：结果根目录
typeset RESULT_ROOT_DIR=$1

# TPC-C：跑实验 → 解析 → 出 3 张图
echo "Running TPC-C experiments..."
./scripts/run_tpcc.sh           $RESULT_ROOT_DIR   # 跑全部 TPC-C 配置(§9)
./scripts/parse/parse_tpcc.py   $RESULT_ROOT_DIR   # 解析 .txt → .csv
./scripts/plot/plot_tpcc_sundial.py $RESULT_ROOT_DIR   # Fig 4(a)
./scripts/plot/plot_tpcc_twopl.py   $RESULT_ROOT_DIR   # Fig 4(b)
./scripts/plot/plot_tpcc.py         $RESULT_ROOT_DIR   # Fig 4(c)

# YCSB：→ Fig 5
echo "Running YCSB experiments..."
./scripts/run_ycsb.sh         $RESULT_ROOT_DIR
./scripts/parse/parse_ycsb.py $RESULT_ROOT_DIR
./scripts/plot/plot_ycsb.py   $RESULT_ROOT_DIR

# HW-cc 预算敏感性：→ Fig 7
echo "Running HWcc Budget experiments..."
./scripts/run_hwcc_budget.sh         $RESULT_ROOT_DIR
./scripts/parse/parse_hwcc_budget.py $RESULT_ROOT_DIR
./scripts/plot/plot_hwcc_budget.py   $RESULT_ROOT_DIR

# 软件缓存一致机制：→ Fig 8
echo "Running SWcc experiments..."
./scripts/run_swcc.sh         $RESULT_ROOT_DIR
./scripts/parse/parse_swcc.py $RESULT_ROOT_DIR
./scripts/plot/plot_swcc.py   $RESULT_ROOT_DIR

# 杂项补充实验(无对应单图，供论文其它章节)
echo "Running Misc experiments"
./scripts/run_misc.sh $RESULT_ROOT_DIR
```

每组都是固定的「`run_*.sh` 跑 → `parse_*.py` 解析 → `plot_*.py` 出图」三段式，串行执行，总计约 7.5 小时。

---

## 11. 串起来：一次完整复现的脚本调用树

```
push_button.sh results/test1
├── run_tpcc.sh results/test1
│   └── common.sh: run_remote_txn_overhead_tpcc(×8 配置)
│       └── run.sh TPCC ...(×7 远程比例/配置)
│           ├── (分发段) 解析21参 → 选环形缓冲尺寸 → LOGGING_TYPE 展开
│           └── run_exp_tpcc()
│               ├── utilities.sh: kill_prev_exps / init_cxl_for_vms(VM0 cxl_init + 其余 cxl_recover_meta)
│               ├── utilities.sh: ssh_command 后台起 host1..7 + 前台起 host0  → bench_tpcc
│               └── (可选) gather_other_output → 清残留
├── parse/parse_tpcc.py → tpcc/*.csv     (抓 "Coordinator.h:610]" 行的吞吐)
├── plot/plot_tpcc*.py  → tpcc/*.pdf     (Fig 4 a/b/c)
├── run_ycsb.sh → ... → Fig 5
├── run_hwcc_budget.sh → ... → Fig 7
├── run_swcc.sh → ... → Fig 8
└── run_misc.sh → 其它实验

（前置一次性：setup.sh HOST → make_vm_img.sh → start_vms.sh[→cxl_global.sh 调优 + ivshmem.py 起共享内存] → setup.sh VMS 8 → run.sh COMPILE_SYNC 8）
```
