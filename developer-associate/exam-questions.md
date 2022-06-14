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
- there are two methods to save configuration options settings. Config files in YAML or JSON can be included in application's source code in the `.ebextensions` and deployed as part of application source bundle. 
- if you want to deploy a worker application that processes periodic background tasks, application source bundle must also include a cron.yaml file
- *immutable deployments*: launch a full set of new instances running the new version of the application in a separate ASG, alongisde the instances running the old version. If new instances don't pass healthchecks, Elastic Beanstalk terminates them leaving the original instances untouched.
- Blue/Green deployments allow you to have a separate deployment environment. Allow avoidance of downtime. Deploy a new version to separate environments and then swap CNAMEs (called swap environment URLs) of two environments to instantly redirect traffic to new version. Need to create a new environment (can clone current) to do this.
- if you can't find an AMI to run your code, use Packer to create a custom platform which will run it.
- custom platform is a more advanced cusomtisation than a custom image. Lets you develop an entire new platform from scratch, customising operating system, additional software, and scripts the beanstalk runs on platfroms instances. Enables building of platforms for applications that use languages or other infrastructure which beanstalk doesn't provide a managed platform for. Can also use automated, scripted way to create and maintain customisation.
- custom images, where you modify an AMI for use with an existing beanstalk platform, and beanstalk still provides the platform scripts and controls the platforms software stack.  maintenance requires manual changes in the running instances.
- custom AMIs can improve provisioning times when instances are launched in your environment if you need to install a lot of software not in standard AMIs. Configuration files are great (.ebextensions), but can be slow and take a long time during environment creation and update. AMI is faster.
- custom_platform.json is the file for creating a custom platform with Packer
    - *source_ami* is required in a packer template. It is the base operating systems used to create a custom AMI. Could be: amazon linux AI, ubuntu1604, rhel7, rhel6
    - the region id required in a packer template and should be the same as the region from which the AMI of the EC2 instance is copied.
- Dockerrun.aws.json version 2 file can be used to specify EC2 container instance and file volumes for multi-container Docker platform in Beanstalk environments. The file has 3 sections
    1. AWSEVDockerrunVersion: version 2 for multi-container Docker environments
    2. ContainerDefinitions: to specify container definitions
    3. Volumes: creates volumes from folders in the EC2 container instance or source bundle
- `docker-compose.yml` file is used to deploy the docker image from a hosted repository to the beanstalk env. No version 1 or 2 associated with the file.
- Dockerrun.aws.json version 1 file is used to deploy a single docker container to the beanstalk env
- with traffic splitting deployment, beanstalk will launch a completely new set of instances with new versions in a separate ASG and forward on a certain percentage of traffic to the new version during the evaluation period. The percentage of traffic to be diverted to the new version is specified in 'NewVersionPercent' while evaluation time is specified in the 'EvaluationTime' parameter.
- EB CLI provides interactive commands for creating, updating and monitoring envs from a local repo. Use as part of everyday dev and test cycle as alternative to the console.
- to ensure that RDS data is not lost as the Beanstalk environment is torn down, set the *Retention* field to "Create Snapshot", this will mean that a snapshot is taken prior to termination and it can be recreated later. The database needs to be in the environment for it to matter, but don't do it for production envs since the database lifecycle is tied to the Beanstalk lifecycle. For production, launch the database outside of beanstalk and then point beanstalk applications to it.
- Beanstalk creates an application version whenever source code is uploaded. Usually occurs when you create an environment or upload and deploy code using the console or CLI. Beanstalk deletes application versions according to lifecycle policy and when you delete the application. Can also upload a source bundle without deploying it. Beanstalk stores source bundles in S3 and doesn't automatically delete them
- to change config options on a running env. [More info](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/using-features.platform.upgrade.html)
    1. *go to the console*
    2. navigate to the environment management page
    3. choose *configuration*
    4. find the config you want to edit
        - hit *modify* on the option/category you want to change
        - turn on table view to search for it if required
- Beanstalk ASG manages environment instances, and the launch configuration of the instances. Modify the launch configuration to change instance type, key pair, EBS storage and other settings that can only be configured at instance launch. Include a YAML env manifest in root of application source bundle to configure environment name, solution stack and [environment links](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/environment-cfg-links.html) to use when creating the environment. An env manifest uses that same format as [saved configurations](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/environment-configuration-savedconfig.html)
- Beanstalk supports deployment of web applications from Docker containers. With Docker containers, can define your own runtime environment, choose platform, programming language and any dependencies that aren't supported by other platforms. Containers are self-contained and include all the config information and software the web application requires to run
- if run on premise, there is an assumption that running on docker containers could be hard as it is entirely unknown, and that developing a custom AMI or platform will be easier

## SAM

- `AWS::Serverless::Application` - used to embed applications from Amazon S3 buckets
- `AWS::Serverless::API` - used for creating API Gateway resources and methods that can be invoked through HTTPS endpoints
- `AWS::Serverless::LayerVersion` - creates a Lambda layered function
- `AWS::Serverless::Function` - describes the configuration for creating a Lambda Function
- function code needs to be at the root level fo the working directory along with the YAML file
- `DeploymentPreference` in a SAM template can be used to specify traffic shifting patterns. Since it is required to migrate traffic to the new application version quickly, `Canary*X*Percent*Y*Minutes` can be used. With the DeploymentPreference parameter as Canary10Percent10Minutes, 10% of traffic will be reouted to the new version for 10 minutes, after which, 100% of traffic will be routed to the new version

## DynamoDB

- does not support versioning
- when scanning a DynamoDB table, minimise the impact of a scan on a table's provisioned throughput by *reducing page size*. A larger number of requests for smaller data sets allows other requests to succeed without throttling. Parallel scans may consume all provisioned capacity if there are many workers.
- to handle throttling due to a surge in requests, use exponential backoff to retry requests, alternatively increase the throughput on the table (WCU & RCU!!!)
- by default a *query* operation does not return data on how much read capacity it consumes. Use the *ReturnsConsumedCapacity* parameter to do so. Options are:
    - NONE - no data returned
    - TOTAL - include aggregate number of read capacity units consumed
    - INDEXES - include agregate number of read capacity units consumed, with the consumed capacity for each table the index accessed.
- recommendations for partition keys:
    - *use high-cardinality attributes*: distinct values for each item
    - *use composite attributes*: combine more than one attribute to form a unique key
- by default for encryption at rest, DynamoDB uses an *AWS owned key*, a multi-tenant key that is created adn managed in a DynamoDB service account. Can encrypt tables under a CMK or AWS managed key in your AWS account
- server-side encryption is enabled on all DynamoDB table data.
- [1 RCU can support 2 eventually-consistent reads](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadWriteCapacityMode.html) TODO: STUDY RCY/WCU ALGORITHM
    - RCU
        1. item size / 4KB, rounding to nearest whole number
        2. 1 read capacity unit per item (strongly consistent read) x number of reads per second
    - WCU
        1. item size / 1KB, rounding to nearest whole number
        2. 1 write capacity unit per item x number of writes per second
