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