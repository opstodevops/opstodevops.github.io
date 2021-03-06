---
layout: post
title:  "Amazon VPC More Specific Routing Demo"
date:   2021-11-10
author: prashant
featured: true
categories: [blog]
tags: [aws, vpc, routing, subnet, cdk]
excerpt_separator: <!--more-->
---

This video demonstration is based on [AWS blog](https://aws.amazon.com/blogs/aws/inspect-subnet-to-subnet-traffic-with-amazon-vpc-more-specific-routing/) covering newly released service, Amazon More Specific Routes. 

The entire demonstration is done in AWS CloudShell. If you want to follow along then spin up a fresh session of AWS CloudShell in your account & start working.

<!--more-->

```bash{% raw %}
nvm --version # most likely not installed in CloudShell
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.0/install.sh | bash
export NVM_DIR="/home/cloudshell-user/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm --version
node --version # should be installed in CloudShell
tsc --version
sudo npm install -g typescript
tsc --version
cdk --version
sudo npm install -g aws-cdk
cdk --version
sudo npm install -g aws-cdk@next
git clone https://github.com/sebsto/cdkv2-vpc-example.git
cd cdkv2-vpc-example/
npm install
cdk bootstrap #first & only time
cdk deploy
cdk destroy # to clean up the resources
```{% endraw %}

![Specific Routing](/assets/morespecific.gif){:class="img-responsive"}

If you have followed so far, you will see bump in the wire, as in the packets being captured in appliance instance which could very well be your inspection/filtering rig.

I hope you enjoyed this post & learned something about more specific routes. Stay safe & I will see in the next.

Happy Specific Routing!