- **when calculating WCU/RCU round off partial values.**
- DynamoDB is an alternative solution that can be used for the storage of session management as latency of access to data is less.
- *global tables* provide a fully managed solution for deploying a multi-region, multi-master database without building and maintaining your own replication solution. When creating a global table, specify the regions you want the table available. DynamoDB performs all the necessary tasks to create identical tables in regions and propogate ongoing data changes to all of them.
- when asked how many reads per second are achievable with certain capacity units, do the maths to see how much data size can be read or written
- can create secondary indexes on a table and issue Query or Scan requests against them, with using a secondary key instead of the primary. 
    - a secondary index is a data structure that contains a subset of attributes from a table, along with an alternate key to support Query operations. With multiple secondary indexes, you can support many different query patterns
-  *global secondary index* does not support Consistent read, only *Eventual Read*. For other tables, Query with Consistent read will get latest results without scanning the whole table.
- does not support complex joins
- *Projection expression* is used to *identify a specific attribute(s)* from a table instead of all items within a table during scan or query operation
- *Update expression* is used to specify *how an update item will modify an item's attribute*.
- *Condition expression* is used to specify a *condition that should be met to modify an item's attribute*
- *Expression attribute names* are used as an *alternate name in an expression instead of an actual attribute name*
- to specify search criteria, use a **key condition expression** - *a string that determines the items to be read from the table or index. Must also specify the partition key name and value as an equality condition*
- encryption is mandatory at time of table creation and is of two types
    1. DEFAULT method using 'AWS owned key'
    2. KMS method using 'AWS managed key'
