# API Gateway

## Overview

Create REST APIs that are exposed to the internet. Proxy requests through to Lambda Functions
- Lambda+Gateway: no infrastructure to manage
- support for the WebSocket Protocol
- handle API versioning (v1,v2)
- handle different environments (dev, test, production)
- handle security (authentication and authorisation)
- create API keys, handle request throttling
- swagger/open API import to quickly define APIs
- transform and validate requests and responses
- generate SDK and API specifications
- cache API responses

### Integrations
- Lambda Function
    - invoke a Lambda Function
    - easy way to expose a REST API backed by a Lambda Function
- HTTP
    - expose HTTP endpoints in the backend
    - Why? Add rate limiting, caching, user authentication, API keys, etc.
- AWS Service
    - expose any AWS API through the Gateway

### Endpoint Types
- *Edge-Optimised* (default): for global clients
    - requests are routed through the CloudFront Edge locations (which improves latency)
    - the API Gateway still lives in only one region
- *Regional*:
    - for clients within the same region
    - could manually combine with CloudFront (more control over the caching strategies and the distribution)
- *Private*:
    - can only be accessed from your VPC using an interface VPC endpoint (ENI)
    - use a resource policy to define access

### hands on
- the Lambda Function that you want to use needs to exist before you can call it in the API GateWay
- max API gateway timeout is 29 seconds
- need to deploy the API for it to be exposed publically to the internet

## Stages and Deployment

- making changes in the API Gateway does not mean they're in effect
- need to make a *deployment* for them to be in effect
- common source of confusion
- changes are deployed to *stages*
- use the naming scheme you like for stages
- each stage has its own configuration parameters
- stages can be rolled back as a history of deployment is kept
- new stages use different URLs

### Stage Variables
- like environment variables for API Gateway
- use them to change often changing configuration variables
- can be used in:
    - lambda function arn
    - http endpoint
    - parameter mapping templates
- use cases:
    - configure HTTP endpoints that stages talk to
    - pass config parameters to AWS Lambda through mapping templates
- stage variables are passed to the 'context' object in AWS Lambda

Common use case:
- stage variable to indicate the corresponding Lambda alias
- API gateway will automatically invoke the right Lambda Function

### Hands On 
- pass stage variables into the Lambda Function using `${stagevariables}`

## Stage Configuration Hands On

- define cache at the stage level
- define method throttling to ensure it is not called too often
- firewall rules with WAF
- certificate setup occurs here as well
- logs and tracing for CloudWatch, access logging
- integration with X-Ray
- SDK generation
- use export to export the API as Swagger or OpenAPI3 documentation
- deployment history
- documentation history

## Canary Deployments
- enable canary deployment to receive traffic for a stage
- choose the % of traffic the canary channel receives
- metrics and logs are separate for better monitoring
- possible to override stage variables for canary deployments
- blue/green deployment with AWS Lambda & API Gateway

### Hands on
- hit the canary tab and hit create
- need to deploy the canary change under actions, selecting the canary to deploy
- 'promote canary' to fully deploy the canary to 100%

*The Developer Associate exam will test on Stage and Deployment concepts, but not the implementations.*

## Integration Types and Mapping

### Integration Types
- Mock
    - API gateway returns a response without sending the request to the backend
- HTTP / AWS (Lambda & AWS Services)
    - must configure both the integration request and integration response
    - setup data mapping using *mapping templates* for the request and response
- AWS_PROXY (Lambda proxy)
    - incoming request from the client is the input to Lambda
    - the Lambda Function is responsible for the logic of request/response
    - *no mapping template, headers, query string parameters, etc. are passed as arguments*
- HTTP_PROXY
    - no mapping template
    - HTTP request is passed to the backend
    - HTTP response from the backend is forwarded by the API Gateway

### Mapping templates
- can be used to modify request/responses
- rename/modify *query string parameters*
- modify *body content*
- *add headers*
- use *velocity template language (VTL)*: for loop, if, etc.
- filter output results - remove unnecessary data

EXAM: mapping soap api to rest is a common one

*For the Developer Associate exam, mapping a SOAP API request to a REST API endpoint in API Gateway is a common question*

#### SOAP to REST example

*SOAP API are XML based, REST is JSON based.*

Gateway should:
- extract data from the request: either path, payload or header
- build SOAP message based on request data (mapping template)
- call SOAP service and receive XML response
- transform XML response to desired format (like JSON), and respond to user

#### Query string parameters example
- mapping template to take queries and pass them to lambdas
- variables can be renamed when mapped to anything you like

### hands on
- want to change what the integration response is for mapping
- use a template to determine what gets changed

## Swagger and Open API 3
- common way of defining REST APIs using API definition as code
- import existing Swagger/Open API 3.0 spec to API Gateway
    - method
    - method request
    - integration request
    - method response
    - + AWS extensions for API Gateway and setup every single option
- can export current API as Swagger / OpenAPI spec
- Swagger can be written in YAML or JSON
- using Swagger we can generate SDK for applications

## Gateway Caching

- caching reduces the number of calls made to the backend
- default TTL is 300 seconds (min 0s, max 3600s)
- *caches are defined per stage*
- possible to override cache settings *per method*
- cache encryption option
- cache capacity between 0.5GB and 237GB
- cache is expensive, makes sense in production uses, not necessarily in dev/test uses

### Cache invalidation
- able to invalidate the entire cache immediately
- clients can invalidate the cache with *header: Cache-Control: max-age=0* (with IAM authorisation)
- if you don't impose an InvalidateCache policy (or choose require authorisation check in the console), any client can invalidate the API cache

## Usage Plans & API Keys

- if you want to make an API available as an offering for $$ for your customers
- *usage plan*
    - who can access one or more deployed API stages and methods
    - how much and how fast they can access them
    - uses API keys to identify API clients and meter access
    - configure throttling limits and quota limits that are enforced on individual clients
