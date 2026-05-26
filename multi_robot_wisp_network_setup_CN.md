# 服务器 SSH 控制两台 Unitree GO2 的通用 WISP / 无线中继网络配置教程

近期在开展多机器人导航相关真机实验时，主包发现 Unitree GO2 本体并不自带可直接用于实验组网的无线网卡。如果完全依赖有线连接，不仅现场布线复杂，也不利于后续 demo 展示和多机器人协同实验。目前公开资料中较少有面向多台 GO2 真机组网的完整教程。从部分相关工作和实验系统配图来看，常见方案主要包括**机载路由器**和**机载主机**两种。

本项目主要围绕第一种方案，即通过**机载路由器**实现多台 GO2 接入同一局域网，并进一步实现服务器对多台机器人进行 SSH 连接与基本通信测试。该方案尽量不修改 GO2 本体的永久网络配置，而是通过临时 IP 的方式完成调试，整体风险较低，便于复现和扩展。对于**机载主机**方案，思路相对直接：为每台机器人外接一台具备无线网卡的小型主机，使其与服务器处于同一局域网内即可。本项目不对该方案展开讨论。

本教程具有一定通用性，不局限于特定品牌路由器和机器人。对于其他支持 WISP / 无线中继 / 万能中继 / Client+AP / 无线桥接功能的路由器，或其他单机器人、多机器人平台，可根据实际网络环境调整对应 IP 地址和设备名称。由于个人经验有限，文中若有不妥之处，欢迎交流与批评指正。

> 适用对象：上层主路由器 + 两台 GO2 机载随身路由器。机载路由器需要支持 **WISP / 无线中继 / 万能中继 / Client+AP / 无线桥接** 中的一种或多种功能。  
> 核心原则：**机载路由器只负责把 GO2 接入 BASE 网络；真正 SSH 的目标不是机载路由器 IP，而是给 GO2 本体临时添加的 `192.168.0.x` 管理 IP。**

---

## 1. 硬件配置与网络目标

### 1.1 设备选型

本方案需要：

```text
1. 上层主路由器 BASE
2. GO2-1 机载随身路由器
3. GO2-2 机载随身路由器
4. 服务器一台
5. 每台 GO2 与对应机载路由器之间的网线
6. 可选：USB 有线网卡，用于初始化阶段临时直连 GO2 所在 LAN
```

路由器要求：

```text
1. 支持 WISP / 无线中继 / 万能中继 / Client+AP / 无线桥接功能，能作为无线客户端连接 BASE；
2. 能通过 LAN 口给 GO2 提供网络接入；
3. 尽量支持关闭 AP 隔离 / 客户端隔离。
```

本次实际硬件示例：

```text
* BASE 主路由器：腾达 AX3000 Ultra
* GO2-1 机载路由器：腾达 AX3000 Ultra
* GO2-2 机载路由器：腾达 AX3000 Ultra
* 服务器 Wi-Fi 示例 IP：192.168.0.132
* 路由器供电：宇树自带的扩展坞提供5V/3A、12V/3A两个充电口，尺寸为XT30，可为路由器进行供电。可根据路由器充电接头型号自行选择供电线。 
```

注意：不同厂商对模式命名不同。优先选择能让下挂设备进入 BASE 同一网段的模式，例如：

```text
无线中继、万能中继、Client+AP、无线桥接、WISP 中继
```

如果某些路由器的 WISP 是独立 NAT 路由模式，则不是本教程的首选模式。NAT/WISP 路由模式需要端口转发，流程会更复杂。

---

### 1.2 推荐网络拓扑

```text
BASE 主路由器
IP：192.168.0.1
网段：192.168.0.0/24

├── 服务器
│   └── 示例 IP：192.168.0.132
│
├── GO2-1 机载路由器
│   ├── 工作模式：WISP / 万能中继 / Client+AP
│   ├── 上级 Wi-Fi：BASE
│   ├── 管理 IP：192.168.0.135（以实际绑定为准）
│   └── GO2-1 本体 eth0
│       ├── 原始 IP：192.168.123.18/24
│       └── 临时 SSH IP：192.168.0.211/24
│
└── GO2-2 机载路由器
    ├── 工作模式：WISP / 万能中继 / Client+AP
    ├── 上级 Wi-Fi：BASE
    ├── 管理 IP：192.168.0.105（以实际绑定为准）
    └── GO2-2 本体 eth0
        ├── 原始 IP：192.168.123.18/24
        └── 临时 SSH IP：192.168.0.212/24
```

