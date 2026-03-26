# AWS DevOps Professional - CLI Commands Cheatsheet

## CloudWatch

```bash
# Put custom metric
aws cloudwatch put-metric-data \
  --namespace "MyApp" \
  --metric-name "PageViews" \
  --value 100 \
  --dimensions Page=home,Browser=Chrome

# Get metric statistics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 3600 \
  --statistics Average

# Set alarm state (for testing)
aws cloudwatch set-alarm-state \
  --alarm-name "HighCPU" \
  --state-value ALARM \
  --state-reason "Testing alarm"

# Create log group
aws logs create-log-group --log-group-name /my-app/logs

# Create export task to S3
aws logs create-export-task \
  --log-group-name /my-app/logs \
  --from 1609459200000 \
  --to 1609545600000 \
  --destination my-bucket \
  --destination-prefix exports
```

## SSM (Systems Manager)

```bash
# Get parameter
aws ssm get-parameter --name /app/config/db-host
aws ssm get-parameter --name /app/secrets/db-password --with-decryption

# Get parameters by path
aws ssm get-parameters-by-path --path /app/config --recursive

# Put parameter
aws ssm put-parameter \
  --name /app/config/db-host \
  --value "mydb.example.com" \
  --type String

# Put secure string
aws ssm put-parameter \
  --name /app/secrets/db-password \
  --value "secret123" \
  --type SecureString \
  --key-id alias/my-key

# Run command
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=instanceids,Values=i-1234567890abcdef0" \
  --parameters commands=["echo hello"]

# Start session
aws ssm start-session --target i-1234567890abcdef0

# List command invocations
aws ssm list-command-invocations \
  --command-id "command-id" \
  --details
```

## CodeCommit

```bash
# Clone repo (HTTPS GRC)
git clone codecommit::us-east-1://my-repo

# Create repository
aws codecommit create-repository --repository-name my-repo

# Create branch
aws codecommit create-branch \
  --repository-name my-repo \
  --branch-name feature-branch \
  --commit-id abc123

# Create pull request
aws codecommit create-pull-request \
  --title "My PR" \
  --targets repositoryName=my-repo,sourceReference=feature,destinationReference=main

# Get differences
aws codecommit get-differences \
  --repository-name my-repo \
  --before-commit-specifier main \
  --after-commit-specifier feature
```

## CodeBuild

```bash
# Start build
aws codebuild start-build --project-name my-project

# Start build with override
aws codebuild start-build \
  --project-name my-project \
  --source-version feature-branch \
  --environment-variables-override name=ENV,value=prod

# Batch get builds
aws codebuild batch-get-builds --ids build-id-1 build-id-2

# List builds for project
aws codebuild list-builds-for-project --project-name my-project
```

## CodeDeploy

```bash
# Create deployment
aws deploy create-deployment \
  --application-name my-app \
  --deployment-group-name my-dg \
  --s3-location bucket=my-bucket,key=app.zip,bundleType=zip

# Stop deployment
aws deploy stop-deployment --deployment-id d-ABC123

# Get deployment
aws deploy get-deployment --deployment-id d-ABC123

# List deployments
aws deploy list-deployments \
  --application-name my-app \
  --deployment-group-name my-dg \
  --include-only-statuses Failed

# Register on-premises instance
aws deploy register-on-premises-instance \
  --instance-name my-server \
  --iam-user-arn arn:aws:iam::123456789012:user/deploy-user
```

## CodePipeline

```bash
# Start pipeline execution
aws codepipeline start-pipeline-execution --name my-pipeline

# Get pipeline state
aws codepipeline get-pipeline-state --name my-pipeline

# Get pipeline execution
aws codepipeline get-pipeline-execution \
  --pipeline-name my-pipeline \
  --pipeline-execution-id execution-id

# Retry stage
aws codepipeline retry-stage-execution \
  --pipeline-name my-pipeline \
  --stage-name Deploy \
  --pipeline-execution-id execution-id \
  --retry-mode FAILED_ACTIONS

# Put approval result
aws codepipeline put-approval-result \
  --pipeline-name my-pipeline \
  --stage-name Approval \
  --action-name ManualApproval \
  --token token-id \
  --result summary="Approved",status=Approved
```

## CloudFormation

```bash
# Create stack
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --parameters ParameterKey=Env,ParameterValue=prod \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM

# Update stack
aws cloudformation update-stack \
  --stack-name my-stack \
  --template-body file://template.yaml

# Create change set
aws cloudformation create-change-set \
  --stack-name my-stack \
  --change-set-name my-changes \
  --template-body file://template.yaml

# Execute change set
aws cloudformation execute-change-set \
  --stack-name my-stack \
  --change-set-name my-changes

# Detect drift
aws cloudformation detect-stack-drift --stack-name my-stack

# Describe drift
aws cloudformation describe-stack-resource-drifts --stack-name my-stack

# Delete stack
aws cloudformation delete-stack --stack-name my-stack

# List exports
aws cloudformation list-exports

# Signal resource (for wait conditions)
aws cloudformation signal-resource \
  --stack-name my-stack \
  --logical-resource-id WaitHandle \
  --unique-id i-1234567890abcdef0 \
  --status SUCCESS
```

