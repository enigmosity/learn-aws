# Integration and Messaging

## Intro

Multiple applications will probably need to communicate, two common patterns of applicaition communications are
1. synchronous communication (app to app)
    - between applications can be problematic if a sudden spike in traffic
    - *need to suddenly do a whole bunch more work*
2. asynchronous communication/event based (app to queue to app)
    - *decouples applications*
    - using SQS: queue model
    - using SNS: pub/sub model
    - using kinesis: real-time streaming model
    - *services can scale separately from app*

## SQS - Simple Queuing Service

**Standard Queue**
- the oldest offering
- fully managed service, used to decouple apps 
- attributes
    - unlimited throughput, unlimited number of messages in the queue
    - default retention of messages: 4 days, max of 14 days
    - low latency (<10ms on publish and receive)
    - limitation of 256KB per message sent
- can have duplicate messages (at least once delivery) occasionally
- can have out of order messages (best effort ordering)

*For the exam, if decoupling is in the question, think of the **Standard Queue**.*

Producing messages
- produced to SQS using SDK (SendMessage API)
- the message is *persisted* in SQS until a consumer deletes it
- message retention: default 4 days, up to 14 days
- SQS standard: unlimited throughput

Consumers
- running on EC2, servers, or Lambda
- poll SQS for messages (receive up to 10 messages at a time)
- process the messages
- delete the messages using the DeleteMessage API from the queue
- can have multiple consumers
- consumers receive and process messages in parallel
- at least once delivery
- best-effort message ordering
- consumers delete messages after processing them
- can scale consumers horizontally to improve throughput of processing

SQS with ASG
- scale ASG using a CloudWatch alarm based on queue length to enable faster processing

*For the exam, it's important to know how to scale SQS with an ASG*

Use SQS to decouple between application tiers (eg. front end and backend)

### SQS Security
- Encryption
    - in-flight encryption using HTTPS API - default
    - at rest encryption using KMS keys - optional
    - client side encryption if the client wants to perform encryption/decryption itself
- access controls: IAM policies regulate access to the SQS API
- SQS access policices (similar to S3 bucket policies)
    - useful for cross-account access to SQS queues
    - useful for allowing other services to write to an SQS queue

### hands on
- message retention can be from 1 second to 14 days
- can send and receive messages within the console for easy playing with
- can fully purge all messages from the queue, not recommended in production use cases though

### SQS Queue Access Policies
- resource policies added directly onto SQS queue
- use cases: cross account access, publish S3 event notifications to SQS queue

### SQS Message Visibility Timeout (MVT)
- after a message is polled by a consumer, it becomes *invisible* to other consumers
- by default the MVT is 30 seconds
- the message has 30 seconds to be processed
- after MVT is over, the message is 'visible' to all consumers in the queue again to be processed
- not processed within MVT, it will be processed *twice*
- consumer should call the *ChangeMessageVisibility API* to get more time
- if MVT timeout is high, and consumer crashes, re-processing will take some time
- if MVT is too low, may get duplicate processing

### SQS dead letter queues
- can set a threshold for how many times a message can return back to queue after MVT fails to be met
- after *MaximumReceives* threshold is exceeded, message goes into a *dead letter queue* (DLQ)
- useful for debugging
- messages in DLQs expire, process or investigate them before they do
- *Redrive to source* 
    - feature to help consume messages in DLQ to understand what is wrong
    - when code is fixed, can redrive messages from DLQ back into the source queue in batches for processing

### Delay queues
- *delivery delay* variable
- delay a message so customers don't see it immediately by up to 15 minutes
- default is 0 seconds (no delay)
- can set default at queue level
- can override default on send using the *DelaySeconds* parameter

### Long polling
- when a consumer requests messages from the queue, it can opt to wait for messages to arrive if there are none in the queue
- *LongPolling decreases the number of API calls made to SQS while increasing the efficiency and latency of your application*
- wait time can be between 1s and 20s (20s is preferable)
- long polling is preferable to short polling
- can be enabled at the queue level or at the API level using *WaitTimeSeconds*

