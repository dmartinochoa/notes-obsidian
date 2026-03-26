# AWS DevOps Professional - Deployment Strategies Cheatsheet

## Deployment Strategy Comparison

| Strategy | Downtime | Rollback Speed | Risk | Cost |
|----------|----------|----------------|------|------|
| **All-at-Once** | Yes | Redeploy | High | Low |
| **Rolling** | No | Redeploy | Medium | Low |
| **Rolling with Additional Batch** | No | Redeploy | Medium | Medium |
| **Immutable** | No | Fast (terminate) | Low | High |
| **Blue/Green** | No | Very Fast (swap) | Low | High |
| **Canary** | No | Fast | Very Low | Medium |
| **Linear** | No | Fast | Low | Medium |

---

## CodeDeploy Deployment Configurations

### EC2/On-Premises

| Config | Behavior |
|--------|----------|
| `AllAtOnce` | Deploy to all instances simultaneously |
| `HalfAtATime` | Deploy to 50% of instances at a time |
| `OneAtATime` | Deploy one instance at a time |
| `Custom` | Define minimum healthy hosts (number or %) |

### Lambda/ECS

| Config | Traffic Shift |
|--------|---------------|
| `AllAtOnce` | 100% immediately |
| `Canary10Percent5Minutes` | 10% for 5 min, then 100% |
| `Canary10Percent10Minutes` | 10% for 10 min, then 100% |
| `Canary10Percent15Minutes` | 10% for 15 min, then 100% |
| `Linear10PercentEvery1Minute` | +10% every 1 min |
| `Linear10PercentEvery2Minutes` | +10% every 2 min |
| `Linear10PercentEvery3Minutes` | +10% every 3 min |
| `Linear10PercentEvery10Minutes` | +10% every 10 min |

---

## CodeDeploy Lifecycle Hooks

### EC2/On-Premises (In-Place)

```
ApplicationStop
    |
DownloadBundle
    |
BeforeInstall
    |
Install
    |
AfterInstall
    |
ApplicationStart
    |
ValidateService
```

### EC2/On-Premises (Blue/Green)

```
BeforeBlockTraffic --> BlockTraffic --> AfterBlockTraffic
                                              |
ApplicationStop --> DownloadBundle --> BeforeInstall
                                              |
Install --> AfterInstall --> ApplicationStart
                                              |
ValidateService --> BeforeAllowTraffic --> AllowTraffic --> AfterAllowTraffic
```

### Lambda/ECS

```
BeforeAllowTraffic --> AllowTraffic --> AfterAllowTraffic
```

---

## Elastic Beanstalk Deployment Policies

| Policy | Description | Use Case |
|--------|-------------|----------|
| **All at once** | Deploy to all instances simultaneously | Dev/test, fast deployment |
| **Rolling** | Deploy in batches | Prod, minimize capacity reduction |
| **Rolling with additional batch** | Launch new batch first, then rolling | Prod, maintain full capacity |
| **Immutable** | Deploy to new ASG, swap when healthy | Prod, quick rollback needed |
| **Traffic splitting** | Canary testing with new ASG | Prod, gradual traffic shift |

### Beanstalk Configuration

```yaml
# .ebextensions/deployment.config
option_settings:
  aws:elasticbeanstalk:command:
    DeploymentPolicy: Rolling
    BatchSizeType: Percentage
    BatchSize: 25
```

---

## API Gateway Deployment

### Stage Variables
- Use for different configurations per stage (dev/staging/prod)
- Reference Lambda aliases: `${stageVariables.lambdaAlias}`

### Canary Deployments
```
Stage --> Canary (% traffic) --> Promote/Rollback
```

- Split traffic between current and canary deployment
- Configure canary percentage (0.0 to 1.0)
- Promote canary to full deployment or rollback

---

## Lambda Deployment with Aliases

### Traffic Shifting

```
Alias (prod)
    |
    +--> Version 1 (90%)
    |
    +--> Version 2 (10%)
```

### SAM Gradual Deployment

```yaml
# template.yaml
Globals:
  Function:
    AutoPublishAlias: live
    DeploymentPreference:
      Type: Canary10Percent5Minutes
      Alarms:
        - !Ref AliasErrorMetricGreaterThanZeroAlarm
      Hooks:
        PreTraffic: !Ref PreTrafficLambdaFunction
        PostTraffic: !Ref PostTrafficLambdaFunction
```

---

## ECS Deployment Strategies

### Rolling Update (Default)
```
minimumHealthyPercent: 50
maximumPercent: 200
```

### Blue/Green with CodeDeploy
- Uses target groups
- Traffic shifting: AllAtOnce, Canary, Linear
- Automatic rollback on alarms

### External Controller
- Third-party deployment controller
- Full control over deployment process

---

## CloudFormation Deployment

### Stack Update Behaviors

| Behavior | Description |
|----------|-------------|
| **Update with no interruption** | No downtime |
| **Update with some interruption** | Brief interruption |
| **Replacement** | New resource created, old deleted |

### Change Sets
1. Create change set
2. Review changes
3. Execute or delete

### StackSets Deployment Order
```yaml
OperationPreferences:
  RegionOrder:
    - us-east-1
    - us-west-2
    - eu-west-1
  FailureTolerancePercentage: 10
  MaxConcurrentPercentage: 50
```

---

## Blue/Green Patterns by Service

| Service | Blue/Green Method |
|---------|-------------------|
| **EC2 + ASG** | Route 53 weighted/swap ASGs behind ELB |
| **Elastic Beanstalk** | Swap environment URLs |
| **ECS** | CodeDeploy with target group swap |
| **Lambda** | Alias traffic shifting |
| **API Gateway** | Canary deployments |
| **RDS** | Blue/Green Deployments (managed) |

---

## Rollback Strategies

| Service | Rollback Method |
|---------|-----------------|
| **CodeDeploy EC2** | Redeploy previous revision or use rollback |
| **CodeDeploy Lambda/ECS** | Automatic on CloudWatch alarm |
| **Elastic Beanstalk** | Redeploy previous version |
| **CloudFormation** | Automatic on failure, manual rollback |
| **ECS Rolling** | Deploy previous task definition |
| **Lambda** | Update alias to previous version |

### CodeDeploy Auto Rollback

```yaml
# In deployment group settings
autoRollbackConfiguration:
  enabled: true
  events:
    - DEPLOYMENT_FAILURE
    - DEPLOYMENT_STOP_ON_ALARM
    - DEPLOYMENT_STOP_ON_REQUEST
```

---

## Key Exam Tips

1. **Minimize downtime** = Blue/Green or Immutable
2. **Quick rollback** = Blue/Green (just swap back)
3. **Cost-conscious** = Rolling deployment
4. **Zero capacity reduction** = Rolling with additional batch
5. **Gradual traffic shift** = Canary or Linear
6. **Lambda safe deployments** = Use aliases + traffic shifting
7. **ECS blue/green** = Requires CodeDeploy + 2 target groups
8. **CloudFormation rollback** = Automatic on stack failure
