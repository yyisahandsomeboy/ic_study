# 05 把项目接到数字 IC 全流程

学习数字 IC 可以同时跟两条线。设计线决定功能怎么变成满足时序、功耗和面积要求的硬件。验证线把需求分解成可观察的属性，用测试、断言和覆盖率衡量证据。这两条线在同一份规格、寄存器表、RTL 版本和变更记录上交汇。

## 1. 按输入、输出和通过标准检查每一步

| 阶段 | 主要输入 | 主要产物和退出条件 | 本机可以看什么 |
| --- | --- | --- | --- |
| 规格、架构和微架构 | 产品需求、协议约束、性能与资源目标 | 数据路径、控制路径、寄存器表、复位和异常语义，需求能转成验收条件 | 用主工程逆向写一页 Endpoint 应用规格 |
| RTL 与子模块验证 | 微架构、时钟复位约定、接口定义 | 可综合 RTL、单元测试、断言，正常路径和边界都能自检 | `rtl/pcie_ep.sv`、第 4 章原创外设 |
| 时钟与静态检查 | RTL、时钟与复位拓扑 | lint、CDC、RDC 问题清单与收敛记录，例外均有依据 | 参考 IP 工作区的 `spyglass/`。确认当前检查运行的配置和范围 |
| 功能验证与覆盖率收敛 | DUT、验证计划、SV/UVM、VIP、参考模型 | 仿真日志、断言和覆盖率，需求能追到确实执行的检查 | VCS、Verdi、项目测试、VIP 日志。可增加自己的覆盖率实验 |
| 综合 | RTL、工艺标准单元 Liberty、SDC、宏模型等 | 映射后网表、约束、面积与时序报告，处理未解析引用和不合理约束 | `syn/`、`export/` 与 `report/` 中的 DesignWare 历史产物。Vivado 综合报告可对照 |
| 逻辑等价 | 参考 RTL、综合网表、映射或引导信息 | 通过匹配和比较的检查点，逐项分析 unmatched 或 failing 点 | `syn/formality.log`、`syn/FM_INFO/` 和 `fm_shell` |
| DFT | RTL 或综合网表、测试约束、测试目标 | 扫描链、可测试性规则、ATPG 向量和故障覆盖率 | 作为 ASIC 独立练习。查看工作区测试策略时，先确认产物属于哪次流程 |
| 物理实现 | ASIC 网表、SDC、工艺与宏物理视图、电源计划 | floorplan、Placement、CTS、Routing 各阶段数据库，拥塞和时序逐步收敛 | 用 Vivado routed 工程练报告阅读。ASIC 需匹配的 PDK、库和物理工具 |
| 多角 STA、功耗、物理签核 | 寄生参数、版图、网表、时钟约束及各工艺角模型 | setup/hold、信号完整性、功耗、IR drop/EM、DRC、LVS、签核报告和版图交付 | 分辨角色，按已有报告练读法。完整 ASIC 签核需项目级工艺和约束 |
| 原型与上板 | FPGA bitstream、板卡、主机、时钟复位约束 | PCIe 枚举、寄存器访问、硬件稳定性与性能证据 | `pcie_gpgpu2` 有历史 bitstream，先阅读时序报告和板卡约束 |

