# Blue Team Infrastructure Hardening & Deployment Guide
**Target System:** `target-server` (Ubuntu Server 22.04 LTS)  
**Target Domain:** `https://dreamsheep.local`  
**Internal Network:** `192.168.1.0/24` | **Management Network:** `192.168.56.0/24`

---

## Part 1: Blue Team System Hardening Plan

### 1.1 Executive Summary & Hardening Objectives
This hardening plan defines the operational security baseline for the **`target-server`** (`192.168.1.5` / `192.168.56.10`). The primary objectives are:
1. **Attack Surface Reduction:** Expose only mandatory ports (`22` restricted to management, `80/443` for web HTTPS, `1883` for internal MQTT ingestion).
2. **Database Isolation:** Restrict PostgreSQL (`5432`) to the internal Docker bridge network, ensuring no exposure to external interfaces.
3. **Network Stack Fortification:** Apply kernel level protections against IP spoofing, ICMP redirect hijacking, and TCP SYN flood attacks.
4. **Access Control Enforcement:** Restrict root SSH access and implement automated dynamic banning via Fail2ban.

---

### 1.2 Security Policy & Controls Matrix

| Control Category | Targeted Service / Asset | Implemented Security Mechanism | Baseline Requirement |
| :--- | :--- | :--- | :--- |
| **Network Perimeter** | UFW Firewall | Default-Deny Incoming Policy | Port 22 limited to `192.168.56.0/24`; Ports 80, 443, 1883 limited to `192.168.1.0/24`. |
| **SSH Security** | OpenSSH Daemon | Strict Access Control Configuration | `PermitRootLogin no`, `MaxAuthTries 3`, `AllowTcpForwarding no`. |
| **Intrusion Prevention** | Fail2ban | SSH Authentication Monitoring | Ban IP for 1 hour after 3 failed login attempts within 10 minutes. |
| **Database Isolation** | PostgreSQL Container | Docker Bridge Isolation | No host port mapping (`5432/tcp` inaccessible outside Docker bridge). |
| **Kernel Hardening** | Linux Kernel (`sysctl`) | Packet Filter & TCP Stack Parameters | Enable TCP SYN Cookies, RPF filtering, disable IP forwarding & ICMP redirects. |
| **Transport Layer** | Nginx Reverse Proxy | TLS 1.2/1.3 Cryptographic Policy | Force HTTP to HTTPS redirection; TLSv1.2/TLSv1.3 only. |

---

## Part 2: Full End-to-End Deployment & Hardening Runbook

Perform the following commands on **`target-server`** starting from a clean installation of Ubuntu Server 22.04 LTS.

---

### Step 1: Storage Cleanup & System Packages Setup

First, clean up residual build caches to avoid `no space left on device` issues and install prerequisite tools:

```bash
# 1. Clean Docker cache and unnecessary files
docker system prune -af --volumes 2>/dev/null || true
sudo apt clean

# 2. Update repository indices & install basic dependencies
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git unzip htop net-tools ufw fail2ban auditd ca-certificates gnupg mosquitto-clients openssl

# 3. Add official Docker GPG key and repository
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg --yes
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 4. Install Docker Engine and Compose plugin
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 5. Enable Docker service & assign user permissions
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

---

### Step 2: System Kernel & OS Level Hardening

Apply OS hardening rules directly to the host kernel, SSH daemon, and firewall.

#### 2.1 Kernel Parameter Hardening (`sysctl`)
```bash
sudo tee /etc/sysctl.d/99-security.conf << 'EOF'
# TCP SYN Flood Protection
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2

# Reverse Path Filtering & IP Spoofing Mitigation
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# Ignore ICMP Redirects & Broadcast Pings
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Disable IP Forwarding (Host is not a router)
net.ipv4.ip_forward = 0
EOF

# Apply sysctl settings immediately
sudo sysctl -p /etc/sysctl.d/99-security.conf
```

#### 2.2 SSH Hardening Configuration
```bash
sudo tee /etc/ssh/sshd_config.d/50-hardening.conf << 'EOF'
PermitRootLogin no
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
AllowTcpForwarding no
EOF

sudo systemctl restart ssh
```

#### 2.3 Fail2ban Intrusion Prevention Configuration
```bash
sudo tee /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 3

[sshd]
enabled = true
port    = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
EOF

sudo systemctl restart fail2ban
```

#### 2.4 UFW Perimeter Firewall Configuration
```bash
# Reset UFW to default state
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Rule 1: SSH Access ONLY from Management Network
sudo ufw allow from 192.168.56.0/24 to any port 22 proto tcp comment 'SSH Management Only'

