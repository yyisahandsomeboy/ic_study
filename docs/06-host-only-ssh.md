# 06 通过 Host-only SSH 使用 EDA

## 有线开关控制的是什么

CentOS 桌面上的「有线已连接」开关控制客户机里的网卡连接。关掉它以后，CentOS 内核、桌面、本地文件和本机进程可以继续运行，VMware 控制台仍可使用。这台 VM 只有一张对外的 VMware 虚拟网卡，关掉后经这张网卡进入的 SSH 也会中断。依赖远程文件或许可证的 EDA 操作要以实际运行验证。

VMware 的 Host-only 模式让这张网卡保持连接，使用只在宿主机上出现的私有网段。Mac 的 SSH 客户端先连接 Ubuntu 宿主机，由 Ubuntu 建立到 CentOS SSH 的第二段连接。CentOS 只需要回复该连接，就能通过终端接收命令和返回输出。访问互联网的本机工具和 AI 客户端仍运行在 Mac 或宿主机。原理见 [VMware Workstation 网络连接方式](https://knowledge.broadcom.com/external/article?articleNumber=309842)。

```text
Mac / AI ── Tailscale + SSH ── Ubuntu 宿主机 ── VMnet1 + SSH ── CentOS
                                   │
                                   └── 宿主机自己的网络
```

## 本机连接方式

Tailscale 已连通宿主机时，在已配置专用 SSH 密钥和别名的 Mac 上执行下面的命令。

```bash
ssh ic-eda
# 非交互检查
ssh -o BatchMode=yes ic-eda 'id -un; cat /etc/centos-release'
```

同一个别名也可用于 `scp`。本次已从 Mac 上传并下载一份临时测试文件，两端内容一致。

进入后，用下面的命令定位 VCS 和 Verdi。若一行式 SSH 命令的环境不同，可在 VM 中打开 bash 登录 shell。

```bash
bash -l
type -P vcs
type -P verdi
cd "$HOME/pcie_gen6_cxl_proj/designware_home/pcie_dev/my_test_project/sim"
ls -lh Makefile sim.log pcie.fsdb
```

专用公钥安装到普通 EDA 账号上，限制连接来源为宿主机在 Host-only 网段的地址，并禁用了该密钥的 SSH 端口、代理和 X11 转发。交互式终端仍可使用。打开 Verdi 图形界面时，使用 VMware 控制台里的 CentOS 桌面。[OpenSSH 密钥选项](https://man.openbsd.org/sshd#AUTHORIZED_KEYS_FILE_FORMAT)

## 怎么核对隔离是否生效

在 VMware 的网卡设置确认模式为 **Host-only**、虚拟网卡 **Connected**、开机自动连接已启用。当前 VM 只有这一张 VMware 网卡，已从 NAT 切换到 VMnet1。客户机的 NetworkManager 使用一个自动连接的静态 IPv4 配置，网关和 DNS 留空。客户机 `ens33` 的 IPv6 已关闭，防火墙仅允许宿主机发起 TCP 22 连接，并阻止客户机通过该网卡新建出站连接。Ubuntu 上的 IPv4、IPv6 全局和 VMnet1 转发已实测处于关闭状态。

在宿主机检查全局与接口的转发开关。四项的预期值都是 `0`。

```bash
sysctl net.ipv4.ip_forward net.ipv6.conf.all.forwarding \
       net.ipv4.conf.vmnet1.forwarding net.ipv6.conf.vmnet1.forwarding
```

在客户机中确认 `ens33` 使用 `ic-hostonly`，IPv4 和 IPv6 均无默认路由。CentOS 7 的 `ip` 可能需要用绝对路径运行。

```bash
nmcli -t -f NAME,DEVICE connection show --active
/usr/sbin/ip -4 addr show dev ens33
/usr/sbin/ip -4 route show default
/usr/sbin/ip -6 route show default
cat /proc/sys/net/ipv6/conf/ens33/disable_ipv6
```

本次从 Mac 经 Ubuntu 成功 SSH 到 CentOS 后，已实际测试：客户机到公网 IPv4 和 IPv6 的 TCP 连接均失败，公网 DNS 查询失败，也无法路由到宿主机的物理局域网。客户机主动连接宿主机的 SSH 端口也被拒绝。设备仍在线，因此可以在同一个 SSH 会话里编译和运行仿真。

若后续 SSH 不通，按顺序确认 Mac 到宿主机的 Tailscale 连接、VMware 网卡连接、`ic-hostonly` 连接配置和 `sshd` 状态。若已经把 CentOS 的「有线」关掉，从 VMware 控制台重新打开即可恢复网卡，再检查连上的配置。

## 离线 EDA 验收

2026 年 11 月 25 日，从 Mac 通过 `ssh ic-eda` 在已隔离的 CentOS 上执行。VCS 主工程从空的输出目录重新编译，生成新的 `simv`，再用固定 seed `20261125` 运行四个用例。

| 检查 | 实际结果 |
| --- | --- |
| VCS V-2023.12-SP2 重新编译 | 成功生成 `simv` |
| `pcie_link_up_test`、`pcie_cfg_pkg_test`、`pcie_enumeration_device_test` | 全部正常退出，各有 1 条 UVM warning、0 error、0 fatal |
| `pcie_mem_pkg_test` | 正常退出，1 条 UVM warning、0 error、0 fatal，四个结果 DW 全部匹配 |
| Verdi 附带的 `fsdb2vcd` | 读取本次生成的 FSDB，导出首笔结果读回的波形片段 |
| Vivado XSim 2023.1 | 从源码编译原创小实验，仿真输出 `PASS: study_vec_regs` |

复做 VCS 编译和测试可直接按 [第 2 章](02-first-run.md)，XSim 小实验见 [第 4 章](04-hands-on.md)。这些结果证明当前 VM 配置下对应的工具与用例可以离线运行。其他商业工具或工程可能依赖不同的许可证与文件服务，应在隔离环境里分别验收。
