# System Hardening Plan — Blue Team Infrastructure Defense

**Mata Kuliah:** TEK1314 - Keamanan Siber  
**Fase:** Implementation & Defense Phase  
**Dokumen:** Hardening Plan & Operational Security Baseline  
**Target Infrasruktur:** Target Server (Ubuntu Server 22.04 LTS), IoT Node (Alpine Linux), & Network Perimeter  

---

## 1. Ringkasan Eksekutif & Tujuan

Dokumen ini disusun sebagai panduan standar operasional bagi **Blue Team** dalam mengamankan, mengkonfigurasi, dan memperkuat (*hardening*) infrastruktur sistem target dan node IoT pada lingkungan simulasi lab. Tujuan utama hardening ini adalah:
1. Meminimalkan permukaan serangan (*Attack Surface Reduction*).
2. Memerapkan prinsip *Least Privilege* dan *Defense-in-Depth*.
3. Mencegah eksploitasi berbasis konfigurasi bawaan (*Default Configuration Vulnerabilities*).
4. Memastikan keterlihatan log (*Log Visibility*) untuk analisis korelasi pada SIEM / Security Onion.

---

## 2. Cakupan Aset Target (*Target Scope*)

| Hostname | Peran System | Komponen / Layanan Kunci | Prioritas Hardening |
| :--- | :--- | :--- | :--- |
| **`target-server`** | Deployment Target | SSH (22), Nginx (80), Mosquitto (1883), PostgreSQL (5432) | **Kritis (Tinggi)** |
| **`iot-node`** | Data Telemetry Generator | Python Telemetry Daemon, OpenRC Service | **Sedang** |
| **`sec-onion`** | SIEM / NIDS | Elastic/Kibana, Suricata, Zeek | **Sedang (Fokus Integrasi Log)** |

---

## 3. Matriks Matrikasi & Domain Hardening

### Domain 1: Manajemen Akses & Akun Sistem (*Identity & Access Control*)

* **Tujuan:** Mencegah akses tak terotorisasi via SSH dan membatasi eskalasi hak akses (*Privilege Escalation*).
* **Rencana Tindakan:**
  * Nonaktifkan login langsung akun `root` via SSH.
  * Terapkan pembatasan percobaan login SSH (*MaxAuthTries*).
  * Hapus akun pengguna yang tidak digunakan atau tidak relevan.
  * Terapkan kebijakan password yang kuat.

* **Perintah Dasar & Evaluasi:**
  ```bash
  # Memeriksa daftar akun berkemampuan login shell
  cat /etc/passwd | grep -E '/bin/bash|/bin/sh'

  # Menguji hak akses sudo akun aktif
  sudo -l

  # Memeriksa status service SSH
  sudo systemctl status ssh
  ```

---

### Domain 2: Keamanan Jaringan & Pembatasan Port (*Network & Perimeter Hardening*)

* **Tujuan:** Membatasi paparan port layanan hanya ke subnet yang sah dan mencegah lalu lintas yang tidak dikenal.
* **Rencana Tindakan:**
  * Aktifkan **UFW (Uncomplicated Firewall)** dengan kebijakan *Default Deny Incoming*.
  * Batasi akses SSH (Port 22) hanya dari subnet Host-Only / Management (`192.168.56.0/24`).
  * Izinkan trafik HTTP (80) dan MQTT (1883) hanya dari subnet internal lab (`192.168.1.0/24`).
  * Matikan *IP Forwarding* pada OS yang tidak bertindak sebagai router/gateway.

* **Perintah Dasar & Evaluasi:**
  ```bash
  # Menampilkan status firewall dan daftar aturan aktif
  sudo ufw status verbose

  # Menampilkan seluruh port jaringan yang sedang mendengarkan (listening)
  sudo ss -tulpn

  # Memeriksa parameter IP forwarding pada kernel
  sysctl net.ipv4.ip_forward
  ```

---

### Domain 3: Penguatan Parameter Kernel & Proteksi Network Stack (*Kernel Sysctl Hardening*)

* **Tujuan:** Melindungi sistem operasi dari serangan DoS/DDoS (seperti SYN Flood), IP Spoofing, dan pembajakan rute ICMP.
* **Rencana Tindakan:**
  * Aktifkan *TCP SYN Cookies*.
  * Aktifkan *Reverse Path Filtering* untuk mencegah paket dengan alamat asal palsu (*IP Spoofing*).
  * Abaikan pesan ICMP Redirects dan broadcast ping.

