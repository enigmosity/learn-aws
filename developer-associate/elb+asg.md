# Scalability and High Availability

**Scalability**: application can handle greater loads by adapting
- *vertical scalability*: increasing the size of the instance
    - very common for non-distributed systems, eg. databases
    - RDS, ElastiCache can scale vertically
    - usually a hardware limit to stop how far you can scale vertically
    - scale up/down
- *horizontal scalability*: increasing the number of instances running application
    - implies distributed systems
    - easy to horizontally scale with cloud computing
    - scale in/out

**High availability**: running system in multiple data centres, with the goal of surviving a data center being lost
- can be active (for horizontal scaling)
- can be passive (just by extras existing in more availability zones)


# ELB (Elastic Load Balancers)

**Load balancers** are servers that forward traffic to multiple servers downstream. This allows you to:
- spread load across downstream instances
- expose a single point of access (DNS) to your application
- seamlessly handle failures of downstream instances
- do regular instance health checks
- provide SSL termination (HTTPS) for your websites
- enforce stickiness with cookies
- high availability across zones
- separate public traffic from private traffic

Why use an ELB?
- *managed load balancer*
    - AWS will guarantee it works
    - AWS handles upgrades, maintenance, high availability, etc.
    - AWS provides only a few configuration options
- costs less to setup your own load balancer, but requires far more effort on your end
- integrated with many AWS offerings/services
    - EC2, EC2 ASGs, ECS
    - ACM, CloudWatch
    - Route 53, WAF, Global Accelerator

*Health Checks*:
- way for ELB to check health of instance
- enable balancer to know if instances are available
- done on a port and a route (eg. /healthcheck)
- if no OK response received, the instance is unhealthy and the ELB will not forward to it

*Security Groups*:
- Users can connect to load balancers, but instances can only be accessed by load balancers 
- Connect the security groups of your instances and your ELBs

## Classic Load Balancer (v1)
- supports TCP (layer 4), HTTP & HTTPS (layer 7)
- health checks can be TCP or HTTP based
- have a fixed hostname
- response timeout must be less than the interval time to create the load balancer.

To allow only the load balancer to interact with the EC2 instance, set the source of the HTTP rule as the security group of the load balancer.

## Application Load Balancer (v2)
- layer 7 (HTTP) only
- load balancing to multiple HTTP applications across machines (target groups)
- load balancing to mutiple applications on the same machine (eg. containers)
- support for HTTP/2 and WebSocket
- supports redirects (eg. from HTTP to HTTPS)
- routing tables to different target groups:
    - routing based on path in URL
    - routing based on hostname in URL
    - routing based on query strings and headers
- great fit for micro services and container-based applications
- a port mapping feature to redirect to a dynamic port in ECS
- comparison: multiple Classic Load Balancers required per application

*Target groups*:
- used to route requests to one of more registered target
- EC2 instances - HTTP
- ECS tasks - HTTP
- lambda functions - HTTP request translated into a JSON event
- IP addresses - must be private IPs

ALBs can route to multiple target groups
Healthchecks are at the target group level

- Fixed hostname
- application servers don't see IP of client directly
- true IP of client is inserted in the header *X-Forwarded-For*
- can also get port in *X-Forwarded-Port* and proto in *X-Forwarded-Proto*

*Only Network Load Balancer provides both static DNS name and static IP. While, Application Load Balancer provides a static DNS name but it does NOT provide a static IP. The reason being that AWS wants your Elastic Load Balancer to be accessible using a static endpoint, even if the underlying infrastructure that AWS manages changes.*

*Network Load Balancer has one static IP address per AZ and you can attach an Elastic IP address to it. Application Load Balancers and Classic Load Balancers as a static DNS name.*

## Network Load Balancer (v2)

- layer 4:
    - *forward TCP & UDP traffic to your instances*
    - handle millions of requests per second
    - less latency ~100ms (vs 400ms for ALB)
- NLB has *one static IP per availability zone* and supports assigning Elastic IP
- *NLB are used for extreme performance of TCP or UDP traffic*
- not included in free tier 

*Target groups*:
- EC2 instances
- IP addresses - must be private
- application load balancer

- When selecting an availability zone, it has an IPv4 address specifically assigned per availability zone
- Ensure that your security groups allow use of TCP, otherwise traffic will not be accepted

*Only Network Load Balancer provides both static DNS name and static IP. While, Application Load Balancer provides a static DNS name but it does NOT provide a static IP. The reason being that AWS wants your Elastic Load Balancer to be accessible using a static endpoint, even if the underlying infrastructure that AWS manages changes.*

## GateWay load balancer

- deploy scale and manage a fleet of 3rd party network virtual appliances in AWS.
- gateway load balancer guides incoming traffic to third party appliances for filtering or security or whatever else.
- layer 3 (Network layer) - IP packets
- combines:
    - *Transparent Network Gateway* - single entry/exit for all traffic
    - *Load Balancer* - distributes traffic to virtual appliances
- uses the **GENEVE protocol** on port 6081

Target groups:
- EC2 instances
- IP addresses - must be private

## Sticky Sessions
- possible to implement stickiness so that the same client is always redirected to the same isntance behind a load balancer.
- Classic Load Balancer and Application Load Balancer
- 'cookie' used for stickiness has a customisable expiration date
- use case: make sure user doesn't lose session data
- enabling stickiness may bring imbalance to the load over the backend EC2 instances
- set on the target group

### Cookie names

