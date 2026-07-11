# AWS MGN Migration: Complete Sequential Steps

## Pre-Migration Setup

### Step 1: Fix AWS Control Tower Landing Zone

```bash
# 1.1 Check current Landing Zone status
aws controltower list-landing-zones --region us-east-1
aws controltower get-landing-zone --landing-zone-identifier arn:aws:controltower:us-east-1:114805761158:landingzone/18OXCQVPGCJUBAID --region us-east-1

# Status: FAILED, Version: 3.3, Drift: DRIFTED
```

```bash
# 1.2 Check organization structure
aws organizations list-roots
aws organizations list-organizational-units-for-parent --parent-id r-28h4
aws organizations list-accounts-for-parent --parent-id r-28h4

# Finding: All accounts at root, zero OUs
```

```bash
# 1.3 Create required OUs
aws organizations create-organizational-unit --parent-id r-28h4 --name "Security"
# Output: ou-28h4-l27f2l7r

aws organizations create-organizational-unit --parent-id r-28h4 --name "Sandbox"
# Output: ou-28h4-lln5in5d
```

```bash
# 1.4 Move service accounts to Security OU
aws organizations move-account --account-id 044789067646 --source-parent-id r-28h4 --destination-parent-id ou-28h4-l27f2l7r
aws organizations move-account --account-id 598361629577 --source-parent-id r-28h4 --destination-parent-id ou-28h4-l27f2l7r
aws organizations move-account --account-id 230352786706 --source-parent-id r-28h4 --destination-parent-id ou-28h4-lln5in5d
```

```bash
# 1.5 Update Landing Zone to version 4.0
aws controltower update-landing-zone \
    --landing-zone-identifier arn:aws:controltower:us-east-1:114805761158:landingzone/18OXCQVPGCJUBAID \
    --version 4.0 \
    --manifest file://manifest.json

# First attempt FAILED: "accounts cannot be placed in the root area" → Fixed in 1.3-1.4
# Second attempt FAILED: "Validation failed with 2 error(s)" on AWSControlTowerLoggingResources
```

### Step 2: Fix Orphaned S3 Buckets Blocking StackSet

```bash
# 2.1 Identify the failure
aws cloudformation list-stack-set-operation-results \
    --stack-set-name AWSControlTowerLoggingResources \
    --operation-id e8cdf4e8-4692-48ab-87fc-48e0961be6f4

# Error: StackSet trying to create S3 buckets that already exist
```

```bash
# 2.2 Assume role into Log Archive account
eval $(aws sts assume-role --role-arn arn:aws:iam::598361629577:role/AWSControlTowerExecution \
    --role-session-name fix-buckets \
    --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
    --output text | awk '{print "export AWS_ACCESS_KEY_ID="$1" AWS_SECRET_ACCESS_KEY="$2" AWS_SESSION_TOKEN="$3}')
```

```bash
# 2.3 Empty and delete the orphaned S3 buckets
aws s3 rm s3://aws-controltower-s3-access-logs-598361629577-us-east-1 --recursive
aws s3api list-object-versions --bucket aws-controltower-s3-access-logs-598361629577-us-east-1 \
    --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json > /tmp/del.json
aws s3api delete-objects --bucket aws-controltower-s3-access-logs-598361629577-us-east-1 --delete file:///tmp/del.json

aws s3api delete-bucket --bucket aws-controltower-logs-598361629577-us-east-1
aws s3api delete-bucket --bucket aws-controltower-s3-access-logs-598361629577-us-east-1
```

```bash
# 2.4 Clean up failed StackSet instance
aws cloudformation delete-stack-instances \
    --stack-set-name AWSControlTowerLoggingResources \
    --accounts 598361629577 \
    --regions us-east-1 \
    --retain-stacks
```

