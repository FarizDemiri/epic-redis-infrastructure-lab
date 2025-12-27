# Redis Sentinel High Availability - Operational Runbook

## Overview

This runbook provides operational procedures for deploying and managing Redis Sentinel for automated high availability. Sentinel eliminates manual intervention during master failures by automatically detecting outages, achieving consensus, and promoting replicas to master status.

---

## Architecture

### Topology

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Sentinel 1  │  │  Sentinel 2  │  │  Sentinel 3  │
│   (26379)    │  │   (26380)    │  │   (26381)    │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │ (Monitoring & Consensus)
                         ▼
              ┌─────────────────┐
              │  Master (6379)  │
              └────────┬────────┘
                       │
                  ┌────┴────┐
                  │         │
                  ▼         ▼
            ┌─────────┐ ┌─────────┐
            │Replica 1│ │Replica 2│
            │ (6380)  │ │ (6381)  │
            └─────────┘ └─────────┘
```

### Components

| Component | Port | Role | Purpose |
|-----------|------|------|---------|
| Sentinel 1 | 26379 | Monitor | Health checks, quorum voting, failover orchestration |
| Sentinel 2 | 26380 | Monitor | Health checks, quorum voting, failover orchestration |
| Sentinel 3 | 26381 | Monitor | Health checks, quorum voting, failover orchestration |
| Master | 6379 | Data Storage | Handles writes, replicates to slaves |
| Replica 1 | 6380 | Data Storage | Serves reads, failover candidate |
| Replica 2 | 6381 | Data Storage | Serves reads, failover candidate |

### Quorum Configuration

**Quorum = 2** (requires 2 of 3 Sentinels to agree)

**Why this prevents false positives:**

- Network partition isolates 1 Sentinel → Cannot achieve quorum → No unnecessary failover
- Actual master failure → All 3 Sentinels detect → Quorum reached → Automatic promotion

---

## Deployment Procedure

### Prerequisites

- Redis master + replicas already deployed (see [Replication Setup Runbook](replication-setup.md))
- Sentinel feature available in Redis installation
- Root or sudo access
- Ports 26379, 26380, 26381 available

### Step 1: Create Sentinel Configuration Directory

```bash
sudo mkdir -p /etc/redis/sentinel
```

### Step 2: Create Sentinel Configuration Files

**Sentinel 1** (`/etc/redis/sentinel/sentinel-26379.conf`):

```
port 26379
dir /var/lib/redis/sentinel-26379
logfile /var/log/redis/sentinel-26379.log

# Monitor master named 'mymaster' at 127.0.0.1:6379 with quorum=2
sentinel monitor mymaster 127.0.0.1 6379 2

# Master considered down after 5 seconds of no response
sentinel down-after-milliseconds mymaster 5000

# Only sync 1 replica at a time during recovery
sentinel parallel-syncs mymaster 1

# Failover must complete within 10 seconds
sentinel failover-timeout mymaster 10000
```

**Sentinel 2** (`/etc/redis/sentinel/sentinel-26380.conf`):

```
port 26380
dir /var/lib/redis/sentinel-26380
logfile /var/log/redis/sentinel-26380.log

sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 10000
```

**Sentinel 3** (`/etc/redis/sentinel/sentinel-26381.conf`):

```
port 26381
dir /var/lib/redis/sentinel-26381
logfile /var/log/redis/sentinel-26381.log

sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 10000
```

**Key Configuration Parameters:**

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `monitor mymaster <ip> <port> <quorum>` | 127.0.0.1 6379 2 | Master to monitor, quorum threshold |
| `down-after-milliseconds` | 5000 | Timeout before marking master as down |
| `parallel-syncs` | 1 | Number of replicas to resync simultaneously |
| `failover-timeout` | 10000 | Maximum failover duration |

### Step 3: Create Sentinel Data Directories

```bash
sudo mkdir -p /var/lib/redis/sentinel-26379
sudo mkdir -p /var/lib/redis/sentinel-26380
sudo mkdir -p /var/lib/redis/sentinel-26381
sudo chown redis:redis /var/lib/redis/sentinel-26379
sudo chown redis:redis /var/lib/redis/sentinel-26380
sudo chown redis:redis /var/lib/redis/sentinel-26381
```

### Step 4: Start Sentinel Instances

```bash
# Start Sentinels in sentinel mode
sudo redis-server /etc/redis/sentinel/sentinel-26379.conf --sentinel &
sudo redis-server /etc/redis/sentinel/sentinel-26380.conf --sentinel &
sudo redis-server /etc/redis/sentinel/sentinel-26381.conf --sentinel &

