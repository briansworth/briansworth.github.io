---
layout: post
title: Building an Exchange 2016 Lab on Server 2016
---

<p>
  I have worked with Exchange quite a bit, and I wanted to take a look at the
  newest version on the newest Server OS.  
  Instead of figuring it out everytime I need to build an Exchange lab,
  I figured it would be a good idea to automate it as much as possible,
  and document with code.
</p>

### The Plan
----
I will be building 2 VMs on Hyper-V:
* 1 Domain Controller - Server core 2016
* 1 Exchange Server - Server 2016 (with GUI)

<p>
  I will be building the DC from a sysprepped template.
  I will be building the Exchange server from an ISO.
</p>

### Building the VMs
-----

This code has been tested on Win8 and Win10, on PowerShell v5.1

**Building the Domain Controller VM**
```powershell
# make domain controller from syspreped vhd file
$templatepath="V:\Templates\core2016.vhdx"
$vmname='exDC01'
$vmpath="V:\VMs\$vmname"
$vhdpath="$vmpath\$vmname.vhdx"
$switchname='external vswitch'
[int16]$generation=2
[int64]$memory=512MB
[int16]$vcpu=2

[bool]$isdir=test-path -path $vmpath
if(!$isdir){
  new-item -path $vmpath -type directory | out-null
}
[String]$vhdFileName=Split-Path -Leaf -Path $templatepath
[String]$tpath=Split-Path -Path $templatepath -Parent
Robocopy.exe $tpath $vmpath $vhdFileName

[String]$vhdNewPath="$vmpath\$vhdFileName"
set-itemproperty -path $vhdNewPath -name isreadonly -value $false
[String]$vhdNewName=Split-Path -Path $vhdpath  -Leaf
Rename-Item -Path $vhdNewPath -NewName $vhdNewName

New-VM -vmname $vmName `
  -Path $vmPath `
  -VHDPath $vhdPath `
  -MemoryStartupBytes $memory `
  -SwitchName $switchName `
  -Generation $generation `
  -Verbose
Set-VM -VMName $vmName -ProcessorCount $vCPU 

# Start and connect to new VM
Start-VM -VMName $vmName 
vmconnect localhost $vmName
```

**Building the Exchange server VM**
```powershell
$vmName='exEX01'
$vSwitchName='External vSwitch'
$vmPath="V:\VMs\$vmName"
$vhdPath="$vmPath\$($vmName).vhdx"
$isoPath="V:\ISOs\Windows\server\server2016.iso"
[int16]$generation=2
[int64]$memory=4Gb
[int64]$vhdSize=50Gb

New-VM -VMName $vmName `
  -SwitchName $vSwitchName `
  -Path $vmPath `
  -NewVhdPath $vhdPath `
  -NewVhdSizeBytes $vhdSize `
  -MemoryStartupBytes $memory `
  -Generation $generation
Set-VM -VMName $vmName -ProcessorCount 2
# if no dvd drive, add one
if(! (Get-VMDvdDrive -VMName $vmName)){
  Add-VMDvdDrive -VMName $vmName -Path $isoPath
}else{
  Set-VMDvdDrive -VMName $vmName -Path $isoPath
}

# Create additional drive for the Exchange Database/Logs
$mxVhdPath=Join-Path -Path $vmpath -ChildPath mxdb1.vhdx
New-VHD -Path $mxVhdPath -SizeBytes 10Gb -Dynamic | Out-Null
Add-VMScsiController -VMName $vmName
Add-VMHardDiskDrive -VMName $vmName `
  -ControllerNumber 1 `
  -ControllerLocation 0 `
  -ControllerType SCSI `
  -Path $mxVhdPath `
  -ErrorAction Stop

