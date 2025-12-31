# Epic EHR Redis Infrastructure Lab

Production-grade Redis administration environment for healthcare systems supporting Epic EHR integration patterns.

## Overview

Comprehensive Redis infrastructure lab demonstrating high-availability configurations, HIPAA compliance, and Epic Systems integration for hospital environments.

**Key Focus Areas:**

- High availability for 24/7 clinical operations
- HIPAA-compliant security hardening  
- Epic EHR integration (MyChart, Interconnect, CLOC)
- Production monitoring and incident response

## Architecture

### Infrastructure Stack

**Local Development (Ubuntu Server VM):**

- Redis v7.0.15 cluster + Sentinel
- 6-node cluster (3 masters + 3 replicas)
- Automated failover testing and validation

**AWS Production Deployment:**

- **Platform:** AWS EC2 (3 x t2.micro, Ubuntu 24.04 LTS)
- **Network:** Multi-AZ VPC (us-east-1a, us-east-1b, us-east-1c)
- **Redis:** v7.0.15 cluster (3 masters, 16384 hash slots)
- **Monitoring:** Prometheus + Grafana + Redis Exporter
  - Real-time metrics collection (30s scrape interval)
  - Pre-built dashboards (memory, commands/sec, hit ratio)
  - Alert configuration for cluster health
- **Encryption:** TLS 1.3 (in-transit) + AES-256 EBS (at-rest)
- **Access Control:** Redis ACLs (admin, app, dev roles)
- **High Availability:** Multi-AZ deployment for fault tolerance

### Epic Integration Scenarios

**1. Session Management (MyChart/Hyperspace)**

```
- Hash-based session storage
- 30-minute TTL with automatic expiration
- Sub-100ms latency requirements
- Supports 10,000+ concurrent clinical users
```

**2. Message Queueing (Epic Interconnect)**

```
- Redis Streams for HL7 message buffering
- FIFO processing with guaranteed delivery
- Dead letter queue handling
- Backpressure management
```

**3. Real-Time Caching (CLOC AI)**

```
- Clinical Operations Center data streaming
- Smart-bed telemetry processing
- Patient monitoring dashboard feeds
- Sub-second data freshness requirements
```

## Security & Compliance

**Three-Layer Defense-in-Depth Architecture:**

**Layer 1: Network Security**

- VPC isolation (10.0.0.0/16 private network)
- Security group firewall rules (restricted ports: 22, 6379, 16379)
- SSH access limited to admin IP addresses
- Redis ports restricted to VPC CIDR only

**Layer 2: Transport Encryption (TLS 1.3)**

- All inter-node communication encrypted
- Certificate-based authentication (CA + server certs)
- Protects PHI data in transit between cluster nodes
- TLS enforced for all client connections

**Layer 3: Access Control (Redis ACLs)**

- Role-based permissions (admin, application, developer)
- Principle of least privilege enforcement
- `admin_user`: Full administrative access
- `app_user`: Limited to session:*and patient:* keys
- `dev_user`: Read-only access for debugging
- Default user disabled (forces authentication)

**Layer 4: Data-at-Rest Protection**

- AES-256 encrypted EBS volumes (AWS KMS)
- Protects against physical disk theft
- Encrypted RDB/AOF persistence files
- Secure key management

**HIPAA Compliance Summary:**

- ✅ Encryption in transit (TLS 1.3)
- ✅ Encryption at rest (AES-256 EBS)
- ✅ Access controls (ACL-based roles)
- ✅ Audit logging (`/var/log/redis/`)
- ✅ Network isolation (VPC + security groups)
- ✅ Automatic session expiration (30-min TTL)
- ✅ Strong authentication (no anonymous access)

## Performance

**Benchmarks:**

- Throughput: 100K+ ops/sec
- Latency: <1ms P99
- Memory: Optimized for streaming workloads
- Failover: <5 second RTO (Recovery Time Objective)

## Monitoring

**Key Metrics Tracked:**

- Cluster health and topology
- Memory usage and eviction rates
- Replication lag
- Command latency (P50, P95, P99)
- Sentinel failover events

## Documentation

### Architecture & Design

- **[AWS Production Architecture](docs/AWS_ARCHITECTURE.md)** - Complete infrastructure design with diagrams, security layers, and HIPAA compliance mapping

### Operational Runbooks

- **[AWS Production Deployment](docs/runbooks/aws-production-deployment.md)** - Complete AWS infrastructure setup with TLS, ACLs, and EBS encryption
- **[Monitoring Setup](docs/runbooks/monitoring-setup.md)** - Prometheus, Grafana, and Redis Exporter configuration
- **[Replication Setup](docs/runbooks/replication-setup.md)** - Master-replica configuration with automatic failover
- **[Sentinel HA](docs/runbooks/sentinel-ha.md)** - High availability automation and failover testingth Sentinel quorum

### Technical Guides

- **[Epic Integration Patterns](docs/epic-integration-patterns.md)** - Redis data structures for MyChart, Interconnect, and CLOC use cases

## Scenarios Implemented

✅ MyChart session management with TTL  
✅ Epic Interconnect message queuing  
✅ CLOC real-time telemetry streaming  
✅ Sentinel automatic failover  
✅ Redis Cluster sharding and resharding  
✅ HIPAA security hardening  
✅ Monitoring and alerting infrastructure  

---

**Built to demonstrate Redis administration patterns for Epic EHR healthcare environments with emphasis on operational excellence and patient care continuity.**

```

---
