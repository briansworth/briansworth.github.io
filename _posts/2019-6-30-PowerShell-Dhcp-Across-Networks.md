---
layout: post
title: PowerShell - Dhcp Across Networks
description: How to use Windows Routing to enable a Dhcp server to work across multiple networks
---

<p>
  With all the benefits that DHCP has to offer, 
  it would be crazy if we needed to put a DHCP server in each 
  network we setup. 
  With this post, I will go through the steps of getting 
  your Windows DHCP server working across whatever networks you want.
</p>

### Prerequisites
----

To follow along with this guide you should have a setup with at least the following:
- Active Directory Forest
  - 1 Domain Controller
  - [Guide](https://codeandkeep.com/Dsc-Install-AD-Forest/)
- Dhcp server role on a server
  - Can be it's own server, or on a DC
  - [Guide](https://codeandkeep.com/PowerShell-Dhcp-Install/)
- RRAS role installed on a server
  - Should be on it's own server
  - [Guide](https://codeandkeep.com/PowerShell-Windows-Routing/)
- 1 test server on new network

### My setup
----

2 networks:
  - 10.0.0.0
    - Has the Domain Controller (10.0.0.5)
    - Has the Dhcp server (10.0.0.3)
    - RRAS server: 
      - IP address: 10.0.0.1
      - NetAdapter Name: Ethernet0
  - 10.0.1.0
    - Has the test server
    - RRAS server:
      - IP address: 10.0.1.1
      - NetAdapter Name: Ethernet1


### The Journey
----

The configuration can be broken down into 3 steps:
  1. Get information about the Dhcp server (IP address, DNS host name)
  2. Create Dhcp scope for the new network
  3. Install and configure the Dhcp relay agent

<p>
  You can run all the code below directly from your RRAS server.
</p>

#### 1. Get Dhcp server information
<p>
  We need to know the hostname or IP address of the Dhcp server. 
  We will also need to add a Dhcp scope for this new network 
  (10.0.1.0 in this example).
</p>

```powershell
# IP address or DnsHostName of your Dhcp Server
$dhcpServer = 'dhcp.codeAndKeep.com'

# Name of the network adapter to enable DHCP on
$routerNetAdapterName = 'Ethernet1'


$dhcpAddress = [Net.Dns]::GetHostEntry($dhcpServer)
if(!$dhcpAddress){
  Write-Warning "Unable to identify IP address of [$dhcpServer]"
  break
}else{
  $dhcpServerIP = $dhcpAddress.AddressList[0]
}
```

#### 2. Create new Dhcp scope

<p>
  We will need to create a new Dhcp scope for our second network. 
  Be sure to change the IP addresses to what you want/have configured. 
</p>

```powershell
# If you have access to the dhcp server 
## You can create your Dhcp scope remotely
$dhcpSession = New-PSSession -ComputerName $dhcpAddress.HostName
Import-PSSession -Session $dhcpSession -Module DhcpServer

# You can run the commands below directly on your DHCP Server
## If you don't want to use a PSSession

# Be sure to switch your Start and End Ranges if they are different
$newScope = Add-DhcpServerv4Scope -Name RemoteNet0 `
  -StartRange 10.0.1.20 `
  -EndRange 10.0.1.200 `
  -SubnetMask 255.255.255.0 `
  -State InActive `
  -PassThru

Set-DhcpServerv4OptionValue -ScopeID $newScope.ScopeId `
  -DnsDomain codeAndKeep.com `
  -DnsServer 10.0.0.5 `
  -Router 10.0.1.1

Set-DhcpServerv4Scope -ScopeId $newScope.ScopeID -State Active

# Not necessary if you ran this on directly on your Dhcp server
Remove-PSSession $dhcpSession
```

#### 3. Install a Dhcp relay agent

<p>
  This was the most difficult thing to figure out, 
  as there is no PowerShell cmdlets to configure this. 
  Having worked with netsh previously, 
  I was able to find some documentation around configuring a relay agent. 
</p>

<p>
  In the code below, we will create a netsh script file to execute. 
  This script will be populated with the IP address of the Dhcp server, 
  as well as the interface alias that the relay agent will use. 
</p>

```powershell
$netshDhcpRelay=@"
pushd routing ip relay
install
set global loglevel=ERROR
add dhcpserver $($dhcpServerIP.IPAddressToString)
add interface name="$routerNetAdapterName"
set interface name="$routerNetAdapterName" relaymode=enable maxhop=6 minsecs=6
popd
"@

$netshDhcpRelayPath="$ENV:TEMP\netshDhcpRelay"

# Create netsh script file
New-Item -Path $netshDhcpRelayPath `
  -Type File `
  -ErrorAction SilentlyContinue | Out-Null

# Populate contents of the script 
Set-Content -Path $netshDhcpRelayPath `
  -Value $netshDhcpRelay.Split("`r`n") `
  -Encoding ASCII

# run it
netsh -f $netshDhcpRelayPath
```

#### Test Dhcp on new network

<p>
  Finally to ensure it is configured and working, 
  we need a computer on the new network. 
  If there is one already running you can restart the network adapter to 
  start the Dhcp client.  
  If it was configured with a static IP address, you should remove it.
</p>

```powershell
Get-NetAdapter | Restart-NetAdapter
```

<p>
  On your RRAS server, you can check the statistics of your relay agent. 
  You can use this for troubleshooting or just to see some statistics. 
</p>

```
netsh routing ip relay show ifstats
```

You should see something like this:
![_config.yml]({{ site.baseurl }}/images/dhcpRelayIFStats.PNG)

<p>
  You can check your Dhcp server as well to see if a lease has been issued. 
</p>

```powershell
Get-DhcpServerv4ScopeStatistics -ScopeID $newScope.ScopeId
```