地址规划表：

| 设备 | 用途 | 推荐 / 示例 IP |
|---|---|---|
| BASE 主路由器 | 网关 | `192.168.0.1` |
| 服务器 | 控制端 | `192.168.0.132`，以实际为准 |
| GO2-1 机载路由器 | 路由器后台管理 | `192.168.0.135`，以实际绑定为准 |
| GO2-2 机载路由器 | 路由器后台管理 | `192.168.0.105`，以实际绑定为准 |
| GO2-1 本体 | SSH 管理 | `192.168.0.211` |
| GO2-2 本体 | SSH 管理 | `192.168.0.212` |
| GO2 默认地址 | 临时初始化入口 | `192.168.123.18` |

最终 SSH 目标：

```bash
ssh unitree@192.168.0.211
ssh unitree@192.168.0.212
```

不要把机载路由器后台地址当成 GO2 本体地址：

```text
192.168.0.135  → GO2-1 机载路由器后台
192.168.0.105  → GO2-2 机载路由器后台
192.168.0.211  → GO2-1 本体 SSH
192.168.0.212  → GO2-2 本体 SSH
```

---

### 1.3 物理连接方式

每台 GO2 接到自己的机载路由器 LAN 口：

```text
GO2-1 eth0  ←网线→  GO2-1 机载路由器 LAN 口
GO2-2 eth0  ←网线→  GO2-2 机载路由器 LAN 口
```

注意：

```text
1. GO2 接 LAN 口，不要接 WAN 口；
2. 机载路由器通过 WISP / 万能中继连接 BASE；
3. GO2 默认 IP 192.168.123.18 保持不变；
4. 不建议修改 GO2 的 netplan / systemd-networkd / NetworkManager 永久配置；
5. 给 GO2 添加的 192.168.0.211 / 192.168.0.212 是临时 IP，重启后会消失。
```

---

## 2. 基本网络配置流程

### 2.1 配置 BASE 主路由器

进入 BASE 后台，配置局域网：

```text
LAN IP：192.168.0.1
子网掩码：255.255.255.0
DHCP：开启
DHCP 地址池：192.168.0.100 - 192.168.0.200
```

建议：

```text
1. 服务器尽量通过有线或高质量 Wi-Fi 接入 BASE；
2. BASE 不要开启会阻断局域网互访的 AP 隔离 / 客户端隔离；
3. DHCP 地址池避免覆盖 GO2 本体手动管理 IP，例如 192.168.0.211 / 192.168.0.212；
4. 如果必须覆盖，需要确保这些地址不会被分配给其它设备。
```

---

### 2.2 配置机载路由器为 WISP / 万能中继模式

每台机载路由器单独配置一次。

通用步骤：

```text
1. 进入机载路由器后台；
2. 切换工作模式为 WISP / 万能中继 / Client+AP / 无线桥接；
3. 选择上级 Wi-Fi：BASE；
4. 输入 BASE 密码；
5. 保存并等待路由器重启；
6. 在 BASE 后台确认该机载路由器已上线。
```

建议将机载路由器自己广播的 Wi-Fi 改名为：

```text
GO2-1
GO2-2
```

不要也叫 `BASE`，否则服务器可能连接到错误 AP，导致链路不稳定。

如果路由器提供以下选项，建议：

```text
关闭访客网络
关闭 AP 隔离 / 客户端隔离
关闭智能漫游 / 双频自动切换类功能
```

---

### 2.3 在 BASE 后台固定机载路由器管理 IP

在 BASE 后台进入类似页面：

```text
网络设置 → 局域网设置 → 静态 IP 分配列表
```

为机载路由器绑定管理 IP。

示例：

