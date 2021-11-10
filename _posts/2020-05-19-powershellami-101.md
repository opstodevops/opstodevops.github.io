---
layout: post
title:  "hey dork! why not PowerShell"
date:   2020-05-19
categories: [blog]
tags: [powershell, aws]
excerpt_separator: <!--more-->
---

This is going to be a short blog and basically a response to an email which I received from one of my three readers and YES one of them is me.\
So in my last post [Ansible 103]({% post_url 2020-05-11-ansible-103 %}), I shared a Cloudformation template where I am querying for the 
latest Windows 2019 AMI ID using AWS Systems Manager Parameter Store.
```yaml
Parameters:
  LatestAmiId:
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/ami-windows-latest/Windows_Server-2019-English-Core-Base'
```

<!--more-->

The subject of the email was "hey dork, why not powershell??" which is a good point.

The template is written in YAML & I am using template Parameters (not to be confused with PowerShell parameters) to query for Windows AMI. 
PowerShell is NOT going to work by design. But that got me thinking, can I query AMIs using PowerShell?\
YES, we can and I referred to this [article](https://aws.amazon.com/blogs/mt/query-for-the-latest-windows-ami-using-systems-manager-parameter-store/) to build base for this post. 
Let's begin, shall we.

```powershell
PS /> Import-Module -Name AWSPowerShell.NetCore
PS /> Get-STSCallerIdentity

Account      Arn                                       UserId
-------      ---                                       ------
912345678908 arn:aws:iam::912345678908:user/opstodevops AKAWSACCESSKEY6L
```

This is the first command which returned a ton on screen so I refined it further.
```powershell
PS /> Get-SSMParametersByPath -Path /aws/service/ami-windows-latest/
```

Refining the above command a bit more.
```powershell
PS /> Get-SSMParametersByPath -Path /aws/service/ami-windows-latest/ | Where-Object -Filter { $_.Name -match 'English' }
```

Taking it further and exluding everything but 2019 AMI image & the output looks good.
```powershell
PS /> Get-SSMParametersByPath -Path /aws/service/ami-windows-latest/ |
 Where-Object -Filter { $_.Name -match 'English' -and $_.Name -notmatch '2003|2008|2012|2016|sql|EKS|ECS' } |
 Sort-Object -Property Name | Format-Table -Property Name, Value

Name                                                                              Value
----                                                                              -----
/aws/service/ami-windows-latest/Windows_Server-1809-English-Core-Base             ami-08ae81d4fe513dd93
/aws/service/ami-windows-latest/Windows_Server-1809-English-Core-ContainersLatest ami-04f8dc2f722e5ab00
/aws/service/ami-windows-latest/Windows_Server-1903-English-Core-Base             ami-0ea28674bdbe222c6
/aws/service/ami-windows-latest/Windows_Server-1903-English-Core-ContainersLatest ami-0288200dad7bef6eb
/aws/service/ami-windows-latest/Windows_Server-1909-English-Core-Base             ami-0d3c08a327ffe1b56
/aws/service/ami-windows-latest/Windows_Server-1909-English-Core-ContainersLatest ami-06fba9d168216bb1e
/aws/service/ami-windows-latest/Windows_Server-2019-English-Core-Base             ami-0372aaeca7591cf5a
/aws/service/ami-windows-latest/Windows_Server-2019-English-Core-ContainersLatest ami-0523431469ff3de57
/aws/service/ami-windows-latest/Windows_Server-2019-English-Deep-Learning         ami-0f192e5424c0e0c48
/aws/service/ami-windows-latest/Windows_Server-2019-English-Full-Base             ami-04a0ee204b44cc91a
/aws/service/ami-windows-latest/Windows_Server-2019-English-Full-ContainersLatest ami-0f17565c1d79faca3
/aws/service/ami-windows-latest/Windows_Server-2019-English-Full-HyperV           ami-05a321f3199e0a903
/aws/service/ami-windows-latest/Windows_Server-2019-English-STIG-Core             ami-0064bcd2ee926c2c6
/aws/service/ami-windows-latest/Windows_Server-2019-English-STIG-Full             ami-028b51908da412d26
/aws/service/ami-windows-latest/Windows_Server-2019-English-Tesla                 ami-0274a920d66f1dc01

PS />
```

Similarly I can use PowerShell to query Linux AMIs.
```powershell
PS /> Get-SSMParametersByPath -Path /aws/service/ami-amazon-linux-latest | Format-Table -Property Name, Value
```

One thing that I thought could be improved in the above command is use of $PSItem instead of $_ simply to speed up the one liner. Here's a simple example to show what I mean.

```powershell
PS /> Measure-Command -Expression {1..100 | ForEach-Object {Write-Output $_}}

Days              : 0
Hours             : 0
Minutes           : 0
Seconds           : 0
Milliseconds      : 31
Ticks             : 318140
TotalDays         : 3.68217592592593E-07
TotalHours        : 8.83722222222222E-06
TotalMinutes      : 0.000530233333333333
TotalSeconds      : 0.031814
TotalMilliseconds : 31.814


PS /> Measure-Command -Expression {1..100 | ForEach-Object {Write-Output $PSItem}}

Days              : 0
Hours             : 0
Minutes           : 0
Seconds           : 0
Milliseconds      : 24
Ticks             : 240207
TotalDays         : 2.78017361111111E-07
TotalHours        : 6.67241666666667E-06
TotalMinutes      : 0.000400345
TotalSeconds      : 0.0240207
TotalMilliseconds : 24.0207
```

Look at the difference in Milliseconds and that is measuring a simple one liner, imaging what it can do for us. Don't be confused by the TotalDays value, that is in scientific notation.

So the final one liner will look like this and it is faster than using $_
```powershell
PS /> Get-SSMParametersByPath -Path /aws/service/ami-windows-latest/ | 
Where-Object -Filter { $PSItem.Name -match 'English' -and $PSItem.Name -notmatch '2003|2008|2012|2016|sql|EKS|ECS' } |
Sort-Object -Property Name | Format-Table -Property Name, Value
```

To be clear, just because I was able to query the images using PowerShell doesn't mean that I can use this in Parameter section of Cloudformation template.

Till next time, happy PowerShell'ing.