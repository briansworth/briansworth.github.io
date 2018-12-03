---
layout: post
title: PowerShell - Active Directory UPN Suffixes
---

<p>
Managing UPN Suffixes for in an Active Directory environment is 
a set-it-and-forget-it kind of task. 
This type of task, by it's very nature, is easily forgotten. 
I have had to re-remember how to do this in PowerShell many times;
as a result, it is time for me to write it down and never forget it. 
</p>

### Getting the UPN Suffixes
----

<p>
UPN Suffixes are controlled at the forest level (not at the domain level). 
To get them using PowerShell and with the Active Directory Module it is easy.
</p>

```powershell
$forest=Get-ADForest 
$forest | Select Name, UPNSuffixes
```
<p>
What if you don't have the ActiveDirectory Module handy? 
</p>
<p>
Great question!  
Not as easy, but still manageable.
</p>
<p>
Before showing you exactly how, 
it is important to remember that UPN suffixes are managed at the forest level. 
More specifically, the Partitions container has a property called uPNSuffixes. 
If we can locate that container, we can tell what our UPN suffix situation is.
</p>

*NOTE: This will only work if you are in the root domain of your forest*
*We will address how to get around this later*

```powershell
$rootDse=New-Object -TypeName DirectoryServices.DirectoryEntry `
  -ArgumentList 'LDAP://rootDse'
$cfgCtx=$rootDse.Properties['configurationNamingContext']
$cfgEntry=New-Object -TypeName DirectoryServices.DirectoryEntry `
  -ArgumentList "LDAP://$cfgCtx"

$searcher=New-Object -TypeName DirectoryServices.DirectorySearcher `
  -ArgumentList 'objectCategory=crossRefContainer'
$searcher.SearchRoot=$cfgEntry
$result=$searcher.FindOne()

$partitionsEntry=$result.GetDirectoryEntry()
$upnSuffix=$partitionsEntry.Properties['uPNSuffixes']

if(!$upnSuffix){
  Write-Output "No UPNSuffixes found"
}else{
  New-Object -TypeName PSObject -Property @{
    "UPNSuffixes"=$upnSuffix;
  }
}
```

<p>
By default there is no UPNSuffixes property because none have been created. 
The standard upn suffix will be the same as the domain DNS name which is not 
stored in this property on the Partitions container.
</p>


### Add or Remove UPN Suffixes
----

<p>
You will need to be a member of the Domain Admins or Enterprise Admins 
security groups in the forest root domain you are modifying.
</p>
#### Add UPN suffixes the easy way
---

<p>
Using the Active Directory PowerShell module:
</p>
```powershell
$forestName='codeAndKeep.local'

$forest=Get-ADForest -Identity $forestName
Set-ADForest -Identity $forest.Name -UPNSuffixes @{Add='codeAndKeep.com'}
```

#### Remove UPN suffixes the easy way
---

<p>
Using the Active Directory PowerShell module:
</p>
```powershell
$forestName='codeAndKeep.local'

$forest=Get-ADForest -Identity $forestName
Set-ADForest -Identity $forest.Name -UPNSuffixes @{Remove='codeAndKeep.com'}
```

#### Add UPN suffixes the harder way
---

<p>
Not using the Active Directory PowerShell module. 
</p>

```powershell
$rootDse=New-Object -TypeName DirectoryServices.DirectoryEntry `
  -ArgumentList 'LDAP://rootDse'
$cfgCtx=$rootDse.Properties['configurationNamingContext']
$cfgEntry=New-Object -TypeName DirectoryServices.DirectoryEntry `
  -ArgumentList "LDAP://$cfgCtx"

$searcher=New-Object -TypeName DirectoryServices.DirectorySearcher `
  -ArgumentList 'objectCategory=crossRefContainer'
$searcher.SearchRoot=$cfgEntry
$result=$searcher.FindOne()

$partitionsEntry=$result.GetDirectoryEntry()

$partitionsEntry.Properties['uPNSuffixes'].Add('codeAndKeep.com')
$partitionsEntry.CommitChanges()
```

#### Remove UPN suffixes the harder way
---

<p>
Not using the Active Directory PowerShell module. 
</p>

```powershell
$rootDse=New-Object -TypeName DirectoryServices.DirectoryEntry `
  -ArgumentList 'LDAP://rootDse'
$cfgCtx=$rootDse.Properties['configurationNamingContext']
$cfgEntry=New-Object -TypeName DirectoryServices.DirectoryEntry `
  -ArgumentList "LDAP://$cfgCtx"

$searcher=New-Object -TypeName DirectoryServices.DirectorySearcher `
  -ArgumentList 'objectCategory=crossRefContainer'
$searcher.SearchRoot=$cfgEntry
$result=$searcher.FindOne()

$partitionsEntry=$result.GetDirectoryEntry()

$partitionsEntry.Properties['uPNSuffixes'].Remove('codeAndKeep.com')
$partitionsEntry.CommitChanges()
```

### UPN Suffixes in Other Scenarios
----

<p>
If you are in another
</p>


```powershell
Param(
  [String]$forestName='codeAndKeep.local'

  [Management.Automation.PSCredential]$forestCred=Get-Credential
)

Function New-ADContext {
  [CmdletBinding()]
  Param(
    [ValidateSet(
      'Forest',
      'Domain',
      'ConfigurationSet',
      'ApplicationPartition',
      'DirectoryServer'
    )]
    [String]$contextType='Forest',

    [String]$targetName,

    [Management.Automation.PSCredential]$credential
  )
  $ctorArgs=@($contextType)
  if($PSBoundParameters.ContainsKey('targetName')){
    $ctorArgs+=$targetName
  }
  if($PSBoundParameters.ContainsKey('credential')){
    $ctorArgs+=$credential.UserName
    $ctorArgs+=$credential.GetNetworkCredential().Password
  }
  New-Object -TypeName DirectoryServices.ActiveDirectory.DirectoryContext `
    -ArgumentList $ctorArgs
}

$forestCtx=New-ADContext -contextType Forest `
  -targetName $forestname `
  -credential $forestCred

$forest=[DirectoryServices.ActiveDirectory.Forest]::GetForest($forestCtx)
$rootDse=New-Object -TypeName DirectoryServices.DirectoryEntry `
  -ArgumentList @(
    "LDAP://$($forest.Name)/rootDse",
    $forestCred.UserName,
    $forestCred.GetNetworkCredential().Password
  )

$cfgCtx=$rootDse.Properties['configurationNamingContext'].Value

$partitionsEntry=New-Object -TypeName DirectoryServices.DirectoryEntry `
  -ArgumentList @(
    "LDAP://$($forest.Name)/CN=Partitions,$cfgCtx",
    $forestCred.Username,
    $forestCred.GetNetworkCredential().Password
  )

$upnSuffixes=$partitionsEntry.Properties['uPNSuffixes']
New-Object -TypeName PSObject -Property @{
  ForestName=$forest.Name;
  Domains=$forest.Domains;
  UPNSuffixes=$upnSuffixes;
}
```