```text
GO2-1：
IP：192.168.0.135
MAC：以 BASE 设备管理页面看到的 MAC 为准

GO2-2：
IP：192.168.0.105
MAC：以 BASE 设备管理页面看到的 MAC 为准
```

重要原则：

```text
在 BASE 中固定 IP 时，使用 BASE 设备管理页面看到的 MAC；
不要强行使用机载路由器后台显示的 LAN MAC / 2.4G MAC / 5G MAC。
```

原因：WISP / 万能中继模式下，路由器可能使用虚拟 MAC、代理 MAC 或无线客户端 MAC 接入上级路由。因此 BASE 看到的 MAC 与机载路由器后台显示的 MAC 不一致是正常现象。

如果静态绑定某个新 IP 后不生效，可以直接绑定当前已经稳定可访问的 IP。机载路由器管理 IP 只用于进入后台，不是 SSH 目标。

---

### 2.4 检查服务器到 BASE 的链路质量

服务器连接 BASE 后，先测试到 BASE 的质量：

```bash
ip -br addr
ip route get 192.168.0.1
ping -c 20 192.168.0.1
```

如果服务器通过 Wi-Fi 接入 BASE，再检查：

```bash
iw dev wlp8s0 link
nmcli -f IN-USE,SSID,BSSID,SIGNAL,CHAN dev wifi list
```

理想状态：

```text
ping 192.168.0.1：0% packet loss
平均延迟：最好 < 10 ms，至少 < 30 ms
Wi-Fi signal：建议优于 -60 dBm
tx/rx bitrate：不应长期只有 6 MBit/s
```

如果连 `192.168.0.1` 都丢包或延迟几百毫秒以上，先解决服务器到 BASE 的链路，不要继续调 SSH。

---

## 3. 给两台 GO2 本体添加临时 SSH 管理 IP

两台 GO2 默认都是：

```text
192.168.123.18
```

因此初始化时建议一次只配置一台 GO2，避免 IP 冲突。

目标：

```text
GO2-1 添加：192.168.0.211/24
GO2-2 添加：192.168.0.212/24
```

---

### 3.1 方案 A：服务器临时连接对应机载路由器 Wi-Fi 完成初始化

适用场景：

```text
没有 USB 网卡，或不方便接线；
只需要临时进入某一台 GO2；
机载路由器 Wi-Fi 信号较好。
```

以 GO2-2 为例。

#### 3.1.1 连接 GO2-2 Wi-Fi

```bash
nmcli -f IN-USE,SSID,BSSID,SIGNAL,CHAN dev wifi list
nmcli dev wifi connect "GO2-2" password "你的GO2-2 WiFi密码"
nmcli -t -f ACTIVE,SSID dev wifi | grep '^yes'
```

确认输出为：

```text
yes:GO2-2
```

#### 3.1.2 给服务器 Wi-Fi 网卡添加临时地址

```bash
sudo ip addr add 192.168.123.10/24 dev wlp8s0
ip -4 addr show dev wlp8s0
```

应看到：

```text
192.168.123.10/24
```

如果提示：

```text
RTNETLINK answers: File exists
```

说明已经添加过，可以忽略。

#### 3.1.3 设置访问 GO2 默认网段的路由

```bash
sudo ip route replace 192.168.123.0/24 dev wlp8s0 src 192.168.123.10 metric 5
sudo ip route flush cache
ip route get 192.168.123.18
```

期望：

```text
192.168.123.18 dev wlp8s0 src 192.168.123.10
```

#### 3.1.4 确认访问到 GO2 本体

```bash
ping -I wlp8s0 -c 5 192.168.123.18
ip neigh show 192.168.123.18 dev wlp8s0
```

判断：

```text
如果 MAC 是 3c:6d:66:xx:xx:xx 等 GO2 本体 MAC，说明正确；
如果 MAC 是机载路由器 MAC，说明访问对象错误。
```

#### 3.1.5 登录 GO2 并添加临时 IP

如果是 GO2-1：

```bash
ssh unitree@192.168.123.18
sudo ip addr add 192.168.0.211/24 dev eth0
ip -4 addr show eth0
```

