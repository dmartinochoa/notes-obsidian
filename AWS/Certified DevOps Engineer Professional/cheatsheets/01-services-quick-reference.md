# AWS DevOps Professional - Services Quick Reference

## CI/CD Services

| Service | Purpose | Key Points |
|---------|---------|------------|
| **CodeCommit** | Git repository | Encryption at rest (KMS), IAM auth, triggers to SNS/Lambda |
| **CodeBuild** | Build & test | buildspec.yml, scales automatically, pay per build minute |
| **CodeDeploy** | Deployment automation | appspec.yml, EC2/Lambda/ECS, hooks for lifecycle events |
| **CodePipeline** | CI/CD orchestration | Stages, actions, artifacts in S3, manual approval |
| **CodeArtifact** | Artifact repository | npm/pip/maven, upstream repos, cross-account sharing |
| **CodeGuru** | Code review & profiling | ML-based recommendations, detects expensive code |

## Infrastructure as Code

| Service | Purpose | Key Points |
|---------|---------|------------|
| **CloudFormation** | IaC declarative | Stacks, StackSets, drift detection, change sets |
| **CDK** | IaC programmatic | TypeScript/Python/Java, synthesizes to CFN |
| **SAM** | Serverless IaC | Extends CFN, sam deploy, local testing |
| **Service Catalog** | Governed products | Portfolios, products, launch constraints |
| **Elastic Beanstalk** | PaaS | .ebextensions, managed updates, deployment policies |

## Configuration Management

| Service | Purpose | Key Points |
|---------|---------|------------|
| **SSM Parameter Store** | Config storage | Standard (free), Advanced (paid), SecureString with KMS |
| **SSM Session Manager** | Shell access | No SSH/bastion, IAM auth, CloudTrail logging |
| **SSM Run Command** | Remote execution | Rate control, output to S3/CW, cross-account |
| **SSM Patch Manager** | OS patching | Patch baselines, maintenance windows |
| **SSM State Manager** | Desired state | Associations, compliance reporting |
| **SSM Automation** | Runbooks | Approval workflows, cross-account/region |
| **AppConfig** | App configuration | Deployment strategies, validators, feature flags |
| **OpsWorks** | Chef/Puppet | Stacks, layers, recipes, lifecycle events |

## Compute & Containers

| Service | Purpose | Key Points |
|---------|---------|------------|
| **EC2 Auto Scaling** | Capacity management | Launch templates, scaling policies, lifecycle hooks |
| **ECS** | Container orchestration | EC2/Fargate, task definitions, service auto scaling |
| **EKS** | Kubernetes | Managed control plane, node groups, Fargate profiles |
| **Lambda** | Serverless compute | 15 min timeout, 10GB memory, versions/aliases |
| **Elastic Beanstalk** | App platform | Worker/Web, Docker support, blue/green |

## Monitoring & Logging

| Service | Purpose | Key Points |
|---------|---------|------------|
| **CloudWatch Metrics** | Metrics collection | Custom metrics (PutMetricData), 15 months retention |
| **CloudWatch Logs** | Log aggregation | Log groups, metric filters, subscriptions |
| **CloudWatch Alarms** | Alerting | Composite alarms, EC2 actions, SNS integration |
| **CloudWatch Synthetics** | Canary testing | Node.js/Python, API and UI monitoring |
| **X-Ray** | Distributed tracing | Segments, subsegments, service map, sampling |
| **CloudTrail** | API auditing | Management/data events, org trail, log file integrity |
| **EventBridge** | Event routing | Rules, targets, event buses, cross-account |

## Security & Compliance

| Service | Purpose | Key Points |
|---------|---------|------------|
| **AWS Config** | Resource compliance | Rules, conformance packs, auto-remediation |
| **GuardDuty** | Threat detection | VPC Flow/DNS/CloudTrail analysis, findings |
| **Inspector** | Vulnerability scanning | EC2/ECR/Lambda, CVE database |
| **Macie** | Data discovery | S3 PII/sensitive data, findings |
| **Security Hub** | Security posture | Aggregates findings, standards (CIS, PCI) |
| **WAF** | Web firewall | Rules, web ACLs, rate limiting, managed rules |
| **Secrets Manager** | Secrets rotation | Auto-rotation for RDS/Redshift/DocumentDB |
| **KMS** | Key management | CMKs, key policies, envelope encryption |
| **IAM Identity Center** | SSO | Permission sets, multi-account access |

## Networking & Content Delivery

| Service | Purpose | Key Points |
|---------|---------|------------|
| **Route 53** | DNS | Routing policies, health checks, failover |
| **CloudFront** | CDN | Origins, behaviors, OAC, Lambda@Edge |
| **ELB (ALB/NLB)** | Load balancing | Target groups, health checks, sticky sessions |
| **API Gateway** | API management | REST/HTTP/WebSocket, stages, throttling |

## Data & Analytics

| Service | Purpose | Key Points |
|---------|---------|------------|
| **Kinesis Data Streams** | Real-time streaming | Shards, retention 1-365 days, enhanced fan-out |
| **Kinesis Firehose** | Data delivery | Near real-time, transforms, S3/Redshift/OpenSearch |
| **DynamoDB** | NoSQL database | On-demand/provisioned, streams, global tables |
| **OpenSearch** | Search & analytics | Log analytics, dashboards, cross-cluster |
| **Athena** | S3 querying | Serverless SQL, Glue catalog integration |

## Disaster Recovery

| Service                         | Purpose            | Key Points                                      |
| ------------------------------- | ------------------ | ----------------------------------------------- |
| **AWS Backup**                  | Centralized backup | Backup plans, vaults, cross-region/account      |
| **DMS**                         | Database migration | Homogeneous/heterogeneous, CDC replication      |
| **S3 Cross-Region Replication** | Data replication   | Versioning required, RTC option                 |
| **RDS Read Replicas**           | Read scaling       | Cross-region, promote to standalone             |
| **Aurora Global Database**      | Global DR          | 1 primary, 5 secondary regions, <1s replication |

## 