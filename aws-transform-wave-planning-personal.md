# How I Used AWS Transform to Plan My Migration Waves — My Experience

## Setting the Stage

So here's the situation — I had 82 VMware virtual machines sitting in my on-premises vCenter that needed to move to AWS. Mix of RHEL (66), Windows (11), Linux (4), spread across prod, alpha, uat, dev, beta environments. Doing this manually? That's weeks of spreadsheet work, dependency mapping, and back-and-forth with app owners.

Instead, I fired up **AWS Transform** and let the AI do the heavy lifting.

## Step 1: Getting Started with AWS Transform

First thing I did was open the AWS Transform console (`transform.us-east-1.on.aws`). It greeted me with options like "Assess Apps", "Migrate Workloads", "Assess Infrastructure" — I went straight to **Migrate Workloads** since I knew what I wanted.

I created a workspace called **Migration-Workspace** and then created a new job. When it asked what type of migration, I picked **End-to-End Migration** because I wanted the full package — planning, landing zone, network, and MGN import all in one shot.

Transform immediately laid out a 5-step job plan for me:
1. Build migration plan
2. Connect target AWS account
3. Build landing zone
4. Migrate Network
5. Configure migration defaults

Clean. Structured. No guessing what comes next.

## Step 2: Feeding It My Server Inventory

Transform asked for my server inventory. It can take pretty much anything — RVTools exports, ServiceNow CMDB dumps, Cloudamize reports, or even a plain CSV with hostnames and IPs.

I had an **RVTools export** from my vCenter (66.35 KB ZIP with vInfo and vDisk tabs). Dragged it into the chat, typed "Here is RVTools export" and hit send.

Within seconds, Transform processed it:
- Extracted 164 rows from both vInfo files
- Detected duplicates (82 unique VMs × 2 source files)
- Deduplicated down to **82 servers**
- Classified by OS: RHEL 66, Windows 11, Linux 4, Other 1
- Identified environments from hostnames: prod (24), alpha (16), uat (14), dev (11), beta (10), unclassified (7)

No manual tagging. No CSV cleanup. It figured out the environment classification from naming conventions in my hostnames. That alone saved me hours.

## Step 3: Configuring the Wave Plan

Transform then asked me a few planning questions:

**"How should servers be grouped into applications?"**
- Options: environment (default), os_type, auto
- I picked **environment** — makes sense for our team structure

**"Keep prod and non-prod in separate waves?"**
- I said **yes** — obviously don't want to mix production with dev in the same migration wave

**"Max servers per wave?"**
- Kept the default **100** — my environment is small enough

**"Migration strategy?"**
- Options: rehost (default), containerize, decide later
- I chose **rehost** — this was a lift-and-shift project, not a modernization exercise

You can also just say "use defaults" and it picks all the recommended options. I appreciated having the choice though.

## Step 4: The Wave Plan Output

This is where it got impressive. Transform analyzed all 82 servers and organized them into **6 applications** with clear priority ordering:

| Priority | Application | Environment | Servers |
|----------|-------------|-------------|---------|
| 1 | Dev-Application | dev | 11 |
| 2 | Alpha-Application | alpha | 16 |
| 3 | Beta-Application | beta | 10 |
| 4 | Uat-Application | uat | 14 |
| 5 | Unclassified-Application | unclassified | 7 |
| 6 | Prod-Application | prod | 24 |

**100% server coverage** — every single server assigned. Non-prod first, prod last. Exactly what you'd want.

The logic here is solid: you migrate dev first (low risk), learn from any issues, then alpha, beta, uat, and finally prod once you've validated the process multiple times.

## Step 5: The R-Strategy Report (This Was the Best Part)

I clicked "Generate R-strategy report" and Transform produced a full **7Rs classification**:

- **Rehost (52 servers):** Straight lift-and-shift to EC2
- **Replatform (18 servers):** Flagged as database servers → candidates for Amazon RDS
- **Retire (12 servers):** Powered-off or idle VMs → candidates for decommissioning

The key insight it gave me: *"63% of servers are recommended for Rehost, but 18 servers show managed-service signals — worth reviewing whether those should Replatform instead of defaulting to EC2."*

That's 18 servers I would have blindly migrated to EC2 if I'd done this manually. Moving them to RDS instead could save 40-60% on operational overhead.

And those 12 retire candidates? Powered-off VMs that nobody's used in months. Why migrate dead weight?

The interactive HTML report it generated had everything — filterable tables, criticality matrix, confidence scores (average 57.3%), and per-server recommendations.

## Step 6: What Happened Next

After wave planning, Transform moved to the next steps in the job plan:
- Connected my target AWS account
- Built the landing zone (Control Tower v4.0 with proper OUs)
- Deployed the network (VPC, subnets, security groups)
- Generated the MGN import file with EC2 recommendations for each server

The whole planning phase — from uploading RVTools to having a complete wave plan with 7Rs analysis — took about **30 minutes**. That includes me reading the output, asking follow-up questions, and generating the R-strategy report.

## My Takeaway

What normally takes a migration team 2-3 weeks of spreadsheet work, dependency mapping workshops, and architecture review meetings was done in a single conversation. Not perfect — I still needed to validate the retire candidates with app owners and review the replatform recommendations — but it gave me a solid 80% starting point that I could refine.

The fact that it handled deduplication, environment classification, and R-strategy analysis automatically means I could focus on the **decisions** rather than the **data wrangling**. That's the real value of AWS Transform.

---

*Next up: How the Landing Zone deployment went sideways (and how I fixed it with some creative StackSet surgery)...*
