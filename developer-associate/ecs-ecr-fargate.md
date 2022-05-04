# Containerisation

## Docker

- software development platform to deploy apps
- apps are packaged in *containers* that can be run on any operating system
- apps run the same regardless of where they're run
    - any machine, operating system, technology
- containers running on same machine don't impact each other

### Images
- stored in Docker Repositories
    - *public*: docker hub
        - base images for many technologies or operating systems
    - *private*: Amazon ECR (Elastic Container Registry)

### Docker vs Virtual Machine
- docker is 'sort of' a virtualisation tech, but not exactly
- resources are shared with the host vs many containers on one server

Dockerfiles describe how to build a docker image. Run a docker image to instantiate a container. push/pull to/from your docker repo.

To manage containers, need a container management platform
1. *ECS*: Amazon's platform
2. *Fargate*: Amazon's Serverless platform
2. *EKS*: Amazon's managed Kubernetes (open source)

## ECS (Elastic Container Service)

- help define how many tasks should run and how they should be run
- ensure the number of tasks desired is running across our fleet of EC2 instances
- can be linked to ELB/NLB/ALB if needed

## Clusters

- logical grouping of EC2 instances
- instances run the ECS agent (docker container)
- ECS agents register the instance to the cluster
- EC2 instances run a special AMI specifically for ECS

Security Groups do not matter when an EC2 instance registers with the ECS service. By default, Security Groups allow all outbound traffic.

### Hands on

- EC2 AMI ID is set so that we can automatically use the custom ECS AMI
- within a cluster you run instances. 
- container is linked to an EC2 instance. 
- CPU resource is shared across different containers on one instance
- ECS cluster comes with an ASG, use this to the control number of instances. 
- Cluster has an IAM role, an ECS role - basically it all gets spun up for you, so hit the public IP and ssh in if you want to see what it's doing

- *ecs.config*
    - `ecs_cluster` is how the instance knows what cluster to register to
    - Docker ECS agent container is what registers the EC2 instance in the cluster
    - `ECS_ENABLE_TASK_IAM_ROLE` controls the roles that ECS tasks can assume. TODO: double check this is accurate, the notes from the course are not clear.

## Task Definitions

- task definitions are metadata in JSON form to tell ECS how to run a docker container
- crucial information
    - image name
    - port binding for container and host
    - memory and CPU required
    - environment variables
    - networking information

### Hands On

- *task role* requires an IAM role to have access to things - if you can't access something, that's why
- *execution role*: allows ecs to create the task
- when you set up a task definition run a container, you need to select the image that you want to run, and the tag/version.
    - can limit memory. 
    - port mappings are about how things access something publically, within the container a specific port is exposed and you need to map them together so that the internet can access the container
- Add volumes to map EBS volumes

### hands on

- Service is configured within the running cluster. 
- replica means you run as many tasks as possible. 
- daemon automatically runs one container per EC2 instance
- *task placement*: how to place task on different ecs instances
- make sure you've got access to the ports in the security groups
- if you hard code the port mappings, you can only have one container running on an instance that is mapped to that port, so you'd need to scale your ECS instance using the ASG. Must be some way of programmitically exposing different ports.

### Service with Load Balancer

Task definitons don't specify host port, only container port. ALB will receive traffic and use *dynamic port forwarding* or *dynamic host port mapping* to spread load. Same tasks on same EC2 instance.

*This is a common exam question*

- leave host port empty, and it will be a random host port. `0` host port in json means undefined/random
- load balancers can only be added on service creation, theload balancer already needs to exist but you don't need a target group.
- load balancer needs to be able to talk to any ports on EC2 instances for dynamic port hosting on ECS instances, and to do so will use security groups. 
- Will create a new IAM role for you automatically on service creation

## ECR (Elastic Container Registry)

- create images from your own software and keep them private
- ECR access is controlled through IAM
- AWS CLI v1 login command
    - `$(aws ecr get-login --no-include-email --region eu-west-1)`
    - $() is because output should be run EXAM: common question as to what you should do with the stuff inside the ()
- AWS CLI v2 login command
    - `aws ecr get-login-password --region eu-west-1 | docker login --username AWS -- password-stdin 1234567890.dkr.ecr.eu-west-1.amazonaws.com`
    - get login password from the first part of the command, then pipe it into the docker login
- Docker push/pull
    - `docker push <full image url including ecr repository>`
    - `docker pull <full image url including ecr repository>`

*It's a common exam question regarding permissions to ECR, the answer is IAM policies*
*It's a common question that may require explaining the difference between the two login options*

### Hands On

