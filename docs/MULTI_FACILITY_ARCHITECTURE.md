# Multi-Facility Architecture & VPN Federation

**System Design:** CareBridge Companion across 2+ facilities  
**Architecture Type:** Distributed edge deployment with optional federation  
**Federation Model:** Peer-to-peer VPN with selective data synchronization  
**Cost per Facility:** ~$1,700 (hardware + networking)

---

## 1. High-Level Architecture

### 1.1 Per-Facility Deployment

Each facility operates **independently and autonomously**:

```
┌─────────────────────────────────────────┐
│           Facility A (West VA)           │
├─────────────────────────────────────────┤
│  Youth Interface                         │
│  (Browser on facility network)           │
│              ↓                           │
│  Express Backend (Port 3000)             │
│  • Youth conversations                   │
│  • Staff briefing generation             │
│  • All real-time operations              │
│              ↓                           │
│  Jetson AGX Orin (Local GPU)             │
│  • Llama 3.1 70B inference               │
│  • Response time: 2-3 seconds            │
│              ↓                           │
│  MongoDB (Local)                         │
│  • Youth data                            │
│  • Conversations                         │
│  • Briefings                             │
│  • Audit logs                            │
│              ↓                           │
│  VPN Gateway (OpenVPN or WireGuard)      │
│  ~ Optional, for federation only         │
└─────────────────────────────────────────┘
         ↓ LAN connections only
    (No external Internet required except
     for VPN federation, which is optional)
```

### 1.2 Federation Network (Optional)

Only enabled if you want to share data across facilities:

```
    Facility A VPN                Facility B VPN
    Gateway (192.168.1.50)        Gateway (192.168.2.50)
              ↓                            ↓
              └──────VPN Tunnel───────────┘
                  IPSec or WireGuard
                  (Only cross-facility
                   briefing sharing,
                   incident escalation,
                   audit log aggregation)

    Throughput: <<100 kbps (batch transfers only)
    Latency: 100-200ms acceptable (not real-time)
    Traffic: Only non-time-sensitive admin data
```

---

## 2. Network Design

### 2.1 Per-Facility Network Topology

**Facility A Network (192.168.1.0/24):**

```
┌──────────────────────────────────────────┐
│        Facility Internet Connection       │
│      (Comcast/AT&T/Local ISP)            │
│       Static IP (Recommended)            │
│       100+ Mbps (for optimal performance)│
└───────────────────┬──────────────────────┘
                    │
        ┌───────────┴──────────────┐
        ↓                          ↓
   Firewall/Gateway         VPN Gateway
   (Gateway: 192.168.1.1)   (192.168.1.50)
        │                        │
        ├─────────┬──────────────┤
        ↓         ↓              ↓
      Server   Admin Staff   Jetson AGX Orin
   (3000/port) Workstations  (GPU Server)
                (Port 3000    (Port 8000)
                via  proxy)   for Llama API
```

**Network Segments:**
- **Management Network:** 192.168.1.0 - 192.168.1.49 (Firewall, gateway)
- **GPU Server:** 192.168.1.51 (Jetson + MongoDB)
- **VPN Gateway:** 192.168.1.50 (For federation)
- **Staff Workstations:** 192.168.1.52-100 (Admin/supervisor access)
- **Guest Network (Optional):** Separate SSID (if WiFi enabled)

### 2.2 Multi-Facility Federation Network

If connecting 2+ facilities:

**Facility A:**
- Local network: 192.168.1.0/24
- VPN IP: 10.10.1.0/24
- Gateway VPN endpoint: 192.168.1.50

**Facility B:**
- Local network: 192.168.2.0/24
- VPN IP: 10.10.2.0/24
- Gateway VPN endpoint: 192.168.2.50

**VPN Tunnel Configuration:**

```
Facility A Backend      Facility B Backend
(192.168.1.51:3000)     (192.168.2.51:3000)
         ↓                      ↓
    VPN Gateway             VPN Gateway
   (10.10.1.1)              (10.10.2.1)
         ↑______IPSec/WireGuard Tunnel______↑

Via VPN tunnel, only share:
• Briefing summaries (not full conversations)
• Incident escalation notices
• Staff access audit trails
• System health metrics
```

---

## 3. VPN Setup Options

### 3.1 Option A: WireGuard (Recommended)

**Why WireGuard?**
- Minimal overhead (kernel-based, ~200ms latency)
- Easy configuration
- Works on low-power Jetson devices
- Modern cryptography

### 3.1.1 Install WireGuard

**On both facilities:**

```bash
# Update system
sudo apt-get update
sudo apt-get upgrade -y

# Install WireGuard
sudo apt-get install -y wireguard wireguard-tools

# Generate keys
umask 077
wg genkey | tee privatekey | wg pubkey > publickey

# Display keys (note these)
cat privatekey
cat publickey
```

