# Identity and Access Management

IAM is a *Global Service* - it isn't constrained to a region.

The root account is created by default, it shouldn't be used except to set up users. **It should never be shared.**

*Users* are physical people within an organisation and can be grouped. ***Groups* only contain users**. Users can belong to multiple *groups*. *User Groups cannot belong to another User Group.*

## Permissions

Users or groups can be assigned policies to control permissions of users, often in JSON docs.

> *Apply the **least privilege principle**: don't give users more permissions than necessary.*

There are pre-existing policies that you can already add to groups or users, but custom is also an option.

*Tags* can be used to hold any kind of information that you want to apply to any user, group or service. This is not just limited to IAM.

## IAM Policies

**Policies:** *JSON documents that define a set of permissions for making requests to AWS services, and can be used by IAM users, User Groups and IAM Roles*

Group level policies apply to all users in groups. Some users have inline policies.

Policy structure:
- *version number*: policy language version
- *id*: optional policy identifier
- *statement*: one or many

Statement:
- *sid*: identifier for statement optional
- *effect*: allow or deny access
- *principal*: account/user/role policy apples to
- *action*: list of actions policy has effect on 
- *resource*: list of resources to which actions are applied to
- *condition*: optional for when the policy is in effect

Permissions from groups are inherited or attached from a group, attached directly means policy is tied directly to a user account.

You can use the visual editor to create your policies, AWS will then, rather than mucking around with writing the json yourself.

Inline Policies

TODO: fill in what inline policies are

## Password policy

Set password requirements:
- uppercase
- lowercase
- numbers
- non alpha numeric

Other password safety options:
- users can change own passwords
- must change after password after set time
- no password reuse

# MFA 
Protects access to your account, you especially want to protect root and IAM.

MFA ensures it's you by: a password you know + security device you own.

MFA options:
- virtual MDA device (eg. Google Authenticator)
- universal 2nd factor security key
- hardware keyfob MFA device
    - (and if in US, specific one for) AWS GovCloud  

## AWS Access Keys, CLI and SDK

AWS CLI
- lets you interact with aws through command line
- direct access to public commands
- allows you to script

SDK:
- embedded within application code

## AWS CloudShell

CloudShell is alternate to using a terminal on a local machine. Allows you to use the CLI through the browser. If you can't install aws cli on your machine this is ideal. Uses the permissions of the account that you're currently logged into. Any files changes are saved for future. Can also upload and download files for use with CloudShell.

Only allowed in some regions.

## IAM Roles

AWS services need permissions to run, same as users. Create an IAM role for each service use that you've got.
*Defined as: An IAM entity that defines a set of permissions for making requests to AWS services, used by an AWS service.*

## IAM Security tools

**Credentials report**: account level
- report that lists all accounts users and the status of their credentials

**Access Advisor**: user level
- shows the service permissions granted to a user and when those services were last accessed
- use this info to revise policies (least privilege)

## IAM Guidelines & Best Practices

- don't use the root account except for AWS account setup
- one physical user  = one AWS user
- assign users to groups and assign permissions to groups
- create a strong password policy
- use and enforce MFA
- create and use roles for giving permissions to AWS services
- use access keys for programmatic access
- audit account permissions with IAM credentials report
- **never shar IAM users or access keys**


# Summary:
- users: mapped to physical user, has password for console
- groups: only contains users
- policies: JSON doc that outlines permissions for users or groups
- roles: for EC2 instances or AWS services
- security: MFA + password policy
- access keys: access AWS programmatically
- audit: IAM credential reports & IAM access advisor

TODO: What is the AWS Shared Responsibility Model?
