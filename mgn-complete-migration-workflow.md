# AWS Application Migration Service (MGN) — Complete End-to-End Migration Workflow

## From On-Premises Discovery to Final Cutover on AWS

---

## Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                           AWS MGN MIGRATION WORKFLOW                                          │
└─────────────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌─────────────┐
│   PHASE 1    │    │   PHASE 2    │    │   PHASE 3    │    │   PHASE 4    │    │   PHASE 5   │
│  DISCOVERY   │───▶│   SETUP &    │───▶│ REPLICATION  │───▶│   TESTING    │───▶│  CUTOVER    │
│              │    │  PLANNING    │    │              │    │              │    │             │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘    └──────┬───────┘    └──────┬──────┘
       │                   │                   │                   │                   │
       ▼                   ▼                   ▼                   ▼                   ▼
  ┌─────────┐        ┌─────────┐        ┌─────────┐        ┌─────────┐        ┌─────────────┐
  │• vCenter │        │• MGN Init│        │• Agent  │        │• Launch │        │• Final      │
  │  Import  │        │• IAM     │        │  Install│        │  Test   │        │  Snapshot   │
  │• Discover│        │• Network │        │• Initial│        │• Validate│       │• Conversion │
  │  Servers │        │• Template│        │  Sync   │        │• Verify │        │• Launch     │
  │• Inventory│       │• Wave    │        │• Contin-│        │  Apps   │        │• DNS Switch │
  │  Mapping │        │  Planning│        │  uous   │        │• Mark   │        │• Finalize   │
  └─────────┘        └─────────┘        └─────────┘        │  Ready  │        └─────────────┘
                                                            └─────────┘
```

---

## Detailed Architecture

```
┌─────────────────── ON-PREMISES ───────────────────┐     ┌──────────────────── AWS CLOUD ────────────────────────┐
│                                                    │     │                                                        │
│  ┌────────────────┐     ┌──────────────────────┐  │     │  ┌─────────────────┐     ┌──────────────────────────┐ │
│  │  Source Server  │     │   MGN Connector      │  │     │  │  MGN Service    │     │  Staging Area            │ │
│  │  (he-dev-app-01)│     │   (vCenter VM)       │  │     │  │  (Control Plane)│     │                          │ │
│  │                 │     │                      │  │     │  │                 │     │  ┌──────────────────┐   │ │
│  │  ┌───────────┐ │     │  ┌────────────────┐  │  │     │  │  ┌───────────┐ │     │  │Replication Server│   │ │
│  │  │Replication│ │     │  │ Discovers VMs  │  │  │     │  │  │ Manages   │ │     │  │   (t3.small)     │   │ │
│  │  │  Agent    │─┼─────┼──┼────────────────┼──┼──┼─────┼──┼──│Replication│ │     │  │                  │   │ │
│  │  │           │ │TCP  │  │ via vCenter API│  │  │HTTPS│  │  │           │ │     │  │  ┌────────────┐  │   │ │
│  │  └───────────┘ │1500 │  └────────────────┘  │  │443  │  │  └───────────┘ │     │  │  │Staging EBS │  │   │ │
│  │                 │     │                      │  │     │  │                 │     │  │  │   Disks    │  │   │ │
│  └────────────────┘     └──────────────────────┘  │     │  └─────────────────┘     │  │  └────────────┘  │   │ │
│                                                    │     │                           │  └──────────────────┘   │ │
│                                                    │     │                           └──────────────────────────┘ │
│                                                    │     │                                                        │
│                                                    │     │  ┌─────────────────────────────────────────────────┐   │
│                                                    │     │  │  Target Area                                     │   │
│                                                    │     │  │                                                   │   │
│                                                    │     │  │  ┌──────────────┐  ┌──────────────────────────┐  │   │
│                                                    │     │  │  │ Conversion   │  │  Target Instance         │  │   │
│                                                    │     │  │  │ Server (m5)  │─▶│  (t3.micro)              │  │   │
│                                                    │     │  │  │ (temporary)  │  │  he-dev-app-01 (final)   │  │   │
│                                                    │     │  │  └──────────────┘  └──────────────────────────┘  │   │
│                                                    │     │  └─────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────┘     └────────────────────────────────────────────────────────┘
```

---

## PHASE 1: Discovery & Assessment

### 1.1 Deploy MGN Connector in vCenter

```
vCenter Environment
    └── Deploy MGN Connector OVA
        ├── Configure network (HTTPS outbound to AWS)
        ├── Register with AWS MGN service
        └── Provide vCenter credentials
