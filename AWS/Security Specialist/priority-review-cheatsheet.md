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

When Secrets Manager rotates a secret it invokes a Lambda that runs through these 4 stages **in order**:

| Step | What happens |
|---|---|
| **`createSecret`** | Lambda generates a new secret value and stores it as a new version of the secret labeled **AWSPENDING** |
| **`setSecret`** | Lambda applies the new secret to the actual service (e.g., updates the password on the RDS user) |
| **`testSecret`** | Lambda verifies the new value works (test connection with new password) |
| **`finishSecret`** | Lambda swaps the **AWSCURRENT** label from the old version to the new version (old becomes AWSPREVIOUS) |

**Version labels (staging labels)** — these are pointers Secrets Manager maintains:
- **AWSCURRENT** — the "active" version that apps retrieve by default (calling `GetSecretValue` returns this)
- **AWSPENDING** — the new candidate version during rotation (not yet active)
- **AWSPREVIOUS** — the previous version (kept for rollback)

**Auto-rotation built-in support** is LIMITED. Secrets Manager ships ready-made Lambda rotation templates for:
- Amazon RDS (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server)
- Amazon DocumentDB
- Amazon Redshift

**For anything else (SSH keys, API tokens, third-party credentials, OAuth keys, etc.):**
- You must **write your own Lambda function** implementing the 4 steps above
- Schedule it via the rotation configuration (cron-like rotation interval)
- **Audit always goes through CloudTrail** — Secrets Manager doesn't have built-in audit logging to S3

**Cross-account access**: attach a **resource policy** to the secret allowing the other account's principal, plus that principal needs KMS permissions on the encryption key.

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

A VPC endpoint policy is **attached to the endpoint object itself** and controls which API actions can be invoked through that endpoint.

**Two categories of action — only one belongs in an endpoint policy:**

| Action category | Example | Belongs in endpoint policy? |
|---|---|---|
| **Runtime action** (data plane) — the actual service operations | `s3:GetObject`, `kms:Decrypt`, `execute-api:Invoke`, `dynamodb:GetItem`, `sqs:SendMessage` | ✅ **YES** — this is what flows through the endpoint |
| **Admin action** (control plane) — managing the endpoint resource itself | `ec2:CreateVpcEndpoint`, `ec2:DeleteVpcEndpoint`, `ec2:ModifyVpcEndpoint` | ❌ NO — those go in IAM policies, never in endpoint policy |

If you see `ec2:*VpcEndpoint*` in an endpoint policy → instant elimination (construction error).

**Other mechanics**:
- **Default endpoint policy = `"Action": "*"` on `"Resource": "*"`** — full passthrough until you replace it
- Endpoint policy is **additive ANDed** with IAM identity policy and resource-based policy — all must allow
- When counting resources for a policy, **count distinct resource ARNs**, not duplicate references
- Endpoint policy doesn't grant — it caps (same model as SCPs)

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

Reads as: at any layer, an Explicit Deny wins. Otherwise, the action must be allowed at every layer that applies (SCP if in an Org, resource policy if cross-account, identity policy, boundary if attached, session policy if applicable). If nothing explicitly allows → implicit Deny.

**`iam:PassRole`** vs **`sts:AssumeRole`** (commonly confused):
- **PassRole** = user HANDS a role to a SERVICE so the service can use it. Examples: when you create a Lambda function and specify its execution role, you need `iam:PassRole` on that role. Same for EC2 instance profiles, ECS task roles, CodeBuild service roles.
- **AssumeRole** = principal BECOMES a role temporarily, getting back STS credentials. Examples: cross-account access ("switch role"), federated user gets IAM credentials, EC2 instance retrieves credentials from its instance profile.
- **Quick rule**: If the role is being given to a service to wear, that's PassRole. If a user/service is taking on the role themselves, that's AssumeRole.