不同流程的产物能相互解释。例如功能仿真告诉你读回正确，STA 告诉你在指定时序模型与约束下数据能否按时送达，等价验证则比较 RTL 与网表的逻辑语义。完成一个阶段靠该阶段的验收证据。过程参考 [Synopsys 数字 IC 设计流程](https://qacf-www.synopsys.com/glossary/what-is-ic-design.html) 和 [Synopsys 物理验证](https://www.synopsys.com/implementation-and-signoff/physical-verification.html)。

## 2. 为 PCIe 用例写一张能执行的验证计划

需求写成“前置条件、刺激、期望、观察位置”。例如“已枚举且 Memory Space Enable 置位。向 BAR0 的 B 区以单字节使能写入。读回时只更新该字节。用 AXI WSTRB 与 PCIe 读回 payload 验证”。

| 需求 | 刺激与检查 | 覆盖点 |
| --- | --- | --- |
| 链路可用 | 正常上电与复位，检查物理链路、L0、2.5 GT/s 与数据链路全部到位 | reset → L0，目标速率和宽度，链路恢复场景 |
| 枚举与地址窗口正确 | 读配置头，探测 BAR，分配地址并读回，检查启用后访问 | Config Read/Write、BAR 探测与分配、窗口边界 |
| 寄存器遵守访问规则 | 对 A/B 做全字节和部分写，读回保留未使能字节。对结果区测试写保护 | 地址类别 × 操作类型 × byte enable |
| 向量加法逐元素正确 | 全零、交替值、各字节溢出、不同 lanes 的边界，逐 8 位独立计算 | 元素溢出、边界值、结果 DW、全部 16 个位置 |
| 背压下返回保持稳定 | 拉低 AXI 的响应 READY，在重启接收后检查一次且仅一次响应 | 无背压、单拍、多拍，读和写通道分别统计 |
| 复位和重新运行的行为明确 | 训练完成、事务进行中或计算中施加复位，按规格检查状态和重新枚举 | 复位时刻 × 链路阶段 × 未完成请求状态 |
| 错误有界而且可诊断 | 在已定义的地址或请求类型场景检查预期错误响应与超时，保存第一处异常 | 可见的错误类别、处理路径与超时上限 |

先收敛确定性 smoke：link → config → enumeration → memory。再为每个边界建独立测试，从固定 seed 扩展到多个 seed。每次记录工程版本或源码指纹、VCS 版本、测试名、seed、编译参数、退出状态、UVM 摘要、通过的检查数和异常。重跑失败 seed 时先复用同版源码和编译配置，再改变一个条件。不同仿真写同名波形的工程应按测试和 seed 创建独立目录。

三类观测各有用途。代码覆盖率显示行、分支、翻转是否执行。功能覆盖率显示计划中的状态和组合是否发生，例如只读地址 × 写请求。断言直接检查性质，例如背压时数据保持。一个用例执行到某行没有自动验证该行效果，应为重要覆盖点配上 scoreboard 或断言。现有项目 Makefile 未默认收集覆盖率。做覆盖率实验时，用 VM 自带相同版本 VCS/URG 帮助核对 `-cm`、`-cm_dir` 和 `urg -dir`，在独立运行目录生成 `.vdb`，再按同一构建和覆盖模型汇总。先给一个空的功能覆盖点写定向测试，复跑观察它填满，再解释它为什么之前未触发。参见 [Synopsys 验证工具](https://www.synopsys.com/verification.html)。

## 3. 从 Vivado 的真实历史结果理解时序

打开 `$HOME/Downloads/pcie_gpgpu2/project_1/project_1.xpr`。先在 GUI 里确认 Top 为 `pcie_ep`、Part 为 `xcku5p-ffvb676-2-i`。在 `project_1.runs/impl_1/` 读 `pcie_ep_timing_summary_routed.rpt`。

```bash
impl="$HOME/Downloads/pcie_gpgpu2/project_1/project_1.runs/impl_1"
grep -nE 'unconstrained_internal_endpoints|Timing constraints are not met|WNS\(ns\)' \
    "$impl/pcie_ep_timing_summary_routed.rpt" | head -n 12
less "$impl/pcie_ep_timing_summary_routed.rpt"
```

历史 routed 报告的 setup WNS 是 **−3.749 ns**，TNS 是 **−10470.497 ns**，有 **13,653** 个 failing endpoints，同时列出 **21** 个 unconstrained internal endpoints。这个历史运行生成了 bitstream，报告依然显示时序未满足。可以从 GUI 的 Open Implemented Design 和 Timing Summary 看到同类信息。先定位一个真实失败路径，不需要直接打开时序最多的分组。报告结构见 [AMD Vivado 2023.1 Timing Summary](https://docs.amd.com/r/2023.1-English/ug906-vivado-design-analysis/Report-Timing-Summary) 和 [Check Timing](https://docs.amd.com/r/2023.1-English/ug906-vivado-design-analysis/Check-Timing-Section)。

读一条路径时把 startpoint、endpoint、发射/捕获时钟、周期、逻辑延迟、布线延迟、required time 和 arrival time 抄出来。setup slack 可从 required − arrival 理解。若 required 为 8.0 ns、arrival 为 9.2 ns，slack 是 −1.2 ns。hold 是最小延迟检查，需按报告的最早到达与最小需求判读。WNS 是最差 slack，TNS 汇总失败端点的负裕量。先检查时钟定义、生成时钟和 I/O 约束，再解释路径为什么长。异步跨域要先理解同步器或 FIFO 结构，再应用与设计意图相符的例外；时序例外是对路径行为的声明。

时序修复按原因选择。组合链长时看能否加流水级并更新功能验证，扇出大时看拓扑或复制，布线占比大时看布局、拥塞和跨区域连接。修改后对比同一个 clock group 的报告，再跑相关功能用例。对于 ASIC，换成标准单元、工艺角和实际寄生参数后还要检查全部工作模式，方法上的起点仍然是核对路径和约束。

把第 4 章 RTL 从仿真交给 Vivado 综合。创建一个新目录，把 `study_vec_regs.sv` 放进去，写入如下两个文件。同一个 DUT 在仿真里需要 testbench 激励，在综合中将 DUT 作为顶层读取。

```tcl
# study.xdc
create_clock -name study_clk -period 8.000 [get_ports clk]
```

```tcl
# synth.tcl
read_verilog -sv study_vec_regs.sv
read_xdc study.xdc
synth_design -top study_vec_regs -part xcku5p-ffvb676-2-i
report_utilization -file utilization.rpt
report_timing_summary -report_unconstrained -file timing.rpt
write_verilog -force study_vec_regs_netlist.v
```

```bash
"$HOME/Downloads/xilinx_software/Vivado/2023.1/bin/vivado" \
  -mode batch -nolog -nojournal -source synth.tcl
```

这组命令已在 VM 的 Vivado 2023.1 综合得到网表、用量和时序报告，样本用量为 43 个 CLB LUT 和 67 个 CLB 寄存器。Setup WNS 为 +6.880 ns，hold 的 WHS 为 −0.068 ns；同时有 I/O 端口缺少外部延迟约束。把这些数字理解为单时钟、综合阶段的教学观测。打开综合网表，从时钟端口找一个触发器，并追到它前面的组合逻辑。接到板卡时需要按真实外部时序填写 I/O delay，布局布线后再读 routed STA。

## 4. 用真实产物了解 ASIC 工具之间怎么交接

在 `$HOME/Downloads/pcie_iip/i_DWC_pcie_ctl_2` 先读 `export/README`，再看 `report/synthesis.html`、`report/fpga_synthesis.html` 和 `syn/Makefile`。历史导出名为 `ku5p_DWC_pcie_ctl`，可看到 `.ddc`、`.v`、`.sdc`。练习时回答三个问题：哪个是工具数据库，哪个是结构化网表，哪个记录时钟和路径约束。区分工具配置、目标技术与输入工艺库。工作区的文件、报告和 Vivado 实现可能来自不同的配置或时间，比较前先核对顶层与目标。

一个小型 ASIC 练习可以拿第 4 章模块做。写包含主时钟、I/O 延迟及真实时钟例外的 SDC，选择获准使用的工艺标准单元库。用综合工具读取、展开、映射并写出 timing/area/netlist，检查未约束或未连接端口。然后用等价工具比对参考 RTL 与综合网表，最后在具备匹配物理库和 PDK 的学习环境练 floorplan、placement、CTS 和 routing。盘点确认了现有 IP 集成和 FPGA 历史产物，完整 ASIC 签核、量产 DFT 和上板联调由各自的实验补齐。

## 5. 六周上手顺序

| 周次 | 实际任务 | 验收件 |
| --- | --- | --- |
| 1 | 学 Linux 的 `pwd/cd/ls/grep/less`，看 Makefile 与路径，复跑链路测试 | 一张环境图和一次可重跑命令，能够从日志确认用例与链路状态 |
| 2 | 跑 cfg、enumeration 和 memory，用 Verdi 追从 TLP 到 AXI 的一笔读 | 手画时序或截图记录，标出地址、tag、数据、一次 AR 和 R 握手 |
| 3 | 完成第 4 章 RTL、正向与故障注入，写寄存器规格 | 自检 PASS 与预期 FAIL 各一份，说明第一次不匹配出现在哪里 |
| 4 | 在练习副本写独立参考函数、绑定断言并扩展 UVM test | 增加溢出、部分写、限时等待和复位场景，每项都有明确观察点 |
| 5 | 建一个带 seed 的回归矩阵并收集覆盖率，关闭一个可解释的缺口 | 需求—测试—检查—覆盖追踪表，保留一个定向命中例子 |
| 6 | 读综合与 routed 报告并追一条时序路径，做一次小模块综合对照 | 解释 WNS、TNS、约束缺口及一个潜在修复，画出 ASIC 后端与签核所需输入 |

完成后自测。你应该能仅凭日志复现一个失败、从请求追到对应的线网和寄存器、写出新的 RTL 行为及其通过标准，并说明这项证据在整个数字 IC 流程中覆盖哪一部分。
