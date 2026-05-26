# General WISP / Wireless Repeater Network Configuration Guide for SSH Control of Two Unitree GO2 Robots from a Server

During recent real-robot experiments related to multi-robot navigation, we found that the Unitree GO2 body does not come with a wireless network adapter that can be directly used for experimental networking. If the setup relies entirely on wired connections, on-site cabling becomes complicated and is also inconvenient for later demos and multi-robot collaborative experiments. At present, there are relatively few complete public tutorials for networking multiple physical GO2 robots. Judging from some related work and experimental system diagrams, the common solutions mainly include **onboard routers** and **onboard computers**.

This document focuses on the first solution: using **onboard routers** to connect multiple GO2 robots to the same LAN, then allowing a server to establish SSH connections to multiple robots and perform basic communication tests. This solution avoids modifying the permanent network configuration of the GO2 body as much as possible. Instead, it uses temporary IP addresses for debugging, which keeps the overall risk low and makes the setup easier to reproduce and extend. For the **onboard computer** solution, the idea is relatively straightforward: attach a small computer with a wireless network adapter to each robot and make sure it is on the same LAN as the server. This document does not discuss that solution further.

This guide is somewhat general and is not limited to a specific router brand or robot. For other routers that support WISP / wireless repeater / universal repeater / Client+AP / wireless bridge functions, or for other single-robot or multi-robot platforms, you can adjust the corresponding IP addresses and device names according to your actual network environment. Because this is based on limited personal experience, comments, corrections, and discussions are welcome if anything is inappropriate.

> Target setup: one upstream primary router + two GO2 onboard portable routers. Each onboard router needs to support one or more of the following functions: **WISP / wireless repeater / universal repeater / Client+AP / wireless bridge**.  
> Core principle: **the onboard router is only responsible for connecting the GO2 to the BASE network; the real SSH target is not the onboard router IP, but the temporary `192.168.0.x` management IP added to the GO2 body.**

---

## 1. Hardware Configuration and Network Goal

### 1.1 Device Selection

This solution requires:

```text
1. Upstream primary router BASE
2. GO2-1 onboard portable router
3. GO2-2 onboard portable router
4. One server
5. Ethernet cable between each GO2 and its corresponding onboard router
6. Optional: USB Ethernet adapter, used for temporary direct connection to the GO2 LAN during initialization
```

Router requirements:

```text
1. Supports WISP / wireless repeater / universal repeater / Client+AP / wireless bridge, and can connect to BASE as a wireless client;
2. Can provide network access to the GO2 through a LAN port;
3. Preferably supports disabling AP isolation / client isolation.
```

Actual hardware example used here:

```text
* BASE primary router: Tenda AX3000 Ultra
* GO2-1 onboard router: Tenda AX3000 Ultra
* GO2-2 onboard router: Tenda AX3000 Ultra
* Example server Wi-Fi IP: 192.168.0.132
* Router power supply: the expansion dock included with Unitree provides two power ports, 5V/3A and 12V/3A, with XT30 connectors, which can be used to power the router. Choose the power cable according to the router's charging connector type.
```

Note: different vendors use different names for operating modes. Prefer a mode that allows downstream devices to enter the same subnet as BASE, such as:

```text
Wireless repeater, universal repeater, Client+AP, wireless bridge, WISP repeater
```

If the WISP mode on some routers is an independent NAT routing mode, it is not the preferred mode for this guide. NAT/WISP routing mode requires port forwarding, which makes the process more complicated.

---

### 1.2 Recommended Network Topology

```text
BASE primary router
IP: 192.168.0.1
Subnet: 192.168.0.0/24

|-- Server
|   `-- Example IP: 192.168.0.132
|
|-- GO2-1 onboard router
|   |-- Operating mode: WISP / universal repeater / Client+AP
|   |-- Upstream Wi-Fi: BASE
|   |-- Management IP: 192.168.0.135 (use the actual bound address)
|   `-- GO2-1 body eth0
|       |-- Original IP: 192.168.123.18/24
|       `-- Temporary SSH IP: 192.168.0.211/24
|
`-- GO2-2 onboard router
    |-- Operating mode: WISP / universal repeater / Client+AP
    |-- Upstream Wi-Fi: BASE
    |-- Management IP: 192.168.0.105 (use the actual bound address)
    `-- GO2-2 body eth0
        |-- Original IP: 192.168.123.18/24
        `-- Temporary SSH IP: 192.168.0.212/24
```

