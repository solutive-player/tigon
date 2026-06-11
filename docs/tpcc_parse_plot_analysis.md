# parse_tpcc.py 与三张 TPC-C 绘图脚本：原理与源码逐行分析

> 覆盖 Figure 4 生成链路的全部脚本：
> - `scripts/parse/parse_tpcc.py` — 把实验 `.txt` 解析成两个 CSV
> - `scripts/plot/plot_tpcc_sundial.py` — Fig 4(a)：Sundial 三变体
> - `scripts/plot/plot_tpcc_twopl.py` — Fig 4(b)：DS2PL(TwoPL) 三变体
> - `scripts/plot/plot_tpcc.py` — Fig 4(c)：Tigon vs Sundial+/DS2PL+/Motor
>
> 数据来源 = `run_tpcc.sh` 跑出的 8 个 `.txt`（见 `docs/experiment_scripts_params.md` §1）。解析引擎 `get_row/parse_results/append_motor_numbers` 定义在 `scripts/parse/common.py`（见 `docs/experiment_scripts_params.md` §4.5）。

---

## 0. 三张图的分工与数据流总览

`run_tpcc.sh` 跑出的 8 个原始结果文件里，论文 Fig 4 用到其中 7 个（TwoPLPashaPhantom 那个不画进主图）：

```
run_tpcc.sh → results/test1/tpcc/*.txt (8 个，每个含 7 档远程比例的 Global Stats 行)
        │
        ▼  parse_tpcc.py
   ┌────────────────────┬─────────────────────────────┐
   │ baseline-tpcc.csv  │ tpcc.csv (含 Motor 列)        │
   │ (6 基线列)          │ (Tigon + 2改进基线 + Motor)  │
   └─────────┬──────────┴──────────────┬──────────────┘
             │                          │
   ┌─────────┴────────┐                 │
   ▼                  ▼                 ▼
plot_tpcc_sundial  plot_tpcc_twopl   plot_tpcc.py
  .py(只取3条       .py(只取3条        (取 Tigon/Sundial+/
  Sundial 列)       TwoPL 列)          DS2PL+/Motor 4条)
   ▼                  ▼                 ▼
tpcc-sundial.pdf   tpcc-twopl.pdf    tpcc.pdf
  = Fig 4(a)        = Fig 4(b)        = Fig 4(c)
```

关键设计：**parse 一次产出两个 CSV**——`baseline-tpcc.csv`（6 条基线，供 a/b 两图共享）和 `tpcc.csv`（主图三系统+Motor，供 c 图）。三张 plot 脚本各取所需列。

---

## 1. `scripts/parse/parse_tpcc.py` 逐行分析

### 1.1 导入与解析引擎（parse_tpcc.py:1-9）

```python
import sys, csv, fileinput, pandas as pd, os
from common import get_row, parse_results, append_motor_numbers
```
- `get_row`：读一个 `.txt`，抓出所有吞吐值，返回 `[标签, t0, t1, …]`。
- `parse_results`：把多个系统的行拼成表并**转置**（系统→列、远程比例→行），写 CSV。
- `append_motor_numbers`：把预存的 Motor 基线 CSV 的 `Motor` 列拼到右边。
- 三者的实现见 `common.py`（本文 §5 回顾）。

### 1.2 基线 CSV：`construct_input_list_tpcc_baseline`（parse_tpcc.py:12-20）

