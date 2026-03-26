# AWS SAM - Serverless Application Model



**1. Shorthand syntax** The main value prop — resources that would take 100+ lines of CF can be a dozen lines in SAM. Lambda + API Gateway + IAM role in CF is verbose; in SAM it's a single `AWS::Serverless::Function` with an `Events` block.

---

**2. Local development (`sam local`)** — local Lambda invocation, local API Gateway, no deploy needed for basic testing.

---

**3. Built-in best practices by default** When SAM expands your template it automatically creates:

- IAM execution roles with least-privilege policies
- CloudWatch log groups with retention set
- Tracing config hooks for X-Ray

Things you'd have to remember to write manually in raw CF.

---

**4. `sam sync` — hot deployment** Pushes code and infrastructure changes to AWS incrementally without a full CF stack update. Significantly faster iteration loop when testing against real AWS services.

---

**5. `sam pipeline`** Generates CI/CD pipeline config for CodePipeline, GitHub Actions, GitLab etc. out of the box — scaffolds the deploy stages, environment separation, and IAM roles needed for a proper pipeline.

---

**6. Policy templates** SAM ships with a library of pre-built IAM policy templates for common patterns:

yaml

```yaml
Policies:
  - DynamoDBCrudPolicy:
      TableName: !Ref MyTable
  - S3ReadPolicy:
      BucketName: !Ref MyBucket
```

Instead of writing raw IAM JSON. Relevant for you in supply chain security — these are audited, least-privilege policies rather than hand-rolled ones that tend to be over-permissive.

---

**7. CF passthrough** Anything SAM can't express you can write as raw CF in the same template — they coexist cleanly. You're never blocked by SAM's abstractions.

---

**8. Open source + AWS native**

- Transforms happen server-side via AWS — no proprietary toolchain lock-in at runtime
- The transform itself is open source so you can inspect exactly what gets generated
- Backed by AWS so it tracks new Lambda features quickly

---

**What SAM is NOT good at:**

- Non-serverless infrastructure — for VPCs, RDS, complex networking, raw CF or CDK is better
- Large multi-service architectures — CDK scales better with programmatic constructs and reuse
- Teams that prefer a real programming language for IaC — CDK (TypeScript/Python) wins there

---

**SAM vs CDK in one line:** SAM is YAML-first and Lambda-focused. CDK is code-first and covers your whole stack. Many teams use SAM for pure serverless microservices and CDK for everything else.


- It is a framework for developing and deploying serverless applications
- All the configurations for SAM are stored in YAML code
- SAML cli generates complex CloudFormation templates from SAM YAML code
- Supports anything from CloudFormation: Outputs, Mappings, Parameters, Resources, etc.
- SAM can be used with CodeDeploy to deploy Lambda functions
- SAM can help use to run Lambda, API Gateway and DynamoDB locally

## Recipe

- SAM YAML template contains the following headers:
    - `Transform: 'AWS::Serverless-2016-10-31`
    - The meaning of this is that CloudFormation knows how to transform this YAML template into CloudFormation template
- Other resources:
    - `AWS::Serverless::Function`
    - `AWS::Serverless::Api` - API Gateway
    - `AWS::Serverless::SimpleTable` - DynamoDB table
- Package and deploy SAM applications:
    - `aws cloudformation package` / `sam package`
    - `aws cloudformation deploy` / `sam deploy`

## SAM CLI

- To run a SAM application locally we can use the SAM cli
- It helps us locally build and debug serverless applications
- SAM has support for IDEs: VSCOde, JetBrains Intellij/PyCharm, AWS Cloud9, etc.
- AWS Toolkit: IDE plugins which allows us to build, test, debug and deploy Lambda functions using AWS SAM

## SAM CLI Commands

1. Download a sample application

    ```
    sam init --runtime java11
    ```

2. Build the application

    ```
    sam build
    ```

3. Invoke function locally

    ```
    sam local invoke <function-name> -e event.json
    sam local start-api
    ```

4. Package the application

    ```
    sam package --output-template packaged.yaml --s3-bucket <bucket-name> --region us-east-2 --profile aws-devops
    ```

5. Deploy the application

    ```
    sam deploy --template-file packaged.yaml --capabilities CAPABILITY_IAM --stack-name <stack-name> --region us-east-2 --profile aws-devops
    ```

## SAM with CodeDeploy

- SAM framework natively uses CodeDeploy to update/deploy Lambda functions
- CodeDeploy will leverage the Traffic Shifting feature using Lambda aliases
- Pre and Post traffic hooks can be defined to validate deployment
- Automated rollbacks are supported with CloudWatch Alarms
- SAM with CodeDeploy YAML configuration:
    - `AutoPublishAlias`:
        - Detects when new code is being deployed
        - Creates and publishes an updated version of the function with the latest code
        - Points the alias to the updated version of the Lambda function
    - `DeploymentPreference`: `Canary`, `Linear`, `AllAtOnce`
    - `Alarms`:
        - Can trigger a rollback
    - `Hooks`:
        - Pre and post traffic shifting Lambda functions used to test the deployment