---
layout: post
title: PowerShell - Sccm Install Example
description: Guide to installing SCCM with only PowerShell
---

After creating 5 posts about automating Sccm primary site installs, 
I think an example bringing everything together is useful. 

<style>
table {
  border-collapse: collapse;
  width: 350px;
}

td, th {
  text-align: left;
  padding: 4px;
}

tr:nth-child(even) {
  background-color: #ececec;
}
</style>

#### Sccm Automation Posts:

<table>
  <tr>
    <td>
      <a href="{{ site.baseurl }}/PowerShell-SCCM-Offline-PreRequisites/">Post 1</a>
    </td>
    <td>Download Prereqs</td>
  </tr>

  <tr padding>
    <td>
      <a href="{{ site.baseurl }}/PowerShell-SCCM-Offline-PreRequisites-Install/">Post 2</a>
    </td>
    <td>Install Prereqs</td>
  </tr>

  <tr>
    <td>
      <a href="{{ site.baseurl }}/PowerShell-Sccm-AD-PreRequisites/">Post 3</a>
    </td>
    <td>AD Prereqs</td>
  </tr>

  <tr>
    <td>
      <a href="{{ site.baseurl }}/PowerShell-Sccm-PreRequisites-SQL/">Post 4</a>
    </td>
    <td>Sql Install</td>
  </tr>

  <tr>
    <td>
      <a href="{{ site.baseurl }}/PowerShell-Sccm-Primary-Site-Install/">Post 5</a>
    </td>
    <td>Sccm Primary Site Install</td>
  </tr>
</table>

### The Environment
----

<p>
  This is a lab environment with the following servers:
</p>

<table>
  <tr>
    <th><strong>Name</strong></th>
    <th><strong>Role</strong></th>
  </tr>

  <tr>
    <td>dc1</td>
    <td>Domain Controller</td>
  </tr>

  <tr>
    <td>cm1</td>
    <td>Will be the Sccm Primary Site Server</td>
  </tr>

  <tr>
    <td>Sql</td>
    <td>Will be the Sql server for the Sccm DB</td>
  </tr>
</table>

<p>
  All servers are Windows Server 2016. 
</p>
<p>
  You can install Sql on the cm1 server to save building another VM.
</p>

### Download
----

<p>
  In my environment, none of my servers have internet access, 
  so all my downloads will done on my host machine and copied to the VMs.
</p>

First, I have downloaded the latest ADK and Windows PE Add-on for the ADK.
Both can be found [here](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install).

<p>
  I have mounted the Sccm Installation ISO image to my host machine. 
  It is mounted on Drive E:.
</p>

Lastly, I have copied my script from [Post 1]({{ site.baseurl }}/PowerShell-ActiveDirectory-Exchange-Part1/) and saved it as SccmPreReqDl.ps1 on my host machine.

```powershell
.\SccmPreReqDl.ps1 -sccmInstallSource E:\ -prereqTargetPath C:\temp\Sccm -Verbose
```

<p>
  It should look something like this while running the script:
</p>

 ![_config.yml]({{ site.baseurl }}/images/SccmPreReqDl.png)

<p>
  You should see something like this folder structure after it has completed:
</p>

```powershell
ls C:\temp\Sccm

    Directory: C:\temp\Sccm

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----         2/2/2019   4:45 PM                adk
d-----         2/2/2019   5:08 PM                adkPeAddon
d-----         2/2/2019   4:57 PM                prereq
```

#### Copy files to Hyper-V VM from Host

<p>
  You only need to follow this section if you are downloading on your host, 
  and are running Hyper-V.
</p>

<p>
  Now I need to copy the folder C:\temp\Sccm to my cm1 server.
  If your host is running Hyper-V and you want to copy this via PowerShell, 
  you can do something like the following.
</p>

```powershell
# On Hyper-V host, enable 'Guest Service Interface' on target VM
Enable-VMIntegrationService -VMName cm1 -Name 'Guest*'

# Compress-Archive only available on PowerShell v5 and up
Compress-Archive -Path C:\temp\Sccm\ -DestinationPath C:\temp\SccmPreReq.zip

# Copy zip file to VM
Copy-VMFile -VMName cm1 `
  -FileSource Host `
  -SourcePath 'C:\temp\SccmPreReq.zip' `
  -DestinationPath 'C:\temp\Sccm\SccmPreReq.zip' `
  -CreateFullPath
```

<p>
Now on the VM:
</p>

```powershell
# Expand-Archive only available on PowerShell v5 and up
Expand-Archive -Path C:\temp\Sccm\SccmPreReq.zip -Destination C:\temp\Sccm
```


### Sccm PreReq Install
----

I have mounted the Windows Server 2016 ISO to the D:\ drive on cm1, 
and saved the script from 
[Post 2]({{ site.baseurl }}/PowerShell-SCCM-Offline-PreRequisites-Install/) 
to the cm1 server as SccmPreReqInstall.ps1.

```powershell
.\SccmPreReqInstall.ps1 -prereqSourcePath C:\temp\Sccm -windowsMediaSourcePath D:\sources\sxs -Verbose
```

<p>
  After this script has finished, 
  you should have all the necessary Windows Features, 
  and all the required ADK components for the Sccm install.
</p>

### Sccm AD PreReqs
----

I have saved the script from 
[Post 3]({{ site.baseurl }}/PowerShell-Sccm-AD-PreRequisites/) 
on the cm1 server as SccmADPreReq.ps1. 
I have mounted the Sccm Installation ISO to the D:\ drive of the cm1 server.
<p>
  I am logged into the cm1 server as a user with membership to 
  the Schema Admins, and the Domain Admins security groups.
</p>

