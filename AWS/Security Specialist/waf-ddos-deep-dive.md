# WAF / DDoS / Threat Detection Deep-Dive Reference

**Built June 5, 2026** for SCS-C03 exam (June 10/17).

**Purpose**: single source of truth for network protection, DDoS mitigation, and threat detection on the exam. Closes the 10-question gap from Mock #1:
- WAF architecture (Q11, Q41, Q51)
- DDoS resilience (Q17, Q27, Q28)
- CloudFront origin restriction (Q47)
- GuardDuty configuration (Q38, Q53, Q63)

**Read order**: skim cover-to-cover once, then drill "Question pattern recognition" at the end.

---

## Mental model: the network protection stack

AWS network protection is layered. Each service does ONE specific thing — pick the right tool for the layer of attack you're defending against.

```
┌─────────────────────────────────────────────────────────────────┐
│ Application Layer (L7)                                           │
│   AWS WAF — HTTP/HTTPS attacks (SQLi, XSS, geo, rate limiting)   │
│   Shield Advanced (L7 support) — escalation to AWS DRT team      │
├─────────────────────────────────────────────────────────────────┤
│ Network/Transport Layer (L3/L4)                                  │
│   Shield Standard — automatic DDoS mitigation (free, always on)  │
│   Shield Advanced — enhanced DDoS protection ($3K/month)         │
│   Network Firewall — DPI, IPS/IDS, stateful filtering            │
├─────────────────────────────────────────────────────────────────┤
│ Management Layer (cross-account)                                 │
│   Firewall Manager — centralizes WAF/Shield/NF/SG policies       │
│   (does NOT inspect traffic itself)                              │
├─────────────────────────────────────────────────────────────────┤
│ Detection (no blocking, just alerts)                             │
│   GuardDuty — threat detection from logs                         │
│   VPC Flow Logs — network metadata                               │
│   VPC Traffic Mirroring — full packet copy for IDS/forensics     │
└─────────────────────────────────────────────────────────────────┘
```

**Key principle**: WAF and Shield are for STOPPING attacks. GuardDuty is for DETECTING them. Different categories, different exam questions.

---

## Glossary (memorize these terms)

| Term | Definition |
|---|---|
| **Web ACL** | Container for WAF rules; attached to a protected resource |
| **Rule statement** | The match condition (IP, geo, regex, rate, managed rule, etc.) |
| **Rule action** | What WAF does on match: Allow / Block / Count / CAPTCHA / Challenge |
| **Managed rule group** | AWS-curated rules (Core, Bot Control, Known Bad Inputs, etc.) |
| **Rate-based rule** | Block requests when rate exceeds threshold (DDoS-like behavior) |
| **IP set** | Reusable list of IP addresses for match statements |
| **Regex pattern set** | Reusable regex patterns for match statements |
| **DRT** | DDoS Response Team — Shield Advanced support team |
| **OAI** | Origin Access Identity (legacy CloudFront → S3 restriction) |
| **OAC** | Origin Access Control (newer, preferred, more features) |
| **SLR** | Service-linked role |

---

## AWS WAF — deep dive

### What WAF IS

A **Layer 7 (HTTP/HTTPS) web application firewall**. Inspects every request to your protected resource and decides Allow/Block/Count/CAPTCHA based on rules.

### What WAF CAN protect (attachment points — MEMORIZE)

WAF web ACLs attach ONLY to these resource types:

| Resource | Supported? | Common use |
|---|---|---|
| **Amazon CloudFront distribution** | ✅ Yes (global) | Edge protection for CDN |
| **Application Load Balancer (ALB)** | ✅ Yes (regional) | Protect EC2/container workloads |
| **Amazon API Gateway** (REST API) | ✅ Yes (regional) | Protect REST APIs |
| **AWS AppSync** GraphQL API | ✅ Yes (regional) | Protect GraphQL endpoints |
| **Amazon Cognito User Pool** | ✅ Yes (regional) | Protect auth endpoints |
| **AWS App Runner** service | ✅ Yes (regional) | Protect containerized apps |
| **AWS Verified Access** instance | ✅ Yes (regional) | Identity-aware access |
| **Network Load Balancer (NLB)** | ❌ **NO** — NLB is L4, WAF is L7 |
| **EC2 instance directly** | ❌ **NO** — must front with CloudFront or ALB |
| **Auto Scaling Group (ASG)** | ❌ **NO** — same; need ALB in front |
| **Amazon S3 bucket** | ❌ **NO** — front with CloudFront, attach WAF to that |
| **DynamoDB** | ❌ **NO** — no public HTTP endpoint to protect |
| **RDS** | ❌ **NO** — not an HTTP service |
| **Route 53** | ❌ **NO** — DNS protection is Shield's job |

**Critical exam reflex**: "WAF on ASG" or "WAF on NLB" or "WAF on EC2" → **construction error**. Eliminate immediately.

### Web ACL anatomy

```
Web ACL
├── Default action (Allow OR Block — what happens when no rules match)
├── Rules (ordered list, evaluated in priority order)
│   ├── Rule 1: Priority 10, Statement={IPSet match}, Action=Block
│   ├── Rule 2: Priority 20, Statement={Geo match}, Action=Block
│   ├── Rule 3: Priority 30, Statement={Rate-based}, Action=Block (rate > 2000 req/5min)
│   └── Rule 4: Priority 40, Statement={Managed rule group AWSManagedRulesCommonRuleSet}, Action=Override-to-Count
├── Capacity (WCU — Web ACL Capacity Units, max 5000 per ACL by default)
└── Visibility (CloudWatch metrics + sampled requests)
```

### Rule statement types

