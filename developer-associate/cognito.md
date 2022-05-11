# Cognito

- give users (outside of AWS) an identity so that they can interact with our application
- Cognito User Pools
    - logging in functionality for app users
    - integrate with API gateway and ALB
- Cognito Identity Pools (federated identity)
    - provide AWS credentials to users so they can access AWS resources directly
    - integrate with Cognito User Pools as an identity provider
- Cognito Sync
    - synchronise data from device to Cognito
- Cognito vs IAM: for "hundreds of users", "mobile users", or "authenticate with SAML", think Cognito. IAM is within the AWS environment

## Cognito User Pools (CUP)

- *create a serverless database of users for web and mobile applications*
- simple login: username or email/password combination
- password reset
- email and phone number verification
- MFA
- *federated identities*: users from Facebook, Google, SAML, etc.
- feature: block users if their credentials are compromised elsewhere
- *login sends back a JSON Web Token (JWT)*
- uses a databases of users
- integrates natively with API Gateway and Application Load Balancers

### Hands On
- can either use a username or email or phone number
- signup information required can be specified
- can set password requirements
- can set if user can create account themselves
- can set MFA requirements
- can set password recovery methods
- attribute verification
- email sending settings
- there are a lot of options
- app client allows login to user pool
    - *callback URL*: where user gets sent on sucessful login
    - *hosted UI*: domain that allows users to login using a Cognito user interface
    - can customise logo and css of the UI
- triggers are events that occur when a user does something with Cognito - can trigger Lambda Functions on these events

### Lambda Triggers

- CUP can invoke a lambda function synchronously on triggers

### Hosted Authentication UI
- has a hosted authentication UI that can add to app to handle sign-up and sign in workflows
- using hosted UI, a foundation for integration with social logins, OIDC or SAML
- can customise with a *custom logo and custom CSS*

## Cognito Identity Pools (Federated Identities)

- get identities for users outside of AWS so they have temporary AWS credentials
- identity pool can include
    - public providers (Amazon, Facebook, Google, Apple)
    - users in Amazon Cognito User Pool
    - openID connect providers and SAML identity providers
    - Cognito Identity Pools allow for *unauthenticated (guest) access*
- users can then access AWS services directly or through API Gateway
    - the IAM policies applied to the credentials are defined in Cognito
    - can be customised based on the user_id for fine grained control

### IAM Roles
- default IAM roles for authenticated and guest users
- define rules to choose the role for each user based on the user's Id
- can partition users' access using *policy variables*
- IAM credentials are obtained by Cognito Identity Pools through STS
- roles must have a 'trust' policy of Cognito Identity Pools

### hands on
- need to edit authenticated and unauthenticated roles to determine what people will have access to with the temporary credentials that they have

**user pools = manager user/password**
**identity pools = access aws services**

## Cognito Sync
- deprecated - use AWS AppSync now
- store preferences, configuration, state of application
- cross device synchronisation
- offline capability (synchronisation when back online)
- store data in datasets (up to 1MB), up to 20 datasets to synchronise
- *push sync*: *silently* notify across all devices when identity data changes
- *Cognito stream*: stream data from Cognito into Kinesis
- *Cognito events*: execute Lambda functions in response to events