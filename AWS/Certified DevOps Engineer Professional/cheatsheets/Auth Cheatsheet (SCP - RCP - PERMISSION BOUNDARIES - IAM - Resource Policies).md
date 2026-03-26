
![[Pasted image 20260326203335.png]]
![[Pasted image 20260326203415.png]]

---

## SCP & IAM
![[Pasted image 20260326201114.png]]
SCPs are your **organizational backstop**. Even if a vendor or CI/CD pipeline role has an overly permissive IAM policy (which happens constantly in supply chains), an SCP can cap what's actually executable — blocking region exfiltration, preventing CloudTrail from being disabled, or restricting service usage to an approved list.

The two critical "does not apply" gaps to watch out for:

**The management account exemption** is the most dangerous — SCPs never apply there, so if a supply chain compromise reaches the management account, there is no SCP-level protection. This is why isolating the management account with near-zero workloads is a best practice.

**Service-linked roles** are another gap — AWS creates and manages these automatically for services like Config, GuardDuty, and Security Hub. SCPs won't block them, so you can't inadvertently break your own security tooling, but equally you can't restrict them either.

---
### RCP

RCPs work like SCPs but from the resource side rather than the principal side. The mental model is:

- SCP says: "principals in this account cannot do X regardless of their IAM policy"
- RCP says: "resources in this account cannot be accessed in way X regardless of what the resource policy says"

RCPs are attached in AWS Organizations — to the root, an OU, or a specific account — exactly like SCPs. Every resource in accounts under that attachment point is subject to the RCP.

An RCP sets the maximum permissions that any resource-based policy within scope can effectively grant. If a bucket policy tries to grant something the RCP denies, the RCP wins — the bucket policy is overridden silently.

A common example — restricting all S3 buckets in your org to only be accessible by principals within your org:

---
## Permission Boundary

![[Pasted image 20260326201613.png]]
If an S3 bucket policy directly grants access to a vendor role, the permission boundary on that role does not block it — resource-based policies are evaluated separately. SCPs do cover this scenario (from the calling account side), but boundaries don't.

The other critical one: **principals without a boundary have no boundary restriction**. SCPs are unconditional — every principal in scope gets them automatically. Boundaries are opt-in per principal, so if your CI/CD pipeline creates a new role and doesn't attach a boundary, that role operates with just its IAM policy and the SCP ceiling — no middle layer of control. This is a common gap in pipeline security postures.

In practice the two are complementary — SCPs for org-wide non-negotiables, boundaries for delegated admin scenarios where you want to allow a team to manage their own IAM without being able to escalate beyond a defined limit.

|Scenario|SCP|IAM|Boundary|
|---|---|---|---|
|Service using your role|Yes|Yes|Yes|
|Service-linked role|No|AWS-managed|No|
|Internal AWS service operation|No|No|No|
|Resource-based policy grant to service|Calling account SCP applies|Must also allow|No|

---
## Resource-based policies

Resource-based policies are attached directly to the resource itself rather than to an identity. Not all AWS services support them:
**Services that support resource-based policies**

|Service|What the policy controls|
|---|---|
|S3|Bucket and object access|
|KMS|Who can use and manage a key|
|Lambda|Who can invoke a function|
|SQS|Who can send/receive from a queue|
|SNS|Who can publish/subscribe to a topic|
|ECR|Who can pull/push container images|
|Secrets Manager|Who can access a secret|
|API Gateway|Who can call an API|
|EventBridge|Who can put events to a bus|
|IAM roles|Trust policy — who can assume the role|