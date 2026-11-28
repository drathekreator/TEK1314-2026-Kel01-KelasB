# Panduan Implementasi Infrastructure, Telemetry & Hardening Blue Team

Dokumen ini berisi panduan komprehensif *End-to-End* untuk tim pertahanan (**Blue Team**) dalam merancang, membangun, mengonfigurasi, dan memperkuat (*hardening*) infrastruktur laboratorium keamanan siber berbasis IoT.

---

## 1. Bedah Topologi & Arsitektur Lab

Sistem simulasi terdiri dari 4 VM utama yang terhubung melalui dua segmen jaringan terpisah: **Jaringan Internal (`sim-cyber-net`)** untuk trafik produksi/simulasi serangan, dan **Jaringan Management (`Host-Only`)** untuk akses admin dan monitoring SIEM.

```
+-----------------------------------------------------------------------------------+
|                                 WINDOWS HOST MACHINE                              |
|                                (VirtualBox Hypervisor)                            |
+-----------------------------------------------------------------------------------+
       |                                                                     |
       | [Host-Only Subnet: 192.168.56.0/24]                                  | [Internal Subnet: 192.168.1.0/24]
       | (Management & Web Console Access)                                   | (IoT Telemetry & Attack Traffic)
       v                                                                     v
+--------------------------+                                 +--------------------------+
|      TARGET SERVER       | <------------------------------ |         IOT-NODE         |
|   (Ubuntu Server 22.04)  |     MQTT Telemetry (1883/TCP)   |      (Alpine Linux)      |
|  IP: 192.168.1.5 / .56.10|                                 |     IP: 192.168.1.201     |
+--------------------------+                                 +--------------------------+
       ^                                                                     ^
       |                                                                     |
       | Attack Vectors (HTTP, SSH, Scans)                                   | Passive Sniffing (SPAN)
       |                                                                     |
+--------------------------+                                 +--------------------------+
|      ATTACKER NODE       |                                 |      SECURITY ONION      |
|   (Cybersecurity LabVM)  |                                 |    (SIEM / NIDS Engine)  |
|     IP: 192.168.1.100     |                                 | Sniffing: eth0 (Promisc), tanpa IP |
+--------------------------+                                 | No management IP; sniffer/SPAN only   |
                                                             +--------------------------+
```

### Peran Setiap Node:
1. **Target Server (`192.168.1.5`):** Main Target / Production Asset. Menjalankan Docker microservices (Nginx, Node.js API, PostgreSQL, Mosquitto MQTT Broker).
2. **IoT Node (`192.168.1.201`):** Sensor Node (Alpine Linux). Mengirimkan telemetri lingkungan ($^\circ\text{C}$, $\%$, $\text{THI}$, $\text{NH}_3$) secara berkala.
3. **Security Onion (tanpa IP (sniffer/SPAN)):** SIEM & NIDS Node. Meniru port *SPAN/Mirroring* untuk mengawasi seluruh lalu lintas `sim-cyber-net` via Suricata dan Zeek.
4. **Attacker Node (`192.168.1.100`):** Red Team Node untuk pengujian eksploitasi dan mitigasi.

---

## 2. Pemetaan Jaringan (IP Address Plan)

| Hostname | Interface 1 (`sim-cyber-net`) | Interface 2 (`Host-Only`) | Interface 3 (`NAT`) | OS / Distro | Fungsi Utama |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`target-server`** | `192.168.1.5/24` | `192.168.56.10/24` | DHCP (`10.0.4.x`) | Ubuntu Server 22.04 | Target Web, MQTT, & DB Server |
| **`iot-node`** | `192.168.1.201/24` | N/A | DHCP (`10.0.3.x`) | Alpine Linux v3.24 | Sensor Telemetry Publisher |
| **`sec-onion`** | Promiscuous (`eth0`) | N/A (sniffer/SPAN only) | N/A | Security Onion 2 | NIDS (Suricata), Zeek, Elastic/Kibana |
| **`cyber-workstation`**| `192.168.1.100/24` | N/A | DHCP | LabVM / Kali | Red Team Testing Node |
| **`gateway-sim`** | `192.168.1.1` | N/A | N/A | Virtual Switch | Gateway Segmen Internal (L3) |

---

