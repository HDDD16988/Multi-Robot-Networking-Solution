# Unitree GO2 Multi-Robot WISP/NAT Network Setup Tutorial

During recent real-robot experiments on multi-robot navigation, we found that the Unitree GO2 body does not include a wireless adapter that can be directly used for experimental networking. Relying entirely on wired connections makes on-site cabling complicated and is inconvenient for later demos and multi-robot collaboration experiments. Publicly available complete tutorials for networking multiple real GO2 robots are still limited. Based on related work and system diagrams, common approaches usually include USB Wi-Fi adapters, onboard routers, and onboard computers.

For the USB Wi-Fi adapter approach, one reference tutorial is available here: https://www.youtube.com/watch?v=Dsntvdr2cho

For the onboard computer approach, the idea is relatively straightforward: attach a small computer with Wi-Fi capability to each robot, then place it in the same LAN as the server.

This project focuses on the second approach: using onboard routers to connect multiple GO2 robots to the same upper-level LAN, and then allowing the server to perform SSH access and basic communication tests with multiple robots. This approach tries to avoid permanent changes to the GO2 body's network configuration. Instead, it relies on router-side WISP/NAT isolation and explicit port forwarding, which keeps the setup relatively low-risk, easy to reproduce, and easy to extend. This tutorial is not limited to a specific router brand or robot platform. For other routers that support WISP, or for other single-robot or multi-robot platforms, you can adjust the IP addresses and device names according to your actual network environment.

Due to limited personal experience, feedback and corrections are welcome.

This document describes a network setup suitable for connecting multiple Unitree GO2 robots to one server at the same time. Each GO2 is connected to an independent onboard router. Each onboard router joins the upper-level `BASE` network in **WISP/NAT** mode. The server only accesses the address and port mappings of each onboard router on the `BASE` network. It does not directly enter each GO2's private LAN.

Core principles:

```text
1. Keep each GO2 body at its default address: 192.168.123.18/24
2. Give each GO2 its own private LAN behind its onboard router
3. Connect each onboard router to the BASE network using WISP/NAT
4. Let the server access explicit TCP services such as SSH and ZMQ through port forwarding
5. Do not merge the private LANs of multiple GO2 robots into the same Layer-2 network
```

With this setup, multiple GO2 robots can use the same default body address at the same time, while low-level communication from different robots remains isolated.

---

## 1. Applicable Scenarios

This tutorial is intended for the following scenarios:

```text
1. One server needs to connect to two or more Unitree GO2 robots at the same time
2. Each GO2 has an independent onboard router
3. There is a shared upper-level BASE router or Wi-Fi hotspot
4. The server and onboard routers are all connected to the BASE network
5. The server accesses each GO2 through explicit ports such as SSH and ZMQ TCP
```

This tutorial uses two GO2 robots as the example. To extend the setup to more robots, add one onboard router for each robot and assign different `BASE`-side addresses and external ports.

---

## 2. Network Architecture Overview

### 2.1 Recommended Topology

An example network topology is shown below. In actual use, please follow the IP addresses shown in the admin page of the `BASE` router.

```text
BASE router
IP: 192.168.0.1
Subnet: 192.168.0.0/24

+-- Server
|   +-- 192.168.0.132
|
+-- GO2-1 onboard router
|   +-- Mode: WISP/NAT
|   +-- Upstream Wi-Fi: BASE or BASE 5G
|   +-- BASE-side IP: 192.168.0.135
|   +-- LAN IP: 192.168.123.1/24
|   +-- GO2-1 body eth0: 192.168.123.18/24
|
+-- GO2-2 onboard router
    +-- Mode: WISP/NAT
    +-- Upstream Wi-Fi: BASE or BASE 5G
    +-- BASE-side IP: 192.168.0.105
    +-- LAN IP: 192.168.123.1/24
    +-- GO2-2 body eth0: 192.168.123.18/24
```

### 2.2 Address Plan