### Extended Client
- message size limit is 256KB
- use the SQS extended client (java library) to send larger messages
- uses S3 bucket to store large messages, send a smaller metadata message about the bucket to the queue, the consumer then reads the data from the S3 bucket at processing time

**Must know API**
- *CreateQueue* (MessageRetentionPeriod), *DeleteQueue*
- *PurgeQueue*: delete all messages in queue
- *SendMessage* (DelaySeconds), *ReceiveMessage*, *DeleteMessage*
- *MaxNumberOfMessages*: default 1, max 10 (for ReceiveMessage API)
- *ReceiveMessageWaitTimeSeconds*: long polling
- *ChangeMessageVisibility*: change the message timeout
- batch APIs for *SendMessage*, *DeleteMessage*, *ChangeMessageVisibility* help decrease costs

### FIFO Queues
- *First in first out*; ordering of messages in the queue
- limited throughput: 300 msg/s without batching, 3000 msg/s with batching
- exactly once send capability by removing duplicates
- messages are processed in order by the consumer
- de-duplication
    - interval is 5 minutes
    - two methods 
        1. *content-based deduplication*: SHA-256 hash of message body generated and then compared
            - removes duplicate messages when sent in a 5 minute window
        2. explicitly provide a *Message Deduplication ID*
- message grouping
    - specify the same value of *MessageGroupId* in FIFO queue, those messages are then processed by the same one consumer and all messages are in order
    - to get ordering at level of a subset of messages, specify different values for *MessageGroupId*
        - messages that share a common message group id will be in order within the group
        - each group id can have a different consumer (parallel processing)
        - ordering across groups is not guaranteed

## SNS

*For sending one message to many receivers.*

**Pub/Sub = Publish/Subscribe**

Send message to SNS topic, which will have many subscribers. Each subscriber will then receive the message.

- '*event producer*' only sends message to one SNS topic
- as many '*event receivers*' (subscriptions) as you want can listen to SNS topic notifications
- each subscriber to the topic will get all the messages (note: new feature to filter messages)
- up to 12,500,000 subscriptions per topic
- 100,000 topic limit
- integrates with lots of AWS services
    - many services can send information directly to an SNS topic
- supports FIFO topics
    - first in first out
    - ordering by message group id
    - same deduplication as SQS FIFO
    - *can only have SQS FIFO queues as subscribers*
    - limited throughput (same as SQS FIFO)

### To Publish:
- *topic publish* (using SDK)
    - create a topic
    - create a subscription (or many)
    - publish to the topic
- *direct publish* (for mobile app SDK)
    - create a platform application
    - create a platform endpoint
    - publish to the platform endpoint
    - works with Google GCM, Apple APNS, Amazon ADM, etc.

### SNS Security
- Encryption
    - in-flight encryption using HTTPS API - default
    - at rest encryption using KMS keys - optional
    - client side encryption if the client wants to perform encryption/decryption itself
- access controls: iam policies regulate access to SNS API
- SNS access policies (similar to S3 bucket policies)
    - useful for cross-account access to SNS topics
    - useful for allowing other services to write to an SNS topic

### SNS + SQS Fan out pattern
- push one message to SNS, receive in all subscriber SQS queues
- fully decoupled, no data loss
- SQS allows for: data persistence, delayed processing and retries of work
- ability to add more SQS subscribers over time
- make sure your SQS queue *access policy* allows for SNS to write to it
- example use case:
    - same combination of: *event type* and *prefix* you can only have one S3 event rule
    - want to send same S3 event to many SQS queues, use Fan-Out

### SNS FIFO + SQS FIFO Fan out pattern
- in case you need fanout + ordering + deduplication

### Message filtering
- JSON policy used to filter messages sent to SNS topic subscriptions
- if a subscription doesn't have a filter policy, it receives every message

### hands on

- if a FIFO topic, the name has to end with '.fifo'

## Kinesis

*It's important to know Kinesis in depth for the Developer Associate exam.*

- makes it easy to *collect, process and analyse* streaming data in real-time
- ingest real-time data such as: application logs, metrics, website clickstreams, IoT telemetry data, etc.
- *Data Streams*: capture, process and store data streams
- *Data Firehose*: load data streams into AWS data stores
- *Data Analytics*: analyse data streams with SQL or Apache Flink
- *Video Streams*: capture, process and store video streams

