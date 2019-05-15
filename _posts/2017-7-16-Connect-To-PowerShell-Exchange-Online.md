---
layout: post
title: Connect PowerShell to Exchange Online
description: Simple function to make connecting to Exchange Online easy
---

<p>
  There are many examples on the internet that show how to
  remotely connect to Exchange Online PowerShell.
  Hopefully this post will provide better code, 
  and better insight into the process.
</p>

The process consists of 2 steps:
1. Creating a PSSession to Exchange Online
2. Importing that PSSession

```powershell
$session=New-PSSession -ConfigurationName 'Microsoft.Exchange' `
  -ConnectionUri 'https://outlook.office365.com/powershell-liveid/' `
  -AllowRedirection `
  -Authentication 'Basic' `
  -Credential (Get-Credential) `
  -ErrorAction 'Stop'

Import-PSSession $session
```
<p>
  Putting this into a function is simple, 
  but you may want some additional features.
</p>

### Additional Considerations
----

<p>
  I like to differentiate the Exchange Online cmdlets vs
  the OnPrem Exchange cmdlets.  
  To do this, I simply add a prefix to the Exchange Online cmdlets.
</p>
<p>
  Adding a prefix will allow you to, at a glance, determine 
  where your code is intended to run.
  It also means that you can use both Exchange Online and OnPrem Exchange
  cmdlets in the same shell.
</p>
<p>
  With that in mind, here's what I have come up with:
</p>

```powershell
function Connect-ExchangeOnline {
  [CmdletBinding()]
  Param(
    [Parameter(Position=0)]
    [Management.Automation.PSCredential]$credential,

    [AllowNull()]
    [String]$prefix='Online',

    [Switch]$allowClobber
  )
  if(!($PSBoundParameters.ContainsKey('credential'))){
    $credential=Get-Credential `
      -ErrorAction Stop
  }

  $exchSession=New-PSSession -ConfigurationName 'Microsoft.Exchange' `
    -ConnectionUri 'https://outlook.office365.com/powershell-liveid/' `
    -AllowRedirection `
    -Authentication 'Basic' `
    -Credential $credential `
    -ErrorAction 'Stop'

  [Hashtable]$importParam=@{
    'Session'=$exchSession;
    'ErrorAction'='Stop';
    'Verbose'=$false;
  }
  if($PSBoundParameters.ContainsKey('allowClobber')){
    $importParam.Add('AllowClobber',$true)
  }
  if($prefix){
    Write-Verbose "Adding prefix [$prefix]"
    $importParam.Add('Prefix',$prefix)
  }
  Import-PSSession @importParam
}
```
<p>
  The -Prefix parameter allows you to set one your self based on your needs.
  The 'AllowNull' attribute will let you specify null if you don't want a prefix.

  You can also AllowClobber if you want to overwrite your Exchange cmdlets.
</p>
<p>
  If you work with Exchange Online regularly, I would recommend putting 
  this function into your PowerShell profile so it is always ready to go.
</p>