```

**What it does:**
- Connects to vCenter API
- Discovers all VMs (hostname, IP, OS, disk size, CPU, RAM)
- Imports VM metadata into MGN as source servers
- No agent installation required at this stage

**Challenges We Faced:**
- ❌ None — Connector successfully imported `he-dev-app-01` from vCenter
- Source server appeared in MGN with state `PENDING_INSTALLATION`

### 1.2 Inventory & Wave Planning

```bash
# View discovered servers
aws mgn describe-source-servers --region us-east-1

# Server details imported:
# - hostname: he-dev-app-01
# - IP: 192.168.12.57
# - OS: LINUX
# - vCenter: Datacenter
# - VM ID: vm-83
```

**Planning considerations:**
- Group servers into migration waves (by application, dependency, priority)
- Identify dependencies between servers
- Plan target VPC, subnets, security groups
- Define instance types and right-sizing strategy

---

## PHASE 2: AWS Environment Setup

### 2.1 AWS Control Tower & Landing Zone

```
AWS Organizations
├── Root
│   └── Management Account (114805761158)
├── Security OU (CT-managed)
│   ├── Audit Account (044789067646)
│   └── Log Archive Account (598361629577)
├── Workloads OU (CT-managed)
│   └── Workload Account (150897468843)
├── Sandbox OU (CT-managed)
└── Infrastructure OU (CT-managed)
```

**Challenges We Faced:**
- ❌ Landing Zone was FAILED at v3.3
- ❌ No OUs existed (all accounts at root)
- ❌ Orphaned S3 buckets blocking StackSet deployment
- ❌ Sandbox OU created outside CT — couldn't apply SCPs

**Resolution:**
1. Created OUs (Security, Sandbox, Workloads, Infrastructure)
2. Moved accounts into appropriate OUs
3. Deleted orphaned S3 buckets in Log Archive account
4. Updated Landing Zone to v4.0 with correct manifest
5. Recreated Sandbox OU and registered with Control Tower baseline

### 2.2 MGN Service Initialization

```bash
# Initialize MGN in the region
aws mgn initialize-service --region us-east-1
```

**What gets created:**
- Default replication configuration template
- IAM service-linked roles
- Default launch configuration template

### 2.3 Network Configuration

```
Target VPC (vpc-0af49170)
├── Subnet: subnet-040f98af126cd1889 (us-east-1b)
│   ├── Used for: Staging (replication servers)
│   └── Used for: Target instances
├── Security Group: sg-7302c328 (default)
└── Internet Gateway (for public IP replication)
```

**Challenges We Faced:**
- ❌ Replication config template pointed to non-existent subnet
- ❌ Launch template referenced deleted security group
- ❌ All stale references from a previously deleted VPC

### 2.4 Configure Replication Settings

```bash
# Replication Configuration (per source server)
aws mgn update-replication-configuration \
    --source-server-id s-38759b0083f7eed55 \
    --staging-area-subnet-id subnet-040f98af126cd1889 \
    --replication-server-instance-type t3.small \
    --data-plane-routing PUBLIC_IP \
    --create-public-ip true \
    --bandwidth-throttling 0 \
    --ebs-encryption DEFAULT
```

### 2.5 Configure Launch Template

```bash
# EC2 Launch Template
aws ec2 create-launch-template-version \
    --launch-template-id lt-073a4879da2602f77 \
    --launch-template-data '{
        "InstanceType": "t3.micro",
        "NetworkInterfaces": [{
            "SubnetId": "subnet-040f98af126cd1889",
            "Groups": ["sg-7302c328"],
            "AssociatePublicIpAddress": true,
            "DeviceIndex": 0
        }]
    }'

# Disable right-sizing (use exact instance type from template)
aws mgn update-launch-configuration \
    --source-server-id s-38759b0083f7eed55 \
    --target-instance-type-right-sizing-method NONE
```

---

## PHASE 3: Replication

### 3.1 Install Replication Agent on Source Server

```bash
# SSH/SSM into source server
aws ssm start-session --target i-0244d3778f7b31463

# Download and install agent
sudo wget -O ./aws-replication-installer-init \
    https://aws-application-migration-service-us-east-1.s3.us-east-1.amazonaws.com/latest/linux/aws-replication-installer-init
