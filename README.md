## Small Business Network Lab (Cisco Packet Tracer)

This project simulates a real-world small business network using Cisco Packet Tracer. It includes VLAN segmentation, inter-VLAN routing, DHCP, static IP assignments, router-on-a-stick topology, and ACLs to restrict server access.

[Small Business Network Topology](./Screenshot%202025-08-01%20130853.png)

---

# Objective

Create a secure, segmented, and functional small business network using multiple VLANs, multiple routers, DHCP, and ACLs for access control. Demonstrate:

- VLAN creation and port assignment
- Router-on-a-stick (subinterfaces)
- Inter-VLAN routing between routers
- DHCP for dynamic IP assignment
- Static IPs for servers and switches
- Access Control Lists (ACLs) to control ICMP access to critical servers

---

# Network Design Overview

## VLAN / Device IP Reference Table

| Device/Group     | VLAN | Subnet             | Gateway IP     | DHCP Range       | Static IPs          |
|------------------|------|--------------------|----------------|------------------|----------------------|
| Admin PCs        | 10   | 192.168.0.0 /27    | 192.168.0.1    | .11 – .30        | —                    |
| Tech PCs         | 20   | 192.168.0.32 /27   | 192.168.0.33   | .34 – .62        | —                    |
| Sales PCs        | 30   | 192.168.0.64 /27   | 192.168.0.65   | .71 – .94        | —                    |
| Router 1 (Subintf)| —  | —                  | —              | —                | .1, .33, .97         |
| Router 2 (Subintf) | —  | —                  | —              | —                | .65, .98             |
| Tech Server      | 20   | 192.168.0.32 /27   | 192.168.0.33   | —                | 192.168.0.41         |
| Shared Server    | 30   | 192.168.0.64 /27   | 192.168.0.65   | —                | 192.168.0.70         |
| Switch 1         | 10   | 192.168.0.0 /27    | 192.168.0.1    | —                |                      |
| Switch 2         | 30   | 192.168.0.64 /27   | 192.168.0.65   | —                |                      |

## Step-by-Step Configuration

## 1. Subnetting Breakdown

I started with the IP block `192.168.0.0/24` and needed to divide it into **3 separate networks** for Admin, Tech, and Sales. Each needed around 20 devices max, so I chose a **/27 subnet** which gives:

- **Subnet mask**: 255.255.255.224
- **Usable host addresses**: 2⁵ - 2 = 30
- **Each block increments by 32**

So I broke it down like this:

- Admin VLAN 10 → `192.168.0.0/27` (hosts .1 to .30, .0 = network ID, .31 = broadcast)
- Tech VLAN 20 → `192.168.0.32/27` (hosts .33 to .62, .32 = network ID, .63 = broadcast)
- Sales VLAN 30 → `192.168.0.64/27` (hosts .65 to .94, .64 = network ID, .95 = broadcast)

Extra blocks like `.96/27` were used for:
- **Router-to-router link** → `192.168.0.96/30` (just 2 usable IPs: .97 and .98)
  
This setup avoids overlap, keeps things clean, and ensures all VLANs are isolated but routable.

## 2. VLAN Configuration

