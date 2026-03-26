![[Pasted image 20260326210409.png]]
# AWS CloudTrail

- Provides governance, compliance and audit for an AWS account
- CloudTrail is enabled by default!
- Provides a history of events/API calls made within an AWS account
- It can put logs into CloudWatch Logs or S3
- A trail can be applied to All Regions (default) or a single Region
- In case of a resource deletion, to investigate it (who did it), we have to look inside CloudTrail first
- The default UI only shows create, modify and delete events
- CloudTrail Trail features:
    - Provides detailed list of all events
    - Ability to store these events in S3 for further analysis
    - It can be region specific or global
- CloudTrail logs are encrypted with SSE-S3 encryption by default when they are stored into S3. There is a possibility to use SSE-KMS encryption
- A CloudTrail log entry contains information about:
    - Who made the request
    - When was the request made and from where
    - What was requested
    - What was the response
- CloudTrail may have a 15 minutes delay to deliver log files into the S3 bucket

## CloudTrail Events

- Management Events:
    - Operations that are performed on resources in our AWS account
    - Example: `AttachRolePolicy`, `CreateSubnet`, `CreateTrail`
    - By default trails are configured to log management events no mather what
    - We can separate Read Events (that don't modify resources) from Write Events (that modify resources)
- Data Events:
    - By default data events are not logged (because high volume of operations)
    - Data events are: 
        - S3 object-level activity (`GetObject`, `DeleteObject`, `PutObject`)
        - AWS Lambda function execution activity
- CloudTrail Insights Events:
    - If enabled, it will analyze events to detect unusual activity in our account. Examples:
        - Inaccurate resource provisioning
        - Hitting service limits
        - Bursts of AWS IAM actions
        - Gaps in periodic maintenance activity
    - CloudTrail Insights analyzes normal management events to create a baseline and then continuously analyzes write events to detect unusual patterns
    - Insights events will appear in CloudTrail console
    - They are also sent to S3 (if enabled)
    - An EventBridge event is generated (for automation needs)

## CloudTrail Events Retention

- Events by default are stored for 90 days in CloudTrail
- To keep events beyond these period we can send these events to S3 and use Athena to analyze them

## Log Integrity

- We can validate the integrity of the logs file using the AWS CLI
- The AWS CLI allows us to detect the following type of changes:
    - Modification or deletion of CloudTrail log files
    - Modification or deletion of CloudTrail digest files
    - Modification or deletion of both of the above
- Validate logs command:
    ```
    aws cloudtrail validate-logs --start-time <time> --trail-arn arn:aws:cloudtrail:us-east-2:123456:trail/my-trail-name --verbose --profile aws-devops
    ```

## Cross Account Logging

- Reference: [https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-receive-logs-from-multiple-accounts.html](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-receive-logs-from-multiple-accounts.html)
- Steps to enable cross account logging:
    1. Turn on CloudTrail in the account where the destination bucket will belong
    2. Update the bucket policy to grant cross-account permission to CloudTrail
    3. Turn on CloudTrail on the other accounts. Configure CloudTrail to use the same bucket from the destination account

---

CloudTrail doesn't emit events directly — it writes logs. To respond to those logs you need to route them through other services. There are two main patterns:

**Pattern 1 — CloudTrail → EventBridge (real time)**

This is the fastest path. CloudTrail management events are automatically delivered to EventBridge without any configuration on your part. You just write a rule that matches the event you care about:

json

```json
{
  "source": ["aws.iam"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventName": ["CreateAccessKey", "AttachRolePolicy", "PutRolePolicy"]
  }
}
```

Then route that rule to a target — Lambda, SNS, SQS, Step Functions. Seconds from the API call to your response.

---

**Pattern 2 — CloudTrail → S3 → Athena (batch analysis)**

CloudTrail writes logs to S3. You can query them with Athena for historical analysis, threat hunting, or compliance reporting:

**The full real-time response stack**
```
API call happens
    ↓
CloudTrail (records within ~15 seconds)
    ↓
EventBridge (matches rule, triggers target)
    ↓
Lambda (evaluates severity, decides response)
    ↓
Remediation (revoke, revert, isolate) + Alert (Slack, PagerDuty, SNS)
    ↓
CloudTrail logs the remediation action too
````

The whole loop from API call to automated response can be under 30 seconds with EventBridge and Lambda. The main latency is CloudTrail's delivery time to EventBridge which is typically 15-30 seconds for management events.

**Faster: prevent it from happening at all**

An SCP or resource policy deny is instant — zero latency because the action never succeeds. If you can express the bad thing as a policy rule, prevention beats any detection and response time. This is the core argument for shifting left into preventive controls.

---

**Faster: AWS Config proactive rules (pre-deployment)**

Config has a mode called proactive evaluation that checks resources before they are created via ONLY CloudFormation. It blocks the deployment if the resource would be non-compliant — so the bad state never exists rather than being detected after the fact.

---

**Faster for non-CloudTrail signals: GuardDuty**

GuardDuty has its own threat detection pipeline that runs independently of CloudTrail. For certain threat types — credential exfiltration, unusual API call patterns, crypto mining, DNS exfiltration — GuardDuty can fire faster than a CloudTrail → EventBridge chain because it has direct access to the underlying data streams (VPC Flow Logs, DNS logs, CloudTrail) and runs ML models against them continuously. GuardDuty findings also arrive in EventBridge so you can wire the same Lambda response to them.

---

**Faster for network-level threats: VPC Network Firewall / Security Groups**

For network-based attacks, blocking at the network layer is faster than any API-level detection. A Security Group rule or Network Firewall policy blocks traffic inline — no detection lag at all.