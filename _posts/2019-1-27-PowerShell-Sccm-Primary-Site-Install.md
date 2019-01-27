---
layout: post
title: PowerShell - Sccm Primary Site Install
---

<p>
The last step in the journey to automating a Sccm Primary Site server install.
</p>

This is a continuation of automating Sccm prerequisites:
- [Part 1](http://codeandkeep.com/PowerShell-SCCM-Offline-PreRequisites/)
- [Part 2]( http://codeandkeep.com/PowerShell-SCCM-Offline-PreRequisites-Install/)
- [Part 3](http://codeandkeep.com/PowerShell-Sccm-AD-PreRequisites/)
- [Part 4](http://codeandkeep.com/PowerShell-Sccm-AD-PreRequisites-SQL/)

<p>
  In this post, the final step.  
  Installing Sccm (Primary Site Server role). 
  If you have followed along from part 1 to now, 
  you will have downloaded all your prerequisites, 
  installed the prerequisites, 
  completed the AD schema extension and system management container creation, 
  and finally installed Sql.
</p>

### Automate the Install
----

<p>
  Now the last piece of the automation puzzle. 
  You can silently install your Primary Site Server using a 
  configuration file paired with the Sccm setup.exe file.  
  As a matter of fact, 
  once you have gone through the installation wizard and made it to the 
  summary page, 
  this file will be generated %TEMP%\ConfigMgrAutoSave.ini. 
  It should look something like this:
</p>

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
SQLServerName=cm1.codeAndKeep.com
DatabaseName=cm\CM_CDE
SQLSSBPort=4022
SQLDataFilePath=S:\Microsoft Sql\Data
SQLLogFilePath=S:\Microsoft Sql\Data

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
  I would recommend going through the wizard with your configuration first, 
  to see what you file looks like. 
  Now that you have your file you will should copy it from the temp location 
  and run your install.
</p>

```powershell
cp $ENV:TEMP\ConfigMgrAutoSave.ini -Destination C:\temp\SccmSetupConfig.ini
```

#### The install
----

```powershell
# Assuming your Sccm install media is on drive D
D:\SMSSETUP\BIN\X64\setup.exe /SCRIPT C:\temp\SccmSetupConfig.ini

# watch the install log for errors
Get-Content -Path C:\ConfigMgrSetup.log -Wait
```

<p>
  If you run in to errors for the install, be sure to check the setup logs. 
  Check ConfigMgrPrereq.log and ConfigMgrSetup.log for details on errors.
</p>