# Switch 1 (Admin + Tech):
Creates VLAN 10 (Admin) and VLAN 20 (Tech) on swicth 1.
```bash
enable
conf t
vlan 10
name Admin
vlan 20
name Tech
exit
```
Assign Admin ports to VLAN 10 - Sets interfaces Fa0/1–2 as access ports for the Admin VLAN.
```bash
interface range fa0/1 - 2
switchport mode access
switchport access vlan 10
exit
```
Assign Tech ports to VLAN 20 - Sets interfaces Fa0/3–4 as access ports for the Tech VLAN.
```bash
interface range fa0/3 - 4
switchport mode access
switchport access vlan 20
exit
```
Configure trunk port on Switch 1 - Enables trunking on the port connecting to the router for inter-VLAN routing.
```bash
interface fa0/5
switchport mode trunk
exit
```
#Switch 2 (Sales):
Creates VLAN 30 (Sales) on Switch 2.
```bash
enable
conf t
vlan 30
name Sales
exit
```
Assign Sales ports to VLAN 30 - Sets interfaces Fa0/1–2 as access ports for the Sales VLAN.
```bash
interface range fa0/1 - 2
switchport mode access
switchport access vlan 30
exit
```
Configure trunk port on Switch 2 - Enables trunking on the port connecting to Router 2.
```bash
interface fa0/3
switchport mode trunk
exit
```
## 3. Inter-VLAN Routing Configuration
# Router 1:
Create subinterface for VLAN 10 on Router 1 - Allows inter-VLAN routing for Admin VLAN through subinterface .10.
```bash
interface g0/0/0.10
encapsulation dot1Q 10
ip address 192.168.0.1 255.255.255.224
exit
```
Create subinterface for VLAN 20 on Router 1 - Allows inter-VLAN routing for Tech VLAN through subinterface .20.
```bash
interface g0/0/0.20
encapsulation dot1Q 20
ip address 192.168.0.33 255.255.255.224
exit
```
Enable main Router 1 interface.
```bash
interface g0/0/0
no shutdown
exit
```
Set Router 1 inter-router IP - Configures Router 1’s IP for connecting to Router 2.
```bash
interface g0/0/0.1
ip address 192.168.0.97 255.255.252.0
exit
```
# Router 2:
Create subinterface for VLAN 30 on Router 2 - Enables routing for the Sales VLAN through subinterface .30.
```bash
interface g0/0/1.30
encapsulation dot1Q 30
ip address 192.168.0.65 255.255.255.224
exit
```
Enable main Router 2 interface.
```bash
interface g0/0/1
no shutdown
exit
```
Set Router 2 inter-router IP - Configures Router 2’s IP to link to Router 1.
```bash
interface g0/0/0
ip address 192.168.0.98 255.255.252.0
exit
```
## 4. DHCP Configuration
# Router 1 (Admin + Tech):
Assign IPs to Admin and Tech VLANs
```bash
enable
conf t
ip dhcp excluded-address 192.168.0.1 192.168.0.10
ip dhcp pool ADMIN
 network 192.168.0.0 255.255.255.224
 default-router 192.168.0.1
 dns-server 8.8.8.8
exit
# Excludes gateway, reserves first 10 IPs for static/manual assignment and Google's DNS.

ip dhcp pool TECH
 network 192.168.0.32 255.255.255.224
 default-router 192.168.0.33
 dns-server 8.8.8.8
exit
# Assigns dynamic IPs to Tech VLAN devices from .34–.62.
```
# Router 2 (Sales):
Assign IPs to Sales VLAN
```bash
enable
conf t
ip dhcp excluded-address 192.168.0.65 192.168.0.70
ip dhcp pool SALES
 network 192.168.0.64 255.255.255.224
 default-router 192.168.0.65
 dns-server 8.8.8.8
exit
# Reserves 6 IPs (.65–.70): gateway, servers, and any other static devices.
```
## 5. Server Configurations
# Shared Server – VLAN 10 (Accessible to All):
```bash
# On Switch 1
enable
conf t
interface fa0/7
 switchport mode access
 switchport access vlan 10
exit
# Connects the Shared Server to Admin VLAN.

# On Server (manual config)
IP Address:      192.168.0.10  
Subnet Mask:     255.255.255.224  
Default Gateway: 192.168.0.1
```
Shared server is intentionally placed in VLAN 10. Even though it's on Admin’s subnet, static routing between VLANs lets others reach it too.
# Tech Server - VLAN 20 (Tech-only access):
```bash
# On Switch 1
enable
conf t
interface fa0/6
 switchport mode access
 switchport access vlan 20
exit
# Puts server in Tech VLAN

# On Server (manual config)
IP Address:      192.168.0.41  
Subnet Mask:     255.255.255.224  
Default Gateway: 192.168.0.33
```
Configured Access Control List (ACL) to Block Tech Server from Other VLANs:
```bash
# Router 1 (Admin + Tech)
enable
conf t
access-list 110 deny icmp any host 192.168.0.41
access-list 110 permit ip any any

interface g0/0/0.10
 ip access-group 110 in
exit
# Blocks Admin (VLAN 10) devices from pinging (ICMP) the Tech Server.
```
```bash
# Router 2 (Sales)
enable
conf t
access-list 110 deny icmp any host 192.168.0.41
access-list 110 permit ip any any

interface g0/0/1.30
 ip access-group 110 in
exit
# Also blocks Sales devices from pinging (ICMP) the Tech Server.
```
# Explanation: How This Works

ICMP traffic (ping) is blocked before it ever hits the Tech VLAN interface.
Access Control List (ACL) is inbound, meaning it filters packets as they arrive on the interface.
Only Tech VLAN devices (192.168.0.32–62) can successfully ping the Tech Server.
This mimics real-world internal segmentation and security boundaries.

# Why This Set Up?

In real-world networks, it’s very common to restrict access to sensitive servers so that only specific groups (like the Tech department) can reach them.
But Why ICMP?
I chose to restrict only ICMP (ping) in this lab because:
- It’s easy to test using the ping command from PCs.
- It demonstrates the core concept of ACL filtering.
- It's a clean way to practice protocol-based filtering without needing to run actual services on the server.

In real networks, you’d usually block or allow specific services — not just ping (ICMP).
Examples of real-world services commonly restricted:
- SSH (port 22)
- RDP (port 3389)
- HTTP/HTTPS (ports 80, 443)
- SMB/File Sharing (ports 445, 139)
- MySQL, FTP, etc.
These would be restricted using extended ACLs, firewalls, or security groups depending on the environment.
## 6. Testing & Verification
# DHCP Assignment Test
### INSERT PICTURE
# Inter-VLAN Routing Test
### INSERT PICTURE
# Server Access Test
### INSERT PICTURE
# Tech Server ACL Test
### INSERT PICTURE