| Statement type | What it matches |
|---|---|
| **IP set match** | Request source IP in a defined IP set |
| **Geographic match** | Request from a specific country |
| **String match** | Substring in header/URI/body/query/cookie |
| **Regex pattern set match** | Regex pattern in inspectable field |
| **Size constraint** | Body/header/URI size beyond a threshold |
| **SQL injection match** | SQLi patterns in inspectable field |
| **XSS match** | XSS patterns in inspectable field |
| **Rate-based** | Source IP exceeds N requests in 5-minute window |
| **Label match** | Match label added by prior rule (chaining) |
| **Managed rule group** | Pre-built rule set from AWS or AWS Marketplace |
| **Logical operators** | AND, OR, NOT to combine statements |

### Rule actions

| Action | What it does |
|---|---|
| **Allow** | Permit the request (overrides later rules) |
| **Block** | Reject with HTTP 403 (default) or custom response |
| **Count** | Log the match but don't act (useful for testing rules) |
| **CAPTCHA** | Challenge user with CAPTCHA (legitimate users solve, bots fail) |
| **Challenge** | Silent JavaScript challenge (transparent to humans, blocks simple bots) |

### Rate-based rules — the DDoS mitigation tool

- Tracks requests per source IP over a **5-minute rolling window**
- Triggers Block (or other action) when rate exceeds threshold
- Threshold range: 100 to 20,000,000 requests per 5 minutes
- Can be scoped by additional match conditions (e.g., "rate-limit only requests to /login")
- Useful for blocking brute-force attacks, scraping, simple DDoS

**Limitation**: 5-minute window means slower response to sudden spikes. For finer-grained, use rate-based with shorter aggregation key combinations.

### Managed rule groups (AWS-curated)

Use these for instant common-attack protection:

| Group | Purpose |
|---|---|
| **AWSManagedRulesCommonRuleSet (CRS)** | OWASP Top 10 baseline (SQLi, XSS, LFI, etc.) |
| **AWSManagedRulesKnownBadInputsRuleSet** | Blocks known malicious patterns (web shells, etc.) |
| **AWSManagedRulesSQLiRuleSet** | Focused SQL injection protection |
| **AWSManagedRulesLinuxRuleSet** | Linux-specific exploits |
| **AWSManagedRulesUnixRuleSet** | Unix-specific exploits |
| **AWSManagedRulesWindowsRuleSet** | Windows-specific exploits |
| **AWSManagedRulesPHPRuleSet** | PHP-specific exploits |
| **AWSManagedRulesWordPressRuleSet** | WordPress-specific exploits |
| **AWSManagedRulesAdminProtectionRuleSet** | Blocks /admin, /wp-admin URLs |
| **AWSManagedRulesAmazonIpReputationList** | Block known malicious IPs (AWS-curated) |
| **AWSManagedRulesAnonymousIpList** | Block TOR/VPN/proxy IPs |
| **AWSManagedRulesBotControlRuleSet** | Bot detection + management (paid) |
| **AWSManagedRulesATPRuleSet** | Account Takeover Prevention (paid) |
| **AWSManagedRulesACFPRuleSet** | Account Creation Fraud Prevention (paid) |

**Free vs paid**: most managed rules are free; Bot Control, ATP, and ACFP are AWS-charged extras.

### WAF logging destinations — CRITICAL (Q11 + Q41 pattern)

WAF can ship full request logs (every inspected request with which rule fired) to **THREE destinations**:

| Destination | When to use |
|---|---|
| **Amazon CloudWatch Logs** | Real-time visibility, CW Logs Insights queries, alerting via metric filters |
| **Amazon S3** | Long-term storage, cost-effective, query via Athena |
| **Amazon Kinesis Data Firehose** | Real-time streaming to S3 / Redshift / OpenSearch / Splunk / Datadog |

**NOT a destination**: ❌ CloudTrail. CloudTrail captures WAF MANAGEMENT API calls (CreateWebACL, UpdateRule), NOT inspected request logs. This is a recurring trap.

### Logging architecture for analytics (Q41 pattern)

For serverless WAF analytics dashboards:

```
WAF inspects requests
      ↓ logs every request
Kinesis Data Firehose (real-time stream)
      ↓ buffers + delivers
S3 (long-term storage, JSON format)
      ↓ schema discovery
AWS Glue Crawler → Glue Data Catalog
      ↓ SQL queries
Amazon Athena (serverless SQL on S3)
      ↓ data source
Amazon QuickSight (BI dashboards)
```

**Memorize this 5-component chain** for analytics questions:
**Firehose → S3 → Glue → Athena → QuickSight**

### What WAF logs contain

Every WAF log entry includes:
- Timestamp
- Source IP, country, headers, URI, query string, body (sampled)
- Which rule matched (or no match)
- Action taken (Allow/Block/Count)
- Labels added by rules
- HTTP response code

### CloudWatch metrics from WAF

WAF reports metrics to **CloudWatch (NOT CloudTrail)** at 1-minute granularity:
- `AllowedRequests`
- `BlockedRequests`
- `CountedRequests`
- `PassedRequests`
- `CaptchaRequests`
- `ChallengeRequests`

Use metrics for high-level monitoring, dashboards, and alarming. Use logs for per-request debugging and forensics.

### WAF Classic vs WAFv2

| Feature | WAF Classic (deprecated) | WAFv2 (current) |
|---|---|---|
| API | Old, separate per resource type | Unified API |
| Capacity | Different rules per region | Web ACL capacity units (WCU) |
| Rule reuse | Limited | Rule groups |
| Managed rules | Few | Many AWS + Marketplace |

**For exam**: assume WAFv2 unless explicitly noted. WAF Classic is rarely tested directly.

---

## AWS Shield — deep dive

### Shield Standard (free, automatic)

- **Automatically enabled** for all AWS customers
- Protects against **Layer 3/4 DDoS** attacks (SYN floods, UDP reflection, etc.)
- Always-on detection and inline mitigation
- **No configuration needed** — runs at AWS edge
- Specifically protects: Route 53, CloudFront, Global Accelerator (and indirectly anything behind them)
- No cost protection (DDoS scaling charges still apply to your resources)

