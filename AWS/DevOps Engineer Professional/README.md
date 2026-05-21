# AWS Certified DevOps Engineer — Professional (DOP-C02)

Study notes organized by exam domain.

## Layout

- [`01-sdlc-automation/`](01-sdlc-automation/) — CodeBuild, CodeDeploy, CodePipeline, CodeArtifact, CodeGuru, EC2 Image Builder, Amplify, Jenkins
- [`02-configuration-management-and-iac/`](02-configuration-management-and-iac/) — CloudFormation, CDK, SAM, OpsWorks, Service Catalog, Step Functions, SSM, AppConfig, EBS
- [`03-resilient-cloud-solutions/`](03-resilient-cloud-solutions/) — ASG, ALB/NLB, ECS/EKS/ECR, Lambda, API Gateway, Route 53, DynamoDB, ElastiCache, Kinesis, DMS, S3, DR
- [`04-monitoring-and-logging/`](04-monitoring-and-logging/) — CloudWatch, Athena, OpenSearch, Lookout
- [`05-incident-and-event-response/`](05-incident-and-event-response/) — CloudTrail, EventBridge, SNS/SQS, X-Ray, EC2 status checks, S3 event notifications, health dashboards
- [`06-security-and-compliance/`](06-security-and-compliance/) — Organizations, IAM Identity Center, Config, Control Tower, GuardDuty, Inspector, Macie, Detective, WAF, Network Firewall, Firewall Manager, Trusted Advisor
- [`07-other-services/`](07-other-services/) — CloudFront, Redshift, QuickSight, Copilot, A2C, tagging
- [`cheatsheets/`](cheatsheets/) — Quick-reference tables (CLI, limits, comparison tables, exam tips)
- [`code examples/`](code%20examples/) — Working CloudFormation templates and SAM apps

## Table of Contents

### 1. SDLC Automation
- [CI/CD overview](01-sdlc-automation/cicd.md)
- [CodeCommit](01-sdlc-automation/codecommit.md)
- [CodeBuild](01-sdlc-automation/codebuild.md)
- [CodeDeploy](01-sdlc-automation/codedeploy.md)
- [CodePipeline](01-sdlc-automation/codepipeline.md)
- [CodeStar (replaced by CodeCatalyst)](01-sdlc-automation/codestar%20-%20replaced%20by%20codecatalyst.md)
- [CodeArtifact](01-sdlc-automation/codeartifact.md)
- [CodeGuru](01-sdlc-automation/codeguru.md)
- [EC2 Image Builder](01-sdlc-automation/ec2-image-builder.md)
- [Amplify](01-sdlc-automation/amplify.md)
- [Jenkins on AWS](01-sdlc-automation/jenkins.md)
- [SDLC Whitepapers](01-sdlc-automation/whitepapers.md)

### 2. Configuration Management and IaC
- [CloudFormation](02-configuration-management-and-iac/cloudformation.md)
- [Service Catalog](02-configuration-management-and-iac/service-catalog.md)
- [Elastic Beanstalk](02-configuration-management-and-iac/ebs.md)
- [SAM — Serverless Application Model](02-configuration-management-and-iac/sam.md)
- [CDK — Cloud Development Kit](02-configuration-management-and-iac/cdk.md)
- [Step Functions](02-configuration-management-and-iac/step-functions.md)
- [AppConfig](02-configuration-management-and-iac/appconfig.md)
- [Systems Manager (SSM)](02-configuration-management-and-iac/ssm.md)
- [OpsWorks](02-configuration-management-and-iac/opsworks.md)

