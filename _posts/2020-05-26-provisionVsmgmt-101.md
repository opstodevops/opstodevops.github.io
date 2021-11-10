---
layout: post
title:  "Configuration Provisioning vs Management??!??"
date:   2020-05-26
categories: [blog]
tags: [terraform, packer, ansible, aws]
excerpt_separator: <!--more-->
---

I am guessing that readers of this blog are aware of certain terms that are often used in any interaction around cloud.
Chit chat or a fire side chat, one word that is used & overused in all such talks is Infrastructure as Code (IaC).\
Not that IaC didn’t exist before but it wasn’t actively practiced or preached because if something is not broken then why fix it, right. 

<!--more-->

But different people have different definitions for IaC and different tooling. 
CloudFormation, Terraform, Ansible, Chef, Puppet, SaltStack, just to name a few off the top of my head. 
The list doesn’t include exclusive CI/CD tools like Jenkins, Spinnaker, Codefresh, CircleCi and a bunch more that I can’t remember right now.

For this blog, I will leave the CI/CD tooling and discuss Provisioning and Managing of Configuration and how I will change my approach of deploying 3 tier architecture.

I will not repeat what’s already there on the internet, a quick search on configuration management vs provisioning will pull up plenty of interesting articles that one can read in leisure. 
I am going to share how I plan to deploy that 3 tier architecture in AWS discussed in one of my earlier [post]({% post_url 2020-05-11-ansible-103 %})

So instead of just Ansible, I will be using Packer, Terraform and Ansible. 
Packer to build a Windows image, for Linux build I will use AWS default AMI, Terraform to deploy the infrastructure, 
VPC, security groups, Availability Zone and all that and finally Ansible to manage EC2 and DB instances.\ 
Management of the configuration item will be done by Ansible. Configuration item deployment and all the resources that we need to keep the CI running will be provisioned using Terraform. 
AMI baking with essential building blocks will be done by Packer, though AWS has [Image Builder](https://aws.amazon.com/image-builder/) as their native solution.\
This is the beginning and over time a lot of improvements will be added to this architecture.

![3tierarch_b](/assets/provVSconfig_wip.PNG)

Stay tuned!