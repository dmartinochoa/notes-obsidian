## CloudWatch's Role First

CloudWatch itself **doesn't collect logs** — it's a **receiver and analyzer**. The sources feed into it:

```
CloudTrail ──────────────────→ CloudWatch Logs → Metric Filters → Alarms
S3 Server Access Logs ───────→ S3 Bucket (not CW directly)
VPC Flow Logs ───────────────→ CloudWatch Logs or S3
Lambda/EC2/ECS ──────────────→ CloudWatch Logs (native)
```

---

## By Service Category

### Compute

|Event|Logged By|
|---|---|
|EC2 instance start/stop/terminate|CloudTrail|
|EC2 OS-level logs (auth, syslog)|CloudWatch Agent (installed on instance)|
|Lambda invocations, errors, duration|CloudWatch Logs (native)|
|ECS/EKS container stdout/stderr|CloudWatch Logs (via log driver)|
|Auto Scaling events|CloudTrail + CloudWatch Events|

---

### Networking

|Event|Logged By|
|---|---|
|VPC traffic (IP flow data)|VPC Flow Logs → CloudWatch or S3|
|DNS queries|Route 53 Resolver Query Logs → CloudWatch or S3|
|CloudFront access logs|S3 (not CloudWatch directly)|
|API Gateway requests|CloudWatch Logs (if enabled) or Access Logs to S3|
|Load balancer access logs|S3 (ALB/NLB — not CloudWatch)|
|WAF allow/block decisions|CloudWatch Logs or S3 or Kinesis Firehose|

---

### Security & Identity

|Event|Logged By|
|---|---|
|IAM role/user/policy changes|CloudTrail|
|Console sign-ins, MFA changes|CloudTrail|
|Assumed role sessions|CloudTrail|
|GuardDuty threat findings|GuardDuty → EventBridge → SNS/Lambda|
|Security Hub findings|Security Hub → EventBridge|
|AWS Config rule violations|AWS Config → SNS or EventBridge|

---

### Databases

|Event|Logged By|
|---|---|
|RDS instance create/modify/delete|CloudTrail|
|RDS slow queries, error logs|CloudWatch Logs (if enabled per engine)|
|DynamoDB table config changes|CloudTrail|
|DynamoDB read/write operations|CloudTrail Data Events (extra cost)|
|ElastiCache config changes|CloudTrail|

---

### Storage (Beyond S3)

|Event|Logged By|
|---|---|
|EBS volume create/delete/attach|CloudTrail|
|EFS mount/config changes|CloudTrail|
|Glacier vault changes|CloudTrail|
|S3 Glacier restore requests|CloudTrail|

---

## The Alternatives Summarized

|Tool|Best For|
|---|---|
|**CloudTrail**|Who did what to which AWS resource (API calls)|
|**CloudWatch Logs**|Application/service runtime logs, OS logs, Lambda output|
|**CloudWatch Metrics**|Numeric performance data (CPU, latency, error rates)|
|**VPC Flow Logs**|Network traffic patterns, IP-level visibility|
|**AWS Config**|Resource configuration state and compliance drift over time|
|**GuardDuty**|Threat detection (anomalous behavior, known bad IPs, etc.)|
|**Security Hub**|Aggregated findings across GuardDuty, Config, Inspector, etc.)|
|**EventBridge**|Event-driven reactions to state changes across all services|
|**S3 Server Access Logs**|HTTP-level object access detail|
|**CloudFront/ALB Access Logs**|HTTP request-level detail for web traffic|

---

## The Mental Model

```
"Who changed an AWS resource config?"  → CloudTrail
"What is my app/OS outputting?"        → CloudWatch Logs (via agent or native)
"What traffic hit my network?"         → VPC Flow Logs
"Is my infrastructure compliant?"      → AWS Config
"Am I under attack?"                   → GuardDuty → Security Hub
"React to an event automatically?"     → EventBridge
```