# Troubleshooting AWS MGN Migration: From Failed Landing Zone to Successful Test Migration

## Overview

This blog documents the end-to-end troubleshooting journey of performing a test migration using AWS Application Migration Service (MGN) with AWS Transform. We encountered multiple issues spanning AWS Control Tower, Organizations, CloudFormation StackSets, and MGN configuration — and resolved them all.

**Source Server:** `he-dev-app-01` (imported via MGN Connector from vCenter)  
**Goal:** Successfully perform a test migration to validate the migration workflow

---

## Issue 1: Control Tower Landing Zone in FAILED State (v3.3)

### Problem
The AWS Control Tower Landing Zone was stuck in `FAILED` + `DRIFTED` state at version 3.3 with multiple configuration issues:
- Version must be 4.0 or higher
- Security roles not enabled
- AWS Config not enabled
- Config account not configured
- Auto-enrollment not enabled

### Root Cause
1. All AWS accounts were sitting directly under the organization **root** — no OUs existed
2. Control Tower 4.0 requires accounts to be in Organizational Units

### Fix
```bash
# Step 1: Create required OUs
aws organizations create-organizational-unit --parent-id r-28h4 --name "Security"
aws organizations create-organizational-unit --parent-id r-28h4 --name "Sandbox"

# Step 2: Move service accounts to Security OU
aws organizations move-account --account-id 044789067646 --source-parent-id r-28h4 --destination-parent-id ou-28h4-l27f2l7r
aws organizations move-account --account-id 598361629577 --source-parent-id r-28h4 --destination-parent-id ou-28h4-l27f2l7r
```

### Manifest for Landing Zone 4.0
Key changes from v3.3 → v4.0:
- `organizationStructure` section **removed**
- `config` section **added** with `enabled` flag
- `securityRoles` now requires `enabled: true` explicitly
- All service integrations require explicit `enabled` boolean

```json
{
    "accessManagement": { "enabled": true },
    "securityRoles": { "enabled": true, "accountId": "044789067646" },
    "backup": { "enabled": false },
    "governedRegions": ["us-east-1"],
    "centralizedLogging": {
        "accountId": "598361629577",
        "enabled": true,
        "configurations": {
            "loggingBucket": { "retentionDays": 365 },
            "accessLoggingBucket": { "retentionDays": 3650 }
        }
    },
    "config": {
        "enabled": true,
        "accountId": "044789067646",
        "configurations": {
            "loggingBucket": { "retentionDays": 365 },
            "accessLoggingBucket": { "retentionDays": 365 }
        }
    }
}
```

---

## Issue 2: StackSet Deployment Failure (Orphaned S3 Buckets)

### Problem
After fixing the OU structure, the Landing Zone update still failed with:
> "Validation failed with 2 error(s)" on `AWSControlTowerLoggingResources` StackSet

### Root Cause
Previous failed Landing Zone setup had created S3 buckets in the Log Archive account, but the CloudFormation stack never completed. These orphaned buckets blocked the StackSet from recreating them:
- `aws-controltower-logs-598361629577-us-east-1`
- `aws-controltower-s3-access-logs-598361629577-us-east-1`

### Fix
```bash
# Assume role into Log Archive account
CREDS=$(aws sts assume-role --role-arn arn:aws:iam::598361629577:role/AWSControlTowerExecution --role-session-name fix-buckets)

# Empty and delete versioned buckets
aws s3 rm s3://aws-controltower-s3-access-logs-598361629577-us-east-1 --recursive
# Delete all object versions and delete markers
aws s3api delete-objects --bucket <bucket> --delete '{"Objects":[...]}'
aws s3api delete-bucket --bucket aws-controltower-logs-598361629577-us-east-1
aws s3api delete-bucket --bucket aws-controltower-s3-access-logs-598361629577-us-east-1

# Clean up failed StackSet instance
aws cloudformation delete-stack-instances --stack-set-name AWSControlTowerLoggingResources \
    --accounts 598361629577 --regions us-east-1 --retain-stacks

# Retry Landing Zone update
aws controltower update-landing-zone --landing-zone-identifier <arn> --version 4.0 --manifest file://manifest.json
```

