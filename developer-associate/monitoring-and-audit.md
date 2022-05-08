# Monitoring and Audit

## Monitoring Overview

Users only care about:
- the application working
- application latency
- outages

*CloudWatch*:
- metrics
- logs
- events
- alarms

*X-Ray*:
- troubleshooting application performance and errors
- distributed tracing of microservices

*CloudTrail*:
- internal monitoring of API calls being made
- audit changes to AWS resources by users

## CloudWatch 

### Metrics
- provides metrics for *every* service in AWS
- *metric* is a variable to monitor
- metrics belong to *namespaces*
- *dimension* is an attribute of a metric
- up to 10 dimensions per metric
- metrics have *timestamps*
- can create a CloudWatch dashboard of metrics

EC2 detailed monitoring
- EC2 instance metrics report metrics every 5 minutes
- with detailed monitoring (for a cost), you can report data every minute
- use detailed monitoring if you want to scale ASG faster
- free tier allows 1 set of detailed monitoring metrics
- NOTE: EC2 Memory usage is by default not pushed as a metric, it must be pushed from inside the instance as a custom metric

High Resolution is only relevant for Custom Metrics. When you publish a Custom Metric, you can define it as either standard resolution or high resolution. You can read and retrieve High-Resolution Custom Metrics at 1 second, 5 seconds, 10 seconds, 30 seconds, or any multiple of 60 seconds.

*Knowing these limits is important for the Associate Developer exam.*

#### Custom Metrics

- define your own metrics
- use the *PutMetricData* API call
- ability to use dimensions (attributes) to segment metrics
- metric resolution (*StorageResolution* API parameter):
    - *standard*: 1 minute
    - *high resolution*: 1/5/10/30 second(s) - higher cost
- **accepts metric data points two weeks in the past and two hours in the future (make sure to configure EC2 instance time correctly)**

*When metric data points are accepted from is a common Associate Developer exam question.*

##### hands on

- you specify a timestamp for every metric that is then used, can use the `metrics.json` file, or just use an API command and specify the value

### Logs

- *log groups*: an arbitrary name, usually representing an application
- *log stream*: instances within application/log files/containers
- can define *log expiration policies*, default is to never expire. 
- retention policy is defined at the log group level
- can send logs to
    - S3 (exports)
    - Kinesis Data Streams
    - Kinesis Data Firehose
    - Lambda
    - ElasticSearch
- Sources:
    - SDK, CloudWatch Logs Agent, CloudWatch Unified Agent
    - Beanstalk: collection of logs from application
    - ECS: collection from containers
    - Lambda: from function logs
    - VPC Flow Logs: VPC specific logs
    - API Gateway
    - CloudTrail based on filter
    - Route53: log DNS queries

#### Metric Filter & Insights
- logs can use filter expressions, eg.
    - find specific IP
    - count occurences of ERROR in logs
- metric filters can be used to trigger CloudWatch Alarms
- CloudWatch Log Insights can be used to query logs and add queries to CloudWatch dashboards

##### S3 Export
- log data can take up to 12 hours to become available for Export
- API call: *CreateExportTask*
- not near real-time or real-time, use Log Subscriptions for that

##### Log Subscriptions
- filter on top of CloudWatch logs
- allows near real-time sharing of logs


log aggregation multi-account/region is possible too

### Agent & Logs Agent

- by default, no logs from EC2 instances go to CloudWatch
- need to run a CloudWatch agent on EC2 to push the log files you want
- make sure IAM permissions are correct
- CloudWatch log agent can be setup on-premises too
- for virtual servers

#### logs agent
- old version
- only send logs to CloudWatch logs

#### Unified agent
- collect additional system-level metrics such as RAM, processes, etc.
- collect logs to send to CloudWatch Logs
- centralised configuration using SSM Parameter store

