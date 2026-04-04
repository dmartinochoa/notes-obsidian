## Complete Hook Comparison

|Hook|Lambda|ECS|EC2 In-Place|EC2 Blue/Green|
|---|---|---|---|---|
|`ApplicationStop`|❌|❌|✅ PREV|✅ PREV|
|`DownloadBundle`|❌|❌|automated ⚙️|automated ⚙️|
|`BeforeInstall`|❌|✅|✅|✅|
|`Install`|❌|❌|automated ⚙️|automated ⚙️|
|`AfterInstall`|❌|✅|✅|✅|
|`ApplicationStart`|❌|❌|✅|✅|
|`ValidateService`|❌|❌|✅|✅|
|`BeforeBlockTraffic`|❌|❌|❌|✅ PREV|
|`BlockTraffic`|❌|❌|❌|automated ⚙️|
|`AfterBlockTraffic`|❌|❌|❌|✅ PREV|
|`AllowTestTraffic`|❌|automated ⚙️|❌|❌|
|`AfterAllowTestTraffic`|❌|✅|❌|❌|
|`BeforeAllowTraffic`|✅|✅|❌|✅|
|`AllowTraffic`|automated ⚙️|automated ⚙️|❌|automated ⚙️|
|`AfterAllowTraffic`|✅|✅|❌|✅|
|`BeforeDeregister`|❌|✅|❌|❌|
|`Deregister`|❌|automated ⚙️|❌|❌|
|`AfterDeregister`|❌|✅|❌|❌|

**PREV = uses previous deployment's AppSpec**

---

## Full Lifecycle Orders

### Lambda:

```
BeforeAllowTraffic (hook ✅)
    ↓
AllowTraffic (automated ⚙️)
    ↓
AfterAllowTraffic (hook ✅)
```

### ECS:

```
BeforeInstall (hook ✅)
    ↓
Install (automated ⚙️)
    ↓
AfterInstall (hook ✅)
    ↓
AllowTestTraffic (automated ⚙️)
    ↓
AfterAllowTestTraffic (hook ✅)
    ↓
BeforeAllowTraffic (hook ✅)
    ↓
AllowTraffic (automated ⚙️)
    ↓
AfterAllowTraffic (hook ✅)
    ↓
BeforeDeregister (hook ✅)
    ↓
Deregister (automated ⚙️)
    ↓
AfterDeregister (hook ✅)
```

### EC2 In-Place:

```
ApplicationStop (hook ✅ PREV AppSpec)
    ↓
DownloadBundle (automated ⚙️)
    ↓
BeforeInstall (hook ✅)
    ↓
Install (automated ⚙️)
    ↓
AfterInstall (hook ✅)
    ↓
ApplicationStart (hook ✅)
    ↓
ValidateService (hook ✅)
```

### EC2 Blue/Green — Blue fleet (original instances):

```
BeforeBlockTraffic (hook ✅ PREV AppSpec)
    ↓
BlockTraffic (automated ⚙️)
    ↓
AfterBlockTraffic (hook ✅ PREV AppSpec)
```

### EC2 Blue/Green — Green fleet (replacement instances):

```
ApplicationStop (hook ✅ PREV AppSpec)
    ↓
DownloadBundle (automated ⚙️)
    ↓
BeforeInstall (hook ✅)
    ↓
Install (automated ⚙️)
    ↓
AfterInstall (hook ✅)
    ↓
ApplicationStart (hook ✅)
    ↓
ValidateService (hook ✅)
    ↓
BeforeAllowTraffic (hook ✅)
    ↓
AllowTraffic (automated ⚙️)
    ↓
AfterAllowTraffic (hook ✅)
```

---

## Previous AppSpec Hooks

```
These three always use PREVIOUS deployment's AppSpec:
→ ApplicationStop
→ BeforeBlockTraffic
→ AfterBlockTraffic

Why:
→ They interact with CURRENTLY RUNNING version
→ Must know how to stop/prepare old version
→ New AppSpec doesn't know about old version

Fix when previous scripts fail:
→ --ignore-application-stop-failures flag
→ Bypasses failures in these three hooks
→ Continues with rest of deployment
```

---

## Key Distinctions

