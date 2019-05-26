---
layout: post
title: PowerShell - New-VMFromTemplate
description: A simple method and function to deploy a Hyper-V VM from an existing template
---

<p>
Create a VM from a sysprepped image with 1 line of code.
</p>

```powershell
New-VMFromTemplate -VMName dc -TemplatePath 'C:\vm\template\core.vhdx'

Name State CPUUsage(%) MemoryAssigned(M) Uptime   Status             Version
---- ----- ----------- ----------------- ------   ------             -------
dc   Off   0           0                 00:00:00 Operating normally 8.3
```

<p>
  Source code at the end of the post. 
  Jump ahead and grab the code, or come along of the journey.
</p>


### The Journey
----

I made a post on how to create a VM template previously
([Build VM Template]({{ site.baseurl }}/Build-VM-From-Template/)),
but I didn't provide a function that could streamline the process.

<p>
  The problem is ensuring the default behaviour is as optimized as possible. 
  It should also be safe to run 
  so that you don't overwrite files, 
  or do more copying of the large template file then is necessary.
</p>

#### My Defaults
----

<p>
  <strong>VM location</strong><br>
  I don't like using the default Hyper-V VM location. 
  I want all my VMs in the same location, 
  and tucked away in folders named after the VM's name. 
  Example: C:\vm\DC1\
</p>

<p>
  <strong>VHD location</strong><br>
  With my VM files in a standard location, 
  I will store the VHD / VHDX files in the same folder structure. 
  I want the main VHD file to be named the same as the VM's name.
  Example: C:\vm\DC1\DC1.vhdx
</p>

<p>
  <strong>VM Settings</strong><br>
  I have pretty consistent VM settings for servers that I like to use. 
  Settings like Processor Count, Memory, and VM Generation 
  are generally the same across the board, 
  but there will be circumstances where I want to modify these defaults. 
  Putting these settings as parameters should work nicely.
</p>


#### Break down the problem
----

<p>
  Time to break down the problem into its basic components. 
</p>

<p>
  <strong>Problem 1: Order</strong><br>
  We will need to know the VM location before we can run New-VM. 
  The VM folder will also need to exist before we can copy the template file. 
  Most settings can be applied when using New-VM, 
  but some settings need to be applied after the VM is created. 
</p>

<p>
  <strong>Solution:</strong><br>
  Ensure the VM folder structure exists before creating VM, 
  and before copying the template. 
</p>

```powershell
# Example
$vmName='dc'
$vmPath="C:\vm\$vmName"
if(!(Test-Path -Path $vmPath)){
  New-Item -Path $vmPath -ItemType Container
}
```

<p>
  <strong>Problem 2: Templates</strong><br>
  The template is the most important piece to this puzzle. 
  We should validate that the template exists, 
  and that it is a valid template file (at least a VHD or VHDX file). 
</p>
<p>
  <strong>Solution:</strong><br>
  Running Get-VHD on the template path 
  will indicate if it is a valid file type.
</p>

```powershell
$templatePath='C:\vm\template\core.vhdx'
$templateVHD=Get-VHD -Path $templatePath
```

<p>
  <strong>Problem 3: Copying Template</strong><br>
  The template file name will not be the same as 
  what we want it to be. 
  We also need to copy it to the VM path location.
</p>
<p>
  <strong>Solution:</strong><br>
  Ensure the VM folder structure exists first. 
  We should also ensure everything that could go wrong has been checked. 
  We do not want to copy the whole file, 
  only to have some error stop the process short. 
  Once validation is complete, 
  we can copy the file and rename it according to the VM name.
</p>

<p>
  <strong>Problem 4: Settings</strong><br>
  We will need to specify the VM Generation when running New-VM. 
  This setting cannot be modified after it is created.  
  Most other settings can be specified when running New-VM.
</p>
<p>
  <strong>Solution:</strong><br>
  Specify all settings for the VM in the New-VM cmdlet. 
  When the VM is created, use Set-VM for setting ProcessorCount. 
  All these settings should have default values, 
  and be parameters for maximum flexibility.
</p>

### Source Code
----

