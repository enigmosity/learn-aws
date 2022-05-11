# Encryption

## Encryption-in-flight (SSL)
- data encrypted before sending and decrypted after receiving
- SSL certifications help with encryption (HTTPS)
- encryption-in-flight ensures no man-in-the-middle attack can happen

## Server Side Encryption at Rest
- data encrypted after being received by the server
- data is decrypted before being sent
- stored in an encrypted form thanks to a key (usually a data key)
- encryption/decryption keys must be managed somewhere and the server must have access to it

## Client Side Encryption
- data encrypted by the client, never decrypted by the server
- data will be decrypted by a receiving client
- server should not be able to decrypt the data
- could leverage *Envelope Encryption*

## Key Management Service (KMS)
- encryption within AWS services, usually KMS
- easy to control access to data, AWS manages keys for us
- fully integrated with IAM for authorisation
- seamlessly integrated into: EBS, S3, Redshift, RDS, SSM
- can also use with CLI & SDK

### Customer Master Key (CMK) types
- *symmetric* (AES-256 keys)
    - first offering of KMS, single encryption key that is used to encrypt and decrypt
    - AWS services are integrated with KMS using symmetric CMKs
    - necessary for Envelope Encryption
    - never get access to key unencrypted (must call KMS API to use)
- *asymmetric* (RSA & ECC key pairs)
    - public (encrypt) and private key (decrypt) pair
    - used for encrypt/decrypt or sign/verify operations
    - public key is downloadable, but can't access private key unencrypted
    - use case: encryption outside of AWS by users who can't call the KMS API
- able to fully manage keys and policies
    - creation
    - rotation policies
    - disabling/enabling
- able to audit key usage (using CloudTrail)
- three types of CMK:
    1. AWS managed service default CMK: *free*
    2. user keys created in KMS: *$1/month*
    3. user keys imported (must be 256-bit symmetric key): *$1/month*
- + pay for API calls to KMS ($0.03/10000 calls)
- anytime you need to share sensitive information, use KMS
    - database passwords
    - credentials for external services
    - private key of SSL certificates
- value in KMS that the CMK used to encrypt data can never be retrieved by the user, and the CMK can be rotated for extra security
- *never store your secrets in plaintext, especially in code*
- encrypted secrets can be stored in the code/environment variables
- *KMS can only help in encrypting up to 4KB of data per call*
- if data > 4KB, use Envelope Encryption
- to give access to KMS to someone
    - make sure the key policy allows the user
    - make sure the IAM policy allows the API calls
- KMS keys are bound to a specific region

*For the Developer Associate exam, expect most questions to be about symmetric keys, but it's still important to understand assymetric keys.*

### KMS Key Policies
- control access keys similar to S3 bucket policies
- difference from bucket policies: cannot control access without them
- *Default KMS Key Policy*
    - created if you don't provide a specific KMS key policy
    - complete access to the key by the root user = entire AWS account
    - gives access to the IAM policies to the KMS key
- *Custom KMS Key Policy*
    - define users and roles that can access the KMS key
    - define who can administer the key
    - useful for cross-account access of your KMS key

### Copying Snapshots across regions
1. create a snapshot, encrypted with CMK
2. *attach a KMS key policy to authorise cross-account access*
3. share the encrypted snapshot
4. in target region, create a copy of the snapshot, encrypt it with a KMS key in your account
5. create a volume from the snapshot

### Hands On
- allowing root user to access key means any role can also access it
- default key policy allows root, allowing allowing everybody in
- 'KMS encrypt' does the encryption, requires region, the file you're encrypting and the output you expect

### Envelope Encryption
- KMS Encrypt API call has a limit of 4KB 
- if you want to encrypt >4KB, need to use envelope encryption
- API that will help is the *GenerateDataKey* API

*In theory, the above summary is all that you need to know for the Developer Associate exam about envelope encryption.*

#### GenerateDataKey API
- call with the SDK
- KMS will check IAM permissions
- send back plain text data key
- called envelope because there is a wrapper around the encrypted DEK and the encrypted file

#### Encryption SDK
- implemented envelope encryption
- exists as a CLI tool
- implementations for Java, Python, C, JavaScript
- Feature - Data Key Caching
    - re-use data keys instead of creating new ones for each encryption
    - helps with reducing the number of calls to KMS with a security tradeoff
    - use LocalCryptoMaterialsCache (max age, max bytes, max number of messages)

## API Summary for Exam
- *Encrypt*: encrypt up to 4KB of data through KMS
- *GenerateDataKey*: generates a unique symmetric data key (DEK)
    - returns a plaintext copy of the data key
    - and a copy that is encrypted under the CMK that you specify
- *GenerateDataKeyWithoutPlaintext*
    - generate a DEK to use at some point (not immediately)
    - DEK that is encrypted under the CMK that you specify (must use Decrypt later)
- *Decrypt*: decrypt up to 4KB of data (including Data Encryption Keys)
- *GenerateRandom*: returns a random byte string

