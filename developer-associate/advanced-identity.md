# Advanced Identity

## STS - Security Token Service

- allows you to grant limited and temporary access to AWS resources (up to 1 hour)
- *AssumeRole*: Assume roles within your account or cross account EXAM
- *AssumeRoleWithSAML*: return credentials for users logged with SAML
- *AssumeRoleWithWebIdentity*
    - return creds for userse logged with an IdP
    - AWS recommends against using this and using *Cognito Identity Pools* instead
- *GetSessionToken*: for MFA, from a user or AWS account root user EXAM
- *GetFederationToken*: obtain temporary creds for a federated user
- *GetCallerIdentity*: return details about the IAM user or role used in the API call EXAM
- *DecodeAuthorizationMessage*: decode error message when an AWS API is denied EXAM

*STS AssumeRole is the most important call to know for the Developer Associate exam.*

### Using STS to assume a role
- define an IAM Role within account or cross-account
- define which principals can access IAM role
- use AWS STS to retrieve credentials and impersonate the IAM role you have access to (*AssumeRole API*)
- temporary credentials can be valid for between 15 minutes to 1 hour

### STS with MFA
- use GetSessionToken from STS
- appropriate IAM policy using IAM conditions
- *aws:MultiFactorAuthPresent:true*
- reminder, GetSessionToken returns:
    - Access ID
    - Secret Key
    - Session Token
    - Expiration date

*STS AssumeRole with MFA is important for the Developer Associate exam.*

## Advanced IAM

> Authorisation Model Evaluation of Policies, simplified
> 1. explicit DENY, end decision and DENY
> 2. if an ALLOW, end decisions with ALLOW
> 3. otherwise DENY
>
> If there's an explicit DENY and an ALLOW in the policy, the DENY will win

### IAM Policies and S3 Bucket Policies
- IAM policies are attached to users, roles, groups
- S3 buckets policies are attached to buckets
- when evaluating if an IAM prinicipal can perform an operation on a bucket, the **union** of its assigned IAM policies and S3 bucket policies will be evaluated

#### Dynamic Policies with IAM
- how do you assign each user a `/home/<user>` folder in an S3 bucket?
    1. create an IAM policy allowing `name` to have access to `/home/name`
        - one policy per user
        - doesn't scale
    2. create one dynamic policy with IAM
        - leverage the special policy variable `@{aws:username}`

#### Policy Types

Managed policy
- maintained by AWS
- good for power users and admins
- updated in case of new services/APIs

Customer Managed policy
- best practice, re-usable, can be applied to many principals
- version controlled + rollback, central change management

Inline Policy
- strict one-to-one relationship between policy and principal
- policy is deleted if you delete the IAM principal

## Granting a user permissions to pass a role to an AWS service

- to configure many AWS services, you must *pass* an IAM role to the service (happens only once during setup)
- service will later assume the role and perform actions
- *need IAM permission **iam:PassRole***
- often comes with `iam:GetRole` to view role being passed
- can a role be passed to any service?
    - *no. roles can only be passed to what their **trust policy** allows*
    - a *trust policy* for the role is what allows the service to assume the role

## Directory Services

### Microsoft Active Directory (AD)
- found on any Windows server with Active Directory domain services
- database of *objects*: user accounts, computers, printers, file shares, security groups
- centralised security management, create accounts, assign permissions
- objects are organised in *trees*
- a group of trees is a *forest*

### AWS Directory Services
- *AWS Managed Microsoft Active Directory*
    - create your own active directory in AWS, manage users locally, supports MFA
    - establish 'trust' connections with your on-premise active directory
- *Active Directory Connector*
    - directory gateway (proxy) to redirect to on-premise active directory, supports MFA
    - users are managed on the on-premise active directory
- *Simple Active Directory*
    - active directory-compatible managed directory on AWS
    - cannot be joined with on-premise active directory 

*In the Developer Associate exam, be prepared to answer a question about which type of active directory you would choose for a certain scenario and so you need to understand why.*