如果是 GO2-2：

```bash
ssh unitree@192.168.123.18
sudo ip addr add 192.168.0.212/24 dev eth0
ip -4 addr show eth0
```

应看到：

```text
192.168.123.18/24
192.168.0.211/24 或 192.168.0.212/24
```

#### 3.1.6 切回 BASE

退出 GO2 后：

```bash
exit
nmcli dev wifi connect "BASE" password "你的BASE密码"
nmcli -t -f ACTIVE,SSID dev wifi | grep '^yes'
```

测试最终 IP：

```bash
ping -c 10 192.168.0.211
ping -c 10 192.168.0.212
```

---

### 3.2 方案 B：服务器使用网线临时接入机载路由器 LAN 口

适用场景：

```text
推荐方案；
Wi-Fi 初始化不稳定；
希望服务器保持连接 BASE，同时用 USB 网卡进入 GO2 默认网段。
```

物理连接：

```text
服务器 USB 网卡  ←网线→  机载路由器 LAN 口
GO2 eth0         ←网线→  同一个机载路由器 LAN 口
```

识别服务器网卡：

```bash
ip -br link
ip -br addr
nmcli device status
```

示例：

```text
enp7s0           10.11.12.6/23       → 光纤/稳定有线，不要动
wlp8s0           192.168.0.132/24     → BASE Wi-Fi
enx00e04c6802ba  无 IPv4              → 临时调试 USB 网卡
```

配置 USB 网卡：

```bash
sudo ip link set enx00e04c6802ba up
sudo ip addr add 192.168.123.10/24 dev enx00e04c6802ba
sudo ip route replace 192.168.123.0/24 dev enx00e04c6802ba src 192.168.123.10 metric 5
sudo ip route flush cache
ip route get 192.168.123.18
```

期望：

```text
192.168.123.18 dev enx00e04c6802ba src 192.168.123.10
```

测试：

```bash
ping -I enx00e04c6802ba -c 10 192.168.123.18
ip neigh show 192.168.123.18 dev enx00e04c6802ba
```

正常情况下：

```text
ping 延迟约 0.5 - 2 ms
MAC 应为 GO2 本体 MAC
```

登录并添加临时 IP：

```bash
ssh unitree@192.168.123.18
```

GO2-1：

```bash
sudo ip addr add 192.168.0.211/24 dev eth0
ip -4 addr show eth0
```

GO2-2：

```bash
sudo ip addr add 192.168.0.212/24 dev eth0
ip -4 addr show eth0
```

---

### 3.3 GO2 重启后的重新添加方法

本方案不修改 GO2 永久配置。GO2 重启后临时 IP 会消失。

GO2-1 重启后重新执行：

```bash
sudo ip addr add 192.168.0.211/24 dev eth0
```

GO2-2 重启后重新执行：

```bash
sudo ip addr add 192.168.0.212/24 dev eth0
```

如果提示：

```text
RTNETLINK answers: File exists
```

说明 IP 已存在，不需要重复添加。

---

## 4. 服务器同时 SSH 控制两台 GO2

### 4.1 最终 SSH 连接方式

正式运行阶段，服务器连接 BASE 网络，同时控制两台 GO2：

```bash
ssh unitree@192.168.0.211
ssh unitree@192.168.0.212
```

不要长期使用：

```bash
ssh unitree@192.168.123.18
```

因为两台 GO2 的默认地址相同。

---

### 4.2 配置服务器 `~/.ssh/config`

编辑：

```bash
nano ~/.ssh/config
```

加入：

```sshconfig
Host go2-1
    HostName 192.168.0.211
    User unitree
    Port 22
    HostKeyAlias go2-1
    ServerAliveInterval 10
    ServerAliveCountMax 3

Host go2-2
    HostName 192.168.0.212
    User unitree
    Port 22
    HostKeyAlias go2-2
    ServerAliveInterval 10
    ServerAliveCountMax 3
```

之后：

```bash
ssh go2-1
ssh go2-2
```

Cursor / VS Code Remote SSH 也可以选择 `go2-1` 或 `go2-2`。

