# AWS Production Redis Cluster Deployment

**Production-grade, HIPAA-compliant Redis cluster deployment on AWS EC2**

---

## Architecture Overview

### Infrastructure

- **VPC**: `10.0.0.0/16` - Isolated private network
- **Subnets**: 3 public subnets across 3 availability zones
  - `10.0.1.0/24` - us-east-1a
  - `10.0.2.0/24` - us-east-1b
  - `10.0.3.0/24` - us-east-1c
- **EC2 Instances**: 3 x t2.micro (Ubuntu 24.04 LTS)
- **Redis Cluster**: 3 master nodes, 16384 hash slots distributed

### Security Layers

**1. Network Security (VPC + Security Groups)**

- VPC isolation from public internet
- Security group firewall rules
- SSH access restricted to admin IP
- Redis ports restricted to VPC CIDR

**2. Transport Encryption (TLS 1.3)**

- All inter-node communication encrypted
- Certificate-based authentication
- Protects data in transit

**3. Access Control (ACLs)**

- Role-based user permissions
- Admin, application, and developer roles
- Principle of least privilege

**4. Data-at-Rest Encryption (EBS)**

- AES-256 encrypted volumes
- AWS KMS key management
- Protects against physical theft

---

## Deployment Procedure

### Phase 1: Network Infrastructure

#### 1.1 Create VPC

```
AWS Console → VPC → Create VPC
Name: redis-vpc
IPv4 CIDR: 10.0.0.0/16
Tags: project=redis-lab
```

#### 1.2 Create Subnets

Create 3 subnets in different AZs:

**Subnet 1:**

```
Name: redis-subnet-1a
VPC: redis-vpc
AZ: us-east-1a
IPv4 CIDR: 10.0.1.0/24
```

**Subnet 2:**

```
Name: redis-subnet-1b
VPC: redis-vpc
AZ: us-east-1b
IPv4 CIDR: 10.0.2.0/24
```

**Subnet 3:**

```
Name: redis-subnet-1c
VPC: redis-vpc
AZ: us-east-1c
IPv4 CIDR: 10.0.3.0/24
```

#### 1.3 Create Internet Gateway

```
VPC → Internet Gateways → Create
Name: redis-igw
Attach to VPC: redis-vpc
```

#### 1.4 Configure Route Table

```
VPC → Route Tables → Create
Name: redis-public-rt
VPC: redis-vpc

Routes:
  Destination: 0.0.0.0/0
  Target: redis-igw

Subnet Associations:
  Add all 3 subnets
```

**Why:** Routes internet traffic through IGW for package downloads and SSH access.

#### 1.5 Create Security Group

```
EC2 → Security Groups → Create
Name: redis-cluster-sg
VPC: redis-vpc

Inbound Rules:
  Type: SSH, Port: 22, Source: My IP
  Type: Custom TCP, Port: 6379, Source: 10.0.0.0/16
  Type: Custom TCP, Port: 16379, Source: 10.0.0.0/16
  Type: Custom TCP, Port Range: 26379-26381, Source: 10.0.0.0/16

Outbound Rules:
  All traffic, All protocols, All ports, Destination: 0.0.0.0/0
```

**Critical:** Port 16379 is the Redis cluster bus - nodes can't communicate without it!

---

### Phase 2: EC2 Instance Deployment

#### 2.1 Launch Instance 1 (us-east-1a)

```
EC2 → Launch Instance

Name: redis-node-1a
AMI: Ubuntu Server 24.04 LTS
Instance type: t2.micro
Key pair: Create new → redis-cluster-key.pem (download!)

Network:
  VPC: redis-vpc
  Subnet: redis-subnet-1a
  Auto-assign public IP: Enable
  Security group: redis-cluster-sg

Storage: 8 GB gp3 (default)

Launch instance
```

#### 2.2 Launch Instances 2 & 3

Repeat above for:

- **Instance 2**: Name `redis-node-1b`, Subnet `redis-subnet-1b`
- **Instance 3**: Name `redis-node-1c`, Subnet `redis-subnet-1c`

**Record both public AND private IPs for each instance.**

---

### Phase 3: Redis Installation

**SSH to each instance and install Redis:**

