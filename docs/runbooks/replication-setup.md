# Redis Master-Replica Replication - Operational Runbook

## Overview

This runbook provides operational procedures for deploying and managing Redis master-replica replication architecture. This configuration provides read scalability and high availability for mission-critical applications, including Epic EHR session management and real-time healthcare data caching.

---

## Architecture

### Topology

```
┌─────────────────┐
│  Master (6379)  │ ◄── Handles all writes
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌─────────┐ ┌─────────┐
│Replica 1│ │Replica 2│ ◄── Handle reads, provide failover capability
│ (6380)  │ │ (6381)  │
└─────────┘ └─────────┘
```

### Components

| Component | Port | Role | Purpose |
|-----------|------|------|---------|
| Master | 6379 | Primary | Handles all write operations |
| Replica 1 | 6380 | Secondary | Serves read traffic, HA standby |
| Replica 2 | 6381 | Secondary | Serves read traffic, HA standby |

### Replication Mode

**Asynchronous replication**: Master propagates changes to replicas with eventual consistency. Typical replication lag: <10ms under normal load.

---

## Deployment Procedure

### Prerequisites

- Redis server installed (`redis-server --version` to verify)
- Root or sudo access
- Ports 6379, 6380, 6381 available

### Step 1: Create Replica Configuration Files

```bash
# Copy master config as template
sudo cp /etc/redis/redis.conf /etc/redis/redis-6380.conf
sudo cp /etc/redis/redis.conf /etc/redis/redis-6381.conf
```

### Step 2: Configure Replica 1 (Port 6380)

Edit `/etc/redis/redis-6380.conf`:

```bash
# Port configuration
port 6380

# Process identification
pidfile /var/run/redis/redis-6380.pid

# Logging
logfile /var/log/redis/redis-6380.log

# Data directory
dir /var/lib/redis-6380

# Replication configuration
replicaof 127.0.0.1 6379
```

**Critical**: Ensure `replicaof` directive points to master IP and port (127.0.0.1:6379 for localhost).

### Step 3: Configure Replica 2 (Port 6381)

Edit `/etc/redis/redis-6381.conf`:

```bash
# Port configuration
port 6381

# Process identification
pidfile /var/run/redis/redis-6381.pid

# Logging
logfile /var/log/redis/redis-6381.log

# Data directory
dir /var/lib/redis-6381

# Replication configuration
replicaof 127.0.0.1 6379
```

### Step 4: Create Data Directories

```bash
sudo mkdir /var/lib/redis-6380
sudo mkdir /var/lib/redis-6381
sudo chown redis:redis /var/lib/redis-6380
sudo chown redis:redis /var/lib/redis-6381
```

### Step 5: Start Replica Instances

```bash
# Start replicas
sudo redis-server /etc/redis/redis-6380.conf &
sudo redis-server /etc/redis/redis-6381.conf &

# Verify processes
ps aux | grep redis-server
```

Expected output shows 3 Redis processes (ports 6379, 6380, 6381).

---

## Verification Procedures

### Check Replication Status

**On Master:**

```bash
redis-cli -p 6379 INFO replication
```

Expected output:

```
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6380,state=online,offset=...
slave1:ip=127.0.0.1,port=6381,state=online,offset=...
```

**On Replicas:**

```bash
redis-cli -p 6380 INFO replication
redis-cli -p 6381 INFO replication
```

Expected output for each:

```
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
```

### Test Data Replication

```bash
# Write to master
redis-cli -p 6379 SET test:replication "data"

# Read from replicas
redis-cli -p 6380 GET test:replication  # Should return "data"
redis-cli -p 6381 GET test:replication  # Should return "data"
```

### Verify Read-Only Enforcement

```bash
# Attempt write on replica (should fail)
redis-cli -p 6380 SET test "value"
```

Expected: `(error) READONLY You can't write against a read only replica`

### Monitor Replication Lag

```bash
# Check replication offsets
redis-cli -p 6379 INFO replication | grep master_repl_offset
redis-cli -p 6380 INFO replication | grep slave_repl_offset
redis-cli -p 6381 INFO replication | grep slave_repl_offset
```

