# CareBridge Companion: Multi-Facility Deployment Checklist

**Scope:** Two Jetson AGX Orin facilities with VPN federation  
**Timeline:** 4-6 weeks for both facilities (pilot approach)  
**Owner:** CareBridge Infrastructure/Operations Team  

---

## Phase 1: Hardware Procurement & Setup (Week 1)

### Hardware Acquisition

- [ ] Order 2x Jetson AGX Orin 64GB modules ($1,500 each)
- [ ] Order 2x Carrier boards (DevKit or industrial: $500-1,500 each)
- [ ] Order 2x 500GB NVME SSDs ($50 each)
- [ ] Order 2x 65W USB-C power supplies
- [ ] Order networking cables (Ethernet, USB, HDMI)
- [ ] Order 2x uninterruptible power supplies (UPS) - optional but recommended
- [ ] Verify all items received and functional

**Cost Check:** ~$4,500 hardware + $500 cables/misc = $5,000 for both facilities

### Physical Installation - Facility A

- [ ] Choose server room location (cool, secure, ventilated)
- [ ] Mount Jetson in rack or shelf (passive cooling, not enclosed)
- [ ] Connect power (65W supply)
- [ ] Connect Ethernet (gigabit preferred) to facility switch
- [ ] Connect USB keyboard/mouse and HDMI monitor for initial setup
- [ ] Power on and verify lights/fans on

### Physical Installation - Facility B

- [ ] Repeat all steps from Facility A
- [ ] Document both physical locations (building, room, coordinates)
- [ ] Create backup power plan for each (UPS if available)

**Deliverable:** Both Jetson devices powered on and accessible

---

## Phase 2: Operating System & Dependencies (Week 1-2)

### OS Installation - Facility A

- [ ] Prepare Ubuntu/Linux development machine with NVIDIA SDK Manager
- [ ] Download JetPack 6.0 image (25GB+, ~30 min)
- [ ] Put Jetson in recovery mode (hold FORCE_RECOVERY)
- [ ] Flash OS to Jetson (10-15 minutes)
- [ ] Complete initial setup wizard:
  - [ ] Create admin user account
  - [ ] Configure network (DHCP or static IP)
  - [ ] Accept NVIDIA license
  - [ ] Update all packages
- [ ] Verify GPU detected: `nvidia-smi`
- [ ] Document Jetson IP address: `_______________`

### OS Installation - Facility B

- [ ] Repeat all steps from Facility A
- [ ] Document Jetson IP address: `_______________`

### Install Core Dependencies (Both Facilities)

- [ ] `sudo apt-get update && upgrade -y` (5 min)
- [ ] Install Python 3.10+: `python3 --version`
- [ ] Install Node.js 20+: `node --version`
- [ ] Install Git: `git --version`
- [ ] Install Docker (optional): `docker --version`
- [ ] Install building tools: `sudo apt-get install build-essential cmake`

### Initial System Configuration (Both Facilities)

- [ ] Enable power mode optimization:
  ```bash
  sudo /usr/sbin/nvpmodel -m 1  # Set to 25W mode
  ```
- [ ] Set static IP address (or document DHCP reservation):
  - [ ] Facility A IP: `___.___.___.___`
  - [ ] Facility B IP: `___.___.___.___ `
- [ ] Configure firewall (UFW):
  ```bash
  sudo ufw enable
  sudo ufw allow 22/tcp  # SSH
  sudo ufw allow 3000/tcp  # Express backend
  sudo ufw allow 8000/tcp  # Llama inference
  sudo ufw allow 51820/udp  # WireGuard (if using VPN)
  ```
- [ ] Configure SSH key-based access (for remote management)

**Deliverable:** Both Jetson systems running Ubuntu, network-accessible, power-optimized

---

## Phase 3: Database Setup (Week 2)

### MongoDB Installation (Both Facilities)