### Shield Advanced ($3,000/month per organization)

Adds significant capabilities over Standard:

| Feature | Shield Standard | Shield Advanced |
|---|---|---|
| L3/L4 auto-mitigation | ✅ Yes | ✅ Yes (enhanced) |
| L7 detection | ❌ No | ✅ Yes (alerts via CloudWatch) |
| L7 auto-mitigation | ❌ No | ❌ No (manual via WAF rules) |
| DRT (DDoS Response Team) access | ❌ No | ✅ Yes — 24/7 expert support |
| Cost protection | ❌ No | ✅ Yes — refunds scaling charges from DDoS |
| Health-based detection | ❌ No | ✅ Yes (uses Route 53 health checks) |
| Real-time attack visibility | ❌ No | ✅ Yes (CloudWatch attack metrics) |
| WAF included | ❌ No | ✅ Yes — bundled at no extra charge |
| Firewall Manager included | ❌ No | ✅ Yes — bundled |
| Application acceleration | ❌ No | ✅ Global Accelerator included |
| Protected resource types | Limited | EIP, CloudFront, ALB, NLB, Global Accelerator, Route 53 hosted zone |

### Shield Advanced supported resources

Shield Advanced can be enabled per-resource on:
- Elastic IP addresses (great for EC2 protection)
- CloudFront distributions
- Application Load Balancers
- Network Load Balancers
- Global Accelerator accelerators
- Route 53 hosted zones

### How L7 attacks are mitigated with Shield Advanced

**Shield Advanced does NOT auto-block L7 attacks** (to avoid blocking legitimate traffic). Two response paths:

1. **You write WAF rules** based on detected attack patterns (CloudWatch alerts you)
2. **Contact AWS DRT** — they help craft mitigation rules during the attack

For **L7 DDoS** questions (Q28 pattern), the right answers are:
- "Create custom WAF rules in your web ACL"
- "Contact AWS Support if you're a Shield Advanced customer"

**NOT**: GuardDuty (just detection), CloudWatch alarms (just notification), SSM automation (no DDoS template exists).

### Cost protection — Shield Advanced's hidden gem

If DDoS causes your auto-scaling to spin up extra capacity (and bill you for it), Shield Advanced **refunds** the scaling charges. This applies to:
- EC2 (ASG scale-out)
- CloudFront (request fees)
- Route 53 (query fees)
- ALB/NLB (LCU charges)

This alone can pay for the $3K/month for sites that get attacked regularly.

---

## AWS Network Firewall — when to use vs WAF

### What Network Firewall IS

A **managed L3/L4/L7 stateful firewall** for VPC traffic. Sits inline in your VPC and inspects ALL traffic (not just HTTP).

### Network Firewall vs WAF comparison

| Aspect | Network Firewall | WAF |
|---|---|---|
| Layer | L3/L4/L7 | L7 (HTTP only) |
| Position | Inline in VPC (Gateway Load Balancer or as VPC endpoint) | Attached to CloudFront/ALB/API GW/AppSync |
| Use case | All VPC traffic filtering, IDS/IPS via Suricata | HTTP attack mitigation |
| Block protocols | TCP, UDP, ICMP, anything | HTTP/HTTPS only |
| Rule engine | Suricata-compatible (IDS rules) | AWS WAF rule language |
| Use cases | Egress filtering, deep packet inspection, network segmentation, threat hunting | Web app protection, geo/IP/rate filtering |
| Cost | Per endpoint hour + per GB processed | Per web ACL + per rule + per request |

### When to use Network Firewall

- **Egress filtering**: control what your workloads can reach (block crypto-mining domains, etc.)
- **Network segmentation**: protect VPC subnets from each other
- **IDS/IPS**: detect known malware signatures via Suricata rules
- **Compliance**: certain frameworks require deep packet inspection
- **Multi-protocol filtering**: SSH, RDP, custom protocols

### When to use WAF

- **HTTP/HTTPS web app protection**: SQLi, XSS, rate limiting, geo blocking
- **Public-facing endpoints**: ALB, API Gateway, CloudFront
- **Bot management**: AWS Managed Rules Bot Control

### When to use BOTH (defense in depth)

Common architecture:
- **WAF at edge** (CloudFront) — L7 attack filtering
- **Network Firewall** in VPC — egress filtering, internal traffic inspection
- **Security groups** at instance — last-mile L4 access control

### Network Firewall capabilities

- **Stateless rules**: drop/pass/forward based on 5-tuple (very fast)
- **Stateful rules**: TCP connection tracking, more sophisticated matching
- **Suricata rule support**: IDS/IPS rule format compatibility
- **TLS inspection**: decrypt + inspect HTTPS (requires private key access)
- **AWS-managed threat intel**: domain/IP reputation lists

---

## AWS Firewall Manager — the centralized POLICY MANAGER

### What Firewall Manager IS

**A management service** that centralizes the deployment of WAF, Shield, Network Firewall, security groups, and Route 53 Resolver DNS Firewall **policies across an entire AWS Organization**.

### What Firewall Manager IS NOT

**It does NOT inspect traffic.** It does not block, allow, or detect anything. It just deploys POLICY CONFIGURATIONS to other services.

This is a recurring trap on the exam.

### Firewall Manager policy types

| Policy type | What it deploys |
|---|---|
| **AWS WAF** | Web ACLs to in-scope CloudFront/ALB/API GW/AppSync resources |
| **AWS Shield Advanced** | Shield protections to in-scope EIP/CloudFront/ALB/NLB |
| **VPC Security Groups** | Standard SG rules across accounts (audit + remediation) |
| **Network Firewall** | Network Firewall endpoints + rule groups in VPCs |
| **Route 53 Resolver DNS Firewall** | DNS firewall rules in VPCs |
| **AWS Network ACL** | NACL rules across accounts |