# Rule 2: Web HTTP/HTTPS ONLY from Internal Lab Subnet
sudo ufw allow from 192.168.1.0/24 to any port 80 proto tcp comment 'Web HTTP Internal'
sudo ufw allow from 192.168.1.0/24 to any port 443 proto tcp comment 'Web HTTPS Internal'

# Rule 3: MQTT Telemetry Ingestion ONLY from Internal Lab Subnet
sudo ufw allow from 192.168.1.0/24 to any port 1883 proto tcp comment 'MQTT Ingestion Internal'

# Enable Firewall
sudo ufw --force enable
```

---

### Step 3: Application Stack Deployment (`dreamsheep-simulasi`)

#### 3.1 Workspace Setup & Repository Cloning
```bash
# Prepare stack directory
sudo mkdir -p /opt/iot-target-stack
sudo chown -R $USER:$USER /opt/iot-target-stack
cd /opt/iot-target-stack

# Clone repository simulation stack
git clone https://github.com/drathekreator/dreamsheep-simulasi.git .
```

#### 3.2 SSL Certificate Generation (`dreamsheep.local`)
```bash
mkdir -p /opt/iot-target-stack/nginx/ssl

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /opt/iot-target-stack/nginx/ssl/dreamsheep.key \
  -out /opt/iot-target-stack/nginx/ssl/dreamsheep.crt \
  -subj "/CN=dreamsheep.local"
```

#### 3.3 Mosquitto Configuration Check (`mosquitto/config/mosquitto.conf`)
```bash
mkdir -p mosquitto/config mosquitto/data mosquitto/log

cat << 'EOF' > mosquitto/config/mosquitto.conf
listener 1883 0.0.0.0
allow_anonymous true
persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log
EOF
```

#### 3.4 Nginx Reverse Proxy Configuration Check (`nginx/default.conf`)
```bash
cat << 'EOF' > nginx/default.conf
server {
    listen 80;
    server_name dreamsheep.local;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name dreamsheep.local;

    ssl_certificate /etc/nginx/ssl/dreamsheep.crt;
    ssl_certificate_key /etc/nginx/ssl/dreamsheep.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://iot-api:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }

    location /health {
        return 200 '{"status":"healthy","node":"target-server-01","protocol":"https"}';
        add_header Content-Type application/json;
    }
}
EOF
```

#### 3.5 Local Host Resolution Mapping
```bash
grep -q "dreamsheep.local" /etc/hosts || echo "127.0.0.1 dreamsheep.local" | sudo tee -a /etc/hosts
```

#### 3.6 Build & Launch Microservice Stack
```bash
cd /opt/iot-target-stack
docker compose up -d --build
```

---

## Part 3: Operational Verification & Auditing Checklist

Run these verification steps to ensure all services are functioning properly and hardened.

### 1. Verify Container Execution
```bash
docker compose ps
```
**Expected Outcome:** All 4 containers (`iot-mosquitto`, `iot-postgres`, `iot-telemetry-api`, `iot-nginx`) report `Up` state.

### 2. Verify Database Isolation (Security Check)
```bash
sudo ss -tulpn | grep 5432
```
**Expected Outcome:** Empty output (Port `5432` must NOT be bound to any host IP).

### 3. Verify Health Endpoint
```bash
curl -k -i https://dreamsheep.local/health
```
**Expected Outcome:** `HTTP/1.1 200 OK` with payload `{"status":"healthy","node":"target-server-01","protocol":"https"}`.

### 4. Verify Active Ingestion Pipeline
```bash
docker exec -it iot-postgres psql -U iot_user -d iot_db -c "SELECT * FROM telemetry ORDER BY id DESC LIMIT 5;"
```
**Expected Outcome:** Database returns live telemetry records published by `Alpine-IoT-Node-01`.

---

## Part 4: Client & Node Setup Instructions

### A. Red Team VM (`secOps` - `192.168.1.100`)
1. Add host resolution entry:
   ```bash
   sudo grep -q "dreamsheep.local" /etc/hosts || echo "192.168.1.5 dreamsheep.local" | sudo tee -a /etc/hosts
   ```
2. Open Firefox and navigate to: `https://dreamsheep.local`
3. Click **Advanced...** -> **Accept the Risk and Continue**.
4. Test demo authentication:
   * **Admin:** `admin` / `admin123`
   * **Peternak:** `peternak` / `ternak123`

### B. IoT Sensor Node (`Alpine Linux` - `192.168.1.201`)
1. Verify telemetry publisher daemon status:
   ```bash
   rc-service iot-sensor status
   ```
2. If inactive, start service:
   ```bash
   rc-service iot-sensor start
   ```