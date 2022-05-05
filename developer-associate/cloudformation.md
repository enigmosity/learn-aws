# CloudFormation

*Infrastructure as Code*
- manual work is tough to reproduce
- create it as code to be able to easily reproduce

- declarative way to outline AWS infrastructure for any resources
- CloudFormation creates resources for you, in the *right order*, with the *specified configuration*

Benefits:
- no resources manually created, excellent for control
- version controlled
- changes to infrastructure can be reviewed as code
- cost:
    - resources within a stack are tagged with an identifier to easily see cost of the stack
    - estimate the costs of resources using CloudFormation template
    - savings strategy: in Dev, automation of deletion of templates outside of business hours
- productivity:
    - destroy and re-create infrastructure in the cloud on the fly
    - automated generation of a Diagram for your templates
    - declarative programming (no need to figure out ordering and orchestration)
- separation of concerns: create many stacks for many apps and layers
    - VPC stacks
    - Network stacks
    - app stacks
- don't re-invent the wheel
    - leverage existing templates on the web
    - leverage documentation

## How CloudFormation works
- templates have to be uploaded in S3 then referenced in CloudFormation
- update the template, you cannot edit previous templates. Have to re-upload a new version of the template to AWS
- stacks are identified by a name
- deleting a stack deletes every single artifact that was created by CloudFormation

### Deploying templates
- manual
    - editing templates in the CloudFormation designer
    - use the console to input parameters, etc.
- automated
    - edit templates in a yaml file
    - use the AWS CLI to deploy templates
    - recommended to fully automate the flow

### Building Blocks - Template Components
- *Resources*: AWS resources declared in the template (mandatory)
    - resources are declared and can reference each other
    - aws handles creation, updates and deletes of resources
    - resource type identifiers are of the form **AWS::aws-product-name::data-type-name**
    - resources must have a type and properties
    - *update requires* explains what is required to update a parameter on a resource
    - cannot dynamically create resources in CloudFormation, there is no code generation, it must be declared
    - a few niche AWS services are not yet supported, but this can be worked around with Lambda custom resources 
- *Parameters*: dynamic inputs for template
    - for reuse of templates and if inputs cannot be determined ahead of time
    - powerful, controlled and ca nprevent errors in templates thanks to types
    - use when answer to `is this resource configuration likely to change in the future?` is *yes*
    - use when values are really user specific
    - therefore *will not have to re-upload template to change content*
    - Parameter settings
        - types: string, number, comma-delimited list, List<type>, AWS Parameter (to help catch invalid values - match against existing values in the AWS account)
        - description
        - constraints
        - ConstraintDescription (string)
        - Min/MaxLength
        - Defaults
        - AllowedValues (array)
        - AllowedPattern (regexp)
        - NoEcho (bool)
    - use Ref function to use parameters (Fn::Ref, shorthand in YAML: !Ref), function can also reference other elements within the template
    - Psuedo Parameters, can be used at any time, enabled by default
- *Mappings*: static variables for your template
    - fixed variables within CloudFormation template
    - handy to differentiate between different environments, regions, ami types, etc.
    - all values are hardcoded in template
    - ideal when you know in advance all values that can be taken and that they can be deduced. 
    - allow safer control over the template
    - access through *Fn::FindInMap* to return a named value from a specific key
        - YAML shortand: *!FindInMap [ MapName, TopLevelKey, SecondLevelKey ]* 
        - *It is important that you know this syntax for the AWS Certified Developer Associate exam*
- *Outputs*: references to what has been created
    - section declares *optional* output values that we can import into other stacks (if you export them first)
    - can view outputs in console or CLI
    - best way to perform collaboration cross stack, as subject matter experts can handle their own part of the stack
    - can't delete a CloudFormation Stack if its outputs are being referenced by another CloudFormation Stack
    - requires the export block and what it will be called when exported
    - exported output names must be unique within a region
    - Cross Stack Reference
        - create a second template that leverages the output variable
        - use *Fn::ImportValue* (YAML: *!ImportValue*)
        - can't delete underlying stack until all references are deleted too
    - *Outputs make for a very popular Certified Developer Associate exam question*