## Lambda

```bash
# Invoke function
aws lambda invoke \
  --function-name my-function \
  --payload '{"key": "value"}' \
  response.json

# Update function code
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function.zip

# Publish version
aws lambda publish-version --function-name my-function

# Create alias
aws lambda create-alias \
  --function-name my-function \
  --name prod \
  --function-version 1

# Update alias (traffic shifting)
aws lambda update-alias \
  --function-name my-function \
  --name prod \
  --function-version 2 \
  --routing-config AdditionalVersionWeights={"1"=0.1}

# Add permission
aws lambda add-permission \
  --function-name my-function \
  --statement-id sns-invoke \
  --action lambda:InvokeFunction \
  --principal sns.amazonaws.com \
  --source-arn arn:aws:sns:us-east-1:123456789012:my-topic
```

## ECS

```bash
# Update service
aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --task-definition my-task:2 \
  --desired-count 3

# Force new deployment
aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --force-new-deployment

# Run task
aws ecs run-task \
  --cluster my-cluster \
  --task-definition my-task \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-1],securityGroups=[sg-1]}"

# Execute command (ECS Exec)
aws ecs execute-command \
  --cluster my-cluster \
  --task task-id \
  --container my-container \
  --interactive \
  --command "/bin/sh"
```

## Auto Scaling

```bash
# Set desired capacity
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name my-asg \
  --desired-capacity 5

# Update ASG
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --min-size 2 \
  --max-size 10

# Create scaling policy
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-asg \
  --policy-name scale-out \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration file://config.json

# Complete lifecycle action
aws autoscaling complete-lifecycle-action \
  --lifecycle-hook-name my-hook \
  --auto-scaling-group-name my-asg \
  --lifecycle-action-result CONTINUE \
  --instance-id i-1234567890abcdef0

# Record lifecycle heartbeat
aws autoscaling record-lifecycle-action-heartbeat \
  --lifecycle-hook-name my-hook \
  --auto-scaling-group-name my-asg \
  --instance-id i-1234567890abcdef0
```

## Config

```bash
# Put config rule
aws configservice put-config-rule \
  --config-rule file://rule.json

# Start remediation
aws configservice start-remediation-execution \
  --config-rule-name my-rule \
  --resource-keys resourceType=AWS::S3::Bucket,resourceId=my-bucket

# Get compliance
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name my-rule

# Describe compliance
aws configservice describe-compliance-by-resource \
  --resource-type AWS::EC2::Instance
```

## Secrets Manager

```bash
# Create secret
aws secretsmanager create-secret \
  --name my-secret \
  --secret-string '{"user":"admin","password":"secret123"}'

# Get secret value
aws secretsmanager get-secret-value --secret-id my-secret

# Rotate secret
aws secretsmanager rotate-secret --secret-id my-secret

# Update secret
aws secretsmanager update-secret \
  --secret-id my-secret \
  --secret-string '{"user":"admin","password":"newsecret"}'
```

## KMS

```bash
# Encrypt
aws kms encrypt \
  --key-id alias/my-key \
  --plaintext fileb://plaintext.txt \
  --output text \
  --query CiphertextBlob

# Decrypt
aws kms decrypt \
  --ciphertext-blob fileb://ciphertext.blob \
  --output text \
  --query Plaintext

# Generate data key
aws kms generate-data-key \
  --key-id alias/my-key \
  --key-spec AES_256

# Create grant
aws kms create-grant \
  --key-id key-id \
  --grantee-principal arn:aws:iam::123456789012:role/my-role \
  --operations Encrypt Decrypt
```

## EventBridge

```bash
# Put rule
aws events put-rule \
  --name my-rule \
  --event-pattern '{"source":["aws.ec2"],"detail-type":["EC2 Instance State-change Notification"]}'

# Put targets
aws events put-targets \
  --rule my-rule \
  --targets Id=1,Arn=arn:aws:lambda:us-east-1:123456789012:function:my-function

# Test event pattern
aws events test-event-pattern \
  --event-pattern '{"source":["aws.ec2"]}' \
  --event '{"source":"aws.ec2","detail-type":"test"}'
```

## X-Ray

```bash
# Get trace summaries
aws xray get-trace-summaries \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z

# Get trace
aws xray batch-get-traces --trace-ids trace-id-1 trace-id-2

# Get service graph
aws xray get-service-graph \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z

# Create sampling rule
aws xray create-sampling-rule --sampling-rule file://rule.json
```

## SAM CLI

```bash
# Initialize new project
sam init --runtime python3.9

# Build
sam build

# Local invoke
sam local invoke MyFunction --event event.json

# Local API
sam local start-api

# Deploy (guided)
sam deploy --guided

# Deploy
sam deploy --stack-name my-stack --capabilities CAPABILITY_IAM

# Sync (hot reload)
sam sync --watch --stack-name my-stack
```
