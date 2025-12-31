# Architecture Diagrams

Visual representations of the Epic Redis infrastructure.

---

## 1. AWS System Architecture

```mermaid
graph TB
    subgraph "AWS Cloud - us-east-1"
        subgraph "VPC 10.0.0.0/16"
            IGW[Internet Gateway]
            
            subgraph "Availability Zone 1a"
                SN1[Subnet 10.0.1.0/24]
                NODE1[Redis Node 1a<br/>10.0.1.102<br/>Slots 0-5460]
                EBS1[(EBS 8GB<br/>AES-256)]
                NODE1 -.-> EBS1
            end
            
            subgraph "Availability Zone 1b"
                SN2[Subnet 10.0.2.0/24]
                NODE2[Redis Node 1b<br/>10.0.2.27<br/>Slots 5461-10922]
                EBS2[(EBS 8GB<br/>AES-256)]
                NODE2 -.-> EBS2
            end
            
            subgraph "Availability Zone 1c"
                SN3[Subnet 10.0.3.0/24]
                NODE3[Redis Node 1c<br/>10.0.3.205<br/>Slots 10923-16383]
                EBS3[(EBS 8GB<br/>AES-256)]
                NODE3 -.-> EBS3
                PROM[Prometheus+Grafana<br/>18.227.49.163]
            end
            
            SG{Security Group<br/>redis-cluster-sg}
            
            IGW --> SG
            SG --> NODE1
            SG --> NODE2
            SG --> NODE3
            SG --> PROM
        end
    end
    
    ADMIN[Admin SSH<br/>Port 22] -->|TLS| IGW
    APP[Application<br/>Redis Client] -->|Port 6379 TLS| IGW
    
    NODE1 <-->|Cluster Bus<br/>Port 16379| NODE2
    NODE2 <-->|Cluster Bus<br/>Port 16379| NODE3
    NODE3 <-->|Cluster Bus<br/>Port 16379| NODE1
    
    PROM -.->|Scrape 9121| NODE1
    PROM -.->|Scrape 9121| NODE2
    PROM -.->|Scrape 9121| NODE3
    
    style NODE1 fill:#e74c3c,stroke:#c0392b,color:#fff
    style NODE2 fill:#e74c3c,stroke:#c0392b,color:#fff
    style NODE3 fill:#e74c3c,stroke:#c0392b,color:#fff
    style PROM fill:#f39c12,stroke:#d68910,color:#fff
    style EBS1 fill:#3498db,stroke:#2980b9,color:#fff
    style EBS2 fill:#3498db,stroke:#2980b9,color:#fff
    style EBS3 fill:#3498db,stroke:#2980b9,color:#fff
    style SG fill:#2ecc71,stroke:#27ae60,color:#fff
```

**Architecture Highlights:**

- **Multi-AZ Deployment:** 3 availability zones for fault tolerance
- **Cluster Mode:** 16,384 hash slots distributed across 3 masters
- **Encrypted Storage:** AES-256 EBS volumes for data-at-rest protection
- **Monitoring:** Dedicated Prometheus/Grafana instance for observability

---

## 2. Security Layers (Defense-in-Depth)

```mermaid
graph LR
    subgraph "Layer 1: Network Security"
        VPC[VPC Isolation<br/>10.0.0.0/16]
        SG[Security Groups<br/>Whitelist Only]
        SSH[SSH Restricted<br/>Admin IP Only]
    end
    
    subgraph "Layer 2: Transport Encryption"
        TLS[TLS 1.3<br/>All Connections]
        CERT[RSA 4096<br/>CA Signed]
        CLUST[Cluster Bus<br/>Encrypted]
    end
    
    subgraph "Layer 3: Access Control"
        ACL[ACL Authentication<br/>Required]
        ADMIN[admin_user<br/>Full Access]
        APP[app_user<br/>session:* only]
        DEV[dev_user<br/>Read Only]
        DEFAULT[default<br/>Disabled]
    end
    
    subgraph "Layer 4: Data-at-Rest"
        EBS[EBS Encryption<br/>AES-256]
        KMS[AWS KMS<br/>Key Management]
        AOF[AOF Files<br/>Encrypted]
    end
    
    VPC --> TLS
    SG --> TLS
    SSH --> TLS
    
    TLS --> ACL
    CERT --> ACL
    CLUST --> ACL
    
    ACL --> ADMIN
    ACL --> APP
    ACL --> DEV
    ACL --> DEFAULT
    
    ADMIN --> EBS
    APP --> EBS
    DEV --> EBS
    
    EBS --> KMS
    EBS --> AOF
    
    style VPC fill:#3498db,stroke:#2980b9,color:#fff
    style TLS fill:#9b59b6,stroke:#8e44ad,color:#fff
    style ACL fill:#e67e22,stroke:#d35400,color:#fff
    style EBS fill:#1abc9c,stroke:#16a085,color:#fff
    style DEFAULT fill:#95a5a6,stroke:#7f8c8d,color:#fff
```

