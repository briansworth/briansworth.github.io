---
layout: post
title: PowerShell - Automate SCCM PreReq Downloads
description: Script to download ADK and SCCM prerequisites
---

<p>
  I have recently started working with SCCM, 
  which means building (and rebuilding) lab environments for it. 
  I have found that just getting all the prerequisites installed can be 
  tedious and time consuming. 
  Even more so when your SCCM server is not connected to the internet.
</p>

### The Problem
----

<p>
  The SCCM Install requires prerequisites that, by default, 
  require internet connectivity.  
  Specifically there is a step in the SCCM Installation wizard that 
  asks you where to download your SCCM prereq files.  
  Additionally there is the ADK (and more recently the ADK WinPE Addon) 
  that require downloading their components from the internet.
</p>

### The Solution
----

<p>
  I have created a script that will be able to do the downloads 
  to a central folder of your choosing.
  Specifically, it will download the SCCM PreReq files, 
  the ADK files, and if required, the ADK WinPE Addon files.
</p>

#### What you need
----

<p>
  I know it's ridiculous that there are prereqs for downloading prereqs, 
  but it is what it is.
  You will need:
  <ol type="1">
    <li>SCCM installation media</li>
    <ul>
      <li>ISO must be mounted to a drive or extracted to a folder</li>
    </ul>
    <li>ADKSetup.exe</li>
    <ul>
      <li>Downloaded from Microsoft</li>
    </ul>
      <li>ADKWinPeSetup.exe</li>
    <ul>
      <li>Optional: The script will tell you if it is needed</li>
    </ul>
  </ol>
</p>
<p>
  With your environment ready, you can run the script. 
  Make sure you execute the script from the directory containing the 
  adksetup.exe file (and adkwinpesetup.exe file if required).
</p>

### Full Script
----

<p>
  I would recommend copying all the code below, 
  pasting it into a text editor, 
  then saving it as a ps1 file (SccmPreReqDl.ps1 for example).
