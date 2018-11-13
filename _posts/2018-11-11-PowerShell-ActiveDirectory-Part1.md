---
layout: post
title: PowerShell - ActiveDirectory and Exchange
---

<p>
ActiveDirectory is of course a huge component in a Microsoft based environment.
One of my favorite things to do is explore how other Microsoft products,
integrate or leverage Active Directory.
</p>

### Active Directory Basics
<p>
First things first, we need to get some info about the ActiveDirectory configuration. 
We should take a look at the RootDSE so we can determine the common naming contexts to use. 
</p>
```powershell
$rootDse=New-Object -TypeName DirectoryServices.DirectoryEntry `
  -ArgumentList 'LDAP://rootDse'
Write-Output $rootDse.Properties
```
<p>
Not the prettiest to look at, but it contains some useful information. 
For now we are interested in the Configuration Naming Context. 
You can get that like so:
</p>
```powershell
$adConfigRoot=$rootDse.Properties['configurationNamingContext'].Value
Write-Output $adConfigRoot
```
<p>
You could of course specify this without all this effort. 
I do find that knowing how to get this info means you don't have to hard code this stuff;
always a good thing in my books.
</p>

<p>
Of course if you have the Active Directory module handy, you can get the same info like so:
</p>
```powershell
$rootDse=Get-ADRootDSE
$cfgCtx=$rootDse.ConfigurationNamingContext
```

### List Exchange Servers
<p>
To make make our lives easier, I will be assuming you have the AD Module available for this part. 
</p>
<p>
It shouldn't surprise you that Exchange dumps a lot of information into ActiveDirectory. 
If you did the install you will know that extending the schema 
is of course a prerequisite for installing Exchange. 
This means lots of new object classes and attributes to play with.
</p>
```powershell
Get-ADObject -Filter "ObjectCategory -eq 'msExchExchangeServer'"
```
<p>
This should not have shown any result. 
The reason for this is that the Get-ADObject will, by default, query in the 
default naming context. 
The information we are after is in the configuration naming context.
</p>
```powershell
$exchServers=Get-ADObject -Filter "ObjectCategory -eq 'msExchExchangeServer'" `
  -SearchBase $cfgCtx
Write-Output $exchServers | FL *
```
<p>
You should see some output here. 
These AD Objects correspond to each Exchange Server in your environment. 
If you add `-Properties *` you will see many properties about each server. 
A lot of the properties have to do with log/data locations. 
You can use the msExchCurrentServerRoles to determine which roles are installed on each server. 
You can find out the FQDN of each server using the networkAddress property. 
You can also determine the version of Exchange from the serialNumber property 
or the versionNumber property (with some work).
</p>
<p>
Given this information we can create a function to list out all 
Exchange servers in a given domain.
</p>
```powershell
Function Get-ADExchangeServer {
  # first a quick function to convert the server roles to a human readable form
  Function ConvertToExchangeRole {
    Param(
      [Parameter(Position=0)]
      [int]$roles
    )
    $roleNumber=@{
      2='MBX';
      4='CAS';
      16='UM';
      32='HUB';
      64='EDGE';
    }
    $roleList=New-Object -TypeName Collections.ArrayList
    foreach($key in ($roleNumber).Keys){
      if($key -band $roles){
        [void]$roleList.Add($roleNumber.$key)
      }
    }
    Write-Output $roleList
  }

  # Get the Configuration Context
  $rootDse=Get-ADRootDSE
  $cfgCtx=$rootDse.ConfigurationNamingContext

  # Query AD for Exchange Servers
  $exchServers=Get-ADObject -Filter "ObjectCategory -eq 'msExchExchangeServer'" `
    -SearchBase $cfgCtx `
    -Properties msExchCurrentServerRoles, networkAddress, serialNumber
  foreach($server in $exchServers){
    Try{
      $roles=ConvertToExchangeRole -roles $server.msExchCurrentServerRoles

      $fqdn=($server.networkAddress | 
        Where-Object {$_ -like 'ncacn_ip_tcp:*'}).Split(':')[1]

      New-Object -TypeName PSObject -Property @{
        Name=$server.Name;
        DnsHostName=$fqdn;
        Version=$server.serialNumber[0];
        ServerRoles=$roles;
      }
    }Catch{
      Write-Error "ExchangeServer: [$($server.Name)]. $($_.Exception.Message)"
    }
  }
}
```

<p>
Of course if you connect to the Exchange Management Shell, 
you can run Get-ExchangeServer which will give more information. 
If you wanted to attempt connecting to any Exchange server, 
try the next if that one fails, you could leverage this function to do just that.
</p>