### 3.1.2 Configure Facility A (Server)

Create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
# Facility A
Address = 10.10.1.1/24
ListenPort = 51820
PrivateKey = <Facility_A_Private_Key>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# Facility B (Peer)
[Peer]
PublicKey = <Facility_B_Public_Key>
AllowedIPs = 10.10.2.1/32
Endpoint = <Facility_B_Public_IP>:51820
PersistentKeepalive = 25
```

### 3.1.3 Configure Facility B (Client)

Create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
# Facility B
Address = 10.10.2.1/24
PrivateKey = <Facility_B_Private_Key>

[Peer]
PublicKey = <Facility_A_Public_Key>
AllowedIPs = 10.10.1.1/32
Endpoint = <Facility_A_Public_IP>:51820
PersistentKeepalive = 25
```

### 3.1.4 Bring Up VPN

```bash
# On both facilities:
sudo systemctl start wg-quick@wg0
sudo systemctl enable wg-quick@wg0

# Verify connection
sudo wg show

# Test tunnel
ping 10.10.2.1  # From Facility A to Facility B
ping 10.10.1.1  # From Facility B to Facility A
```

### 3.2 Option B: OpenVPN

If WireGuard not available or preferring OpenVPN:

```bash
# Install OpenVPN
sudo apt-get install -y openvpn openvpn-easy-rsa

# Use easy-rsa to generate certificates
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
./easyrsa init-pki
./easyrsa build-ca

# Generate server/client certificates (detailed in OpenVPN docs)
# Then configure server and client conf files
```

**OpenVPN vs WireGuard:**

| Aspect | WireGuard | OpenVPN |
|--------|-----------|---------|
| **Setup** | Simpler (4-5 files) | Complex (certs, keys, configs) |
| **Performance** | Faster (~50ms overhead) | Moderate (~100-150ms) |
| **Stability** | Modern, stable | Very mature, battle-tested |
| **Support** | Growing | Excellent, many tutorials |

**Recommendation:** Use **WireGuard** for this deployment due to simplicity and Jetson optimization.

---

## 4. Backend Synchronization API

### 4.1 Data Sharing Protocol

Only share non-real-time, aggregated data:

**Do NOT sync:**
- ❌ Individual youth conversations
- ❌ Personal identifiable information (PII) raw
- ❌ Real-time briefing generation requests

**DO sync (asynchronously, nightly):**
- ✅ Incident escalation alerts (youth at risk)
- ✅ Briefing summaries (aggregated text, no voice)
- ✅ De-identified staff access audit trails
- ✅ System health metrics (uptime, error rates)

### 4.2 Implement Sync Endpoint

Add to Express backend (`server/routes/federation.js`):

```javascript
const express = require('express');
const router = express.Router();
const BriefingSummary = require('../models/BriefingSummary');
const IncidentLog = require('../models/IncidentLog');
const AuditLog = require('../models/AuditLog');

// Sync incident alerts to partner facilities
router.post('/sync/incidents', authenticateVPN, async (req, res) => {
  try {
    const { facility_id, incidents } = req.body;
    
    // Validate facility ID (whitelist known partners)
    const allowedFacilities = ['facility-02', 'facility-03'];
    if (!allowedFacilities.includes(facility_id)) {
      return res.status(403).json({ error: 'Unauthorized facility' });
    }
    
    // Store external incident for staff review
    const stored = await ExternalIncidentAlert.insertMany(
      incidents.map(inc => ({
        source_facility: facility_id,
        incident_summary: inc.summary,
        severity: inc.severity,
        timestamp: new Date(),
        requires_review: true
      }))
    );
    
    res.json({ stored: stored.length, status: 'success' });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// GET briefing summaries for federation (de-identified)
router.get('/briefings/summary', authenticateVPN, async (req, res) => {
  try {
    const { days = 1 } = req.query;
    const since = new Date(Date.now() - days * 24 * 60 * 60 * 1000);
    
    const summaries = await BriefingSummary.find({ 
      created_at: { $gte: since } 
    })
    .select('category_summary patterns issues generated_at')
    .limit(100);
    
    res.json({
      count: summaries.length,
      period_days: days,
      summaries: summaries
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Sync audit logs (aggregated, no PII)
router.post('/sync/audit-summary', authenticateVPN, async (req, res) => {
  try {
    const { facility_id } = req.body;
    
    // Log federation access
    await AuditLog.create({
      action: 'FEDERATION_SYNC',
      actor_facility: facility_id,
      timestamp: new Date(),
      details: 'Cross-facility audit summary exchange'
    });
    
    // Return aggregated stats only
    const stats = await AuditLog.aggregate([
      { $match: { timestamp: { $gte: new Date(Date.now() - 7*24*60*60*1000) } } },
      { $group: {
          _id: '$action',
          count: { $sum: 1 }
        }
      }
    ]);
    
    res.json({ status: 'logged', stats });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

module.exports = router;
```