```python
def construct_input_list_tpcc_baseline(tpcc_res_dir):
        input_file_list = list()
        input_file_list.append(("Sundial-CXL-improved", tpcc_res_dir + "/tpcc-Sundial-8-3-1-0-NoMoveOut-OnDemand-0-0-NoOP-None-GROUP_WAL-10000-0.txt"))
        input_file_list.append(("TwoPL-CXL-improved",   tpcc_res_dir + "/tpcc-TwoPL-8-3-1-0-NoMoveOut-OnDemand-0-0-NoOP-None-GROUP_WAL-10000-0.txt"))
        input_file_list.append(("Sundial-CXL",          tpcc_res_dir + "/tpcc-Sundial-8-2-1-1-NoMoveOut-OnDemand-0-0-NoOP-None-GROUP_WAL-10000-0.txt"))
        input_file_list.append(("TwoPL-CXL",            tpcc_res_dir + "/tpcc-TwoPL-8-2-1-1-NoMoveOut-OnDemand-0-0-NoOP-None-GROUP_WAL-10000-0.txt"))
        input_file_list.append(("Sundial-NET",          tpcc_res_dir + "/tpcc-Sundial-8-2-0-1-NoMoveOut-OnDemand-0-0-NoOP-None-GROUP_WAL-10000-0.txt"))
        input_file_list.append(("TwoPL-NET",            tpcc_res_dir + "/tpcc-TwoPL-8-2-0-1-NoMoveOut-OnDemand-0-0-NoOP-None-GROUP_WAL-10000-0.txt"))
        return input_file_list
```

返回 6 个 `(图例标签, 文件路径)` 元组。文件名严格遵循 `common.sh` 的命名契约
`tpcc-<协议>-<host>-<worker>-<CXL>-<输出>-<迁移>-<迁出>-<预算>-<SCC>-<机制>-<预迁移>-<日志>-<epoch>-<model>.txt`。把 6 个文件名拆开对照 `run_tpcc.sh:44-49` 的 6 条基线调用：

| 标签 | 协议 | worker | CXL | 输出线程 | 对应 run_tpcc.sh 行 |
|---|---|---|---|---|---|
| Sundial-CXL-improved | Sundial | **8-3** | **1** | **0** | :45 |
| TwoPL-CXL-improved | TwoPL | 8-3 | 1 | 0 | :44 |
| Sundial-CXL | Sundial | **8-2** | 1 | **1** | :47 |
| TwoPL-CXL | TwoPL | 8-2 | 1 | 1 | :46 |
| Sundial-NET | Sundial | 8-2 | **0** | 1 | :49 |
| TwoPL-NET | TwoPL | 8-2 | 0 | 1 | :48 |

文件名里 `NoMoveOut-OnDemand-0-0-NoOP-None`（迁移关、预算0、SCC关、不预迁移）对所有基线一致——这是「关掉全部 Pasha 优化」的标志，所以解析端能用同一套后缀匹配。

```python
def parse_tpcc_baseline(tpcc_res_dir):
        input_file_list = construct_input_list_tpcc_baseline(tpcc_res_dir)
        output_file_name = tpcc_res_dir + "/baseline-tpcc.csv"          # 输出文件
        header_row = ["Remote_Ratio", "0/0", "10/15", "20/30", "30/45", "40/60", "50/75", "60/90"]  # 7 档远程比例
        parse_results(input_file_list, output_file_name, header_row)
```
- `header_row` 第 1 列 `Remote_Ratio` 是行标签列名；后 7 列是 TPC-C 的 7 档「远程 NewOrder%/远程 Payment%」，**顺序必须与 `common.sh:35-41` 七次调用的顺序一致**（0/0,10/15,…,60/90），因为 `get_row` 是按吞吐值在文件里出现的先后顺序收集的。

### 1.3 主图 CSV：`construct_input_list_tpcc`（parse_tpcc.py:30-37）

```python
def construct_input_list_tpcc(tpcc_res_dir):
        input_file_list = list()
        input_file_list.append(("Tigon", tpcc_res_dir + "/tpcc-TwoPLPasha-8-3-1-0-Clock-OnDemand-209715200-1-WriteThrough-NonPart-GROUP_WAL-10000-0.txt"))
        # input_file_list.append(("Tigon-Phantom", .../tpcc-TwoPLPashaPhantom-...txt))   # 被注释：主图不画关幻读防护的版本
        input_file_list.append(("Sundial-CXL-improved", tpcc_res_dir + "/tpcc-Sundial-8-3-1-0-NoMoveOut-OnDemand-0-0-NoOP-None-GROUP_WAL-10000-0.txt"))
        input_file_list.append(("TwoPL-CXL-improved",   tpcc_res_dir + "/tpcc-TwoPL-8-3-1-0-NoMoveOut-OnDemand-0-0-NoOP-None-GROUP_WAL-10000-0.txt"))
        return input_file_list
```