# Verify processes started
ps aux | grep sentinel
```

**Expected output**: 3 processes showing `redis-server *:26379 [sentinel]`, `*:26380 [sentinel]`, `*:26381 [sentinel]`

---

## Verification Procedures

### Check Sentinel Status

```bash
# Connect to Sentinel CLI
redis-cli -p 26379

# View monitored masters
SENTINEL masters

# View other Sentinel nodes
SENTINEL sentinels mymaster

# View replicas
SENTINEL slaves mymaster

# Get current master address
SENTINEL get-master-addr-by-name mymaster

# Exit
exit
```

**Expected output from `SENTINEL masters`:**

- Name: mymaster
- IP: 127.0.0.1
- Port: 6379
- Quorum: 2
- Flags: master

**Expected output from `SENTINEL sentinels mymaster`:**

- 2 other Sentinel nodes listed (excluding the one you're querying)

### Monitor Sentinel Logs

```bash
# Tail logs in real-time
sudo tail -f /var/log/redis/sentinel-26379.log
```

**Healthy log patterns:**

```
+sentinel sentinel <id> <ip> <port> @ mymaster <master_ip> <master_port>
# Configuration saved
```

**No errors or `+sdown` messages indicate healthy state**

---

## Automatic Failover Process

### Failover Trigger Conditions

Sentinel initiates failover when:

1. Master does not respond to PING for `down-after-milliseconds` (5 seconds)
2. Quorum of Sentinels (2 of 3) agree master is down (`+odown` state)
3. Sentinel leader elected to orchestrate failover

### Failover Sequence (Automated)

**1. Detection Phase (0-5 seconds):**

```
+sdown master mymaster 127.0.0.1 6379
# Subjectively down - individual Sentinel marks master as down
```

**2. Quorum Phase (5-6 seconds):**

```
+odown master mymaster 127.0.0.1 6379
# Objectively down - quorum achieved (2 of 3 Sentinels agree)
```

**3. Leader Election (6-7 seconds):**

```
+vote-for-leader <sentinel_id> <epoch>
# Sentinels vote for leader to perform failover
```

**4. Replica Selection (7-8 seconds):**

```
+failover-triggered master mymaster 127.0.0.1 6379
+failover-state-select-slave
+selected-slave slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6379
```

**Replica selection criteria (in order):**

- Lowest `replica-priority` value (default 100)
- Highest replication offset (most up-to-date)
- Lexicographically lowest Run ID (tiebreaker)

**5. Promotion (8-9 seconds):**

```
+failover-state-send-slaveof-noone slave 127.0.0.1:6380
# Sentinel executes: REPLICAOF NO ONE on selected replica
```

**6. Reconfiguration (9-10 seconds):**

```
+failover-state-reconf-slaves
+slave-reconf-sent slave 127.0.0.1:6381
# Remaining replicas reconfigured to follow new master
```

**7. Completion:**

```
+failover-end
+switch-master mymaster 127.0.0.1 6379 127.0.0.1 6380
# Failover complete - new master at 6380
```

**Total Time: 5-10 seconds**

### Post-Failover Verification

```bash
# Verify new master address
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster

# Check new master role
NEW_MASTER_PORT=$(redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster | tail -1)
redis-cli -p $NEW_MASTER_PORT INFO replication | grep role

# Verify replicas are following new master
redis-cli -p 26379 SENTINEL slaves mymaster
```

---

## Troubleshooting Guide

### Issue: Sentinel Cannot Start

**Symptoms:**

- Sentinel process exits immediately
- No log file created

**Diagnosis:**

```bash
# Test config file syntax
redis-server /etc/redis/sentinel/sentinel-26379.conf --test-config

# Check for port conflicts
sudo netstat -nltp | grep 26379

# Verify data directory permissions
ls -ld /var/lib/redis/sentinel-26379
```

**Common causes:**

1. **Config syntax error**: Fix typo in config file
2. **Port already in use**: Kill conflicting process or change port
3. **Permission denied**: `sudo chown redis:redis /var/lib/redis/sentinel-*`

### Issue: Sentinel Shows Master as Down (`+sdown`) but It's Running

**Symptoms:**

- Sentinel logs show `+sdown master mymaster`
- Master responds to direct PING

**Diagnosis:**

```bash
# Check if master is actually responding
redis-cli -p 6379 PING

# Check master bind address
redis-cli -p 6379 CONFIG GET bind

