# CICD

What is CICD: code pushes automatically go through automation right to production

*CodeCommit*: storing code
*CodePipeline*: automating pipeline from code to Beanstalk
*CodeBuild*: building and testing code
*CodeDeploy*: deploying the code to EC2 instances (not Beanstalk)
*CodeStar*: manage software development activities in one place
*CodeArtifact*: store, public and share software packages
*CodeGuru*: automated code review using Machine Learning

*Continuous Integration*:
- developers push code to a repository
- testing/build server checks code as soon as pushed
- developer checks feedback about tests and checks if they've passed/failed
- find bugs early and fix 'em early
- deliver faster as code is tested earlier and often
- deploy often 
- happier developers, with unblocked workflows

*Continuous Delivery*:
- deploy as soon as the changes are built and tested
- deployments happen often and quickly
- automated deployment

## CodeCommit

*Version control*: ability to understand various changes that have happened to code over time
- enabled by using a version control system like *git*
- repository can be synced to a local computer but usually uploaded to an online repository
- benefits:
    - collaboration
    - backed up code
    - fully viewable and auditable
- git can be expensive
- *CodeCommit*
    - private repositories
    - no size limit
    - fully managed, highly available
    - code only in AWS cloud account -> increased security and compliance
    - *security* (encrypted, access control, etc.)
        - interactions done using Git
        - *Authentication*
            - SSH keys - AWS users can configure SSH keys in IAM console
            - HTTPS - AWS CLI Credential helper or Git Credentials for IAM users
        - *Authorisation*
            - IAM policies to manage users/roles permissions to repositories
        - *Encryption*
            - repositories are automatically encrypted at rest using AWS KMS
            - encrypted-in-transit
        - *Cross-account access*
            - do not share SSH keys or AWS credentials
            - use an IAM role in AWS Account and use AWS STS
    - integrated with CI tools

### Hands on

- lives in developer tools
- notification rules can act when something occurs in your repository
- CodeCommit has specific credentials for Users in IAM

## CodePipeline

- Visual workflow to orchestrate CICD
- *Source*: CodeCommit, ECR, S3, BitBucket, GitHub
- *Build*: CodeBuild, Jenkins, CloudBees, TeamCity
- *Test*: CodeBuild, AWS Device Farm, 3rd party tools, etc.
- *Deploy*: CodeDeploy, Beanstalk, CloudFormation, ECS, S3, etc.
- *consists of stages:
    - each stage can have sequential actions and/or parallel actions
    - manual approval can be defined at any stage
- *Artifacts*: 
    - each pipeline stage can create artifacts
    - artifacts are stored in an S3 bucket and passed on to the next stage
- troubleshooting:
    - use CloudWatch events
    - if a stage fails, pipeline stops and can developers get information from the console
    - if the pipeline cannot perform an action, make sure the IAM service role attached has the required permissions on the IAM policy
    - CloudTrail can be used to audit AWS API calls

### Hands on

- create a new service role, each pipeline should probably have its own role for security reasons
- detection options: CloudWatch is recommended, considered to be faster
- as soon as the pipeline is created, it automatically runs
- action groups go within stages, *stages can have multiple action groups*, action groups can be sequential or parallel
- source determines what triggers the pipeline

## CodeBuild

- *source*: CodeCommit, S3, BitBucket, GitHub
- *build instructions*: code file *buildspec.yml* or insert manually in console (EXAM: best option is buildspec.yml and what you will be tested on)
- *output logs* can be stored in S3 oe CloudWatch logs
- use CloudWatch metrics to monitor build stats
- use CloudWatch Events to detect failed builds and trigger notifications
- use CloudWatch Alarms to notify if you need 'thresholds' for failures
- *build projects can be defined within CodePipeline or CodeBuild*

*For the exam,the best way to define build instructions is using the `buildspec.yml`*

Supported Environments: many standard languages, you can also create your own docker image that you need to support too if what you require doesn't already exist
- artifacts are output to an s3 bucket

*buildspec.yml*
- must be at root of the code
- environment: define environment variables
    - *variables*: plaintext
    - *parameter store*: variables stored in SSM Parameter Store
    - *secrets manager*: variables stored in AWS Secrets Manager
- phases: specify commands to run
    - *install*: install dependencies you may need for your build
    - *pre_build*: final commands to execute before building
    - *build*: actual build commands
    - *post_build*: finishing touches after the build has completed
- *artifacts*: what to upload to S3 (KMS encrypted)
- *cache*: files to cache (usually dependencies) to S3 for speeding up future builds 

*Local Build*
- in case of deep troubleshooting beyond logs
- can run locally (with docker installed)
- leverage the CodeBuild Agent

*Inside VPC*
- by default, CodeBuild containers are are launched outside of VPC
    - cannot access resources within VPCs
- can specify a VPC configuration
    - VPC id
    - subnet ids
    - security group ids
- then builds can access resources within the VPC (RDS, EC2, ALB, etc.)
- use cases: integration tests, data query, internal load balancers, etc.

CodeBuild containers are deleted at the end of their execution (success or failure). You can't SSH into them, even while they're running.

## CodeDeploy

- want to deploy application automatically to many EC2 instances
- only EC2 instances not managed by Beanstalk
- several ways to handle deployments using Open Source tools
- can use managed service of AWS CodeDeploy
- each EC2 instance/on-premises server must be running the agent 
- agent continously polling CodeDeploy for work to do
- application + *appspec.yml* is pulled from GitHub or S3
- EC2 instances will run the deployment instructions in the *appspec.yml*
- CodeDeploy Agent will report of success/failure of the deployment