只取 3 个系统：
- **Tigon** = `TwoPLPasha`，文件名后缀 `Clock-OnDemand-209715200-1-WriteThrough-NonPart`（迁移 Clock、预算 200MB、SCC WriteThrough、预迁移 NonPart）—— 全功能。
- **Sundial-CXL-improved / TwoPL-CXL-improved** = 两个最强基线（3 worker、CXL 传输、无 Pasha）。主图 Fig 4(c) 只跟「改进版基线」比，不放 CXL/NET 弱版（那些在 a/b 子图里展示）。
- `Tigon-Phantom` 被注释掉：幻读防护开销不进主吞吐图。

```python
def parse_tpcc(tpcc_res_dir, motor_tpcc_csv):
        input_file_list = construct_input_list_tpcc(tpcc_res_dir)
        output_file_name = tpcc_res_dir + "/tpcc.csv"
        header_row = ["Remote_Ratio", "0/0", ..., "60/90"]            # 同 7 档
        parse_results(input_file_list, output_file_name, header_row)  # 先写 Tigon/Sundial+/DS2PL+ 三列
        append_motor_numbers(output_file_name, motor_tpcc_csv)        # 再把 Motor 列拼上
```

### 1.4 主程序（parse_tpcc.py:48-62）

```python
if len(sys.argv) != 2:                                      # 必须 1 个参数
        print("Usage: ... RESULT_ROOT_DIR"); sys.exit(-1)
res_root_dir = sys.argv[1]                                  # 结果根目录(如 results/test1)
tpcc_res_dir = res_root_dir + "/tpcc"                       # TPCC 子目录
script_path = os.path.abspath(__file__)                    # 本脚本绝对路径
script_directory = os.path.dirname(script_path)            # 本脚本所在目录(scripts/parse)
motor_tpcc_csv = script_directory + "/../../results/motor/tpcc.csv"   # 仓库内预存的 Motor 基线
parse_tpcc_baseline(tpcc_res_dir)                          # 产出 baseline-tpcc.csv(6 列基线)
parse_tpcc(tpcc_res_dir, motor_tpcc_csv)                   # 产出 tpcc.csv(Tigon+2基线+Motor)
```

- **Motor CSV 路径用脚本自身位置算**（`__file__` → `../../results/motor/tpcc.csv`），不受调用时工作目录影响，保证总能找到仓库里预存的 Motor 数据。
- Motor 是需要 4 台 RDMA 机器的外部基线（README:31），无法在本机跑，所以用预测好的 CSV 直接合并。

### 1.5 Motor CSV 的结构与合并

`results/motor/tpcc.csv` 实际内容：
```
Remote_Ratio,Motor
0/0,28528.4
10/15,28129.8
20/30,28428.6
30/45,28896.8
40/60,28489.6
50/75,28909.7
60/90,28034.4
```
7 行恰好对应 7 档远程比例。`append_motor_numbers`（common.py:40-44）只取它的 `Motor` 一列，按行号对齐 `concat` 到 `tpcc.csv` 右侧（**靠行序对齐，不做 key join**——所以两边远程比例的顺序必须一致）。

> **解析阶段产物小结**：
> - `tpcc/baseline-tpcc.csv`：8 列（Remote_Ratio + 6 基线），供 a/b 两图。
> - `tpcc/tpcc.csv`：5 列（Remote_Ratio + Tigon + Sundial-CXL-improved + TwoPL-CXL-improved + Motor），供 c 图。

---

## 2. `plot_tpcc_sundial.py` 逐行分析（Fig 4(a)）

### 2.1 字体与启动（plot_tpcc_sundial.py:1-17）