---

### 4.3 解决 SSH host key 冲突

现象：

```text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
Host key for 192.168.123.18 has changed
```

原因：

```text
GO2-1 和 GO2-2 默认都叫 192.168.123.18，但它们是两台不同机器，SSH host key 不同。在配置时服务器之前记住了其中一台的 key，再连接另一台时会触发告警。
```

初始化阶段可清理旧记录：

```bash
ssh-keygen -f "/home/maker002/.ssh/known_hosts" -R "192.168.123.18"
```

或者使用通用写法：

```bash
ssh-keygen -f ~/.ssh/known_hosts -R "192.168.123.18"
```

最终使用 `192.168.0.211 / 192.168.0.212` 后，不再依赖重复的 `192.168.123.18`，不会影响同时 SSH。

---

## 5. ROS2 多机器人通讯配置

> 本部分为推荐配置思路，需根据实际 Unitree ROS2 驱动和 launch 文件进一步验证。

### 5.1 三端互通检查：服务器、GO2-1、GO2-2

在服务器上：

```bash
ping -c 5 192.168.0.211
ping -c 5 192.168.0.212
```

在 GO2-1 上：

```bash
ping -c 5 192.168.0.132
ping -c 5 192.168.0.212
```

在 GO2-2 上：

```bash
ping -c 5 192.168.0.132
ping -c 5 192.168.0.211
```

三端均能稳定互通后，再测试 ROS2。

---

### 5.2 ROS2 基本环境变量设置

三端都 source ROS2 环境。例如 Foxy：

```bash
source /opt/ros/foxy/setup.bash
```

如果有工作空间：

```bash
source ~/your_ws/install/setup.bash
```

---

### 5.3 `ROS_DOMAIN_ID` 与 `ROS_LOCALHOST_ONLY`

三端使用相同 Domain ID：

```bash
export ROS_DOMAIN_ID=0
export ROS_LOCALHOST_ONLY=0
```

可写入 `~/.bashrc`：

```bash
echo 'export ROS_DOMAIN_ID=0' >> ~/.bashrc
echo 'export ROS_LOCALHOST_ONLY=0' >> ~/.bashrc
```

检查：

```bash
echo $ROS_DOMAIN_ID
echo $ROS_LOCALHOST_ONLY
```

---

### 5.4 多机器人 namespace / topic / node / tf 命名建议

多机器人 ROS2 最容易冲突的是：

```text
node name
topic name
service name
tf frame
robot_id
```

建议为两台 GO2 分配命名空间：

```text
GO2-1：/go2_1
GO2-2：/go2_2
```

示例：

```bash
export ROS_NAMESPACE=/go2_1
```

```bash
export ROS_NAMESPACE=/go2_2
```

如果 launch 文件支持参数，优先使用：

```text
namespace
robot_name
robot_id
frame_prefix
```

如果 `ros2 topic list` 看不到对方，但 ping 和 SSH 正常，可能是 DDS multicast 被无线中继阻断。可考虑后续配置 CycloneDDS / FastDDS 的 unicast discovery 或 discovery server。

---

## 6. 报错与问题解决方案

### Q1：BASE 静态 IP 分配后，机载路由器 IP 没有变化怎么办？

**Answer：**

常见原因：

```text
1. 机载路由器仍持有旧 DHCP 租约；
2. 静态分配后没有点击保存；
3. 绑定的 IP 不在 DHCP 地址池内，部分路由器不实际下发；
4. 绑定的 MAC 不是 BASE 看到的 MAC。
```

处理：

```text
1. 使用 BASE 设备管理页面看到的 MAC；
2. 点击保存；
3. 重启机载路由器；
4. 必要时重启 BASE 后再重启机载路由器；
5. 如果仍不生效，直接绑定当前已稳定在线的 IP。
```

机载路由器管理 IP 只用于后台管理，不是 SSH 目标。

---

### Q2：BASE 后台 MAC 与机载路由器后台 MAC 不一致，是否正常？

**Answer：**

正常。机载路由器有多个 MAC：

