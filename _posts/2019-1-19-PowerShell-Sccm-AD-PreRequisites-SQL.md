---
layout: post
title: PowerShell - Automate SQL Install - SCCM PreReqs
---

This is a continuation of automating Sccm prerequisites [part 1](
http://codeandkeep.com/PowerShell-SCCM-Offline-PreRequisites/), 
[part 2](
http://codeandkeep.com/PowerShell-SCCM-Offline-PreRequisites-Install/) 
and [part 3](
http://codeandkeep.com/PowerShell-Sccm-AD-PreRequisites/
).

<p>
  In this post I will be covering how you can automate the 
  installation of Microsoft Sql Server. 
  We will only install the Sccm required Sql components, 
  Reporting Services and the Sql Database Engine.
</p>

<p>
  Automatically installing Sql server can be accomplished in a few 
  different ways.
  You can use the command line supported options of the setup.exe 
  file that comes with the Sql installation media. 
  You can also create a Sql install configuration file, 
  and run the same setup.exe file pointing to that configuration file. 
</p>
<p>
  The alternative to these options, 
  is to use Desired State Configuration (DSC). 
  If you are not familiar with this product I would highly recommend 
  looking into it. 
  In essence, DSC is a technology that can automate infrastructure. 
  Using DSC, you will be writing code that can configure, build, and 
  maintain your environment (it can also manage Linux systems). 
  Think infrastructure as code. 
  I will be posting a lot more on DSC, 
  and hopefully this has peaked your interest if you are not familiar. 
</p>

### What You Need
----

<p>
  <ol>
    <li>PowerShell v5</li>
      <ul>
        <li>Default on Win10 and Server 2016</li>
      </ul>
    <li>SqlServerDsc PowerShell Dsc Module</li>
      <ul>
        <li>Download from PowerShell Gallery, directly from PowerShell</li>
      </ul>
    <li>Sql Installation Media</li>
      <ul>
        <li>Mount or extract ISO on target server</li>
      </ul>
  </ol>
</p>

#### Check your PowerShell version:

```powershell
$PSVersionTable

Name                           Value
----                           -----
PSVersion                      5.1.17134.407
PSEdition                      Desktop
PSCompatibleVersions           {1.0, 2.0, 3.0, 4.0...}

# if you see PSVersion start with 5, you are on v5
```

If you aren't on PSv5, download and install WMF 5.1 
[here](https://www.microsoft.com/en-us/download/details.aspx?id=54616) 
and download and install the PowerShell package manager 
[here](https://www.microsoft.com/en-us/download/details.aspx?id=51451)


#### Get the SqlServerDsc module

<p>
  You should be able to run 'Find-Module -Name SqlServerDsc' and 
  see something like below.
</p>

```powershell
Find-Module -Name SqlServerDsc

Version    Name            Repository     Description
-------    ----            ----------     -----------
12.2.0.0   SqlServerDsc    PSGallery      Module with DSC Resources...
```
<p>
  If the server you are installing Sql on has internet access, 
  you can simply pipe that command to Install-Module.
</p>

```powershell
Find-Module -Name SqlServerDsc | Install-Module 

# Or just -> Install-Module -Name SqlServerDsc
```

<p>
  If you need to download it first and copy to another computer:
</p>

```powershell
# Choose where to save the module
$path='C:\temp\SqlDsc'

Save-Module -Name SqlServerDsc -Path $path
```

<p>
  Now you can copy the SqlServerDsc folder 
  location to your server and place it here: 
  C:\Program Files\WindowsPowerShell\Modules\
</p>

```powershell
# Should look something like this
C:\Program Files\WindowsPowerShell\Modules\SqlServerDsc
 - 12.2.0.0
   - DSCResources
   - en-US
   - Examples
   - sv-SE
   - Tests
 ...
```

<br>

#### Test SqlServerDsc module installation

```powershell
Get-DscResource -Module SqlServerDsc

ImplementedAs   Name                      ModuleName                     Version
-------------   ----                      ----------                     -------
PowerShell      SqlAG                     SqlServerDsc                   12.2.0.0
PowerShell      SqlAGDatabase             SqlServerDsc                   12.2.0.0
PowerShell      SqlAGListener             SqlServerDsc                   12.2.0.0
PowerShell      SqlAGReplica              SqlServerDsc                   12.2.0.0
PowerShell      SqlAlias                  SqlServerDsc                   12.2.0.0
PowerShell      SqlAlwaysOnService        SqlServerDsc                   12.2.0.0
PowerShell      SqlDatabase               SqlServerDsc                   12.2.0.0
PowerShell      SqlDatabaseDefaultLoca... SqlServerDsc                   12.2.0.0
...
```

<br>

### The Code
----

<p>
  I have broken the code into it's different components. 
  First is the 'configuration' which is just like a PowerShell function. 
  It has parameters that you can specify and 
  you run it the same as you would a function.
</p>
<p>
  Next is the Configuration Data. 
  Using Configuration Data is not mandatory, 
  but can be required under certain circumstances. 
  In this example, if you want to use credentials 
  (IE: configure SQL Services with service accounts) 
  then you will need to use the Configuration Data.
</p>
<p>
  Lastly, the MOF file generation and the 'PUSH'; 
  sending the MOF to the 'Local Configuration Manager' 
  so it can do your bidding.
</p>


#### Configuration

```powershell
configuration SccmSqlInstallation {
  Param(
    [Parameter(Position=0)]
    [String]$computerName=$ENV:COMPUTERNAME,

    [Parameter(Position=1,Mandatory=$true)]
    [String]$sqlSourceFiles,
    
    [Parameter(Position=2)]
    [String]$sqlInstanceName='MSSQLSERVER',
    
    [Parameter(Position=3)]
    [PSCredential]$agentSvcCredential,

    [Parameter(Position=4)]
    [PSCredential]$sqlSvcCredential,

    [Parameter(Position=5)]
    [String[]]$sysAdminAccounts=(whoami),

    [Parameter(Position=6)]
    [String]$features='SQLENGINE',

    [Parameter(Position=7)]
    [String]$instanceDir="$ENV:ProgramFiles\Microsoft SQL Server",

    [Parameter(Position=8)]
    [String]$dataDir,

    [Parameter(Position=9)]
    [String]$sharedDir="$ENV:ProgramFiles\Microsoft SQL Server",

    [Parameter(Position=10)]
    [String]$sharedWOWDir="${ENV:ProgramFiles(x86)}\Program Files (x86)\Microsoft SQL Server"
  )
  Import-DscResource -ModuleName SqlServerDsc
  Import-DscResource -ModuleName PSDesiredStateConfiguration

  node $computerName {

    WindowsFeature 'NetFramework' {
      Name = 'Net-Framework-45-Core';
      Ensure = 'Present';
    }

    SqlSetup 'SqlInstall' {
      InstanceName = $sqlInstanceName;
      SourcePath = $sqlSourceFiles;
      Action = 'Install';
      Features = $features;
      InstanceDir = $instanceDir;
      InstallSQLDataDir = $dataDir;
      InstallSharedDir=$sharedDir;
      InstallSharedWOWDir=$sharedWOWDir;
      SQLSvcStartupType = 'Automatic';
      AgtSvcStartupType = 'Automatic';
      AgtSvcAccount = $agentSvcCredential;
      SQLSysAdminAccounts = $sysAdminAccounts;
      SQLSvcAccount = $sqlSvcCredential;
      SQLCollation = 'Latin1_General_CP1_CI_AS';
      DependsOn = '[WindowsFeature]NetFramework';
    }

    # This part can be removed 
    # If you aren't using credentials, OR
    # If you have opted to store credentials in Plain Text
    LocalConfigurationManager {
      CertificateId = $node.Thumbprint
    }
    # Remove to here
  }
}
```

#### Configuration Data

<p>
  If you plan on using credentials in your configuration, 
  you will need to use this. 
  There is the right way to do this 
  (using certificates to encrypt your credentials), 
  and there is the wrong way to do this 
  (storing your credentials in plain text). 
  To do the insecure thing, you can uncomment out the 
  PsDscAllowPlainTextPassword key below and remove the 
  CertificateFile and Thumbprint entries. 
</p>
Doing it the right way takes more work of course. 
There is a great document that explains how to set this up 
[here](https://docs.microsoft.com/en-us/powershell/dsc/pull-server/securemof).

```powershell
$config = @{
  AllNodes = @(
    @{ 
      NodeName = $ENV:COMPUTERNAME 
      #PsDscAllowPlainTextPassword = $true;
      #PsDscAllowDomainUser = $true;
      CertificateFile = 'C:\cert\cert.cer';
      Thumbprint = '4B1EF9E6C194098257E93120A4A39DA853F23434'
    }
  )
}
```

#### Generate the MOF

<p>
  It is here where you can customize how Sql will install. 
  You can remove the service accounts if you want the 
  sql services running as default. 
  If you don't use credentials at all, 
  you do not need the ConfigurationData parameter. 
</p>

```powershell
$svcCred=Get-Credential 

SccmSqlInstallation -OutputPath C:\temp\sqlInstall `
  -computerName $ENV:COMPUTERNAME `
  -ConfigurationData $config `
  -sqlInstanceName cm `
  -sqlSourceFiles 'D:\' `
  -sqlSvcCredential $svcCred `
  -sysAdminAccounts 'codeAndKeep\cmAdmin' `
  -agentSvcCredential $svcCred `
  -features "SQLENGINE,RS"

# SQLENGINE to install the database engine
# RS will install the reporting services

# You do not need to specify service account credentials 
# for Sccm to work
```

#### Push it real good

<p>
  Finally we can push the configuration to the server 
  and watch the magic happen. 
</p>

```powershell
# This step is only required if you used credentials 
# with certificate encryption
Set-DscLocalConfigurationManager -Path C:\temp\sqlInstall

Start-DscConfiguration -Path C:\temp\sqlInstall -Wait -Force -Verbose
```
<p>
  This is prone to have errors if you have misconfigured something.  
  You can check the Sql Installation Summary file for insight into 
  any errors you get. 
  Generally located here: 
  %ProgramFiles%\Microsoft SQL Server\nnn\Setup Bootstrap\Log\Summary.txt 
  where nnn is the SQL version (ex: 130).
</p>