## 3. Setup & Konfigurasi Target Server (`Ubuntu Server 22.04`)

### 3.1. Konfigurasi Jaringan (Netplan Dual-Interface)

Buka konfigurasi Netplan di Target Server:

```bash
sudo nano /etc/netplan/70-netplan-set.yaml
```

Isikan konfigurasi berikut:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3: # Internal Network (sim-cyber-net)
      addresses:
        - "192.168.1.5/24"
    enp0s8: # Host-Only Network (Management Access)
      addresses:
        - "192.168.56.10/24"
    enp0s9: # NAT Network (Internet Access)
      dhcp4: true
      dhcp4-overrides:
        route-metric: 10
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

Terapkan jaringan:

```bash
sudo netplan apply
```

### 3.2. Instalasi Resource & Container Engine (Docker)

```bash
# Update sistem dan install prasyarat
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git unzip htop net-tools ufw fail2ban auditd ca-certificates gnupg

# Pasang Repository Resmi Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg --yes
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Aktifkan Docker Service & User Group
sudo systemctl enable --now docker
sudo usermod -aG docker admin1
```

### 3.3. Deployment Microservices Stack (Docker Compose)

Buat struktur direktori aplikasi:

```bash
mkdir -p /opt/iot-target-stack/{mosquitto/config,mosquitto/data,mosquitto/log,nginx,app}
cd /opt/iot-target-stack
```

1. **Konfigurasi Mosquitto (`mosquitto/config/mosquitto.conf`):**
   ```text
   listener 1883 0.0.0.0
   allow_anonymous true
   persistence true
   persistence_location /mosquitto/data/
   log_dest file /mosquitto/log/mosquitto.log
   ```

2. **Konfigurasi Nginx (`nginx/default.conf`):**
   ```nginx
   server {
       listen 80;
       server_name _;

       location / {
           proxy_pass http://iot-api:3000;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       }

       location /health {
           return 200 '{"status":"healthy", "node":"Target-Server-01"}';
           add_header Content-Type application/json;
       }
   }
   ```

3. **Infrastruktur API & Database (`app/index.js` & `app/package.json`):**

   *`app/package.json`:*
   ```json
   {
     "name": "iot-ingestion",
     "version": "1.0.0",
     "main": "index.js",
     "dependencies": {
       "express": "^4.18.2",
       "mqtt": "^5.3.0",
       "pg": "^8.11.3"
     }
   }
   ```

   *`app/index.js`:*
   ```javascript
   const express = require('express');
   const mqtt = require('mqtt');
   const { Pool } = require('pg');

   const app = express();
   app.use(express.json());

   const pool = new Pool({
     connectionString: process.env.DATABASE_URL || 'postgres://iot_user:iot_password@postgres:5432/iot_db'
   });

   pool.query(`
     CREATE TABLE IF NOT EXISTS telemetry (
       id SERIAL PRIMARY KEY,
       device_id VARCHAR(50),
       temperature NUMERIC,
       humidity NUMERIC,
       thi_index NUMERIC,
       nh3_ppm NUMERIC,
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
     );
   `).catch(err => console.error('[DB ERROR]', err));

   const mqttClient = mqtt.connect('mqtt://mosquitto:1883');

   mqttClient.on('connect', () => {
     console.log('[MQTT] Terhubung ke Broker Mosquitto');
     mqttClient.subscribe('iot/sensor/telemetry');
   });

   mqttClient.on('message', async (topic, message) => {
     try {
       const data = JSON.parse(message.toString());
       console.log('[MQTT DATA RECEIVED]', data);
       await pool.query(
         'INSERT INTO telemetry (device_id, temperature, humidity, thi_index, nh3_ppm) VALUES ($1, $2, $3, $4, $5)',
         [data.device_id, data.temperature_c, data.humidity_pct, data.thi_index, data.nh3_ppm]
       );
     } catch (err) {
       console.error('[INGESTION ERROR]', err.message);
     }
   });

   app.get('/api/telemetry', async (req, res) => {
     try {
       const { rows } = await pool.query('SELECT * FROM telemetry ORDER BY id DESC LIMIT 20');
       res.json({ status: 'success', data: rows });
     } catch (err) {
       res.status(500).json({ error: err.message });
     }
   });

   app.listen(3000, () => console.log('[API] Telemetry Ingestion Service aktif di port 3000'));
   ```

   *`app/Dockerfile`:*
   ```dockerfile
   FROM node:18-alpine
   WORKDIR /usr/src/app
   COPY package*.json ./
   RUN npm install
   COPY . .
   EXPOSE 3000
   CMD ["node", "index.js"]
   ```