sudo chmod +x aws-replication-installer-init
sudo ./aws-replication-installer-init --region us-east-1
```

**What happens during installation:**
1. Downloads agent binaries (~200MB)
2. Installs kernel replication driver (`aws_replication_driver.ko`)
3. Registers with MGN service (creates new source server or connects to existing)
4. Starts initial disk scan

**Challenges We Faced:**
- ❌ Agent registered as NEW source server (`s-38759b0083f7eed55`) instead of the vCenter-imported one (`s-31dd971512e7520d0`)
- This is expected when installing on a fresh instance — the agent uses instance metadata for identity

### 3.2 Replication Initiation

```
Replication Steps (automated by MGN):
┌─────────────────────────────────────────────┐
│ 1. WAIT                          ✅ SUCCEEDED │
│ 2. CREATE_SECURITY_GROUP         ✅ SUCCEEDED │
│ 3. LAUNCH_REPLICATION_SERVER     ✅ SUCCEEDED │
│ 4. BOOT_REPLICATION_SERVER       ✅ SUCCEEDED │
│ 5. AUTHENTICATE_WITH_SERVICE     ✅ SUCCEEDED │
│ 6. DOWNLOAD_REPLICATION_SOFTWARE ✅ SUCCEEDED │
│ 7. CREATE_STAGING_DISKS          ✅ SUCCEEDED │
│ 8. ATTACH_STAGING_DISKS          ✅ SUCCEEDED │
│ 9. PAIR_REPLICATION_SERVER       ✅ SUCCEEDED │
│ 10. CONNECT_AGENT_TO_SERVER      ✅ SUCCEEDED │
│ 11. START_DATA_TRANSFER          ✅ SUCCEEDED │
└─────────────────────────────────────────────┘
```

**Challenges We Faced:**
- ❌ Step 2 (CREATE_SECURITY_GROUP) failed due to non-existent staging subnet
- Fixed by updating replication configuration to valid subnet

### 3.3 Initial Sync

```bash
# Monitor sync progress
aws mgn describe-source-servers \
    --filters '{"sourceServerIDs":["s-38759b0083f7eed55"]}' \
    --query 'items[0].dataReplicationInfo'

# Progress: 0 GB → 8.59 GB (100%)
# Duration: ~10-15 minutes for 8 GB disk
# State: INITIATING → INITIAL_SYNC → CONTINUOUS
```

**Data flow during initial sync:**
```
Source Disk ──(block-level read)──▶ Agent ──(TCP 1500)──▶ Replication Server ──▶ Staging EBS
   8 GB                                                                          8 GB copy
```

### 3.4 Continuous Replication

```bash
# Verify healthy continuous replication
# replication_state: CONTINUOUS
# lag: P0D (zero lag)
# backlog: 0 bytes
```

**Data flow during continuous replication:**
```
Source Disk ──(kernel driver captures writes)──▶ Agent ──(real-time)──▶ Staging EBS
                                                                        (always up-to-date)
```

---

## PHASE 4: Testing

### 4.1 Launch Test Instance

```bash
aws mgn start-test --source-server-ids s-38759b0083f7eed55
```

**What happens:**
```
┌─────────────────────────────────────────────────────────────────┐
│ Test Launch Pipeline                                             │
│                                                                  │
│ 1. SNAPSHOT_START ──▶ Creates EBS snapshot from staging disks   │
│ 2. SNAPSHOT_END                                                  │
│ 3. CONVERSION_START ──▶ Launches m5.large conversion server     │
│    │   • Mounts snapshot as volume                              │
│    │   • Injects AWS drivers (NVMe, ENA)                        │
│    │   • Modifies boot configuration (GRUB)                     │
│    │   • Adjusts fstab entries                                  │
│    │   • Installs cloud-init/EC2 tools                          │
│    │   • Removes on-prem specific agents                        │
│ 4. CONVERSION_END ──▶ Terminates conversion server              │
│ 5. LAUNCH_START ──▶ Launches target EC2 from converted snapshot │
│ 6. LAUNCH_END ──▶ Instance running                              │
└─────────────────────────────────────────────────────────────────┘

