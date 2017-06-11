---
layout: post
title: PowerShell and JSON Configuration Files
---
<p>
  If you run a lot of code in any environment, or have multiple environments that you manage,
  configuration files are incredibly useful. 
  If you do not already use them, you should seriously consider it.
  JSON is my preferred format this task, and here is how I use it in PowerShell. 
</p>
<p>
  In the Windows world, it seems that these files are typically created in 
  eXtensible Markup Language (XML).  
  Xml will work, and if you are stuck with PowerShell version 2.0 or earlier, 
  you may have to stick with that.
</p>

<p>
  For those of us living in a more modern world, 
  JavaScript Object Notation (JSON) is the way to go.  
  It is is a much more human readable format than xml, 
  and it is incredibly lightweight. 
  JSON is only increasing in popularity and is simple and easy to work with.
  If you haven't used it before, this should be enough to get you started.
</p>

Example:
**XML**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<hyperv>
  <vmpath>V:\VMs</vmpath>
  <templatepath>V:\Templates</templatepath>
  <vmswitch> 
    <external>External vSwitch</external>
    <internal>Internal1 vSwitch</internal>
    <internal>Internal2 vSwitch</internal>
    <private>>Private vSwitch</private>
  </vmswitch>
  <support>
    <email>brian@codeAndKeep.com</email>
    <phone>555-555-5555</phone>
  </support>
</hyperv>
```
Here's a simple xml example with some info for a Hyper-V config file.  
<br>
Now let's see the same info in JSON.

**JSON**
```
{
  "hyperv": {
    "vmpath": "V:\\VMs",
    "templatepath": "V:\\Templates",
    "vmswitch": {
      "external": "External vSwitch",
      "internal": [
        "Internal1 vSwitch",
        "Internal2 vSwitch"
      ],
      "private": "Private vSwitch"
    },
    "support": {
      "email": "brian@codeAndKeep.com",
      "phone": "555-555-5555"
    }
  }
}
```
<p>
  The way I have formatted this example makes it technically longer in terms of lines, 
  but it contains the same information in significantly fewer characters (421 vs 360; ~15% less).  
  This means less clutter, and better readability.
  As you can see it is just as extensible as xml.
  Adding and removing data is straightforward, and won't interfere with what is already there.
</p>
### Working with JSON
----
<p>
  I have shown you how you can create a JSON (.json) document, 
  but how can you manipulate and work with this file type in PowerShell?
</p>  
<p>
  I have saved this configuration file in my current working directory as hypervConf.json.
  If I want to actually use it, I will need to load it into memory.
  This is where you may run into issues if you are using anything before PowerShell v3.
  There are new(ish) Cmdlets that allow you to use JSON effectively in PowerShell v3 and up.
</p>

```powershell
$config=Get-Content -Path .\hypervConf.json -Raw |
  ConvertFrom-Json
# ConvertFrom-Json is only available by default in PSv3 and up 
```
That's it.  
<p>
  You can now manupilate this object like any PSObject in PowerShell.
  If you wanted to use the 'templatepath' for example just type $config.hyperv.templatepath.
  If you want the list of internal VMSwitches; $config.hyperv.vmswitch.internal.
</p>

<p>
  You can also load this data into memory by simply pasting it into your shell as a big string,
  then piping it to the same command ConvertFrom-Json:
</p>
```powershell
# just copy and paste this into your shell
$configData=@"
{
  "hyperv": {
    "vmpath": "V:\\VMs",
    "templatepath": "V:\\Templates",
    "vmswitch": {
      "external": "External vSwitch",
      "internal": [
        "Internal1 vSwitch",
        "Internal2 vSwitch"
      ],
      "private": "Private vSwitch"
    },
    "support": {
      "email": "brian@codeAndKeep.com",
      "phone": "555-555-5555"
    }
  }
}
"@
$config=$configData | ConvertFrom-Json
```

#### What about xml?
<p>
  If you want to do the same kind of thing with xml, you can do the following.
</p>
```powershell
[xml]$config=Get-Content -Path .\hypervConf.xml
```
<p>
  Now you can manipulate this data the same way as you would any other PSObject.
</p>
