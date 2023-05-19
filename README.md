# CDK Lab - Serverless Task Scheduler<!-- omit from toc -->

- [Components](#components)
- [Deployment](#deployment)
  - [Step 1: Configure AWS Credentials](#step-1-configure-aws-credentials)
  - [Step 2: Set the stacks' prefix](#step-2-set-the-stacks-prefix)
  - [Step 3: Create Task Definitions](#step-3-create-task-definitions)
  - [Step 4: Install Python Lambda Layer (Python dependencies)](#step-4-install-python-lambda-layer-python-dependencies)
  - [Step 5: Add Business Logics code](#step-5-add-business-logics-code)
  - [Step 6: Change time related parameters of SQS Queue](#step-6-change-time-related-parameters-of-sqs-queue)
  - [Step 7: Deploy with CDK toolkit (`cdk` command)](#step-7-deploy-with-cdk-toolkit-cdk-command)
  - [(Optional) Step 8: Clean all resources](#optional-step-8-clean-all-resources)
- [Some Development Tips \& Explainations](#some-development-tips--explainations)
  - [Role creation](#role-creation)
  - [JSON serialization in Lambda function](#json-serialization-in-lambda-function)


I used to schedule some cron tasks and send the results to IM application, such as Slack, Discord ...etc.

What I did before was implementing a gigantic, stable, famous workflow platform like [Apache Airflow](https://airflow.apache.org) and [Prefect](https://www.prefect.io). They are still good choices, but it's time for me to adopt a lighter, event-driven architecture.

When talk about AWS, we can use Amazon EventBridge, AWS Lambda, Amazon SQS to build the same architecture to fulfill the same job:

![](./diagram-services.jpg)

Hope this will cost less money and effort. Let's get hands dirty 🛠️ and have the party started! 🎉

## Components

![](diagram-detailed.jpg)

All we need are:

* **EventBridge Scheduler (Event Scheduler)**: Schedule to invoke a Lambda function with a Role attached
  * **Service Role**: Has permission to
    * Invoke a Lambda function (`lambda:InvokeFunction`)
* **Lambda function (Send Tasks)**: Send messages to queue with a Role attached
  * **Service Role**: Has permission to
    * Send messages to SQS (`sqs:SendMessage`)
    * Manage CloudWatch logs (`logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`)
* **Queue**: Store the tasks
* **Lambda function (Run Task)**: Receive messages from queue with a Role attached
  * **Service Role**: Has permission to
      * Receive message from queue (`sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes`)
      * Manage CloudWatch logs (`logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`)

The creation order might be:

> [TODO] Update the latest workflow

1. Create Lambda functions with roles
2. Create queue
3. Grant SQS permissions to Lambda's roles
   1. `aws_cdk.aws_sqs.Queue.grant_send_messages()`
   2. `aws_cdk.aws_sqs.Queue.grant_consume_messages()`
4. ([NOTE]) Create a role for scheduler, with `lambda:InvokeFunction` permission
5. ([NOTE]) Create a scheduler, attach a Role, set a Lambda target

> [NOTE] According to the [document](https://docs.aws.amazon.com/cdk/api/v2/python/aws_cdk.aws_scheduler/README.html): Currently (May 2023, version `v2.79.1`) there is still no L2 construct yet, so the scheduler object doesn't have a method like `grant_a_permission()` to use.
>
> If L2 construct is available in the future, change the order as below:

1. Create Lambda functions with roles
2. Create queue
3. Grant SQS permissions to Lambda's roles
   1. `aws_cdk.aws_sqs.Queue.grant_send_messages()`
   2. `aws_cdk.aws_sqs.Queue.grant_consume_messages()`
4. Create a scheduler, set a Lambda target
5. Grant Lambda permission to scheduler role
   1. `aws_cdk.aws_lambda.Function.grant_invoke()`

## Deployment

### Step 1: Configure AWS Credentials

Install [AWS CLi](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) first, then create the config file by executing:

```bash
aws configure
```

### Step 2: Set the stacks' prefix

The deployed stack has a prefix, the default value is `cdklab`. Edit the value in [runtime context](https://docs.aws.amazon.com/cdk/v2/guide/context.html) file (`cdk.context.json`):

```json
{
  "prefix": "cdklab"
}
```

### Step 3: Create Task Definitions

By default, the tasks are stored in the `tasks` List object in `lambda/send_task/index.py`. Such as:

```python
tasks = [
  "Hello from Lambda #1!",
  "Hello from Lambda #2!"
]
```

Simply modify the content and each String object will be the task that the next (run-task) Lambda function should execute with.

If the task definitions is much larger than this example, try store into another text file then read it in:

```python
# e.g. The tasks are now in tasks.txt, each line represents a unique task definition
with open("tasks.txt", "r") as file:
    tasks = file.open().splitlines()
```

### Step 4: Install Python Lambda Layer (Python dependencies)

All Python dependencies must stored under `layer/python` as [Lambda Layer](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html) then pack and send to AWS Lambda.

First add the dependencies to `requirements-layer.txt`, then install the dependencies with the command below: 

```bash
pip install --target ./layer/python -r requirements-layer.txt
```

> [NOTE] Remember to manually add dependencies to `requirements-layer.txt`.

### Step 5: Add Business Logics code

Insert the code into `lambda/run_task/index.py`.

### Step 6: Change time related parameters of SQS Queue

There are 4 parameters:

* [`visibility_timeout`](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html): How long does a Lambda function process the task?
* [`retention_period`](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SetQueueAttributes.html#API_SetQueueAttributes_RequestSyntax): How long does the task remain in the queue?
* [`receive_message_wait_time`](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/): How long does the "ReceiveMessage" (Polling) action?takes 

Estimate the settings by the program you goona run, end override them by editing the `stack/sqs_stack.py` file.

### Step 7: Deploy with CDK toolkit (`cdk` command)

[Install the CDK toolkit](https://docs.aws.amazon.com/cdk/v2/guide/cli.html) then deploy by executing:

```bash
cdk deploy --all --require-approval=never
```

### (Optional) Step 8: Clean all resources

Not going to use these anymore? Remove them with:

```bash
cdk destroy --all
```

## Some Development Tips & Explainations

### Role creation

CDK will try to create roles and add necessary permission policies by evaluate the resources that are going to be created. So, if we create a Lambda function, and set a SQS Queue to trigger the function, CDK will automatically create the policies below:

* CloudWatch log related policies
* Read SQS Queue message related policies

But there are exceptions in this project:

1. If the action is written in Lambda function, CDK won't understand what permission does the function need (Refer to `stacks.lambda_stack.lambda_sendtask`)
2. If we use L1 construct to create the resources, everything in the CloudFormation template should be created by ourselves (Refer to `stacks.scheduler_stack.scheduler_lambda`)

If these situations exists, try to create the Roles and Policies manually first.

### JSON serialization in Lambda function

We can use the standard `JSON` library and call `json.dumps()` function to acomplish the serialization process. But try to do with the code below:

```python
import json

json.dumps(json)
# TypeError: Object of type module is not JSON serializable
```

Error occurs: module object can not be JSON serialized. Sometimes it's annoying because we can't realize if the object is serializable or not at a glance.

[JSONPickle](https://github.com/jsonpickle/jsonpickle) is an amazing project for JSON serialization. The reason to use JSONPickle better than calling `json.dumps` is based on the project itself's description:

> "It can take almost any Python object and turn the object into JSON"

We added into the Python Lambda Layer by default and already implemented into the Lambda function. Try and enjoy it! 🍻