- [ ] Add MongoDB repository
- [ ] Install MongoDB Server: `mongod --version`
- [ ] Enable auto-start: `sudo systemctl enable mongod`
- [ ] Start service: `sudo systemctl start mongod`
- [ ] Verify connection: `mongosh --eval "db.adminCommand('ping')"`
- [ ] Create collections (see JETSON_AGX_ORIN_DEPLOYMENT_GUIDE.md Phase 4)
- [ ] Create indexes for performance
- [ ] Test backup/restore procedure

### Database Security (Both Facilities)

- [ ] Create MongoDB admin user with strong password
- [ ] Enable authentication in `/etc/mongod.conf`
- [ ] Restart MongoDB service
- [ ] Test authentication: `mongosh -u <user> -p <pass> --authenticationDatabase admin`
- [ ] Document MongoDB admin password (store in secure location)

**Deliverable:** Both facilities running MongoDB with authentication enabled, backups tested

---

## Phase 4: Application Deployment (Week 2-3)

### Clone & Configure - Facility A

- [ ] SSH into Facility A Jetson
- [ ] Clone CareBridge repository:
  ```bash
  mkdir -p /opt/carebridge
  cd /opt/carebridge
  git clone https://github.com/DarkCommander27/CareBridge-Companion.git .
  ```
- [ ] Create .env file in `/opt/carebridge/server/`:
  ```
  FACILITY_ID=facility-01
  FACILITY_NAME=Facility A
  MONGODB_URI=mongodb://localhost:27017/carebridge
  JWT_SECRET=<secure-random-string>
  PORT=3000
  NODE_ENV=production
  ```
- [ ] Install Node dependencies:
  ```bash
  cd /opt/carebridge/server
  npm install
  npm test  # Run test suite to verify all 121 tests pass
  ```

### Clone & Configure - Facility B

- [ ] SSH into Facility B Jetson
- [ ] Repeat all steps from Facility A with:
  ```
  FACILITY_ID=facility-02
  FACILITY_NAME=Facility B
  ```
- [ ] Verify both test suites pass (121/121 tests each)

### Create Systemd Services (Both Facilities)

- [ ] Create `/etc/systemd/system/carebridge-backend.service`:
  - [ ] Set correct paths
  - [ ] Set correct NODE_ENV
  - [ ] Enable: `sudo systemctl enable carebridge-backend`
  - [ ] Start: `sudo systemctl start carebridge-backend`
- [ ] Verify service running: `sudo systemctl status carebridge-backend`
- [ ] Check logs: `journalctl -u carebridge-backend -f`

### Frontend Deployment (Both Facilities)

- [ ] Serve frontend from `/opt/carebridge/`:
  ```bash
  cd /opt/carebridge
  python3 -m http.server 8080
  # Or create systemd service for persistent serving
  ```
- [ ] Test frontend access from staff workstation:
  ```bash
  curl http://<Facility A IP>:8080
  ```

**Deliverable:** Both facilities running CareBridge with all 121 tests passing, services auto-starting on reboot

---

## Phase 5: Llama 70B Model Deployment (Week 3-4)

### Install llama.cpp (Both Facilities)

- [ ] Clone llama.cpp:
  ```bash
  cd /opt && git clone https://github.com/ggerganov/llama.cpp.git
  ```
- [ ] Compile with CUDA support (take ~10 min):
  ```bash
  cd llama.cpp && mkdir build && cd build
  cmake .. -DCMAKE_BUILD_TYPE=Release -DLLAMA_CUDA=ON
  make -j4
  ```
- [ ] Verify GPU compilation: `./bin/llama-cli --help | grep cuda`

### Download Model - Facility A

- [ ] Install huggingface-cli:
  ```bash
  pip3 install huggingface-hub
  ```
- [ ] Download Llama 3.1 70B Q4_K_M (~38GB, 20-30 min):
  ```bash
  huggingface-cli download \
    meta-llama/Llama-3.1-70B-Instruct \
    --local-dir ./models/llama-3.1-70b-q4km
  ```
