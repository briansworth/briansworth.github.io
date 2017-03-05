---
layout: post
title: Rename Computer with WMI
---

I recently found myself having to do some testing on a few Windows Server 2008r2 machines.  

From a clean install (SP1 included but nothing else), I found it strange that there was no 'Rename-Computer' cmdlet!
Many other cmdlets were missing as well, which made setting up the networking more difficult.  We can get into that another time.

Instead of looking for the cmd tool that could do the job (netdom), I decided to look around in wmi for the potential answer.

I know you can get the hostname of the machine from the Win32_ComputerSystem class so let's see what methods we find.
```powershell
Get-WmiObject -Class Win32_ComputerSystem | GM -MemberType Method

Name                    MemberType Definition
----                    ---------- ----------
JoinDomainOrWorkgroup   Method     System.Management.ManagementBaseObj...
Rename                  Method     System.Management.ManagementBaseObj...
SetPowerState           Method     System.Management.ManagementBaseObj...
UnjoinDomainOrWorkgroup Method     System.Management.ManagementBaseObj...
```

'Rename' looks like it should do it.
The 'Definition' of this method indicates that it accepts 3 arguments:
1. Name (string)
2. Password (string)
3. UserName (string)

```
Get-WmiObject -Class Win32_ComputerSystem | GM -Name Rename | Select -ExpandProperty Definition

System.Management.ManagementBaseObject Rename(System.String Name, System.String Password, System.String UserName)
```
The method args: `Rename(System.String Name, System.String Password, System.String UserName)`

In practice, you do not need to specify Password, or UserName.
The following example should work for a name of your choosing:
```
$newName='upgradeMe'
$system=Get-WmiObject -Class Win32_ComputerSystem
$system.Rename($newName)

__GENUS          : 2
__CLASS          : __PARAMETERS
__SUPERCLASS     :
__DYNASTY        : __PARAMETERS
__RELPATH        :
__PROPERTY_COUNT : 1
__DERIVATION     : {}
__SERVER         :
__NAMESPACE      :
__PATH           :
ReturnValue      : 0
```

A return value of 0 means success!
Be sure to restart the machine and enjoy your newly named computer!


## Common return values

### 5 - Access denied

I found providing the credentials of an elevated user while running PowerShell as a user without permissions still did not work.

```
$system.Rename($newName,"P@ssw0rd!","AdminUser")

__GENUS          : 2
__CLASS          : __PARAMETERS
__SUPERCLASS     :
__DYNASTY        : __PARAMETERS
__RELPATH        :
__PROPERTY_COUNT : 1
__DERIVATION     : {}
__SERVER         :
__NAMESPACE      :
__PATH           :
ReturnValue      : 5
``` 

I had to run a new PowerShell window with elevated privileges to get this to work.  As long as the user has the right permissions, there's no need to specify the credentials (Password and Username).

That being said, if you are using an account with proper permissions, you can get away with just specifying the Name.

In fact, I would recommend not specifying the credentials as you may run into the following return value if you mistype.

### 1326 - Logon Failure

You will likely get this if you specified a password and a username in the command:
```
$system.Rename($newName,"P@ssw0rd!","Admnstr")

__GENUS          : 2
__CLASS          : __PARAMETERS
__SUPERCLASS     :
__DYNASTY        : __PARAMETERS
__RELPATH        :
__PROPERTY_COUNT : 1
__DERIVATION     : {}
__SERVER         :
__NAMESPACE      :
__PATH           :
ReturnValue      : 1326
```

Ensure the Password and User are in the correct spot (Password is the 2nd argument, User is the 3rd) and that both are correctly spelled.

### 87 - Invalid parameter

You have either specified an invalid character in the name, or you have some how exceeded the number of characters allowed for a computer name (63 character limit).  

```
$newName='Computer~~02'
$system=GetWmiObject -Class Win32_ComputerSystem
$system.Rename($newName)

__GENUS          : 2
__CLASS          : __PARAMETERS
__SUPERCLASS     :
__DYNASTY        : __PARAMETERS
__RELPATH        :
__PROPERTY_COUNT : 1
__DERIVATION     : {}
__SERVER         :
__NAMESPACE      :
__PATH           :
ReturnValue      : 87

```
In this example we get the 87 return value because there are invalid characters in the new name. '~' is invalid in this case.

### 2697 - Computer account could not be found

This one is interesting as it is a network error. You can fix this be rebooting and trying again.

This seems to only be a problem in a domain environment, and only occurs if you have already successfully renamed the computer without restarting the machine. 

Your domain controllers update the computer object name as soon as this command is run successfully.  If you try to run it again before rebooting, it will look in Active Directory for the 'old' name and not find it.
After a reboot, your machine will now have the matching name to the one in Active Directory, and you can try again.

```
$newName='upgradeMe'
$system=Get-WmiObject -Class Win32_ComputerSystem
$system.Rename($newName)

__GENUS          : 2
__CLASS          : __PARAMETERS
__SUPERCLASS     :
__DYNASTY        : __PARAMETERS
__RELPATH        :
__PROPERTY_COUNT : 1
__DERIVATION     : {}
__SERVER         :
__NAMESPACE      :
__PATH           :
ReturnValue      : 0

$system.Rename("NewComputerName")

__GENUS          : 2
__CLASS          : __PARAMETERS
__SUPERCLASS     :
__DYNASTY        : __PARAMETERS
__RELPATH        :
__PROPERTY_COUNT : 1
__DERIVATION     : {}
__SERVER         :
__NAMESPACE      :
__PATH           :
ReturnValue      : 2697
```

I know there are plenty other return values; however, these were the codes that I encountered that were common enough to elaborate on.

*Thanks for reading,*

PS> exit