- Dynamo streams allow Lambda functions to trigger on database modifications. Enable streams, associate the stream ARN with a Lambda function, and immediately after the table is modified, a record appears in the stream. Lambda polls the stream for records and invokes synchronously.
- *BatchGetItem* API allows you to pass multiple partitions key values in a single request, so you can then query for multiple items at once
- *HTTP 400 indicates problem with the request* (authentication failure, missing required parameters, or exceeding a table's provisioned throughput). Will have to resolve issue in application, and resubmit to DynamoDB
- [understand how to write to a table using the CLI](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/WorkingWithTables.Basics.html)
- key definitions:
    - `HASH` is for a partition key
    - `RANGE` is for a sort key
- not to be used to table joins
- best practices for defining secondary indexes:
    - consider projecting fewer attributes to minimise the size of items written to the index. This only applies if the soze of projected attributes would otherwise be larger than a single WCU (1KB)
    - avoid projecting variables that you know will rarely be needed in queries
    - specify ALL only if you want queries to return the entire table items sorted by a different key. Projecting all attributes eliminates the need for table fetches, but in most cases doubles storage costs and write activity

### DAX

*Remember that DAX is also called the DynamoDB accelerator!*

Addresses 3 core scenarios:
1. in-memory cache, reduces response times of eventually-consistent read workloads by order of magnitude (milliseconds -> microseconds)
2. fully managed service requiring minimal functional changes for use with existing applications of DynamoDb
3. for read-heavy or bursty workloads, DAX provides increased throughput and potential operational cost savings by reducing the need to over-provision read capacity units. Especially beneficial for applications that require repeated reads for individual keys.

- for strongly-consistent reads, a DAX cluster will pass all requests directly  to Dynamo and not cache anything
- write-through cache, issue writes directly so that your writes are immediately reflected in the item cache. Write to both the data store and the data cache
- client side cache with redis caches keys that the client uses to access resources.
- write around is where you don't write to the cache when writing data to the data store

## Lambda

- VPC-specific configuration information such as VPC subnets IDs and security grup IDs are required in order to enable Lambda functions to access resources within a VPC.
- qualified ARN: function ARN with a reference to a specific version
- unqualified ARN implicitly invokes the $LATEST version (and therefore publishes that version if it hasn't been already)
- a lambda function version includes
    - the function code and all associated dependencies
    - the lambda runtime that invokes the function
    - all the function settings, including environment variables
    - an ARN to identify the specific version
- a *Publish version* is a snapshot copy of a Lambda Function code and configuration in $LATEST version. No configuration changes can be done to a published version and it has a unique ARN which cannot be modified.
- $LATEST is the latest unpublished change to the lambda function, it could be the current latest published change as well.
- best practice is to avoid using recursive Lambda functions, as it could lead to an unintended volu,e of function invocations and escalated costs. If you do create one, set the concurrent execution limit to Zerp to immediately throttle all invocations to the function while you update the code.
- Lambda functions have a default timeout of 3 seconds. This timeout can be expanded to 900 seconds
- default memory of 128 MB. CPU is allocated by AWS automatically, proportional to memory.
- to deal with `LambdaThrottledExceptions` while using Cognito Events you need to implement retries on synchronous operations while writing the lambda function
- minimise deployment package to runtime necessities. For functions in Java and .Net, avoid uploading the entire AWS SDK library. Instead selectively depend on modules which pick up components of the SDK you need.
- by default, an alias points to a single Lambda version. When the alias is updated to point to a different version incoming request traffic instantly points to the updated version. This exposes that lias to any potential instabilities introduced by the new version. To minimise impact, implement the routing-config parameter of the Lambda alias that allows you to point to two different versionf of the lambda and dictate what percentage of incoming traffic is sent to each.
- with *Lambda proxy integration*, entire request received from client is sent to backend AWS Lambda function using ANY method. Actual HTTP method is received from the client which can be any of the HTTP methods.
- `Routing-Config` parameter of the Lambda alias allows one to point to two different versions of the Lambda function and determine what percentage of incoming traffic is sent to each version. Use percentage values out of 1 (ie, 5% = 0.05) [Read more.](https://docs.aws.amazon.com/lambda/latest/dg/lambda-traffic-shifting-using-aliases.html)
- only pay for the compute time you consume, no charge when not running. administrative headache is minimised by AWS handling the infrastructure, spin up and spin down etc.
- any state under step functions can encounter runtime errors. Lambda can experience transient service errors. By Default, when a state reports an error, Step Functions case execution to fail entirely. For occassionally expected errors, best practice is to retry. In *Retry code*, `ErrorEquals` field is required string that matches error names and all other fields are optional. `BackoffRate` is optional, if not specified, default of 2.0
- Lambda `Catch code` is only used after function performs several retries - `ErrorEquals` and `Next` are required strings.
- Lambda and ECS support blue/green deployments, not in place
- use Cloudwatch events to set up lambda functions to run on a schedule, finest resolution using a cron expression is a minute. Might be seconds in delay between event and the lambda running. Triggered within the minute, but not exactly on the zero second.
- Lambda limits compute and storage resources used to run and store function. Limits apply per-region and can be increased using the support centre console
    - concurrent executions: 1000 default limit
    - function and layer storage: 75 GB default limit
- can run docker containers
- lifecycle event hooks:
    - *BeforeAllowTraffic*: hooks to run tasks before shifting traffic to the new Lambda function version
    - *AfterAllowTraffic*: hooks to run tasks after shiting traffic to the new Lambda version
    - Start, AllowTraffic & end cannot be scripted
- recommended to separate lambda handler from core logic to make function clean and unit-testable
- Lambda's can have dead letter queues which can then be reprocessed, even if you're not initially using SQS or SQS to invoke the function, you can use a DLQ to create an SQS or SNS queue and reprocess later. 

## CloudFormation

- *nested stacks* are stacks that create other stacks.
- *handler*: name of the method within a code that Lambda calls to execute the function
- when launching stacks, install and configure software on EC2 isntances using the `cfn-init` helper script and `AWS::CloudFormation::Init` resource. Using `AWS::CloudFormation::Init` you can describe the configurations you want rather than scripting procedural steps.
- use the *Parameters* section to take in values at runtime. Then use the values of those parameters to define how the template gets executed. Can be refered to from the Resources and Outputs sections of of the template.
    - supported types: string, number, list, CommaDelimitedList, AWS-Specific Parameter types, SSM Parameter types
- *Outputs* section: used to describe the values that are returned whenever you view your stack's properties.
- *Metadata* section: used to specify objects that provide additional information about the template
- *Transform* section: used to specify options for the SAM model
- *StackSets* extend the functionality of stacks by enabling creation, updating or deletion of stacks across multiple accounts and regions with a single operation. Using an admin account, define and manage a CloudFormation template and use the template as the basis for provisioning stacks into selected target accounts across specific regions
- *Nested Stacks* are stacks that are part of other stacks
- *ChangeSets* are used to make changes to running resources in a stack
- optional *Conditions* section contains statements that define the circumstances under which entities are created or configured, only creating specific resources/outputs or setting specific property values if conditions are true, if false setting different values or creating other resources. Common use case is for different environment names

## CodeBuild

- to override default build spec file name or location
    - run create-project or update-project command, setting buildspec value path to an alternate build spec file 
    - run start-build command, setting buildspecOverride value to path to alternate buildspec file
- TODO: check all possible basic documentation errors for AWS services.
- resources in VPC not typically accessible by CodeBuild. *To access, provide additional VPC specific configuration info as part of CodeBuild project config*. Includes VPC ID, subnet IDs, and VPC security group IDs. VPC enabled builds are then able to access resources inside a VPC, making it possible to
    - run integration tests from build against RDS databases in private subnets
    - query data in elasticache cluster from tests
    - interact with internal web services hosted on EC2, ECS or via ELBs

## Step Functions
- require timeouts as without the functions relies on a response from an activity worker to know a task is complete, without a response, it will wait forever.
- executions that pass large payloads between states can be terminated. If data over 32KB use S3 and pass the ARN of the raw data instead. Alternatively, adjust implementation to pass smaller payloads.
- states can have multiple incoming transitions from other states

## Data Pipeline

- web service that you can use to automate data movement and transformation. Define data driven workflows so tasks can depend on successful completion of previous tasks.
- define parameters of data transformations, and Data Pipeline enfores the logic.
- a *data node* defines the location and type of data that a pipeline activity uses as input or output. Types of data node:
    - DynamoDBDataNote: dynamo table contains data for HiveActivity or EmrActivity to use
    - SQLDataNode: sql table and database query that represent data for a pipeline activity to use
    - RedshiftDataNode: redshift table that contains data for RedshiftCopyActivity to use
    - S3DataNode: S3 location that contains one or more files for a pipeline activity to use

## SQS

- SQS extended client library for *Java* is the only way to manage messages in the SQS queue using S3. For messages greater than 256KB, and then referenced using the extended client library for Java.
- to allow for prioritisation of messages in a queue, create two SQS queues, one with higher priority, then messages can be processed by the application from the high priority queue first.
- default settings for SQS queues are a 30 second visbility timeout. If the application needs more time for processing, you need to change this.
- for application polling multiple queues with a single thread: long polling will wait for a message or timeout value for each queue which may delay processing of messages in other queues that have messages to be processed, therefore use *short polling* with default timeout values
- delay queues let you postpone the delivery of new messages to a queue for several seconds. If you create a delay queue, any messages sent to the queue remain invisible to consumers for the delay period (default & minimum is 0s, max of 15m)
- to set delay seconds on *individual* messages, use `message timers` to allow SQS to use the message timer's *DelaySeconds* value instead of the delay queue's

### FIFO SQS

- *existing standard queues cannot be converted into FIFO queues.*
- designed to enhance messaging between applications when order of operations and events is critical or where duplicates can't be tolerated

## SNS

- SNS supports the delivery of message attributes which let you provide structured metadata items (timestamps, geospatial data, signatures, identifiers, etc) about the message, each message can have up to 10 attributes.
- by default a subscriber of an SNS topic receives every message published to the topic. *A subscriber assigns a filter policy to the topic subscription to receive only a subset of messages*. A filter policy is a simple JSON object. It contains attributes that define which messages the subscriber receives. When a message is published to a topic, SNS compares the message attributes to the attributes in the filter policy for each of the topic's subscriptions. If a match between attributes, SNS send the message to the subscriber, otherwise SNS skips the subscriber without sending anything. If a subscription lacks a filter policy, the subscription receives every message published to its topic

## ElastiCache

- ElastiCache is the perfect solution for session state. For scalability and to provide shared data storage for sessions from any web server, abstract HTTP sessions from web servers themselves. Common solution is to use In-Memory Key/Value store.
- Memecached supports:
    - simple caching model
    - multithreaded performance with utilisation of multiple cores
    - [redis vs memached.](https://aws.amazon.com/elasticache/redis-vs-memcached/)
- Redis
    - Redis sorted sets move the computational complexity associated with leaderboard from your application to your redis cluster. Sorted sets guarantee both uniqueness and element ordering. Each time a new element is added to the sorted set, it's reranked in real-time, then added to the set in its appropriate numeric position
- *Write Around Cache* is useful when there is a considerable amount of data to be written to the database
- *Side Cache using Redis* is eventually consistent and non-durable which may add an additional delay
- *Write Through cache using Redis* means chances of missing data during new scaling out
- *Write through*: adds data or updates data in cache whenever data is written to the database
    - data in cache never stale
    - every write involves two trips, adding latency to process. Users are typically more tolerant of latency when writing
        1. write to cache
        2. write to database
- *Parameter groups*: easy way to manage runtime settings for supported engine software. Used to control memory usage, eviction policies, item sizes, etc. Named collection of engine-specific parameters that can apply to a cluster. Allows ensurance that all nodes in cluster are configured the same.
- *Endpoint*: unique address applications use to connext to an elasticache node or cluster
- *Security group*: controls access
- *Subnet group*: cannot be used to define values in Elasticache


## IAM

- permissions of IAM useres and roles assumed are not cumulative, there is *only one set of permissions active at a time*. When you assume a role, you temporarily give up your previous permissions and work with the permissions assigned to the role.
- STS:GetSessionToken is used if you want to use MDA to protect programmatic calls to specific AWS API operations. MFA-enabled IAM users would need to call GetSessionToken and submit an MDA code associated with their MFA device.
- *[cross-account access](https://docs.aws.amazon.com/codecommit/latest/userguide/cross-account.html)* for whenever you want users and groups to access stuff in another account
- policy simulator commands typically require calling API operations to do two things
    1. evaluate the policies and return the list of context keys that they reference. Need to know what context keys are referenced so you can supply values for them in the next step
    2. simulate the policies, providing a list of actions, resources and context keys that are used during the simulation
    - *condition context keys*/condition keys are keys included in the request of all AWS requests. They provide information about the request itself or the resources that the request references. Can check that keys have specified values before allowing the action requested by the user with the `Condition` element of the json policy.
- IAM policy simulator testing to test resource-based policies requires resources to be included in the simulator and the resource policy needs to be selected for that resource.
- *on-premise should not use IAM Roles/instance profiles* [more info](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html) [about what is](https://docs.aws.amazon.com/en_pv/codedeploy/latest/userguide/instances-ec2-configure.html) [needed for on prem](https://docs.aws.amazon.com/codedeploy/latest/userguide/tutorials-on-premises-instance.html)
- security tokens shouldn't be used for development practices, should instead be using temporary security credentials to minimise risk. Assume them using STS.
- *Access Keys*: consist of an access key ID and a secret access key, used to sign programmatic requests made to aws using SDKs, REST or Query API operations
- external IDs are optional and can be applied to trust policies on IAM roles that can be assumed by other accounts - to require an external id when assuming a role, update the trust policy with the external id. It allows the user that is assuming the role to assert the circumstances in which they are operating. It also provides a way for accound owners to permit the role to only be assumed under specific circumstances. [Creating roles with external ID's](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user_externalid.html)
- customer managed policies allow access to resources, they have versions, which you can revert/rollback to as required.
- IAM roles give services/account ability to interact with other services. *If a service needs to interact with another service, give it a role!*
- trust policies give other roles/services the ability to assume a role

## RDS

- RDS supports using TDE (Transparent Data Encryption) to encrypt stored data on DB instances running Microsoft SQL Server. TDE automatically encrypts data before it is written to storage and automatically decrypts data when the data is read from storage. To enable TDE for a DB instance running SQL Server, specify the TDE option in an RDS option group associated with that DB instance.
- web service that makes it easier to set up, operate and scale a relational database in the cloud. Provides cost-efficient, resizable capacity for an industry-standard relational database and manages common database administration tasks. 
- contains several log types
    - *error log*: diagnostic messages generated by database engine, along with startup and shutdown times
    - *general query log*: a record of all SQL statements received from clients and client connect and disconnect times
    - *slow query log*: record of SQL statements that took longer to execute than a set amount of time and that examined more than a defined number of rows - thresholds are configurable
- connect to an RDS instance using endpoints, not IP addresses
    - the DescribeDbInstances API and get the endpoint of the database instance
    - Request an endpoint for the instance

## API Gateway

- Stage variables are name-value pairs that you can define as configuration attributes associated with a deployment stage of an API. They act like environment variables and can be used in your API setup and mapping templates.
- what is the HTTP integration request of the API? 
    - [probably start here](https://docs.aws.amazon.com/apigateway/latest/developerguide/stage-variables.html)
    - [or with these gateway docs](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html#api-gateway-overview-developer-experience)
    - [or with these ones](https://aws.amazon.com/blogs/compute/using-amazon-api-gateway-as-a-proxy-for-dynamodb/)
    - [or these integration types](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-integration-types.html)
- integration types
    - `AWS`: lets an API expose AWS service actions. Must configure both integration request and response and set up necessaru data mappings from the method request to the integration request, and from the integration response to the method response
    - `AWS_PROXY`: lets an API method be intergrated with the Lambda function invocation action with a flexible, versatile and streamlined integration setup. Relies on direct interactions between the client and the integrated lambda function. Also known as Lambda Proxy Integration
    - `HTTP`: lets an API expose HTTP endpoints in the backend. Must configure both the integration request and integration response, data mappings from method request to integration request, and integration response to method response.
    - `HTTP_PROXY`: allows a client access to the backend HTTP endpoints with a streamlined integration setop on a single API method. Do not set the integration request or response. Gateway passes the request from client ot the HTTP endpoint and passes the outgoing response from the HTTP endpoint to the client
    - `MOCK`: lets API Gateway return a response without sending the request further to the backend. Useful for API testing as it can be used to test integration set up without incurring charges for backend use and to enable collaborative dev on an API
    - lambda custom integration with API gateway is where the endpoint corresponds to invoking the function action of the lambda service
    - HTTP integration is the
- developer controls behaviour of API's frontend interactions by configuring the method request and a method response. Control behaviour of backend API interactions by setting up the integration request and integration response. These involve data mappings between a method and its corresponding integration.
- stage variable can be used as part of the HTTP integration URL in the folowing cases
    - full URI without protocol
    - full domain
    - subdomain
    - path
    - query string
- an API stage is: 'a logical reference to a lifecycle state of your API. API stages are identified by API ID and stage name'
- a deployment is represented by a `Deployment` resource. Like an executable of an API represented by a RestApi resource. For a client to call an API, must create a deployment and associate a stage to it. A stage is represented by a Stage resource and represents a snapshot of the API, including methods, integrations, models, mapping templates, Lambda authorisers etc.
- to support CORS API resources need to implement an OPTIONS method to respond to the OPTIONS preflight request with the following headers
    - Access-Control-Allow-Headers 
    - Access-Control-Allow-Origin
    - Access-Control-Allow-Methods
- CORS is set on the API gateway level, not on the EC2 instance or lambda
- to handle different template types defined (Content-Type header vs Accept header) for clients and apis, need to map the payloads. Use Request and Response Data mapping to do so.
- canary deployments have their own logs and metrics generated, which can be viewed as part of a production CloudWatch Logs group, and a canary CloudWatch Logs group. Same applies to access logging.
- while enabling CORS resources on API Gateway for all responses apart from 200 responses of the OPTIONS method, need to manually configure to return Access-Control-Allow-Origin header with '*' or specific origins to fulfill pre-flight handshakes.
- if API Gateway fails to process an incoming request, it returns the client an error response without forwarding the request to the integration backend. By default, the error response contains a short descriptive error message. For some error responses, API Gateway allows customisation by API developers to return the responses in different formats. 
- supports multiple mechanisms for controlling API access
    - resource policies let you create resource-based policies to allow or deny access to APIs and methods from specified IP addresses or VPC endpoints
    - standard IAM roles and policies offer flexible and robust acces controls that can be applied to an entire API or individual methods
    - CORS lets you control how API responds to cross-domain resource requests
    - *lambda authorizers* are lambda functions that control access to AI methods using bearer token authentication as well as info described by headers, paths, query strings, stage variables or context variables request parameters.
    - Cognito user pools create customisable authentication and authorisation solutions
    - client-side SSL certs can be used to verify that HTTP requests to backend systems are from API Gateway
    - usage plans let you provide API keys to customers, then track and limit the usage of API stages and methods for each API key
- control Gateway API method access with IAM *policies*, *roles* and *resource* *policies*
- `Aws_Proxy` can be used for an API method to be integrated with a Lambda where incoming requests from the clients are passed as input to the Lambda.
- to deploy an API, create an API deployment and associate it with a stage. Each stage is a snapshot of the API and is made available for the client to call. Stages enable robust version control of APIs. For each stage, optimise API performance by adjusting the default account-level request throttling limits and enabling API caching. In addition, can override stage-level settings for individual methods and define stage varaibles to pass stage-specific environment contexts to the API integration at runtime. *Every time you update an API, which includes modifications of methods, integrations, authorisers, and anything else other than stage settings, you must redploy the API to an existing or new stage.*
- API Gatewau Lambda proxy integration is a mechanism to build an API with a setup of a single API method. The Lambda proxy integration allows the client to call a single Lambda function in the backend. The function accesses many resources or features of other AWS services, including calling other lambda functions. When a client submits a request, the API Gateway passes the entire request raw as-is to the lambda function, except the order of request parameters is not preserved. Configuration data for the gateway can also be included. A Lambda proxy integration is more potent when it is configured for an API method involving a generic proxy resource. The generic proxy resource can be denoted by a special templated path variable of {proxy+}, the catch-all ANY method placeholder, or both. The client can pass the input to the backend Lambda function in the incoming request as request parameters or applicable payload. The request parameters include headers, URL path variables, query string parameters, and the applicable payload. The integrated Lambda function verifies all of the input sources before processing the request and responding to the client with meaningful error messages if any of the required input is missing. If you want more information, read [this page.](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html)

## CloudWatch

- when creating an alarm, specify three settings to evaluate when to change alarm state:
    - *period*: length of time to evaluate the metric to create each individual data point for a metric. Expressed in seconds.
    - *evaluation period*: number of most recent data points to evaluate when determining alarm state
    - *datapoints to alarm*: number of data points within the evaluation  period that must be breached to cause the alarm to go into the *ALARM* state. Breaching data points do not have to be consecutive. All must be within the last number of data points equal to the evaluation period.
- install the CloudWatch agent on the machine and then configure it to send the web server's logs to a central location in CloudWatch
- using existing `PutMetricData API`, can publish Custom Metrics to a 1 second resolution. More immediate visibility and greater granularity into the state and performance of custom applications, like short lived spikes and functions. Can alert sooner with High-Res alarms, frequently as 10 second periods, which allow you to react and take action faster and support the same actions available with standard 1min alarms. Add these metrics alarms and widgets to dashboard for easy observability of critical components. 
- Detailed monitoring is only pertinant for existing services available in AWS
- unified CloudWatch agent enables:
    - collection of more system-level metrics from EC2 instances, including in-quest metrics, in addition to metrics listed in EC2 Metrics and Dimensions. Additional metrics are listed 'Metric Collected by the CloudWatch Agent'.
        - CloudWatch metric: a metric represents a time-ordered set of data points that are published to CloudWatch. Think a variable to monitor. Metrics only exist in the region they are created, cannot be deleted but expire after 15 months if no new data is published to them. Must be uniquely identified by name, namespace and zero or more dimensions. Each data point in a metric has a time stamp, and optionally a measure.
        - CloudWatch Dimension: is a name/value pair that is part of the identity metric, assign up to 10 dimensions to a metric. Metrics have characteristics to describe them, dimensions can be thought of as categories for those characteristics.
    - collect system-level metrics from on-premise servers. Can include servers in hybrid environments as well as servers not managed by AWS
    - collect logs from EC2 instances and on-premise servers running Windows Server or Linux.

## S3 (3 = simple storage service)

- *simple storage* is object based storage
- Two ways to configure server-side encryption for S3 artifacts
    - CodePipeline creates an S3 artifact bucket and default AWS-managed SSE-KMS encryption keys when creating a pipeline using the Create Pipeline wizard. The master key is encrypted along with object data and managed by AWS.
    - can create and manage your own customer-managed SSE-KMS keys.
- to optimise performance and security while effectively managing cost recommended to set up CloudFront for the S3 bucket to serve and protext content.
- S3 Transfer Acceleration improves transfer performance by routing uploads and downloads through CloudFront's edge locations and over AWS backbone networks, using network protocol optimisations
- cross-region replication is only for high availability, latency improvement and disaster recovery
- CRR helps with
    - compliance requirements
    - latency minimisation
    - increase operational efficiency
    - maintain object copies under different ownership
- configuring bucket for website hosting required configurations
    - enabling website hosting
    - configuring index document support
    - permissions required for website access
- to enable SSE for all objects in a bucket, request should include `x-amz-server-side-encryption` as AES256 request header. Bucket policy can be created to deny all other requests.
- Multipart API enables uploading of multiple objects in parts. Use this API to upload new large objects or copy existing objects. Upon receiving multipart request, S3 constructs object from uploaded parts. The objects can be accessed from the bucket like any other object. Initial three step process
    1. initiate the uplod
    2. upload object parts
    3. after all parts uploaded, complete multipart
- when using CloudFormation, ensure that all objects in the bucket are deleted before deleting the S3 bucket using the *DeletionPolicy* of the CloudFormation template. Default *DeletionPolicy* is **Delete**.
- *provides 3500 PUT requests/second/prefix in a bucket, 5500 GET requests/sec/prefix in a bucket*.
- invoke a lambda with S3 events and pass event data as a parameter, enabling the response to S3 bucket changes. SQS doesn't in itself allow you to do anything, you need more consumers.
- to host a static website, configure an S3 bucket for website hosting and then upload content to the bucket. bucket must have public read access. It is intentinoal that everyone in the world will have read access to this bucket.
- if given the choice, seriously consider whether or not the overhead of multiple buckets is worthwhile. Suspect amazon will typically lean to many buckets not being worthwhile after answering one question.

### S3 Inventory

- if you notice a significant increase in the number of HTTP 503-slow down responses received for Amazon S3 PUT or DELETE object requests to a bucket that has versioning enabled, you might have one or more objects in the bucket for which there are millions of versions. When you have objects with millions of versions, S3 automatically throttles requests to the bucket to protext the customer from an excessive amount of request traffic, which could potentially impede other requests made to the same bucket. *To determine which S3 objects have millions of version, use S3 Inventory. It generates a report that provides a flat file list of the objects in a bucket*

### CORS

- use CORS to get information from between S3 buckets. Using bucket policies opens up access to the entire bucket which is sub-optimal
- if you're using the fully qualified domain name for a bucket, you're leaving the initial origin and CORS needs to be enabled.
- if you're making a request to outside of an APIs own domain, you also need to enables CORS
- in config, can specify values for which methods are allowed using the *AllowedMethod* element
    - GET, PUT, POST, DELETE, HEAD
- for simple cross-origin POST method requests, response from resource needs to include the header `Access-Control-Allow-Origin` where the value of the header key is set to `*` or to the origins allowed to access that resource. When a browser receives a non-simple HTTP request, the CORS protocol requires the browser to send a preflight request to the server and wait for approval (or a request for credentials) from the server before sending the actual request. The *preflight request* appears to your API as an HTTP request that
    - includes an origin header
    - uses the OPTIONS method
    - includes headers:
        - `Access-Control-Request-Method`
        - `Access-Control-Request-Headers`

## Envelope Encryption

- unencrypted data is encrypted using a plaintext Data key. Data key is further encrypted using a plaintext Master key. Plaintext master key isstored in KMS and known as Cusomter master keys.

## SDK

- implements exponential backoffs for you.

## CodePipeline

- an action is a task performed on an artifact in a stage. If an action of a set of parallel actions is not completed successfully, the pipeline stops running
- as a *best practice*, when using Jenkins as a build provider for pipeline's build or test action, install Jenkins on an EC2 instance and configure a separate EC2 instance profile. Make sure the instance profile grants Jenkins only the AWS permissions required to perform tasks for your project. Instance profile provides applications running on an EC2 instance with the credentials to access other services, as a result, you don't need to configure AWS credentials.
    - Jenkins cannot be installed on a lambda function
- includes several actions that help configure, build, test, and deploy resources for automated release process. If release process includes activities that are not included in the default actions, can create a *custom action* for that purpose and include it in the pipeline. Can use CLI to create custom actions in pipelines associated with AWS account.

## EBS

- local block storage for EC2 instances
- encryption needs to be enabled at volume creation time

## Elastic File Store

- create network file systems that can be mounted by instances across multiple availability zones. An EFS is an AWS resource that sues security groups to control access over the network in your default or custom VPC.
- in Elastic Beanstalk envs, use EFS to create a shared directory that stores files uploaded or modified by users of the application. The application can treat a mounted EFS volume like local storage. Don't have to change application code to scale up to multiple instances

## CodeStar

- service for creating, managing and working with software development projects on AWS. Creates and integrates AWS service for project development toolchains. Also manages permissions required for project users. 
- basically handles the development process side of things
- service accelerates release with help of CodePipeline. Each project comes pre-configured with an automated pipeline that continuously builds, tests, and deploy code with each commit.
- to quickly build and deploy applications onn EC2 instances using Ruby, CodeStar project templates can be used. Pre-configured delivery toolchains for developing, building, testing and deploying projects on AWS. Project templates support development of applications on various AWS services (eg. EC2, Elastic Beanstalk, Lambda) and various languages (eg. Java, JS, PHP, Ruby, Python). Provide built-in security access policies to control access to team working on new applications

## CodeDeploy

- if using Lambda, AppSpec file is used to specify
    - Lambda version to deploy
    - functions to be used as validation tests
- to access secure parameters in Parameter Store, use ssm `get-parameters` and specify the `--with-decryption`option, which allows the CodeDeploy service to decrypt the passowrd so it can be used. Use IAM roles to ensure CodeDeploy can access Parameter Store.
- if CodeDeploy generates "HEALTH_CONSTRAINTS_INVALID" error, make sure the required number of healthy instances are available during deployment, the number should be less than or equal to the total number of instances.
- during automatic rollback, will try to retrieve diles that were part of previous versions. If these files are deleted or missing, needs to manually add files to the instance or create a new application version
- 3 ways traffic can shift during deployment
    1. canary: traffic shifted in two increments
    2. linear: traffic shifted in equal increments with an equal number of minutes between each increment.
    3. all-at-once
- appspec.yml is used to specify how to deploy an application to instances in a deployment group or which lambda dunction version to deploy. Specific to CodeDeploy
- when deploying EC2 instance
    - timeout parameter in AfterInstall event for hooks has a default clue of 3600s. Define a timeout value in the afterinstall hook to change it
    - afterinstall hook is supported with in-place deployment and blue/green in case of replacement instances
    - directory for storing scripts to be executed in hooks should be at root level of the EC2 instance.
    - 'runas is an optional parameter in the hooks section
- deployment to on-premise do not use IAM instance profiles

## ELB

- access logs on the load balancer capture detailed information about requests sent to the load balancer. Each log contains information such as time request received, client IP, latencies, request paths and server responses.
- ALBs allow containers to use dynamic host port mapping so that multiple tasks from the same service are allowed per container instance
- ALBs support path-based routing and priority rules so that multiple services can use the same listener port on a single ALB
- NLBs do not support dynamic host port mapping
- 

## ECR

- to retagdocker images, not required to pull or push images to the repository. The `--image-tag` option of the put-image command can be used to retag the existing image in the repository

## OpsWorks

- OpsWorks is a configuration management service that provides managed instances of Chef and Puppet. These are automation platforms that allows use of code to aumatomate configurations of servers. OpsWorks allows for automation of how servers are configured, deployed anad managed across EC2 instances or on-premise compute environments

## EC2

- deploying
    - *rolling with additonal batch deployment*: a new batch of EC2 instance is launched before taking a batch of instances out of service for deploying a new version. Once all EC2 instances are upgraed to a new version of the application, additional batch of EC2 instance is terminated. Ensures full capacity during deployment.
    - *immutable*: spins up entirely new ASG of instances and replaces the existing ones. If replacement is not required, this option is a no go.
- Applications must sign their API requests with AWS credentials. Therefore if you are an application developer, you need a strategy for managing credentials for your applications that run on EC2 instances. IAM roles are designed so that applications can securely make API requests from instances without requiring management of security credentials that applications use. Instead of creating and distributing AWS credentials, delegate permission to make API requests using IAM roles by:
    1. create an IAM role
    2. define which accounts or AWS services can assume the role
    3. define which API actions and resources the application can use after assuming the role
    4. specify the role when you launch your instance, or attach the role to a running or stopped instance
    5. have the application retrieve a set of temporary credentials and use them
- instance types
    - with M5 General-Purpose instance, Elastic Network Adapter is used to support Enhance Networking. M5 GP also supports network performance with 10Gbps to 25 Gbps based upon instance type. T2 does not support Enhance Networking and only supports network performance to 1Gbps
    - Amazon EC2 provides enhanced networking capabilities through the Intel 82599 VF interface which is available on some instances.
- instance types
    - M5: general purpose instance
    - Mac: apple system, suitable for applications that require XCode
    - C5: compute optimized instances
    - R5: memory optimized instances
    - TODO: make sure you understand the differences between [EC2 instance types](https://aws.amazon.com/ec2/instance-types/)
- supports in place deployment for EC2 and on-premise, Lambda and ECS only support blue/green deployments

## Auto Scaling Groups

- with target tracking scaling policies, select a predefined metric or configure a custom metric and set a target value. Application auto scaling creates and manages CloudWatch alarms that trigger the scaling policy and calculates the scaling adjustment based on the metric and the target value. The scaling policy adds or remoces capacity as required to keep the metric at or close to the specified target value. In addition to keeping the metric close to the target value, a target tracking scaling policy also adjusts to changes in the metric due to a changing load pattern. It minimises changes to the capacity of the scalable target. Since you can define a custom metric based on concurrent usage, can scale the ASG based on that.


## X-Ray

- `~/xray-daemon$./xray -o` command option can be used while running the X-Ray daemon locally and not on the EC2 instance. Will then skip checking EC2 instance metadata
- [info on configuring AWS X-Ray daemon](https://docs.aws.amazon.com/xray/latest/devguide/xray-daemon-configuration.html)
- on ECS, create a docker image that runs the X-Ray daemon, upload it to a docker image repository, then deploy to an ECS cluster. Use port mappings and network mode settings in task definition files to allow the application to communicate with the daemon container.
- with Default Sampling Rule, X-Ray records are one request per second, and 5% of any additional requests per host
- [read more about sampling](https://docs.aws.amazon.com/xray/latest/devguide/xray-api-sampling.html)
    - Resevoir rate + fixed rate
    - number = total requests - resevoir rate, take the fixed rate % from the number
- SDK provides
    - `interceptors` to add to code to trace incoming HTTP requests
    - `client handlers` to instrument SDK clients that application uses to call other services
    - `HTTP client` to instrument calls to other internal and external HTTP web services
- Lambda uses environment variables to facilitate comms with X-Ray daemon and configure X-Ray SDK
    - `_X_AMZN_TRACE_ID`: contains tracing header, which includes the sampling decision, trace ID, and parent segment ID. If Lambda receives a tracing header when function is invoked, the header will be used to populate this environment variable. If a tracing header was not received, Lambda will generate one.
    - `AWS_XRAY_CONTEXT_MISSING`: X-Ray SDK uses this variables to determine behaviour in the even that the function tries to record X-Ray data, but a tracing header is not available. Lambda sets values to LOG_ERROR by default
    - `AWS_XRAY_DAEMON_ADDRESS`: environment variable exposes daemon's address as: `IP_ADDRESS:PORT`. Use the daemon's address to send trace data to the daemon directly without using the X-Ray SDK
- to run on ECSm create a Docker image that runs the X-Ray daemon, upload it to a docker image repository, then deploy to the ECS cluster. Use port mappings and metwork mode settings in task definition file to allow application to communicate with the daemon container.

## Kinesis

- encryption at rest is most easily handled using in-built Server-side encryption available with Kinesis Streams. Automatically encrypts data before at rest using a KMS CMK you specify. Data is encrypted before written to Kinesis stream storage layer and decrypted after retrieval from storage. 
- cannot track successful completion of each item independently
- queue can be replaued
- able to route records to selected shards using a partition key
- maintains order of log statements

### Kinesis Data Streams

- expects you to call PutRecord API to write serially to a shard while using the `sequenceNumberForOrdering` parameters. This parameter guarantees strictly increasing of sequence numbers for puts from same client and to same partition key.
- use server-side encryption for data encryption at rest with an AWS KMS CMK. Encrypted before written to the stream storage layer and decrypted after retrieval.
- can select either an existing kinesis stream or create a new one to have Cognito Streams push all sync data to streams.
- shard throughput capacity: 1MB/sec/shard of data input, and 1000 PUT records/sec/shard
- before creating a stream, need to determine initial size. After creation can dynamically scale up or down using the console for `UpdateShardCount` API. To determine initial size you need:
    - average size of the data record written to the stream in KB, rounded to the nearest 1KB, the data size - `average_data_size_in_KB`
    - number of data records written to and read from the stream/second - `records_per_second`
    - number of Kinesis Data Streams applications that consume data concurrently and independently from the stream, the consumers - `number_of_consumers`
    - incoming write bandwidth in KB - `incoming_write_bandwidth_in_KB`, which is equal to the `average_data_size_in_KB` X `records_per_second`
    - outgoing read bandwidth in KB - `outgoing_read_bandwidth_in_KB` which is equal to the `incoming_write_bandwidth_in_KB` X `number_of_consumers`
    - calculate initial shards - `number_of_shards` using:
        `number_of_shards = max(incoming_write_bandwidth_in_KB/1024, outgoing_read_bandwidth_in_KB/2048)`
    - [read more](https://github.com/awsdocs/amazon-kinesis-data-streams-developer-guide/blob/master/doc_source/amazon-kinesis-streams.md)
    - TODO: ensure you understand the above formulas

### Kinesis data firehose

- fully managed service for delivering *real-time stream data* to destinations such as S3, Redshift, ElasticSearch and Splunk. [What is this service](https://docs.aws.amazon.com/firehose/latest/dev/what-is-this-service.html)
- can enable encryption (at rest), when Kinesis Streams are chosen as source, encryption of data at rest is enabled automatically


## KMS

- requests directly to KMS and using AWS services both have the KMS request limit apply. This can result in throttling.
- AWS Encryption SDK is a client-side encryption library that makes it easier to implement cryptography best practices in applications. Includes secure default behaviour for developers who are not encryption experts, while being flexible enough to work for most experienced users. By default, generate a new data key for each encryption operation
    - to encrypt data: use the CMK to generate a data key for the encryption process
    - to decrypt: use the generated data key
- you don't use the CMK to encrypt and decrypt data directly. Recommended pattern to encrypt client side:
    1. GenerateDataKey to get a data encryption key
    2. plaintext data encryption key to encrypt data locally, then erase plaintext key from memory
    3. store encrypted data key alongside locally encrypted data
- unless using customer provided keys, the keys are managed by AWS!

## Cognito

- Roles can have rules assigned to them. When multiple rules are assigned, rules are evaluated in a sequential order & the IAM role for the first matching rule is used unless a 'CustomRoleArn' attribute is added to modify this sequence.
- Cognito with MFA allows MFA be required with a User Pool
- supports authentication with identity providers through SAML (security assertion markup language 2.0). Identity provider supporting SAML specifies IAM roles that can be assumed by users to different users can be granted different sets of permissions
- User gets authenticated by SAML-based 3rd party identity prodiver and gets a token. Cognito Identity pool provides credentials based on these tokens. Applications cam access AWS services using these creds. When using on-premises, the user will be authenticated by on-prem servers. Tokens provided by on-prem servers will be exchanged with Cognito identity pool to get AWS credentials, which are then used to access AWS services.
- Cognito `User pool` is for sign-in access for users to web applications, not to grant user access to AWS services
- Cognito `Identity pool` is to grant users access to AWS services
- federated identity providers are Amazon, Google, etc. Not on-prem.
- `ĀssumeRoleWithWebIdentity`: returns a set of temporary security credentials for users who have been authenticated in a mobile or web app with a web identity provider.
- Cognito can detect if a user's creds have been compromised elsewhere. Use *Advanced Security* in console to choose whether to `allow` or `block use` if compromised details are detected. If allow, all attempted uses of comprmised creds are publised to CloudWatch

### Cognito Sync

- Cognito Sync is a service and client library that enable cross-device syncing of application-related user data. Can use it to synchronise user profile data across mobile devices and web applications. Client libraries cache data locally so app can read and write data regardless of device connectivity status. When device online, can synchronise data, and if push sync set up, notify other devices immediately that an update is available. [Read more](https://docs.aws.amazon.com/cognito/latest/developerguide/getting-started-with-cognito-sync.html)

## STS

- aspects that get incorporated into the STS call are:
    - ARN of the role the app should assume
    - duration of the temporary security credentials
    - role session name, string value used to identify the session, which can be captured and logged by CloudTrail to help distinguish between role users during an audit
- `aws sts decode-authorization-message` command decodes information about authorisation status of a request from an encoded mesage returned in an AWS response
- a trail enables ongoing delivery of events as log files to an S3 bucket. Ensures a record of events extending past 90 days, option to automatically monitor and alarm on specified event by sending log events to CloudWatch Logs, and an option to query logs and analyse service activity with Athena
- to track a role and activity, use cloudwatch agents to detect the ARN in the event record of CloudTrail, which will trigger an alarm. [Read more](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/send-cloudtrail-events-to-cloudwatch-logs.html)

## ECS

- ECS schedules containers for execution on customer-controlled EC2 instances or with Fargate and builds on the same isolation controls and compliance that is available for EC2 customers. Compute instances are located in a VPC within an IP range you specify. You decide which instances are exposed to the net, and which remain private. The EC2 instances use an IAM role to access the ECS service
    - ECS tasks use an IAM role to access services and resources
    - security groups and network ACLs allow control of inbound and outbound network access to and from instances
    - can connect existing infrastructure to resources in VPC using industry-standard encrypted IPsec VPN connections
    - can provision EC2 resources as Dedicated Instances, which are EC2 instances that run on hardware dedicated to a single customer for additional isolation
- when an ECS container is stopped, the container instance status remains `Active` but the container Agent status changes to `FALSE` after a few minutes
- Lambda and ECS only support blue/green deployments


## CloudTrail

- CloudTrail logs āll authenticated requests to IAM and STS APIs, except DecodeAuthorizationMessage. Also logs nonauthenticated requests to STS actions, AssumeRoleWithSAML and AssumeRoleWithWebIdentity, and logs information provided by the identity provider. Can use this information to map calls made by a federated user with an assumed role back to the originating external federated caller.

## Redshift

- an internet hosting service and data warehouse product that forms part of the larger cloud-computing platform AWS. Built on top of technology from the massive parallel processing data warehouse company ParAccel, to handle large data sets and database migrations

## CodeCommit

- migrate a Git repo to CodeCommit by: cloning, mirroring migrating all or some branches. Can also migrate local unversioned content to CodeCommit
- can migrate to CodeCommit from other version control systems, but need to migrate to Git first
- HTTPS connections require Git creds which IAM users can generate themselves in IAM, or an AWS access key. Repository users must configure credential helper included in AWS CLI. Root or federated users must use HTTPS.  Git creds are easiest for repo setup and use. 
    - SSH connections require users to generate a public-privte key pair, store the public key, associate the public key with their IAM user, config known hosts file on local and create and maintain a config file on local machines. More complex that HTTPS and git creds.
    - HTTPS auth method: static username and password. works with all OS supported by CodeCommit. Compatible with IDEs and other dev tolls supporting git creds

## Route 53

- *weighted routing* lets you associate multiple resources with a single domain or subdomain name, and choose how much traffic is routed to each resource.
    - when combined with an ELB, you can progressibly scale the weighted routing over time, so that more customers are served the new version after initial testing.

## VPC

- VPC flow logs is a feature that enables capturing of information about the IP traffic to and from network interfaces in VPC
- NAT gateway is a Network Address Translation service. You can use a NAT gateway to give instances in a private subnet can connect to services outof the the VPC, but external services cannot innitiate a connection with those instances.
    - `Public` NAT gateways allow instances in private subnets to connect to the internet through the gateway. Must associate an elastic IP address with the fateway at creation. Route traffic from the NAT gateway to the internet gatewau for the VPC. Alternatively use a public gatewau to connect to other VPCs or an on-premise network. 
    -  `Private` gateways allow instance in private subnets to connect to instances in other VPCs or on-prem networks through the private gateway. Route traffic from the NAT gateway through a transit gateway or a virtual private gateway. Cannot associate an elastic IP address with a private NAT gateway. Can attach an internet gateway to a VPC with a private nat gateway, but if you route traffic from the private NAT gateway to the internet gateway, the internet gateway drops traffic
    - The NAT gateway replaces the source IP address of the instances with the IP address of the instancxes with the IP address of the NAT gatewau. When sending response traffic to instance, the NAT gateway translates addresses back to the original source IP address

## CloudFront

- for web distributions, to control how long objects stay in a CloudFront cache before CloudFront forwards another request to the origin
    - configure origin to add a `Cache-Control` or `Expires` header field to each object
    - specify a value or minimum TTL in CloudFront cache behaviours
    - use the default value of 24 hours
- to require HTTPS between viewers and CloudFront, specify either
    - *Redirect HTTP to HTTPS*
        - viewers can use both protocols, GET and HEAD requests are automatically redirected to HTTPS requests, CloudFront returns HTTP 301 (moved permanently) along with the new HTTPS URL. The viewer then resubmits the request to CloudFront using the HTTPS URL.
        - for other HTTP requests, responses with a HTTP 907 (temporary redirect), ensuring the same message gets sent (for HTTP 1.1 or above)
    - *HTTPS only*
        - change the origin protocol policy for applicable origins
        - HTTPS only means CloudFront uses only HTTPS to communicate with custom origin
        - Match Viewer means CloudFront communicates with custom origin using HTTP or HTTPS depending on the protocal of the viwer request.
- allows for very high GET performance. Using CloudFront, users can access data locally with low-latency and high throughput from the nearest POP. Also results in a decrease in the number of read requests directly to S3 buckets


## Other

- 429 errors are throttling. Best practice often includes exponential backoffs.
- To make requests to AWS you must supply *AWS credentials to the AWS SDK for Java*.
    - options:
        - use the default credential chain provider
        - use a specific credential provider or provider chain (or create your own)
        - supply the credentials yourself (root account, IAM or temporary AWS STS credentials)
    - when you initialise a new service client without supplying any arguments, the SDK for Java attempts to find AWS credentials using the default credential provider chain implemented by the DefaultAWSCredentialsProviderChain class. The default credential provider chain looks for credentials in this order:
        1. *Environment variables* - AWS_KEY_ID + AWS_SECRET_ACESS_KEY
        2. *Java system properties* - aws.accessKeyId and aws.secretKey
        3. *Web Identity Token credentials* from the environment or container
        4. *Default credential profiles file* - typically located at ~/.aws/credentials and shared by many AWS SDKs and CLIs.
        5. *ECS container credentials* - loaded form ECS if the environment variable AWS_CONTAINER_CREDENTIALS_RELATIVE_URI is set
        7. *Instance profile credentials* - used on EC2 instances and delivered through EC2 metadata service
- Fault Injection Simulator
    - FIS experiment templates consist of 
        1. action set
        2. targets
        3. stop conditions
        - resources can be specified in targets using the following components
            - resource IDs
            - resource tags
            - resource filters
    - to specify resources, paths and values can be specified in resource filters components.
    - TODO: have a skim of this service to ensure you understand how to apply tags and IDs as well as the action set 
- setting up client side encryption recommended pattern
    1. use `GenerateDataKey` to get a data encryption key
    2. use the plaintext data encryption key from the plaintext field of previous response to encrypt data locally, then erase the plaintext data key from memory
    3. store the encrypted data key (retunred previously in the CiphertextBlob field of the response) alongside the locally encrypted data
    4. then send the data.
