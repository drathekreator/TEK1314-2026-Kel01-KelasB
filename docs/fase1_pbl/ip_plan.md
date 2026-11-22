# IP Address Plan — Kelompok 01 (Kelas B)

**Mata Kuliah:** TEK1314 - Keamanan Siber  
**Fase:** Design & Implementation Phase  
**Subnet Jaringan Internal (`sim-cyber-net`):** `192.168.1.0/24`  
**Subnet Jaringan Management (Host-Only):** `192.168.56.0/24`  
**Netmask:** `255.255.255.0`  
**Gateway Segmen Internal:** `192.168.1.1`  

---

## 1. Tabel Alokasi IP & Interface Node

| Hostname              | IP Internal          | IP Management (Host-Only) | OS / Platform              | Peran Sistem                         | 
|                       | (`sim-cyber-net`)    |                           |                            |                                      |
|-----------------------|----------------------|---------------------------|----------------------------|--------------------------------------|
| **server-target**     | `192.168.1.5`        | `192.168.56.10`           | Ubuntu Server 22.04 LTS    | Deployment Target / Blue Team Asset  |
| **cyber-workstation** | `192.168.1.20`       | Dynamic (NAT)             | Cybersecurity LabVM / Kali | Red Team / Attacker Node             |
| **sec-onion**         | Promiscuous (`eth0`) | `192.168.56.30`           | Security Onion 2           | Blue Team — NSM / NIDS / SIEM        |
| **iot-node**          | `192.168.1.40`       | Dynamic (NAT)             | Alpine Linux v3.24         | Telemetry Generator (MQTT Publisher) |
| **gateway-sim**       | `192.168.1.1`        | —                         | Virtual Router / Switch    | Gateway Segmen Internal (L3)         |

---

## 2. Arsitektur Jaringan & Layanan

### A. Alokasi Port & Service pada Target Server (`192.168.1.5`)
* **SSH Remote Access:** Port `22/TCP` (Management & Security Testing)
* **Web Server (Nginx Reverse Proxy):** Port `80/TCP` & `443/TCP`
* **MQTT Broker (Mosquitto Container):** Port `1883/TCP` (Target Telemetri IoT)
* **Database (PostgreSQL Container):** Port `5432/TCP` (Internal Data Store)

### B. Catatan Skenario Monitoring & Pengujian (Blue Team vs Red Team)
1. **Out-of-Band Monitoring:** Security Onion memisahkan interface pemantauan log (*Management Interface* `192.168.56.30`) dengan interface menyadap paket (*Sniffing Interface* `eth0`). Interface `eth0` dipasang pada port *Promiscuous / SPAN* segmen `sim-cyber-net` untuk merekam lalu lintas tanpa mengganggu jalur komunikasi utama.
2. **Attacker Node Identification:** Red Team dikonfigurasi dengan IP statis `192.168.1.20` untuk memudahkan korelasi *alert* pada Suricata dan Zeek di dashboard Security Onion.
3. **IoT Telemetry Traffic:** `IoT-Node` (`192.168.1.40`) secara rutin mengalirkan payload JSON (Suhu, Kelembapan, THI, NH3) ke MQTT Broker (`192.168.1.5:1883`) sebagai *baseline* trafik normal jaringan.
