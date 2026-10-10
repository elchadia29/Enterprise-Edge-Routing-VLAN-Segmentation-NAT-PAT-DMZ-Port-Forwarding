[enterprise_edge_routing_nat_pat_portfolio_readme.md](https://github.com/user-attachments/files/33272230/enterprise_edge_routing_nat_pat_portfolio_readme.md)
# Enterprise Edge Routing, VLAN Segmentation, NAT/PAT, & DMZ Port Forwarding

## 📌 Project Overview
This project demonstrates the design, configuration, and verification of an enterprise edge network infrastructure using Cisco Packet Tracer. It integrates Layer 2 VLAN segmentation, Inter-VLAN routing (Router-on-a-Stick), dynamic DHCP address allocation, Dynamic Network Address Translation with Port Address Translation (NAT Overload), and Static Port Forwarding for a secure Demilitarized Zone (DMZ) web server.

---

## 🏗️ Topology Architecture & Addressing Table

```
 [ Inside LAN ]
 VLAN 10 (Sales): 192.168.10.0/24 ──┐
 VLAN 20 (IT):    192.168.20.0/24 ──┼── [ Edge Router (R1) ] ── (203.0.113.0/30) ── [ ISP Router ]
 DMZ (Server):    172.16.1.0/29   ──┘      (G0/0/0 - Inside Subinterfaces)          (G0/0/1 - Outside)
                                            Public IP: 203.0.113.2
```

| Device | Interface | IP Address / Mask | Description |
| :--- | :--- | :--- | :--- |
| **R1 (Edge Router)** | `Gig0/0/0.10` | `192.168.10.1 / 24` | Default Gateway for VLAN 10 (Sales) |
| **R1 (Edge Router)** | `Gig0/0/0.20` | `192.168.20.1 / 24` | Default Gateway for VLAN 20 (IT) |
| **R1 (Edge Router)** | `Gig0/0/0.30` | `172.16.1.1 / 29` | Default Gateway for DMZ (VLAN 30) |
| **R1 (Edge Router)** | `Gig0/0/1` | `203.0.113.2 / 30` | WAN Interface (Public IP facing ISP) |
| **ISP Router** | `Gig0/0/1` | `203.0.113.1 / 30` | Upstream Next-Hop IP |
| **Web-Server (DMZ)** | `FastEthernet0` | `172.16.2 / 29` | Internal HTTP Server hosted in DMZ |

---

## ⚙️ Key Technical Implementations

### 1. VLAN Segmentation & Router-on-a-Stick
* Configured Access ports on `Switch0` for VLAN 10 (`Fa0/1 - Fa0/2`) and VLAN 20 (`Fa0/3 - Fa0/4`), plus VLAN 30 (`Fa0/5`) for the DMZ server.
* Configured an 802.1Q trunk link on port `Fa0/5` connecting `Switch0` to Edge Router `R1`.
* Configured subinterfaces on `R1` (`Gig0/0/0.10`, `Gig0/0/0.20`, and `Gig0/0/0.30`) to provide inter-VLAN routing.

### 2. Router-Based DHCP Services
* Excluded static and gateway addresses using `ip dhcp excluded-address`.
* Established dynamic address pools for internal corporate networks to automate endpoint provisioning:
  ```text
  ip dhcp pool vlan10_sales
   network 192.168.10.0 255.255.255.0
   default-router 192.168.10.1
   dns-server 8.8.8.8

  ip dhcp pool vlan20_it
   network 192.168.20.0 255.255.255.0
   default-router 192.168.20.1
   dns-server 8.8.8.8
  ```

### 3. Dynamic NAT with PAT (NAT Overload)
* Mapped internal private subnets (`192.168.10.0/24` and `192.168.20.0/24`) to share a single public IP address (`203.0.113.2`) for Internetbound traffic:
  ```text
  ip access-list standard NAT-ALLOWED-LAN
   permit 192.168.10.0 0.0.255.255
   permit 192.168.20.0 0.0.255.255
  
  ip nat inside source list NAT-ALLOWED-LAN interface GigabitEthernet 0/0/1 overload
  ```

### 4. Static NAT Port Forwarding (DMZ Hosting)
* Published the internal DMZ Web Server (`172.16.1.2:80`) to the public Internet via port translation on the edge router's public IP:
  ```text
  ip nat inside source static tcp 172.16.1.2 80 203.0.113.2 80
  ```

---

## 📊 Verification & CLI Proofs

### Active NAT Translation Table (`show ip nat translations`)
```text
R1# show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
tcp 203.0.113.2:80     172.16.1.2:80      203.0.113.1:1025   203.0.113.1:1025
icmp 203.0.113.2:1     192.168.20.10:1    203.0.113.1:1      203.0.113.1:1
```

### NAT Statistics (`show ip nat statistics`)
```text
R1# show ip nat statistics
Total translations: 1 (0 static, 1 dynamic, 0 extended)
Outside Interfaces: GigabitEthernet0/0/1
Inside Interfaces: GigabitEthernet0/0/0.10, GigabitEthernet0/0/0.20, GigabitEthernet0/0/0.30
Hits: 3  Misses: 4
```

---

## 🔐 Security Considerations
* **Private IP Obfuscation:** Internal corporate subnets (`192.168.x.x`) are masked behind a single public address (`203.0.113.2`), preventing external hosts from directly routing to internal workstations.
* **DMZ Isolation:** Placing the web server in a segregated subnet (`172.16.1.0/29`) ensures that even if external traffic penetrates the web service via static port forwarding, direct access to the internal enterprise LAN remains isolated.