- [ ] Verify model file exists:
  ```bash
  ls -lh /opt/llama.cpp/models/llama-3.1-70b-instruct-q4_k_m.gguf
  # Should be ~38GB
  ```

### Download Model - Facility B

- [ ] Repeat model download (or use faster mirror/direct transfer if available)

### Test Inference (Both Facilities)

- [ ] Run test inference:
  ```bash
  cd /opt/llama.cpp
  ./bin/llama-cli \
    -m ./models/llama-3.1-70b-instruct-q4_k_m.gguf \
    -n 100 \
    -p "A young person asks me about anxiety. I respond: " \
    --gpu-layers 80
  ```
- [ ] Measure response time (should be 10-25 seconds for 100 tokens)
- [ ] Document actual performance:
  - [ ] Facility A response time: `___ seconds`
  - [ ] Facility B response time: `___ seconds`

### Start Llama Server (Both Facilities)

- [ ] Create `/etc/systemd/system/llama-inference.service`:
  - [ ] Set GPU layers: 80
  - [ ] Set batch size: 512
  - [ ] Set context size: 2048
  - [ ] Listen on 0.0.0.0:8000
- [ ] Enable and start:
  ```bash
  sudo systemctl enable llama-inference
  sudo systemctl start llama-inference
  ```
- [ ] Verify service running and GPU loaded:
  ```bash
  nvidia-smi | grep llama-server
  ```

### Test Llama API (Both Facilities)

- [ ] Test completion endpoint:
  ```bash
  curl -X POST http://localhost:8000/completion \
    -H "Content-Type: application/json" \
    -d '{"prompt": "Youth asks: What should I do about stress? Response: ", "n_predict": 100}'
  ```
- [ ] Verify response returned successfully
- [ ] Test response time: typical inference should take 10-20 seconds

**Deliverable:** Both facilities running Llama 70B inference with tested response times

---

## Phase 6: Integration & Testing (Week 4)

### Connect Backend to Llama

- [ ] Update `server/services/llamaService.js` with correct LLM_API_URL
- [ ] Add LLM_API_URL to `.env` file:
  ```
  LLM_API_URL=http://localhost:8000
  ```
- [ ] Restart backend service

### End-to-End Testing - Facility A

- [ ] Access frontend: `http://<Facility A IP>:8080`
- [ ] Log in as staff member
- [ ] Create test youth profile
- [ ] Send test message ("Hi, I'm worried about school")
- [ ] Verify Llama response returned within 20 seconds
- [ ] Check response is coherent and appropriate
- [ ] Verify conversation saved to MongoDB
- [ ] Check audit log entry created for conversation

### End-to-End Testing - Facility B

- [ ] Repeat all testing steps for Facility B
- [ ] Verify both facilities independently functional

### Load Testing (Optional)

- [ ] Send 5 concurrent message requests to each facility
- [ ] Verify no crashes or memory issues
- [ ] Check response times under load
- [ ] Monitor GPU temperature (should stay < 70°C)

### Performance Benchmarking

- [ ] Measure response time for 100-token response:
  - [ ] Facility A: `___ seconds`
  - [ ] Facility B: `___ seconds`
- [ ] Measure response time for 256-token response:
  - [ ] Facility A: `___ seconds`
  - [ ] Facility B: `___ seconds`
- [ ] Document findings for future optimization

**Deliverable:** Both facilities tested, deployed, and performing within expected parameters

---

## Phase 7: VPN Federation Setup (Week 4-5)

### Install WireGuard (Both Facilities)

- [ ] Install WireGuard:
  ```bash
  sudo apt-get install -y wireguard wireguard-tools
  ```
- [ ] Generate key pair on each Jetson:
  ```bash
  umask 077
  wg genkey | tee privatekey | wg pubkey > publickey
  cat privatekey  # Save this securely
  cat publickey   # Share with other facility
  ```

