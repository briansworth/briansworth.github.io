---
layout: post
title: Build a VM from template
---

Creating VMs from a base image can make your life a lot easier.  Not having to sit through and select the same options during an OS install is a big time save.  
A lot of work can go into creating a base image, but it doesn't have to.  I will briefly touch on how to create a base Windows image and then get into the automation of creating a Hyper-V VM from that template.


<br>

In my last [post ](http://codeandkeep.com/Build-VM-With-PowerShell/), I showed a simple way of creating a VM using only PowerShell.
I would recommend looking at that post first and testing it out to ensure you know the basics of PowerShell and Hyper-V.

<br>

To start, build a VM with the Windows OS that you wish to use (I will be using Server 2016 core).  The process should be the same for Windows 7/Server 2008 and up. 
Once it is up and running make whatever configurations that you want to be applied to all machines that you will be using this operating system on.  
I have installed VIM for Windows and applied the latest Windows Updates, that's it.

When all that configuration is complete, open up PowerShell and navigate to the 'Sysprep' folder (generally here C:\Windows\System32\Sysprep).

```powershell
cd $ENV:SystemRoot\System32\Sysprep
```

In this directory you will find a folder called 'Panther', which can be used when applying an AutoUnattend.xml file for additional configuration.  More importantly, is the 'sysprep.exe' executable.

```powershell
sysprep.exe /?
```

Here you can see the options available.  
You can simply run sysprep.exe and use the GUI, but that's not my style (there's also a switch I will use that's only available from the cmd line).

# ![_config.yml]({{ site.baseurl }}/images/sysprep.PNG)

<br>

For this use case the switches I will be using are:
1. Generalize - Allows this image to be applied to more then just this machine
2. Oobe - Out of Box Experience: The alternative is Audit Mode, but that's not necessary for my use case
3. Mode:VM - This switch makes the first boot faster by skipping some hardware detection. \* 
4. Shutdown - I want to use this sysprep'd image for other VMs, so I don't want to restart and customize more

\* - NOTE: This option is only supported on Win8/Server 2012 and up.  It should not be used if you want to use this image on a physical device, or use this image on another hypervisor.

```powershell
.\sysprep.exe /generalize /oobe /mode:vm /shutdown
```
<br>
# ![_config.yml]({{ site.baseurl }}/images/sysprepProc.PNG)

The VM will shutdown after successfully completing the sysprep process.
<br>


## Using the Image
----

First things first, you should remove the vm from Hyper-V (just keep the VHDX file).
You should know where this file is located.  In my case it is V:\VMs\serverCore\serverCore.vhdx
You can find the file location using the following steps:
```powershell
$vmName='serverCore'
Get-VMHardDiskDrive -VMName $vmName | Select -expand Path
```
As long as you only have 1 vhd file for that vm, this will tell you your path.

I have created a Templates folder to keep these template files for easy access.

```powershell
$vmName='serverCore'
# Get VHD Path
$vhdPath=Get-VMHardDiskDrive -VMName $vmName | Select -expand Path

# Destination/new name of the template vhd file (change to your value)
$templatePath='V:\Templates\core2016.vhdx'

Remove-VM -VMName $vmName
Move-Item -Path $vhdPath -Destination $templatePath

# set the vhd file to ReadOnly so it doesn't get modified
Set-ItemProperty -Path $templatePath -Name isReadOnly -Value $true
```
Now we are ready to build a new virtual machine using this template.
<br>

### Build from template
----

We can use a similar approach that we used in the previous post.

```powershell
# make sure you update these variable to match your configuration
$templatePath="V:\Templates\core2016.vhdx"
$vmName='server1'
$vmPath="V:\VMs\$vmName"
$vhdPath="$vmPath\$vmName.vhdx"
$switchName='External vSwitch'
[int16]$generation=2
[int64]$memory=1Gb
[int16]$vCPU=2

[bool]$isDir=Test-Path -Path $vmPath
if(!$isDir){
  New-Item -Path $vmPath -Type Directory | Out-Null
}
Copy-Item -Path $templatePath -Destination $vhdPath -Verbose

Set-ItemProperty -Path $vhdPath -Name isReadOnly -Value $false
New-VM -VMName $vmName `
  -Path $vmPath `
  -VHDPath $vhdPath `
  -MemoryStartupBytes $memory `
  -SwitchName $switchName `
  -Generation $generation `
  -Verbose

Set-VM -VMName $vmName -ProcessorCount $vCPU 
Start-VM -VMName $vmName
vmconnect.exe localhost $vmName
```
It's important to note that the VMs that you create from this template should match the generation of the VM you created to make this template.

<br>
#### Additional Comments
----

The Copy-Item will take a while to finish and unfortunately doesn't provide an indication of progress. You could replace this with Robocopy for a more reliable copy of the file.

There is no validation in the code used above.  If the VM name of the VHD path is already taken, there will be errors and the VM will not build.  If the vmname is already taken, it may start the VM and modify the Processor Count of that VM.

I did a quick comparison of the first boot of a machine sysprep'd with /mode:VM and without /mode:VM.
Some VM configuration and same operating system on the same hypervisor.  No VMs were running and the system had more than enough memory to quickly start the VM.
1. Without /mode:vm -  3:19.50
2. With /mode:vm -  1:50.23
3. From DVD - 5:31.46 \*

\*After copy and install (time from boot after 'Restart Now' prompt to console being loaded)


16:30.37 -VM Creation 14.25 -boot 5:31.46 Install

6:05.98 - Copy: 4:02.79 -VM Creation 12.96

9:31.84 - Copy: 5:57.03 -VM Creation 15.31 - Boot 

|VM Creation Mode| Total (mm:ss.msms) |Boot (mm:ss.msms)|Copy/Install (mm:ss.msms)|VM Provisioning (mm:ss.msms )| 
|----------------|--------------------|-----------------|-------------------------|-----------------------------|
| DVD            |16:30.37 | 5:31.46 |10:44:66 | 0:14.25  |
| Sysprep        | 9:31.84 | 3:19.50 | 5:57.03 | 0:15.31  |
| Sysprep VM Mode| 6:05:98 | 1:50.23 | 4:02:79 | 0:12:96  |

It should be noted that the image I used in the 'Sysprep' creation mode, was approximately 2Gb larger than the 'Sysprep VM Mode' image.  This contributed to the longer copy time, but should not influence the difference in boot time.

<br>

*Thanks for reading,*

PS> exit
