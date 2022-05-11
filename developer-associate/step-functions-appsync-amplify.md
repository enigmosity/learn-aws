# Step Functions

- model *workflows* as *state machines (one per workflow)*
    - order fulfillment, data processing
    - web applications, any workflow
- written in JSON
- visualisation of the workflow and the execution of the workflow, as well as history
- start the workflow with an SDK call, API gateway or Event Bridge (CloudWatch Event)

## Task States
- do some work in your state machine
- invoke one AWS service
    - invoke a Lambda Function
    - run an AWS batch job
    - run an ECS task and wait for it to complete
    - insert an item from DynamoDB
    - publish message to SNS or SQS
    - launch another Step Function workflow
- run one Activity
    - EC2, Amazon ECS, on-premise
    - Activities poll the Step Functions for work
    - Activities send results back to Step Functions

### States
- *choice state*: test for a condition to send to a branch (or default branch)
- *fail or succeed state*: stop execution with failure or success
- *pass state*: simply pass input to output or inject some fixed data, without performing work
- *wait state*: provide a delay for a certain amount of time or until a specified date/time
- *map state*: dynamically iterate steps
- *parallel state*: begin parallel branches of execution

> Step Functions capture and clearly represent the logic behind the decision tree

## Error Handling

- any state can encounter runtime errors for various reasons
    - state machine definition issues
    - task failures
    - transient issues
- use *retry* (to retry failed state) and *catch* (transition to failure path) in the state machine to handle errors instead of inside the application code
- predefined error codes:
    - *States.ALL*: matches any error name
    - *States.Timeout*: task ran longer than *TimeoutSecond* or no heartbeat was received
    - *States.TaskFailed*: execution failure
    - *States.Permissions*: insufficient privileges to execute the code
- the state may report its own errors

### Retry (task or parallel state)
- allows definition of what happens and how many times to retry based on errors
- evaluated from top to bottom
- *ErrorEquals*: match a specific kind of error
- *IntervalSeconds*: initial delay before retrying
- *BackoffRate*: multiply delay after each retry
- *MaxAttempts*: default to 3, set to 0 for never retry
- when max attempts is reached, the *Catch* kicks in

### Catch (task or parallel state)
- evaluated top to bottom
- *ErrorEquals*: match a specific kind of error
- *Next*: state to send to
- *ResultPath*: path that determines what input is sent to the state specified in the Next field
    - use '$' to include the error in the input
    - how you pass the error into the next task

## Standard vs Express

Standard: longer, slower throughput
Express: fast, high-throughput workflows

# AppSync

- managed service that uses *GraphQL*
- *GraphQL* makes it easy for applications to get exactly the data they need
- includes combining data from *one or more sources*
    - NoSQL data stores, relational databases, HTTP APIs, etc.
    - integrates with DynamoDB, Aurora, ElasticSearch, etc.
    - custom sources with AWS Lambda
- retrieves data in *real-time with WebSocket or MQTT on WebSocket*
- for mobile apps: local data access and data synchronisation
- all starts with uploading one *GraphQL schema*

## Security
- four ways can authorise applications to interact with AWS AppSync GraphQL API:
    - *API_KEY*
    - *AWS_IAM*: IAM users/roles/cross-account access
    - *OPENID_CONNECT*: OpenID Connect provider/ JSON Web Token
    - *AMAZON_COGNITO_USER_POOLS*
- for custom domain and HTTPS, use CloudFront in front of AppSync

*For the Developer Associate exam, it's important to know that AppSync is GraphQL, that it may involve WebSockets, and can be used for mobile application offline data synchronisation.*

# Amplify

- tools to get started creating mobile and web applications
- 'Elastic Beanstalk for mobile and web applications'
- must have features such as *data storage, authentication, storage, and machine learning* all powered by AWS services
- *front-end libraries* with ready to use components for React.js, Vue, Javascript, iOS, Android, Flutter, etc.
- incorporates AWS best practices for reliability, security and scalability
- build and deploy with Amplify CLI or Studio
- important features
    - *authentication*
        - leverages Amazon Cognito
        - user registration, authentication, account recovery and other operations
        - support MFA, Social Sign-in, etc.
        - pre-built UI components
        - fine-grained authorisation
    - *data store*
        - leverages Amazon AppSync and  DynamoDB
        - work with local data and have *automatic synchronisation to the cloud* without complex code
        - powered by GraphQL
        - offline and real-time capabilities
        - visual data modeling with Amplify Studio
- Amplify Studio
    - visually build a full-stack app, both UI and a backend
- Amplify CLI
    - configure an Amplify backend with a guided CLI workflow
- Amplify Libraries
    - connect app to existing AWS services
- Amplify Hosting
    - host secure, reliable fast web apps or websites via the AWS Content Delivery Network
    - build and host modern web apps
    - CICD
    - PR reviews
    - custom domains
    - monitoring
    - redirect and custom headers
    - password protection