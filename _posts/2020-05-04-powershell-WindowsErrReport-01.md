---
layout: post
title:  "PWSH Get-WinEvent WER"
date:   2020-05-04
categories: [blog]
tags: [powershell]
excerpt_separator: <!--more-->
---

I had to collect Windows Error Reporting (WER) for past 7 days from one of the Windows server (Ugh!), not detailed but enough to see what apps were crashing the most.
Thought I would share what I wrote.

<!--more-->

{% highlight powershell %}

Get-WinEvent -FilterHashtable `
@{'providername' = 'Windows Error Reporting';starttime=(Get-Date).AddDays(-7);Id=1001} |
Select-Object -Property TimeCreated, @{name='App';expression={$_.Properties[5].value}} |
Group-Object -Property App |
Select-Object -Property Name,Count |
Sort-Object -Property Count -Descending

{% endhighlight %}

Here's the output of the exact command run on the said server.

![pwsh_01](/assets/pwrshell-wer-01.PNG)