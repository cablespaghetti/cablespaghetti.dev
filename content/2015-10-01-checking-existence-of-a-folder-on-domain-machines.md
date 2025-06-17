---
layout: post
title:  "Checking the existence of a folder on all domain machines"
description: "I needed a way of finding a list of machines where the JRE install had failed or not run because the user hasn't rebooted for a while. So basically I needed to check that the Java install directory was present on all PCs on the Windows domain."
date:   2015-10-01 18:00
tags: scripts, powershell
redirect_from:
  - /2015/10/01/checking-existence-of-a-folder-on-domain-machines.html
---

I haven't posted since March which is pretty shameful. I've been extremely busy working on some web services magic that I can't really share at this point if I value my pay check. However I had a sysadmin problem today that warranted some script writing which I thought I'd share.

I've been trying to get Oracle Java updates to deploy via Group Policy. I've tried a number of different approaches before using Puppet, but all of them proved unreliable. I think I've got the Group Policy method cracked now and I'll share it here soon. 

Anyway, I needed a way of finding a list of machines where the JRE install had failed or not run because the user hasn't rebooted for a while. So basically I needed to check that the Java install directory was present on all PCs on the Windows domain.

The script just gets a list of the computers in the relevant OU and then loops over them, printing a message if it finds 'Program Files' but not the specific Java directory. This weeds out machines that are unreachable or not running Windows (we've got a few Ubuntu boxes in our Workstations OU). I expected it to be much more work!

```powershell
$Computers = Get-ADComputer -filter * -Searchbase "OU=ComputerOU,DC=Domain,DC=Local" | % {$_.Name}
Foreach($Computer in $Computers) {
  if ((Test-Path "\\$Computer\c$\Program Files") -And !(Test-Path "\\$Computer\c$\Program Files\Java\jre1.8.0_60")) {
      echo "$Computer doesn't have the latest Java"
  }
}
```
