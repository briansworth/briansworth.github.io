---
layout: post
title: Check if you are running as Administrator
---

I don't know about you, but sometimes I try to run commands that need administrative rights without checking if I have those permissions in my current session.
There have even been cases where I have built a function or script that needed a single command to run as an Administrator somewhere near the end, only to have the function get half way before saying 'Access denied'.

This post will help to resolve all that.

I will be creating a simple function that allows you to check this quickly and easily.
This should be used more as a utility/helper function than a standalone command.
<br>
First we need to get your current identity:
```powershell
[System.Security.Principal.WindowsIdentity]::GetCurrent()
```
You should get something like this (I took out the last few properties):
```
AuthenticationType : NTLM
ImpersonationLevel : None
IsAuthenticated    : True
IsGuest            : False
IsSystem           : False
IsAnonymous        : False
Name               : DESKTOP-EGGBBQF\Brian
Owner              : S-1-5-32-544
User               : S-1-5-21-3002136612-2508034227-3987083645-1001
Groups             : {S-1-5-21-3002136612-2508034227-3987083645-513, S-1-1-0, S-1-5-114, S-1-5-32-544...}
```

This contains some interesting information, like your SID and all the group SIDs that you belong to.
<br>

Next we need to get the principal from the Windows Identity we just obtained:
```powershell
$wid=[System.Security.Principal.WindowsIdentity]::GetCurrent()
$principal=new-object Security.Principal.WindowsPrincipal($wid)
```

A user's principal contains the user's Identity and their Role.
We will use this object to validate whether the current identity has the Administrator role.
<br>
The $principal object has a method 'IsInRole' that we can use against the Administrator role to get our answer.  First we will need to get the actual built-in Administrator role stored into an object.
```
$adminRole=[System.Security.Principal.WindowsBuiltInRole]::Administrator
```
Now that we have all the parts, we can finally get our answer.
<br>

### Putting it all together

-----

```powershell
$wid=[System.Security.Principal.WindowsIdentity]::GetCurrent()
$principal=New-Object -TypeName System.Security.Principal.WindowsPrincipal -ArgumentList $wid
$adminRole=[System.Security.Principal.WindowsBuiltInRole]::Administrator
$principal.IsInRole($adminRole)

```

Just these 4 lines of code will give us a True or False answer.
Making this into a helper function is incredibly straight forward.
<br>

### Here's the function:

-----
```powershell
function isAdministrator {
  $wid=[System.Security.Principal.WindowsIdentity]::GetCurrent()
  $principal=New-Object -TypeName System.Security.Principal.WindowsPrincipal -ArgumentList $wid
  $adminRole=[System.Security.Principal.WindowsBuiltInRole]::Administrator
  
  return $principal.IsInRole($adminRole)
}
```

Hopefully you find this to be a useful tool to keep in your inventory.

*Thanks for reading,*

PS> exit
