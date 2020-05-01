---
layout: post
title: How to create a CodeCommit to Slack Bot
author: Wassim Kallel
date: '2020-05-01 22:00:00 +01'
tags:
    - serverless
    - aws
image: codecommit-slack.png
summary: How to create a CodeCommit to Slack Bot
thumbnail: codecommit-slack.png
comments: true
---

## Introduction

**1. What’s AWS CDK?** ​

AWS CDK is an Infrastructure as Code solution, similar to AWS CloudFormation and HashiCorp’s Terraform, Where you can have you can have your infrastructure defined as Code in a project which you can deploy easily, keep track of its changes (Assuming you use a Version Control system like Git for that) and acts as a documentation for your simple or complex solution design. ​

**2. How does it compare to the others?** ​
<div class="container table-responsive table-body">
<table class="table table-bordered">
    <tr>
        <th scope="col"></th>
        <th scope="col">Aws Cloudformation</th>
        <th scope="col">Aws CDK</th>
        <th scope="col">Terraform</th>
    </tr>
    <tr>
        <td>Execution</td>
        <td>Server side execution in aws cloudformation’s service</td>
        <td>Client side synthesis, Server side execution in CloudFormation’s service</td>
        <td>Client side execution</td>
    </tr>
    <tr>
        <td>Language</td>
        <td>YAML or JSON</td>
        <td>TypeScript, Python, Java or C#</td>
        <td>HashiCorp Language</td>
    </tr>
    <tr>
        <td>Supported providers</td>
        <td>AWS only</td>
        <td>AWS only</td>
        <td>Plenty of providers <a href="https://www.terraform.io/docs/providers/index.html">see here</a></td>
    </tr>
</table>
</div>
​

​ ​ **3. CDK project lifecycle** ​

When you create a project you define your infrastructure in one of the supported languages. Your CDK project gets parsed and a CloudFormation template is generated. The CloudFormation template gets executed in aws Cloudformation’s service and all infrastructure is set up for you.

## Let’s get our hands dirty

Here’s what we’re trying to create in this tutorial, we’re going to make any action in our CodeCommit Repository triggers a notification in Slack. ​ Under the hood a Cloudwatch Rule will listen to any actions happening in the repository and triggers a Lambda with the action’s details (commit message, pull request title, etc).

**1. Installation** ​

Assuming you have NPM installed.

``` bash
npm install -g cdk
```

* Bootstrapping your account ​

AWS CDK creates a stack of resources it needs in your aws account to insure it can deploy your projects later, like an S3 bucket to upload assets.

``` bash
cdk bootstrap
```

**2. Creating a project** ​

In our case we’re going to use python as our language of choice

``` bash
mkdir codecommit-slack-bot
cd codecommit-slack-bot
cdk init --language=python
```

A boilerplate project gets created alongside a virtualenv.

``` bash
.
├── .env
├── README.md
├── app.py
├── cdk.json
├── requirements.txt
├── setup.py
└── tuto_cdk    
	├── __init__.py    
	└── tuto_cdk_stack.py
```

To activate the virtualenv and install the requirements.

``` bash
source .env/bin/activatepip install -r requirements.txt
```

**3. Understanding the boilerplate**

* *app.py* file ​

*app.py* is the project starting point and it looks like this:

``` python
#!/usr/bin/env python3

​from aws_cdk import core
​from codecommit_slack_bot.codecommit_slack_bot_stack import CodecommitSlackBotStack

​​app = core.App()
CodecommitSlackBotStack(app, "codecommit-slack-bot")​
app.synth()
```

`app = core.App()` initializes the CDK.

`CodecommitSlackBotStack(app, "codecommit-slack-bot")` creates our only stack, but we can have multiple ones. 

`app.synth()` this method is what transforms the stacks we defined into cloudformation templates.

* *codecommit_slack_bot_stack.py* file ​

Now let’s have a look at the only stack created for us.

``` python
from aws_cdk import core
​​class CodecommitSlackBotStack(core.Stack):
​    def __init__(self, scope: core.Construct, id: str, **kwargs) -> None:
        super().__init__(scope, id, **kwargs)
        # The code that defines your stack goes here
```

​ There’s already a comment in the boilerplate which gives us a hint on where to put our code that defines our stack of resources, all should be in the constructor of our class. 

This class gets instantiated in the `app.py` into an object, that’s all what CDK needs to understand that you want to create a stack with all the resources you define in it. ​

**4. Writing our own code** ​

In our stack file called *codecommit_slack_bot_stack.py* we are going to define the resources we want to add. ​ We started by modifying the constructor of our stack to accept 3 more parameters which are the `lambda_path` , `slack_webhook_url` and `codecommit_repo_arn`. 

``` python
def __init__(self, scope: core.Construct, id: str, lambda_path: str,
                 slack_webhook_url: str, codecommit_repo_arn: str,
                 **kwargs) -> None:
```

Our goal is to achieve the workflow below:

![complete workflow](/assets/img/posts/codecommit-slack2.png){:class="img-fluid"}

* **IAM Role for our Lambda Function** ​

We start by importing `aws_iam` from cdk

``` python
from aws_cdk import core, aws_iam
```

​This role should allow us to access codecommit to retrieve details about the repository and the more details about the actions triggered. It also allows our Lambda function to store Logs in Cloudwatch. For That we start by creating our Policy document.

``` python
lambda_role_policy = aws_iam.PolicyDocument(
    statements=[
        aws_iam.PolicyStatement(
            effect=aws_iam.Effect.ALLOW,
            actions=['codecommit:*'],
            resources=[codecommit_repo_arn]
        ),
        aws_iam.PolicyStatement(
            effect=aws_iam.Effect.ALLOW,
            actions=[
                'logs:CreateLogGroup',
                'logs:CreateLogStream',
                'logs:PutLogEvents'
            ],
            resources=['*']
        )
    ]
)
```

​ Now that the policy is ready, we can create our role that can be assumed by aws lambda service and attach the policy document to it. ​

``` python
lambda_role = aws_iam.Role(
    self,
    id='role',
    assumed_by=aws_iam.ServicePrincipal('lambda.amazonaws.com'),
    inline_policies={
        'policy': lambda_role_policy
    }
)
```

* **AWS Lambda function** ​

It’s time to create our Lambda function, for that we’re going to add aws_lambda to the import statement.

``` python
from aws_cdk import core, aws_iam, aws_lambda
```

​ Our Lambda function will get the source code from the lambda_path argument and also will set an environment variable called `SLACK_WEBHOOK_URL` with the value of the argument `slack_webhook_url` . We’re also going to attach the role we created earlier to it. ​ ​

``` python
lambda_function = aws_lambda.Function(
    self,
    id='handler',
    code=aws_lambda.Code.from_asset(lambda_path),
    handler='lambda_function.lambda_handler',
    timeout=core.Duration.seconds(300),
    runtime=aws_lambda.Runtime.PYTHON_3_7,
    role=lambda_role,
    environment={
        'SLACK_WEBHOOK_URL': slack_webhook_url
    }
)
```

​

* **Cloudwatch Rule** ​

Now that we have our lambda defined, the missing part is creating the CloudWatch event rule which will listen to all events in our repository and triggers a lambda every time an event occurs. For that we are going to add both `aws_events` and `aws_events_targets` to the import statement. 

``` python
from aws_cdk import core, aws_iam, aws_lambda, aws_events, aws_events_targets
```

​ Then we define the rule by defining the pattern first, then the actual rule ​

``` python
pattern = aws_events.EventPattern(
    resources=[codecommit_repo_arn],
    source=['aws.codecommit']
)​
aws_events.Rule(
    self,
    id='cloudwatch-rule',
    event_pattern=pattern,
    targets=[aws_events_targets.LambdaFunction(lambda_function)]
)
```

Now that our stack is fully defined, we can start working on the other files. ​

First of all, we’re going to put some configuration in the *cdk.json* file, notably the `repository arn` and the `slack webhook url` using the [context](https://docs.aws.amazon.com/cdk/latest/guide/context.html) mechanism offered by CDK

``` json
{
    "app": "python3 app.py",
    "context": {
        "codecommit_repo_arn": "arn:aws:codecommit:<region>:<account>:<repository_name>",
        "slack_webhook_url": "<slack_webhook_url>"
    }
}
```

Then we’re going to try to access those from the `app.py` and extract information in there

``` 
codecommit_repo_arn = app.node.try_get_context('codecommit_repo_arn')
slack_webhook_url = app.node.try_get_context('slack_webhook_url')
​repository_details = core.Arn.parse(codecommit_repo_arn)
account = repository_details.account
region = repository_details.region
repository_name = repository_details.resource
```

​ And we’re also going to define the path of our aws lambda’s source code

``` 
import osproject_root_path = path.dirname(path.realpath(__file__))
lambda_source_code_path = path.join(project_root_path, 'lambda_handler')
```

The source code of our lambda function can be found [here](https://github.com/Think-iT-Labs/CodeCommit-Slackbot/tree/master/lambda_handler).​

Finally, we’re going to modify the instantiation of our stack to pass all new arguments needed for it. ​

``` python
env = core.Environment(account=account, region=region)
​CodecommitSlackBotStack(
    app,
    id=f'codecommit-slack-bot-{repository_name}',
    lambda_path=lambda_source_code_path,
    slack_webhook_url=slack_webhook_url,
    codecommit_repo_arn=codecommit_repo_arn,
    env=env
)
​app.synth()
```

The source code for the project can be found [here](https://github.com/Think-iT-Labs/CodeCommit-Slackbot)

## Conclusion
Infrastructure as Code (IaC) is powerful,  it is capable of keeping track of your infrastructure. With more than one single language, CDK gives developers the ability to define their stacks of infrastructure using their preferred programming language, saving time invested for learning purposes required for using other solutions.