4. **File Orchestration (`docker-compose.yml`):**
   ```yaml
   version: '3.8'

   services:
     mosquitto:
       image: eclipse-mosquitto:2.0
       container_name: iot-mosquitto
       restart: always
       ports:
         - "1883:1883"
       volumes:
         - ./mosquitto/config/mosquitto.conf:/mosquitto/config/mosquitto.conf
         - ./mosquitto/data:/mosquitto/data
         - ./mosquitto/log:/mosquitto/log

     postgres:
       image: postgres:15-alpine
       container_name: iot-postgres
       restart: always
       environment:
         POSTGRES_USER: iot_user
         POSTGRES_PASSWORD: iot_password
         POSTGRES_DB: iot_db
       volumes:
         - pgdata:/var/lib/postgresql/data

     iot-api:
       build: ./app
       container_name: iot-telemetry-api
       restart: always
       environment:
         DATABASE_URL: postgres://iot_user:iot_password@postgres:5432/iot_db
       depends_on:
         - postgres
         - mosquitto

     nginx:
       image: nginx:alpine
       container_name: iot-nginx
       restart: always
       ports:
         - "80:80"
       volumes:
         - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
       depends_on:
         - iot-api

   volumes:
     pgdata:
   ```

5. **Jalankan Microservice Stack:**
   ```bash
   docker compose up -d
   ```

---

## 4. Setup & Konfigurasi IoT Node (`Alpine Linux`)

### 4.1. Jaringan & Repositori Permanen

```bash
# Set repository mirror
echo "https://dl-cdn.alpinelinux.org/alpine/latest-stable/main" > /etc/apk/repositories
echo "https://dl-cdn.alpinelinux.org/alpine/latest-stable/community" >> /etc/apk/repositories
apk update
apk add nano python3 py3-pip py3-paho-mqtt openrc
```

Konfigurasi `/etc/network/interfaces`:
```text
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.1.201
    netmask 255.255.255.0

auto eth1
iface eth1 inet dhcp
```

Terapkan jaringan:
```bash
service networking restart
```

### 4.2. Telemetry Daemon Script (`/opt/iot-sensor/mqtt_publisher.py`)

```python
#!/usr/bin/env python3
import time
import json
import random
import sys
import paho.mqtt.client as mqtt

BROKER_IP = "192.168.1.5"
PORT = 1883
TOPIC = "iot/sensor/telemetry"
CLIENT_ID = "Alpine-IoT-Node-01"

def calculate_thi(temp_c, humidity_pct):
    return round(temp_c - (0.55 - 0.0055 * humidity_pct) * (temp_c - 14.5), 2)

def main():
    client = mqtt.Client(client_id=CLIENT_ID)
    try:
        client.connect(BROKER_IP, PORT, keepalive=60)
        client.loop_start()
    except Exception as e:
        print(f"[!] Gagal terhubung ke Broker: {e}")
        sys.exit(1)

    while True:
        temp_c = round(random.uniform(26.0, 35.0), 2)
        humidity_pct = round(random.uniform(50.0, 85.0), 2)
        thi_index = calculate_thi(temp_c, humidity_pct)
        nh3_ppm = round(random.uniform(5.0, 30.0), 2)

        payload = {
            "device_id": CLIENT_ID,
            "timestamp": int(time.time()),
            "temperature_c": temp_c,
            "humidity_pct": humidity_pct,
            "thi_index": thi_index,
            "nh3_ppm": nh3_ppm
        }
        client.publish(TOPIC, json.dumps(payload), qos=1)
        time.sleep(5)

if __name__ == "__main__":
    main()
```

### 4.3. OpenRC Service Background Execution

Buat file `/etc/init.d/iot-sensor`:
```bash
#!/sbin/openrc-run

name="IoT Sensor Service"
command="/usr/bin/python3"
command_args="/opt/iot-sensor/mqtt_publisher.py"
command_background="yes"
pidfile="/run/iot-sensor.pid"

depend() {
    need net
}
```

