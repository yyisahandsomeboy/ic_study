# 01 找到虚拟机、工程和工具

## 先分清三个环境

| 位置 | 主要操作 | 怎么确认 |
| --- | --- | --- |
| 本机 | SSH 到宿主机，浏览 GitHub 笔记 | 登录前运行 `hostname` |
| Ubuntu 宿主机 | 运行 VMware Workstation、查看哪台虚拟机正在运行 | `hostname; vmrun -T ws list` |
| CentOS 7.9 客户机 | 看 RTL、运行 VCS、打开 Verdi 和 Vivado | `id -un; echo "$HOME"; cat /etc/centos-release` |

在本机使用已配置的宿主机 SSH 别名。如果直连超时且宿主机经 Tailscale 可达，可使用当前机器已安装的 `tailscale nc` 作为 SSH 的传输。

```bash
ssh YOUR_HOST_ALIAS
# 适用于通过 Tailscale 连接且 SSH 已配好的机器
tailscale status
# 如果显示 Stopped，先在本机终端运行 tailscale up
ssh -o 'ProxyCommand=tailscale nc %h %p' YOUR_HOST_ALIAS
```

在宿主机查 VMware 当前登记的 VM。把命令里的示例路径换成上一条命令输出的实际 `.vmx` 路径。可以用 VMware 控制台登录，也可以在宿主机用已有的客户机 SSH 授权连接。

```bash
vmrun -T ws list
vmrun -T ws getGuestIPAddress /path/to/JJT_IC2022.vmx
ssh YOUR_EDA_USER@YOUR_GUEST_IP
```

本次检查确认正在运行的 VM 显示名为 `serdes21_digital`，客户机版本为 CentOS 7.9.2009。下文的 `$HOME` 指客户机普通 EDA 账号的主目录。在 Ubuntu 或 root 的终端里执行会展开为另一个路径。日常仿真和练习都使用普通 EDA 账号。

已配置 Host-only SSH 的 Mac 可以直接运行 `ssh ic-eda`。登录链路和隔离检查见 [第 6 章](06-host-only-ssh.md)。

## 在 CentOS 打开工程

```bash
export DESIGNWARE_HOME="$HOME/pcie_gen6_cxl_proj/designware_home"
export PCIE="$DESIGNWARE_HOME/pcie_dev"
export PROJECT="$PCIE/my_test_project"

cd "$PROJECT"
pwd
ls -d rtl tb tc sim
find tb tc -maxdepth 2 -type f -print | sort
```

| 客户机路径 | 实际内容 | 学习用途 |
| --- | --- | --- |
| `$PCIE/my_test_project` | Endpoint RTL、Xilinx PHY/原语模型、AXI/AXI-Lite 应用逻辑、UVM 测试 | 主线。先学链接、枚举和向量读回 |
| `$PCIE/my_test_project_generic` | Generic PHY 目录、另一份 `pcie_ep.sv`、类似的 TB 和历史产物 | 对照 PHY 与应用接口。其 filelist 有指向 `my_test_project/rtl` 的硬编码路径，核对生效源码后再比较 |
| `$PCIE/examples/sverilog/pcie_svt/tb_pcie_svt_uvm_unified_vip_sys` | VIP 自带示例，包含 PIPE、SerDes 等拓扑，以及独立的 Gen6 用例文件 | 看厂商如何搭建和配置 VIP。其结果应单独记录 |
| `$HOME/Downloads/pcie_iip/i_DWC_pcie_ctl_2` | IP 配置/集成工作区，含 `examples/`、`sim/`、`syn/`、`spyglass/`、`upf/`、`vclp/`、`export/`、`report/` | 学习 IP 配置、交付件、综合与低功耗分析产物 |
| `$HOME/Downloads/pcie_gpgpu2/project_1/project_1.xpr` | Vivado 工程，顶层 `pcie_ep`，器件 `xcku5p-ffvb676-2-i`，有历史综合、布线及 bitstream 产物 | 从熟悉的 GUI 对照综合、时序约束、实现和时序收敛 |

