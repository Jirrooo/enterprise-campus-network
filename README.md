# Multi-Site Enterprise Campus Network Architecture with High Availability & Security

## Overview
Proyek ini mengimplementasikan infrastruktur jaringan skala enterprise multi-lokasi (*Headquarters* dan *Branch Office*) menggunakan pendekatan **Cisco 3-Layer Hierarchical Model** (Core, Distribution, Access). Desain ini dirancang untuk menjamin ketersediaan tinggi (*High Availability*), redundansi link, serta keamanan akses menuju area *Server Farm* (DMZ).

Seluruh arsitektur dikonfigurasi, diuji, dan diverifikasi penuh pada simulator **Cisco Packet Tracer**.

---

## Technical Highlights & Key Features

* **High Availability & Link Redundancy (Layer 2 & Layer 3)**:
  * **HSRP (Hot Standby Router Protocol)**: Redundansi *default gateway* virtual (`.254`) antar-Multilayer Switch (MLS) dengan skema *Active-Standby failover* (Priority 110/100) dan fitur *preempt*.
  * **LACP EtherChannel (IEEE 802.3ad)**: Penggabungan interface fisik (*link aggregation*) pada *trunk link* antar-Distribution Switch untuk meningkatkan *throughput* dan redundansi *port*.
  * **Spanning Tree Protocol (PVST+)**: Pencegahan *looping* Layer 2 pada jalur redundansi ke *Access Switch*.
* **VLAN Segmentation & Inter-VLAN Routing**:
  * Segregasi departemen untuk keamanan dan pembatasan *broadcast domain*.
  * Penerapan **Native VLAN 99** khusus pada port *trunk* untuk mencegah serangan *VLAN Hopping*.
  * *Switch Virtual Interface* (SVI) pada Distribution Switch untuk *Inter-VLAN Routing* berkecepatan tinggi.
* **Dynamic Routing & Multi-Site WAN Connection**:
  * **OSPF Area 0 (Single-Area OSPF)**: Distribusi rute otomatis antar *Core Router*, *Distribution Switches*, dan *Branch Router*.
  * **Passive-Interface**: Diterapkan pada port yang mengarah ke jaringan pengguna untuk keamanan dan efisiensi *bandwidth*.
* **Server Farm / DMZ Services**:
  * Penggelaran area terpisah (VLAN 50 DMZ) dengan layanan **DNS** (`portal.enterprise.com`) dan **HTTP/HTTPS Web Services**.
* **Network Hardening & Remote Management Security**:
  * Enkripsi manajemen remote menggunakan **SSH v2** dengan enkripsi RSA 1024-bit.
  * Pemasangan **Extended Access Control List (ACL 100)** untuk membatasi lalu lintas menuju area DMZ (hanya mengizinkan trafik HTTP, HTTPS, dan DNS).

---

## Network Architecture & Design Breakdown

```text
                    +-----------------------+
                    |      RT-CORE-ISP      |
                    +-----------+-----------+
                                | (WAN 172.16.10.0/30)
                                |
                    +-----------+-----------+
                    |       RT-BRANCH       |
                    +-----------+-----------+
                                |
                    +-----------+-----------+
                    |     SW-ACC-BRANCH     |
                    +-----------------------+
                                |
                         [PC-BRANCH LAN]

                                | (Point-to-Point 10.10.100.0/30)
                    +-----------+-----------+
                    |      MLS-DIST-01      | <=== HSRP Active (Priority 110)
                    +-----+-----------+-----+
                          |   LACP    |
                          | (Po1/Gi0) |
                    +-----+-----------+-----+
                    |      MLS-DIST-02      | <=== HSRP Standby (Priority 100)
                    +-----------+-----------+
                                |
               +----------------+----------------+
               |                                 |
    +----------+----------+           +----------+----------+
    |   SW-ACC-GEDUNG-A   |           |   SW-ACC-GEDUNG-B   |
    +----------+----------+           +----------+----------+
               |                                 |
        [VLAN 10, 20, 40]                   [VLAN 30, 40]
```

### Topology Visual
<img width="465" height="271" alt="topology diagram" src="https://github.com/user-attachments/assets/ec0abb8f-dfcb-4791-9665-76f374a88393" />


---

## Topology & Addressing Scheme

### 1. Enterprise Subnetting & VLAN Allocation
| VLAN ID | VLAN Name | Subnet | IP Range / Hosts | Default Gateway (HSRP IP) | Assigned Ports / Location |
| :---: | :--- | :--- | :--- | :--- | :--- |
| **VLAN 10** | HR | `192.168.10.0/24` | `192.168.10.1 - .250` | `192.168.10.254` | SW-ACC-GEDUNG-A (Fa0/1) |
| **VLAN 20** | IT & Wi-Fi | `192.168.20.0/24` | `192.168.20.1 - .250` | `192.168.20.254` | SW-ACC-GEDUNG-A (Fa0/2, AP) |
| **VLAN 30** | Finance | `192.168.30.0/24` | `192.168.30.1 - .250` | `192.168.30.254` | SW-ACC-GEDUNG-B (Fa0/1) |
| **VLAN 40** | Voice | `192.168.40.0/24` | `192.168.40.1 - .250` | `192.168.40.254` | IP Phones (Gedung A & B) |
| **VLAN 50** | Server DMZ | `10.10.50.0/24` | `10.10.50.1 - .250` | `10.10.50.254` | MLS-DIST-01 (Fa0/10, Fa0/11) |
| **VLAN 99** | Native VLAN | N/A | N/A | N/A | Trunk Port Management |