</p>
*NOTE: There will be some examples below the code here*
```powershell
[CmdletBinding()]
Param(
  [Parameter(Mandatory=$true,Position=0)]
  [String]$sccmInstallSource,

  [Parameter(Position=1)]
  [String]$prereqTargetPath="$ENV:SYSTEMDRIVE\temp\sccm"
)
Try{
  $sccmSetupDlFilePath=Join-Path -Path $sccmInstallSource `
    -ChildPath "SMSSETUP\BIN\X64\setupdl.exe" `
    -ErrorAction Stop
  
  $tempSccmPreReqPath="$prereqTargetPath\prereq"
  $tempAdkPath="$prereqTargetPath\adk"
  $tempAdkWinPEPath="$prereqTargetPath\adkPeAddon"

  if(!(Test-Path -Path $sccmSetupDlFilePath)){
    $setupDlNotFound=[String]::Format(
      "File: {0} not found. Ensure {1} {2}",
      $sccmSetupDlFilePath,
      "that the SCCM Installation Media location specified is correct",
      "[$sccmInstallSource]"
    )
    Write-Error $setupDlNotFound -ErrorAction Stop
  }

  if(!(Test-Path -Path .\adksetup.exe)){
    $adkNotFound=[String]::Format(
      "File: {0} not found. Please {1} '{0}'",
      "adksetup.exe",
      "navigate to the directory containing"
    )
    Write-Error $adkNotFound -ErrorAction Stop
  }

  [version]$adkVersionNoPE='10.1.17763.1'

  $adk=Get-Item -Path .\adksetup.exe -ErrorAction Stop
  [version]$adkVersion=$adk.VersionInfo.FileVersion

  $requirePEAddon=$false
  if($adkVersion -ge $adkVersionNoPE){
    $requirePEAddon=$true

    $adkPeWarn=[String]::Format(
      "The adksetup.exe version you have {0}. {1}. {2}, {3}. {4}",
      "does not contain the WinPE Feature that SCCM requires",
      "You will need to download the AdkWinPE add-on for this AdkVersion",
      "Once you have it downloaded",
      "please copy it to this file location and try again",
      "Ignore this warning if you have already done so."
    )
    Write-Warning $adkPeWarn

    $adkPeFileExist=Test-Path -Path .\adkwinpesetup.exe
    if(!$adkPeFileExist){
      $adkPeFileNotFound=[String]::Format(
        "File: {0} not found. Please {1} '{0}'",
        "adkwinpesetup.exe",
        "navigate to the directory containing"
      )
      Write-Error $adkPeFileNotFound -ErrorAction Stop
    }else{
      if(!(Test-Path -Path $tempAdkWinPEPath)){
        mkdir $tempAdkWinPEPath -ErrorAction Stop | Out-Null
      }
    }
  } # End adk version check


  Function WaitForProcessEnd {
    [CmdletBinding()]
    Param(
      [String]$processName,

      [String]$msg
    )
    $processRunning=$true
    Write-Host $msg -NoNewline
    Do{
      Write-Host '.' -NoNewline
      Start-Sleep -Seconds 30
      $process=Get-Process -Name $processName `
        -ErrorAction SilentlyContinue
      if(!$process){
        $processRunning=$false
      }
    }While($processRunning)
  }

  Write-Verbose "Creating downloads folder structure [$prereqTargetPath]"
  $prereqTargetPath, $tempSccmPreReqPath, $tempAdkPath | Foreach { 
    if(!(Test-Path -Path $_)){
      mkdir $_ -ErrorAction Stop | Out-Null
    }
  }

  Write-Verbose "Downloading SCCM PreReq files [$tempSccmPreReqPath]"
  & $sccmSetupDlFilePath $tempSccmPreReqPath

  Write-Verbose "Downloading ADK Setup files [$tempAdkPath]"
  .\adksetup.exe /layout $tempAdkPath /q

  WaitForProcessEnd -processName adksetup `
    -msg "Downloading ADK... Will take some time"

  if($requirePEAddon){
    Write-Verbose "Downloading ADK PE Addon files [$tempAdkWinPEPath]"
    .\adkwinpesetup.exe /layout $tempAdkWinPEPath /q

    WaitForProcessEnd -processName adkwinpesetup `
      -msg "Downloading ADK WinPE addon... Will take some time"
  }
  WaitForProcessEnd -processName setupdl `
    -msg "Waiting for SCCM PreReq download to finish"

  Write-Host "PreReq Download completed. Check folder [$prereqTargetPath]"
}Catch{
  Write-Error $_
}
```

#### Example 1

```powershell
.\SccmPreReqDl.ps1 -sccmInstallSource D:\ -prereqTargetPath C:\temp\sccmPreReqs
```
<p>
  Description: This will download all files to the 'C:\temp\sccmPreReqs 
  folder.  The SCCM ISO file has been mounted to the D:\ drive, 
  so we point the -sccmInstallSource parameter to that drive letter.
</p>

#### Example 2

```powershell
.\SccmPreReqDl.ps1 -sccmInstallSource C:\Temp\SccmExtracted -Verbose
```
<p>
  Description: If you have extracted the SCCM ISO file to your C:\Temp drive 
  you can specify the path to the folder that contains the Installation media. 
  This will use the default location for the downloaded files.
</p>


### What now
----

I will be writing a 
[Part 2]({{ site.baseurl }}/PowerShell-SCCM-Offline-PreRequisites/)
to this post that can handle the install.  
See below if you want to do it yourself.
<p>
  In short; you just need to copy the folder over to the SCCM server. 
  Once there, you can run the adksetup.exe file from the adk folder. 
  and run the adkwinpesetup.exe file from the corresponding folder if required.
  Then during the SCCM installation wizard, 
  choose the 'prereq' folder when asked to choose a folder with the prereq files.
</p>
