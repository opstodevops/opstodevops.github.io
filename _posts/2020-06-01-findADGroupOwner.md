---
layout: post
title:  "Find Active Directory Group Ownership (Who owns AD group?)"
date:   2020-06-01
categories: [blog]
tags: [powershell, active directory, AD, groups]
excerpt_separator: <!--more-->
---

I had an interesting request today, I was asked to pull AD groups owned by 2 users who were leaving the company. 
Not a biggie, a simple PowerShell query should reveal that in no time. With their LAN ID handy, I ran a query in PowerShell.
<!--more-->

```powershell
OpsToDevOps:\> Get-ADUser firstuser -Properties ManagedObjects | Select -ExpandProperty ManagedObjects

CN=group2,OU=Application Security,OU=Security,OU=NZ,DC=bogus,DC=local
CN=group1,OU=Application Security,OU=Security,OU=NZ,DC=bogus,DC=local

OpsToDevOps:\> Get-ADUser seconduser -Properties ManagedObjects | Select -ExpandProperty ManagedObjects

CN=group5,OU=Application Security,OU=Security,OU=NZ,DC=bogus,DC=local
CN=group3,OU=Application Security,OU=Security,OU=NZ,DC=bogus,DC=local
```

I could simply export the result, hand over to requester and close the ticket off.\
Turns out that the users `firstuser` and `seconduser` own more groups but the `ManagedBy` field of those groups was not populated when the groups were created. 
Instead the notes section was filled with ownership information. So I wrote a quick script which will query both the fields and return a nice table which I can export.

```powershell
$firstUser  = (Get-ADUser firstuser).DistinguishedName

$secondUser = (Get-ADUser seconduser).DistinguishedName

$searchBase = "OU=Application Security,OU=Security,OU=NZ,DC=bogus,DC=local"

$Output =@()

Get-ADGroup -Filter {Name -like "Group*"} -Properties managedBy,Info -SearchBase $searchBase | 
Where-Object {$PSitem.info -match "firstuser|seconduser" -or $PSitem.managedby -eq $firstUser -or $PSitem.managedby -eq $secondUser} | ForEach-Object {
    $Output += New-Object psobject -Property @{
        ADGroup    = $PSitem.Name
        GroupOwner = $PSitem.ManagedBy
        Notes      = $PSitem.Info
    
    }
} 

$Output | Export-Csv -Path "Temp\GroupOwners-$(Get-Date -f dd-MM-yyyy).csv" -NoTypeInformation –Append
```

I can also pipe out `$Output` and see what's been exported. Neat, eh!

![findadgroups_1](/assets/adgroups_1.PNG)

See you in the next one.

Stay safe!



