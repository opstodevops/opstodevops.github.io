---
layout: post
title:  "Event-driven architecture using AWS EventBridge Part 2"
date:   2020-09-16
categories: [blog]
tags: [aws, event-driven architecture, event driven, serverless, eventbridge, SAM]
excerpt_separator: <!--more-->
---
Continuing from my last post on Eventing [AWS EventBridge 101]({% post_url 2020-08-17-aws-eventbridge-101 %}), in this post I will attempt to simplify AWS EventBridge with a use case. 
For sake of simplicity, our case is going to be linear though there is nothing stopping you from branching out.

For this use case, we have a function that takes orders from clients, once the order is processed, we want a function triggered to generate an invoice.

Below is an illustration of what I mean, as you can see it is a simple structure.

![ec2 create](/assets/eb-lin-demo-1.PNG)

<!--more-->

I will be using AWS SAM to deploy our structure.\
Beware that there is no actual website involved & all the code does is create an order function (simulating an order) & when that function is invoked, it will trigger the invoice function based on AWS EventBridge rule.

You can find the code on my [GitHub repo](https://github.com/opstodevops/eventbridge-sam-python-example)

Let us clone the repository and have some fun with AWS EventBridge.

```bash{% raw %}
OpsToDevOps: $ git clone https://github.com/opstodevops/eventbridge-sam-python-example.git
Cloning into 'eventbridge-sam-python-example'...
remote: Enumerating objects: 40, done.
remote: Counting objects: 100% (40/40), done.
remote: Compressing objects: 100% (26/26), done.
remote: Total 40 (delta 11), reused 34 (delta 8), pack-reused 0
Unpacking objects: 100% (40/40), done.
OpsToDevOps: $ cd eventbridge-sam-python-example/
OpsToDevOps: $ sam build

        SAM CLI now collects telemetry to better understand customer needs.

        You can OPT OUT and disable telemetry collection by setting the
        environment variable SAM_CLI_TELEMETRY=0 in your shell.
        Thanks for your help!

        Learn More: https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-telemetry.html

Building function 'orderServiceFunction'
Running PythonPipBuilder:ResolveDependencies
Running PythonPipBuilder:CopySource
Building function 'invoiceServiceFunction'
Running PythonPipBuilder:ResolveDependencies
Running PythonPipBuilder:CopySource

Build Succeeded

Built Artifacts  : .aws-sam/build
Built Template   : .aws-sam/build/template.yaml

Commands you can use next
=========================
[*] Invoke Function: sam local invoke
[*] Deploy: sam deploy --guided
    
OpsToDevOps: $
```{% endraw %}

Now that our build has succeeded, we are going to use ```sam deploy –guided``` to deploy our code. 
I covered AWS SAM in one of my earlier articles [AWS SAM]({% post_url 2020-07-18-aws-sam-python-lambda %}).

```bash{% raw %}
OpsToDevOps: $ sam deploy --guided

Configuring SAM deploy
======================

        Looking for samconfig.toml :  Not found

        Setting default arguments for 'sam deploy'
        =========================================
        Stack Name [sam-app]: eb-demo
        AWS Region [us-east-1]: 
        #Shows you resources changes to be deployed and require a 'Y' to initiate deploy
        Confirm changes before deploy [y/N]: N
        #SAM needs permission to be able to create roles to connect to the resources in your template
        Allow SAM CLI IAM role creation [Y/n]: Y
        Save arguments to samconfig.toml [Y/n]: Y

        Looking for resources needed for deployment: Not found.
        Creating the required resources...
        Successfully created!

                Managed S3 bucket: aws-sam-cli-managed-default-samclisourcebucket-s3s16b35vbvs
                A different default S3 bucket can be set in samconfig.toml

        Saved arguments to config file
        Running 'sam deploy' for future deployments will use the parameters saved above.
        The above parameters can be changed by modifying samconfig.toml
        Learn more about samconfig.toml syntax at 
        https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-config.html
Uploading to eb-demo/6450414f06417847fdcd1d879940e2f2  538699 / 538699.0  (100.00%)
Uploading to eb-demo/4e65c9c6d740249e9bc3c310d63e45d6  538649 / 538649.0  (100.00%)

        Deploying with following values
        ===============================
        Stack name                 : eb-demo
        Region                     : us-east-1
        Confirm changeset          : False
        Deployment s3 bucket       : aws-sam-cli-managed-default-samclisourcebucket-s3s16b35vbvs
        Capabilities               : ["CAPABILITY_IAM"]
        Parameter overrides        : {}

Initiating deployment
=====================
Uploading to eb-demo/46b57646b6bbb3c7b111d9f344edab90.template  2230 / 2230.0  (100.00%)

Waiting for changeset to be created..

CloudFormation stack changeset
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Operation                                                            LogicalResourceId                                                    ResourceType                                                       
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+ Add                                                                EventRule                                                            AWS::Events::Rule                                                  
+ Add                                                                PermissionForEventsToInvokeLambda                                    AWS::Lambda::Permission                                            
+ Add                                                                invoiceServiceFunctionRole                                           AWS::IAM::Role                                                     
+ Add                                                                invoiceServiceFunction                                               AWS::Lambda::Function                                              
+ Add                                                                orderServiceFunctionRole                                             AWS::IAM::Role                                                     
+ Add                                                                orderServiceFunction                                                 AWS::Lambda::Function                                              
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Changeset created successfully. arn:aws:cloudformation:us-east-1:853595176342:changeSet/samcli-deploy1600216408/d128c213-d6e0-48f0-8e50-20012db4b444


2020-09-16 00:33:39 - Waiting for stack create/update to complete

CloudFormation events from changeset
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
ResourceStatus                                      ResourceType                                        LogicalResourceId                                   ResourceStatusReason                              
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
CREATE_IN_PROGRESS                                  AWS::IAM::Role                                      orderServiceFunctionRole                            -                                                 
CREATE_IN_PROGRESS                                  AWS::IAM::Role                                      invoiceServiceFunctionRole                          -                                                 
CREATE_IN_PROGRESS                                  AWS::IAM::Role                                      invoiceServiceFunctionRole                          Resource creation Initiated                       
CREATE_IN_PROGRESS                                  AWS::IAM::Role                                      orderServiceFunctionRole                            Resource creation Initiated                       
CREATE_COMPLETE                                     AWS::IAM::Role                                      invoiceServiceFunctionRole                          -                                                 
CREATE_COMPLETE                                     AWS::IAM::Role                                      orderServiceFunctionRole                            -                                                 
CREATE_IN_PROGRESS                                  AWS::Lambda::Function                               invoiceServiceFunction                              -                                                 
CREATE_IN_PROGRESS                                  AWS::Lambda::Function                               invoiceServiceFunction                              Resource creation Initiated                       
CREATE_COMPLETE                                     AWS::Lambda::Function                               invoiceServiceFunction                              -                                                 
CREATE_IN_PROGRESS                                  AWS::Lambda::Function                               orderServiceFunction                                -                                                 
CREATE_IN_PROGRESS                                  AWS::Lambda::Function                               orderServiceFunction                                Resource creation Initiated                       
CREATE_IN_PROGRESS                                  AWS::Events::Rule                                   EventRule                                           -                                                 
CREATE_COMPLETE                                     AWS::Lambda::Function                               orderServiceFunction                                -                                                 
CREATE_IN_PROGRESS                                  AWS::Events::Rule                                   EventRule                                           Resource creation Initiated                       
CREATE_COMPLETE                                     AWS::Events::Rule                                   EventRule                                           -                                                 
CREATE_IN_PROGRESS                                  AWS::Lambda::Permission                             PermissionForEventsToInvokeLambda                   Resource creation Initiated                       
CREATE_IN_PROGRESS                                  AWS::Lambda::Permission                             PermissionForEventsToInvokeLambda                   -                                                 
CREATE_COMPLETE                                     AWS::Lambda::Permission                             PermissionForEventsToInvokeLambda                   -                                                 
CREATE_COMPLETE                                     AWS::CloudFormation::Stack                          eb-demo                                             -                                                 
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

CloudFormation outputs from deployed stack
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Outputs                                                                                                                                                                                                      
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Key                 OrderServiceFunctionIamRole                                                                                                                                                              
Description         Implicit IAM Role created for OrderService function                                                                                                                                      
Value               arn:aws:iam::853595176342:role/eb-demo-orderServiceFunctionRole-LFQKH98QGWLR                                                                                                             

Key                 InvoiceServiceFunction                                                                                                                                                                   
Description         InvoiceService Lambda Function ARN                                                                                                                                                       
Value               arn:aws:lambda:us-east-1:853595176342:function:eb-demo-invoiceServiceFunction-IJE26LD70776                                                                                               

Key                 InvoiceServiceFunctionIamRole                                                                                                                                                            
Description         Implicit IAM Role created for InvoiceService function                                                                                                                                    
Value               arn:aws:iam::853595176342:role/eb-demo-invoiceServiceFunctionRole-G0KPT8M9K2HS                                                                                                           

Key                 OrderServiceFunction                                                                                                                                                                     
Description         OrderService Lambda Function ARN                                                                                                                                                         
Value               arn:aws:lambda:us-east-1:853595176342:function:eb-demo-orderServiceFunction-S2W9XQ4MRP1L                                                                                                 
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Successfully created/updated stack - eb-demo in us-east-1

OpsToDevOps: $
```{% endraw %}

Alright, so our functions are deployed successfully. We have not invoked any function so far but before doing anything, let us have a look at CloudWatch logs.

![ec2 create](/assets/eb-lin-demo-2.PNG)

There are no log groups so let us get some logs in there by invoking our `order service` function which simulates an order being put through.

![ec2 create](/assets/eb-lin-demo-3.PNG)

```bash{% raw %}
OpsToDevOps: $ aws lambda invoke --function-name eb-demo-orderServiceFunction-S2W9XQ4MRP1L out --log-type Tail --output text --query LogResult | base64 -d
START RequestId: 551c721e-b0e3-4fd3-a95a-45a94a0795f2 Version: $LATEST
[INFO]  2020-09-16T00:42:51.176Z        551c721e-b0e3-4fd3-a95a-45a94a0795f2    Found credentials in environment variables.
[INFO]  2020-09-16T00:42:52.62Z 551c721e-b0e3-4fd3-a95a-45a94a0795f2    ## RESPONSE
{"FailedEntryCount": 0, "Entries": [{"EventId": "ff7993a5-245b-50ca-b09e-6b5c2caed067"}], "ResponseMetadata": {"RequestId": "e2722b3f-606f-4cc4-8e9c-387c2a40bd9e", "HTTPStatusCode": 200, "HTTPHeaders": {"x-amzn-requestid": "e2722b3f-606f-4cc4-8e9c-387c2a40bd9e", "content-type": "application/x-amz-json-1.1", "content-length": "85", "date": "Wed, 16 Sep 2020 00:42:51 GMT"}, "RetryAttempts": 0}}
END RequestId: 551c721e-b0e3-4fd3-a95a-45a94a0795f2
REPORT RequestId: 551c721e-b0e3-4fd3-a95a-45a94a0795f2  Duration: 1192.59 ms    Billed Duration: 1200 ms        Memory Size: 128 MB     Max Memory Used: 68 MB  Init Duration: 333.33 ms
OpsToDevOps: $
```{% endraw %}

Checking CloudWatch logs & we see two new log groups created.
![ec2 create](/assets/eb-lin-demo-4.PNG)

Going through the log stream of `orderServiceFunction`, we can see that it was invoked successfully.

![ec2 create](/assets/eb-lin-demo-5.PNG)

Checking log stream of `invoiceServiceFunction` & there is our order.

![ec2 create](/assets/eb-lin-demo-6.PNG)

If you were thinking how `order service` pushed an event to event bus which is gone to `invoice service` then look below.

![ec2 create](/assets/eb-lin-demo-8.PNG)

The rule that we created has a source of `demo.orders` which you can check in `template.yaml’. That rule has to match the event that was sent to it. 

The event described in `orderService/app.py` has `demo.order` which matches the source in rule.

But wait, there is more. We will now invoke the `order service` function again & check SAM logs from command line.

![ec2 create](/assets/eb-lin-demo-7.PNG)

So, in the left pane, I am simply invoking the function & in the right pane I am tailing SAM logs with the command `sam logs --name invoiceServiceFunction --stack-name eb-demo --tail`

If you have followed so far, you will see that the process is fast, reliable & scalable. 
By no means this is an in-depth coverage of the product, but it should give you a fairly good idea of what Amazon EventBridge is capable of.

I hope you enjoyed this post & learned something from it. Stay safe & I will see in the next.

Happy Eventing!
