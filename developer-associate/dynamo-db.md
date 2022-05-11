# DynamoDB

## Overview

*Traditional Architecture*
- leverage RDBMS databases
- databases have the SQL query language
- strong requirements about how the data should be modeled
- ability to do query joins, aggregations, complex computations
- vertical scaling (getting a more powerful CPU/RAM/IO)
- horizontal scaling (increasing reading capability by adding EC2/RDS read replicas)

*NoSQL Databases*
- non-relational databases which are *distributed*
- include MongoDB, DynamoDB, etc.
- do not support query joins (or limited support)
- all data needed for a query is present in one row
- do not perform aggregations such as "SUM", "AVG", etc.
- *scale horizontally*

*No "right or wrong" for NoSQL vs SQL, just require data to be modeled differently and think about user queries differently*

## DynamoDB
- fully managed, highly available with replication across multiple availability zones
- *NoSQL* - not a relational database
- scales to massive workloads, is a distributed database
- millions of requests per second, trillions of rows, 100s of TB of storage
- fast and consistent in performance (low latency on retrieval)
- integrated with IAM for security, authorisation and administration
- enables event driven programming with DynamoDB streams
- low cost and auto-scaling capabilities
- standard and infrequent access (IA) table class

### Basics
- made of *tables*
- each table has a *primary key* which must be decided at table creation time
- each table can have an infinite number of items (rows)
- each item has *attributes* which can be added over time (can be null values)
- maximum size of an item is 400KB
- supported data types:
    - *scalar types*: string, number, binary, boolean, null
    - *document types*: list, map
    - *set types*: string set, number set, binary set

### Primary Keys
EXAM: how to choose a primary key

Option 1: Partition key (HASH)
- partition key must be unique for each item
- partition key must be 'diverse' so that the data is distributed

Option 2: Partition key + Sort key (HASH + RANGE)
- *combination* must be unique for each item
- data is grouped by partition key

Exercise: what is the best partition key to maximise data distribution?

*For the Developer Associate exam, it is important to know how to select a primary key. Always select the partition key with the highest cardinality.*

### hands on
- class
    - *standard*: for most use cases. reads and writes are the dominant table cost
    - *standard IA (Infrequently Accessed)*: infrequently accessed data. storage is the dominant table cost
- can view and create items in the console, through a form or a json document
- when attributes are added over time, that's perfectly fine as complete data is not required to create an object - *only need the primary key (and maybe sort key) to exist*
- cannot specify that non-primary key attributes cannot be null.

## Read/Write Capacity Modes

Control how you manage your tables' capacity
- *provisioned mode* (default)
    - specify the number of reads/writes per second
    - need to plan capacity beforehand
    - pay for *provisioned* read and write capacity units
    - you *must* have provisioned read and write capacity units
    - *read capacity units* (RCU): throughput for reads
    - *write capacity units* (RCU): throughput for writes
    - RCU and WCU are decoupled so they can be changed independently
    - there is the option to setup *auto-scaling* of throughput to meet demand
    - throughput can be exceeded temporarily using *burst capacity*
    - if burst capacity has been consumed, you will get a *ProvisionedThroughputExceededException*, it is then advised to do an *exponential backoff* retry
- *on-demand mode*
    - read/writes automatically scale up/down with your workloads
    - no capacity planning needed
    - pay for what you use, more expensive ($$$)
    - unlimited WCU and RCU, no throttling, more expensive
    - charged for reads/writes that you use in terms of RRU and WRU
    - *Read Request Units* (RRU): throughput for reads (same as RCU)
    - *Write Request Units* (WRU): throughput for writes (same as RCU)
    - on-demand is 2.5 times more expensive than provisioned capacity
    - use cases: unknown workloads and unpredicatable application traffic
- can switch between different modes once every 24 hours

### WCU (Write Capacity Unit)
- one *WCU* represents one write per second for an item up to 1KB in size
- if an item is larger than 1 KB, more WCUs are consumed
- WCUs required = number of items per second * (item size in KB / 1KB)
- *item size always rounded up to the upper KB size*
- need to get items per second

### RCU (Read Capacity Unit)
- one RCU represents *one Strongly Consistent Read* per second, or *two Eventually Consistent Reads* per second, for an item up to 4 KB in size
- if items are larger than 4KB, more RCUs are consumed
- SCR = Strongly Consistent Read, ECR = Eventually Consistent Read
- number of SCR per second * (item size in KB / 4KB)
- (number of ECR per second / 2) * (item size in KB / 4KB)
- must round up item size to the nearest multiple of 4

*For the Developer Associate exam, you will be asked to remember and use the formulas for both RCUs and WCUs to determine the required capacity provisioning.*