### 4.3 Sync Scheduler (Nightly)

Add to `server/services/federationSync.js`:

```javascript
const cron = require('node-cron');
const axios = require('axios');

const FACILITY_ENDPOINTS = {
  'facility-02': process.env.FACILITY_B_VPN_ENDPOINT || 'https://10.10.2.1:3000'
};

// Run every night at 2 AM
cron.schedule('0 2 * * *', async () => {
  console.log('[FederationSync] Starting daily sync...');
  
  for (const [facility_id, endpoint] of Object.entries(FACILITY_ENDPOINTS)) {
    try {
      // Send this facility's incident summary
      const response = await axios.post(
        `${endpoint}/api/federation/sync/incidents`,
        {
          facility_id: process.env.FACILITY_ID,
          incidents: await getRecentIncidents()
        },
        { 
          headers: { 'Authorization': `Bearer ${getVPNToken()}` },
          timeout: 30000
        }
      );
      
      console.log(`[FederationSync] Incident sync to ${facility_id}: ${response.data.stored} items`);
      
    } catch (error) {
      console.error(`[FederationSync] Failed to sync with ${facility_id}:`, error.message);
      // Log but don't crash - federation is optional
    }
  }
  
  console.log('[FederationSync] Completed');
});

async function getRecentIncidents() {
  // Return incidents from last 24 hours (summary only)
  return db.IncidentLog.find({
    timestamp: { $gte: new Date(Date.now() - 24*60*60*1000) }
  }).select('summary severity youth_id').limit(50);
}

function getVPNToken() {
  // Return JWT for VPN auth between facilities
  return sign({ facility_id: process.env.FACILITY_ID }, process.env.JWT_SECRET);
}
```

---

## 5. Security for Federation

### 5.1 Inter-Facility Authentication

Only facilities in whitelist can sync data:

```javascript
// Middleware for VPN requests
const authenticateVPN = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'Missing authorization' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    
    // Check if facility is registered partner
    const allowedFacilities = ['facility-02', 'facility-03'];
    if (!allowedFacilities.includes(decoded.facility_id)) {
      return res.status(403).json({ error: 'Unauthorized facility' });
    }
    
    req.facility_id = decoded.facility_id;
    next();
  } catch (error) {
    res.status(403).json({ error: 'Invalid token' });
  }
};

module.exports = authenticateVPN;
```

### 5.2 Data Encryption in Transit

WireGuard handles tunnel encryption (AES-256 equivalent). For additional layer:

```javascript
// Encrypt sensitive sync data
const crypto = require('crypto');

function encryptSyncPayload(data, key) {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-cbc', Buffer.from(key, 'hex'), iv);
  
  let encrypted = cipher.update(JSON.stringify(data));
  encrypted = Buffer.concat([encrypted, cipher.final()]);
  
  return iv.toString('hex') + ':' + encrypted.toString('hex');
}

function decryptSyncPayload(encryptedData, key) {
  const [ivHex, encryptedHex] = encryptedData.split(':');
  const iv = Buffer.from(ivHex, 'hex');
  const encrypted = Buffer.from(encryptedHex, 'hex');
  
  const decipher = crypto.createDecipheriv('aes-256-cbc', Buffer.from(key, 'hex'), iv);
  let decrypted = decipher.update(encrypted);
  decrypted = Buffer.concat([decrypted, decipher.final()]);
  
  return JSON.parse(decrypted.toString());
}
```

---

## 6. Monitoring & Health Checks

### 6.1 VPN Tunnel Status

```bash
# Monitor WireGuard tunnel
watch -n 5 'sudo wg show'

# Check tunneling latency
ping -c 5 10.10.2.1  # From Facility A to Facility B

# Monitor tunnel uptime
journalctl -u wg-quick@wg0 -f
```

### 6.2 Sync Status Dashboard

Add to Express admin dashboard:

```javascript
router.get('/admin/federation-status', authenticateAdmin, async (req, res) => {
  try {
    const lastSync = await SyncLog.findOne().sort({ timestamp: -1 });
    const vpnStatus = await checkVPNTunnel();
    
    res.json({
      vpn_connected: vpnStatus.connected,
      vpn_latency_ms: vpnStatus.latency,
      last_sync: lastSync?.timestamp,
      last_sync_status: lastSync?.status,
      failed_syncs_24h: await SyncLog.countDocuments({
        status: 'failed',
        timestamp: { $gte: new Date(Date.now() - 24*60*60*1000) }
      })
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

async function checkVPNTunnel() {
  const { exec } = require('child_process');
  const start = Date.now();
  
  return new Promise((resolve) => {
    exec('ping -c 1 10.10.2.1 -W 2', (error) => {
      const latency = Date.now() - start;
      resolve({
        connected: !error,
        latency: latency
      });
    });
  });
}
```

