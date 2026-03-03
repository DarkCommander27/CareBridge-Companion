# Jetson AGX Orin 64GB Deployment Guide

**Target Deployment:** CareBridge Companion with Llama 3.1 70B  
**Hardware:** NVIDIA Jetson AGX Orin (64GB LPDDR5)  
**Performance:** 275 TOPS (INT8), 50W power draw  
**Cost per Facility:** ~$1,500 (hardware) + $200/year (operating)

---

## 1. Hardware & Prerequisites

### 1.1 What You Need

**Required:**
- NVIDIA Jetson AGX Orin 64GB Module ($1,500)
- Jetson Orin DevKit Carrier Board (~$500) OR Industrial carrier board ($800-1,500)
- 65W USB-C Power Supply (included with DevKit)
- microSD Card or NVME SSD (500GB minimum recommended)
- Ethernet cable (RJ45)
- HDMI monitor, USB keyboard/mouse (for initial setup only)

**Network:**
- Commercial-grade internet (100+ Mbps recommended)
- Gigabit LAN for local connections
- Static IP assignment recommended

**Environmental:**
- Operating temp: 0–40°C (facility climate controlled)
- Humidity: Non-condensing, <95% RH
- Ventilation: Passive cooling sufficient; avoid enclosed cabinets
- UPS backup recommended (optional but valuable)

### 1.2 Estimated Hardware Costs (2 Facilities)

| Component | Unit Cost | Qty | Subtotal |
|-----------|-----------|-----|----------|
| Jetson AGX Orin 64GB Module | $1,500 | 2 | $3,000 |
| DevKit Carrier Board | $500 | 2 | $1,000 |
| 500GB NVME SSD | $50 | 2 | $100 |
| Network/Power cables | $50 | 2 | $100 |
| **Total Hardware** | | | **$4,200** |
| Annual Operating (power) | $200 | 2 | **$400/yr** |

---

## 2. Operating System & Runtime Setup

### 2.1 Install JetPack

**Requirements:**
- Linux development machine (Ubuntu 20.04 or 22.04)
- NVIDIA SDK Manager or manual image flashing

**Automated Installation (Recommended):**

```bash
# On your development machine:
# 1. Download NVIDIA SDK Manager from https://developer.nvidia.com/sdk-manager
# 2. Install SDK Manager dependencies
sudo apt-get install libfuse2 libstdc++6 libssl1.1

# 3. Run SDK Manager
# Select:
# - Target Hardware: Jetson AGX Orin
# - JetPack: 6.0 (latest stable for Orin)
# - Target OS: Linux (Ubuntu 22.04 based)

# 4. Follow on-screen prompts to:
#    - Connect Jetson in recovery mode
#    - Flash OS to storage
#    - Install runtime libraries
```

**Manual Installation (Alternative):**

```bash
# Download JetPack image
cd ~/Downloads
wget https://developer.nvidia.com/downloads/jetpack6.0-orin-aarch64.tar.gz

# Extract and flash
tar -xzf jetpack6.0-orin-aarch64.tar.gz
cd Linux_for_Tegra
# Put Jetson in recovery mode: press and hold FORCE_RECOVERY, press RESET
sudo ./flash.sh jetson-orin-agx nvme0n1

# System will reboot and complete setup (5-10 minutes)
```

### 2.2 First Boot & Initial Setup

**On the Jetson, after JetPack installation:**

```bash
# System will prompt for:
# - Username and password (create facility admin account)
# - Network configuration (DHCP or static IP)
# - Accept NVIDIA license terms

# After login, verify installation:
nvidia-smi

# Expected output:
# NVIDIA Jetson AGX Orin (Grace ARM cores + NVIDIA GPU)
# GPU Memory: 64GB LPDDR5
```

### 2.3 Update System & Install Base Dependencies

```bash
# Update package manager
sudo apt-get update
sudo apt-get upgrade -y

# Install core dependencies
sudo apt-get install -y \
    python3-pip \
    python3-venv \
    git \
    curl \
    wget \
    build-essential \
    libssl-dev \
    libffi-dev \
    cmake

# Install Node.js (for Express backend)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify installations
python3 --version  # Python 3.10+
node --version     # v20+
npm --version      # 10+
```

### 2.4 Enable GPU Memory Scaling

Jetson supports dynamic power scaling. Configure for optimal power/performance:

```bash
# Check current power mode
sudo /usr/sbin/nvpmodel -q

# Set to NVPMODEL_MODE_25W_CLIPPED (recommended for 24/7 operations)
# Runs at ~25W sustained (room for spikes to 50W)
sudo /usr/sbin/nvpmodel -m 1

# Verify
sudo /usr/sbin/nvpmodel -q

# To make persistent, add to /etc/rc.local:
echo "sudo /usr/sbin/nvpmodel -m 1" | sudo tee -a /etc/rc.local
sudo chmod +x /etc/rc.local
```

### 2.5 Configure Docker (Optional but Recommended)

Docker simplifies deployment and isolation:

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group (avoid sudo)
sudo usermod -aG docker $(whoami)
newgrp docker

