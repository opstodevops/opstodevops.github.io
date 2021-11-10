---
layout: post
title:  "PowerShell Docker AWS"
date:   2020-05-15
categories: [blog]
tags: [powershell, aws, docker, cloudformation]
excerpt_separator: <!--more-->
---

This post is going to be about PowerShell & Docker & Windows & whole lot of other things.\
I passed my Azure fundamentals (AZ-900) exam early this morning. Coming from AWS background helped in preparation, though I will be honest I didn’t put in a lot of effort. 
The exam is aimed at Azure beginners & I have worked with Azure couple of years back. 
I attended the [Azure training day]({% post_url 2020-05-05-azure-virtual-day %}), got my exam voucher, attempted the exam & passed.

<!--more-->

The one post that helped me map AWS to Azure & one that I’d recommend anyone with experience in AWS trying to dabble in Azure is [AWS to Azure comparison]( https://docs.microsoft.com/en-us/azure/architecture/aws-professional/services)

![az-900](/assets/az-900.png)

Celebration out of the way, today’s post is about PowerShell running in Docker with AWS tools to deploy a Web Server in AWS. Someone who read my post emailed me asking for some PowerShell content & so I decided to throw in some Docker in there with CloudFormation template that would deploy an EC2 instance running Windows Server with IIS installed. 
Note that you will also need to specify a different key pair name, as the “pwsh-cfn” key pair name is unique to my AWS account.

```yaml
FROM mcr.microsoft.com/powershell
LABEL Description="PowerShell Container" Vendor="Microsoft" Version="1.0"
RUN pwsh -Command Install-Module -Name AWSPowerShell.NetCore -Force
CMD [ "pwsh" ]
```
Spinning the PowerShell Docker container which now has AWS tool for PowerShell installed.

```bash
docker container run --rm -it --name pwsh01 --env AWS_ACCESS_KEY_ID=awsaccesskeyid" --env "AWS_SECRET_ACCESS_KEY=awsSecretAccessKey" --env "AWS_REGION=us-east-1" pwsh:docker

PS /pwsh> Get-STSCallerIdentity

Account      Arn                                       UserId
-------      ---                                       ------
912345678908 arn:aws:iam::912345678908:user/opstodevops AKAWSACCESSKEY6L

PS /pwsh> ./RUN-SampleWindowsEC2.ps1
arn:aws:cloudformation:us-east-1:912345678908:stack/SampleWindowsStack/4b2d3ec0-9683-11ea-917c-0ea6b7109c22
```

The script that I called above simply imports the module for AWS tools for PowerShell, configures the parameter value & run the Cloudformation template written in YAML.
Here is the content of `RUN-SampleWindowsEC2.ps1`

```powershell
if (-not (Get-Module -Name AWSPowerShell.NetCore)) {
    Import-Module -Name AWSPowerShell.NetCore
}

# First parameter
$KeyPairName = New-Object Amazon.CloudFormation.Model.Parameter
$KeyPairName.ParameterKey = "KeyPairName"
$KeyPairName.ParameterValue = "pwsh-cfn"

$WindowsStackTemplate = Get-Content -Path ./sample_Windows_ec2.yml -Raw

New-CFNStack -StackName SampleWindowsStack -TemplateBody $WindowsStackTemplate -Parameters $KeyPairName
```

Here is the [Cloudformation stack](https://raw.githubusercontent.com/opstodevops/pwsh-docker/master/sample_Windows_ec2.yml)

Connecting to the Windows Server Core instance and checking if IIS was installed. Yes, it did.

![check-webserver](/assets/check-webserver01.PNG)

Keep your game hat ON!