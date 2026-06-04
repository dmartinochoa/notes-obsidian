# SCS-C03 Reinforcement Cheatsheet

Complement to `priority-review-cheatsheet.md`.

**Purpose**: lock in the 36 concepts from Mock #1 (June 4) that you got RIGHT. These are confirmed strengths — don't let them slip while patching weaknesses.

**Usage**: read on taper days, exam morning, or as a confidence pass after intense remediation sessions.

**Updated**: June 4, 2026 — built from Mock #1 (55% / 36-of-65 correct).

---

## Confirmed strong patterns by domain

### D4 IAM — 8/11 correct (73%) — STRONG

You consistently nail IAM mechanics. Keep these reflexes fresh:

#### Cross-account role assumption (Q18)
- **Source account**: principal needs `sts:AssumeRole` on the target role ARN
- **Target account**: trust policy must include the source account as Principal
- **MFA condition**: if trust policy has `aws:MultiFactorAuthPresent: true`, the caller MUST authenticate with MFA first or AssumeRole fails
- **Trust policy account mismatch**: if the trust policy lists the wrong account ID, no permission grant elsewhere will fix it — the trust policy is the entry door

#### IAM credential precedence (Q29)
**Order of credential resolution** (the chain that breaks role-based access):
1. Command-line options (`--profile`, `--region`)
2. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
3. AWS CLI credentials file (`~/.aws/credentials`) ← **takes precedence over instance role**
4. AWS CLI config file (`~/.aws/config`)
5. Container credentials (ECS task role)
6. **Instance profile (IAM role attached to EC2)** ← last resort

**The trap**: if someone has run `aws configure` on the instance, the credentials file overrides the IAM role you attached. Fix = delete the credentials file.

#### SCP behavior (Q23) — covered in primary cheatsheet
✅ Applies to ALL principals in member accounts (including root)
❌ Does NOT affect the management account
❌ Does NOT affect service-linked roles

#### IAM Access Analyzer policy generation (Q24)
**Pattern for least-privilege replacement of broad managed policies:**
1. Keep the broad managed policy attached (e.g., `AmazonEC2FullAccess`)
2. Enable CloudTrail for management events
3. Run the workload normally so CloudTrail captures actual API usage
4. Use IAM Access Analyzer to generate a policy from the CloudTrail activity for the role
5. Replace the broad policy with the generated tighter customer-managed policy

**Why this beats alternatives**: IAM Last Accessed Info gives only service-level granularity. Removing policies first breaks the workload before CloudTrail can capture the pattern.

#### Cross-account access for multi-account workforce (Q37)
**Pattern for letting developers access multiple AWS accounts without per-account IAM users:**
1. Create IAM roles in target accounts (QA, Prod) with `sts:AssumeRole` capability
2. Create corresponding IAM roles or grants in source account (Dev)
3. Trust policies allow cross-account role assumption
4. Developers authenticate once in Dev account, then use STS to switch
5. **No long-term access keys distributed**

**Anti-pattern**: shared IAM users with access keys distributed to developers. Violates accountability and least-privilege.