- create a docker image from dockerfile using `docker build`
- repo doesn't already exist, need to create it yourself
- need to create a new task definition to use a new docker image. 
- need to use the full image name from AWS ECR. 
- task definition pulls from ECR using IAM. 
- rolling update slowly deploys task definitions over time. 
- need to wait for draining of instance traffic to occur, unless you want to kill the instances yourself.

*In the exam you can expect to be tested on how to push from different operating systems*

## Fargate

When launching an ECS cluster, you have to create EC2 instance. To scale, you need to add EC2 instances. ECS requires self managed infrastructure.

- Fargate is serverless
- there is no need to provision EC2 instances
- create task definitions, AWS will run containers.
- scale by increasing task number, no EC2s involved

### Hands on

- Networking only cluster is powered by Fargate. 
- Task role is for assigning role to task.
- Still need to create services. 
- Be aware on load balancers if you try to have the listener on the same path you'll have an issue so double check the load balancer rules

## ECS IAM Roles deep dive

need to attach an EC2 instance profile to the EC2 instance
- used by ECS agent
- makes API calls to the ECS service
- send container logs to CloudWatch Logs
- pull docker images from ECR

*ECS Task IAM role*
- specific role to each task with the minimum permissions
- different task roles for the different ECS services you run
- Task Role is defined in the *task definition*

## ECS Task Placement and Constraints

- when a task of type EC2 is launched, ECS must determine where to place it, with the constraints of CPU, memory and available ports
- when a service scales in, ECS needs to determine which task(s) to terminate
- to assist, define a *task placement strategy* and *task placement constraints*
- *only for ECS with EC2, not Fargate*
- task placement strategies are best effort
- uses following process to select container instances:
    1. identify instances that satisfy CPU, memory and port requirements in the task definition
    2. identify instances that satisfy the task placement constraints
    3. identify instances that satisfy the task placement strategies
    4. select instances for task placement

### Task Placement Strategies

#### Binpack
- place tasks based on least available amount of CPU or memory
- minimises number of instances in use (cost savings)
- packs all containers on EC2 as much as possible to save $$$

#### Random
- place task randomly

#### Spread
- place the task evenly based on specified value
- eg. instanceid, attribute:ecs.availability-zone

Can mix the placement strategies together.

EXAM: mostly focus on explaining the differences between the three strategies

### Task Placement Constraints

- *distinctInstance*: place each task on a different container instance
- *memberOf*: places task on instances that satisfy an expression
    - uses the Cluster Query Language

### Hands On

- constraints and strategies are defined on the creation of a service

## ECS Auto Scaling

*Service*: CPU and RAM is tracked in CloudWatch at the ECS service level
- *target tracking*: target a specific average CloudWatch metric
- *step scaling*: scale based on CloudWatch alarms
- *scheduled scaling*: based on predictable changes

ECS service scaling (task level) != EC2 auto scaling (instance level)

Fargate auto scaling is much easier to setup because it's serverless so you don't have to manage infrastructure

*Cluster Capacity Provider*
- used in association with a cluster to determine the infrastructure that a task runs on
    - for ECS and Fargate, FARGATE and FARGATE_SPOT capacity providers are added automatically
    - ECS on EC2, need to associate the capacity provider with an Auto Scaling Group
- When running a task or service, define a capacity provider strategy to prioritise in which provider to run
- allows capacity provider to automatically provision infrastructure for you

### Hands On

- in cluster, create capacity provider, need an ASG
- as soon as target capacity is met, new infrastructure will be spun up
- ASG needs to allow more infrastrucutre for capacity provider to add more instances
- switch to capacity provider strategy rather than Fargate or ECS on service creation on the cluster

## Data Volumes

### EC2 task strategies

#### EC2 + EBS volume
- EBS volume already mounted on EC2 instance
- allows docker containers to mount the EBS volume and extend the storage capacity of your task
- *problem*: if task moves from one EC2 instance to another, won't be the same EBS volume and data
- use case: 
    - mount data volume between different containers on same instance
    - extend temporary storage of a task

#### EC2 + EFS NFS
- works for both EC2 and Fargate tasks
- ability to mount EFS volumes onto tasks
- tasks launched in any availability zone will be able to share the same data in the EFS volume
- Fargate + EFS = serverless + data storage without managing servers
- use case: persistent multi-availability zone shared storage for containers

#### Bind Mounts
- works for both EC2 tasks (using local EC2 instance storage) and Fargate tasks (get 4gb for volume mounts)
- useful to share ephemeral storage between multiple containers part of the same ECS task
- great for 'sidecar' container pattern where the sidecar can be used to send metrics/logs to other destinations (separation of concerns)

*For the exam, you should know the difference between the 3 above options*

### Hands on

- added in the task definitions. 
- docker volume mounts EBS volume to extend storage of task.
- BindMount: no source path, AWS creates one for you by default.