```bash
# 2.5 Retry Landing Zone update
aws controltower update-landing-zone \
    --landing-zone-identifier arn:aws:controltower:us-east-1:114805761158:landingzone/18OXCQVPGCJUBAID \
    --version 4.0 \
    --manifest file://manifest.json

# Operation ID: e727008d-247f-474b-a345-d6e41f3a7d3b
# Status: SUCCEEDED (took ~21 minutes)
```

### Step 3: Fix Sandbox OU for AWS Transform

```bash
# 3.1 Remove suspended account and delete unmanaged Sandbox OU
aws organizations move-account --account-id 230352786706 --source-parent-id ou-28h4-lln5in5d --destination-parent-id r-28h4
aws organizations delete-organizational-unit --organizational-unit-id ou-28h4-lln5in5d
```

```bash
# 3.2 Create new Sandbox OU
aws organizations create-organizational-unit --parent-id r-28h4 --name "Sandbox"
# Output: ou-28h4-umqxvvwg
```

```bash
# 3.3 Register Sandbox OU with Control Tower (enable baseline)
aws controltower enable-baseline \
    --baseline-identifier arn:aws:controltower:us-east-1::baseline/17BSJV3IGJ2QSGA2 \
    --baseline-version 5.0 \
    --target-identifier arn:aws:organizations::114805761158:ou/o-ei5z31f63f/ou-28h4-umqxvvwg \
    --parameters '[{"key":"IdentityCenterEnabledBaselineArn","value":"arn:aws:controltower:us-east-1:114805761158:enabledbaseline/XAET31Q9MFI5K2LYN"}]'

# Status: SUCCEEDED
```

### Step 4: Create Workload Account

```bash
# 4.1 Create account
aws organizations create-account \
    --email ramandeep.chandna+workload@gmail.com \
    --account-name Workload \
    --role-name AWSControlTowerExecution

# Account ID: 150897468843
```

```bash
# 4.2 Move to Workloads OU (Control Tower managed)
aws organizations move-account \
    --account-id 150897468843 \
    --source-parent-id r-28h4 \
    --destination-parent-id ou-28h4-b1nastew
```

---

## Migration Execution

### Step 5: Create Dummy Source Server for MGN Testing

```bash
# 5.1 Launch EC2 instance as the source server (simulating on-prem he-dev-app-01)
aws ec2 run-instances \
    --image-id ami-0df80e66b6b8a0056 \
    --instance-type t3.micro \
    --count 1 \
    --network-interfaces DeviceIndex=0,SubnetId=subnet-040f98af126cd1889,AssociatePublicIpAddress=true \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=he-dev-app-01},{Key=Purpose,Value=MGN-Migration-Test}]' \
    --user-data IyEvYmluL2Jhc2gKaG9zdG5hbWVjdGwgc2V0LWhvc3RuYW1lIGhlLWRldi1hcHAtMDEK

# Instance ID: i-0244d3778f7b31463
# Public IP: 52.90.140.133
# SSM Profile: AmazonSSMRoleForInstancesQuickSetup (pre-attached)
```

```bash
# 5.2 Verify SSM connectivity
aws ssm describe-instance-information \
    --filters "Key=InstanceIds,Values=i-0244d3778f7b31463"

# PingStatus: Online
```

### Step 6: Install MGN Replication Agent

```bash
# 6.1 Connect to the instance via SSM
aws ssm start-session --target i-0244d3778f7b31463 --region us-east-1

# 6.2 Download and run the MGN agent installer (run inside the instance)
sudo wget -O ./aws-replication-installer-init \
    https://aws-application-migration-service-us-east-1.s3.us-east-1.amazonaws.com/latest/linux/aws-replication-installer-init
sudo chmod +x aws-replication-installer-init
sudo ./aws-replication-installer-init --region us-east-1

# Agent registered as new source server: s-38759b0083f7eed55
# (Note: Different from the vCenter-imported s-31dd971512e7520d0)
```

### Step 7: Fix Replication Configuration (Staging Subnet)

