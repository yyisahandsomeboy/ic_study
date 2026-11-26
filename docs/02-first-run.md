# 02 用 VCS 跑仿真，用 Verdi 看波形

Vivado 的 Run Behavioral Simulation 会替你安排编译、展开和运行。这个工程把对应操作写进了 `sim/Makefile`。`make compile` 调 VCS，读入 RTL、UVM 与 VIP 源码，展开 `tb_top`，生成可执行文件 `simv`。`make run` 调用 `simv`，把用例名、随机 seed、UVM 日志级别和超时作为运行参数。同一个 `simv` 可以运行多个已经编译进去的 UVM 测试类。

## 1. 先读一份已有结果

在普通 EDA 账号的 CentOS 终端执行。每开一个新终端，变量都要重新定义。

```bash
project="$HOME/pcie_gen6_cxl_proj/designware_home/pcie_dev/my_test_project"
cd "$project/sim"
ls -lh Makefile simv compil.log sim.log pcie.fsdb transaction.log
grep -nE 'Running test|link ip|BAR0|VEC_ADD|UVM_(WARNING|ERROR|FATAL)[[:space:]]*:' sim.log
```

已有 `sim.log` 记录的是 `pcie_mem_pkg_test`。它枚举出一个 64 KiB 的 32 位 Memory BAR0，并分配地址 `0x0040_0000`。四个结果 DW 依次是 `0x18161412`、`0x201e1c1a`、`0x28262422`、`0x302e2c2a`。UVM 汇总为 1 个 warning、0 个 error、0 个 fatal。先查 warning 所在行，再看它是否影响功能检查。汇总区按日志 ID 统计，搜索 `VEC_ADD` 时把四行具体数值和最后的计数区分开。

一次通过至少需要三项证据同时成立。

1. 实际启动了指定测试，命令正常退出。
2. 测试走到了规定检查点。例如链路用例检查物理链路、L0、目标速率和数据链路状态，向量用例还应执行四次结果比较。
3. 日志的 UVM error/fatal 为 0，且核对了 warning。有些环境的 `simv` 在 UVM 报错后仍可返回 0，所以退出码和日志都要检查。

## 2. 在独立目录复跑

主工程的 filelist 和若干库选项包含绝对路径。以下方法在同一台 VM 使用原始源码，只将构建产物写入新目录。主工程的 `sim/` 已有历史波形，运行以下命令时先确认 `pwd` 显示新目录。Makefile 的 `clean` 会按通配符删除日志、波形和可执行文件。

```bash
project="$HOME/pcie_gen6_cxl_proj/designware_home/pcie_dev/my_test_project"
work=$(mktemp -d "$HOME/ic-study-run.XXXXXX")
mkdir "$work/sim"
for dir in rtl tb tc; do
  ln -s "$project/$dir" "$work/$dir"
done
cp "$project/sim/Makefile" "$work/sim/"
cd "$work/sim"
pwd
type -P vcs
type -P verdi
make -n compile run UVM_TESTNAME=pcie_link_up_test SEED=20261125 > recipe.txt
less recipe.txt
```

在 `less` 按 `q` 退出。先认出三组参数。编译的 `-f` 指向 `rtl/pcie_rtl.f`，`+incdir+` 供编译器查找 include，`-y` 指定模块库目录。`-top tb_top` 是测试平台顶层，主工程还包含 Xilinx 仿真所需的 `glbl`。`-ntb_opts uvm-1.2` 选择 UVM 版本。运行的 `+UVM_TESTNAME` 选测试类，`+ntb_random_seed` 固定随机性。编译版本、有效 filelist、测试参数和 seed 都应保存在运行记录里。

```bash
make compile > compile.console.log 2>&1
echo "compile exit=$?"
tail -n 25 compile.console.log
ls -lh simv
```

首次编译会处理大量 PCIe 与 Xilinx 仿真文件。编译结束并成功生成 `simv` 后，再运行以下命令。这里分开保存两次运行的产物，因为工程固定使用 `sim.log`、`pcie.fsdb` 和 `transaction.log`。

```bash
# 以下命令仍在 "$work/sim" 执行
run_case() {
  local name=$1 rc=0 file
  local evidence
  evidence=$(mktemp -d "$work/$name.XXXXXX") || return
  if [ -e sim.log ] || [ -e pcie.fsdb ] || [ -e transaction.log ]; then
    printf 'Save the existing outputs in %s first.\n' "$PWD"
    return 2
  fi
  make run UVM_TESTNAME="$name" SEED=20261125 > "$evidence/console.log" 2>&1 || rc=$?
  for file in sim.log pcie.fsdb transaction.log; do
    if [ -f "$file" ]; then mv "$file" "$evidence/"; fi
  done
  printf '%s exit=%s evidence=%s\n' "$name" "$rc" "$evidence"
  grep -nE 'Running test|link ip|BAR0|VEC_ADD|UVM_(WARNING|ERROR|FATAL)[[:space:]]*:' \
    "$evidence/console.log" | tail -n 22
  return "$rc"
}
run_case pcie_link_up_test
# 确认链路测试符合上述通过标准后再继续
run_case pcie_mem_pkg_test
```

下一遍可分别选 `pcie_cfg_pkg_test`（最小配置访问）和 `pcie_enumeration_device_test`（完整示例枚举）。修改 RTL、include、宏或编译选项之后重新编译。单独改已编入的 UVM 测试名或 seed，可以复用原 `simv`。Makefile 的 UVM 超时基于仿真时间，和程序在服务器上已跑多久是两回事。源代码中的 `timescale` 与 UVM 日志所用的时间显示精度也应一起核对。

