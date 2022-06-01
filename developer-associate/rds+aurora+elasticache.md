# RDS (Relational Database Service)

Managed database service for databases that use SQL as a query language.
Create databases in the cloud managed by AWS
- Postgres
- MySQL
- MariaDB
- Oracle
- Microsoft SQL Server
- Aurora (AWS proprietary database)

*The above is fundamental knowledge for the exam*

RDS is a managed service meaning:
- automated provisioning, operating system patching
- continuous backups and ability to restore to specific timestamp
- monitoring dashboards
- read replicas for improved read performance
- multi-availability zone for disaster recovery
- maintenance windows for upgrades
- scaling capability (horizontal and vertical)
- storage backed by EBS (gp2 or io1)
*You cannot SSH into your instance*s

### RDS backups:
- automatically enabled in RDS
- automated backups:
    - daily full backup of the database
    - transaction logs are backed-up by RDS every 5 minutes
    - ability to restore to any point in time (oldest backup to 5 minutes ago)
    - 7 days retention (can be increased to 35 days)
- database snapshots
    - manually triggered by the user
    - allows retention of backup for as long as you want

### Storage Auto Scaling

- helps you increase storage on your RDS database instance dynamically
- when RDS detects you are running out of available database storage, it scales automatically
- avoid having to manually scaling your database storage
- set the *Maximum Storage Threshold*: the maximum limit for database storage
- automatically modifies storage if:
    - free storage is less than 10% of allocated storage
    - low storage lasts at least 5 minutes
    - 6 hours have passed since the last modification
- useful for applications with *unpredictable workloads*
- supports all RDS database engines

### RDS Read Replicas for read scalability

Read replica vs Multi-Availabity Zone is fundamental

- create up to 5 Read Replicas
- within Availability Zone, cross Availability Zone or cross region
- replication is *asynchronous* so reads are eventually consistent
- replicas can be promoted to their own database
    - once promoted, they are no longer included in the replication process
- applications must update the connection string to leverage read replicas
- *read replicas are only used for **SELECT** statements*

*Read Replicas add new endpoints with their own DNS name. We need to change our application to reference them individually to balance the read load.*

Example use case:
- production database taking normal load
- want application to run some analytics
- create Read Replica to run the new workload against without impacting the load on the production database
- the production application is unaffected

### Network Cost:
- There is normally a cost when data goes from one Availability Zone to another
*For RDS Read Replicas within the same region, there is no data transfer fee. There is a cross region fee though.*

### RDS Multi-Availability Zone
- *synchronous* replication, when master DB is written to the RDS is immediately replicated to
- one DNS name - automatic app failover to standby
- increased availability
- failover in case there is a loss of Availability Zone, network, instance or storage failure
- no manual intervention in apps
- not used for scaling
- *Read Replicas can be set up with multi-Availability Zone for disaster recovery*

*Multi-Availability Zone keeps the same connection string regardless of which database is up.*

#### From single-Availability Zone to multi-Availability Zone:
- zero downtime
- just click 'modify' for database

*Process*:
1. a snapshot is taken
2. a new database is restored from the snapshot in a new AZ
3. synchronisation is established between the two databases

### Hands on:

Using SQL electron to connect to the database.

RDS can have delete protection turned on that you have to turn off before being able to delete the database that you've created.

## RDS Security 

### Encryption

*At rest*:
- possible to encrypt the master and read replicas with AWS KMS - AES-256 encryption
- encryption has to be defined at launch time
- **if the master is not encrypted, the read replicas cannot be encrypted**
- *TDE* (transparent data encryption) is available for Oracle and SQL server

*In flight*:
- SSL certificates are used to encrypt in flight data
- provide SSL options with trust certificates when connecting to the database
- to enforce SSL
    - *PostgreSQ*: rds.force_ssl=1 in the AWS RDS console
    - *MySql*: within the DB: GRANT USAGE ON \*.\* TO 'mysqluser'@'%' REQUIRE SSL;
    - *You have a MySQL RDS database instance on which you want to enforce SSL connections. What should you do?* Execute a `REQUIRE SSL` SQL statement to all your DB users

#### Operations
- *encrypting RDS backups*
    - snapshots of un-encrypted RDS databases are un-encrypted
    - snapshots of encrypted RDS databases are encrypted
    - can copy a snapshot into an encrypted DB
- *to encrypt an un-encrypted DB*
    1. create a snapshot of the un-encrypted database
    2. copy the snapshot and enable encryption for the snapshot
    3. restore the database from the encrypted snapshot
    4. migrate applications to the new database and delete the old database

### Network
- RDS databases are usually deployed within a private subnet
- RDS security works by leveraging security groups, controlling which IP/security group can communicate with the RDS

### Access Management
- IAM policies help control who can manage AWS RDS
- traditional username and password can be used to login to the database
- IAM-based authentication can be used to login into RDS MySql & PostgreSql

#### IAM Authentication
- IAM-based authentication can be used to login into RDS MySql & PostgreSql
- don't need a password, just an authtoken obtained through IAM and RDS API calls
- Authtoken has a 15 min lifetime

*Benefits*
- Network in/out must be encrypted using SSL
- IAM to centrally manage users instead of DB
- can leverage IAM roles and EC2 instance profiles for easy integration

#### Your Responsibility:
- check the ports / IP / security group inbound rules in databases' security groups
- in-database user creation and permissions or manage through IAM
- creating a database with or without public access
- ensuring parameter groups or database is configured to only allow SSL connections

#### AWS responsibility:
- no SSH access
- no manual DB patching
- no manual OS patching
- no way to audit the underlying instance

# Amazon Aurora