### Data Streams

- Way to stream data in systems
- data stream is made of *shards* which can be scaled
- shards need to be provisioned before use
- *producers*: produce records for the data stream
    - AWS SDK, Kinesis Producer Library (KPL), Kinesis Agent
- *record*: the data from a producer that is added to the data streams. Contains a *partition key* and *data blob up to 1 mb*
    - 1 MB/s or 1000 msg/s per shard
- *consumers*: record passed to consumer from the kinesis data stream
    - write your own: kinesis client library (KCL), AWS SDK
    - managed: lambda, kinesis data firehose, kinesis data analytics
- retention between 1 day and 1 year
- ability to reprocess (replay) data
- once data is inserted in Kinesis, it can't be deleted (immutability)
- data that shares the same partition go to the same shard (ordering)
- *capacity modes*
    - *provisioned mode*
        - you choose the number of shards provisioned, scale manually or using API
        - each shard gets 1MB/s in (or 1000 records/s)
        - each shard gets 2MB/s out (classic or enhanced fan-out consumer)
        - pay per shard provisioned per hour
    - *on demand mode*
        - no need to provision or manage the capacity
        - default capacity provisioned (4MB/s in or 4000 records/s)
        - scales automatically based on observed throughput peak during the last 30 days
        - pay per stream per hour and data in/out per GB
- security
    - control access/authorisation using IAM policies
    - encryption in flight using HTTPS endpoints
    - encryption at rest using KMS
    - can implement encryption/decryption of data on client side
    - VPC endpoints available for Kinesis to access within VPC
    - monitor API calls using CloudTrail

*Producers*
- puts data records into data streams
- data records consist of:
    - sequence number (unique per partition-key within shard)
    - partition key (must specify while put records into stream)
    - data blob (up to 1MB)
- producers:
    - AWS SDK: simple producer
    - Kinesis Producer Library (KPL): C++, Java, batch, compression, retries
    - Kinesis Agent: monitor log files
- write throughput: 1MB/s or 1000 records/s per shard
- *PutRecord API*
- using batching with PutRecords API to reduce costs and increase throughput
- hash the partition key for a single device to determine the shard to go to
    - use a highly distributed partition key to avoid 'hot partition'
- *ProvisionThroughputExceeded* exception: too much throughput
    - solutions:
        - use highly distributed partition key
        - retries with exponential backoff
        - increase shards (scaling)

*The ProvisionThroughputExceeded exception is important for the Developer Associate exam.*

*Consumers*
- get data records from data streams and process them
- custom consumer
    - *shared Classic fan-out consumer*
        - use GetRecords()
        - all consumers must share the available throughput, therefore you have a limitation of how much each consumer can consume
        - *pull model*:
            - low number of consuming applications
            - read throughput 2MB/sec per shard across all consumers
            - max 5 GetRecords API calls/s
            - latency ~200ms
            - minimise cost
            - consumers poll data from Kinesis using GetRecords API call
            - returns up to 10MB (then throttle for 5s) or up to 10000 records
    - *enhanced fan-out consumer*
        - use SubscribeToShard()
        - limit is per consumer per shard
        - *push model*:
            - multiple consuming applications for the same stream
            - 2MB/s per consumer per shard
            - latency ~70ms
            - higher cost
            - Kinesis pushes data to consumers over HTTP/2(SubscribeToShard API)
            - soft limit of 5 consumer applications (KCL) per data stream (default)
- Lambda
    - call GetBatch()
    - supports classic and enhanced fan-out consumers
    - reads records in batches
    - can configure *batch size* and *batch window*
    - if error occurs, Lambda retries until succeeds or the data expires
    - can process up to 10 batches per shard simultaneously

### Hands On
- use shard estimator to figure out how much infrastructure to provision if not using the auto scaling option

## Kinesis Client Library

