<div align="center">
  <h1>🌐 Arsitektur Jaringan Enterprise Terkonvergensi</h1>
  <p>
    <b>Simulasi Cisco Packet Tracer tingkat lanjut dengan OSPF Area 0, VoIP (CME), segmentasi VLAN, dan routing P2P berbasis VLSM.</b>
  </p>

  ![Cisco](https://img.shields.io/badge/Cisco-Packet_Tracer_8.x-1BA0D7?style=for-the-badge&logo=cisco)
  ![Routing](https://img.shields.io/badge/Dynamic_Routing-OSPF_Area_0-FF6600?style=for-the-badge)
  ![VoIP](https://img.shields.io/badge/Telephony-Cisco_CME-32CD32?style=for-the-badge)
  ![VLSM](https://img.shields.io/badge/Subnetting-VLSM_%2F30-8A2BE2?style=for-the-badge)

  [🇬🇧 English](README.md) | **🇮🇩 Bahasa Indonesia**
</div>

<br />

## 📸 Diagram Topologi Jaringan
<img width="902" height="365" alt="Diagram Topologi Jaringan" src="https://github.com/user-attachments/assets/71ba9623-38b1-4191-9008-c0e2aa8b65da" />

## 📖 Gambaran Proyek
Repositori ini berisi simulasi topologi jaringan skala enterprise yang berfungsi penuh. Arsitekturnya dibangun di atas **Ring Backbone dengan High-Availability** yang memakai 3 router (Dev, CS, dan DATA). Proyek ini mendemonstrasikan praktik standar industri, mulai dari *Variable Length Subnet Masking* (VLSM) untuk link WAN point-to-point, dynamic routing, sampai segmentasi trafik yang ketat antara jaringan Data dan Voice.

### 🌟 Teknologi Inti yang Diimplementasikan
* **Dynamic Routing (OSPF Area 0):** Konvergensi cepat dan pemilihan jalur dinamis di seluruh backbone WAN dengan *wildcard mask* `0.0.0.3`.
* **IP Telephony (VoIP):** Cisco CallManager Express (CME) terpusat dengan DHCP Option 150 untuk provisioning IP Phone dan pengikatan perangkat berdasarkan MAC address.
* **Segmentasi VLAN:** Isolasi ketat antar departemen (Full Stack Dev, Data Science, Customer Service) dan trafik suara (Voice VLAN).
* **VLSM (Point-to-Point):** Alokasi IP efisien dengan subnet `/30` (`255.255.255.252`) pada koneksi Serial antar-router, sehingga tidak ada IP yang terbuang atau saling tumpang tindih.

---

## 🗺️ Arsitektur Pengalamatan IP & Segmentasi

| Segmen / Departemen | VLAN ID | Network Address | Subnet Mask | Default Gateway |
| :--- | :---: | :--- | :--- | :--- |
| **Full Stack Dev** | VLAN 10 | `192.168.10.0/24` | `255.255.255.0` | `192.168.10.1` |
| **Customer Service (Data)** | VLAN 20 | `192.168.20.0/24` | `255.255.255.0` | `192.168.20.1` |
| **Data Scientist** | VLAN 30 | `192.168.30.0/24` | `255.255.255.0` | `192.168.30.1` |
| **CS Telephony (Voice)** | VLAN 50 | `192.168.50.0/24` | `255.255.255.0` | `192.168.50.1` |
| **Link WAN (Dev ↔ CS)** | N/A | `200.20.20.0/30` | `255.255.255.252` | N/A |
| **Link WAN (Dev ↔ Data)** | N/A | `200.10.10.0/30` | `255.255.255.252` | N/A |
| **Link WAN (CS ↔ Data)** | N/A | `200.30.30.0/30` | `255.255.255.252` | N/A |

---

## 🛠️ Logika Network Engineering & Konfigurasi (CLI)

Bagian berikut berisi perintah konfigurasi murni yang diterapkan pada 3 router enterprise utama. Default bawaan IOS yang dibuat otomatis sudah dihapus agar lebih ringkas.

### 💻 1. ROUTER DEV (Kiri Atas)
**Peran:** Menangani departemen Full Stack Developer dan menjadi pemula (initiator) ring OSPF.

**Efisiensi VLSM:** Interface Serial memakai subnet mask `/30` (`255.255.255.252`) yang hanya menyediakan 2 IP usable per link, sehingga tidak ada pemborosan IP pada koneksi WAN.

**Presisi OSPF:** Perintah `network` memakai *wildcard mask* yang tepat (`0.0.0.3` untuk WAN dan `0.0.0.255` untuk LAN) agar OSPF tidak mengiklankan subnet yang salah.

```text
! --- KONFIGURASI DHCP POOL ---
ip dhcp pool FullStack-Dev
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1

! --- SUB-INTERFACE LAN ---
interface GigabitEthernet0/0
 no shutdown

interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0

! --- INTERFACE SERIAL WAN (/30 VLSM) ---
interface Serial0/3/0
 ip address 200.10.10.1 255.255.255.252
 no shutdown

interface Serial0/3/1
 ip address 200.20.20.1 255.255.255.252
 no shutdown

! --- DYNAMIC ROUTING OSPF ---
router ospf 15
 router-id 1.1.1.1
 network 192.168.10.0 0.0.0.255 area 0
 network 200.10.10.0 0.0.0.3 area 0
 network 200.20.20.0 0.0.0.3 area 0
```

<br>

### 📞 2. ROUTER CS (Tengah Bawah)

**Peran:** Node paling kompleks di topologi ini. Berfungsi sebagai server PBX (Cisco CallManager Express) sekaligus me-routing data lewat skema Router-on-a-Stick.

**Router-on-a-Stick:** Interface fisik `FastEthernet0/0` dibagi secara logis menjadi sub-interface (`.20` dan `.50`) menggunakan enkapsulasi `802.1Q`. Dengan begitu, trafik Voice terisolasi dari trafik Data dan *broadcast storm* bisa dicegah.

**DHCP Option 150:** Parameter ini mengarahkan IP Phone ke alamat IP router untuk mengunduh firmware dan konfigurasi direktori lewat TFTP.

```text
! --- DHCP POOL (DATA & VOICE) ---
ip dhcp pool VOICE
 network 192.168.50.0 255.255.255.0
 default-router 192.168.50.1
 option 150 ip 192.168.50.1

ip dhcp pool CS
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1

! --- SUB-INTERFACE LAN (ROUTER-ON-A-STICK) ---
interface FastEthernet0/0
 no shutdown

interface FastEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0

interface FastEthernet0/0.50
 encapsulation dot1Q 50
 ip address 192.168.50.1 255.255.255.0

! --- INTERFACE SERIAL WAN (/30 VLSM) ---
interface Serial0/3/0
 ip address 200.20.20.2 255.255.255.252
 clock rate 2000000
 no shutdown

interface Serial0/3/1
 ip address 200.30.30.1 255.255.255.252
 clock rate 2000000
 no shutdown

! --- DYNAMIC ROUTING OSPF ---
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

! Directory Number (Nomor Ekstensi)
ephone-dn 1
 number 101
ephone-dn 2
 number 102
ephone-dn 3
 number 103
```
<br>

### 📊 3. ROUTER DATA (Kanan Atas)
**Peran:** Menangani departemen Data Scientist dan menutup arsitektur OSPF yang redundan.


```text
! --- KONFIGURASI DHCP POOL ---
ip dhcp pool Data
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1

! --- SUB-INTERFACE LAN ---
interface GigabitEthernet0/0
 no shutdown

interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0

! --- INTERFACE SERIAL WAN (/30 VLSM) ---
interface Serial0/3/0
 ip address 200.10.10.2 255.255.255.252
 no shutdown

interface Serial0/3/1
 ip address 200.30.30.2 255.255.255.252
 no shutdown

! --- DYNAMIC ROUTING OSPF ---
router ospf 15
 router-id 3.3.3.3
 network 192.168.30.0 0.0.0.255 area 0
 network 200.10.10.0 0.0.0.3 area 0
 network 200.30.30.0 0.0.0.3 area 0
```

---

## 🚀 Cara Menjalankan Simulasi
1. Buka file `.pkt` menggunakan **Cisco Packet Tracer**.
2. Tunggu sekitar 15–30 detik sampai konvergensi Spanning Tree Protocol (STP) selesai dan adjacency OSPF terbentuk.
3. **Verifikasi:**
   - Buka terminal salah satu PC lalu jalankan `ping 192.168.30.x` untuk menguji routing antar-departemen.
   - Buka GUI dua IP Phone dan hubungi ekstensi yang sudah terdaftar (misalnya `101` ke `102`).
   - Jalankan `show ip route` atau `show ip ospf neighbor` di CLI router untuk memverifikasi tabel routing dinamis.