#### Strongly Consistent Read
- if read just after a write, will get the correct data
- set the *ConsistentRead* parameter to *True* in API calls (GetItem, BatchGetItem, Query, Scan)
- consumes twice the RCU

#### Eventually Consistent Read (default)
- if read just after a write, it's possible we'll get some stale data because of replication
- TODO: what is the behaviour of AWS that results in ECR and SCR being different?

### Partitions Internal
- partitions are copies of a table living on different servers with different data stored in them
- partition keys go through a hashing algorithm to determine which partition they go to
- to compute the number of partitions required
    - number of partitions by capacity = (RCUs total / 3000) + (WCUs total / 1000)
    - number of partitions by size = total size / 10GB
    - number of partitions = ceil(max(# of partitions by capacity, number of partitions by size))
- the partition formula is not required for the Developer Associate exam
- *WCUs and RCUs are spread evenly across partitions*

### Throttling
- if we exceed provisioned RCUs or WCUs, we get a *ProvisionedThroughputExceededException*
- reasons
    - hot keys: one partition key is being read too many times
    - hot partitions
    - very large items, remember RCU and WCU depends on size of items
- solutions:
    - *exponential backoff* when exception is encountered (already in SDK)
    - *distribute partition keys* as much as possible
    - if an RCU issue, *DynamoDB Accelerator* (DAX) can be used

### hands on
- capacity is in additional settings
- can switch between capacity modes as required
- provisioning can change over time
- the console will tell you what the RCU and WCU are based on your settings
- autoscaling on table capacity with the target utilisation will then do its best to make things fit as expected

## Basic APIs

*The Developer Associate exam will refer to the API calls required for DynamoDB.*

Writes
- *PutItem*:
    - creates a new item or fully replaces an old item if they have the same primary key
    - consumes WCUs
- *UpdateItem*
    - edits an existing item's attributes or adds a new item if it doesn't exist
    - can be used to implement *Atomic Counters* - a numeric attribute that's unconditionally incremented
- *ConditionalWrites*
    - accept a write/update/delete only if conditions are met, otherwise returns an error
    - helps with concurrent access to items
    - has no performance impact

Reads
- *GetItem*
    - read item based on primary key
    - primary key can be HASH or HASH+RANGE
    - Eventually Consistent Read (default)
    - option to use Strongly Consistent Reads (more RCU, might take longer)
    - *ProjectExpression* can be specified to retrieve only certain attributes
- *Query* returns items based on:
    - *KeyConditionExpression*
        - Partition key value (*must be `=` operator*) - required
        - Sort key value (`=`,`<`,`<=`,`>`,`>=`, `between`, `begins with`) - optional
    - *FilterExpression*
        - additional filtering after the Query operation but before data is returned to you
        - use only with non-key attributes (does not allow HASH or RANGE attributes)
    - returns:
        - number of items specified in *Limit*
        - or up to 1MB of data
    - ability to do pagination on results
    - can query a table, a Local Secondary Index, or a Global Secondary Index
- *Scan* the entire table and then filter out data (which is quite inefficient)
    - returns up to 1MB of data - use pagination to keep reading
    - consumes a lot of RCU
    - limit impact using *Limit* or reduce the size of the result and pause
    - for faster performance use *ParallelScan*
        - multiple workers scan multiple data segments at the same time
        - increases the throughput and RCU consumed
        - limit the impact of parallel scans just like you would for Scans
    - can use *ProjectExpression* and *FilterExpression* (no changes to RCU)
- *DeleteItem*
    - delete an individual item
    - ability to perform a conditional delete
- *DeleteTable*
    - delete a whole table and all its items
    - much quicker than calling *DeleteItem* on all items, you would then re-create an empty table
    - *The Developer Associate exam is likely to ask you the more efficient method to delete all items in the table than item by item. The answer is to delete the entire table, and the recreate it empty.*

Batch operations
- allow you to save latency by reducing the number of API calls
- operations are done in parallel for better efficiency
- path of a batch can fail; in which case we need to try again for the failed items
- *BatchWriteItem*
    - up to 25 *PutItem* and/or *DeleteItem* in one call
    - up to 16MB of data written, up to 400KB of data per item
    - can't update items (use *UpdateItem*)
- *BatchGetItem*
    - return items from one or more tables
    - up to 100 items, up to 16 MB of data
    - items are retrieved in parallel to minimise latency

### hands on
- creating an item is a put item
- editing item is an update item API call
- filtering of scan is done client side
- query filtering is also done client side

## DynamoDB Indexes

### Local Secondary Indexes (LSI)
- *alternative sort key* for your table (same *partition key* as that of the base table)
- sort key consists of one scalar attribute (string, number, or binary)
- up to 5 LSI per table
- *must be defined at table creation time*
- *Attribute Projections*: can contain some or all the attributes of the base table (KEYS_ONLY, INCLUDE, ALL)

### Global Secondary Index (GSI)
- *alternative primary key (HASH or HASH+RANGE)* from the base table
- speed up queries on non-key attributes
- index key consists of scalar attributes (string, number or binary)
- *Attribute Projections*: can contain some or all of the attributes of the base table (KEYS_ONLY, INCLUDE, ALL)
- must provision RCUs and WCUs for the index
- *can be added/modified after table creation*

### Indexes and Throttling
- GSI:
    - *if the writes are throttled on the GSI, then the main table will be throttled*
    - even if the WCU on the main tables are fine
    - choose your GSI partition key carefully
    - assign your WCU capacity carefully
- LSI:
    - uses the WCUs and RCUs of the main table
    - no special throttling considerations

*GSI throttling is important to know for the Developer Associate exam.*

## PartiQL

- uses a SQL-like syntax to manipulate DynamoDB tables
- supports some (but not all) statements
    - INSERT
    - UPDATE
    - SELECT
    - DELETE
- supports batch operations

## Optimistic Locking

- feature called *Conditional Writes*
- strategy to ensure an item hasn't changed before you update/delete it
- each item has an attribute that acts as a *version number*, the version number is checked that it is the value expected before the item is altered

*The Developer Associate exam will likely test on Optimistic Locking.*

## DAX (DynamoDB Accelerator)

- fully managed, highly available, seamless in-memory cache for DynamoDB
- microseconds latency for cached reads and queries
- doesn't require application logic modification (compatible with existing DynamoDB APIs)
- solves the 'Hot Key' problem (too many reads)
- 5 minute TTL for cache (default)
- up to 10 nodes in the cluster
- multi-availability zone (3 nodes minimum recommended for production uses)
- secure (encryption at rest with KMS, VPC, IAM,CloudTrail, etc.)

*A likely question in the Developer Associate exam is to determine which caching service (ElastiCache or DAX) to use for a given scenario.*

DAX:
- individual objects cache
- query and scan cache
ElastiCache: store an aggregation result

DAX Types:
- *t-type*: baseline capacity with the ability to burst, recommended for use cases with lower throughput
- *r-type*: node allocated with fixed resources, an always ready capacity

### hands on
- create a cluster
- *size*: how many nodes you want. 3 nodes ensures multi-availability zone
- not in the free tier
- to access cluster from an application, enable inbound access on port 8111 for the security group, or 9111 if encrypted-in-transit
- can limit to only some tables
- parameters determine how to long cache for etc.
    - default: item 5 min TTL, query 5min TTL
- *maintenance window*: for patching etc. 
    - either you set a specific window, or DAX will pick a time to complete maintenance
- *cluster endpoint*: endpoint for applications to hit to use the cluster
- can add more nodes over time, but cannot change the node types

## Streams

- ordered stream of item-level modifications (create/update/delete) in a table
- stream records can be
    - sent to Kinesis Data Streams
    - read by Lambda Functions
    - read by Kinesis Client Library applications
- data retention for up to 24 hours
- use cases:
    - react to changes in real-time
    - analytics
    - insert into derivatibe tables
    - insert into ElasticSearch
    - implement cross-region replication
- architecture:
    - changes to a table then appear in the DynamoDB Stream
    - can use a custom processing layer using a Lambda Function or Kinesis Client Library application
- ability to choose information that will be written to the stream
    - KEYS_ONLY: only the key attributes of the modified item
    - NEW_IMAGE: the entire item as it appears after it was modified
    - OLD_IMAGE: entire item, as it appeared before it was modified
    - NEW_AND_OLD_IMAGES: both the new and old images of the item
- streams are made of shards, just like Kinesis Data Streams
- don't provision shards manually, shard provisioning is automated by AWS
- *records are not retroactively populated in a stream after enabling it*

### Streams and Lambda
- need to define an EventSourceMapping to read from a DynamoDB stream
- need to ensure the Lambda has the appropriate IAM permissions to read from DynamoDB
- *the lambda is invoked synchronously*

### hands on

- enbale DynamoDB stream
- *trigger*: what is the stream going to trigger/kick off?
- *window*: time to gather records before invoking the function, in seconds
- *starting position*: read from the start or end of the stream

## TTL (Time-To-Live)

- automatically delete items after an expiry timestamp
- doesn't consume WCUs (no extra cost)
- TTL attribute must be a `number` data type with a "Unix Epoch timestamp" value
- expired items are deleted within 48 hours of expiration
- expired items that haven't been deleted appear in reads/queries/scans (if unwanted, filter them out)
- expired items are deleted from both LSIs and GSIs
- the delete operation for each expired item enters the DynamoDB Streams (which can help recover expired items)
- use cases: reduce stored data by keeping only current items, adhere to regulatory obligations

### hands on
- use a *custom* `expire on` attribute on the items
- tell the TTL configuration what the attribute that it needs to look for to delete based on

## CLI

- *--projection-expression*: one or more attributes to retrieve
- *--filter-expression*: filter items before they are returned to you
- general AWS CLI pagination options
    - *--page-size*: specify the AWS CLI retrieves the full list of items but with a larger number of API calls instead of one API call (default 1000 items)
    - *--max-items*: max number of items to show in the CLI (returns *NextToken*)
    - *--starting-token*: specify the last *NextToken* to retrieve the next set of items

*It's important for the Developer Associat exam to understand all of these CLI commands and options.*

## Transactions

- coordinated, all-or-nithing operations (add/delete/update) to multiple items across one or more tables
- provides atomicity, consistency, isolation and durability (ACID)
- *read modes*: 
    - eventual consistency
    - strong consistency
    - transactional
- *write modes*: 
    - standard
    - transactional
- *consumes 2X WCUs and RCUs*
    - DynamoDB performs 2 operations for every item (prepare and commit)
- two operations: (up to 25 unique items or up to 4MB of data)
    - *TransactGetItems*: one or more *GetItem* operations
    - *TransactWriteItems*: one or more *PutItem,UpdateItem and DeleteItem* operations
- use cases: financial transactions, managing orders, multiplayer games
- *a transaction is to all affected tables, or none of them!*
- capacity computations
    - transactional cost = 2
    - transactional writes per second * (item size/WCU capability) * transactional cost
    - transactional reads per second * (item size/RCU capability) * transactional cost
    - always round up the item size to the nearest multiple!

*Capacity Computations are important to know for the Developer Associate exam.*

## Session State

- common to use DynamoDB to store session state
- ElastiCache is in memory, DynamoDB is serverless, both are key/value stores
- EFS must be attached to EC2 instances as a network drive
- EBS and Instance Store can only be used for local caching, not shared caching
- S3 has a higher latency, it's not meant for small objects

## Partitioning Strategies

Write Sharding:
- distribute items evenly across partitions by *adding a suffix to partition key value*
- two methods:
    1. shard using a random suffix
    2. shard using a calculated suffix

## Conditional Writes, Concurrent Writes and Atomic Writes

### Concurrent Writes
- one of the writes is overwritten
- it's hard to know which will go second and overwrite the other
### Conditional Writes
- optimistic locking, can only update if the expected value is initially there, therefore one will fail after the other succeeds and there is no interference
### Atomic Writes:
- both writes succeed, combining together
### Batch Writes
- writes update many items at a time

## Patterns with S3

### Large Objects Pattern
- store large objects in S3
- store smaller metadata to get to the S3 object in the database and then query both systems for the object

### Indexing S3 Object Metadata
- S3 bucket has data sent to it from an application and invokes a Lambda Function to then store object metadata in the DynamoDB table
- client can then access the metadata to get the object

*It's important to know the DynamoDB an S3 patterns for the Developer Associate exam.*

## Operations

### Table cleanup
1. scan + delete items
    - slow, consumes RCU and WCU, expensive
2. drop table + recreate table
    - fast, efficient, cheap

### Copying a DynamoDB table
1. AWS Data Pipeline
2. Backup and restore into a new table
    - takes some time
3. scan + putitem or batchwriteitem
    - write your own code

## Security and other features

### Security
- VPC endpoints available to access DynamoDB without using the internet
- access fully controlled by IAM
- encryption at rest using KMS and in-transit using SSL/TLS

Backup and restore feature available
- point in time recovery (PITR) like RDS
- no performance impact

Global Tables
- multi-region, multi-active, fully replicated, high performance

DynamoDB Local
- develop and test apps locally without accessing the DynamoDB web service (no internet required)

AWS Database Migration Service (AWS DMS) can be used to migrate to DynamoDB

Users interact with DynamoDB directly
- use an identity provider to exchange temporary IAM access credentials

Fine grained access control
- using Web Identity Federation or Cognito Identity Pools, each user gets AWS credentials
- can assign an IAM role to these users with a *Condition* to limit their API access to DynamoDB
- *LeadingKeys*: limit row-level access for users of the *primary key*
- *Attributes*: limit specific attributes the user can see