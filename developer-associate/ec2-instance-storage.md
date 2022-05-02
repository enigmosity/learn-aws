# EC2 instance storage

## EBS (Elastic Block Store) volume

- network drive you can attach to your instances while they run
- allows instances to persist data, even after their termination
- can only be mounted to one instance at a time (there is a 'multi-attach' feature for some EBS)
- bound to a specific availability zone

analogy: *network USB stick*
Free tier: 30 GB of free EBS storage of type General Purpose (SSD) or Magnetic per month

- a network drive and so uses network to communicate, there may be latency
    - can be detached from one instance and attached to another instance
- locked to an availability zone
    - to move volume across availability zones, snapshot it first
- have a provisioned capacity (in GBs, and IOPS)
    - can increase size over time
    - cost based on capacity

Can have multiple EBS volumes on one instance
EBS volumes do not have to be attached, can exist and then be attached on demand.

*Delete on termination* controls EBS behaviour when an EC2 instance terminates
- root EBS deleted by default on termination
- any other attached EBS not deleted by default
- you can customise it to do whatever you want on termination

Can only attach a new volume once it is created and is marked as `available`.

To know how to use a non-root attached EBS volume, google along the lines of `format EBS volume to attach on EC2 instance`. It is out of scope of the developer exam.

### EBS Snapshots

- make a backup (snapshot) of your EBS volume at a point in time. Idea is you can restore if required. 
- not necessary to detach volume to do a snapshot, but it's recommended so that you can guarantee the state of the volume.
- can copy snapshots across availability zones and regions.
- snapshots are stored within region, not availability zones
- you pay for storing snapshots

## AMI (Amazon Machine Image)

A customisation of an EC2 instance
- add your own software, configuration, operating system, monitoring, etc.
- faster boot/config time because required software is pre-packaged
- built for a specific region and can be copied across to other regions
- can launch EC2 instances from
    - *public AMI*: AWS provided
    - *your own AMI*: made and maintained yourself
    - *AWS marketplace AMI*: AMI someone else made and potentially sells

*AMIs are built for a specific AWS Region, they're unique for each AWS Region. You can't launch an EC2 instance using an AMI in another AWS Region, but you can copy the AMI to the target AWS Region and then use it to create your EC2 instances.*


### AMI process from EC2 instance

1. start an EC2 instance and customise it
2. stop the instance to ensure data integrity
3. build an AMI, which will also create EBS snapshots
4. launch instances from the AMI

## EC2 instance store

EBS volumes are network drives with good but 'limited' performance. If you need a high performance hardware disk, use EC2 instance store
- better I/O performance
- EC2 instance store lose their storage if they're stopped (ephemeral)
- good for buffer/cache/scratch data/temporary content
- risk of data loss if hardware fails
- *backups* and *replication* are your responsibility

### EBS volume types

EBS volumes are characterised in Size|Throughput|IOPS(I/O Operations Per Second)

- *gp2/gp3* (SSD): general purpose SSD volume that balances price and performance for a wide variety of workloads
    - cost effective storage, low-latency
    - system boot volumes, virtual desktops, development and test environments
    - 1 GiB - 16 TiB TODO: what are GiB, MiB and TiB?
    - *gp3*:
        - baseline of 3,000 IOPS and throughout of 125 MiB/s
        - can increase IOPS up to 16,000 and throughput up to 1000 MiB/s independently
    - *gp2*:
        - small gp2 volumes can burst IOPS to 3,000
        - size of volume and IOPS are linked, max IOPS is 16,000
        - 3 IOPS per GB, which means at 5,334 GB IOPS is maximised
    - *how IOPS and volume can be scaled are the main differences between gp2 and gp3*
- *io1/io2* (SSD): highest performance SSD volume for mission-critical low-latency or high-throughput workloads
    - *provisioned IOPS*
    - critical business applications with sustained IOPS performance or need more than 16,000 IOPS
    - Great for database workloads (sensitive to storage performance and consistency)
    - *io1/io2* (4 GiB - 16 TiB):
        - Max PIOPS: 64,000 for Nitro EC2 instances and 32,000 for other 
        - can increase PIOPS indepedently from storage size
        - io2 have more durability and more IOPS per GiB (at the same price as io1)
    - *io2 Block Express* (4GiB - 64TiB)
        - sub-millisecond latency
        - Max PIOPS: 256,000 with an IOPS:GiB ratio of 1,000:1
    - supports EBS multi-attach 
- *st1* (HDD): low cost HDD volume designed for frequently access, throughput intensive workloads
    - big data, data warehouses, log processing
    - max throughput: 500 MiB/s - max IOPS 500
- *sc1* (HDD): lowest cost HDD volume designed for less frequently accessed workloads
    - for data that is infrequently accessed
    - scenarios where lowest cost is important
    - max throughput 250 MiB/s - max IOPS 250

*HDD drives*:
- cannot be a boot volume
- 125 MiB - 26 TiB

*Only gp2/gp3 and io1/io2 can be used as boot volumes*

When in doubt consult the documentation!

For the exam, you don't need to remember all the differences, but need to know the high level.

### EBS Multi-Attach
- io1/io2 family only
- attach the same EBS volume to multiple EC2 instances in the same AZ
- each instance has full read and write permissions to the volume
- use case: 
    - achieve *higher application availability* in clustered Linux applications
    - applications must manage concurrent write operations
- must use a file system that's cluster aware

## EFS (Elastic File System)

- managed network file system that can be mounted on many EC2 instances
- EFS works with EC2 instances in multi-availability zones
- highly available, scalable, expensive (3x gp2), pay per use
- use cases:
    - content management, web serving, data sharing, wordpress
- uses NDSv4.I protocol
- uses security group to control access to EFS
- *compatible with Linux based AMIs (but not Windows)*
- encryption at rest using KMS
- only for a POSIX file system that has a standard file API 
- file system scales automatically, pay-per-use, no capacity planning

*Scale*:
- 1000s of concurrent NFS clients, 10GB+/s throughput
- grow to petabyte-scale network file system automatically
*Performance mode* (set as EFS creation time):
- general purpose (default): latency-sensitive use cases (web server, CMS, etc.)
- max I/O: higher latency, throughput, highly parallel (big data, media processing)
*Throughput mode*:
- bursting (1TB = 50MiB/s+ burst of up to 100 MiB/s)
- provisioned: set your throughout regardless of storage size
*Storage Tiers* (lifecycle management feature - move file after N days):
- standard: for frequently accessed files
- infrequent access (EFS-IA): cost to retrieve files, lower price to store

For the Developer Associate certification, the file system policy is out of scope

### Hands on
- you can kick off a similar EC2 instance by selecting the EC2 instance and hitting 'run more like this'
- to attach EFS on EC2 instances you need
    - to mount via DNS or IP address, instructions can be found in the AWS console.
    - the `amazon-efs-utils` package on the instance for DNS, which you need to ssh into the instance for
- to mount an EFS, you need to enable inbound calls to the EC2 instance from the EFS

EBS vs EFS

EBS:
- attached to one instance at a time
- locked to the availability zone they're created in
- *gp2*: IO increases if the disk size increases
- *io1*: IO can increase independently
- to migrate across availability zones:
    - take a snapshot
    - restore the snapshot to another availability zone
- EBS backups use I/O and should run while application is handling a lot of traffic
- root EBS volumes of instances are terminated by default when the instance is terminated

EFS:
- mounting to 100s of instances across availability zones
- EFS shared website files (like wordpress)
- only for linux instances
- higher price point than EBS
- can leverage EFS-IA for cost savings

For the certification exam, make sure you understand EFS versus EBS versus Instance Store