Address planning table:

| Device | Purpose | Recommended / Example IP |
|---|---|---|
| BASE primary router | Gateway | `192.168.0.1` |
| Server | Control side | `192.168.0.132`, use the actual address |
| GO2-1 onboard router | Router admin interface | `192.168.0.135`, use the actual bound address |
| GO2-2 onboard router | Router admin interface | `192.168.0.105`, use the actual bound address |
| GO2-1 body | SSH management | `192.168.0.211` |
| GO2-2 body | SSH management | `192.168.0.212` |
| GO2 default address | Temporary initialization entry point | `192.168.123.18` |

Final SSH targets:

```bash
ssh unitree@192.168.0.211
ssh unitree@192.168.0.212
```

Do not treat the onboard router admin address as the GO2 body address:

```text
192.168.0.135  -> GO2-1 onboard router admin interface
192.168.0.105  -> GO2-2 onboard router admin interface
192.168.0.211  -> GO2-1 body SSH
192.168.0.212  -> GO2-2 body SSH
```

---

### 1.3 Physical Connection Method

Each GO2 connects to the LAN port of its own onboard router:

```text
GO2-1 eth0  <-> Ethernet cable <->  GO2-1 onboard router LAN port
GO2-2 eth0  <-> Ethernet cable <->  GO2-2 onboard router LAN port
```

Notes:

```text
1. Connect the GO2 to a LAN port, not a WAN port;
2. The onboard router connects to BASE through WISP / universal repeater;
3. Keep the GO2 default IP 192.168.123.18 unchanged;
4. It is not recommended to modify the permanent GO2 netplan / systemd-networkd / NetworkManager configuration;
5. The 192.168.0.211 / 192.168.0.212 addresses added to the GO2 are temporary IPs and will disappear after reboot.
```

---

## 2. Basic Network Configuration Procedure

### 2.1 Configure the BASE Primary Router

Enter the BASE admin interface and configure the LAN:

```text
LAN IP: 192.168.0.1
Subnet mask: 255.255.255.0
DHCP: enabled
DHCP address pool: 192.168.0.100 - 192.168.0.200
```

Recommendations:

```text
1. Connect the server to BASE through Ethernet or high-quality Wi-Fi if possible;
2. Do not enable AP isolation / client isolation on BASE if it blocks LAN-to-LAN communication;
3. Avoid letting the DHCP address pool cover the manually assigned GO2 body management IPs, such as 192.168.0.211 / 192.168.0.212;
4. If the pool must cover those addresses, make sure they will not be assigned to other devices.
```

---

### 2.2 Configure Each Onboard Router as WISP / Universal Repeater

Configure each onboard router separately.

General steps:

```text
1. Enter the onboard router admin interface;
2. Switch the operating mode to WISP / universal repeater / Client+AP / wireless bridge;
3. Select the upstream Wi-Fi: BASE;
4. Enter the BASE password;
5. Save and wait for the router to reboot;
6. Confirm in the BASE admin interface that the onboard router is online.
```

It is recommended to rename the Wi-Fi broadcast by each onboard router to:

```text
GO2-1
GO2-2
```

Do not also name it `BASE`; otherwise, the server may connect to the wrong AP and cause an unstable link.

If the router provides the following options, it is recommended to:

```text
Disable guest network
Disable AP isolation / client isolation
Disable smart roaming / automatic dual-band switching features
```

---

### 2.3 Bind the Onboard Router Management IPs in the BASE Admin Interface

In the BASE admin interface, enter a page similar to:

```text
Network Settings -> LAN Settings -> Static IP Assignment List
```

Bind a management IP for each onboard router.

Example:

