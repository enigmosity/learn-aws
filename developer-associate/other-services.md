# SES - Simple Email Service

- send emails to people using
    - SMTP interface
    - AWS SDK
- ability to receive emails. integrates with
    - S3
    - SNS
    - Lambda
- integrated with IAM for allowing to send emails

# Summary of Databases
- RDS: relational databases, OLTP (Online transaction processing)
    - PostgreSQL, MySQL, Oracle
    - Aurora + Aurora Serverless
    - provisioned database
- DynamoDB: NoSQL database
    - managed, key value, document
    - serverless
- ElastiCache: in memory database
    - Redis/Memcached
    - Cache capability
- Redshift: OLAP - Analytic processing
    - data warehousing / data lake
    - analytic queries
- Neptune: graph database
- DMS: Database Migration Service
- DocumentDB: managed MongoDB for AWS

# Amazon Certificate Manager (ACM)

- provision, manage and deploy *SSL/TLS certificates*
- used to provide in-flight encryption for websites (HTTPS)
- supports both public and private TLS certificates
- free of charge for public TLS certificates
- automatic TLS certificate renewal
- integrations with (load TLS certificates on)
    - ELBs
    - CloudFront Distributions
    - APIs on API Gateway
 
# Cloud Map

- fully managed resource discovery service
- creates a map of backend services/resources that your applications depend on
- register your application components, their locations, attributes, and health status' with AWS Cloud Map
- applications can query Cloud Map using AWS SDK, API or DNS

# Fault Injection Simulator (FIS)

- fully managed service for running fault injection experiments on AWS workloads
- based on *chaos engineering* - stressing an application by creating disruptive events, observing how the system responds and implementing improvements
- helps you uncover hidden bugs and performance bottlenecks
- supports the following AWS services: EC2, ECS, EKS, RDS
- use pre-built templates that generate the desired disruptions