- *Conditionals*: list of conditions to perform resource creation
    - used to control the creation of resources or outputs based on a condition
    - can be whatever you want them to be, common examples:
        - environment
        - region
        - any parameter value
    - each condition can reference another condition, parameter value or mapping
    - uses a condition block
    - logical ID is for you to choose, how you name the condition
    - intrinsic function (logic) can be:
        - Fn::And
        - Fn::Equals
        - Fn::If
        - Fn::Not
        - Fn::Or
    - can be applied to resources/outputs/etc. included within a resource block
- *Metadata*

Template helpers:
- references
- functions

*Only goes deep enough to answer all questions in exam and so all preceeding knowledge is necessary. The exam doesn't require writing of CloudFormation, but does require reading of it.*

### createstack hands on

*Only creating new resources is within the exam.*
- CloudFormation information exists in the tagging of the instances created by CloudFormation

### update and delete stack hands on

- update stack by selecting stack, and selecting update, then replace the current template with the new one
- can view changes list to see what is about to happen
- events will tell you how things are progressing
- terminating one thing created by CloudFormation will leave the rest of the infrastructure running. if you delete the entire stack in CloudFormation, everything gets deleted.

## YAML crash course
- can use YAML and JSON for CloudFormation
- key-value pairs
- '-' means an array of information
- '|' means multi-line string
- can nest objects

## Must Know Instrinsic Functions

- Ref
    - leveraged to reference 
        - parameters: returns value of parameter
        - resources: returns physical ID of the underlying resource
    - YAML shorthand: `!Ref`
- Fn::GetAtt
    - attributes attached to any resources you create
    - to know resources attributes, check docs
    - YAML shorthand: `!GetAtt <resource name>.<attribute you want>`
    - *very popular Certified Developer Associate exam question*
- Fn::FindInMap
    - return an named value from a specific key
    - YAML shorthand: `!FindInMap [ MapName, TopLevelKey, SecondLevelKey ]`
- Fn::ImportValue
    - import values are exported from other templates
    - YAML shorthand: !ImportValue
- Fn::Join
    - join values with a delimiter
    - YAML shorthand: `!Join [ delimiter, [comma-delimited list of values]]`
    - !Join [":", [a, b, c]] = "a:b:c"
- Fn::Sub
    - used to substitute variables from a text
    - string must contain ${VariableName} and will substitute them
    - YAML shorthand: 
        `!Sub
        - String
        - {Var1Name: Var1Value, Var2Name: Var2Value }`
- Condition functions

## CloudFormation Rollbacks

- stack creation fails:
    - Default: everything rolls back (gets deleted). Can look at logs
    - option to disable rollback and troubleshoot what happened
- stack update fails
    - stack automatically rolls back to the previous known working state
    - ability to see in the log what happened and error messages

## ChangeSets

When you update a stack, you need to know what changes will happen before it happens for increased confidence. Changesets won't say if the update will be successful.

## Nested Stacks
- nested stacks are part of other stacks
- allow you to isolate repeated patterns/common components in separate stacks and call them from other stacks
- considered best practice
- to update a nested stack, update the parent (root) stack

*Cross vs Nested Stacks*
- cross
    - helpful when stacks have different lifecycles
    - use outputs, export and Fn::ImportValue
    - when you need to pass export values to many stacks
- nested
    - helpful when components must be reused
    - nested stack only important to the higher level stack (not shared)

## StackSets
- create, update or delete stacks across *multiple accounts and regions* with a single operation
- administrator account required to create StackSets
- trusted accounts to create, update, delete stack instances from StackSets
- when you update a stack set, *all* associated stack instances are updated thoughout all accounts and regions

## Cloudformation Drift
- allows creation of infrastructure
- doesn't protect against manual configuration change, called *Drift*
- drift can be used productively
- use drift to see if stack is doing what you expect it to and what may need to change based on what has been changed manually