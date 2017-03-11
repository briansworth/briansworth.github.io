---
layout: post
title: Check if you are running as Administrator
---

I don't know about you, but sometimes I try to run commands that need administrative rights without checking if I am actually running in a shell with those permissions.
There have even been cases where I have built a function or script that needed a single command to run as an Administrator somewhere near the end, only to have the function get half way before saying 'Access denied'.

This post help to resolve all that.

I will be creating a simple function that allows you to check this quickly and easily.
This should be used more as a utility/helper function; not a standalone function.

First we need to get the current identity:
```
[System.Security.Principal.WindowsIdentity]::GetCurrent()
```
You should get something like this returned:
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
Token              : 2376
AccessToken        : Microsoft.Win32.SafeHandles.SafeAccessTokenHandle
UserClaims         : {http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name: DESKTOP-EGGBBQF\Brian,
                     http://schemas.microsoft.com/ws/2008/06/identity/claims/primarysid:
                     S-1-5-21-3002136612-2508034227-3987083645-1001...}
Claims             : {http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name: DESKTOP-EGGBBQF\Brian,
                     http://schemas.microsoft.com/ws/2008/06/identity/claims/primarysid:
                     S-1-5-21-3002136612-2508034227-3987083645-1001...}
DeviceClaims       : {}
Actor              :
BootstrapContext   :
Label              :
NameClaimType      : http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name
RoleClaimType      : http://schemas.microsoft.com/ws/2008/06/identity/claims/groupsid
```

This contains some interesting information, like your SID and all the group SIDs that you belong to.
Next we need to get the principal from the Windows Identity we just obtained:
```
$wid=[System.Security.Principal.WindowsIdentity]::GetCurrent()
$principal=new-object Security.Principal.WindowsPrincipal($wid)
```

The $principal object we just got back has a method 'IsInRole' that we can use against the Administrator role to get our answer:
```
$adminRole=[System.Security.Principal.WindowsBuiltInRole]::Administrator
```
Putting it all together:
```
$wid=[System.Security.Principal.WindowsIdentity]::GetCurrent()
$principal=New-Object -TypeName System.Security.Principal.WindowsPrincipal -ArgumentList $wid
$adminRole=[System.Security.Principal.WindowsBuiltInRole]::Administrator
$principal.IsInRole($adminRole)

```

Just these 4 lines of code will give you a True or False answer.
Putting this into a helper function is incredibly straight forward.

### Here's the function:

-----
```
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