---

## 7. Firewall Rules for Federation

### 7.1 Port Configuration

```bash
# On Facility A firewall
sudo ufw allow from 192.168.2.0/24 to any port 3000 comment "Federation API from Facility B"
sudo ufw allow from 192.168.2.0/24 to any port 22 comment "SSH from Facility B admin"
sudo ufw allow out to any port 51820 proto udp comment "WireGuard to Facility B"

# Verify
sudo ufw status numbered
```

### 7.2 firewall Exceptions for Sync

```bash
# Allow only federation endpoints
sudo ufw allow from 10.10.2.0/24 to 10.10.1.0/24 port 3000 proto tcp

# Log access
sudo ufw logging on
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

---

## 8. Disaster Recovery for Federation

### 8.1 Independent Operation

If VPN tunnel fails:

```javascript
// Graceful degradation - don't require federation for core operations
try {
  await federationSync();
} catch (error) {
  console.warn('Federation unavailable, continuing with local operation');
  // Continue serving local facility without federation
}
```

**Impact of tunnel failure:**
- ❌ Cross-facility incident sharing delayed until tunnel restores
- ❌ Audit trail aggregation paused
- ✅ **Local operations unaffected** (conversations, briefings, youth-facing features work normally)

### 8.2 Restore Procedure

```bash
# After tunnel is back online, sync queued data
# (All queued incidents/summaries synced automatically)

# Check queue:
db.SyncQueue.count()

# Force immediate sync (if needed):
curl -X POST http://localhost:3000/admin/federation/sync-now \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

---

## 9. Scalability to 10+ Facilities

This architecture supports linear scaling:

```
Current: 2 facilities
├─ Facility A ← WireGuard Mesh ─→ Facility B

Future: 5 facilities (Star Topology)
├─ Facility A ─┐
├─ Facility B  │
├─ Facility C  ├─ Central Hub (low power Jetson)
├─ Facility D  │
├─ Facility E ─┘

Future: 10+ facilities (Ring Topology)
├─ A ↔ B ↔ C ↔ D ↔ E ↔ F ↔ G ... ↔ A (loop)
```

**Per Facility Cost Remains ~$1,700**
- Add 10th facility: +$1,700 (linear scaling)
- Add VPN endpoint: +$0 (uses existing hardware)
- Add federation data: ~+$0 (same sync schedule)

---

## 10. Checklist: Federation Deployment

**Pre-Launch:**
- [ ] WireGuard/OpenVPN software installed on both facilities
- [ ] Public IPs documented for both facilities
- [ ] Private keys generated and securely stored
- [ ] VPN tunnel tested with ping
- [ ] Firewall rules configured (port 3000, 51820)

**Before First Sync:**
- [ ] Facility whitelist configured in both backends
- [ ] Federation routes deployed (sync endpoints)
- [ ] Encryption keys exchanged securely
- [ ] Cron job enabled for nightly sync
- [ ] Monitoring dashboard tested

**During Operation:**
- [ ] Check VPN tunnel status daily
- [ ] Monitor sync logs for failures
- [ ] Review federation audit trail monthly
- [ ] Verify no PII leakage in synced data
- [ ] Test incident escalation flow

---

## 11. Reference Architecture Diagrams

### Single Facility (Typical)

```
Users → [Facility A - Jetson] → Local GPU → Local DB
         (Standalone, no federation)
```

### Two Facilities (Recommended for Pilot)

```
[Facility A]              [Facility B]
├─ Youth → GPU           ├─ Youth → GPU
├─ Staff → API           ├─ Staff → API
└─ Jetson/DB    ─VPN─    └─ Jetson/DB
   (Local ops)            (Local ops)
   (VPN: optional)        (VPN: optional)
```

### Future Multi-Facility (5-10)

```
              [Facility A] 
               ↙ ↖ ↑
         [Facility B]  [Facility C]
         ↙ ↘  ↑ ↓   ↙ ↖ ↑
    [Facility D] ... [Facility E]
    
    All peer-to-peer over WireGuard mesh
    Each facility operates independently
    Federation is optional, non-critical
```

---

**Version:** 1.0  
**Last Updated:** March 3, 2026  
**Scope:** 2-10 facility deployments  
**Maintenance Contact:** CareBridge Infrastructure Team
