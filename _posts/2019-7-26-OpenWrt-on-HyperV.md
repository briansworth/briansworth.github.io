---
layout: post
title: OpenWrt on Hyper-V
description: Setup a firewall (OpenWRT) as a VM on Hyper-V.
---

As my recent posts indicate, 
I have been exploring networking related tasks in PowerShell. 
Sticking with that theme, 
I thought it would be nice to play with an actual firewall in my lab. 
With that being said, lets get into the OpenWRT setup on Hyper-V. 


*NOTE: While I am targeting Hyper-V as the platform, 
the same general process can be used to setup OpenWRT on another Hypervisor.*

### Problems with Windows
---

Unfortunately, 
I was unable to get this whole thing setup using just my Windows host. 
Getting the image created, 
and adding in the necessary network drivers to the image required a Linux touch. 
If you can figure out a way without using Linux I would be willing to give it a try.

This makes step 1, setting up a Linux VM (if you don't already have one).

### Step 1: Create Linux VM
---

I won't go into the details of setting up a Linux VM on Hyper-V. 
It is pretty straight forward, and if you are on a new Windows 10 version, 
you can setup Ubuntu incredibly easy following this 
[guide](https://blogs.windows.com/windowsdeveloper/2018/09/17/run-ubuntu-virtual-machines-made-even-easier-with-hyper-v-quick-create/#zbJYvKflS4WoAod7.97). 

Once you have a Linux VM configured, power it off and proceed to step 2.

### Step 2 : The VHD
---

Time to create the VHD file that we will use for the openWrt VM.

```powershell
$wrtName='openWrt'
mkdir "V:\VMs\$wrtName"
$vhd=New-VHD -Path "V:\VMs\$wrtName\$wrtName.vhd" -SizeBytes 500MB -Fixed
Write-Output $vhd
```

Now, we can attach this VHD to our Linux VM so we can write to it.

```powershell
# Ensure your Linux VM is powered off

# change to your VM name
$vmname='ubuntu'

Add-VMHardDiskDrive -VMName $vmname `
  -ControllerType IDE `
  -ControllerNumber 0 `
  -ControllerLocation 1 `
  -Path $vhd.Path 

Get-VMHardDiskDrive -VMName $vmname
# You should see at least 2 disks show up
# with the 2nd one being the new vhd

Start-VM $vmname
```


### Step 3: Download OpenWRT
---

This should be done on your Linux VM, from the terminal. 
I will be downloading the latest (at the time of writing) image available, 18.06.1.
Feel free to choose your own.

```bash
sudo curl http://downloads.openwrt.org/releases/18.06.1/targets/x86/64/openwrt-18.06.1-x86-64-combined-ext4.img.gz \
-o /tmp/openwrt-18.06.1.img.gz

# Unzip the openwrt img file
sudo gzip -dk /tmp/openwrt-18.06.1.img.gz
```

### Step 4: Write the OpenWRT Image to Disk
---

We will need to identify the new disk we added to our Linux VM in step 2. 
You can list out the disks like so:

```bash
sudo fdisk -l
```

You should see something like this:

```
Disk /dev/sda: 30 GiB, 32212254720 bytes, 62914560 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x57b7b33b

Device     Boot    Start      End  Sectors Size Id Type
/dev/sda1  *        2048 58720255 58718208  28G 83 Linux
/dev/sda2       58722302 62912511  4190210   2G  5 Extended
/dev/sda5       58722304 62912511  4190208   2G 82 Linux swap / Solaris

Partition 2 does not start on physical sector boundary.


Disk /dev/sdb: 500 MiB, 524288000 bytes, 1024000 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

Note the first disk is shown: "Disk **/dev/sda**: 30 GiB".
Which makes the second one: "Disk **/dev/sdb**: 500MiB. 

This means /dev/sdb is our target disk to write the OpenWrt image to.

```bash
sudo dd if=/tmp/openwrt-18.06.1.img of=/dev/sdb bs=4M conv=fsync status=progress
```

Now you can power off your Linux VM for the moment so you can remove this disk.

```bash
sudo poweroff
```

### Step 5: Create your OpenWRT VM
---

Back on your Windows host machine:

```powershell
# In PowerShell on your host machine

# Remove VHD from your Linux VM
Remove-VMHardDiskDrive -VMName $vmname `
  -ControllerType IDE `
  -ControllerNumber 0 `
  -ControllerLocation 1

# Create openWRT VM
New-VM -VMName $wrtName `
  -Path "V:\VMs\$wrtName" `
  -VHDPath $vhd.Path `
  -MemoryStartUpBytes 512MB `
  -Generation 1

Get-VMNetworkAdapter -VMName $wrtName | Remove-VMNetworkAdapter

Start-VM $wrtName
vmconnect localhost $wrtName
```

It should do its configuration and boot into the OS. 
After it has completed the boot, (you can press enter to get to the cmd line)
you can run some commands to see what you have. 

```bash
# Will only show you the loopback adapter 'lo'
ifconfig 
```

Once you have looked around, you can power it off to get onto the next step
```bash
poweroff
```

### Step 6: Add VHD back on to Linux VM
---

Now we will need add this disk back onto our Linux VM for further processing.  

I recommend removing all snapshots (in case there are any) from the VM, 
to make removing and adding disks easier. 

```powershell
# Back on your Windows host
$snapshots=Get-VMSnapshot -VMName $wrtName 
if($snapshots){
  $snapshots | Remove-VMSnapshot
}

Remove-VMHardDiskDrive -VMName $wrtName `
  -ControllerType IDE `
  -ControllerNumber 0 `
  -ControllerLocation 0
```

Now add the VHD back onto your Linux VM:

```powershell
# Add the vhd file back to your linux vm
Add-VMHardDiskDrive -VMName $vmname `
  -ControllerType IDE `
  -ControllerNumber 0 `
  -ControllerLocation 1 `
  -Path $vhd.Path

Start-VM $vmname
```


### Step 7: Get OpenWRT network driver
---

You will need the 'tulip' drivers for the VM Network adapters to work on Hyper-V.
We can download this on our Linux VM like so:

```bash
# Download the tulip driver
sudo curl http://downloads.openwrt.org/releases/18.06.1/targets/x86/64/packages/kmod-tulip_4.14.63-1_x86_64.ipk \
-o /tmp/tulip-driver.ipk
```

Now we can copy this file to our OpenWRT disk. 

```bash
# check for the disk we added 
# There should be 2 partitions under that disk now
sudo fdisk -l
```

You should see something like this:

```
Disk /dev/sdb: 500 MiB, 524288000 bytes, 1024000 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xcbad8a62

Device     Boot Start    End Sectors  Size Id Type
/dev/sdb1  *      512  33279   32768   16M 83 Linux
/dev/sdb2       33792 558079  524288  256M 83 Linux
```

We want to copy the tulip driver to the non-boot partition on this disk. 
This is accomplished by mounting the non-boot partition.

```bash
sudo mount /dev/sdb2 /mnt

# you should be able to ls /mnt and see the folder structure
ls /mnt
```

Now, copy the tulip driver to the mounted partition

```bash
sudo cp /tmp/tulip-driver.ipk /mnt/root/tulip-driver.ipk

# Un-mount the partition
sudo umount /dev/sdb2
```

We are now finished with the Linux VM, and you can power it down.

```bash
sudo poweroff
```

### Step 8: Add VHD Back and Add Network Adapters
---

After this, we are done playing hot-potato with this VHD file. 
We can add it to our OpenWRT VM for good this time. 

```powershell
# Back into the host powershell session
Remove-VMHardDiskDrive -VMName $vmname `
  -ControllerType IDE `
  -ControllerNumber 0 `
  -ControllerLocation 1

Add-VMHardDiskDrive -VMName $wrtName `
  -ControllerType IDE `
  -ControllerNumber 0 `
  -ControllerLocation 0 `
  -Path $vhd.Path
```

Now we can add our Hyper-V network adapters to our OpenWRT VM.
You will need to use your own names for you VMNetadapters.

**Ensure you use the IsLegacy parameter with $true**.

```powershell
# Add network adapters to vm
$switch1='External vSwitch'
$switch2='internal1'

Add-VMNetworkAdapter -VMName $wrtName `
  -IsLegacy $true `
  -SwitchName $switch1

Add-VMNetworkAdapter -VMName $wrtName `
  -IsLegacy $true `
  -SwitchName $switch2

# Start your VM
Start-VM -VMName $wrtName
```

### Step 9: Install Network Driver
---

On your OpenWRT VM you can now install your tulip driver, 
and get your networking...working.

```bash
# Inside the openwrt vm
opkg install /root/tulip-driver.ipk
```

Now you can take a look at your network interfaces:

```bash
# you should now have some network adapters visible
ifconfig -a
```

Now you can edit your network config file to setup your networks. 
Do this by editing the /etc/config/network file.

```bash
vi /etc/config/network
```

**SAMPLE FILE**
 
```
config interface 'loopback'
  option ifname 'lo'
  option proto 'static'
  option ipaddr '127.0.0.1'
  option netmask '255.0.0.0'

config globals 'globals'
  option ula_prefix 'fd72:83ef:4007::/48'

config interface 'wan'
  option ifname eth0
  option proto dhcp

config interface 'lan1'
  option type 'bridge'
  option ifname 'eth1'
  option proto 'static'
  option ipaddr '10.0.0.1'
  option netmask '255.255.255.0'
  option ip6assign '60'
```

In this file, I have configured my 'wan' interface, 
which is on the same network as my host machine, 
to use DHCP and be assigned an IP Address from my router. 
My 'lan1' interface is using my internal VM Switch, giving it access
to my VMs using the same VM Switch. 
I will be assigning it a static IP, along with the other network configuration.

<p>
  Once you have your network settings just right, restart the network deamon.
</p>

```bash
# Restart the network deamon
/etc/init.d/network restart

# check the interface settings
ifconfig -a
```

### Step 10: OPTIONAL: Setup https on management GUI (LUCI)
---

This will create a self-signed certificate to use on the web UI, 
so you can use a secure connection (https).

```bash
# Install Luci SSL
opkg install luci-ssl-openssl

# restart uhttp daemon to generate self-sigend cert for the website
/etc/init.d/uhttpd restart
```

Now you can access the LUCI (web UI) over https in your browser.
'https://10.0.0.1' for my example.