```python
import sys, math, pandas as pd
import matplotlib
import matplotlib.pyplot as plt
import matplotlib.ticker as ticker
matplotlib.rcParams['pdf.fonttype'] = 42       # PDF 内文字用 TrueType(Type 42)，可被编辑/复制，论文出版要求
matplotlib.rcParams['ps.fonttype'] = 42        # 同理 PS 输出
if len(sys.argv) != 2: ...; sys.exit(-1)        # 1 个参数
res_root_dir = sys.argv[1]
```
- `fonttype=42`：把字体嵌成 Type42(TrueType)，避免默认 Type3 位图字体——这样 PDF 里的文字是矢量可选的（投稿、缩放都不糊）。

### 2.2 全局样式（plot_tpcc_sundial.py:19-29）

```python
DEFAULT_PLOT = {                                # 所有曲线共用的线/点样式
    "markersize": 12.0,                         # 标记点大小
    "markeredgewidth": 2.6,                     # 标记边线宽
    "markevery": 1,                             # 每个数据点都画标记(不跳点)
    "linewidth": 2.6,                           # 线宽
}
plt.rcParams["font.size"] = 14                  # 全局字号 14
plt.grid(axis='y')                              # 只画水平(y)网格线，便于读吞吐数值
```

### 2.3 读数据、取列（plot_tpcc_sundial.py:31-45）

```python
res_csv = res_root_dir + "/tpcc/baseline-tpcc.csv"   # 用基线 CSV(6 列)
res_df = pd.read_csv(res_csv)                         # 读成 DataFrame
x = res_df["Remote_Ratio"]                            # X 轴 = 远程比例(0/0,10/15,...) 字符串当类别轴

sundial_cxl_improved_y = res_df["Sundial-CXL-improved"]   # 三条 Sundial 曲线的 Y
sundial_cxl_y          = res_df["Sundial-CXL"]
sundial_net_y          = res_df["Sundial-NET"]

twopl_cxl_improved_y = res_df["TwoPL-CXL-improved"]   # 这三条 TwoPL 也读了
twopl_cxl_y          = res_df["TwoPL-CXL"]            # 但本脚本最终只画 Sundial 三条
twopl_net_y          = res_df["TwoPL-NET"]            # (读 TwoPL 只为下面算 Y 上限时凑数，实际未用于 ylim)
```
- X 轴直接用 `Remote_Ratio` 字符串列，matplotlib 当**类别轴**等距排开 7 个刻度（"0/0"…"60/90"）。
- 注意：脚本读了全部 6 列，但**只绘制 3 条 Sundial 线**（见 2.5）。读 TwoPL 列是模板复制残留。

### 2.4 轴标签与 Y 上限计算（plot_tpcc_sundial.py:47-65）

```python
plt.xlabel("Multi-partition Transaction Percentage")   # X 轴标题
plt.ylabel("Throughput (txns/sec)")                    # Y 轴标题

tmp_list = list()                                       # 汇总所有 Y 值算最大值
tmp_list.extend(sundial_cxl_improved_y); ...; tmp_list.extend(twopl_net_y)
max_y = max(tmp_list)
max_y_rounded_up = math.ceil(max_y / 200000.0) * 200000.0   # 把最大值向上取整到 20 万的倍数
                                                            # (本脚本算了但没用，被下面写死的 ylim 覆盖)
ax = plt.subplot(111)                                   # 单子图
plt.ylim(0, 800000)                                     # Y 轴范围写死 0~80万(a/c 图统一上限便于横比)
ax.yaxis.set_major_formatter(ticker.FuncFormatter(
    lambda x, pos: '{:,.0f}'.format(x/1000) + 'K' if x != 0 else 0))   # 刻度格式化：除以1000加'K'，0 显示为 0
```
- `max_y_rounded_up` 计算后未使用（`plt.ylim` 写死 800000）——是从主图模板沿用的死代码。
- Y 轴 formatter：把 `200000` 显示成 `200K`、`0` 显示成 `0`，让纵轴更易读。

### 2.5 画三条 Sundial 线（plot_tpcc_sundial.py:67-70）

