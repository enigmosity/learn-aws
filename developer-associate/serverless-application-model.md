# Serverless Application Model (SAM)

## Overview

- framework for developing and deploying serverless applications
- all configuration is YAML code
- generate complex CloudFormation from a simple SAM YAML file
- supports anything from CloudFormation: outputs, mappings, parameters, resources
- only two commands are required to deploy to AWS
- SAM can use CodeDeploy to deploy Lambda functions
- SAM can help you to run Lambda, API Gateway, and DynamoDB locally

## Recipe
- *transform header* indicates it's a SAM template
    - `Transform: 'AWS::Serverless-2016-10-31'`
- write code
    - `AWS::Serverless::Function`
    - `AWS::Serverless::Api`
    - `AWS::Serverless::SimpleTable`
- package & deploy
    - `aws cloudformation package` / `sam package`
    - `aws cloudformation deploy` / `sam deploy`

## Deployment
- build the application locally
    - `sam build`
    - transforms the SAM YAML into a CloudFormation template
- package the application
    - `sam package` / `aws cloudformation package`
    - zip and upload to an S3 bucket
- deploy the application
    - `sam deploy`/ `aws cloudformation deploy`
    - creates or executes the changeset

## SAM CLI
- locally build, test and debug serverless applications defined using AWS SAM templates
- provides a Lambda-like execution environment locally
- SAM CLI+AWS toolkits => step-through and debug your code
- supported IDEs: Cloud9, VSCode, JetBrains, Pycharm, IntelliJ, etc.
- AWS Toolkits: IDE plugins which allow you to build, test, debug, deploy and invoke Lambda Functions built using AWS SAM

## Creating First SAM project

- `sam init` generates a SAM project for you, and includes most of the files you need
- *template.yaml* is the main SAM file, it describes how your function configuration
- the transform is indicative of using SAM template
- *properties* talk about what your resource has
- *codeuri* indicates where the code is locally
- *commands.sh* contains the commands required for building and deploying the project

## Deploying SAM project
- need an S3 bucket to exist
- call CLI commands with the parameters that it requires to build and deploy successfully
- deploying creates the changeset, and when ready deploys it
- may need a capability flag so that the required resources can be created by CloudFormation using the capability flag

## SAM API Gateway
- modify template.yaml to add API gateway
- only adding an event into the Lambda configuration in the template file. SAM figures out that API gateway is required

## SAM DynamoDB
- need to add the table as a resource in the template.yaml
- need to have the Lambda code in the application code file access the table
- configuration of the database is done in the template file (primary keys, provisioning of reads/writes etc.)
- need policies on the Lambda in template.yml so that it can access the DynamoDB database

## SAM CloudFormation Designer and Application Repository
- the CloudFormation created by SAM is very complex and large
- serverless application repository contains templates made by others that you can use 

## SAM Policy Templates
- list of templates to apply permissions to your Lambda functions
- important examples
    - *S3ReadPolicy*: gives read only permissions to objects in S3
    - *SQSPollerPolicy*: allows polling of an SQS queue
    - *DynamoDBCrudPolicy*: CRUD = create, read, update, delete
- don't need to create an IAM role, these policies become attached to the Lambda Function

## SAM with CodeDeploy
- SAM framework natively uses CodeDeploy to update Lambda Functions
- traffic shifting feature
- pre and post traffic hook features to validate deployment (before and after traffic shift)
- easy and automated rollback using CloudWatch Alarms

- in the template.yaml file, you can define how you want codedeploy to deploy a resource (canary etc.)
- may request that you confirm the changesets that you want to create and deploy


## SAM Exam Summary
- built on CloudFormation
- requires the *Transform* and *Resources* sections
- commands to know
    - *sam build*: fetch dependencies and create local deployment artifacts
    - *sam package*: package and upload to S3, generates a CloudFormation template
    - *sam deploy*: deploy to CloudFormation
- SAM Policy templates for easy IAM policy definition
- SAM is integrated with CodeDeploy to deploy to Lambda Functions

*The above summary is important for the Developer Associate exam.*

## Serverless Application Repository (SAR)
- managed repository for serverless applications
- *the applications are packaged using SAM*
- build and publish applications that can be re-used by organisations 
    - can share publically
    - can share with specific AWS accounts
- prevents duplicate work, go straight to deploying
- application settings and behaviour can be customised using *environment variables*
- public is applications are from AWS verified authors
- use the SAM publish command to publish versions to SAR