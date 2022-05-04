# CloudFront

- *Content delivery network* (CDN)
- improves read performace, content is cached at the edge
- Points of presence (edge locations)
- DDoS protection, integration with Shield, AWS Web Application Firewall
- can expose external HTTPS and can talk to internal HTTPS backends
- information is taken from region, stored in a cache at an edge location in the client region

## Origins:
- *S3 bucket*
    - distributing files, caching them at the edge
    - enhanced security with *CloudFront Origin Access Identity* (OAI)
    - CloudFront can be used as an ingress (to upload files to S3)
- *Custom Origin (HTTP)*
    - application load balancer
        - ALB security group must allow public ip of all edge locations
    - EC2 instance
        - security group must allow access through group to IP of all edge locations
    - S3 website (first enable the bucket as a static S3 website)
    - any HTTP backend you want

## Geo Restriction:
- whitelist
- blacklist
- country detrmined using third party geo IP database
- use case: copyright laws to control access to content

## CloudFront vs S3 CRR

*CloudFront*:
- global edge network
- files attached with a TTL
- *great for static content that must be available everywhere*

*S3 Cross Region Replication*:
- must be setup for each region you want replication in
- files updated in near real-time
- readonly
- *great for dynamic content that needs to be available at low-latency in few regions*

### Hands on

- CloudFront is global, not region limited
- Origin Access Identity for S3 bucket access

## Caching

- based on
    - headers
    - session cookies
    - query string parameters
- cache lives at each CloudFront Edge Location
- maximise cache hit rate to minimise requests on origin
- control TTL (0s - 1y): can be set by origin using `Cache-control` or `Expires` headers
- can invalidate part of cache using the `CreateInvalidation` API
- in CloudFront, separate static and dynamic distribution. 
    - *Static requests*: no headers/session caching rules, maximise caching hits
    - *Dynamic requests*: cache based on correct headers and cookies

### hands on

- *Behaviours* is where you check the policies and behaviours of caching in CloudFront, TTL is in these policies for example. 
- Invalidate objects in your CloudFront cache to force the retrieval of new responses from the end service.

## Geo Restriction

- restrict who can access your distribution
    - *whitelist*: users only access content in approved list of countries
    - *blacklist*: block users from specific countries
- location is determined using 3rd party service
- use case: copyright laws
- added through editing policies

- HTTPS
    - *Viewer Protocol policy*
        - redirect HTTP to HTTPS (recommended IRL)
        - or use HTTPS only
    - *Origin Protocol policy (HTTP or S3)*
        - HTTPS only
        - or Match Viewer (HTTP => HTTP & HTTPS => HTTPS)
    - S3 bucket 'websites' don't support HTTPS

## Signed URL / Cookies

- distribute paid shared content to premium users all over the world
- use CloudFront signed url/cookie. Attach a policy with:
    - url expiration
    - IP ranges to access the data from
    - trusted signers (which AWS accounts can create signed URLs)
- how long should the URL be valid for?
    - *shared content*: make it short
    - *private content*: make it last for years

- *Signed URL*: access to individual files (one signed URL per file)
- *Signed Cookies*: access to multiple files (one signed cookie for many files)

*CloudFront pre-signed URL*:
- allow access to a path, no matter the origin
- account managed key pair, only root can manage
- can filter by IP, path, date, expiration
- can leverage caching features

*S3 pre-signed URL*:
- issue a request as the person who pre-signed the URL
- uses the IAM key of the signing IAM principal
- limited lifetime

Two types of signers:
1. trusted key group
    - leverage APIs to create and rotate keys (and IAM for API security)
2. AWS account contains a CloudFront key pair
    - need to manage keys using root account and AWS console
    - not recommended because you shouldn't use root account for this

In CloudFront distribution, create one or more trusted key groups
- generate your own public/private key
    - private key used by your applications to sign urls
    - public key (uploaded) is used by CF to verify URLs

### Hands on

- need to generate your own RSA key somewhere.
- save the private key somewhere.
- huck the public key into CloudFront.
- Create a keygroup and add up to 5 public keys.
- Public keys can then be used to sign urls.

## Advanced Concepts

### Pricing

Edge locations all around the world, cost of data out per edge location varies

#### Price Classes
- reduce number of edge locations for cost reductions
- three price class
1. All: all regions - best performance
2. 200: most regions, excluding the most expensive
3. 100: only the least expensive regions

### Multiple Origin

- may want to route to different kind of origins based on content type
- eg. based on path pattern:
    - /images/*
    - /api/

### Origin Groups:
- increase high-availability and do failover
- *origin group*: one primary and one secondary origin
- if primary fails, secondary is used

### Field level encryption
- protect user sensitive information through the application stack
- additional layer of security along with HTTPS
- sensitive information is encrypted at edge close to user
- uses asymetric encryption
- usage:
    - specify set of fields in POST requests that you want to be encrypted (up to 10 fields)
    - specify the public key to encrypt them
    - web server has private key, only web server can decrypt. requires custom application logic