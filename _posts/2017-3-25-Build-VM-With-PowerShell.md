---
layout: post
title: Build a VM Using Only PowerShell
---

In this post I will be building a Hyper-V Virtual Machine from scratch, using only PowerShell
<br>

### Prerequisites

----

1. Hardware: Should be run on a physical machine (not a virtual machine)
2. Operating System: Windows 8.1 Pro/Enterprise, Windows 10 Pro/Enterprise, Windows Server 2012 R2, Windows Server 2016
3. No other Hypervisor: If you have Virtual Box or VMWare installed be aware that installing Hyper-V may break your other Hypervisor.
5. Operating System ISO to install on the VM
4. Permissions: Local Administrator, or Hyper-V Administrator

**My Setup:**
I will be running the following on a Windows 10 Pro OS, with 8Gb Memory, and a 3rd Gen Intel i7 @ 2.20Ghz processor.  This is a pretty old system but it won't have any issues getting this up and running.
<br>

## Getting Started
----

First things first, let's make sure we have all of the necessary information.  I want my VM to be on my network with internet access.
To do this, I will need an 'External Virtual Switch'.

```powershell
Get-VMSwitch

Name             SwitchType NetAdapterInterfaceDescription
----             ---------- ------------------------------
internal         Internal
```

Looks like I need to create one.  
*If you already have an External Virtual Switch skip ahead to 'Time to Build'* 
<br>

### Create an External Virtual Switch

---


```powershell
Get-NetAdapter

Name                      InterfaceDescription                    ifIndex Status
----                      --------------------                    ------- ------
vEthernet (internal)      Hyper-V Virtual Ethernet Adapter #2          16 Up
Ethernet                  Realtek PCIe GBE Family Controller            6 Disconnected
Wi-Fi                     Intel(R) Centrino(R) Wireless-N 2230         15 Up
```

Since I'm on a laptop, I will be using my **Wi-Fi** adapter.  
*NOTE: You can see that my ethernet port is not plugged in: Status 'Disconnected', and that my 'internal' virtual switch shows up in this list.*


```powershell
$adapterName='Wi-Fi'
New-VMSwitch -Name 'External vSwitch' -NetAdapterName $adapterName -AllowManagementOS $true
``` 

It may take a while to create, but you should now have an External Virtual Switch.

> Name             SwitchType NetAdapterInterfaceDescription
> ----             ---------- ------------------------------
> External vSwitch External   Intel(R) Centrino(R) Wireless-N 2230
<br>

## Time to Build

----

I am very particular about VM organization.  I have even created a partition on my hard drive specifically for virtual machines.  I always place the VM files and VHD files for that VM in the same folder.
This keeps it organized and easy to manage and maintain (maybe overkill for most people).

Before I begin, I setup and test each value that I will be using when creating this VM.

```powershell
$vmName='Win10'
$vSwitchName='External vSwitch'
$vmPath="V:\VMs\$vmName"
$vhdPath="$vmPath\$($vmName).vhdx"
$isoPath="V:\ISOs\14393.0.160715-1616.RS1_RELEASE_CLIENTENTERPRISE_S_EVAL_X64FRE_EN-US.ISO"
# If your ISO is for a version of windows before the Win8 generation
# use Generation 1, otherwise use Generation 2
# for linux use Gen 1
[int16]$generation=2
[int64]$memory=3Gb
[int64]$vhdSize=30Gb

New-VM -VMName $vmName `
  -SwitchName $vSwitchName `
  -Path $vmPath `
  -NewVhdPath $vhdPath `
  -NewVhdSizeBytes $vhdSize `
  -MemoryStartupBytes $memory `
  -Generation $generation

```

Now that we have our VM made, lets do a little customization.

```powershell
# Give the VM more CPU
Set-VM -VMName $vmName -ProcessorCount 2
# if no dvd drive, add one
if(! (Get-VMDvdDrive -VMName $vmName)){
  Add-VMDvdDrive -VMName $vmName -Path $isoPath
}else{
  Set-VMDvdDrive -VMName $vmName -Path $isoPath
}
Start-VM -VMName $vmName
# Open up the vmconnect window for the new vm
vmconnect.exe localhost $vmName
```

Depending on your VM Generation, you may have to press a button to tell the vm to install from the Dvd drive.
There is much more that can be done, but this is a quick and easy way to get started with Hyper-V and PowerShell.


*Thanks for reading,*

PS> exit
