
**78.67% — solid score!** And looking at the breakdown:

**Strong areas:**

- **Resilient Cloud Solutions** — 100% 🎯
- **Security and Compliance** — 100% 🎯
- **SDLC Automation** — 77.78%
- **Configuration Management and IaC** — 70.83%

**Areas to focus on:**

- **Incident and Event Response** — 66.67%
- **Monitoring and Logging** — 63.64%

The two weaker areas make sense given the questions we worked through — some of the trickier ones involved:

- CloudWatch metrics vs EventBridge event detection
- Correct lifecycle hooks for CodeDeploy
- CloudTrail vs Config vs GuardDuty for specific monitoring scenarios

**For Monitoring and Logging, review:**

- CloudWatch Agent for custom metrics
- Subscription filters vs CloudWatch Alarms
- CloudTrail vs Config vs EventBridge use cases

**For Incident and Event Response, review:**

- CodeDeploy lifecycle hooks order
- Auto Scaling event patterns
- Trusted Advisor notification patterns

**Monitoring and Logging (63.64%)**

- CloudWatch Agent vs default EC2 metrics — disk/memory require the agent
- Subscription filters vs CloudWatch Alarms vs Logs Insights — when to use each
- CloudTrail vs Config vs EventBridge — audit vs compliance vs event-driven
- Personal Health Dashboard vs CloudWatch vs GuardDuty for different alert types
- Trusted Advisor notification patterns — built-in, EventBridge+SNS, Lambda+SNS
- CloudWatch Logs Insights vs Athena vs OpenSearch — when each is appropriate

---

**Incident and Event Response (66.67%)**

- CodeDeploy lifecycle hook order — AfterInstall, AllowTestTraffic, AfterAllowTestTraffic, BeforeAllowTraffic, AllowTraffic, AfterAllowTraffic
- BeforeAllowTraffic vs AfterAllowTestTraffic — static checks vs traffic validation
- Blue/green deployment — BlueInstanceTerminationOption and terminationWaitTimeInMinutes
- Auto Scaling event patterns — EventBridge vs lifecycle hooks vs scheduled actions
- EventBridge vs CloudWatch Alarms vs CloudTrail for event detection
- CodeDeploy deployment content options — Retain, Overwrite, Fail

---

**Configuration Management and IaC (70.83%)**

- CloudFormation cross-stack references vs Parameter Store — same account/region vs cross-account/region
- CloudFormation custom resources — Lambda-backed for dynamic values like AMI IDs
- CloudFormation deployment strategies — nested stacks vs stack sets vs cross-stack references
- SSM Parameter Store vs Secrets Manager — config vs secrets, rotation requirement
- Fn::ImportValue limitations — same account and region only
- CloudFormation triggers — configuration changes vs periodic for Config rules

---

**SDLC Automation (77.78%)**

- CodePipeline branch strategies — certain branch vs master branch
- CodeDeploy tag group logic — single group OR logic vs multiple groups AND logic
- CodeDeploy deployment content — Retain vs Overwrite vs Fail the deployment
- AFT feature flags — aft_feature_enterprise_support for Enterprise Support
- AppSpec.yaml lifecycle hooks for ECS vs EC2 vs Lambda deployments
- CodeBuild environment variables — never hardcode, always use Secrets Manager
- Manual approval stages — TEST auto-deploy vs PROD manual approval pattern

---

**Security and Compliance (100% — maintain knowledge)**

- SCP inheritance — parent restrictions flow down, child can only restrict further
- SCP allow list vs deny list — FullAWSAccess removal vs explicit deny
- ABAC in IAM Identity Center — user attributes, attribute passing, PrincipalTag conditions
- GuardDuty vs Inspector vs Macie vs Config — know exactly what each does
- KMS key rotation — Secrets Manager vs manual rotation with Config custom rules
- VPC endpoints — Gateway vs Interface, egress-only IGW for IPv6
- SAML SSO — Golden SAML attack, signing cert storage in Secrets Manager
- RAM vs VPC Peering vs PrivateLink vs Transit Gateway — when to use each

---

**Resilient Cloud Solutions (100% — maintain knowledge)**

- Blue/green vs rolling vs immutable deployments — when each is appropriate
- Session persistence — ElastiCache for Redis as centralized session store
- RDS decoupled from Elastic Beanstalk — never couple database to app stack lifecycle
- Instance store limitations — no built-in recovery support, ephemeral data
- DynamoDB LSI vs GSI — LSI strongly consistent but must be created at table creation
- Multi-AZ vs Read Replicas — availability vs performance