# TechCorp Atlanta — Enterprise Network Security Lab

A full-stack enterprise network simulation built in Cisco Packet Tracer, featuring multi-VLAN segmentation, EtherChannel link aggregation, inter-VLAN routing, DHCP, Layer 2/3 security, and Cisco ASA 5505 firewall integration with ISP connectivity.

---

## Scenario

**TechCorp** is a 50-employee software company based in Atlanta, GA. This project simulates the design and implementation of their office network infrastructure — from core switching and VLAN segmentation to perimeter firewall and internet connectivity.

| Detail | Value |
|--------|-------|
| Company | TechCorp Atlanta |
| Employees | 50 |
| Departments | HR, IT Admin, Sales, Guest, Servers |
| Tool | Cisco Packet Tracer |
| Topology | 3-tier (Core / Distribution / Access) |

---

## Network Topology

```
[INTERNET]
     |
[ISP-ROUTER] — GigEth0/0 — 203.0.113.1/30
     |
[ASA-FW (5505)] — eth0/7 → VLAN2 (outside) 203.0.113.2/30
                  Vlan1 (inside)   10.0.0.1/30
     |
[R1 (2911)]  — GigEth0/1 → ASA inside  10.0.0.2/30
               GigEth0/0 → SW1-CORE    192.168.0.1/30
     |
[SW1-CORE (3560 L3)] — Fa0/24 → R1  192.168.0.2/30
     |              \
[SW2 (2960)]    [SW3 (2960)]
  |    |           |    |
PC-HR  PC-IT   PC-Sales  PC-Guest
(V10)  (V20)   (V30)     (V40)
     |
  [SRV1]
  (V50)
```

---

## VLAN Design

| VLAN | Name | Subnet | Gateway | Purpose |
|------|------|--------|---------|---------|
| 10 | HR | 192.168.10.0/24 | 192.168.10.1 | Human Resources |
| 20 | IT_ADMIN | 192.168.20.0/24 | 192.168.20.1 | IT Administration |
| 30 | SALES | 192.168.30.0/24 | 192.168.30.1 | Sales Department |
| 40 | GUEST | 192.168.40.0/24 | 192.168.40.1 | Guest WiFi |
| 50 | SERVERS | 192.168.50.0/24 | 192.168.50.1 | Web/DB Servers |
| 99 | MANAGEMENT | 192.168.99.0/24 | 192.168.99.1 | Switch Management |

---

## IP Addressing

| Device | Interface | IP Address | Description |
|--------|-----------|------------|-------------|
| ISP-ROUTER | GigEth0/0 | 203.0.113.1/30 | ISP uplink to ASA |
| ASA-FW | Vlan2 (outside) | 203.0.113.2/30 | Internet-facing |
| ASA-FW | Vlan1 (inside) | 10.0.0.1/30 | Internal-facing |
| R1 | GigEth0/1 | 10.0.0.2/30 | Link to ASA |
| R1 | GigEth0/0 | 192.168.0.1/30 | Link to SW1-CORE |
| SW1-CORE | Fa0/24 | 192.168.0.2/30 | Link to R1 |
| SW1-CORE | Vlan10 SVI | 192.168.10.1/24 | HR gateway |
| SW1-CORE | Vlan20 SVI | 192.168.20.1/24 | IT gateway |
| SW1-CORE | Vlan30 SVI | 192.168.30.1/24 | Sales gateway |
| SW1-CORE | Vlan40 SVI | 192.168.40.1/24 | Guest gateway |
| SW1-CORE | Vlan50 SVI | 192.168.50.1/24 | Servers gateway |
| SW1-CORE | Vlan99 SVI | 192.168.99.11/24 | Management |
| SW2 | Vlan99 | 192.168.99.12/24 | Management |
| SW3 | Vlan99 | 192.168.99.13/24 | Management |

---

## Features Implemented

### Layer 2
- **6 VLANs** with departmental segmentation
- **802.1Q Trunk ports** between all switches
- **EtherChannel (LACP)** — Port-Channel 1 (SW1↔SW2), Port-Channel 2 (SW1↔SW3)
- **Port Security** — sticky MAC learning, 60-minute aging on access ports
- **STP optimization** — SW1-CORE as root bridge (priority 24576), PortFast + BPDU Guard on access ports
- **Unused port disabling** — all non-active ports administratively shutdown

### Layer 3
- **Inter-VLAN routing** via SW1-CORE Layer 3 SVI interfaces
- **DHCP pools** for HR, IT, Sales, and Guest VLANs (first 10 IPs excluded per subnet)
- **Static routing** — default routes chained: SW1-CORE → R1 → ASA → ISP
- **SSH v2** — all devices (Telnet disabled)

