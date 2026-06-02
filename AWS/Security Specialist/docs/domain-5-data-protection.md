# Domain 5: Data Protection (18%)

![Domain 5 Data Protection Infographic](../assets/domain-5-data-protection.webp)

## Overview

This domain covers data protection in transit, at rest, and management of confidential data, credentials, secrets, and cryptographic keys in AWS.

---

## Task Statements & Skills (SCS-C03)

### Task 5.1: Design and implement controls for data in transit

| Skill | Key Services |
|-------|-------------|
| 5.1.1: Design and configure mechanisms to require encryption when connecting to resources | **ELB security policies, TLS configurations** |
| 5.1.2: Design and configure mechanisms for secure and private access to resources | **AWS PrivateLink, VPC endpoints, AWS Client VPN, AWS Verified Access** |
| 5.1.3: Design and configure inter-resource encryption in transit | **Amazon EMR inter-node encryption, Amazon EKS, SageMaker AI, Nitro encryption** |

### Task 5.2: Design and implement controls for data at rest

| Skill | Key Services |
|-------|-------------|
| 5.2.1: Design, implement, and configure data encryption at rest | **AWS CloudHSM, AWS KMS, client-side/server-side encryption** |
| 5.2.2: Design and configure mechanisms to protect data integrity | **S3 Object Lock, S3 Glacier Vault Lock, versioning, digital code signing** |
| 5.2.3: Design automatic lifecycle management and retention solutions | **S3 Lifecycle policies, S3 Object Lock, Amazon EFS Lifecycle policies, Amazon FSx for Lustre backup policies** |
| 5.2.4: Design and configure secure data replication and backup solutions | **Amazon Data Lifecycle Manager, AWS Backup, ransomware protection, AWS DataSync** |

### Task 5.3: Design and implement controls to protect confidential data, credentials, secrets, and cryptographic key materials

| Skill | Key Services |
|-------|-------------|
| 5.3.1: Design management and rotation of credentials and secrets | **AWS Secrets Manager** |
| 5.3.2: Manage and use imported key material | **KMS imported key material, external key stores (XKS)** |
| 5.3.3: Describe differences between imported key material and AWS generated key material | KMS key material origins |
| 5.3.4: Mask sensitive data | **CloudWatch Logs data protection policies, Amazon SNS message data protection** |
| 5.3.5: Create and manage encryption keys and certificates across Regions | **AWS KMS customer managed keys, AWS Private Certificate Authority** |

---

## Key Services

| Service | Purpose |
|---------|---------|
| **AWS KMS** | Key management and encryption |
| **AWS CloudHSM** | Hardware security modules |
| **AWS Secrets Manager** | Secrets rotation and management |
| **AWS Certificate Manager** | SSL/TLS certificates |
| **AWS Private Certificate Authority** | Private CA for internal certificates |
| **Amazon Macie** | Data discovery and classification |
| **S3 Encryption** | Object encryption options |
| **AWS PrivateLink** | Private connectivity to services without internet |
| **AWS Client VPN** | Managed VPN for remote access |
| **AWS Verified Access** | Zero-trust access to applications |
| **Amazon Data Lifecycle Manager** | Automated EBS snapshot/AMI lifecycle |
| **AWS Backup** | Centralized backup across AWS services |
| **AWS DataSync** | Secure data transfer and replication |
| **S3 Object Lock / Glacier Vault Lock** | Immutable data storage (WORM) |
| **Amazon EFS** | Shared file storage with lifecycle policies |
| **Amazon FSx for Lustre** | High-performance file storage with backup policies |

---

## 🔐 AWS KMS

### Key Types

| Type | Management | Use Case |
|------|------------|----------|
| **AWS Managed Keys** | AWS manages | Default encryption for AWS services |
| **Customer Managed Keys** | You manage | Custom key policies, rotation control |
| **AWS Owned Keys** | AWS owns | Shared across accounts (SSE-S3) |

### Key Operations

- **Encrypt/Decrypt**: Up to 4KB of data
- **GenerateDataKey**: Create data key for client-side encryption
- **GenerateDataKeyWithoutPlaintext**: For envelope encryption

### Key Policies

```json
{
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::111122223333:role/ExampleRole"},
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:GenerateDataKey"
  ],
  "Resource": "*"
}
```

### Grants

- **Temporary permissions** without modifying key policy
- **GranteePrincipal**: Who receives permissions
- **RetiringPrincipal**: Who can retire the grant
- Use for **service integrations**

### Key Rotation