```text
GO2-1:
IP: 192.168.0.135
MAC: use the MAC shown in the BASE device management page

GO2-2:
IP: 192.168.0.105
MAC: use the MAC shown in the BASE device management page
```

Important principle:

```text
When binding a fixed IP in BASE, use the MAC shown in the BASE device management page;
do not force the use of the LAN MAC / 2.4G MAC / 5G MAC shown in the onboard router admin interface.
```

Reason: in WISP / universal repeater mode, the router may use a virtual MAC, proxy MAC, or wireless client MAC to access the upstream router. Therefore, it is normal for the MAC seen by BASE to differ from the MAC shown in the onboard router admin interface.

If a newly bound static IP does not take effect, you can directly bind the IP that is already stable and accessible. The onboard router management IP is only used to enter its admin interface; it is not the SSH target.

---

### 2.4 Check the Link Quality from the Server to BASE

After the server connects to BASE, first test the quality of the connection to BASE:

```bash
ip -br addr
ip route get 192.168.0.1
ping -c 20 192.168.0.1
```

If the server connects to BASE through Wi-Fi, also check:

```bash
iw dev wlp8s0 link
nmcli -f IN-USE,SSID,BSSID,SIGNAL,CHAN dev wifi list
```

Ideal state:

```text
ping 192.168.0.1: 0% packet loss
Average latency: preferably < 10 ms, at least < 30 ms
Wi-Fi signal: recommended better than -60 dBm
tx/rx bitrate: should not remain at only 6 MBit/s for a long time
```

If even `192.168.0.1` has packet loss or latency of several hundred milliseconds or more, first fix the server-to-BASE link. Do not continue debugging SSH yet.

---

## 3. Add Temporary SSH Management IPs to the Two GO2 Bodies

Both GO2 robots use the default address:

```text
192.168.123.18
```

Therefore, during initialization, it is recommended to configure only one GO2 at a time to avoid IP conflicts.

Goal:

```text
Add to GO2-1: 192.168.0.211/24
Add to GO2-2: 192.168.0.212/24
```

---

### 3.1 Method A: Temporarily Connect the Server to the Corresponding Onboard Router Wi-Fi for Initialization

Applicable scenarios:

```text
No USB Ethernet adapter, or wiring is inconvenient;
You only need temporary access to one GO2;
The onboard router Wi-Fi signal is good.
```

Use GO2-2 as an example.

#### 3.1.1 Connect to the GO2-2 Wi-Fi

```bash
nmcli -f IN-USE,SSID,BSSID,SIGNAL,CHAN dev wifi list
nmcli dev wifi connect "GO2-2" password "your GO2-2 WiFi password"
nmcli -t -f ACTIVE,SSID dev wifi | grep '^yes'
```

Confirm that the output is:

```text
yes:GO2-2
```

#### 3.1.2 Add a Temporary Address to the Server Wi-Fi Adapter

```bash
sudo ip addr add 192.168.123.10/24 dev wlp8s0
ip -4 addr show dev wlp8s0
```

You should see:

```text
192.168.123.10/24
```

If it reports:

```text
RTNETLINK answers: File exists
```

it means the address has already been added and can be ignored.

#### 3.1.3 Set the Route to the GO2 Default Subnet

```bash
sudo ip route replace 192.168.123.0/24 dev wlp8s0 src 192.168.123.10 metric 5
sudo ip route flush cache
ip route get 192.168.123.18
```

Expected:

```text
192.168.123.18 dev wlp8s0 src 192.168.123.10
```

#### 3.1.4 Confirm That You Are Accessing the GO2 Body

```bash
ping -I wlp8s0 -c 5 192.168.123.18
ip neigh show 192.168.123.18 dev wlp8s0
```

How to judge:

```text
If the MAC is a GO2 body MAC such as 3c:6d:66:xx:xx:xx, it is correct;
If the MAC is the onboard router MAC, you are accessing the wrong target.
```

#### 3.1.5 Log in to the GO2 and Add the Temporary IP

For GO2-1:

```bash
ssh unitree@192.168.123.18
sudo ip addr add 192.168.0.211/24 dev eth0
ip -4 addr show eth0
```

For GO2-2:

```bash
ssh unitree@192.168.123.18
sudo ip addr add 192.168.0.212/24 dev eth0
ip -4 addr show eth0
```

You should see:

```text
192.168.123.18/24
192.168.0.211/24 or 192.168.0.212/24
```

#### 3.1.6 Switch Back to BASE

After exiting the GO2:

```bash
exit
nmcli dev wifi connect "BASE" password "your BASE password"
nmcli -t -f ACTIVE,SSID dev wifi | grep '^yes'
```

Test the final IPs:

```bash
ping -c 10 192.168.0.211
ping -c 10 192.168.0.212
```

---

### 3.2 Method B: Temporarily Connect the Server to the Onboard Router LAN Port with Ethernet

Applicable scenarios:

```text
Recommended method;
Wi-Fi initialization is unstable;
You want the server to stay connected to BASE while using a USB Ethernet adapter to enter the GO2 default subnet.
```

Physical connection:

```text
Server USB Ethernet adapter  <-> Ethernet cable <->  Onboard router LAN port
GO2 eth0                     <-> Ethernet cable <->  Same onboard router LAN port
```

Identify the server network adapters:

```bash
ip -br link
ip -br addr
nmcli device status
```

Example:

```text
enp7s0           10.11.12.6/23       -> Fiber / stable wired link, do not touch
wlp8s0           192.168.0.132/24     -> BASE Wi-Fi
enx00e04c6802ba  no IPv4              -> Temporary debugging USB Ethernet adapter
```

Configure the USB Ethernet adapter:

```bash
sudo ip link set enx00e04c6802ba up
sudo ip addr add 192.168.123.10/24 dev enx00e04c6802ba
sudo ip route replace 192.168.123.0/24 dev enx00e04c6802ba src 192.168.123.10 metric 5
sudo ip route flush cache
ip route get 192.168.123.18
```

Expected:

```text
192.168.123.18 dev enx00e04c6802ba src 192.168.123.10
```

Test:

```bash
ping -I enx00e04c6802ba -c 10 192.168.123.18
ip neigh show 192.168.123.18 dev enx00e04c6802ba
```

Normally:

```text
ping latency is about 0.5 - 2 ms
The MAC should be the GO2 body MAC
```

Log in and add the temporary IP:

```bash
ssh unitree@192.168.123.18
```

GO2-1:

```bash
sudo ip addr add 192.168.0.211/24 dev eth0
ip -4 addr show eth0
```

GO2-2:

```bash
sudo ip addr add 192.168.0.212/24 dev eth0
ip -4 addr show eth0
```

---

### 3.3 How to Re-add the Address After a GO2 Reboot

This solution does not modify the permanent GO2 configuration. After the GO2 reboots, the temporary IP will disappear.

After GO2-1 reboots, run again:

```bash
sudo ip addr add 192.168.0.211/24 dev eth0
```

After GO2-2 reboots, run again:

```bash
sudo ip addr add 192.168.0.212/24 dev eth0
```

If it reports:

```text
RTNETLINK answers: File exists
```

the IP already exists and does not need to be added again.

---

## 4. Use the Server to SSH into Two GO2 Robots Simultaneously

### 4.1 Final SSH Connection Method

During formal operation, the server connects to the BASE network and controls both GO2 robots:

```bash
ssh unitree@192.168.0.211
ssh unitree@192.168.0.212
```

Do not use the following for long-term operation:

```bash
ssh unitree@192.168.123.18
```

because the two GO2 robots have the same default address.

---

### 4.2 Configure `~/.ssh/config` on the Server

Edit:

```bash
nano ~/.ssh/config
```

Add:

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

Then:

```bash
ssh go2-1
ssh go2-2
```

Cursor / VS Code Remote SSH can also select `go2-1` or `go2-2`.

---

### 4.3 Resolve SSH Host Key Conflicts

Symptom:

```text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
Host key for 192.168.123.18 has changed
```

Reason:

```text
GO2-1 and GO2-2 both use 192.168.123.18 by default, but they are two different machines with different SSH host keys. During configuration, the server may have previously remembered the key of one robot. When connecting to the other robot, this warning is triggered.
```

During initialization, you can remove the old record:

```bash
ssh-keygen -f "/home/maker002/.ssh/known_hosts" -R "192.168.123.18"
```

Or use the general form:

```bash
ssh-keygen -f ~/.ssh/known_hosts -R "192.168.123.18"
```

After switching to `192.168.0.211 / 192.168.0.212`, the setup no longer depends on the duplicated `192.168.123.18`, so simultaneous SSH access is not affected.

---

## 5. ROS2 Multi-Robot Communication Configuration

> This section provides recommended configuration ideas and needs to be further verified according to the actual Unitree ROS2 driver and launch files.

### 5.1 Three-Endpoint Connectivity Check: Server, GO2-1, GO2-2

On the server:

```bash
ping -c 5 192.168.0.211
ping -c 5 192.168.0.212
```

On GO2-1:

```bash
ping -c 5 192.168.0.132
ping -c 5 192.168.0.212
```

On GO2-2:

```bash
ping -c 5 192.168.0.132
ping -c 5 192.168.0.211
```

After all three endpoints can communicate stably, test ROS2.

---

### 5.2 Basic ROS2 Environment Variable Setup

Source the ROS2 environment on all three endpoints. For example, Foxy:

```bash
source /opt/ros/foxy/setup.bash
```

If there is a workspace:

```bash
source ~/your_ws/install/setup.bash
```

---

### 5.3 `ROS_DOMAIN_ID` and `ROS_LOCALHOST_ONLY`

Use the same Domain ID on all three endpoints:

```bash
export ROS_DOMAIN_ID=0
export ROS_LOCALHOST_ONLY=0
```

You can write them to `~/.bashrc`:

```bash
echo 'export ROS_DOMAIN_ID=0' >> ~/.bashrc
echo 'export ROS_LOCALHOST_ONLY=0' >> ~/.bashrc
```

Check:

```bash
echo $ROS_DOMAIN_ID
echo $ROS_LOCALHOST_ONLY
```

---

### 5.4 Multi-Robot namespace / topic / node / tf Naming Recommendations

The items most likely to conflict in multi-robot ROS2 are:

```text
node name
topic name
service name
tf frame
robot_id
```

It is recommended to assign namespaces to the two GO2 robots:

```text
GO2-1: /go2_1
GO2-2: /go2_2
```

Example:

```bash
export ROS_NAMESPACE=/go2_1
```

```bash
export ROS_NAMESPACE=/go2_2
```

If the launch file supports parameters, prefer using:

```text
namespace
robot_name
robot_id
frame_prefix
```

If `ros2 topic list` cannot see the other side but ping and SSH are normal, DDS multicast may be blocked by the wireless repeater. You can later consider configuring CycloneDDS / FastDDS unicast discovery or a discovery server.

---

## 6. Errors and Troubleshooting

### Q1: What should I do if the onboard router IP does not change after static IP assignment in BASE?

**Answer:**

Common causes:

```text
1. The onboard router still holds the old DHCP lease;
2. The static assignment was not saved;
3. The bound IP is outside the DHCP address pool, and some routers may not actually hand it out;
4. The bound MAC is not the MAC seen by BASE.
```

Handling:

```text
1. Use the MAC shown in the BASE device management page;
2. Click Save;
3. Reboot the onboard router;
4. If necessary, reboot BASE first, then reboot the onboard router;
5. If it still does not take effect, directly bind the IP that is already stable and online.
```

The onboard router management IP is only for admin access. It is not the SSH target.

---

### Q2: Is it normal that the MAC shown in the BASE admin interface differs from the MAC shown in the onboard router admin interface?

**Answer:**

Yes. The onboard router has multiple MACs:

```text
LAN MAC
2.4G Wi-Fi MAC
5G Wi-Fi MAC
WISP / repeater virtual MAC
```

In BASE static IP assignment, use **the MAC seen by BASE**.

