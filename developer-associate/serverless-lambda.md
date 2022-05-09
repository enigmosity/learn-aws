# Serverless

Serverless is a new paradigm in which developers no longer have to manage servers

# Lambda

- virtual functions
- limited by runtime - short executions
- run on-demand
- auto scaling

*Benefits*
- easy pricing
    - pay per request and compute time
    - big free tier
- integrated with whole AWS suite of services
- integrated with many programming languages
- easy monitoring through AWS CloudWatch
- easy to get more resources per function (up to 10GB of RAM)
- increasing RAM will also improve CPU, network quality and performance

*Language support*
- Node.js
- Python
- Java
- C# (.NET Core)
- Golang
- C# / PowerShell
- Ruby
- Custom Runtime API
- Lambda Container Image
    - image must implement the Lambda Runtime API
    - ECS/Fargate is preferred for running arbitrary Docker images
    - EXAM: unless asked to run on a custom image, Lambda is not for Docker

*For the Developer Associate exam, unless you're asked to run it on a custom image, Lambda functions are not for Docker.*

*Main Integrations*
- API Gateway: create a rest API and invoke Lambda functions
- Kinesis: data transformations on the fly
- DynamoDB: create some triggers from a database
- S3: trigger on file creation
- CloudFront
- EventBridge: trigger when something happens in your infrastructure
- Logs: put logs wherever you like
- SNS: trigger as an SNS topic
- SQS: as a consumer
- Cognito: react to whenever something happens, like a user logging into a database

## Synchronous Invocations

- CLI, SDK, API Gateway, ALB
    - results returned right away
    - error handling must happen client side (retries, exponential backoff, etc.)
- user invoked
    - ELB (ALB)
    - API Gateway
    - CloudFront (Lambda@Edge)
    - S3 batch
- service invoked
    - Cognito
    - Step Functions
- other services
    - Lex
    - Alexa
    - Kinesis Data Firehose
- hitting test in the console on a Lambda function is a synchronous invocation

## Lambda Integration with ALB
- to expose a Lambda function as an HTTP/S endpoint
- use an ALB or API gateway
- Lambda function must be registered in a *target group*

*Request from ALB to Lambda*: HTTP is translated by the ALB into JSON 
On the return: the ALB translates from JSON to HTTP

*ALB Multi-Header Values*
- ALB can support multi-header values
- when enabled, *HTTP headers and query string parameters* that are sent with *multiple values* are shown as *arrays* within the AWS Lambda event and response objects

### hands on
- ALBs are still in the EC2 section even when configured for use with Lambda Functions
- return a status code for the ALB to send back over the internet
- resource based policy on the Lambda Function is what gives permissions to be invoked by the ALB

## Lambda@Edge 
- deploy Lambda functions alongside CloudFront CDN
    - build more responsive applications
    - don't manager servers, the Lambda is deployed globally
    - customise the CDN content
    - pay only for what you use
- use Lambda to change CloudFront requests and responses
    - after CloudFront receives a request from a viewer (viewer request)
    - before CloudFront forwards the request to the origin (origin request)
    - after CloudFront receives the response from the origin (origin request)
    - before CloudFront forwards the response to the viewer (viewer response)
- can also generate responses to viewers without sending the request to the origin
- use cases:
    - security and privacy
    - dynamic web application at the edge
    - search engine optimisation
    - intelligently route across origins and data centers
    - bot mitigation at the edge
    - real-time image transformation
    - A/B testing
    - user authentication and authorisation
    - user prioritisation
    - user tracking and analytics
- for integrating the Lambda function with CloudFront distribution

## Asynchronous Invocations

- S3, SNS, CloudWatch Events...
- events are placed in an *Event Queue*
- Lambda attempts to retry on errors
    - 3 tries total
    - 1 minute wait after first, then 2 minute wait
- make sure the processing is *idempotent* (in case of retries)
- if function is retried, will see *duplicate log entries in CloudWatch logs*
- can define a DLQ - SNS or SQS - for failed processing (need correct IAM permissions)
- async invocations allow you to speed up the processing if you don't need to wait for the result
- services
    - *S3*
    - *SNS*
    - *CloudWatch Events/EventBridge*
    - CodeCommit
    - CodePipeline
    - etc.

*For the Developer Associate exam, the services in italics are the ones that matter*

### Hands on
- cannot get the result when running asynchronously, would need to check the logs
- cannot test from within the console, need to use CLI
- as it's asynchronous, we don't get the result back or care about it

## Lambda and Cloudwatch Events

- CRON or Rate EventBridge Rule runs at every set time and triggers a Lambda.
- CodePipeline EventBridge rule triggers a Lambda on code change.

