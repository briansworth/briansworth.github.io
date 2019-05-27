---
layout: post
title: PowerShell - Automate SCCM AD PreReqs
description: Script to automate Active Directory tasks for an SCCM deployment
---

This is a continuation of automating Sccm prerequisites [part 1](
{{ site.baseurl }}/PowerShell-SCCM-Offline-PreRequisites/) and 
[part 2](
{{ site.baseurl }}/PowerShell-SCCM-Offline-PreRequisites-Install/). 
<p>
  This post will be about setting up the Active Directory prerequisites. 
  Specifically creating the System Management container and adding the 
  relevant permissions to that container. 
  I will not be tackling the AD Schema extension. 
  The executable in the Sccm installation media (ExtADSch.exe) does the job 
  perfectly and quickly. 
</p>

<p>
  Just as in my previous Sccm prereq posts, 
  I will be writing a full script for the process. 
  There will be examples at the end of the post as well. 
  Jump to the end for the script, 
  or go through the journey of figuring it out with me.
</p>

### Creating the System Management Container
----

<p>
The creation of this container is made trivial with the ActiveDirectory 
PowerShell module.
</p>

```powershell
$rootDse=Get-ADRootDse
$path="CN=System,$($rootDse.DefaultNamingContext)"
New-ADObject -Name 'System Management' -Path $path -Type Container -PassThru
```

<p>
  The New-ADObject cmdlet will even allow you to target a different domain 
  using the -Server parameter, 
  and use different credentials using the -Credential parameter.
</p>

```powershell
$cred=Get-Credential
$targetDomain='trustme.local'

$rootDse=Get-ADRootDse -Server $targetDomain -Credential $cred
$path="CN=System,$($rootDse.DefaultNamingContext)"
New-ADObject -Name 'System Management' `
  -Path $path `
  -Type Container `
  -Server $targetDomain `
  -Credential $cred `
  -PassThru
```

<p>
  This should work assuming you have a separate domain / forest named 
  trustme.local, and you can contact that domain. 
</p>
<br>

#### No Active Directory Module?
----

<p>
  I am incredibly lazy, and when I'm setting up my Sccm server, 
  I don't feel like leaving that server to connect to a domain controller. 
  Setting up a PSSession to a domain controller to get access to 
  the Active Directory module would be simple enough, 
  but I want to see what it takes to create a container without it.
</p>
<p>
  To accomplish this, we can leverage the DirectoryServices .Net namespace. 
  If you haven't explored what you can do with this, you are missing out. 
  I will primarily be using the DirectoryEntry object as you will see.
</p>

```powershell
$rootDse=New-Object -TypeName DirectoryServices.DirectoryEntry `
  -ArgumentList 'LDAP://rootDse'
$rootName=$rootDse.Properties['defaultNamingContext'].Value

$path="CN=System,$rootName"
$systemEntry=New-Object -TypeName DirectoryServices.DirectoryEntry `
  -ArgumentList "LDAP://$path"

