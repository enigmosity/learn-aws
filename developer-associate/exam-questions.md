# Learnings from sitting practice exams

## Elastic Beanstalk

- configuration options
    - during environment creation, configuration options are applied from multiple sources with precedence:
    1. *Settings applied directly to the environment* - setting specified during a create or update environment operation by a client. AWS Console and the EB CLI also apply recommended values for some options at this level unless overridden.
    2. *Saved configurations* - settings for ay options that are not applied directly to the environment are loaded from a saved configuration if one is specified
    3. *Configuration files (.ebextensions)* - executed in alphabetical order.
    - *Default Values* apply if the option is not set an any of the other levels
    - setting in location with highest precedence is applied.
    - *Settings in configuration files are not applied directly to the environment and cannot be removed without modifying the configuration files and deploying a new application version.*
- if you want to deploy a worker application that processes periodic background tasks, application source bundle must also include a cron.yaml file

## SAM

- `AWS::Serverless::Application` - used to embed applications from Amazon S3 buckets
- `AWS::Serverless::API` - used for creating API Gateway resources and methods that can be invoked through HTTPS endpoints
- `AWS::Serverless::LayerVersion` - creates a Lambda layered function
- `AWS::Serverless::Function` - describes the configuration for creating a Lambda Function

## DynamoDB

- does not support versioning
- when scanning a DynamoDB table, minimise the impact of a scan on a table's provisioned throughput by *reducing page size*. A larger number of requests for smaller data sets allows other requests to suceed without throttling. Parallel scans may consume all provisioned capacity if there are many workers.
- a *query* operation does not return data on how much read capacity it consumes. Use the *ReturnsConsumedCapacity* parameter to do so. Options are:
    - NONE - no data returned
    - TOTAL - include aggregate number of read capacity units consumed
    - INDEXES - include agregate number of read capacity units consumed, with the consumed capacity for each table the index accessed.

### DAX

*Remember that DAX is also called the DynamoDB accelerator!*

Addresses 3 core scenarios:
1. in-memory cache, reduces response times of eventually-consistent read workloads by order of magnitude (milliseconds -> microseconds)
2. fully managed service requiring minimal functional changes for use with existing applications
TODO: what is the third scenario

- for strongly-consistent reads, a DAX cluster will pass all requests directly  to Dynamo and not cache anything

## Lambda

- VPC-specific configuration information such as VPC subnets IDs and security grup IDs are required in order to enable Lambda functions to access resources within a VPC.

## CloudFormation

- *nested stacks* are stacks that create other stacks.

## CodeBuild

- to override default build spec file name or location
    - run create-project or update-project command, setting buildspec value path to an alternate build spec file 
    - run start-build command, setting buildspecOverride value to path to alternate buildspec file
- TODO: check all possible basic documentation errors for AWS services.

## Step Functions
- require timeouts as without the functions relies on a response from an activity worker to know a task is complete, without a response, it will wait forever.
- executions that pass large payloads between states can be terminated, use Simple Storage Service to deal with this.
- states can have multiple incoming transitions

## Data Pipeline

- web service that you can use to automate data movement and transformation. Define data driven workflows so tasks can depend on successful completion of previous tasks.
## Other

- *simple storage* is object based storage
