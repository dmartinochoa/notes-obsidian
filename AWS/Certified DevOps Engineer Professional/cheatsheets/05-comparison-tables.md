# AWS DevOps Professional - Comparison Tables Cheatsheet

## Secrets Storage: Parameter Store vs Secrets Manager

| Feature | SSM Parameter Store | Secrets Manager |
|---------|---------------------|-----------------|
| **Cost** | Free (Standard), $0.05/10K calls (Advanced) | $0.40/secret/month + API calls |
| **Auto Rotation** | No (manual via Lambda) | Yes (built-in for RDS/Redshift/DocumentDB) |
| **Cross-Account** | Yes (via RAM or resource policy) | Yes (via resource policy) |
| **Max Size** | 4 KB (Standard), 8 KB (Advanced) | 64 KB |
| **Versioning** | Yes | Yes |
| **Encryption** | Optional (SecureString) | Always encrypted |
| **Hierarchy** | Yes (/app/prod/db/password) | No |
| **Use Case** | Config values, non-rotating secrets | Database credentials, API keys needing rotation |

**Exam Tip:** If question mentions "rotation" → Secrets Manager. If mentions "hierarchy" or "cost-effective" → Parameter Store.

---

## ECS Launch Types: EC2 vs Fargate

| Feature | EC2 Launch Type | Fargate |
|---------|-----------------|---------|
| **Infrastructure** | You manage EC2 instances | AWS manages infrastructure |
| **Pricing** | Pay for EC2 instances | Pay for vCPU and memory |
| **Scaling** | Must scale EC2 + tasks | Only scale tasks |
| **GPU Support** | Yes | No |
| **Persistent Storage** | EBS volumes | EFS only |
| **SSH Access** | Yes | No (use ECS Exec) |
| **Placement Strategies** | Yes (spread, binpack) | No |
| **Spot Instances** | Yes | Yes (Fargate Spot) |
| **Windows Containers** | Yes | Yes (limited) |

**Exam Tip:** Fargate = serverless, no instance management. EC2 = more control, GPU, specific placement needs.

---

## Kinesis Data Streams vs Kinesis Firehose

| Feature | Data Streams | Firehose |
|---------|--------------|----------|
| **Latency** | Real-time (~200ms) | Near real-time (60s buffer min) |
| **Scaling** | Manual (resharding) | Automatic |
| **Data Retention** | 24 hours - 365 days | No retention (delivery only) |
| **Consumers** | Multiple (Lambda, KCL, etc.) | Single destination |
| **Destinations** | Custom consumers | S3, Redshift, OpenSearch, Splunk, HTTP |
| **Data Transformation** | No (do in consumer) | Yes (Lambda) |
| **Replay** | Yes | No |
| **Pricing** | Per shard hour + PUT payload | Per GB processed |

**Exam Tip:** Need replay or multiple consumers → Data Streams. Need easy delivery to S3/Redshift → Firehose.

---

## SQS Standard vs FIFO

| Feature | Standard | FIFO |
|---------|----------|------|
| **Throughput** | Unlimited | 3,000 msg/sec (with batching) |
| **Ordering** | Best-effort | Strict FIFO |
| **Duplicates** | Possible | Exactly-once processing |
| **Message Groups** | No | Yes (parallel processing) |
| **Queue Name** | Any | Must end with .fifo |
| **Deduplication** | No | Content-based or ID-based |

**Exam Tip:** Need ordering or exactly-once → FIFO. High throughput, order doesn't matter → Standard.

---

## ALB vs NLB

| Feature | Application LB | Network LB |
|---------|----------------|------------|
| **Layer** | 7 (HTTP/HTTPS) | 4 (TCP/UDP/TLS) |
| **Performance** | Millions RPS | Millions RPS, ultra-low latency |
| **Static IP** | No (use Global Accelerator) | Yes |
| **Preserve Source IP** | X-Forwarded-For header | Yes (native) |
| **SSL Termination** | Yes | Yes (TLS) |
| **Path Routing** | Yes | No |
| **Host Routing** | Yes | No |
| **WebSockets** | Yes | Yes |
| **Health Checks** | HTTP/HTTPS | TCP, HTTP, HTTPS |
| **Sticky Sessions** | Yes | Yes (newer feature) |
| **PrivateLink** | No | Yes |

**Exam Tip:** HTTP routing rules → ALB. Static IP, extreme performance, PrivateLink → NLB.

---

## CloudWatch Agent vs X-Ray

| Feature | CloudWatch Agent | X-Ray |
|---------|------------------|-------|
| **Purpose** | Metrics and logs collection | Distributed tracing |
| **Data Type** | System metrics, custom metrics, logs | Traces, segments, annotations |
| **Visualization** | Dashboards, Log Insights | Service map, trace details |
| **Use Case** | Server monitoring, log analysis | Debug distributed apps, find bottlenecks |
| **Integration** | EC2, on-premises | Lambda, ECS, Beanstalk, API GW |

**Exam Tip:** Understand application flow/latency → X-Ray. Monitor system resources → CloudWatch.

---

## CloudTrail vs Config vs CloudWatch

