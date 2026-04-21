# AWS DevOps Professional - Exam Tips & Patterns

## Key Exam Patterns

### "Minimize Downtime" Questions
- **Blue/Green deployment** - Fastest rollback, parallel environments
- **Immutable deployment** - New instances, terminate on failure
- **Route 53 weighted routing** - Gradual traffic shift
- **Multi-AZ** - Automatic failover
- **Read replicas promotion** - For DR scenarios

### "Automate Remediation" Questions
- **AWS Config + Lambda** - Custom remediation
- **AWS Config + SSM Automation** - Built-in remediation
- **EventBridge + Lambda** - Event-driven remediation
- **CloudWatch Alarms + Auto Scaling** - Scale-based remediation
- **GuardDuty + EventBridge + Lambda** - Security remediation

### "Cross-Account" Questions
- **IAM Roles with trust policy** - Most common pattern
- **Resource-based policies** - S3, Lambda, KMS, SNS, SQS
- **AWS Organizations + SCPs** - Governance and guardrails
- **StackSets** - Deploy infrastructure across accounts
- **EventBridge event bus** - Cross-account events

### "Compliance/Audit" Questions
- **AWS Config** - Resource compliance and history
- **CloudTrail** - API audit trail, log file integrity
- **AWS Config Conformance Packs** - Compliance frameworks
- **Security Hub** - Aggregated security findings
- **AWS Audit Manager** - Audit evidence collection

### "Cost Optimization" Questions
- **Spot Instances** - Fault-tolerant workloads
- **Reserved Instances/Savings Plans** - Predictable workloads
- **Auto Scaling** - Match capacity to demand
- **S3 Lifecycle policies** - Move to cheaper storage classes
- **Lambda Provisioned Concurrency** - Only for cold start sensitive apps

---

## Critical Service Relationships

### CodePipeline Sources
- CodeCommit, S3, ECR, GitHub, Bitbucket

### CodePipeline Actions
- Build: CodeBuild, Jenkins
- Test: CodeBuild, Device Farm, third-party
- Deploy: CodeDeploy, CloudFormation, ECS, Elastic Beanstalk, S3, Service Catalog
- Approval: Manual approval, SNS notification

### EventBridge Integration
```
Event Sources → EventBridge Rules → Targets
     ↓                                  ↓
AWS Services                      Lambda, SNS, SQS,
SaaS Apps                         Step Functions,
Custom Apps                       CodePipeline, SSM
```

### CloudWatch Logs Flow
```
Logs → Log Groups → Metric Filters → Alarms → SNS/Lambda/Auto Scaling
         ↓
    Subscriptions → Kinesis/Lambda/OpenSearch
         ↓
    Insights (queries)
         ↓
    S3 Export (batch)
```

---

## Common Troubleshooting Patterns

### CodeDeploy Failures
| Issue | Solution |
|-------|----------|
| AppSpec not found | Check file name (appspec.yml), location (root) |
| Lifecycle hook timeout | Increase timeout, fix script |
| IAM permissions | Check instance profile, service role |
| Agent not running | Install/start CodeDeploy agent |
| AllowTraffic fails | Check health checks, ELB config |

### CloudFormation Failures
| Issue | Solution |
|-------|----------|
| CREATE_FAILED | Check resource dependencies, IAM permissions |
| UPDATE_ROLLBACK_FAILED | Manual intervention, continue-update-rollback |
| DELETE_FAILED | Check for retained resources, dependencies |
| Drift detected | Update stack or accept drift |
| Circular dependency | Use DependsOn, reorder resources |

### Lambda Failures
| Issue | Solution |
|-------|----------|
| Timeout | Increase timeout, optimize code, check VPC config |
| Out of memory | Increase memory allocation |
| Cold start | Use Provisioned Concurrency, optimize package size |
| Permission denied | Check execution role, resource policies |
| Throttling | Request limit increase, implement retry |

---

## Infrastructure as Code Patterns

### CloudFormation Intrinsic Functions
```yaml
!Ref                    # Reference resource/parameter
!GetAtt                 # Get attribute of resource
!Sub                    # String substitution
!Join                   # Join strings
!If                     # Conditional
!Equals                 # Compare values
!ImportValue            # Import from another stack
!FindInMap              # Lookup in mappings
!Select                 # Select from list
!Split                  # Split string into list
!GetAZs                 # Get availability zones
!Cidr                   # Generate CIDR blocks
```

### CloudFormation Resource Attributes
```yaml
DependsOn               # Explicit dependency
Condition               # Conditional creation
DeletionPolicy          # Retain, Snapshot, Delete
UpdateReplacePolicy     # Retain, Snapshot, Delete
CreationPolicy          # Wait for signals
UpdatePolicy            # How to update (ASG)
```

