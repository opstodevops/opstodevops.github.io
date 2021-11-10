---
layout: post
title:  "Event-driven architecture using AWS EventBridge Part 1"
date:   2020-08-17
categories: [blog]
tags: [aws, event-driven architecture, event driven, serverless, eventbridge]
excerpt_separator: <!--more-->
---
Event-driven architecture, yes, that's what I will be focusing on in my next few posts. I have been fascinated with the concept & I am going to trace my steps back to eventing by writing a primer. 
This series of posts will mature over period of time & work as a stepping stone into serverless, so stick around.

My tech stack of choice happens to be AWS & so I will be discussing eventing features/services available with AWS. 
I will be making few assumptions along the way like familiarity with AWS as cloud provider, idea of serverless (think of WiFi router at your office, you use the service Wi-Fi but don't have to manage the router) and bit about decoupled/distributed applications.

Monolithic applications have worked so far & not every application needs to be broken down. 
Small applications which are not complex don't have lots of moving parts or are categorised as legacy don't require scrutiny. 
But looking around, almost every other application which is getting regular updates & is actively maintained has tons of independent components.

<!--more-->

Think of a food ordering website, let's go with pizza ordering site (hungry yet???), that website has modernized fast to handle the volume of orders. 
Order run-of-the-mill pizza or customize it the way you like with toppings, sides and extras. Don't like what you have in your cart, cancel that start again. 
Payment processing and a tracker on website so you know what's happening to your pizza.\ 
If I have to guess, I'd say many people can customize order and pay for their pizza at the same time & can track their orders too. 
If tracker is broken, it doesn't mean that one can't order a pizza, it simply means that there's no visibility to pizza processing but it is still being made. 
Tracker is just one part of the website, ordering a pizza & paying for it is another. All these parts work together but are separate with mostly no interdependence.

Keep that analogy in your head while you read the next few lines. 
An event-driven architecture uses events to trigger and communicate between decoupled services and is common in modern applications built with microservices (application as a collection of loosely coupled services). 
An event is a change in state, or an update, like a pizza being ordered on a website or grocery items put in a shopping cart of supermarket. 

Events can either carry the state (your customized pizza, its price, and a delivery address) or events can be identifiers (a notification that your pizza has left the store).

The architecture below is fairly simple but shows that event-driven architectures has three key components: event producers, event routers, and event consumers. 
A producer publishes an event to the event router, which filters and pushes the events to event consumers.

![evtbridge_0](/assets/evtbridge_0.PNG)

AWS has a service which enables serverless builders to decouple the architecture using routing rules to deliver events to selected targets. 
It is called Amazon EventBridge & if I have to summarize it in one line then I'd say "CloudWatch on roids". 
Amazon EventBridge is a serverless event bus service that helps connect external SaaS providers with your own applications and AWS services. 
You can read more about [EventBridge](https://aws.amazon.com/eventbridge/)

As a first step in event-driven architecture, I am going to demonstrate a straightforward example of an event produced by invoking lambda, routed by Amazon EventBridge & consumed by CloudWatch. 
I am using `Cloud 9` as my IDE but you can install aws-cli on Windows or Mac by following these [instructions](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html).

Every AWS will have a default Event Bus but here I am creating a new one.

```bash{% raw %}
OpsToDevOps $ aws events create-event-bus --name manual-order-bus
{
    "EventBusArn": "arn:aws:events:us-east-1:803943555226:event-bus/manual-order-bus"
}
OpsToDevOps $ aws events list-event-buses
{
    "EventBuses": [
        {
            "Name": "default",
            "Arn": "arn:aws:events:us-east-1:803943555226:event-bus/default"
        },
        {
            "Name": "manual-order",
            "Arn": "arn:aws:events:us-east-1:803943555226:event-bus/manual-order-bus"
        }
    ]
}
```{% endraw %}

For routing incoming events, the Event Bus needs rules. Without the rules, Event Bus has no way to understand what to do with the events.
Here I am creating a very simple `catch all` event rule.

```bash{% raw %}  
OpsToDevOps $ aws events put-rule --name "manual-order-rule" --event-pattern "{\"account\":[\"803943555226\"]}" --state ENABLED --event-bus-name "manual-order-bus"           
{
    "RuleArn": "arn:aws:events:us-east-1:803943555226:rule/manual-order-bus/manual-order-rule"
}
OpsToDevOps $ aws events list-rules --event-bus-name manual-order
{
    "Rules": [
        {
            "Name": "manual-order-rule",
            "Arn": "arn:aws:events:us-east-1:803943555226:rule/manual-order-bus/manual-order-rule",
            "EventPattern": "{\"account\":[\"803943555226\"]}",
            "State": "ENABLED",
            "EventBusName": "manual-order-bus"
        }
    ]
}
```{% endraw %}

The rule is created but it needs a target, where to route the event, what is the consumer consuming the event. 
For simplicity, I will do that from the console. Here I am selecting the target by creating a new CloudWatch log group.

![evtbridge_1](/assets/evtbridge_1.PNG)

#### NOTE ABOUT IAM ROLE ####

I have an IAM role that I will using to invoke my lambda function & that role has a policy attached to it giving the role full access to Event Bridge. 
This IAM role is only for demonstration purposes, obviously the policies should be filtered in production environments.

```bash{% raw %}
OpsToDevOps $ aws iam list-attached-role-policies --role-name myManualOrder-role-hcqiducd                                                                                 
{
    "AttachedPolicies": [
        {
            "PolicyName": "AWSLambdaBasicExecutionRole-6af4c077-cafc-4890-884c-974e782ff00a",
            "PolicyArn": "arn:aws:iam::803943555226:policy/service-role/AWSLambdaBasicExecutionRole-6af4c077-cafc-4890-884c-974e782ff00a"
        },
        {
            "PolicyName": "AmazonEventBridgeFullAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AmazonEventBridgeFullAccess"
        }
    ]
}
```{% endraw %}

This is how the role is looking in console. You can see the Event Bridge full access policy attached to the role.

![evtbridge_2](/assets/evtbridge_2.PNG)

Now comes the interesting part of deploying & invoking lambda function. Here is a simple Python function that uses Boto library to `PUT` an event in CloudWatch. 
The `EventBusName` is the name of the bus that was created earlier. 

#### MAKE SURE TO PUT IN THE NAME OF YOUR EVENT BUS IF YOU HAVE CREATED ONE WITH A DIFFERENT NAME. ####

```python{% raw %}
import json
import boto3


def lambda_handler(event, context):
    
    order_details = {
        "orderNumber": "123",
        "productId": "ISN123",
        "price": 100,
        "customer": {
            "name": "Jane Doe",
            "customerId": "1",
            "address": "galaxy far far away"
        }
    }
    
    event_bridge = boto3.client('events')
    
    response = event_bridge.put_events(
        Entries=[
            {
                'Source': 'Order Service',
                'DetailType': 'New Order',
                'Detail': json.dumps(order_details),
                'EventBusName': 'manual-order-bus'
            },
        ]
    )
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }
```{% endraw %}

Alright, here I am zipping the Python code and deploying it using that `IAM role` I showed before.

```bash{% raw %}
OpsToDevOps $ zip -r manual_order.zip lambda_function.py                                                                                                                  
  adding: lambda_function.py (deflated 53%)
OpsToDevOps $ aws lambda create-function --function-name "manual-order-function" --runtime "python3.6" --role arn:aws:iam::803943555226:role/service-role/myManualOrder-role-hcqiducd --handler "lambda_function.lambda_handler" --zip-file fileb://manual_order.zip                                                                                             
{
    "FunctionName": "manual-order-function",
    "FunctionArn": "arn:aws:lambda:us-east-1:803943555226:function:manual-order-function",
    "Runtime": "python3.6",
    "Role": "arn:aws:iam::803943555226:role/service-role/myManualOrder-role-hcqiducd",
    "Handler": "lambda_function.lambda_handler",
    "CodeSize": 546,
    "Description": "",
    "Timeout": 3,
    "MemorySize": 128,
    "LastModified": "2020-08-16T08:57:54.443+0000",
    "CodeSha256": "2H/v+KY2YysbRrDLh7IXJae3qQDGvyQj1JBcf4U+0H4=",
    "Version": "$LATEST",
    "TracingConfig": {
        "Mode": "PassThrough"
    },
    "RevisionId": "45673371-0e8e-4d3b-988f-5658810d6f86",
    "State": "Active",
    "LastUpdateStatus": "Successful"
}
OpsToDevOps $ aws lambda list-functions
{
    "Functions": [
        {
            "FunctionName": "manual-order-function",
            "FunctionArn": "arn:aws:lambda:us-east-1:803943555226:function:manual-order-function",
            "Runtime": "python3.6",
            "Role": "arn:aws:iam::803943555226:role/service-role/myManualOrder-role-hcqiducd",
            "Handler": "lambda_function.lambda_handler",
            "CodeSize": 546,
            "Description": "",
            "Timeout": 3,
            "MemorySize": 128,
            "LastModified": "2020-08-16T08:57:54.443+0000",
            "CodeSha256": "2H/v+KY2YysbRrDLh7IXJae3qQDGvyQj1JBcf4U+0H4=",
            "Version": "$LATEST",
            "TracingConfig": {
                "Mode": "PassThrough"
            },
            "RevisionId": "45673371-0e8e-4d3b-988f-5658810d6f86"
        }
    ]
}
```{% endraw %}

Here is the part to see all of our work coming together. The `CloudWatch` log group has no logs in it but once I invoke the lambda function, I have a log in there.

```bash{% raw %}
OpsToDevOps $ aws logs describe-log-streams --log-group-name /aws/events/manual-order-cw
{
    "logStreams": []
}
OpsToDevOps $ aws lambda invoke --function-name manual-order-function out --log-type Tail --query LogResult --output text | base64 -d
START RequestId: 7f175dc7-2157-4e26-995c-a1bfd262e198 Version: $LATEST
END RequestId: 7f175dc7-2157-4e26-995c-a1bfd262e198
REPORT RequestId: 7f175dc7-2157-4e26-995c-a1bfd262e198  Duration: 1056.12 ms    Billed Duration: 1100 ms        Memory Size: 128 MB     Max Memory Used: 67 MB  Init Duration: 190.34 ms
OpsToDevOps $ aws logs describe-log-streams --log-group-name /aws/events/manual-order-cw
{
    "logStreams": [
        {
            "logStreamName": "902b3e54-bfe0-3b64-8933-318bcded6ab1",
            "creationTime": 1597568952243,
            "firstEventTimestamp": 1597568951000,
            "lastEventTimestamp": 1597568951000,
            "lastIngestionTime": 1597568952258,
            "uploadSequenceToken": "49605671008691322765777015740483179716136754318630568082",
            "arn": "arn:aws:logs:us-east-1:803943555226:log-group:/aws/events/manual-order-cw:log-stream:902b3e54-bfe0-3b64-8933-318bcded6ab1",
            "storedBytes": 0
        }
    ]
}
```{% endraw %}

If I check the log from console, the log is the event that we put in by invoking lambda. EventBridge successfully routed our manual event & delivered to CloudWatch logs.

![evtbridge_3](/assets/evtbridge_3.PNG)

Hopefully this blog helped you understand a bit about eventing and EventBridge. More to come in next post so stay tuned & stay safe!