# Test connectivity from Sentinel perspective
telnet 127.0.0.1 6379
```

**Common causes:**

1. **Firewall blocking**: Sentinel cannot reach master (check `iptables`, security groups)
2. **Master bind address**: Master bound to different interface (update Sentinel config)
3. **Network latency**: Increase `down-after-milliseconds` if latency >5 seconds

### Issue: Failover Not Triggered Despite Master Failure

**Symptoms:**

- Master is down
- Sentinel logs show `+sdown` but not `+odown`
- No failover occurs

**Diagnosis:**

```bash
# Check how many Sentinels are alive
redis-cli -p 26379 SENTINEL sentinels mymaster

# Check quorum configuration
redis-cli -p 26379 SENTINEL master mymaster | grep quorum
```

**Common causes:**

1. **Insufficient Sentinels**: <2 Sentinels alive → cannot reach quorum
   - Solution: Restart failed Sentinels
2. **Quorum misconfigured**: Quorum set too high (e.g., quorum=3 with 3 Sentinels)
   - Solution: Set quorum to `floor(N/2) + 1` (for 3 Sentinels → quorum=2)
3. **Network partition**: Sentinels isolated from each other
   - Solution: Fix network connectivity

### Issue: Failover Triggered but Stuck

**Symptoms:**

- `+failover-triggered` logged
- `+failover-end` never appears
- System stuck in intermediate state

**Diagnosis:**

```bash
# Check failover state
redis-cli -p 26379 SENTINEL master mymaster | grep failover

# Review Sentinel logs for errors
sudo tail -100 /var/log/redis/sentinel-26379.log
```

**Common causes:**

1. **All replicas down**: No replica available to promote
   - Solution: Restore at least one replica
2. **Replica not responding**: Selected replica unresponsive
   - Solution: Sentinel will retry with different replica after timeout
3. **Network issues**: Sentinel cannot reach replica
   - Solution: Fix network connectivity

### Issue: Old Master Rejoins as Master (Split-Brain)

**Symptoms:**

- Original master comes back online
- Acts as master instead of replica
- Two masters exist simultaneously

**Diagnosis:**

```bash
# Check both instances
redis-cli -p 6379 INFO replication | grep role
redis-cli -p 6380 INFO replication | grep role
# Both show role:master = problem!

# Check Sentinel's view
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
```

**Resolution:**

```bash
# Demote old master to replica
redis-cli -p 6379 REPLICAOF 127.0.0.1 6380

# Sentinel should detect and update configuration
# Verify after 10 seconds
redis-cli -p 6379 INFO replication | grep role
# Should show role:slave
```

**Prevention**: Sentinel automatically handles this via `+convert-to-slave` when old master rejoins

---

## Monitoring & Alerting

### Critical Metrics

| Metric | Command | Healthy State | Alert Threshold |
|--------|---------|---------------|-----------------|
| Sentinel status | `redis-cli -p 26379 PING` | PONG | No response |
| Master status | `SENTINEL masters` → flags | `master` | `s_down` or `o_down` |
| Quorum reachable | `SENTINEL sentinels mymaster` | 2+ nodes | <2 nodes |
| Failover in progress | `SENTINEL masters` → failover-state | `none` | stuck >60 sec |
| Master switches | Count `+switch-master` in logs | <1/day | >3/day |

### Health Check Script

```bash
#!/bin/bash
# sentinel-health-check.sh

MASTER_NAME="mymaster"
SENTINEL_PORT=26379
EXPECTED_SENTINELS=3

# Check Sentinel is responding
if ! redis-cli -p $SENTINEL_PORT PING | grep -q PONG; then
    echo "CRITICAL: Sentinel on port $SENTINEL_PORT not responding"
    exit 2
fi

# Check master status
MASTER_STATUS=$(redis-cli -p $SENTINEL_PORT SENTINEL master $MASTER_NAME)
if echo "$MASTER_STATUS" | grep -q "odown\|sdown"; then
    echo "CRITICAL: Master is down"
    exit 2
fi

# Check quorum
SENTINEL_COUNT=$(redis-cli -p $SENTINEL_PORT SENTINEL sentinels $MASTER_NAME | grep -c "name")
SENTINEL_COUNT=$((SENTINEL_COUNT + 1))  # Add self
if [ "$SENTINEL_COUNT" -lt 2 ]; then
    echo "WARNING: Only $SENTINEL_COUNT Sentinels available (quorum cannot be reached)"
    exit 1
fi

# Check for ongoing failover
if echo "$MASTER_STATUS" | grep -q "failover-state"; then
    FAILOVER_STATE=$(echo "$MASTER_STATUS" | grep failover-state | cut -d: -f2)
    if [ "$FAILOVER_STATE" != "none" ]; then
        echo "WARNING: Failover in progress (state: $FAILOVER_STATE)"
        exit 1
    fi
fi

