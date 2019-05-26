---
layout: post
title: PowerShell - Get Network ID and Subnet Info
description: How to guide to get network / subnet information from an IP address and a subnet mask. Also includes a function that can do this, and more.
---

<p>
  I wanted a simple command that would give me the basic subnet 
  information I commonly want.
  Identifying the Network ID was tedious, 
  as was converting an IP / Subnet to CIDR notation and vice-versa. 
  Now I have one command that can do it all.
</p>

### Example 1
----

```powershell
Get-IPv4Subnet -IPAddress 192.168.153.0 -SubnetMask 255.255.128.0


CidrID       : 192.168.128.0/17
NetworkID    : 192.168.128.0
SubnetMask   : 255.255.128.0
PrefixLength : 17
HostCount    : 32766
FirstHostIP  : 192.168.128.1
LastHostIP   : 192.168.255.254
Broadcast    : 192.168.255.255
```

### Example 2
----

```powershell
Get-IPv4Subnet -IPAddress 10.148.72.201 -PrefixLength 26


CidrID       : 10.148.72.192/26
NetworkID    : 10.148.72.192
SubnetMask   : 255.255.255.192
PrefixLength : 26
HostCount    : 62
FirstHostIP  : 10.148.72.193
LastHostIP   : 10.148.72.254
Broadcast    : 10.148.72.255
```

<p>
  The source code will be at the end of the post.
</p>

### The Journey
----

<p>
  I have recently starting working more with networking, 
  which is not something I have spent a lot of time doing. 
  I was constantly having to use online subnet calculators to 
  translate what subnet IP addresses belonged to. 

  Getting the prefix length from a subnet mask, 
  and the subnet mask from a prefix length 
  was also something I wanted to look into.
</p>

#### Convert subnet mask to prefix length

<p>
  How does 255.255.255.0 translate to a prefix length of 24? 
  Its pretty simple really; 
  the first 24 bits of the 32 bits in the IPv4 address are 1, 
  and the remaining bits are all 0.
</p>

<p>
  Ex: 11111111111111111111111100000000
</p>

<p>
  If we can get the bytes from the subnet mask, 
  we should be able to determine how many leading 1s there are 
  and get our prefix length.
</p>

```powershell
# Convert to an IPAddress object
$netMaskIP=[IPAddress]'255.255.255.0'

# Convert to byte array
$netMaskIP.GetAddressBytes()
```
<p>
  Now that we have a way of getting the bytes of an IP address, 
  we are almost done.
</p>

```powershell
$netMaskIP=[IPAddress]'255.255.255.0'

$binaryString=[String]::Empty
$netMaskIP.GetAddressBytes() | Foreach {
  # combine each
  $binaryString+=[Convert]::ToString($_, 2)
}

# remove the trailing 0s since we only care about the 1s, and get the length
Write-Output $binaryString.TrimEnd('0').Length
```

#### Convert prefix length to subnet mask

<p>
  This one is pretty easy armed with the knowledge we have from the previous 
  example. 
  If the prefix length is 24, 
  that means the first 24 bits are 1s, and the rest are 0. 
</p>

```powershell
$prefixLength=24

# create as many 1s as the prefix length
# use PadRight to add the needed 0s required to make it 32 long
('1' * $prefixLength).PadRight(32, '0')
```

<p>
  The only thing left is to convert this string to an IP address. 
  We can split the string into 4 separate strings of 8
  to make the required 4 bytes. 
  Then convert the byte strings into the integer equivalent.
</p>

```powershell
$bitString=('1' * $prefixLength).PadRight(32,'0')

$ipString=[String]::Empty

# make 1 string combining a string for each byte and convert to int
for($i=0;$i -lt 32;$i+=8){
  $byteString=$bitString.Substring($i,8)
  $ipString+="$([Convert]::ToInt32($byteString, 2))."
}

Write-Output $ipString.TrimEnd('.')
```

#### Get network ID

<p>
  As long as you have an IP address and a prefix length or subnet mask, 
  you  can find the network ID. 
  The subnet mask will determine the number of bits assigned to the network. 
  These same bits in the IP address provided 
  will stay the same in the network Id. 
  The remaining bits will be 0s. 
</p>

<p>
  If the subnet mask is 255.255.255.128 and the IP is 192.168.24.71:
</p>

Mask:  1111111111111111111111111**0000000**

IP:    1100000010101000110000001**0001110**

NetID: 1100000010101000110000001**0000000**