```python
plt.plot(x, sundial_cxl_improved_y, color="#4372c4", marker="s", **DEFAULT_PLOT, markerfacecolor="none", label="Sundial+")
plt.plot(x, sundial_cxl_y,          color="#4372c4", marker="^", **DEFAULT_PLOT, markerfacecolor="none", label="Sundial-CXL")
plt.plot(x, sundial_net_y,          color="#4372c4", marker=">", **DEFAULT_PLOT, markerfacecolor="none", label="Sundial-NET")
```
- 同色 `#4372c4`(蓝)，靠**标记形状**区分三个变体：`s`=方块(improved/「+」)、`^`=上三角(CXL)、`>`=右三角(NET)。
- `markerfacecolor="none"`：空心标记（只描边不填充）。
- `label="Sundial+"`：注意 CSV 列名 `Sundial-CXL-improved` 在图例里显示为论文术语 **Sundial+**。

### 2.6 图例与导出（plot_tpcc_sundial.py:72-76）

```python
legend = ax.legend(loc='upper right', bbox_to_anchor=(1.019, 1.026),
                   columnspacing=1, frameon=True, fancybox=False, framealpha=1, ncol=1, edgecolor='black')
legend.get_frame().set_linewidth(0.7)                  # 图例边框线宽
plt.savefig(res_root_dir + "/tpcc/tpcc-sundial.pdf", format="pdf", bbox_inches="tight")
```
- `loc='upper right'` + `bbox_to_anchor=(1.019,1.026)`：图例钉在右上角略微出框（精调位置）。
- `ncol=1`：图例单列；`frameon/framealpha/edgecolor`：黑色实线边框、不透明。
- `bbox_inches="tight"`：导出时裁掉多余白边。
- 产出 → **`tpcc/tpcc-sundial.pdf` = 论文 Figure 4(a)**。

---

## 3. `plot_tpcc_twopl.py` 逐行分析（Fig 4(b)）

与 `plot_tpcc_sundial.py` **结构完全相同**，仅 3 处差异：

| 项 | sundial 版 | twopl 版 | 行号 |
|---|---|---|---|
| Y 上限 `plt.ylim` | `0, 800000` | **`0, 700000`** | :64 |
| 画哪三条线 | Sundial 三变体 | **TwoPL 三变体**（`twopl_cxl_improved_y/twopl_cxl_y/twopl_net_y`） | :68-70 |
| 颜色 + 图例标签 | `#4372c4` 蓝 / Sundial+/Sundial-CXL/Sundial-NET | **`#ffc003` 黄** / **DS2PL+/DS2PL-CXL/DS2PL-NET** | :68-70 |
| 输出文件 | tpcc-sundial.pdf | **tpcc-twopl.pdf** | :76 |

```python
res_csv = res_root_dir + "/tpcc/baseline-tpcc.csv"     # 同样读基线 CSV
...
plt.ylim(0, 700000)                                     # DS2PL 吞吐峰值低些，Y 上限设 70 万
plt.plot(x, twopl_cxl_improved_y, color="#ffc003", marker="s", ..., label="DS2PL+")     # 黄色方块
plt.plot(x, twopl_cxl_y,          color="#ffc003", marker="^", ..., label="DS2PL-CXL")  # 黄色上三角
plt.plot(x, twopl_net_y,          color="#ffc003", marker=">", ..., label="DS2PL-NET")  # 黄色右三角
plt.savefig(res_root_dir + "/tpcc/tpcc-twopl.pdf", ...)
```
- **术语映射**：代码里的 `TwoPL` = 论文里的 **DS2PL**（Distributed Strict 2PL）。CSV 列名是 `TwoPL-*`，图例标签换成 `DS2PL-*`。
- 标记形状约定与 a 图一致（`s`/`^`/`>` = +/CXL/NET），方便读者横向对照两个子图。
- 产出 → **`tpcc/tpcc-twopl.pdf` = 论文 Figure 4(b)**。

> a 图蓝、b 图黄的配色，与 c 图里 Sundial+(蓝)/DS2PL+(黄) 保持一致——同一系统在所有子图同色。

---

## 4. `plot_tpcc.py` 逐行分析（Fig 4(c)，主图）

### 4.1 读主图 CSV、取 4 列（plot_tpcc.py:31-43）

