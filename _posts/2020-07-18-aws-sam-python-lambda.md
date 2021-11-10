---
layout: post
title:  "Deploying Python Lambda function using AWS SAM"
date:   2020-07-18
categories: [blog]
tags: [aws, sam, serverless, s3, python]
excerpt_separator: <!--more-->
---
This post is going to be different in that you will see AWS SAM in somewhat different action, not just API and Node.js.

I like the concept of native tooling, picking a cloud vendor & then using their tools to build and manage infrastructure is cost effective & in most cases makes a lot of sense.\
In few AWS projects I solely used CloudFormation for spinning up infrastructure. I also have a liking for [AWS SAM]( https://aws.amazon.com/serverless/sam/) which is an open source framework for building serverless applications.
Most of the examples on the internet are about deploying API using AWS SAM & I thought I will try something slightly different.
How about just deploying a Python Lambda function to shutdown EC2 instances and see what SAM can do. After all SAM is an extension to AWS CloudFormation but it is strictly not only for API.\
NOTE --> There is an AWS solution to configure start and stop schedules for EC2 and Amazon RDS instances known as [AWS Instance Scheduler](https://aws.amazon.com/about-aws/whats-new/2018/02/introducing-the-aws-instance-scheduler/)

<!--more-->

A note here that I am using [AWS Cloud9]( https://aws.amazon.com/cloud9/) as IDE & it comes with almost all AWS tools pre-installed. 
I understand that CloudFormation should have been used for work like this but I wanted to play with SAM & nothing simpler was coming to my mind.

I started with generating a scaffold for our Lambda function which is fairly easy with AWS SAM.

```bash
OpsToDevOps $ sam init --name ec2_shutdown --runtime python3.6
Which template source would you like to use?
        1 - AWS Quick Start Templates
        2 - Custom Template Location
Choice: 1

Allow SAM CLI to download AWS-provided quick start templates from Github [Y/n]: n

-----------------------
Generating application:
-----------------------
Name: ec2_shutdown
Runtime: python3.6
Dependency Manager: pip
Application Template: hello-world
Output Directory: .

Next steps can be found in the README file at ./ec2_shutdown/README.md
```
Next I switched into the root of newly created directory & changed the name of `default` directory to something meaningful that relates to my Python code. 
That `hello_world` directory is default but it has template and files that I will be using.

```bash    
OpsToDevOps/ec2_shutdown $ cd ec2_shutdown/
OpsToDevOps/ec2_shutdown $ mv hello_world/ ec2_shutdown
OpsToDevOps/ec2_shutdown $ tree
.
├── ec2_shutdown
│   ├── app.py
│   ├── __init__.py
│   ├── __pycache__
│   │   ├── app.cpython-36.pyc
│   │   └── __init__.cpython-36.pyc
│   └── requirements.txt
├── events
│   └── event.json
├── README.md
├── template.yaml
└── tests
    └── unit
        ├── __init__.py
        ├── __pycache__
        │   ├── __init__.cpython-36.pyc
        │   └── test_handler.cpython-36.pyc
        └── test_handler.py

6 directories, 12 files
```

That’s a lot of files but the most important are the `app.py` which is the Python Lambda function; the `template.yaml` which is the SAM template; the `event.json` which contains a sample event for testing; and the `README.md` which contains further documentation.
The `requirements.txt` file specifies the Lambda function’s Python dependencies.

The remaining `__init__.py` and the `.pyc` files are Pythonisms that I am not going to be using, the unit tests are in the tests directory; `test_handler.py` file specifically.

I am going to make changes to only 2 files in there, `template.yaml` file and `app.py`. 

`app.py`
```python{% raw %}
import boto3
import os
import sys

region = os.environ['AWSREGION']
instances = os.environ['INSTANCE1'], os.environ['INSTANCE2']
ec2 = boto3.client('ec2', region_name=region)

def lambda_handler(event, context):
    try:
        ec2.stop_instances(InstanceIds=instances)
        print(f"Stopped your instance {instances}")

    except Exception as err:
        print(f"An error occured {err}")
        sys.exit()
```{% endraw %}
The Python code is nothing fancy, no looping, no arguments. Plain code for shutting down EC2 instances which are being passed as `ENVIRONMENT VARIABLES`. I could have written a better code but the objective here is to simply deploy a Lambda function.


`template.yaml`
```yaml{% raw %}
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  ec2_shutdown

  Sample SAM Template for ec2_shutdown

Globals:
  Function:
    Timeout: 60

Resources:
  ec2ShutDownIAMRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
              - lambda.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      Policies:
        - PolicyName: ec2shutdownrolepolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action: [
                  "logs:CreateLogGroup",
                  "logs:CreateLogStream",
                  "logs:PutLogEvents"
                ]
                Resource: "arn:aws:logs:*:*:*"
              - Effect: Allow
                Action: [
                  "ec2:Start*",
                  "ec2:Stop*"
                ]
                Resource: "*"
  ec2ShutDownFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: ec2_shutdown
      Description: Schedule shutdown of ec2 instances
      Role: !GetAtt ec2ShutDownIAMRole.Arn
      CodeUri: ec2_shutdown/
      Handler: app.lambda_handler
      Runtime: python3.6
      Environment:
        Variables:
          AWSREGION: 'us-east-1'
          INSTANCE1: 'i-XXXXXXXXXXXXXXXXX'
          INSTANCE2: 'i-XXXXXXXXXXXXXXXXX'
      Events:
        ec2ShutDown:
          Type: Schedule
          Properties:
            Schedule: cron(0 12 * * ? *)

Outputs:
  ec2ShutDownFunction:
    Description: "ec2 shutdown Lambda Function ARN"
    Value: !GetAtt ec2ShutDownFunction.Arn
  ec2ShutDownIAMRole:
    Description: "IAM Role created for ec2 shutdown function"
    Value: !GetAtt ec2ShutDownIAMRole.Arn
  ec2ShutDownFunctionRole:
    Description: "Implicit IAM Role created for ec2 shutdown function"
    Value: !GetAtt ec2ShutDownFunction.Arn
```{% endraw %}

I have made modifications to the `YAML` file and the only resource I am adding in there is the IAM role for Lambda to interact with EC2 instances. 
Make sure to change the `AWSREGION`, `INSTANCE` or `INSTANCE2` to whatever values you have in your environment. The default Lambda role will be created by SAM on its own.\
You can read more about [CRON jobs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html)\
Next, I am going to create a S3 bucket which I will use to upload the code.

```bash
OpsToDevOps/ec2_shutdown $ aws s3 mb s3://sam-ec2shutdown-bucket
make_bucket: sam-ec2shutdown-bucket
OpsToDevOps/ec2_shutdown $ sam validate --template template.yaml
/ec2_shutdown/template.yaml is a valid SAM Template
```
Now comes the part of building, packaging and deploying the code. Take note that in my code, I am not creating a Lambda role for my Lambda function. That part is taken care of by AWS SAM. You will see the role in the output at the end. Also, because SAM is an extension to CloudFormation, you can see what has been built, all resources, the Lambda code and the template by logging in to the AWS console with you credentials.

```bash
OpsToDevOps/ec2_shutdown $ sam build
Building resource 'ec2ShutDownFunction'
Running PythonPipBuilder:ResolveDependencies
Running PythonPipBuilder:CopySource

Build Succeeded

Built Artifacts  : .aws-sam/build
Built Template   : .aws-sam/build/template.yaml

Commands you can use next
=========================
[*] Invoke Function: sam local invoke
[*] Deploy: sam deploy –guided

OpsToDevOps/ec2_shutdown $ sam package --output-template-file ec2shutdown-packaged.yaml --s3-bucket sam-ec2shutdown-bucket
Uploading to a7a6e47c7ccca0228aa7109c9c45269e  537806 / 537806.0  (100.00%)

Successfully packaged artifacts and wrote output template to file ec2shutdown-packaged.yaml.
Execute the following command to deploy the packaged template
sam deploy --template-file /ec2-shutdown/ec2shutdown-packaged.yaml --stack-name <YOUR STACK NAME>

OpsToDevOps/ec2_shutdown $ sam deploy --template-file /ec2-shutdown/ec2shutdown-packaged.yaml --stack-name ec2shutdown-stack --capabilities CAPABILITY_IAM --region us-east-1 --s3-bucket sam-ec2shutdown-bucket

        Deploying with following values
        ===============================
        Stack name                 : ec2shutdown-stack
        Region                     : us-east-1
        Confirm changeset          : False
        Deployment s3 bucket       : sam-ec2shutdown-bucket
        Capabilities               : ["CAPABILITY_IAM"]
        Parameter overrides        : {}

Initiating deployment
=====================
Uploading to 4db2e6632065a65808aefba1246fd68f.template  2051 / 2051.0  (100.00%)

Waiting for changeset to be created..

CloudFormation stack changeset
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Operation                                                             LogicalResourceId                                                     ResourceType                                                        
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+ Add                                                                 ec2ShutDownFunctionec2ShutDownPermission                              AWS::Lambda::Permission                                             
+ Add                                                                 ec2ShutDownFunctionec2ShutDown                                        AWS::Events::Rule                                                   
+ Add                                                                 ec2ShutDownFunction                                                   AWS::Lambda::Function                                               
+ Add                                                                 ec2ShutDownIAMRole                                                    AWS::IAM::Role                                                      
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Changeset created successfully. arn:aws:cloudformation:us-east-1:031600986769:changeSet/samcli-deploy1595061533/6db347a5-e15a-4dcb-bc1c-5d982a016a6c


2020-07-18 08:39:04 - Waiting for stack create/update to complete

CloudFormation events from changeset
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
ResourceStatus                                      ResourceType                                        LogicalResourceId                                   ResourceStatusReason                              
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
CREATE_IN_PROGRESS                                  AWS::IAM::Role                                      ec2ShutDownIAMRole                                  -                                                 
CREATE_IN_PROGRESS                                  AWS::IAM::Role                                      ec2ShutDownIAMRole                                  Resource creation Initiated                       
CREATE_COMPLETE                                     AWS::IAM::Role                                      ec2ShutDownIAMRole                                  -                                                 
CREATE_IN_PROGRESS                                  AWS::Lambda::Function                               ec2ShutDownFunction                                 -                                                 
CREATE_IN_PROGRESS                                  AWS::Lambda::Function                               ec2ShutDownFunction                                 Resource creation Initiated                       
CREATE_COMPLETE                                     AWS::Lambda::Function                               ec2ShutDownFunction                                 -                                                 
CREATE_IN_PROGRESS                                  AWS::Events::Rule                                   ec2ShutDownFunctionec2ShutDown                      -                                                 
CREATE_IN_PROGRESS                                  AWS::Events::Rule                                   ec2ShutDownFunctionec2ShutDown                      Resource creation Initiated                       
CREATE_COMPLETE                                     AWS::Events::Rule                                   ec2ShutDownFunctionec2ShutDown                      -                                                 
CREATE_IN_PROGRESS                                  AWS::Lambda::Permission                             ec2ShutDownFunctionec2ShutDownPermission            Resource creation Initiated                       
CREATE_IN_PROGRESS                                  AWS::Lambda::Permission                             ec2ShutDownFunctionec2ShutDownPermission            -                                                 
CREATE_COMPLETE                                     AWS::Lambda::Permission                             ec2ShutDownFunctionec2ShutDownPermission            -                                                 
CREATE_COMPLETE                                     AWS::CloudFormation::Stack                          ec2shutdown-stack                                   -                                                 
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Stack ec2shutdown-stack outputs:
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
OutputKey-Description                                                                                   OutputValue                                                                                           
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
ec2ShutDownFunctionRole - Implicit IAM Role created for Hello World function                            arn:aws:lambda:us-east-1:031600986769:function:ec2_shutdown                                           
ec2ShutDownIAMRole - IAM Role created for ec2 shutdown function                                         arn:aws:iam::031600986769:role/ec2shutdown-stack-ec2ShutDownIAMRole-FB05I5CIB973                      
ec2ShutDownFunction - ec2 shutdown Lambda Function ARN                                                  arn:aws:lambda:us-east-1:031600986769:function:ec2_shutdown                                           
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Successfully created/updated stack - ec2shutdown-stack in us-east-1
```

The code is successfully deployed. I could have opted for the `--guided` option which prompts for user inputs along the way before deploying the function. 
Here I don't have to give S3 bucket name & SAM will create one to upload and manage the code with `versioning` enabled.

```bash
OpsToDevOps/ec2_shutdown$ sam deploy --guided

Configuring SAM deploy
======================

        Looking for samconfig.toml :  Not found

        Setting default arguments for 'sam deploy'
        =========================================
        Stack Name [sam-app]: ec2shutdown-stack
        AWS Region [us-east-1]: 
        #Shows you resources changes to be deployed and require a 'Y' to initiate deploy
        Confirm changes before deploy [y/N]: y
        #SAM needs permission to be able to create roles to connect to the resources in your template
        Allow SAM CLI IAM role creation [Y/n]: Y
        Save arguments to samconfig.toml [Y/n]: n

        Looking for resources needed for deployment: Found!

                Managed S3 bucket: aws-sam-cli-managed-default-samclisourcebucket-32bdiubojyr8
                A different default S3 bucket can be set in samconfig.toml

        Deploying with following values
        ===============================
        Stack name                 : ec2shutdown-stack
        Region                     : us-east-1
        Confirm changeset          : True
        Deployment s3 bucket       : aws-sam-cli-managed-default-samclisourcebucket-32bdiubojyr8
        Capabilities               : ["CAPABILITY_IAM"]
        Parameter overrides        : {}

Initiating deployment
=====================
Uploading to ec2shutdown-stack/5df3704680c937de9704359ef640cccc  537842 / 537842.0  (100.00%)
Uploading to ec2shutdown-stack/c7a3b56abef48d06afb625611339a04b.template  1925 / 1925.0  (100.00%)

Waiting for changeset to be created..

CloudFormation stack changeset
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Operation                                                             LogicalResourceId                                                     ResourceType                                                        
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
* Modify                                                              ec2ShutDownFunctionec2ShutDownPermission                              AWS::Lambda::Permission                                             
* Modify                                                              ec2ShutDownFunctionec2ShutDown                                        AWS::Events::Rule                                                   
* Modify                                                              ec2ShutDownFunction                                                   AWS::Lambda::Function                                               
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Changeset created successfully. arn:aws:cloudformation:us-east-1:565314625672:changeSet/samcli-deploy1595048954/f8953337-fb09-4aba-9d69-06f89755cdb2


Previewing CloudFormation changeset before deployment
======================================================
Deploy this changeset? [y/N]: y

2020-07-18 05:09:29 - Waiting for stack create/update to complete

CloudFormation events from changeset
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
ResourceStatus                                      ResourceType                                        LogicalResourceId                                   ResourceStatusReason                              
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
UPDATE_IN_PROGRESS                                  AWS::Lambda::Function                               ec2ShutDownFunction                                 -                                                 
UPDATE_COMPLETE                                     AWS::Lambda::Function                               ec2ShutDownFunction                                 -                                                 
UPDATE_COMPLETE_CLEANUP_IN_PROGRESS                 AWS::CloudFormation::Stack                          ec2shutdown-stack                                   -                                                 
UPDATE_COMPLETE                                     AWS::CloudFormation::Stack                          ec2shutdown-stack                                   -                                                 
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Stack ec2shutdown-stack outputs:
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
OutputKey-Description                                                                                   OutputValue                                                                                           
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
ec2ShutDownFunctionRole - Implicit IAM Role created for Hello World function                            arn:aws:lambda:us-east-1:031600986769:function:ec2_shutdown                                           
ec2ShutDownIAMRole - IAM Role created for ec2 shutdown function                                         arn:aws:iam::031600986769:role/ec2shutdown-stack-ec2ShutDownIAMRole-FB05I5CIB973                      
ec2ShutDownFunction - ec2 shutdown Lambda Function ARN                                                  arn:aws:lambda:us-east-1:031600986769:function:ec2_shutdown                                           
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Successfully created/updated stack - ec2shutdown-stack in us-east-1 1
```

I can use `sam logs` command to retrieve the Lambda log files. If I want the logs for the function I just called, in real time I can use `-tail` or for debugging use ` --debug`.

```bash
OpsToDevOps $ sam logs --name ec2ShutDownFunction --stack-name ec2shutdown-stack --tail
```
```bash
2020/07/18/[$LATEST]f3cb3dff3f494d5d99f14599ee6054a5 2020-07-18T07:02:01.549000 START RequestId: 71101d4f-18fc-406b-b231-9abda6b5af6e Version: $LATEST
2020/07/18/[$LATEST]f3cb3dff3f494d5d99f14599ee6054a5 2020-07-18T07:02:02.185000 stopped your instance ('i-09049786b25eXXXXX', 'i-094005052b02XXXXX')
2020/07/18/[$LATEST]f3cb3dff3f494d5d99f14599ee6054a5 2020-07-18T07:02:02.186000 END RequestId: 71101d4f-18fc-406b-b231-9abda6b5af6e
2020/07/18/[$LATEST]f3cb3dff3f494d5d99f14599ee6054a5 2020-07-18T07:02:02.186000 REPORT RequestId: 71101d4f-18fc-406b-b231-9abda6b5af6e  Duration: 636.61 ms     Billed Duration: 700 ms Memory Size: 128 MB     Max Memory Used: 74 MB    Init Duration: 326.28 ms
```
![sam-tail_1a](/assets/sam-tail-01.PNG)

You can do a whole lot with SAM and this is just the surface, pass in parameters, use intrinsic functions & the whole gamut. This was a simple demonstration to show the beginnings.
Oh and if you want to START the instances instead of STOPPING then change the code in `app.py` with `ec2.start_instances(InstanceIds=instances)`

Hope you enjoyed this post. 

Stay safe!