```bash
# 7.1 Check replication config — found non-existent subnet
aws mgn get-replication-configuration --source-server-id s-38759b0083f7eed55
# stagingAreaSubnetId: subnet-0704f97e7910c9269 ← DOES NOT EXIST

# 7.2 Update to valid subnet
aws mgn update-replication-configuration \
    --source-server-id s-38759b0083f7eed55 \
    --staging-area-subnet-id subnet-040f98af126cd1889 \
    --associate-default-security-group \
    --replication-servers-security-groups-ids '[]' \
    --replication-server-instance-type t3.small \
    --use-dedicated-replication-server false \
    --default-large-staging-disk-type GP3 \
    --ebs-encryption DEFAULT \
    --bandwidth-throttling 0 \
    --data-plane-routing PUBLIC_IP \
    --create-public-ip
```

### Step 8: Wait for Replication to Complete

```bash
# 8.1 Monitor replication progress
aws mgn describe-source-servers \
    --filters '{"sourceServerIDs":["s-38759b0083f7eed55"]}' \
    --query 'items[0].{state:lifeCycle.state,replication:dataReplicationInfo.dataReplicationState,disk:dataReplicationInfo.replicatedDisks[0]}'

# Progress:
# PENDING_INSTALLATION → NOT_READY (INITIATING) → NOT_READY (INITIAL_SYNC) → READY_FOR_TEST (CONTINUOUS)
#
# Final state:
# lifecycle_state: READY_FOR_TEST
# replication_state: CONTINUOUS
# disk: 8.59 GB / 8.59 GB (100%)
# lag: 0
```

### Step 9: Fix EC2 Launch Template (Subnet + Security Group)

```bash
# 9.1 Check launch template — found stale references
aws ec2 describe-launch-template-versions \
    --launch-template-id lt-073a4879da2602f77 \
    --versions '$Default'

# NetworkInterfaces.SubnetId: subnet-0704f97e7910c9269 ← DOES NOT EXIST
# NetworkInterfaces.Groups: sg-0ed55d56d2a253c82 ← DOES NOT EXIST

# 9.2 Create corrected launch template version
aws ec2 create-launch-template-version \
    --launch-template-id lt-073a4879da2602f77 \
    --source-version 2 \
    --launch-template-data '{
        "NetworkInterfaces": [{
            "AssociatePublicIpAddress": true,
            "DeleteOnTermination": true,
            "DeviceIndex": 0,
            "Groups": ["sg-7302c328"],
            "SubnetId": "subnet-040f98af126cd1889"
        }]
    }'

# 9.3 Set as default version
aws ec2 modify-launch-template \
    --launch-template-id lt-073a4879da2602f77 \
    --default-version 6
```

### Step 10: Configure Instance Type (Disable Right-Sizing)

```bash
# 10.1 Disable MGN right-sizing (prevents override to c5.large)
aws mgn update-launch-configuration \
    --source-server-id s-38759b0083f7eed55 \
    --target-instance-type-right-sizing-method NONE

# 10.2 Set instance type to t3.micro in launch template
aws ec2 create-launch-template-version \
    --launch-template-id lt-073a4879da2602f77 \
    --source-version 6 \
    --launch-template-data '{"InstanceType": "t3.micro"}'

aws ec2 modify-launch-template \
    --launch-template-id lt-073a4879da2602f77 \
    --default-version 8
```

### Step 11: Launch Test Migration

```bash
# 11.1 Start test
aws mgn start-test --source-server-ids s-38759b0083f7eed55

# Job ID: mgnjob-3b715a773b82e978b
```

```bash
# 11.2 Monitor job progress
aws mgn describe-job-log-items --job-id mgnjob-3b715a773b82e978b

# Events in sequence:
# 1. JOB_START
# 2. CLEANUP_START (terminate previous test instance)
# 3. CLEANUP_END
# 4. SNAPSHOT_START (create EBS snapshot from replication disk)
# 5. SNAPSHOT_END
# 6. CONVERSION_START (launch m5 conversion server)
# 7. CONVERSION_END (inject drivers, modify boot config)
# 8. LAUNCH_START (launch final target instance)
# 9. LAUNCH_END ✅
# 10. JOB_END ✅
```