### 2. Infrastructure & WAN Interconnections
| Link Connection | Subnet | Node A IP | Node B IP | Protocol |
| :--- | :---: | :---: | :---: | :---: |
| **MLS-DIST-01 $\leftrightarrow$ RT-CORE-ISP** | `10.10.100.0/30` | `10.10.100.1` | `10.10.100.2` | OSPF Area 0 |
| **RT-CORE-ISP $\leftrightarrow$ RT-BRANCH** | `172.16.10.0/30` | `172.16.10.1` | `172.16.10.2` | OSPF Area 0 (WAN) |
| **RT-BRANCH $\leftrightarrow$ SW-ACC-BRANCH** | `192.168.100.0/24` | `192.168.100.1` | Device IP (`.10`) | OSPF Area 0 (LAN) |

---

## Key Configurations Summary

### 1. High Availability (HSRP) Configuration
Diterapkan pada SVI di `MLS-DIST-01` (Active, Priority 110) dan `MLS-DIST-02` (Standby, Priority 100 bawaan):

```text
! MLS-DIST-01 (Active Gateway Sample - VLAN 10)
interface vlan 10
 ip address 192.168.10.251 255.255.255.0
 standby 10 ip 192.168.10.254
 standby 10 priority 110
 standby 10 preempt
exit

! MLS-DIST-02 (Standby Gateway Sample - VLAN 10)
interface vlan 10
 ip address 192.168.10.252 255.255.255.0
 standby 10 ip 192.168.10.254
exit
```

### 2. Dynamic Routing (OSPF Area 0)
Konfigurasi pengiklan rute jaringan internal di `MLS-DIST-01`:

```text
router ospf 1
 router-id 1.1.1.1
 passive-interface default
 no passive-interface fastEthernet 0/24
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.30.0 0.0.0.255 area 0
 network 192.168.40.0 0.0.0.255 area 0
 network 10.10.50.0 0.0.0.255 area 0
 network 10.10.100.0 0.0.0.3 area 0
exit
```

### 3. DMZ Extended Access Control List (Security Hardening)
Diterapkan pada interface Inbound `Gi0/0` di `RT-CORE-ISP` untuk menyaring trafik menuju Server DMZ:

```text
access-list 100 permit tcp any host 10.10.50.2 eq www
access-list 100 permit tcp any host 10.10.50.2 eq 443
access-list 100 permit udp any host 10.10.50.2 eq domain
access-list 100 permit icmp any any echo-reply
access-list 100 permit icmp any any unreachable
access-list 100 permit ip any any

interface gigabitEthernet 0/0
 ip access-group 100 in
exit
```

---

## Verification & Testing Results

1. **HSRP Redundancy Verification**:
   * Menjalankan `show standby brief` pada `MLS-DIST-01` mengonfirmasi seluruh SVI (VLAN 10–50) berstatus **`Active`** dengan Virtual IP **`.254`**.
   * <img width="330" height="157" alt="hrsp verication" src="https://github.com/user-attachments/assets/4cea011c-0b54-446b-83d7-8ec6999b02a2" />

2. **OSPF Neighbor Adjacency**:
   * Menjalankan `show ip ospf neighbor` pada `RT-CORE-ISP` mengonfirmasi status **`FULL/DR`** atau **`FULL/BDR`** dengan `MLS-DIST-01` (`1.1.1.1`) dan `RT-BRANCH` (`3.3.3.3`).
   * <img width="332" height="100" alt="ospf neighbor check" src="https://github.com/user-attachments/assets/57bf4e98-fa8d-4818-9223-9ca2e674a32b" />

3. **End-to-End Connectivity & Service Testing**:
   * Berhasil melakukan *ICMP Ping* dari `PC-BRANCH` (`192.168.100.10`) melintasi koneksi WAN OSPF menuju PC HR Gedung A (`192.168.10.10`) dan Server DMZ (`10.10.50.2`).
   * Resolusi domain berhasil via Web Browser: Pengujian URL `portal.enterprise.com` dari `PC-BRANCH` berhasil menampilkan halaman Web DMZ.
   * <img width="347" height="346" alt="web dmz test" src="https://github.com/user-attachments/assets/3d6cad27-0d3f-4cde-9929-a36ec5582f1a" />

4. **Security Audit (SSH Access)**:
   * Pengujian remote CLI via terminal PC menggunakan perintah `ssh -l admin 10.10.100.2` berhasil terhubung dengan enkripsi SSH v2.

---

## Project File Structure

```text
enterprise-campus-network/
├── docs/
│   ├── topology-diagram.png
│   ├── hsrp-verification.png
│   ├── ospf-neighbor-check.png
│   └── web-dmz-test.png
├── pkt/
│   └── Enterprise-Campus-Network.pkt
├── configs/
│   ├── RT-CORE-ISP.txt
│   ├── RT-BRANCH.txt
│   ├── MLS-DIST-01.txt
│   ├── MLS-DIST-02.txt
│   ├── SW-ACC-GEDUNG-A.txt
│   └── SW-ACC-GEDUNG-B.txt
├── .gitignore
└── README.md
```