```powershell
Function New-VMFromTemplate {
  [CmdletBinding()]
  Param(
    [Parameter(Position=0,Mandatory=$true,ValueFromPipeline=$true)]
    [String]$VMName,

    [Parameter(Position=1,Mandatory=$true)]
    [ValidateScript({Test-Path -Path $_ -IsValid})]
    [String]$TemplatePath,

    [Parameter(Position=2)]
    [int64]$MemoryStartupBytes=1GB,

    [Parameter(Position=3)]
    [int16]$ProcessorCount=2,

    [Parameter(Position=4)]
    [String]$SwitchName,

    [Parameter(Position=5)]
    [ValidateRange(1,2)]
    [int16]$Generation=2
  )
  Begin{
    Try{
      Write-Verbose "Validating template..."
      $templatePathExist=Test-Path -Path $TemplatePath
      if(!$templatePathExist){
        Write-Error -Message "Cannot find template path [$TemplatePath]" `
          -Category ObjectNotFound `
          -ErrorAction Stop
      }

      $templateVhd=Get-VHD -Path $TemplatePath -ErrorAction Stop

      if($PSBoundParameters.ContainsKey('SwitchName')){
        $vmSwitch=Get-VMSwitch -Name $SwitchName -ErrorAction SilentlyContinue
        if(!$vmSwitch){
          Write-Error -Message "VM Switch with name [$SwitchName] not found" `
            -Category ObjectNotFound `
            -ErrorAction Stop
        }
      }

    }Catch{
      Write-Error -Exception $_.Exception `
        -Category $_.CategoryInfo.Category

      break
    }
  }
  Process{
    Try{
      Write-Verbose "Validating VM info..."
      # Change this as desired
      $vmDefaultPath="C:\vm\$VMName"

      $newVHDPath="$vmDefaultPath\$($VMName).$($templateVhd.VhdFormat)"
      $vmVHDExists=Test-Path -Path $newVHDPath

      if($vmVHDExists){
        Write-Error -Message "VM VHD already exists [$newVHDPath]" `
          -Category ResourceExists `
          -ErrorAction Stop
      }

      $vmExist=Get-VM -VMName $VMName -ErrorAction SilentlyContinue
      if($vmExist){
        Write-Error -Message "VM already exists [$VMName]" `
          -Category ResourceExists `
          -ErrorAction Stop
      }

      ### Copy template VHD file to target VM Path
      $vmPathExist=Test-Path -Path $vmDefaultPath
      if(!$vmPathExist){
        New-Item -Path $vmDefaultPath -ItemType Container -ErrorAction Stop | 
          Out-Null
      }

      Write-Verbose "Copying template to new VHD path"
      Copy-Item -Path $TemplatePath -Destination $newVHDPath -ErrorAction Stop

      Set-ItemProperty -Path $newVHDPath `
        -Name isReadOnly `
        -Value $false `
        -ErrorAction Stop

      ### Create new VM
      $newVMParameters=@{
        'VMName'=$VMName;
        'Path'=$vmDefaultPath;
        'VHDPath'=$newVHDPath;
        'Generation'=$Generation;
        'MemoryStartupBytes'=$MemoryStartUpBytes;
        'ErrorAction'='Stop';
      }

      if($PSBoundParameters.ContainsKey('SwitchName')){
        $newVMParameters.Add('SwitchName',$SwitchName)
      }

      Write-Verbose "Creating VM [$VMName]"
      $newVM=New-VM @newVMParameters

      if($newVM.ProcessorCount -ne $ProcessorCount){
        $newVM=Set-VM -VMName $VMName `
          -ProcessorCount $ProcessorCount `
          -PassThru `
          -ErrorAction Stop
      }
      Write-Output $newVM
    }Catch{
      Write-Error -Exception $_.Exception `
        -Category $_.CategoryInfo.Category
    }
  }
  End{}
}
```

#### Example 1
----

```powershell
New-VMFromTemplate -VMName dc `
  -TemplatePath C:\vm\template\Gen1Core.vhdx `
  -Generation 1 `
  -SwitchName 'External vSwitch'
```

<p>
  Create a new VM called dc, using the C:\vm\template\Gen1Core.vhdx template. 
  This command assumes that the template is a Generation 1 template.
</p>

#### Example 2
----

```powershell
'dc1','dc2' | New-VMFromTemplate -TemplatePath 'C:\vm\template\core.vhdx' `
  -ProcessorCount 1 `
  -Memory 1.5GB
```

<p>
  Create 2 VMs, dc1 and dc2, using the C:\vm\template\core.vhdx template. 
  Assign 1.5GB for the MemoryStartUpBytes on each VM, 
  and ensure only 1 Virtual CPU is assigned. 
  No VM Network Adapter will be provided when the VMs are made.
</p>

#### Example 3
----

```powershell
New-VMFromTemplate -VMName CM1
  -TemplatePath 'C:\vm\template\gui.vhdx' `
  -Memory 3GB `
  -SwitchName 'Internal0'
```

<p>
  Create a VM using the C:\vm\template\gui.vhdx template. 
  Assign 3GB of memory and assign the 'Internal0' VM Network Adapter.
</p>
