# Epic EHR Redis Integration Patterns

Quick reference for Redis use cases in Epic healthcare environments.

---

## Pattern 1: Session Management (MyChart/Hyperspace)

**Epic Systems**: MyChart patient portal, Hyperspace clinician workspace  
**Data Structure**: `HASH` + `TTL`  
**Volume**: 10,000+ concurrent sessions  

**Pattern**:

```redis
# Store clinician session on login
# Key format: session:{session_id}
# TTL enforces HIPAA 30-minute timeout requirement
HSET session:abc123 \
  user_id "dr_smith_12345" \
  role "physician" \
  department "emergency" \
  portal "Hyperspace" \
  last_activity "2025-12-30T22:00:00Z"
EXPIRE session:abc123 1800  # 30 minutes (HIPAA requirement)

# Fast permission check on every page load
# Sub-1ms response enables seamless clinical workflows
HGET session:abc123 role
# Returns: "physician"

# Update last activity (refresh TTL)
HSET session:abc123 last_activity "2025-12-30T22:15:00Z"
EXPIRE session:abc123 1800  # Reset 30-min countdown

# Check if session is still valid
TTL session:abc123
# Returns: 1785 (seconds remaining)

# Cleanup on explicit logout
DEL session:abc123
```

**Real-World Scenario:**

```
10:00 AM: Dr. Smith logs into Hyperspace
          → Session created with 30-min TTL
10:25 AM: Still reviewing patient chart
          → Each page load resets TTL
10:30 AM: Called away to emergency
          → No activity for 30 minutes
11:00 AM: Session automatically expires
          → HIPAA compliance: inactive session removed
11:05 AM: Dr. Smith returns, must re-authenticate
          → Security: no orphaned sessions
```

**Why Redis**:

- **Sub-1ms lookup** vs 50-100ms database (critical for page loads)
- **Automatic expiration** ensures HIPAA compliance without cron jobs
- **Handles millions** of session checks/day across 10,000+ concurrent clinicians
- **No stale data** - TTL guarantees cleanup

---

## Pattern 2: Message Queueing (Interconnect HL7)

**Epic Systems**: Interconnect integration engine  
**Data Structure**: `LIST` (FIFO)  
**Volume**: 10,000+ HL7 messages/hour  

**Pattern**:

```redis
LPUSH interconnect:queue "ADT^A01|Patient admitted"  # Epic pushes
RPOP interconnect:queue                               # Worker pulls
LPUSH interconnect:dlq "FAILED: Invalid message"     # Dead letter queue
```

**Why Redis**:

- FIFO ordering guaranteed
- Buffers message spikes (8am shift change)
- Atomic operations (no lost messages)

---

## Pattern 3: Real-Time Caching (CLOC Dashboards)

**Epic Systems**: Clinical Operations Center AI  
**Data Structure**: `String` + LRU eviction  
**Volume**: 120,000 queries/minute  

**Pattern**:

```redis
CONFIG SET maxmemory 2gb
CONFIG SET maxmemory-policy allkeys-lru

SET patient:bed427:vitals '{"hr":78,"bp":"120/80"}' EX 60
GET patient:bed427:vitals  # Cache hit = fast
```

**Why Redis**:

- 200x faster than PostgreSQL for reads
- Automatic eviction (most-viewed stays cached)
- Short TTL (60 sec) keeps data fresh

---

## Data Structure Decision Matrix

| Use Case | Data Structure | Why |
|----------|---------------|-----|
| User sessions | HASH | Multiple fields per session, efficient updates |
| Message queues | LIST | FIFO ordering, atomic push/pop |
| Real-time cache | String | Simple key-value, fast get/set |
| Patient locations | String/Hash | Depends on query pattern |
| Nurse assignments | Set | Unique list, fast membership checks |

---

## Production Considerations

**Security**:

- TLS 1.3 encryption (PHI in transit)
- Redis ACLs (role-based access)
- Encrypted persistence (RDB/AOF files)
- Audit logging

**High Availability**:

- Sentinel for <10 sec automatic failover
- Cluster for horizontal scaling (>64GB datasets)
- Replicas for read scaling + redundancy

**Monitoring**:

- Memory usage & eviction rates
- Session TTL patterns (detect timeout issues)
- Queue depth (Interconnect backlog alerts)
- Cache hit ratio (CLOC performance)

---

**Architecture**: See [Replication Setup](runbooks/replication-setup.md) | [Sentinel HA](runbooks/sentinel-ha.md)
