# VPC (Virtual private cloud)

AWS Certified Developer should know:
- VPC, subnets, Internet Gateways & NAT Gateways
- security groups, network ACL (NACL), VPC Flow Logs
- VPC peering, VPC endpoints
- Site to site VPN & Direct Connect

*VPC*: private network to deploy your resources (a regional resource)

*Subnets*: allow you to partition your network inside your VPC
- A public subnet is a subnet that is accessible from the internet
- A private subnet is a subnet not accessible from the internet
To define access to the internet and between subnets, use *Route tables*

*Internet Gateway*
- helps VPC instances connect with the internet
- public subnets have a route to the internet gateway

*NAT Gateway*
- gateways are AWS managed
- NAT instances are self managed
- allow instances in private subnets to access the internet while remaining private
- live in the public subnet so that they can be contacted and forward the private subnet requests to the internet

## Network ACL & Security Groups

*Network ACL*
- firewall which controls traffic to and from subnets
- ALLOW and DENY rules
- attached at subnet level
- rules only include IP addresses
- default allows everything in and everything out
- all traffic hits these before an instance

*Security groups*
- firewall that controls traffic to and from an ENI (Elastic Network Interface) / EC2 instance
- only ALLOW rykes
- rules include IP addresses and other security groups

*Security groups are stateful and if traffic can go out, then it can go back in.*

*VPC Flow Logs*
- captures info about IP traffic going into intrefaces
    - VPC flow logs
    - subnet flow logs
    - ENI flow logs
- helps to monitor and troubleshoot connectivity issues
- captures network information from AWS managed interaces to: 
    - ELBs
    - ElastiCache
    - RDS 
    - Aurora, etc.
- VPC flow log data can go to S3/CloudWatch logs

## VPC Peering

- connect two VPC, privately using AWS network
- make them behave as if in the same network
- must no thave an overlapping CIDR (IP address range)
- peering connection is *not transitive* (must establish for each VPC that needs to communicate with each other)

**VPC Endpoints**
- allow you to connect to AWS services using a private network instead of the public www network
- enhanced security and lower latency to access AWS services

*VPC Endpoint Gateway*: S3 & DynamoDB
*VPC Endpoint Interface*: the rest of the AWS services

*Only used within the VPC*

EXAM: VPC endpoints are important for the exam, likely to be a question.

*Site-to-Site VPN*
- connect on premises VPN to AWS
- connection automaticaly encrypted
- over public internet

*Direct Connect (DX)*
- physical connection between on-premise and AWS
- connection is private, secure and fast
- over a private network
- at least a month to establish
NOTE: site-to-site VPN & DX cannot access VPC endpoints

## Summary:
- *VPC*: 1 default per region used
- *Subnets*: tied to AZ, network partition of the VPC
- *Internet Gateway*:  VPC level, provide internet access
- *NAT Gateway/Instances*: internet access to private subnets
- *NACL*: stateless, subnet rules for inbound and outbound
- *Security Groups*: stateful, operate at the EC2 instance or ENI level
- *VPC Peering*: connect two VPC with non-overlapping IP ranges, non transitive
- *VPC Endpoints*: private access to AWS services within VPC
- *VPC flow logs*: network traffic logs
- *Site to Site VPN*: VPN over public internet between on permises DC and AWS
- *Direct Connect*: direct private connection to AWS

## Three Tier Architecture

- *Public subnet*: Elastic Load Balancer
- *Private Subnet*: Auto-Scaling Group with EC2 instances
- *Data subnet*: ElastiCache, Amazon RDS

LAMP Stack on EC2
- *Linux*: Operating system for EC2 instances
- *Apache*: Web Server that runs on Linux
- *MySQL*: database on RDS
- PHP: application logic running on the EC2
- Can add ElastiCache for caching
- Store local application data and software = root EBS drive