* **Perintah Dasar & Evaluasi:**
  ```bash
  # Mengaplikasikan perubahan parameter kernel tanpa reboot
  sudo sysctl --system

  # Memeriksa status aktif TCP SYN Cookies
  sysctl net.ipv4.tcp_syncookies
  ```

---

### Domain 4: Isolasi Container & Keamanan Microservice (*Service Hardening*)

* **Tujuan:** Memastikan kontainer Docker berjalan terisolasi dan tidak mengakses *socket* host secara tidak aman.
* **Rencana Tindakan:**
  * Batasi akses port database PostgreSQL (`5432`) agar hanya dapat diakses secara internal antar-kontainer (*Docker Network*), tidak diekspos ke IP host luar.
  * Jalankan layanan Nginx sebagai *Reverse Proxy* untuk memfilter *request* ke backend application.
  * Batasi hak akses penulisan direktori log Mosquitto broker.

* **Perintah Dasar & Evaluasi:**
  ```bash
  # Memeriksa port kontainer yang terbuka ke publik
  docker ps --format "table {{.Names}}\t{{.Ports}}\t{{.Status}}"

  # Memeriksa konsumsi resource kontainer
  docker stats --no-stream
  ```

---

### Domain 5: Proteksi Brute-Force, Logging, & Auditing (*Detection & Intrusion Prevention*)

* **Tujuan:** Memantau aktivitas mencurigakan secara *real-time* dan memblokir otomatis percobaan login berulang.
* **Rencana Tindakan:**
  * Aktifkan **Fail2ban** untuk memantau log autentikasi SSH (`/var/log/auth.log`) dan menerapkan *banning* IP otomatis jika gagal login > 3 kali.
  * Aktifkan **Auditd** untuk merekam perubahan pada file sistem sensitif (`/etc/passwd`, `/etc/shadow`, `/etc/sudoers`).
  * Verifikasi ketersediaan log untuk disadap oleh Security Onion via interface *Promiscuous*.

* **Perintah Dasar & Evaluasi:**
  ```bash
  # Memeriksa status proteksi jail Fail2ban
  sudo fail2ban-client status sshd

  # Memeriksa status daemon auditd
  sudo systemctl status auditd

  # Melihat daftar IP yang terblokir sementara
  sudo fail2ban-client status
  ```

---

### Domain 6: Manajemen Pembaruan Keamanan (*Patch & Vulnerability Management*)

* **Tujuan:** Menutup celah keamanan yang sudah dipublikasikan (*Known Vulnerabilities/CVE*).
* **Rencana Tindakan:**
  * Lakukan pembaruan rutin indeks repositori dan paket sistem.
  * Aktifkan paket `unattended-upgrades` untuk pembaruan *security patch* otomatis.

* **Perintah Dasar & Evaluasi:**
  ```bash
  # Memeriksa daftar paket yang membutuhkan pembaruan keamanan
  sudo apt list --upgradable

  # Memeriksa status pembaruan otomatis
  systemctl status unattended-upgrades
  ```

---

## 4. Matriks Pemeliharaan & Checklist Verifikasi Harian (Verifikasi Blue Team)

Gunakan checklist ini untuk memverifikasi kondisi sistem target secara berkala:

| No | Parameter Pemeriksaan | Perintah Verifikasi | Standard Normal / Expected Result | Status |
| :-: | :--- | :--- | :--- | :-: |
| 1 | UFW Firewall Status | `sudo ufw status` | `Status: active` dengan aturan port terbatas | [ ] |
| 2 | SSH Root Login | `grep "PermitRootLogin" /etc/ssh/sshd_config` | `PermitRootLogin no` | [ ] |
| 3 | Docker Isolation | `docker ps` | Port 5432 tidak di-map ke `0.0.0.0` | [ ] |
| 4 | Fail2ban Active Jail | `sudo fail2ban-client ping` | Response `Server-ping-pong` (Active) | [ ] |
| 5 | Network Listening Ports | `sudo ss -tulpn` | Hanya Port 22, 80, 1883 yang aktif open | [ ] |

---

## 5. Rencana Tindak Lanjut & Integrasi Monitoring

1. **Sinkronisasi Log ke SIEM:** Pastikan seluruh log dari `/var/log/auth.log`, `/var/log/syslog`, dan log kontainer Docker tercatat dengan timestamp yang sinkron (menggunakan UTC atau WIB secara konsisten).
2. **Evaluasi Reguler:** Setiap perubahan aplikasi atau penambahan port layanan baru harus melalui persetujuan Lead dan memperbarui file `ip_plan.md` serta `hardening_plan.md`.