echo "OK: Sentinel cluster healthy"
exit 0
```

### Recommended Alerts

**Critical Alerts:**

- Sentinel process down
- Master marked as `+odown`
- <2 Sentinels available (quorum unreachable)
- Failover stuck for >60 seconds

**Warning Alerts:**

- Individual Sentinel marks master as `+sdown` (transient network issue)
- Failover occurred (requires review of root cause)
- >5 master switches in 24 hours (flapping issue)

---

## Configuration Best Practices

### Production Hardening

**1. Geographic Distribution:**

```
Datacenter 1: Sentinel 1 + Master
Datacenter 2: Sentinel 2 + Replica 1
Datacenter 3: Sentinel 3 + Replica 2
```

**2. Replica Priority Configuration:**

```bash
# Prefer replica in same datacenter as original master
redis-cli -p 6380 CONFIG SET replica-priority 50

# Fallback to geographically distant replicas
redis-cli -p 6381 CONFIG SET replica-priority 100
```

**3. Authentication:**

```
# In sentinel.conf
sentinel auth-pass mymaster <strong_password>

# In master + replica redis.conf
requirepass <strong_password>
masterauth <strong_password>
```

**4. TLS Encryption:**

```
# Encrypt Sentinel → Master communication
sentinel tls yes
sentinel tls-cert-file /path/to/cert.pem
sentinel tls-key-file /path/to/key.pem
```

### Tuning Parameters for Epic/Healthcare

**High availability over consistency:**

```
# Faster detection (3 sec vs 5 sec)
sentinel down-after-milliseconds mymaster 3000

# Aggressive failover timeout
sentinel failover-timeout mymaster 8000
```

**Minimize client impact during failover:**

```
# Sync replicas sequentially to avoid thundering herd
sentinel parallel-syncs mymaster 1
```

---

## Maintenance Procedures

### Restarting Sentinels (Zero Downtime)

```bash
# Restart one Sentinel at a time
redis-cli -p 26379 SHUTDOWN
sudo redis-server /etc/redis/sentinel/sentinel-26379.conf --sentinel &

# Wait 10 seconds for it to rejoin cluster
sleep 10

# Verify it rejoined
redis-cli -p 26380 SENTINEL sentinels mymaster

# Repeat for other Sentinels
redis-cli -p 26380 SHUTDOWN
sudo redis-server /etc/redis/sentinel/sentinel-26380.conf --sentinel &
sleep 10

redis-cli -p 26381 SHUTDOWN
sudo redis-server /etc/redis/sentinel/sentinel-26381.conf --sentinel &
```

**Note**: Quorum is maintained throughout (2 of 3 always available)

### Changing Quorum Dynamically

```bash
# Reduce quorum to 1 (for maintenance)
redis-cli -p 26379 SENTINEL SET mymaster quorum 1

# Perform maintenance...

# Restore quorum to 2
redis-cli -p 26379 SENTINEL SET mymaster quorum 2
```

**Warning**: Setting quorum=1 means single Sentinel can trigger failover (use only during emergency maintenance)

---

## Comparison with Manual Failover

| Aspect | Manual Failover | Sentinel Failover |
|--------|----------------|-------------------|
| Detection time | Minutes (human response) | 5 seconds (automated) |
| Promotion time | 1-5 minutes (SSH + commands) | 2-3 seconds (automated) |
| Human intervention | Required | None |
| False positive risk | Low (human judgment) | Very low (quorum voting) |
| Night/weekend coverage | Requires on-call | Fully automated |
| Total downtime | 5-30 minutes | 5-10 seconds |

---

## Epic EHR Integration

### Sentinel in Healthcare Context

**Use Case**: MyChart session management during master failure

- 10,000 active clinician sessions
- Master Redis fails
- Sentinel promotes replica in 8 seconds
- Sessions automatically restored from new master
- **Zero session loss**, minimal user impact

**Application Configuration**:

```javascript
// Instead of hardcoding master address
const redis = new Redis({
  sentinels: [
    { host: '127.0.0.1', port: 26379 },
    { host: '127.0.0.1', port: 26380 },
    { host: '127.0.0.1', port: 26381 }
  ],
  name: 'mymaster'
});
// Client automatically discovers current master via Sentinel
```

---

## References

- [Redis Sentinel Documentation](https://redis.io/docs/management/sentinel/)
- [Replication Setup Runbook](replication-setup.md)
- [HIPAA Compliance Controls](../compliance/hipaa-controls.md)

---

**Document Version**: 1.0  
**Applicable Redis Versions**: 6.x, 7.x  
**Tested Environment**: Ubuntu 24.04 LTS, Redis 7.x
