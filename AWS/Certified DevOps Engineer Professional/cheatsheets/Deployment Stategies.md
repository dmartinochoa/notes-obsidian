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

## Complete Comparison

|Strategy|Lambda|ECS|EC2|
|---|---|---|---|
|**Canary**|✅|✅|✅|
|**Linear**|✅|✅|✅|
|**All-at-once**|✅|✅|✅|
|**Rolling**|❌|✅ (native)|✅|
|**Blue/Green**|✅ (via aliases)|✅|✅|
|**In-place**|❌|❌|✅|

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

## Canary vs Linear vs All-at-once

```
Canary:
→ Quick initial validation (small % first)
→ Then full cutover
→ Two steps only
→ Best for: want fast deployment with safety check

Linear:
→ Gradual consistent rollout
→ Equal increments over time
→ Many steps
→ Best for: want to monitor metrics during rollout

All-at-once:
→ Immediate full deployment
→ No gradual shift
→ One step
→ Best for: dev/test environments, fastest deployment
→ Highest risk in production
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