
> **Note:** Amazon EventBridge is the evolution of CloudWatch Events. They share the same underlying API — existing CloudWatch Events rules appear in EventBridge automatically. AWS recommends EventBridge for all new projects.

|Feature|CloudWatch Events|EventBridge|
|---|---|---|
|**Current Status**|Legacy (no new features)|✅ Actively developed, recommended|
|**Relationship**|Original service|Superset of CloudWatch Events|
|**Console Access**|CloudWatch console|Dedicated EventBridge console|
|**API Compatibility**|Full|✅ Fully backward-compatible|
|**Event Buses**|Single default bus only|✅ Default + custom + partner buses|
|**Custom Event Buses**|❌ No|✅ Yes — isolate domains or teams|
|**SaaS / Partner Integrations**|❌ No|✅ Yes — Datadog, Zendesk, Shopify, etc.|
|**Cross-Account Events**|⚠️ Limited|✅ Full cross-account event routing|
|**Cross-Region Events**|❌ No|✅ Yes — via event bus targets|
|**Schema Registry**|❌ No|✅ Yes — auto-discover & version event schemas|
|**Schema Discovery**|❌ No|✅ Yes — infer schemas from live traffic|
|**Event Replay**|❌ No|✅ Yes — replay archived events|
|**Event Archiving**|❌ No|✅ Yes — configurable retention|
|**Pipes (point-to-point)**|❌ No|✅ Yes — filter, enrich, route between sources & targets|
|**Input Transformation**|✅ Basic|✅ Advanced (with Pipes)|
|**Event Pattern Matching**|✅ Yes|✅ Yes (same syntax, more options)|
|**Scheduling (cron/rate)**|✅ Yes|✅ Yes + EventBridge Scheduler (separate, more powerful)|
|**EventBridge Scheduler**|❌ No|✅ Yes — millions of scheduled tasks, per-target timezone|
|**Targets Supported**|~20 AWS services|✅ 20+ AWS services + HTTP endpoints|
|**HTTP / Webhook Targets**|❌ No|✅ Yes — API destinations (OAuth, API key auth)|
|**Dead Letter Queues (DLQ)**|✅ Yes|✅ Yes|
|**Retry Policies**|✅ Yes|✅ Yes|
|**IAM Resource Policies**|⚠️ Limited|✅ Full resource-based policies per bus|
|**Pricing Model**|Per event published (first 1M free/month)|Per event published (custom/partner buses chargeable)|
|**Global Endpoints**|❌ No|✅ Yes — active/active multi-region failover|
|**Best For**|Simple AWS service event reactions|Enterprise event-driven architectures, SaaS integrations, cross-account workflows|
![[Pasted image 20260408193458.png]]
![[Pasted image 20260408193510.png]]
---

## When You Need EventBridge

**Rule of thumb — EventBridge is needed when:**

**1. The source emits events but has no native target integration**

Most AWS services emit events to EventBridge automatically but cannot directly invoke Lambda, SQS etc.:

- AWS Health events → need EventBridge to route to Lambda
- Trusted Advisor checks → need EventBridge
- CodePipeline state changes → need EventBridge to route to Slack/Lambda
- EC2 instance state changes → need EventBridge
- CloudTrail API calls → need EventBridge to detect specific API calls
- Config compliance changes → need EventBridge for specific rule alerts
- GuardDuty findings → need EventBridge

**2. You need content-based filtering**

EventBridge can filter on specific field values within the event JSON. Direct integrations typically cannot filter — they send everything:

json

```json
// Only route FAILED CodePipeline events
{
  "source": ["aws.codepipeline"],
  "detail": { "state": ["FAILED"] }
}
```

Without EventBridge you would receive all pipeline events and Lambda would have to filter — more code, more cost.

**3. You need to fan-out to multiple targets**

S3 → Lambda directly = one Lambda function S3 → EventBridge → multiple rules → multiple targets = fan-out with filtering per target

**4. The target is not natively supported by the source**

CloudWatch Alarm cannot invoke Lambda directly — only SNS. If you need Lambda:

```
CloudWatch Alarm → SNS → Lambda  (native path)
or
CloudWatch Alarm → SNS → EventBridge → Lambda  (unnecessary extra hop)
```

**5. You need scheduled triggers**

EventBridge Scheduler/rules with cron or rate expressions to trigger any target on a schedule.

**6. Cross-account or cross-region event routing**

EventBridge supports event buses across accounts and regions. Direct integrations are single-account only.

---

## The Decision Framework

```
Does the source have a NATIVE direct integration with the target?
        │
        ├─ YES → Use it directly
        │         Examples: S3→Lambda, SQS→Lambda, SNS→Lambda
        │
        └─ NO → Does the source emit EventBridge events?
                  │
                  ├─ YES → Use EventBridge rule to route to target
                  │         Examples: Health→Lambda, CodePipeline→Lambda
                  │
                  └─ NO → Need a bridge (Lambda reads source, publishes to EventBridge)
                           or use polling pattern
```

---

## Common Confusions Resolved

