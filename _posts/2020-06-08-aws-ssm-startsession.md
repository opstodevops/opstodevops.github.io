---
layout: post
title:  "AWS SSM Manage Windows From Terminal"
date:   2020-06-08
categories: [blog]
tags: [aws, terraform, ssm]
excerpt_separator: <!--more-->
---

On a new pet project where I am working with both Linux & Windows instances, I came across a solution which I’d like to share with everyone.
I was working on my Mac terminal & connecting to a Linux EC2 instance is straight forward but connecting to a Windows instance is different situation.\
Either I use SSM manager from the AWS console or I get a RDP client installed and then remote into Windows instance or enable PowerShell web access or enable PowerShell remoting and start working on configuring a connection.

I was looking for a clean solution that I could later on automate & the answer came from AWS [Session Manager Plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)

<!--more-->


Here's a simple visualization from architecture point of view.

![sss-start-session_1](/assets/ssm-start-session_1.PNG)

Using SSM plugin, I could start a session right from my Mac to a Windows instance.
To use the AWS CLI to run session commands, the Session Manager plugin must also be installed on your local machine. So first step is to install the plugin on Mac.
Here’s a quick bash script that I wrote which will download, unzip and install plugin on Mac.

```bash
opstodevops$ vi ssmplugin-macos.sh

#!/bin/bash

curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/mac/sessionmanager-bundle.zip" -o "sessionmanager-bundle.zip"

unzip sessionmanager-bundle.zip

sudo ./sessionmanager-bundle/install -i /usr/local/sessionmanagerplugin -b /usr/local/bin/session-manager-plugin

opstodevops$ chmod u+x ssmplugin-macos.sh

opstodevops$ ./ssmplugin-macos.sh
```
By default, AWS Systems Manager doesn't have permission to perform actions on the Windows instances. I have to grant access by using an IAM instance profile. An instance profile is a container that passes IAM role information to an EC2 instance at launch. So next step is to create an instance profile.

```go{% raw %}
resource "aws_iam_role" "ssm_role" {
  name = "ssm_role"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Effect": "Allow",
      "Sid": ""
    }
  ]
}
EOF

  tags = {
      Project = "pet-project"
  }
}

resource "aws_iam_instance_profile" "ssm_profile" {
  name = "ssm_profile"
  role = aws_iam_role.ssm_role.name
}

data "aws_iam_policy" "ssm_instancecore" {
  arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_role_policy_attachment" "ssm_attach_policy" {
  role       = aws_iam_role.ssm_role.name
  policy_arn = data.aws_iam_policy.ssm_instancecore.arn
}
```{% endraw %}

The Last step is to launch a Windows EC2 instance using the instance profile cerated earlier and in the process installing SSM plugin. Here, I am using Terraform to launch a Windows EC2 instance with an instance profile. 

```go{% raw %} 
provider "aws" {
  region     = "us-east-1"
  access_key = "YOUR ACCESS KEY"
  secret_key = "YOUR SECRET ACCESS KEY"
}

resource "aws_instance" "ssm-test" {
  count = 3 # Create 3 similar EC2 instances
  ami = "ami-04a0ee204b44cc91a"
  instance_type = "t2.medium"
  iam_instance_profile = aws_iam_instance_profile.ssm_profile.name
  key_name = "ec2keypair" #YOUR KEY PAIR
  user_data = <<EOF
<powershell>
  Invoke-WebRequest `
  https://s3.amazonaws.com/session-manager-downloads/plugin/latest/windows/SessionManagerPluginSetup.exe `
  -Out $env:USERPROFILE\Desktop\SSMPluginSetup.exe
  
  Start-Process `
  -FilePath $env:USERPROFILE\Desktop\SSMPluginSetup.exe `
  -ArgumentList "/S" -Wait

  Remove-Item -Force $env:USERPROFILE\Desktop\SSMPluginSetup.exe

</powershell>
EOF
}
```{% endraw %}

I can just use the PowerShell block as user data when launching Windows EC2 instance from AWS console to download & install SSM plugin.

```powershell
<powershell>
  Invoke-WebRequest `
  https://s3.amazonaws.com/session-manager-downloads/plugin/latest/windows/SessionManagerPluginSetup.exe `
  -Out $env:USERPROFILE\Desktop\SSMPluginSetup.exe
  
  Start-Process `
  -FilePath $env:USERPROFILE\Desktop\SSMPluginSetup.exe `
  -ArgumentList "/S" -Wait

  Remove-Item -Force $env:USERPROFILE\Desktop\SSMPluginSetup.exe

</powershell>
```

Take note that I am not defining port 3389 for RDP access and that my `AWSCLI` is configured with a user profile that has full access to Systems Manager.
Once I had a running Windows instance, all I need was instance id & then from Mac terminal, I execute `aws ssm start-session --target <instance-id>`

![sss-start-session_1a](/assets/ssm-start-session_1a.PNG)

There I have it, a PowerShell session to the Windows EC2 instance from Mac terminal. The process is automated so I can spin up 1 or 10 or 100 instances & never have to RDP to any of them.
I can also do port forwarding & get a RDP session from my browser window by doing something like 
{% raw %}`aws ssm start-session --target <instance-id> --document-name AWS-StartPortForwardingSession --parameters '{"portNumber":[3389],"localPortNumber":["9999"]}`{% endraw %}
but my requirement of gaining access to a Windows instance like SSH is met.\
This setup can be further tweaked for security and locked down permitted user actions, further reading available [here](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started.html)

In the next post, Packer template for Windows AMI.

Stay Safe!