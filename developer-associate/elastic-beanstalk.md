# Elastic Beanstalk

## Overview

Beanstalk is a developer centric view of deploying an application on AWS.
- managed service
    - automatically handles capacity provisioning, load balancing, scaling, application health monitoring, instance configuration, etc.
    - just the application code is the responsibility of the developer
- still have full control over the configuration
- free, but pay for the underlying instances and resources

*Components*:
- *application*: collection of Elastic Beanstalk components (environments, versions, configurations, etc.)
- *application version*: an iteration of your application code
- *environment*:
    - collection of AWS resources running an application version (one version at a time)
    - *tiers*: web server environment tier and worker environment tier
    - can create multiple environments

*Web environment*: traditional architecture. client -> elb -> servers
*Worker environment*: 
- no client can directly accessing EC2 instances
- messages are sent to SQS and then sent to EC2 to process
- scale EC2s based on number of SQS messages. 
- can push messages to queue from a web tier 
- client -> queue -> servers

First Environment:
- Events show you all events that occur in your environment.
- S3 bucket holds information
- uses an Elastic IP address 
- Beanstalk basically creates everything for you

Second Environment
- web server vs worker environment
- configure more options
    - check for free tier eligibility
    - can fully customise everything if you want to
    - software:
        - configuration of resources that your software will utilise
    - capacity
        - ASG, standard configuration of the ASG
    - load balancer, can also be modified
    - can automatically spin up a database too
        - deleting beanstalk environment will also delete RDS (database) data
    - automatically spins up security groups

## Beanstalk Deployment Modes

- *Single instance deployment*: great for development
    - one availability zone
    - security group
    - ASG, Elastic IP and a web app server

- *high availability with load balancer*: great for production
    - ELB
    - 3 availability zones
    - ASG with an EC2 instance security group per web app server
    - additional security groups too

Beanstalk deployment options for updates:

- *All at once (deploy all in one go)*: fastest, but there is downtime as instances cannot serve traffic all at the same time
    - stop all instances, put new versions on
    - great for dev as downtime doesn't matter
    - no additional cost

- *Rolling*: deploys to afew instances at a time (a bucket), then moves to the next bucket upon completion
    - application running below capacity, some instances are stopped, some continue running
    - can set bucket size
    - runs both versions simultaneously
    - no additional cost as the same number of instances are running
    - if bucket size is low, there is a long deployment time

- *Rolling with additional batches*: spins up new instances to move the batch (so old application is still available)
- application is always running at capacity
- can set bucket size
- run both versions at same time
- small additional cost
- additional batch is removed at the end of deployment
- longer deployment
- good for production

- *Immutable*: spins up new instances in a new ASG, deploys new version to the new instances, and then swaps all the instances if everything is healthy
    - zero downtime
    - new code  deployed to new instances on a temporary ASG
    - high cost, double capacity
    - longest deployment
    - quick rollback in case of failures (just terminate the temporary ASG)
    - great for production

- *Blue/Green*:
    - not a 'direct feature' of beanstalk
    - zero downtime and release facility
    - create a new 'stage' environment and deploy v2 there
    - new environment (green) can be validated independently and rolled back if there are issues
    - Route 53 can be setup using weighted policies to redirect a bit of traffic to the stage environment
    - using Beanstalk, 'swap URLs' when done with the environment test
    - very manual

- *Traffic Splitting*
    - canary testing
    - new application version deployed to temporary ASG with the same capacity
    - small percentage of traffic sent to temporary ASG for a configurable amount of time
    - deployment health monitored
    - if deployment failure occurs, automated rollback is triggered
    - no application downtime
    - new instances are migrated from the temporary to the original ASG, then old instances are terminated - the temporary ASG is dumped

*Beanstalk deployment modes and options are common exam questions*

### Hands on

- Deployment options are configured in 'rolling options and deployments' section. 
- Deployment preferences are about controlling healthchecks
- zip file and upload to deploy to your environment
    - there must be a way to set this up with version control
- swap is done through the action button at the overview level

## Beanstalk CLI

'EB CLI' makes working with Beanstalk from CLI easier
- helpful for your automated deployment piplines
- describe dependences
- package code as zip, and describe dependenes
- console: upload zip file (creates new app version), and then deploy
- CLI: create new app version using CLI (uploads zip), and then deploy
- Beanstalk will deploy the zip on each EC2 instance, resolve dependencies and start the application

*This out out of scope of the developer associate exam.*

## Lifecycle Policy

