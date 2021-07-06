---
rfc pr: [#335](https://github.com/aws/aws-cdk-rfcs/pull/335)
tracking issue: https://github.com/aws/aws-cdk-rfcs/issues/1
---

# CDK Accelerate

Shorten the development iteration speed -- the "edit-compile-test" loop -- for
CDK applications.

## Working Backwards

### CHANGELOG

- feat(cli): implement `cdk deploy --accelerated` and
  `cdk deploy --accelerated --watch`

### Help

```
cdk deploy --help

    --accelerated[=only|auto|no]   Perform an accelerated deployment of the stack.
                                (`--accelerated=only` will fail if there are
                                non-accelerated updates)
    -w --watch   Watch for file changes and deploy any updates. Implies
                 --accelerated=auto if --accelerated is not specified.

Examples:
    cdk deploy -w
    cdk deploy --accelerated
    cdk deploy --accelerated=only --watch
```

### README

The `cdk deploy --accelerated` ("CDK accelerate") command accelerates the
edit-compile-test loop for your CDK application during development by inspecting
the assets and stack resources in your application, identifying those that can
be updated in-place without a full CloudFormation stack update, and doing so
using AWS service APIs directly. It can either do this as an interactive command
or by running continuously and monitoring the input files.

For supported construct types (see list, below) the update command will identify
assets or other stack resources that have changed since the last time the CDK
application was deployed, and update them in-place according to their types.
Assets will be uploaded to their storage medium. The CDK will then use AWS SDK
commands to directly modify the stack resources in-place to synchronize them
with your local code. The accelerate command can act on one or more stacks in
the application.

#### Usage

To run a one-shot update, you can invoke the `cdk deploy --accelerated` command
on some or all of your stacks:

```
$ cdk deploy --accelerated ApplicationStack
Checking stack ApplicationStack for possible accelerated update
 * LambdaFunction[AppFunction]: File asset changed
ApplicationStack can be updated
1/1 stacks can be updated:
Updating LambdaFunction in ApplicationStack
...
Update complete!
```

If the update is to an attribute of the stack that cannot be updated, the
command will offer to perform a full deployment:

```
$ cdk deploy --accelerated ApplicationStack
Checking stack ApplicationStack for possible accelerated update
 * LambdaFunction[Appfunction]: Function data changed
ApplicationStack has changes that can not be rapidly updated.
Perform a full deployment of ApplicationStack? [Y/n]: Y
Deploying ApplicationStack
...
Done!
```

When multiple stacks are provided, update only occurs if all pending changes are
suitable for update. Otherwise, a full deployment is done:

```
$ cdk deploy --accelerated ApplicationStack DataStack
Checking stack ApplicationStack for possible accelerated update
 * LambdaFunction[Appfunction]: Function data changed
ApplicationStack has changes that can not be rapidly updated.
Checking stack DataStack for possible accelerated update
 * LambdaFunction[StreamFunction]: File asset changed
DataStack can be updated
1/2 stacks can be updated: Perform a full deployment? [Y/n]? Y
Deploying ApplicationStack
...
Deploying DataStack
...
Done!
```

In addition to running in a one shot mode, the `cdk deploy --accelerated`
command also has a `--watch` command line option that enable it to monitor the
assets on disk and perform an update when they change.

```
$ cdk deploy --accelerated --watch ApplicationStack DataStack
Checking stack ApplicationStack for possible accelerated update
 * LambdaFunction[Appfunction]: No Changes
ApplicationStack can *not* be updated
Checking stack DataStack for possible accelerated update
 * LambdaFunction[StreamFunction]: No Changes

Watching stack inputs for changes:
 * LambdaFunction[Appfunction]: <spinner>
 * LambdaFunction[StreamFunction]: <spinner>

```

The watcher can only watch for file changes. For Lambda code, that means that
directories will be watched for changes and zipped if they change. If the file
asset is a zip file, then the update will fire whenever that changes. The
existing docker asset builder will be used to watch for changes in local docker
images.

#### Resource Support

- AWS Lambda `Function`
  - file and directory assets
- AWS Fargate
  - image assets
- ECS
  - image assets

#### Future Support Plans

- StepFunctions
- API Gateway models

### PRESS RELEASE

DATE - AWS announces the `cdk deploy --accelerated` command for the AWS CDK
toolkit. The `accelerate` command allows CDK application developers to rapidly
update code within their CDK application when no other AWS resources are
changing, bypassing the more correct, but time consuming, stack update
procedure.

The CDK toolkit uses CloudFormation under the hood to manage changes to AWS
resources, which safely updates resources. This update process is slow, however,
and during development it performs safety checks that add overhead to developer
workflows. Those delays are interposed between every change a developer makes,
and over the course of a product may add hours to the total development time.

The `cdk deploy --accelerated` command provides a way to bypass those checks
when working in development, directly updating the code in Lambda functions and
ECS services without a full CDK deployment. When the `accelerate` command can
identify that all changes in the stack have shortcut support, it will directly
publish the assets to AWS and then modify the underlying CDK resources directly.

> Using CDK accelerate dramatically improves our development iteration speed for
> Lambda functions. It cuts down the change-deploy-test cycle to seconds, where
> it was minutes before with a full cdk deploy each time. Unlike some other
> solutions, it actually updates our function running on AWS, which means there
> are no problems with the Lambda behaving differently during testing, and then
> in production - Adam Ruka, developer

Any CDK users who are deploying code to Lambda functions or containers on ECS or
Fargate can take advantage of this tool today, by configuring their stacks for
interactive update, and then using `cdk deploy --accelerated` instead of
`cdk deploy`.

## FAQ

### What are we launching today?

The `cdk deploy --accelerated` and `cdk deploy --accelerated --watch` commands,
with support for rapid update of Lambda function code, images for ECS and
Fargate task definitions, and AWS StepFunction workflows.

### Why should I use this feature?

If you are developing a CDK application and want to publish your code changes
without waiting for CloudFormation to perform a full, safe stack update, you
should use this feature.

### Does it work when my Lambda is in Go or Java?

To an extent, yes, but there are caveats. In order to update the lambda code you
will need to run your build process to create the jar or executable that you are
uploading as lambda code.

### Does this feature work with `aws-lambda-go` for compiled Lambdas?

Yes, if your lambda is defined by an `aws-lambda-go`, `aws-lambda-python`, or
`aws-lambda-nodejs` construct, the accelerate and watch capabilities will work
within those source trees to automatically deploy changes to your code as you
make it.

### What about "other resource type"?

Based on customer response, we will consider expanding the set of AWS resources
and asset types implemented in the CDK directly. The plugin mechanism for
resource refreshing will allow other resource types to be supported by either
community or vendor addition.

## Internal FAQ

### Why are we doing this?

The overhead of developing a Lambda application using the CDK is significant,
since each code change currently requires a CloudFormation stack update to apply
new code, _or_ manually introducing drift to the application by inspecting the
stack and manipulating the resources directly.

The CDK accelerate tool will allow the CDK to handle the resource changes rather
than this manual process, introducing some implicit safety and reducing the
manual labor of the update.

### Why should we _not_ do this?

This solution has a risk of introducing a deployment tool that users might use
to shortcircuit the safe CloudFormation deployment process. If a user runs CDK
update on their production stack, it can perform updates without adequate stack
update safety checks. Releasing a public tool with instant update capability
into the CDK may not be the right way to make this functionality public. To
mitigate this, `accelerate` only runs on stacks that have explicitly opted in to
the capability.

### What changes are required to enable this change?

1. An implementation of the `cdk deploy --accelerated` command in the CDK
   toolkit that can examine the Application stacks for updates that can be
   applied and apply them.
2. The CDK toolkit must be able to query a CDK resource for the set of
   filesystem resources it must monitor for the `--watch` operation, and run a
   filesystem monitor for that.

### Is this a breaking change?

No, it is not.

### What are the drawbacks of this solution?

- Updating like this still entails a fair amount of preprocesing time, since for
  complex projects (Golang and Java lambdas, for example) there remains a
  compilation step.
- The update model requires the AWS account used to publish the stack to also
  have sufficient permissions to update the underlying resources, rather than
  simply requiring the account to have CloudFormation access.
- Runtimes that require compilation or assembly -- Java lambdas, docker images
  -- do not benefit from the `--watch` support as naturally, and require some
  manual steps.
- updating Lambda functions that are using aliases and provisioned concurrency
  can take several minutes to switch over between the two versions.

### What alternative solutions did you consider?

We considered SAM accelerate for a similar purpose, but SAM covers a relatively
small set of possible application structures in AWS, while CDK is intended to
address the whole set of them.

We also considered introspecting the CDK model ourselves, but concluded that
there was little value in reinventing the wheel, so to speak, when CDK already
had all of the information we'd need to deliver this.

For identifying stacks that are subject to accelerated deployment, we are
considering defining a full "Personal Development Stack" model, possibly based
off of information in the CDK `App` context.

### What is the high level implementation plan?

We will start from the prototype CDK update command that identifies the Lambda
resources and then publishes them using the CDK toolkit, and extend that to
implement support for ECR images associated with ECS and Fargate tasks, API
Gateway definitions, and Step Function workflows. Those will be implemented
directly in the toolkit code as part of the launch for the feature. The toolkit
implementation will be designed to conform to an interface that provides:

- Watchable filesystem resources
- A way of updating the watchable resource list
- A method for determining whether the construct can be updated in place
- A method for updating the construct in place

If changes to the CDK constructs are necessary to implement the accelerated
development process, we will make those changes as well. In the longer term we
must lay the groundwork for moving the logic defining the update process into
the Construct library, which implies a design for passing these values by way of
the Cloud Assembly.

We will implement a filesystem watcher for the CDK toolkit that works on one or
more directory trees, watching for changes. It will base its watch list on the
set of files indicated by the toolkit, and update them when those responses
change.

Additionally, a `--watch` flag and a file watcher will be added to support
monitoring the inputs of stack resources for changes.

### Are there any open issues that need to be addressed later?

- This RFC can be extended to add support for further pluggable asset and update
  targets. The accelerate capabilities are attached to the CDK constructs, not
  the CDK CLI toolkit, so any CDK construct that can perform accelerated
  deployment can implement that capability in whatever manner is appropriate.
- This RFC will be enhanced significantly when the CDK asset model is enriched
  to support asset construction directly.
- Direct support for monitoring container repositories for changes (possibly via
  polling) instead of only supporting local rebuild.

# How do we keep production from being affected by CDK accelerate?

CDK accelerate uses the AWS IAM access controls already in place in your account
to maintain the safety of your production deployments. The interface to CDK
accelerate requires the developer to specify the stacks that they will be
deploying explicitly, so by default it is affecting only the developer's stacks,
and when a production stack is defined it is up to the AWS account administrator
to ensure that the interactive developer's roles do not have modification access
to the accelerated resources.