### WireGuard Configuration - Facility A

- [ ] Create `/etc/wireguard/wg0.conf`:
  ```ini
  [Interface]
  Address = 10.10.1.1/24
  ListenPort = 51820
  PrivateKey = <Facility_A_Private_Key>
  PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
  PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

  [Peer]
  PublicKey = <Facility_B_Public_Key>
  AllowedIPs = 10.10.2.1/32
  Endpoint = <Facility_B_Public_IP>:51820
  PersistentKeepalive = 25
  ```
- [ ] Set permissions: `sudo chmod 600 /etc/wireguard/wg0.conf`
- [ ] Enable and start:
  ```bash
  sudo systemctl enable wg-quick@wg0
  sudo systemctl start wg-quick@wg0
  ```

### WireGuard Configuration - Facility B

- [ ] Create `/etc/wireguard/wg0.conf` (as server role):
  ```ini
  [Interface]
  Address = 10.10.2.1/24
  PrivateKey = <Facility_B_Private_Key>

  [Peer]
  PublicKey = <Facility_A_Public_Key>
  AllowedIPs = 10.10.1.1/32
  Endpoint = <Facility_A_Public_IP>:51820
  PersistentKeepalive = 25
  ```
- [ ] Enable and start WireGuard

### Test VPN Tunnel

- [ ] From Facility A, ping Facility B:
  ```bash
  ping -c 5 10.10.2.1
  ```
- [ ] From Facility B, ping Facility A:
  ```bash
  ping -c 5 10.10.1.1
  ```
- [ ] Verify ping time < 200ms (should be ~50-100ms)
- [ ] Monitor tunnel status:
  ```bash
  sudo wg show
  ```

### Deploy Federation Endpoints (Both Facilities)

- [ ] Add `/server/routes/federation.js` implementing sync endpoints
- [ ] Update Express app to include federation router
- [ ] Restart backend service on both facilities
- [ ] Test federation endpoint:
  ```bash
  curl -X GET http://10.10.2.1:3000/api/federation/health \
    -H "Authorization: Bearer <JWT_TOKEN>"
  ```

### Schedule Nightly Sync (Both Facilities)

- [ ] Create `/server/services/federationSync.js` sync scheduler
- [ ] Add to main Express startup
- [ ] Update cron to run at 2 AM:
  ```bash
  0 2 * * * /opt/carebridge/server/services/federationSync.js
  ```
- [ ] Verify first sync completes successfully

**Deliverable:** VPN tunnel established between facilities, federation endpoints tested, nightly sync scheduled

---

## Phase 8: Security Hardening (Week 5)

### Network Security

- [ ] Configure firewall rules (both facilities):
  ```bash
  sudo ufw allow from 10.10.2.0/24 to any port 3000  # Allow Facility B
  sudo ufw allow from 192.168.X.0/24 to any port 3000  # Allow staff network
  sudo ufw default deny incoming
  sudo ufw default allow outgoing
  ```
- [ ] Verify no unnecessary ports open: `sudo ufw status numbered`
- [ ] Enable UFW logging: `sudo ufw logging on`

### Data Encryption

- [ ] Verify HTTPS enabled on backend:
  ```bash
  # Create self-signed cert (or use facility's signed cert)
  openssl req -x509 -newkey rsa:4096 -keyout server.key -out server.cert -days 365
  ```
- [ ] Update Express to use HTTPS:
  ```javascript
  const https = require('https');
  const fs = require('fs');
  const app = require('./app');
  
  const options = {
    key: fs.readFileSync('server.key'),
    cert: fs.readFileSync('server.cert')
  };
  
  https.createServer(options, app).listen(3000);
  ```
- [ ] Restart backend and verify HTTPS:
  ```bash
  curl -k https://localhost:3000
  ```

### Authentication & Authorization