- *API keys*
    - alphanumeric string values to distribute to your customers
    - can use with usage plans to control access
    - throttling limits are applied to the API keys
    - quota limits the overall number of maximum requests
    - *to configure a usage plan*
        1. create one or more APIs, configure the methods to require an API key and deploy the APIs to stages
        2. generate or import API keys to distribute to application developers who will be using your API
        3. create the usage plan with the desired throttle and quota limits
        4. *associate API stages and API keys with the usage plan*
        - callers of the API must supply an assigned API key in the `x-api-key` header in requests to the API

### hands on
- extension allows you to extend what the API key can do if required
- can see data for how much an API key is used

## Logging and Tracing

- CloudWatch Logs
    - enable CloudWatch logging at the stage level with Log Level
    - can override settings on a per API basis
    - logs contain information about requests/response bodies
- X-Ray
    - enable tracing to get extra information about requests in API Gateway
    - X-Ray + API Gateway + AWS Lambda gives you the full picture
- Cloudwatch Metrics
    - metrics are by stage, with the possibility to enable detailed metrics
    - *CacheHitCount* and *CacheMissCount*: efficiency of the cache
    - *Count*: total number of API requests in a given period
    - *IntegrationLatency*: time between when API Gateway relays a request to the backend and when it receives a response from the backend
    - *Latency*: time between when API Gateway receives a request from a client and when it returns a response to the client, the latency includes the integration latency and the other API Gateway overhead.
    - *4XXError* (client side) & *5XXError* (server side)

*Logging and tracing in API Gateway is important for the Developer Associate exam.*

## Gateway Throttling
- *Account limit*
    - API gateway throttles requests at 10000 requests/s across all API
    - soft limit that can be increased upon request
- in case of throttling => *429 Too Many Requests* (retriable error)
    - can set *Stage limit & method limits* to improve performance
    - can define *usage plans* to throttle per customer
- *one API that is overloaded, if not limited, can cause other APIs to also be throttled*

## Gateway Errors
- *4XX means Client errors*
    - 400: bad request
    - 403: access denied, WAF filtered
    - 429: quota exceeded, throttle
- *5XX means Server errors*
    - 502: bad gateway exception, usually for an incompatible output returned from a Lambda proxy integration backend and occasionally for out-of-order invocations due to heavy loads
    - 503: service unavailable exception
    - 504: integration failure - eg. endpoint request timed-out exception *API gateway requests time out after max 29 seconds*

## CORS (Cross-Origin Resource Sharing)

- CORS must be enabled when you receive API calls from another domain
- the OPTIONS pre-flight request must contain the following headers
    - Access-Control-Allow-Methods
    - Access-Control-Allow-Headers
    - Access-Control-Allow-Origin
- CORS can be enabled through the console

## Authentication and Authorisation

IAM Permissions
- create an IAM policy authorisation and attach to user/role
- *Authentication = IAM | Authorisation = IAM policy*
- good to provide access within AWS (EC2, Lambda, IAM users, etc.)
- leverages 'Sig v4' capability where IAM credentials are in headers

Resource Policies
- allow you to set a JSON policy on an API Gateway to define who and what can access it
- *allow for cross account access (combined with IAM security)*
- allow for a specific source IP address
- allow for a VPC endpoint

Cognito User Pools
- fully managed user lifecycle, token expires automatically
- API Gateway verifies identity automatically from Cognito
- no custom implementation required
- *Authentication = Cognito User Pools | Authorisation = API Gateway Methods*

Lambda Authoriser
- *token based authoriser* (bearer token) - eg. JWT or Oauth
- a *request parameter-based* Lambda authoriser (*headers, query string,* stage variable)
- Lambda must return an IAM policy for the user, result policy is cached
- *Authentication = External | Authorisation = Lambda Function*

### Summary
- IAM:
    - great for users/roles already within AWS account + resource policy for cross account
    - handle authentication + authorisation
    - leverages signature v4
- Custom Authoriser
    - great for 3rd party tokens
    - very flexible in terms of what IAM policy is returned
    - handle authentication verification + authorisation in the Lambda Function
    - pay per Lambda invocation, results are cached
- Cognito User Pool
    - manage your own user pool (can be backed by 3rd party logins)
    - no need to write any custom code
    - must implement authorisation in the backend

*For the Developer Associate exam, the summary should include most of the required knowledge.*

## HTTP vs Rest API
- HTTP APIs
    - low-latency, cost-effective AWS Lambda proxy, HTTP proxy APIs and private integration (no data mapping)
    - support OIDC and OAuth 2.0 authorisation, and built-in support for CORS
    - no usage plans and API keys
- REST APIs
    - all features except Native OpenID Connect/OAuth 2.0

## Websocket API

- what's Websocket?
    - two-way interactive communication between a user's browser and a server
    - server can push information to the client
    - enables *stateful* application use cases 
- WebSocket APIs are often used in *real-time applications* such as chat apps, collaboration platforms, multiplayer games and financial trading platforms
- works with AWS services (Lambda, DynamoDB) or HTTP endpoints

### Connecting to the api
- websocketurl
- client establishes a connection with the websocket API
- is a persistent connection
- connectionid is re-used

server to client messaging
- connection URL callback which allows communication with the client

Routing
- incoming JSON messages are routed to a different backend
- if no routes, sent to $default
- request a *route selection expression* to select the field in JSON to route from
- the result is evaluated against the route keys available in API Gateway
- route is then connected to the backend setup through API gateway

## Architecture

- create a single interface for all microservices
- use API endpoints with various resources
- apply a domain name and SSL certificates
- can apply forwarding and transformation rules at the API Gateway level