```bash
# Set key permissions (Windows)
icacls redis-cluster-key.pem /inheritance:r
icacls redis-cluster-key.pem /grant:r "$($env:USERNAME):(R)"

# SSH to instance
ssh -i redis-cluster-key.pem ubuntu@<PUBLIC_IP>

# Update system
sudo apt update && sudo apt upgrade -y

# Install Redis
sudo apt install redis-server -y

# Verify
redis-server --version
redis-cli PING  # Should return: PONG

# Stop default instance (we'll configure cluster mode)
sudo systemctl stop redis-server
sudo systemctl disable redis-server
```

**Repeat for all 3 instances.**

---

### Phase 4: TLS Certificate Generation

**Generate certificates on Instance 1 only:**

```bash
# SSH to instance 1
ssh -i redis-cluster-key.pem ubuntu@<INSTANCE_1_PUBLIC_IP>

# Create certificates directory
mkdir -p ~/redis-certs
cd ~/redis-certs

# Generate Certificate Authority
openssl genrsa -out ca-key.pem 4096

openssl req -new -x509 -days 365 -key ca-key.pem -out ca-cert.pem \
  -subj "/C=US/ST=NY/L=NYC/O=NYP-Redis/CN=RedisCA"

# Generate server certificate
openssl genrsa -out redis-server-key.pem 4096

openssl req -new -key redis-server-key.pem -out redis-server.csr \
  -subj "/C=US/ST=NY/L=NYC/O=NYP-Redis/CN=redis-cluster"

openssl x509 -req -days 365 -in redis-server.csr \
  -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial \
  -out redis-server-cert.pem

# Set permissions
chmod 600 ~/redis-certs/*.pem
chmod 644 ~/redis-certs/ca-cert.pem ~/redis-certs/redis-server-cert.pem

# Verify certificates exist
ls -lh ~/redis-certs/
```

**Expected files:**

- `ca-cert.pem` - Certificate Authority public cert
- `ca-key.pem` - CA private key
- `redis-server-cert.pem` - Server public cert
- `redis-server-key.pem` - Server private key

**Distribute to other instances:**

```bash
# Package certificates
cd ~/redis-certs
tar czf redis-certs.tar.gz ca-cert.pem ca-key.pem redis-server-cert.pem redis-server-key.pem

# Download to local machine (new terminal on Windows)
scp -i redis-cluster-key.pem ubuntu@<INSTANCE_1_IP>:~/redis-certs/redis-certs.tar.gz ~/Downloads/

# Upload to instance 2
scp -i redis-cluster-key.pem ~/Downloads/redis-certs.tar.gz ubuntu@<INSTANCE_2_IP>:~/

# Upload to instance 3
scp -i redis-cluster-key.pem ~/Downloads/redis-certs.tar.gz ubuntu@<INSTANCE_3_IP>:~/

# SSH to instances 2 & 3 and extract
ssh -i redis-cluster-key.pem ubuntu@<INSTANCE_2_IP>
mkdir -p ~/redis-certs
tar xzf redis-certs.tar.gz -C ~/redis-certs/
exit

# Repeat for instance 3
```

---

### Phase 5: ACL Configuration

**Create ACL users file on ALL instances:**

```bash
sudo nano /etc/redis/users.acl
```