<p>
  So the network ID is equivalent to all of the 1s that match 
  between the subnet mask, and the provided IP address. 
  So this means that we can do a 'bitwise AND' comparison of the 
  subnet mask and IP address to get the network ID.
</p>

<p>
  If we convert the subnet mask and the IP address to integers, 
  we can do a bitwise AND comparison, 
  and convert the output back into an IP to get our network ID:
</p>

```powershell
# The ConvertIPv4ToInt and ConvertIntToIPv4 functions are  in the source code
$maskInt=ConvertIPv4ToInt -IPv4Address 255.255.255.128
$ipInt=ConvertIPv4ToInt -IPv4Address 192.168.24.71

$netIdInt=$maskInt -band $ipInt
ConvertIntToIPv4 -Integer $netIdInt
```

<br>

### Source Code
----

<p>
  The main function is of course Get-IPv4Subnet. 
  One other function in here that I like is Convert-IPv4AddressToBinaryString. 
  It will visually represent the binary form of an IP address. 
</p>

```powershell
Function Convert-IPv4AddressToBinaryString {
  Param(
    [IPAddress]$IPAddress='0.0.0.0'
  )
  $addressBytes=$IPAddress.GetAddressBytes()

  $strBuilder=New-Object -TypeName Text.StringBuilder
  foreach($byte in $addressBytes){
    $8bitString=[Convert]::ToString($byte,2).PadRight(8,'0')
    [void]$strBuilder.Append($8bitString)
  }
  Write-Output $strBuilder.ToString()
}

Function ConvertIPv4ToInt {
  [CmdletBinding()]
  Param(
    [String]$IPv4Address
  )
  Try{
    $ipAddress=[IPAddress]::Parse($IPv4Address)

    $bytes=$ipAddress.GetAddressBytes()
    [Array]::Reverse($bytes)

    [System.BitConverter]::ToUInt32($bytes,0)
  }Catch{
    Write-Error -Exception $_.Exception `
      -Category $_.CategoryInfo.Category
  }
}

Function ConvertIntToIPv4 {
  [CmdletBinding()]
  Param(
    [uint32]$Integer
  )
  Try{
    $bytes=[System.BitConverter]::GetBytes($Integer)
    [Array]::Reverse($bytes)
    ([IPAddress]($bytes)).ToString()
  }Catch{
    Write-Error -Exception $_.Exception `
      -Category $_.CategoryInfo.Category
  }
}

Function Add-IntToIPv4Address {
  Param(
    [String]$IPv4Address,

    [int64]$Integer
  )
  Try{
    $ipInt=ConvertIPv4ToInt -IPv4Address $IPv4Address `
      -ErrorAction Stop
    $ipInt+=$Integer

    ConvertIntToIPv4 -Integer $ipInt
  }Catch{
    Write-Error -Exception $_.Exception `
      -Category $_.CategoryInfo.Category
  }
}

Function CIDRToNetMask {
  [CmdletBinding()]
  Param(
    [ValidateRange(0,32)]
    [int16]$PrefixLength=0
  )
  $bitString=('1' * $PrefixLength).PadRight(32,'0')

  $strBuilder=New-Object -TypeName Text.StringBuilder

  for($i=0;$i -lt 32;$i+=8){
    $8bitString=$bitString.Substring($i,8)
    [void]$strBuilder.Append("$([Convert]::ToInt32($8bitString,2)).")
  }

  $strBuilder.ToString().TrimEnd('.')
}

Function NetMaskToCIDR {
  [CmdletBinding()]
  Param(
    [String]$SubnetMask='255.255.255.0'
  )
  $byteRegex='^(0|128|192|224|240|248|252|254|255)$'
  $invalidMaskMsg="Invalid SubnetMask specified [$SubnetMask]"
  Try{
    $netMaskIP=[IPAddress]$SubnetMask
    $addressBytes=$netMaskIP.GetAddressBytes()

    $strBuilder=New-Object -TypeName Text.StringBuilder

    $lastByte=255
    foreach($byte in $addressBytes){

      # Validate byte matches net mask value
      if($byte -notmatch $byteRegex){
        Write-Error -Message $invalidMaskMsg `
          -Category InvalidArgument `
          -ErrorAction Stop
      }elseif($lastByte -ne 255 -and $byte -gt 0){
        Write-Error -Message $invalidMaskMsg `
          -Category InvalidArgument `
          -ErrorAction Stop
      }

      [void]$strBuilder.Append([Convert]::ToString($byte,2))
      $lastByte=$byte
    }

    ($strBuilder.ToString().TrimEnd('0')).Length
  }Catch{
    Write-Error -Exception $_.Exception `
      -Category $_.CategoryInfo.Category
  }
}

