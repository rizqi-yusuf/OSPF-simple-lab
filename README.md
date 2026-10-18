<div align="center">
  <h1>🌐 Enterprise Converged Network Architecture</h1>
  <p>
    <b>Advanced Cisco Packet Tracer Simulation featuring OSPF Area 0, VoIP (CME), VLAN Segmentation, and VLSM P2P Routing.</b>
  </p>
  
  ![Cisco](https://img.shields.io/badge/Cisco-Packet_Tracer_8.x-1BA0D7?style=for-the-badge&logo=cisco)
  ![Routing](https://img.shields.io/badge/Dynamic_Routing-OSPF_Area_0-FF6600?style=for-the-badge)
  ![VoIP](https://img.shields.io/badge/Telephony-Cisco_CME-32CD32?style=for-the-badge)
  ![VLSM](https://img.shields.io/badge/Subnetting-VLSM_%2F30-8A2BE2?style=for-the-badge)
</div>

<br />

## 📸 Network Topology Diagram
<img width="894" height="363" alt="{64A8C974-07CA-42CD-89F5-009545F2531B}" src="https://github.com/user-attachments/assets/433ead00-ceee-4d13-a705-bd2aa1c009fd" />


## 📖 Project Overview
This repository contains a fully functional, enterprise-grade network topology simulation. The architecture is built on a **High-Availability Ring Backbone** using 3 routers (Dev, CS, and DATA). It demonstrates industry-standard practices including Variable Length Subnet Masking (VLSM) for point-to-point WAN links, dynamic routing, and strict traffic segmentation between Data and Voice networks.

### 🌟 Core Technologies Implemented
* **Dynamic Routing (OSPF Area 0):** Fast convergence and dynamic path selection across the WAN backbone using `0.0.0.3` wildcard masks.
* **IP Telephony (VoIP):** Centralized Cisco CallManager Express (CME) with DHCP Option 150 for IP Phone provisioning and precise MAC-address hardware binding.
* **VLAN Segmentation:** Strict isolation between organizational departments (Full Stack Dev, Data Science, Customer Service) and Voice traffic (Voice VLAN).
* **VLSM (Point-to-Point):** Efficient IP allocation utilizing `/30` (`255.255.255.252`) subnets for inter-router Serial connections to prevent IP overlap.

---

## 🗺️ IP Addressing & Segmentation Architecture

| Segment / Department | VLAN ID | Network Address | Subnet Mask | Default Gateway |
| :--- | :---: | :--- | :--- | :--- |
| **Full Stack Dev** | VLAN 10 | `192.168.10.0/24` | `255.255.255.0` | `192.168.10.1` |
| **Customer Service (Data)** | VLAN 20 | `192.168.20.0/24` | `255.255.255.0` | `192.168.20.1` |
| **Data Scientist** | VLAN 30 | `192.168.30.0/24` | `255.255.255.0` | `192.168.30.1` |
| **CS Telephony (Voice)** | VLAN 50 | `192.168.50.0/24` | `255.255.255.0` | `192.168.50.1` |
| **WAN Link (Dev ↔ CS)** | N/A | `200.20.20.0/30` | `255.255.255.252` | N/A |
| **WAN Link (Dev ↔ Data)** | N/A | `200.10.10.0/30` | `255.255.255.252` | N/A |
| **WAN Link (CS ↔ Data)** | N/A | `200.30.30.0/30` | `255.255.255.252` | N/A |

---

## 🛠️ Network Engineering Logic & Configurations (CLI)

The following sections contain the pure configuration commands deployed across the 3 core Enterprise Routers. Auto-generated IOS defaults have been stripped for clarity.

### 💻 1. ROUTER DEV (Top Left)
**Role:** Handles the Full Stack Developer department and initiates the OSPF Ring.

**VLSM Efficiency:** The Serial interfaces use a `/30` Subnet Mask (`255.255.255.252`) which strictly allocates only 2 usable IPs per link, eliminating IP wastage on the WAN connections.

**OSPF Precision:** The `network` command utilizes exact Wildcard Masks (`0.0.0.3` for WAN and `0.0.0.255` for LAN) to prevent OSPF from advertising incorrect subnets.

```text
! --- DHCP POOL CONFIGURATION ---
ip dhcp pool FullStack-Dev
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1

! --- LAN SUB-INTERFACES ---
interface GigabitEthernet0/0
 no shutdown

interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0

! --- WAN SERIAL INTERFACES (/30 VLSM) ---
interface Serial0/3/0
 ip address 200.10.10.1 255.255.255.252
no shutdown

interface Serial0/3/1
 ip address 200.20.20.1 255.255.255.252
 no shutdown

! --- OSPF DYNAMIC ROUTING ---
router ospf 15
 router-id 1.1.1.1
 network 192.168.10.0 0.0.0.255 area 0
 network 200.10.10.0 0.0.0.3 area 0
 network 200.20.20.0 0.0.0.3 area 0
```

<br>

### 📞 2. ROUTER CS (Bottom Center)

**Role:** The most complex node in the topology. Acts as the PBX Server (Cisco CallManager Express) while also routing Data via a Router-on-a-Stick setup.

**Router-on-a-Stick:** Physical interface `FastEthernet0/0` is logically divided into sub-interfaces (`.20` and `.50`) using `802.1Q` encapsulation. This securely isolates the Voice traffic from the Data traffic to prevent broadcast storms.

**DHCP Option 150:** This parameter directs the IP Phones to the router's IP address to download their firmware and directory configurations via TFTP.

```text
! --- DHCP POOLS (DATA & VOICE) ---
ip dhcp pool VOICE
 network 192.168.50.0 255.255.255.0
 default-router 192.168.50.1
 option 150 ip 192.168.50.1

ip dhcp pool CS
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1

! --- LAN SUB-INTERFACES (ROUTER-ON-A-STICK) ---
interface FastEthernet0/0
 no shutdown

interface FastEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0

interface FastEthernet0/0.50
 encapsulation dot1Q 50
 ip address 192.168.50.1 255.255.255.0

! --- WAN SERIAL INTERFACES (/30 VLSM) ---
interface Serial0/3/0
 ip address 200.20.20.2 255.255.255.252
 clock rate 2000000
 no shutdown

interface Serial0/3/1
 ip address 200.30.30.1 255.255.255.252
 clock rate 2000000
 no shutdown

! --- OSPF DYNAMIC ROUTING ---
router ospf 15
 router-id 2.2.2.2
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.50.0 0.0.0.255 area 0
 network 200.20.20.0 0.0.0.3 area 0
 network 200.30.30.0 0.0.0.3 area 0

! --- CISCO CALLMANAGER EXPRESS (CME) ---
telephony-service
 max-ephones 5
 max-dn 5
 ip source-address 192.168.50.1 port 2000
 auto assign 1 to 5

! Directory Numbers (Extensions)
ephone-dn 1
 number 101
ephone-dn 2
 number 102
ephone-dn 3
 number 103
```
<br>

### 📊 3. ROUTER DATA (Top Right)
**Role:** Handles the Data Scientist department and closes the redundant OSPF architecture.


```text
! --- DHCP POOL CONFIGURATION ---
ip dhcp pool Data
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1

! --- LAN SUB-INTERFACES ---
interface GigabitEthernet0/0
 no shutdown

interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0

! --- WAN SERIAL INTERFACES (/30 VLSM) ---
interface Serial0/3/0
 ip address 200.10.10.2 255.255.255.252
 no shutdown

interface Serial0/3/1
 ip address 200.30.30.2 255.255.255.252
 no shutdown

! --- OSPF DYNAMIC ROUTING ---
router ospf 15
 router-id 3.3.3.3
 network 192.168.30.0 0.0.0.255 area 0
 network 200.10.10.0 0.0.0.3 area 0
 network 200.30.30.0 0.0.0.3 area 0
```

---

## 🚀 How to Run the Simulation
1. Open the `.pkt` file using **Cisco Packet Tracer**.
2. Wait approximately 15-30 seconds for Spanning Tree Protocol (STP) convergence and OSPF Adjacency establishment.
3. **Verification:**
   - Open a PC terminal and run `ping 192.168.30.x` to test inter-departmental routing.
   - Open the GUI of two IP Phones and dial the registered extension (e.g., `101` to `102`).
   - Run `show ip route` or `show ip ospf neighbor` in the Router CLI to verify dynamic routing tables.