```python
res_csv = res_root_dir + "/tpcc/tpcc.csv"              # 用含 Motor 的主 CSV(不是 baseline)
res_df = pd.read_csv(res_csv)
x = res_df["Remote_Ratio"]
tigon_y                = res_df["Tigon"]               # Tigon
sundial_cxl_improved_y = res_df["Sundial-CXL-improved"]# Sundial+
twopl_cxl_improved_y   = res_df["TwoPL-CXL-improved"]  # DS2PL+
motor_y                = res_df["Motor"]               # Motor(预存基线)
```
- 这里用的是 `parse_tpcc` 产出的 `tpcc.csv`，已含 4 个可画列（Tigon/Sundial+/DS2PL+/Motor）。

### 4.2 Y 上限（plot_tpcc.py:45-62）

```python
plt.xlabel("Multi-partition Transaction Percentage")
plt.ylabel("Throughput (txns/sec)")
tmp_list = [...tigon_y, sundial_cxl_improved_y, twopl_cxl_improved_y, motor_y...]
max_y = max(tmp_list)
max_y_rounded_up = math.ceil(max_y / 200000.0) * 200000.0
plt.ylim(0, max_y_rounded_up)                          # 先按数据动态设上限
ax = plt.subplot(111)
plt.ylim(0, 800000)                                     # 又写死 80 万(覆盖上一行；与 a 图统一)
ax.yaxis.set_major_formatter(ticker.FuncFormatter(lambda x, pos: '{:,.0f}'.format(x/1000)+'K' if x!=0 else 0))
```
- 这里出现了**两次 `plt.ylim`**：先按 `max_y_rounded_up` 动态算，紧接着又写死 `800000`。后者生效——作者最终固定 80 万上限让主图与 a 图同尺度。动态那行是开发中途的残留。

### 4.3 画 4 条线（plot_tpcc.py:64-68）

```python
plt.plot(x, tigon_y,                color="#000000", marker="s", **DEFAULT_PLOT,                      label="Tigon")     # 黑色实心方块，突出主角
plt.plot(x, sundial_cxl_improved_y, color="#4372c4", marker="^", **DEFAULT_PLOT, markerfacecolor="none", label="Sundial+")  # 蓝空心上三角
plt.plot(x, twopl_cxl_improved_y,   color="#ffc003", marker=">", **DEFAULT_PLOT, markerfacecolor="none", label="DS2PL+")    # 黄空心右三角
plt.plot(x, motor_y,                color="#ed7d31", marker="o", **DEFAULT_PLOT, markerfacecolor="none", label="Motor")     # 橙空心圆
```
- **Tigon 用黑色 + 实心标记**（没有 `markerfacecolor="none"`），刻意比其它三条更醒目——它是论文主角。
- 配色与 a/b 图呼应：Sundial+ 蓝(`#4372c4`)、DS2PL+ 黄(`#ffc003`)，加上 Motor 橙(`#ed7d31`)。

### 4.4 图例与导出（plot_tpcc.py:70-74）

```python
legend = ax.legend(loc='upper right', bbox_to_anchor=(1.019, 1.026),
                   columnspacing=1, frameon=True, fancybox=False, framealpha=1, ncol=2, edgecolor='black')
legend.get_frame().set_linewidth(0.7)
plt.savefig(res_root_dir + "/tpcc/tpcc.pdf", format="pdf", bbox_inches="tight")
```
- 与 a/b 唯一图例差异：`ncol=2`（4 条曲线排成 2 列，更紧凑）。
- 产出 → **`tpcc/tpcc.pdf` = 论文 Figure 4(c)**。

---

## 5. 回顾解析引擎 `common.py`（数值如何变成曲线）

三张图的 Y 值最终都源自 `get_row` 从日志里抓的一个数。再次明确（详见 `docs/experiment_scripts_params.md` §4.5）：