```powershell
Function Connect-ExchangeServer {
  $exchangeServers=Get-ADExchangeServer

  foreach($exServer in $exchangeServers){
    Try{
      $winrmTest=Test-WSMan -ComputerName $exServer.DnsHostName `
        -ErrorAction Stop
      if($winrmTest){
        $exSession=New-PSSession -ConfigurationName Microsoft.Exchange `
          -ConnectionUri "http://$($exServer.DnsHostName)/powershell" `
          -ErrorAction Stop

        return $exSession
      }
    }Catch{
      Write-Warning "Failed to connect to: [$($exServer.DnsHostName)]"
    }
  }
}
```
<p>
Quick and dirty but it should work. 
I can't help myself, so I will need to make this better by adding support for 
credentials, and specifying a target server / domain for the AD Queries. 
Also it would be nice for the function to allow manually specifying a server 
instead of always enumerating through the entries in Active Directory.
I think site level enumeration would be nice as well, 
since you may have a better connection to one site over the other.
</p>

```powershell
Function Get-ADExchangeServer {
  [CmdletBinding()]
  Param(
    [Parameter(Position=0)]
    [Management.Automation.PSCredential]$credential,

    [Parameter(Position=1)]
    [String]$server,

    [Parameter(Position=2)]
    [String]$siteName
  )
  Begin{
    Function ConvertToExchangeRole {
      Param(
        [Parameter(Position=0)]
        [int]$roles
      )
      $roleNumber=@{
        2='MBX';
        4='CAS';
        16='UM';
        32='HUB';
        64='EDGE';
      }
      $roleList=New-Object -TypeName Collections.ArrayList
      foreach($key in ($roleNumber).Keys){
        if($key -band $roles){
          [void]$roleList.Add($roleNumber.$key)
        }
      }
      Write-Output $roleList
    }

    $adParameters=@{
      ErrorAction='Stop';
    }
    $adExchProperties=@(
      'msExchCurrentServerRoles',
      'networkAddress',
      'serialNumber',
      'msExchServerSite'
    )
  }
  Process{
    Try{
      $filter="objectCategory -eq 'msExchExchangeServer'"
      if($PSBoundParameters.ContainsKey('credential')){
        $adParameters.Add('Credential',$credential)
      }
      if($PSBoundParameters.ContainsKey('server')){
        $adParameters.Add('Server',$server)
      }
      $rootDse=Get-ADRootDse @adParameters
      $adParameters.Add('SearchBase',$rootDse.ConfigurationNamingContext)

      if($PSBoundParameters.ContainsKey('siteName')){
        Write-Verbose "Getting Site: $siteName"
        $site=Get-ADObject @adParameters `
          -Filter "ObjectClass -eq 'site' -and Name -eq '$siteName'"
        if(!$site){
          Write-Error "Site not found: [$siteName]" `
            -ErrorAction Stop
        }
        $filter="$filter -and msExchServerSite -eq '$($site.DistinguishedName)'"
      }
      $adParameters.Add('Filter',$filter)

      $exchServers=Get-ADObject @adParameters `
        -Properties $adExchProperties

      foreach($exServer in $exchServers){
        $roles=ConvertToExchangeRole -roles $exServer.msExchCurrentServerRoles

        $fqdn=($exServer.networkAddress | 
          Where-Object {$_ -like 'ncacn_ip_tcp:*'}).Split(':')[1]
        New-Object -TypeName PSObject -Property @{
          Name=$exServer.Name;
          DnsHostName=$fqdn;
          ExchangeVersion=$exServer.serialNumber[0];
          ServerRoles=$roles;
          DistinguishedName=$exServer.DistinguishedName;
          Site=$exServer.msExchServerSite;
        }
      }
    }Catch{
      Write-Error $_
    }
  }
  End{}
}

Function Connect-ExchangeServer {
  [CmdletBinding(DefaultParameterSetName='enumerate')]
  Param(
    [Parameter(Position=0,Mandatory=$true,ParameterSetName='name')]
    [String]$name,

    [Parameter(Position=0,ParameterSetName='Enumerate')]
    [Bool]$enumerate=$true,

    [Parameter(Position=1)]
    [Management.Automation.PSCredential]$credential,

    [Parameter(Position=2)]
    [String]$domainController,


    [Parameter(Position=3,ParameterSetName='Enumerate')]
    [String]$siteName
  )
  Begin{
    $adParameters=@{
      ErrorAction='Stop';
    }
  }
  Process{
    if($PSBoundParameters.ContainsKey('credential')){
      $adParameters.Add('credential',$credential)
    }
    if($PSBoundParameters.ContainsKey('domainController')){
      $adParameters.Add('server',$domainController)
    }
    if($PSBoundParameters.ContainsKey('siteName')){
      $adParameters.Add('siteName',$siteName)
    }
    Try{
      if($enumerate){
        Write-Verbose "Getting list of Exchange Servers"
        $exchServers=Get-ADExchangeServer @adParameters
      }else{
        $exchServers=New-Object -TypeName PSObject -Property @{
          DnsHostName=$name;
        }
      }
      $winrmParameters=@{
        'ErrorAction'='Stop';
      }

      $snParameters=@{
        'ErrorAction'='Stop';
        'ConfigurationName'='Microsoft.Exchange';
      }
      if($adParameters.credential){
        $snParameters.Add('Credential',$adParameters.Credential)
      }
      foreach($exServer in $exchServers){
        Write-Verbose "Testing WinRm: $($exServer.DnsHostName)"
        $winrm=Test-WSMan @winrmParameters `
          -ComputerName $exServer.DnsHostName
        if($winrm){
          Write-Verbose "Connecting to: $($exServer.DnsHostName)"
          $exSn=New-PSSession @snParameters `
            -ConnectionUri "http://$($exServer.DnsHostName)/powershell"
        }
        return $exSn
      }
    }Catch{
      Write-Error $_
    }
  }
  End{}
}
```
