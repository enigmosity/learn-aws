# DNS (Domain name system)

- translates human friendly hostnames into machine IP address
- backbone of the internet
- uses a hierarchical naming structure
    - .com
    - example.com
    - www.example.com
    - api.example.com

*Domain Registrar*: where domain names are registered
*DNS records* : A, AAAA, CNAME, NS, etc.
*Zone file*: contains DNS records
*Name server*: resolves DNS queries
*Top Level Domain* (TLD): .com, .us, .org
*Second level domain* (SLD): amazon.com, google.com

*FQDN (fully qualified domain name)* = protocol://domainname.subdomain.secondleveldomain.topleveldomain.

# Route 53

- highly available, scalable, fully managed and Authoritative DNS
    - *authoritative*: the customer can update the DNS records
- Route 53 is also a Domain Registrar
- ability to check the health of your resources
- only AWS service which provides a 100% availability SLA
- 53 is a reference to the traditional DNS port

## Records
- define how you want to route traffic for a domain
- each record contains
    - domain/subdomain name
    - record type
    - value
    - routing policy - how route 53 responds to queries
    - TTL (time-to-live) - the amount of time the record is cached at DNS resolvers
- Route 53 supports DNS types:
    - A / AAAA / CNAME / NS (these four are important for the Developer Associate certification)
    - CAA / DS / MX / NAPTR / PTR / SOA / TXT / SPF / SRV (*advanced so not required*)

## Record Types
- *A*: maps hostname to IPv4
- *AAAA*: hostname to IPv6
- *CNAME*: maps hostname to hostname
    - target is a domain name which must have an A or AAAA record
    - can't create a CNAME record for the top node of a DNS namespace (zone apex)
    - eg can't for `example.com` but can for `www.example.com`
- *NS*: name servers for the hosted zone
    - controls how traffic is routed for a domain

## Hosted Zones
- container for records that define how to route traffic to a domain and it's subdomains
- *public hosted zones*: contain records that specify how to route traffic on the internet
- *private hosted zones*: contain records that specify how you route traffic within one or more VPCs (private domain names)
- $0.50 per month per hosted zone

## Registering a domain name

- pick your domain name
- when you make it, definitely ensure that you enable privacy so that people do not ping you too much

## Creating Records

nslookup for windows. on linux, need to install nslookup. lets you see what the record resolves to.

## Record TTL (Time-To-Live)

Caching the result of DNS query for later. Don't expect DNS to change that much

- *High TTL*:
    - less traffic on Route 53
    - possibly outdated records
- *Low TTL*:
    - more traffic on Route 53 ($$)
    - records outdated for less time
    - easy to change records
- can change TTL when going to change the record so it has less impact
- except Alias records, TTL is mandatory for each DNS record

If you do a DNS query, you will also see the TTL number

## CNAME vs Alias

AWS resources expose an AWS hostname

### CNAME
- allow you to point a hostname to any other hostname
- **only for a non-root domain**

### Alias
- points a hostname to an AWS resource
- works for root domain and non root domain
- free of charge
- built-in healthcheck

#### Alias Records
- maps a hostname to an AWS resource
- an extension to DNS functionality
- automatically recognises changes in the resources IP addresses
- unlike CNAME, it can be used for the top node of a DNS namespace (zone apex) 
- Alias record is always of type A/AAAA for AWS resources (IPv4/IPv6)
- cannot set the TTL

#### Alias Record Targets
- ELB
- CloudFront Distributions
- API gateway
- Elastic beanstalk environments
- S3 websites
- VPC interface endpoints
- Global Accelerator accelerator
- Route 53 record in the same hosted zone

*Cannot set an ALIAS reocrd for an EC2 DNS name*

## Routing Policies

- define how Route 53 responds to DNS queries
- don't get confused by the word "routing"
    - not the same as load balancer routing which routes traffic
    - *DNS does not route any traffic, only responds to the DNS queries*