- [ ] Verify JWT tokens have short expiration (15 min recommended)
- [ ] Implement token refresh mechanism
- [ ] Verify role-based access control working:
  - [ ] Youth can only access own conversations
  - [ ] Staff can access youth data for assigned youth
  - [ ] Admin can access facility-wide reports
- [ ] Test permission boundaries with unauthorized requests

### Audit Logging

- [ ] Verify all operations logged to MongoDB:
  - [ ] User login/logout
  - [ ] Data access
  - [ ] Youth conversations
  - [ ] Configuration changes
- [ ] Test audit log integrity:
  ```bash
  mongosh carebridge --eval "db.audit_logs.count()"
  ```

### Secrets Management

- [ ] Store all secrets in environment variables or secure vault:
  - [ ] JWT_SECRET
  - [ ] MONGODB_PASSWORD
  - [ ] ENCRYPTION_KEY
- [ ] Document secrets (store separately in secure location):
  - [ ] Facility A secrets: `____________`
  - [ ] Facility B secrets: `____________`
- [ ] Verify .env not committed to git

**Deliverable:** Network hardened, encryption enabled, audit logging verified, secrets secured

---

## Phase 9: Monitoring & Maintenance (Week 5)

### Setup Monitoring (Both Facilities)

- [ ] Install monitoring tools:
  ```bash
  sudo apt-get install -y jtop
  ```
- [ ] Create monitoring dashboard (or use existing tools):
  - [ ] GPU utilization (should be 70-90% during inference)
  - [ ] GPU memory (should be ~38GB used by model)
  - [ ] CPU utilization (should be 20-30%)
  - [ ] System temperature (should stay < 75°C)
  - [ ] Disk space (verify ~20GB free for logs, snapshots)

### Setup Logging

- [ ] Configure log rotation:
  ```bash
  sudo apt-get install -y logrotate
  ```
- [ ] Create `/etc/logrotate.d/carebridge`:
  ```
  /var/log/carebridge/*.log {
    daily
    rotate 7
    compress
    delaycompress
    notifempty
  }
  ```

### Setup Backups (Both Facilities)

- [ ] Create backup script `/opt/carebridge/backup.sh`:
  ```bash
  #!/bin/bash
  mongodump --out /mnt/backups/mongodb_$(date +%Y%m%d)
  tar -czf /mnt/backups/app_config_$(date +%Y%m%d).tar.gz /opt/carebridge/.env
  ```
- [ ] Schedule daily at 2 AM:
  ```bash
  0 2 * * * /opt/carebridge/backup.sh
  ```
- [ ] Test restore procedure
- [ ] Document backup location and retention (7 days recommended)

### Create Runbooks

- [ ] Document restart procedures
- [ ] Document emergency access procedures
- [ ] Document troubleshooting steps
- [ ] Create on-call escalation procedures

**Deliverable:** Monitoring, logging, and backups configured; runbooks created

---

## Phase 10: Staff Training & Go-Live (Week 6)

### Staff Training Sessions

- [ ] Conduct training for facility A staff (2 hours):
  - [ ] System overview
  - [ ] Access procedures
  - [ ] How to use youth interface
  - [ ] How to generate briefings
  - [ ] Emergency access procedures
  - [ ] Incident reporting
- [ ] Conduct training for facility B staff (2 hours)

### Pilot Period

- [ ] Run pilot with small group of youth at Facility A (1-2 weeks)
- [ ] Collect feedback on:
  - [ ] Response time acceptability
  - [ ] Response quality/appropriateness
  - [ ] User interface usability
  - [ ] System stability
- [ ] Document issues and bugs discovered

### Issue Resolution

- [ ] Track all issues in ticketing system
- [ ] Resolve blocking issues before Facility B launch
- [ ] Document workarounds for non-blocking issues
- [ ] Update runbooks based on issues discovered

### Facility B Launch

