# GuardDuty

- Intelligent Threat discovery to protect AWS accounts
- Uses machine learning algorithms to perform anomaly detection, uses 3rd party data
- One click to enable, has 30 day trial, no need to install any software
- Input data includes:
    - CloudTrail Logs: unusual API calls, unauthorized deployments
    - VPC Flow Logs: unusual internal traffic, unusual IP addresses
    - DNS Logs: compromised EC2 instances sending encoded data within DNS queries
    - Optional Features: EKS Audit Logs, RDS and Aurora, EBS, Lambda, S3 Data Events, etc.
- We can set up EventBridge rules to notify us in case of findings
- EventBridge rules can target AWS Lambda or SNS
- Can protect against CryptoCurrency attacks (has a dedicated "finding" for it)

## Multi-Account Strategy

- We can manage multiple accounts in GuardDuty by using AWS Organizations
- Member accounts can be managed with the administrator account
- Administrator can do the following:
    - Add/remove member accounts in GuardDuty
    - Manage GuardDuty within the associated member accounts
    - Manage findings, suppression rules, trusted IP lists, threat list
- The administrator of GuardDuty does not necessarily have to be the administrator of the organization, we can have delegated administrators for GuardDuty

## Findings Automated Response

- Findings are potential security issues for malicious events happening in our AWS account
- We can automate the response to security issues revealed by GuardDuty Findings using EventBridge
- Events are published to both the administrator account and the member account that it originated from

## Findings

- GuardDuty pulls independent streams of data directly from CloudTrail logs, VPC Flow Logs and EKS logs
- Each finding has a severity value between 0.1 and 8+ (High, Medium, Low)
- There is a finding naming convention: `ThreatPurpose:ResourceTypeAffected/ThreatFamilyName.DetectionMechanism!Artifact`
    - `ThreatPurpose`: primary purpose of the threat (Backdoor, CryptoCurrency, etc.)
    - `ResourceTypeAffected`: AWS resource affected
    - `ThreatFamilyName`: describes the potential malicious activity (NetworkPortUnusual, etc.)
    - `DetectionMechanism`: method used by GuardDuty to detect the finding (TCP, UDP, etc.)
    - `Artifact`: resource that is used in the malicious activity (DNS, etc.)
- We can generate sample findings in GuardDuty to test our automations
- Finding types:
    - EC2 Finding Types
    - IAM Finding Types
    - Kubernetes Audit Logs Finding Types
    - Malware Protection Finding Types
    - RDS Protection Finding Types
    - S3 Finding Types

## Trusted and Threat IP Lists

- Works only for public IP addresses
- We can define a list of trusted IP addresses (Trusted IP List):
    - These are CIDR ranges that we trust
    - GuardDuty does not generate findings for these
- Threat IP List:
    - List of known malicious IP addresses and CIDR ranges
    - GuardDuty will generate findings based on these IP ranges
    - Can be supplied by 3rd party threat intelligence or created custom by us

## CloudFormation Integration

- We can enable GuardDuty from CloudFormation template
- If GuardDuty is already enabled, the CloudFormation Stack deployment will fail
- We can use Custom Resources (Lambda) to conditionally enable GuardDuty if it is not enabled yet
- We can then deploy this across all the organization using Stack Sets