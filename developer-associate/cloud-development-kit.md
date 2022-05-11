# Cloud Development Kit (CDK)

- define your cloud infrastructure using a familiar language:
    - JS/TS, Python, Java, .NET
- contains high level components called *constructs*
- code is 'compiled' into a CloudFormation template (JSON/YAML)
- *can therefore deploy infrastructure and application runtime code together*
    - great for Lambdas
    - great for Docker containers in ECS/EKS
- considered an improvement over YAML because if it doesn't compile, it will error out, shortening the feedback loop 
    - with YAML you only find out when you hit AWS if it works or not

SAM:
- serverless focussed
- write your template declaratively in JSON or YAML
- great for quickly getting started with Lambda Functions
- leverages CloudFormation

CDK:
- all AWS services
- write infrastructure in a programming language
- leverages CloudFormation

## hands on
- need to install the CDK
- use `cdk init` to initialise applications
- need to use package management to install the required cdk constructs
- `cdk destroy` will take down everything created by the cdk
- need to empty the S3 bucket before it can be destroyed