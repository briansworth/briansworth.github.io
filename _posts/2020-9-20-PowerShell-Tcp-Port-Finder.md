---
layout: post
title: Powershell - Simple TCP Port Finder
description: Simple TCP port finder for PowerShell.
---

For when you want to see what TCP ports are in use, or what TCP ports are available within a range.

## Overview

---

The source code is available on 
[GitHub](https://github.com/briansworth/SimpleTcpPortFinder/blob/master/SimpleTcpPortFinder.psm1), 
and it will work on both Windows and Linux.
Keep reading to go on the journey of figuring this out.

### Source code examples

#### List all active TCP port numbers

```powershell
Get-ActiveTcpPortNumber

53
22
...
55868
```

#### Find a random available port in a range

```powershell
Get-InactiveTcpPortNumber -StartRange 20000 -EndRange 30000 -Random

20768
```

## The Journey

---

Being able to list available or inactive TCP ports
relies on being able to identify what ports are currently in use. 
There are obviously tools that already do this (netstat or ss for example), 
but running and parsing these utilities can become difficult and 
tiresome in some cases.
Finding a 'PowerShell way' of getting this information, 
will make things better in the long run.

### Find active ports
---

Getting the active ports is relatively straight forward. 
There is a class that makes it easy, `IPGlobalProperties`.

Construct this class like so:
```powershell
$properties = [Net.NetworkInformation.IPGlobalProperties]::GetIPGlobalProperties()

$properties | Get-Member -Name *Tcp*
```

As you can see from the output of the command above,
there are some useful methods 
that may be able to give us the information we need.

Get details about the ports assigned to active TCP connections:

```powershell
$properties.GetActiveTcpConnections()
```

This doesn't produce a complete list of TCP ports that are in use.
There are still ports assigned to services / listeners
that are not in this list. 
Those will need to be included to get a full list of unavailable ports.

Get information about the TCP ports assigned to listeners:

```powershell
$properties.GetActiveTcpListeners()
```

With these 2 lists, 
we have a complete picture about the TCP ports on the system that are in use,
and unavailable for our purposes.

Here is a simple function to combine these into a single output:

```powershell
Function Get-ActiveTcpPort
{
  # Use a hash set to avoid duplicates
  $portList = New-Object -TypeName Collections.Generic.HashSet[uint16]

  $properties = [Net.NetworkInformation.IPGlobalProperties]::GetIPGlobalProperties()

  $listener = $properties.GetActiveTcpListeners()
  $active = $properties.GetActiveTcpConnections()

  foreach ($serverPort in $listener)
  {
    [void]$portList.Add($serverPort.Port)
  }
  foreach ($clientPort in $active)
  {
    [void]$portList.Add($clientPort.LocalEndPoint.Port)
  }

  return $portList
}

Get-ActiveTcpPort
```

### Inactive ports

---

With the ability to get all active TCP ports, the heavy lifting is done. 
Now we can simply get a port that is not in this active list and we are done.

```powershell
Function Get-InactiveTcpPort
{
  [CmdletBinding()]
  Param(
    [Parameter(Position=0)]
    [uint16]$Start = 1024,

    [Parameter(Position=1)]
    [uint16]$End = 5000
  )
  $attempts = 100
  $counter = 0

  $activePorts = Get-ActiveTcpPort

  while ($counter -lt $attempts)
  {
    $counter++
    $port = Get-Random -Minimum ($Start -as [int]) -Maximum ($End -as [int])

    if ($port -notin $activePorts)
    {
      return $port
    }
  }
  $emsg = [string]::Format(
    'Unable to find available TCP Port. Range: {0}, Attempts: [{1}]',
    "[$Start - $End]",
    $attempts
  )
  throw $emsg
}

Get-InactiveTcpPort
```

With very minimal code, we have something up and running that works.
There is plenty of room for added features and better design,
but for the simple use-case of finding a free port, it works pretty well.

## Conclusion

---

There are some limitations to this approach.
For example, it will always pick a random port for a given range.
There may be a requirement to pick the first available port in a range, 
and this approach will not cater to that.

Additionally, with always picking a random value within a provided range, 
it may choose the same port multiple times.  
This only gets worse if a small port range is specified.
Even more so if there are one, or no ports available in a small range,
as it will continue to guess randomly until the max attempt counter is exceeded.

For a more polished version, check out my 
[SimpleTcpPortFinder](https://github.com/briansworth/SimpleTcpPortFinder) 
repository on Github.
It has fixes for all of the issues mentioned above.

