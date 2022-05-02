# EC2 Fundamentals

## Budgets

You can create an AWS budget that will alert and take action if you hit a specified threshold of cost for a month.

**EC2**: Elastic Compute Clouds, *Infrastructure as a Service*

Mainly consists of the capability to:
- rent virtual machines (EC2)
- store data on virtual drives (EBS)
- distribute load across machines (ELB)
- scale the services using an auto-scaling group (ASG)

Configuration options:
- Operating systems: Linux, Windows or Mac
- compute power and cores (CPU)
- RAM
- storage
    - network attached (EBS & EFS)
    - hardware (EC2 Instance Store)
- network card: speed of card, public IP address
- firewall rules: security group
- bootstrap script(configure at first launch): EC2 user data

EC2 User Data Script:
- used to automate boot tasks
- script runs with the root user (sudo) access
- script is only run once as the machine starts

### Hands on

-*Delete on termination*: delete the data from the EC2 when the EC2 instance is terminated.
- Key pair is used to SSH onto an instance.
- Cannot directly open IPV4 address as it then requires HTTPS which is not yet set up
- When stopping and starting an instance, public ipv4 will change. Turn your machines on and off whenever you feel like it.

## EC2 instance types

m.5.2xlarge
m: instance class
5: generation
2xlarge: size within instance class.

7 types:
- *General purpose*:
    - great for diversity of workloads like web servers or code repositories
    - balance between
        - compute
        - memory
        - networking
- *Compute optimised*:
    - compute-intensive tasks
        - batch processing workloads
        - media transcoding
        - high performance web servers 
        - high performance computing 
        - scientifiv modeling and machine learning
        - dedicated gaming servers
- *Memory optimised*:
    - fast performance for workloads that process large data sets in memory
        - high performance, relational/non-relational databases
        - distributed web scale cache stores
        - in-memory databases optimised for business intelligence
        - applications performing real-time processing of big unstructured data
- *Storage optimised*:
    - Great for storage-instensive tasks that require high, sequential read and write access to large data sets on local storage
        - high frequency online transaction processing systems
        - relational and NoSQL databases
        - cache for in-memory databases
        - data warehousing applications
        - distributed file systems
- *Accelerated Computing*
    - based on the fact that this type is skipped in the course, I do not expect it to be included in the developer associate certification

## Security groups

Fundamental to network security of AWS. Control how traffic is allowed into or out of EC2 instances.

Only contain *allow* rules. Can reference by IP or by security group.

*Firewall*:
- access to ports
- authorised IP ranges - IPv4 & IPv6
- inbound network
- outbound network

not required to know how to configure for the developer certification.

- can be attached to multiple instances
- locked to region/VPC
- lives 'outside' the EC2, EC2 won't see blocked traffic
- good to maintain one separate security group for SSH access
- if application is not accessible (timeout error), security group issue
- if application gives connection refused error, then it's an application error or application not launched
- inbound traffic is blocked by default
- outbound traffic authorised by default

*Ports you need to know*:
- 22: SSH (secure shell)
- 21: FTP (file transfer protocol)
- 22: SFTP (secure file transfer protocol)
- 80: HTTP
- 443: HTTPS
- 3389: RDP (remote desktop protocol)

Security groups can be applied to many EC2 instances. An EC2 instance can also have many security groups, their rules just add up.

## SSH

SSH can be used on Mac, Linux or Windows 10 (or higher).

EC2 instance connect is similar, works with all operating systems but only with Amazon EC2.

SSH is the most common problem. At worst, use EC2 instance connect to make connecting easier.

### SSH on windows 10

'chmod' doesn't exist on windows. In security, make the key owner your administrator account then remove any other user as it has inheritance. The command to run is: 

`ssh -i <location of PEM file> ec2-user@<public ip of instance>`

### Troubleshooting

- Timeout:
    - a security group issue, ensure you have the required access in your security group or firewall
- Still having timeout issues:
    - corporate or personal firewall, try using EC2 instance connect
- SSH command not found
    - ssh is not installed, use Putty on windows
- Connection refused:
    - reachable instance, but no SSH utility on the instance
        - reboot the instance
        - terminate and create a new one
        - make sure it's Amazon Linux 2
- Permission denied
    - wrong or no security key
    - wrong user
- Worked yesterday but not today
    - is your instance still running?

## EC2 instance connect

Select the instance in the console and hit the `connect` button. Fires up a 'terminal' within your browser to work on the instance. Still need to know the user (ec2-user). Treat it the same as any other terminal.

## IAM roles for EC2 instances

Comes installed with AWS CLI on the instance. Don't connect to an EC2 instance and enter your IAM secrets, as anyone else on the account could log in and see your personal data.

Changes from IAM roles across the rest of AWS can take some time.

## EC2 instance launch types

Instance purchasing options:
- **On-demand instances**:
    - short workload
    - predictable pricing
        - pay for what you use
        - highest cost but no upfront payment
        - no long term commitment
        - recommended for *short term* and *un-interrupted workloads*, where you can't predict how the application will behave
- **Reserved instance** (min 1 year)
    - *reserved instances*: long workloads
        - up to 75% discount
        - reservation period, 1 - 3 years - You must reserve the instance for 1 or 3 years, not any time in between.
        - purchasing options: no upfront - partial upfront - all upfront (varying discounts)
        - reserve a specific instance type
        - recommended for steady state usage applications (eg. databases)
    - *convertible reserved instances*: long workloads with flexible instances
        - can change EC2 instance type over time
        - up to 54% discount
    - *scheduled reserved instances*: eg. set regular time slots
        - launch within time window you reserve
        - when need for a fraction of time
        - still need to commit to long time
        - **deprecated but may still come up in exam**
- **Spot instances**:
    - short worklaods
    - cheap
    - less reliable, can lose instances
        - up to 90% discount
        - can lose an instance at any time if your max price is less than current spot price
        - most cost-efficient
        - workloads must be resilient to failure
            - batch jobs
            - data analysis
            - image processing
            - distributed workloads
            - workloads with flexible start and end
        - **not suitable for critical jobs or databases**
- **Dedicated hosts**:
    - book an entire physical server
    - control instance placement
    - best used for:
        - compliance requirements 
        - allows you to use existing server-bound software licenses
    - 3 years at a time
    - more expensive
- **Dedicated instances**:
    - instances running on hardware dedicated to you
    - may share hardware with other instances in same account
    - not control over placement