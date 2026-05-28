# Domain 4: Identity & Access Management (20%)

![Domain 4 Identity and Access Management Infographic](../assets/domain-4-iam.webp)

> **SCS-C03 Update:** This is now the highest weighted domain at 20% (was 16% in SCS-C02)

## Overview

This domain covers identity management, access control, and federation in AWS.

---

## Task Statements & Skills (SCS-C03)

### Task 4.1: Design, implement, and troubleshoot authentication strategies

| Skill | Key Services |
|-------|-------------|
| 4.1.1: Design and establish identity solutions for human, application, and system authentication | **IAM Identity Center, Amazon Cognito, MFA, IdP integration** |
| 4.1.2: Configure mechanisms to issue temporary credentials | **AWS STS, Amazon S3 presigned URLs** |
| 4.1.3: Troubleshoot authentication issues | **CloudTrail, Amazon Cognito, IAM Identity Center permission sets, AWS Directory Service** |

### Task 4.2: Design, implement, and troubleshoot authorization strategies

| Skill | Key Services |
|-------|-------------|
| 4.2.1: Design and evaluate authorization controls for human, application, and system access | **Amazon Verified Permissions, IAM paths, IAM Roles Anywhere, resource policies (cross-account), IAM role trust policies** |
| 4.2.2: Design ABAC and RBAC strategies | Tag-based and attribute-based access control |
| 4.2.3: Design, interpret, and implement IAM policies following least privilege | **Permission boundaries, session policies** |
| 4.2.4: Analyze authorization failures to determine causes or effects | **IAM Policy Simulator, IAM Access Analyzer** |
| 4.2.5: Investigate and correct unintended permissions | **IAM Access Analyzer** |

---

## Key Services

| Service | Purpose |
|---------|---------|
| **AWS IAM** | Users, groups, roles, policies |
| **IAM Identity Center** | Centralized SSO (formerly AWS SSO) |
| **AWS Organizations** | Multi-account management |
| **AWS STS** | Temporary security credentials |
| **Amazon Cognito** | User identity for applications |
| **IAM Access Analyzer** | Analyze resource access and validate policies |
| **Amazon Verified Permissions** | Fine-grained authorization for applications (Cedar policy language) |
| **IAM Roles Anywhere** | IAM roles for workloads outside AWS |
| **AWS Directory Service** | Managed Active Directory and AD Connector |
| **IAM Policy Simulator** | Test and debug IAM policies |

---

## 🔑 AWS IAM

### Policy Evaluation Logic

1. **Explicit Deny** → DENY
2. **Organization SCP** → must allow
3. **Resource-based policy** → may allow
4. **Identity-based policy** → may allow
5. **IAM Permissions Boundary** → must allow
6. **Session policy** → must allow
7. **Implicit Deny** → DENY

### Policy Types

| Type | Attached To | Use Case |
|------|-------------|----------|
| **Identity-based** | Users, Groups, Roles | Grant permissions |
| **Resource-based** | Resources (S3, KMS, etc.) | Cross-account access |
| **Permissions Boundaries** | Users, Roles | Delegate permission management |
| **SCPs** | OUs, Accounts | Restrict maximum permissions |
| **Session Policies** | STS sessions | Limit session permissions |

### Best Practices

- Enable MFA for all users
- Use roles for applications
- Grant least privilege
- Use groups for permissions
- Rotate credentials regularly
- Remove unused credentials

### IAM Condition Keys (high-frequency exam topic)

Conditions are typed — the operator must match the value type. Top mismatch trap: `DateLessThan` with a numeric seconds value (always wrong unless seconds are interpreted as epoch).

#### Operator–value type pairings

