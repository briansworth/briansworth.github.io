---
layout: post
title: Dsc - Install Active Directory Forest
---

<p>
  Building and rebuilding labs is something I do frequently. 
  Active Directory is one the biggest requirements for a Windows lab. 
  This means constantly installing AD Forests with the same configurations. 
  Automating this with PowerShell is pretty straightforward, 
  but Dsc can make it even more so.
</p>

#### Prerequisites
----

<p>
  You will need the following Dsc resources to follow along:
  <ul>
    <li>xActiveDirectory</li>
    <li>NetworkingDsc</li>
  </ul>
  Both are available from the PSGallery. 
  PowerShell version 5 is also required for this.
</p>

### Configuration Overview
----

<p>
  I will keep the configuration pretty simple for now. 
  Just 1 domain controller will be configured as follows:
  <ul>
    <li>Static IPv4 Address</li>
    <li>Disable Windows Firewall</li>
    <li>Install AD Forest</li>
    <li>Basic OU Structure</li>
  </ul>
</p>

Installing a new AD Forest does require specifying credentials. 
You should always ensure your credentials are encrypted for Dsc configurations. 
See my previous [post](http://codeandkeep.com/Dsc-Encrypting-Credentials/) 
for instructions on how to do so. 
I will be using a self-signed certificate for encryption in this example.

### Dsc Configuration
----

```powershell
configuration DomainInit {
  Param(
    [Parameter(Position=0)]
    [String]$DomainMode='WinThreshold',

    [Parameter(Position=1)]
    [String]$ForestMode='WinThreshold',

    [Parameter(Position=2,Mandatory=$true)]
    [PSCredential]$DomainCredential,

    [Parameter(Position=3,Mandatory=$true)]
    [PSCredential]$SafemodePassword,

    [Parameter(Position=4)]
    [String]$NetAdapterName='Ethernet0'
  )
  Import-DscResource -ModuleName PSDesiredStateConfiguration
  Import-DscResource -ModuleName xActiveDirectory
  Import-DscResource -ModuleName NetworkingDsc
  

  Node $ENV:COMPUTERNAME {
    WindowsFeature ADDSFeatureInstall {
      Ensure = 'Present';
      Name = 'AD-Domain-Services';
      DependsOn = '[NetAdapterName]InterfaceRename';
    }
    $domainContainer="DC=$($Node.DomainName.Split('.') -join ',DC=')"

    xADDomain 'ADDomainInstall' {
      DomainName = $Node.DomainName;
      DomainNetbiosName = $Node.DomainName.Split('.')[0];
      ForestMode = $ForestMode;
      DomainMode = $DomainMode;
      DomainAdministratorCredential = $DomainCredential;
      SafemodeAdministratorPassword = $SafemodePassword;
      DependsOn = '[WindowsFeature]ADDSFeatureInstall';
    }

    xWaitForADDomain 'WaitForDomainInstall' {
      DomainName = $Node.DomainName;
      DomainUserCredential = $DomainCredential;
      RebootRetryCount = 2;
      RetryCount = 10;
      RetryIntervalSec = 60;
      DependsOn = '[xADDomain]ADDomainInstall';
    }

    xADOrganizationalUnit 'CreateAccountsOU' {
      Name = 'Accounts';
      Path = $DomainContainer;
      Ensure = 'Present';
      Credential = $DomainCredential;
      DependsOn = '[xWaitForADDomain]WaitForDomainInstall';
    }

    xADOrganizationalUnit 'AdminOU' {
      Name = 'Admin';
      Path = "OU=Accounts,$DomainContainer";
      Ensure = 'Present';
      Credential = $DomainCredential;
      DependsOn = '[xADOrganizationalUnit]CreateAccountsOU';
    }
    
    xADOrganizationalUnit 'BusinessOU' {
      Name = 'Business';
      Path = "OU=Accounts,$DomainContainer";
      Ensure = 'Present';
      Credential = $DomainCredential;
      DependsOn = '[xADOrganizationalUnit]CreateAccountsOU';
    }

    xADOrganizationalUnit 'ServiceOU' {
      Name = 'Service';
      Path = "OU=Accounts,$DomainContainer";
      Ensure = 'Present';
      Credential = $DomainCredential;
      DependsOn = '[xADOrganizationalUnit]CreateAccountsOU';
    }

    NetAdapterName InterfaceRename {
      NewName = $NetAdapterName;
    }

    IPAddress StaticIP {
      InterfaceAlias = $NetAdapterName;
      AddressFamily = 'IPv4';
      IPAddress = $Node.IPv4Address;
      DependsOn = '[NetAdapterName]InterfaceRename';
    }

    DnsServerAddress SetDnsServer {
      InterfaceAlias = $NetAdapterName;
      AddressFamily = 'IPv4';
      Address = '127.0.0.1';
      DependsOn = '[NetAdapterName]InterfaceRename';
    }

    FirewallProfile DomainFirewallOff {
      Name = 'Domain';
      Enabled = 'False';
    }

    FirewallProfile PublicFirewallOff {
      Name = 'Public';
      Enabled = 'False';
    }

    FirewallProfile PrivateFirewallOff {
      Name = 'Private';
      Enabled = 'False';
    }

    LocalConfigurationManager {
      CertificateId = $Node.Thumbprint;
      RebootNodeIfNeeded = $true;
    }
  }
}
```

#### Environment Variables / Customization
----

<p>
  This section will give you the options to customize your AD Forest install. 
  Be sure to specify your certificate details and domain info.
</p>

```powershell
# Self signed certificate in the local computer certificate store
$cert=Get-Item -Path 'Cert:\LocalMachine\My\F9R4936DE1DBC4D5CEB407C8DEA2E2A3EC8C9F32'

# The certificate has been exported to this path
$certFilePath='C:\dsc\cert\DscPubKey.cer'


# Customize this with your details
$config=@{
  AllNodes=@(
    @{
      NodeName=$ENV:COMPUTERNAME;
      DomainName='codeAndKeep.com';
      IPV4Address='10.0.0.5/24';
      Thumbprint=$cert.Thumbprint;
      CertificateFile=$certFilePath;
    }
  )
}

$domainCred=Get-Credential

# Generate configuration MOF files
DomainInit -ConfigurationData $config `
  -OutputPath C:\dsc\AD `
  -DomainCredential $domainCred `
  -SafemodePassword $domainCred 
```

<p>
  You should have 2 files generated at this point. 
  One is a .mof file, the other is a meta.mof. 
  The meta.mof file is used to configure the Local Configuration Manager (LCM) 
  on the target node (the local machine in this case). 
  The regular .mof file is the configuration document for your server.
</p>

```powershell
# Configure the LCM
Set-DscLocalConfigurationManager -Path C:\dsc\AD -Force -Verbose

# Apply the Dsc Configuration 
Start-DscConfiguration -Path C:\dsc\AD -Force -Wait -Verbose
```

<p>
  We configured the LCM to reboot if the configuration requires it, 
  so the server should reboot to finalize the AD Forest installation. 
  When it comes back online and completes the final configurations, 
  you will have a new AD Forest created. 
  Take a look at what Organizational Units are there, 
  and of course what the server IP Address is.
</p>

<p>
  This is just the tip of the iceberg for what can be accomplished with Dsc. 
  This is also just the start for my posts about it. 
  Stay tuned.
</p>
