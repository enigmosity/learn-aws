# S3

## MFA-Delete

- MFA forces users to generate a code on a device before doing important operations on S3 buckets
- to use MFA-Delete (MFA required for permanent deletion) enable Versioning on the S3 bucket
- will need MFA to
    - permanently delete an object version
    - suspend versioning on the bucket
- won't need to
    - enable versioning
    - listing deleted versions
- *only the bucket owner (root account) can enable/disable MFA-Delete*
- currently only enabled using the CLI, can also only delete files using the CLI when MFA-Delete is enabled
- need to configure AWS CLI to use the root access keys to enable MFA-Delete. Don't do this for any other activity

## Default Encryption vs Bucket Policies

- one way to force encryption is to use a bucket policy and refuse any API call to PUT an S3 object without encryption headers
- another way is to use the *default encryption* option in S3
- bucket policies are evaluated before *default encryption*
- if you load an object in an unencrypted way, default encryption will handle the encrypting for you

## Access Logs

- For audit purposes, may want to log all access to S3 buckets
- any request made to S3, from any account, authorised or denied, will be logged into another S3 bucket
- data can be analysed using data analysis tools or Amazon Athena

**Never set logging bucket to be the monitored bucket. This will create a logging loop, and the bucket will grow in size exponentially.**

## Hands on

- in the console, just jump into the S3 bucket you want to monitor and configure the server access logging. permissions to add the logs to an S3 are automatically added to the monitored bucket by AWS.

## Replication

*CRR*: **Cross** Region Replication
- compliance, lower latency access, replication across accounts
*SRR*: **Same** Region Replication
- log aggregation, live replication between production and test accounts

- must enable versioning in source and destination buckets
- buckets can be in different accounts
- copying is *asynchronous*
- must give proper IAM permissions to S3

Notes:
- After activating, only new objects are replicated (*not retroactive*)
- DELETE operations:
    - can replicate delete markers from source to target (optional setting)
    - deletions with a version ID are not replicated (to avoid malicious deletes)
- no 'chaining' of replication
    - bucket 1 has replication into bucket 2, which has replication into bucket 3, objects created in bucket 1 are not replicated in bucket 3


### Hands on

- must have bucket versioning enabled
- Replication controlled in the `management` tab in the console. Create a rule for replication. Need to have another bucket to replicate into. Need an IAM role.
- Delete markers can optionally be replicated.
- Replication objects (created off the original bucket) should have the same version ID as on the original bucket.
- Without deletion markers, no information regarding deletion is passed to the replication bucket

## Pre-Signed URLs

- can generate pre-signed URLs using SDK or CLI
    - downloads, use CLI
    - uploads, use SDK
- valid for default time of 3600 seconds, timeout can be changed with `--expires-in <time in seconds>` argument
- users given a pre-signed URL inherit the permissions of the person who generated the URL for GET/PUT

### Hands on

- `aws s3 presign s3://<bucketname>/<pathtoobjec>t --region <region>`
    - region is the region of the bucket
    - in the cli

## Storage Classes

- *Standard - General Purpose*
    - 99.99% availability
    - used for frequently accessed data
    - low latency and high throughput
    - sustain 2 concurrent facility failures
    - use cases
        - big data analytics
        - mobile and gaming applications
        - content distribution
- *Standard - Infrequent Access (IA)*
    - less frequently accessed, but requires rapid access when needed
    - lower cost than standard, but costs for retrieval of data
    - 99.9% availability
    - use cases: disaster recovery, backups
- *One Zone - Infrequent Access*
    - high durability 99.999999999% in a single availability zone, data lost when availability zone destroyed
    - 99.5% availability
    - use cases: storing secondary backup copies of on-premise data, or recreatable data
- *Glacier Instant Retrieval*
    - low cost object storage meant for archiving/backup
    - pricing: price for storage + object retrieval cost
    - millisecond retrieval, great for data accessed once a quarter
    - minimum storage duration of 90 days
- *Glacier Flexible Retrieval*
    - expedited (1-5 min), standard (3-5 h), bulk (5-12h) - free
    - minimum storage duration of 90 days