|Scenario|Direct or EventBridge?|Why|
|---|---|---|
|S3 upload triggers Lambda|Direct — S3 event notification|Native integration exists|
|CodePipeline failure alerts Slack|EventBridge → Lambda|No native Slack integration, pipeline emits to EventBridge|
|CloudWatch Alarm triggers Lambda|Alarm → SNS → Lambda|Alarm cannot directly invoke Lambda|
|Config rule violation alerts team|Config → EventBridge → SNS|Direct Config→SNS streams ALL events, EventBridge filters per rule|
|Trusted Advisor alert|EventBridge → Lambda|Trusted Advisor publishes to EventBridge only|
|DynamoDB item change triggers Lambda|Direct — event source mapping|Native KCL-style integration|
|DynamoDB DeleteTable alerts team|CloudTrail → EventBridge → SNS|Streams only captures item events, not API calls|
|EC2 instance terminated triggers cleanup|EventBridge → Lambda|EC2 state changes emit to EventBridge|
|Schedule Lambda every hour|EventBridge scheduled rule|Purpose-built for scheduling|
|API call detected (specific API)|CloudTrail → EventBridge|"AWS API call via CloudTrail" event pattern|

---

## One-Line Summary

> Use **direct integration** when the source has a native connection to the target. Use **EventBridge** when you need filtering, the source only emits events without native targets, you need scheduling, or you need to fan-out to multiple filtered targets.

# EventBridge vs SNS vs SQS Comparison

> **Quick mental model:** SQS = pull-based queue for decoupling workers · SNS = push-based fan-out to many subscribers · EventBridge = content-based event routing with filtering, schemas, and SaaS integrations

| Feature                         | EventBridge                                                                 | SNS                                                            | SQS                                              |
| ------------------------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------ |
| **Core Pattern**                | Event routing & choreography                                                | Pub/sub fan-out                                                | Message queuing                                  |
| **Delivery Model**              | Push                                                                        | Push                                                           | ✅ Pull (consumer-driven)                         |
| **Primary Use Case**            | Route events between AWS services, SaaS, and custom apps based on content   | Broadcast notifications to multiple subscribers simultaneously | Decouple producers & consumers; buffer workloads |
| **Message Ordering**            | ❌ No                                                                        | ❌ No (FIFO topic available)                                    | ⚠️ FIFO queue only                               |
| **Message Retention**           | No storage (fire & forget)                                                  | No storage (fire & forget)                                     | ✅ Up to 14 days                                  |
| **Replay / Redelivery**         | ✅ Archive & replay                                                          | ❌ No                                                           | ✅ Visibility timeout + DLQ retry                 |
| **Filtering**                   | ✅ Rich content-based (pattern matching on any field)                        | ✅ Attribute-based (subscription filters)                       | ❌ No native filtering                            |
| **Fan-out to Multiple Targets** | ✅ Yes (multiple rules/targets)                                              | ✅ Yes (core feature)                                           | ❌ No (one consumer group)                        |
| **Max Targets per Event**       | 5 targets per rule                                                          | 12.5M subscriptions per topic                                  | 1 consumer (or competing consumers)              |
| **SaaS / Partner Integrations** | ✅ Yes (Datadog, Zendesk, etc.)                                              | ❌ No                                                           | ❌ No                                             |
| **HTTP / Webhook Targets**      | ✅ Yes (API destinations)                                                    | ✅ Yes (HTTPS endpoints)                                        | ❌ No                                             |
| **Cross-Account Routing**       | ✅ Yes                                                                       | ✅ Yes                                                          | ⚠️ Via resource policies                         |
| **Cross-Region Routing**        | ✅ Yes                                                                       | ⚠️ Limited                                                     | ❌ No                                             |
| **Schema Registry**             | ✅ Yes                                                                       | ❌ No                                                           | ❌ No                                             |
| **Dead Letter Queue**           | ✅ Yes                                                                       | ✅ Yes                                                          | ✅ Yes (native)                                   |
| **Exactly-Once Delivery**       | ❌ No                                                                        | ❌ No                                                           | ✅ FIFO queues only                               |
| **At-Least-Once Delivery**      | ✅ Yes                                                                       | ✅ Yes                                                          | ✅ Yes                                            |
| **Max Message Size**            | 256 KB                                                                      | 256 KB                                                         | 256 KB                                           |
| **Throughput**                  | Soft limits (adjustable)                                                    | Very high (near unlimited)                                     | ✅ Very high (unlimited standard)                 |
| **Latency**                     | Low (ms)                                                                    | Very low (ms)                                                  | Low–medium (polling interval)                    |
| **Scheduling**                  | ✅ Yes (cron/rate + EventBridge Scheduler)                                   | ❌ No                                                           | ❌ No                                             |
| **Long Polling**                | ❌ No                                                                        | ❌ No                                                           | ✅ Yes                                            |
| **Visibility Timeout**          | ❌ No                                                                        | ❌ No                                                           | ✅ Yes                                            |
| **Batch Processing**            | ❌ No                                                                        | ❌ No                                                           | ✅ Yes (up to 10 messages)                        |
| **Common Pairing**              | Triggers Lambda, Step Functions, SQS                                        | Fan-out into SQS queues                                        | Backed by Lambda or EC2 workers                  |
| **Pricing Model**               | Per event published                                                         | Per message + per HTTP delivery                                | Per request + data transfer                      |
| **Relative Cost**               | Medium                                                                      | Low                                                            | ✅ Lowest                                         |
| **Best For**                    | Event-driven architectures with complex routing, auditing, SaaS integration | Simple broadcast to many consumers                             | Reliable async task queues, load leveling        |

## Common Architecture Patterns

| Pattern                       | Services                                                |
| ----------------------------- | ------------------------------------------------------- |
| **Fan-out with buffering**    | SNS → multiple SQS queues                               |
| **Event routing + buffering** | EventBridge → SQS → Lambda workers                      |
| **Audit + reaction**          | EventBridge (log/archive) + SNS (alert ops team)        |
| **Cross-account pipeline**    | EventBridge (account A) → EventBridge (account B) → SQS |
| **SaaS trigger → workflow**   | EventBridge (partner event) → Step Functions            |