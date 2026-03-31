## Key Differences from CodeDeploy

```
CodeDeploy traffic shifting:
→ Automated gradual shift
→ Automatic rollback on failure
→ Hook-based validation
→ Manages deployment lifecycle
→ Purpose-built for deployments

Route 53 weighted routing:
→ Manual weight adjustment
→ Manual rollback
→ No deployment lifecycle management
→ DNS level (TTL considerations)
→ Cross-region capable

ALB weighted target groups:
→ Manual weight adjustment
→ Manual rollback
→ No deployment lifecycle management
→ Layer 7, instant
→ Single region only
```

---

## When to Use Each

```
CodeDeploy:
→ Automated deployment pipeline
→ Need automatic rollback
→ Hook validation required
→ Canary/Linear/All-at-once patterns
→ CI/CD integration

Route 53:
→ Cross-region traffic shifting
→ Migrating between different stacks
→ DNS-level blue/green
→ Multi-region deployments
→ Different ALBs or endpoints

ALB weighted target groups:
→ Same region traffic shifting
→ Fine-grained request routing
→ A/B testing at load balancer
→ No DNS changes needed
→ Instant traffic shift
```

---

## Complete Comparison

|Feature|Route 53|ALB|CodeDeploy|
|---|---|---|---|
|**Level**|DNS|Layer 7|Application|
|**Instant?**|❌ TTL delay|✅ Instant|✅ Instant|
|**Granularity**|Request level|Request level|Request level|
|**Rollback**|Manual|Manual|Automatic|
|**Monitoring**|Manual|Manual|CloudWatch Alarms|
|**Works with**|Any endpoint|EC2/ECS/Lambda|EC2/ECS/Lambda|
|**Cross-region**|✅ Yes|❌ No|❌ No|
|**Cross-account**|✅ Yes|❌ No|❌ No|

## Route 53 Traffic Shifting

### Weighted Routing Policy

```
Route 53 weighted routing:

Record 1: app-v1.example.com → weight 90
Record 2: app-v2.example.com → weight 10

Traffic distribution:
→ 90% to version 1
→ 10% to version 2

Adjust weights gradually:
90/10 → 70/30 → 50/50 → 0/100
```

### How It Works

```
DNS level traffic shifting:
→ Returns different IP/endpoint based on weight
→ Client resolves DNS
→ Gets routed to weighted endpoint

TTL consideration:
→ Lower TTL = faster traffic shift
→ Higher TTL = clients cached longer
→ DNS changes not instant due to TTL
```

### Use Cases

```
→ Blue/green deployments at DNS level
→ Gradual migration between regions
→ A/B testing
→ Canary releases across entire stacks
→ Migrating between ALBs
```

---

## ALB Traffic Shifting

### Weighted Target Groups

```
ALB supports weighted target groups:

Target Group 1 (v1): weight 90
Target Group 2 (v2): weight 10

ALB distributes:
→ 90% requests to TG1
→ 10% requests to TG2

Adjust weights:
90/10 → 70/30 → 0/100
```

### How It Works

```
Layer 7 traffic shifting:
→ Happens at ALB level
→ Below DNS level
→ More granular control
→ Instant (no TTL delay)
→ Same DNS endpoint used

Listener rule:
{
  "Actions": [
    {
      "Type": "forward",
      "ForwardConfig": {
        "TargetGroups": [
          {"TargetGroupArn": "TG1-ARN", "Weight": 90},
          {"TargetGroupArn": "TG2-ARN", "Weight": 10}
        ]
      }
    }
  ]
}
```

### Use Cases

```
→ Blue/green deployments
→ Canary releases
→ A/B testing
→ Gradual migration between EC2 fleets
→ Testing new instance types
```

## Elastic Beanstalk Traffic Shifting

Worth mentioning since it's related:

```
Elastic Beanstalk blue/green:
→ Uses Route 53 CNAME swap
→ Or ALB weighted target groups
→ "Swap Environment URLs" feature
→ Instant DNS swap
→ Manual process
→ Old environment kept as rollback option
```

---

## NLB Traffic Shifting

```
NLB weighted target groups:
→ Does NOT support weighted target groups ← exam trap
→ Unlike ALB
→ Cannot do traffic splitting at NLB level
→ Must use Route 53 or ALB instead
```