- [ ] Deploy same configuration to Facility B
- [ ] Apply any improvements from Facility A pilot
- [ ] Conduct go-live with proper supervision
- [ ] Monitor first 24-48 hours closely

### Post-Launch Monitoring

- [ ] Monitor both facilities closely for 1 week
- [ ] Check daily:
  - [ ] System uptime
  - [ ] Error rates
  - [ ] Performance metrics
  - [ ] Audit logs for unusual activity
- [ ] Schedule weekly ops review meetings
- [ ] Plan for ongoing maintenance and updates

**Deliverable:** Staff trained, pilot completed, both facilities live and operational

---

## Success Criteria Checklist

By end of deployment, verify:

### Functional Requirements
- [ ] Both facilities independently operational
- [ ] Youth can initiate conversations
- [ ] Llama 70B responses generated within 20 seconds
- [ ] Staff can generate briefings from conversations
- [ ] Break-glass emergency access working
- [ ] All 121 tests passing on both facilities

### Performance Requirements
- [ ] Inference response time < 25 seconds for typical conversation
- [ ] Backend API response time < 500ms (not including inference)
- [ ] Database query response time < 100ms
- [ ] System uptime > 99% during operational hours

### Security Requirements
- [ ] All data encrypted at rest (MongoDB)
- [ ] HTTPS enabled for all API endpoints
- [ ] JWT authentication working
- [ ] Role-based access control enforced
- [ ] All operations audit logged
- [ ] No PII in federation sync data

### Operational Requirements
- [ ] Backup and restore tested and working
- [ ] Monitoring dashboard operational
- [ ] Staff trained and confident
- [ ] Runbooks created and accessible
- [ ] On-call procedures defined
- [ ] Hotline for emergency support established

### Business Requirements
- [ ] Cost per facility: < $2,000/year operating
- [ ] Deployment time per facility: < 1 week
- [ ] Data remains local (no cloud dependency)
- [ ] Youth of facility privacy maintained

---

## Budget Summary

| Category | Cost | Notes |
|----------|------|-------|
| **Hardware (2x)** | $4,200 | Jetson + carrier + SSD |
| **Networking** | $500 | Cables, USB hubs |
| **First-year Operating** | $400 | Power costs (~$200/facility) |
| **Labor (estimated)** | $8,000 | 160 hours @ $50/hr |
| **Contingency (15%)** | $1,900 | For unforeseen costs |
| **TOTAL** | **$15,000** | For 2-facility pilot |
| **Per Facility** | **$7,500** | Hardware + labor |

**No recurring cloud costs** (MongoDB, inference all local)

---

## Timeline Summary

| Phase | Duration | Key Deliverables |
|-------|----------|------------------|
| 1: Hardware | 1 week | Both Jetson systems powered on |
| 2: OS Setup | 1 week | JetPack installed, services ready |
| 3: Database | 1 week | MongoDB running, backups working |
| 4: Application | 1 week | CareBridge running, 121 tests pass |
| 5: Llama 70B | 1 week | Inference tested, response times benchmarked |
| 6: Integration | 1 week | End-to-end tested on both facilities |
| 7: VPN Federation | 1 week | Tunnel established, sync working |
| 8: Security | 1 week | Hardened, encrypted, audit logged |
| 9: Monitoring | 1 week | Backups, logging, monitoring configured |
| 10: Training & Go-Live | 1 week | Staff trained, pilot completed, production launch |
| **TOTAL** | **10 weeks** | **Both facilities production-ready** |

---

## Sign-Off

- [ ] Infrastructure Lead: __________________ Date: _______
- [ ] Facility A Director: __________________ Date: _______
- [ ] Facility B Director: __________________ Date: _______
- [ ] Security Officer: __________________ Date: _______

---

**Document Version:** 1.0  
**Created:** March 3, 2026  
**Owner:** CareBridge Infrastructure Team  
**Last Updated:** March 3, 2026