⚠️  Replication CONTINUES during testing (test uses point-in-time snapshot)
```

**Challenges We Faced:**
- ❌ Attempt 1: `InvalidSubnetID.NotFound` — launch template had deleted subnet
- ❌ Attempt 2: `InvalidGroup.NotFound` — launch template had deleted security group
- ❌ Attempt 3: Launched as c5.large (right-sizing override)
- ✅ Attempt 4: Launched as t3.micro after fixing all issues

### 4.2 Validate Test Instance

```bash
# Validation checklist:
# ✅ Instance boots successfully
# ✅ OS is accessible (SSH/SSM)
# ✅ Hostname is correct (he-dev-app-01)
# ✅ Network interfaces are configured
# ✅ Disk mounts are correct
# ✅ Applications start properly
# ✅ Data integrity verified
```

**Common boot failures and causes:**
| Failure | Cause | Fix |
|---------|-------|-----|
| Kernel panic | Missing NVMe/ENA drivers | Install drivers pre-migration |
| Can't mount root | fstab uses /dev/sdX instead of UUID | Update fstab to use UUID |
| No network | ENA driver missing | Install ENA module |
| Hung on boot | SELinux relabeling | Wait 10-30 min or disable SELinux |
| Wrong instance type | Right-sizing override | Set method to NONE |

### 4.3 Mark Ready for Cutover

```bash
aws mgn change-server-life-cycle-state \
    --source-server-id s-38759b0083f7eed55 \
    --life-cycle '{"state":"READY_FOR_CUTOVER"}'

# ⚠️  This TERMINATES the test instance (expected behavior)
# ✅  Replication continues
```

---

## PHASE 5: Cutover

### 5.1 Pre-Cutover Checklist

```
□ Test instance validated successfully
□ Applications confirmed working
□ DNS TTL lowered (if switching DNS)
□ Maintenance window scheduled
□ Stakeholders notified
□ Rollback plan documented
□ Replication lag is 0 (no backlog)
```

### 5.2 Launch Cutover

```bash
aws mgn start-cutover --source-server-ids s-38759b0083f7eed55
```

**Same pipeline as test:**
```
SNAPSHOT → CONVERSION (m5) → LAUNCH → CUTOVER INSTANCE RUNNING
```

**Key difference from test:**
- Uses the LATEST snapshot (most recent data)
- This is your PERMANENT production instance
- Replication still running (safety net)

### 5.3 Post-Cutover Validation

```bash
# Verify cutover instance
aws ec2 describe-instances --instance-ids i-091fcf1e1a78e0833

# Validate:
# ✅ Instance running
# ✅ Applications working
# ✅ Data is current (latest sync)
# ✅ Network/DNS configured
# ✅ Monitoring/alerting active
```

### 5.4 DNS/Traffic Switchover

```
BEFORE CUTOVER:
  users ──▶ DNS (app.company.com) ──▶ On-Prem Server (192.168.12.57)

AFTER CUTOVER:
  users ──▶ DNS (app.company.com) ──▶ AWS Instance (cutover instance IP)
```

### 5.5 Finalize Cutover

```bash
aws mgn finalize-cutover --source-server-id s-38759b0083f7eed55
```

**What happens:**
```
┌───────────────────────────────────────────┐
│ Finalize Actions (IRREVERSIBLE):          │
│                                            │
│ ✗ Replication agent STOPPED               │
│ ✗ Replication server TERMINATED           │
│ ✗ Staging EBS disks DELETED               │
│ ✗ State → CUTOVER (DISCONNECTED)          │
│                                            │
│ ✓ Cutover instance KEEPS RUNNING          │
│ ✓ Source server UNAFFECTED                │
│ ✓ EBS snapshots RETAINED                  │
└───────────────────────────────────────────┘
```

### 5.6 Post-Migration Cleanup

```bash
# Archive source server in MGN
aws mgn mark-as-archived --source-server-id s-38759b0083f7eed55

# Terminate dummy source instance (if testing)
aws ec2 terminate-instances --instance-ids i-0244d3778f7b31463

# Decommission on-prem server (production)
# - Shut down after 2-week bake period
# - Keep as cold standby if needed
```

---

## Complete State Machine

```
                    ┌──────────────────┐
                    │  NOT_CONNECTED   │ ← Agent not installed
                    └────────┬─────────┘
                             │ Install agent
                             ▼
                    ┌──────────────────┐
                    │   NOT_READY      │ ← Replication initiating/syncing
                    └────────┬─────────┘
                             │ Initial sync complete
                             ▼
                    ┌──────────────────┐
         ┌────────▶│ READY_FOR_TEST   │◀────────────┐
         │         └────────┬─────────┘             │
         │                  │ Start Test             │ Revert
         │                  ▼                        │
         │         ┌──────────────────┐             │
         │         │    TESTING       │─────────────┘
         │         └────────┬─────────┘
         │                  │ Mark Ready for Cutover
         │                  ▼
         │         ┌──────────────────┐
         │         │READY_FOR_CUTOVER │◀────────────┐
         │         └────────┬─────────┘             │
         │                  │ Start Cutover          │ Revert
         │                  ▼                        │
         │         ┌──────────────────┐             │
         │         │    CUTTING_OVER  │─────────────┘
         │         └────────┬─────────┘
         │                  │ Finalize Cutover
         │                  ▼
         │         ┌──────────────────┐
         │         │ CUTOVER COMPLETE │ ← DONE (irreversible)
         │         │  (DISCONNECTED)  │
         │         └──────────────────┘
         │
         │ (Can restart from any failed state)
         └─────────────────────────────────────
