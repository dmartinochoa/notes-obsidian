
| Feature                   | EC2/On-Premises                    | Lambda        | ECS                                |
| ------------------------- | ---------------------------------- | ------------- | ---------------------------------- |
| Canary deployment config  | ❌ Not via CD, need ALB             | ✅ Yes - Alias | ✅ Yes LB and task sets             |
| Linear deployment config  | ❌ Not via CD, need ALB             | ✅ Yes - Alias | ✅ Yes LB and task sets             |
| All-at-once               | ✅ Yes (rolling %)                  | ✅ Yes         | ✅ Yes                              |
| In-place deployment       | ✅ Yes                              | ❌ No          | ❌ No                               |
| Blue/Green deployment     | ✅ EC2 only, not On-Premises        | ✅ Yes         | ✅ Yes                              |
| Rolling                   | ✅ Indirectly via min healthy hosts | ❌ No          | ❌ Not via CodeDeploy, Yes natively |
| AppSpec revision location | S3 or GitHub                       | S3 only       | S3 only                            |

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

## Lambda Deployment Strategies

### 1. Canary

```
Traffic shifts in TWO increments:

CodeDeployDefault.LambdaCanary10Percent5Minutes:
→ 10% traffic to new version for 5 minutes
→ Then 100% if no issues

CodeDeployDefault.LambdaCanary10Percent10Minutes:
→ 10% traffic for 10 minutes
→ Then 100%

CodeDeployDefault.LambdaCanary10Percent15Minutes
CodeDeployDefault.LambdaCanary10Percent30Minutes

Pattern: LambdaCanary{percent}Percent{minutes}Minutes
```

### 2. Linear

```
Traffic shifts in EQUAL increments at EQUAL intervals:

CodeDeployDefault.LambdaLinear10PercentEvery1Minute:
→ 10% more every 1 minute
→ Takes 10 minutes to reach 100%

CodeDeployDefault.LambdaLinear10PercentEvery2Minutes
CodeDeployDefault.LambdaLinear10PercentEvery3Minutes
CodeDeployDefault.LambdaLinear10PercentEvery10Minutes

Pattern: LambdaLinear{percent}PercentEvery{n}Minute(s)
```

### 3. All-at-once

```
CodeDeployDefault.LambdaAllAtOnce:
→ 100% traffic immediately
→ No gradual shift
→ Fastest but highest risk
```

---

## ECS Deployment Strategies

### 1. Canary

```
Traffic shifts in TWO increments:

CodeDeployDefault.ECSCanary10Percent5Minutes:
→ 10% traffic to new task set for 5 minutes
→ Then 100% if no issues

CodeDeployDefault.ECSCanary10Percent15Minutes:
→ 10% traffic for 15 minutes
→ Then 100%

Pattern: ECSCanary{percent}Percent{minutes}Minutes
```

### 2. Linear

```
Traffic shifts in EQUAL increments at EQUAL intervals:

CodeDeployDefault.ECSLinear10PercentEvery1Minutes:
→ 10% more every 1 minute
→ Takes 10 minutes to reach 100%

CodeDeployDefault.ECSLinear10PercentEvery3Minutes
CodeDeployDefault.ECSLinear10PercentEvery10Minutes

Pattern: ECSLinear{percent}PercentEvery{n}Minutes
```

### 3. All-at-once

```
CodeDeployDefault.ECSAllAtOnce:
→ 100% traffic immediately
→ No gradual shift
→ Fastest but highest risk
```

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
## Automatic Rollback

```
All strategies support automatic rollback:

Triggers:
→ BeforeAllowTraffic hook fails
→ AfterAllowTraffic hook fails
→ CloudWatch Alarm threshold breached

Rollback behavior:
→ Traffic shifts back to original version
→ Happens automatically
→ No manual intervention needed

Configure in CodeDeploy deployment group:
→ "Roll back when deployment fails"
→ "Roll back when alarm thresholds are met"
```

---

## CloudWatch Alarms Integration

```
Can monitor during traffic shift:

Lambda metrics:
→ Error rate
→ Duration
→ Throttles

ECS metrics:
→ CPU utilization
→ Memory utilization
→ Task health

If alarm triggers during deployment:
→ Automatic rollback
→ Traffic shifts back to original
→ Deployment marked failed
```

---

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


- Contributed to the development of a vulnerability scanner, implementing detectors for source code, infrastructure as code, version control systems, and CI/CD environments. Extended secret detection capabilities to identify exposed credentials in code.

- Manually reviewed packages flagged as potentially malicious across major package registries (npm, PyPI, Maven Central, Packagist), resulting in thousands of malicious packages being removed from their respective repositories. Authored technical write-ups documenting the design and behavior of relevant cases.

- Built internal tooling to automate regression testing of the SAST scanner and systematically benchmark its results against competing tools. Automated backend end-to-end testing.