*Application-based cookie*:
- custom cookie:
    - generated by the target
    - can include any custom attributes required by the application
    - cookie name must be specified individually for each target group
    - don't use AWSALB, AWSALBAPP or AWSALBTG as these are reserved by the ELB
- application cookie:
    - generated by the load balancer
    - cookie name is AWSALBAPP
*Duration-based cookie*:
- generated by the load balancer
- cookie name is AWSALB for ALB, AWSELB for CLB
- expiry based on definition from the load balancer

## Cross Zone Load Balancing

- each load balancer instance distributes evenly across all registered instances in all AZs. 
- without, each Load Balancer *only* distributes across the instances within its availability zone.

ALB:
- always on, cannot be disabled
- no charges for inter AZ data

NLB:
- disabled by default
- charges for inter AZ data if enabled

CLB:
- disabled by default
- no charges for inter AZ data

## SSL Certificates

- allows traffic between clients and load balancer to be encrypted in transit
- SSL (secure sockets layer) is used to encrypt connections
- TLS (Transport Layer Security) is a newer version of SSL
- mostly TLS used now
- public certificates are issued by Certificate Authorities
- certificates have an expiration date that you set and replace them at that time
- load balancer uses an X.509 certificate
- certificates can be managed by ACM (AWS Certificate Manager)
- can also create and upload your own certificates
- HTTPS listener
    - specify a default certificate
    - add an optional list of certificates to support multiple domains
    - *clients can use SNI (server name indication) to specify hostname that they reach*
    - ability to set security policy to support older versions of SSL/TLS (legacy clients)

## SNI (Server Name Indication)
- solves problem of loading multiple SSL certs onto one web server to serve multiple websites
- a 'newer' protocol and requires the client indicate the hostname of the target server in the initial SSL handshake
- server will then find the correct cert, or return the default one
Note:
- only works for ALB & NLB, or CloudFront
- does not work for CLB

*Server Name Indication (SNI) allows you to expose multiple HTTPS applications each with its own SSL certificate on the same listener. Read more here: https://aws.amazon.com/blogs/aws/new-application-load-balancer-sni/*

CLB:
- supports one SSL certificate
- multiple CLB for multiples hostnames with multiple certificates

ALB & NLB:
- support multiple listeners with multiple SSL certicates
- uses SNI to make it work
- each rule can have a unique certificate

## Connection Draining / Deregistration Delay

*Connection Draining* for CLB
*Deregistration Delay* for ALB & NLB

- time to complete 'in-flight requests' while the instance is de-registering or unhealthy
- stops sending new requests to the EC2 instance which is deregistering
- 1 - 3600 seconds (default: 300s)
- can be disabled (0s)
- set to low value if requests are short
- tradeoff is time to take instances down

## Auto Scaling Groups
- Scale out (add EC2 instances) to match an increased load.
- Scale in (remove EC2 instances) to match a decreased load.
- Ensure a minimum and macimum numver of machines running
- automatically register new instances to a load balancer

*Minimum size*: number of instances guaranteed to be running
*Actual size/Desired capacity*: what is running at current time in ASG
*Maximum size*: how many instances can be added in scale out as required

*Attributes*:
- a launch configuration
    - AMI instance type
    - EC2 user data
    - EBS volumes
    - Security groups
    - SSH key pair
- Min/Max/Initial capacity
- network + subnets information
- load balancer information
- scaling policies (what will trigger a scale out/in)

*Auto scaling alarms*
- cloudwatch alarms can cause ASGs to scale
- alarms monitor a metric
- **metrics are computed for the overall ASG instances**
- based on the alarm:
    - can create scale-out policies
    - can create scale-in policies

*AutoScaling new rules*
- target average CPU usage
- number of requssts on the ELB per instance
- average network in
- average network out
These new rules are easier to setup and can make more sense to reason about

*Auto scale per custom metric*
1. send custrom metric from app on EC2 to CloudWatch (PutMetric API)
2. create CloudWatch alarm to react to low/high values
3. use the CloudWatch alarm as the scaling policy for the ASG

*Brain dump*

- scaling policies can be on CPU, network, custom metrics or a schedule
- ASGs use Launch configurations or Launch templates
- to update an ASG, provide a new launch configuration/template
- IAM roles attached to an ASG will get assigned to EC2 instances
- ASG are free, but you pay for the underlying instances launched
- instances under an ASG means that if they get terminated, the ASG will automatically create replacements
- ASG can terminate instances marked unhealthy by an ELB and then replace them
- can also have ELB healthchecks to ensure that the ELB is functioning too.

If you fire up an ASG, and it gets unhealthy instances and keeps replacing them, the EC2 instance is probably misconfigured and the healthcheck may be failing.

### Scaling policies:

*Dynamic scaling policies*:
- target tracking scaling
    - most simple and easy to set up
    - eg. want average ASG CPU to stay around 40%
- simple / step scaling
    - when a CloudWatch alarm is triggered, do x
- scheduled actions
    - anticipate a scaling based on known usage patterns

*Predictive scaling*: continuously forecast load and schedule scaling ahead

Good metrics to scale on:
- CPU Utilisation: avg. cpu use across instances
- Request count per target: make sure the number of requests per EC2 instance is stable
- Average network in/out if you're network bound
- any custom metric

*Scaling cooldown*:
- after a scaling activity happens, there is a cooldown period (default: 600s)
- during the cooldown period, the ASG will not launch or terminate additional instances (allowing for metrics to stabilise)
- advice: use ready-to-use AMI to reduce configuration time in order to be servering requests dater and reduce the cooldown period

can scale by percent of group and other useful things
you can install libraries and packages to easily stress test instances and check scaling responses