- **Automatic rotation**: Configurable period **90–2560 days** (default ~1 year). The fixed "annual only" limit changed in 2022.
- **Auto-rotation only works for**: symmetric CMKs with AWS-generated key material. **Not supported for**: asymmetric keys, imported key material, custom key stores (CloudHSM-backed).
- **On-demand / manual rotation** (universal pattern, works for all key types):
  1. Create new CMK
  2. Update the **alias** to point at the new CMK
  3. Keep the old CMK active so existing ciphertext can still be decrypted
- Old key material is retained for decryption — **rotation does NOT re-encrypt existing data**. Each encrypted blob stores an EDK referencing the exact CMK that wrapped it, and KMS auto-routes decryption to that CMK regardless of aliases.
- **Re-encryption** (separate operation) is required only when:
  - Key was compromised and you want to retire it
  - Compliance demands no live data depends on the old key
  - You want to delete the old CMK (7–30 day waiting period)
- Use the **`kms:ReEncrypt` API** for re-encryption — safer than Decrypt+Encrypt because the plaintext DEK never leaves KMS.

### Exam decoder for KMS rotation

| Phrase in question | Probable answer |
|---|---|
| "Annual rotation, minimal effort, symmetric, AWS-generated" | Enable auto-rotation |
| "Immediate / on-demand rotation", "customer requested", "compliance event" | Manual: new CMK + alias swap |
| "Customer brought their own key material" / "imported" | Manual only (auto disabled) |
| "Asymmetric key needs rotation" | Manual only (auto disabled) |
| "Custom key store / CloudHSM-backed" | Manual only (auto disabled) |
| "Minimal application changes" | Use alias (don't hardcode key IDs in app) |
| "Key was compromised, retire it" | Manual rotation + `ReEncrypt` existing data + schedule old CMK deletion |

### KMS Key Store compatibility matrix (TRAP — imported keys can ONLY go in default store)

> **Imported key material can ONLY go into the DEFAULT KMS key store, never into a Custom Key Store or External Key Store.** Custom Key Stores rely on CloudHSM hardware to generate keys — bypassing that with imports would defeat the hardware-rooted security model.

| Feature | Default Key Store | Custom Key Store (CloudHSM) | External Key Store (XKS) |
|---|---|---|---|
| AWS-generated key material | ✅ Yes (default) | ✅ Yes (via CloudHSM) | N/A (key lives outside AWS) |
| **Imported key material** | ✅ **YES — only place it works** | ❌ **NO** | N/A |
| Symmetric keys | ✅ Yes | ✅ Yes | ✅ Yes |
| Asymmetric keys | ✅ Yes | ✅ Yes | Limited |
| Auto-rotation | ✅ (symmetric AWS-gen only) | ❌ No | ❌ No |
| **Custom expiration date** | ✅ **Only with imported material** | ❌ No | Depends on external HSM |
| **Immediate delete** (no 7-day wait) | ✅ **Only with imported material** via `DeleteImportedKeyMaterial` | ❌ No | Depends |
| Customer manages HSM | No | ✅ Yes | ✅ Yes |
| FIPS 140-2 Level | 3 | 3 | Depends on your HSM |

**Reflex rule**:
- "Import own key material + custom expiration date" → **Default key store, customer-managed key, EXTERNAL origin**
- "Custom key store" → CloudHSM-backed, AWS-generated keys, no imports allowed
- "HSM must be outside AWS entirely" → XKS

### Key Documentation Links

- [AWS KMS best practices - Introduction](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-kms-best-practices/introduction.html)
- [Key management best practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-kms-best-practices/key-management.html)
- [AWS KMS keys concepts](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html)
- [Create symmetric encryption KMS key](https://docs.aws.amazon.com/kms/latest/developerguide/create-symmetric-cmk.html)
- [Best practices for KMS grants](https://docs.aws.amazon.com/kms/latest/developerguide/grant-best-practices.html)

📖 **Documentation**: [KMS Developer Guide](https://docs.aws.amazon.com/kms/latest/developerguide/)

---

## 🔒 AWS CloudHSM

### What You Need to Know

- **FIPS 140-2 Level 3** validated
- **Single-tenant HSM** in your VPC
- **You control keys** - AWS cannot access
- **Cluster for HA** - minimum 2 HSMs across AZs
- **Custom Key Store**: Integrate with KMS

### Use Cases

- SSL/TLS offloading
- PKCS#11, JCE, CNG interfaces
- Certificate authority (CA)
- Oracle TDE, SQL Server TDE

📖 **Documentation**: [CloudHSM User Guide](https://docs.aws.amazon.com/cloudhsm/latest/userguide/)

---

## 🤫 AWS Secrets Manager

### What You Need to Know

- **Automatic rotation** for RDS, Redshift, DocumentDB
- **Lambda-based rotation** for custom secrets
- **Cross-account access** via resource policies
- **Versioning**: AWSCURRENT, AWSPENDING, AWSPREVIOUS

### Rotation Process

1. **createSecret**: Create new secret version (AWSPENDING)
2. **setSecret**: Update database credentials
3. **testSecret**: Verify new credentials work
4. **finishSecret**: Move AWSPENDING to AWSCURRENT

### Key Documentation Links

- [What is AWS Secrets Manager?](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
- [Rotate Secrets Manager secrets](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html)
- [Rotate external secrets](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotate-secrets_external.html)
- [Secrets in Lambda functions](https://docs.aws.amazon.com/secretsmanager/latest/userguide/retrieving-secrets_lambda.html)
- [Manage credentials pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/manage-credentials-using-aws-secrets-manager.html)

📖 **Documentation**: [Secrets Manager User Guide](https://docs.aws.amazon.com/secretsmanager/latest/userguide/)

---

## 📜 AWS Certificate Manager (ACM)

### What You Need to Know

- **Free public certificates** for AWS services
- **Automatic renewal** for ACM-issued certificates
- **Import external certificates** (you manage renewal)
- **Private CA**: ACM Private Certificate Authority

### Certificate exportability — the exam-critical distinction

| Cert origin | Private key exportable? | Where can you install it? |
|---|---|---|
| **ACM-issued (Amazon-issued, public)** | ❌ **No** — locked inside ACM | Only on AWS-managed services that integrate with ACM: ALB, NLB, CloudFront, API Gateway, App Runner, ELB Classic, Cognito custom domains, Elastic Beanstalk |
| **Imported third-party cert** | ✅ Yes — you already have the private key | Anywhere — ALB *and* EC2, on-prem, containers, anywhere a `.pem` works |
| **ACM Private CA-issued end-entity cert** | ✅ Yes (`ExportCertificate` API) | Anywhere internal. Designed for backend services, mTLS, IoT devices |
| **Self-signed** | ✅ Yes | Anywhere — but no public trust (browser warnings) |

**Key exam trap**: any question that asks you to install the *same* cert on an ALB *and* on EC2 instances cannot use an ACM-issued cert (non-exportable). You need either:
- **Imported third-party cert** → use on both
- **ACM-issued cert on ALB + a different cert on EC2** (Private CA-issued or self-signed both work for the backend hop, since ALB doesn't validate backend chains — see below)

### End-to-end TLS patterns (client → ALB → EC2)

| Pattern | Front (client → ALB) | Back (ALB → EC2) | When to use |
|---|---|---|---|
| **TLS terminate at ALB only** | ACM-issued public cert | HTTP (plaintext) | When backend hop is private and compliance allows; cheapest, lowest CPU |
| **Re-encrypt to backend** | ACM-issued public cert | EC2 with self-signed or Private CA cert | Most common production pattern. ALB does NOT validate backend cert chain, so self-signed works. |
| **Same imported cert end-to-end** | Imported 3rd-party cert | Same cert installed on EC2 | The pattern enforced when answer choices restrict you (per the exam question above) |
| **Private CA for backend** | ACM-issued public cert | AWS Private CA cert on EC2, deployed via SSM, auto-renewable | Cleanest production pattern: separate rotation cycles, internal trust hierarchy |

**Key nuance**: **ALB does NOT validate the backend certificate chain**. When ALB connects to EC2 over HTTPS, it negotiates TLS but accepts whatever cert the backend presents (signed, self-signed, expired — encryption still happens, no trust check). The cert trust requirement only applies on the **client → ALB** hop (browsers validate). This means the "self-signed on EC2" answer is only wrong when self-signed is also used on the public-facing ALB.

### Integration Points

- CloudFront
- Elastic Load Balancer
- API Gateway
- Elastic Beanstalk

📖 **Documentation**: [ACM User Guide](https://docs.aws.amazon.com/acm/latest/userguide/)  
📖 **Importing certs**: [Importing certificates into ACM](https://docs.aws.amazon.com/acm/latest/userguide/import-certificate.html)  
📖 **ALB HTTPS listener**: [Create HTTPS listener](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-https-listener.html)

---

## 🔍 Amazon Macie

### What You Need to Know

- **Discovers sensitive data** in S3 (PII, PHI, financial)
- **Machine learning** for data classification
- **Findings** in Security Hub
- **Managed data identifiers**: 100+ predefined
- **Custom data identifiers**: Regex patterns

📖 **Documentation**: [Macie User Guide](https://docs.aws.amazon.com/macie/latest/user/)

---

## 🔒 AWS Private Certificate Authority (Skill 5.3.5)

### What You Need to Know

- **Managed private CA** for issuing and managing private certificates
- Supports **X.509 certificates** for internal services, IoT devices, and code signing
- **Certificate hierarchies**: Root CA and subordinate CAs
- **Short-lived certificates**: Mode for cost-effective ephemeral certificates
- Integrates with **ACM** for automatic deployment to AWS services
- **Cross-Region**: Can issue certificates for resources in any region

### Use Cases

- Internal TLS between microservices
- Mutual TLS (mTLS) authentication
- Code signing certificates
- IoT device certificates
- **IAM Roles Anywhere** trust anchors

📖 **Documentation**: [AWS Private CA User Guide](https://docs.aws.amazon.com/privateca/latest/userguide/)

---

## 📦 S3 Encryption Options

| Type | Key Storage | Key Management |
|------|-------------|----------------|
| **SSE-S3** | AWS | AWS |
| **SSE-KMS** | AWS KMS | You control policy |
| **SSE-C** | Customer-provided | Customer |
| **Client-side** | Customer | Customer |

## 🔐 Client-side encryption library family (high-frequency exam topic)

AWS ships multiple client-side encryption libraries, each tailored to a specific storage target. They all do envelope encryption + KMS-backed key wrapping + signing, but differ in **structural awareness** of the data they protect.

| Library | Tailored for | Key trait |
|---|---|---|
| **AWS Encryption SDK** | Arbitrary application data (strings, files, byte streams) | Produces opaque ciphertext blob; no schema awareness |
| **AWS Database Encryption SDK for DynamoDB** *(formerly DynamoDB Encryption Client, renamed 2023)* | DynamoDB items | **Attribute-aware** — encrypts individual attributes; leaves primary key plaintext so the table remains queryable |
| **Amazon S3 Encryption Client** (in AWS SDKs) | S3 objects | Object-stream-aware; integrates with `s3:PutObject` / `s3:GetObject` flows |

### Why the DynamoDB Encryption Client (not Encryption SDK) for DynamoDB

If you used the general AWS Encryption SDK on a DynamoDB item, you'd encrypt the whole item as a blob and **lose**:
- Query by partition / sort key (now ciphertext)
- GSI / LSI queries
- Conditional updates (DynamoDB can't compare to ciphertext)
- Filtering in Scan/Query results

The DynamoDB Encryption Client:
- Encrypts **only the attributes you mark sensitive**, leaving primary key + selected attributes plaintext for querying
- **Signs each item** so tampering is detected per-item on read
- Adds two reserved attributes per item: `*amzn-ddb-map-sig*` (signature) and `*amzn-ddb-map-desc*` (material description)

### End-to-end protection + tamper detection — answer decoder

| Phrasing in question | Points to |
|---|---|
| "Protects from point of creation until storage" / "end-to-end" | **Client-side encryption** (in the app, before transit) |
| "Detect unauthorized modification" / "verify integrity" | **Cryptographic signing** (NOT Streams, NOT PITR) |
| "DynamoDB" + client-side + signing | **AWS Database Encryption SDK / DynamoDB Encryption Client** |
| Any storage + client-side + signing | **AWS Encryption SDK** |
| S3 + client-side encryption | **S3 Encryption Client** in the SDK |
| Server-side encryption only | SSE-S3 / SSE-KMS / DynamoDB SSE — does **not** satisfy "end-to-end" |

### PITR vs Streams vs Signing — disambiguate

| Feature | What it does | What it does NOT do |
|---|---|---|
| **DynamoDB PITR** | Continuous backup, restore to any second within last 35 days | Detect or prove tampering — only enables *recovery* |
| **DynamoDB Streams** | Change data capture — events for each modification | Provide cryptographic integrity; can be disabled with sufficient IAM perms |
| **Item signing** (Database Encryption SDK) | Cryptographic per-item signature; tamper-proof on read | Restore data; you *detect* tampering, not recover from it |
| **CloudTrail data events for DynamoDB** | Audit log of API calls (`GetItem`, `PutItem`, etc.) | Verify item-level cryptographic integrity directly |

Defense in depth combines them; but for "detect unauthorized changes" specifically → **signing**.

📖 **AWS Database Encryption SDK**: [Documentation](https://docs.aws.amazon.com/database-encryption-sdk/latest/devguide/what-is-database-encryption-sdk.html)  
📖 **AWS Encryption SDK**: [Documentation](https://docs.aws.amazon.com/encryption-sdk/latest/developer-guide/introduction.html)

### S3 Bucket Keys

- Reduces KMS requests for SSE-KMS
- Saves costs
- Reduces CloudTrail events

### S3 Cross-Region Replication with SSE-KMS — THREE permission layers (TRAP)

When you set up CRR between two buckets that are SSE-KMS encrypted (different keys in each region — KMS keys are regional, can't be shared cross-region), the replication role needs permissions at **three places**, not just one:

```
Source Bucket (eu-west-1, encrypted with SOURCE-KMS-KEY)
   │
   │ Replication Role (must have permissions on BOTH keys)
   ▼
Destination Bucket (eu-central-1, encrypted with DEST-KMS-KEY)
```

**Layer 1: Replication Role identity policy** needs:
- `kms:Decrypt` on Source-KMS-Key
- `kms:Encrypt` / `kms:GenerateDataKey` on Dest-KMS-Key
- `kms:ReEncryptFrom` and `kms:ReEncryptTo` (best practice)
- Standard S3 replication: `s3:GetObjectVersion`, `s3:GetObjectVersionTagging`, `s3:ReplicateObject`, `s3:ReplicateDelete`, `s3:ReplicateTags`

**Layer 2: Source KMS key policy** must allow:
- Replication Role principal to `kms:Decrypt`

**Layer 3: Destination KMS key policy** must allow:
- Replication Role principal to `kms:Encrypt` (and `kms:GenerateDataKey`)

**Symptom of incomplete config**: unencrypted objects replicate fine but SSE-KMS-encrypted objects fail. This means the role has S3 permissions but lacks KMS permissions at one or more layers.

**Reflex**: "CRR + SSE-KMS not working" → check all THREE permission layers (role identity policy + both key policies). Don't fall for distractors that only address S3 permissions.

📖 **Documentation**: [S3 Encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingEncryption.html)  
📖 **CRR with KMS**: [Replicating SSE-KMS encrypted objects](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-config-for-kms-objects.html)

---

## 🔐 Encryption In Transit — what's encrypted by default (TRAP)

Memorize these four defaults — they come up in multi-fact recall questions:

| Traffic path | Encrypted by default? | Notes |
|---|---|---|
| **Between AWS regions** (over AWS global network backbone) | ✅ **YES** | Automatic at physical layer for all inter-region traffic |
| **Between Availability Zones** (within a region) | ✅ **YES** | Automatic encryption between AZs |
| **Between EC2 instances** (intra-region/same VPC) | ⚠️ **DEPENDS on instance type** | Nitro-based instances: yes (since ~2021). Older instance types: no. |
| **Direct Connect** | ❌ **NO** | DX is a private connection but NOT encrypted. Use Site-to-Site VPN over DX or MACsec to encrypt. |
| **Site-to-Site VPN** | ✅ Yes | IPsec encryption |
| **Internet traffic to AWS endpoints** | ⚠️ Depends on app | TLS depends on the app/SDK using HTTPS endpoints |
| **PrivateLink / VPC endpoint traffic** | ❌ Not by default | Private path but not encrypted at network layer; rely on app-layer TLS |

**Common traps**:
- "All intra-region traffic is encrypted between instances **for all instance types**" — FALSE (Nitro only)
- "Direct Connect encrypts traffic by default" — FALSE (private ≠ encrypted)
- "VPC endpoints encrypt traffic" — FALSE (private path, but rely on TLS at app layer)
- "Inter-region traffic is unencrypted unless you set up VPN" — FALSE (auto-encrypted on AWS backbone)

**Reflex rule**: **Private path ≠ encrypted path.** Always check if encryption is explicit (TLS, IPsec, MACsec) or automatic.

---

## 🔐 Data Integrity & Immutability (Skill 5.2.2)

### S3 Object Lock

- **WORM (Write Once Read Many)** protection for S3 objects
- **Retention modes**:
  - **Governance mode**: Users with special permissions can override
  - **Compliance mode**: No one can delete or modify, including root account
- **Legal hold**: Independent hold on object (no expiration date)
- Requires **versioning** enabled on the bucket

### S3 Glacier Vault Lock

- **Immutable vault policy** for Glacier archives
- Once locked, policy **cannot be changed or deleted**
- Use for regulatory compliance (e.g., SEC Rule 17a-4)
- Time-based retention controls

### Digital Code Signing (AWS Signer)

- **AWS Signer**: Managed code signing for Lambda, IoT, container images
- Validates code integrity and publisher identity
- **Signing profiles** define cryptographic algorithms and validity periods
- **Code Signing for Lambda**: configured at the function level with `CodeSigningConfig`. `UntrustedArtifactOnDeployment` can be `WARN` (log) or `ENFORCE` (block).

#### Revocation APIs — know the blast radius

| API | Scope | Reversible? | When to use |
|---|---|---|---|
| `RevokeSignature` | One specific signature (by job ID) | **No** — permanent | Revoke a single bad artifact |
| `RevokeSigningProfile` | Entire profile + **ALL signatures ever created with it** | **No** — permanent | Profile compromise (use carefully) |
| `CancelSigningProfile` | Stops profile from creating *new* signatures; existing signatures remain valid | Different from revoke — safer | "Stop signing new things but keep existing valid" |

#### The "rotate, migrate, revoke" pattern (departing signer / compromised profile)

`RevokeSigningProfile` invalidates every signature from that profile — **immediately breaks every Lambda with `ENFORCE` code signing**. Correct sequence:

1. **Create a new signing profile** (without the departing signer's access)
2. **Update CI/CD pipelines** to use the new profile going forward
3. **Re-sign existing artifacts** with the new profile (so production keeps working)
4. **Then revoke the old profile** — now safe because nothing depends on it
5. **Defense in depth**: also remove the departing signer's IAM `signer:*` permissions (immediate, no production impact)

#### Common trap

- "Schedule revocation for [future date]" alone is wrong — `RevokeSigningProfile` retroactively invalidates ALL past signatures, not just future ones. Setting an effective date just delays the breakage.
- "RevokeSignature for past month signatures" is wrong granularity — doesn't prevent future signing access (only IAM removal does that).

📖 **S3 Object Lock**: [Object Lock Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)  
📖 **Glacier Vault Lock**: [Vault Lock Documentation](https://docs.aws.amazon.com/amazonglacier/latest/dev/vault-lock.html)  
📖 **AWS Signer**: [AWS Signer Documentation](https://docs.aws.amazon.com/signer/latest/developerguide/)

---

## 🔗 Data in Transit - Secure Access (Skill 5.1.2)

### AWS PrivateLink

- Access AWS services and third-party services **privately** (no internet gateway, NAT, VPN)
- Creates **interface VPC endpoints** with private IP addresses in your VPC
- Traffic stays on **AWS backbone network**
- Use for: accessing SaaS services, cross-account service access, compliance-sensitive workloads

📖 **Documentation**: [AWS PrivateLink](https://docs.aws.amazon.com/vpc/latest/privatelink/)

### AWS Client VPN

- **Managed VPN service** for remote access to AWS and on-premises resources
- **OpenVPN-based** client
- Supports **Active Directory** and **SAML-based** authentication
- **Mutual TLS** certificate authentication
- Authorization rules control access to specific networks
- Split-tunnel or full-tunnel modes

📖 **Documentation**: [Client VPN Admin Guide](https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/)

### ELB Security Policies (Skill 5.1.1)

- **Predefined security policies** define TLS versions and cipher suites
- **ALB/NLB**: Choose policy to control minimum TLS version (e.g., TLS 1.2, TLS 1.3)
- **Forward secrecy**: Use ECDHE-based cipher suites
- Enforce HTTPS-only with HTTP-to-HTTPS redirect rules

📖 **Documentation**: [ELB Security Policies](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-https-listener.html)

---

## 🔒 Inter-Resource Encryption in Transit (Skill 5.1.3)

### Nitro System Encryption

- **Automatic encryption** of traffic between Nitro-based EC2 instances
- No application changes required
- Encrypted at the **hardware level** (Nitro Card)
- Applies to traffic within the same VPC and across peered VPCs

### Service-Specific Encryption in Transit

| Service | Encryption Feature |
|---------|-------------------|
| **Amazon EMR** | In-transit encryption with TLS for inter-node communication |
| **Amazon EKS** | Envelope encryption for Kubernetes secrets, pod-to-pod encryption |
| **SageMaker AI** | Inter-container encryption for training jobs |
| **Amazon RDS** | SSL/TLS for client connections, encrypted replication |

📖 **Nitro Encryption**: [Encryption in Transit on EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/data-protection.html)

---

## 📂 Lifecycle Management & Retention (Skill 5.2.3)

### S3 Lifecycle Policies

- **Transition actions**: Move objects between storage classes (Standard -> IA -> Glacier)
- **Expiration actions**: Delete objects after specified days
- Can apply to specific prefixes or tags
- Use with Object Lock for combined retention + lifecycle management

### Amazon EFS Lifecycle Policies

- Transition files between **Standard** and **Infrequent Access (IA)** storage classes
- Based on **last access time** (e.g., move to IA after 30 days)
- Reduces storage costs automatically

### Amazon FSx for Lustre Backup Policies

- **Automatic daily backups** with configurable retention (0-90 days)
- **Manual backups** for point-in-time recovery
- Backups stored in S3 with encryption

📖 **S3 Lifecycle**: [Managing S3 Lifecycle](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)  
📖 **EFS Lifecycle**: [EFS Lifecycle Management](https://docs.aws.amazon.com/efs/latest/ug/lifecycle-management-efs.html)

---

## 💾 Backup & Replication Security (Skill 5.2.4)

### AWS Backup

- **Centralized backup** for EC2, EBS, RDS, DynamoDB, EFS, FSx, S3, and more
- **Backup policies**: Define schedules, retention, lifecycle rules
- **Backup Vault Lock**: WORM protection for backups (compliance mode)
- **Cross-account backup**: Copy backups to another account for ransomware protection
- **Cross-Region backup**: Copy backups to another region for disaster recovery
- **Backup Audit Manager**: Audit backup compliance

#### Schedule expressions — rate vs cron (cross-service concept)

Backup plans (and EventBridge, Lambda schedules, DLM, Step Functions, Glue, State Manager, etc.) accept two schedule expression formats:

| Type | Format | Use for |
|---|---|---|
| **Rate** | `rate(value unit)` — e.g. `rate(12 hours)`, `rate(7 days)` | Fixed periodic intervals counted from creation time. "Every N units." |
| **Cron** | `cron(min hr day-of-month month day-of-week year)` — **6 fields**, UTC | Calendar-specific times. "On these specific days/times." |

**AWS cron specifics**:
- **6 fields** (Unix cron has 5) — AWS adds a `year` field
- All times are **UTC**
- Cannot specify both `day-of-month` and `day-of-week` — use `?` in whichever you don't care about
- `*` = every, `,` = list, `-` = range, `/` = increment

**Decoder**:

| Requirement | Expression |
|---|---|
| Every 12 hours | `rate(12 hours)` |
| Every 30 days | `rate(30 days)` |
| Daily at 3 AM UTC | `cron(0 3 * * ? *)` |
| Sundays at midnight | `cron(0 0 ? * SUN *)` |
| 1 AM on the 10th and 20th of every month | `cron(0 1 10,20 * ? *)` |
| First day of every month at 6 AM | `cron(0 6 1 * ? *)` |
| Every weekday at 9 AM | `cron(0 9 ? * MON-FRI *)` |

**Common trap**: question requires backups "on the 10th and 20th" — rate expressions cannot target specific calendar dates. Must use cron.

#### When NOT to use AWS Backup for DynamoDB

- **Continuous restore** within last 35 days → use **Point-in-Time Recovery (PITR)** instead. PITR is *not* a substitute for scheduled backups with long retention — it gives you any-second restore within 35 days but doesn't satisfy "retain monthly backups for 4 months" requirements.
- **Data transfer between storage systems** → use **AWS DataSync**, not Backup. DataSync is for moving data (NFS/SMB/HDFS ↔ S3/EFS/FSx), not creating backup plans.

### Amazon Data Lifecycle Manager

- Automate **EBS snapshot** and **AMI** creation, retention, and deletion
- **Lifecycle policies**: Schedule-based or event-based
- **Cross-account copy**: Share snapshots with other accounts
- Supports **fast snapshot restore** for critical volumes

### AWS DataSync

- **Secure data transfer** between on-premises storage and AWS (S3, EFS, FSx)
- **In-transit encryption** (TLS) automatic
- **At-rest encryption** at destination
- **Data integrity verification** with checksums
- Bandwidth throttling and scheduling controls

📖 **AWS Backup**: [AWS Backup Developer Guide](https://docs.aws.amazon.com/aws-backup/latest/devguide/)  
📖 **Backup Vault Lock**: [Backup Vault Lock](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html)  
📖 **Data Lifecycle Manager**: [DLM User Guide](https://docs.aws.amazon.com/dlm/latest/APIReference/Welcome.html)  
📖 **DataSync**: [DataSync User Guide](https://docs.aws.amazon.com/datasync/latest/userguide/)

---

## 🔑 Imported Key Material & External Key Stores (Skills 5.3.2, 5.3.3)

### Imported Key Material

- Bring your **own key material** into KMS (BYOK)
- **You control** the key material lifecycle (can set expiration date)
- Key material can be **deleted and re-imported**
- **Limitations**: No automatic rotation, no export, manual rotation only
- If you delete imported key material, the KMS key becomes **unusable** until re-import

### AWS vs Imported Key Material Comparison

| Feature | AWS Generated | Imported |
|---------|--------------|----------|
| **Rotation** | Automatic (annual) | Manual only |
| **Availability** | AWS guarantees durability | You must maintain backup |
| **Expiration** | No expiration | Optional expiration date |
| **Deletion** | 7-30 day waiting period | Immediate (key material only) |

### External Key Store (XKS)

- Use KMS keys backed by keys **outside AWS** (your own HSM or key manager)
- **XKS proxy**: Component you manage that connects KMS to your external key manager
- Meets requirements for keys that must never leave your control
- Higher latency due to external key operations

📖 **Imported Keys**: [Importing Key Material](https://docs.aws.amazon.com/kms/latest/developerguide/importing-keys.html)  
📖 **External Key Store**: [External Key Store](https://docs.aws.amazon.com/kms/latest/developerguide/keystore-external.html)

---

## 🛡️ Sensitive Data Masking (Skill 5.3.4)

### CloudWatch Logs Data Protection Policies

- **Automatically detect and mask** sensitive data in log events
- Supports PII patterns: email, SSN, credit card numbers, etc.
- Uses **managed data identifiers** or custom patterns
- Masked data shown as `[REDACTED]` in log output
- Can send unmasked data to a specific audit destination

### Amazon SNS Message Data Protection

- **Audit, mask, or block** sensitive data in SNS messages
- Apply data protection policies to SNS topics
- Detects PII in message payloads using data identifiers
- Actions: **Audit** (log findings), **De-identify** (mask), **Deny** (block message)

📖 **CloudWatch Data Protection**: [Protect Sensitive Log Data](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/mask-sensitive-log-data.html)  
📖 **SNS Data Protection**: [SNS Message Data Protection](https://docs.aws.amazon.com/sns/latest/dg/sns-message-data-protection.html)

---

## 📺 Recommended Videos

- [AWS re:Invent 2022: Protecting secrets, keys, and data (SEC403)](https://www.youtube.com/watch?v=9vr3oMODIUE)
- [AWS re:Invent 2022: Amazon S3 security best practices (STG301)](https://www.youtube.com/watch?v=VeE-O0imUVY)
- [AWS re:Inforce 2019: Achieving Security Goals with CloudHSM (SDD333)](https://www.youtube.com/watch?v=_gezaWmwzYY)

---

## ✅ Exam Tips

- Know KMS key types and when to use each
- Understand envelope encryption
- Know difference between KMS and CloudHSM
- Understand Secrets Manager rotation Lambda steps
- Know S3 encryption options and their characteristics
- Understand ACM certificate types and renewal
- Know **AWS PrivateLink** and **VPC endpoints** for private access to services (Skill 5.1.2)
- Understand **S3 Object Lock** (Governance vs Compliance mode) and **Glacier Vault Lock** for immutability (Skill 5.2.2)
- Know **AWS Backup Vault Lock** for ransomware-resilient backups (Skill 5.2.4)
- Understand **imported key material** vs AWS-generated key material limitations (Skill 5.3.2-5.3.3)
- Know **External Key Store (XKS)** for keys that must remain outside AWS (Skill 5.3.2)
- Understand **CloudWatch Logs data protection** and **SNS message data protection** for masking PII (Skill 5.3.4)
- Know **AWS Private Certificate Authority** for private TLS certificates (Skill 5.3.5)
- Understand **Nitro encryption** for automatic inter-instance encryption (Skill 5.1.3)
- Know **Data Lifecycle Manager** for automated EBS snapshot management (Skill 5.2.4)
- Understand **ELB security policies** for enforcing TLS versions (Skill 5.1.1)
- Know **Client VPN** authentication options: AD, SAML, mutual TLS (Skill 5.1.2)


---

## Practice Quizzes

| Topic | Questions | Quiz |
|-------|-----------|------|
| **AWS KMS** | 25 | [View](../quizzes/kms-comprehensive.html) |
| **Secrets Manager and CloudHSM** | 20 | [View](../quizzes/secrets-manager-cloudhsm-comprehensive.html) |
| **ACM and S3 Encryption** | 18 | [View](../quizzes/acm-s3-encryption-comprehensive.html) |
| **Amazon Macie** | 18 | [View](../quizzes/macie-comprehensive.html) |
| **AWS PrivateLink and VPC Endpoints** | 25 | [View](../quizzes/privatelink-comprehensive.html) |
| **S3 Object Lock and Glacier Vault Lock** | 25 | [View](../quizzes/s3objectlock-glaciervaultlock-comprehensive.html) |
| **AWS Backup and Data Lifecycle Manager** | 25 | [View](../quizzes/awsbackup-vaultlock-dlm-comprehensive.html) |
| **KMS Imported Keys and External Key Store** | 25 | [View](../quizzes/kms-imported-keys-xks-comprehensive.html) |
| **CloudWatch Logs and SNS Data Protection** | 25 | [View](../quizzes/cwlogs-sns-dataprotection-comprehensive.html) |
| **AWS Private Certificate Authority** | 25 | [View](../quizzes/privateca-comprehensive.html) |
| **Nitro / EMR / EKS Encryption** | 25 | [View](../quizzes/nitro-emr-eks-encryption-comprehensive.html) |

> 11 quizzes | 256 questions total for Domain 5

---

*Last updated: February 2026 (SCS-C03)*
