# Open CDK Guide

# Table of Contents

<!-- toc -->

- [Purpose](#purpose)
  * [Why This Guide?](#why-this-guide)
  * [Contributions](#contributions)
  * [Credits](#credits)
  * [Legend](#legend)
- [Why CDK?](#why-cdk)
- [Tips and Best Practices](#tips-and-best-practices)
  * [Constructs](#constructs)
  * [Stacks](#stacks)
  * [Structure](#structure)
  * [Naming](#naming)
  * [Tagging](#tagging)
  * [Config](#config)
  * [Patterns](#patterns)
  * [Tools and Libraries](#tools-and-libraries)
  * [Deployments](#deployments)
  * [Limitations](#limitations)
  * [Get Help](#get-help)
  * [Further Reading](#further-reading)
- [Legal](#legal)
  * [Disclaimer](#disclaimer)
  * [License](#license)

<!-- tocstop -->

---
# Purpose

## Why This Guide?

The [AWS CloudDevelopment Kit](https://github.com/aws/aws-cdk) (CDK) is a framework built on top of CloudFormation that makes it delightful for users to manage AWS [Infrastructure as Code](https://en.wikipedia.org/wiki/Infrastructure_as_code) (IaC). When everything is going right, the CDK will make you feel like a devops wizard. That being said, the cloud is complicated, CloudFormation coverage of AWS is incomplete, and the CDK itself (and IaC in general) is still a young framework with little in the way of established best practices.

This guide is an opinionated set of tips and best practices for working with the CDK. It is meant to be a living document, updated on an ongoing basis by the community as the CDK and practices around it mature.

This guide assumes that you are familiar with CDK concepts or have gone through the [getting started documentation](https://docs.aws.amazon.com/cdk/latest/guide/getting_started.html)

Before using the guide, please read the [license](#license) and [disclaimer](#disclaimer).

## Contributions

Trying to keep up with AWS is a [Sisyphean](https://en.wikipedia.org/wiki/Sisyphus) task and this guide is far from complete. If you have something to add, please submit a pull request and help define conventions of the CDK.

## Credits

This guide was started by [Kevin S Lin](https://kevinslin.com), an early adopter of the CDK. It is heavily inspired in format and layout from the [og-aws guide](https://github.com/open-guides/og-aws).

Below are a list of awesome folks who have helped out on the project:
- [James Baker](https://github.com/nzspambot)

## Legend
- 🔸 A gotcha, limitation, or quirk
- 🚧 Areas where correction or improvement are needed
- 🔦 Hard to find feature
- 🚪 Third party library or service solutions

---

# Why CDK?

Before diving into best practices, a question that naturally arises when considering a new IaC in 2019 is why. AWS already has CloudFormation and there are many third party solutions (eg. Terraform, Troposphere, etc). This section exists to highlight reasons for considering.

- true Infrastructure as **code**
    - no configs or config like languages, use actual code to create infrastructure
    - full power of programming language at your disposal including conditionals, loops, string interpolation, etc.
    - see [here](https://kevinslin.com/aws/cdk_all_the_things/#just-code) for some examples

- type checking for infrastructure code
    - aws service properties are mapped to explicit types and results in compile time error for wrong properties

- superior cloudformation tooling
    - cdk deploy combines `create-changeset`, `execute-changeset`, `delete-changeset`, and live tailing of `describe-stack-events`
    - cdk diff creates a diff that actually show line item IAM and AWS resource changes (see [here](https://kevinslin.com/aws/cdk_all_the_things/#updating-a-service) for example)

- CDK lets you do things that are impossible to do in CloudFormation without relying on multiple stacks/multiple deployments with flags/ hard coding resource names/creating your own custom resources
    - eg1. create an s3 bucket notification that triggers a lambda. see entire code below
        - for why this is hard using cloudformation, see this [post](https://read.acloud.guru/cloudformation-is-an-infrastructure-graph-management-service-and-needs-to-act-more-like-it-fa234e567c82)
        ```typescript
        const bucket = new Bucket(this, 'fooBucket');

        const fn = new lambda.Function(this, 'fooFunc', {
          runtime: lambda.Runtime.NODEJS_10_X,
          handler: 'lambda.handler',
          code: lambda.Code.asset('functions/fooFunc'),
          timeout: Duration.seconds(60)
        });

        bucket.addEventNotification(
          EventType.OBJECT_CREATED,
          new s3n.LambdaDestination(fn)
        );
        ```
    - eg2. create a new cert using CertificateManager and [automatically validate it with route53](https://docs.aws.amazon.com/cdk/api/latest/docs/aws-certificatemanager-readme.html#automatic-dns-validated-certificates-using-route53)

- for features that CDK doesn't support, CDK offers many [escape hatches](https://docs.aws.amazon.com/cdk/latest/guide/cfn_layer.html)

- even if you don't plan on using CDK, the library is a superb reference for cloudformation documentation
    - all generated constructs come with property descriptions and links back to the original cloudformation documentation
    - eg. [definition for s3 bucket](https://gist.github.com/kevinslin/aca4eb60f01735545ed30a4fe7668178)

---

# Tips and Best Practices

## Constructs
- CDK has three types of [constructs](https://docs.aws.amazon.com/cdk/latest/guide/constructs.html)
    - Low Level constructs which are automatically generated from and map 1:1 to all CloudFormation resources and properties - they have a `CfnXXX` prefix
    - High Level constructs that initialize CFN resources with sane defaults and provide a high level interface to said resource
    - CDK pattern constructs which stitch together multiple resources together
        - eg. [LambdaRestApi](https://docs.aws.amazon.com/cdk/api/latest/docs/@aws-cdk_aws-apigateway.LambdaRestApi.html) which creates an api gateway that's backed by Lambda

- high level constructs
    - 🔸 read the small print in high level constructs, sometimes the default deployment is overkill for small projects
        - eg. [default vpc](https://docs.aws.amazon.com/cdk/api/latest/docs/@aws-cdk_aws-ec2.Vpc.html) will create a NAT gateway in all AZs of the given region
        -  a NAT gateway is $0.045/h and an additional $0.045 per GB processed
        -  creating a default VPC in us-west-2 which has 4 AZs would result in a monthly charge of $0.045 * 24 * 30 * 4 = $129.60 from idle NAT gateways alone
    - 🔸some constructs take raw strings to initialize (eg. `ServicePrinciple`, `ManagedPolicy`)
        - these constructs are not checked and errors in the name will only show up in deploy time
        - 🚪[cdk-constants](https://github.com/kevinslin/cdk-constants) is a collection of constants to mitigate this exact issue
    - most constructs allow importing existing resources via a `{construct}.from[Name|Attr|Name|...}` pattern
        ```typescript
        let importedBucket = Bucket.fromBucketName(scope, "fooBucket")
        ```
        - 🔸 constructs imported in this manner are not managed by the CDK but their properties can be read and used by the CDK (eg. existing S3 bucket to be used to store codebuild artifacts)
        - 🔸constructs imported in this manner have the signature of `I{Construct}` (eg. an imported bucket has the signature of `IBucket`)
        - can be an issue if your functions expect concrete construct types as IBucket is not a valid Bucket
    - 🔦 cross service integrations are packaged together into separate packages
        - you will generally find integrations with a `-targets` or `-actions` suffix
        - see [aws-route53-targets](https://docs.aws.amazon.com/cdk/api/latest/docs/aws-route53-targets.html), [aws-codepipeline-actions](https://docs.aws.amazon.com/cdk/api/latest/docs/aws-codepipeline-actions.html)
    - even though the CDK is now generally available, many of the constructs in the library are not. check the module documentation for status of specific modules (eg. [DocumentDB is still experimental]( https://docs.aws.amazon.com/cdk/api/latest/docs/aws-docdb-readme.html ))

- low level constructs
    - working with CFN resources is just like working with cloudformation, except instead of json/yaml, you get to write type checked code
    - use [cloudformation reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-reference.html) to find property values
    - querying cloudtrail logs is great to find additional IAM permissions necessary to launch a service
    - 🚪[former2](https://former2.com/) is a tool created by the brilliant [Ian Mckay](https://github.com/iann0036) that can generate CDK/cloudformation/terraform from your existing infrastructure

## Stacks

- a [stack](https://docs.aws.amazon.com/cdk/latest/guide/stacks.html) is the basic unit of deployment in the CDK
- stacks are composable - one stack can consist of multiple stacks
- stacks can share values - one stack can consume resources created in another stack (behind the scenes, CDK generates cloudformation [exports](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-exports.html) and [Fn::ImportValue](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-importvalue.html) to pass around values)
- a useful mental model of thinking about stacks is to separate them as `outer` and `inner`
    - outer stacks describe the stack purpose (eg. service stack)
    - inner stack are a component that makes up the outer stack (eg. monitoring)
- common outer stacks
    - foundation: vpc, security groups, etc
    - tools: ci/cd, automation, opsworks, etc
    - service|app: your user facing service or app
    - business|analytics: aws budgets, quicksignt, etc
    - security: config rules, acm certs, etc
    - ml|data pipeline: sagemaker, athena, etc
- common inner stacks
    - for applications
        - frontend
        - backend
        - database
    - for services
        - control plane
        - data plane
    - for any all stacks
        - logs and monitoring
        - automation

## Structure
- split the `bin` directory so you have one command per app
- split the `lib` by application
- create a `functions` directory if you plan on importing [lambda functions](https://docs.aws.amazon.com/cdk/api/latest/docs/aws-lambda-readme.html) via code assets
```
- bin/
    - create-serviceA.ts
    - create-serviceB.ts
- lib
    - serviceA/
    - serviceB/
    - common/
- functions/
    - func1/
    - func2
```

## Naming
- have a consistent naming scheme for stack ids
    -  use a fully qualified naming scheme
        - eg. {org}-{app}-{stage}
        - this value will be prepended to all constructs created inside the stack
```typescript
const app = new cdk.App();
const stage = app.node.tryGetContext('stage')
const infra = new InfraStack(app, `fooOrg-fooApp-prod`, {...})
// a bucket defined inside InfraStack (eg. new Bucket(this, userImages))
// will have the following name -> fooOrg-fooApp-dev-userImages87437600-kke5zcq506k7
```
- use simple ids for constructs
    -  stick to camelCase with no spaces, punctuation or underscores
        - every AWS service has different naming restrictions so its best to use a compatible default naming scheme
- resist temptation to name resources directly
    - cdk/cloudformation will generate a unique name based on the construct `id`
    - explicitly defining a name will cause operations that require `Replacement` to fail
        - eg. changing primary key of dynamo table requires a replacement. if you explicitly defined a name, you'll get an error like the following `CloudFormation cannot update a stack when a custom-named resource requires replacing. Rename testTable and update the stack again.`
- if you do need to give resources a human readable name, use tags instead
    - many aws services will use the value of the `Name` tag as the resource name

## Tagging
- tag your stacks - CDK makes this [really easy](https://docs.aws.amazon.com/cdk/latest/guide/tagging.html) and tags are applied recursively throughout the stack
- use constants to ensure consistent tagging
```typescript
import {TAGS} from './tags'
const infra = new InfraStack(app, "fooOrg-fooApp-dev", {...})
let {KEYS: K, VALUES: V} = TAGS
Tag.add(infra, K.APP, V.APP.FOO_APP)
Tag.add(infra, K.STAGE, V.STAGE.DEV)
```
- 🔸tag your low level constructs - stack tags don't apply to low level constructs so you'll have to manually tag them
- 🔸not all resources can be tagged because it is not supported in cloudformation - you'll need to use the `awscli` or create a custom resource
    - eg. [LogGroup](https://github.com/aws-cloudformation/aws-cloudformation-coverage-roadmap/issues/77) can't be tagged via cloudformation

## Config
- use cdk.json to store config values
    - for secrets, store the SSM|secretsmanager identifier
        - 🔸to create SSM|secretmanager values, you will still need to bootstrap by using the cli
-  scope config values by stage
    ```yaml
    "config": {
        {
            "dev": {
                "account": 1234567,
                "region": "us-west-2",
                # ...
            },
            "prod": {
                # ...
            }
        }
    }
    ```
    - use a helper function to retrieve values
    ```typescript
    function getEnv(stage, key) {
        scope.node.tryGetContext(stage)[key]
    }
    ```
- avoid use CloudFormation [parameters](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html)
    - parameters are resolved during deployed time which prevents the CDK from validating values
    - use context variables instead since the values can be validated before deployment

 - 🔸creating the same stack with different context parameters will result in different cloudformation template
    - can be confusing to those coming from CloudFormation parameters which will update the existing stack instead of creating a new one

## Patterns

- 🚧 creating default constructs
    - as a company, you might want to enforce certain defaults and policies when creating constructs
    - consider using  composition, inheritance and factories. see discussion [here](https://github.com/aws/aws-cdk/issues/3235)
    - factories example: creates s3 bucket with overridable defaults
```typescript
export function createBucket({scope, id, bucketProps = {}, accessLogBucket}: {
  scope: Construct|App,
  id: string,
  bucketProps?: BucketProps
  accessLogBucket?: IBucket
}) {
  bucketProps = _.defaults({}, bucketProps, {
    encryption: BucketEncryption.KMS_MANAGED,
    blockPublicAccess: BlockPublicAccess.BLOCK_ALL
  })
  const bucket = new Bucket(scope, id, bucketProps)

  if (!_.isUndefined(accessLogBucket)) {
    let logFilePrefix = `${id}/`
    addS3AccessLogs({srcBucket: bucket, destBucket: accessLogBucket, logFilePrefix})
  }
  return bucket
}
```

## Tools and Libraries

Collection of tools and libraries that make it easier to work with CDK

- constructs and construct libraries
    - [aws-cdk-examples](https://github.com/aws-samples/aws-cdk-examples): official AWS repository for example CDK constructs
    - [cdk-constants](https://github.com/kevinslin/cdk-constants): collection of useful constants for the cdk
    - [punchcard](https://github.com/sam-goodwin/punchcard): combine infra and runtime code

- tools
    - [aws-vault](https://github.com/99designs/aws-vault): A vault for securely storing and accessing AWS credentials in development environments. Supports switching between profiles, MFA access, assuming roles and more
    - [former2](https://former2.com/): tool created by the brilliant [Ian Mckay](https://github.com/iann0036) that can generate CDK/cloudformation/terraform from your existing infrastructure
    - [AWSConsoleRecorder](https://github.com/iann0036/AWSConsoleRecorder): tool created by the brilliant [Ian Mckay](https://github.com/iann0036) that can generate CDK/cloudformation/terraform/cli/boto3 (and more) from actions performed on the console.

## Deployments
- when checking in CR for CDK code, include output of `cdk diff` in the code review
    - 🔸 CDK diff will exit with error code of 1 if diff is detected. this is [expected behavior](https://github.com/aws/aws-cdk/issues/1440)
- its a good idea to version control the CloudFormation template in addition to the cdk code
    - can compare against past templates to get diff since a particular commit
    - easy to grep cloudformation to preview current state of infra
- 🚧 deploy CDK changes as part of CI/CD pipeline (same benefits as putting code in a pipeline - eg. reduce manual actions, better monitoring and better IAM isolation)

## Limitations
- aws cdk doesn't [support everything](https://github.com/aws/aws-cdk/issues/1656) that `awscli` does specifically
    - issue: CDK CLI will not read your region from your `default` profile
        - workaround: set region in `cdk.json`
    - issue: CDK CLI does not support MFA authentication
        - workaround: 🚪 use [aws-vault](https://github.com/99designs/aws-vault) to generate session tokens
    - issue: CDK CLI does not support `credential_source`
        - workaround: use `source_profile`
    - issue: Cannot have a profile named "default" in the config file
        - workaround: do not have `[default]` or `[profile default]` in config
- cdk diff won't show detailed construct properties when a construct is first created
    - use `cdk synthesize` to verify template properties before doing deployments that involve creating new constructs
- as far as features, CDK is built on top of cloudformation which means anything that cloudformation doesn't support is also not supported with the CDK
- high level constructs don't always support all the options of the cloudformation resource they are modifying
    - in this case, either add a property override or use the low level construct directly
    - don't forget to also submit an [issue](https://github.com/aws/aws-cdk/issues/new/choose) on github

## Get Help
- refer to the [api reference](https://docs.aws.amazon.com/cdk/api/latest/docs/aws-construct-library.html)
- figure out if the feature is a hole in cdk or one in cloudformation
    - [cloudformation open road map issues](https://github.com/aws-cloudformation/aws-cloudformation-coverage-roadmap/issues)
    - [cdk issues](https://github.com/aws/aws-cdk/issues)
- update to latest cdk - releases are frequent and they're constantly adding new things
- bring it up on the [cdk gitter](https://gitter.im/awslabs/aws-cdk) (this is actively monitored by CDK folks - I've had [issues](https://github.com/aws/aws-cdk/issues/3467) that I brought up be patched within a few hours)
- checkout the [cdk code](https://github.com/aws/aws-cdk) and see the tests
    - cdk has great test coverage and they have great examples here
    - `/packages/@aws-cdk/aws-codepipeline/test/*`
- check out the [CDK official examples](https://github.com/aws-samples/aws-cdk-examples)

## Further Reading
- [awesome-cdk](https://github.com/eladb/awesome-cdk): collection of CDK resources by one of the devs that created it
- [CDK All The Things](https://kevinslin.com/aws/cdk_all_the_things/): article covering general impressions of CDK and features

# Legal

## Disclaimer
The authors and contributors to this content cannot guarantee the validity of the information found here. Please make sure that you understand that the information provided here is being provided freely, and that no kind of agreement or contract is created between you and any persons associated with this content or project. The authors and contributors do not assume and hereby disclaim any liability to any party for any loss, damage, or disruption caused by errors or omissions in the information contained in, associated with, or linked from this content, whether such errors or omissions result from negligence, accident, or any other cause.

## License

[![Creative Commons License](https://i.creativecommons.org/l/by-sa/4.0/88x31.png)](http://creativecommons.org/licenses/by-sa/4.0/)

This work is licensed under a [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/).