**HIPAA Compliance:**

- ✅ Encryption in Transit (TLS 1.3)
- ✅ Encryption at Rest (AES-256)
- ✅ Access Controls (Role-based ACLs)
- ✅ Network Isolation (VPC + Security Groups)

---

## 3. Monitoring Stack Architecture

```mermaid
graph TB
    subgraph "Redis Cluster Nodes"
        R1[Redis Node 1a<br/>10.0.1.102:6379]
        R2[Redis Node 1b<br/>10.0.2.27:6379]
        R3[Redis Node 1c<br/>10.0.3.205:6379]
        
        E1[Redis Exporter<br/>:9121]
        E2[Redis Exporter<br/>:9121]
        E3[Redis Exporter<br/>:9121]
        
        R1 --> E1
        R2 --> E2
        R3 --> E3
    end
    
    subgraph "Monitoring Instance"
        PROM[Prometheus<br/>:9090]
        GRAF[Grafana<br/>:3000]
        
        PROM --> GRAF
    end
    
    E1 -.->|HTTP Scrape<br/>every 30s| PROM
    E2 -.->|HTTP Scrape<br/>every 30s| PROM
    E3 -.->|HTTP Scrape<br/>every 30s| PROM
    
    PROM -->|Data Source| GRAF
    
    USER[User Browser] -->|View Dashboards| GRAF
    
    subgraph "Metrics Collected"
        M1[redis_up<br/>Instance Health]
        M2[redis_memory_used_bytes<br/>Memory Usage]
        M3[redis_commands_total<br/>Throughput]
        M4[redis_keyspace_hits<br/>Cache Efficiency]
        M5[redis_cluster_state<br/>Cluster Health]
    end
    
    PROM -.-> M1
    PROM -.-> M2
    PROM -.-> M3
    PROM -.-> M4
    PROM -.-> M5
    
    style R1 fill:#e74c3c,stroke:#c0392b,color:#fff
    style R2 fill:#e74c3c,stroke:#c0392b,color:#fff
    style R3 fill:#e74c3c,stroke:#c0392b,color:#fff
    style E1 fill:#3498db,stroke:#2980b9,color:#fff
    style E2 fill:#3498db,stroke:#2980b9,color:#fff
    style E3 fill:#3498db,stroke:#2980b9,color:#fff
    style PROM fill:#e67e22,stroke:#d35400,color:#fff
    style GRAF fill:#f39c12,stroke:#d68910,color:#fff
```

**Monitoring Capabilities:**

- **Real-time Metrics:** 30-second scrape interval
- **Grafana Dashboards:** Pre-built Redis dashboard (ID: 11835)
- **Alert-Ready:** Prometheus rules for critical thresholds
- **Production-Grade:** Enterprise observability stack

---

## 4. Data Flow & Request Routing

```mermaid
sequenceDiagram
    participant Client
    participant DNS
    participant SecurityGroup
    participant RedisCluster
    participant Node1a
    participant Node1b
    participant Node1c
    participant EBS

    Client->>DNS: redis.example.com
    DNS-->>Client: 10.0.1.102, 10.0.2.27, 10.0.3.205
    
    Client->>SecurityGroup: SET session:user123 "data"<br/>(Port 6379, TLS)
    SecurityGroup->>RedisCluster: Firewall Check (Allow VPC)
    
    RedisCluster->>RedisCluster: Calculate: CRC16("session:user123") % 16384 = 4521
    RedisCluster->>RedisCluster: Route to: Slots 0-5460 (Node 1a)
    
    RedisCluster->>Node1a: Forward Write (TLS 1.3)
    Node1a->>Node1a: ACL Check: app_user allowed session:*
    Node1a->>EBS: Write to encrypted volume (AES-256)
    EBS-->>Node1a: Persisted (AOF)
    
    Node1a->>Node1b: Cluster Bus: Topology update (Port 16379)
    Node1a->>Node1c: Cluster Bus: Topology update (Port 16379)
    
    Node1a-->>Client: OK (via TLS)
    
    Note over Client,EBS: Request Time: <1ms P99
    Note over Node1a,EBS: Encryption at Rest: AES-256
    Note over Client,Node1a: Encryption in Transit: TLS 1.3
```