## S3 and Event Notifications

- Events into S3
- events then fire something off to the Lambda
- S3 event notifications typically deliver events in seconds but can sometimes take a minute or longer
- if two writes are made to a single non-versioned object at the same time, it is possible that only a single event notification will be sent
- if you want to ensure that an event notification is sent for every successful write, you can enable versioning on your bucket
- invoke Lambda from S3 using 'Event Notification' configuration on S3

## Lambda Event Source Mapping

- applies to
    - Kinesis Data Stream
    - SQS & SQS FIFO
    - DynamoDB streams
- common denominator: records need to be polled from the source
- *Lambda function is invoked synchronously*

### Streams
- an event source mapping creates an iterator for each shard, which processes items in order
- start with new items, from the beginnning or from timestamp
- processed items aren't removed from the stream (other consumers can read them)
- *low traffic*: use batch window to accumulate records before processing
- you can process multiple batches in parallel
    - up to 10 batches per shard
    - in-order processing is still guaranteed for each partition key
- Error handling
    - *by default, if a Lambda function returns an error, the entire batch is reprocessed until the function succeeds, or the items in the batch expire*
    - to ensure in-order processing, processing for the affected shared is paused until the error is resolved
    - can configure the event source mapping to:
        - discard old events
        - restrict number of retries
        - split the batch or error (to workaround Lambda timeouts)
    - discarded events can go to a *destination*

### Queues
- for SQS event source mapping will poll with Long Polling
- specify *batch size*: 1 - 10
- recommended: set the queue visibility timeout to 6x the timeout of Lambda function
- to use a DLQ
    - set up on the SQS queue, not Lambda (is for asynchronous Lambda invocations only)
    - or use a Lambda destination for failures
- Lambda also supports in-order processing for FIFO queues, *scaling up to the number of active messages*
- for standard queues, items aren't necessarily processed in order
- Lambda scales up to process a standard queue as quickly as possible
- when an error occurs, batches are returned to the queue as individual items and might be processed in a different grouping than the original batch
- occasionally, the event source mapping might receive the same item from the queue twice, even if no function error occurred
- Lambda deletes items from the queue after they're processed successfully
- can configure the source queue to send items to a DLQ if they cannot be processed

### Lambda event mapper scaling
- Streams
    - one Lambda invocation per stream shard
    - if you use parallelisation, up to 10 batches processed per shard simultaneously
- SQS standard
    - Lambda adds 60 more instances per minute to scale up
    - up to 1000 batches of messages processed simultaneously
- SQS FIFO
    - messages with the same GroupId will be processed in order
    - the Lambda function scales to the number of active message groups


Make sure the Queue or Stream can be read from by the Lambda

## Lambda Destinations

- hard to see if asynchronous invocations succeeded or failed
- send the result of an asynchronous invocation to a destination so you can see what happened
- asynchronous invocations: can *define destinations for successful and failed events*
- possible destinations
    - SQS
    - SNS
    - Lambda
    - EventBridge Bus
- AWS recommends you use destinations instead of a DLQ (you can use both at same time)
- *Event source mapping*: for discarded event batches
    - SQS
    - SNS
    - can send events to a DLQ directly from SQS
- can have separate destinations for failures and successes

## Lambda Execution Roles
- grants the Lambda function permissions to AWS services and resources
- sample managed policies
    - AWSLambdaBasicExecutionRole - upload logs to CloudWatch
    - AWSLambdaKinesisExecutionRole - read from Kinesis
    - AWSLambdaDynamoDBExecutionRole - read from DynamoDB streams
    - AWSLambdaSQSQueueExecutionRole - read from SQS
    - AWSLambdaVPCAccessExecutionRole - deploy Lambda function in VPC
    - AWSXRayDaemonWriteAccess - upload trace data to X-Ray
- *when you use an **event source mapping** to invoke a function, Lambda uses the execution role to read event data*
- best practice: create one Lambda execution role per function

### Lambda Resource Based Polices
- use resource-based policies to give other accounts and AWS services permission to use your Lambda resources
- similar to S3 bucket policies for S3 buckets
- an IAM principal can access Lambda
    - if the IAM policy attached to the principal authorises it
    - or if the resource-based policy authorises it
- when as AWS service like S3 calls a Lambda function, the resource-based policy gives it access

### hands on
- every single Lambda function must have an IAM role
- resource based policies are automatically created as triggers etc. and are added to the Lambda

## Lambda Environment Variables

- environment variable: key value pair in 'string' form
- adjust the function behaviour without updating the code
- the environment variables are available to the code
- the Lambda service adds its own system environment variables as well
- helpful to store secrets (KMS encrypted)
- can be encrypted by the Lambda service key or your own CMK (custom master key)