# Start and connect to the vm
Start-VM -VMName $vmName
vmconnect.exe localhost $vmName
# Complete the OS installation before continuing
```
*End of VM creation*
<br>

#### Setup and configure Domain Controller
----

Rename and reboot:
```
powershell
Rename-Computer -NewName 'exDC01' -Restart
```

After reboot:
```powershell
#start powershell
[String]$dcIP='10.0.0.5'
[String]$gateway='10.0.0.1'
[PSObject]$netAdapter=Get-NetAdapter -Name 'Ethernet*' 
$netAdapter | New-NetIPAddress -IPAddress $dcIP `
    -AddressFamily IPv4 `
    -DefaultGateway $gateway `
    -PrefixLength 24
$netAdapter | Set-DnsClientServerAddress -ServerAddresses 127.0.0.1,$gateway

Install-WindowsFeature -Name AD-Domain-Services
[Security.SecureString]$pass=ConvertTo-SecureString -String 'P@ssw0rd1' `
  -AsPlainText `
  -Force

# server will reboot after install
# This server will be the domain controller for a new AD Forest
[String]$dName='codeAndKeep.com'
[String]$netbios=$dName.Split('.')[0]
[Collections.Hashtable]$adForestParam=@{
    DomainMode='WinThreshold';
    ForestMode='WinThreshold';
    DomainName=$dName;
    DomainNetbiosName=$netbios;
    InstallDNS=$true;
    SafeModeAdministratorPassword=$pass;
    ErrorAction='Stop';
    Confirm=$false;
}
Install-ADDSForest @adForestParam
```
You should now have a new forest setup for testing.
Of course you will likely need to change the IP configuration to your network.

#### Setup Exchange server
----

Rename and reboot:

```powershell
# Start powershell
Rename-Computer -NewName exEX1 -Restart
```

After reboot, join the domain from above:
```powershell
[String]$exIP='10.10.0.6'

[String]$dcIP='10.10.0.5'
[String]$gateway='10.10.0.1'

[PSObject]$netAdapter=Get-NetAdapter -Name 'Ethernet*'
 $netAdapter | New-NetIPAddress -IPAddress $exIP `
    -AddressFamily IPv4 `
    -DefaultGateway $gateway `
    -PrefixLength 24
$netAdapter | Set-DnsClientServerAddress -ServerAddresses $dcIP

[String]$dName='codeAndKeep.com'
[String]$netbios=$dName.Split('.')[0]
[Security.SecureString]$pass=ConvertTo-SecureString -String 'P@ssw0rd1' `
  -AsPlainText `
  -Force
$cred=New-Object -TypeName PSCredential `
  -ArgumentList "$netbios\Administrator",$pass
Add-Computer -DomainName $dName -DomainCredential $cred -Restart
```