**Request Flow Details:**

1. **Client Discovery:** DNS/cluster-aware client knows all node IPs
2. **Slot Calculation:** `CRC16(key) % 16384` determines routing
3. **TLS Encryption:** All client-server communication encrypted
4. **ACL Verification:** User permissions checked on target node
5. **Persistence:** Write to encrypted EBS volume (AOF)
6. **Cluster Sync:** Topology changes propagated via cluster bus

---

## 5. Epic EHR Integration Pattern

```mermaid
graph TB
    subgraph "Epic Hyperspace / MyChart"
        USER1[Clinician Login]
        USER2[Patient Portal]
    end
    
    subgraph "Application Layer"
        APP[Epic Integration Service<br/>Session Manager]
    end
    
    subgraph "Redis Cluster"
        direction LR
        NODE1[Node 1a<br/>Sessions 0-5460]
        NODE2[Node 1b<br/>Sessions 5461-10922]
        NODE3[Node 1c<br/>Sessions 10923-16383]
    end
    
    subgraph "Session Data Structure"
        HASH[HSET session:abc123<br/>user_id: patient_456<br/>portal: MyChart<br/>last_activity: 2025-12-30T22:00:00Z<br/>TTL: 1800s]
    end
    
    USER1 -->|Login| APP
    USER2 -->|Login| APP
    
    APP -->|Write Session<br/>TLS + ACL| NODE1
    APP -->|Write Session<br/>TLS + ACL| NODE2
    APP -->|Write Session<br/>TLS + ACL| NODE3
    
    NODE1 -.-> HASH
    NODE2 -.-> HASH
    NODE3 -.-> HASH
    
    APP -->|Read Session<br/>Every Page Load| NODE1
    APP -->|Read Session<br/>Every Page Load| NODE2
    APP -->|Read Session<br/>Every Page Load| NODE3
    
    HASH -->|Auto-Expire<br/>After 30 min| NODE1
    
    style USER1 fill:#3498db,stroke:#2980b9,color:#fff
    style USER2 fill:#3498db,stroke:#2980b9,color:#fff
    style APP fill:#9b59b6,stroke:#8e44ad,color:#fff
    style NODE1 fill:#e74c3c,stroke:#c0392b,color:#fff
    style NODE2 fill:#e74c3c,stroke:#c0392b,color:#fff
    style NODE3 fill:#e74c3c,stroke:#c0392b,color:#fff
    style HASH fill:#2ecc71,stroke:#27ae60,color:#fff
```

**Epic Use Case Benefits:**

- **Performance:** Sub-1ms session lookup vs 50-100ms database
- **Compliance:** Automatic 30-minute session expiration (HIPAA)
- **Scalability:** Handles 10,000+ concurrent clinician sessions
- **High Availability:** Multi-AZ deployment ensures 24/7 uptime

---

## Diagram Usage

These diagrams are created using Mermaid and render automatically on GitHub. They can also be:

- **Exported as PNG** - For presentations or documentation
- **Embedded in Slides** - Copy diagram code into Mermaid Live Editor
- **Updated Easily** - Text-based diagrams are version controlled

**Editing:** Modify the diagram code directly in this markdown file. GitHub will automatically render the updated diagrams.

**Tools:**

- [Mermaid Live Editor](https://mermaid.live/) - Test and export diagrams
- [GitHub Markdown](https://github.com) - Native Mermaid rendering
- VS Code: Mermaid Preview extension for local editing

---

**Last Updated:** 2025-12-30  
**Maintainer:** Infrastructure Team  
**Diagram Count:** 5 architectural views
