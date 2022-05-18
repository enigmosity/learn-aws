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
- *immutable deployments*: launch a full set of new instances running the new version of the application in a separate ASG, alongisde the instances running the old version. If new instances don't pass healthchecks, Elastic Beanstalk terminates them leaving the original instances untouched.
- Blue/Green deployments allow you to have a separate deployment environment.
- if you can't find an AMI to run your code, use Packer to create a custom platform which will run it.


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
- recommendations for partition keys:
    - *use high-cardinality attributes*: distinct values for each item
    - *use composite attributes*: combine more than one attribute to form a unique key
- by default for encryption at rest, DynamoDB uses an *AWS owned key*, a multi-tenant key that is created adn managed in a DynamoDB service account. Can encrypt tables under a CMK or AWS managed key in your AWS account
- server-side encryption is enabled on all DynamoDB table data.
- 1 RCU can support 2 eventually-consistent reads
- **when calculating WCU/RCU round off partial values.**
- DynamoDB is an alternative solution that can be used for the storage of session management as latency of access to data is less.

### DAX

*Remember that DAX is also called the DynamoDB accelerator!*

Addresses 3 core scenarios:
1. in-memory cache, reduces response times of eventually-consistent read workloads by order of magnitude (milliseconds -> microseconds)
2. fully managed service requiring minimal functional changes for use with existing applications
3. for read-heavy or bursty workloads, DAX provides increased throughput and potential operational cost savings by reducing the need to over-provision read capacity units. Especially beneficial for applications that require repeated reads for individual keys.

- for strongly-consistent reads, a DAX cluster will pass all requests directly  to Dynamo and not cache anything

## Lambda