### CloudFormation Helper Scripts
```bash
cfn-init      # Read metadata, install packages, create files
cfn-signal    # Signal success/failure to CloudFormation
cfn-get-metadata  # Retrieve metadata
cfn-hup       # Daemon to check for updates
```

---

## Security Patterns

### Encryption at Rest
| Service | Encryption |
|---------|------------|
| S3 | SSE-S3, SSE-KMS, SSE-C |
| EBS | KMS (default or CMK) |
| RDS | KMS |
| DynamoDB | AWS owned, AWS managed, CMK |
| EFS | KMS |
| Secrets Manager | KMS (mandatory) |
| Parameter Store | KMS (SecureString) |

### Encryption in Transit
- Use HTTPS endpoints
- TLS for RDS connections
- VPC endpoints for private connectivity
- ACM for SSL/TLS certificates

### Least Privilege Patterns
1. Start with minimal permissions
2. Use IAM Access Analyzer
3. Use CloudTrail to audit
4. Use SCPs for guardrails
5. Use permission boundaries for delegation

---

## High Availability Patterns

### Multi-AZ vs Multi-Region
| Pattern | Use Case |
|---------|----------|
| Multi-AZ | High availability within region |
| Multi-Region Active-Passive | Disaster recovery |
| Multi-Region Active-Active | Global applications |

### Database HA
| Database | HA Method |
|----------|-----------|
| RDS | Multi-AZ (sync standby) |
| Aurora | Multi-AZ (6 copies, auto-failover) |
| DynamoDB | Multi-AZ by default, Global Tables for multi-region |
| ElastiCache | Multi-AZ with automatic failover |

### Compute HA
| Service | HA Method |
|---------|-----------|
| EC2 | ASG across AZs |
| ECS | Service across AZs |
| Lambda | Multi-AZ by default |
| EKS | Nodes across AZs |

---

## Key Configuration Files

### buildspec.yml (CodeBuild)
```yaml
version: 0.2
env:
  variables:
    KEY: "value"
  parameter-store:
    SECRET: /path/to/param
phases:
  install:
    commands:
      - npm install
  build:
    commands:
      - npm run build
artifacts:
  files:
    - '**/*'
cache:
  paths:
    - 'node_modules/**/*'
```

### appspec.yml (CodeDeploy - EC2)
```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/html
hooks:
  BeforeInstall:
    - location: scripts/before.sh
      timeout: 300
  AfterInstall:
    - location: scripts/after.sh
  ApplicationStart:
    - location: scripts/start.sh
  ValidateService:
    - location: scripts/validate.sh
```

### appspec.yml (CodeDeploy - Lambda)
```yaml
version: 0.0
Resources:
  - MyFunction:
      Type: AWS::Lambda::Function
      Properties:
        Name: my-function
        Alias: live
        CurrentVersion: 1
        TargetVersion: 2
Hooks:
  - BeforeAllowTraffic: PreTrafficHook
  - AfterAllowTraffic: PostTrafficHook
```

### taskdef.json (ECS)
```json
{
  "family": "my-task",
  "containerDefinitions": [{
    "name": "my-container",
    "image": "my-image:latest",
    "memory": 512,
    "cpu": 256,
    "essential": true,
    "portMappings": [{
      "containerPort": 80,
      "hostPort": 80
    }]
  }]
}
```

---

## Quick Decision Tree

### Which deployment service?
```
Need CI/CD pipeline? → CodePipeline
Need to build/test? → CodeBuild
Need to deploy to EC2/Lambda/ECS? → CodeDeploy
Need infrastructure deployment? → CloudFormation/CDK/SAM
Need managed app platform? → Elastic Beanstalk
Need serverless deployment? → SAM
```

### Which monitoring approach?
```
Need metrics? → CloudWatch Metrics
Need logs? → CloudWatch Logs
Need tracing? → X-Ray
Need API auditing? → CloudTrail
Need compliance? → AWS Config
Need threat detection? → GuardDuty
```

### Which messaging service?
```
Need pub/sub? → SNS
Need queue? → SQS
Need event routing? → EventBridge
Need streaming? → Kinesis
Need workflow? → Step Functions
```

---

## Numbers to Memorize

| Item | Value |
|------|-------|
| Lambda max timeout | 15 min |
| API Gateway timeout | 29 sec |
| SQS max retention | 14 days |
| SQS visibility timeout max | 12 hours |
| Kinesis max retention | 365 days |
| CloudWatch metrics retention | 15 months |
| S3 object max size | 5 TB |
| DynamoDB item max size | 400 KB |
| CodeBuild max timeout | 8 hours |
| Step Functions Standard max | 1 year |
| Step Functions Express max | 5 min |
| CloudFormation resources per stack | 500 |
| ASG max instances per group | 500 (default) |
| Lambda concurrent executions | 1,000 (default) |