**Permissions Boundary use case** — the delegation pattern:
- Let a team create new IAM roles (so they don't bottleneck on the security team)
- BUT cap what those new roles can do (so they can't create god-mode roles)
- Implement via an SCP requiring all roles created to attach a specific boundary
- Example SCP condition: `"iam:PermissionsBoundary": "arn:aws:iam::123:policy/DevBoundary"`

**STS regional vs global endpoints**:
- **Global STS** (`sts.amazonaws.com`) = legacy single endpoint in us-east-1, returns **v1 session tokens** (smaller). v1 tokens **fail in opt-in regions** (Bahrain, Cape Town, Hong Kong, Jakarta, Hyderabad, Melbourne, etc.) because those regions don't recognize them.
- **Regional STS** (`sts.<region>.amazonaws.com`) = per-region endpoint, returns **v2 session tokens**. Work in ALL regions including opt-in.
- **Best practice**: set SDK to use regional endpoints (`AWS_STS_REGIONAL_ENDPOINTS=regional`).

**Condition key operator-value typing** — get the operator right or the policy silently fails open:
- `DateLessThan` / `DateGreaterThan` need **ISO 8601 timestamp** like `2026-12-31T23:59:59Z`, NOT a number of seconds
- `aws:MultiFactorAuthAge` is **Numeric** (seconds since MFA auth) — use `NumericLessThan` operator
- `MaxSessionDuration` is **NOT a condition key** — it's an **attribute of the role itself**, set at role creation. It defines the upper bound; the actual session duration is set by `DurationSeconds` on `AssumeRole` call. Don't use it in a policy Condition.

**IAM Paths** = a directory-like prefix in an IAM resource ARN. Example: a role named `DevOpsAdmin` with path `/platform/` has ARN `arn:aws:iam::123:role/platform/DevOpsAdmin`. Paths let you:
- Use wildcards in IAM policies (`arn:aws:iam::*:role/platform/*` matches all roles under /platform/)
- Implement the delegation pattern: "Team A can create roles only under `/teamA/`" via SCP/boundary using path-based wildcards
- Visually organize roles (no functional effect beyond ARN pattern matching)

**Identity-based policy vs Resource-based policy** — fastest elimination signal:
- **Identity-based** (attached to user/group/role): NO `Principal` element (the principal is the entity it's attached to)
- **Resource-based** (attached to S3 bucket, KMS key, SQS queue, IAM role trust policy, etc.): REQUIRES `Principal` element
- If a question says "identity-based policy" and the JSON has `Principal` → construction error, eliminate
- If a question says "resource-based policy" and the JSON has no `Principal` → construction error, eliminate

**Cross-account access requires BOTH**:
- The source account: identity policy on the principal (user/role) allowing the action
- The target account: resource policy (or role trust policy if assuming a role) allowing the source principal
- Either side alone is insufficient. This is why cross-account is "two-key auth."

**AD federation 2-step elimination**:
1. **AD Connector + trust** = construction error. AD Connector is just a **proxy/gateway** to your on-prem AD; it has no directory of its own. You cannot establish a forest/realm trust with a proxy.
2. Among "Managed Microsoft AD + trust" options for IAM Identity Center → it must be a **2-way trust** (Identity Center needs to enumerate users/groups bidirectionally for assignments). One-way trusts work for some scenarios but not Identity Center user enumeration.

**Cognito User Pool vs Identity Pool**:
- **User Pool** = AUTHENTICATION service (a managed user directory). Returns **JWT tokens** (ID token, access token, refresh token). Use it for app sign-up/sign-in.
- **Identity Pool** (now called Cognito Federated Identities) = AUTHORIZATION exchange. Takes an authenticated token (from User Pool, Google, Facebook, SAML, etc.) and returns **temporary AWS STS credentials** so the client can directly call AWS services like S3, DynamoDB.
- **Typical mobile/SPA flow**: User Pool authenticates → app calls Identity Pool → gets AWS creds → makes scoped S3 calls.

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

These option patterns are wrong on construction — recognize and eliminate instantly:

- **`aws:SourceIp` with VPC endpoint traffic** — when traffic flows through a VPC endpoint (gateway or interface), the source IP visible to the service is the endpoint's internal IP, NOT the original caller. `aws:SourceIp` condition won't match expected values. Use `aws:SourceVpc` or `aws:SourceVpce` instead.
- **Security group ID as Principal in S3 bucket policy** — S3 bucket policies can only have IAM principals (account, user, role, federated, service) as Principal. SGs are network-level constructs unknown to S3.
- **Trust between AD Connector and on-prem AD** — AD Connector is just a proxy/gateway with no directory of its own. Forest trusts require a real directory. Use AWS Managed Microsoft AD instead.
- **Lambda IAM policy attached directly to function** — Lambda IS the resource. The function HAS an execution role. Identity policies attach to that role, not to the function itself. (Resource-based "function policies" exist for invocation permissions but that's different.)
- **Resource policy with `Principal` claimed as "identity-based policy"** — only resource-based policies have `Principal`. If the question labels a policy with Principal as "identity-based" → construction error.
- **EC2 detach root volume from running instance** — root EBS volume cannot be detached from a running instance because the OS is actively using it. Must STOP the instance first.
- **Private key in EC2 `authorized_keys`** — `authorized_keys` holds PUBLIC keys (the OS uses them to verify signatures from clients holding the matching private key). Private keys never leave the client.
- **`MaxSessionDuration` as condition key** — it's a role attribute set at role creation (max value an AssumeRole call's `DurationSeconds` can request). Not a condition key for use in policies.
- **`ec2:*VpcEndpoint*` in VPC endpoint policy** — those are EC2 admin actions (CreateVpcEndpoint, etc.). Endpoint policies hold the RUNTIME actions of the target service (`s3:GetObject`, `kms:Decrypt`, etc.).
- **EventBridge SES as target** — Amazon SES is not a direct target type for EventBridge rules. Use SNS as the target, then SNS can email or trigger Lambda.
- **"Cancel deletion" past the 7-30 day waiting period** — once the KMS key deletion completes, it's permanent. Cancel only works DURING the waiting period via `CancelKeyDeletion`.
- **AWS Support restores deleted KMS keys** — impossible. Past the waiting period, the key material is gone. (Imported material is the only exception — you can re-import the same key material into a NEW key, but the old key ID is gone.)
- **Root user has implicit access to KMS keys** — false. KMS is unique: root has NO implicit access. The default key policy includes a statement granting root access — delete that statement and you're locked out (AWS Support recovery only).
- **ALB MTU 9001 across regions** — jumbo frames (9001 MTU) work within a region but cross-region VPC peering forces MTU 1500. No way to negotiate jumbo across regions.
- **Promiscuous mode on EC2 ENI** — promiscuous mode (accepting packets not addressed to you) is not supported on AWS ENIs. AWS hypervisor filters traffic by destination MAC/IP. Use VPC Traffic Mirroring for packet capture instead.
- **"Composite principal with every AWS service" in trust policy** — a trust policy's Principal defines what can ASSUME the role. Listing every service principal there is nonsensical (only services actually assuming the role belong). Trust policy is narrow; the role's permissions policy is what's broad.
- **"Excluded administrators" from SCP** — SCPs apply to ALL principals in member accounts including root and admins. You cannot structurally exclude admins. The only way to exempt specific principals is with a `Condition` using `aws:PrincipalArn` NotEquals (and that gets fragile).

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

These option patterns are wrong by construction — eliminate immediately. Each is annotated with WHY so you can recognize related variants:

- **`aws:SourceIp` matching VPC endpoint traffic** — endpoint traffic loses the original source IP; use `aws:SourceVpc` / `aws:SourceVpce` instead
- **Security group ID as Principal in S3 bucket policy** — SGs are L4 network constructs; S3 policies only accept IAM principals
- **Trust policy between AD Connector and on-prem AD** — AD Connector is a proxy with no directory; nothing to trust. Use AWS Managed Microsoft AD for trusts.
- **IAM policy attached directly to Lambda function** — policies attach to the execution ROLE, not the function. (Resource-based "function policy" controls who can INVOKE, but that's a different thing.)
- **"Identity-based policy" answer that includes `Principal`** — identity-based policies imply the principal (the entity they're attached to); they never have a Principal element
- **"Resource-based policy" answer without `Principal`** — resource policies REQUIRE Principal (defines who is allowed to use the resource)
- **EC2 detach root volume from running instance** — the OS holds it open; must stop first. Data volumes can be detached live.
- **Private key in EC2 `authorized_keys`** — `authorized_keys` holds PUBLIC keys for verification; private keys stay with the client
- **`MaxSessionDuration` used as condition key** — it's a role attribute (max value of DurationSeconds at AssumeRole), not a condition
- **`ec2:*VpcEndpoint*` action in VPC endpoint policy** — those are admin actions for managing the endpoint; endpoint policies use runtime actions like `s3:GetObject`
- **EventBridge SES as target** — SES isn't a target type. Use SNS, Lambda, or Step Functions; they can call SES.
- **"Cancel KMS deletion" past waiting period** — `CancelKeyDeletion` only works while the key is still in `PendingDeletion` state (during the 7-30 day window)
- **"AWS Support restores deleted KMS keys"** — KMS key material is gone after deletion completes. Imported keys allow re-importing the same material into a NEW key, but the original key ID is dead.
- **"Root user has implicit access to KMS keys"** — KMS is unique. Root has access ONLY because the default key policy grants it. Delete that statement → root locked out (AWS Support recovery is the only way back).
- **Cross-region peering with SG ID reference** — cross-region peering only supports CIDR-based SG rules, not SG ID references
- **ALB MTU 9001 across regions** — cross-region traffic is capped at MTU 1500 regardless of instance type
- **CloudWatch metric filter on S3 logs** — metric filters only work on CW Logs log groups. For S3 → Athena query + Lambda + PutMetricData.
- **Promiscuous mode on EC2 ENI** — AWS hypervisor filters by destination; promiscuous capture isn't possible. Use VPC Traffic Mirroring for packet capture.
- **Importing key material into Custom Key Store (CloudHSM-backed)** — Custom Key Store keys must be generated INSIDE the HSM. Imports only work in the default key store.
- **"VPC Peering" between a VPC and an AWS service** — peering is VPC-to-VPC only. Use VPC endpoints for VPC-to-service.
- **CNAME at apex/root domain** — DNS RFC forbids CNAME at the zone apex (e.g., `example.com` cannot be CNAME). Use Route 53 Alias record (an Alias looks like A or AAAA from the resolver's view).
- **"Encryption context generates unique keys"** — encryption context is additional authenticated data (AAD) used for integrity verification and audit. It does NOT influence the key derivation. Unique keys per object come from DEKs (data encryption keys), which SSE-S3 / SSE-KMS generate per object automatically.
- **"Single endpoint for SSM" in private subnet** — Session Manager needs THREE interface endpoints: `ssm`, `ssmmessages`, `ec2messages` ("SSM + two messages")
- **"Restrict outbound traffic by modifying IAM role"** — IAM controls authorization to AWS APIs, not network traffic flow. Outbound network restriction = security groups, NACLs, route tables, or VPC config (in case of Lambda).

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

---

## ⚠️ June 3 mini-test misses (80% — 5 specific gaps)

Mini-test #3 on June 3 returned 80% (20/25). All 5 misses below were specific factual gaps rather than recurring patterns. Each one is a single, well-defined fact worth memorizing.

### Access key rotation — the disable-test-delete sequence

When you need to rotate an access key (employee left, key leaked, periodic rotation), **the order matters**. Wrong order = downtime or unsafe rollback.

**The canonical 5-step sequence**:

```
1. CREATE new access key (old key still active)
2. UPDATE applications to use new key
3. DISABLE old key  ←  NOT delete yet
4. TEST that applications still work with old key disabled
5. DELETE old key  ←  only after testing confirms success
```

**Why disable-not-delete in step 3**:
- **Disable is reversible** (re-enable in seconds if something breaks)
- **Delete is permanent** — once gone, no rollback
- The window between disable and delete is your safety net to discover any missed application that still uses the old key

**The seductive trap**: "Disable old key and revoke all active sessions."
- Access keys are **long-lived credentials**, not session tokens. They don't have "sessions" to revoke.
- The "revoke sessions" answer pattern appears for STS temporary credentials (an `aws:TokenIssueTime` condition can effectively invalidate sessions issued before a cutoff), but that's a different scenario.
- For an IAM user access key compromise: disable the key (or rotate). The key itself IS the credential.

**Construction-error variants**:
- ❌ "Delete the old key immediately, then create the new one" — causes downtime + no rollback
- ❌ "Delete the old key and revoke active sessions" — wrong concept (keys don't have sessions)
- ❌ "Apply IAM policy disabling the user's temporary credentials" — keys aren't temporary credentials
- ❌ "Use Lambda to rotate without testing" — skipping the test step risks production breakage

**For STS / temporary credentials** (different scenario): use `aws:TokenIssueTime` condition with `DateGreaterThan` to invalidate sessions issued before a cutoff timestamp.

### CloudFront SecurityHeadersPolicy — the managed response headers policy for MITM/XSS defense

CloudFront ships **managed response headers policies** that you can attach to a distribution without writing custom Lambda@Edge. These are pre-built bundles of headers.

**Key managed response headers policies**:

| Policy name | What it adds | Defends against |
|---|---|---|
| **SecurityHeadersPolicy** | HSTS + Content-Security-Policy + X-Content-Type-Options + X-Frame-Options + Referrer-Policy + X-XSS-Protection | MITM (via HSTS), XSS, clickjacking, MIME sniffing |
| **CORS-and-SecurityHeadersPolicy** | Above + CORS headers | When you need both |
| **SimpleCORS** | Permissive CORS only | Simple cross-origin allow |
| **CORS-With-Preflight** | CORS + preflight handling | Cross-origin POST/PUT with auth |
| **CORS-With-Preflight-and-SecurityHeadersPolicy** | All combined | Full coverage |

**HSTS specifically defends against MITM**:
- HTTP Strict-Transport-Security header tells browsers "always use HTTPS for this domain for X seconds"
- After first visit, the browser refuses to talk plaintext HTTP to that origin — even if the user types `http://`
- Prevents SSL-stripping attacks where an attacker downgrades HTTPS to HTTP in transit

**Question-to-answer mapping**:

| Question phrase | Correct CloudFront answer |
|---|---|
| "Defend against man-in-the-middle" | **SecurityHeadersPolicy** (for HSTS) |
| "Defend against XSS / clickjacking" | SecurityHeadersPolicy (for CSP + X-Frame-Options) |
| "Cross-origin AJAX request" | CORS policy variant |
| "Add custom security header not in any managed policy" | Custom response headers policy (or Lambda@Edge) |
| "Lambda@Edge to inject HSTS" | ❌ Over-engineered when SecurityHeadersPolicy exists |

**Construction-error variants**:
- ❌ "BasicCORS to defend against MITM" — CORS is about cross-origin requests, not transport security
- ❌ "Content-Security-Policy alone defends against MITM" — CSP defends against XSS/injection, not MITM
- ❌ "Use Lambda@Edge for HSTS" — works but managed policy is simpler (less operational overhead)
- ❌ "Geo restriction to defend against MITM" — geo restriction blocks countries, not active interception

### Secrets Manager rotation — built-in scope is DATABASE-ONLY

Secrets Manager has TWO concepts that get confused:

1. **Built-in rotation templates** — AWS provides ready-made Lambda functions for specific services
2. **Custom rotation** — you write your own Lambda following the 4-step contract

**Built-in rotation supports ONLY these services**:
- Amazon RDS (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server)
- Amazon DocumentDB
- Amazon Redshift

**For anything else** (SSH keys, third-party API keys, OAuth tokens, generic credentials), the answer pattern is always:

```
1. Store secret in Secrets Manager
2. Write CUSTOM Lambda function implementing the 4-step rotation contract
3. Configure rotation schedule (e.g., every 90 days) → Lambda fires
4. CloudTrail captures all Secrets Manager API calls → S3 for audit
```

**Audit logging clarification**:
- Secrets Manager does NOT have built-in audit logging to S3
- All audit goes through **CloudTrail** (which captures Secrets Manager management + data events)
- You configure CloudTrail to deliver to an S3 bucket
- For per-secret access tracking, enable CloudTrail data events for Secrets Manager

**Question-to-answer mapping**:

| Question phrase | Right approach |
|---|---|
| "Rotate RDS password automatically" | Built-in Secrets Manager rotation |
| "Rotate SSH key pairs every 90 days" | Secrets Manager + **custom Lambda** + CloudTrail |
| "Rotate third-party API token" | Secrets Manager + custom Lambda + CloudTrail |
| "Audit trail of secret access" | CloudTrail (not Secrets Manager native logging) |

**Construction-error variants**:
- ❌ "Enable automatic rotation for SSH keys in Secrets Manager" — auto-rotation doesn't support SSH keys
- ❌ "Use Secrets Manager native audit logging" — Secrets Manager doesn't have native audit logging
- ❌ "Use EC2 console to enable SSH key auto-rotation" — no such EC2 feature exists
- ❌ "CloudWatch alarm on key age + Lambda to delete" — deleting keys without replacement = lockout

### IAM Role anatomy — Trust Policy vs Permissions Policy

A role has **TWO categories** of policy attached, and they answer different questions:

| Policy type | Question it answers | How many? | Has Principal? |
|---|---|---|---|
| **Trust policy** (a.k.a. assume role policy) | WHO is allowed to assume this role? | **Exactly ONE** per role | ✅ YES (defines who can assume) |
| **Permissions policy** (identity policy) | WHAT can this role do once assumed? | **0 to N** per role | ❌ NO (identity-based) |

**Mental model — the role is a "doorway":**
- **Door (trust policy)** → who has the key to enter? Defines the Principal.
- **Room (permissions policy)** → what's available inside? Defines the actions/resources.

**Trust policy example** (allows EC2 service to assume):
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

**Permissions policy example** (what the role can do once assumed — S3 read):
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"]
  }]
}
```

**CLI mapping for `create-role`**:

| Step | Command | What it does |
|---|---|---|
| 1. Create the role | `aws iam create-role --role-name X --assume-role-policy-document file://trust.json` | The `--assume-role-policy-document` parameter IS the trust policy. REQUIRED. |
| 2. Attach managed permissions | `aws iam attach-role-policy --role-name X --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess` | Adds an AWS-managed permissions policy |
| 3. Or inline permissions | `aws iam put-role-policy --role-name X --policy-name MyInline --policy-document file://perms.json` | Adds an inline permissions policy |

**Policy attachment types (orthogonal to trust vs permissions)**:

| Attachment style | What it means | Reusable? |
|---|---|---|
| **AWS-managed** | Policy authored and maintained by AWS (e.g., `AmazonS3ReadOnlyAccess`) | Yes, across many entities |
| **Customer-managed** | Policy authored by you, stored as a standalone policy ARN | Yes, across many entities |
| **Inline** | Policy embedded directly in a single user/group/role | No, lives only on that entity |

**The exam trap**: "What policy is needed to allow EC2 to assume the role?" → **Trust policy**, NOT inline policy. Inline/managed/customer-managed are attachment styles for the permissions policy (the WHAT), not the trust mechanism (the WHO).

**Construction-error variants**:
- ❌ "Bucket policy" allows role assumption — bucket policy controls S3 bucket access, not role assumption
- ❌ "Inline policy" allows role assumption — inline is just an attachment style; the trust policy is what governs assumption
- ❌ "Managed policy" defines who can assume — managed/inline/customer-managed all describe permissions policies
- ❌ Trust policy contains `Action: s3:GetObject` — trust policy actions are always `sts:AssumeRole` family

### Lambda resource policy ≠ outbound control

**Lambda has TWO permission concepts** that get conflated:

| Concept | What it controls | Where attached |
|---|---|---|
| **Execution role** (identity policy) | What the Lambda CAN DO (outbound API calls to S3, DDB, etc.) | Attached to the role |
| **Function policy** (resource-based) | WHO can INVOKE the Lambda (other AWS services, accounts) | Attached to the function itself |

**Neither of these controls network outbound (internet) access**. That's a VPC-level concern.

**Lambda's network model — TWO states**:

| State | Network behavior | Internet? | Private VPC resources? |
|---|---|---|---|
| **Default (no VPC config)** | Runs in AWS-managed VPC | ✅ Yes (Lambda gets internet by default) | ❌ Cannot reach your VPC's private resources |
| **VPC-attached** | Runs in your VPC subnets with ENIs | ❌ No (unless NAT Gateway added) | ✅ Yes (private subnet access) |

**The exam pattern**: "Lambda must access private DynamoDB without internet exposure."

**Right answer**:
1. **Attach Lambda to a private subnet** in the VPC (now it has private VPC access, but no internet)
2. **Add a VPC endpoint for DynamoDB** (Gateway endpoint — free) so Lambda can reach DynamoDB without leaving the VPC

**Common confusion**:
- ❌ "Use a resource policy on Lambda to block internet" — function policy controls INVOCATION, not outbound. There's no policy that controls Lambda's outbound internet.
- ❌ "Configure DynamoDB to connect to a private subnet" — DynamoDB is a managed service; it doesn't sit in your subnet
- ❌ "Use a resource policy on DynamoDB to restrict the VPC" — works (using condition keys) but unnecessary; VPC endpoint policy is the simpler answer
- ❌ "Add NAT Gateway to prevent internet exposure" — NAT Gateway ENABLES outbound internet; doesn't restrict it

**Lambda invocation patterns** (where function policy DOES matter):

| Scenario | Use function policy? |
|---|---|
| S3 event triggers Lambda | ✅ Allow `s3.amazonaws.com` in function policy |
| API Gateway invokes Lambda | ✅ Allow `apigateway.amazonaws.com` in function policy |
| EventBridge rule targets Lambda | ✅ Allow `events.amazonaws.com` in function policy |
| Cross-account invocation | ✅ Allow the other account's principal in function policy |
| Lambda function URLs (public/restricted) | ✅ Configure auth and function policy |
| Lambda calls other AWS services | ❌ That's the execution role's identity policy, not the function policy |
| Restrict Lambda's outbound internet | ❌ Not possible via any policy; use VPC config |

---

## 🧭 Cross-cutting reflexes — synthesis of all sessions

After three mock tests, the patterns that explain >80% of my misses:

| Reflex | What to do | Triggers |
|---|---|---|
| **"Disable, then delete"** | When rotating credentials/keys, disable old before deleting | Access key rotation, RDS snapshot, etc. |
| **"Where does the policy live?"** | Identify trust vs identity vs resource vs endpoint vs SCP vs boundary | Any policy question |
| **"Does the action match the layer?"** | IAM controls API authz; SG controls network; endpoint policy filters endpoint traffic | "Block internet" / "restrict access" questions |
| **"Is this the over-engineered option?"** | Simpler usually wins unless requirements demand complexity | "Minimum overhead" / "easily" / "simple" |
| **"What's the question's keyword?"** | "Real-time" → events; "audit posture" → state; "single" / "all" — read literally | Every word-hunt question |
| **"Construction error check"** | Eliminate impossible-by-design options first | Speed up by 30s/question |
| **"Layer all the layers"** | Network reachability + IAM + endpoint policy + (resource policy if cross-account) | Endpoint / cross-account questions |

---

*Updated June 3, 2026 — added Tier 5 section for June 3 mini-test misses (Q4 access key rotation, Q7 SecurityHeadersPolicy, Q17 Secrets Manager rotation scope, Q21 trust policy vs inline, Q25 Lambda networking). Also deepened: Secrets Manager rotation Lambda steps, VPC endpoint policy mechanics (runtime vs admin actions), D4 IAM section (PassRole vs AssumeRole, IAM Paths, condition key typing, AD federation), construction errors with WHY annotations.*

---

## ⚠️ Tier 6 — Mock #1 (June 4) misses — 28 questions

Mock #1 (full 65-question timed mock, ~3 hours) scored 55% (36/65). Below are detailed remediation notes for each miss. KMS-specific misses reference `kms-deep-dive.md` for full context.

### Miss inventory at a glance

| Q | Domain | Topic | Category |
|---|---|---|---|
| Q2 | D5 | S3 Access Points limitations | 🔴 Knowledge gap |
| Q5 | D5 | EBS snapshot cross-account protection | 🟢 Defensibly correct |
| Q6 | D2 | IR notification response (3-select) | 🟠 Trap |
| Q11 | D3 | WAF rules validation | 🔴 Knowledge gap |
| Q13 | D1 | CloudWatch Agent troubleshooting | 🔴 Knowledge gap |
| Q15 | D5 | KMS GenerateDataKey | 🔴 Knowledge gap (see kms-deep-dive.md) |
| Q16 | D3 | Account compromise response (3-select) | 🟠 Trap |
| Q17 | D3 | DDoS resilient architecture (3-select) | 🔴 Knowledge gap |
| Q21 | D4 | Admin permissions troubleshooting (2-select) | 🟡 Borderline (subtle wording) |
| Q26 | D3 | NACL ephemeral ports | 🔴 Knowledge gap (foundational!) |
| Q27 | D3 | DDoS revamp architecture (2-select) | 🟠 Trap |
| Q28 | D2 | Layer 7 DDoS response (2-select) | 🔴 Knowledge gap |
| Q31 | D5 | MFA BoolIfExists | 🔴 Knowledge gap (see kms-deep-dive.md) |
| Q33 | D5 | Multi-region KMS replication | 🔴 Knowledge gap (see kms-deep-dive.md) |
| Q35 | D3 | Compromised EC2 access control | 🔴 Reading miss |
| Q38 | D3 | GuardDuty trusted IP list precedence (2-select) | 🔴 Knowledge gap |
| Q40 | D2 | AWS abuse notice response | 🟠 Trap |
| Q41 | D1 | WAF analytics architecture | 🔴 Knowledge gap |
| Q46 | D6 | Service Catalog launch constraint | 🔴 Knowledge gap |
| Q47 | D4 | CloudFront origin restriction (2-select) | 🔴 Knowledge gap |
| Q51 | D5 | Firewall Manager web ACL behavior | 🟠 Trap |
| Q53 | D3 | GuardDuty DNS log analysis | 🔴 Knowledge gap |
| Q54 | D5 | EBS encryption for existing ASG | 🟠 Trap |
| Q55 | D4 | KMS CreateGrant for EBS (2-select) | 🔴 Knowledge gap (see kms-deep-dive.md) |
| Q58 | D5 | Cross-account KMS for ASG (2-select) | 🔴 Knowledge gap (see kms-deep-dive.md) |
| Q59 | D2 | EC2Launch v2 password reset (3-select) | 🔴 Knowledge gap |
| Q60 | D3 | Block IMDS access | 🟢 Defensibly correct |
| Q63 | D3 | GuardDuty trusted IP issues (2-select) | 🔴 Knowledge gap |
| Q64 | D1 | CloudTrail validation scope (2-select) | 🟡 Borderline (bad question) |

---

### Q2 — S3 Access Points limitations

**Question**: Which configuration characteristics apply when defining access points for S3 buckets? (Select 3)

**You picked**: "Aliases for S3 Access Points are interchangeable with S3 bucket names. Aliases can be used as a logging destination for AWS CloudTrail logs and S3 server access logs."

**Correct answer included**: "You can only use access points to perform operations on objects" + "Cross-account access points don't grant access until permissions from bucket owner" + "You can't configure Cross-Region Replication through an access point"

**The trap**: The first half of your pick is TRUE (aliases ARE interchangeable with bucket names). The second half is FALSE (aliases CANNOT be used as logging destinations).

**Key facts about S3 Access Points**:
- Each access point has its own IAM policy + network controls (VPC-restricted possible)
- Aliases are auto-generated; usable wherever bucket name works
- Aliases CANNOT be CloudTrail/S3 server access log destinations
- After creation, VPC configuration CANNOT be changed (must recreate)
- Cross-account access points need bucket owner permission too
- Cannot use access points as replication destination
- Access points support HTTPS only

**Discriminator**: when a multi-clause option says "X is true AND Y is true" — both must be true. Read each clause separately.

---

### Q5 — EBS snapshot cross-account protection

**Already covered above and in detail mid-mock.** Your answer was technically flawed but the question's "correct" answer is also flawed (cross-account doesn't prevent deletion of source CMK). For exam: pick cross-account + share CMK + customer-managed key. For real architecture: also re-encrypt with destination CMK + Backup Vault Lock.

**Cheatsheet patch**: See KMS deep-dive "Cross-account KMS patterns" section.

---

### Q6 — IR notification response (3-select)

**Question**: After receiving AWS suspicious activity notification, which 3 actions are needed BEFORE responding to AWS Support?

**You picked**: Got 2 right (CompromisedKeyQuarantineV2 check + Access key rotation). **Missed**: "If you must retain an EC2 instance for regulatory/legal reasons, create EBS snapshot before terminating."

**Your wrong pick**: "If exposed EC2 cannot be shut down, move it to Isolation VPC to contain exposure while keeping the instance working"

**Why the wrong pick fails**: **You cannot move a running EC2 instance between VPCs.** Isolation VPC pattern requires shutting down → relaunching in new VPC. The option's claim about "having the ability to keep the instance working" is a construction error.

**Why the correct pick is right**: Forensic preservation BEFORE termination — even if you must terminate for compliance, snapshot first to preserve evidence.

**Construction error to memorize**: "Move running EC2 between VPCs without restart" = impossible.

**Also note**: "An IAM principal can have up to five access keys" — actual limit is **TWO** access keys per IAM user. The "5 keys" number poisons that whole option.

---

### Q11 — WAF rules validation

**Question**: How can the engineer validate AWS WAF rules are working?

**You picked**: "AWS WAF reports metrics once a minute to CloudTrail. Use statistics in Amazon CloudTrail to gather insights about WAF responses."

**Correct**: "Enable WAF comprehensive logs that are delivered through Amazon Kinesis Firehose to a destination of your choice."

**The trap**: Your option said **CloudTrail** for WAF metrics. WAF reports metrics to **CloudWatch**, not CloudTrail. If the option had said "CloudWatch" it would have been correct (CloudWatch metrics from WAF run at 1-minute granularity).

**WAF logging architecture**:
- **CloudWatch Metrics** (1-minute granularity): for monitoring rule counts, allowed/blocked
- **CloudWatch Logs**: WAF can ship logs directly here
- **S3**: WAF can ship logs directly here
- **Kinesis Data Firehose**: WAF can ship to Firehose → then to S3, Splunk, ES, etc.

**For "full request inspection and rule debugging" → Kinesis Firehose logs** (full HTTP headers, which rules triggered, request body).
**For "metrics-level monitoring" → CloudWatch** (counts only).

**Construction error**: "WAF metrics to CloudTrail" — WAF doesn't write metrics to CloudTrail. CloudTrail captures WAF management API calls (CreateWebACL, etc.), not metrics.

---

### Q13 — CloudWatch Agent troubleshooting (2-select)

**Question**: Why isn't the CloudWatch agent pushing log events?

**You picked**: IAM permissions (correct) + "VPC endpoints" option (wrong)

**Correct second pick**: "Creating an AMI AFTER the CloudWatch agent is installed can lead to errors in the CloudWatch agent."

**Why your "VPC endpoints" pick was wrong**: CloudWatch Logs endpoint can be either public (via IGW/NAT) OR via VPC endpoint. Both work — no requirement to use VPC endpoint specifically.

**Why the AMI timing matters**: When you create an AMI from an instance WITH the agent already running, the AMI captures instance-specific metadata (instance ID, etc.). Instances launched from that AMI have stale metadata → CW Agent breaks.

**Best practice**: Install CW Agent AT LAUNCH (via UserData, CloudFormation, SSM Document) rather than baking into AMI.

**Also valid reasons for missing logs**:
- IAM role missing `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`, `logs:DescribeLogStreams`
- Agent installed BEFORE AMI was created (stale metadata)
- run_as_user parameter set to non-root WITHOUT log directory perms (works as non-root, but needs perms)

**Construction errors**:
- ❌ "VPC endpoints required for CW Logs" — public endpoint also works
- ❌ "run_as_user must be root" — non-root works if permissions are correct
- ❌ "Install agent only after creating AMI" — backward, install AT launch

---

### Q15 — KMS GenerateDataKey

**See `kms-deep-dive.md` section "Envelope encryption" for full coverage.**

TL;DR: Client-side encryption uses `kms:GenerateDataKey` (envelope pattern), NOT `kms:Encrypt` (4KB limit). The error "forbidden" on PUT means the IAM policy is missing `kms:GenerateDataKey`.

---

### Q16 — Account compromise response (3-select)

**Question**: How to handle a compromised AWS account?

**You picked**: Trusted Advisor security check + Inspector + AWS Git projects (got 1 right)

**Correct 3**: Rotate all root/IAM access keys + Check AWS account bill + Use AWS Git projects

**Why your other 2 picks were wrong**:
- **Trusted Advisor security check**: provides general best-practice recommendations. NOT useful for confirming what specific resources were compromised.
- **Amazon Inspector**: scans for vulnerabilities (known CVEs in software). NOT useful for detecting active compromise.

**The right response actions**:
1. **Rotate keys**: stops further unauthorized API calls
2. **Check the bill**: unusual resource spikes (e.g., crypto mining EC2 fleet in another region) reveal what was created
3. **AWS Git projects**: scan source repos for leaked credentials (Git Secrets tool)

**Pattern**: For account compromise response, think CONTAINMENT (rotate creds) → DETECTION (bill, CloudTrail) → PREVENTION (Git Secrets to stop recurrence).

**Construction trap**: "Use Inspector/Macie/Trusted Advisor to detect compromised resources" — these are general-purpose tools NOT designed for active compromise investigation. Use CloudTrail + bill review + GuardDuty findings for that.

---

### Q17 — DDoS resilient architecture (3-select)

**Question**: 3 best practices for DDoS-resilient architecture.

**You picked**: API Gateway edge-optimized endpoint (wrong) + others

**Correct 3**:
1. Use CloudFront with Forward all headers to API Gateway REGIONAL endpoint
2. Register Elastic IPs as Protected Resources in Shield Advanced
3. **Configure SGs to NOT use connection tracking** (allow all 0.0.0.0/0 0-65535)

**Why your API Gateway edge-optimized pick was wrong**: 
- Edge-optimized API GW uses an AWS-MANAGED CloudFront distribution that you can't control
- For DDoS mitigation, you want YOUR OWN CloudFront distribution with WAF
- So: API GW REGIONAL endpoint + YOUR CloudFront in front = better control

**SG connection tracking trick**:
- SGs are stateful — they track connections by default
- Under DDoS, the connection table can fill up → new connections rejected
- Workaround: rules covering all traffic (0.0.0.0/0, 0-65535) make connections UNTRACKED
- Untracked connections = no table fill = better DDoS resilience

**Construction trap**: 
- "NLB routes based on content" — false (NLB is L4, doesn't see HTTP content)
- "WAF on S3 buckets" — false (WAF supports CloudFront/ALB/API GW/AppSync only)
- "API Gateway edge-optimized for DDoS protection" — gives less control vs regional + own CloudFront

---

### Q21 — Admin permissions troubleshooting (2-select)

**Question**: IAM entity has admin permissions but receives AccessDenied. Why? (Select 2)

**You picked**: VPC endpoint policy (correct) + permissions boundary description (wrong)

**Correct second pick**: "A session policy is in place and is causing an authorization issue"

**Why your second pick was wrong**: The option said "permissions boundary defines maximum permissions an identity-based policy can grant to an entity" — TRUE so far. Then: "Check for any restrictive permissions boundary referenced by the **concerned resource-based policy**." The mention of "resource-based policy" is the trap — permissions boundary is on the principal, NOT referenced by resource policies.

**Permissions boundary basics**:
- Attached to IAM user or role (NOT resource)
- Sets MAXIMUM permissions the entity can have
- Effective permissions = identity policy ∩ permissions boundary
- The boundary doesn't reference resource-based policies; it's its own layer

**Session policy**:
- Passed when creating a temporary session (`AssumeRole`, `GetFederationToken`)
- Further narrows the role's permissions for that session only
- Check CloudTrail logs for `AssumeRole` API calls to see if session policy was passed

**Layers to check when admin gets AccessDenied**:
1. Explicit Deny in any policy
2. SCP in Organizations
3. VPC endpoint policy (if routed through endpoint)
4. Resource-based policy
5. IAM identity policy
 

---

### Q26 — NACL ephemeral ports (FOUNDATIONAL)

**Question**: EC2 has SG allowing inbound HTTP from 0.0.0.0/0 + NACL allowing inbound HTTP from 0.0.0.0/0 (both with default outbound). What's missing for the EC2 to respond?

**You picked**: "The configuration is complete" — WRONG.

**Correct**: "An outbound rule must be added to the NACL to allow the response to be sent to the client on the ephemeral port range."

**Why this is fundamental**: 
- **Security Groups are STATEFUL** — return traffic for an allowed inbound connection is automatically allowed back out
- **NACLs are STATELESS** — must explicitly allow both directions

When client (browser) connects to your EC2:
- Source port (browser's): random ephemeral (1024-65535 typically)
- Destination port: 80
- For your server to respond, packet flows:
  - OUTBOUND from EC2 with source port 80, destination port = client's ephemeral (1024-65535)

**Your NACL must allow OUTBOUND on ports 1024-65535** for return traffic.

**Ephemeral port ranges by OS**:
- Linux: 32768-60999
- Windows Server 2003: 1025-5000
- Windows Server 2008+: 49152-65535
- Mac OS: 49152-65535

**For exam: allow outbound 1024-65535** to cover all OS variants.

**The reflex**: when you see NACL + connection problem → check BOTH directions. SGs handle stateless implicitly; NACLs don't.

**Construction trap**: "SG outbound rule for return traffic" — SGs are stateful, no outbound rule needed for return traffic.

---

### Q27 — DDoS revamp architecture (2-select)

**Question**: After DDoS attack, how to revamp security for an EC2-hosted website with RDS backend and static images?

**You picked**: "Configure ASG + WAF" (wrong) + correct option

**Correct 2**:
1. Use Amazon Route 53 (provides DDoS protection via Shield Standard + shuffle sharding + anycast)
2. Move static content to S3 + CloudFront distribution + WAF on the distribution

**Why "ASG + WAF" is wrong**:
- **WAF cannot attach directly to ASG** or EC2 instances
- WAF supports: CloudFront, ALB, API Gateway, AppSync, Cognito User Pools
- For EC2-hosted sites: must front with CloudFront or ALB to use WAF

**Why Route 53 helps with DDoS**:
- Provides DNS resilience under query floods
- Shuffle sharding + anycast spread traffic across edge locations
- Shield Standard auto-protects Route 53 from L3/L4 attacks
- Health-checks + failover for resilient routing

**Static content offload benefits**:
- S3 handles massive request rates natively
- CloudFront caches at edge → reduces origin load
- WAF on CloudFront blocks attacks at edge before hitting origin

**Construction trap**: "WAF in front of ASG" — WAF doesn't attach to ASG/EC2. Always need CloudFront/ALB/API GW intermediary.

---

### Q28 — Layer 7 DDoS response (2-select)

**Question**: Application-layer DDoS is underway. Immediate response actions?

**You picked**: GuardDuty + correct CloudWatch metric (got 1 right)

**Correct 2**:
1. Create your own AWS WAF rules in your web ACL to mitigate the attack
2. Contact AWS Support Center if you're a Shield Advanced customer

**Why GuardDuty alone isn't enough**: GuardDuty DETECTS, doesn't BLOCK. For active L7 DDoS mitigation, you need WAF rules (blocking) and/or Shield Advanced support team (expert mitigation).

**Shield Standard vs Advanced for L7**:
- **Shield Standard**: auto-mitigates L3/L4 (network layer) attacks
- **Shield Advanced**: same + L7 visibility + DRT support + cost protection
- Shield Advanced does NOT auto-mitigate L7 to avoid blocking valid traffic
- For L7: YOU write WAF rules OR Shield Advanced team helps you write them

**Immediate response actions for L7 DDoS**:
1. Identify attack pattern (which URLs/headers/user-agents)
2. Write WAF rate-based rule or IP set rule to block source
3. Contact AWS DDoS Response Team (DRT) if Shield Advanced
4. NOT: GuardDuty (passive detection only)
5. NOT: CloudWatch alarms (notification, not mitigation)
6. NOT: SSM document (no native DDoS-blocking SSM template)

---

### Q31 — MFA BoolIfExists

**See `kms-deep-dive.md` section "Pattern 4: Mandate MFA" for full coverage.**

TL;DR: Use `Deny + BoolIfExists + aws:MultiFactorAuthPresent: false`. Catches all three caller scenarios including long-term access keys.

---

### Q33 — Multi-region KMS replication

**See `kms-deep-dive.md` section "Multi-Region keys" for full coverage.**

TL;DR: You CANNOT convert single-Region KMS key to multi-Region. Must create new multi-Region key from scratch, then re-encrypt data into a new bucket using the new key.

---

### Q35 — Compromised EC2 access control (READING MISS)

**Question**: EC2 with IAM role accessing S3 might be compromised. Cannot terminate (critical app). How to block further S3 access fastest?

**You picked**: "Update S3 bucket policy to deny IAM role + Create EBS snapshot + terminate the instance" — **violates the constraint**

**Correct**: "Revoke all active sessions for the IAM role + Update S3 bucket policy + Remove IAM role from EC2 instance profile"

**Why your pick failed**: The question explicitly said **"the instance cannot be immediately terminated."** You selected an option that includes terminating. That's a reading miss, not a knowledge gap.

**The correct sequence**:
1. **Revoke active sessions for the IAM role** (use `aws iam put-role-policy` with `aws:TokenIssueTime` deny condition) — invalidates any current STS sessions
2. **Update S3 bucket policy to deny the IAM role** — explicit deny overrides everything
3. **Remove IAM role from EC2 instance profile** — stops new sessions

**Why "revoke sessions" matters first**: even if you remove the role from the instance profile, any STS session token already issued is still valid until expiration. Revoke first to invalidate active sessions.

**For exam-day reading discipline**: 
- Circle "**cannot terminate**" or similar constraints BEFORE reading options
- Eliminate any option that violates the constraint
- This is the #1 preventable miss category

---

### Q38 — GuardDuty trusted IP list precedence (2-select)

**Question**: If same IP is in both trusted IP list AND threat list, what happens? Also which managed policy for full GuardDuty management?

**You picked**: "Threat list processed first → generates finding" (wrong) + correct managed policy

**Correct**: "Trusted IP list processed FIRST → NO finding generated"

**Why**: Trusted IP list has PRIORITY. If an IP is in both:
- GuardDuty checks trusted IP list first → match found → skip generating finding
- The IP is effectively whitelisted regardless of threat list presence

**Practical implication**: don't accidentally whitelist an IP that's also a known threat — your trusted list always wins.

**GuardDuty list limits per account per region**:
- 1 trusted IP list
- 6 threat lists
- IPs must be publicly routable (no private RFC1918)
- Same region as your GuardDuty deployment

**Permissions for full GuardDuty list management**:
- `AmazonGuardDutyFullAccess` managed policy
- PLUS `iam:PutRolePolicy` and `iam:DeleteRolePolicy` on the service-linked role (for upload/rename/delete operations)

**Construction trap**: "Attach AWSServiceRoleForAmazonGuardDuty policy to IAM entities" — this is the service-linked role itself, you can't attach it to other entities.

---

### Q40 — AWS abuse notice response

**Question**: Received abuse notice from AWS. What's the right response?

**You picked**: "AWS Trust & Safety Team provides technical support, contact them" — WRONG

**Correct**: "Review the abuse notice and reply explaining how you will prevent the abusive activity from recurring."

**Why your pick failed**: **AWS Trust & Safety is a POLICY/LEGAL team, not technical support.** They:
- Investigate abuse complaints
- Send abuse notices to account owners
- Track responses for compliance
- Do NOT provide technical troubleshooting

**For technical issues**: AWS Support (Developer, Business, Enterprise tiers) provides one-on-one technical help. This is a separate channel.

**Response to abuse notice**:
1. Review the report (logs are usually included)
2. Investigate what was happening (CloudTrail, GuardDuty findings)
3. **Respond within 24 hours** with explanation + remediation steps
4. If no response in 24h → AWS may block resources or suspend account

**Common abuse triggers**:
- Compromised EC2 sending spam
- Port scanning from your IPs
- Crypto mining on stolen credentials
- DDoS sources

---

### Q41 — WAF analytics architecture

**Question**: Build a serverless WAF analytics dashboard with real-time data.

**You picked**: "WAF logs to CloudWatch Logs + CloudWatch Logs Insights + QuickSight" — wrong

**Correct**: "WAF → Kinesis Data Firehose → S3 → AWS Glue crawler → Athena table → QuickSight dashboards"

**Why your pick failed**:
- CloudWatch and QuickSight cannot directly connect
- Even if they could, the question requires "build multiple dashboards" — better suited to BI tool with structured data (Athena tables)

**The canonical WAF analytics pipeline**:
```
WAF → Firehose (real-time stream)
     → S3 (storage)
     → Glue Crawler (schema discovery)
     → Athena (SQL queries)
     → QuickSight (dashboards/visualizations)
```

**Why each component**:
- **Firehose**: real-time WAF log delivery, no servers
- **S3**: cheap long-term storage, queryable
- **Glue Crawler**: auto-discovers schema from JSON logs
- **Athena**: serverless SQL on S3 logs
- **QuickSight**: BI dashboards with Athena as data source

**Construction trap**: "Redshift Spectrum for serverless" — Redshift needs a cluster. NOT serverless.

**Memorize**: "WAF analytics serverless" → Firehose+S3+Glue+Athena+QuickSight chain.

---

### Q46 — Service Catalog launch constraint

**Question**: Org wants to add products to Service Catalog. End users should NOT need their own IAM credentials to launch products. Easiest way?

**You picked**: "Use Service Actions" — WRONG

**Correct**: "Add launch constraint(s) to each product in the Service Catalog portfolio"

**Service Catalog constraints**:
| Constraint type | Purpose |
|---|---|
| **Launch constraint** | Specifies the IAM role Service Catalog assumes when END USER launches/updates/terminates the product. Removes need for end-user IAM permissions. |
| **Notification constraint** | SNS topic for stack events |
| **Tag update constraint** | Allow/disallow end users from updating tags |
| **Template constraint** | Narrow CloudFormation parameter values |
| **Stack set constraint** | Multi-account deployment configuration |

**Service Actions** (your wrong pick): operational tasks end users can perform AFTER provisioning (restart, snapshot, etc.). NOT for delegating launch permissions.

**Why launch constraints exist**:
- Without launch constraint: end users need their own permissions for CloudFormation + every AWS service the product uses
- With launch constraint: Service Catalog assumes the launch role (which has the broad permissions); end users only need Service Catalog read perms

**Constraint scope**: launch constraints attach at the PRODUCT-PORTFOLIO ASSOCIATION level (one product per portfolio). Cannot attach at portfolio-wide level.

---

### Q47 — CloudFront origin restriction (2-select)

**Question**: Migrating static content from EC2 to S3 + CloudFront. Only specific IP ranges should access content. Pick 2.

**You picked**: WAF ACL on CloudFront with IP match (correct) + missed second answer

**Correct second pick**: "Configure an OAI (Origin Access Identity) or OAC (Origin Access Control) and associate it with the CloudFront distribution. Set S3 bucket policy to only allow the OAI/OAC."

**Why both are needed**:
1. **WAF IP restriction**: blocks unwanted IPs from reaching CloudFront (frontend)
2. **OAI/OAC restriction**: ensures S3 bucket only accepts requests THROUGH CloudFront (backend)

Without OAI/OAC, users could bypass CloudFront and access S3 directly (with the bucket URL), defeating the IP restriction.

**OAI vs OAC**:
- **OAI** (Origin Access Identity): older, IAM-based
- **OAC** (Origin Access Control): newer (2022), preferred — supports SSE-KMS, all regions, POST/PUT, signature V4
- AWS recommends OAC for new setups; OAI still works

**The pattern**:
1. Create OAC, attach to CloudFront distribution
2. S3 bucket policy: only allow `Service: cloudfront.amazonaws.com` with `aws:SourceArn = distribution ARN`
3. Block all other S3 public access

**Construction trap**: "Create new SG for CloudFront" — CloudFront doesn't sit in a VPC, no SG applies. Use WAF + OAC instead.

---

### Q51 — Firewall Manager web ACL behavior

**Question**: Engineer created web ACL via Firewall Manager policy, but it's not associated with in-scope resources. Why?

**You picked**: First option about replacing web ACLs (wrong)

**Correct**: "If `auto remediate any non-compliant resources` isn't turned on, the Firewall Manager-created web ACL won't be associated with in-scope resources."

**Firewall Manager auto-remediation behavior**:
| Auto-remediate | Replace existing ACLs | Result |
|---|---|---|
| OFF | (irrelevant) | WAF policy creates web ACL but doesn't attach to anything |
| ON | OFF | Attaches to resources without existing ACL; skips resources WITH existing ACL |
| ON | ON | Replaces existing ACLs with Firewall Manager-created ACL |

**Why your pick was wrong**: The option you chose was complex (replacing existing Web ACLs), but the simple reason for "web ACL not associated" is the toggle being off.

**Pattern recognition**: when a question asks "why isn't X working," check the most basic enablement toggle first before going to complex scenarios.

---

### Q53 — GuardDuty DNS log analysis

**Question**: Company uses Active Directory (on-premises servers) for DNS. GuardDuty isn't reporting on DNS logs. Why?

**You picked**: "GuardDuty analyzes DNS logs from Route 53 Resolver query logging feature" — wrong

**Correct**: "If you use a custom DNS resolver (not AWS DNS), GuardDuty cannot access/process DNS data."

**Why**: GuardDuty's DNS analysis uses the AWS internal DNS resolver. If your instances use:
- Default AWS DNS resolver → GuardDuty sees DNS queries
- Custom DNS (on-prem AD, OpenDNS, GoogleDNS, etc.) → GuardDuty BLIND to DNS

**Practical impact**: 
- If you must use custom DNS, lose GuardDuty DNS-based findings
- Other GuardDuty findings (VPC Flow Logs, CloudTrail, Kubernetes audit) still work
- Workaround: enable Route 53 Resolver query logging separately to S3 for DNS analysis

**Why your "Route 53 Resolver query logging" pick was wrong**: GuardDuty's DNS analysis is INDEPENDENT of Route 53 Resolver query logging. They're two separate data streams. GuardDuty uses internal AWS DNS resolver data, not the query logging feature.

---

### Q54 — EBS encryption for existing ASG

**Question**: Existing ASG with unencrypted EBS. Ensure ALL EBS volumes encrypted (current AND future).

**You picked**: "Modify the launch template + Auto Scaling instance refresh" — wrong (modify isn't possible)

**Correct**: "Enable default EBS encryption in EC2 console + use Auto Scaling instance refresh to replace existing instances."

**Why your pick failed**: 
- **Cannot modify launch templates** — you create new VERSIONS
- The "modify" wording is a construction error

**The simpler correct approach**:
1. **Enable default EBS encryption** at account/region level (one click in EC2 console)
2. **All new EBS volumes** in that region are encrypted by default
3. Use ASG **instance refresh** to roll all existing instances → new ones launched WITH encrypted volumes
4. Old unencrypted volumes terminated with old instances

**Why this beats launch template version changes**:
- No template editing required (default encryption applies regardless of template)
- Cannot accidentally launch unencrypted volumes from old template version
- Operationally simpler

**Construction trap**: "Modify launch template" — templates are versioned and immutable; you create new versions.

**Also note**: default EBS encryption setting doesn't affect EXISTING volumes — only new ones. To encrypt existing, snapshot+copy with encryption+create new volume from copy.

---

### Q55 — KMS CreateGrant for EBS

**See `kms-deep-dive.md` section "EC2 + EBS encrypted (Q55 pattern)" for full coverage.**

TL;DR: User starting EC2 with encrypted EBS needs `kms:CreateGrant` action + `kms:GrantIsForAWSResource: true` condition. EC2 service uses the grant to decrypt the EBS volume.

---

### Q58 — Cross-account KMS for ASG

**See `kms-deep-dive.md` section "ASG + encrypted EBS (Q58 pattern)" for full coverage.**

TL;DR: Two-side setup — Account A (key owner) key policy allows Account B; Account B creates KMS grant for ASG service-linked role.

---

### Q59 — EC2Launch v2 password reset (3-select)

**Question**: Reset lost Windows administrator password using EC2Launch v2. Steps?

**You picked**: "Select Offline Instance Option → Diagnose and Rescue → Reset Administrator Password" — wrong (that's EC2Launch v1, not v2)

**Correct 3** (EC2Launch v2 sequence):
1. Verify EC2Launch v2 service is running. Detach EBS root volume from the instance.
2. Launch a temporary instance + attach the volume as SECONDARY. Delete the `.run-once` file at `%ProgramData%/Amazon/EC2Launch/state/.run-once`.
3. Reattach the volume to ORIGINAL instance as ROOT. Connect using key pair to retrieve new admin password.

**Why your pick was wrong**: Your selected option was for the OLDER EC2Launch v1 (via EC2Rescue tool with offline rescue feature). EC2Launch v2 uses a different approach — it generates a new password on launch if `.run-once` file is absent.

**EC2Launch v1 vs v2**:
| Aspect | EC2Launch v1 | EC2Launch v2 |
|---|---|---|
| Password reset | EC2Rescue tool with offline rescue | Delete `.run-once` file |
| Configuration | XML config files | YAML config |
| AMI selection for temp instance | Different version OK | DIFFERENT version REQUIRED (avoid disk signature collision) |

**Critical detail**: temporary instance must use a DIFFERENT Windows version AMI (e.g., if original is 2019, use 2016 temp instance) to avoid disk signature collisions.

---

### Q60 — Block IMDS access

**Question**: Block EC2 instance metadata service for shared instances.

**You picked**: "Configure IMDSv2 (session-oriented method)" — defensibly correct in practice, but exam wants:

**Correct**: "Implement local firewall rules using iptables-based restrictions"

**Why both answers are defensible**:
- **IMDSv2**: AWS-recommended hardening (mitigates SSRF attacks via session tokens). Doesn't fully block IMDS access, but makes exploitation much harder.
- **iptables**: hard block at the OS level. Truly prevents specific processes from accessing 169.254.169.254.

**The question's framing**: "block the EC2 instance metadata service" suggests TOTAL blocking, which only iptables achieves. IMDSv2 hardens but doesn't fully block.

**iptables pattern**:
```bash
sudo iptables --append OUTPUT --proto tcp --destination 169.254.169.254 \
  --match owner --uid-owner apache --jump REJECT
```

This rejects metadata requests from the apache user only.

**For exam**: 
- "Block IMDS for specific users/processes" → iptables
- "Prevent SSRF exploitation of IMDS" → IMDSv2
- "Completely disable IMDS" → `aws ec2 modify-instance-metadata-options --http-endpoint disabled`

---

### Q63 — GuardDuty trusted IP issues (2-select)

**Question**: Trusted IP list is configured but findings still generated. Why? (Select 2)

**You picked**: "Multi-account environments generate findings based on admin's trusted IPs" — wrong + correct option

**Correct 2**:
1. Ensure IP addresses are publicly routable IPv4 (no RFC1918 private addresses)
2. Ensure trusted IP lists are in the SAME REGION as the GuardDuty findings

**Trusted IP list constraints**:
- Per-account, per-region (not global)
- Only ONE trusted IP list per account per region (vs 6 threat lists)
- Must be publicly routable IPv4 (no private addresses)
- Affects VPC Flow Logs and CloudTrail findings (NOT DNS findings)
- Trusted IP takes PRIORITY over threat IP (Q38)

**Why your pick was wrong**: In multi-account setups, GuardDuty admin's THREAT lists propagate to members (so members benefit from admin's threat intelligence). TRUSTED lists do NOT propagate the same way — they're per-account.

**Common configuration mistakes**:
1. Adding private IPs (10.x, 172.16-31.x, 192.168.x) — GuardDuty ignores these in trusted IP lists
2. Uploading list to wrong region
3. Trying to have multiple trusted IP lists per account (only 1 allowed)
4. Expecting trusted IPs to suppress DNS-based findings (they don't — DNS findings ignore IP lists)

---

### Q64 — CloudTrail validation scope (BAD QUESTION)

**Already discussed above.** This was a borderline/bad question. The "correct" answer rewards narrow interpretation. Your answer was defensible.

**Lesson**: when two options differ by a single adjective ("all" vs "business units"), the narrower one usually wins on exam graders' interpretation, even if the wording's not perfectly clear.

---

### Pattern summary across Mock #1 misses

**Top clusters identified**:

1. **KMS depth (6 misses: Q5, Q15, Q31, Q33, Q55, Q58)** — see `kms-deep-dive.md`
2. **WAF/DDoS/CloudFront architecture (7 misses: Q11, Q17, Q27, Q28, Q41, Q47, Q51)** — needs dedicated study
3. **GuardDuty configuration (3 misses: Q38, Q53, Q63)** — list constraints, DNS data sources
4. **Reading constraint misses (1 critical: Q35)** — circle "cannot," "must not," "least" before answering
5. **Niche services (Q2, Q13, Q46, Q59)** — Access Points, CW Agent installation, Service Catalog, EC2Launch v2

**Behavior patterns to fix**:
- ❌ Over-engineered options (Q54: modify launch template vs enable default encryption)
- ❌ Plausible-sounding services with wrong purpose (Q11: CloudTrail metrics, Q40: AWS Trust & Safety tech support)
- ❌ Construction errors I should have caught (Q6: move EC2 between VPCs, Q27: WAF on ASG)

---

*Updated June 4, 2026 — added Tier 6 section with full remediation notes for Mock #1 (28 misses). KMS-related misses cross-reference `kms-deep-dive.md`. WAF/DDoS deep dive coming June 5.*

---

## ⚠️ Tier 7 — June 5 TD D3 practice misses + ACM cert mastery

Tutorials Dojo D3 practice set (June 5 evening, 15 questions). Score: 10/15 = 67% (up from Mock #1 D3 41%).

The 5 misses surface recurring patterns that go beyond the specific questions — these mental models apply across multiple exam topics.

### Pattern A — Detection ≠ Auto-Remediation (TD Q14)

**The miss**: Picked "AWS Config `access-keys-rotated` managed rule" for auto-disabling IAM access keys >90 days old.

**Why wrong**: AWS Config is a DETECTION/COMPLIANCE service. It marks resources non-compliant but does NOT auto-fix.

**Right answer**: Custom Lambda function:
1. `GenerateCredentialReport` API (trigger report)
2. `GetCredentialReport` API (download CSV)
3. Parse `access_key_X_last_rotated` column for keys >90 days
4. `UpdateAccessKey` with `Status=Inactive`
5. Schedule via EventBridge cron rule

**The rule reflex** (memorize):

| Question phrasing | Right mechanism |
|---|---|
| "Detect non-compliant resources" | AWS Config (rule generates findings) |
| "Audit compliance posture" | Config + Security Hub aggregation |
| "**Automatically fix** / auto-remediate / auto-disable" | Config + **SSM Automation document** (if exists) OR **custom Lambda** |
| "Automate IAM access key disable" | Custom Lambda (no built-in SSM document for this) |

**Config + SSM auto-remediation exists for some rules**, e.g.:
- `AWS-DisableS3BucketPublicReadWrite`
- `AWS-EnableCloudTrail`
- `AWS-RemoveS3BucketPolicyStatement`

But NOT for IAM access key disable. Always custom Lambda for that scenario.

**Construction errors**:
- ❌ "CloudWatch Alarm on access key age" — not a CW metric
- ❌ "IAM dashboard manual check" — not automated
- ❌ "Config auto-disables via built-in Lambda" — Config detects only

---

### Pattern B — ACM cert regions (the complete master reference)

This pattern has been bitten multiple times. Master the FULL table once.

#### The two foundational rules

1. **CloudFront viewer cert → us-east-1 ALWAYS** (regardless of where anything else lives)
2. **Origin cert → origin's region** (ALB cert in ALB's region, API GW cert in API GW's region)

#### Complete cert region table by service

| Service / scenario | Cert region |
|---|---|
| **CloudFront viewer (client ↔ CF)** | **us-east-1 ALWAYS** |
| **CloudFront → ALB origin** | ALB's region |
| **CloudFront → API GW regional origin** | API GW's region |
| **API Gateway edge-optimized** | **us-east-1** (uses AWS-managed CF underneath) |
| **API Gateway regional** | API GW's region |
| **API Gateway private** | N/A (VPC, internal PKI) |
| **ALB / NLB / GWLB** | Load balancer's region |
| **App Runner** | Service's region |
| **AppSync** | Service's region |
| **Elastic Beanstalk** | Region |
| **Verified Access** | Region |
| **Cognito User Pool custom domain** | us-east-1 (uses CF) |
| **WorkSpaces** | Region |
| **Elastic Transcoder** | N/A |

#### The recall trick

**"Edge gets East"** — anything edge-optimized or CloudFront viewer-facing → us-east-1
**"Regional gets local"** — anything regional gets a cert in its own region

#### Compound architecture: Regional API GW behind your own CloudFront (Q17 / Q27 pattern)

This is the DDoS-resilient pattern. Requires TWO certs:

```
Client ──HTTPS──→ Your CloudFront ──HTTPS──→ Regional API GW
            cert #1: us-east-1         cert #2: API GW's region
```

Both certs can be for the SAME custom domain. Just requested in different regions.

#### Why use Regional API GW + own CloudFront vs Edge-optimized

| Option | When to use |
|---|---|
| **Edge-optimized API GW** | Simple, AWS handles edge CF, but **you can't attach YOUR WAF**, can't customize cache, can't add Shield Advanced features |
| **Regional API GW + your own CloudFront** | Full WAF control, Shield Advanced integration, cache customization, DDoS resilience |

For DDoS protection or WAF: **always Regional + own CloudFront**.

#### Custom domain vs default endpoint

- **Default endpoint** (`abc123.execute-api.us-west-2.amazonaws.com`) — uses AWS-managed cert, no ACM needed
- **Custom domain** (`api.mycompany.com`) — requires ACM cert in the right region

#### mTLS at API Gateway

If question mentions mutual TLS / client cert auth:
- **Only Regional API GW supports mTLS** (edge-optimized doesn't)
- Requires S3 truststore with PEM-encoded CA certs
- Plus ACM cert in API GW's region for server side

#### Construction errors to recognize

- ❌ "CloudFront cert in us-west-1" → must be us-east-1
- ❌ "Edge-optimized API GW custom domain cert in API GW's region" → must be us-east-1
- ❌ "Regional API GW custom domain cert in us-east-1" → must be API GW's region (unless you want it specifically)
- ❌ "Edge-optimized API GW with mTLS" → not supported
- ❌ "Edge-optimized API GW with attached WAF for DDoS" → can't attach WAF to AWS-managed CF

---

### Pattern C — Packet inspection vs metadata logging (TD Q9)

**The miss**: Picked VPC Flow Logs for "inspect IP packet data."

**Why wrong**: Flow Logs capture METADATA ONLY (src/dst/port/protocol/bytes). No packet contents.

**The full inspection capability table**:

| Service / mechanism | What it captures |
|---|---|
| **VPC Flow Logs** | METADATA ONLY (src IP, dst IP, ports, protocol, byte count, accept/reject) |
| **VPC Traffic Mirroring** | **FULL PACKET CONTENTS** (headers + payload, the actual bytes) |
| **Host-based IDS/IPS agent on EC2** | Full content AT the instance OS level |
| **Proxy software** intercepting outbound traffic | Application-layer content (including decrypted TLS at proxy) |
| **ALB access logs** | HTTP request metadata (URL, status, latency) — NOT packet contents |
| **CloudWatch Logs agent** | System/app log files — NOT network |
| **CloudTrail** | API calls — NOT network |
| **Network Firewall (with TLS inspection)** | Inline L3/L4/L7 inspection with optional TLS decrypt |

**Discriminator reflexes**:

| Question phrase | Right tool |
|---|---|
| "Inspect packet data" / "deep packet inspection" / "content" | Traffic Mirroring / host agent / proxy / Network Firewall |
| "Network flow visibility" / "connection metadata" | VPC Flow Logs |
| "Detect data exfiltration" | Traffic Mirroring → IDS appliance OR host agent |
| "Forensic packet capture" | Traffic Mirroring → S3 |
| "Detect SSL/TLS-encrypted attacks" | Host agent on instance OR Network Firewall TLS inspection |
| "Audit API actions" | CloudTrail |

**Construction errors**:
- ❌ "Flow Logs for packet data inspection" — metadata only
- ❌ "CloudWatch Logs agent for network packet inspection" — system logs, not network
- ❌ "ALB access logs for malware detection" — HTTP metadata only
- ❌ "CloudTrail for network traffic analysis" — API calls only

---

### Pattern D — CloudWatch Logs agent troubleshooting (TD Q6 — recurring pattern)

This appeared in Mock #1 (Q13) too. Lock the diagnostic checklist.

**Diagnostic checklist** (verify in order, most common first):

1. **IAM role on instance profile has**:
   - `logs:CreateLogGroup`
   - `logs:CreateLogStream`
   - `logs:PutLogEvents`
   - `logs:DescribeLogStreams`
2. **awslogs / CloudWatch agent service is RUNNING**
   - Verify via SSM Run Command: `systemctl status amazon-cloudwatch-agent`
   - Scales to many instances without SSH
3. **Network connectivity to CloudWatch Logs endpoint**
   - Public: NAT + internet route
   - Private: Interface VPC endpoint for `logs`
4. **Log file paths in agent config exist and are readable**
   - Check `/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json`
   - run_as_user must have file read permissions
5. **Region in agent config matches target log group region**
6. **AMI ordering** — AMI created BEFORE agent install
   - If agent installed THEN AMI created, instance metadata becomes stale on new instances
   - Best practice: install agent at LAUNCH (UserData / SSM / CFN), not bake into AMI

**Construction errors / distractors**:
- ❌ X-Ray for tracing CW Logs agent (X-Ray is for app request flow)
- ❌ Detailed Monitoring (just metric frequency 5min→1min, not logs)
- ❌ `/var/cloudwatch/rejects.log` (fictional file, doesn't exist)
- ❌ "Manually create log group first" (agent auto-creates with permissions)
- ❌ "Run as root only" (run_as_user can be any user with file perms)

**The SSM Run Command angle**: when question says "many EC2 instances need diagnosis" → SSM Run Command is the scalable answer (vs SSHing into each).

---

### Pattern E — Edge security architecture (TD Q15)

**The miss**: Picked NAT Gateway / geo-restriction / Direct Connect for hardening a public-facing multi-tier app.

**Why wrong**:
- NAT Gateway = OUTBOUND only, doesn't help inbound security
- Geo-restriction = filters geography, not attack types
- Direct Connect = private connectivity for known partners, not internet-facing security

**The right architecture: minimum-exposure pattern**

```
Internet
   ↓
Route 53 (DNS, Shield auto)
   ↓
CloudFront distribution (WAF web ACL)
   ↓
ALB (public-facing — the ONLY public entry point)
   ↓ (private subnets behind here)
Web servers (PRIVATE subnet, no public IPs)
   ↓
App servers (PRIVATE subnet)
   ↓
RDS (PRIVATE subnet)
```

**The two changes** to harden "public-facing multi-tier":
1. **Move backends to PRIVATE subnets, remove public IPs/EIPs** — backends no longer directly internet-reachable
2. **CloudFront + WAF in front of ALB** — edge protection at global scale + regional WAF on ALB for defense in depth

**Subnet placement rules**:

| Component | Subnet type |
|---|---|
| ALB (public) | Public subnet |
| NAT Gateway | Public subnet |
| Internet Gateway | Attached to VPC (no subnet) |
| Web servers | PRIVATE subnet |
| App servers | PRIVATE subnet |
| Databases (RDS, ElastiCache, etc.) | PRIVATE subnet |
| Lambda functions | VPC-attached to PRIVATE subnet (if VPC needed) |
| Bastion / Jump host | Public subnet (with very narrow SG) |

**Construction errors**:
- ❌ NAT Gateway for inbound security (outbound only)
- ❌ Geo-restriction as primary defense (wrong tool for general attacks)
- ❌ Direct Connect for public app security (private connectivity service)
- ❌ "Web servers need public IPs to receive traffic from internet" — false, ALB forwards
- ❌ Security group "deny" rules (SGs are allow-only; use NACL for deny if needed)

---

### Cross-cutting reflex: the "what does this service actually DO?" check

For every AWS service in an exam option, ask 5 questions:

| # | Question | Discriminator example |
|---|---|---|
| 1 | **Is it DETECTION or PREVENTION?** | Config detects; Lambda remediates |
| 2 | **What LAYER does it operate at?** | Flow Logs = metadata; Traffic Mirroring = packets |
| 3 | **What's its ATTACHMENT POINT?** | WAF on CloudFront/ALB only, not EC2 |
| 4 | **What's its REGIONAL CONSTRAINT?** | CloudFront cert = us-east-1; ALB cert = origin region |
| 5 | **What's its DEFAULT BEHAVIOR?** | CW Logs agent needs IAM + service running |

If you can answer these 5 about each service in the options, you eliminate most wrong answers in <30 seconds.

---

### Summary patches for Mock #2 prep

| Pattern | One-line reflex |
|---|---|
| A | Config detects, Lambda remediates (no built-in SSM doc for access key disable) |
| B | CloudFront viewer cert = us-east-1 ALWAYS; origin cert = origin region; Edge-opt API GW = us-east-1 |
| C | Flow Logs = metadata only; Traffic Mirroring / host agent / proxy = packet content |
| D | CW Logs agent: IAM perms (4 actions) + service running (verify via SSM Run Command) + network connectivity |
| E | Public-facing app hardening = move backends to private subnets behind ALB; CloudFront + WAF for edge |

---

*Updated June 6, 2026 — added Tier 7 with TD D3 practice misses + complete ACM cert region master table + 5 cross-cutting patterns for Mock #2 prep.*

---

## ⚠️ Tier 8 — TD D4 IAM practice misses (June 6 afternoon)

D4 IAM practice on TD: 9/15 = 60% (regression from Mock #1 73%). Diagnosis: AD/Federation cluster is the weak spot (3 of 6 misses). Patches below.

### Pattern F — AD trust direction semantics (TD Q5)

**THE RULE that trips people up:**

> In an Active Directory one-way trust where **"X trusts Y"**: users in **Y can access X's resources**. The DIRECTION of trust determines who can access what.

Think of it as: "X trusts Y" = "X says I trust Y's users" = "Y's users can come INTO X"

### The Q5 scenario decoded

Requirement: "Cloud-based users must be PREVENTED from accessing on-premises systems."

| Trust setup | What happens | Satisfies "cloud users can't access on-prem"? |
|---|---|---|
| **AWS AD trusts on-prem AD** | On-prem users → can access AWS<br>AWS users → CANNOT access on-prem | ✅ YES — this is the answer |
| On-prem AD trusts AWS AD | AWS users → can access on-prem<br>On-prem users → CANNOT access AWS | ❌ NO — violates requirement |
| Two-way trust | Both directions work | ❌ NO — allows cloud users into on-prem |

### Memorization aid (the one-line rule)

```
"X trusts Y" = "Y's users access X"

The trusting side is the RESOURCE owner.
The trusted side is the IDENTITY provider.

For "AWS users CANNOT access on-prem":
  → On-prem must NOT trust AWS
  → AWS must trust on-prem (one-way, AWS = trusting side)
```

### Pattern G — Federation uses ROLES, NOT GROUPS (TD Q13)

**THE RULE**:
- Federated users from on-prem AD (via SAML/ADFS) get **temporary STS credentials by ASSUMING an IAM role**
- IAM groups are containers for IAM users; they DO NOT generate temporary credentials
- The mapping is: AD group/claim → **IAM role** (not IAM group)

### The federation architecture (memorize)

```
On-prem AD user logs in
   ↓
ADFS (Active Directory Federation Services) authenticates
   ↓
ADFS generates a SAML assertion containing AD group/claim
   ↓
User submits SAML to AWS STS via AssumeRoleWithSAML API
   ↓
STS returns temporary AWS credentials (based on the IAM ROLE the SAML maps to)
   ↓
User makes AWS API calls with those temporary credentials
```

### Required components for on-prem AD → AWS federation

| Component | Purpose |
|---|---|
| **IAM SAML 2.0 Identity Provider** | Tells AWS to trust the IdP (your ADFS) |
| **IAM Roles** (one per AD group/permission set) | What federated users assume |
| **AssumeRoleWithSAML** STS API | Returns temporary credentials |
| **ADFS Relying Party Trust** for AWS | On the ADFS side, says "trust AWS as a relying party" |

**Construction error to recognize**:
- ❌ "IAM groups for federation" — groups can't be assumed by STS
- ❌ "Cognito Identity Pool for on-prem AD" — Cognito Identity Pool is for AWS service access from mobile/web apps, not enterprise federation
- ❌ "Cognito User Pool to authenticate AD users" — User Pool is for app users, not enterprise SSO
- ❌ "AWS RAM for AD federation" — RAM shares AWS resources, not for identity federation

### Pattern H — AWS Directory Service identification (TD Q14)

For "connect on-prem Microsoft AD with AWS" → **AWS Directory Service** (specifically **AWS Managed Microsoft AD**).

**Quick recall map**:

| Need | Right service |
|---|---|
| Run Microsoft AD in AWS that supports trusts to on-prem AD | AWS Directory Service for Microsoft AD (Enterprise Edition) |
| Proxy existing on-prem AD to AWS (no AD in AWS) | AD Connector |
| Small standalone Samba 4-compatible directory | Simple AD |
| Mobile/web app authentication | Cognito User Pool |
| Mobile/web app authorization to AWS resources | Cognito Identity Pool |
| Workforce SSO across AWS accounts | IAM Identity Center (formerly AWS SSO) |
| Hierarchical data store for app developers | Amazon Cloud Directory (NOT for AD use cases) |

**The exam trap**: Cloud Directory and Cognito Identity Pool both contain "directory" or "identity" but are NOT for AD integration. AWS Directory Service is the one.

### AD service comparison table (memorize cold)

| Service | Use case | Trust to on-prem AD? |
|---|---|---|
| **AWS Managed Microsoft AD** | Full AD in AWS, AD-aware apps | ✅ Yes (1-way or 2-way) |
| **AD Connector** | Proxy to on-prem AD (no AD stored in AWS) | ❌ N/A (it IS the bridge) |
| **Simple AD** | Small dirs, basic LDAP/Samba 4 | ❌ No trust support |
| **Cognito User Pool** | App user authentication (JWT tokens) | ❌ Different service |
| **Cognito Identity Pool** | App user → AWS credentials | ❌ Different service |
| **IAM Identity Center** | Workforce SSO across AWS accounts | ✅ Via Managed AD or external IdP |

---

### Pattern I — GuardDuty depth: EKS Protection + Runtime Monitoring (TD Q6)

For Kubernetes audit logs AND runtime signals → **BOTH features required**:

| GuardDuty feature | What it sees |
|---|---|
| **EKS Protection** | Kubernetes API audit logs (privilege escalation, anomalous K8s API calls) |
| **Runtime Monitoring** (formerly EKS Runtime Monitoring) | OS activity, network behavior, file access ON CLUSTER NODES |

**The trap**: questions split these into two options and you have to recognize BOTH are needed for "audit logs AND runtime signals."

GuardDuty feature naming (memorize):
- **GuardDuty foundational** — VPC Flow Logs, CloudTrail mgmt events, DNS
- **EKS Protection** — Kubernetes audit logs
- **Runtime Monitoring** — EKS + ECS + EC2 (process-level via agent)
- **Malware Protection (EBS scan)** — snapshot-based malware detection
- **S3 Protection** — CloudTrail S3 data events
- **RDS Protection** — RDS login anomaly detection
- **Lambda Protection** — Lambda network anomaly

**Exam pattern**: "Kubernetes threats" alone → EKS Protection. "Kubernetes threats AND runtime signals" → EKS Protection + Runtime Monitoring.

---

### Pattern J — "Least permissive" trap: single action > managed policy (TD Q7)

**THE RULE**: When question says **"LEAST permissive"** or **"least privilege"** for IAM:
- **Single API action** wins (e.g., `cloudwatch:PutMetricData`)
- **AWS-managed policies** ALMOST ALWAYS lose (they're broad-scoped)

### CloudWatch managed policies vs single action

| Policy | Permissions |
|---|---|
| `CloudWatchFullAccess` | Full read + write on CloudWatch + EC2 metadata | ❌ Way over-permissive |
| `CloudWatchReadOnlyAccess` | Read-only CloudWatch | ❌ No write, breaks scenario |
| `CloudWatchActionsEC2Access` | Read CW + EC2 metadata + Stop/Terminate/Reboot EC2 | ❌ Includes destructive EC2 actions |
| **`cloudwatch:PutMetricData` only** | Just publish metrics | ✅ Least privilege for "publish custom metrics" |

### The least-privilege heuristic

When matching to "LEAST permissive":
1. Look for SINGLE API action options
2. Eliminate managed policies (`*FullAccess`, `*Access`, etc.) unless they're scoped to exactly the action
3. Check for tangential permissions (e.g., `CloudWatchActionsEC2Access` sneakily includes EC2 destructive actions)
4. If multiple single-action options exist, pick the most specific one to the use case

---

### Pattern K — IoT Core policy with Thing.ThingName (TD Q15)

For preventing client ID injection attacks on AWS IoT Core:

**Right pattern** (validates against IoT registry):
```json
{
  "Effect": "Allow",
  "Action": "iot:Connect",
  "Resource": "arn:aws:iot:region:account:client/${iot:Connection.Thing.ThingName}"
}
```

**Wrong pattern** (vulnerable to special char injection):
```json
{
  "Effect": "Allow",
  "Action": "iot:Connect",
  "Resource": "arn:aws:iot:region:account:client/${iot:ClientId}"
}
```

### Why this matters

- `iot:ClientId` = whatever the device claims its ID is (attacker-controllable)
- `iot:Connection.Thing.ThingName` = name registered in AWS IoT registry (verified during connection)
- Using `ClientId` allows special chars / injection
- Using `ThingName` forces match against the IoT registry → injection impossible

### Plus: AWS IoT Device Defender for fleet security

For ongoing IoT fleet security:
- Audit configurations
- Detect anomalies
- Mitigate device threats
- Recommended companion to scoped IoT Core policies

---

### Cross-cutting reflex updates for D4 IAM

Adding to the existing 5-question check (Tier 7):

| # | Question | New discriminator |
|---|---|---|
| 6 | **"Is this AD/federation?"** | If yes: IAM roles + AssumeRoleWithSAML + ADFS relying party trust + Directory Service |
| 7 | **"What's the trust direction?"** | "X trusts Y" = Y's users access X |
| 8 | **"Is this 'LEAST permissive'?"** | Single API action > managed policy |
| 9 | **"Do I need BOTH features?"** | GuardDuty audit logs (EKS Protection) + runtime (Runtime Monitoring) |
| 10 | **"Is the policy resource attacker-controllable?"** | Use registered identifiers (Thing.ThingName, not ClientId) |

---

### Summary patches for Mock #2 (D4 cluster)

| Pattern | One-line reflex |
|---|---|
| F | AD one-way trust direction: "X trusts Y" = Y's users access X (so AWS trusts on-prem = on-prem users access AWS, AWS users blocked from on-prem) |
| G | Federation uses IAM ROLES (not groups) + AssumeRoleWithSAML + ADFS relying party trust |
| H | Connecting on-prem AD to AWS = AWS Directory Service (Managed Microsoft AD with trust) |
| I | GuardDuty for K8s audit + runtime = EKS Protection + Runtime Monitoring (BOTH) |
| J | "LEAST permissive" = single API action > managed policy |
| K | IoT Core policy resource = `client/${iot:Connection.Thing.ThingName}` (NOT `iot:ClientId`) |

---

*Updated June 6, 2026 (afternoon) — added Tier 8 with TD D4 IAM misses, focus on AD/Federation cluster (3 misses), GuardDuty EKS depth, least-privilege managed-policy trap, IoT Core policy attacker-controllable resources.*

---

## ⚠️ Tier 9 — TD D1 + D2 practice (June 6 evening)

D1 Detection: **15/15 = 100%** (no misses — domain locked)
D2 Incident Response: **11/15 = 73%** (massive jump from Mock #1's 20%)

The 4 D2 misses cluster on a single high-value distinction (Route 53 logging types) plus 2 service-selection traps. Patches below.

### Pattern L — Route 53 Query Logging types (TD Q3 + Q13 — bit me TWICE)

This is THE recurring trap in D1/D2 questions. Lock the distinction cold.

#### Two completely different services with similar names

| Service | Logs WHAT | Destinations |
|---|---|---|
| **Route 53 DNS Query Logging** | **PUBLIC** DNS queries TO your hosted zones (queries FROM the internet TO your domain) | **CloudWatch Logs ONLY** |
| **Route 53 Resolver Query Logging** | **VPC-INTERNAL** DNS queries FROM resources INSIDE your VPC (EC2/Lambda/etc.) | CloudWatch Logs, S3, Kinesis Firehose |

#### The discriminator (memorize this exact phrasing)

| Question phrase | Right answer |
|---|---|
| "Log queries TO my public domain / hosted zone" | **DNS Query Logging** (CW Logs only) |
| "Public DNS queries" + "for my domain" | **DNS Query Logging** (CW Logs only) |
| "Log DNS queries FROM resources in my VPC" | **Resolver Query Logging** |
| "VPC instances making DNS queries" | **Resolver Query Logging** |
| "Track DNS queries from EC2/Lambda" | **Resolver Query Logging** |
| "Detect DNS exfiltration from VPC" | **Resolver Query Logging** (+ optional DNS Firewall) |

#### Memorization mnemonic

**"DNS = your DOMAIN getting queried (public)"**
**"Resolver = your RESOURCES doing the querying (internal)"**

#### Construction errors

- ❌ "Route 53 DNS query logs to S3" — DNS query logs go ONLY to CloudWatch Logs
- ❌ "Route 53 DNS query logs to Firehose" — same
- ❌ "Route 53 Resolver query logging for public hosted zone queries" — wrong scope

### Pattern M — CloudWatch Contributor Insights for "top N" / "frequently occurring"

When the question asks for "MOST FREQUENTLY OCCURRING," "TOP CONTRIBUTORS," "top talkers," "top-N" patterns:

**Right answer**: **CloudWatch Contributor Insights**

#### Analytical tool selection

| Need | Right tool |
|---|---|
| "Top N frequently occurring" / "top contributors" | **CloudWatch Contributor Insights** |
| Ad-hoc SQL queries on S3-stored logs | **Athena** |
| Query language for CW Logs (interactive) | **CW Logs Insights** |
| Time-series metric visualization | CloudWatch dashboards |
| Aggregated security findings dashboard | Security Hub |

#### Why Contributor Insights wins over Athena for "top N"

- Contributor Insights is purpose-built for ranking — extracts the top contributors automatically
- Athena requires writing SQL + processing results — more steps
- Contributor Insights works directly on CW Logs — no S3 export step

**Exam reflex**: "frequently occurring" + "DNS queries" → Resolver Query Logging → **Contributor Insights**

### Pattern N — Amazon Detective is the INVESTIGATION tool (TD Q14)

When the question says **"investigate a GuardDuty finding"** or **"determine the scope of the compromise"** or **"analyze suspicious activity"** → **Amazon Detective**.

#### Detection vs Investigation vs Aggregation (memorize the categories)

| Service | Category | When to pick |
|---|---|---|
| **GuardDuty** | DETECTION | "Detect threats from logs" |
| **Inspector** | DETECTION (vulnerabilities) | "Find CVEs in EC2/Lambda/ECR" |
| **Macie** | DETECTION (sensitive data) | "Find PII in S3" |
| **Config** | DETECTION (config compliance) | "Audit resource configurations" |
| **Detective** | **INVESTIGATION** | "Investigate finding scope," "analyze attack pattern" |
| **Security Hub** | AGGREGATION | "Central security dashboard," "CIS Benchmarks" |
| **CloudTrail** | LOGGING | "Audit API calls" |

#### Detective's unique capability

Detective is the ONLY service that does **graph-based behavioral investigation**:
- Pivots directly from a GuardDuty finding
- Auto-correlates VPC Flow Logs + CloudTrail + EKS audit + GuardDuty data
- Visualizes connections between entities (IPs, users, roles, resources)
- Determines scope and timeline of suspicious activity
- Cannot generate findings itself — it's a pivot tool

**Prerequisite**: GuardDuty must be enabled for at least 48 hours before Detective can be enabled in the same account.

#### Exam pattern recognition

| Question phrase | Right answer |
|---|---|
| "Investigate the scope of a GuardDuty finding" | Detective |
| "Determine if a flagged instance is compromised" | Detective |
| "Analyze attack patterns / timeline" | Detective |
| "Visualize relationships between AWS entities" | Detective |
| "Generate findings about compromised instances" | GuardDuty (not Detective — Detective doesn't generate) |
| "Centralize findings from multiple services" | Security Hub (not Detective) |
| "Scan for vulnerabilities" | Inspector (not Detective) |

### Pattern O — GuardDuty specific finding names

The exam occasionally tests specific finding name recognition. You don't need to memorize all of them — recognize the structure.

#### Finding name structure

```
Category : Resource / Action [.SubAction]
```

Examples:
- `UnauthorizedAccess:IAM/InstanceCredentialExfiltration.InsideAWS`
- `UnauthorizedAccess:IAM/InstanceCredentialExfiltration.OutsideAWS`
- `UnauthorizedAccess:EC2/TorClient`
- `CryptoCurrency:EC2/BitcoinTool.B!DNS`
- `Execution:EC2/MaliciousFile`
- `Recon:EC2/PortProbeUnprotectedPort`

#### Key finding categories to recognize

| Category | What it indicates |
|---|---|
| **UnauthorizedAccess** | Suspicious access to AWS resources (credential exfiltration, anonymous access, etc.) |
| **CryptoCurrency** | Crypto mining activity (Bitcoin/Monero) |
| **Backdoor** | Indicator of compromise (C2 communication) |
| **Execution** | Malware execution on instance |
| **Recon** | Reconnaissance/scanning activity |
| **Trojan** | Trojan/RAT-like activity |
| **Stealth** | Anti-forensics behavior (disabling logging, etc.) |
| **Impact** | Active attack (DoS, data destruction) |

#### Key resource types in finding names

- **EC2** — instance-level findings
- **IAM** — credential/identity findings
- **S3** — bucket/object findings
- **EKS/Kubernetes** — Kubernetes findings
- **Lambda** — Lambda function findings
- **RDS** — RDS database findings
- **Runtime** — agent-based runtime monitoring findings

---

### Pattern P — S3 Inventory for compliance audit (Q2 supplementary)

For "audit replication and encryption status of S3 objects" use **S3 Inventory**.

| Need | Right tool |
|---|---|
| Audit S3 object encryption status across many objects | **S3 Inventory** (daily/weekly CSV/Parquet reports) |
| Detect PII in S3 objects | Macie |
| Real-time S3 access alerts | CloudTrail data events + EventBridge |
| Investigate S3 access patterns | Detective |

**S3 Inventory output includes**: encryption status, replication status, object size, last modified, storage class, tags. Great for compliance reports.

---

### Cross-cutting reflex updates for D1 + D2

| # | Question | Discriminator |
|---|---|---|
| 11 | **"Public DNS or VPC DNS?"** | Public → DNS Query Logging (CW Logs only) / VPC → Resolver Query Logging (3 destinations) |
| 12 | **"Top N / frequently occurring?"** | CW Contributor Insights (NOT Athena, NOT CW Logs Insights) |
| 13 | **"Investigate vs Detect vs Aggregate?"** | Detective = investigate, GuardDuty/Inspector/Macie = detect, Security Hub = aggregate |
| 14 | **"What's the GuardDuty finding category?"** | Read Category:Resource/Action format to confirm relevance |
| 15 | **"Audit S3 objects' state?"** | S3 Inventory (encryption, replication, etc.) |

---

### Summary patches for Mock #2 (D1 + D2 cluster)

| Pattern | One-line reflex |
|---|---|
| L | Route 53 DNS QL = public domain queries → CW Logs only; Resolver QL = VPC queries → CW Logs/S3/Firehose |
| M | "Top N" / "frequently occurring" → CW Contributor Insights (not Athena) |
| N | "Investigate finding scope" → Detective (uniquely graph-based investigation) |
| O | GuardDuty finding names: Category:Resource/Action[.SubAction] (e.g., InstanceCredentialExfiltration.OutsideAWS) |
| P | "Audit S3 encryption/replication status" → S3 Inventory reports |

---

### Today's domain trajectory (Mock #1 → June 6 TD)

| Domain | Mock #1 | Today | Real-exam projection (with +28% delta) |
|---|---|---|---|
| D1 Detection | 73% | **100%** | ~95%+ 🔒 |
| **D2 IR** | **20%** (5q sample) | **73%** | ~85%+ ⬆️🚀 |
| D3 Infrastructure | 41% | 67% | ~75-80% |
| D4 IAM | 73% | 60% | ~75% (assuming Tier 8 AD/Fed patch lands) |
| D5 Data Protection | 47% | **93%** | ~95%+ 🔒 |
| D6 Governance | 83% | 87.5% | ~90%+ 🔒 |

**Weighted overall projection**: ~83-87% real exam. Comfortably above 75% pass line.

### Mock #2 protocol reminders (for tomorrow)

1. **USE THE FULL 210 MINUTES** — D2 test today was 18 min (1:14/q), way too fast. Slow down.
2. **Pace checks at Q22 (~75 min) and Q44 (~150 min)** — should be on track
3. **First-pass commit** — flag uncertain, move on, don't ruminate
4. **Last 30 min** — review ONLY flagged questions
5. **Don't change first-instinct answers** unless concrete reason
6. **Read constraint words** — "cannot," "must not," "least," "only," "all"
7. **Eliminate construction errors first** to speed up obvious wrong answers

### Decision criteria for Mock #2 (locked in, no goalpost moving)

| Raw Mock #2 score | Action |
|---|---|
| 80%+ | Stay June 10 — strong confidence |
| 70–79% | Stay June 10 — on track |
| 65–69% | Tough call — lean reschedule to June 17 |
| <65% | Reschedule to June 17 (free until 24h before) |

---

*Updated June 6, 2026 (evening) — added Tier 9 with TD D1 (100%) + D2 IR (73%) patches. Focus: Route 53 DNS vs Resolver query logging distinction (bit twice — lock cold), CW Contributor Insights for top-N, Detective as the investigation pivot tool, GuardDuty finding name structure, S3 Inventory for compliance audits. Trajectory complete for tomorrow's Mock #2 go/no-go.*

*Last updated: June 6, 2026 (evening)*