## Lambda monitoring, logging and tracing

- CloudWatch logs
    - Lambda execution logs stored in CloudWatch logs
    - *make sure Lambda function has an execution role with an IAM policy that authorises writes to CloudWatch logs*
- CloudWatch metrics
    - Lambda metrics displayed in CloudWatch metrics
    - invocations, durations, concurrent executions
    - error count, success rates, throttles
    - asynchronous delivery failures
    - iterator age (Kinesis and DynamoDB streams)
- tracing with X-Ray
    - enable with Lambda config (*active tracing*)
    - runs X-Ray daemon for you
    - use X-Ray SDK in code
    - ensure Lambda function has a correct IAM execution role
        - managed policy is called AWSXRayDaemonWriteAccess
    - environment variables to communicate with X-Ray
        - _X_AMZN_TRACE_ID: contains the tracing header
        - AWS_XRAY_CONTEXT_MISSING: by default, LOG_ERROR
        - AWS_XRAY_DAEMON_ADDRESS: X-Ray Daemon IP_ADDRESS:PORT

## Lambda in VPC
- by default, Lambda functions are launched outside your own VPC (in an AWS-owned VPC), therefore cannot access resources in your VPC
- to deploy in own VPC
    - must define the VPC id, the subnets and the security groups
    - Lambda will create an ENI (Elastic Network Interface) in your subnets
    - Lambda will access your resources through the ENI
    - *AWSLambdaVPCAccessExecutionRole*
- a Lambda in your VPC does not have access to the internet
- *deploying a Lambda function in a public subnet does not give it internet access or a public IP*
- deploying a Lambda function in a private subnet gives it internet access if you have a *NAT Gateway/Instance*
- note: CloudWatch logs work even without an endpoint or NAT gateway

EXAM: Lambda VPC internet access questions expected
*Lambda connection to the internet through a VPC is expected for the Developer Associate exam.*

### hands on
- creating a security group for the Lambda function
- doesn't have internet access from VPC unless the VPC provides internet access. 
- to provide internet access route outbound traffic to a NAT gateway in a public subnet

## Lambda Function Performance

- RAM:
    - from 125MB to 10GB in 1MB increments
    - the more RAM you add, the more vCPU credits you get
    - at 1792 MB, Lambda function has the equivalent of one full vCPU
    - after 1792 MB, you get more than one CPU and need to use multi-threading in code to benefit from it
- *if application is CPU bound (compute heavy), increase RAM*
- *timeout*: default 3 seconds, max 900 seconds (15 mins)

*A common exam question is that if the application is compute heavy, then respond by increasing RAM.*

- Execution context
    - execution context is a temporary runtime environment that intitialises any external dependencies of Lambda code
    - great for database connections, HTTP clients, and SDK clients
    - execution context maintained for some time in anticipation of another Lambda function invocation
    - next function invocation can 're-use' the context at execution time and save time in initialising connections and objects
    - the execution context include the /tmp directory

Every time a Lambda is called, it is expensive to get the database connection by initialising it in the handler. The recommendation is to establish the database connection once and re-use it across invocations/handling to reduce time and complexity

- /tmp space
    - if a Lambda needs to download a big file to work or need disk space to perform operations it can use the /tmp directory
    - max size is 512MB
    - directory content remains when the execution context is frozen, providing a transient cache that be used for multiple invocations (helpful to checkpoint work)
    - for permanent persistence of objects, use S3

### hands on
- no way to change CPU separately from RAM
- test timeouts using a Thread.Sleep 

## Lambda Concurrency and Throttling

- concurrency limit: max 1000 concurrent executions
- can set a *reserved concurrency* at the function level
- each invocation over the concurrency limit will trigger a *throttle*
- throttle behaviour:
    - if synchronous invocation: return ThrottleError - 429
    - if asynchronous invocation: retry automatically and then go to DLQ
- if you need a higher limit, open a support ticket

### Concurrency Issue
- concurrency limit applies to all Lambdas in an account at the same time, so if one Lambda is runnning 1000 executions at once, the rest of the functions will fail

#### Asynchronous Concurrency:
- if a function doesn't have enough concurrency available to process all events, additional requests are throttled
- for throttling error and system errors, Lambda returns the event to the queue and attempts to run the function again for up to 6 hours
- the retry interval increases exponentially from 1 second after the first attempt to a max of 5 minutes

#### Cold starts & provisioned concurrency
- *cold start*
    - new instance: code is loaded and code outside the hanlder run (init)
    - if init is large, process can take some time
    - first request served by new instances has higher latency than the rest