## KMS Limits

Request Quotas
- when exceeding a request quota, you get a *ThrottlingException*
- to respond, use *exponential backoff*
- for cryptographioc operationss, they share a quota
- includes requests made by AWS on your behalf
- for GenerateDataKey, consider using DEK caching from the Encryption SDK
- *can request a Request Quotas increase through API or AWS support*

## S3 Security

4 methods of encrypting objects in S3
- *SSE-S3*: encrypts S3 objects using keys handled and managed by AWS
- *SSE-KMS*: leverage AWS KMS to manage encryption keys
- *SSE-C*: when you want to manage your own encryption keys
- Client Side Encryption

*For the Developer Associate exam, you need to know which S3 encryption type to use for different scenarios.*

### SSE-KMS
- encryption using keys handled and managed by KMS
- KMS advantages: user control + audit trail
- object is encrypted server side
- must set header: `"x-amz-server-side-encryption":"AWS:KMS"`
- leverages *GenerateDataKey* and *Decrypt* KMS API calls
- KMS API calls will show up in CloudTrail, helpful for logging
- to perform SSE-KMS:
    - KMS key policy that authorises the user/role
    - IAM policy that authorises access to KMS
    - otherwise will get an access denied error
- S3 calls to KMS for SSE-KMS count against KMS limits
    - if throttling, try exponential backoff
    - if throttling, can request an increase in KMS limits
    - throttling service is KMS, not S3

### S3 Bucket Policies
- to force SSL usage,create an S3 bucket policy with a DENY on the condition *AWS:SecureTransport=false*
    - Note: using an allow on *AWS:SecureTransport=true* would allow anonymous users to GetObject if using SSL
- force encryption of SSE-KMS
    1. deny incorrect encryption header: make sure it includes AWS:KMS (=SSE-KMS)
    2. deny no encryption header to ensure objects are not uploaded un-encrypted
        - could swap (2) for S3 default encryption of SSE-KMS

## S3 Bucket Key

- new setting to decrease
    - number of API calls made to KMS from S3 by 99%
    - cost of overall KMS encryption with S3 by 99%
- leverages data keys
    - S3 bucket key is generated
    - that key is used to encrypt KMS objects with new data keys
- will see less KMS CloudTrail events in CloudTrail

## SSM Parameter Store

- secure storage for configuration and secrets
- optional seamless encryption using KMS
- serverless, scalable, durable, easy SDK
- version tracking of configuration/secrets
- configuration management using path and IAM
- notifications with CloudWatch Events
- integration with CloudFormation
- suggested hierarchy
>    - /my-department/
>        - app/
>            - dev/
>                - db-url
>                - db-password
>            - prod/
>                - db-url
>                - db-password
>        - other-app/
>    - other-department
- teirs
    - standard - free
    - advanced - paid
- parameter policies (advanced)
    - allow to assign a TTL to a parameter to force updating or deleting sensitive data such as passwords
    - can assign multiple policies at a time

- SSM Parameters Store can be used to store secrets and has built-in version tracking capability. Each time you edit the value of a parameter, SSM Parameter Store creates a new version of the parameter and retains the previous versions. You can view the details, including the values, of all versions in a parameter's history.

### Hands On
- can request parameters by path and then see secrets that are stored in the hierarchy
- use wildcards to enforce which parameters can be accessed by the different roles and services

## Secrets Manager
- newer service meant for storing secrets
- capability to force *rotation of secrets* every X days
- automate generation of secrets on rotation (uses Lambda)
- integration with Amazon RDS (MySQL, ProstgreSQL, Aurora)
- secrets are encrypted using KMS
- mostly meant for RDS integration

### Hands On
- idea is that because you can enforce rotations and it's more secure
- managed by IAM
- key-value pairs
- can select your own encryption key

**Secrets manager**
- more expensive
- automatic rotation of secrets with AWS Lambda
- Lambda function is provided for RDS, Redshift, DocumentDB
- KMS encryption is mandatory
- can integrate with CloudFormation
**SSM Parameter Store**
- some cost
- simple API
- no secret rotation (can enable rotation using Lambda triggered by CloudWatch events)
- KMS encryption is optional
- can integrate with CloudFormation
- can pull a secrets manager secret using the SSM Parameter Store API

## CloudWatch Log Encryption

- can encrypt CloudWatch logs with KMS
- encryption is enabled at the log group level, by associating a CMK with a log group, either when you create the log group or after it exists
- cannot associate a CMK with a log group using the CloudWatch console
- must use the CloudWatch Logs API, cannot be done through console
- can integrate with CloudFormation
    - *associate-KMS-key*: if log group already exists
    - *create-log-group*: if log group doesn't exist yet

## CodeBuild Security

- to access resources in VPC, make sure you specify a VPC configuration for your CodeBuild
- secrets in CodeBuild:
    - **don't store them as plaintext in environment variables**
    - environment variables can reference parameter store parameters
    - environment variables can reference secrets manager secrets