```powershell
.\SccmADPrereq.ps1 -computerName $ENV:COMPUTERNAME -Verbose

# Extend the Active Directory schema
D:\SMSSETUP\BIN\X64\ExtADSch.exe
```

### Sql Install
----

<p>
  The Sql Server installation ISO image has been mounted to the D:\ drive. 
  I have created a service account in AD, 
  that I will use for the Sql Agent and Sql Server services. 
  Those services are not permitted to run as a local account in an 
  Sccm installation.
</p>

<p>
  Since I have opted to use Desired State Configuration (DSC) 
  to automate the Sql install, 
  you will need to download the SqlServerDsc resource from 
  the PowerShell Gallery. 
  I have also setup a certificate on the Sql server that the 
  Dsc local configuration manager will use to encrypt my passwords. 
</p>

<p>
  If your Sql server has internet access:
</p>

```powershell
Install-Module -Name SqlServerDsc
```

<p>
  If your Sql server does not have internet access, 
  run the following command on a computer with internet access: 
</p>

```powershell
Save-Module -Name SqlSeverDsc -Path C:\temp\

# Copy the folder to the Sql server to the PS Modules folder
Copy-Item -Path C:\temp\SqlServerDsc `
  -Recurse `
  -Destination "\\Sql\c$\Program Files\WindowsPowerShell\Modules"
```

#### DSC Configuration
----

Copy the 'Configuration' from 
[Part 4]({{ site.baseurl }}/PowerShell-Sccm-PreRequisites-SQL/) 
Directly into an Admin PowerShell window on your Sql server.

Create your DSC configuration data.
Reference the [Sql post]({{ site.baseurl }}/PowerShell-Sccm-PreRequisites-SQL/) 
 for more information on this if required:


```powershell
$config = @{
  AllNodes = @(
    @{ 
      NodeName = $ENV:COMPUTERNAME 
      # Path to your certificate to use for encryption
      CertificateFile = 'C:\cert\cert.cer';
      
      # This will need to match your Certificate
      Thumbprint = '4B1EF9E6C194098257E93120A4A39DA853F23434'
    }
  )
}
```
<p>
  Now to generate your Sql Mof file:
</p>

```powershell
# Service account created for running Sql server services
$svcCred=Get-Credential

$sccmAdmin='codeAndKeep\cmAdmin'

$installDir='E:\Program Files\Microsoft SQL Server'
$installx86Dir='E:\Program Files (x86)\Microsoft SQL Server'


SccmSqlInstallation -OutputPath C:\temp\sqlInstall `
  -ConfigurationData $config `
  -computerName $ENV:COMPUTERNAME `
  -sqlSourceFiles 'D:\' `
  -features "SQLENGINE,RS" `
  -sqlSvcCredential $svcCred `
  -agentSvcCredential $svcCred `
  -sysAdminAccounts $sccmAdmin `
  -instanceDir $installDir `
  -dataDir $installDir `
  -sharedDir $installDir `
  -sharedWOWDir $installx86Dir
```

<p>
  Now start the configuration:
</p>

```powershell
Set-DscLocalConfigurationManager -Path C:\temp\sqlInstall

Start-DscConfiguration -Path C:\temp\sqlInstall -Wait -Force -Verbose
```

<p>
  Lastly, if you are installing Sql on a separate server, 
  you will need to add the Sccm server as an admin on the Sql server.
</p>

```powershell
Add-LocalGroupMember -Name Administrators -Member 'codeAndKeep\cm1$'
```

### The Sccm Install
----

<p>
  I have mounted the SCCM installation ISO to the cm1 server on the D:\ drive. 
  I have also modified my Sccm install configuration file 
  to what you see below.
</p>

For more information on this part, check out 
[Post 5 ]({{ site.baseurl }}/PowerShell-Sccm-Primary-Site-Install/).

```
[Identification]
Action=InstallPrimarySite

[Options]
ProductID=EVAL
SiteCode=CDE
SiteName=Code And Keep Managing Computers
SMSInstallDir=E:\Program Files\Microsoft Configuration Manager
SDKServer=cm1.codeAndKeep.com
RoleCommunicationProtocol=HTTPorHTTPS
ClientsUsePKICertificate=1
PrerequisiteComp=1
PrerequisitePath=C:\temp\sccm\prereq
MobileDeviceLanguage=0
ManagementPoint=cm1.codeAndKeep.com
ManagementPointProtocol=HTTP
DistributionPoint=cm1.codeAndKeep.com
DistributionPointProtocol=HTTP
DistributionPointInstallIIS=0
AdminConsole=1
JoinCEIP=0

[SQLConfigOptions]
SQLServerName=sql.codeAndKeep.com
DatabaseName=CM_CDE
SQLSSBPort=4022
SQLDataFilePath=E:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\DATA
SQLLogFilePath=E:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\DATA

[CloudConnectorOptions]
CloudConnector=0
CloudConnectorServer=cm1.codeAndKeep.com
UseProxy=0
ProxyName=
ProxyPort=

[SystemCenterOptions]

[HierarchyExpansionOption]
```

<p>
  The SQLDataFilePath and SQLLogFilePath will be in a different location 
  depending on your Sql version and/or instance name.
</p>
<p>
  I have saved this file as SccmSetupConfig.ini under C:\temp. 
  If you haven't rebooted the cm1 server since installing the ADK/prereqs, 
  I would recommend doing so before installing.
</p>


#### Install
----

```powershell
D:\SMSSETUP\BIN\X64\setup.exe /SCRIPT C:\temp\SccmSetupConfig.ini

# check the setup log for errors
Get-Content -Path C:\ConfigMgrSetup.log -Wait
```

<p>
  After all this you should have a fully functional Sccm 
  Primary Site server installed in your lab environment.
</p>