$systemMgtEntry=$systemEntry.Children.Add('CN=System Management','Container')
$systemMgtEntry.CommitChanges()
$systemMgtEntry
```

<p>
  A bit more involved, but not too crazy.  
  It does get more complicated when you want to use alternate credentials 
  and/or target a different domain. 
  I will address that later in the post. 
  Now lets get into the permissions.
</p>


### Add Container Permissions
----

<p>
  This was always the part I was scared of. 
  Security and permissions have always seemed a bit daunting. 
  Especially with all the different security permissions you can 
  setup for an AD container. 
  The nice thing with the ActiveDirectory module, 
  is that you can use the same cmdlets to get / set permissions 
  that you would use for files / folders on your file system 
  (Get-Acl / Set-Acl). 
  You will just need to change directories into the AD:\ drive 
  to do so.
</p>
<p>
  I have opted to go the different route, 
  and use DirectoryServices to get the job done. 
  I want to assume the Active Directory module is not available, 
  and that I can run the script directly from the Sccm server. 
</p>
<br>
<p>
  Lets take a look at what permissions are set on the container.
</p>

```powershell
$systemMgtEntry.ObjectSecurity.GetAccessRules(
  $true, $true, [Security.Principal.NTAccount]
)
```
<p>
  The output should fill up the PowerShell window. 
  The most useful properties in this output are 
  'ActiveDirectoryRights' and 'IdentityReference'. 
  This should tell what access is given to whom.
</p>
<br>
<p>
  Now to add permissions for the Sccm server.
</p>

```powershell
  # Get the AD Computer for the Sccm server first
  $computerName='CM1'

  $cmComputer=New-Object -TypeName DirectoryServices.DirectoryEntry `
    -ArgumentList "LDAP://CN=$computerName,CN=Computers,$rootName"

  # We need the Security Identifier (SID) for the Sccm AD Computer
  [byte[]]$cmSidBytes=$cmComputer.Properties['objectsid'][0]
  $cmSid=New-Object -TypeName Security.Principal.SecurityIdentifier `
    -ArgumentList $cmSidBytes, 0
  

  # Create the Access Control Entry (ACE) for the Sccm computer
  $adRights=[DirectoryServices.ActiveDirectoryRights]::GenericAll
  $accessType=[Security.AccessControl.AccessControlType]::Allow
  $inheritance=[DirectoryServices.ActiveDirectorySecurityInheritance]::All

  $fullAccessACE=New-Object -TypeName DirectoryServices.ActiveDirectoryAccessRule `
    -ArgumentList @($cmSid, $adRights, $accessType, $inheritance)


  # Add the ACE to the container
  $systemMgtEntry.ObjectSecurity.AddAccessRule($fullAccessACE)
  $systemMgtEntry.CommitChanges()
```

<p>
  Now you should have setup Full Control security permissions 
  for your Sccm server on the System Management container. 
</p>

#### A couple problems
----

<p>
  As mentioned before, 
  this get more complicated when you want to do this for a remote domain. 
  Even more so if the target domain and the 
  Sccm server are in separate domains or forests. 
  Additionally, this will make the use of credentials a requirement. 
  Again, I want to be able to run this script directly from my Sccm server, 
  because I am incredibly lazy. 
  -- Why am I making this so hard for myself then? --
</p>
<br>

### The Script
---

<p>
  I would recommend copying all the code below, 
  pasting it into a text editor, 
  then saving it as a ps1 file (SccmADPreReq.ps1 for example).
</p>
*NOTE: There will be some examples below the code here*

```powershell
[CmdletBinding()]
Param(
  [Parameter(Position=0)]
  [String]$computerName=$ENV:COMPUTERNAME,

  [Parameter(Position=1)]
  [Management.Automation.PSCredential]$sccmDomainCredential,

  [Parameter(Position=2)]
  [String]$sccmDomainController,

  [Parameter(Position=3)]
  [Management.Automation.PSCredential]$containerDomainCredential,

  [Parameter(Position=4)]
  [String]$containerDomainController
)

# Helper Function Definition
Function Get-LdapDirectoryEntry {
  [CmdletBinding()]
  Param(
    [Parameter(Position=0,Mandatory=$true)]
    [String]$Identity,

    [Parameter(Position=1)]
    [Management.Automation.PSCredential]$Credential,

    [Parameter(Position=2)]
    [String]$Server
  )
  Begin{
    $strBuilder=New-Object -TypeName Text.StringBuilder `
      -ArgumentList "LDAP://"
  }
  Process{
    Try{
      if($PSBoundParameters.ContainsKey('Server')){
        [void]$strBuilder.Append("$Server/")
      }

      if($Identity -match "(^(CN|DC|OU)=.+\,(CN|DC|OU)=.+)|rootDse"){
        [void]$strBuilder.Append($Identity)
      }else{
        Write-Error "Identity [$Identity] is not a valid format." `
          -ErrorAction Stop
      }
      $ctorArgs=@(
        $strBuilder.ToString()
      )

      Write-Verbose "Query $($strBuilder.ToString())"
      if($PSBoundParameters.ContainsKey('Credential')){
        $ctorArgs+=$Credential.Username
        $ctorArgs+=$($Credential.GetNetworkCredential().Password)
      }

      $entry=New-Object -TypeName DirectoryServices.DirectoryEntry `
        -ArgumentList $ctorArgs `
        -ErrorAction Stop
      if(!$entry.Path){
        Write-Error "Directory Entry [$Identity] not found." `
          -ErrorAction Stop
      }
      Write-Output $entry
    }Catch{
      Write-Error $_
    }
  }
  End{}
}

