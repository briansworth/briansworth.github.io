---
layout: post
title: Get List of Installed Applications
---

<p>
  Getting a list of installed applications seems like something 
  a lot of Windows admins would like to do.<br>
  Unfortuneately, there isn't an Out-of-the-Box way to do this with PowerShell.
</p>

### Common ways of listing applications
-----

#### Win32_Product
<p>
  The most common method that I have seen is a simple WMI query to the
  Win32 Product class.
</p>

<div class="language-powershell highlighter-rouge">
  <div class="highlight">
    <pre>
      <code>
        <span class="nb">gwmi </span>Win32Reg_AddRemovePrograms
      </code>
    </pre>
  </div>
</div>

<p>
  The first thing you will notice about this method, 
  is that it takes a very long time to populate the list.
  Also, this will only retreive MSI installed applications.
  Anything installed by another method (like exe) will not show up here.
</p>

#### Win32Reg_AddRemoveProduct
This class is much better than the above; but,
it will **only exist if your machine or the target machine has
an SCCM/SMS client installed on it.**<br>
This class is much faster at retrieve this info, 
although I have not used it very much because of this prerequisite.

```powershell
gwmi Win32Reg_AddRemovePrograms
```

------
<p>
  The great thing about these methods is that they use the
  Get-WmiObject command.
  This command makes it incredibly easy to query other computers,
  and use alternate credentials.
</p>

<br>

### The Better Way
-----
<p>
  The better way to get this information would be to use the registry.
  When an application is installed (the Windows way), 
  It creates a key in 1 or 2 locations in the registry
  depending on its architecture.
</p>

<p>
  While it's not as easy as a one line WMI call, 
  it is not too difficult to get this information with Get-ChildItem.
</p>
The 2 locations are as follows:

1. HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\ 
  * For 32-bit applications
2. HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\
  * For 64-bit applications

```powershell
$paths=@(
  'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\',
  'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\'
)
foreach($path in $paths){
  Get-ChildItem -Path $path | 
    Get-ItemProperty | 
      Select DisplayName, Publisher, InstallDate, DisplayVersion
}
```
*You may want to use Format-List to view the data nicely.*
<br>

<p>
  As you can see, this is both fast and simple.
  There are many other properties you can select as well.
  If your environment is setup to use PowerShell remoting,
  you can simply pass the example above to the Invoke-Command command.
</p>

<p>
  In environments that do not have PowerShell remoting setup,
  there is a way to connect to a remote registry on other machines.
  It isn't as straightforward, so I have made it into a nice function
  that should take out that complexity.
</p>

```powershell
Function Get-InstalledApplication {
  [CmdletBinding()]
  Param(
    [Parameter(
      Position=0,
      ValueFromPipeline=$true,
      ValueFromPipelineByPropertyName=$true
    )]
    [String[]]$ComputerName=$ENV:COMPUTERNAME,

    [Parameter(Position=1)]
    [String[]]$Properties,

    [Parameter(Position=2)]
    [String]$IdentifyingNumber,

    [Parameter(Position=3)]
    [String]$Name,

    [Parameter(Position=4)]
    [String]$Publisher
  )
  Begin{
    $regPath = @(
      'SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall',
      'SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall'
    )
  }
  Process{
    foreach($computer in $computerName){
      Try{
        $hive=[Microsoft.Win32.RegistryKey]::OpenRemoteBaseKey(
          [Microsoft.Win32.RegistryHive]::LocalMachine, 
          $computer
        )
        if(!$hive){
          continue
        }
        foreach($path in $regPath){
          $key=$hive.OpenSubKey($path)
          if(!$key){
            continue
          }
          foreach($subKey in $key.GetSubKeyNames()){
            $subKeyObj=$null
            if($PSBoundParameters.ContainsKey('IdentifyingNumber')){
              if($subKey -ne $IdentifyingNumber -and 
                $subkey.TrimStart('{').TrimEnd('}') -ne $IdentifyingNumber){
                continue
              }
            }
            $subKeyObj=$key.OpenSubKey($subKey)
            if(!$subKeyObj){
              continue
            }
            $outHash=New-Object -TypeName Collections.Hashtable
            $appName=[String]::Empty
            $appName=($subKeyObj.GetValue('DisplayName'))
            if($PSBoundParameters.ContainsKey('Name')){
              if($appName -notlike $name){
                continue
              }
            }
            if($appName){
              if($PSBoundParameters.ContainsKey('Properties')){
                foreach ($prop in $Properties){
                  $outHash.$prop=($hive.OpenSubKey("$path\$subKey")).GetValue($prop)
                }
              }
              $outHash.Name=$appName
              $outHash.IdentifyingNumber=$subKey
              $outHash.Publisher=$subKeyObj.GetValue('Publisher')
              if($PSBoundParameters.ContainsKey('Publisher')){
                if($outHash.Publisher -notlike $Publisher){
                  continue
                }
              }
              $outHash.ComputerName=$computer
              $outHash.Path=$subKeyObj.ToString()
              New-Object -TypeName PSObject -Property $outHash
            }
          }
        }
      }Catch{
        Write-Error $_
      }
    }
  }
  End{}
}
```

This is a fair bit of code, but using it is pretty straight forward.

#### Example 1
```powershell
Get-InstalledApplication
```
This will give you all the installed apps on the current computer
(assuming you have the necessary permissions).

#### Example 2
```powershell
Get-InstalledApplication -ComputerName Computer1 -Name "Google Chrome"
```
This will seach Computer1 for an application named Google Chrome.
Wildcards are also accepted.

#### Example 3
```powershell
Get-InstalledApplication -ComputerName Server1, Server2 -Publisher 'Microsoft*'
```
As you might expect, this will list all applications installed on both
server1 and server2 that are published my Microsoft.

#### Example 4
```powershell
Get-InstalledApplication -ComputerName Server1, Server2 `
  -Name '7-zip*' `
  -Properties DisplayVersion, UninstallString, InstallDate
```
This will query Server1&2 for 7-zip, and include the listed properties,
in the result.

#### Example 5
```powershell
Get-InstalledApplication -ComputerName server1,server2,server3 `
  -IdentifyingNumber {5FCE6D76-F5DC-37AB-B2B8-22AB8CEDB1D4}
```
If you know the guid of an application, you can specify IdentifyingNumber.