| Device | Purpose | Example |
|---|---|---|
| BASE router | Upper-level gateway | `192.168.0.1` |
| Server | Control side | `192.168.0.132` |
| GO2-1 onboard router | BASE-side entry for GO2-1 | `192.168.0.135` |
| GO2-2 onboard router | BASE-side entry for GO2-2 | `192.168.0.105` |
| GO2-1 router LAN | Private gateway for GO2-1 | `192.168.123.1` |
| GO2-2 router LAN | Private gateway for GO2-2 | `192.168.123.1` |
| GO2-1 body | Address inside GO2-1 private LAN | `192.168.123.18` |
| GO2-2 body | Address inside GO2-2 private LAN | `192.168.123.18` |

The LAN-side addresses of the two onboard routers can be the same because they are located in separate NAT private networks. On the server side, access is performed through different `BASE`-side IP addresses and ports. The server does not directly access `192.168.123.18`.

### 2.3 Port Plan

| Robot | Onboard Router BASE-side IP | Service | External Port | Internal Address | Internal Port |
|---|---:|---|---:|---:|---:|
| GO2-1 | `192.168.0.135` | SSH | `2211` | `192.168.123.18` | `22` |
| GO2-1 | `192.168.0.135` | TCP | `5557` | `192.168.123.18` | `5557` |
| GO2-2 | `192.168.0.105` | SSH | `2212` | `192.168.123.18` | `22` |
| GO2-2 | `192.168.0.105` | TCP | `5557` | `192.168.123.18` | `5557` |

Example server access:

```bash
ssh -p 2211 unitree@192.168.0.135
ssh -p 2212 unitree@192.168.0.105
```

Example ZMQ TCP addresses:

```text
GO2-1: tcp://192.168.0.135:5557
GO2-2: tcp://192.168.0.105:5557
```

---

## 3. Preparation

### 3.1 Network Prerequisites

Before configuration, confirm the following:

```text
1. The BASE router can provide Wi-Fi and DHCP normally
2. The server is already connected to the BASE network
3. Both onboard routers can connect to the BASE Wi-Fi
4. Each GO2 can connect to the LAN port of its own onboard router through eth0
5. You can enter the admin page of each onboard router
```

### 3.2 Replacing the Example Addresses

This document uses the following example addresses:

```text
Server: 192.168.0.132
GO2-1 onboard router BASE-side IP: 192.168.0.135
GO2-2 onboard router BASE-side IP: 192.168.0.105
```

During actual configuration, check the real IP obtained by each onboard router in the `BASE` router admin page, and replace the example addresses in the commands below with your on-site addresses.

---

## 4. Physical Connection

Each GO2 should only connect to the LAN port of its own onboard router:

```text
GO2-1 eth0  <-- Ethernet cable -->  GO2-1 onboard router LAN port
GO2-2 eth0  <-- Ethernet cable -->  GO2-2 onboard router LAN port
```

The onboard routers connect to the upper-level `BASE` network through Wi-Fi:

```text
GO2-1 onboard router  <-- Wi-Fi WISP/NAT -->  BASE
GO2-2 onboard router  <-- Wi-Fi WISP/NAT -->  BASE
Server                <-- Wi-Fi or wired -->  BASE
```

Connection checklist:

```text
1. The GO2 Ethernet cable is plugged into the LAN port of the onboard router
2. The server is connected to the BASE network
3. GO2-1 and GO2-2 are connected to their own onboard routers
4. Both onboard routers join BASE through WISP/NAT
5. Both GO2 bodies remain at 192.168.123.18/24
```

---

## 5. Onboard Router Configuration

The following steps need to be completed separately in the admin pages of the `GO2-1` and `GO2-2` onboard routers. Menu names may vary slightly across router brands, so please follow the actual router interface.

### 5.1 Set WISP/NAT Mode

Enter the onboard router admin page and find the operation mode page, for example:

```text
More Functions -> Operation Mode
```

Select:

```text
Hotspot Signal Repeater Mode (WISP)
```

or an equivalent name:

```text
WISP
WISP Router
Wireless ISP
Wireless Repeater Router Mode
NAT Repeater Mode
```

Select the upstream Wi-Fi:

```text
BASE or BASE 5G
```

Save the configuration, then wait for the router to reboot and reconnect to the `BASE` network.

### 5.2 Set the LAN Address

