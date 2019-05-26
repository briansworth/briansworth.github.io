---
layout: post
title: PowerShell - Dhcp Install
description: Step by step guide to install and configure a Windows DHCP Server
---

<p>
  Setting up a DHCP Windows server is made very easy in PowerShell. 
  In this post, we use PowerShell to setup a DHCP server, 
  and do some additional configuration. 
  This will ultimately lead to faster server builds, 
  as the IP configuration will automatically be taken care of!
</p>

<p>
  As always, you can skip ahead to the end for the source code. 
  If you want to go through the journey of figuring this out with me, 
  be sure to read each section.
</p>

### Prerequisites
-----

I am assuming you have at least the following configuration setup:
  - Active Directory Forest
  - A VM for the DHCP server
    - You can use a DC for the DHCP role if you want
  - The DHCP Server should already be joined to the domain
    - If you are using an existing DC, this is already done
  - My GetIPv4Subnet module
    - On [GitHub](https://github.com/briansworth/GetIPv4Address/blob/master/GetIPv4Subnet.psm1)
    - Or from a previous [blog post]({{ site.baseurl }}/PowerShell-Get-Subnet-NetworkID/)

### The Journey
----

<p>
  Having a Dhcp server in your environment can speed up new server builds. 
  It can take away the hastle of manually configuring static IPs, 
  DNS servers, default gateways, and more. 
  If you love automation like me, you will love Dhcp.
</p>

<p>
  Installing this on a Windows server is very simple: 
</p>

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
```

<p>
  The hard part is customizing it for your environment, 
  and knowing what to configure.
</p>

<p>
  In an Active Directory environment, 
  you will need to authorize your DHCP server.
  This will allow your server to start leasing IP addresses.
</p>

```powershell
Add-DhcpServerInDC -DnsName dhcp.codeAndKeep.com -IPAddress 10.0.0.3
```

<p>
  You are likely going to want DNS to be updated after the DHCP server 
  leases an IP address to a server. 
  This will make your environment easy to navigate using 
  DNS names instead of IP addresses. 
  You can do this like so:
</p>

```powershell
Set-DhcpServerv4DnsSetting -DynamicUpdates 'Always' `
  -DeleteDnsRROnLeaseExpiry $true
```

*NOTE: This setting above 'DeleteDnsRROnLeaseExpiry' also tells DHCP to remove DNS entries when a lease has expired.*

<p>
  A DHCP server can manage IP address leasing across multiple networks. 
  It can also be customized to only lease out IP addresses in a certain range. 
  To accomplish this, you create 'Scopes'. 
  A scope will identify the network that it can lease out to,
  as well as the start and end range it can use.
  You can create one like so:
</p>

```powershell
Add-DhcpServerV4Scope -Name "Corp Net" `
  -StartRange 10.0.0.50 `
  -EndRange 10.0.0.200 `
  -SubnetMask 255.255.255.0 `
  -State InActive

# State InActive means the DHCP server won't start leasing IPs yet
# for this network
```

<p>
  This should create a scope with a 'ScopeID' of '10.0.0.0'. 
  With this scope setup, when a new server is added to this network, 
  the DHCP server will assign it an IP address between 
  10.0.0.50 and 10.0.0.200.
</p>

```powershell
# Check the scope you just created
Get-DhcpServerv4Scope
```

<p>
  Assigning an IP address is great, 
  but we can go further than that. 
  Setting the DNS server addresses will make our lives easier.
</p>

```powershell
# Use the Scope Id from what shows up in Get-DhcpServerv4Scope

# Set the default gateway 
Set-DhcpServerv4OptionValue -ScopeID 10.0.0.0 `
  -OptionID 3 `
  -Value 10.0.0.1

# Change the DnsServer value to the IP(s) of your DC(s)
Set-DhcpServerv4OptionValue -ScopeID 10.0.0.0 `
  -DnsDomain codeAndKeep.com `
  -DnsServer 10.0.0.5
```

<p>
  Validate the changes we just made to this scope:
</p>

```powershell
Get-DhcpServerv4OptionValue -ScopeID 10.0.0.0 

OptionId   Name            Type       Value
--------   ----            ----       -----
3          Router          IPv4Add... {10.0.0.1}
15         DNS Domain Name String     {codeAndKeep.com}
6          DNS Servers     IPv4Add... {10.0.0.5}
51         Lease           DWord      {1209600}
```

<p>
  Now we have some good customization setup on the Dhcp server, 
  and for this network / scope. 
  You can probably tell, 
  there is plenty more that could be done with Option Values. 
  This is a good start, and if you are happy with how it looks, 
  we can go ahead and set this scope to active:
</p>

```powershell
# Activate your scope
Set-DhcpServerv4Scope -ScopeId 10.0.0.0 -State Active

# Validate that it is now active
Get-DhcpServerv4Scope -ScopeId 10.0.0.0
```

*NOTE: If you are running Hyper-V with an internal VMSwitch, 
your NetAdapter on your host may get a lease when you turn on your scope. 
This may cause your network connectivity to slow down on your host. 
You can assign a static APIPA address to the NetAdapter as a quick fix*

*Example: New-NetIPAddress -InterfaceAlias 'vEthernet (internal1)'
  -IPAddress 169.254.0.1*


### The Code
----

Before you run this code, ensure that you have the GetIPv4Subnet module imported. 
As stated in the prereq section, you can get that module [here](https://github.com/briansworth/GetIPv4Address/blob/master/GetIPv4Subnet.psm1).

```powershell
Install-WindowsFeature -Name DHCP

Write-Verbose "Adding DHCP Server local security groups..."
netsh dhcp add securitygroups
Restart-Service -Name DhcpServer

Write-Verbose "Authorizing Dhcp server in AD..."
$hostDnsInfo=[Net.Dns]::GetHostByName("$ENV:COMPUTERNAME")
Add-DhcpServerInDC -DnsName $hostDnsInfo.HostName `
  -IPAddress $hostDnsInfo.AddressList[0] | Out-Null

Write-Verbose "Setting Dhcp to use Dynamic Updates for expired lease deletion"
Set-DhcpServerv4DnsSetting -ComputerName $hostDnsInfo.HostName `
  -DynamicUpdates 'Always' `
  -DeleteDnsRROnLeaseExpiry $true

######### IP / Network discovery ##########

Write-Verbose "Getting IP Configuration..."
$ipConfig=Get-NetIPConfiguration
$ipInfo=$ipConfig[0].IPv4Address[0]

$subnetInfo=Get-IPv4Subnet -IPAddress $ipInfo.IPAddress `
  -PrefixLength $ipInfo.PrefixLength

######### Back To DHCP Configuration #########

#### Add First DHCP IPv4 Scope
Write-Verbose "Adding DHCP Scope for subnet [$($subnetInfo.NetworkID)]"
$scopeName='CorpNet0'
$leaseTimespan=New-Timespan -Days 14

# Hardcoded to exclude 9 IP Addresses at the start of the subnet
# and reserve 5 IP Addresses from the end of the subnet from the scope
$startIP=Add-IntToIPv4Address -IPv4Address $subnetInfo.FirstHostIP `
  -Integer 9

$endIP=Add-IntToIPv4Address -IPv4Address $subnetInfo.LastHostIP `
  -Integer -5

Add-DhcpServerV4Scope -Name $scopeName `
  -StartRange $startIP `
  -EndRange $endIP `
  -SubnetMask $subnetInfo.SubnetMask `
  -LeaseDuration $leaseTimespan

Write-Verbose "Setting scope Default Gateway option..."
Set-DhcpServerv4OptionValue -ScopeID $subnetInfo.NetworkID `
  -OptionID 3 `
  -Value $subnetInfo.FirstHostIP

Write-Verbose "Setting scope DNS options..."
$csDomain=Get-WmiObject -Query "SELECT Domain FROM Win32_ComputerSystem"
$domainDnsInfo=[Net.Dns]::GetHostByName($csDomain.Domain)

Set-DhcpServerv4OptionValue -ScopeID $subnetInfo.NetworkID `
  -DnsDomain $domainDnsInfo.HostName `
  -DnsServer $domainDnsInfo.AddressList
```