- *Glacier Deep Archive*
    - standard (12h), bulk (48h)
    - minimum storage of 180 days
- *Intelligient Tiering*
    - small monthly monitoring and auto-tiering fee
    - moves objects automatically between *AccessTiers* based on usage
    - no retrieval charges in S3 intelligent-tiering
    - *frequent access* tier (automatic): default
    - *infrequent access* tier (automatic): objects not accessed for 30 days
    - *archive instant access* tier (automatic): objects not accessed for 90 days
    - *archive access* tier (optional): configurable from 90 days to 700+ days
    - *deep archive access* tier (optional): configurable from 180 days to 700+ days

Can morv between classes manually or using S3 lifecycle configurations

*Durability*
- how many times an object is going to be lost by S3
- high dureability (11 9s)
- 10 mil objects, one object lost once every 10000 years
- same for all storage classes

*Availability*
- measure how readily available a service is
- depends on storage class

### Hands on

- editable after you've created the bucket. 
- select a storage class during object upload time.

## Lifecycle rules

Can transition objects between storage classes.
IA, move to standard IA. for archive objects not needed in real time, glacier or deep archive.
moving objects can be automated using a *lifecycle configuration*

*Rules*:
- *transition actions*: defines when objects are transitioned to another storage class
- *expiration actions*: configure objects to expire (delete) after some time
- can be created for a certain prefix
- can be created for certain object types

*Exam will ask scenario questions about when you'll move objects between storage classes*

### hands on

- *current*: most recent object
- _previous_: all previous versions of objects

- make sure things going into archive or glacier are big enough to not cost more

## Performance

- automatically scales to high request rates, latency 100-200 ms
- application can achieve at least *3500 put/copy/post/delete and get 5500 get/head requests per second per prefix in a bucket*
- no limits to the number of prefixes in a bucket
- if spread reads across four prefixes evenly, can achieve 22000 requests per second for get and head


### KMS limitations
- using SSE-KMS may be impacted by KMS limits
- when upload, calls *GenerateDataKey* KMS API
- download, calls *Decrypt* KMS API
- count towards the KMS quota per second
- can request quota increase using Service Quotas Console

### Optimising performance
- _multi-part upload_
    - recommended for files > 100mb, must used for files >5gb
    - can help parallelise uploads (speed up transfers)
- _S3 transfer acceleration_
    - increase transfer speed by transferring file to an AWS edge location which will forward the data to the S3 bucket in the target region
    - compatible with multi-part upload
- *S3 Byte-Range Fetches*
    - parallelise GETs by requesting specific byte ranges
    - better resilience in case of failure
    - can be used to speed up downloads
    - can be used to retrieve of partial data (eg. head of a file)

## S3 Select & Glacier Select
- retrieve less data using SQL by performing *server side filtering*
- can filter by rows and columns
- less network transfer, less CPU cost client-side

For exam: Filtering of data server side to get less data, think S3 Select and Glacier Select. This is the simple version of it.

## Event Notifications

- events happen in the S3 bucket
- want to be able to react, do so using rules
- object name filtering is possible
- use cases: generate thumbnails of images uploaded to S3
- SNS, SQS, Lambda functions are the 3 targets for S3 events
- S3 event notifications typically deliver events in seconds but can take a minute or longer
- two writes made to one non-versioned object at same time may only send a single event notification
- enable versioning to ensure all event notifications are sent

### Hands on

- Event notifications are in the S3 bucket properties
- Ensure S3 bucket has the policies required to trigger something on the target.

# Athena

- *serverless* query service to *perform analytics against S3 objects*
- uses standard SQL language to query the files
- supports CSV, JSON, ORC, Avro, and Parquet
- pricing: $5/TB of data scanned
- use compressed or columnar data for cost-savings (less data scanned)
- use cases: business intelligence/analytics/reporting, analyse and query VPC flow logs, ELB logs, CloudTrail trails, etc.

Exam tip: to analyse data in S3 using serverless, use Athena

### Hands on

- Need an S3 bucket to store query results in.
- Need to create a database for Athena in SQL. 
- Write a query to fill the table from the S3 bucket. 
- Write queries to be able to handle collecting the information that you want to see