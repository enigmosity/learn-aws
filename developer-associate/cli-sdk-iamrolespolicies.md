# IAM Roles and Policies

- Attached policies that can be managed, or you can create your own.
- When go to EC2 role, can add inline policies to a role. Can only be used by one role. Not recommended since they don't scale well and are hard to manage
- Policy summary is an easy table to read. json is just json.
- Resources are what you're allowed to access. Can add specific ARNs or use '*'
- Can use AWS policy generator or generator in the IAM management console.

## Policy Simulator (testing a policy)

- tool online to test policies,  google it and hit the link.
- select the role you want to test, what action you want to take, and pick the service it's for, then hit the simulation button and it'll test it.

## CLI dry runs (testing a policy)

The **--dry-run** option allows simulation of API calls without actually firing up any resources. just checks that you can do it, fails if you don't have the required permissions

- specify --dry-run first, it'll save you from shenanigans. 
- give roles minimum permissions required to run.
- if you do have access, you'll get a 'DryRunOperation' error

## CLI STS Decode Errors

When you run API calls and they fail, you can get a long error message. 
- The error message can be decoded using the STS command line: **sts decode-authorization-message**
- this allows you to decode additional information about the authorization status of a request from an encoded message returned in response to an AWS request
- dump in the entire decoded method
- need to be authorized with the right permissions to run the command
- decoding is a write action

## EC2 Instance Metadata

- allows AWS EC2 instances to 'learn about themselves' *without using an IAM Role*
- URL is `http://169.254.169.254/latest/meta-data`
    - only accessibly from the EC2 instance itself
    - the same on every EC2 instance
- can retrieve IAM Role name from metadata, CANNOT retrieve the IAM policy

responses ending with '/' means there is more within that object that you can access. request endpoint must also end with '/'

If you query the security credentials and the roles in there, you can see the credentials that are given to your EC2. EC2 is given temporary credentials that will expire by the role, and is given new credentials as required.

## CLI Profiles

How do you manage having multiple AWS accounts? *Profiles*.
- define a name and configure it
- `aws configure --profile <name>` allows you to create a new account with different credentials that you can use
- to execute actions against profiles that aren't the default account, use `--profile <account name>` as an option on the command you're using

## MFA with CLI

- using MFA with CLI, you must create a temporary session
- must run *STS GetSessionToken* API call 

`aws sts get-session-token --serial-number <arn-of-mfa-device> --token-code <code-from-token> --duration-seconds 3600`

suggested way to use the credentials:
- create a profile for the MFA
- edit the configured file for the profile and paste in the credentials you've temporarily been assigned
- this would save you from typing everything individually

# AWS SDK

Want to perform actions directly on AWS from code? Use an SDK! 
- technically the CLI uses the python SDK
- exam expects you to know when you should use an SDK
- if you don't specify or configure a default region, `us-east-1` will be chosen by default

## AWS Limits / Quotas

API rate limits:
- how many times you can call an AWS API in a row
- for *intermittent errors*: implement an exponential backoff
- for *consistent errors*: request an API throttling limit increase

Service Quotas / Limits
- how many resources you can run of something
- request a service limit increase by opening a ticket
- request a service quota increase by using the Service Quotas API


### Exponential backoff

- For a throttlingException intermittently
- retry mechanism already included in AWS SDL API calls
- must implement yourself if using the AWS API as is or in specific cases
    - only implement retries if 5XX server errors and throttling
    - do not implement for 4XX errors

*Exponential backoff is common in exam questions*

## CLI Credentials Provider Chain

- cli will looks for credentials in order of:
1. command line options: --region, --output, and --profile
2. environment variables: AWS_ACCESS_KEY_ID, AWS_SECRET_KEY, and AWS_SESSION_TOKEN
3. CLI credentials file: aws configure
4. CLI configuration file: aws configure
5. Container credentials: for ECS tasks
6. Instance profile credentials: for EC2 instance profiles

*Best Practices*
- never store credentials in code
- if using AWS, use IAM roles
- outside of AWS, use environment variables/named profiles

Exam scenario:
    Service is assigned assigned specific permissions object that should limit it's access, but it doesn't. Why? *Credentials chain is still giving priority to something else in the chain*.

## Signature v4 Signing

Call AWS HTTP API, sign the request so that AWS can identify you with AWS creds
- some requests to S3 don't need to be signed
- if you use the SDK or CLI, HTTP requests are signed for you
- sign an AWS HTTP request using Signature v4 (SigV4)

Examples: 
- sign using an HTTP header
- sign using a query string (pre-signed URL for example)