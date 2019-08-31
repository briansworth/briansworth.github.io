---
layout: post
title: PowerShell - SCCM Configure SUP
description: Install and configure the Software Update Point role in SCCM with PowerShell
---

If you have followed my guide for 
[automating the Sccm primary site server install]({{ site.baseurl }}/PowerShell-Sccm-Install-Example/), 
you have a quick way to deploy SCCM. 
Now it's time to automate the configuration. 
In this post, the Software Update Point (SUP).

You can jump ahead in the post for a script to configure this, 
or follow along on the journey with me.

### Prerequisites
---

You should have a SCCM primary site server installed and running.
If you want to setup the SUP on a remote server, 
it will need to be up and running. 

You should also have Wsus configured on the server you want to use
for your Software Update Point. I have created a script to accomplish this,
(available on [GitHub](https://github.com/briansworth/Install-Sccm/blob/master/Install-SccmWsus.ps1))
along with a small 
[post]({{ site.baseurl }}/PowerShell-Sccm-Wsus-SUP-Install) to explain the process in more detail.


### The Journey
----

I will be deploying the SUP on the primary site server in this post.

<p>
  The first thing we need to do is import the ConfigurationManager 
  PowerShell module. 
  It isn't in the normal paths available in the $ENV:PSModulePath variable. 
  There is a different environment variable that should be available though, 
  $ENV:SMS_ADMIN_UI_PATH. 
</p>

<p>
  Once we have the module imported, we can start interacting with Sccm via 
  PowerShell.
</p>

```powershell
# You should be able to import the ConfigMgr PowerShell module like so
Import-Module "$($ENV:SMS_ADMIN_UI_PATH)\..\ConfigurationManager.psd1"

# Now you should have a CMSite PSDrive to use with your ConfigMgr module
$smsDrive = Get-PSDrive -PSProvider CMSite
cd "$($smsDrive.Name):\"
```

<p>
  Now we can check what roles are currently installed.
</p>

```powershell
# Get your CM Site
$site = Get-CMSite

# Check the roles currently installed
$preRoles = Get-CMSiteRole -SiteCode $site.SiteCode
$preRoles | Select RoleName
```

Now we can add our SUP role to Sccm.
You will need the FQDN of the server your want to deploy it to first. 
In my case, I will be using the local Primary Site Server.

*Note: If you are using a remote server for your SUP, 
you will need to add it as a CMSiteSystemServer. 
Use the 'New-CMSiteSystemServer' cmdlet for this before proceeding.*

```powershell
# If you are not using the local Sccm server change computer name accordingly
$dnsHostName = [net.dns]::GetHostEntry($ENV:COMPUTERNAME).HostName
Add-CMSoftwareUpdatePoint -SiteSystemServerName $dnsHostName `
  -SiteCode $site.SiteCode

Get-CMSiteRole -SiteCode $site.SiteCode | Select RoleName
```

The SUP role is there, but it isn't configured yet. 
You can configure it completely with PowerShell, 
although it isn't as simple as the GUI.
Some basic configuration is shown below.

```powershell
# Create new sync schedule
$supSchedule = New-CMSchedule  -Start 0:00:00 -RecurInterval Days -RecurCount 1

# Remove all default languages except english
$removeLang = @(
 'French',
 'German',
 'Chinese (Simplified, PRC)',
 'Russian',
 'Japanese'
)

# The RemoveProductFamily param will remove all default products
# You may want to keep some or add some default values
$supParameters = @{
 'SiteCode' = $site.SiteCode;
 'RemoveLanguageUpdateFile' = $removeLang;
 'RemoveLanguageSummaryDetail' = $removeLang;
 'ImmediatelyExpireSupersedence' = $true;
 'EnableCallWsusCleanupWizard' = $true;
 'Schedule' = $supSchedule;
 'RemoveProductFamily' = @('Office', 'Windows');
 'AddUpdateClassification' = 'Critical Updates';
 }

Set-CMSoftwareUpdatePointComponent @supParameters

Sync-CMSoftwareUpdate -FullSync $true
```

It may take some time for the Sync to complete. 
You also may need to reboot the computer to get it working.
You can check the following logs to track progress / errors:
  - wsyncmgr.log
  - WCM.log
You can also run the following command to see if it was successful.

```powershell
Get-CMSoftwareUpdateSyncStatus
```
After the sync has completed successfully, 
there should be many more product categories available.
I will add the Server 2019 and Sql Server 2017 products below:

```powershell
Set-CMSoftwareUpdatePointComponent -SiteCode $site.SiteCode `
  -AddProduct 'Windows Server 2019', 'Microsoft SQL Server 2017'

Sync-CMSoftwareUpdate -FullSync $true
```

Now after waiting a while (10 minutes or more), 
you should be able to see some software updates show up.

```powershell
Get-CMSoftwareUpdate
```

#### Review
----

So that is all there is to it. 
If you are seeing errors in the WCM.log file, 
you can try rebooting your server and checking again. 
Alternatively,
you may need to look into the error message more to find out the issue.


As usual, there is a script below that can do the whole thing for you.

### The Script
----

As promised, the script below will be able to configure a local or remote SUP. 
It is currently configured to add the 
'Windows Server 2019' product after the initial sync has been run. 
Feel free to change this yourself, 
but there is no validation in the script if the product is invalid.
If the sync times out, try rebooting SCCM, and check the logs mentioned above.

```powershell
[CmdletBinding()]
Param(
  [Parameter(Position=0)]
  [String]$SiteSystemServerName = $ENV:COMPUTERNAME,

  [Parameter(Position=1)]
  [ValidatePattern("^[a-zA-Z\d]{3}$")]
  [String]$SiteCode,

  [Parameter(Position=2)]
  [ValidatePattern("^\w+\\\w+$")]
  [String]$WsusConnectionAccountName,

  [Parameter(Position=3)]
  [uint16]$HttpPort = 8530,

  [Parameter(Position=4)]
  [uint16]$HttpsPort = 8531
)
Try{
  # Modify to what products you want to sync on your sup 
  # after it has synced all products / classifications
  # Be careful what you add, as there is no validation if these are valid
  $productsToAddAfterSync = @(
    'Windows Server 2019',
    'Microsoft SQL Server 2017'
  )

  ### DNS Resolution Validation ###
  Write-Verbose "Verifying CMSiteServer [$SiteSystemServerName] DNS Entry"
  $cmSiteSystemIsLocalHost = $false
  $cmServerDNSEntry = [Net.DNS]::GetHostEntry($SiteSystemServerName)

  $localHostDNSEntry = [Net.DNS]::GetHostEntry($ENV:COMPUTERNAME)

  if($localHostDNSEntry.HostName -eq $cmServerDNSEntry.HostName){
    Write-Verbose "CMSiteSystemServer is localhost [$ENV:COMPUTERNAME]"
    $cmSiteSystemIsLocalHost = $true
  }
  ### END DNS Resolution Validation ###


  ### ConfigurationManager PS Module Validation ###
  $cmModule = Get-Module -Name ConfigurationManager `
    -ErrorAction SilentlyContinue

  if(!$cmModule){
    Write-Verbose "Searching for ConfigurationManager PowerShell Module"
    $modulePaths = @()
    $adminUIPath = Get-ItemPropertyValue -Name 'UI Installation Directory' `
      -Path 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\ConfigMgr10\Setup\' `
      -ErrorAction Continue

    if($adminUIPath){
      $modulePaths += Join-Path -Path $adminUIPath `
        -ChildPath "bin\ConfigurationManager.psd1"
    }

    $modulePaths += "$($ENV:SMS_ADMIN_UI_PATH)\..\ConfigurationManager.psd1"

    foreach($modulePath in $modulePaths){
      $moduleExist = Test-Path -Path $modulePath
      if($moduleExist){
        break
      }
    }

    Import-Module -Name $modulePath -ErrorAction Stop
  }
  ### END ConfigurationManager PS Module Validation ###


  ### Setup environment for running CM Cmdlets
  Write-Verbose "Locating CMSite PSDrive"
  $smsDrive = Get-PSDrive -PSProvider CMSite -ErrorAction SilentlyContinue
  if($SiteCode -and $SiteCode -ne $smsDrive.Name){
    $smsDrive = Get-PSDrive -Name $SiteCode `
      -PSProvider CMSite `
      -ErrorAction Stop
  }
  Push-Location -Path "$($smsDrive.Name):\" -ErrorAction Stop

  Write-Verbose "Retreiving CMSite info: [$($smsDrive.Name)]"
  $site = Get-CMSite -SiteCode $smsDrive.Name -ErrorAction Stop
}Catch [Management.Automation.MethodInvocationException]{
  Write-Error "Error resolving dns name [$SiteSystemServerName]: $($_.Exception)"
  Pop-Location
  break
}Catch{
  Write-Error "Error setting up PowerShell environment: $($_.Exception)"
  Pop-Location
  break
}

### Begin to make configuration changes for Wsus / SUP server ###
Try{
  $siteSystem = Get-CMSiteSystemServer -SiteCode $site.SiteCode `
    -Name $cmServerDNSEntry.HostName

  if(!$siteSystem){
    Write-Verbose "[$($cmServerDNSEntry.HostName)] is not a CMSiteSystemServer"
    Write-Verbose "Adding [$($cmServerDNSEntry.HostName)] as CMSiteSystemServer"
    New-CMSiteSystemServer -SiteCode $site.SiteCode `
      -Name $cmServerDNSEntry.HostName `
      -ErrorAction Stop
  }

  $supParameters = @{
    'SiteSystemServerName' = $cmServerDNSEntry.HostName;
    'SiteCode' = $site.SiteCode;
    'WsusIisPort' = $HttpPort;
    'WsusIisSslPort' = $HttpsPort;
    'ErrorAction' = 'Stop';
  }

  if($PSBoundParameters.ContainsKey('WsusConnectionAccountName')){
    $wsusAccount = Get-CMAccount -UserName $WsusConnectionAccountName `
      -ErrorAction Stop

    if(!$wsusAccount){
      Write-Error "WSUS Access Account not found: [$WsusConnectionAccountName]" `
        -Category ObjectNotFound `
        -ErrorAction Stop
    }
    $supParameters.Add('ConnectionAccountUserName', $WsusConnectionAccountName)
  }

  $sup = Get-CMSoftwareUpdatePoint -SiteCode $site.SiteCode `
    -SiteSystemServerName $cmServerDNSEntry.HostName

  if($sup){
    Write-Verbose "SUP already exists, modifying with updated settings"
    $supParameters.Remove('ConnectionAccountUserName')
    if($PSBoundParameters.ContainsKey('WsusConnectionAccountName')){
      $supParameters.Add('WsusAccessAccount', $WsusConnectionAccountName)
    }
    Set-CMSoftwareUpdatePoint @supParameters
  }else{
    Write-Verbose "Adding Software Update Point to [$cmServerDNSEntry.HostName]"
    $sup = Add-CMSoftwareUpdatePoint @supParameters
  }

  ### Configure SUP ###
  $supSchedule = New-CMSchedule -Start 0:00:00 `
    -RecurInterval Days `
    -RecurCount 1

  $removeLang = @(
   'French',
   'German',
   'Chinese (Simplified, PRC)',
   'Russian',
   'Japanese'
  )

  $supComponentParameters = @{
   'SiteCode' = $site.SiteCode;
   'RemoveLanguageUpdateFile' = $removeLang;
   'RemoveLanguageSummaryDetail' = $removeLang;
   'ImmediatelyExpireSupersedence' = $true;
   'EnableCallWsusCleanupWizard' = $true;
   'Schedule' = $supSchedule;
   'RemoveProductFamily' = @('Office', 'Windows');
   'AddUpdateClassification' = 'Critical Updates';
   'DefaultWsusServer' = $cmServerDNSEntry.HostName;
   'ErrorAction' = 'Stop';
   }

  $beforeSync = Get-Date
  Set-CMSoftwareUpdatePointComponent @supComponentParameters

  Restart-Service -Name sms_executive
  Sync-CMSoftwareUpdate -FullSync $true

  ### Wait for Software Update Sync ###
  Start-Sleep -Seconds 300
  $syncStatus = Get-CMSoftwareUpdateSyncStatus
  Sync-CMSoftwareUpdate -FullSync $true

  Function Write-ProgressWheel {
    Param([int]$Seconds=60)
    $milliSecondCounter=0
    $milliSecondInterval=150
    $spinner = @('-', '\', '|', '/')
    
    while(($milliSecondCounter/1000) -lt $Seconds){
      foreach($char in $spinner){
        Write-Host -NoNewline "`r$char"
        Start-Sleep -Milliseconds $milliSecondInterval
        $milliSecondCounter+=$milliSecondInterval
      }
    }
  }

  $syncFail = $false
  $maxSeconds = (60*30)
  $endWait = $beforeSync.AddSeconds($maxSeconds)
  $intervalSeconds = 60

  Write-Host "  Waiting on Software Update Sync Cycle..."
  while($now -lt $endWait){
    $now = Get-Date
    $syncStatus = Get-CMSoftwareUpdateSyncStatus
    
    if($now -ge $endWait){
      $syncFail = $true
      $errorMsg = [String]::Format(
        "SUP Sync time limit hit [Mins: $($maxSeconds/60)]! {0}",
        'Check the wsyncmgr.log for more details'
      )
      Write-Error -Message $errorMsg
      break
    }

    if($null -eq $syncStatus.LastSuccessfulSyncTime -or 
       $syncStatus.LastSuccessfulSyncTime -lt $beforeSync)
    {
      Write-ProgressWheel -Seconds $intervalSeconds
      continue
    }else{
      Write-Host ""
      break
    }
  }

  if($syncFail){
    $wsusSyncError = [String]::Format(
      "An error has occurred waiting for {0}. {1}, {2}. {3}, {4}.",
      "the Software update sync to complete",
      "Consult the Wsus Site Component logs",
      "and the wsyncmgr.log for more details",
      "If this install is using a SQL Database for WSUS",
      "you may need to configure an proxy account on the SUP"
    )
    Write-Error -Message $syncFail -ErrorAction Stop
  }

  Set-CMSoftwareUpdatePointComponent -SiteCode $site.SiteCode `
    -AddProduct 'Windows Server 2019'
    
}Catch{
  $supErrorMsg = ""
  Write-Error -Exception $_.Exception `
    -Category $_.CategoryInfo.Category
}
```