---

### Q3: The server has packet loss even when pinging BASE. What if SSH to the GO2 gets stuck?

**Answer:**

Do not debug the GO2 yet. Run:

```bash
ping -c 20 192.168.0.1
iw dev wlp8s0 link
nmcli -f IN-USE,SSID,BSSID,SIGNAL,CHAN dev wifi list
```

If `ping 192.168.0.1` already has packet loss or high latency, the server-to-BASE link is unstable.

Solutions:

```text
1. Prefer connecting the server to BASE with Ethernet;
2. Or move the server / BASE to improve the Wi-Fi signal;
3. Do not name the onboard router Wi-Fi BASE as well;
4. Confirm that the server is connected to the correct BASE BSSID.
```

---

### Q4: What should I do if the server connects to the wrong AP or a Wi-Fi network with the same name?

**Answer:**

Scan:

```bash
nmcli -f IN-USE,SSID,BSSID,SIGNAL,CHAN dev wifi list
```

If there are multiple networks named `BASE`, you should:

```text
1. Rename the GO2-1 / GO2-2 onboard router Wi-Fi networks;
2. Make the server connect only to the real BASE;
3. Specify the target by BSSID if necessary.
```

---

### Q5: What should I do if `ssh unitree@192.168.123.18` reports host key changed?

**Answer:**

This is because the two GO2 robots have the same default IP but different host keys.

Remove the old record:

```bash
ssh-keygen -f ~/.ssh/known_hosts -R "192.168.123.18"
```

Then reconnect and type `yes`.

During formal operation, use:

```bash
ssh go2-1
ssh go2-2
```

The setup will no longer depend on the duplicated IP.

---

### Q6: What should I do if `ssh` gets stuck at `Connection timed out during banner exchange`?

**Answer:**

First test:

```bash
nc -vz -w 5 192.168.123.18 22
timeout 15 nc -v 192.168.123.18 22
```

If `nc -v` can show:

```text
SSH-2.0-OpenSSH_xxx
```

then try SSH again.

If you cannot see it, check whether you reached the GO2 body:

```bash
ip route get 192.168.123.18
ping -I <network-adapter> -c 10 192.168.123.18
ip neigh show 192.168.123.18 dev <network-adapter>
```

---

### Q7: What should I do if `ping` works but SSH gets stuck?

**Answer:**

Check latency and packet loss. If latency is several hundred milliseconds or even several seconds, or packet loss is obvious, SSH may get stuck.

It is recommended to reboot the BASE network first and see whether the network quality is restored, then try SSH again.

---

### Q8: What should I do if `ip route replace` reports `Invalid prefsrc address`?

**Answer:**

The reason is that the specified source address has not yet been added to that network adapter.

Error example:

```bash
sudo ip route replace 192.168.123.0/24 dev wlp8s0 src 192.168.123.10 metric 5
Error: Invalid prefsrc address.
```

Correct order:

```bash
sudo ip addr add 192.168.123.10/24 dev wlp8s0
ip -4 addr show dev wlp8s0
sudo ip route replace 192.168.123.0/24 dev wlp8s0 src 192.168.123.10 metric 5
```

---

### Q9: What should I do if ARP points to the onboard router MAC when accessing `192.168.123.18`?

**Answer:**

It means you did not reach the GO2 body.

Check:

```bash
ip neigh show 192.168.123.18 dev <network-adapter>
```

If the MAC is the onboard router MAC, check:

```text
1. Whether the GO2 is connected to a LAN port;
2. Whether the server is connected to the correct LAN;
3. Whether traffic mistakenly went through BASE / repeater path;
4. Whether another device is also using 192.168.123.18.
```

---

### Q10: What should I do if I accidentally switch the onboard router to normal router / NAT mode?

**Answer:**

Recovery method:

```text
1. Connect the server or computer directly to the onboard router LAN port;
2. Use ip route to view the default gateway;
3. Open the default gateway in a browser to enter the admin interface;
4. Switch back to WISP / universal repeater / Client+AP;
5. Reconnect to BASE.
```

