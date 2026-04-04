| Feature                   | EC2/On-Premises                    | Lambda        | ECS                                |
| ------------------------- | ---------------------------------- | ------------- | ---------------------------------- |
| Canary deployment config  | ❌ Not via CD, need ALB             | ✅ Yes - Alias | ✅ Yes LB and task sets             |
| Linear deployment config  | ❌ Not via CD, need ALB             | ✅ Yes - Alias | ✅ Yes LB and task sets             |
| All-at-once               | ✅ Yes (rolling %)                  | ✅ Yes         | ✅ Yes                              |
| In-place deployment       | ✅ Yes                              | ❌ No          | ❌ No                               |
| Blue/Green deployment     | ✅ EC2 only, not On-Premises        | ✅ Yes         | ✅ Yes                              |
| Rolling                   | ✅ Indirectly via min healthy hosts | ❌ No          | ❌ Not via CodeDeploy, Yes natively |
| AppSpec revision location | S3 or GitHub                       | S3 only       | S3 only                            |

---
**In-place** means you update the existing running thing. This only makes sense when there is a persistent server to update. Lambda and ECS don't have persistent servers you manage — AWS manages the underlying compute. So:
- EC2/On-Premises → **in-place makes sense** — there's a real server to update in place
- Lambda/ECS → **in-place doesn't exist** — there's no persistent server to update

**Canary/Linear traffic shifting** requires a load balancer or alias that can split traffic between two versions simultaneously. This needs to be a first-class platform feature:
- **Lambda** → aliases can split traffic between two function versions natively — canary/linear work perfectly
- **ECS** → ALB/NLB can split traffic between two task sets natively — canary/linear work perfectly
- **EC2** → no native traffic splitting between two versions of instances — you have to simulate it manually with two deployment groups and ALB traffic shifting

**Blue/Green** requires the ability to launch a parallel environment and shift traffic:
- **EC2** → can launch replacement instances and shift ALB traffic — supported
- **Lambda** → versions + aliases handle this natively — supported
- **ECS** → new task set + ALB — supported
- **On-Premises** → cannot programmatically provision new servers — **not supported**
---
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

## CodeDeploy Lifecycle Hooks - [[Lifecycle Hooks]]

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
## How Code Deploy Traffic Shifting Works

### Lambda:

```
Uses ALIASES to shift traffic:

Alias → 90% Version 1
      → 10% Version 2 (canary)

After validation:
Alias → 100% Version 2
```

### ECS:

```
Uses TWO TASK SETS:

Original task set → 90% traffic
Replacement task set → 10% traffic

After validation:
Original task set → 0%
Replacement task set → 100%
    ↓
Original task set deregistered
```

### EC2:

```
In-place:
→ Updates instances directly
→ No traffic shifting

Blue/Green:
→ Original fleet → blocked
→ New fleet → receives traffic
→ Original fleet terminated
```

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

## Exam Patterns

```
"Deploy to small subset first then full"
→ Canary deployment
→ LambdaCanary10Percent5Minutes
→ ECSCanary10Percent5Minutes

"Gradually roll out over time"
→ Linear deployment
→ LambdaLinear10PercentEvery1Minute
→ ECSLinear10PercentEvery1Minutes

"Deploy everything immediately"
→ All-at-once
→ LambdaAllAtOnce
→ ECSAllAtOnce

"Automatically rollback if errors increase"
→ CloudWatch Alarm + CodeDeploy rollback config

"Test with small traffic before production"
→ Canary

"Monitor metrics during gradual rollout"
→ Linear
```

---

## Key Facts

```
Lambda uses aliases for traffic shifting:
→ Alias points to two versions simultaneously
→ Max two versions per alias
→ Traffic split defined as weights

ECS uses task sets:
→ Two task sets running simultaneously
→ ALB routes traffic based on weights
→ Original deregistered after successful deployment

Both support:
→ Canary, Linear, All-at-once
→ CloudWatch Alarm rollback
→ Hook-based validation
→ Automatic rollback

EC2 unique:
→ In-place deployment
→ No traffic shifting for in-place
→ Blue/Green available but different mechanism
```