```

---

## Challenges Summary & Resolutions

| # | Challenge | Root Cause | Resolution | Time to Fix |
|---|-----------|-----------|------------|-------------|
| 1 | Landing Zone FAILED v3.3 | No OUs in organization | Created OUs, moved accounts | 5 min |
| 2 | StackSet deployment failure | Orphaned S3 buckets from previous failed setup | Deleted buckets via cross-account role | 10 min |
| 3 | Landing Zone update failed | Manifest format change in v4.0 | Updated manifest (added `config`, `enabled` flags) | 21 min (deploy) |
| 4 | AWS Transform can't apply SCPs | Sandbox OU created outside Control Tower | Recreated OU + registered with CT baseline | 5 min |
| 5 | Replication STALLED | Staging subnet doesn't exist | Updated replication config with valid subnet | 2 min |
| 6 | Test launch — subnet not found | Launch template references deleted subnet | Created new LT version with valid subnet | 2 min |
| 7 | Test launch — SG not found | Launch template references deleted SG | Updated LT with default VPC security group | 2 min |
| 8 | Instance too large (c5.large) | Right-sizing set to BASIC | Disabled right-sizing, set t3.micro in LT | 2 min |
| 9 | Agent creates new source server | Fresh instance = new identity | Expected behavior with agent-based install | N/A |
| 10 | Test instance terminated | Marked ready for cutover | Expected behavior — test is temporary | N/A |

---

## Key Metrics (Our Migration)

| Metric | Value |
|--------|-------|
| Source disk size | 8 GB |
| Initial sync time | ~10 minutes |
| Replication lag (steady state) | 0 seconds |
| Snapshot time | ~1 minute |
| Conversion time | ~4-5 minutes |
| Launch time | ~2 minutes |
| Total test launch (snapshot to running) | ~7-8 minutes |
| Total cutover (snapshot to running) | ~7-8 minutes |
| End-to-end (discovery to cutover complete) | ~3 hours |

---

## Network Requirements

```
SOURCE SERVER → AWS (Required Ports):
├── TCP 443 (HTTPS) → MGN Service Endpoint (mgn.us-east-1.amazonaws.com)
├── TCP 1500 → Replication Server (staging area)
└── TCP 443 (HTTPS) → S3 (for agent download)

REPLICATION SERVER → AWS:
├── TCP 443 → MGN Service
└── TCP 443 → S3, EBS

CONVERSION SERVER → AWS:
├── TCP 443 → EC2, EBS, S3
└── Instance Metadata Service (169.254.169.254)
```

---

## Cost Breakdown

| Resource | Duration | Approximate Cost |
|----------|----------|-----------------|
| Replication Server (t3.small) | Runs until cutover finalized | ~$0.02/hr |
| Staging EBS (8 GB gp3) | Runs until cutover finalized | ~$0.08/month |
| Conversion Server (m5.large) | ~5 min per test/cutover | ~$0.01 per launch |
| Target Instance (t3.micro) | Permanent after cutover | ~$0.01/hr |
| Data Transfer (replication) | One-time + ongoing changes | Varies |

---

## Best Practices

1. **Pre-migration preparation:**
   - Install NVMe/ENA drivers on source BEFORE migration
   - Use UUIDs in fstab (not device names)
   - Document all application dependencies

2. **Network planning:**
   - Ensure staging subnet has internet access (or VPC endpoints)
   - Use dedicated staging subnet separate from production
   - Pre-create security groups with required rules

3. **Testing strategy:**
   - Always run at least one test before cutover
   - Validate applications, not just OS boot
   - Test with production-like traffic if possible

4. **Cutover strategy:**
   - Lower DNS TTL 24-48 hours before cutover
   - Schedule during maintenance window
   - Keep replication running until fully validated
   - Don't finalize immediately — validate first

5. **Rollback plan:**
   - Before finalization: revert cutover → back to READY_FOR_CUTOVER
   - After finalization: restore from EBS snapshot (manual process)
   - Keep source server running for 2-week bake period
