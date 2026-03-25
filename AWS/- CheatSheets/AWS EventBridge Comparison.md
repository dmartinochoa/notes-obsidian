
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