| Operator family | Examples | Value type |
|---|---|---|
| **String** | `StringEquals`, `StringLike`, `StringNotEquals` | String, wildcards (`*`, `?`) with `StringLike` |
| **Numeric** | `NumericEquals`, `NumericLessThan`, `NumericGreaterThanEquals` | Number |
| **Date** | `DateEquals`, `DateLessThan`, `DateGreaterThan` | ISO 8601 (`"2026-12-31T00:00:00Z"`) or epoch seconds |
| **Bool** | `Bool` | `"true"` / `"false"` (strings) |
| **IpAddress** | `IpAddress`, `NotIpAddress` | CIDR (`"203.0.113.0/24"`) |
| **Arn** | `ArnEquals`, `ArnLike` | ARN with optional wildcards |
| **Null** | `Null` | `"true"` (key absent) or `"false"` (key present) |

#### High-yield global condition keys

| Condition key | Operator | Purpose |
|---|---|---|
| `aws:MultiFactorAuthPresent` | Bool | Request used MFA |
| `aws:MultiFactorAuthAge` | Numeric (seconds) | How long ago MFA was performed |
| `aws:SourceIp` | IpAddress | Caller IP address |
| `aws:SourceVpc` | String | Caller's VPC ID |
| `aws:SourceVpce` | String | Caller's VPC Endpoint ID |
| `aws:RequestedRegion` | String | Region of API call (great for SCPs) |
| `aws:SecureTransport` | Bool | Request over HTTPS (TLS) |
| `aws:PrincipalOrgID` | String | Caller's AWS Organizations ID |
| `aws:PrincipalOrgPaths` | StringLike | Caller's OU path |
| `aws:PrincipalTag/<key>` | String | Tag on the calling principal (ABAC) |
| `aws:ResourceTag/<key>` | String | Tag on the target resource (ABAC) |
| `aws:RequestTag/<key>` | String | Tag included in a `Create*` request (ABAC) |
| `aws:SourceArn` / `aws:SourceAccount` | Arn / String | Service-side caller (confused deputy prevention) |
| `aws:ViaAWSService` | Bool | Request made by service on user's behalf |
| `aws:CalledVia` | StringLike | Specific intermediary AWS service |
| `aws:CurrentTime` | Date | Time of request |
| `aws:EpochTime` | Numeric / Date | Time in epoch seconds |
| `kms:ViaService` | String | Which AWS service is using the KMS key |
| `s3:x-amz-server-side-encryption` | StringEquals | Encryption header on `PutObject` |
| `s3:x-amz-server-side-encryption-aws-kms-key-id` | StringEquals | Specific KMS key ID required for upload |

#### Common policy patterns

**Require MFA, max 3-hour MFA-authenticated session**:
```json
"Condition": {
  "Bool":             { "aws:MultiFactorAuthPresent": "true" },
  "NumericLessThan":  { "aws:MultiFactorAuthAge": "10800" }
}
```

**Deny non-HTTPS requests** (S3 bucket policy example):
```json
{
  "Effect": "Deny",
  "Action": "s3:*",
  "Principal": "*",
  "Resource": ["arn:aws:s3:::bucket/*", "arn:aws:s3:::bucket"],
  "Condition": { "Bool": { "aws:SecureTransport": "false" } }
}
```

**Restrict KMS usage to specific VPC endpoint**:
```json
"Condition": {
  "StringNotEquals": { "aws:sourceVpce": "vpce-1234abcdf5678c90a" }
}
```

**Confused deputy prevention** (resource-based policy for service trust):
```json
"Condition": {
  "StringEquals": { "aws:SourceAccount": "111122223333" },
  "ArnLike":      { "aws:SourceArn":     "arn:aws:s3:::my-bucket" }
}
```

**Enforce SSE-KMS on S3 uploads**:
```json
{
  "Effect": "Deny",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::bucket/*",
  "Condition": {
    "StringNotEquals": {
      "s3:x-amz-server-side-encryption": "aws:kms"
    }
  }
}
```

### `MaxSessionDuration` vs `aws:MultiFactorAuthAge` (don't confuse)