### Security
- **ACL-based traffic filtering** on SW1-CORE:
  - HR cannot reach Sales (HR_TO_SALES_DENY)
  - Sales cannot reach HR (SALES_TO_HR_DENY)
  - Guest isolated from all internal VLANs (GUEST_INTERNET_ONLY)
- **Cisco ASA 5505 Firewall:**
  - Security zones: OUTSIDE (level 0) / INSIDE (level 100)
  - Outside → Inside: blocked by default (INSIDE_NEW / OUTSIDE_NEW ACLs)
  - Inside → Outside: HTTP, HTTPS, DNS, ICMP permitted
  - ICMP explicitly permitted through OUTSIDE_NEW for connectivity testing
  - NAT (PAT) configured — internal 192.168.0.0/16 translated to outside interface

### Management
- **Banner MOTD** on R1 and SW3
- **Enable secret** + **service password-encryption**
- **SSH v2** enabled on all devices
- **Console/VTY access control** with local authentication

---

## ASA 5505 — Important Configuration Note

The Cisco ASA 5505 uses a built-in switch fabric. Physical ports **cannot** be assigned `nameif` directly — they must first be assigned to a VLAN, then the VLAN interface is configured:

```
! Step 1 — assign physical port to VLAN
interface ethernet 0/7
 switchport access vlan 2

! Step 2 — configure VLAN interfaces
interface vlan 2
 nameif outside
 security-level 0
 ip address 203.0.113.2 255.255.255.252

interface vlan 1
 nameif inside
 security-level 100
 ip address 10.0.0.1 255.255.255.252
```

> **Connection detail:** ISP-ROUTER **GigabitEthernet0/0** (203.0.113.1) connects to **ASA eth0/7** (VLAN 2 / outside, 203.0.113.2).

> **ICMP note:** By default the ASA blocks ICMP from outside. During testing, an explicit `permit icmp any any` was added to the OUTSIDE_NEW ACL so that connectivity tests (ping) between the ISP router and the firewall succeed. In production this would be tightened.

---

## Security Policy Summary

| Source | Destination | Action |
|--------|-------------|--------|
| HR (VLAN 10) | Sales (VLAN 30) | DENY |
| Sales (VLAN 30) | HR (VLAN 10) | DENY |
| Guest (VLAN 40) | All internal VLANs | DENY |
| All internal | Internet (HTTP/HTTPS/DNS) | PERMIT |
| Outside | Inside | DENY (ASA default) |
| ICMP (testing) | Both directions | PERMIT |

---

## Testing Results

| Test | Result |
|------|--------|
| PC1-HR → DHCP IP | ✅ 192.168.10.11 |
| PC1-HR → PC2-IT (inter-VLAN) | ✅ 0% loss |
| PC1-HR → PC3-Sales (ACL) | ❌ 100% loss (blocked) |
| PC4-Guest → PC1-HR (ACL) | ❌ 100% loss (blocked) |
| ASA-FW → ISP-ROUTER | ✅ 100% success |
| ISP-ROUTER → ASA-FW | ✅ 100% success |
| EtherChannel Po1 + Po2 | ✅ SU (LACP active) |
| Port Security SW2 Fa0/10 | ✅ Secure-up, sticky MAC |

---

## Project Structure

```
network-security-lab/
├── README.md
├── TechCorp-Network.pkt
├── docs/
│   ├── network-diagram.png
│   └── screenshots/
│       ├── 01-topology.png
│       ├── 02-vlan-brief.png
│       ├── 03-etherchannel-status.png
│       ├── 04-dhcp-test.png
│       ├── 05-inter-vlan-ping.png
│       ├── 08-port-security.png
│       ├── 09-acl-hr-sales-denied.png
│       ├── 10-acl-hr-it-allowed.png
│       ├── 11-guest-isolated.png
│       ├── 12-asa-to-isp.png
│       ├── 13-isp-to-asa.png
│       └── 14-asa-interfaces.png
└── configs/
    ├── ISP-ROUTER.txt
    ├── ASA-FW.txt
    ├── R1.txt
    ├── SW1-CORE.txt
    ├── SW2.txt
    └── SW3.txt
```

---

## Future Improvements

- DMZ zone for public-facing servers
- VPN tunnel between branch offices
- Dynamic routing (OSPF) replacing static routes
- Cisco ASA replaced with next-gen firewall (FTD) for IPS/IDS
- Syslog server for centralized logging
- SNMP monitoring integration
- IPv6 dual-stack deployment
- Tighten ICMP rules after testing phase

---

## Author

**Emre Alli**
CompTIA Security+ Certified | A.A.S. Cybersecurity — Gwinnett Technical College (GPA 3.76)
GitHub: [@ealli6615](https://github.com/ealli6615)