Function Get-IPv4Subnet {
  [CmdletBinding(DefaultParameterSetName='PrefixLength')]
  Param(
    [Parameter(Mandatory=$true,Position=0)]
    [IPAddress]$IPAddress,

    [Parameter(Position=1,ParameterSetName='PrefixLength')]
    [Int16]$PrefixLength=24,

    [Parameter(Position=1,ParameterSetName='SubnetMask')]
    [IPAddress]$SubnetMask
  )
  Begin{}
  Process{
    Try{
      if($PSCmdlet.ParameterSetName -eq 'SubnetMask'){
        $PrefixLength=NetMaskToCidr -SubnetMask $SubnetMask `
          -ErrorAction Stop
      }else{
        $SubnetMask=CIDRToNetMask -PrefixLength $PrefixLength `
          -ErrorAction Stop
      }
      
      $netMaskInt=ConvertIPv4ToInt -IPv4Address $SubnetMask     
      $ipInt=ConvertIPv4ToInt -IPv4Address $IPAddress
      
      $networkID=ConvertIntToIPv4 -Integer ($netMaskInt -band $ipInt)

      $maxHosts=[math]::Pow(2,(32-$PrefixLength)) - 2
      $broadcast=Add-IntToIPv4Address -IPv4Address $networkID `
        -Integer ($maxHosts+1)

      $firstIP=Add-IntToIPv4Address -IPv4Address $networkID -Integer 1
      $lastIP=Add-IntToIPv4Address -IPv4Address $broadcast -Integer -1

      if($PrefixLength -eq 32){
        $broadcast=$networkID
        $firstIP=$null
        $lastIP=$null
        $maxHosts=0
      }

      $outputObject=New-Object -TypeName PSObject 

      $memberParam=@{
        InputObject=$outputObject;
        MemberType='NoteProperty';
        Force=$true;
      }
      Add-Member @memberParam -Name CidrID -Value "$networkID/$PrefixLength"
      Add-Member @memberParam -Name NetworkID -Value $networkID
      Add-Member @memberParam -Name SubnetMask -Value $SubnetMask
      Add-Member @memberParam -Name PrefixLength -Value $PrefixLength
      Add-Member @memberParam -Name HostCount -Value $maxHosts
      Add-Member @memberParam -Name FirstHostIP -Value $firstIP
      Add-Member @memberParam -Name LastHostIP -Value $lastIP
      Add-Member @memberParam -Name Broadcast -Value $broadcast

      Write-Output $outputObject
    }Catch{
      Write-Error -Exception $_.Exception `
        -Category $_.CategoryInfo.Category
    }
  }
  End{}
}
```

Additionally, I have put this source code, along with help, on [GitHub](https://github.com/briansworth/GetIPv4Address/blob/master/GetIPv4Subnet.psm1).

#### Example Script
----

<p>
  Now a quick example of a script to get NIC configuration information 
  on a computer using this function:
</p>

```powershell
$nics=[Net.NetworkInformation.NetworkInterface]::GetAllNetworkInterfaces()

foreach($interface in $nics){
  if($interface.NetworkInterfaceType -eq 'Loopback'){
    continue
  }
  if(!($interface.Supports('IPv4'))){
    continue
  }
  $ipProperties=$interface.GetIPProperties()

  $ipv4Properties=$ipProperties.GetIPv4Properties()

  $ipProperties.UnicastAddresses | Foreach {
    if(!($_.Address.IPAddressToString)){
      continue
    }
    if($ipProperties.GatewayAddresses){
      $gateway=$ipProperties.GatewayAddresses.Address.IPAddressToString
    }else{
      $gateway=$null
    }

    $subnetInfo=Get-IPv4Subnet -IPAddress $_.Address.IPAddressToString `
      -PrefixLength $_.PrefixLength
    New-Object -TypeName PSObject -Property @{
      InterfaceName=$interface.Name;
      InterfaceType=$interface.NetworkInterfaceType;
      NetworkID=$subnetInfo.NetworkID;
      IPAddress=$_.Address.IPAddressToString;
      SubnetMask=$subnetInfo.SubnetMask;
      CidrID=$subnetInfo.CidrID;
      DnsAddresses=$ipProperties.DnsAddresses.IPAddressToString;
      GatewayAddresses=$gateway;
      DhcpEnabled=$ipv4Properties.IsDhcpEnabled;
    }
  }
}
```