- Beanstalk can store at most 1000 application versions
- if you don't remove old versions, won't be able to deploy anymore
- to phase out old application versions, use a *lifecycle policy*
    - based on time (remove old versions); or
    - based on space (remove only when have too many)
- versions currently in use will not be deleted
- option not to delete the source bundle in S3 to prevent data loss
- *service role*: role that allows Beanstalk to remove things from S3

## Beanstalk Extensions

- a zip file containing your code must be deployed to Beanstalk
- all parameters set in the UI can be configured with code using files
- Requirements:
    - in `.ebextensions/` directory in the root of source code
    - YAML/JSON format
    - must have a `.config` extension on the file
    - able to modify some default settings using: `option_settings`
    - ability to add resources such as RDS, ElastiCache, DynamoDB, etc.
- Resources managed by .ebextensions get deleted if the environment is deleted

## Beanstalk Under the Hood

- Beanstalk relies on CloudFormation
- CloudFormation is used to provision other AWS services (infrastructure as code)
- use case: define CloudFormation resources in *.ebextensions* to provision anything you want

### Hands on

- multiple stacks, one for each environment. 
- each stack corresponds to a cloudformation configuration. 
- resources shows all resources created for a stack.

## Beanstalk Cloning

- clone an existing environment with the exact same configuration
- useful for deploying a 'test' version of your application
- all resources and configuration are preserved
    - load balancer type and configuration
    - RDS database type (*the data is not preserved*)
    - environment variables
- after cloning an environment, you can change settings

### hands on

- Select the environment, hit the 'actions' button, and select 'clone'
- can't change a lot whilst cloning, but fully customisable once cloned

## Beanstalk Migrations

*Load balancer:*
- cannot change load balancer type whilst creating beanstalk environment
- to migrate
    1. create a new environment with the same configuration except the load balancer (cannot be cloned)
    2. deploy the application onto the new environment
    3. perform a CNAME swap or Route 53 update

*RDS*:
- can be provisioned with Beanstalk, great for dev/test environments or use cases
- not great for production as the database lifecycle is tied to the beanstalk environment lifecycle
- best for production is to separately create an RDS database and provide the Beanstalk application with the connection string
- decouple RDS by:
    1. create a snapshot of RDS database
    2. go to RDS console and protect the RDS database from deletion
    3. create a new beanstalk environment, without RDS, and point your application to the existing RDS
    4. perform a CNAME swap (blue/green) or Route 53 update, then confirm it works
    5. terminate the old environment (RDS won't be deleted)
    6. delete the CloudFormation stack (in DELETE_FAILED state)

## Beanstalk with Docker

*Single Docker*
- run application as a single docker container
- provide:
    - *DOCKERFILE*: Beanstalk will build and run the Docker container
    - *Dockerrun.aws.json (v1)*: Describe where *already built* Docker image is (also include ports, volumes, logging, etc.)
- does not use ECS

*Multi Docker Container*
- run multiple container per EC2 instance in EB
- will create
    - ECS cluster
    - EC2 instances, configured to use the ECS cluster
    - load balancer (in high availability mode)
    - task definitions and execution
- requires a config Dockerrun.aws.json (v2) at the root of source code
- *Dockerrun.aws.json is used to generate the ECS task definition*
- Docker images must be pre-built and stored in ECR or another repository

## Beanstalk Advanced Concepts

*HTTPS*:
- idea: load SSL certificate onto the load balancer
- can be done from the Console (Beanstalk console, load balancer configuration)
- can be done from code (.ebextensions./securelistener-alb.config)
- SSL certificate can be provisioned using ACM or CLI
- must configure a security group rule to allow incoming port 443 (HTTPS port)

*Beanstalk redirect HTTP to HTTPS*:
- configure instances to redirect
- OR configure ALB with a rule (ALB only)
- make sure health checks are not redirected (return 200 OK)

*Web Servers*:
- where the EC2 instances and ELB operate

*Worker Environment*:
- long tasks should be offloaded to dedicated worker environment
- decoupling application into two tiers is common. 
- can define periodic tasks in a `cron.yaml` file

*Web servers and worker environments are more of a focus for the DevOps Certificate exam.*

*Custom Platform*:
- very advanced, allow you to define from scratch:
    - operating system
    - additional software
    - scripts that Beanstalk runs on these platforms
- use case: app language is incompatible with Beanstalk and doesn't use Docker
- to create own platform:
    - define an AMI using the `Platform.yaml` file
    - build thes platform using the *Packer software* (open source tools to create AMIs)
- *Custom Platform vs Custom Image (AMI)*
    - custom image is to tweak an *existing* Beanstalk platform
    - custom platform is to create an *entirely new* Beanstalk platform