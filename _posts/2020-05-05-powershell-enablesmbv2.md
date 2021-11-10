---
layout: post
title:  "PWSH Enable-SMBv2"
date:   2020-05-05
categories: [blog]
tags: [powershell]
excerpt_separator: <!--more-->
---

I was working with someone having tons of servers with SMBv1 enabled, somehow the Group Policy to enable SMBv2 didn't take effect on big portion of 2008/2008R2 fleet. 
The investigation began to identify why the policy isn't taking effect but in the meantime, I decided to come up with a fix which will enable SMBv2 for the time and later on enforced by Group Policy.
I wrote a simple PowerShell script to get the job done.
<!--more-->

``` powershell
# Function will check for SMB value in registry & create SMBv2 key if missing
function Enable-SMBv2 {
$registryPath = "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters"
$Name = "SMB2"
$Value = "1"
$SMBKey = Get-Item -Path $registryPath
    if($SMBKey.GetValue("SMB2") -eq $null) {
        New-ItemProperty -Path $registryPath -Name $Name -Value $Value -PropertyType DWORD -Force | Out-Null
    } else {
        Set-ItemProperty -Path $registryPath -Name $Name -Value -Force | Out-Null
    }
}

# This part will run the Enable-SMBv2 function using PowerShell background jobs & return failure
foreach ($server in @('server01', 'server02', 'server03')) { # Shown as example, there are plenty of other ways to pull in servers
    Invoke-Command -ComputerName $server -ScriptBlock ${Function:Enable-SMBv2} -AsJob | Out-Null
}

Get-Job | Wait-Job | Out-Null

[psobjectect]$failedjobs = @()
[psobjectect]$SMBstatus = @()

IF ((Get-Job -State Failed).count -ne 0) {
    foreach ($failure in (Get-Job -State Failed)) {
        $failedjobs += New-Object psobject -Property @{
            'Server' = $failure.Location
            'Failure' = ($Failure.ChildJobs.JobStateInfo.Reason.Transportmessage)
            }
        }
    }

$SMBstatus += foreach ($Job in Get-Job) {
    Receive-Job $Job -ErrorAction SilentlyContinue
    Remove-Job $Job
}

$SMBstatus

if ($failedjobs) {
    Write-Output `n'Check the failures below...'`n
    $failedjobs | Format-Table -AutoSize
}
```

Here's an [article](https://docs.microsoft.com/en-us/windows-server/storage/file-server/troubleshoot/detect-enable-and-disable-smbv1-v2-v3) on detecting, enabling and disabling SMB v2, v2, v3 from Microsoft.
Hope this helps.