```text
LAN MAC
2.4G Wi-Fi MAC
5G Wi-Fi MAC
WISP / 中继虚拟 MAC
```

在 BASE 静态 IP 分配中，应使用 **BASE 看到的 MAC**。

---

### Q3：服务器 ping BASE 都丢包，SSH GO2 卡住怎么办？

**Answer：**

先不要调 GO2。执行：

```bash
ping -c 20 192.168.0.1
iw dev wlp8s0 link
nmcli -f IN-USE,SSID,BSSID,SIGNAL,CHAN dev wifi list
```

如果 `ping 192.168.0.1` 都丢包或延迟很高，说明服务器到 BASE 链路不稳定。

解决：

```text
1. 优先让服务器有线连接 BASE；
2. 或移动服务器 / BASE，增强 Wi-Fi 信号；
3. 不要让机载路由器 Wi-Fi 也叫 BASE；
4. 确认服务器连接的是正确的 BASE BSSID。
```

---

### Q4：服务器连接到了错误 AP 或同名 Wi-Fi 怎么办？

**Answer：**

扫描：

```bash
nmcli -f IN-USE,SSID,BSSID,SIGNAL,CHAN dev wifi list
```

如果存在多个同名 `BASE`，应：

```text
1. 将 GO2-1 / GO2-2 机载路由器 Wi-Fi 改名；
2. 服务器只连接真正的 BASE；
3. 必要时按 BSSID 指定连接对象。
```

---

### Q5：`ssh unitree@192.168.123.18` 出现 host key changed 怎么办？

**Answer：**

这是因为两台 GO2 默认 IP 相同，但 host key 不同。

清理：

```bash
ssh-keygen -f ~/.ssh/known_hosts -R "192.168.123.18"
```

然后重新连接并输入 `yes`。

正式运行阶段使用：

```bash
ssh go2-1
ssh go2-2
```

不会再依赖重复 IP。

---

### Q6：`ssh` 卡在 `Connection timed out during banner exchange` 怎么办？

**Answer：**

先测试：

```bash
nc -vz -w 5 192.168.123.18 22
timeout 15 nc -v 192.168.123.18 22
```

如果 `nc -v` 能看到：

```text
SSH-2.0-OpenSSH_xxx
```

再 SSH。

如果看不到，检查是否打到了 GO2 本体：

```bash
ip route get 192.168.123.18
ping -I <网卡> -c 10 192.168.123.18
ip neigh show 192.168.123.18 dev <网卡>
```

---

### Q7：`ping` 能通但 SSH 卡住怎么办？

**Answer：**

看延迟和丢包。如果延迟几百毫秒甚至几秒，或丢包明显，SSH 可能会卡住。

建议重启BASE网络看看能否修复网络质量，再 SSH。

---

### Q8：`ip route replace` 出现 `Invalid prefsrc address` 怎么办？

**Answer：**

原因是指定的源地址还没有添加到该网卡。

错误示例：

```bash
sudo ip route replace 192.168.123.0/24 dev wlp8s0 src 192.168.123.10 metric 5
Error: Invalid prefsrc address.
```

正确顺序：

```bash
sudo ip addr add 192.168.123.10/24 dev wlp8s0
ip -4 addr show dev wlp8s0
sudo ip route replace 192.168.123.0/24 dev wlp8s0 src 192.168.123.10 metric 5
```

---

### Q9：访问 `192.168.123.18` 时 ARP 指向机载路由器 MAC 怎么办？

**Answer：**

说明没有打到 GO2 本体。

检查：

```bash
ip neigh show 192.168.123.18 dev <网卡>
```

如果 MAC 是机载路由器 MAC，应检查：

```text
1. GO2 是否接 LAN 口；
2. 服务器是否接到正确 LAN；
3. 是否误走了 BASE / 中继路径；
4. 是否有另一台设备也在用 192.168.123.18。
```

---

### Q10：误把机载路由器切到普通路由 / NAT 模式怎么办？

**Answer：**

恢复方法：

