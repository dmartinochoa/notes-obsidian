# AWS DevOps Professional - Limits and Quotas Cheatsheet

## Lambda

| Resource | Limit |
|----------|-------|
| Memory | 128 MB - 10,240 MB (10 GB) |
| Timeout | 15 minutes (900 seconds) |
| Environment variables | 4 KB total |
| Deployment package (zip) | 50 MB compressed, 250 MB uncompressed |
| Container image | 10 GB |
| /tmp storage | 512 MB - 10,240 MB |
| Concurrent executions | 1,000 (soft limit, can be increased) |
| Reserved concurrency | Account limit minus 100 |
| Burst concurrency | 500-3,000 (varies by region) |
| Invocation payload (sync) | 6 MB |
| Invocation payload (async) | 256 KB |
| Layers | 5 layers per function |
| Layer size | 250 MB unzipped total |

## API Gateway

| Resource | Limit |
|----------|-------|
| Regional APIs per account | 600 |
| Throttle (steady-state) | 10,000 RPS |
| Throttle (burst) | 5,000 requests |
| Payload size | 10 MB |
| Timeout | 29 seconds |
| WebSocket message size | 128 KB |
| WebSocket connection duration | 2 hours |
| Stages per API | 10 |
| Resources per API | 300 |
| Cache size | 0.5 GB - 237 GB |

## CloudFormation

| Resource | Limit |
|----------|-------|
| Stacks per account | 2,000 |
| Resources per stack | 500 |
| Outputs per stack | 200 |
| Parameters per stack | 200 |
| Mappings per stack | 200 |
| Template body size (inline) | 51,200 bytes |
| Template body size (S3) | 1 MB |
| Nested stacks | 200 |
| StackSets per account | 100 |
| Stack instances per StackSet | 2,000 |

## CodePipeline

| Resource | Limit |
|----------|-------|
| Pipelines per region | 1,000 |
| Stages per pipeline | 50 |
| Actions per stage | 50 |
| Artifacts per pipeline | 200 |
| Webhooks per region | 300 |
| Custom action types | 50 |

## CodeBuild

| Resource | Limit |
|----------|-------|
| Build timeout | 8 hours (480 minutes) |
| Concurrent builds | 60 (default) |
| Build projects per region | 5,000 |
| Compute types | small, medium, large, 2xlarge |
| Environment variables | 200 per project |
| VPC subnets | 16 per project |
| Tags per project | 50 |

## CodeDeploy

| Resource | Limit |
|----------|-------|
| Applications per account | 1,000 |
| Deployment groups per application | 1,000 |
| Instances per deployment | 500 (on-premises), unlimited (EC2) |
| Concurrent deployments | 100 |
| Lifecycle event timeout | 1 hour (default), 2 hours (max) |
| AppSpec file size | 1 MB |

## Auto Scaling

| Resource | Limit |
|----------|-------|
| Auto Scaling groups per region | 500 |
| Launch configurations per region | 200 |
| Scaling policies per ASG | 50 |
| Scheduled actions per ASG | 125 |
| Lifecycle hooks per ASG | 50 |
| SNS topics per ASG | 10 |
| Step adjustments per policy | 20 |
| Target tracking policies per ASG | 10 |

## ECS

| Resource | Limit |
|----------|-------|
| Clusters per region | 10,000 |
| Services per cluster | 5,000 |
| Tasks per service | 5,000 |
| Container instances per cluster | 5,000 |
| Containers per task definition | 10 |
| Task definition revisions | Unlimited |
| CPU units per task (Fargate) | 16 vCPU |
| Memory per task (Fargate) | 120 GB |

## EKS

| Resource | Limit |
|----------|-------|
| Clusters per region | 100 |
| Nodes per managed node group | 450 |
| Managed node groups per cluster | 30 |
| Fargate profiles per cluster | 10 |
| Pods per node (max) | 110-250 (depends on instance type) |
| Label/taint per node group | 50 |

## CloudWatch

| Resource | Limit |
|----------|-------|
| Metrics per PutMetricData call | 1,000 |
| Dimensions per metric | 30 |
| Alarms per region | 5,000 |
| Dashboard widgets | 500 |
| Metric data points per GetMetricData | 100,800 |
| Log groups per region | 1,000,000 |
| Log events per PutLogEvents | 10,000 |
| Log event size | 256 KB |
| Subscription filters per log group | 2 |
| Metric filters per log group | 100 |

## CloudWatch Data Retention