# Verify
docker --version

# Note: NVIDIA Container Runtime integration is automatic on Jetson
```

---

## 3. Local Development Environment

### 3.1 Clone CareBridge Companion Repository

```bash
# Create application directory
mkdir -p /opt/carebridge
cd /opt/carebridge

# Clone repository
git clone https://github.com/DarkCommander27/CareBridge-Companion.git .

# Navigate to server
cd server
```

### 3.2 Install Node Dependencies

```bash
npm install

# Verify all dependencies installed
npm list --depth=0

# Expected:
# express@4.x
# mongoose@7.x
# jsonwebtoken@9.x
# bcryptjs@2.x
# (plus others)
```

### 3.3 Set Up Environment Variables

Create `.env` file in `/opt/carebridge/server/`:

```bash
# Database
MONGODB_URI=mongodb://localhost:27017/carebridge
DB_NAME=carebridge

# Authentication
JWT_SECRET=<generate-secure-random-string>
NODE_ENV=production

# Server
PORT=3000
HOST=0.0.0.0

# Facility Configuration
FACILITY_ID=facility-01
FACILITY_NAME=Primary Youth Facility
FACILITY_LOCATION=West Virginia

# LLM Configuration (see Llama section)
LLM_API_URL=http://localhost:8000/v1

# Security
CORS_ORIGIN=http://localhost:3000
SESSION_TIMEOUT=3600
ENCRYPTION_KEY=<generate-secure-random-string>

# Optional: VPN Configuration (see Multi-Facility section)
# VPN_SERVER=vpn.carebridge.local
# VPN_ENABLED=false
```

**Generate secure random strings:**

```bash
# For JWT_SECRET and ENCRYPTION_KEY:
openssl rand -base64 32
```

---

## 4. MongoDB Setup

### 4.1 Install MongoDB

```bash
# Add MongoDB repository (ARM64 version)
wget -qO - https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse" | \
    sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list

# Install
sudo apt-get update
sudo apt-get install -y mongodb-org mongodb-mongosh

# Start service
sudo systemctl start mongod
sudo systemctl enable mongod

# Verify
mongosh --version
mongosh --eval "db.adminCommand('ping')"
```

### 4.2 Initialize Database

```bash
# Create application database and indexes
mongosh << EOF
use carebridge

// Create collections with validation
db.createCollection("youth", {
  validator: {
    \$jsonSchema: {
      bsonType: "object",
      required: ["name", "age", "facility_id"],
      properties: {
        name: { bsonType: "string" },
        age: { bsonType: "int" },
        facility_id: { bsonType: "string" }
      }
    }
  }
})

db.createCollection("briefings")
db.createCollection("conversations")
db.createCollection("audit_logs")
db.createCollection("staff_access")

// Create indexes
db.youth.createIndex({ facility_id: 1 })
db.briefings.createIndex({ youth_id: 1 })
db.conversations.createIndex({ youth_id: 1, timestamp: -1 })
db.audit_logs.createIndex({ timestamp: -1, staff_id: 1 })
db.staff_access.createIndex({ staff_id: 1, facility_id: 1 })

// Verify
db.getCollectionNames()
EOF
```

### 4.3 Enable Authentication (Production)

```bash
# Create admin user
mongosh << EOF
use admin
db.createUser({
  user: "carebridge_admin",
  pwd: "<secure-password>",
  roles: ["dbOwner"]
})
EOF

# Update /etc/mongod.conf
# Find `security:` section and add:
# security:
#   authorization: enabled

sudo systemctl restart mongod

# Verify with authentication
mongosh -u carebridge_admin -p <secure-password> --authenticationDatabase admin
```

---

## 5. Start Services

### 5.1 Run Express Backend

```bash
cd /opt/carebridge/server

# Start backend (development mode)
npm start

# Expected output:
# Server running on port 3000
# Connected to MongoDB at mongodb://localhost:27017
```

### 5.2 Serve Frontend

```bash
cd /opt/carebridge

# Serve static files (in another terminal)
python3 -m http.server 8080 --directory .

# OR use Node serving:
npx http-server -p 8080
```

### 5.3 Verify Services

```bash
# In another terminal:

# Test backend
curl http://localhost:3000/api/health

# Test frontend
curl http://localhost:8080
```

---

## 6. System Monitoring & Health

### 6.1 Monitor GPU/CPU Usage

```bash
# Install monitoring tools
sudo apt-get install -y jtop

# Real-time monitoring
jtop

# Or use system commands
nvidia-smi -l 1  # Update every 1 second
top -b -n 1 | head -20  # CPU usage
```

### 6.2 Check Temperatures

```bash
cat /sys/devices/virtual/thermal/cooling_device0/cur_state
# Safe ranges: < 50°C idle, < 75°C under load
```

### 6.3 Memory Usage

```bash
# Check system RAM
free -h

