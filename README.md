# 从 Vivado 入门 PCIe 数字 IC 设计与验证

用 CentOS 虚拟机里的 PCIe 工程，走一遍真正的端到端验证：让 Root Complex 模型训练链路、枚举 Endpoint、通过 BAR 写入两个向量、启动计算，再读回并比较结果。另一套 Vivado 工程提供 FPGA 实现和时序报告，可以接上你已经熟悉的工具。

**建议从 `my_test_project` 开始。** 它集成 PCIe 控制器、PHY 模型和向量加法 RTL，再使用 Synopsys PCIe VIP 和 UVM 测试。目录名里虽然有 `gen6_cxl`，这条入门用例的实际配置是单 lane、2.5 GT/s。第一堂课以定位测试、确认链路进入 L0、读懂一笔 BAR 读回为目标。

VM 上已用 VCS 重新编译并复跑链路、配置访问、枚举和向量四个用例，四个结果 DW 全部匹配。原创实验在 VCS 和 Vivado XSim 中都通过，故意改错高字节算术时，XSim 自检报告了首个错误。

切换到 Host-only 后，又从空的构建目录离线完成了 VCS 编译、四个用例、Verdi 的 FSDB 读取和 Vivado XSim 实验。

| 章节 | 学完后能做什么 |
| --- | --- |
| [01 找到虚拟机、工程和工具](docs/01-environment.md) | 区分 Ubuntu 宿主机与 CentOS 虚拟机，找到项目入口、EDA 可执行文件与本地手册 |
| [02 用 VCS 跑仿真，用 Verdi 看波形](docs/02-first-run.md) | 理解 Makefile，运行链路与端到端测试，保留 seed、日志和波形，定位首个失败 |
| [03 顺着 PCIe 事务读源码](docs/03-project-walkthrough.md) | 从 UVM test 追到配置请求、BAR、AXI、寄存器状态机和返回的 Completion |
| [04 从零写一个向量加法外设](docs/04-hands-on.md) | 用原创 RTL 和自检 testbench 练习规格、RTL、编译、仿真与故障注入 |
| [05 把项目接到数字 IC 全流程](docs/05-ic-flow.md) | 写验证计划、做回归和覆盖率，再理解综合、STA、DFT、布局布线与签核 |
| [06 通过 Host-only SSH 使用 EDA](docs/06-host-only-ssh.md) | 理解有线开关、隔离网络、两跳登录与离线仿真的验收方法 |

## 第一遍怎么读

1. 先用第 1 章进入工程，再按第 2 章打开已有日志。找到用例名、`L0`、BAR0 和四个 `VEC_ADD` 检查。
2. 用第 3 章画出一笔 Memory Read 的来回路径，在 Verdi 中找出一次 `ARVALID && ARREADY` 和对应的 `RVALID && RREADY`。
3. 做第 4 章实验并故意引入一个算术错误。确认 testbench 报出预期与实测差异，再恢复正确设计。
4. 用第 5 章选一个场景，比如 busy 太短、复位中断或 AXI 背压。从规格写到测试、检查和覆盖率，再读已有的综合与时序产物。

仓库内是学习文档与原创实验。真实工程源码、商业 IP、工具和运行产物位于虚拟机。