Enter the LAN settings page, for example:

```text
More Functions -> Network Settings -> LAN Settings
```

Set the LAN-side address of each onboard router to:

```text
LAN IP address: 192.168.123.1
Subnet mask: 255.255.255.0
```

After saving, the LAN-side admin address of the onboard router usually becomes:

```text
http://192.168.123.1
```

Because each onboard router's LAN is an independent private network, both `GO2-1` and `GO2-2` can use `192.168.123.1/24` on the LAN side.

### 5.3 Configure GO2-1 Port Forwarding

Enter the port forwarding page, for example:

```text
More Functions -> Advanced Settings -> Port Forwarding
```

Add SSH port forwarding:

```text
Device selection: Manual
Internal IP address: 192.168.123.18
Internal port: 22
External port: 2211
Protocol: TCP
```

Add ZMQ TCP port forwarding:

```text
Device selection: Manual
Internal IP address: 192.168.123.18
Internal port: 5557
External port: 5557
Protocol: TCP
```

### 5.4 Configure GO2-2 Port Forwarding

Add SSH port forwarding:

```text
Device selection: Manual
Internal IP address: 192.168.123.18
Internal port: 22
External port: 2212
Protocol: TCP
```

Add ZMQ TCP port forwarding:

```text
Device selection: Manual
Internal IP address: 192.168.123.18
Internal port: 5557
External port: 5557
Protocol: TCP
```

If the router interface only provides `TCP&UDP`, you can choose `TCP&UDP` for now. SSH and ZMQ TCP actually use TCP.

### 5.5 Confirm the BASE-Side IP of Each Onboard Router

After switching the server back to the `BASE` network, enter the admin page of the `BASE` router:

```text
Device Management / Online Devices / DHCP Client List
```

Find the two onboard routers and record their `BASE`-side IP addresses. Example:

```text
GO2-1 onboard router: 192.168.0.135
GO2-2 onboard router: 192.168.0.105
```

If the on-site IP addresses differ from the examples, use the on-site IP addresses in the following commands.

### 5.6 Firewall, DMZ, and UPnP Recommendations

The following settings are recommended:

```text
1. DMZ host: disabled
2. UPnP: disabled or unused
3. Port forwarding: only expose the required TCP ports
4. Static routes: no need to add 192.168.123.0/24 to BASE
```

If `Block WAN Ping` is enabled on some routers, the server may not be able to `ping` the onboard router. This does not necessarily mean port forwarding has failed. Use `nc` and `ssh` test results as the reference.

---

## 6. GO2 Body Network Configuration

In this section, first connect to the Wi-Fi of the corresponding onboard router, then enter each GO2's private LAN and run the commands below.

### 6.1 Directly Connect to the GO2 Body

After connecting to the Wi-Fi of the `GO2-1` onboard router, test the GO2 body:

```bash
ping -c 3 192.168.123.18
nc -vz -w 3 192.168.123.18 22
ssh unitree@192.168.123.18
```

After connecting to the Wi-Fi of the `GO2-2` onboard router, run the same checks:

```bash
ping -c 3 192.168.123.18
nc -vz -w 3 192.168.123.18 22
ssh unitree@192.168.123.18
```

After logging into the GO2, confirm the `eth0` address:

```bash
ip -4 -br addr show eth0
```

Expected output:

```text
eth0 UP 192.168.123.18/24
```

### 6.2 Add the Return Route

Each GO2 needs to know how to send return packets back to the server through its own onboard router. Assuming the server is in the `192.168.0.0/24` subnet and the onboard router LAN address is `192.168.123.1`, run the following command on each GO2 body:

```bash
sudo ip route replace 192.168.0.0/24 via 192.168.123.1 dev eth0
```

If your server or `BASE` network uses a different subnet, replace `192.168.0.0/24` with your on-site subnet.

### 6.3 Verify the Return Path

Assuming the server address is `192.168.0.132`, run the following command on the GO2 body:

```bash
ip route get 192.168.0.132
```

Expected output should look similar to:

```text
192.168.0.132 via 192.168.123.1 dev eth0 src 192.168.123.18
```

This means packets sent from the GO2 to the server will return to the `BASE` network through the onboard router.