```
Automated actions ⚙️:
→ Performed by CodeDeploy
→ Cannot attach Lambda/scripts
→ Cannot be customized

Hooks ✅:
→ Lambda functions for ECS/Lambda deployments
→ Shell scripts for EC2 deployments
→ Fully customizable
→ SUCCESS/FAILURE controls deployment flow

EC2 In-Place vs Blue/Green:
→ In-Place: no traffic management hooks
→ Blue/Green: has BlockTraffic + AllowTraffic hooks
→ Blue/Green: two separate lifecycles (blue + green fleets)

ECS unique hooks:
→ AllowTestTraffic + AfterAllowTestTraffic
→ BeforeDeregister + Deregister + AfterDeregister
→ Most complex lifecycle

Lambda unique:
→ Only three hooks total
→ Simplest lifecycle
→ No install or stop hooks
```

---

## Failure Troubleshooting

```
Failure + no logs:
→ ALB health check misconfiguration
→ Fails at AllowTraffic

Failure + logs at ApplicationStop:
→ Bug in PREVIOUS deployment's AppSpec
→ Use --ignore-application-stop-failures

Failure + logs at custom hook:
→ Bug in current deployment's hook script
→ Fix the script

Failure at AllowTestTraffic (ECS):
→ Test traffic validation failed
→ Check AfterAllowTestTraffic Lambda

Failure at BeforeAllowTraffic:
→ Pre-traffic validation failed
→ DB not ready, dependency missing etc
```


---

## EC2 Instance Lifecycle States

### Normal lifecycle:

```
Pending
    ↓
Running
    ↓
Stopping
    ↓
Stopped
    ↓
Terminated
```

---

## Auto Scaling Lifecycle States

Auto Scaling adds additional states around lifecycle hooks:

### Scale-Out (Launch) lifecycle:

```
Pending
    ↓
Pending:Wait ← lifecycle hook PAUSES here
    ↓          (you perform custom actions)
Pending:Proceed
    ↓
InService ← instance receives traffic
```

### Scale-In (Terminate) lifecycle:

```
InService
    ↓
Terminating
    ↓
Terminating:Wait ← lifecycle hook PAUSES here
    ↓               (you perform custom actions)
Terminating:Proceed
    ↓
Terminated
```

---

## All Auto Scaling States

```
Pending
→ Instance launching
→ Not yet receiving traffic

Pending:Wait
→ Launch lifecycle hook active
→ Instance paused
→ Waiting for custom action to complete
→ Or waiting for timeout

Pending:Proceed
→ Lifecycle hook completed
→ Instance moving to InService

InService
→ Fully launched and healthy
→ Receiving traffic from load balancer

Terminating
→ Instance marked for termination
→ Being deregistered from load balancer

Terminating:Wait
→ Termination lifecycle hook active
→ Instance paused before termination
→ Waiting for custom action
→ e.g. drain connections, backup logs

Terminating:Proceed
→ Lifecycle hook completed
→ Instance proceeding to termination

Terminated
→ Instance fully terminated
→ No longer exists

Detached
→ Instance removed from ASG
→ Still running
→ Not receiving ASG-managed traffic

Detaching
→ In process of being detached from ASG

EnteringStandby
→ Moving to standby mode

Standby
→ Instance in ASG but not receiving traffic
→ Used for maintenance
→ Still counts toward ASG capacity

Warmed:Pending
→ Instance in warm pool launching
→ Pre-warming for faster scale-out

Warmed:Pending:Wait
→ Warm pool lifecycle hook active

Warmed:Pending:Proceed
→ Warm pool lifecycle hook completed

Warmed:Running
→ Instance in warm pool running
→ Ready for quick promotion to InService

Warmed:Stopped
→ Instance in warm pool stopped
→ Saves cost while pre-warmed

Warmed:Hibernated
→ Instance in warm pool hibernated
→ Fastest resume time
```

---

## Lifecycle Hook Timeout

```
Default wait time: 1 hour

Options during wait:
1. Complete the hook → proceed immediately
2. Send heartbeat → extend timeout
3. Do nothing → timeout → default result

Default result if timeout:
→ CONTINUE (for launch hooks)
→ ABANDON (for termination hooks)

You can configure default result:
→ CONTINUE = proceed normally
→ ABANDON = terminate instance (launch)
           = skip remaining hooks (terminate)
```

---

## How to Control Lifecycle Hook

