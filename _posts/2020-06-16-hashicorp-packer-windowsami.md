---
layout: post
title:  "Creating Windows EC2 AMI with Packer"
date:   2020-06-16
categories: [blog]
tags: [aws, hashicorp, packer]
excerpt_separator: <!--more-->
---
Windows is not going anywhere, especially in enterprise environments for authentication and other requirements. Launching an instance with Windows AMI isn't that tricky, 
I have covered it in one of my earlier articles. In this post, I will discuss how to automate building and configuring a Windows 2019 Server on AWS using [HashiCorp Packer](https://www.packer.io/).

With some PowerShell scripting and HashiCorp Packer, one can securely build and configure Windows AMIs for a particular environment. 
A fair warning before I start, Packer works beautifully with Linux but not so much with Windows.
I am still trying to iron out few things but what I am going to share should work As-Is, use it as a base template and then adjust as per requirement.

<!--more-->

Start with downloading Packer and the adding the executable to PATH environment variable.

```bash
OpsToDevOps:~/environment $ packer -v

Command 'packer' not found, but can be installed with:

sudo snap install packer  # version 1.0.0-2, or
sudo apt  install packer

See 'snap info packer' for additional versions.

OpsToDevOps:~/environment $ sudo -i
root@ip-172-31-16-25:~# cd /opt
root@ip-172-31-16-25:/opt# wget https://releases.hashicorp.com/packer/1.6.0/packer_1.6.0_linux_amd64.zip
--2020-06-16 02:18:27--  https://releases.hashicorp.com/packer/1.6.0/packer_1.6.0_linux_amd64.zip
Resolving releases.hashicorp.com (releases.hashicorp.com)... 151.101.29.183, 2a04:4e42:7::439
Connecting to releases.hashicorp.com (releases.hashicorp.com)|151.101.29.183|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 29588144 (28M) [application/zip]
Saving to: ‘packer_1.6.0_linux_amd64.zip’

packer_1.6.0_linux_amd64.zip                                  100%[==============================================================================================================================================>]  28.22M   188MB/s    in 0.2s    

2020-06-16 02:18:27 (188 MB/s) - ‘packer_1.6.0_linux_amd64.zip’ saved [29588144/29588144]

root@ip-172-31-16-25:/opt# unzip packer_1.6.0_linux_amd64.zip 
Archive:  packer_1.6.0_linux_amd64.zip
  inflating: packer                  
root@ip-172-31-16-25:/opt# logout
OpsToDevOps:~/environment $ export PATH=$PATH:'/opt'
OpsToDevOps:~/environment $ packer -v
1.6.0
OpsToDevOps:~/environment $ 
```

Packer can build machine images for a number of different cloud platforms but here I will focus on the amazon-ebs builder, which will create an EBS-backed AMI.  
At a high level, Packer performs these steps:

Read configuration settings from a json template file\
Uses the AWS API to spin up an EC2 instance\
Connect to the instance and provision it using WinRM on Port 5986\
Shut down and snapshot the instance\
Create an Amazon Machine Image (AMI) from the snapshot\
Clean up resources like security group, ec2 key-pair and ec2 instance used in the process

Take a note that amazon-ebs builder establishes a basic communicator for provisioning & for Windows it is WinRM on 5985. 
But for better security & to prevent eavesdropping during image provisioning, I will be using WinRM for HTPPS on Port 5986 by encrypting the traffic using self signed certificate.

Creating userdata file that Packer will use to bootstrap the EC2 instance. Open up a new file, copy and paste this code and save as win_boostrap.txt.

```powershell
<powershell>
# Start
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope LocalMachine -Force -ErrorAction Ignore

# Don't set this before Set-ExecutionPolicy as it throws an error
$ErrorActionPreference = "stop"
 
# Remove any existing Windows Management listeners
Remove-Item -Path WSMan:\Localhost\listener\listener* -Recurse

# Create self-signed cert for encrypted WinRM on port 5986
$Cert = New-SelfSignedCertificate -CertstoreLocation Cert:\LocalMachine\My -DnsName $ENV:COMPUTERNAME
New-Item -Path WSMan:\LocalHost\Listener -Transport HTTPS -Address * -CertificateThumbPrint $Cert.Thumbprint -Force

# Configure UAC to allow privilege elevation in remote shells
$Key = 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System'
$Setting = 'LocalAccountTokenFilterPolicy'
Set-ItemProperty -Path $Key -Name $Setting -Value 1 -Force
 
# Configure WinRM to allow encrypted communication, and provide the self-signed cert to the WinRM listener
cmd.exe /c winrm quickconfig -q
cmd.exe /c winrm set "winrm/config" '@{MaxTimeoutms="1800000"}'
cmd.exe /c winrm set "winrm/config/winrs" '@{MaxMemoryPerShellMB="1024"}'
cmd.exe /c winrm set "winrm/config/service" '@{AllowUnencrypted="false"}'
cmd.exe /c winrm set "winrm/config/client" '@{AllowUnencrypted="false"}'
cmd.exe /c winrm set "winrm/config/service/auth" '@{Basic="true"}'
cmd.exe /c winrm set "winrm/config/client/auth" '@{Basic="true"}'
cmd.exe /c winrm set "winrm/config/service/auth" '@{CredSSP="true"}'
cmd.exe /c winrm set "winrm/config/listener?Address=*+Transport=HTTPS" "@{Port=`"5986`";Hostname=`"$($ENV:COMPUTERNAME)`";CertificateThumbprint=`"$($Cert.Thumbprint)`"}"
cmd.exe /c netsh advfirewall firewall add rule profile=any name="Allow WinRM HTTPS" dir=in localport=5986 protocol=TCP action=allow

# Restart WinRM, and set it so that it auto-launches on startup
Stop-Service -Name "WinRM" -ErrorAction Stop
Set-Service -Name "WinRM" -StartupType Automatic
Start-Service -Name "WinRM" -ErrorAction Stop
</powershell>
```

NOTE--> For configuring instance with WinRM for HTTP on Port 5985, refer to steps outlined in this [link](https://learn.hashicorp.com/packer/getting-started/build-image)

Now in the same directory, open a new file by name windows_packer.json, copy and paste the this code.

```json{% raw %}
{
    "variables": {
        "aws_access_key": "{{env `AWS_ACCESS_KEY_ID`}}",
        "aws_secret_key": "{{env `AWS_SECRET_ACCESS_KEY`}}",
        "region": "us-east-1",
        "vpc_id": "",
        "subnet_id": "",
        "instance_size": "t2.micro",
        "source_ami": "",
        "security_group_id": "",
        "winrm_username": "Administrator"
    },
    "builders": [
        {
            "type": "amazon-ebs",
            "access_key": "{{ user `aws_access_key` }}",
            "secret_key": "{{ user `aws_secret_key` }}",
            "region": "{{user `region`}}",
            "vpc_id": "{{user `vpc_id`}}",
            "subnet_id": "{{user `subnet_id`}}",
            "security_group_id": "{{user `security_group_id`}}",
            "source_ami_filter": {
                "filters": {
                    "name": "Windows_Server-2019-English-Core-Base-*",
                    "root-device-type": "ebs",
                    "virtualization-type": "hvm"
                },
                "most_recent": true,
                "owners": [
                    "801119661308"
                ]
            },
            "ami_name": "WIN2019-CUSTOM-{{timestamp}}",
            "instance_type": "{{user `instance_size`}}",
            "user_data_file": "./win_bootstrap.txt",
            "associate_public_ip_address": true,
            "communicator": "winrm",
            "winrm_username": "{{user `winrm_username`}}",
            "winrm_port": 5986,
            "winrm_timeout": "5m",
            "winrm_use_ssl": true,
            "winrm_insecure": true
            
        }
    ],
    "provisioners": [
        {
            "type": "windows-restart",
            "restart_check_command": "powershell -command \"& {Write-Output 'restarted.'}\""
        }    
    ]
}
```{% endraw %}

AWS credentials are stored in my `aws_profile` & so I didn't have to specify, it will be picked from credentials file.
I am doing this in my DEV account which is configured for default access to `public internet` thus I am not specifying `vpc_id`. 
Just making sure that VPC is in the `aws_region` that I am authorized to work in & that I have traffic flowing for `subnet_id` and `security_group_id` before building the AMI. 
Most issues during Packer build are related to traffic, security groups or VPC.  Verify that the build machine running Packer can reach the VPC and EC2 instance using port 5986.

```bash
OpsToDevOps:~/environment $ packer validate windows_packer.json 
OpsToDevOps:~/environment $ time packer build windows_packer.json
amazon-ebs: output will be in this color.

==> amazon-ebs: Prevalidating any provided VPC information
==> amazon-ebs: Prevalidating AMI Name: WIN2019-CUSTOM-1592259842
    amazon-ebs: Found Image ID: ami-045391e125fb95c5e
==> amazon-ebs: Creating temporary keypair: packer_5ee7f502-8cc1-f3b8-03e7-3112512bcbe4
==> amazon-ebs: Creating temporary security group for this instance: packer_5ee7f507-b0fa-bc70-bc03-e959dea7a848
==> amazon-ebs: Authorizing access to port 5986 from [0.0.0.0/0] in the temporary security groups...
==> amazon-ebs: Launching a source AWS instance...
==> amazon-ebs: Adding tags to source instance
    amazon-ebs: Adding tag: "Name": "Packer Builder"
    amazon-ebs: Instance ID: i-0a41b374aea349e88
==> amazon-ebs: Waiting for instance (i-0a41b374aea349e88) to become ready...
==> amazon-ebs: Waiting for auto-generated password for instance...
    amazon-ebs: It is normal for this process to take up to 15 minutes,
    amazon-ebs: but it usually takes around 5. Please wait.
    amazon-ebs:  
    amazon-ebs: Password retrieved!
==> amazon-ebs: Using winrm communicator to connect: 52.87.154.151
==> amazon-ebs: Waiting for WinRM to become available...
    amazon-ebs: WinRM connected.
==> amazon-ebs: #< CLIXML
==> amazon-ebs: <Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04"><Obj S="progress" RefId="0"><TN RefId="0"><T>System.Management.Automation.PSCustomObject</T><T>System.Object</T></TN><MS><I64 N="SourceId">1</I64><PR N="Record"><AV>Preparing modules for first use.</AV><AI>0</AI><Nil /><PI>-1</PI><PC>-1</PC><T>Completed</T><SR>-1</SR><SD> </SD></PR></MS></Obj><Obj S="progress" RefId="1"><TNRef RefId="0" /><MS><I64 N="SourceId">1</I64><PR N="Record"><AV>Preparing modules for first use.</AV><AI>0</AI><Nil /><PI>-1</PI><PC>-1</PC><T>Completed</T><SR>-1</SR><SD> </SD></PR></MS></Obj></Objs>
==> amazon-ebs: Connected to WinRM!
==> amazon-ebs: Provisioning with Powershell...
==> amazon-ebs: Provisioning with powershell script: /tmp/powershell-provisioner485538378
    amazon-ebs:
    amazon-ebs: TaskPath                                       TaskName                          State
    amazon-ebs: --------                                       --------                          -----
    amazon-ebs: \                                              Amazon Ec2 Launch - Instance I... Ready
==> amazon-ebs: Stopping the source instance...
    amazon-ebs: Stopping instance
==> amazon-ebs: Waiting for the instance to stop...
==> amazon-ebs: Creating AMI WIN2019-CUSTOM-1592259842 from instance i-0a41b374aea349e88
    amazon-ebs: AMI: ami-0b7a347790dce02d2
==> amazon-ebs: Waiting for AMI to become ready...
==> amazon-ebs: Terminating the source AWS instance...
==> amazon-ebs: Cleaning up any extra volumes...
==> amazon-ebs: No volumes to clean up, skipping
==> amazon-ebs: Deleting temporary security group...
==> amazon-ebs: Deleting temporary keypair...
Build 'amazon-ebs' finished.

==> Builds finished. The artifacts of successful builds are:
--> amazon-ebs: AMIs were created:
us-east-1: ami-0b7a347790dce02d2


real	4m35.474s
user	0m1.230s
sys	0m2.099s
OpsToDevOps:~/environment $
```

There, I have a custom Windows 2019 AMI built in less than 5 minutes, the time depends on level of customization but the initial process is straight forward.
If I want to increase the verbosity for troubleshooting then I can use `PACKER_LOG=1 packer build windows_packer.json` & that will output ton of data on screen.\
Next step, I am trying to advance this template and make it one stop shop for AMI customization, for example get few AWS packages installed (Cloudwatch agent, SSM agent).

Stay Safe!