**Result:** Landing Zone update succeeded in ~21 minutes ✅

---

## Issue 3: Sandbox OU Created Outside Control Tower

### Problem
AWS Transform couldn't apply SCPs to the Sandbox OU because:
1. It was created manually (outside Control Tower)
2. It contained a suspended/closed account blocking operations

### Fix
```bash
# Move suspended account out and delete the unmanaged OU
aws organizations move-account --account-id 230352786706 --source-parent-id ou-28h4-lln5in5d --destination-parent-id r-28h4
aws organizations delete-organizational-unit --organizational-unit-id ou-28h4-lln5in5d

# Create new Sandbox OU
aws organizations create-organizational-unit --parent-id r-28h4 --name "Sandbox"

# Register with Control Tower (enable baseline)
aws controltower enable-baseline \
    --baseline-identifier arn:aws:controltower:us-east-1::baseline/17BSJV3IGJ2QSGA2 \
    --baseline-version 5.0 \
    --target-identifier arn:aws:organizations::114805761158:ou/o-ei5z31f63f/ou-28h4-umqxvvwg \
    --parameters '[{"key":"IdentityCenterEnabledBaselineArn","value":"arn:aws:controltower:us-east-1:114805761158:enabledbaseline/XAET31Q9MFI5K2LYN"}]'
```

---

## Issue 4: Creating a Workload Account for Migration Testing

### Problem
The source server `he-dev-app-01` was imported from vCenter via MGN Connector but didn't physically exist. Needed a dummy server to install the replication agent.

### Fix
```bash
# Create new workload account (for proper OU governance)
aws organizations create-account --email ramandeep.chandna+workload@gmail.com --account-name Workload --role-name AWSControlTowerExecution
aws organizations move-account --account-id 150897468843 --source-parent-id r-28h4 --destination-parent-id ou-28h4-b1nastew

# Launch dummy EC2 instance as the "source server"
aws ec2 run-instances --image-id ami-0df80e66b6b8a0056 --instance-type t3.micro --count 1 \
    --network-interfaces DeviceIndex=0,SubnetId=subnet-040f98af126cd1889,AssociatePublicIpAddress=true \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=he-dev-app-01}]'
```

---

## Issue 5: MGN Replication Agent — FAILED_TO_CREATE_SECURITY_GROUP

### Problem
After installing the MGN agent, replication stalled at `CREATE_SECURITY_GROUP` step with error `FAILED_TO_CREATE_SECURITY_GROUP`.

### Root Cause
The MGN **Replication Configuration Template** pointed to a non-existent subnet (`subnet-0704f97e7910c9269`) that had been deleted.

### Fix
```bash
# Update replication configuration with valid subnet
aws mgn update-replication-configuration \
    --source-server-id s-38759b0083f7eed55 \
    --staging-area-subnet-id subnet-040f98af126cd1889 \
    --associate-default-security-group \
    --replication-server-instance-type t3.small \
    --data-plane-routing PUBLIC_IP \
    --create-public-ip
```

**Result:** Replication started → Initial sync completed (8 GB) → Continuous replication with 0 lag ✅

---

## Issue 6: Test Launch Failed — InvalidSubnetID.NotFound

### Problem
Conversion (m5) succeeded, but the final test instance launch failed:
> "An error occurred (InvalidSubnetID.NotFound) when calling the DescribeSubnets operation: The subnet ID 'subnet-0704f97e7910c9269' does not exist"

### Root Cause
The **EC2 Launch Template** (`lt-073a4879da2602f77`) still referenced the deleted subnet in its NetworkInterfaces configuration.

