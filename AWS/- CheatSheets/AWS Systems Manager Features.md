
---

### 1. Systems Manager Parameter Store

Secure storage for configuration data and secrets:

- Stores strings, string lists, and **SecureStrings** (KMS-encrypted)
- Used to store DB passwords, API keys, config values
- Versioned — can roll back to previous values
- Free tier available; advanced parameters cost extra
- Alternative to Secrets Manager (which adds automatic rotation)

---

### 2. Systems Manager AppConfig

Manages **application configuration** separately from code:

- Deploy config changes without redeploying the app
- Supports **feature flags**, throttle rates, allow/deny lists
- Has built-in validation and **safe deployment strategies** (gradual rollout, rollback on errors)
- Useful for toggling features in Lambda, ECS, EC2

---

### 3. Systems Manager Automation

Runs **predefined or custom runbooks** against AWS resources:

- Automate common ops tasks (patch AMIs, restart instances, remediate findings)
- Can be triggered by EventBridge, alarms, or manually
- Supports approval steps for sensitive actions
- Think of it as scripted ops workflows without needing Lambda

---

### 4. Systems Manager Run Command

Executes commands **on EC2 instances or on-prem servers** without SSH:

- No need to open port 22 or RDP
- Runs shell scripts, PowerShell, or predefined documents
- Logs output to CloudWatch or S3
- Useful for bulk operations across many instances at once

---

### 5. Systems Manager Change Manager

Formal **change request and approval workflow**:

- Requires approval before making changes to infrastructure
- Integrates with Automation runbooks to execute approved changes
- Tracks who approved, who ran, and what changed
- Designed for teams needing audit trails and change control (SOC2, ISO compliance)

---

### 6. Systems Manager Patch Manager

Automates **OS and application patching** across instances:

- Defines patch baselines (which patches are approved/rejected)
- Schedules patch runs via **Maintenance Windows**
- Works on EC2 and on-prem, Windows and Linux
- Reports compliance status to CloudWatch and Config

---

### 7. Systems Manager Inventory

Collects **metadata about your instances and software**:

- Gathers installed applications, OS details, network config, running services, Windows roles
- Data stored in S3 and queryable via **Resource Data Sync**
- Can be visualized in **AWS Config** or queried with **Athena**
- Useful for software license auditing and compliance reporting
- Runs on a schedule via SSM Agent — no SSH needed

---

### 8. Systems Manager Session Manager

**Browser or CLI-based shell access** to instances without SSH/RDP:

- No need to open inbound ports or manage bastion hosts
- Sessions logged to S3 or CloudWatch for full audit trail
- Access controlled via IAM policies
- Works on EC2 and on-prem, Windows and Linux
- Replaces the need for a bastion host entirely

---

### 9. Systems Manager OpsCenter

Centralized place to **view and resolve operational issues (OpsItems)**:

- Aggregates findings from CloudWatch Alarms, Config, GuardDuty, and more
- Each OpsItem has context, related resources, and recommended runbooks
- Can trigger Automation runbooks directly from an OpsItem
- Think of it as a lightweight ops ticketing system within AWS

---

### 10. Systems Manager Maintenance Windows

Defines **scheduled time windows** for running tasks:

- Used by Patch Manager but can run any Automation, Run Command, or Lambda
- Supports cron and rate expressions
- Prevents changes from running at bad times (e.g., business hours)
- Can be scoped to specific instance groups via tags

---

### 11. Systems Manager Distributor

**Packages and deploys software** to managed instances:

- Create your own software packages or use AWS-provided ones
- Deploys via Run Command or on a schedule
- Supports versioning — can pin instances to specific package versions
- Useful for deploying agents (CloudWatch Agent, security tools, etc.)

---

### 12. Systems Manager Compliance

Aggregates **patch and configuration compliance** across your fleet:

- Shows which instances are compliant/non-compliant with patch baselines
- Tracks custom compliance rules (e.g., required software installed)
- Data can be synced to S3 and queried with Athena
- Feeds into Security Hub for a unified compliance view