### Auto-remediation toggle (Q51 pattern)

Firewall Manager has TWO toggles per policy:

| Toggle | What it does |
|---|---|
| **Auto-remediate any non-compliant resources** | Apply the policy to all in-scope resources automatically |
| **Replace existing web ACLs with FM-created ACLs** | Override any existing ACLs (vs only attaching to resources without one) |

**The Q51 trap**: if "auto-remediate" is OFF, the Firewall Manager-created web ACL exists but doesn't attach to anything. The simple fix is enabling auto-remediation.

### Firewall Manager prerequisites

- AWS Organizations enabled
- All-features mode (not consolidated billing only)
- AWS Config enabled in all member accounts
- Firewall Manager administrator account designated (delegated admin)
- AWS Resource Access Manager (for shared resources)

### Why use Firewall Manager

- **Org-wide consistency**: every account gets the same baseline protection
- **New accounts auto-onboarded**: policies apply to newly created accounts automatically
- **Single pane of glass**: see compliance across all accounts
- **Audit + remediation**: catch when someone modifies a managed resource

### Firewall Manager vs Organizations SCPs

| Concern | Firewall Manager | SCPs |
|---|---|---|
| Layer | Application/network protection | API authorization |
| What it does | Deploys protection policies | Restricts API actions |
| Inspection? | Manages other services that inspect | No (just blocks API calls) |
| Common use | "Every account must have WAF" | "Member accounts can't use us-east-1" |

---

## Amazon GuardDuty — threat detection deep dive

### What GuardDuty IS

A **threat detection service** that continuously analyzes log streams to identify malicious activity, compromised credentials, and unusual behavior. Generates **findings** (alerts) — does NOT block anything.

### What GuardDuty DOES NOT do

