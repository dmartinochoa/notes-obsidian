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

| Control | Scope | Set via | Use case |
|---|---|---|---|
| **SCP** | Org / OU / account-wide | AWS Organizations | "No one in this OU can do X" |
| **Permissions Boundary** | Per-principal | IAM (on the user/role) | "This role can never exceed Y" (delegation pattern) |
| **RCP** (Resource Control Policy, newer) | Org / OU / account-wide on resources | AWS Organizations | Restricts external access TO your resources |

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

*Last updated: May 30, 2026*