---

### Q11: Can I bind a device IP on the "Static Route" page?

**Answer:**

No.

To bind a device IP, enter:

```text
Network Settings -> LAN Settings -> Static IP Assignment List
```

Do not add rules such as `192.168.0.211` or `192.168.123.18` on the "Static Route" page.

---

### Q12: Both GO2 robots use the default IP `192.168.123.18`. Does this affect final simultaneous control?

**Answer:**

It does not affect the final solution. `192.168.123.18` is only used for initialization.

Formal operation uses:

```text
GO2-1: 192.168.0.211
GO2-2: 192.168.0.212
```

---

### Q13: How can I avoid mistakes when the server has multiple Ethernet cables?

**Answer:**

Identify the network adapters first:

```bash
ip -br addr
nmcli device status
```

Example:

```text
enp7s0           10.11.12.6/23       -> Fiber / stable wired link, do not touch
wlp8s0           192.168.0.132/24     -> BASE Wi-Fi
enx00e04c6802ba  no IPv4              -> Temporary debugging USB Ethernet adapter
```

Do not run `flush` or change routes on the fiber network adapter.

---

### Q14: What should I do if `192.168.0.211 / 192.168.0.212` disappears after a GO2 reboot?

**Answer:**

This is normal. After logging in again through the GO2 default address, run:

```bash
sudo ip addr add 192.168.0.211/24 dev eth0
```

Or:

```bash
sudo ip addr add 192.168.0.212/24 dev eth0
```

---

### Q15: What should I do if Cursor / VS Code Remote SSH connection fails?

**Answer:**

First confirm that a normal terminal works:

```bash
ssh go2-1
ssh go2-2
```

If the terminal cannot connect, Cursor / VS Code will not connect either. Check:

```text
1. Whether ~/.ssh/config is correct;
2. Whether the GO2 temporary IP still exists;
3. Whether the GO2 has rebooted;
4. Whether the server is stably connected to BASE.
```

---

## 7. Final Recommended Procedure Summary

### 7.1 Initialization Stage

```text
1. Configure BASE: 192.168.0.1/24;
2. Set the two onboard routers to WISP / universal repeater / Client+AP;
3. Bind the admin IPs of the two onboard routers in BASE;
4. Initialize only one GO2 at a time;
5. Temporarily access the GO2 default address 192.168.123.18;
6. Add 192.168.0.211 to GO2-1;
7. Add 192.168.0.212 to GO2-2.
```

---

### 7.2 Formal Operation Stage

```text
1. Connect the server to BASE;
2. Connect GO2-1 to the LAN port of the GO2-1 onboard router;
3. Connect GO2-2 to the LAN port of the GO2-2 onboard router;
4. The server SSHes into both GO2 robots simultaneously through 192.168.0.211 / 192.168.0.212.
```

---

### 7.3 Recommended SSH Commands

```bash
ssh unitree@192.168.0.211
ssh unitree@192.168.0.212
```

Or:

```bash
ssh go2-1
ssh go2-2
```

---

### 7.4 Recommended ROS2 Communication Check Commands

Server:

```bash
ping -c 5 192.168.0.211
ping -c 5 192.168.0.212
ros2 node list
ros2 topic list
```

GO2-1:

```bash
ping -c 5 192.168.0.132
ping -c 5 192.168.0.212
```

GO2-2:

```bash
ping -c 5 192.168.0.132
ping -c 5 192.168.0.211
```

---

### 7.5 Minimum Viable Checklist

```text
1. BASE works normally, and its subnet is 192.168.0.0/24;
2. GO2-1 / GO2-2 onboard routers are both connected to BASE through WISP / universal repeater;
3. GO2-1 / GO2-2 are each connected to the LAN port of their own onboard router;
4. The GO2-1 body has 192.168.0.211;
5. The GO2-2 body has 192.168.0.212;
6. The server can ping 192.168.0.1 stably;
7. The server can ping 192.168.0.211 / 192.168.0.212 stably;
8. The server can run ssh go2-1 / ssh go2-2.
```