# END Helper Function Definition


Try{
  $computerParam=@{
    'ErrorAction'='Stop';
    'Identity'='rootDse';
  }
  if($PSBoundParameters.ContainsKey('sccmDomainCredential')){
    $computerParam.Add('Credential',$sccmDomainCredential)
  }
  if($PSBoundParameters.ContainsKey('sccmDomainController')){
    $computerParam.Add('Server',$sccmDomainController)
  }

  $rootDse=Get-LdapDirectoryEntry @computerParam

  $nameCtx=$rootDse.Properties['defaultNamingContext'].Value

  if(!$nameCtx){
    Write-Error "Error connecting to 'SCCM Computer' RootDse" `
      -ErrorAction Stop
  }
  $computerParam.Identity=$nameCtx
  $domainContainer=Get-LdapDirectoryEntry @computerParam

  # Get CM Computer in AD 

  $computerSearcher=New-Object -TypeName DirectoryServices.DirectorySearcher `
    -ArgumentList $domainContainer

  $computerSearcher.Filter="(&(objectCategory=computer)(cn=$computerName))"
  [void]$computerSearcher.PropertiesToLoad.Add('objectsid')
  $cmComputer=$computerSearcher.FindOne()
  if(!$cmComputer){
    Write-Error "Unable to find AD Computer: $computerName" `
      -ErrorAction Stop
  }
  Write-Verbose "SCCM Computer Found: $($cmComputer.Path)"

  # Get CM Computer SID

  [byte[]]$cmSidBytes=$cmComputer.Properties['objectsid'][0]
  $cmSid=New-Object -TypeName Security.Principal.SecurityIdentifier `
    -ArgumentList $cmSidBytes, 0
  $cmIdentityReference=$cmSid.Translate([Security.Principal.NTAccount])


  # Add System Management Container
  $containerParam=@{
    'Identity'='rootDse';
    'ErrorAction'='Stop';
  }
  if($PSBoundParameters.ContainsKey('containerDomainCredential')){
    $containerParam.Add('Credential',$containerDomainCredential)
  }
  if($PSBoundParameters.ContainsKey('containerDomainController')){
    $containerParam.Add('Server',$containerDomainController)
  }

  $containerRootDse=Get-LdapDirectoryEntry @containerParam
  $containerNameCtx=$containerRootDse.Properties['defaultNamingContext'].Value
  if(!$containerNameCtx){
    Write-Error "Error connecting to 'Container' RootDse" `
      -ErrorAction Stop
  }
  
  $containerParam.Identity=$containerNameCtx
  $containerDomainContainer=Get-LdapDirectoryEntry @containerParam


  $sysMgtName="CN=System Management"
  $systemDN="CN=System,$($containerDomainContainer.distinguishedName.Value)"

  $containerParam.Identity=$systemDN

  $systemEntry=Get-LdapDirectoryEntry @containerParam

  $searcher=New-Object -TypeName DirectoryServices.DirectorySearcher `
    -ArgumentList $systemEntry
  $searcher.SearchScope=1
  $searcher.Filter="(&(objectcategory=container)(cn=System Management))"
  $result=$searcher.FindOne()
  if(!$result){
    Write-Verbose "Adding $sysMgtName container"
    $sysMgtEntry=$systemEntry.Children.Add($sysMgtName,'Container')
    $sysMgtEntry.CommitChanges()
    $result=$searcher.FindOne()
    if(!$result){
      Write-Error "Unable to create System Management container" `
        -ErrorAction Stop
    }
  }else{
    $sysMgtEntry=$result.GetDirectoryEntry()
  }

  # Add AccessRights for CM Computer on System Management container

  $sysMgtDN=$sysMgtEntry.distinguishedName.Value
  Write-Verbose "Checking $sysMgtName permissions"
  $adRights=[DirectoryServices.ActiveDirectoryRights]::GenericAll
  $accessType=[Security.AccessControl.AccessControlType]::Allow
  $inheritance=[DirectoryServices.ActiveDirectorySecurityInheritance]::All

  $fullAccessACE=New-Object -TypeName DirectoryServices.ActiveDirectoryAccessRule `
    -ArgumentList @($cmSid, $adRights, $accessType, $inheritance)

  $sysMgtACE=$sysMgtEntry.ObjectSecurity.GetAccessRules(
    $true, 
    $true, 
    [Security.Principal.NTAccount]
  )

  $cmRightsExist=$sysMgtACE | 
    Where-Object {
      $_.IdentityReference.Value -eq $cmSid.Value -or
      $_.IdentityReference.Value -eq $cmIdentityReference.Value
    }
  if(!$cmRightsExist){
    Write-Verbose "Adding permissions $($cmIdentityReference.Value) to $sysMgtDN"
    $sysMgtEntry.ObjectSecurity.AddAccessRule($fullAccessACE)
    $sysMgtEntry.CommitChanges()
  }else{
    $warn=[String]::Format(
      "Computer: {0} permissions were already added to {1}",
      "$computerName",
      "System Management. Please Verify..."
    )
    Write-Warning $warn 
    Write-Output $cmRightsExist
  }
}Catch{
  Write-Error $_
}Finally{
  if($computerSearcher){
    $computerSearcher.Dispose()
  }
  if($searcher){
    $searcher.Dispose()
  }
}
```

<br>

<p>
  When running the script, 
  I highly recommend using the -Verbose switch to see what it is doing. 
  It will make troubleshooting easier if you run into problems.  
</p>

#### Example 1

```powershell
.\SccmADPrereq.ps1 -computerName cm1 -Verbose
```
<p>
  Description: This will create the System Management container and add the 
  required permissions for the computer 'cm1'.  
  It will do this for the domain you are logged in to.
  It assumes both the Sccm computer and System Management container 
  are in the same domain. 
</p>

#### Example 2

```powershell
$domain='CodeAndKeep.com'
$domainCred=Get-Credential
.\SccmADPrereq.ps1 -computerName cm1 -containerDomainCredential $domainCred -containerDomainController $domain -Verbose
```
<p>
  Description: This will create the System Management container and add the 
  required permissions for the computer 'cm1'.
  However, the Sccm computer 'cm1' is located in the current domain. 
  The System Management container will be created and modified 
  in the 'CodeAndKeep.com' domain. 
  Alternate credentials are specified for creating the container. 
  The current logged on user's permissions will be used to query the 
  Sccm AD computer.
</p>

#### Example 3

```powershell
$sccmDomain='CodeAndKeep.com'
$sccmDomainCred=Get-Credential
.\SccmADPrereq.ps1 -computerName cm1 -sccmDomainCredential $sccmDomainCred -sccmDomainController $sccmDomain -Verbose
```

<p>
  Description: This will create the System Management container and add the 
  required permissions for the computer 'cm1'.
  However, the Sccm computer 'cm1' is in a different domain 'CodeAndKeep.com'. 
  Alternate will be used for querying the remote domain 
  for the Sccm Computer cm1. 
  The System Management container will be added to the current domain, 
  using the current logged on user's permissions. 
</p>

```powershell
$sccmDomain='CodeAndKeep.com'
$cred=Get-Credential
.\SccmADPrereq.ps1 -computerName cm1 -sccmDomainController $sccmDomain -containerDomainCredential $cred -Verbose
```

<p>
  Description: This will create the System Management container and add the 
  required permissions for the computer 'cm1'.
  However, the Sccm computer 'cm1' is in a different domain 'CodeAndKeep.com'. 
  The logged in user's permissions will be used to query that remote domain 
  for the Sccm computer.
  The System Management container will be added to the current domain, 
  and alternate credentials would be used to add/modify the 
  System Management container. 
</p>