**Healthy state**: Offsets match or differ by <1000 bytes under normal load.

---

## Manual Failover Procedure

### Scenario: Master Failure

**Symptoms:**

- Master unreachable
- Application errors indicating write failures
- Replicas show `master_link_status:down`

### Failover Steps

**1. Verify Master Failure**

```bash
# Check master connectivity
redis-cli -p 6379 PING

# Check replica status
redis-cli -p 6380 INFO replication | grep master_link_status
```

If master is confirmed down, proceed to promotion.

**2. Promote Replica to Master**

```bash
# Promote replica 1 (6380) to master
redis-cli -p 6380 REPLICAOF NO ONE

# Verify promotion
redis-cli -p 6380 INFO replication
```

Expected: `role:master` (replica is now independent master)

**3. Reconfigure Remaining Replicas**

```bash
# Point replica 2 to new master
redis-cli -p 6381 REPLICAOF 127.0.0.1 6380

# Verify connection
redis-cli -p 6381 INFO replication | grep master_port
```

Expected: `master_port:6380`

**4. Update Application Configuration**

Update application connection strings to point to new master (127.0.0.1:6380).

**5. Test New Topology**

```bash
# Write to new master
redis-cli -p 6380 SET test:failover "success"

# Verify replication to replica
redis-cli -p 6381 GET test:failover
```

Expected: Data successfully replicated to remaining replica.

**6. Investigate Original Master**

Once service is restored:

- Review logs: `/var/log/redis/redis-server.log`
- Determine root cause (system failure, OOM, network issue)
- Restore original master as replica to new master if needed

---

## Troubleshooting Guide

### Issue: Replica Shows `master_link_status:down`

**Diagnosis:**

```bash
# Check replica logs
sudo tail -50 /var/log/redis/redis-6380.log
```

**Common causes:**

1. **Incorrect master IP/port** in `replicaof` directive
   - Verify: `grep replicaof /etc/redis/redis-6380.conf`
   - Fix: Correct IP/port, restart replica

2. **Network connectivity**
   - Test: `telnet 127.0.0.1 6379`
   - Fix: Check firewall, master bind address

3. **Master authentication required**
   - Check: Master has `requirepass` set
   - Fix: Add `masterauth <password>` to replica config

### Issue: High Replication Lag

**Diagnosis:**

```bash
# Monitor lag continuously
watch -n 1 'redis-cli -p 6379 INFO replication | grep offset'
```

**Common causes:**

1. **Heavy write load on master**
   - Verify: `redis-cli -p 6379 INFO stats | grep instantaneous_ops_per_sec`
   - Solution: Optimize write patterns, consider sharding

2. **Network latency**
   - Test: `ping <master_ip>`
   - Solution: Colocate master and replicas

3. **Slow replica disk I/O**
   - Check: `iostat -x 1`
   - Solution: Use faster storage, disable AOF on replicas (if acceptable)

### Issue: Replica Cannot Start

**Diagnosis:**

```bash
# Check for port conflicts
sudo netstat -nltp | grep 6380

# Check data directory permissions
ls -ld /var/lib/redis-6380
```

**Common causes:**

1. **Port already in use**
   - Fix: Kill conflicting process or change replica port

2. **Permission denied on data directory**
   - Fix: `sudo chown redis:redis /var/lib/redis-6380`

3. **Corrupted RDB/AOF files**
   - Fix: Clear data directory (data will re-sync from master)

---

## Monitoring & Alerting

### Key Metrics to Monitor

| Metric | Command | Threshold | Alert Action |
|--------|---------|-----------|--------------|
| Replication lag | `INFO replication` → `master_repl_offset` vs `slave_repl_offset` | >5000 bytes | Investigate write load, network |
| Master link status | `INFO replication` → `master_link_status` | `down` | Critical: Potential failover needed |
| Connected slaves | `INFO replication` → `connected_slaves` | <2 | Warning: Reduced HA capability |
| Memory usage | `INFO memory` → `used_memory_human` | >80% maxmemory | Scale or evict data |

### Health Check Script