- Java library that helps read records from a Kinesis Data Stream with distributed applications sharing the read workload
- each shard is read by only one KCL instance
- progress is checkpointed into DynamoDB (needs IAM access)
- track other workers and share the work amongst shards using DynamoDB
- KCL can run on EC2, Benstalk and on-premises
- records are read in order at the shard level
- versions
    - KCL 1.x (supports shared consumer)
    - KCL 2.x (supports shared & enhanced fan-out consumer)

### Kinesis Operation - Shard Splitting
- used to increase the stream capacity (1MB/s data in per shard)
- used to divide a 'hot shard'
- also increases cost
- old shard is closed and deleted once the data is expired
- no auto scaling (all manual)
- can't split into more than two shards in a single operation

#### Merging Shards
- decreasing the stream capacity and save costs
- can be used to group two shards with low traffic (cold shards)
- old shards are closed and will be deleted once the data is expired
- can't merge more than two shards in a single operation

## Kinesis Data Firehose
- takes records from producers, optionally transforms data, then batch writes to a destination, or straight to an S3 bucket as a backup
- fully managed service, no admin, auto scaling, serverless
- desinations: 
    - AWS: redshift (COPY through S3), S3, ElasticSearch
    - 3rd party partners
    - custom: send to any HTTP endpoint
- pay for data going through Firehose
- *near real time* EXAM
    - 60s latency minimum for non full batches
    - or min 32MB of data at a time
- supports many data formats, conversions, transformations, compression
- supports custom data transformations using AWS Lambda
- can send failed or all data to a backup S3 bucket

*AWS destinations are important for the Developer Associate exam.*

*For the Developer Associate exam, any time near real time data transformation is required, think Kinesis Data Firehose.*

### Data Streams vs Firehose
Data Streams:
- streaming service for ingestion at scale
- custom code (producer/consumer)
- real-time (~200ms)
- manage scaling (shard splitting/merging)
- data storage for 1 - 365 days
- supports replay capability

Firehose:
- load streaming data into S3/redshift/ES/3rd party/custom HTTP
- fully managed
- near real time (buffer time min. 60s)
- auto scaling
- no data storage
- doesn't support relay capability

### Hands On
- set up delivery streams
- convert record format allows you to transform the format if you want to
- buffer allows you to wait for a certain amount of information before delivering to the target, increase the size for efficiency or decrease for speed
- buffer interval is how long it waits before sending regardless of size
- doesn't take in old data, only data sent since Firehose was set up

## Kinesis Data Analytics

- for a SQL application
- takes data from data streams or firehose
- then uses SQL statements in the data analytics to complete analysis
- Kinesis Data Analytics then sends the information off to a sink which could be another Data Stream or Firehose
- perform real-time analytics on Kinesis Streams using SQL
- fully managed, no servers to provision
- auto scaling
- pay for actual consumption rate
- can create streams out of the real-time queries
- use cases:
    - time series analytics
    - real-time dashboards
    - real-time metrics

### SQS vs SNS vs Kinesis

SQS
- consumers 'pull data'
- data deleted after consumption
- as many consumers as you want
- no need to provision throughput
- ordering only guaranteed on FIFO queues
- individual message delay capability

SNS
- push data to many subscribers
- up to 12,500,000 subscribers
- data not persisted (lost if not delivered)
- publish/subscribe
- up to 100,000 topics
- no need to provision throughput
- integrates with SQS for fan-out architecture pattern
- FIFO capability for SQS FIFO

Kinesis
- Standard: pull data
    - 2MB per shared
- Enhanced-fan out: push data
    - 2MB per shard per consumer
- possibility to replay data
- meant for real-time big data, analytics and ETL
- ordering at the shard level
- data expires after X days
- provisioned mode or on-demand capacity mode

### Data ordering for Kinesis vs SQS FIFO

Kinesis
- use a 'partition key' value for each message to ensure they end up in order
- messages with the same partition key will always go to the same shard
- data will then be in order at the shard level
- better for dynamic numbers of consumers etc.

SQS
- use FIFO to guarantee FIFO
- can only have 1 FIFO queue
- do not use a group ID if only want to use one consumer and ensure messages processed in order sent
- to scale consumers and/or group messages together use a group id
- one consumer per groupid, then they're dealt with in order on the consumer
- better with a static set of requirements