Aktifkan service:
```bash
chmod +x /etc/init.d/iot-sensor
rc-service iot-sensor start
rc-update add iot-sensor default
```

---

## 5. Konfigurasi Monitoring & SIEM Node (`Security Onion`)

### 5.1. Dual Interface Setup
* **Interface 1 (`eth0`):** Promiscuous / Sniffing mode terhubung ke VirtualBox Internal Network `sim-cyber-net`.
* **Interface 2 (`eth1`):** Management IP N/A (sniffer/SPAN only) terhubung ke Host-Only.

### 5.2. Sensor Engine Alignment
1. **Suricata (NIDS):** Memantau aturan `ET Open` untuk memicu alert saat pemindaian port (Nmap), eksekusi command abnormal, atau payload terlarang terdeteksi.
2. **Zeek (Network Security Monitoring):** Mencatat log lalu lintas HTTP (`http.log`), koneksi TCP (`conn.log`), dan percakapan MQTT secara terstruktur.
3. **Kibana Dashboard:** Diakses melalui host via Security Onion console untuk visualisasi log dan korelasi insiden siber.

---

## 6. Implementation Hardening Plan (Sistem & Jaringan)

### 6.1. SSH Hardening (`/etc/ssh/sshd_config`)

Buka dan sesuaikan konfigurasi SSH pada `target-server`:

```text
Port 22
PermitRootLogin no
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
AllowTcpForwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
```

Terapkan:
```bash
sudo systemctl restart ssh
```

### 6.2. UFW Firewall Rules (Strict Filtering)

Terapkan prinsip *Default Deny*:

```bash
# Reset & Set Default Policy
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Izinkan SSH HANYA dari Subnet Host-Only (Management)
sudo ufw allow from 192.168.56.0/24 to any port 22 proto tcp comment 'SSH Management'

# Izinkan Web (80) & MQTT (1883) HANYA dari Subnet Internal Lab
sudo ufw allow from 192.168.1.0/24 to any port 80 proto tcp comment 'Web App Internal'
sudo ufw allow from 192.168.1.0/24 to any port 1883 proto tcp comment 'MQTT Ingestion'

# Aktifkan UFW
sudo ufw enable
```

### 6.3. Fail2ban Protection (`/etc/fail2ban/jail.local`)

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 3

[sshd]
enabled = true
port    = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
```

Aktifkan Fail2ban:
```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

### 6.4. Linux Kernel Sysctl Hardening (`/etc/sysctl.d/99-security.conf`)

Buat file hardening kernel:

```ini
# Proteksi SYN Flood
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2

# Proteksi IP Spoofing & Source Routing
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# Mengabaikan ICMP Redirects & Broadcast Ping
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Nonaktifkan IP Forwarding
net.ipv4.ip_forward = 0
```

Terapkan tanpa reboot:
```bash
sudo sysctl -p /etc/sysctl.d/99-security.conf
```

---

## 7. Audit & Checklist Verifikasi Blue Team

Gunakan tabel checklist ini untuk memverifikasi kesiapan operasional pertahanan:

| No | Parameter Auditing | Perintah Evaluasi | Baseline yang Diharapkan | Status |
| :--- | :--- | :--- | :--- | :--- |
| 1 | Status Firewall | `sudo ufw status verbose` | `Status: active`, port terbuka terbatas pada 22, 80, 1883 | [ ] |
| 2 | Hak Akses SSH Root | `grep "PermitRootLogin" /etc/ssh/sshd_config` | `PermitRootLogin no` | [ ] |
| 3 | Isolasi Database | `docker ps` | PostgreSQL (Port 5432) **tidak** ter-bind ke `0.0.0.0` | [ ] |
| 4 | Status Fail2ban | `sudo fail2ban-client ping` | Response: `Server-ping-pong` | [ ] |
| 5 | Telemetri Ingestion | `curl http://192.168.1.5/api/telemetry` | Mengembalikan JSON telemetri dari `Alpine-IoT-Node-01` | [ ] |
| 6 | Sniffing SIEM | Security Onion console | Log `conn.log` dan `http.log` tercatat di Kibana | [ ] |