#### IAM policy with resource tags (Q44)
**Pattern**: restrict EC2 operations to instances with specific tags using `ec2:ResourceTag` condition:
```json
"Condition": {
  "StringEquals": {
    "ec2:ResourceTag/Project": "Alpha"
  }
}
```
- Use `ec2:ResourceTag` (NOT `ec2:CreateTags` — that's an action, not a condition)
- Use `ec2:ResourceTag` (NOT `aws:sourceVPC` — that's for endpoint policies)

#### Access key compromise response (Q4) — also in primary cheatsheet
1. Rotate/disable the leaked key
2. Review CloudTrail logs in ALL regions
3. Delete any unauthorized resources

#### SCP for CloudTrail protection (Q3)
**Pattern**: when developers get root in their own dev accounts but you need to prevent them from disabling org-wide CloudTrail → attach SCP at the OU level with `Deny` on `cloudtrail:StopLogging`, `cloudtrail:DeleteTrail`, `cloudtrail:UpdateTrail`, `cloudtrail:PutEventSelectors`.

Why this works: SCPs apply to member account root, so even the developer-root can't disable logging.

---

### D1 Detection — 8/11 correct (73%) — STRONG

#### CloudWatch unified agent + metric filters (Q32)
**Pattern for scalable log analysis on EC2:**
1. Install unified CloudWatch agent on EC2
2. Configure agent to push application logs to CloudWatch Logs
3. Create metric filters that turn log patterns into metrics
4. Set CloudWatch Alarms on those metrics → SNS notifications

**The metric filter operates only on CW Logs** — for S3-stored logs, use Athena + Lambda + PutMetricData instead.

#### CloudTrail metric filter for API anomaly detection (Q45)
**Pattern**: detect spikes in failed API calls via CloudWatch:
1. CloudTrail logs to CloudWatch Logs
2. Create metric filter matching error patterns (`{ $.errorCode = "*UnauthorizedOperation*" }`)
3. CloudWatch Alarm on metric rate → SNS to security team

**Bonus**: CloudTrail Insights also flags unusual `write` API activity automatically.

#### WAF logging architecture (Q30)
**Correct path for WAF logs analysis:**
WAF → Kinesis Data Firehose → S3 → Athena query → QuickSight dashboards

Logging destinations supported by WAF:
1. CloudWatch Logs
2. Amazon S3
3. **Kinesis Data Firehose** (for scalable, real-time)

NOT: CloudTrail (CloudTrail captures management events, not data plane)

#### Detective + GuardDuty for root cause investigation (Q36)
- **GuardDuty**: detects threats, generates findings
- **Detective**: pivots from a finding to investigation — auto-aggregates VPC Flow Logs, CloudTrail, EKS audit, GuardDuty findings into a linked graph
- **Prerequisite**: Detective requires GuardDuty enabled for at least 48 hours

**For root cause questions**: Detective beats Security Hub because Detective auto-builds investigation context. Security Hub is for aggregation/posture, not investigation.

#### AWS Config managed rule + SSM remediation (Q52)
**Pattern for auto-remediation:**
1. AWS Config rule (e.g., `cloudtrail-enabled`) detects non-compliance
2. Config triggers SSM Automation document
3. SSM document executes the remediation (e.g., re-enable CloudTrail)

**This is the canonical "detect-and-remediate" pattern.** Use over Lambda when AWS-provided automation documents exist.

#### CloudTrail review options (Q42) — 3 ways to audit security group changes
1. **CloudTrail Event history** (last 90 days, in-console)
2. **CloudTrail + Athena** (S3-stored logs, SQL queries)
3. **AWS Config configuration history** (must have Config recorder on)

NOT: AppConfig (that's for application configs, not AWS resources)

#### WAF custom rules + managed rules combo (Q62)
**Pattern for User-Agent filtering:**
- Block requests WITH specific User-Agent → **custom rule** (regex/string match)
- Block requests WITHOUT User-Agent header → **AWS Managed Rules** (`NoUserAgent_HEADER` from Core rule set, OR `SignalNonBrowserUserAgent` from Bot Control)

Both can coexist in the same web ACL.

#### Security monitoring sequence (Q39)
**Canonical security ops sequence:**
1. Establish baseline VISIBILITY (Flow Logs, CloudTrail, CW Logs Insights)
2. DETECT early indicators (failed logins, privileged anomalies)
3. ANALYZE outbound traffic for exfiltration
4. CORRELATE findings (GuardDuty + Security Hub)
5. RESPOND (auto-isolation only AFTER correlation)

**Anti-pattern**: starting with isolation/response before visibility is established.

---

### D6 Governance — 5/6 correct (83%) — STRONGEST

#### Patch Manager for software updates (Q22)
**The right SSM capability for the job:**
- **Patch Manager** — automated patch scanning + installation + compliance reports
- NOT Change Manager (that's for change approval workflows)
- NOT "Version Manager" (made-up name)
- NOT Inspector (vulnerability scanning, not patching)

**Pattern**: schedule via maintenance window → Patch Manager scans + installs → compliance reports to S3.

#### S3 Object Lock for WORM compliance (Q25)
**For "data cannot be deleted until retention expires":**
- **S3 Object Lock** (Compliance mode = no one can delete, including root)
- NOT MFA Delete (just an extra layer, can be disabled)
- NOT Glacier Vault Lock (that's for Glacier vaults, not S3 standard)
- NOT CRR (replication doesn't prevent deletion)

#### AWS Config aggregator with delegated admin (Q56)
**Pattern**: instead of sharing management account credentials, use a delegated admin account for AWS Config aggregator:
- Aggregator collects Config + compliance data across accounts/regions
- Delegated admin can aggregate org-wide without mgmt account access
- Eliminates need to share mgmt account credentials

#### Macie + Security Hub + Athena combo (Q49)
**For sensitive data security architecture:**
- **Macie**: discover + classify + protect S3 sensitive data
- **Security Hub**: aggregate alerts from all sources (GuardDuty, Macie, Inspector) + dashboard
- **Athena**: ad-hoc SQL queries on S3-stored data

This trio is the AWS-native answer to "comprehensive data security platform."

---

### D5 Data Protection — 7/15 correct (47%) — WEAK BUT THESE 7 SOLID

#### KMS pending deletion recovery (Q14)
- Schedule deletion → key enters `PendingDeletion` (7-30 day window, default 30)
- During the window: `CancelKeyDeletion` to recover
- **After the window ends: key is GONE.** AWS Support cannot recover.

#### KMS imported key material rotation (Q1)
**The manual rotation pattern for imported key material:**
1. You CANNOT auto-rotate imported key material
2. You CANNOT change a key's imported material (it's permanently bound)
3. You CAN re-import the SAME material (e.g., to extend expiration)

**To rotate**: 
1. Create a NEW CMK
2. Import new key material into the new CMK
3. **Point the alias** from the old CMK to the new CMK
4. Applications using the alias automatically use the new key

Why alias-based: aliases abstract the underlying key ID. Apps reference `alias/myKey` and don't break when the underlying key changes.

#### WAF geo match + IP set combo (Q19)
**For "block country X but allow company office in country X":**
- WAF geo match statement → block the country
- WAF IP set statement → allow specific office IPs
- Combine with priority order — IP set match (allow) BEFORE geo match (block)

#### Encrypted RDS snapshot sharing cross-account (Q20)
**The pattern for sharing encrypted RDS data with auditor:**
1. Create snapshot encrypted with customer-managed KMS key
2. **Share both the snapshot AND the KMS key** with the auditor's AWS account
3. Auditor can copy the snapshot and restore as their own DB

**Why customer-managed CMK**: you cannot share snapshots encrypted with the default `aws/rds` key.

#### Secrets Manager multi-region replication (Q34)
**Pattern for DR/multi-region resilience:**
1. Create secret in primary region with AWS managed KMS key
2. Enable secret replication to secondary region
3. Secrets Manager auto-provisions an AWS managed KMS key in the secondary region
4. Applications in either region decrypt locally (no cross-region calls)

**Survivability**: if primary region fails, secondary region apps still retrieve + decrypt locally.

#### Security group inbound rule sources (Q57)
**Valid SG inbound rule sources:**
- IP address (single or CIDR)
- Range of IPs in CIDR notation
- Another security group ID (same VPC, or peer VPC with constraints)
- AWS prefix lists (managed or customer)

**INVALID**: Internet Gateway ID. IGW is not a source/destination construct — it's a routing target.

#### aws:PrincipalOrgID for KMS cross-account restriction (Q65)
**Pattern**: restrict KMS key usage to your AWS Organization without listing every account:
```json
"Condition": {
  "StringEquals": {
    "aws:PrincipalOrgID": "o-xxxxxxxx"
  }
}
```
- Use in KMS key policy `Principal` element
- Automatically updates as accounts are added/removed from the org
- More elegant than listing all account IDs

**Note**: this is for keys shared within an Org. For service principals (e.g., CloudTrail writing to S3), use `aws:PrincipalIsAWSService` instead.

---

### D3 Infrastructure Security — 7/17 correct (41%) — VERY WEAK BUT THESE 7 SOLID

#### EC2 + Inspector for vulnerability scanning (Q8)
**For "find software vulnerabilities on EC2 + harden network":**
- **Configure SGs** to allow only necessary ports
- **Use Amazon Inspector** for periodic vulnerability scans

**NOT**: Systems Manager directly (Inspector USES SSM Agent for data collection, but SSM itself doesn't scan vulnerabilities).
**NOT**: ACM SSL/TLS certs on EC2 (ACM-issued certs aren't installable on EC2 except for Nitro Enclaves).

#### GuardDuty threat lists in multi-account (Q9)
**Pattern for sharing threat IP lists across accounts:**
1. Designate a GuardDuty administrator account
2. Member accounts join (via invitation or Organizations)
3. **Upload threat list to S3 in admin account**
4. Admin account configures GuardDuty to reference the S3 threat list
5. Admin propagates to member accounts

**Constraint**: only the admin account can manage threat lists in a multi-account setup.

#### Security group rules for 3-tier app (Q12)
**Memorize this exact pattern:**
| Tier | Inbound source | Port |
|---|---|---|
| **ALB SG** | `0.0.0.0/0` | **443** (HTTPS) |
| **EC2 SG** | **ALB SG** | **80** (HTTP, terminated TLS) |
| **RDS SG** | **EC2 SG** | **5432** (PostgreSQL) or **3306** (MySQL) |

**Direction**: traffic flows DOWN. Each tier's inbound rule references the tier above.

**Common trap**: outbound rules on EC2 for response traffic — NOT needed. Security groups are stateful; return traffic is automatic.

#### SG connection tracking for instance isolation (Q43)
**Subtle but exam-relevant:**
When you change an SG rule, existing tracked connections AREN'T immediately interrupted (until they timeout). To force immediate isolation:
1. Identify the SG of the instance
2. Delete all existing rules
3. Add a single rule `0.0.0.0/0 (0-65535)` for ALL traffic in both inbound and outbound (converts existing connections to UNTRACKED)
4. Remove the `0.0.0.0/0 (0-65535)` rules to terminate all connections immediately

**Why**: untracked connections terminate immediately when their allow rule is removed. Tracked connections continue until timeout.

#### Shield Advanced CloudWatch alarms for DDoS (Q48)
**For DDoS attack alerting:**
- Set CloudWatch alarm on **Shield Advanced metrics** (`DDoSDetected`)
- NOT Macie (data security, not DDoS)
- NOT Inspector (vulnerabilities, not DDoS)
- NOT Firewall Manager (management, doesn't detect attacks)

#### CloudFront features (Q50)
**3 advanced CloudFront capabilities:**
1. **Multiple origins by content type**: dynamic path → ALB, static path → S3 (cache behaviors route by URL pattern)
2. **Origin group for failover**: primary + secondary origins, auto-failover on specific HTTP error codes
3. **Field-level encryption**: encrypt up to 10 specific fields (e.g., credit card #) at edge, decrypt only at backend

**NOT**: geo-restriction for HA (that's for content blocking)
**NOT**: routing by price class (price class is about edge location selection)
**NOT**: KMS for sensitive field encryption (use field-level encryption)

#### WAF geo match for country-level blocking (Q61)
**For "block all traffic except from specific country":**
- WAF web ACL on ALB
- Geo match rule with Deny + countries NOT in allow list
- WAF can attach to CloudFront / ALB / API GW / AppSync
- WAF CANNOT attach to: NLB, ASG, EC2 directly, NACLs

---

### D2 Incident Response — 1/5 correct (20%) — VERY WEAK BUT ONE STRONG

#### CloudTrail bucket prefix configuration (Q7)
**For "add prefix to CloudTrail logs" error "There is a problem with the bucket policy":**
1. Update the S3 bucket policy to grant CloudTrail permission to write under the new prefix
2. Update the CloudTrail trail to use the matching prefix

Both must agree — bucket policy resource ARN must match the prefix configured in CloudTrail.

---

## High-leverage patterns to keep WARM (small adjustments away from your weaknesses)

These you got right but they're adjacent to your weakest areas. Keep them fresh so they don't slip:

### KMS patterns you nailed (but adjacent to misses)
- **Q14 CancelKeyDeletion** — keep this fresh since you missed multiple KMS questions (Q5/Q15/Q31/Q33/Q55/Q58)
- **Q1 Imported key alias re-pointing** — same KMS depth concern
- **Q20 Encrypted snapshot + share CMK** — same cross-account KMS pattern that hurt you in Q58
- **Q65 aws:PrincipalOrgID in key policy** — KMS condition keys are an exam hot spot

### WAF patterns you nailed (but adjacent to misses)
- **Q19, Q61, Q62 WAF geo/IP/managed rules** — keep fresh since you missed several WAF architecture questions
- **Q30 WAF → Firehose → S3 architecture** — exactly what you needed for Q41 but missed there

### GuardDuty patterns you nailed (but adjacent to misses)
- **Q9 GuardDuty admin + member accounts** — same service area where you missed Q38, Q53, Q63

---

## Quick-recall tables to skim on exam morning

### CloudTrail bucket policy actions (memorize)
```
✅ s3:PutObject       (write logs)
✅ s3:GetBucketAcl    (verify bucket ownership)
❌ s3:GetObject       (CloudTrail doesn't read)
❌ s3:GetObjectAcl    (irrelevant)
```

### Security group sources (Q57 reinforcement)
```
✅ IP address (single)
✅ CIDR range
✅ Another security group ID
✅ AWS prefix list
❌ Internet Gateway ID
❌ Route table ID
❌ NAT Gateway ID
❌ VPC ID alone
```

### Service Catalog constraint types
```
Launch constraint    → IAM role used during provisioning (Q46 — you missed this!)
Notification        → SNS topic for events
Tag update          → control end-user tag changes
Template            → narrow CloudFormation parameter options
Stack set           → multi-account deployment
```

### Detective vs Security Hub vs GuardDuty (Q36 reinforcement)
| Service | Role |
|---|---|
| **GuardDuty** | Detect threats → produce findings |
| **Security Hub** | Aggregate findings from all sources → posture + dashboard |
| **Detective** | Investigate a finding → auto-graph context for root cause |

---

## Exam-day confidence reminders

You demonstrated solid mastery in:
- ✅ **IAM cross-account access patterns** (Q18, Q37, Q3)
- ✅ **IAM credential precedence chain** (Q29)
- ✅ **SCP behavior** (Q23, Q3)
- ✅ **CloudWatch + CloudTrail monitoring architecture** (Q30, Q32, Q42, Q45)
- ✅ **WAF geo/IP filtering patterns** (Q19, Q61, Q62)
- ✅ **Detective for investigation** (Q36)
- ✅ **AWS Config for compliance + remediation** (Q52, Q56)
- ✅ **3-tier SG architecture** (Q12)
- ✅ **CloudFront advanced features** (Q50)
- ✅ **Patch Manager for software updates** (Q22)
- ✅ **S3 Object Lock for WORM** (Q25)
- ✅ **KMS lifecycle (deletion, rotation)** (Q1, Q14)
- ✅ **Macie + Security Hub combo for data security** (Q49)
- ✅ **Secrets Manager multi-region replication** (Q34)

Don't second-guess yourself on these on exam day. First instinct wins.

---

## Reading discipline (added after Q35 reading miss)

Before answering ANY question, mentally circle these constraint words:
- "**Cannot**," "**must not**," "**without**," "**only**," "**all**," "**any**"
- "**Real-time**" vs "**scheduled**" vs "**historical**"
- "**Existing**" vs "**new**"
- "**Same account**" vs "**cross-account**"
- "**Least**" / "**most**" (operational overhead, efficient, restrictive, privilege)

If you answer something that violates an explicit constraint (like picking "terminate" when question says "cannot terminate"), that's a preventable miss worth 1 full point.

---

*Built June 4, 2026 — from Mock #1 (55%, 36/65 correct). Use as reinforcement complement to priority-review-cheatsheet.md. Read on taper days only — not during active remediation.*