*The Certified Developer exam is likely to test on what the EC2 instance and on-prem instances must be running for CodeDeploy to work.*

Primary Components
- *application*: unique name functions as a container
- *compute platform*: EC2/On-Premises, Lambda, or ECS
- *deployment configuration*: a set of deployment rules for success/failure
    - *EC2/On-Prem*: specify the minimum number of healthy instances for the deployment
    - *Lambda or ECS*: specify how traffic is routed to your updated versions
- *deployment group*: group of tagged EC2 instances (allows you to deploy gradually, or dev, test, prod...)
- *deployment type*: method used to deploy the application to a deployment group
    - *in-place deployment*: supports EC2/on-prem
    - *blue/green deployment*: supports EC2 instances only, Lambda, ECS
- *IAM instance profile*: give EC2 instances the permissions to access both S3/GitHub
- *application revision*: application code + *appspec.yml* file
- *service role*: an IAM role for CodeDeploy to perform operations on EC2 instances, ASGs, ELBs
- *target revision*: most recent version you want to deploy to a deployment group

### appspec.yml file

- *files*: how to source and copy from S3/GitHub to the filesystem
    - source
    - destination
- *hooks*: set of instructions to do to deploy the new versions (can have timeouts), order is:
    1. ApplicationStop - stop of already running application
    2. DownloadBundle
    3. BeforeInstall
    4. Install
    5. AfterInstall
    6. ApplicationStart
    7. *ValidateService* - run last to ensure that service is deployed successfully

### Deployment Configuration
Configurations:
- *one at a time*: one EC2 instance at a time, if one instance fails, deployment stops
- *half at a time*
- *all at once*: quick but no healthy host, there is downtime, good for dev
- *custom*: min healthy hosts = 75%

Failures:
- EC2 instances stay in 'failed' state
- new deployments will first be deployed to failed instances
- *to rollback, redeploy old deployment or enable automated rollback for failures*

Deployment Groups:
- set of *tagged* EC2 instances
- directly to ASG
- mix of ASG/Tags so build deployment segments
- customisation in scripts with DEPLOYMENT_GROUP_NAME environment variables

Blue/Green deployments are also possible

### CodeDeploy for EC2

- define how to deploy the application using *appspec.yml* & deployment strategy
- do in-place updates to update fleets of EC2 instances
- can use hooks to verify the deployment after each deployment phase

### CodeDeploy for ASG

- in-place deployment
    - updates existing EC2 instances
    - newly created ec2 instances by an asg will also get automated deployments
- blue/green deployments
    - a new ASG is created (with settings copied)
    - choose how long to keep old EC2 instances (the old ASG)
    - must be using an ELB

*Rollback*: redeploy a previously deployed revision of your application
- deployments can be rolled back
    - automatically: when a deployment fails or rollback when a CloudWatch Alarm threshold is met
    - manually
- disable rollbacks: do not perform rollbacks for this deployment
- *if a rollback happens, CodeDeploy redeploys the last known good revision as a new deployment (not a restored version)*

*The Certified Developer Associate exam is likely to test on redeployments*

## CodeStar 

- *integrated solution that groups:* GitHub, CodeCommit, CodeBuild, CodeDeploy, CloudFormation, CodePipeline, CloudWatch, etc.
- quickly create 'cicd-ready' projects for EC2, Lambda, Elastic Beanstalk
- supports some languages: C#, Go, HTML 5, Java, Node.js, PHP, Python, Ruby
- issue tracking integration with JIRA/GitHub issues
- ability to integrate with Cloud9 to obtain a web IDE
- one dashboard to view all your components
- free service, pay only for the underlying usage of other services
- limited customisation

### hands on

- uses iam so needs a service role
- need to use a project template, pick your language and platform
- CodeStar needs to know where you want to store your code (CodeCommit or GitHub)
- backed by CloudFormation to set up resources

## CodeArtifact

- packages depend on each other to be built, and new ones are created
- storing and retrieving dependencies is called *artifact management*
- traditionally you need to setup your own artifact management system
- *CodeArtifact* is a secure, scalable and cost-effective artifact management system
- works with common dependency management tools like Maven, npm, yarn, Nuget etc.
- *devs and CodeBuild can then retrieve dependencies straight from CodeArtifact*
- artifacts from public repositories can also be pulled into the CodeArtifact repo

## CodeGuru

- a ML-powered service for *automated code reviews* and *application performance recommendations*
- provides two functionalities
    - *CodeGuru Reviewer*: automated code reviews for static code analysis (dev)
        - identify critical issues, security vulnerabilities, and hard-to-find bugs
        - uses Machine Learning and automated reasoning
        - hard-learned lessons across millions of code reviews on 1000s of open-source and Amazon repositories
        - supports Java and Python
        - integrates with GitHub, BitBucket and CodeCommit
    - *CodeGuru Profiler*: visbility/recommendations about application performance during runtime (production)
        - helps understand runtime behaviour of application
        - features:
            - identify and remove code inefficiencies
            - improve application performance
            - decrease compute costs
            - provides heap summary (identifying objects using memory)
            - anomaly detection
        - support applications running on AWS or on-prem
        - minimal overhead on application
    