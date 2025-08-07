# Small Business Network Lab (Cisco Packet Tracer)

This project simulates a real-world small business network using Cisco Packet Tracer. It includes VLAN segmentation, inter-VLAN routing, DHCP, static IP assignments, router-on-a-stick topology, and ACLs to restrict server access.

![Small Business Network Topology](https://github.com/user-attachments/assets/1466f3a1-9ca9-43b8-ae28-7fa686a089cf)

---

## Objective

Create a secure, segmented, and functional small business network using multiple VLANs, multiple routers, DHCP, and ACLs for access control. Demonstrate:

- VLAN creation and port assignment
- Router-on-a-stick (subinterfaces)
- Inter-VLAN routing between routers
- DHCP for dynamic IP assignment
- Static IPs for servers and switches
- Access Control Lists (ACLs) to control ICMP access to critical servers

---

## Network Design Overview

### VLAN / Device IP Reference Table

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

# 1. Subnetting Breakdown

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

# 2. VLAN Configuration

## Switch 1 (Admin + Tech):
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
# 3. Inter-VLAN Routing Configuration
## Router 1:
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
## Router 2:
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
# 4. DHCP Configuration
## Router 1 (Admin + Tech):
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
## Router 2 (Sales):
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
# 5. Server Configurations
### Shared Server – VLAN 10 (Accessible to All):
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
### Tech Server - VLAN 20 (Tech-only access):
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
## Explanation: How This Works

ICMP traffic (ping) is blocked before it ever hits the Tech VLAN interface.
Access Control List (ACL) is inbound, meaning it filters packets as they arrive on the interface.
Only Tech VLAN devices (192.168.0.32–62) can successfully ping the Tech Server.
This mimics real-world internal segmentation and security boundaries.

## Why This Set Up?

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
# 6. Testing & Verification
## DHCP Assignment Test
Here’s an example showing two PCs — Tech PC 1 and Sales PC 1 — using the ipconfig command to verify their DHCP-assigned IP addresses. Only two are shown here as examples, but all VLANs were successfully assigned addresses from their correct DHCP pools.

Tech PC 1 IP Configuration:

![Tech PC 1 IP Configuration](https://github.com/user-attachments/assets/6c38e075-0a19-4bab-9904-c91997c228cd)

Sales PC 1 IP Configuration:

![Sales PC 1 IP Configuration](https://github.com/user-attachments/assets/e18d7f74-8c0f-4177-8637-01233366091c)

Successfully received IP addresses via DHCP:
- Admin PC 1: 192.168.0.12
- Admin PC 2: 192.168.0.11
- Tech PC 1: 192.168.0.34
- Tech PC 2: 192.168.0.35
- Sales PC 1: 192.168.0.72
- Sales PC 2: 192.168.0.71
This confirms DHCP is correctly configured per VLAN.
## Inter-VLAN Routing Test
Here’s an example showing Admin PC 1 and Sales PC 1 successfully pinging Tech PC 1, using the ping command. Only two devices are shown here as examples, but devices across VLANs are able to communicate as expected through inter-VLAN routing.

Admin PC 1 successfully pinging Tech PC 1:

![Adminpingtech](https://github.com/user-attachments/assets/5afc3c6c-e1f7-49a0-a880-d830bd74f559)

Sales PC 1 successfully pinging Tech PC 1:

![Salespingtech](https://github.com/user-attachments/assets/28c5b0e0-4717-4fce-bbe1-94854fe777db)

Successful inter-VLAN communication:

- Admin VLAN ↔ Tech VLAN ✅
- Tech VLAN ↔ Sales VLAN ✅
- Sales VLAN ↔ Admin VLAN ✅

This confirms that inter-VLAN routing is working correctly through the router-on-a-stick configuration.
## Server Access Test
Here’s an example showing one PC from each VLAN successfully pinging the Shared Server using the ping command. This demonstrates that the server is reachable from all departments.

Admin PC 1 successfully pinging Shared Server:

![Adminpingsharedserver](https://github.com/user-attachments/assets/ae6f676c-dcc5-4781-a316-649b373d113c)

Tech PC 1 successfully pinging Shared Server:

![Techpingsharedserver](https://github.com/user-attachments/assets/27d97373-ff18-44f5-86cf-cbb729de1ef0)

Sales PC 1 successfully pinging Shared Server:

![Salespingsharedserver](https://github.com/user-attachments/assets/5216eb2b-2a8e-40b8-8616-00b3f6c8ceb0)

In addition to ping tests, HTTP access was verified. PCs across VLANs were able to open the server’s hosted webpage using a browser.

Shared Server Webpage (viewed from Admin PC 1):

![Adminsharedserverweb](https://github.com/user-attachments/assets/33b8b8b5-4f5a-46e0-ba9a-b110ea9246eb)

Successful server access:

Admin VLAN → Shared Server (Ping + HTTP) ✅

Tech VLAN → Shared Server (Ping + HTTP) ✅

Sales VLAN → Shared Server (Ping + HTTP) ✅

This confirms that the Shared Server is accessible and its web services are functioning correctly across all VLANs.
## Tech Server ACL Test
Here’s an example showing Tech PC 1 successfully pinging the Tech Server, while Admin PC 1 and Sales PC 1 are blocked as expected. This confirms that the extended ACL is properly restricting access to the Tech Server based on VLAN.

Tech PC 1 pinging Tech Server (allowed):

![TechpingTechserver](https://github.com/user-attachments/assets/482dee48-e98e-4196-b3ac-842e583fc689)

Admin PC 1 pinging Tech Server (denied):

![AdminpingTechserver](https://github.com/user-attachments/assets/8bd7a8eb-46d6-492a-8664-cab137f631f3)

Sales PC 1 pinging Tech Server (denied):

![SalespingTechserver](https://github.com/user-attachments/assets/c3cd44ca-26ff-40cc-9c66-add228ef06c6)

Tech Server ACL Results:

✅ Tech VLAN → Access allowed

❌ Admin VLAN → Access denied

❌ Sales VLAN → Access denied

This verifies that the extended ACL is successfully enforcing security by allowing only the Tech VLAN to access the Tech Server, while blocking others.
# 7. Conclusion

This project simulated a small business network using VLAN segmentation, inter-VLAN routing, DHCP, and access control lists (ACLs). I configured and tested:

- Dynamic IP assignment using DHCP per VLAN
- Inter-VLAN communication using router-on-a-stick
- Shared server access (Ping + HTTP) from all VLANs
- Restricted Tech Server access using extended ACLs

All configurations were verified through `ipconfig`, `ping`, and HTTP browser tests. Screenshots were included for key devices across VLANs.

## Key Takeaways:
- I gained hands-on experience with VLAN setup, routing, DHCP pools, and ACL security.
- I learned how to troubleshoot VLAN communication issues, DHCP assignment problems, and ACL behavior.
- I reinforced my understanding of network segmentation, subnetting, and access control in a realistic setup.
- I used real-world logic to simulate a secure and functional network.

## What I’d Do Next:
- Expand the network topology to include additional departments or remote locations
- Incorporate wireless communication, possibly using a mesh system for better coverage
- Add redundancy and backups to improve reliability (e.g., secondary routers or Layer 3 switches)
- Enhance security by applying more specific ACLs to restrict access by protocol, port, or IP
- Simulate additional services (e.g., DNS, remote management, or VPN access)

This project helped me connect theoretical networking concepts to real configurations and problem-solving workflows. It strengthened my interest in pursuing networking as a career path and showed me what I’m capable of building with persistence and curiosity.
