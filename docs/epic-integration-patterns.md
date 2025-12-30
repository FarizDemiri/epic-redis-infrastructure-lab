# Epic EHR Redis Integration Patterns

Quick reference for Redis use cases in Epic healthcare environments.

---

## Pattern 1: Session Management (MyChart/Hyperspace)

**Epic Systems**: MyChart patient portal, Hyperspace clinician workspace  
**Data Structure**: `HASH` + `TTL`  
**Volume**: 10,000+ concurrent sessions  

**Pattern**:

```redis
HSET session:{id} user_id "dr_smith" role "physician" department "emergency"
EXPIRE session:{id} 1800  # 30-minute HIPAA timeout
HGET session:{id} role    # Fast permission check
```

**Why Redis**:

- Sub-1ms lookup vs 50-100ms database
- Automatic expiration (HIPAA compliance)
- Handles millions of session checks/day

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