2026 年 11 月 25 日，在这台 VM 用 VCS V-2023.12-SP2 从独立目录重编译，再用 seed `20261125` 运行同一个 `simv`。四个用例的进程退出码均为 0，各有 1 条 UVM warning，均有 0 条 UVM error 和 fatal；链路到达测试的检查点，枚举得到预期 BAR0，向量用例四个 DW 全部匹配。记下这些检查点，今后用同一个 seed 比较源码改动前后的结果。

## 3. 用 Verdi 从日志回到信号

打开 CentOS 桌面终端。真实工程自带的入口是 `make verdi`，它用设计文件列表加载顶层和源代码，并读取当前目录下的 `pcie.fsdb`。若只看波形，可以直接给 Verdi 指定已经保存的 FSDB。

```bash
cd "$project/sim"
make -n verdi
make verdi
# 或者把下面的示例路径换成上一步输出的独立运行目录
verdi -ssf /path/to/saved/run/pcie.fsdb
```

1. 在层级窗口找 `tb_top` 和 `u_pcie_ep`。先加参考时钟、复位和 TX/RX lane 0。确认复位解除后信号开始活动，再从日志获取进入 L0 的时间。
2. 以一次 `BAR0 + 0x30` 的 Memory Read 为索引。在 `transaction.log` 找请求地址、方向、长度和 tag，再找相同请求对应的 Completion。路径和时间单位以本次日志为准。
3. 到 `u_pcie_ep` 的用户 AXI 一侧加 AR/R 通道的地址、数据、VALID 和 READY。把光标放在上升沿，记录 AR 握手在哪一拍，R 握手在哪一拍。对于写操作同时看 AW、W、B，地址和数据允许不同拍到达。
4. 展开向量寄存器实例，观察 CTRL、计算状态及结果更新。把 TLP 请求、AXI 握手、结果寄存器或读响应各自的时间填到同一张表。抓不到内部信号时先确认波形是否记录到该层级，再检查编译的 debug 选项。

`pcie.fsdb` 记录事件值与时间，`transaction.log` 展示解码后的协议事务，`sim.log` 记录 UVM 的检查结论。把三者对齐能定位第一处异常，例如 PCIe 请求已经收到，但 AXI 读通道仍没有握手。也可以保留更小的信号集合，避免每次产生全层级波形。Verdi 功能说明见 [Synopsys Verdi](https://www.synopsys.com/verification/debug/verdi.html)。

用本次复跑的首个结果读回练习。以 `tb_top.u_pcie_ep.axi_clk` 的上升沿采样 `m_axi` 和 `axi_lite_m`，再和 `transaction.log` 对齐。

| 仿真时间 | 从哪看 | 观察 |
| --- | --- | --- |
| 257645 ns | RC 侧事务日志 | `MRd32` 开始，地址 `0x0040_0030`，长度 1 DW，tag `0x1f` |
| 258080.405 ns | `m_axi` 与 `axi_lite_m` | 两侧都出现 AR 握手，`ar_addr` 均为 `0x0040_0030` |
| 258088.405 ns | `axi_lite_m` 与 `m_axi` | 两侧都出现 R 握手，`r_data` 均显示为 `0x12141618` |
| 258293 ns | RC 侧事务日志 | 对应 `CplD` 开始，完成状态成功，payload 显示为 `0x18161412` |

把 AXI 和 PCIe 日志的显示值按字节拆开，可以看到相同四个字节以不同顺序展示。对于这份日志，用 requester ID 和 tag 关联请求与完成。不同 seed 的 tag 可能变化。附近的 ACK 属于数据链路层，携带读回数据的是事务层 CplD。

无桌面会话时，可在保存了 `pcie.fsdb` 的目录提取这段信号。Verdi 随附的 `fsdb2vcd` 已用以下参数验证，可用支持 VCD 的波形工具打开输出。

```bash
"$VERDI_HOME/bin/fsdb2vcd" pcie.fsdb -bt 257640ns -et 258395ns \
  -s /tb_top/u_pcie_ep -level 0 -o first-result-read.vcd
```

## 4. 按第一处失败定位

| 症状 | 先查哪里 | 观察什么 |
| --- | --- | --- |
| shell 找不到 `vcs` | `id -un`、`echo "$HOME"`、`type -P vcs` | 是否在 CentOS 普通账号，是否加载 bash 环境 |
| VCS 找不到源码或编译包 | `recipe.txt`、`compil.log`、`rtl/pcie_rtl.f` | 第一条错误的文件和行，绝对路径是否有效，package 定义是否先于使用 |
| module unresolved 或原语找不到 | `-y`、`+libext`、`glbl`、Vivado 仿真库 | 编译日志是否纳入具体模块，参数和顶层是否正确 |
| 工具启动时提示授权错误 | 第一处工具报错和本机工具手册 | 由环境管理员核对工具 feature、许可证服务与配置 |
| 找不到 UVM 测试或虚接口 | `tb/tb_include.sv`、测试 factory 注册、`uvm_config_db` | 测试类是否编译，set/get 的类型、作用域和键是否一致 |
| 链路用例卡在 wait | 复位、参考时钟、lane 连接、LTSSM、物理与数据链路状态 | 链路卡在第几项条件。先看 lane 0 两侧的方向和 reset 时序 |
| 枚举成功但 BAR 读写超时 | BAR、Memory Space Enable、TLP tag、AXI 握手 | 请求在地址译码、AW/W/AR、B/R 或 Completion 的哪一步消失 |
| 结果只有个别字节错 | 数据字节序、byte enable、WSTRB、strobe 边界、按字节加法 | 找第一笔写入真值，沿 TLP 到寄存器逐层对比 |
| Verdi 图形窗口打不开 | VM 桌面会话和 `DISPLAY` | 用 CentOS 图形终端启动。SSH 中可先看日志 |

[下一章：从用例读到 Endpoint RTL](03-project-walkthrough.md)
