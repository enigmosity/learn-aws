# Identity and Access Management

IAM is a *Global Service* - it isn't constrained to a region.

The root account is created by default, it shouldn't be used except to set up users. **The root account should never be shared.**

*Users* are physical people within an organisation and can be grouped. ***Groups* only contain users**. Users can belong to multiple *groups*. *User Groups cannot belong to another User Group.*

## Permissions

Users or groups can be assigned policies to control permissions of users, typically using JSON docs.

> *Apply the **least privilege principle**: don't give users more permissions than necessary.*

*Tags* can be used to hold any kind of information that you want to apply to any user, group or service. This is not just limited to IAM.

## IAM Policies

**Policies:** *JSON documents that define a set of permissions for making requests to AWS services, and can be used by IAM users, User Groups and IAM Roles*

Group level policies apply to all users in groups. Some users have inline policies.

### AWS Managed Policies

Pre-existing policies controlled by AWS that you can add to users, groups or roles. You cannot change what the policy can do.

### Custom Managed Policies

Custom created roles that you need to manage. You determine the appropriate permissions and what they are applied to. Can be shared across identities.

### Inline Policies

An inline policy is a policy that is embedded in an IAM user/group/role. The policy is an inherent part of the identity. It cannot be shared across identities.

### Policy structure:
- *version number*: policy language version
- *id*: optional policy identifier
- *statement*: one or many statements

**Statement**:
- *sid*: identifier for statement optional
- *effect*: allow or deny access
- *principal*: the account/user/role policy apples to
- *action*: list of actions the policy has effect on 
- *resource*: list of resources which actions are applied to
- *condition*: optional to control when the policy is in effect

Permissions from groups are inherited or attached from a group, attached directly means policy is tied directly to a user account.

You can use the visual editor to create your policies, AWS will then translate this into JSON for you if you're not confident with the JSON configuration.

## Password policy

You can set password requirements:
- uppercase
- lowercase
- numbers
- non alpha numeric

Other password safety options are:
- users can change own passwords
- must change after password after set time
- no password reuse

# Multi-Factor Authentication (MFA) 
Protects access to your account, you especially want to protect root and IAM.

MFA ensures it's you with: *a password you know + security device you own*.

MFA options:
- virtual MDA device (eg. Google Authenticator)
- universal 2nd factor security key
- hardware keyfob MFA device
    - and if in US, there is a  specific device for AWS - AWS GovCloud  

## AWS Access Keys, CLI and SDK

AWS CLI
- lets you interact with aws through command line
- direct access to public commands
- allows you to script

SDK:
- embedded within application code

## AWS CloudShell

CloudShell is alternate to using a terminal on a local machine. It allows you to use the AWS CLI through the browser. If you can't install the AWS CLI on your machine this is ideal. It uses the permissions of the account that you're currently logged into. Any files changes are saved for future. Can also upload and download files for use with CloudShell.

Only allowed in some regions.

## IAM Roles

An IAM Role is *an IAM entity that defines a set of permissions for making requests to AWS services by an AWS service.*

AWS services need permissions to run, same as users. Best practice is to create an IAM role for each service use case.

## IAM Security tools

**Credentials report**: account level
- report that lists all account users and the status of their credentials

**Access Advisor**: user level
- shows service permissions granted to a user and when those services were last accessed
- use this information to revise policies (following the least privilege principle)

## IAM Guidelines & Best Practices

- don't use the root account except for AWS account setup
- one physical user = one AWS user
- assign permissions to groups and then control access by assigning users to the appropriate groups
- create a strong password policy
- use and enforce MFA
- create and use roles for giving permissions to AWS services
- use access keys for programmatic access
- audit account permissions with IAM credentials report
- **never share IAM users or access keys**


# Summary:
- users: mapped to physical user, has password for console
- groups: only contain users
- policies: JSON documents that outlines permissions for users and/or groups
- roles: for EC2 instances or AWS services to determine permissions to other AWS services
- security: MFA + password policy
- access keys: access AWS programmatically
- audit: IAM credential reports & IAM access advisor

# AWS Shared Responsibility Model

**Customer**:*responsibility for security 'in' the cloud*
**AWS**:*responsibility for security 'of' the cloud*

The shared responsibility model boils down simply to what AWS is responsible for versus what the user/developer/configuration owner of AWS services is responsible for. AWS is responsible for ensuring that the services that they offer are running and to miminise any bugs in their system for example. The responsibility of the user to configure the AWS service that they use to meet their requirements. For example, AWS is not responsible for unencrypted S3 Buckets if you have not opted in to configuration as part of the S3 configuration.