| Resolution | Retention |
|------------|-----------|
| < 60 seconds (high resolution) | 3 hours |
| 60 seconds | 15 days |
| 5 minutes | 63 days |
| 1 hour | 15 months (455 days) |

## SSM Parameter Store

| Type | Limit |
|------|-------|
| Standard parameters | 10,000 per account |
| Advanced parameters | 100,000 per account |
| Standard parameter value size | 4 KB |
| Advanced parameter value size | 8 KB |
| Parameter policies | Advanced only |
| History | 100 versions |

## Secrets Manager

| Resource | Limit |
|----------|-------|
| Secrets per region | 500,000 |
| Secret value size | 64 KB |
| Versions per secret | ~100 (managed automatically) |
| Labels per version | 20 |
| Resource policy size | 20 KB |

## KMS

| Resource | Limit |
|----------|-------|
| CMKs per region | 100,000 |
| Aliases per CMK | 50 |
| Grants per CMK | 50,000 |
| Key policy size | 32 KB |
| Cryptographic operations | 5,500-30,000 RPS (varies by key type) |

## SNS

| Resource | Limit |
|----------|-------|
| Topics per account | 100,000 |
| Subscriptions per topic | 12,500,000 |
| Message size | 256 KB |
| Filter policies per topic | 200 |
| Filter policy size | 256 KB |

## SQS

| Resource | Limit |
|----------|-------|
| Queues per account | Unlimited |
| Message size | 256 KB |
| Message retention | 1 minute - 14 days (default 4 days) |
| Visibility timeout | 0 seconds - 12 hours |
| Long polling wait | 1-20 seconds |
| Batch size | 10 messages |
| In-flight messages (standard) | 120,000 |
| In-flight messages (FIFO) | 20,000 |
| FIFO throughput | 3,000 msg/sec with batching |

## DynamoDB

| Resource | Limit |
|----------|-------|
| Tables per region | 2,500 |
| Item size | 400 KB |
| Partition key | 2,048 bytes |
| Sort key | 1,024 bytes |
| LSIs per table | 5 |
| GSIs per table | 20 |
| Projected attributes (all indexes) | 100 |
| Provisioned capacity per table | 40,000 RCU/WCU |
| On-demand capacity | 40,000 RRU/WRU |

## S3

| Resource | Limit |
|----------|-------|
| Buckets per account | 100 (can be increased to 1,000) |
| Object size | 5 TB |
| Single PUT | 5 GB |
| Multipart upload part | 5 MB - 5 GB |
| Lifecycle rules per bucket | 1,000 |
| Replication rules per bucket | 1,000 |
| Tags per object | 10 |
| Bucket policy size | 20 KB |
| Requests per second per prefix | 3,500 PUT/COPY/POST/DELETE, 5,500 GET/HEAD |

## Kinesis Data Streams

| Resource | Limit |
|----------|-------|
| Shards per region | 500 (soft limit) |
| Data retention | 24 hours - 365 days |
| Record size | 1 MB |
| Write capacity per shard | 1,000 records/sec or 1 MB/sec |
| Read capacity per shard | 5 transactions/sec or 2 MB/sec |
| Enhanced fan-out consumers | 20 per stream |
| GetRecords limit | 10,000 records per call |

## EventBridge

| Resource | Limit |
|----------|-------|
| Rules per event bus | 300 |
| Targets per rule | 5 |
| Event buses per account | 100 |
| Event size | 256 KB |
| Invocations per second | 10,000 (varies by target) |
| PutEvents entries | 10 per request |

## Config

| Resource | Limit |
|----------|-------|
| Config rules per region | 400 |
| Conformance packs per account | 50 |
| Remediation actions concurrent | 25 |
| Aggregators per account | 50 |
| Retention period | 30 days - 7 years |

## X-Ray

| Resource | Limit |
|----------|-------|
| Trace document size | 500 KB |
| Segment document size | 64 KB |
| Sampling rules | 25 |
| Trace retention | 30 days |
| Annotations per segment | 50 |
| Metadata per segment | No limit |

## Key Numbers to Remember

| Item | Value |
|------|-------|
| Lambda timeout | 15 min |
| API Gateway timeout | 29 sec |
| Step Functions Standard execution | 1 year |
| Step Functions Express execution | 5 min |
| CodeBuild timeout | 8 hours |
| SQS message retention | 14 days max |
| SQS visibility timeout | 12 hours max |
| Kinesis retention | 365 days max |
| CloudWatch metrics retention | 15 months |
| S3 object size | 5 TB |
| DynamoDB item size | 400 KB |