```
Complete hook (move forward):
aws autoscaling complete-lifecycle-action \
  --lifecycle-action-result CONTINUE \
  --instance-id i-xxxxxxxxx \
  --lifecycle-hook-name MyHook \
  --auto-scaling-group-name MyASG

Abandon (stop/terminate):
aws autoscaling complete-lifecycle-action \
  --lifecycle-action-result ABANDON \
  --instance-id i-xxxxxxxxx \
  --lifecycle-hook-name MyHook \
  --auto-scaling-group-name MyASG

Extend timeout (heartbeat):
aws autoscaling record-lifecycle-action-heartbeat \
  --instance-id i-xxxxxxxxx \
  --lifecycle-hook-name MyHook \
  --auto-scaling-group-name MyASG
```

---

## Warm Pool States

```
Warm pool = pre-initialized instances
waiting to be promoted to InService

Purpose:
→ Reduce scale-out latency
→ Instances pre-configured and ready
→ Much faster than cold launch

Warm pool states:
Warmed:Running → running, costs full price
Warmed:Stopped → stopped, costs less
Warmed:Hibernated → hibernated, fastest resume

When scale-out needed:
Warmed:Running/Stopped/Hibernated
    ↓
Warmed:Pending:Wait (lifecycle hook)
    ↓
Warmed:Pending:Proceed
    ↓
Pending:Wait (ASG lifecycle hook)
    ↓
InService
```

---

## Standby State

```
Used for:
→ Instance maintenance
→ Troubleshooting
→ Manual updates

Standby behavior:
→ Instance stays in ASG
→ Does NOT receive traffic
→ DOES count toward desired capacity
→ Load balancer deregisters it

Move to standby:
aws autoscaling enter-standby \
  --instance-ids i-xxxxxxxxx \
  --auto-scaling-group-name MyASG \
  --should-decrement-desired-capacity

Return from standby:
aws autoscaling exit-standby \
  --instance-ids i-xxxxxxxxx \
  --auto-scaling-group-name MyASG
```

---

## Detached State

```
Detached behavior:
→ Instance removed from ASG management
→ Still running independently
→ Not receiving ASG-managed traffic
→ Does NOT count toward ASG capacity
→ ASG launches replacement instance

Use cases:
→ Keep instance for debugging
→ Convert to standalone instance
→ Investigate issues without ASG interference
```

---

## Complete State Flow Diagram

```
                    WARM POOL
                    ─────────
                    Warmed:Pending
                    Warmed:Pending:Wait
                    Warmed:Pending:Proceed
                    Warmed:Running
                    Warmed:Stopped
                    Warmed:Hibernated
                         ↓
                    (promoted to ASG)

AUTO SCALING GROUP
──────────────────
Pending
    ↓
Pending:Wait ← launch hook
    ↓
Pending:Proceed
    ↓
InService ←→ Standby
    ↓          ↑↓
Detaching  EnteringStandby
    ↓
Detached
    
InService
    ↓
Terminating
    ↓
Terminating:Wait ← termination hook
    ↓
Terminating:Proceed
    ↓
Terminated
```

---

## Common Use Cases Per State

```
Pending:Wait (launch hook):
→ Install software
→ Configure application
→ Join to domain
→ Register with monitoring
→ Pull secrets from Parameter Store

Terminating:Wait (termination hook):
→ Drain connections
→ Backup logs to S3
→ Deregister from service discovery
→ Update DynamoDB instance registry
→ Send final metrics

Standby:
→ Manual patching
→ Troubleshooting
→ Taking AMI snapshot

Detached:
→ Forensic investigation
→ Keep instance alive after ASG replacement
```

---

## Exam Patterns

```
"Run custom action before instance receives traffic"
→ Launch lifecycle hook → Pending:Wait state

"Run custom action before instance terminated"
→ Termination lifecycle hook → Terminating:Wait state

"Remove instance from traffic without terminating"
→ Move to Standby state

"Update DynamoDB when instance launches"
→ Launch lifecycle hook → EventBridge → Lambda → DynamoDB

"Drain connections before termination"
→ Termination lifecycle hook → custom drain script

"Pre-initialize instances for fast scaling"
→ Warm pool

"Instance stuck in Pending:Wait"
→ Lifecycle hook not completed
→ Check Lambda/script responding
→ Or waiting for timeout
```