### Fix
```bash
# Create new launch template version with valid subnet
aws ec2 create-launch-template-version --launch-template-id lt-073a4879da2602f77 \
    --source-version 2 \
    --launch-template-data '{"NetworkInterfaces":[{"AssociatePublicIpAddress":true,"DeleteOnTermination":true,"DeviceIndex":0,"Groups":["sg-7302c328"],"SubnetId":"subnet-040f98af126cd1889"}]}'

# Set as default
aws ec2 modify-launch-template --launch-template-id lt-073a4879da2602f77 --default-version 5
```

---

## Issue 7: Test Launch Failed — InvalidGroup.NotFound

### Problem
After fixing the subnet, next launch failed with:
> "The security group 'sg-0ed55d56d2a253c82' does not exist in VPC 'vpc-0af49170'"

### Root Cause
The launch template also referenced a security group from the same deleted VPC.

### Fix
```bash
# Update launch template with default VPC's security group
aws ec2 create-launch-template-version --launch-template-id lt-073a4879da2602f77 \
    --source-version 4 \
    --launch-template-data '{"NetworkInterfaces":[{"AssociatePublicIpAddress":true,"DeleteOnTermination":true,"DeviceIndex":0,"Groups":["sg-7302c328"],"SubnetId":"subnet-040f98af126cd1889"}]}'
```

**Result:** Test migration LAUNCHED successfully ✅

---

## Issue 8: Right-Sizing Override (c5.large → t3.micro)

### Problem
MGN launched a c5.large instance despite wanting t3.micro for testing.

### Root Cause
MGN's `targetInstanceTypeRightSizingMethod` was set to `BASIC`, which overrides the launch template instance type based on source server specs.

### Fix
```bash
# Disable right-sizing
aws mgn update-launch-configuration \
    --source-server-id s-38759b0083f7eed55 \
    --target-instance-type-right-sizing-method NONE

# Update launch template to t3.micro
aws ec2 create-launch-template-version --launch-template-id lt-073a4879da2602f77 \
    --source-version 6 \
    --launch-template-data '{"InstanceType":"t3.micro"}'
aws ec2 modify-launch-template --launch-template-id lt-073a4879da2602f77 --default-version 8
```

**Result:** Test instance launched as t3.micro ✅ (`i-051daeb302b30ca4f`)

---

## Key Takeaways

1. **Always validate subnet/SG references** — When VPCs are deleted, MGN replication configs AND launch templates retain stale references. Check both.

2. **Landing Zone 4.0 manifest is different** — The `organizationStructure` block is removed, and `config` + explicit `enabled` flags are required.

3. **Orphaned resources block StackSets** — If a previous Control Tower setup failed, clean up S3 buckets and stack instances before retrying.

4. **OUs must be registered with Control Tower** — Manually created OUs can't be managed by AWS Transform. Use `EnableBaseline` API to register them.

5. **MGN agent creates a NEW source server** — When installing the agent on a fresh instance (not the original imported server), it registers as a new source server with a different ID.

6. **Disable right-sizing for cost control** — Set `targetInstanceTypeRightSizingMethod: NONE` if you want the launch template's instance type to be respected.

7. **The m5 conversion server is internal to MGN** — It's temporary (runs ~5 min), cannot be changed, and is separate from your final target instance.

---

## Final Architecture

```
Source: he-dev-app-01 (i-0244d3778f7b31463, t3.micro)
    ↓ MGN Replication Agent (continuous block-level replication)
Staging: Replication Server (t3.small) + EBS staging disks
    ↓ Snapshot → Conversion (m5, temporary) → Launch
Target: Test Instance (i-051daeb302b30ca4f, t3.micro) ✅
```

## Organization Structure (Final)

| OU | Registered | Accounts |
|----|-----------|----------|
| Security | ✅ CT-managed | Log Archive, Audit |
| Workloads | ✅ CT-managed | Workload (150897468843) |
| Sandbox | ✅ CT-managed | — |
| Infrastructure | ✅ CT-managed | — |
| Root | — | Management (114805761158) |