- *provisioned concurrency*
    - concurrency is allocated before the function is invoked (in advance)
    - cold start never happens and all invocations have low latency
    - application auto scaling can manage concurrency (schedule or target utilisation)
    - cost associated with provisioned concurrency
Note: cold starts in VPC have been reduced in late 2019

## Lambda Function Dependencies

- *need to install packages alongside code and zip it together*
- upload the *zip* straight to Lambda if less than 50MB, otherwise upload to S3 first
- Native libraries work: they need to be compiled on Amazon Linux
- AWS SDK comes by default with every Lambda function

### hands on

- Lambda folder needs an index file in it.
- need to make sure Lambda has access to dependencies
- install the packages that you need locally
- make sure you've got the proper permissions for a project on the local machine, as well as the permissions required by the Lambda in its IAM role

## Lambda and CloudFormation

- *Inline*
    - inline within the CloudFormation template
    - inline functions are very simple
    - use the Code.ZipFile property
    - you cannot include function dependencies with inline functions
- S3 
    - store the Lambda function in S3
    - must refer to the S3 zip location in the CloudFormation code
        - S3Bucket
        - S3Key: full path to zip
        - S3ObjectVersion: if a versioned bucket
    - *if you update the code in S3, but don't update S3Bucket, S3Key or S3ObjectVersion, CloudFormation won't update your function*
- Lambda and CloudFormation - through multiple accounts
    use a bucket policy to allow CloudFormation in other accounts to access in the S3 bucket

## Lambda Layers
- create Custom Runtimes
    - Eg. C++ or Rust
- externalise dependencies to re-use them
    - application package 
    - external dependencies are stored in another Lambda layer for use across Lambda functions
- AWS has some layers that are already available for you to use
- then get to import libraries/dependencies without being packed as dependencies directly in the code

## Lambda Container Images

- deploy Lambda functions as container images of up to 10GB from ECR
- pack complex or large dependencies in a container
- base images are available for Python, Node.js, .NET, Go, Ruby
- can create your own image as long as it implements the *Lambda Runtime API*
- test the containers locally using the *Lambda Runtime Interface Emulator*
- unified workflow to build apps

## Lambda Versions and Aliases

### Versions
- working on Lambda, work on $LATEST
- when ready to publish a Lambda function, we create a version
- versions are immutable
- versions have increasing version numbers
- versions get their own ARN
- version = code + configuration (nothing can be changed - immutable)
- each version of the Lambda function can be accessed

### Aliases
- 'pointers' to Lambda function versions
- can define 'dev', 'test', 'prod' aliases and have them point at different Lambda versions
- aliases are mutable
- enable blue/green deployment by assigning weights to Lambda functions
- enable stable configuration of event triggers/destinations
- aliases have their own ARNs
- *aliases cannot reference aliases*
- alias can point to '$LATEST' as a version 
- changing alias weight configuration allows you to provide two versions at the same time so that you can get some data on the success of changes
- aliases can be edited to then change what aliases run

## Lambda and CodeDeploy
- CodeDeply can help automate traffic shift for Lambda Aliases
- feature is integrated within the SAM framework
- strategies
    - *Linear*: grow traffic every X minutes until 100%.
        - Linear10PercentEvery3Minutes
        - Linear10PercentEvery10Minutes
    - *Canary*: try X percent then 100%
        - Canary10Percent5Minutes
        - Canary10Percent30Minutes
    - *AllAtOnce*: immediate
    - Can create pre and post traffic hooks to check health of Lambda function

## Lambda Limits

- Limits are *per region*
- *Execution*:
    - memory allocation: 128MB-10GB (1MB increments)
    - maximum execution time: 900 seconds (15 minutes)
    - environment variables (4KB)
    - disk capacity in the "function container" (in /tmp folder): 512MB
    - concurrency executions: 1000 (can be increased)
- *Deployment*
    - Lambda function deployment size (compressed .zip): 50MB
    - size of uncompressed deployment (code + dependencies): 250MB
    - can use the /tmp directory to load other files at startup
    - size of environment variables: 4KB

*It is important to know Lambda limits for the Developer Associate exam.*

## Lambda Best Practices
- perform heavy-duty work outside of your function handler
    - connect to databases outside of your function handler
    - initialise the AWS SDK outside of your function handler
    - pull in dependencies or datasets outside of your function handler
- use environment variables for:
    - database connection strings, S3 bucket, etc. - values you don't want in code
    - passwords, sensitive values - things that should be encrypted in KMS
- minimise deployment package size to runtime necessities
    - break down the function if need be
    - remember the AWS Lambda limits
    - use Layers where necessary
- avoid using recursive code, never have a Lambda function call itself