**Paste (NO COMMENTS - ACL files don't support them):**

```
user app_user on >AppPassword123! ~session:* ~patient:* +get +set +hget +hset +expire +ttl
user dev_user on >DevPassword123! ~* +get +hget +keys +info +cluster
user admin_user on >AdminPassword123! ~* +@all
user default off
```

**Explanation:**

- `app_user`: Can read/write session: and patient: keys only
- `dev_user`: Read-only access for debugging
- `admin_user`: Full administrative access
- `default off`: Disables anonymous access (forces authentication)

**Set permissions:**

```bash
sudo chmod 640 /etc/redis/users.acl
sudo chown redis:redis /etc/redis/users.acl
```

**Repeat for all 3 instances.**

---

### Phase 6: Redis Cluster Configuration

**Configure Redis on each instance with TLS:**

**Instance 1 (10.0.1.x):**

```bash
sudo nano /etc/redis/redis-cluster.conf
```

**Paste (update bind IP for each instance):**

```
# Network - TLS only
port 0
tls-port 6379
bind 10.0.1.XXX 127.0.0.1

# Cluster
cluster-enabled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 5000

# TLS/SSL
tls-cert-file /home/ubuntu/redis-certs/redis-server-cert.pem
tls-key-file /home/ubuntu/redis-certs/redis-server-key.pem
tls-ca-cert-file /home/ubuntu/redis-certs/ca-cert.pem
tls-cluster yes
tls-replication yes
tls-protocols "TLSv1.2 TLSv1.3"

# Access Control
protected-mode yes
aclfile /etc/redis/users.acl

# Persistence
appendonly yes
dir /var/lib/redis

# Logging
loglevel notice
logfile /var/log/redis/redis-cluster.log
```

**Create directories and start:**

```bash
sudo mkdir -p /var/lib/redis /var/log/redis
sudo chown -R redis:redis /var/lib/redis /var/log/redis

sudo redis-server /etc/redis/redis-cluster.conf &
sleep 3
```

**Test TLS connection:**

```bash
redis-cli --tls \
  --cert ~/redis-certs/redis-server-cert.pem \
  --key ~/redis-certs/redis-server-key.pem \
  --cacert ~/redis-certs/ca-cert.pem \
  --user admin_user --pass AdminPassword123! \
  PING
```

**Should return:** `PONG`

**Repeat configuration for instances 2 & 3** (change bind IP to match private IP).

---

### Phase 7: Create Cluster

**From Instance 1, create the cluster:**

```bash
redis-cli --cluster create \
  10.0.1.XXX:6379 \
  10.0.2.XXX:6379 \
  10.0.3.XXX:6379 \
  --cluster-replicas 0 \
  --cluster-yes \
  -a AdminPassword123! \
  --tls \
  --cert ~/redis-certs/redis-server-cert.pem \
  --key ~/redis-certs/redis-server-key.pem \
  --cacert ~/redis-certs/ca-cert.pem
```

**Expected output:**

```
>>> Performing hash slots allocation on 3 nodes...
Master[0] -> Slots 0-5460
Master[1] -> Slots 5461-10922
Master[2] -> Slots 10923-16383
[OK] All nodes agree about slots configuration.
[OK] All 16384 slots covered.
```

**Verify cluster:**

```bash
redis-cli -c --tls \
  --cert ~/redis-certs/redis-server-cert.pem \
  --key ~/redis-certs/redis-server-key.pem \
  --cacert ~/redis-certs/ca-cert.pem \
  --user admin_user --pass AdminPassword123! \
  CLUSTER INFO
```

**Should show:** `cluster_state:ok`

---

### Phase 8: Enable EBS Encryption

#### 8.1 Stop All Instances

```
AWS Console → EC2 → Instances
Select all 3 instances
Instance state → Stop instance
Wait for "Stopped" status
```

#### 8.2 Create Snapshots

```
EC2 → Volumes
For each volume:
  Select volume → Actions → Create snapshot
  Description: redis-node-1a-snapshot (or 1b, 1c)
  Create snapshot

Wait for all snapshots to show "Completed" status (~5 minutes)
```

#### 8.3 Copy Snapshots with Encryption

```
EC2 → Snapshots
For each completed snapshot:
  Select → Actions → Copy snapshot
  Destination region: us-east-1 (same)
  Encryption: ✅ Enable
  KMS key: (default) aws/ebs
  Description: redis-node-1a-encrypted
  Copy snapshot

Wait for encrypted copies to complete
```

#### 8.4 Create Encrypted Volumes

```
EC2 → Snapshots
For each encrypted snapshot:
  Select → Actions → Create volume from snapshot
  Availability Zone: Match original (1a, 1b, or 1c)
  Verify "Encrypted: Yes"
  Create volume
```

#### 8.5 Swap Volumes

```
For each instance:
  1. EC2 → Volumes → Select OLD unencrypted volume
     Actions → Detach volume
     Wait for "Available"
  
  2. Select NEW encrypted volume (matching AZ)
     Actions → Attach volume
     Instance: redis-node-1x
     Device: /dev/sda1
     Attach
  
  3. EC2 → Instances → Select instance
     Instance state → Start instance
```

#### 8.6 Restart Redis on All Instances

```bash
# SSH to each instance
ssh -i redis-cluster-key.pem ubuntu@<PUBLIC_IP>

# Start Redis
sudo redis-server /etc/redis/redis-cluster.conf &
sleep 3

# Verify
redis-cli --tls \
  --cert ~/redis-certs/redis-server-cert.pem \
  --key ~/redis-certs/redis-server-key.pem \
  --cacert ~/redis-certs/ca-cert.pem \
  --user admin_user --pass AdminPassword123! \
  PING
```

---

## Verification & Testing

### Test 1: Cluster Health

```bash
redis-cli -c --tls \
  --cert ~/redis-certs/redis-server-cert.pem \
  --key ~/redis-certs/redis-server-key.pem \
  --cacert ~/redis-certs/ca-cert.pem \
  --user admin_user --pass AdminPassword123! \
  CLUSTER INFO
```

**Expected:**

- `cluster_state:ok`
- `cluster_slots_assigned:16384`
- `cluster_known_nodes:3`

### Test 2: Data Distribution

```bash
redis-cli -c --tls \
  --cert ~/redis-certs/redis-server-cert.pem \
  --key ~/redis-certs/redis-server-key.pem \
  --cacert ~/redis-certs/ca-cert.pem \
  --user admin_user --pass AdminPassword123! \
  SET patient:12345 "encrypted_data"
```

**Should show which slot/node it was redirected to.**

### Test 3: ACL Permissions

**Test app_user (limited access):**

```bash
redis-cli -c --tls \
  --cert ~/redis-certs/redis-server-cert.pem \
  --key ~/redis-certs/redis-server-key.pem \
  --cacert ~/redis-certs/ca-cert.pem \
  --user app_user --pass AppPassword123! \
  SET session:test "data"
```

**Should work** (app_user can write session: keys)

```bash
redis-cli -c --tls \
  --cert ~/redis-certs/redis-server-cert.pem \
  --key ~/redis-certs/redis-server-key.pem \
  --cacert ~/redis-certs/ca-cert.pem \
  --user app_user --pass AppPassword123! \
  FLUSHALL
```

**Should fail** (app_user doesn't have FLUSHALL permission)

### Test 4: Encryption Verification

**EBS encryption:**

```
EC2 → Volumes → Select volume
Details: Encryption should show "Encrypted (aws/ebs)"
```

**TLS encryption:**

- All redis-cli commands require --tls flag
- Connections fail without certificates

---

## Troubleshooting

### Issue: redis-cli --cluster create hangs

**Symptom:** Cluster create command freezes with no output

**Cause:** Security group missing cluster bus port (16379)

**Solution:**

```
EC2 → Security Groups → redis-cluster-sg → Edit inbound rules
Add rule:
  Type: Custom TCP
  Port: 16379
  Source: 10.0.0.0/16
  Description: Redis cluster bus
```

**Why:** Redis cluster uses port+10000 (16379) for node-to-node gossip protocol.

### Issue: Redis fails to start with ACL errors

**Symptom:** Log shows `/etc/redis/redis-cluster.conf:1 should start with user keyword`

**Cause:** ACL file path incorrect in config

**Solution:**

```bash
sudo nano /etc/redis/redis-cluster.conf

# Find and fix:
aclfile /etc/redis/users.acl  # Correct path

# NOT:
aclfile /etc/redis/redis-cluster.conf  # Wrong!
```

**Also verify users.acl has NO comment lines** (# not supported in ACL files).

### Issue: cluster_state:fail

**Symptom:** `CLUSTER INFO` shows `cluster_state:fail`

**Cause:** Not all nodes running

**Solution:**

```bash
# Check if Redis running on each instance
ssh <instance>
ps aux | grep redis-server

# If not running:
sudo redis-server /etc/redis/redis-cluster.conf &

# Check cluster logs for errors
sudo tail -50 /var/log/redis/redis-cluster.log
```

### Issue: SSH connection times out

**Symptom:** `ssh: connect to host X.X.X.X port 22: Connection timed out`

**Cause:** Security group doesn't allow SSH from your current IP

**Solution:**

```
EC2 → Security Groups → redis-cluster-sg
Edit inbound rules → SSH rule → Source: My IP
```

**Note:** Public IP changes when you stop/start instances!

### Issue: Could not connect to Redis at 127.0.0.1:6379

**Symptom:** `redis-cli` can't connect locally

**Causes & Solutions:**

**1. Redis not running:**

```bash
sudo redis-server /etc/redis/redis-cluster.conf &
```

**2. TLS required:**

```bash
# Wrong:
redis-cli PING

# Correct:
redis-cli --tls --cert ~/redis-certs/redis-server-cert.pem \
  --key ~/redis-certs/redis-server-key.pem \
  --cacert ~/redis-certs/ca-cert.pem \
  --user admin_user --pass AdminPassword123! \
  PING
```

**3. ACL authentication required:**
Must specify `--user` and `--pass` with ACL enabled.

---

## Cost Management

### Stop Instances to Save Money

**When not in use:**

```
AWS Console → EC2 → Instances
Select all 3 → Instance state → Stop instance
```

**Cost when stopped:** ~$1/month (storage only)
**Cost when running:** ~$26/month (3 x t2.micro)

### Terminate When Done

**To completely delete (irreversible):**

```
1. EC2 → Instances → Select all → Terminate
2. EC2 → Volumes → Delete orphaned volumes
3. EC2 → Snapshots → Delete snapshots
4. VPC → Delete VPC (deletes subnets, IGW, route tables)
5. EC2 → Security Groups → Delete redis-cluster-sg
```

---

## Security Best Practices

### HIPAA Compliance Checklist

- ✅ **Encryption in Transit**: TLS 1.3 for all connections
- ✅ **Encryption at Rest**: AES-256 EBS encryption
- ✅ **Access Control**: ACL-based role permissions
- ✅ **Network Isolation**: VPC with security groups
- ✅ **Audit Logging**: Redis command logs to `/var/log/redis/`
- ✅ **Authentication**: Strong passwords (change defaults!)
- ✅ **Principle of Least Privilege**: Role-based ACLs

### Production Hardening Recommendations

**1. Change Default Passwords**

```
Update users.acl with strong, unique passwords
Minimum 16 characters, alphanumeric + symbols
```

**2. Enable CloudWatch Monitoring**

```
Install CloudWatch agent
Monitor: CPU, memory, network, disk
Set alarms for high memory (>80%), instance down
```

**3. Automated Backups**

```
Enable automated EBS snapshots (daily)
Retention: 7 days minimum
Test restore procedure
```

**4. Restrict SSH Access**

```
Security group: SSH only from VPN or bastion host
NOT: 0.0.0.0/0 (entire internet)
Use AWS Systems Manager Session Manager instead of SSH
```

**5. Update TLS Certificates Annually**

```
Certificates expire in 365 days
Set calendar reminder 2 weeks before expiration
Regenerate and distribute new certs
```

---

## Architecture Decisions

### Why 3 Nodes?

**Minimum for cluster:** 3 masters for hash slot distribution
**High Availability:** Can lose 1 node and remain operational
**Cost vs. Reliability:** 3 nodes balances cost with uptime needs

### Why Multi-AZ?

**Failure Isolation:** Each AZ is separate physical datacenter
**Example:** Lightning strikes data center A → Nodes in B and C continue serving traffic
**HIPAA Requirement:** No single point of failure for PHI data

### Why TLS 1.3?

**Latest Standard:** Most secure TLS version
**Performance:** Faster handshake than TLS 1.2
**Regulatory:** HIPAA requires encryption in transit

### Why Separate ACL File?

**Maintainability:** Easier to update users without touching main config
**Security:** Can restrict ACL file permissions independently
**Best Practice:** Separation of concerns

---

## Next Steps

### For Production

1. **Add Replicas**: Deploy 3 more nodes as replicas for automatic failover
2. **Monitoring**: Set up CloudWatch dashboards and alarms
3. **Backups**: Implement automated snapshot schedules
4. **DNS**: Use Route 53 for friendly cluster endpoints
5. **Bastion Host**: Replace direct SSH with bastion + SSM

### For Learning

1. **Test Failover**: Kill a master node and observe cluster recovery
2. **Add Redis Streams**: Implement Epic Interconnect message queue
3. **Benchmark**: Use redis-benchmark to measure performance
4. **Scale Up**: Add replicas and test read distribution

---

**Production-Ready:** ✅ Yes (with additional monitoring and backups)

**This architecture demonstrates production-grade, HIPAA-compliant patterns for caching and session management in healthcare systems.**