### Metrics
- collected directly on Linux server/EC2 instance
- example metrics
    - CPU (active, guest, idle, system, user, steal)
    - Disk metrics (free, used, total), Disk IO (writes, reads, bytes, iops)
    - RAM (free, inactive, used, total, cached)
    - Netstat (# of TCP/UDP connections, net packets, bytes)
    - Processes (total, dead, bloqed, idle, running, sleep)
    - Swap Space (free, used, used %)
- reminder: *out-of-the box metrics for EC2* - disk, cpu, network (high level)

#### Metric Filters

- metric filers can be used to filter expressions and trigger alarms
- *filters do not retroactively filter data - filters only publish the metric data points for events that happen after the filter was created*

### Alarms

- used to trigger notifications for any metric
- various options (sampling, %, max, min, etc.)
- States:
    - OK (not triggered)
    - INSUFFICIENT_DATA
    - ALARM
- period:
    - length of time in seconds to evaluate the metric
    - high resolution custom metrics: 10s, 30s or multiple of 60s

*Alarm periods are a common question in the Developer Associate exam*

#### Alarm targets
- Stop, Terminate, Reboot, or Recover EC2 instance
    - EC2 instance recovery:
        - status check
            - *instance status*: check EC2 VM
            - *system status*: check underlying hardware
            - *recovery*: same private, public, Elastic IP, metadata, placement group
- trigger auto scaling action
- send notification to SNS

*Good to know:*
- alarms can be created based on Log Metric Filters
- to test alarms and notifications, set the alarm state with CLI
    *aws cloudwatch set-alarm-state --alarm-name "myalarm" --state-value ALARM --state-reason "testing purposes"*
- The number of EC2 instances in an ASG cannot go below the minimum capacity, even if the CloudWatch alarm would in theory trigger an EC2 instance termination.

### Events

- *event pattern*: intercept events from AWS services (sources)
    - can intercept any API call with CloudTrail integration
- schedule or Cron
- a JSON payload is created from the event and passed to a target
    - *compute*: lambda, batch, ECS task
    - *integration*: SQS, SNS, Kinesis Data Streams, Kinesis Data Firehose
    - *orchestration* Step Functions, CodePipeline, CodeBuild
    - *maintenance*: SSM, EC2 actions

## EventBridge
- next evolution of cloudwatch events
- *default event bus*: generated by AWS services
- *partner event bus*: receive events from SaaS services or applications
- *custom event buses*: for your own applications
- event buses can be accessed by other AWS accounts
- can *archive events*(all/filter) sent to an event bus (indefinitely or for a set period)
- ability to *replay archived events*
- *Rules*: how to process the events

### Schema Registry
- EventBridge can analyse the events in your bus and infer the *schema*
- the *schema registry* allows you to generate code for your application, that will know in advance how data is structured in the event bus
- schema can be versioned

### Resource-based policy
- manage permissions for a specific EventBus
- use case: aggreate all events from you AWS org in a single AWS account or region

EventBridge vs CloudWatch
- EventBridge builds on and extends CloudWatch Events
- uses the same service API and endpoint, and the same underlying service infrastructure
- EventBridge allows extensions to add event buses for your custom applications and your third-party SaaS applications
- EventBridge has the schema registry capability
- EventBridge has a different name to mark its new capabilities
- over time, CloudWatch Events name will be replaced with EventBridge

## X-Ray

- provides visual analysis of applications
- *advantages*
    - troubleshooting performace (bottlenecks)
    - understand dependencies in a microservice architecture
    - pinpoint service issues
    - review request behaviour
    - find errors and exceptions
    - check SLAs
    - find throttling
    - identify impacted users
- compatability
    - Lambda
    - Elastic Beanstalk
    - ECS
    - ELB
    - API Gateway
    - EC2 instance or any application server (including on-premises)

*Leverages tracing*
- tracing end to end is a way to follow a `request`
- each component dealing with the request adds its own `trace`
- tracing is made of *segments* and *subsegments*
- annotations can be added to traces to provide extra information
- ability to trace
    - every request
    - sample request (eg. % for an example or a rate per min)
- security
    - IAM for authorisation
    - KMS for encryption at rest

*For the Developer Associate exam, it is important to know how to enable x-Ray.*

To enable:
1. code must import AWS X-Ray SDK
2. install X-Ray daemon or enable X-Ray AWS integration
    - works as a low level UDP packet interceptor
    - Lambda/other AWS services already run the daemon for you
    - each application must have the IAM rights to write data to X-Ray

The magic:
- X-Ray service collects data from all different services
- service map is computed from all segments and traces
- graphical, so even non-technical people can help troubleshoot

*Troubleshooting*:
- not working on EC2:
    - ensure the EC2 IAM role has the proper permissions
    - ensure the EC2 instance is running the X-Ray daemon
- enable on Lambda
    - ensure IAM execution role with proper policy (*AWSX-RayWriteOnlyAccess*)
    - ensure X-Ray is imported in the code

### hands on

- will need to write some code that allows the writing of segments into traces.
- use .ebextenstions/xray-daemon.config for enabling X-Ray on Elastic Beanstalk

### Instrumentation in code
- *instrumentation* means the measure of a product's performance, to diagnose errors, and to write trace information
- to instrument your application code, use the X-Ray SDK
- many SDK require only configuration changes
- can modify application code to customise and annotate data that the SDK sends to X-Ray, using *interceptors, filters, handlers, middleware*, etc.

### Concepts
- *segments*: each application / service will send them
- *subsegments*: when you need more details in your segment
- *trace*: segments collected together to form an end-to-end trace
- *sampling*: a portion of all the requests, to decrease the amount of requests sent to X-Ray and therefore reduce cost
- *annotations*: key-value pairs used to *index* traces and use with *filters*
- *metadata*: key value pairs, *not indexed*, not used for searching
- the daemon/agent has a configuration to send traces cross account
    - make sure IAM permissions are correct, as the agent will assume the role
    - allows a central account for all application tracing

### Sampling rules
- rules control the amount of data that you record
- can modify sampling rules without changing code
- by default X-Ray SDK records the first request *each second* and **five percent** of any additional requests
- *one request per second is the **reservoir***, which ensures that at least one trace is recorded each second as long as the service is serving requsts
- **five percent is the *rate*** at which additional requests beyond the reservoir size are sampled

#### Custom Sampling Rules:
- can create your own rules with the reservoir and rate
- on change, you don't need to change anything in your applications, the daemon automatically knows how to supply the required information

### APIs

#### Write API
- used by the X-Ray daemon
- *PutTraceSegments*: uploads segment documents to AWS X-Ray
- *PutTelemetryRecords*: used by the daemon to upload telemetry
- *GetSamplingRules*: retrieve all sampling rules (to know what/when to send)
- *GetSamplingTargets & GetSamplingStatisticSummaries*: advanced
- daemon needs to have an IAM policy authorising correct API calls to function correctly

#### Read APIs
- *GetServiceGraph*: retrieves the main graph
- *BatchGetTraces*: retrieves a list of traces specified by ID, each trace is a collection of segment documents that originates from a single requeset
- *GetTraceSummaries*: retrieves IDs and annotations for traces available for a specified time frame using an optional filter - to get full traces, pass the trace IDs to *BatchGetTraces*
- *GetTraceGraph*: retrieves a service graph for one or more specific trace IDs

*For the Developer Associate exam, be prepared to talk about which API call to use and why.*

### With Beanstalk
- Beanstalk platforms include the X-Ray daemon
- can run the daemon by setting an option in the Beanstalk console or a with a configuration file
- make sure the EC2 instance profile has the correct IAM permissions so the daemon can operate
- make sure application code is intrumented with X-Ray SDK
- NOTE: daemon is not provided for multi-container docker

### ECS + X-Ray Integration Options

1. *ECS cluster, X-Ray container as a Daemon* - one daemon runs on each EC2 instance alongside the other containers
2. *ECS cluster, X-Ray container as a '**sidecar**'* - daemon runs side to side per application container, one daemon per container
3. *Fargate cluster, X-Ray container as a '**sidecar**'*, daemon runs as a sidecar on each task
    - *port mapping*: container port: 2000, protocol: udp
    - *environment variable*: AWS_XRAY_DAEMON_ADDRESS, so SDK knows where daemon exists, must include port number (2000)
    - links must include 'xray-daemon'

## CloudTrail
- *provides governance, compliance and audit for AWS account*
- enabled by default
- get a *history of events/API calls made within aws account* by:
    - console
    - SDK
    - CLI
    - AWS services
- can put logs from CloudTrail into CloudWatch logs or S3 if required to keep logs for more than 90 days
- *a trail can be applied to all regions (default) or a single region*
- if a resource is deleted in AWS, investigate CloudTrail first

### Events

- management events
    - operations that are performed on resources in your AWS account
    - *by default, trails are configured to log management events*
    - can separate *read events* from *write events*
- data events
    - *by default, data events are not logged because they're high volume operations*
    - *S3 object-level activity*: can separate Read and Write events
    - Lambda function execution acitivity (*invoke api*)
- CloudTrail insight events
    - enable CloudTrail insights to detect unusual activity in your account
        - innacurate resource provisioning
        - hitting service limits
        - bursts of AWS IAM actions
        - gaps in periodic maintenance activity
    - CloudTrail insights analyses *write* events to detect unusual patterns
        - anomalies appear in the CloudTrail console
        - event is sent to S3
        - EventBridge event is generated (for automation needs)
    - got to pay for insights


#### Event retention
- stored for 90 days in CloudTrail
- to keep events beyond 90 days, log them to S3 and use Athena to analyse

### hands on

CloudTrail can take up to 15 minutes for events to appear


## CloudTrail vs CloudWatch vs X-Ray 

CloudTrail:
- audit API calls made by users/services/AWS console
- useful to detect unauthorised calls or root cause of changes
CloudWatch:
- metrics over time for monitoring
- logs for storing application logs
- alarms to send notifications in case of unexpected metrics
X-Ray:
- automated trace analysis and central service map visualisation
- latency, errors and fault analysis
- request tracking across distributed systems