With the ActiveDirectory stuff mostly out of the way, 
we will now need to get all of the prerequisites setup for Exchange 2016.
There are 3 rather large prerequisites that need to be downloaded:
1. Exchange Server 2016 CU 4
  * CU 4 is needed on Server 2016 
  * Download [here](https://www.microsoft.com/en-us/download/details.aspx?id=54450)
2. Windows Update kb3206632
  * Download [here](https://www.catalog.update.microsoft.com/Search.aspx?q=kb3206632)
3. Unified Communications Managed API 4.0 Runtime (UCMA)
  * Download [here](https://www.microsoft.com/en-ca/download/details.aspx?id=34992)

All the above downloads would be useful to save on your Host machine,
in case you want to deploy this lab again in the future.
Otherwise you can install these on the Exchange server.

You can also download using PowerShell:
```powershell
# prereq windows update for Exchange CU 4
Invoke-WebRequest -Uri 'https://download.windowsupdate.com/d/msdownload/update/software/secu/2016/12/windows10.0-kb3206632-x64_b2e20b7e1aa65288007de21e88cd21c3ffb05110.msu' `
  -OutFile "$HOME\Downloads\windows10.0-kb3206632-x64_b2e20b7e1aa65288007de21e88cd21c3ffb05110.msu"

# Exchange 2016 CU4 because base Exchange 2016 can't install on server 2016
[String]$fileName='ExchangeServer2016-x64-cu4.iso'
Invoke-WebRequest -Uri "$baseuri/B/9/F/B9F59CF4-7C60-49EF-8A5B-8C2B7991FA86/$fileName" `
  -OutFile "~\Downloads\$fileName"

# UCMA 
[String]$baseuri='https://download.microsoft.com/download'
[String]$fileName='UcmaRuntimeSetup.exe'
Invoke-WebRequest -Uri "$baseuri/2/C/4/2C47A5C1-A1F3-4843-B9FE-84C0032C61EC/$fileName" `
  -OutFile "~\Downloads\$fileName"
```
Which ever method you choose; 
I have assumed these files all exist on the Exchange server in the Downloads folder
for the current user.

**Server pre-requisites:**
```powershell
[String[]]$exchangePreReqs=@(
  'NET-Framework-45-Features',
  'RPC-over-HTTP-proxy',
  'RSAT-Clustering',
  'RSAT-Clustering-CmdInterface',
  'RSAT-Clustering-Mgmt',
  'RSAT-Clustering-PowerShell',
  'Web-Mgmt-Console',
  'WAS-Process-Model',
  'Web-Asp-Net45',
  'Web-Basic-Auth',
  'Web-Client-Auth',
  'Web-Digest-Auth',
  'Web-Dir-Browsing',
  'Web-Dyn-Compression',
  'Web-Http-Errors',
  'Web-Http-Logging',
  'Web-Http-Redirect',
  'Web-Http-Tracing',
  'Web-ISAPI-Ext',
  'Web-ISAPI-Filter',
  'Web-Lgcy-Mgmt-Console',
  'Web-Metabase',
  'Web-Mgmt-Console',
  'Web-Mgmt-Service',
  'Web-Net-Ext45',
  'Web-Request-Monitor',
  'Web-Server',
  'Web-Stat-Compression',
  'Web-Static-Content',
  'Web-Windows-Auth',
  'Web-WMI',
  'Windows-Identity-Foundation',
  'RSAT-ADDS'
)
Install-WindowsFeature -Name $exchangePreReqs -ErrorAction Stop
``` 

Install downloaded pre-requistes:
```powershell
# UCMA Install
& "$HOME\Downloads\UcmaRuntimeSetup.exe"

# Windows update 
wusa.exe "$HOME\Downloads\windows10.0-kb3206632-x64_b2e20b7e1aa65288007de21e88cd21c3ffb05110.msu" /quiet /forcerestart
# This will install in the background and forcibly reboot after completion
```

#### Installing Exchange
----

After the pre-requisites from above have been installed,
the Exchange setup can finally begin.

```powershell
# Initialize and create new volume using added vhd file when creating vm
Initialize-Disk -Number 1 -PartitionStyle mbr -PassThru | 
  New-Partition -UseMaximumSize -DriveLetter M | 
    Format-Volume -FileSystem NTFS -NewFileSystemLabel MXDB1
    
[String]$fileName='ExchangeServer2016-x64-cu4.iso'
$vol=Mount-DiskImage -ImagePath "$HOME\Downloads\$fileName" -PassThru | Get-Volume

$setupPath="$($vol.DriveLetter):\setup.exe"
# Prepare AD Schema
$orgName='codeAndKeep.com'
& "$setupPath" /PrepareSchema /IAcceptExchangeServerLicenseTerms

# Prepare AD
$orgName=$orgName.Split('.')[0]
& "$setupPath" /PrepareAD /OrganizationName:$orgName /IAcceptExchangeServerLicenseTerms


# EXCHANGE INSTALL  -- This will take a while --
& "$setupPath" /Mode:Install /Role:Mailbox /DbFilePath:"M:\Database\mxdb01.edb" /MdbName:"mxdb01" /LogFolderPath:"M:\Logs" /IAcceptExchangeServerLicenseTerms

# Dismount iso used for Exchange Install
Dismount-DiskImage -ImagePath "$HOME\Downloads\$fileName"
```
The last command above will do the unattended installation for Exchange 2016.
It will install the Mailbox Database and Logs in the M drive.

A lot more automation could be built in to this, like using unattend files
to join the domain, or copying the Exchange prereq files on VM creation.
It's a start and it has worked for me several times now.