### 6.4 Make the Return Route Persistent After Reboot

If the temporary route is lost after a system reboot, write the return route into a startup script or network configuration. The minimal usable script should include:

```bash
sudo ip route replace 192.168.0.0/24 via 192.168.123.1 dev eth0
```

Before making the route persistent, use `ip route get` to confirm that the route is correct, then persist it according to the startup mechanism used by your on-site system.

---

## 7. Server-Side Access Verification

Run all commands in this section on the server. The server needs to be connected to the `BASE` network.

### 7.1 Confirm That the Server Is on the BASE Network

Check the current Wi-Fi:

```bash
nmcli -t -f ACTIVE,SSID dev wifi | grep '^yes'
```

Check the route to the `BASE` gateway:

```bash
ip route get 192.168.0.1
```

Test connectivity to the `BASE` gateway:

```bash
ping -c 5 192.168.0.1
```

### 7.2 Test GO2-1 SSH

Assuming the `BASE`-side IP of the `GO2-1` onboard router is `192.168.0.135`:

```bash
nc -vz -w 3 192.168.0.135 2211
ssh-keygen -f ~/.ssh/known_hosts -R "[192.168.0.135]:2211"
ssh -p 2211 unitree@192.168.0.135
```

After the connection succeeds, the server can enter the `GO2-1` body through port forwarding.

### 7.3 Test GO2-2 SSH

Assuming the `BASE`-side IP of the `GO2-2` onboard router is `192.168.0.105`:

```bash
nc -vz -w 3 192.168.0.105 2212
ssh-keygen -f ~/.ssh/known_hosts -R "[192.168.0.105]:2212"
ssh -p 2212 unitree@192.168.0.105
```

After the connection succeeds, the server can enter the `GO2-2` body through port forwarding.

### 7.4 Remotely Check the GO2 Body Address

On the server, check the `eth0` address of each GO2:

```bash
ssh -p 2211 unitree@192.168.0.135 'ip -4 -br addr show eth0'
ssh -p 2212 unitree@192.168.0.105 'ip -4 -br addr show eth0'
```

Both robots are expected to show:

```text
eth0 UP 192.168.123.18/24
```

### 7.5 Test the ZMQ TCP Port

If a ZMQ TCP service listening on `5557` has already been started on the GO2 body, test the port from the server:

```bash
nc -vz -w 3 192.168.0.135 5557
nc -vz -w 3 192.168.0.105 5557
```

The connection is expected to succeed. At this point, the server is accessing the `BASE`-side IP addresses of the two onboard routers, and the requests are forwarded separately to `192.168.123.18:5557` inside each private LAN.

### 7.6 Checks When the SSH Port Is Reachable but Login Hangs

If `nc` shows that the port is reachable but `ssh` hangs during login, for example:

```bash
nc -vz -w 3 192.168.0.135 2211
ssh -p 2211 unitree@192.168.0.135
```

First check the SSH banner:

```bash
timeout 10 nc -v 192.168.0.135 2211
```

Normally, you should see something similar to:

```text
SSH-2.0-OpenSSH_...
```

If no banner appears, first go back to the corresponding GO2 body and confirm the return route:

```bash
sudo ip route replace 192.168.0.0/24 via 192.168.123.1 dev eth0
ip route get 192.168.0.132
```

If the banner appears but SSH still hangs, use verbose logs for debugging:

```bash
ssh -vvv \
  -o ConnectTimeout=10 \
  -o GSSAPIAuthentication=no \
  -o IPQoS=none \
  -p 2211 unitree@192.168.0.135
```

The check for `GO2-2` is the same. Replace the IP and port with your on-site configuration, for example:

```bash
timeout 10 nc -v 192.168.0.105 2212
ssh -vvv \
  -o ConnectTimeout=10 \
  -o GSSAPIAuthentication=no \
  -o IPQoS=none \
  -p 2212 unitree@192.168.0.105
```

After the above verification is complete, the server can access both GO2 robots separately through the `BASE` network. Each GO2 remains inside its own `192.168.123.0/24` private LAN, so multiple robots will not conflict even though they use the same default body address.
