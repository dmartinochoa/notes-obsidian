# SCS-C03 Priority Review Cheatsheet

Built from mock exam results (May 29, 2026): 61% overall, weakest in D5 (38%), D3 (42%), D6 (50%).
Exam: **June 10, 2026 at 10:30 AM**.

Use this as your **final reference document** — read it daily, drill from it, re-read on taper day.

---

## Tier 1 — CRITICAL: Domain 5 Data Protection

### KMS rotation matrix (memorize cold)

| Key type | Auto-rotation? | Immediate delete? | Customer manages? |
|---|---|---|---|
| **AWS-owned** | AWS managed (invisible) | N/A | No |
| **AWS-managed** (`aws/service`) | Required, ~yearly | No (can't manage) | No |
| **Customer-managed, AWS-generated symmetric** | ✅ Optional (90–2560 days) | No (7–30 day wait) | Yes |
| **Customer-managed, IMPORTED material** | ❌ **NEVER** | ✅ **YES** (`DeleteImportedKeyMaterial`) | Yes (you bring it) |
| **Customer-managed, asymmetric** | ❌ No | No | Yes |
| **Custom Key Store (CloudHSM)** | ❌ No | Depends on cluster | Yes (HSM in AWS) |
| **External Key Store (XKS)** | ❌ No | Depends on external | Yes (HSM outside AWS) |

**The reflex**: "auto-rotate" → only symmetric AWS-generated CMK works.
"Immediate delete (under 7 days)" → only imported key material via `DeleteImportedKeyMaterial`.

### KMS access — the unique quirk

> **Root has NO implicit access to KMS keys.** Unlike most AWS services, the root user only has access if the key policy explicitly grants it.

- Default key policy includes a statement granting root full access — that's the ONLY reason root works
- Delete that statement = lockout (AWS Support recovery only)
- Three ways to grant KMS access: **key policy**, **IAM policy** (only works if key policy delegates), **KMS grants** (no service principals as grantees)

### KMS rotation operations

- **Auto-rotation**: rotates backing key material; old material retained for decryption; transparent to apps
- **Manual rotation** (universal pattern): new CMK → repoint alias → keep old CMK active
- **Rotation does NOT re-encrypt existing data.** Each ciphertext stores an EDK referencing the exact CMK that wrapped it. Use `kms:ReEncrypt` only if you need to retire the old key.

### Client-side encryption library family

| Library | Tailored for | Key trait |
|---|---|---|
| **AWS Encryption SDK** | Arbitrary data | Opaque ciphertext blob |
| **AWS Database Encryption SDK for DynamoDB** (formerly DynamoDB Encryption Client) | DynamoDB items | Attribute-aware, leaves PK plaintext |
| **S3 Encryption Client** (in SDKs) | S3 objects | Object-stream-aware |

**Decoder**: DynamoDB-specific → DB Encryption SDK; generic → Encryption SDK; "server-side encryption" → eliminate all of these.

### S3 encryption variants

| Type | Key managed by | Use when |
|---|---|---|
| **SSE-S3** | AWS (default since Jan 2023) | Default, no special requirements |
| **SSE-KMS** | KMS CMK | Customer key control, audit |
| **DSSE-KMS** | KMS CMK, dual-layer | High security requirements |
| **SSE-C** | Customer-provided | You manage keys outside AWS |
| **Client-side** | Customer | Encrypt before sending to AWS |

**Reflex**: "different key per file/object" — both SSE-S3 and SSE-KMS already do this automatically. SSE-S3 wins when no customer key control needed.

### ACM certificates — region + exportability rules

| Service using cert | ACM cert region |
|---|---|
| **CloudFront** | **us-east-1 ALWAYS** |
| API Gateway edge-optimized | us-east-1 ALWAYS |
| ALB / NLB / regional API Gateway | Same region as the resource |

| Cert origin | Exportable? | Where can it run? |
|---|---|---|
| ACM-issued (Amazon-issued) | ❌ No | AWS-managed services only (ALB/NLB/CloudFront/API GW) |
| Imported 3rd-party | ✅ Yes | Anywhere |
| ACM Private CA end-entity | ✅ Yes | Anywhere internal |
| Self-signed | ✅ Yes | Anywhere, no public trust |

**Validation method for auto-renewal**: DNS validation (CNAME stays) = indefinite auto-renewal. Email validation = manual click required.

### Secrets Manager rotation Lambda (4 steps)

`createSecret` → `setSecret` → `testSecret` → `finishSecret`

- AWSCURRENT, AWSPENDING, AWSPREVIOUS versions
- Auto-rotation natively supported for RDS, Redshift, DocumentDB
- Cross-account access via resource policy

### S3 Object Lock + Glacier Vault Lock

- **S3 Object Lock Governance**: bypassable with special permission (s3:BypassGovernanceRetention)
- **S3 Object Lock Compliance**: NO ONE can bypass (including root) until retention ends
- **Glacier Vault Lock**: policy immutable once committed; for SEC Rule 17a-4 compliance
- Requires versioning enabled on the bucket

### Macie

- S3 only (sensitive data discovery)
- ML-based + 100+ managed data identifiers (PII, PHI, financial)
- Findings flow to Security Hub

### KMS access mechanism triangle

```
Key Policy (resource-based) — primary, must exist
       ↓ (delegates to IAM via root statement)
IAM Policy (identity-based) — only works if key policy allows
       ↓
KMS Grants — programmatic temporary delegation, no service principals as grantees
```

### CloudHSM vs KMS vs Custom Key Store vs XKS

| | KMS default | KMS Custom Key Store (CloudHSM-backed) | XKS (External Key Store) | Standalone CloudHSM |
|---|---|---|---|---|
| HSM location | AWS multi-tenant | YOUR CloudHSM in AWS | YOUR HSM outside AWS | YOUR CloudHSM in AWS |
| Customer manages HSM | No | ✅ Yes | ✅ Yes | ✅ Yes |
| KMS API works (S3 SSE-KMS, etc.) | ✅ | ✅ | ✅ | ❌ Manual integration |
| FIPS 140-2 | Level 3 | Level 3 | Depends on your HSM | Level 3 |

---

## Tier 2 — HIGH: Domain 3 Infrastructure Security

### CloudFront policies trio (corrected May 29)

| Policy | Controls | Cache impact |
|---|---|---|
| **Cache Policy** | Cache key + forwards what's in the key | Yes |
| **Origin Request Policy** | Forwards to origin WITHOUT cache key impact | No |
| **Response Headers Policy** | Headers added to responses (HSTS, CSP, X-Frame-Options) | N/A |

**CRITICAL RULE**: **Authorization header MUST go in Cache Policy.** Origin Request Policy returns HTTP 400 if you try to include Authorization. This is security-by-design (prevents auth bypass via cached responses).

### VPC endpoints

| Type | Services | Cost | Works from on-prem? |
|---|---|---|---|
| **Gateway** | S3, DynamoDB ONLY | Free | ❌ No (route table is VPC-local) |
| **Interface (PrivateLink)** | Everything else | ~$0.01/hr/AZ + data | ✅ Yes (private IP via DX/VPN) |
| **S3 Interface** (newer, since 2021) | S3 specifically when on-prem access needed | Paid | ✅ Yes |

### VPC endpoint policy mechanics

- Action must be **runtime action** of target service (`execute-api:Invoke`, `s3:GetObject`, `kms:Decrypt`)
- NEVER `ec2:*VpcEndpoint*` (that's for managing the endpoint object)
- Default policy = full access; replace to restrict
- Count distinct resource IDs (not duplicates)

### Network protection services (when to pick each)

| Service | Layer | Action capability | When |
|---|---|---|---|
| **AWS WAF** | L7 (HTTP) | Allow/Block/Count/CAPTCHA | HTTP attacks (SQLi, XSS, rate limiting, geo) |
| **Network Firewall** | L3/L4/L7 | Allow/Block (active inline) | DPI, IPS/IDS via Suricata rules |
| **Shield Standard** | L3/L4 | Auto DDoS mitigation | Free, automatic |
| **Shield Advanced** | L3/L4/L7 | Enhanced DDoS + DRT + cost protection | $3K/mo, regulated workloads |
| **Firewall Manager** | Management | NONE — it manages other services | Centralize WAF/Shield/NF/SG policies |
| **VPC Traffic Mirroring** | L3+ | Passive copy only | Forensics, IDS, threat hunting |

**Firewall Manager doesn't inspect traffic itself** — common exam trap. It centralizes management of WAF/Shield/Network Firewall policies across an Organization.

### mTLS / TLS termination location

| Listener | Who terminates TLS? | mTLS feasible at server? |
|---|---|---|
| **NLB TCP** | Server (passthrough) | ✅ Yes — server sees raw TLS |
| NLB TLS | NLB | ❌ No, server gets decrypted |
| ALB HTTPS | ALB | ❌ No, server gets decrypted |
| ALB mTLS (newer) | ALB | ❌ ALB does mTLS check |

**When question says "server must terminate TLS" → NLB TCP listener (passthrough).**

### VPC peering quirks

| Scenario | SG ID reference? |
|---|---|
| Same account, same region | ✅ Standard SG ID |
| Cross account, same region | ✅ `account-id/sg-id` format |
| Same account, **cross region** | ❌ CIDR only |
| Cross account, **cross region** | ❌ CIDR only |

**Region boundary breaks SG references, not the account boundary.**

Other quirks:
- No transitive peering (A↔B + B↔C ≠ A↔C; use Transit Gateway for transitive)
- No overlapping CIDRs (must refactor IPs)
- Cross-region MTU = 1500 (no jumbo frames)

### Traffic Mirroring vs Flow Logs

| | Traffic Mirroring | VPC Flow Logs |
|---|---|---|
| Captures | **Full packet contents** | Metadata only (src/dst/port/protocol/bytes) |
| Use for | Content inspection, DPI, IDS, forensics | Network flow analysis, SG validation |
| Filtering | Filters by src/dst/protocol/port | All matching traffic |
| Cost | Pay per mirror session + processed bytes | $0.50/GB ingested |

**Flow Logs do NOT capture**:
- 169.254.169.254 (IMDS)
- 169.254.169.123 (time sync)
- DHCP
- Default VPC router
- **Traffic between Endpoint ENI and NLB ENI (PrivateLink)** ← bit you on the PrivateLink troubleshooting question

### SSM Session Manager 3 endpoints (memorize: "SSM + two messages")

For SSM in a private subnet without internet, you need ALL three Interface Endpoints:
- `com.amazonaws.<region>.ssm`
- `com.amazonaws.<region>.ssmmessages`
- `com.amazonaws.<region>.ec2messages`

Optional adds: `s3` Gateway (for patches/output), `logs`, `kms`.

### Hybrid connectivity

| Connection | Private? | Encrypted? | Bandwidth | Latency |
|---|---|---|---|---|
| Internet + Site-to-Site VPN | No (over internet) | ✅ IPsec | Variable | Variable |
| Direct Connect alone | ✅ | ❌ **NOT encrypted by default** | High consistent | Low |
| **DX + Site-to-Site VPN** | ✅ | ✅ IPsec (L3) | High | Low + small VPN overhead |
| **DX + MACsec** | ✅ | ✅ L2 (10/100 Gbps Dedicated only) | Full DX speed | Lowest |

**Private ≠ encrypted.** Direct Connect is a private path but unencrypted unless you add VPN or MACsec.

### EC2 IR — modern preservation sequence

1. Enable termination protection
2. Capture metadata + tag "Compromised"
3. **SSM Run Command** for memory acquisition (BEFORE stopping)
4. Detach from ASG (`--should-decrement-desired-capacity`)
5. Deregister from ELB target groups
6. **Isolate via SG** (deny-all or diagnostics-only)
7. **Snapshot EBS volumes** (don't detach originals; preserve via snapshot, attach copies to forensic instance)
8. Investigate via Detective + CloudTrail + Flow Logs
9. Terminate only after evidence preserved

**Rule**: NEVER attach the original (suspect) EBS volume to forensic instance. Always snapshot first → create new volume from snapshot → attach the COPY.

**SG vs NACL for isolation**:
- Multiple instances in subnet → SG modification (instance-scoped)
- Single instance alone in subnet → NACL outbound deny-all (faster, supports deny which SGs don't)

---

## Tier 3 — MEDIUM: Domain 6 Governance

### SCP behavior (memorize cold)

- ✅ Applies to ALL principals in member accounts (including root)
- ❌ Does NOT affect the management account by default
- ❌ Does NOT affect service-linked roles
- ❌ Cannot exclude member-account admins structurally (only via Condition with `aws:PrincipalArn`)
- ❌ SCPs NEVER grant; only restrict ("cap" pattern)

### SCP vs Permissions Boundary vs RCP

| Control                                  | Scope                                | Set via                | Use case                                            |
| ---------------------------------------- | ------------------------------------ | ---------------------- | --------------------------------------------------- |
| **SCP**                                  | Org / OU / account-wide              | AWS Organizations      | "No one in this OU can do X"                        |
| **Permissions Boundary**                 | Per-principal                        | IAM (on the user/role) | "This role can never exceed Y" (delegation pattern) |
| **RCP** (Resource Control Policy, newer) | Org / OU / account-wide on resources | AWS Organizations      | Restricts external access TO your resources         |

### Region restriction SCP pattern (recognize on sight)

```json
{
  "Effect": "Deny",
  "NotAction": [
    "iam:*", "cloudfront:*", "route53:*", "kms:*",
    "sts:*", "waf:*", "shield:*", "support:*",
    "organizations:*", "billing:*"
  ],
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["us-west-2"]
    }
  }
}
```

Reads: "Deny everything (except global services in NotAction) when region is not us-west-2."

### AWS RAM + VPC sharing

| Pattern | What it does | Each account has own VPC? |
|---|---|---|
| **VPC Peering** | Connects two separate VPCs | ✅ Yes |
| **Transit Gateway** | Hub-and-spoke between VPCs | ✅ Yes |
| **VPC Sharing (via RAM)** | ONE VPC's subnets shared across accounts | ❌ No — participants launch INTO shared VPC |

**Picking signal**:
- "Centralized network, decentralized workloads, no per-dept VPC" → VPC sharing
- "Connect separate VPCs each account autonomous" → Peering or TGW
- "Mesh of many VPCs at scale" → Transit Gateway

### Organizations key concepts

- **Delegated administrator** for: GuardDuty, Security Hub, Macie, Config, IAM Access Analyzer, CloudTrail (org trails), Inspector, Audit Manager, Firewall Manager
- **Organization trail** in mgmt/delegated-admin: member accounts CAN'T disable
- **SCP at root** affects all member OUs/accounts; mgmt account exempt

### Control Tower

- Mandatory + Strongly Recommended + Elective guardrails
- SCPs (preventive) + Config rules (detective) under the hood
- Account Factory for new account provisioning
- Three shared accounts: Management, Log Archive, Audit

### Config conformance packs

- Bundles of Config rules + remediation as one deployable artifact
- AWS-authored: CIS, PCI-DSS, HIPAA, NIST
- Custom packs for org-specific compliance
- Org-level deployment via Organizations

### IaC validation tools

| Tool | Purpose | Language |
|---|---|---|
| **cfn-guard** | Compliance policy validation | DSL (no programming required) |
| **cfn-lint** | Syntax + structure validation | YAML/JSON validation |
| **CDK** | Define IaC in real languages | TypeScript/Python/etc. |
| **CloudFormation Drift Detection** | Compare deployed vs template state | Built-in |
| **CFN Hooks** | Runtime validation during stack ops | Lambda-backed |

For "policy validation without writing code" → **cfn-guard**.

### Security Hub key facts

- **Aggregator** — receives findings from GuardDuty, Inspector, Macie, IAM Access Analyzer, Firewall Manager, Config, Patch Manager
- **Regional** (NOT global) — cross-region aggregation is opt-in
- **Prerequisites**: AWS Config MUST be enabled. GuardDuty/Inspector/Macie are OPTIONAL.
- **Does NOT retroactively detect** findings generated before enablement
- **ASFF** = AWS Security Finding Format (standard schema)

### Direction of Security Hub finding flow

```
INTO Security Hub          OUT of Security Hub
GuardDuty ──────→          ──→ Detective (investigation pivot)
Inspector ──────→ SH ─────→ ──→ Trusted Advisor (FSBP results)
Macie ──────────→          ──→ EventBridge (automation)
Firewall Manager →         ──→ Security Lake (OCSF)
Config ──────────→
```

**Detective and Trusted Advisor receive FROM Security Hub.** Do not generate findings INTO it.

---

## Tier 4 — MAINTAIN: D1, D2, D4

### D1 Detection — the canonical alert pattern

```
CloudTrail → CloudWatch Logs log group → Metric Filter → CloudWatch Alarm → SNS
                                       OR
                       → EventBridge rule → SNS (more efficient)
```

For real-time alerts → **EventBridge** (preferred).
For CIS Benchmark compliance metric filters → **CW Logs metric filter**.

### High-value CloudTrail events to recognize

- `ConsoleLogin` (with `responseElements.ConsoleLogin: Failure` for failed logins)
- `StopLogging` / `DeleteTrail` / `UpdateTrail` / `PutEventSelectors` (CloudTrail tampering)
- `AuthorizeSecurityGroupIngress` (SG changes)
- `CreateAccessKey` / `DeleteAccessKey` (credential events)
- Root account: `userIdentity.type = "Root"`

### Native log destinations

| Service | Destination(s) |
|---|---|
| **ELB access logs** | **S3 ONLY** |
| CloudFront standard logs | S3 |
| CloudFront real-time logs | Kinesis Data Streams |
| VPC Flow Logs | CW Logs / S3 / Firehose |
| TGW Flow Logs | CW Logs / S3 / Firehose |
| API Gateway logs | CW Logs / Firehose |
| Lambda logs | CW Logs (automatic) |
| RDS engine logs | CW Logs |
| WAF logs | S3 / CW Logs / Firehose |
| GuardDuty findings | EventBridge / Security Hub |

**CloudWatch metric filters operate ONLY on CW Logs, not S3.** For S3-stored logs → Athena query + Lambda + PutMetricData.

### D2 IR sequence (memorize)

```
PRESERVE → DOCUMENT → CAPTURE VOLATILE (memory) → CONTAIN → INVESTIGATE → ERADICATE → RECOVER → POST-INCIDENT
```

Never:
- Delete first (loses evidence)
- Stop before memory capture (loses RAM forensics)
- Attach original EBS to forensic instance (use snapshot copies)
- Sign into compromised host directly

### D4 IAM — high-yield reflexes

**Policy evaluation order**: Explicit Deny → SCP → Resource policy → Identity policy → Permission boundary → Session policy → Implicit Deny.

**`iam:PassRole`** vs **`sts:AssumeRole`**:
- PassRole = user HANDS a role to a SERVICE (e.g., Lambda execution role)
- AssumeRole = principal BECOMES a role

**Permissions Boundary use case**: delegation — let team create new roles but cap what those roles can do.

**STS regional vs global endpoints**:
- Global STS (`sts.amazonaws.com`) = v1 tokens, fail in opt-in regions
- Regional STS (`sts.<region>.amazonaws.com`) = v2 tokens, work everywhere

**Condition key operator-value typing**:
- `DateLessThan` needs ISO 8601 timestamp, not seconds
- `aws:MultiFactorAuthAge` is Numeric (seconds)
- `MaxSessionDuration` is NOT a condition key (it's a role attribute)

**IAM Paths** = directory-like grouping (`/platform/`, `/product/`); use with SCPs for delegation patterns.

**Identity-based policy vs Resource-based policy**: only resource-based policies have `Principal` element. If you see `Principal` in an answer marked as "identity-based" → eliminate.

**Cross-account access requires BOTH**: identity policy in source + resource policy (or trust policy) in target.

**AD federation 2-step elimination**:
1. AD Connector + trust = construction error (Connector is a proxy, can't have trusts)
2. Among "Managed AD + trust" options for IAM Identity Center → must be 2-way

**Cognito User Pool vs Identity Pool**:
- User Pool = authentication (JWT tokens)
- Identity Pool = AWS credentials (STS tokens for client-side AWS access)

---

## Behavioral patterns to stay alert to (your traps)

### 1. The over-engineering trap (6+ misses across sessions)

When two options share the same first component and one adds more services/policies:

**Counter-heuristic**: re-read the question's requirements. Does the simpler option fail any requirement? If no → simpler wins. If question says "least operational overhead," "efficient," "minimum effort" → simpler wins.

Past traps:
- NACL+subnet migration over SG mod (Q24)
- S3 batch over CloudWatch streaming (Q12)
- Service Catalog over CFN service role (Q32)
- Imported key material over symmetric AWS-generated (Q9)
- SSE-KMS over SSE-S3 when per-object keys requested (Q18)
- IAM role + resource-based policies over just IAM role + boundary (Q53)

### 2. KMS imported material rule (3+ misses)

> **Imported key material = MANUAL ROTATION ONLY. NEVER auto-rotates.**
>
> If the question says "automatic rotation" + customer wants control → eliminate imported material.
> If the question says "delete in under 7 days" → ONLY imported material can do this.

### 3. Event-driven vs state-based discriminators

| Question phrase | Pick |
|---|---|
| "As soon as," "real-time," "immediately," "uploaded" | Event-driven (EventBridge → SNS/Lambda) |
| "Find existing," "audit," "compliance posture" | State-based (Config, Access Analyzer) |
| "Single notification" / "consolidated" | Scheduled batch query, not per-event |
| "Per-event notification" | Event-driven |

### 4. Word-hunt question discriminators

Common keywords that flip the answer:
- "Single" vs "multiple"
- "Real-time" vs "scheduled"
- "Within X hours" — if X < 7 days → imported KMS material
- "Server must terminate TLS" → NLB TCP listener
- "Most efficient" / "minimum complexity" → simpler answer
- "All accounts" / "organization-wide" → SCP (not permissions boundary)
- "ALL access must use temporary credentials" → IAM Identity Center, not IAM users
- "Domains outside Route 53" → DNS validation works fine

### 5. Construction-error options (always wrong)

These option patterns are wrong on construction:
- `aws:SourceIp` with VPC endpoint traffic (doesn't fire)
- Security group in S3 bucket policy as Principal/Condition (not supported)
- Trust between AD Connector and on-prem AD (Connector is a proxy)
- Lambda IAM policy attached directly to function (must be on execution role)
- Resource policy with `Principal` claimed as identity-based policy
- EC2 detach root volume from running instance (must stop first)
- Private key in `authorized_keys` (public key goes there)
- `MaxSessionDuration` as condition key (it's a role attribute)
- `ec2:*VpcEndpoint*` in VPC endpoint policy (admin action, not runtime)
- EventBridge SES as target (doesn't exist; use SNS)
- "Cancel deletion" past the 7-30 day waiting period (irreversible)
- AWS Support restores deleted KMS keys (impossible)
- Root user has implicit access to KMS keys (false — must be in key policy)
- ALB MTU 9001 across regions (cross-region is 1500)

---

## Exam-day logistics (June 10, 10:30 AM)

### Morning checklist

- 7:30 — wake, coffee, protein breakfast
- 8:00–8:30 — light review of this cheatsheet ONLY (no new material)
- 8:45 — leave for testing center (30 min drive + buffer)
- 9:30 — arrive, check in, locker
- 10:30 — exam begins
- ~1:20 PM — exam ends

### What to bring

- TWO IDs matching your AWS Certification profile name exactly
- Confirmation email (printed backup)
- Comfortable layers (testing centers are cold)

### Strategy during the exam

- 170 minutes / 65 questions = ~2:36 per question
- Flag-and-return: don't get stuck. Mark and move on.
- Read keywords carefully on first pass — circle "real-time," "single," "all accounts," etc.
- Eliminate construction-error options first
- For ambiguous "which architecture": prefer simpler unless question demands defense in depth
- Last 30 minutes: review flagged questions

### The 5-rule mental checklist before submitting any answer

1. Did I read EVERY keyword in the question (especially "single," "all," "real-time," "least")?
2. Are any options construction errors I can eliminate?
3. Of the remaining options, does the simpler one meet the requirement?
4. Did I check for the over-engineering trap?
5. Did I match the question's domain context (D1 = detection, D2 = response, etc.)?

---

## Final reminders

- **D1 + D4 are your strongest** — don't over-study these, just maintain
- **D5 + D3 + D6 need the most attention** — 4 + 3 + 2 days respectively
- **Free retake voucher is your safety net** — take the exam confidently knowing it exists
- **Sleep 8+ hours nightly** — exam-day brain matters more than 1 extra hour of studying
- **No studying past 18:00 the day before exam** — taper hard

---

## ⚠️ COMMON TRAPS — recognize on sight, eliminate fast

This section is the single highest-value review item. Read it daily.

### Network direction traps

| Trap pattern | Why wrong | Right answer |
|---|---|---|
| "ALB-SG inbound from WebAppSG" | ALB receives from internet, not from web apps | ALB-SG inbound from `0.0.0.0/0`; outbound to WebAppSG |
| "EC2 needs outbound port 1024-65535 for ALB response" | SGs are stateful — return traffic is automatic | No outbound rule needed |
| "Use NACL for instance-level isolation" | NACL is subnet-level, affects all instances in subnet | Use SG for per-instance |
| "Source/dest check enabled on NAT/firewall instance" | AWS drops forwarded packets | **Disable source/dest check** on the ENI |
| "Cross-region peering with SG ID reference" | Cross-region doesn't support SG ID refs | Use CIDR instead |

### Route 53 / DNS traps

| Trap | Why wrong | Right answer |
|---|---|---|
| "CNAME for apex domain" | DNS RFC forbids CNAME at apex | Use Route 53 **Alias** record |
| "Route 53 geolocation routing blocks traffic" | Geolocation ROUTES, doesn't BLOCK | Use CloudFront geo-restriction or WAF geo match |
| "Latency routing for security" | Latency routing is for performance | Use geolocation or WAF for security |

### KMS traps (your stubborn miss area)

| Trap | Why wrong | Right answer |
|---|---|---|
| "Imported key material + automatic rotation" | Imported NEVER auto-rotates | Use AWS-generated symmetric for auto; manual rotation for imported |
| "Imported key material in Custom Key Store" | Custom stores don't accept imports | Default key store only for imports |
| "Schedule KMS key deletion within 24 hours" | Min waiting period is 7 days | Imported material via `DeleteImportedKeyMaterial` (immediate) |
| "Root user has KMS access by default" | Root has NO implicit access to KMS | Must be in key policy explicitly |
| "AWS Support restores deleted KMS keys" | Past waiting period = permanent | No recovery; gone forever |
| "Cancel KMS deletion past waiting period" | Only works DURING waiting period | Imported material can be re-imported; AWS-gen is gone |
| "KMS grants can be given to AWS service principals" | Grants need real principals (account/user/role) | Use key policy for services |
| "Encryption context generates different keys" | Context is metadata/AAD, not key generation | Use different CMK or DEKs (which are unique per object automatically) |

### CloudFront / ACM / cert traps

| Trap | Why wrong | Right answer |
|---|---|---|
| "Origin Request Policy forwards Authorization header" | Authorization header is BLOCKED in Origin Request Policy (HTTP 400) | **Cache Policy** for Authorization |
| "ACM-issued cert installed on EC2" | ACM-issued certs are NOT exportable | Imported cert or ACM Private CA cert |
| "CloudFront ACM cert in us-west-2" | CloudFront needs cert in us-east-1 ALWAYS | Request cert in us-east-1 |
| "Email validation auto-renews indefinitely" | Email requires manual click each cycle | DNS validation for indefinite auto-renewal |
| "AWS KMS for SSL/TLS certificate generation" | KMS is for at-rest encryption keys, not TLS certs | Use ACM for TLS certificates |

### IAM / Policy traps

| Trap | Why wrong | Right answer |
|---|---|---|
| "Identity-based policy with Principal element" | Only resource-based policies have Principal | Identity-based: no Principal (implicit) |
| "Resource-based policy without Principal" | Resource-based REQUIRES Principal | Add Principal block |
| "MaxSessionDuration as condition key" | It's a role attribute, not a condition key | Use `aws:MultiFactorAuthAge` for session age |
| "DateLessThan with numeric seconds value" | DateLessThan expects ISO 8601 timestamp | Use `NumericLessThan` with `aws:MultiFactorAuthAge` for seconds |
| "aws:SourceIp works through VPC endpoint" | Endpoint traffic doesn't carry original SourceIp | Use `aws:SourceVpc` or `aws:SourceVpce` |
| "Composite principal with every service in trust policy" | Trust policy defines who ASSUMES, not what role acts on | Single service principal in trust; broad actions in permissions |
| "AD Connector + forest trust" | AD Connector is a proxy, not a directory — no trust | Trust requires AWS Managed Microsoft AD |
| "One-way trust for IAM Identity Center" | Identity Center needs to enumerate users bidirectionally | Two-way trust required |
| "Global STS endpoint works in opt-in regions" | Global STS v1 tokens fail in opt-in regions | Use regional STS endpoints (v2 tokens) |
| "Excluded administrators from SCP" | SCPs apply to ALL principals in member accounts | Can't structurally exclude admins (only via Condition) |
| "Permissions boundary applied via Organizations" | Boundaries are per-principal in IAM, not via Orgs | Use SCP for org-wide |

### Compromised resource response traps

| Trap | Why wrong | Right answer |
|---|---|---|
| "Stop instance immediately when compromised" | Loses RAM forensics | Capture memory first (SSM Run Command), THEN stop |
| "Detach root EBS from running instance" | Impossible — root volume requires stopped instance | Stop first, or use snapshot approach |
| "Attach original suspect EBS to forensic instance" | Contaminates evidence | Snapshot first, attach the COPY |
| "Sign into compromised instance to investigate" | Tips off attacker, contaminates evidence | Investigate from clean diagnostics box |
| "Private key in EC2 authorized_keys" | authorized_keys takes PUBLIC keys | Add public key, keep private on client |
| "Revoke STS sessions BEFORE inactivating leaked key" | Attacker can just generate new sessions | Inactivate key FIRST, then revoke sessions |
| "Delete compromised IAM role immediately" | Destroys forensic trail | Disable + investigate, delete later |

### Security service capability traps

| Trap | Why wrong | Right answer |
|---|---|---|
| "Firewall Manager inspects traffic" | FW Manager is a MANAGEMENT tool — manages WAF/Shield/NF policies | Network Firewall does inspection |
| "GuardDuty blocks attacks" | GuardDuty DETECTS, doesn't block | Use WAF / Network Firewall / SG for blocking |
| "Macie monitors all S3 buckets globally" | Macie is regional, per-region enablement | Use Organizations delegated admin for cross-account |
| "AWS Inspector monitors API calls" | Inspector scans vulnerabilities, not API activity | Use CloudTrail + EventBridge for API monitoring |
| "Trusted Advisor sends real-time alerts" | TA does periodic checks, weekly email | Use CloudWatch/EventBridge for real-time |
| "Detective generates findings" | Detective is for investigation, not detection | GuardDuty/Inspector/Macie generate findings |
| "Security Hub is global" | Regional service, cross-region aggregation is opt-in | Designate aggregation region |
| "Access Analyzer blocks unauthorized access" | Access Analyzer DETECTS, doesn't block | Use bucket policy or SCP to block |

### Logging / monitoring traps

| Trap | Why wrong | Right answer |
|---|---|---|
| "ELB access logs to CloudWatch Logs" | ELB logs go to **S3 ONLY** | S3 + Athena for querying |
| "CW metric filter on S3 logs" | Metric filters work ONLY on CW Logs | Athena query + Lambda + PutMetricData |
| "Flow Logs capture endpoint-to-NLB PrivateLink traffic" | Flow Logs SKIP this traffic | No native logging — use Traffic Mirroring |
| "VPC Flow Logs capture packet content" | Metadata only | Use Traffic Mirroring for content |
| "S3 event notifications filter by ACL/permission" | S3 events filter only by event type/prefix/suffix | Use CloudTrail data events + EventBridge for ACL-based detection |
| "Public S3 detection via Access Analyzer in real-time" | Access Analyzer is state-based, not event-driven | Use CloudTrail data events for real-time |

### VPC endpoint traps

| Trap | Why wrong | Right answer |
|---|---|---|
| "VPC endpoint policy with `ec2:*VpcEndpoint*` action" | That's admin action for managing endpoints | Use runtime action (`s3:GetObject`, `execute-api:Invoke`, etc.) |
| "Enable private DNS to access public APIs" | Private DNS BREAKS public API access from VPC | Don't enable private DNS if you need public APIs |
| "Gateway endpoint for S3 from on-prem" | Gateway is VPC-local, doesn't work from on-prem | Use Interface endpoint for S3 |
| "SSM Session Manager with 2 endpoints (ssm + ssmmessages)" | Need THREE: ssm + ssmmessages + ec2messages | "SSM + two messages" mnemonic |

### Encryption in transit traps (recurring multi-fact)

| Trap | Why wrong | Right answer |
|---|---|---|
| "All intra-region traffic encrypted between EC2 instances" | Only Nitro instances; not universal | Depends on instance type |
| "Direct Connect encrypts traffic by default" | DX is private but unencrypted | Use VPN over DX or MACsec |
| "VPC endpoint traffic is encrypted at network layer" | Private path, but no automatic encryption | Rely on app-layer TLS |
| "Cross-region traffic needs VPN to be encrypted" | AWS auto-encrypts inter-region backbone traffic | No setup needed |

### Word-hunt discriminators (read carefully)

| Phrase in question | Forces what answer |
|---|---|
| "as soon as" / "immediately" / "real-time" / "uploaded" | Event-driven (EventBridge/CloudTrail) |
| "find existing" / "audit posture" | State-based (Config/Access Analyzer) |
| "single notification" / "consolidated" | Scheduled batch, not per-event |
| "within X hours" where X < 7 days | Imported KMS material (immediate delete) |
| "server must terminate TLS" | NLB TCP listener (passthrough), not TLS listener |
| "most efficient" / "least operational overhead" / "minimum effort" | Simpler answer wins; eliminate over-engineering |
| "all accounts" / "organization-wide" | SCP (not permissions boundary) |
| "delegate IAM role creation safely" | SCP + Permissions Boundary combo |
| "domains hosted outside Route 53" | DNS validation works fine (any DNS provider) |
| "apex/root domain pointing to AWS resource" | Route 53 Alias (not CNAME) |
| "single key per file/object" | SSE-S3 (already per-object); don't reach for SSE-KMS unless audit needed |
| "FIPS 140-2 Level 3 + customer manages HSM" | CloudHSM or KMS Custom Key Store |
| "import customer key material" | Default key store, customer-managed CMK |

### Construction errors (always wrong on sight)

These option patterns are wrong by construction — eliminate immediately:

- `aws:SourceIp` matching VPC endpoint traffic (doesn't fire)
- Security group ID as Principal in S3 bucket policy
- Trust policy between AD Connector and on-prem AD (Connector has no identity)
- IAM policy attached directly to Lambda function (must be on execution role)
- "Identity-based policy" answer that includes `Principal` element
- "Resource-based policy" answer without `Principal` element
- EC2 detach root volume from running instance (must stop first)
- Private key in EC2 `authorized_keys` (public key goes there)
- `MaxSessionDuration` used as condition key (it's a role attribute)
- `ec2:*VpcEndpoint*` action in VPC endpoint policy (admin action, not runtime)
- EventBridge SES as target (SES not supported; use SNS)
- "Cancel KMS deletion" past waiting period
- "AWS Support restores deleted KMS keys" (impossible)
- "Root user has implicit access to KMS keys" (false)
- Cross-region peering with SG ID reference (CIDR only)
- ALB MTU 9001 across regions (cross-region peering is 1500)
- CloudWatch metric filter on S3 logs (CW Logs only)
- Promiscuous mode on EC2 ENI (not supported)
- Importing key material into Custom Key Store (default store only)
- "VPC Peering" between a VPC and an AWS service (peering is VPC-to-VPC only)
- CNAME at apex/root domain (DNS-level violation)
- "Encryption context generates unique keys" (context is metadata, not key generation)
- "Single endpoint for SSM" in private subnet (need three)

---

*Updated June 1, 2026 — added Common Traps section based on Mock #2 misses (Q7, Q15, Q16, Q17, Q18, Q19, Q20, Q25)*

---

## ⚠️ Additional Traps — June 1 mini-test #2 misses

### RDS encryption — set at creation ONLY

> **You CANNOT enable encryption on an existing unencrypted RDS database.** Encryption can only be set when the DB is created. To "add encryption" to an existing DB:
>
> 1. Take a snapshot (will be unencrypted)
> 2. **Copy the snapshot** and during copy, specify encryption with a KMS key
> 3. **Restore** a new DB from the encrypted snapshot
> 4. Cut over apps (new endpoint), then terminate old DB

**Related traps**:
- ❌ "Modify storage settings to add KMS key" — not possible on existing DB
- ❌ "Create encrypted read replica of unencrypted master" — replicas inherit encryption state; unencrypted master → unencrypted replica only
- ❌ "Promote encrypted replica to primary" — impossible because you can't make an encrypted replica from an unencrypted master in the first place

**Same pattern applies to**: changing the KMS key on encrypted RDS (also requires snapshot+copy+restore with new key).

**Detection of unencrypted RDS**: AWS Config managed rule `rds-storage-encrypted` + SNS notification.

### EC2 key pair rotation — `authorized_keys` edit required

> **You CANNOT change an EC2 instance's SSH key via any AWS API after launch.** Rotation requires editing the OS-level `~/.ssh/authorized_keys` file on the instance itself.

**Procedure to rotate / revoke SSH access**:
1. Generate new key pair in EC2 console
2. Connect to instance (via existing key, Session Manager, or EC2 Instance Connect)
3. Edit `~/.ssh/authorized_keys`:
   - Add new public key
   - Remove old public key (to revoke access)
4. Test connection with new key BEFORE closing existing session
5. Update any automation/scripts that referenced the old key

**Construction errors to eliminate**:
- ❌ `modify-instance-attribute` API to change key pair — no such option
- ❌ "Use EC2 console to change key pair on running instance" — console only shows the key at launch time; can't be changed post-launch
- ❌ "Modify key pair in AMI" — only affects FUTURE instances launched from the AMI, not existing ones
- ❌ "Delete and recreate key pair in EC2 console" — deletes the AWS-stored key pair record but doesn't remove the public key from any running instance's `authorized_keys`

**Use case**: departing employee with copy of private key → SSH in, replace `authorized_keys`, kill their access immediately.

### AWS Organizations does NOT have password policy

> **AWS Organizations does NOT support org-wide password policies.** Password policies are configured per-account in IAM (for IAM users), in the IdP for federated users, or in Cognito user pool for Cognito users.

**Password policy locations**:

| User type | Where password policy lives |
|---|---|
| **IAM users** | IAM → Account settings → Password policy (per-account) |
| **Federated (SAML/AD/Okta)** | At the upstream IdP — not in AWS |
| **Amazon Cognito User Pool** | In the User Pool configuration |
| **Amazon Cognito Identity Pool** | N/A (identity pools don't have users) |
| **IAM Identity Center store** | In Identity Center configuration |

**Construction errors to eliminate**:
- ❌ "Set password policy in AWS Organizations" — not a feature
- ❌ "Set IAM password policy for federated users" — IAM users only; federation goes to IdP
- ❌ "Configure password policy in Cognito Identity Pool" — identity pools authorize, don't authenticate

### AWS Config + S3 + SNS — three-policy setup

For Config to deliver findings to S3 and notify via SNS, you need permissions at THREE places:

| Layer | What |
|---|---|
| **1. Config role TRUST policy** | Allow `config.amazonaws.com` to assume the role |
| **2. Config role IDENTITY policy** | Grant `s3:PutObject` on the target bucket |
| **3. SNS topic RESOURCE policy** | Allow `config.amazonaws.com` to `sns:Publish` |

**Common traps**:
- ❌ "S3 bucket ACL to allow Config" — ACLs not used for service access
- ❌ "Trust policy allowing `s3.amazonaws.com`" — Config assumes the role, not S3
- ❌ "SNS access policy with `sns:write`" — correct action is `sns:Publish`, not `sns:write`

If any one layer is missing → Config doesn't deliver. Always check all three.

### API Gateway usage analytics — access logging required

> **API Gateway access logs are NOT enabled by default.** To analyze API usage with CloudWatch Logs Insights, you must FIRST enable access logging on the stage.

**Two-step setup**:
1. **Enable access logging** on the API stage → choose CloudWatch Logs as destination, configure log format ($context variables)
2. **Query with CloudWatch Logs Insights** for analysis

**Distinction from other API Gateway logs**:

| Log type | What it captures | Use for |
|---|---|---|
| **Access logs** | Per-request data (status, latency, user agent, source IP) | Usage analytics, troubleshooting |
| **Execution logs** | API Gateway internal processing details | Debugging API Gateway behavior |
| **CloudTrail logs** | Management API calls (create/update/delete API) | Audit of who configured what |
| **Detailed CloudWatch Metrics** | Per-method metrics (count, latency, 4xx, 5xx) | Dashboarding, alarms |

**Construction errors**:
- ❌ "Enable Detailed Metrics for usage analysis" — gives counts/latency but not per-request usage data
- ❌ "Use CloudTrail for API usage" — CloudTrail captures management plane, not data plane (API invocations)

### Parameter Store + KMS troubleshooting

When using Parameter Store SecureString with a customer-managed KMS key, common error causes:

**Real causes (these break it)**:
- ❌ Application IAM role lacks `kms:Encrypt` / `kms:Decrypt` permission on the key
- ❌ Key state is `Disabled` (or `PendingDeletion` / `PendingImport`)
- ❌ KMS key policy doesn't allow the calling principal
- ❌ Cross-region: key is in different region than the parameter
- ❌ Cross-account: key policy doesn't allow other account
- ❌ Encryption context mismatch (if context was used at encrypt time)

**NOT causes (these are red herrings)**:
- ✅ Key alias vs key ID — both work interchangeably
- ✅ Multiple secrets using same key — KMS keys can encrypt unlimited secrets
- ✅ Standard tier vs Advanced tier — both support customer-managed KMS keys
- ✅ Parameter name format — no relation to KMS errors

### Direct Connect encryption (reinforce — recurring miss)

> **Direct Connect is PRIVATE but NOT encrypted.** Private path ≠ encrypted path.

To encrypt traffic over DX:
- **Option A**: Site-to-Site VPN over DX → IPsec encryption at L3
  - Requires creating a **Virtual Private Gateway (VGW)** on the AWS side
  - VPN tunnel runs over the DX connection
  - Slight latency overhead from IPsec processing
- **Option B**: MACsec (Layer 2 encryption)
  - Available on Dedicated Connections of 10 Gbps or 100 Gbps
  - Hardware-accelerated, no software overhead
  - Requires MACsec-capable routers on both sides

**Construction errors**:
- ❌ "Enable IPSec on the Direct Connect connection" — DX doesn't have an IPSec option; you create a VPN over DX
- ❌ "Add KMS key to Direct Connect" — DX has no KMS integration; KMS is for data-at-rest
- ❌ "Configure Direct Connect Gateway for encryption" — DX Gateway is for connecting to multi-region VPCs, NOT encryption

### Recurring SG direction trap (3x missed today — must lock in)

> **The instance that RECEIVES traffic needs the inbound rule. The instance that SENDS traffic doesn't need an outbound rule (default allow-all + stateful).**

For a 3-tier app (Web → App → DB):

| Security Group | Direction | Rule |
|---|---|---|
| **WebSG** | Inbound | 80/443 from `0.0.0.0/0` (or CloudFront IPs) |
| **AppSG** | Inbound | App port from **WebSG** ← (Web initiates to App) |
| **DBSG** | Inbound | DB port (e.g., 3306) from **AppSG** ← (App initiates to DB) |

**Critical "who initiates?" reflex**:
- Web → App: Web initiates → App needs inbound from WebSG
- App → DB: App initiates → DB needs inbound from AppSG
- ❌ "AppSG inbound from DBSG" — WRONG direction (DB doesn't initiate to App)
- ❌ "WebSG inbound from AppSG" — WRONG direction (App doesn't initiate to Web)

**Mnemonic**: traffic flows DOWN the stack (Web→App→DB), so inbound rules cascade DOWN (App accepts from Web, DB accepts from App).

### Explicit Deny overrides Explicit Allow (core IAM evaluation)

> **An explicit Deny in ANY policy always overrides any Allow, regardless of where the Allow comes from.**

When troubleshooting "user can't access X" issues, check Deny statements first:
1. **Bucket policy explicit Deny** — checked even before IAM policy Allows
2. **SCP explicit Deny** — caps the entire account
3. **IAM identity policy explicit Deny** — caps the user/role
4. **Permissions boundary** — caps the role
5. **Session policy** — caps the session

**Common scenario**: lockdown policy applied during incident has a wide Deny, then later you try to grant exception via IAM policy → still denied because the bucket policy's Deny overrides.

**Fix**: modify the Deny statement to exclude the exception (using Condition with `aws:PrincipalArn` NotEquals), don't try to "out-allow" it.

---

*Updated June 1, 2026 (later) — added gaps from mini-test #2: RDS encryption, EC2 key rotation, Orgs password policy, Config tri-policy, API Gateway access logs, Parameter Store + KMS troubleshooting, DX encryption reinforcement, SG direction reinforcement, explicit Deny rule*

*Last updated: May 30, 2026*