| Feature | CloudTrail | Config | CloudWatch |
|---------|------------|--------|------------|
| **Purpose** | API activity logging | Resource compliance | Metrics & logs |
| **Question Answered** | Who did what when? | Is resource compliant? | How is it performing? |
| **Data Type** | API events | Configuration snapshots | Metrics, logs, events |
| **Remediation** | No | Yes (auto-remediation) | Alarms trigger actions |
| **Retention** | 90 days (console), S3 unlimited | Up to 7 years | 15 months (metrics) |

**Exam Tip:** "Who changed?" → CloudTrail. "Is it compliant?" → Config. "Is it healthy?" → CloudWatch.

---

## Step Functions: Standard vs Express

| Feature | Standard | Express |
|---------|----------|---------|
| **Max Duration** | 1 year | 5 minutes |
| **Execution Model** | Exactly-once | At-least-once (async), At-most-once (sync) |
| **Pricing** | Per state transition | Per execution, duration, memory |
| **Execution History** | Yes (console) | CloudWatch Logs only |
| **Use Case** | Long-running workflows, human approval | High-volume, short duration |
| **Max Executions** | 2,000/sec | 100,000/sec |

**Exam Tip:** Long running, exactly-once → Standard. High volume, short duration → Express.

---

## EventBridge vs SNS

| Feature | EventBridge | SNS |
|---------|-------------|-----|
| **Event Filtering** | Content-based (any JSON field) | Message attributes only |
| **Sources** | AWS services, SaaS, custom | Custom publishers |
| **Schema Registry** | Yes | No |
| **Archive & Replay** | Yes | No |
| **Targets** | 20+ AWS services | Lambda, SQS, HTTP, email, SMS |
| **Cross-Account** | Event bus sharing | Topic subscriptions |
| **Pricing** | Per event | Per message + delivery |

**Exam Tip:** Complex event routing, SaaS integration → EventBridge. Simple pub/sub, SMS/email → SNS.

---

## RDS vs Aurora

| Feature | RDS | Aurora |
|---------|-----|--------|
| **Storage** | EBS (up to 64 TB) | Distributed (up to 128 TB) |
| **Replication** | Async to read replicas | Sync within cluster (6 copies) |
| **Read Replicas** | Up to 15 (depends on engine) | Up to 15 |
| **Failover** | 60-120 seconds | ~30 seconds |
| **Global Database** | Cross-region read replicas | Yes (< 1 second replication) |
| **Backtrack** | No | Yes (MySQL only) |
| **Serverless** | No | Yes (v2) |
| **Performance** | Standard | Up to 5x MySQL, 3x PostgreSQL |
| **Cost** | Lower | ~20% more than RDS |

**Exam Tip:** High availability, fast failover, global → Aurora. Cost-sensitive, simpler needs → RDS.

---

## GuardDuty vs Inspector vs Macie

| Service | Purpose | Data Sources | Findings |
|---------|---------|--------------|----------|
| **GuardDuty** | Threat detection | VPC Flow Logs, DNS, CloudTrail | Compromised instances, crypto mining |
| **Inspector** | Vulnerability assessment | EC2, ECR, Lambda | CVEs, network exposure |
| **Macie** | Data discovery | S3 | PII, sensitive data |

**Exam Tip:** Suspicious activity → GuardDuty. Software vulnerabilities → Inspector. S3 sensitive data → Macie.

---

## Deployment Rollback Methods

| Service | Rollback Method |
|---------|-----------------|
| **CloudFormation** | Automatic on failure, update with previous template |
| **Elastic Beanstalk** | Redeploy previous version |
| **CodeDeploy (EC2)** | Automatic or manual redeploy of previous revision |
| **CodeDeploy (Lambda/ECS)** | Traffic shift back to previous version |
| **ECS (rolling)** | Deploy previous task definition |
| **Lambda** | Update alias to previous version |
| **API Gateway** | Redeploy to previous stage |

---

## Cross-Account Access Methods

| Method | Use Case |
|--------|----------|
| **IAM Roles (AssumeRole)** | Temporary access, most common |
| **Resource Policies** | S3, Lambda, SNS, SQS, KMS, Secrets Manager |
| **AWS RAM** | VPC subnets, Transit Gateway, License Manager |
| **Organizations SCPs** | Guardrails across accounts |
| **StackSets** | Deploy CFN across accounts |
| **EventBridge** | Cross-account event routing |
| **CodePipeline** | Cross-account deployments |

---

## Log Destinations by Service

| Service | Destinations |
|---------|--------------|
| **VPC Flow Logs** | CloudWatch Logs, S3 |
| **CloudTrail** | S3, CloudWatch Logs |
| **Route 53 Query Logs** | CloudWatch Logs only |
| **S3 Access Logs** | S3 only |
| **ELB Access Logs** | S3 only |
| **CloudFront Logs** | S3, Kinesis Firehose |
| **Lambda Logs** | CloudWatch Logs |
| **ECS Logs** | CloudWatch Logs, Firehose, Splunk |
| **API Gateway Logs** | CloudWatch Logs |

**Exam Tip:** Route 53 → CloudWatch Logs only. S3/ELB → S3 only. Most others → CloudWatch Logs.