```python
def get_row(input):                              # input = (标签, .txt 路径)
        tputs = [input[0]]                       # 行首是图例标签
        for line in fileinput.FileInput(input[1]):
                tokens = line.strip().split()
                if len(tokens) > 7 and tokens[3] == "Coordinator.h:610]":
                        tputs.append(float(tokens[7]))   # tokens[7] = total_commit 数值
        return tputs                             # [标签, 第0档吞吐, 第1档, ...]
```
对日志行 `I... 数字 PID Coordinator.h:610] Global Stats: total_commit: 360162 ...`，`tokens[7]`=`360162`=测量窗口内总提交事务数（作吞吐指标）。一个 `.txt` 含 7 档远程比例 → 7 个 total_commit 行 → 7 个 Y 值。

```python
def parse_results(input_list, output_file_name, header_row):
        rows = [header_row] + [get_row(i) for i in input_list]   # 行=系统
        rows = zip(*rows)                                        # 转置：行=远程比例、列=系统
        csv.writer(open(output_file_name,"w")).writerows(rows)
```
转置后 CSV 的每一**行**是一个远程比例点、每一**列**是一个系统——正好让 plot 脚本 `res_df["Remote_Ratio"]` 当 X、`res_df["Tigon"]` 等当各条曲线的 Y。

```python
def append_motor_numbers(output_file_name, motor_csv_name):
        orig_df = pd.read_csv(output_file_name)                  # 刚写好的 tpcc.csv
        motor_df = pd.read_csv(motor_csv_name, usecols=['Motor'])# 只取 Motor 列
        orig_df = pd.concat([orig_df, motor_df], axis=1)         # 按行序横向拼接
        orig_df.to_csv(output_file_name, index=False)            # 覆盖写回
```
`axis=1` + 按行号对齐（**非键连接**）——所以 Motor CSV 的 7 行必须和实验的 7 档远程比例同序，否则会错位。

---

## 6. 四脚本横向对比与易错点

| 项 | parse_tpcc.py | plot_tpcc_sundial.py | plot_tpcc_twopl.py | plot_tpcc.py |
|---|---|---|---|---|
| 角色 | 解析(txt→csv) | 绘图 Fig 4(a) | 绘图 Fig 4(b) | 绘图 Fig 4(c) |
| 读入 | 8 个 .txt | baseline-tpcc.csv | baseline-tpcc.csv | **tpcc.csv** |
| 产出 | baseline-tpcc.csv + tpcc.csv | tpcc-sundial.pdf | tpcc-twopl.pdf | tpcc.pdf |
| 画几条线 | — | 3(Sundial) | 3(DS2PL) | 4(Tigon/Sundial+/DS2PL+/Motor) |
| 主色 | — | 蓝 #4372c4 | 黄 #ffc003 | 黑(Tigon)+蓝+黄+橙 |
| Y 上限 | — | 800000 | 700000 | 800000 |
| 图例列数 | — | 1 | 1 | 2 |
| 含 Motor | tpcc.csv 含 | 否 | 否 | 是 |

**贯穿全链的隐性契约 / 易错点**：

1. **文件名即接口**：parse 端硬编码的 `.txt` 文件名（含 `8-3`、`209715200`、`WriteThrough` 等）必须逐字段匹配 `run_tpcc.sh` 跑出的名字。改了实验配置（如换 worker 数、预算）却不同步改 parse_tpcc.py，会"文件找不到"。
2. **顺序对齐而非键对齐**：header 的 7 档顺序、Motor CSV 的 7 行顺序、日志里 7 个 total_commit 的出现顺序——三者必须一致，全靠位置对齐。
3. **术语映射**：代码 `TwoPL` → 图例 `DS2PL`；CSV 列 `*-CXL-improved` → 图例 `*+`。
4. **死代码**：`max_y_rounded_up` 算了不用、`plt.ylim` 写两遍、sundial 脚本读了 TwoPL 列却不画——都是从同一模板复制衍生的痕迹，不影响结果。
5. **`fonttype=42`**：保证 PDF 文字可编辑/矢量，是论文投稿的硬要求。
6. **Tigon 视觉强调**：主图里唯独 Tigon 用黑色实心标记，其余系统空心——刻意的主角高亮。