### 3. Resilient Cloud Solutions
- [Lambda](03-resilient-cloud-solutions/lambda.md)
- [API Gateway](03-resilient-cloud-solutions/api-gw.md)
- [ECS — Elastic Container Service](03-resilient-cloud-solutions/ecs.md)
- [ECR — Elastic Container Registry](03-resilient-cloud-solutions/ecr.md)
- [EKS — Elastic Kubernetes Service](03-resilient-cloud-solutions/eks.md)
- [Kinesis](03-resilient-cloud-solutions/kinesis.md)
- [Route 53](03-resilient-cloud-solutions/route53.md)
- [Auto Scaling Groups](03-resilient-cloud-solutions/asg.md)
- [Application Auto Scaling](03-resilient-cloud-solutions/application-auto-scaling.md)
- [RDS Read Replicas](03-resilient-cloud-solutions/rds-read-replicas.md)
- [ElastiCache](03-resilient-cloud-solutions/elasticache.md)
- [DynamoDB](03-resilient-cloud-solutions/dynamodb.md)
- [DMS — Database Migration Service](03-resilient-cloud-solutions/dms.md)
- [S3](03-resilient-cloud-solutions/s3.md)
- [Elastic Load Balancer](03-resilient-cloud-solutions/elb.md)
- [NAT Gateways](03-resilient-cloud-solutions/nat.md)
- [Resilient architectures](03-resilient-cloud-solutions/resilient-architectures.md)
- [Disaster recovery](03-resilient-cloud-solutions/disaster-recovery.md)
- [Data Lifecycle Manager](03-resilient-cloud-solutions/amazon-data-lifecycle-manager.md)

### 4. Monitoring and Logging
- [CloudWatch](04-monitoring-and-logging/cloudwatch.md)
- [Lookout for Metrics](04-monitoring-and-logging/lookout.md)
- [OpenSearch](04-monitoring-and-logging/opensearch.md)
- [Athena](04-monitoring-and-logging/athena.md)

### 5. Incident and Event Response
- [EventBridge (formerly CloudWatch Events)](05-incident-and-event-response/eventbridge.md)
- [S3 Event Notifications & object integrity](05-incident-and-event-response/s3-event-notifications.md)
- [Health Dashboards](05-incident-and-event-response/health-dashboards.md)
- [EC2 Status Checks](05-incident-and-event-response/ec2-status-checks.md)
- [CloudTrail](05-incident-and-event-response/cloudtrail.md)
- [SNS and SQS](05-incident-and-event-response/sns-sqs.md)
- [X-Ray](05-incident-and-event-response/x-ray.md)
- [Distro for OpenTelemetry](05-incident-and-event-response/aws-distro-for-opentelemetry.md)

### 6. Security and Compliance
- [Config](06-security-and-compliance/config.md)
- [Organizations](06-security-and-compliance/organizations.md)
- [Control Tower](06-security-and-compliance/control-tower.md)
- [IAM Identity Center](06-security-and-compliance/iam-identity-center.md)
- [WAF](06-security-and-compliance/waf.md)
- [Network Firewall](06-security-and-compliance/network-firewall.md)
- [Firewall Manager](06-security-and-compliance/firewall-manager.md)
- [GuardDuty](06-security-and-compliance/guard-duty.md)
- [Detective](06-security-and-compliance/amazon-detective.md)
- [Inspector](06-security-and-compliance/amazon-inspector.md)
- [EC2 Compliance](06-security-and-compliance/ec2-compliance.md)
- [Trusted Advisor](06-security-and-compliance/trusted-advisor.md)
- [Data Protection](06-security-and-compliance/data-protection.md)
- [Cost Allocation Tags](06-security-and-compliance/cost-allocation-tags.md)
- [Secrets / License Manager](06-security-and-compliance/manager.md)
- [Macie](06-security-and-compliance/macie.md)

### 7. Other Services
- [Tagging strategies & Tag Editor](07-other-services/tagging.md)
- [QuickSight](07-other-services/amazon-quicksight.md)
- [App2Container (A2C)](07-other-services/aws-a2c.md)
- [Copilot](07-other-services/aws-copilot.md)
- [CloudFront](07-other-services/cloudfront.md)
- [Redshift](07-other-services/aws-redshift.md)

## Exam Description

The AWS Certified DevOps Engineer — Professional (DOP-C02) exam validates technical expertise in provisioning, operating, and managing distributed application systems on AWS. It targets engineers who:

- Implement and manage continuous delivery systems and methodologies on AWS.
- Implement and automate security controls, governance processes, and compliance validation.
- Define and deploy monitoring, metrics, and logging systems on AWS.
- Implement systems that are highly available, scalable, and self-healing on AWS.
- Design, manage, and maintain tools to automate operational processes.