| | `MaxSessionDuration` (role attribute) | `aws:MultiFactorAuthAge` (condition key) |
|---|---|---|
| **Where set** | On the IAM role itself | In a policy `Condition` block |
| **What it limits** | Max duration of `AssumeRole` sessions | How recently MFA must have been performed |
| **Default** | 1 hour | N/A — only if condition added |
| **Max value** | 12 hours | Any number of seconds |

`MaxSessionDuration` is NOT a condition key and cannot appear in a policy condition. Common distractor.

### Key Documentation Links

- [Security best practices and use cases in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices-use-cases.html)
- [Security best practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [IAM users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users.html)
- [What is IAM?](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)
- [IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)

📖 **Documentation**: [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)

---

## 🏢 AWS Organizations

### Service Control Policies (SCPs)

- **Maximum permissions** for accounts in OU
- **Do not grant permissions** - only restrict
- **Don't affect management account**
- **Inheritance**: Parent → Child

### Common SCP Patterns

```json
// Deny all regions except specific ones
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["us-east-1", "eu-west-1"]
    }
  }
}
```

### Key Documentation Links

- [Service Control Policies (SCPs)](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html)
- [SCP examples](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_examples.html)
- [Management policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_management_policies.html)
- [Choosing an AWS cloud governance service](https://docs.aws.amazon.com/decision-guides/latest/cloud-governance-on-aws-how-to-choose/cloud-governance-on-aws-how-to-choose.html)

📖 **Documentation**: [SCPs User Guide](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html)

---

## 🔐 IAM Identity Center (AWS SSO)

### What You Need to Know

- **Centralized access** to multiple AWS accounts
- **Identity sources**: Identity Center directory, AD Connector, external IdP
- **Permission sets**: Collections of IAM policies
- **SAML 2.0** and **OIDC** federation

📖 **Documentation**: [IAM Identity Center User Guide](https://docs.aws.amazon.com/singlesignon/latest/userguide/)

---

## 🎫 AWS STS

### Temporary Credentials

- **AssumeRole**: For AWS resources or cross-account
- **AssumeRoleWithSAML**: For SAML federation
- **AssumeRoleWithWebIdentity**: For web/mobile apps (OIDC)
- **GetSessionToken**: For MFA-protected API access
- **GetFederationToken**: For federated users

### Session duration limits (memorize)

| API | Default | Maximum |
|---|---|---|
| `AssumeRole` | 1h | 12h (via role's `MaxSessionDuration`) |
| `AssumeRoleWithSAML` | 1h | 12h |
| `AssumeRoleWithWebIdentity` | 1h | 12h |
| `GetSessionToken` (IAM user) | 1h | 36h (root: max 1h only) |
| `GetFederationToken` | 12h | 36h |

### Regional vs Global STS endpoints (the opt-in region trap)

By default, AWS SDKs may call the **global STS endpoint** (`sts.amazonaws.com`, served from us-east-1). Tokens from the global endpoint are:

- **Version 1 format** (~1 KB+)
- **NOT valid in opt-in regions** (Hong Kong `ap-east-1`, Bahrain `me-south-1`, Cape Town `af-south-1`, Jakarta `ap-southeast-3`, UAE `me-central-1`, Hyderabad `ap-south-2`, Melbourne `ap-southeast-4`, Spain `eu-south-2`, Zurich `eu-central-2`, Tel Aviv `il-central-1`)
- Symptom: `InvalidClientTokenId` or token-rejected errors when calling services in opt-in regions

**Fix**: use **regional STS endpoints** (`sts.<region>.amazonaws.com`):

- Tokens are **version 2** (smaller, ~600 bytes)
- **Valid in ALL regions** including opt-in
- Lower latency

How to enable:
- SDK env var: `AWS_STS_REGIONAL_ENDPOINTS=regional`
- Boto3: `boto3.client('sts', region_name='us-east-1')`
- Account-level toggle: **IAM Console → Account settings → STS Region compatibility** → "Active in all AWS Regions" for each opt-in region

### Cross-Account Access

1. Create role in target account with trust policy
2. Grant assume role permission in source account
3. Use STS AssumeRole to get temporary credentials

### Service Roles & `iam:PassRole` (high-frequency exam pattern)

A **service role** is an IAM role that an AWS service assumes to perform actions on your behalf. The pattern shows up everywhere:

| Service | What the role does |
|---|---|
| **CloudFormation** | Creates/updates/deletes stack resources |
| **Lambda** | Function execution: write logs, access other AWS services |
| **EC2 instance profile** | Apps on the instance get temporary credentials |
| **ECS task role** | Task containers get AWS API access |
| **CodeBuild service role** | Build can fetch artifacts, push images, write logs |
| **Glue job role** | Job reads/writes S3, accesses Data Catalog, decrypts with KMS |
| **AWS Backup role** | Service can snapshot resources |
| **Auto Scaling service-linked role** | ASG can manage EC2 lifecycle |

#### Trust policy vs Permissions policy — distinct purposes (commonly confused)

| Policy | Question it answers | Example |
|---|---|---|
| **Trust policy** | **Who can assume this role?** | `"Principal": { "Service": "cloudformation.amazonaws.com" }` |
| **Permissions policy** | **What can the role do once assumed?** | `"Action": ["ec2:RunInstances", "iam:CreateRole", ...]` |

The role needing to **act on** many services goes in the *permissions* policy. The role being **assumed by** many services goes in the *trust* policy. **Don't conflate them.** "Composite principal containing every AWS service that might need deployment permissions" is the classic conflation trap — wrong answer pattern.

Composite principals in trust policies are **legitimate when**: multiple services genuinely need to assume the same role (rare). They're **wrong when**: the question is about granting the role permission to act on multiple services (that's the permissions policy's job).

#### `iam:PassRole` vs `sts:AssumeRole` — DO NOT confuse

Both involve "using a role" but at different lifecycle points:

| | `iam:PassRole` | `sts:AssumeRole` |
|---|---|---|
| **Who needs it** | User/principal **handing** a role to a service | Principal **becoming** a role |
| **Controls** | "Can I tell AWS service X to use role R?" | "Can I take on role R's permissions?" |
| **Typical scenario** | `lambda:CreateFunction --role arn:...:role/LambdaRole` | `aws sts assume-role --role-arn arn:...:role/DevAdmin` |
| **Where defined** | Identity policy on the user/role doing the passing | Identity policy on assumer + trust policy on target |
| **Key partner condition** | `iam:PassedToService` (lock to a specific service) | `sts:ExternalId` (confused deputy prevention) |

**Question decoder**:
- "Delegate role to an AWS service" / "Lambda must run as role X" / "EC2 instance profile" → **`iam:PassRole`**
- "User assumes a role" / "cross-account access" / "federation" → **`sts:AssumeRole`**

#### `iam:PassRole` — required scenarios

When a user hands a role to a service ("here, use this role"), the user needs `iam:PassRole`. The service then assumes the role.

Required for:
- `CreateStack --role-arn` (CloudFormation)
- `CreateFunction` with execution role (Lambda)
- `RunInstances` with instance profile (EC2)
- `CreateService` with task role (ECS)
- Any operation where the user assigns a role to a service to use

**Privilege escalation risk**: a user with broad `iam:PassRole` can pass a powerful role to a service they control (e.g., create a Lambda with admin role and invoke it). Mitigate with the **`iam:PassedToService` condition**:

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::*:role/CFN-DeploymentRole",
  "Condition": {
    "StringEquals": { "iam:PassedToService": "cloudformation.amazonaws.com" }
  }
}
```

This lets the user pass the role only to CloudFormation, not to any other service they might use to abuse it.

### IAM Paths — directory-like grouping of IAM resources (Skill 4.2.1)

IAM resources (users, roles, policies, groups) support **paths**, which act like directories in a file hierarchy. Default path is `/`.

Set at create time:
```bash
aws iam create-role --role-name MyRole --path /platform/
```

Path appears in the ARN:
```
arn:aws:iam::123456789012:role/platform/MyRole
arn:aws:iam::123456789012:role/product/DataPipelineRole
arn:aws:iam::123456789012:role/security/AuditorRole
```

#### Why paths matter for the exam

- **Structural grouping** — team encoded in the ARN itself; can't be untagged or stripped accidentally
- **Policy targeting** — write policies that match by path with `ArnLike` / `ArnNotLike`:
  ```json
  "Condition": {
    "ArnNotLike": {
      "aws:PrincipalArn": "arn:aws:iam::*:role/platform/*"
    }
  }
  ```
- **Listing/filtering** — `iam:ListRoles --path-prefix /platform/`
- **Org-wide guardrails** — pair IAM paths with SCPs for centralized control across hundreds of accounts

#### Paths vs tags for grouping IAM resources

| Approach | Pros | Cons |
|---|---|---|
| **Paths** | Structural, in the ARN, can't be accidentally removed, queryable via path-prefix | Set at create time only — can't change after creation (without delete + recreate) |
| **Tags** | Mutable, support arbitrary key/value | Tagging discipline required, easy to forget; `aws:ResourceTag` conditions don't apply cleanly to all IAM actions (e.g., `iam:PassRole` evaluates on the role ARN, not on tags attached to it) |

For SCS-C03: when the question asks about **grouping IAM ROLES by team** for organizational policy targeting, paths are usually the right answer over tags.

### SCP vs Permissions Boundary — when to use each

Both restrict permissions, but they operate at different scopes:

| Control | Scope | Granularity | Best for |
|---|---|---|---|
| **SCP** | Org / OU / account-wide | Affects ALL principals in the account/OU | **Centralized preventive guardrails across many accounts** |
| **Permissions boundary** | Per-role / per-user | Affects only that one principal | Delegated admin (a junior admin creates roles but the boundary caps what they can grant) |

**Question phrasing → which to pick**:
- "Hundreds of accounts" / "across the organization" / "centralized control" → **SCP**
- "Delegate IAM admin to a team but cap what they can grant" → **Permissions boundary**
- "Prevent member account root users from doing X" → **SCP** (boundaries don't affect root)
- "Limit one specific role's max permissions" → **Permissions boundary**

**Don't confuse SCP behavior**:
- SCPs are filters, not grants — they cap maximum permissions
- They affect ALL principals in member accounts (including root)
- They do NOT affect the management account
- They do NOT affect service-linked roles

#### CloudFormation service role — the textbook pattern

**Problem**: by default, CFN executes stack operations with the **caller's permissions**. If team members have different IAM permissions, the same stack deploys for some and fails for others.

**Solution**: dedicated CloudFormation service role.

1. Create role with trust on `cloudformation.amazonaws.com`
2. Attach permissions policies scoped to the **AWS resources CFN must create/update/delete** (EC2, IAM, Lambda, etc.) — NOT the CloudFormation stack ARNs themselves
3. Update each stack to use the role (`--role-arn` or `RoleARN` parameter)
4. Grant team members only `cloudformation:*` and `iam:PassRole` for that specific service role

Result: uniform deployment behavior regardless of who triggers. Centralized, least-privilege, auditable.

**Common trap distractors** for this pattern:
- "Composite principal with all needed services" → wrong (trust vs permissions confusion)
- "Scope policies to CFN stack ARNs" → wrong (need to scope to the underlying service resources)
- "Use Service Catalog instead" → over-engineering; doesn't fix the permission issue, just relocates it
- "Grant team members IAM admin so they can do anything" → violates least privilege

### Confused Deputy Problem (Skill 4.2.1)

When a 3rd-party service assumes a role in your account, an attacker could trick them into using your role on someone else's behalf. **Mitigation**: require `sts:ExternalId` in the role's trust policy — a shared secret between you and the 3rd party that they must pass on every `AssumeRole` call.

📖 **Documentation**: [STS User Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html)  
📖 **Regional endpoints**: [Manage STS in a Region](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_enable-regions.html)

---

## 👤 Amazon Cognito

### User Pools

- **User directory** for sign-up/sign-in
- **MFA support**
- **Lambda triggers** for customization
- **JWT tokens** for authentication

### Identity Pools

- **Federated identities** (Google, Facebook, SAML, etc.)
- **Temporary AWS credentials**
- **Guest access** support

📖 **Documentation**: [Cognito User Guide](https://docs.aws.amazon.com/cognito/latest/developerguide/)

---

## 🔍 IAM Access Analyzer

### What You Need to Know

- Identifies resources shared with **external entities**
- **Zone of trust**: Account or Organization
- **Findings**: Resources accessible outside zone of trust
- **Policy validation**: Check policies for errors
- **Policy generation**: Generate policies from CloudTrail

📖 **Documentation**: [Access Analyzer User Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html)

---

## ✅ Amazon Verified Permissions (Skill 4.2.1)

### What You Need to Know

- **Fine-grained authorization** service for applications
- Uses the **Cedar policy language** (developed by AWS) for expressive, auditable policies
- **Externalized authorization**: Remove authorization logic from application code
- **Policy stores**: Collections of Cedar policies for an application
- **Schema**: Define entity types, actions, and relationships
- Supports both **RBAC** and **ABAC** patterns in Cedar

### Key Concepts

| Concept | Description |
|---------|------------|
| **Policy Store** | Container for all policies related to an application |
| **Cedar Policies** | Allow/forbid rules defining who can do what on which resources |
| **Entities** | Principals (users), actions, and resources |
| **Schema** | Type definitions for entities and actions |
| **IsAuthorized API** | Runtime authorization check |

### Use Cases

- API authorization: determine if a user can perform an action on a resource
- Multi-tenant application authorization
- Replace complex application-side authorization logic with centralized policies
- Integrate with **Cognito** for user identity and **Verified Permissions** for authorization decisions

📖 **Documentation**: [Amazon Verified Permissions User Guide](https://docs.aws.amazon.com/verifiedpermissions/latest/userguide/)  
📖 **Cedar Language**: [Cedar Policy Language Reference](https://docs.cedarpolicy.com/)

---

## 🏢 AWS Directory Service (Skill 4.1.3)

### What You Need to Know

- **AWS Managed Microsoft AD**: Full Microsoft Active Directory in AWS
- **AD Connector**: Proxy to on-premises Active Directory (no caching)
- **Simple AD**: Samba-based basic AD (limited features)

### AWS Managed Microsoft AD

- Two-way **trust relationships** with on-premises AD
- Supports **Group Policy**, **Kerberos**, **LDAP**
- Integrates with: RDS for SQL Server, Amazon WorkSpaces, IAM Identity Center, FSx for Windows
- **Multi-Region replication** for low-latency access
- **Automatic patching** and **snapshot backups**

### AD Connector

- **Does not store directory data** in AWS — it is a **proxy/redirector** to on-premises AD
- **No trust relationship is established** by AD Connector — it just forwards auth requests. Any exam answer that says "create a forest trust with AD Connector" is wrong on construction.
- Use when you want to use existing on-premises AD for AWS authentication
- Supports MFA with existing RADIUS server
- Requires consistent network connectivity (VPN/Direct Connect) — no offline survival

### IAM Identity Center + on-premises AD (exam-critical)

Two valid architectures:

| Approach | When to use | Trust required? |
|---|---|---|
| **AD Connector** → on-prem AD | Simpler setup, no AWS-side directory to manage. Requires reliable on-prem connectivity. | **No** — AD Connector is a proxy. |
| **AWS Managed Microsoft AD** with **two-way forest trust** to on-prem AD | Preferred for multi-account, resilient setups. Identity Center can enumerate on-prem users/groups bidirectionally. | **Yes — two-way (bidirectional)**. |

**Why two-way trust (not one-way) for IAM Identity Center**: per AWS docs, Identity Center needs to query the trusted on-premises domain to look up users/groups for assignment to permission sets. A one-way trust (AWS Managed AD trusts on-prem only) is **insufficient** — users could authenticate but Identity Center can't enumerate the trusted directory for assignments.

**Don't confuse with ADFS**: ADFS is the older SAML 2.0 pattern (you run an ADFS server on-prem, AWS is configured as the Trusted Relying Party). That's a different mechanism — separate from IAM Identity Center which is AWS-managed SSO.

### Quick exam decoder for AD questions

| Phrase | Service / answer |
|---|---|
| "Active Directory" + "trust relationship" | AWS Managed Microsoft AD |
| "Active Directory" + "no infrastructure to manage" + "proxy" / "redirect" | AD Connector |
| "IAM Identity Center" + "on-prem AD" + "efficient multi-account" | Managed Microsoft AD + **two-way** forest trust |
| "SAML" + "ADFS" + "Relying Party" | Old-school SAML federation (not Identity Center) |
| "Samba-based" / "small directory" / "basic AD features" | Simple AD |

### Security & Troubleshooting (Skill 4.1.3)

- Verify VPC connectivity and DNS resolution between AD Connector and on-premises AD
- Check security group rules for LDAP (389), LDAPS (636), Kerberos (88), DNS (53)
- Monitor with CloudTrail and CloudWatch for authentication failures
- Use AD Connector health checks for connectivity status

📖 **Documentation**: [AWS Directory Service Admin Guide](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/)

---

## 🌐 IAM Roles Anywhere (Skill 4.2.1)

### What You Need to Know

- Provides **temporary AWS credentials** to workloads **outside of AWS**
- Uses **X.509 certificates** issued by a trusted Certificate Authority (CA)
- **Trust anchors**: Reference to CA (ACM Private CA or external CA)
- **Profiles**: Define which IAM role to assume and session policies
- Eliminates need for **long-term access keys** for on-premises workloads

### Key Components

| Component | Description |
|-----------|------------|
| **Trust Anchor** | CA certificate that IAM Roles Anywhere trusts |
| **Profile** | Maps to an IAM role with optional session policies |
| **Subject** | The workload presenting a certificate |
| **CRL** | Certificate Revocation List for revoked certificates |

### Use Cases

- On-premises servers accessing AWS APIs without access keys
- Hybrid workloads that need temporary AWS credentials
- IoT devices or edge computing with certificate-based authentication

📖 **Documentation**: [IAM Roles Anywhere User Guide](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/)

---

## 🧪 IAM Policy Simulator (Skill 4.2.4)

### What You Need to Know

- **Test IAM policies** without deploying them
- Simulate API calls to see if they would be allowed or denied
- Works with **identity-based**, **resource-based**, **SCPs**, and **permissions boundaries**
- Available as **console tool** and **API** (for automation)
- Use for **troubleshooting** access denied errors

### Use Cases

- Test new policies before attaching to users/roles
- Debug "Access Denied" errors by simulating the exact API call
- Validate SCPs don't block required operations
- Verify cross-account access configurations

📖 **Documentation**: [IAM Policy Simulator](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_testing-policies.html)

---

## 🔗 S3 Presigned URLs (Skill 4.1.2)

### What You Need to Know

- **Time-limited URLs** for S3 object access without AWS credentials
- Generated by IAM users or roles with S3 permissions
- **Expiration**: Configurable (default varies, max depends on credential type)
- Permissions of the presigned URL match the **generating identity's permissions**
- STS temporary credentials: URL expires when the **shorter** of URL expiration or token expiration

### Security Considerations

- Use short expiration times for sensitive content
- Presigned URLs can be used by anyone who has the URL (no additional auth)
- Audit generation via CloudTrail
- Consider using CloudFront signed URLs for broader distribution control

📖 **Documentation**: [S3 Presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)

---

## 🏷️ ABAC and RBAC Strategies (Skill 4.2.2)

### Attribute-Based Access Control (ABAC)

- Authorization based on **tags/attributes** attached to principals and resources
- Use `aws:PrincipalTag`, `aws:ResourceTag`, and `aws:RequestTag` condition keys
- **Scalable**: New resources automatically get correct permissions based on tags
- Fewer policies needed (one policy can cover many resources)

### Role-Based Access Control (RBAC)

- Authorization based on **job function/role** membership
- Create IAM policies per role (e.g., Admin, Developer, ReadOnly)
- More traditional approach, easier to understand
- Requires policy updates when new resources are added

### When to Use Each

| Criteria | ABAC | RBAC |
|----------|------|------|
| **Scale** | Better for large, dynamic environments | Better for smaller, static environments |
| **Policy count** | Fewer policies | More policies |
| **New resources** | Auto-covered by tags | Requires policy updates |
| **Auditing** | Tag-based audit trails | Role membership audit |

📖 **Documentation**: [ABAC for AWS](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction_attribute-based-access-control.html)

---

## 📺 Recommended Videos

- [AWS re:Inforce 2022: Security best practices with AWS IAM (IAM201)](https://www.youtube.com/watch?v=SMjvtxXOXdU)
- [AWS re:Invent 2022: Harness power of IAM policies with Access Analyzer (SEC313)](https://www.youtube.com/watch?v=x-Kh8hKVX74)
- [AWS re:Invent 2018: Become an IAM Policy Master (SEC316-R1)](https://www.youtube.com/watch?v=YQsK4MtsELU)

---

## ✅ Exam Tips

- Memorize the policy evaluation order
- Know the difference between policy types
- Understand SCPs (don't grant, only restrict)
- Know cross-account access patterns
- Understand Cognito User Pools vs Identity Pools
- Know Access Analyzer zone of trust concept
- Understand **Amazon Verified Permissions** and **Cedar policy language** for application-level authorization (Skill 4.2.1)
- Know **IAM Roles Anywhere** for providing temporary credentials to on-premises workloads (Skill 4.2.1)
- Understand **AWS Directory Service** types: Managed Microsoft AD vs AD Connector vs Simple AD (Skill 4.1.3)
- Know **IAM Policy Simulator** for testing and debugging policies (Skill 4.2.4)
- Understand **S3 presigned URLs** and their expiration behavior with STS credentials (Skill 4.1.2)
- Know **ABAC vs RBAC** patterns and when to use each (Skill 4.2.2)
- Understand **IAM paths** for organizing IAM resources (Skill 4.2.1)


---

## Practice Quizzes

| Topic | Questions | Quiz |
|-------|-----------|------|
| **IAM Fundamentals: Users, Groups, Policy Evaluation and Credentials** | 25 | [View](../quizzes/iam-fundamentals-comprehensive.html) |
| **IAM Policies and Permissions** | 25 | [View](../quizzes/iam-policies-comprehensive.html) |
| **AWS Organizations and SCPs** | 20 | [View](../quizzes/organizations-scps-comprehensive.html) |
| **IAM Identity Center and Federation** | 20 | [View](../quizzes/identity-center-federation-comprehensive.html) |
| **Amazon Cognito and AWS STS** | 18 | [View](../quizzes/cognito-sts-comprehensive.html) |
| **AWS Verified Permissions (Cedar)** | 25 | [View](../quizzes/verified-permissions-comprehensive.html) |
| **IAM Roles Anywhere and Directory Service** | 25 | [View](../quizzes/rolesanywhere-directoryservice-comprehensive.html) |
| **IAM Access Analyzer, Policy Simulator and ABAC/RBAC** | 25 | [View](../quizzes/accessanalyzer-policysim-abac-comprehensive.html) |

> 8 quizzes | 183 questions total for Domain 4

---

*Last updated: January 2026 (SCS-C03)*