- Route 53 supports the following routing policies:
    - *simple*
        - typically route traffic to a single resource
        - can have multiple values in the same record
        - *if multiple values are returned, a random one is chosen by the **client***
        - when Alias enabled, specify only one AWS resource
        - can't be associated with health checks
    - *weighted*
        - control the percentage of the requests that go to each specific resource
        - assign each record a relative weight
            - weights don't need to sum to 100
            - traffic % = (weight for a specific record / sum of all the weights for all records)
        - DNS records must have the same name and type
        - can be associated with health checks
        - use cases: load balancing between regions, testing new application versions, etc.
        - *assign a weight of 0 to a record to stop sending traffic to a resource*
        - *if all records have a weight of 0, then all records will be returned equally*
        - add multiple records all weighted under the same record name
    - *failover (active-passive)*
        - R53 in the middle, primary record is associated with a healthcheck, if primary healthy then return route to primary. if not, secondary address is served
    - *latency based*
        - redirect to the resource that has the least latency close to user
        - super helpful when latency for users is a priority
        - *latency is based on traffic between users and AWS regions*
        - Germany users may be directed to the US (if that is the lowest latency)
        - can be associated with health checks (has a failover capability)
        - add the records all under the same name with the latency routing policy
    - *geolocation*
        - based on where user is located
        - specify by continent, country or us state (if overlap, most precise location used)
        - should create a 'default' record in case there's no match on location
        - uses: website localisation, content distribution restriction, load balancing
        - can be associated with health checks
    - *multi-value answer*
        - use when you want to route traffic to multiple resources
        - R53 returns multiple values/resources
        - can be associated with health checks
        - up to 8 healthy records are returned for each multi-value query
        - *multi-value is not a substitute for having an ELB*
        - client side picks which record it wants to interact with
    - *geoproximity (using route 53 traffic flow feature)*
        - route traffic to resources based on geographic location of users and resources
        - ability to shift more traffic to resources based on the defined bias
        - to change the size of the geographic region, specify bias values:
            - to expand (1-99): more traffic to the resource
            - to shrink (-1--99): less traffic to the resource
        - resources can be
            - AWS resources (specify AWS region)
            - non-AWS resources (specify lat & long)
        - must use R53 Traffic Flow (advanced) to use this feature

## Health Checks
- only for public resources
- integrated with CloudWatch metrics
- health check -> Automated DNS failover
    1. health checks that monitor an endpoint
    2. health checks that monitor other health checks (calculated health checks)
    3. health checks that monitor CloudWatch Alarms

### Monitor an Endpoint

- 15 global health checkers will check the endpoint health
    - healthy/unhealthy threshold (default threshold is 3 requests)
    - interval - 30 seconds (can reduce time for more $$)
    - supported protocol: HTTP, HTTPS and TCP
    - if > 18% of health checkers report the endpoint is healthy, Route 53 considers it *Healthy*. Otherwise it's *Unhealthy*
    - ability to choose which locations you want Route 53 to use
- health checks pass only when the endpoint responds with 2XX or 3XX codes
- can be setup to pass/fail based on the text in the first 5120 bytes of the response
- need to configure router/firewall to allow incoming requests from Route 53 health checkers

### Calculated health checks

- combine the result of multiple HCs into one
- use *OR*, *AND* or *NOT*
- can monitor up to 256 child health checks
- specify how many of the health checks need to pass to make the parent pass
- usage: perform maintenance to your website without causing all health checks to fail

### Private hosted zones

- R53 health checkers are outside the VPC
- can't access private endpoints
- can create a CloudWatch Metric and associate a CloudWatch Alarm then create a health check for the alarm itself

### Hands on:
- you can test health checks by messing with the security groups.

## Traffic Flow

- simplify the process of creating and maintaining records in large and complex configurations
- visual editor to manage complex routing decision trees
- configurations can be saved as Traffic Flow Policies
    - can be applied to different R53 Hosted Zones (domain names)
    - supports versioning

### Hands On:
- Use the UI to understand where the line will then be drawn for users.

## Domain Registrar vs DNS Service

- buy or register domain name with a domain registrar typically with annual charges
- the domain registrar usually provides you with a DNS service to manage your DNS records
- you can use another DNS service to manage your DNS records

### To use a 3rd party registrar with R53
1. created a Hosted Zone in R53
2. Update Name Servers records on the 3rd party website to use R53 Name Servers