- ❌ Block traffic (that's WAF / Network Firewall / SG)
- ❌ Quarantine instances (your automation must do that based on findings)
- ❌ Revoke credentials (your automation must do that)
- ❌ Scan vulnerabilities (that's Inspector)
- ❌ Find sensitive data (that's Macie)

### GuardDuty data sources (memorize)

GuardDuty analyzes these data streams (most are enabled automatically):

| Data source | What it sees | Always on? |
|---|---|---|
| **VPC Flow Logs** | Network metadata (src/dst/port/protocol) | ✅ Yes — internal stream, independent of your Flow Logs |
| **DNS logs** | DNS queries from EC2 instances | ✅ Yes — IF using AWS DNS resolver (see below) |
| **CloudTrail management events** | API calls (account level) | ✅ Yes — internal stream |
| **CloudTrail S3 data events** | S3 object-level API calls | Opt-in (S3 Protection) |
| **EKS audit logs** | Kubernetes API server activity | Opt-in (EKS Protection) |
| **EKS runtime monitoring** | Container behavior (agent-based) | Opt-in |
| **EC2 runtime monitoring** | Process/network behavior on EC2 (agent-based) | Opt-in |
| **ECS runtime monitoring** | Container behavior (Fargate or EC2) | Opt-in |
| **RDS login activity** | Failed authentication, anomalous logins | Opt-in (RDS Protection) |
| **Lambda network activity** | Lambda function network behavior | Opt-in (Lambda Protection) |
| **EBS volume malware scan** | Snapshot-based malware detection | Opt-in (Malware Protection) |

### The DNS gotcha (Q53 pattern)

**GuardDuty DNS analysis works ONLY when you use the AWS DNS resolver** (the default for VPCs).

If your instances use:
- ✅ AWS DNS resolver (Route 53 Resolver, the default) → GuardDuty sees DNS
- ❌ On-premises AD DNS server → GuardDuty BLIND to DNS
- ❌ Google DNS (8.8.8.8) → GuardDuty BLIND
- ❌ OpenDNS / Cloudflare / custom DNS → GuardDuty BLIND

**Why**: GuardDuty taps into the internal AWS DNS resolver service. It doesn't see queries that go to other DNS resolvers.

**Workaround**: enable Route 53 Resolver query logging separately to S3 → analyze with Athena. This won't give you GuardDuty findings, but you keep DNS visibility.

### Trusted IP list vs Threat IP list

GuardDuty supports TWO custom IP lists per account per region:

| List type | Purpose | Limit per account/region |
|---|---|---|
| **Trusted IP list** | Whitelist — GuardDuty IGNORES these IPs for VPC Flow Logs + CloudTrail findings | **1 list** |
| **Threat IP list** | Blacklist — GuardDuty generates findings on activity from these IPs | **6 lists** |

### Critical list rules (Q38 + Q63 patterns)

1. **Trusted IP takes PRIORITY over threat IP** — if an IP appears in both, no finding is generated (whitelist wins)
2. **IPs must be PUBLICLY ROUTABLE IPv4** — no RFC1918 private addresses (10.x, 172.16-31.x, 192.168.x), no IPv6
3. **Same region as GuardDuty** — lists are per-region; uploading to wrong region = no effect
4. **Affects VPC Flow Logs + CloudTrail findings ONLY** — does NOT affect DNS findings
5. **Per-account** — not propagated across organization automatically

### Multi-account GuardDuty

GuardDuty in AWS Organizations:
- Delegate an admin account (often the security account)
- Delegated admin manages member accounts' GuardDuty
- Admin's THREAT lists propagate to all member accounts (centralized threat intel)
- Admin's TRUSTED lists do NOT propagate (per-account configuration)
- Findings aggregated in admin account

### Permissions for managing GuardDuty lists

| Permission level | Capabilities |
|---|---|
| `AmazonGuardDutyFullAccess` managed policy | View, rename, deactivate lists |
| `AmazonGuardDutyFullAccess` + `iam:PutRolePolicy` + `iam:DeleteRolePolicy` on the SLR | Upload, rename, deactivate, AND delete lists |

The extra IAM perms are needed because list operations modify the GuardDuty service-linked role's inline policy.

### GuardDuty finding categories

GuardDuty findings are categorized by attack pattern:

| Category | Example findings |
|---|---|
| **Reconnaissance** | Port scanning, brute force attempts, no-data instances |
| **Instance Compromise** | Crypto-mining, malicious tools, suspicious DNS |
| **Account Compromise** | Anomalous API calls, root user activity, credential exfiltration |
| **Bucket Compromise** | Unusual data access patterns, public bucket changes |
| **Kubernetes Compromise** | Anomalous pod behavior, suspicious K8s API calls |
| **Malware** | Malicious files detected on EBS scans |

### GuardDuty severity levels

| Severity | Range | Meaning |
|---|---|---|
| **Low** | 1.0 - 3.9 | Informational, suspicious but not confirmed |
| **Medium** | 4.0 - 6.9 | Likely threat, investigate |
| **High** | 7.0 - 8.9 | Confirmed threat, immediate action |

### Integrations with other security services

GuardDuty findings can be:
- Forwarded to **Security Hub** (aggregation + dashboards)
- Pivoted to **Detective** (investigation graph)
- Triggered via **EventBridge** (automation pipeline)
- Acted on by **Lambda** (auto-remediation)

For "automatically respond to findings": always **EventBridge rule + Lambda function**.

---

## CloudFront — origin restriction + WAF combos

### The OAI vs OAC distinction (Q47 pattern)

When you front an S3 bucket with CloudFront, you want to PREVENT direct S3 access (so users must go through CloudFront, where WAF/caching/HTTPS apply).

Two mechanisms exist:

| Mechanism | Status | What it does |
|---|---|---|
| **Origin Access Identity (OAI)** | Legacy (still works) | IAM-based — special CloudFront identity, S3 bucket policy allows OAI |
| **Origin Access Control (OAC)** | Modern (preferred since 2022) | SigV4-based — CloudFront signs requests, S3 bucket policy allows specific distribution |

**AWS recommendation**: Use OAC for new setups. OAI still works but lacks features:
- OAC supports SSE-KMS encrypted S3 objects (OAI doesn't)
- OAC supports POST/PUT methods (OAI is read-only)
- OAC supports all AWS regions including opt-in (OAI doesn't)
- OAC uses SigV4 (more secure than OAI's mechanism)

### OAC + S3 bucket policy example

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT:distribution/DISTRIBUTION_ID"
      }
    }
  }]
}
```

This says: "only CloudFront, ONLY from MY specific distribution, can read from this bucket."

### The full restriction pattern (Q47 — two parts needed)

For "only specific IPs allowed, and only via CloudFront":

| Layer | What | Why |
|---|---|---|
| **CloudFront + WAF** | WAF web ACL on CloudFront distribution with IP set match (allow only the allowed IPs) | Blocks unwanted IPs at edge |
| **CloudFront + OAI/OAC + S3 bucket policy** | OAC restricts S3 bucket to only accept requests from this specific CloudFront distribution | Prevents bypass via direct S3 URL |

Both are needed. Without OAC: users could discover the S3 bucket URL and bypass CloudFront (defeating the WAF IP restriction).

### Other CloudFront security features

| Feature | Use case |
|---|---|
| **HTTPS Only** | Reject HTTP requests, redirect to HTTPS, or HTTPS-only listener |
| **TLS 1.2+** | Enforce minimum TLS version |
| **Geo restriction** | Block/allow countries (whitelist/blacklist) — separate from WAF geo |
| **Field-level encryption** | Encrypt specific request fields at edge (e.g., credit card #) |
| **Signed URLs / signed cookies** | Time-limited access to specific content |
| **Custom headers from CloudFront to origin** | Origin verifies header presence to ensure CF-only access |
| **AWS Shield Standard** | Auto L3/L4 DDoS protection (free) |
| **AWS Shield Advanced** | Enhanced DDoS protection |
| **Response Headers Policy** | Add security headers like HSTS, CSP, X-Frame-Options |

---

## DDoS resilience reference architecture

### The defensive layers (memorize this stack)

```
Internet
   ↓
AWS Edge (Route 53 + CloudFront + Shield Standard)
   ├── Route 53 — DNS DDoS resilience (anycast, shuffle sharding)
   ├── CloudFront — caches dynamic + static, absorbs L7 attacks
   └── Shield Standard — auto L3/L4 mitigation (free)
   ↓
AWS WAF (on CloudFront) — L7 attack filtering
   ├── Rate-based rules — throttle abusive IPs
   ├── IP set match — block known bad IPs
   ├── Geo match — block bad regions
   └── Managed rules — OWASP Top 10 baseline
   ↓
Application Load Balancer — health checks + connection management
   ↓ (security group with NO connection tracking exhaustion)
   ↓
Auto Scaling Group (absorbs scale via EC2 spin-up)
   ↓
Application
   ↓
Database (in private subnets)
```

### DDoS best practices (Q17 + Q27 patterns)

1. **Use Route 53 + CloudFront** for ALL public-facing apps (even non-static)
2. **Move static content to S3 + CloudFront** to reduce origin load
3. **Front API Gateway with your OWN CloudFront** (not edge-optimized which uses AWS-managed CF)
4. **Forward ALL headers to API Gateway REGIONAL endpoint** via your CloudFront (so content is treated as dynamic, not cached badly)
5. **Register Elastic IPs in Shield Advanced** for faster L3/L4 mitigation
6. **Configure SGs to NOT use connection tracking** (rules covering all traffic make connections untracked)
7. **ALB instead of NLB** for L7 apps (ALB integrates with WAF; NLB doesn't see HTTP)
8. **CloudFront in front of ALB** for additional edge protection
9. **Auto-scaling for absorption** (let traffic spread across more instances)
10. **Shield Advanced for cost protection** (refunds DDoS scaling charges)

### SG connection tracking optimization

Security groups are stateful by default — they track every connection. Under DDoS, the connection table can fill up, causing legitimate connections to be rejected.

**The fix** (best practice for DDoS-resilient SGs):
- Configure SG rules covering ALL traffic (`0.0.0.0/0`, ports 0-65535)
- Connections matching such broad rules are UNTRACKED (no table entry)
- Untracked connections don't exhaust the connection table

**Trade-off**: less granular tracking. Use this on resources that have minimal connection-specific needs and benefit from DDoS resilience (ALBs primarily).

### L3/L4 vs L7 DDoS attack response

| Attack type | Auto-mitigation | Manual response |
|---|---|---|
| **L3/L4** (SYN flood, UDP reflection, fragmentation) | ✅ Shield Standard auto-mitigates | None usually needed |
| **L7** (HTTP flood, slowloris, application abuse) | ❌ NOT auto-mitigated | WAF rules + Shield Advanced DRT support |

For L7 DDoS questions: the answer is ALWAYS WAF rules + possibly Shield Advanced DRT — NEVER GuardDuty (just detection), NEVER CloudWatch alarms (notification only).

### API Gateway DDoS architecture (Q17 trap)

**The trap**: edge-optimized API Gateway endpoints use AWS-managed CloudFront. You CAN'T attach your own WAF/Shield/customization to them.

**The right answer**: use API Gateway REGIONAL endpoint + YOUR OWN CloudFront distribution in front. This gives you:
- Full CloudFront configuration control
- Ability to attach WAF web ACL
- Custom cache behaviors (forward ALL headers to treat API as dynamic)
- Better DDoS visibility

---

## Detection services comparison (knowing which to use)

| Service | Category | What it sees | Action capability |
|---|---|---|---|
| **GuardDuty** | Threat detection | Logs analysis (Flow Logs, CloudTrail, DNS, EKS, etc.) | Findings only (no blocking) |
| **Inspector** | Vulnerability scanning | EC2 packages, ECR images, Lambda functions | Findings only (no patching) |
| **Macie** | Sensitive data discovery | S3 objects (PII, PHI, financial) | Findings only (no action) |
| **Detective** | Investigation pivot | Aggregates Flow Logs, CloudTrail, EKS, GuardDuty into graph | Read-only, supports investigation |
| **Security Hub** | Aggregation + posture | Aggregates findings from GuardDuty, Inspector, Macie, etc. | Aggregation, scoring, dashboards |
| **Trusted Advisor** | Best-practice checks | Account-level configuration checks | Recommendations only |
| **Access Analyzer** | External access detection | Resource policies (S3, IAM, KMS, Secrets Manager, etc.) | Findings only |
| **Config** | Configuration tracking | Resource configuration history | Compliance evaluation + remediation via SSM |

### "Detection vs prevention" exam trap

These services DETECT only — NOT prevent:
- GuardDuty
- Inspector
- Macie
- Detective
- Security Hub
- Access Analyzer
- Trusted Advisor
- Config

These services PREVENT (block) — NOT just detect:
- AWS WAF (L7 HTTP)
- AWS Shield (L3/L4 DDoS)
- AWS Network Firewall (L3/L4/L7)
- Security Groups (L4)
- Network ACLs (L4, stateless)
- IAM policies / SCPs (authorization)
- Firewall Manager (orchestrates the above)

For "automatically respond to a detected threat": chain detection service + EventBridge + Lambda + prevention service.

---

## Logging architectures (common patterns)

### WAF analytics dashboard
```
WAF → Kinesis Data Firehose → S3 → AWS Glue → Athena → QuickSight
```

### CloudFront access analysis
```
CloudFront → S3 (standard logs) → Athena
        OR
CloudFront → Kinesis Data Streams (real-time logs) → Lambda / Firehose
```

### ALB access analysis
```
ALB → S3 (only destination) → Athena
```

### VPC Flow Logs
```
VPC Flow Logs → CloudWatch Logs / S3 / Firehose
```

### GuardDuty findings forwarding
```
GuardDuty → EventBridge rule → Lambda / SNS / Security Hub / Step Functions
```

### Security event correlation
```
Multiple sources → Security Hub (aggregation) → Detective (investigation)
                                              → EventBridge (automation)
                                              → Security Lake (OCSF storage)
```

---

## Common traps (recognize on sight)

| Trap pattern | Why wrong | Right answer |
|---|---|---|
| "Attach WAF to NLB" | NLB is L4; WAF is L7 | Use ALB or CloudFront in front |
| "Attach WAF to ASG / EC2 directly" | WAF doesn't attach to compute | Use ALB or CloudFront in front |
| "WAF logs to CloudTrail" | CloudTrail is for management events | Use CloudWatch Logs / S3 / Firehose |
| "WAF metrics in CloudTrail" | Metrics go to CloudWatch | CloudWatch metrics at 1-min granularity |
| "Firewall Manager inspects traffic" | FM is MANAGEMENT only | NF/WAF/SG inspect; FM just deploys policies |
| "GuardDuty blocks malicious IPs" | GuardDuty only detects | Use WAF/NF/SG for blocking; chain via EventBridge |
| "Shield Standard requires configuration" | Always-on, free, no config | Just use Route 53/CloudFront/Global Accelerator |
| "Shield Advanced auto-blocks L7 attacks" | L7 is manual via WAF rules or DRT support | Write WAF rules + contact DRT |
| "Use edge-optimized API GW for DDoS protection" | AWS-managed CloudFront, less control | Use REGIONAL API GW + YOUR CloudFront with WAF |
| "GuardDuty sees DNS via custom resolver" | Custom DNS resolvers bypass GuardDuty DNS analysis | Use AWS DNS resolver OR Route 53 Resolver query logging |
| "GuardDuty trusted IP processed AFTER threat IP" | Trusted IP wins (whitelist priority) | Trusted IP list checked first |
| "Trusted IP list accepts private IPv4" | Only publicly routable IPv4 | RFC1918 IPs ignored |
| "Multiple trusted IP lists per account/region" | Only ONE trusted list allowed (6 threat lists) | Use one consolidated trusted list |
| "OAI supports SSE-KMS S3 objects" | OAI doesn't; OAC does | Use OAC for SSE-KMS buckets |
| "Network Firewall protects against L7 HTTP attacks" | NF is general L3/L4/L7 but not optimized for HTTP attacks | Use WAF for L7 HTTP |
| "Inspector detects compromised credentials" | Inspector scans vulnerabilities, not creds | GuardDuty detects credential compromise |
| "Macie scans EC2 instances" | Macie scans S3 only | Inspector for EC2, Macie for S3 |
| "Detective generates findings" | Detective investigates findings | GuardDuty/Inspector/Macie generate findings |
| "Trusted Advisor monitors security in real-time" | TA does periodic checks | Use CW/EventBridge for real-time |
| "Security Hub blocks threats" | SH aggregates, doesn't block | Aggregation only |

---

## Question pattern recognition (drill these)

### Pattern 1: "Attach WAF to my workload running on EC2"
**Trigger**: EC2 workload + WAF protection request

**Answer**: Must front EC2 with ALB or CloudFront. WAF web ACL attaches to ALB or CloudFront. CONSTRUCTION ERROR if option says "WAF on EC2/ASG/NLB."

### Pattern 2: "Build serverless WAF analytics dashboard"
**Trigger**: "serverless," "WAF logs analysis," "dashboards"

**Answer**: WAF → Firehose → S3 → Glue → Athena → QuickSight chain.

### Pattern 3: "Validate WAF rules are working"
**Trigger**: "validate WAF," "check if rules fire"

**Answer**: Enable comprehensive WAF logs to Kinesis Firehose (or CloudWatch Logs, or S3). NOT CloudTrail (CT is for management events).

### Pattern 4: "Defend against L3/L4 DDoS"
**Trigger**: "SYN flood," "UDP reflection," "network DDoS"

**Answer**: Shield Standard (auto, free) covers it. For enhanced: Shield Advanced + register EIPs as protected resources.

### Pattern 5: "Defend against L7 DDoS / HTTP flood"
**Trigger**: "HTTP flood," "application DDoS," "L7 attack"

**Answer**: Write WAF rules (rate-based, geo, IP set) + contact AWS DRT if Shield Advanced. NOT GuardDuty, NOT CloudWatch alarms.

### Pattern 6: "Restrict S3 access to CloudFront only"
**Trigger**: "S3 + CloudFront," "prevent direct S3 access"

**Answer**: Create OAC (or OAI), configure S3 bucket policy to allow only CloudFront via the OAC's SourceArn. Combine with WAF IP restriction on CloudFront.

### Pattern 7: "DDoS-resilient architecture revamp"
**Trigger**: "after DDoS attack," "redesign for resilience"

**Answer**: Route 53 + CloudFront (with WAF) + offload static to S3 + Shield. NOT WAF on ASG directly.

### Pattern 8: "GuardDuty DNS analysis not working"
**Trigger**: "GuardDuty not seeing DNS," "Active Directory DNS"

**Answer**: Custom DNS resolvers (on-prem AD, Google DNS, etc.) bypass GuardDuty DNS. Use AWS DNS resolver OR enable Route 53 Resolver query logging separately.

### Pattern 9: "GuardDuty trusted IP list misbehavior"
**Trigger**: "configured trusted IPs but still alerting"

**Answer**: Check IPs are publicly routable IPv4 (not RFC1918), same region as GuardDuty, and only ONE trusted list per account/region.

### Pattern 10: "Centralize WAF policy across org"
**Trigger**: "AWS Organizations," "consistent WAF across accounts"

**Answer**: AWS Firewall Manager (with WAF policy type). Requires Organizations + Config + delegated admin.

### Pattern 11: "Firewall Manager web ACL not associated"
**Trigger**: "Firewall Manager created web ACL but not applied to resources"

**Answer**: Auto-remediation toggle is OFF. Enable "auto remediate any non-compliant resources."

### Pattern 12: "Block specific country / region"
**Trigger**: "block country," "geo restriction"

**Answer**: WAF geo match statement (rule action: Block) on the web ACL attached to CloudFront/ALB.

### Pattern 13: "Block specific IP range"
**Trigger**: "block IPs," "malicious IP"

**Answer**: WAF IP set match rule (rule action: Block). Up to 10,000 IPs per IP set. NOT security group deny (SGs can't deny).

### Pattern 14: "Allow only company office IPs"
**Trigger**: "allowlist," "whitelist specific IPs"

**Answer**: WAF IP set with Allow action + default action Block on web ACL.

### Pattern 15: "L7 attack auto-response"
**Trigger**: "automatic remediation," "respond to GuardDuty finding"

**Answer**: GuardDuty finding → EventBridge rule → Lambda → take action (modify SG, add WAF rule, isolate instance).

---

## Final exam-day quick reference

### Service-to-attack mapping
| Attack type | Right service |
|---|---|
| SQL injection / XSS | WAF |
| L3/L4 DDoS (SYN flood, UDP flood) | Shield Standard (auto) / Shield Advanced (enhanced) |
| L7 DDoS (HTTP flood) | WAF rules + Shield Advanced DRT |
| Specific IP blocking | WAF IP set match |
| Country blocking | WAF geo match OR CloudFront geo restriction |
| Egress malware C2 | Network Firewall (Suricata rules) |
| DNS exfiltration | Network Firewall DNS Firewall / Route 53 Resolver DNS Firewall |
| Compromised credentials | GuardDuty (detect) + Lambda (respond) |
| Public S3 buckets | Macie / Access Analyzer (detect) + S3 Block Public Access (prevent) |
| Vulnerable EC2 software | Inspector (detect) + Patch Manager (remediate) |

### Logging destinations cheat
| Service | Supported destinations |
|---|---|
| **WAF** | CloudWatch Logs, S3, Kinesis Firehose (NOT CloudTrail) |
| **CloudFront standard logs** | S3 only |
| **CloudFront real-time logs** | Kinesis Data Streams |
| **ALB access logs** | S3 only |
| **VPC Flow Logs** | CloudWatch Logs, S3, Firehose |
| **TGW Flow Logs** | CloudWatch Logs, S3, Firehose |
| **GuardDuty findings** | EventBridge, Security Hub |
| **Network Firewall logs** | CloudWatch Logs, S3, Firehose |
| **DNS Firewall logs** | CloudWatch Logs, S3, Firehose |

### Attachment points cheat
| Service | Can attach to |
|---|---|
| **WAF web ACL** | CloudFront, ALB, API GW (REST), AppSync, Cognito User Pool, App Runner, Verified Access |
| **Shield Advanced protection** | EIP, CloudFront, ALB, NLB, Global Accelerator, Route 53 hosted zone |
| **Network Firewall** | VPC subnet (via VPC endpoint) |
| **Firewall Manager policy scope** | OUs, accounts, tagged resources |

---

## Cross-references

- Primary cheatsheet: `priority-review-cheatsheet.md` — Tier 2 D3 + Tier 6 Mock #1 misses
- Reinforcement notes: `reinforcement-cheatsheet.md` — D3 patterns you got right
- KMS deep-dive: `kms-deep-dive.md` — encryption layer
- Domain notes: `docs/domain-3-infrastructure-security.md`

---

## Self-test (do these from memory)

1. List ALL resource types WAF web ACLs can attach to. ✏️
2. Where do WAF inspected request logs go? (Three destinations.) ✏️
3. What does Shield Standard cover, and what does it cost? ✏️
4. Why is "WAF on ASG" a construction error? ✏️
5. What's the auto-mitigation status for L7 DDoS attacks even with Shield Advanced? ✏️
6. What chain of services builds a serverless WAF analytics dashboard? ✏️
7. What's the difference between OAI and OAC for CloudFront → S3 restriction? ✏️
8. What's the trap with edge-optimized API Gateway endpoints for DDoS protection? ✏️
9. Why might GuardDuty miss DNS-based findings even when enabled? ✏️
10. If an IP is on both a trusted IP list and a threat IP list, what happens? ✏️
11. How many trusted IP lists vs threat IP lists per account per region? ✏️
12. Can Firewall Manager inspect traffic? ✏️
13. What's the SG connection tracking optimization for DDoS resilience? ✏️
14. What's the right architecture for a public-facing app to be DDoS-resilient? ✏️
15. What service compares for "detect threats" vs "block threats"? Name them. ✏️
16. For L7 DDoS active response, what are the two correct actions? ✏️

**Answers** (verify against memory):
1. CloudFront, ALB, API GW (REST), AppSync, Cognito User Pool, App Runner, Verified Access. NOT NLB, NOT EC2, NOT ASG, NOT S3 directly.
2. CloudWatch Logs, Amazon S3, Kinesis Data Firehose. NOT CloudTrail.
3. Auto L3/L4 DDoS mitigation. Free. Always on. No config.
4. WAF is L7 (HTTP). ASG is a compute scaling construct, not an HTTP endpoint. WAF needs a web-facing resource (CloudFront/ALB/etc.) to attach to.
5. NOT auto-mitigated. Shield Advanced detects L7 attacks but you must write WAF rules OR contact AWS DRT for help.
6. WAF → Kinesis Data Firehose → S3 → AWS Glue Crawler → Athena → QuickSight
7. OAC is newer and supports SSE-KMS S3 objects, all regions including opt-in, POST/PUT methods, SigV4. OAI is legacy, lacks these features. AWS recommends OAC.
8. Edge-optimized API GW uses AWS-managed CloudFront that you can't configure or attach WAF to. Use REGIONAL API GW + YOUR own CloudFront with WAF instead.
9. GuardDuty DNS analysis works only with AWS DNS resolver. If instances use on-prem AD DNS, Google DNS, or any custom resolver, GuardDuty is blind to DNS.
10. Trusted IP list takes PRIORITY — no finding is generated. Whitelist wins.
11. ONE trusted IP list per account per region. SIX threat IP lists per account per region.
12. NO. Firewall Manager is a MANAGEMENT service — it deploys WAF/Shield/NF/SG policies across accounts. It doesn't inspect traffic itself.
13. Configure SG rules covering ALL traffic (0.0.0.0/0, ports 0-65535) so connections become UNTRACKED, avoiding connection table exhaustion under DDoS.
14. Route 53 + CloudFront (with WAF web ACL) + offload static to S3 + Shield Standard (auto) or Shield Advanced (enhanced). For dynamic origins: regional ALB behind CloudFront.
15. **Detection only**: GuardDuty, Inspector, Macie, Detective, Security Hub, Access Analyzer, Trusted Advisor, Config. **Blocking/prevention**: WAF, Shield, Network Firewall, Security Groups, NACLs, IAM/SCPs.
16. Write custom WAF rules (rate-based, IP set, geo) to block attack patterns. Contact AWS DRT if you have Shield Advanced.

If you got 14-16/16, WAF/DDoS is locked. If you got 11-13, drill the misses. If <11, re-read the relevant sections.

---

*WAF/DDoS Deep-Dive v1 — June 5, 2026. Drill before Mock #2. Single source of truth for network protection topics on SCS-C03.*