```text
1. 服务器或电脑直连机载路由器 LAN 口；
2. 用 ip route 查看默认网关；
3. 浏览器访问默认网关进入后台；
4. 切回 WISP / 万能中继 / Client+AP；
5. 重新连接 BASE。
```

---

### Q11：能否在“静态路由”页面固定设备 IP？

**Answer：**

不能。

固定设备 IP 应进入：

```text
网络设置 → 局域网设置 → 静态 IP 分配列表
```

不要在“静态路由”页面添加 `192.168.0.211`、`192.168.123.18` 等规则。

---

### Q12：两台 GO2 默认 IP 都是 `192.168.123.18`，是否影响最终同时控制？

**Answer：**

不影响最终方案。`192.168.123.18` 只用于初始化。

正式运行使用：

```text
GO2-1：192.168.0.211
GO2-2：192.168.0.212
```

---

### Q13：服务器有多根网线时怎么避免误操作？

**Answer：**

先识别网卡：

```bash
ip -br addr
nmcli device status
```

示例：

```text
enp7s0           10.11.12.6/23       → 光纤/稳定有线，不要动
wlp8s0           192.168.0.132/24     → BASE Wi-Fi
enx00e04c6802ba  无 IPv4              → 临时调试 USB 网卡
```

不要对光纤网卡执行 `flush` 或改路由。

---

### Q14：GO2 重启后 `192.168.0.211 / 192.168.0.212` 消失怎么办？

**Answer：**

这是正常现象。重新登录 GO2 默认地址后执行：

```bash
sudo ip addr add 192.168.0.211/24 dev eth0
```

或：

```bash
sudo ip addr add 192.168.0.212/24 dev eth0
```

---

### Q15：Cursor / VS Code Remote SSH 连接失败怎么办？

**Answer：**

先确认普通终端可以：

```bash
ssh go2-1
ssh go2-2
```

如果终端不通，Cursor / VS Code 也不会通。检查：

```text
1. ~/.ssh/config 是否正确；
2. GO2 临时 IP 是否还存在；
3. GO2 是否重启过；
4. 服务器是否稳定连接 BASE。
```

---

## 7. 最终推荐流程总结

### 7.1 初始化阶段

```text
1. 配置 BASE：192.168.0.1/24；
2. 两台机载路由器设为 WISP / 万能中继 / Client+AP；
3. 在 BASE 中固定两台机载路由器后台 IP；
4. 一次只初始化一台 GO2；
5. 临时进入 GO2 默认地址 192.168.123.18；
6. GO2-1 添加 192.168.0.211；
7. GO2-2 添加 192.168.0.212。
```

---

### 7.2 正式运行阶段

```text
1. 服务器连接 BASE；
2. GO2-1 接 GO2-1 机载路由器 LAN 口；
3. GO2-2 接 GO2-2 机载路由器 LAN 口；
4. 服务器通过 192.168.0.211 / 192.168.0.212 同时 SSH 两台 GO2。
```

---

### 7.3 推荐 SSH 命令

```bash
ssh unitree@192.168.0.211
ssh unitree@192.168.0.212
```

或：

```bash
ssh go2-1
ssh go2-2
```

---

### 7.4 推荐 ROS2 通讯检查命令

服务器：

```bash
ping -c 5 192.168.0.211
ping -c 5 192.168.0.212
ros2 node list
ros2 topic list
```

GO2-1：

```bash
ping -c 5 192.168.0.132
ping -c 5 192.168.0.212
```

GO2-2：

```bash
ping -c 5 192.168.0.132
ping -c 5 192.168.0.211
```

---

### 7.5 最小可行操作清单

```text
1. BASE 正常工作，网段为 192.168.0.0/24；
2. GO2-1 / GO2-2 机载路由器均以 WISP / 万能中继接入 BASE；
3. GO2-1 / GO2-2 均接各自机载路由器 LAN 口；
4. GO2-1 本体存在 192.168.0.211；
5. GO2-2 本体存在 192.168.0.212；
6. 服务器 ping 192.168.0.1 稳定；
7. 服务器 ping 192.168.0.211 / 192.168.0.212 稳定；
8. 服务器可执行 ssh go2-1 / ssh go2-2。
```
