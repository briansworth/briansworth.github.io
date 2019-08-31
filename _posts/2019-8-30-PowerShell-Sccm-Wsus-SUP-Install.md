---
layout: post
title: PowerShell - SCCM Configure WSUS for SUP
description: Install and configure WSUS for a SUP install
---

In this one, we install and configure Wsus on a server. 
This is a supplementary post to the 
[SCCM Configure SUP]({{ site.baseurl }}/PowerShell-Sccm-Configure-SUP/)
 post. 

As usual, you can skip to the bottom to grap a script to do all
of this for you.  
The script is also available on 
[GitHub](https://github.com/briansworth/Install-Sccm/blob/master/Install-SccmWsus.ps1). 
If you want to join me on the journey of discovery, keep on reading.

### The Journey
----

<p>
  A prerequisite for the SUP is Windows Server Update Services (WSUS). 
  The server that you are going configure as the SUP will need the 
  WSUS features installed, 
  and some basic configuration done before the SUP can be deployed. 
</p>

There are 2 different ways to configure the WSUS database:
1. Windows Internal Database (WID)
2. SQL Server Database

I will explore both options below, starting with the WID.

#### WSUS Windows Internal Database
----

<p>
  Using the WID is easy and a common deployment strategy. 
  You would not want to use the WID if you required a 
  load balanced WSUS solution. 
</p>

<p>
  The UpdateServices-WidDB feature is needed for this configuration.
</p>

```powershell
$updateFeatures = @(
  'UpdateServices-WidDB',
  'UpdateServices-Services',
  'UpdateServices-RSAT',
  'UpdateServices-API',
  'UpdateServices-UI'
)

Install-WindowsFeature -Name $updateFeatures
```

After installing the required roles, 
we need to configure it for proper use.

**Post feature install configuration:**

```powershell
# Create directory for WSUS storage
$wsusContentPath = 'C:\wsus'
New-Item -Path $wsusContentPath -ItemType Directory

# Configure WSUS to use the WID in your specified location
& 'C:\Program Files\Update Services\Tools\WsusUtil.exe' postinstall CONTENT_DIR=$wsusContentPath
```

<p>
  Wsus is now setup with a WID to work with your SUP using a WID.
</p>

#### WSUS SQL Database
----

<p>
  Since SCCM already utilizes a SQL server, 
  you can use the same one for WSUS. 
</p>

<p>
  The UpdateServices-DB feature is needed for this configuration.
</p>

```powershell
$updateFeatures = @(
  'UpdateServices-DB',
  'UpdateServices-Services',
  'UpdateServices-RSAT',
  'UpdateServices-API',
  'UpdateServices-UI'
)

Install-WindowsFeature -Name $updateFeatures
```

After installing the required roles, 
we need to configure it for proper use.

**Post feature install configuration:**

```powershell
# Create directory for WSUS storage
$wsusContentPath = 'C:\wsus'
New-Item -Path $wsusContentPath -ItemType Directory

# Populate Sql server/instance details
$sqlInstance = '' # Leave blank if default instance MSSQLSERVER
$sqlServerName = $ENV:COMPUTERNAME

$sqlInstanceName = "$sqlServerName\$sqlInstance"
$sqlInstanceName = $sqlInstanceName.TrimEnd('\')

# Configure WSUS to use the WID in your specified location
New-Alias -Name WsusUtil -Value 'C:\Program Files\Update Services\Tools\WsusUtil.exe'

WsusUtil postinstall SQL_INSTANCE_NAME=$sqlInstanceName CONTENT_DIR=$wsusContentPath
```
<p>
  Wsus is now setup, with a SQL DB, to work with your SUP using a SQL DB.
</p>

#### Review
-----

<p> 
  Armed with this new knowledge for installing Wsus with PowerShell, 
  we can come up with a more sophisticated script to do so. 
  As mentioned above, 
  both of these configurations are supported options for an SCCM SUP. 
  I usually go with the WID, as the installation is simpler.
</p>


### The Script
----

<p>
  The script below is capable of doing the Wsus install and configuration 
  remotely or locally.  
  I would recommend doing it locally if you are going with a SQL DB option.
  If you choose the WID, this should be just fine to run remotely. 
</p>

As stated at the start of the post, this script is also available on
[GitHub](https://github.com/briansworth/Install-Sccm/blob/master/Install-SccmWsus.ps1). 

```powershell
[CmdletBinding(DefaultParameterSetName='Wid')]
Param(
  [Parameter(Position=0,ParameterSetName='Wid')]
  [Parameter(Position=0,ParameterSetName='SqlDB')]
  [String]$ComputerName = $ENV:COMPUTERNAME,

  [Parameter(Position=1,ParameterSetName='Wid')]
  [Parameter(Position=1,ParameterSetName='SqlDB')]
  [String]$ContentPath,

  [Parameter(Position=2,Mandatory=$true,ParameterSetName='SqlDB')]
  [String]$SqlServerName,

  [Parameter(Position=3,ParameterSetName='SqlDB')]
  [String]$SqlInstanceName
)
Try{
  $localHostIsWsus = $false
  $localHostIsSql = $false
  $wsusRebootRequired = $false

  Write-Verbose "Verifying DNS resolution of [$ComputerName]"
  $computerDNS = [Net.Dns]::GetHostEntry($ComputerName)
  $localhostDNS = [Net.Dns]::GetHostEntry($ENV:COMPUTERNAME)

  if($PSCmdlet.ParameterSetName -eq 'SqlDB'){
    Write-Verbose "Verifying DNS resolution of SQL Server [$SqlServerName]"
    $sqlServerDNS = [Net.Dns]::GetHostEntry($SqlServerName)
    if($computerDNS.HostName -eq $localhostDNS.HostName){
      $localHostIsSql = $true
    }
  }

  if($computerDNS.HostName -eq $localhostDNS.HostName){
    Write-Verbose "Target computer is localhost [$($computerDNS.HostName)]"
    $localHostIsWsus = $true
  }else{
    Write-Verbose "Testing WinRM connectivity with [$ComputerName]"
    $wsusSession = New-PSSession -ComputerName $ComputerName `
      -ErrorAction Stop
  }

  $features = @(
    'UpdateServices-Services',
    'UpdateServices-WidDB',
    'UpdateServices-DB'
  )

  $featuresParameters = @{
    'Name' = $features;
    'ErrorAction' = 'Stop';
  }

  if(!$localHostIsWsus){
    $featuresParameters.Add('ComputerName', $ComputerName)
  }

  $usFeatures = Get-WindowsFeature @featuresParameters

  $usMainFeature = $usFeatures | Where-Object {$_.Name -eq 'UpdateServices-Services'}

  if($usMainFeature.Installed){
    Write-Verbose "Feature: [$($usMainFeature.Name)] is already installed"
    $widUsFeature = $usFeatures | Where-Object {$_.Name -eq 'UpdateServices-WidDB'}
    $sqlUsFeature = $usFeatures | Where-Object {$_.Name -eq 'UpdateServices-DB'}

    if($widUsFeature.Installed -and !$SqlServerName){
      Write-Verbose "WSUS Windows Internal DB feature already installed"

    }elseif($sqlUsFeature.Installed -and $SqlServerName){
      Write-Verbose "WSUS SQL DB feature already installed"

    }elseif($SqlServerName){
      Write-Verbose "Will install SQL DB for WSUS"
      $featuresParameters.Name = 'UpdateServices-Services', 'UpdateServices-DB'
      Install-WindowsFeature @featuresParameters -IncludeManagementTools

    }else{
      Write-Verbose "Will install Windows Internal DB for WSUS"
      $featuresParameters.Name = 'UpdateServices-Services', 'UpdateServices-WidDB'
      $installResult = Install-WindowsFeature @featuresParameters `
        -IncludeManagementTools

      if($installResult.RestartNeeded -ne 'No'){
        $wsusRebootRequired = $true
      }
    }

  }else{ ### WSUS Features not already installed
    if($SqlServerName){
      $featuresParameters.Name = 'UpdateServices-Services', 'UpdateServices-DB'
      $installResult = Install-WindowsFeature @featuresParameters `
        -IncludeManagementTools
    }else{
      $featuresParameters.Name = 'UpdateServices-Services', 'UpdateServices-WidDB'
      $installResult = Install-WindowsFeature @featuresParameters `
        -IncludeManagementTools
    }

    if($installResult.RestartNeeded -ne 'No'){
      $wsusRebootRequired = $true
    }
  }

  ### Validate the 'WSUS Post Install' state
  $usRegKey = [String]::Format(
    'HKLM:\SOFTWARE\Microsoft\Update Services\{0}',
    'Server\Setup\Installed Role Services'
  )

  if($wsusSession){
    $usState = Invoke-Command -Session $wsusSession `
      -ArgumentList $usRegKey `
      -ScriptBlock { Get-ItemProperty -Path $args[0] } `
      -ErrorAction Stop
  }else{
    $usState = Get-ItemProperty -Path $usRegKey -ErrorAction Stop
  }

  if($usState.'UpdateServices-Services' -eq 2){
    # Post install has completed
    Write-Verbose "WSUS Post install has been run previously"
    break
  }

  ### Create WsusUtil command for post install ###
  $stringScriptBlock = [String]::Format(
    '& "$ENV:PROGRAMFILES\Update Services\Tools\{0}" {1}',
    'WsusUtil.exe',
    'postinstall'
  )

  if($SqlServerName){
    $sqlServerInstanceName = "$($sqlServerDNS.HostName)\$SqlInstanceName"
    $sqlServerInstanceName = $sqlServerInstanceName.TrimEnd('\')

    $stringScriptBlock+=" SQL_INSTANCE_NAME=$sqlServerInstanceName" 
  }
  if($ContentPath){
    $stringScriptBlock+=" CONTENT_DIR=$ContentPath"
  }

  $wsusPostInstallScriptBlock=[ScriptBlock]::Create($stringScriptBlock)
  Write-Verbose "PostInstall command: [$wsusPostInstallScriptBlock]"

  if($localHostIsWsus){
    $wsusPostInstallResult = $wsusPostInstallScriptBlock.Invoke()
  }else{
    $wsusPostInstallResult =Invoke-Command -Session $wsusSession `
      -ScriptBlock $wsusPostInstallScriptBlock `
      -ErrorAction Stop
  }
  if($wsusRebootRequired){
    $rebootMsg = [String]::Format(
      'UpdateServices requires a reboot to complete setup Computer: {0}',
      "[$ComputerName]"
    )
    Write-Warning -Message $rebootMsg
  }

  if($wsusPostInstallResult[-1] -ne 'Post install has successfully completed'){
    $wsusErrorMsg = [String]::Format(
      "Wsus Post Install failed with the following message: {0}. {1}",
      "$($wsusPostInstallResult[-1])",
      "$($wsusPostInstallResult[0]) on server [$ComputerName]"
    )

    $anonLoginFail = [String]::Format(
      "Fatal Error: Login Failed for user {0}",
      "'NT AUTHORITY\ANONYMOUS LOGON'."
    )

    if($wsusPostInstallResult[-1] -eq $anonLoginFail -and !$localHostIsWsus){
      $doubleHopWarning = [String]::Format(
        "This error message is likely a result of {0} {1}. {2} {3}: {4}",  
        "a limitation in Kerberos authentication",
        'when using PowerShell remoting',
        "You should be able to directly login to [$ComputerName]",
        "and run the following command locally to resolve the issue",
        "[$wsusPostInstallScriptBlock]"
      )
    }

    Write-Error $wsusErrorMsg `
      -Category InvalidResult

    if($doubleHopWarning){
      Write-Warning -Message $doubleHopWarning
    }
  }
  # Raw output from the Wsus util post install command
  # Write-Output -InputObject $wsusPostInstallResult

}Catch{
  Write-Error -Exception $_.Exception `
    -Category $_.CategoryInfo.Category
}Finally{
  if($wsusSession){
    Remove-PSSession -Session $wsusSession
  }
}
```

#### Example 1

```powershell
PS> Install-SccmWsus -ContentPath 'D:\wsus'
```

This command will install Wsus using the Windows Internal DB.
The content path for Wsus will 'D:\wsus'

#### Example 2

```powershell
Install-SccmWsus -ComputerName SUP -ContentPath D:\wsus
```

This command will install Wsus using the Windows Internal DB.
It will install on the SUP server with the Wsus content path of 'D:\wsus'.

#### Example 3

```powershell
Install-SccmWsus -SqlServerName cmsql -ContentPath 'C:\wsus'
```

This command will install Wsus on the local computer, and configure it to
store updated on the Sql server 'cmsql'.
The local Wsus content will be stored in 'C:\wsus'.
