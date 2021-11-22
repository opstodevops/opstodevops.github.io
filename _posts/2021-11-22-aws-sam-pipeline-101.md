---
layout: post
title:  "Amazon SAM Pipeline with Github Actions"
date:   2021-11-22
author: prashant
featured: true
categories: [blog]
tags: [aws, SAM, Github, pipeline, automation]
excerpt_separator: <!--more-->
---

Few months ago, AWS announced a new capability of [AWS Serverless Application Model](https://aws.amazon.com/serverless/sam/) called SAM Pipelines. I haven't had the chance to 
play with it so far but that's going to change today. I like pipelines for obious reasons - mostly one time setup, accelerated release cycles, enhanced reliability, secure, 
compliant, performant & whole lot of other benefits. 
There are so many CI/CD systems available in the market, namely AWS Code Pipeline, Jenkins, Team City, Github actions, Circle CI, Spinnaker etc. Each comes with own set of features &
complexity. What if I told you that AWS SAM Pipelines takes all the pain away & makes it easy to crate a secure continuous integration and continuous deployment (CI/CD) pipelines 
for your preferred CI/CD system. Now at the time of writing, not all CI/CD systems are in the supported list but few of the popular ones are there for you to start working with. 
This blog post shows how to use AWS SAM Pipelines to create a CI/CD deployment pipeline that integrates with Github Actions.

<!--more-->

If you want to follow along then spin up a fresh Cloud 9 environment or start a session in your favorite IDE session.

<B>PREREQUISITES</B>


✅  An AWS account with permissions to create the necessary resources. \
✅  Install AWS Command Line Interface (CLI) and AWS SAM CLI (shown below 👇).  \
✅  A verified Github account: This post assumes you have the required permissions to configure Github projects, create workflows, and configure Github variables. \
✅  Create a new Github project and clone it to your local environment 

![sam cli download](/assets/sam-cli-dload.png)

```bash{% raw %}
$ wget https://github.com/aws/aws-sam-cli/releases/latest/download/aws-sam-cli-linux-x86_64.zip
$ unzip aws-sam-cli-linux-x86_64.zip -d sam-installation
$ sudo ./sam-installation/install
$ sam --version

# IF UPGRADING SAM
$ sudo ./sam-installation/install --update
$ sam --version
```{% endraw %}

📝  `THE AWS ACCESS & SECRET KEYS IN HERE ARE ONLY FOR DEMONSTRATION PURPOSES.`

```bash{% raw %}
$ git clone https://github.com/GITHUBACCOUNT/GITHUBREPO.git
$ export AWS_ACCESS_KEY_ID=AKIK6TVV757ML6AV2DBD7
$ export AWS_SECRET_ACCESS_KEY=WAE5Eo6fPJut1XQjXxzBemXlWLyTBEhOCmDkqkG8
$ sam init --runtime go1.x --app-template hello-world --name sam-app && sam pipeline init --bootstrap
What package type would you like to use?
        1 - Zip (artifact is a zip uploaded to S3)
        2 - Image (artifact is an image uploaded to an ECR image repository)
Package type: 1

Cloning from https://github.com/aws/aws-sam-cli-app-templates

    -----------------------
    Generating application:
    -----------------------
    Name: sam-app
    Runtime: go1.x
    Architectures: x86_64
    Dependency Manager: mod
    Application Template: hello-world
    Output Directory: .

    Next application steps can be found in the README file at ./sam-app/README.md
    

    Commands you can use next
    =========================
    [*] Create pipeline: cd sam-app && sam pipeline init --bootstrap
    

sam pipeline init generates a pipeline configuration file that your CI/CD system
can use to deploy serverless applications using AWS SAM.
We will guide you through the process to bootstrap resources for each stage,
then walk through the details necessary for creating the pipeline config file.

Please ensure you are in the root folder of your SAM application before you begin.

Select a pipeline template to get started:
        1 - AWS Quick Start Pipeline Templates
        2 - Custom Pipeline Template Location
Choice: 1

Cloning from https://github.com/aws/aws-sam-cli-pipeline-init-templates.git
Select CI/CD system
        1 - Jenkins
        2 - GitLab CI/CD
        3 - GitHub Actions
        4 - Bitbucket Pipelines
        5 - AWS CodePipeline
Choice: 3
You are using the 2-stage pipeline template.
 _________    _________ 
|         |  |         |
| Stage 1 |->| Stage 2 |
|_________|  |_________|

Checking for existing stages...

[!] None detected in this account.

Do you want to go through stage setup process now? If you choose no, you can still reference other bootstrapped resources. [y/N]: y

For each stage, we will ask for [1] stage definition, [2] account details, and [3]
reference application build resources in order to bootstrap these pipeline
resources.

We recommend using an individual AWS account profiles for each stage in your
pipeline. You can set these profiles up using aws configure or ~/.aws/credentials. See
[https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-getting-started-set-up-credentials.html].


Stage 1 Setup

[1] Stage definition
Enter a configuration name for this stage. This will be referenced later when you use the sam pipeline init command:
Stage configuration name: dev

[2] Account details
The following AWS credential sources are available to use.
To know more about configuration AWS credentials, visit the link below:
https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html                
        1 - Environment variables
        2 - default (named profile)
        q - Quit and configure AWS credentials
Select a credential source to associate with this stage: 1
Associated account 399635643159 with configuration dev.

Enter the region in which you want these resources to be created [us-east-1]: us-west-2
Enter the pipeline IAM user ARN if you have previously created one, or we will create one for you []: 

[3] Reference application build resources
Enter the pipeline execution role ARN if you have previously created one, or we will create one for you []: 
Enter the CloudFormation execution role ARN if you have previously created one, or we will create one for you []: 
Please enter the artifact bucket ARN for your Lambda function. If you do not have a bucket, we will create one for you []: 
Does your application contain any IMAGE type Lambda functions? [y/N]: N

[4] Summary
Below is the summary of the answers:
        1 - Account: 399635643159
        2 - Stage configuration name: dev
        3 - Region: us-west-2
        4 - Pipeline user: [to be created]
        5 - Pipeline execution role: [to be created]
        6 - CloudFormation execution role: [to be created]
        7 - Artifacts bucket: [to be created]
        8 - ECR image repository: [skipped]
Press enter to confirm the values above, or select an item to edit the value: 

This will create the following required resources for the 'dev' configuration: 
        - Pipeline IAM user
        - Pipeline execution role
        - CloudFormation execution role
        - Artifact bucket
Should we proceed with the creation? [y/N]: y
        Creating the required resources...
        Successfully created!
The following resources were created in your account:
        - Pipeline IAM user
        - Pipeline execution role
        - CloudFormation execution role
        - Artifact bucket
Pipeline IAM user credential:
        AWS_ACCESS_KEY_ID: AAIKV2DBD7MDHHILQRE5
        AWS_SECRET_ACCESS_KEY: M6sOFZQup7Ij5+m67LUR+lLNKAoycFqZM+2iL0SI
View the definition in .aws-sam/pipeline/pipelineconfig.toml,
run sam pipeline bootstrap to generate another set of resources, or proceed to
sam pipeline init to create your pipeline configuration file.

Before running sam pipeline init, we recommend first setting up AWS credentials
in your CI/CD account. Read more about how to do so with your provider in
https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-generating-example-ci-cd-others.html.

Checking for existing stages...

Only 1 stage(s) were detected, fewer than what the template requires: 2.

Do you want to go through stage setup process now? If you choose no, you can still reference other bootstrapped resources. [y/N]: y

For each stage, we will ask for [1] stage definition, [2] account details, and [3]
reference application build resources in order to bootstrap these pipeline
resources.

We recommend using an individual AWS account profiles for each stage in your
pipeline. You can set these profiles up using aws configure or ~/.aws/credentials. See
[https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-getting-started-set-up-credentials.html].


Stage 2 Setup

[1] Stage definition
Enter a configuration name for this stage. This will be referenced later when you use the sam pipeline init command:
Stage configuration name: prod

[2] Account details
The following AWS credential sources are available to use.
To know more about configuration AWS credentials, visit the link below:
https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html                
        1 - Environment variables
        2 - default (named profile)
        q - Quit and configure AWS credentials
Select a credential source to associate with this stage: 1
Associated account 399635643159 with configuration prod.

Enter the region in which you want these resources to be created [us-east-1]: 
Pipeline IAM user ARN: arn:aws:iam::399635643159:user/aws-sam-cli-managed-dev-pipeline-reso-PipelineUser-17BVVAM1963CU

[3] Reference application build resources
Enter the pipeline execution role ARN if you have previously created one, or we will create one for you []: 
Enter the CloudFormation execution role ARN if you have previously created one, or we will create one for you []: 
Please enter the artifact bucket ARN for your Lambda function. If you do not have a bucket, we will create one for you []: 
Does your application contain any IMAGE type Lambda functions? [y/N]: N

[4] Summary
Below is the summary of the answers:
        1 - Account: 399635643159
        2 - Stage configuration name: prod
        3 - Region: us-east-1
        4 - Pipeline user ARN: arn:aws:iam::399635643159:user/aws-sam-cli-managed-dev-pipeline-reso-PipelineUser-17BVVAM1963CU
        5 - Pipeline execution role: [to be created]
        6 - CloudFormation execution role: [to be created]
        7 - Artifacts bucket: [to be created]
        8 - ECR image repository: [skipped]
Press enter to confirm the values above, or select an item to edit the value: 

This will create the following required resources for the 'prod' configuration: 
        - Pipeline execution role
        - CloudFormation execution role
        - Artifact bucket
Should we proceed with the creation? [y/N]: y
        Creating the required resources...
        Successfully created!
The following resources were created in your account:
        - Pipeline execution role
        - CloudFormation execution role
        - Artifact bucket
View the definition in .aws-sam/pipeline/pipelineconfig.toml,
run sam pipeline bootstrap to generate another set of resources, or proceed to
sam pipeline init to create your pipeline configuration file.

Checking for existing stages...


This template configures a pipeline that deploys a serverless application to a testing and a production stage.

What is the GitHub secret name for pipeline user account access key ID? [AWS_ACCESS_KEY_ID]: 
What is the GitHub Secret name for pipeline user account access key secret? [AWS_SECRET_ACCESS_KEY]: 
What is the git branch used for production deployments? [main]: 
What is the template file path? [template.yaml]: sam-app/template.yaml 
We use the stage configuration name to automatically retrieve the bootstrapped resources created when you ran `sam pipeline bootstrap`.

Here are the stage configuration names detected in .aws-sam/pipeline/pipelineconfig.toml:
        1 - dev
        2 - prod
Select an index or enter the stage 1's configuration name (as provided during the bootstrapping): dev
What is the sam application stack name for stage 1? [sam-app]: sam-app-dev
Stage 1 configured successfully, configuring stage 2.

Here are the stage configuration names detected in .aws-sam/pipeline/pipelineconfig.toml:
        1 - dev
        2 - prod
Select an index or enter the stage 2's configuration name (as provided during the bootstrapping): prod
What is the sam application stack name for stage 2? [sam-app]: sam-app-prod 
Stage 2 configured successfully.

SUMMARY
We will generate a pipeline config file based on the following information:
        What is the GitHub secret name for pipeline user account access key ID?: AWS_ACCESS_KEY_ID
        What is the GitHub Secret name for pipeline user account access key secret?: AWS_SECRET_ACCESS_KEY
        What is the git branch used for production deployments?: main
        What is the template file path?: sam-app/template.yaml
        Select an index or enter the stage 1's configuration name (as provided during the bootstrapping): dev
        What is the sam application stack name for stage 1?: sam-app-dev
        What is the pipeline execution role ARN for stage 1?: arn:aws:iam::399635643159:role/aws-sam-cli-managed-dev-pipe-PipelineExecutionRole-1LV6QTCJB978X
        What is the CloudFormation execution role ARN for stage 1?: arn:aws:iam::399635643159:role/aws-sam-cli-managed-dev-p-CloudFormationExecutionR-10BO3OVCCC5Q
        What is the S3 bucket name for artifacts for stage 1?: aws-sam-cli-managed-dev-pipeline-artifactsbucket-a38wt94ma2ha
        What is the ECR repository URI for stage 1?: 
        What is the AWS region for stage 1?: us-west-2
        Select an index or enter the stage 2's configuration name (as provided during the bootstrapping): prod
        What is the sam application stack name for stage 2?: sam-app-prod
        What is the pipeline execution role ARN for stage 2?: arn:aws:iam::399635643159:role/aws-sam-cli-managed-prod-pip-PipelineExecutionRole-KV2T4HDCJLFX
        What is the CloudFormation execution role ARN for stage 2?: arn:aws:iam::399635643159:role/aws-sam-cli-managed-prod-CloudFormationExecutionR-PN5P4852263R
        What is the S3 bucket name for artifacts for stage 2?: aws-sam-cli-managed-prod-pipeline-artifactsbucket-ed5pyo38bpbp
        What is the ECR repository URI for stage 2?: 
        What is the AWS region for stage 2?: us-east-1
Successfully created the pipeline configuration file(s):
        - .github/workflows/pipeline.yaml
```{% endraw %}

The important section to take note of before proceeding is Pipeline IAM user credential. Shown below is an example of what you need to make note of & configure 
github `repository>Settings>Secrets`.

![pipeline iam](/assets/pipeline-iam.png)

![github secrets](/assets/github-secrets.png)

What is left now is to commit the changes to your Github repository.

![git push sam-app](/assets/git-sam-cli.png)

Once the files are commited, you can click on Actions to see the pipeline in action.

![github_action_samcli](/assets/sam-github.gif){:class="img-responsive"}

You can view the definition file 👉 `.aws-sam/pipeline/pipelineconfig.toml`

The steps showed how to use AWS SAM Pipelines to create a deployment pipeline for Github Actions. For each commit to the Main branch, the pipeline builds, tests, and deploys to production account.

I hope you enjoyed this post & learned the basics of AWS SAM Pipelines. 

Stay safe & I will see you in the next!