# Check GPU memory
nvidia-smi | grep "Memory-Usage"
```

---

## 7. Systemd Service Setup (For Auto-Start)

### 7.1 Create Backend Service

Create `/etc/systemd/system/carebridge-backend.service`:

```ini
[Unit]
Description=CareBridge Companion Backend
After=network.target mongod.service
Wants=mongod.service

[Service]
Type=simple
User=carebridge
WorkingDirectory=/opt/carebridge/server
ExecStart=/usr/bin/node /opt/carebridge/server/server.js
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
Environment="NODE_ENV=production"

[Install]
WantedBy=multi-user.target
```

### 7.2 Enable Auto-Start

```bash
# Create carebridge user
sudo useradd -m -s /bin/bash carebridge

# Set permissions
sudo chown -R carebridge:carebridge /opt/carebridge

# Enable service
sudo systemctl daemon-reload
sudo systemctl enable carebridge-backend
sudo systemctl start carebridge-backend

# Check status
sudo systemctl status carebridge-backend
```

---

## 8. Network Configuration

### 8.1 Set Static IP (Recommended)

```bash
# Find network interface
ip link show

# Edit netplan configuration
sudo nano /etc/netplan/50-cloud-init.yaml

# Set static IP:
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.1.100/24  # Adjust to facility network
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]

# Apply
sudo netplan apply
```

### 8.2 Firewall Configuration

```bash
# Install UFW
sudo apt-get install -y ufw

# Enable firewall
sudo ufw enable

# Allow SSH (for remote management)
sudo ufw allow 22/tcp

# Allow backend API
sudo ufw allow 3000/tcp

# Allow Llama inference API
sudo ufw allow 8000/tcp

# Check rules
sudo ufw status
```

---

## 9. Backup & Recovery

### 9.1 Backup Strategy

```bash
# Create backup script
cat > /opt/carebridge/backup.sh << 'EOF'
#!/bin/bash

BACKUP_DIR="/mnt/backups"
DATE=$(date +%Y%m%d_%H%M%S)

# Backup MongoDB
mongodump --out $BACKUP_DIR/mongodb_$DATE

# Backup application config
tar -czf $BACKUP_DIR/app_config_$DATE.tar.gz /opt/carebridge/.env

# Retention: Keep only last 7 days
find $BACKUP_DIR -type f -mtime +7 -delete

echo "Backup completed: $DATE"
EOF

chmod +x /opt/carebridge/backup.sh

# Schedule daily at 2 AM
crontab -e
# Add: 0 2 * * * /opt/carebridge/backup.sh
```

### 9.2 Disaster Recovery

```bash
# Restore from backup
mongorestore --dir /mnt/backups/mongodb_<DATE>

# Verify restored data
mongosh carebridge --eval "db.youth.count()"
```

---

## 10. Deployment Checklist

Before going live on facility:

- [ ] Hardware installed and powered on
- [ ] JetPack OS flashed and updated
- [ ] All system dependencies installed
- [ ] MongoDB initialized with collections
- [ ] Backend service started and healthy
- [ ] Frontend accessible on port 8080
- [ ] Static IP configured and stable
- [ ] Firewall rules applied
- [ ] MongoDB backups scheduled
- [ ] GPU monitoring tools installed (jtop)
- [ ] Thermal sensors reading normal (< 50°C)
- [ ] Network connectivity tested (ping external)
- [ ] Facility staff trained on device location/access
- [ ] Logs being written to `/var/log/carebridge/`

---

## 11. Troubleshooting

### Issue: GPU Not Detected

```bash
# Verify GPU presence
nvidia-smi

# If missing, check device tree
cat /proc/device-tree/model

# Reboot if kernels out of sync
sudo reboot
```

### Issue: MongoDB Connection Refused

```bash
# Check status
sudo systemctl status mongod

# Check logs
sudo journalctl -u mongod -n 50

# Restart service
sudo systemctl restart mongod
```

### Issue: High Temperature (> 75°C)

```bash
# Check thermal throttling
cat /sys/devices/virtual/thermal/cooling_device*/cur_state

# Ensure ventilation
# Reduce power mode:
sudo /usr/sbin/nvpmodel -m 1  # Reduced to ~25W

# Monitor improvement
watch -n 1 'cat /sys/class/thermal/thermal_zone*/temp'
```

### Issue: Out of Memory

```bash
# Check available RAM
free -h

# Check GPU memory usage
nvidia-smi --query-gpu=memory.used,memory.total --format=csv

# Monitor processes
top -b -n 1 | head -20

# Reduce Llama batch size (see Llama optimization section)
```

---

## 12. Reference Resources

- **NVIDIA Jetson Documentation:** https://docs.nvidia.com/jetson/
- **JetPack Release Notes:** https://developer.nvidia.com/jetpack-release-notes
- **Jetson Performance Tuning:** https://docs.nvidia.com/jetson/archives/l4t-multimedia-user-guide/
- **MongoDB on ARM64:** https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/
- **Node.js on ARM:** https://nodejs.org/en/download/package-manager/

---

**Version:** 1.0  
**Last Updated:** March 3, 2026  
**Maintenance Contact:** CareBridge Infrastructure Team
