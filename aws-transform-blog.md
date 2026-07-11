# End-to-End Cloud Migration with AWS Transform: From VMware Discovery to MGN Cutover

## An AWS Ambassador Level 300 Deep Dive

**Author:** Ramandeep Chandna | AWS Community Builder  
**Level:** 300 (Advanced)  
**Services:** AWS Transform, AWS MGN, AWS Control Tower, AWS Organizations, Amazon VPC  
**Date:** July 2026

---

## Abstract

This blog provides a comprehensive, hands-on walkthrough of using **AWS Transform** — AWS's agentic AI-powered migration planning service — to orchestrate a complete VMware-to-AWS migration. We'll cover how AWS Transform ingests server inventory, generates intelligent wave plans with R-Strategy (7Rs) recommendations, provisions a Control Tower Landing Zone, designs target network architecture, and produces MGN-ready import files — all through a conversational AI interface. We'll then execute the actual migration in the MGN console, covering replication, testing, and cutover, along with the real-world challenges encountered and how they were resolved.

---

## Table of Contents

1. [Introduction to AWS Transform](#introduction)
2. [Phase 1: Discovery & Inventory Import](#phase-1)
3. [Phase 2: Migration Planning & Wave Generation](#phase-2)
4. [Phase 3: R-Strategy (7Rs) Analysis](#phase-3)
5. [Phase 4: Landing Zone Deployment](#phase-4)
6. [Phase 5: Network Architecture Deployment](#phase-5)
7. [Phase 6: MGN Import File Generation](#phase-6)
8. [Phase 7: Migration Execution in MGN](#phase-7)
9. [Challenges Faced & Resolutions](#challenges)
10. [Best Practices](#best-practices)
11. [Common Issues & Troubleshooting](#common-issues)
12. [Conclusion](#conclusion)

---

## Introduction to AWS Transform {#introduction}

AWS Transform is an **agentic AI service** for large-scale cloud migration and modernization. Unlike traditional migration tools that require manual configuration at every step, AWS Transform uses generative AI to understand your infrastructure, make intelligent recommendations, and execute migration workflows through natural language conversation.

### Key Capabilities

- **VMware-to-EC2 rehosting** — Lift-and-shift virtual machines
- **Wave planning** — Sequence workloads based on dependencies and priorities
- **R-Strategy (7Rs) analysis** — Classify servers as Rehost, Replatform, Refactor, Retire, etc.
- **Landing Zone generation** — Deploy Control Tower with proper governance
- **Network migration** — Design and deploy target VPC architecture
- **MGN integration** — Generate import files for AWS Application Migration Service

### Access Methods

AWS Transform is available through:
1. **Web Console** — `https://<workspace-id>.transform.us-east-1.on.aws/workspaces`
2. **CLI (atx)** — Terminal-based interactive agent for automation

### The Four-Phase Workflow

```
Assess → Plan → Execute → Monitor
```

1. **Assess** — Understand your current environment (applications, tech debt, dependencies, cloud readiness)
2. **Plan** — Sequence workloads into migration waves, define priorities, identify risks
3. **Execute** — Modernize and migrate using transformation agents tailored to your workload type
4. **Monitor** — Validate and optimize post-migration

### Migration Job Types Available

When creating a new job in AWS Transform, you can select:
- **End-to-End Migration** — Complete workflow from planning to execution
- **Network Migration** — VPC/subnet design and deployment only
- **Landing Zone** — Control Tower setup only
- **Landing Zone, Network and Server Migration** — Combined infrastructure + workload
- **Migration Planning and Server Migration** — Planning + MGN execution
- **Migration Planning Only** — Wave plans and recommendations without execution
- **Server Migration Only** — Direct MGN import without planning

For our migration, we selected **End-to-End Migration** which created a structured 5-step job plan.

---

## Phase 1: Discovery & Inventory Import {#phase-1}

### 1.1 Creating the Migration Workspace

The first step in AWS Transform is creating a **Workspace** — a container for managing jobs, artifacts, collaborators, and connectors.

**Workspace Configuration:**
- **Name:** Migration-Workspace
- **Description:** A workspace for creating jobs, managing artifacts, and collaborating with your team on transformation workflows

*Figure 1: Creating Migration-Workspace in AWS Transform console*
<!-- Screenshot: Screenshot 2026-07-09 at 12.35.03PM.png -->

### 1.2 Creating the Migration Job

Within the workspace, we created a new job and selected the **Migration** job type. AWS Transform presented several sub-options:

- End-to-End Migration
- Network Migration
- Landing Zone
- Landing Zone, Network and Server Migration
- Migration Planning and Server Migration
- Source Code Containerization
- Migration Planning Only
- Server Migration Only

We selected **End-to-End Migration**, which automatically generated a structured **Job Plan** with 5 phases:

| Step | Task | Description |
|------|------|-------------|
| 1 | Build migration plan | Group applications into migration waves |
| 2 | Connect target AWS account | Configure target account connector |
| 3 | Build landing zone | Generate and deploy Landing Zone configuration |
| 4 | Migrate Network | Define and deploy necessary VPC networks to AWS |
| 5 | Configure migration defaults | Set up EC2 recommendations, launch templates, and generate migration inventory |

*Figure 2: End-to-End Migration job plan with 5 structured phases*
<!-- Screenshot: Screenshot 2026-07-09 at 12.37.45PM.png -->

### 1.3 Uploading Server Inventory (RVTools Export)

AWS Transform requested server inventory data. It accepts multiple formats:
- **Discovery tool exports** (AWS Transform Discovery, Movere, Cloudamize, Modelizeit)
- **RVTools exports** (vInfo, vMemory, vDisk tabs)
- **CMDB reports** (ServiceNow, Lansweeper, or similar)
- **Network discovery data** (IP/hostname lists, network scans)
- **Any file with relevant server data** (hostnames, IPs, OS versions, CPU/memory specs)

We uploaded our **RVTools_export_all.zip** (66.35 KB) exported from our vCenter environment.

*Figure 3: Uploading RVTools export to AWS Transform*
<!-- Screenshot: Screenshot 2026-07-09 at 12.39.51PM.png -->

### 1.4 Inventory Analysis Results

AWS Transform processed the RVTools export and provided an **Ingestion Summary**:

| Category | Details |
|----------|---------|
| **Total Servers** | 82 (deduplicated from 164 raw rows) |
| **Applications** | 0 (RVTools doesn't contain application definitions — created in next step) |
| **Network Connections** | None detected |

**Operating System Breakdown:**

| OS Type | Count |
|---------|-------|
| RHEL | 66 |
| Windows | 11 |
| Linux | 4 |
| Other | 1 |

**Environment Breakdown:**

| Environment | Count |
|-------------|-------|
| prod | 24 |
| alpha | 16 |
| uat | 14 |
| dev | 11 |
| beta | 10 |
| unclassified | 7 |

**Data Quality Notes:**
- 14 servers missing IP addresses (powered-off VMs)
- 2 servers missing MAC addresses
- No utilization metrics available from RVTools

*Figure 4: Ingestion summary showing 82 servers across 6 environments*
<!-- Screenshot: Screenshot 2026-07-09 at 12.46.14PM.png -->

> **Key Insight:** AWS Transform automatically deduplicated the 164 raw rows (82 unique VMs × 2 source files) and classified servers by environment based on hostname naming conventions — no manual tagging required.


---

## Phase 2: Migration Planning & Wave Generation {#phase-2}

### 2.1 Planning Configuration

AWS Transform asked for planning preferences before generating the wave plan:

| Parameter | Options | Selected |
|-----------|---------|----------|
| **Grouping** | environment (default), os_type, auto | environment |
| **Separation** | Keep prod/non-prod in separate waves? | yes (default) |
| **Wave Size** | Max servers per wave (default: 100) | 100 |
| **Migration Strategy** | rehost (default), containerize, decide later | rehost |

You can say **"use defaults"** to accept all recommended settings, or customize each parameter individually.

*Figure 5: Migration planning configuration options*
<!-- Screenshot: Screenshot 2026-07-09 at 12.47.37PM.png -->

### 2.2 Application Grouping Results

AWS Transform analyzed 82 servers across 6 environments and organized them into **6 applications** using environment-based grouping with prod/non-prod separation:

| Priority | Application | Environment | Servers |
|----------|-------------|-------------|---------|
| 1 | Dev-Application | dev | 11 |
| 2 | Alpha-Application | alpha | 16 |
| 3 | Beta-Application | beta | 10 |
| 4 | Uat-Application | uat | 14 |
| 5 | Unclassified-Application | unclassified | 7 |
| 6 | Prod-Application | prod | 24 |

**Key Details:**
- 100% server coverage — all 82 servers assigned
- Non-production environments prioritized first for migration
- Rehost strategy applied to all applications

*Figure 6: Application grouping — 82 servers organized into 6 applications*
<!-- Screenshot: Screenshot 2026-07-09 at 1.02.53PM.png -->

> **Design Decision:** AWS Transform intentionally sequences non-production environments (dev → alpha → beta → uat) before production. This allows teams to validate the migration process on lower-risk workloads, build confidence, and catch issues before migrating critical production systems.

### 2.3 Wave Generation

Based on the application grouping, AWS Transform generated migration waves that respect:
- Application boundaries (servers in the same app move together)
- Environment isolation (prod separate from non-prod)
- Wave size limits (max 100 servers per wave)
- Dependency chains (prerequisite services first)

---

## Phase 3: R-Strategy (7Rs) Analysis {#phase-3}

### 3.1 R-Strategy Classification

AWS Transform automatically generated an **R-Strategy (7Rs) Report** analyzing each server's optimal migration path:

| Strategy | Servers | Description |
|----------|---------|-------------|
| **Rehost** | 52 | Lift and shift to EC2 |
| **Replatform** | 18 | Migrate to managed services (e.g., RDS) |
| **Retire** | 12 | Candidates for decommissioning |

### 3.2 Executive Summary Dashboard

The generated interactive HTML report (`r_strategy_report.html`) provided:

- **82 servers** analyzed across **6 applications**
- **Average Confidence:** 57.3%
- **Modernization Pathways Detected:** OSS DB → RDS (18 servers)

### 3.3 Key Findings

1. **⚠️ Potential EC2 bias:** 63% of servers recommended for Rehost, but 18 servers show managed-service signals — worth reviewing whether those should Replatform instead of defaulting to EC2

2. **Modernization opportunity:** 18 servers detected with open-source database signals → candidates for **Amazon RDS** migration (vm-101, vm-27, vm-29, vm-36, vm-37...)

3. **Retire candidates:** 12 servers flagged for potential retirement (likely powered-off or idle VMs) — these require owner confirmation before decommissioning

### 3.4 Criticality × Strategy Matrix

| Criticality | Rehost | Replatform | Refactor | Retire | Retain | Repurchase | Relocate |
|-------------|--------|------------|----------|--------|--------|------------|----------|
| **high** | 52 | 18 | 0 | 12 | 0 | 0 | 0 |
| **medium** | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| **low** | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

### 3.5 Recommended Next Steps

1. Evaluate RDS migration feasibility for 18 open-source database hosts
2. Validate retirement of 12 candidate servers with application owners
3. Review applications with mixed strategies to confirm move-group composition

*Figure 7: R-Strategy Migration Report — Executive Summary with Criticality Matrix*
<!-- Screenshot: Screenshot 2026-07-09 at 1.10.48PM.png -->

> **Level 300 Insight:** The R-Strategy report isn't just classification — it feeds directly into the wave plan. Servers marked "Retire" are excluded from MGN import files. Servers marked "Replatform" get flagged for manual review before proceeding with rehost. This prevents unnecessary migration of workloads that should be modernized or decommissioned.


---

## Phase 4: Landing Zone Deployment {#phase-4}

### 4.1 Landing Zone Configuration

AWS Transform generated a Control Tower Landing Zone configuration based on migration requirements:

**Landing Zone Components:**
- **Control Tower version:** 4.0
- **Governed Regions:** us-east-1
- **Organization Structure:**
  - Security OU → Audit account + Log Archive account
  - Workloads OU → Migration target accounts
  - Sandbox OU → Testing and development
  - Infrastructure OU → Shared services

**Service Integrations:**
- AWS Config: Enabled (aggregator in Audit account)
- Security Roles: Enabled
- Centralized Logging: Enabled (Log Archive account)
- Access Management (IAM Identity Center): Enabled
- Backup: Disabled

### 4.2 Landing Zone Deployment

AWS Transform deployed the Landing Zone configuration to our AWS account. This process:
1. Created the organizational unit structure
2. Configured Control Tower with v4.0 manifest
3. Enabled baselines on each OU
4. Applied Service Control Policies (SCPs)
5. Set up centralized logging and security roles

*Figure 8: Landing Zone deployment progress in AWS Transform*
<!-- Screenshot: Screenshot 2026-07-09 at 1.15.25PM.png -->

> **Challenge Encountered:** The Landing Zone deployment initially failed due to pre-existing orphaned resources from a previous failed setup. See [Challenges section](#challenges) for the full resolution.

---

## Phase 5: Network Architecture Deployment {#phase-5}

### 5.1 Network Design Generation

Based on the server inventory and network topology provided, AWS Transform designed the target AWS network architecture:

**VPC Configuration:**
- **VPC CIDR:** Mapped from on-premises network segments
- **Availability Zones:** Multi-AZ deployment for high availability
- **Subnet Design:**
  - Public subnets (for load balancers, bastion hosts)
  - Private subnets (for application servers)
  - Database subnets (for data tier)
- **Route Tables:** Configured for proper traffic flow
- **Security Groups:** Based on existing firewall rules

### 5.2 Network Deployment

AWS Transform deployed the network components using CloudFormation:
- VPC with proper CIDR allocation
- Subnets across availability zones
- Internet Gateway and NAT Gateways
- Route tables with appropriate routes
- Security groups matching on-premises firewall rules

*Figure 9: Network deployment configuration in AWS Transform*
<!-- Screenshot: Screenshot 2026-07-09 at 1.20.08PM.png -->

---

## Phase 6: MGN Import File Generation {#phase-6}

### 6.1 Configure Migration Defaults

The final step in AWS Transform's job plan is **Configure migration defaults**, which:
1. Maps source servers to target EC2 instance types (right-sizing)
2. Configures launch templates with proper subnet/SG assignments
3. Generates the MGN-compatible import file
4. Creates the migration inventory for tracking

### 6.2 EC2 Instance Recommendations

AWS Transform analyzed each server's CPU, memory, and disk specifications to recommend optimal EC2 instance types:

**Right-sizing approach:**
- Maps vCPU count to EC2 instance family
- Considers memory requirements for instance size
- Factors in disk IOPS requirements
- Recommends current-generation instances (m5, c5, r5, t3)

### 6.3 MGN Import File

AWS Transform generated the MGN import file containing:
- Source server identifiers (hostname, IP, vCenter UUID)
- Target configuration (instance type, subnet, security group)
- Replication settings (staging subnet, replication server type)
- Launch configuration (boot mode, licensing, post-launch actions)

### 6.4 The Read-Only Handoff

**Critical Architecture Decision:** Once AWS Transform generates the MGN import file, it operates in **read-only mode** for the migration execution phase. This means:

- ✅ Transform can **view** migration status, replication progress, and job state
- ❌ Transform **cannot** modify replication settings, trigger tests, or initiate cutover
- ✅ All execution actions must be performed in the **MGN console** directly

**Why read-only?** This separation ensures:
1. **Audit trail** — All migration actions are traceable to human operators
2. **Safety** — No AI-initiated production changes without explicit approval
3. **Control** — Migration teams maintain full authority over cutover decisions
4. **Compliance** — Meets change management requirements for production workloads

*Figure 10: MGN import file generation and read-only mode indicator*
<!-- Screenshot: Screenshot 2026-07-09 at 1.31.25PM.png -->

---

## Phase 7: Migration Execution in MGN {#phase-7}

### 7.1 Source Server Registration

After the MGN import, source servers appeared in the MGN console with state `PENDING_INSTALLATION`. For our test migration of `he-dev-app-01`:

1. **Installed MGN Replication Agent** on the source server
2. **Agent registered** with MGN service (source server ID: `s-38759b0083f7eed55`)
3. **Replication initiated** — block-level data sync began

### 7.2 Replication Progress

| Step | Status | Duration |
|------|--------|----------|
| Create Security Group | ✅ Succeeded | ~30s |
| Launch Replication Server | ✅ Succeeded | ~2 min |
| Boot Replication Server | ✅ Succeeded | ~1 min |
| Authenticate with Service | ✅ Succeeded | ~10s |
| Download Replication Software | ✅ Succeeded | ~1 min |
| Create Staging Disks | ✅ Succeeded | ~30s |
| Attach Staging Disks | ✅ Succeeded | ~10s |
| Pair with Agent | ✅ Succeeded | ~1 min |
| Connect to Replication Server | ✅ Succeeded | ~30s |
| Start Data Transfer | ✅ Succeeded | ~10 min |

**Total time to CONTINUOUS replication:** ~15 minutes for 8 GB disk

### 7.3 Test Migration

```bash
aws mgn start-test --source-server-ids s-38759b0083f7eed55
```

**Test pipeline:** Snapshot → Conversion (m5.large, ~5 min) → Launch (t3.micro) → Running ✅

### 7.4 Cutover

```bash
aws mgn start-cutover --source-server-ids s-38759b0083f7eed55
aws mgn finalize-cutover --source-server-id s-38759b0083f7eed55
```

**Final state:** CUTOVER COMPLETE, replication DISCONNECTED ✅


---

## Challenges Faced & Resolutions {#challenges}

### Challenge 1: Control Tower Landing Zone in FAILED State

**Problem:** Landing Zone was stuck at version 3.3 in FAILED + DRIFTED state with multiple configuration issues.

**Root Cause:** All AWS accounts were sitting directly under the organization root — no OUs existed. Control Tower 4.0 requires accounts in Organizational Units.

**Resolution:**
1. Created Security and Sandbox OUs
2. Moved Audit and Log Archive accounts into Security OU
3. Updated Landing Zone manifest to v4.0 format (added `config` section, explicit `enabled` flags, removed `organizationStructure`)

**Time to resolve:** 5 minutes (+ 21 minutes for LZ deployment)

---

### Challenge 2: Orphaned S3 Buckets Blocking StackSet Deployment

**Problem:** Landing Zone update failed with "Validation failed with 2 error(s)" on `AWSControlTowerLoggingResources` StackSet.

**Root Cause:** A previous failed Landing Zone setup had created S3 buckets in the Log Archive account (`aws-controltower-logs-*` and `aws-controltower-s3-access-logs-*`), but the CloudFormation stack never completed. CloudFormation couldn't recreate existing buckets.

**Resolution:**
1. Assumed role into Log Archive account via `AWSControlTowerExecution`
2. Emptied versioned objects from both buckets
3. Deleted the orphaned buckets
4. Cleaned up failed StackSet instances
5. Retried Landing Zone update

**Time to resolve:** 10 minutes

---

### Challenge 3: Sandbox OU Not Manageable by AWS Transform

**Problem:** AWS Transform reported: "Sandbox OU was created outside of AWS Transform and cannot be modified through this tool."

**Root Cause:** The Sandbox OU was manually created (not through Control Tower), so it lacked the required baseline registration. Additionally, it contained a suspended/closed account blocking SCP operations.

**Resolution:**
1. Moved suspended account to root
2. Deleted the unmanaged Sandbox OU
3. Created new Sandbox OU
4. Registered with Control Tower using `EnableBaseline` API with Identity Center parameter

**Time to resolve:** 5 minutes

---

### Challenge 4: MGN Replication Stalled — FAILED_TO_CREATE_SECURITY_GROUP

**Problem:** After MGN agent installation, replication was stuck at `CREATE_SECURITY_GROUP` → FAILED.

**Root Cause:** The default replication configuration template pointed to a non-existent subnet (`subnet-0704f97e7910c9269`) from a previously deleted VPC.

**Resolution:**
```bash
aws mgn update-replication-configuration \
    --source-server-id s-38759b0083f7eed55 \
    --staging-area-subnet-id subnet-040f98af126cd1889
```

**Time to resolve:** 2 minutes

---

### Challenge 5: Test Launch Failed — Stale Launch Template References

**Problem:** Conversion server (m5) succeeded, but final instance launch failed with `InvalidSubnetID.NotFound` and subsequently `InvalidGroup.NotFound`.

**Root Cause:** The EC2 Launch Template referenced both a deleted subnet AND a deleted security group from the same removed VPC.

**Resolution:**
1. Created new launch template version with valid subnet and default VPC security group
2. Set new version as default
3. Disabled right-sizing (`NONE`) to prevent instance type override

**Time to resolve:** 4 minutes (across 2 iterations)

---

### Challenge 6: Instance Type Override by Right-Sizing

**Problem:** MGN launched a c5.large instance despite the launch template specifying t3.micro.

**Root Cause:** MGN's `targetInstanceTypeRightSizingMethod` was set to `BASIC`, which overrides the launch template based on source server CPU/RAM specs.

**Resolution:**
```bash
aws mgn update-launch-configuration \
    --source-server-id s-38759b0083f7eed55 \
    --target-instance-type-right-sizing-method NONE
```

**Time to resolve:** 1 minute

---

## Best Practices {#best-practices}

### Planning Phase

1. **Start with RVTools** — It's the fastest way to get comprehensive VMware inventory into AWS Transform. Export all tabs (vInfo, vMemory, vDisk) for maximum data richness.

2. **Use environment-based grouping** — AWS Transform's default grouping by environment (dev/test/prod) creates a natural migration sequence with built-in risk reduction.

3. **Don't skip the R-Strategy report** — The 7Rs classification catches servers that should be retired or replatformed before you waste effort migrating them.

4. **Review "Replatform" candidates** — 18 of our 82 servers showed database signals. Migrating these to RDS instead of EC2 could save 40-60% on operational costs.

5. **Validate "Retire" candidates with owners** — Powered-off VMs might be seasonal workloads, not decommission candidates.

### Infrastructure Setup

6. **Fix Landing Zone before migration** — A healthy Control Tower foundation prevents cascading failures during server migration. Don't skip this step.

7. **Register all OUs with Control Tower** — OUs created manually outside CT cannot be managed by AWS Transform. Always use `EnableBaseline` to register them.

8. **Verify subnet/SG references after VPC changes** — If you ever delete a VPC, check ALL MGN replication configs AND launch templates for stale references. This is the #1 cause of migration failures.

9. **Use dedicated staging subnets** — Don't put replication servers in the same subnet as production workloads. Use a separate staging subnet with appropriate routing.

### Migration Execution

10. **Always run a test before cutover** — The test launch validates the entire pipeline (snapshot → conversion → launch) without affecting replication.

11. **Disable right-sizing for testing** — Set `targetInstanceTypeRightSizingMethod: NONE` during testing to control costs. Re-enable for production cutover if desired.

12. **Lower DNS TTL 24-48 hours before cutover** — This ensures fast DNS propagation when you switch traffic.

13. **Don't finalize cutover immediately** — Keep replication running as a safety net until you've fully validated the cutover instance. Finalization is irreversible.

14. **Use SSM for instance access** — Avoid SSH key management complexity. Attach `AmazonSSMRoleForInstancesQuickSetup` instance profile for Session Manager access.

### AWS Transform Specific

15. **Export conversations** — AWS Transform generates conversation exports (`.md` files) that serve as audit trails and documentation.

16. **Use workspaces for isolation** — Separate migration projects into different workspaces to avoid cross-contamination.

17. **Leverage the CLI (atx) for automation** — The `atx` CLI supports the same conversational interface as the web console but enables scripting and CI/CD integration.

---

## Common Issues & Troubleshooting {#common-issues}

### AWS Transform Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| "OU created outside AWS Transform" | OU not registered with Control Tower | Delete and recreate OU, then `EnableBaseline` |
| Transform can't apply SCPs | Suspended account in target OU | Move suspended accounts out of governed OUs |
| Workspace creation fails | Region not supported | Use us-east-1 (primary supported region) |
| File upload rejected | Unsupported format or too large | Use ZIP for RVTools, ensure < 100MB |

### Landing Zone Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| "Accounts cannot be placed in root" | Missing OUs | Create OUs and move accounts |
| StackSet validation failed | Orphaned resources from previous attempt | Delete orphaned S3 buckets/stacks |
| Manifest format error | v3.3 manifest used for v4.0 | Add `config` section, explicit `enabled` flags |
| Baseline enable fails | Missing IdentityCenterEnabledBaselineArn | Include the parameter from ListEnabledBaselines |

### MGN Replication Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| FAILED_TO_CREATE_SECURITY_GROUP | Invalid staging subnet | Update replication config with valid subnet |
| Agent registers as new server | Fresh instance, not original VM | Expected — use new source server ID |
| Replication STALLED | Network connectivity or IAM permissions | Check TCP 1500 outbound and IAM role |
| "Asked to sleep for 300 seconds" | MGN rejecting agent connection | Fix underlying config issue (subnet/SG) |

### MGN Launch Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| InvalidSubnetID.NotFound | Launch template references deleted subnet | Create new LT version with valid subnet |
| InvalidGroup.NotFound | Launch template references deleted SG | Update SG to default VPC security group |
| Wrong instance type launched | Right-sizing override (BASIC mode) | Set `targetInstanceTypeRightSizingMethod: NONE` |
| Boot failure — kernel panic | Missing NVMe/ENA drivers | Install drivers on source pre-migration |
| Boot failure — can't mount root | fstab uses /dev/sdX instead of UUID | Update fstab to use UUID before migration |
| Instance hangs on boot | SELinux relabeling | Wait 10-30 min or disable SELinux pre-migration |

### Post-Migration Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Test instance terminated unexpectedly | Marked "Ready for Cutover" | Expected behavior — test instances are temporary |
| Can't re-run test | Server in wrong lifecycle state | Use `ChangeServerLifeCycleState` API to reset |
| Replication stopped | Finalized cutover | Irreversible — restore from EBS snapshot if needed |

---

## Conclusion {#conclusion}

AWS Transform represents a paradigm shift in how organizations approach cloud migration. By combining generative AI with deep AWS service integration, it reduces a traditionally weeks-long planning process to hours — while maintaining the rigor and governance that enterprise migrations demand.

### Key Takeaways

1. **AI-driven planning isn't a replacement for expertise — it's an accelerator.** AWS Transform handles the repetitive analysis (inventory parsing, environment detection, grouping), freeing engineers to focus on architectural decisions.

2. **The read-only handoff to MGN is intentional.** Separating planning (AI-assisted) from execution (human-controlled) provides the safety net that production migrations require.

3. **Infrastructure issues compound.** A failed Landing Zone → stale subnet references → failed replication → failed launches. Fix the foundation first.

4. **Test everything.** AWS Transform's structured approach (plan → infrastructure → test → cutover) exists because each phase validates assumptions made in the previous one.

5. **82 servers, 6 applications, 3 strategies — planned in under 30 minutes.** That's the power of AI-assisted migration planning.

### What's Next

- Execute remaining waves (alpha, beta, uat, prod) following the validated pattern
- Evaluate 18 database servers for RDS migration (Replatform path)
- Validate and retire 12 powered-off VM candidates
- Set up monitoring and optimization using AWS Compute Optimizer post-migration

---

## References

- [AWS Transform Documentation](https://docs.aws.amazon.com/transform/)
- [AWS MGN User Guide](https://docs.aws.amazon.com/mgn/latest/ug/)
- [AWS Control Tower Landing Zone API](https://docs.aws.amazon.com/controltower/latest/userguide/lz-api-launch.html)
- [MGN Replication Agent Installation](https://docs.aws.amazon.com/mgn/latest/ug/agent-installation.html)
- [AWS Well-Architected Migration Lens](https://docs.aws.amazon.com/wellarchitected/latest/migration-lens/welcome.html)

---

*This blog was written as part of the AWS Ambassador program submission. All screenshots and configurations are from a real migration performed on July 9-11, 2026.*