- proprietary tech from AWS (so not open source)
- Postgres and MySQL are both supported as Aurora databases, meaning drivers work as if AuroraDB was a Postgres or MySQL database
- "AWS cloud optimised" and claims a 5x performance improvement over MySQL on RDS and 3x over Postgres on RDS
- *storage automatically grows in increments of 10GB up to 64TB*
- can have 15 replicas while MySQL only has 5, and the replication process is faster (sub 10ms lag)
- failover in Aurora is instantaneous, it's High Availability native
- costs ~20% more than RDS but is more efficient

## High Availability + Read Scaling

- 6 copies of data across 3 Availability Zones
    - requires 4 copies out of 6 needed for writes (to succeed)
    - requires 3 copies of 6 for reads (to succeed)
    - self healing with peer-to-peer replication
    - storage is striped across 100s of volumes
- one instance takes writes (master)
- automated failover for master in less than 30 seconds
- Master + up to 15 Read replicas serve reads
- support for cross region replication

## Aurora DB cluster

*Writer endpoint*: points to master. address doesn't change. client talks to writer endpoint. only ever talks to the master instance
*Reader endpoint*: connection load balancing. address doesn't change. client talks to reader endpoint for reads. reader endpoint gets reads from all read replicas and master.

### Features:
- auto failover
- backup and recovery
- isolation and security
- industry compliance
- push button scaling
- auto patching with zero downtime
- advanced monitoring
- routine maintenance
- *backtrack*: restore data at any point of time without using backups

### Security:
- similar to RDS because it uses the same engines
- encrypt at rest using KMS
- auto backups, snapshots and replicas also encrypted
- encryption in flight using SSL
- *possible to authenticate using IAM token (same as RDS)*
- you are responsbile for protecting your instances with security groups
- cannot SSH

### Hands on:

- *Serverless*: "You specify the minimum and maximum amount of resources needed, and Aurora scales the capacity based on database load. This is a good option for intermittent or unpredictable workloads"
- if you don't use a replica, no matter what storage layer is still across 3 Availability Zones
- clients should really only connect through reader and writer endpoints
- replica autoscaling ensures that there is never too much load on the replicas
- *Global database feature*: add database to other regions, allows for global aurora. need a big enough instance to use. 
- need to delete the replicas before you can delete the instance

# Elasicache

- same way RDS is used to manage Relational Databases, *Elasticache is used to manage Redis or Memcached*
- caches are in-memory databases with really high performance and low latency
- helps reduce load off of databases for read intesive workloads
- helps make application stateless 
- AWS takes care of operating system maintenance/patching, optimisations, setup, configuration, monitoring, failure recovery and backups
- *using ElastiCache involves heavy application code changes*

## Architecture:

- *DB Cache*:
    - application queries ElastiCache, if data is not available, make a call to the RDS and store the response in the cache.
    - relieve load in RDS
    - cache must have an invalidation strategy to make sure only the most current data is stored in it
- *User Session Store*:
    - user logs into any of the application instances
    - application writes the session data into ElastiCache
    - user hits another instance of the application
    - instance retrieves the data and the user is already logged in


## Redis vs Memcached

### Redis:
- multi-availability zone with auto-failover
- read replicas to scale reads and have high availability
- data durability using AOF persistence
- backup and restore features
- good when you really don't want to lose cached data

### Memcached:
- multi-node for partitioning data (sharding)
- no high availability (replication)
- non persisten
- no backup and restore
- multi-threaded
- basically good for when you aren't worried about loss of cached data

## ElastiCache Strategies

- *safe to cache data?* yes, but data may be out of date, eventually consistent. only cache data if it is appropriate to cache
- cache effective for that data?*
    - *Pattern*: data changing slowly, few keys are frequently needed
    - *anti-pattern*: data changing rapidly, all large key space needed frequently
- *is data well structured for caching?* key value or caching aggregations of results
- **which design pattern is the most appropriate?**

### Lazy loading/ Cache Aside / Lazy population

*Cache miss*: data not in cache
*cache hit*: data in cache

*Only write data to cache when it is required*

Pros:
- only requested data is cached
- node failures are not fatal
Cons:
- cache miss penalty that results in 3 round trips, noticable delay for request
- stale data: data can be updated in the db and outdated in cache

The exam requires you to be able to read the code for a lazy loading pattern

### Write Through
*Add or update cache when database is updated*

Pros:
- data in cache is never stale, reads are quick
- write penalty vs read penalty. each write requires two calls, which users understand better
Cons:
- missing data until databse is added/updated. Mitigation is to implement lazy loading as well
- *cache churn* - a lot of the data will never be read

### Cache Eviction and Time-to-live
- cache eviction can occur in 3 ways
    - delete item explicitly ion the cache
    - item is evicted because the memory is full and not recently used
    - set item time-to-live
- Time-To-Live is helpful for any kind of data
    - leaderboads
    - comments
    - activity streams
- Time-To-Live can range from second to hours to days
- if too many evictions happen due to lack of memory, you should scale up or out

Lazy loadinjg is easy to implement and works for many situations as a foundation
write-thorugh is usually combined with lazy loading as targeted for the queries or workloads that benegit froim this optimisation
- setting a TTL is usually not a bad idea, except when using Write-Through. set it to a sensible value
- only cache the data that makes sense

## ElastiCache Cluster Modes

### Cluster mode disabled
- one node, up to 5 replicas
- async replication
- primary node for read & write
- other nodes readonly
- one shard, all nodes have all data
- guard against data loss in case of node failure
- multi-availability zone enabled by default for failover
- helpful to scale read performance

### Cluster mode enabled
- data partitioned across shards (helpful to scale writes)
- each shard has a primary node and up to 5 replica nodes
- multi-Availability Zone
- up to 500 nodes per cluster
    - 500 shards with single master
    - 250 shards with one master and 1 replica
    - 93 shards with one master and 5 replicas