```text
my_test_project/
├── rtl/
│   ├── pcie_ep.sv            Endpoint wrapper、AXI 桥和向量加法
│   ├── pcie_rtl.f            VCS 设计文件列表
│   ├── src/Phy/xilinx/       Xilinx PHY 仿真模型
│   ├── axi/                  AXI 与 AXI-Lite 通用组件
│   └── common_cells/         AXI 组件依赖的通用逻辑
├── tb/
│   ├── tb_top.sv             HDL 顶层与 FSDB 输出
│   ├── dut_instance/         DUT 接线
│   ├── vip_instance/         VIP 的 SerDes 接线
│   ├── pcie_vip/             UVM 环境与 RC 配置
│   └── seq_lib/              枚举和 Memory 访问序列
├── tc/                       四个可实际运行的 UVM 测试与基类
└── sim/                      Makefile、日志、波形、simv
```

主工程根下，`rtl/` 装设计，`tb/` 装测试平台的连接、环境和激励，`tc/` 放可选测试类，`sim/` 放 Makefile 与仿真产物。Vivado 在 `.xpr` 记录源文件，VCS 实际要编译的内容由 Makefile、filelist、include 链和库选项共同决定。同名文件或备份出现在磁盘上时，以各自的编译入口和日志确认它是否生效。已核对三份 `pcie_ep.sv` 的文件哈希，Vivado 工程使用自己的导入副本，和两份 VCS 工程目录中的文件内容均有差异。

## 工具已经在哪里

打开 CentOS 的 bash 终端。这个 VM 的账号启动文件配置了常用 Synopsys 路径。普通终端可直接检查 `PATH`。某些 SSH 非交互 shell 的初始化不同，按需开一个登录 shell（`bash -l`），再用 `type -P` 找实际可执行文件。

```bash
for tool in vcs vlogan verdi dve urg dc_shell fm_shell spyglass; do
  type -P "$tool" || true
done
printf 'VCS_HOME=%s\nVERDI_HOME=%s\n' "${VCS_HOME:-}" "${VERDI_HOME:-}"
```

| 工具 | 客户机位置（相对 `$HOME`） | 在本教程里做什么 |
| --- | --- | --- |
| VCS、vlogan、DVE、URG | `install/synopsys/vcs/vcs/V-2023.12-SP2/bin/` | VCS 编译和展开，`simv` 跑动态仿真。DVE 查看仿真，URG 可整理 VCS 覆盖率数据库 |
| Verdi | `install/synopsys/verdi/verdi/V-2023.12-SP2/bin/verdi` | 读取 FSDB，按层级、波形及源码定位问题 |
| Design Compiler | `install/synopsys/dc/syn/T-2022.03-SP2/bin/dc_shell` | 阅读综合脚本，对照映射网表和 timing/area 报告 |
| Formality | `install/synopsys/fm/fm/T-2022.03/bin/fm_shell` | 学习 RTL 与网表的逻辑等价检查 |
| SpyGlass | `install/synopsys/spyglass/spyglass/T-2022.06-1/SPYGLASS_HOME/bin/spyglass` | 阅读 lint、CDC 检查报告和 waiver |
| Vivado、XSim | `Downloads/xilinx_software/Vivado/2023.1/bin/` | FPGA 工程和教学实验的仿真 |

盘点确认上述入口文件存在。PrimeTime、独立安装的 ICC2、StarRC 在启动文件中有路径声明，声明指向的安装子目录未找到。DC 树下存在 `icc2_shell` 同名入口，是否可独立运行需另行验证。工具能在 `PATH` 中被找到、能够启动、能够取得运行授权、能够跑指定工程，是四个可分别检查的条件。工具用途参见 [Synopsys 验证工具](https://www.synopsys.com/verification.html) 和 [AMD Vivado 逻辑仿真文档](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation)。

VM 桌面上有 `vcs_doc/`、`verdi_doc/` 和 `pcie_svt_doc/`。当命令或界面和网上教程不同时，先看本机对应版本的帮助。PCIe VIP 可从本地 Getting Started 和 UVM User Guide 入手。DesignWare IP 工作区的 `doc/` 中有 Endpoint 数据手册、寄存器说明与用户指南，适合查从 wrapper 连出的接口。

[下一章：跑出第一条端到端仿真](02-first-run.md)
