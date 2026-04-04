
---

## CodePipeline — Cross-Account Deploy

### CodePipeline → S3 (cross-account)
**Scope:** Cross-account, same region

**Required:**
- **KMS customer-managed key** in source account — grant usage to pipeline role AND destination account. Default encryption cannot be decrypted cross-account.
- **S3 input bucket versioning** must be enabled — CodePipeline hard requirement.
- **Cross-account IAM role** lives in the *destination* account. Source pipeline role gets `sts:AssumeRole` on it.
- For multi-region: use a **KMS multi-region key** with replicas in each target region.

> **Gotcha:** KMS key + cross-account role must both exist before the pipeline runs or the deploy action fails silently.

---

### CodePipeline → CloudFormation (cross-account)
**Scope:** Cross-account

**Three mandatory steps:**

1. **Account A:** KMS key grants usage to pipeline role + account B. S3 artifact bucket policy grants account B access.
2. **Account B:** Cross-account IAM role — account A pipeline role assumes it.
3. **Account B:** CloudFormation service role with permissions for resources being deployed. Account A CodePipeline config references all account B resources (role ARNs, region).

---

## IAM — Cross-Account Role Patterns

### Trust relationship direction
- Trust policy goes on the **role being assumed** (the destination account role).
- The **caller account** gets `sts:AssumeRole` permission to invoke it.
- Reversing this is a common exam trap — always ask "which account owns the role being assumed?"

### iam:PassRole requirement
- Any user/role handing a service role to AWS (CloudFormation, CodeBuild, etc.) needs **`iam:PassRole`** explicitly.
- Creating the role is not sufficient — PassRole is a separate, additional permission.
- Do not use `aws:SourceIp` conditions with PassRole — CloudFormation calls from its own IP, not the user's.

---

## EKS — Cross-Account Access

### CodeBuild (account A) → EKS cluster (account B)
**Scope:** Cross-account — 3 steps required

1. **Trust relationship** on account B's deployment IAM role, trusting account A. Use `sts:AssumeRole` — not `AssumeRoleWithSAML` (that is for SAML federation only).
2. **Grant** account B's deployment role the necessary EKS IAM permissions.
3. **Update the EKS aws-auth ConfigMap** to map the role to appropriate Kubernetes RBAC permissions.

> **Gotcha:** The aws-auth ConfigMap is the most commonly missed step. IAM allows the call at the AWS layer; Kubernetes still needs its own internal mapping or it will deny access regardless.

---

## CloudFormation — Cross-Account & Multi-Region

### StackSets
**Scope:** Cross-account, multi-region

- Deploy the same stack to many accounts/regions from a single management account.
- With service-managed permissions: auto-deploys when an account joins the org, auto-removes when it leaves.
- The StackSet itself is a regional resource — it is created in one region.
- Choosing concurrent deployment options controls how many accounts are targeted in parallel and failure thresholds.

### Change sets vs. stacks
**Scope:** Single account, single region

- Change sets: preview the impact of updates before applying — not for multi-region deployment.
- `create-stack` CLI only accepts a single `--region` flag, not multiple.
- `ContinueUpdateRollback` resolves the `UPDATE_ROLLBACK_FAILED` state — drift detection and StackSets cannot.

---
## Aurora — Endpoint Routing

### Aurora DB endpoint types
**Scope:** Multi-AZ, single region

| Endpoint | Routes to | Use for |
|---|---|---|
| Cluster (writer) | Current primary | Writes — supports automatic failover |
| Reader | All reader instances | Read traffic distribution |
| Instance | A specific instance | Avoid in production — does not failover |
| Custom | Defined subset of instances | Specialised workloads only |


---