```bash
# 11.3 Verify test instance
aws mgn describe-jobs \
    --filters '{"jobIDs":["mgnjob-3b715a773b82e978b"]}' \
    --query 'items[0].{status:status,launchStatus:participatingServers[0].launchStatus,ec2Instance:participatingServers[0].launchedEc2InstanceID}'

# status: COMPLETED
# launchStatus: LAUNCHED
# ec2Instance: i-051daeb302b30ca4f (t3.micro) ✅
```

---

## Post-Test Steps (Next Actions)

### Step 12: Validate Test Instance

```bash
# 12.1 Check instance is running
aws ec2 describe-instances --instance-ids i-051daeb302b30ca4f \
    --query 'Reservations[0].Instances[0].{State:State.Name,Type:InstanceType,IP:PublicIpAddress}'

# 12.2 Connect and validate OS, hostname, services
aws ssm start-session --target i-051daeb302b30ca4f
```

### Step 13: Mark Ready for Cutover

```bash
# After validation passes
aws mgn change-server-life-cycle-state \
    --source-server-id s-38759b0083f7eed55 \
    --life-cycle '{"state": "READY_FOR_CUTOVER"}'
```

### Step 14: Perform Cutover

```bash
# Launch final production instance
aws mgn start-cutover --source-server-ids s-38759b0083f7eed55
```

### Step 15: Finalize Cutover

```bash
# After verifying cutover instance works
aws mgn finalize-cutover --source-server-id s-38759b0083f7eed55

# This:
# - Stops replication
# - Terminates the replication server
# - Cleans up staging disks
# - Marks migration as COMPLETE
```

---

## Timeline Summary

| Time | Action | Result |
|------|--------|--------|
| T+0 | Checked Landing Zone status | FAILED v3.3 |
| T+2 min | Created OUs + moved accounts | ✅ |
| T+3 min | Landing Zone update attempt 1 | ❌ Accounts at root |
| T+5 min | Landing Zone update attempt 2 | ❌ Orphaned S3 buckets |
| T+10 min | Deleted orphaned S3 buckets | ✅ |
| T+12 min | Landing Zone update attempt 3 | ✅ SUCCEEDED (21 min) |
| T+35 min | Fixed Sandbox OU + registered with CT | ✅ |
| T+38 min | Created Workload account | ✅ |
| T+40 min | Launched dummy source server | ✅ |
| T+42 min | Installed MGN replication agent | ✅ |
| T+45 min | Fixed replication config (subnet) | ✅ |
| T+55 min | Replication completed (8 GB synced) | ✅ |
| T+58 min | Test launch attempt 1 | ❌ Invalid subnet |
| T+60 min | Fixed launch template subnet | ✅ |
| T+63 min | Test launch attempt 2 | ❌ Invalid security group |
| T+65 min | Fixed launch template SG | ✅ |
| T+68 min | Test launch attempt 3 | ✅ c5.large launched |
| T+72 min | Changed to t3.micro + disabled right-sizing | ✅ |
| T+80 min | Test launch attempt 4 (final) | ✅ t3.micro launched |

---

## Resources Created

| Resource | ID | Purpose |
|----------|-----|---------|
| Security OU | ou-28h4-l27f2l7r | Audit + Log Archive accounts |
| Sandbox OU | ou-28h4-umqxvvwg | CT-registered, for Transform SCPs |
| Workloads OU | ou-28h4-b1nastew | Workload accounts |
| Workload Account | 150897468843 | New governed account |
| Source Instance | i-0244d3778f7b31463 | Dummy he-dev-app-01 |
| Test Instance | i-051daeb302b30ca4f | Migrated t3.micro |
| MGN Source Server | s-38759b0083f7eed55 | Agent-registered server |
| Launch Template | lt-073a4879da2602f77 (v8) | t3.micro, valid subnet+SG |