- VPC-specific configuration information such as VPC subnets IDs and security grup IDs are required in order to enable Lambda functions to access resources within a VPC.
- TODO: look into the $LATEST alias, $LATEST version has two ARNs associated with it, Qualified ARN and Unqualified ARN. What does that mean? [Probably start here.](https://docs.aws.amazon.com/lambda/latest/dg/configuration-versions.html)
- a *Publish version* is a snapshot coy of a Lambda Function code and configuration in $LATEST version. No configuration changes can be done to a published version and it has a unique ARN which cannot be modified.
- best practice is to avoid using recursive Lambda functions, as it could lead to an unintended volu,e of function invocations and escalated costs. If you do create one, set the concurrent execution limit to Zerp to immediately throttle all invocations to the function while you update the code.

## CloudFormation

- *nested stacks* are stacks that create other stacks.
- *handler*: name of the method within a code that Lambda calls to execute the function


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

## SQS

- SQS extended client library for *Java* is the only way to manage messages in the SQS queue using S3. For messages greater than 256KB, and then referenced using the extended client library for Java.
- to allow for prioritisation of messages in a queue, create two SQS queues, one with higher priority, then messages can be processed by the application from the high priority queue first.

## ElastiCache

- ElastiCache is the perfect solution for sessio state. For scalability and to provide shared data storage for sessions from any web server, abstract HTTP sessions from web servers themselves. Common solution is to use In-Memory Key/Value store.
- Memecached supports:
    - simple caching model
    - multithreaded performance with utilisation of multiple cores
    - TODO: [check out redis vs memached.](https://aws.amazon.com/elasticache/redis-vs-memcached/)
- Redis
    - Redis soted sets move the computational complexity associated with loeaderboard from your application to your redis cluster. Sorted sets guarantee both uniqueness and element ordering. Each time a new element is added to the sorted set, it's reranked in real-time, then added to the set in its appropriate numeric position

## IAM

- permissions of IAM useres and roles assumed are not cumulative, there is *only one set of permissions active at a time*. When you assume a role, you temporarily give up your previous permissions and work with the permissions assigned to the role.
- STS:GetSessionToken is used if you want to use MDA to protect programmatic calls to specific AWS API operations. MFA-enabled IAM users would need to call GetSessionToken and submit an MDA code associated with their MFA device.

## RDS

- RDS supports using TDE (Transparent Data Encryption) to encryt stored data on DB instances running Microsoft SQL Server. TDE automatically encrypts data before it is written to storage and automatically decrypts data when the data is read from storage. To enable TDE for a DB instance running SQL Server, specify the TDE option in an RDS option group associated with that DB instance.

## API Gateway

- Stage variables are name-value pairs that you can define as configuration attributes associated with a deployment stage of an API. They act like environment variables and can be used in your API setup and mapping templates.
- TODO: what is the HTTP integration request of the API? [probably start here](https://docs.aws.amazon.com/apigateway/latest/developerguide/stage-variables.html)
- developer controls behaviour of API's frontend interactions by configuring the method request and a method response. Control behaviour of backend API interactions by setting up the integration request and integration response. These involve data mappings between a method and its corresponding integration.
- stage variable can be used as part of the HTTP integration URL in the folowing cases
    - full URI without protocol
    - full domain
    - subdomain
    - path
    - query string

## CloudWatch

- when creating an alarm, specify three settings to evaluate when to change alarm state:
    - *period*: length of time to evaluate the metric to create each individual data point for a metric. Expressed in seconds.
    - *evaluation period*: number of most recent data points to evaluate when determining alarm state
    - *datapoints to alarm*: number of data points within the evaluation  period that must be breached to cause the alarm to go into the *ALARM* state. Breaching data points do not have to be consecutive. All must be within the last number of data points equal to the evaluation period.
- install the CloudWatch agent on the machine and then configure it to send the web server's logs to a central location in CloudWatch

## S3

- *simple storage* is object based storage
- Two ways to configure server-side encryption for S3 artifacts
    - CodePipeline creates an S3 artifact bucket and default AWS-managed SSE-KMS encryption keys when creating a pipeline using the Create Pipeline wizard. The master key is encrypted along with object data and managed by AWS.
    - can create and manage your own customer-managed SSE-KMS keys.

### CORS

- use CORS to get information from between S3 buckets. Using bucket policies opens up access to the entire bucket which is sub-optimal

## Envelope Encryption

- unencrypted data is encrypted using a plaintext Data key. Data key is further encrypted using a plaintext Master key. Plaintext master key isstored in KMS and known as Cusomter master keys.

## SDK

- implements exponential backoffs for you.

## CodePipeline

- an action is a task performed on an artifact in a stage. If an action of a set of parallel actions is not completed successfully, the pipeline stops running

## EBS

- encryption needs to be enabled at volume creation time

## CodeStar

- service for creating, managing and working with software development projects on AWS. Creates and integrates AWS service for project development toolchains. Also manages permissions required for project users. 
- basically handles the development process side of things

## CodeDeploy

- if using Lambda, AppSpec file is used to specify
    - Lambda version to deploy
    - functions to be used as validation tests

## ELB

- access logs on the load balancer capture detailed information about requests sent to the load balancer. Each log contains information such as time request received, client IP, latencies, request paths and server responses.

## ECR

- to retagdocker images, not required to pull or push images to the repository. The `--image-tag` option of the put-image command can be used to retag the existing image in the repository

## OpsWorks

- OpsWorks is a configuration management service that provides managed instances of Chef and Puppet. These are automation platforms that allows use of code to aumatomate configurations of servers. OpsWorks allows for automation of how servers re configured, deployed anad managed across EC2 instances or on-premises compute environments

## EC2

- deploying
    - *rolling with additonal batch deployment*: a new batch of EC2 instance is launched before taking a batch of instances out of service for deploying a new version. Once all EC2 instances are upgraed to a new version of the application, additional batch of EC2 instance is terminated. Ensures full capacity during deployment.
    - *immutable*: spins up entirely new ASG of instances and replaces the existing ones. If replacement is not required, this option is a no go.