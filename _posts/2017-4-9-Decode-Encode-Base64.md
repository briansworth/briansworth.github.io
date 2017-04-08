---
layout: post
title: String Encoding and Decoding - Base64
---

Here's a quick one.  In this post we'll create 2 basic functions for encoding and decoding strings.
<br>

We will need to use some .net classes for this.
Here's the classes for encoding:
1. [System.Text.Encoding]
2. [System.Convert]

Since [System] is always loaded, it will be omitted.
<br>

### Encoding Strings
----

```powershell
$encodeMe='Gibberish'
$uniBytes=[Text.Encoding]::Unicode.GetBytes($encodeMe)
$encoded=[Convert]::ToBase64String($uniBytes)
Write-Output "Now it really is gibberish -> $encoded"
```

The some process can be used using different encodings, ASCII for example. 

```powershell
$encodeMe='Gibberish'
$asciiBytes=[Text.Encoding]::ASCII.GetBytes($encodeMe)
$encoded=[Convert]::ToBase64String($asciiBytes)
Write-Output "Now it's different gibberish -> $encoded"
```
Pretty straightforward stuff: 
1. Define your string
2. Get the byte array of the string 
3. Convert to Base64
<br>

### Decoding Strings
----

As you may expect, decoding is essentially the same process, just in reverse.

```powershell
# Output from first example
$decodeMe='RwBpAGIAYgBlAHIAaQBzAGgA'
$baseBytes=[Convert]::FromBase64String($decodeMe)
$decoded=[Text.Encoding]::Unicode.GetString($baseBytes)
Write-Output "No longer -> $decoded"
```
And if you wanted to use ASCII:

```powershell
# Output from second example
$decodeMe='R2liYmVyaXNo'
$baseBytes=[Convert]::FromBase64String($decodeMe)
$decoded=[Text.Encoding]::ASCII.GetString($baseBytes)
Write-Output "No longer -> $decoded"
```

Again, this is quite simple to follow:
1. Define your encoded string
2. Get byte array from that string - from Base64
3. Decode the byte array

#### Making functions
----

Let's make a couple functions for encoding and decoding strings.

##### Encode

```powershell
function ConvertTo-EncodedString {
    [CmdletBinding()]
    Param(
        [Parameter(
            Mandatory=$true,
            Position=0,
            ValueFromPipeline=$true
        )]
        [String]$String
    )
    Begin{}
    Process{
        [Byte[]]$uniBytes=[Text.Encoding]::Unicode.GetBytes($String)
        [String]$encode=[Convert]::ToBase64String($uniBytes)
        return $encode
    }
    End{}
}
```

##### Decode

```powershell
function ConvertFrom-EncodedString {
    [CmdletBinding()]
    Param(
        [Parameter(
            Mandatory=$true,
            Position=0,
            ValueFromPipeline=$true
        )]
        [String]$String
    )
    Begin{}
    Process{
        [Byte[]]$baseBytes=[Convert]::FromBase64String($String)
        [String]$decode=[Text.Encoding]::Unicode.GetString($baseBytes)
        return $decode
    }
    End{}
}
```
You could make this much more versatile by adding support for more encoding options like ASCII, but here's a start.
We will also be able to use these tools for future projects.

<br>
*Thanks for reading,*

PS> exit