```bash
#!/bin/bash
# redis-replication-health.sh

MASTER="127.0.0.1:6379"
REPLICA1="127.0.0.1:6380"
REPLICA2="127.0.0.1:6381"

# Check master
redis-cli -h ${MASTER%:*} -p ${MASTER#*:} PING || echo "Master DOWN"

# Check replicas connected
SLAVES=$(redis-cli -h ${MASTER%:*} -p ${MASTER#*:} INFO replication | grep connected_slaves | cut -d: -f2)
if [ "$SLAVES" -lt 2 ]; then
    echo "WARNING: Only $SLAVES replicas connected"
fi

# Check replication lag
MASTER_OFFSET=$(redis-cli -h ${MASTER%:*} -p ${MASTER#*:} INFO replication | grep master_repl_offset | cut -d: -f2)
REPLICA_OFFSET=$(redis-cli -h ${REPLICA1%:*} -p ${REPLICA1#*:} INFO replication | grep slave_repl_offset | cut -d: -f2)
LAG=$((MASTER_OFFSET - REPLICA_OFFSET))

if [ "$LAG" -gt 5000 ]; then
    echo "WARNING: Replication lag is $LAG bytes"
fi
```

---

## Performance Considerations

### Read Scaling

With 1 master + 2 replicas:

- **Master**: Handles 100% of writes
- **Replicas**: Can handle 66% of read load each (assuming even distribution)
- **Aggregate read capacity**: 3x single-node capacity

### Write Performance

Replication does NOT improve write performance:

- All writes processed by master
- Replicas add network overhead for propagation
- For write scaling, consider Redis Cluster (sharding)

### Epic EHR Use Case

**Session Management Pattern:**

- 10,000 concurrent clinician sessions
- Writes: Login/logout + activity updates (~1,000 writes/sec)
- Reads: Session validation on every page load (~10,000 reads/sec)
- **Replication benefit**: 10:1 read/write ratio benefits significantly from read replicas

---

## Security Considerations

### Production Hardening

1. **Enable authentication**:

   ```
   requirepass <strong_password>
   masterauth <strong_password>
   ```

2. **Bind to specific interfaces** (not 0.0.0.0):

   ```
   bind 127.0.0.1 <internal_ip>
   ```

3. **Enable TLS** for replication (Epic/HIPAA requirement):

   ```
   tls-replication yes
   tls-cert-file /path/to/cert.pem
   tls-key-file /path/to/key.pem
   ```

4. **Disable dangerous commands**:

   ```
   rename-command FLUSHALL ""
   rename-command FLUSHDB ""
   ```

---

## Maintenance Procedures

### Restarting Replicas (Zero Downtime)

```bash
# Restart replica 1
redis-cli -p 6380 SHUTDOWN
sudo redis-server /etc/redis/redis-6380.conf &

# Verify reconnection
redis-cli -p 6380 INFO replication | grep master_link_status

# Repeat for replica 2
redis-cli -p 6381 SHUTDOWN
sudo redis-server /etc/redis/redis-6381.conf &
```

**Note**: Replicas can be restarted without application impact (reads distributed to remaining replicas).

### Restarting Master (Brief Downtime)

**Planned maintenance:**

1. Notify applications of maintenance window
2. Optionally promote replica to master (zero write downtime)
3. Restart original master
4. Demote temporary master back to replica

---

## Next Steps: Automated Failover

**Manual failover limitations:**

- Requires human intervention (slow)
- Subject to human error
- No automatic recovery during off-hours

**Solution**: Deploy **Redis Sentinel** for automated monitoring and failover.

See: [Sentinel High Availability Runbook](sentinel-ha.md) *(Day 3 implementation)*

---

## References

- [Redis Replication Documentation](https://redis.io/docs/management/replication/)
- [Epic EHR Integration Patterns](../architecture/epic-integration.md)
- [HIPAA Compliance Controls](../compliance/hipaa-controls.md)

---

**Document Version**: 1.0  
**Applicable Redis Versions**: 6.x, 7.x  
**Tested Environment**: Ubuntu 24.04 LTS, Redis 7.x
