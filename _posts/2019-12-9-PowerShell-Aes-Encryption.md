---
layout: post
title: PowerShell - Aes Encryption
description: Encrypt with Aes in PowerShell.
---

This post will go over a way to implement AES encryption with PowerShell.
It will allow you to protect a string with a password.

#### Encryption Example
----

Create a password as a secure string, and encrypt it.

```powershell
$password = Read-Host -AsSecureString
$encrypted = Protect-AesString -String 'Encrypt me!_!' -Password $password

$encrypted.CipherText
ag7sGO01sz/CXi7mHyK97A=
```

#### Decryption Example
----

Decrypt the text from above using the same password.

```powershell
$encrypted | Unprotect-AesString -Password $password
Encrypt me!_!
```

----

As always, you can jump ahead for the source code
(or find it on [Github](https://github.com/briansworth/AesEncryption/) ),
or stick along for the journey of figuring this out.


### The Journey
----

First things first, we need to generate a key to use for the encryption. 
As mentioned above, I want to use a password to protect my string. 
So we need to generate a key using a password.

#### Derive key from password
----

We can leverage the class Rfc2898DeriveBytes in .Net for this.
To contruct this class we will need a salt, a password, 
and some other things if more customization is needed.

The salt needs to be represented in a byte array

```powershell
# Needs to be at least 8 bytes in length
$salt = 'Salty123'
$saltBytes = [Text.Encoding]::UTF8.GetBytes($salt)
```

Armed with our salt bytes, we can derive our password bytes.

```powershell
$password = 'P@ssw0rd1'
$passDerive = New-Object Security.Cryptography.Rfc2898DeriveBytes `
  -ArgumentList @($password, $saltBytes)
```

If you take a look at the variable $passDerive,
you will see the default hash algorithm is SHA1, 
and the number of iterations is 1000.
This can be customized when constructing the object.

We can specify the SHA256 hash algorithm and interation count like so:

```powershell
$iterations = 1000
$passDerive = New-Object Security.Cryptography.Rfc2898DeriveBytes `
  -ArgumentList @($password, $saltBytes, $iterations, 'SHA256')
```

Now we can get our key out of this class.

```powershell
$keySize = 256
$key = $passDerive.GetBytes($keySize / 8)
```

We have a key generated from our password. Fantastic!
<p>
  You could generate a key using the raw password, 
  but that is not the secure way of doing it. 
  Using this class we introduce entropy to make the process cryptographically sound.
</p>


#### Encrypt
----

In .Net there are easy to use classes to handle the difficult stuff. 
We can leverage the SymmetricAlgorithm class to create an object 
that allows for encryption.  
Using this class also allows for creating different objects for managing 
different algorithms (like TripleDES, RC2, etc.).

```powershell
$cipher = [Security.Cryptography.SymmetricAlgorithm]::Create('AesManaged')
Write-Output $cipher
```

You will notice that there is already a key created, 
along with default values for key size, mode, and padding. 
Additionally you will see the IV (initilization vector) property.
When using the CBC cipher mode (which is what we are using), 
it is best to generate a unique IV each time we encrypt something. 
Luckily, this class generates one each time it is initialized, 
but we will need to know what the IV is in order to decrypt it.


Now to actually use this thing to encrypt some data.
We will create an encryptor object using our key from above, 
and the pre-generated IV.

```powershell
$iv = $cipher.IV
$encryptor = $cipher.CreateEncryptor($key, $iv)

$memoryStream = New-Object -TypeName IO.MemoryStream
$cryptoStream = New-Object -TypeName Security.Cryptography.CryptoStream `
  -ArgumentList @( $memoryStream, $encryptor, 'Write' )
```

We now have a crypto stream object, ready with our key, 
to write to our memory stream.
Lets encrypt.

```powershell
$string = 'Hello, World!'
$strBytes = [Text.Encoding]::UTF8.GetBytes($string)

$cryptoStream.Write($strBytes, 0, $strBytes.Length)
$cryptoStream.FlushFinalBlock()
$encryptedBytes = $memoryStream.ToArray()

# Base64 Encode the encrypted bytes to get a string
$encryptedString = [Convert]::ToBase64String($encryptedBytes)
Write-Output $encryptedString
```

Looks like some random gibberish to me. Excellent.

Something to note. 
You can re-run the code (starting with generating a new cipher), 
and you will get a completely different encrypted string. 
Pretty neat!

#### Decrypt
----

We will re-use the cipher we generated above, 
since it already has the properties that we need. 
The process for decrypting is similar to the one used for encrypting.

To create the decryptor, we will also need the key we generated above.

```powershell
$decryptor = $cipher.CreateDecryptor($key, $iv)

# Get the bytes from the encrypted string
$bytes = [Convert]::FromBase64String($encryptedString)

# Create memory and crypto stream
$stream = New-Object -TypeName IO.MemoryStream `
  -ArgumentList @(, $encryptedBytes)
$cryptoReader = New-Object -TypeName Security.Cryptography.CryptoStream `
  -ArgumentList @( $stream, $decryptor, 'Read' )
```

We are now setup to read from the crypto stream using our decryptor.

```powershell
$decrypted = New-Object -TypeName Byte[] -ArgumentList $bytes.Length
$decryptedByteCount = $cryptoReader.Read($decrypted, 0, $decrypted.Length)
$decryptedString = [Text.Encoding]::UTF8.GetString($decrypted, 0, $decryptedByteCount)
Write-Output $decryptedString
```

There you have it, AES encryption and decryption in PowerShell.
For pre-made functions that do exactly this, see below.

### The Code

----

There is a fair bit of code needed to pull this off. 
The source code is available on 
[Github](https://github.com/briansworth/AesEncryption/) and below.
Cryptography is complicated, and I have done my best to implement best practices.

```powershell
Function DecryptSecureString
{
  Param(
    [SecureString]$SecureString
  )
  $bstr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($SecureString)
  $plain = [Runtime.InteropServices.Marshal]::PtrToStringAuto($bstr)
  return $plain
}

Function NewPasswordKey 
{
  [CmdletBinding()]
  Param(
    [SecureString]$Password,

    [String]$Salt
  )
  $saltBytes = [Text.Encoding]::ASCII.GetBytes($Salt) 
  $iterations = 1000
  $keySize = 256

  $clearPass = DecryptSecureString -SecureString $Password
  $passwordType = 'Security.Cryptography.Rfc2898DeriveBytes'
  $passwordDerive = New-Object -TypeName $passwordType `
    -ArgumentList @( 
      $clearPass, 
      $saltBytes, 
      $iterations,
      'SHA256'
    )

  $keyBytes = $passwordDerive.GetBytes($keySize / 8)
  return $keyBytes
}

Class CipherInfo
{
  [String]$CipherText
  [Byte[]]$IV
  [String]$Salt

  CipherInfo([String]$CipherText, [Byte[]]$IV, [String]$Salt)
  {
    $this.CipherText = $CipherText
    $this.IV = $IV
    $this.Salt = $Salt
  }
}

Function Protect-AesString 
{
  [CmdletBinding()]
  Param(
    [Parameter(Position=0, Mandatory=$true, ValueFromPipeline=$true)]
    [String]$String,

    [Parameter(Position=1, Mandatory=$true)]
    [SecureString]$Password,

    [Parameter(Position=2)]
    [String]$Salt = 'qtsbp6j643ah8e0omygzwlv9u75xcfrk4j63fdane78w1zgxhucsytkirol0v25q',

    [Parameter(Position=3)]
    [Security.Cryptography.PaddingMode]$Padding = 'PKCS7'
  )
  Try 
  {
    $valueBytes = [Text.Encoding]::UTF8.GetBytes($String)
    [byte[]]$keyBytes = NewPasswordKey -Password $Password -Salt $Salt

    $cipher = [Security.Cryptography.SymmetricAlgorithm]::Create('AesManaged')
    $cipher.Mode = [Security.Cryptography.CipherMode]::CBC
    $cipher.Padding = $Padding
    $vectorBytes = $cipher.IV

    $encryptor = $cipher.CreateEncryptor($keyBytes, $vectorBytes)
    $stream = New-Object -TypeName IO.MemoryStream
    $writer = New-Object -TypeName Security.Cryptography.CryptoStream `
      -ArgumentList @(
        $stream,
        $encryptor,
        [Security.Cryptography.CryptoStreamMode]::Write
      )

    $writer.Write($valueBytes, 0, $valueBytes.Length)
    $writer.FlushFinalBlock()
    $encrypted = $stream.ToArray()

    $cipher.Clear()
    $stream.SetLength(0)
    $stream.Close()
    $writer.Clear()
    $writer.Close()
    $encryptedValue = [Convert]::ToBase64String($encrypted)
    New-Object -TypeName CipherInfo `
      -ArgumentList @($encryptedValue, $vectorBytes, $Salt)
  }
  Catch
  {
    Write-Error $_
  }
}

Function Unprotect-AesString 
{
  [CmdletBinding(DefaultParameterSetName='String')]
  Param(
    [Parameter(Position=0, Mandatory=$true, ParameterSetName='String')]
    [Alias('EncryptedString')]
    [String]$String,

    [Parameter(Position=1, Mandatory=$true)]
    [SecureString]$Password,

    [Parameter(Position=2, ParameterSetName='String')]
    [String]$Salt = 'qtsbp6j643ah8e0omygzwlv9u75xcfrk4j63fdane78w1zgxhucsytkirol0v25q',

    [Parameter(Position=3, Mandatory=$true, ParameterSetName='String')]
    [Alias('Vector')]
    [Byte[]]$InitializationVector,

    [Parameter(Position=0, Mandatory=$true, ParameterSetName='CipherInfo', ValueFromPipeline=$true)]
    [CipherInfo]$CipherInfo,

    [Parameter(Position=3, ParameterSetName='String')]
    [Parameter(Position=2, ParameterSetName='CipherInfo')]
    [Security.Cryptography.PaddingMode]$Padding = 'PKCS7'
  )
  Process
  {
    Try
    {
      if ($PSCmdlet.ParameterSetName -eq 'CipherInfo')
      {
        $Salt = $CipherInfo.Salt
        $InitializationVector = $CipherInfo.IV
        $String = $CipherInfo.CipherText
      }
      $iv = $InitializationVector

      $valueBytes = [Convert]::FromBase64String($String)
      $keyBytes = NewPasswordKey -Password $Password -Salt $Salt

      $cipher = [Security.Cryptography.SymmetricAlgorithm]::Create('AesManaged')
      $cipher.Mode = [Security.Cryptography.CipherMode]::CBC
      $cipher.Padding = $Padding

      $decryptor = $cipher.CreateDecryptor($keyBytes, $iv)
      $stream = New-Object -TypeName IO.MemoryStream `
        -ArgumentList @(, $valueBytes)
      $reader = New-Object -TypeName Security.Cryptography.CryptoStream `
        -ArgumentList @(
          $stream,
          $decryptor,
          [Security.Cryptography.CryptoStreamMode]::Read
        )

      $decrypted = New-Object -TypeName Byte[] -ArgumentList $valueBytes.Length
      $decryptedByteCount = $reader.Read($decrypted, 0, $decrypted.Length)
      $decryptedValue = [Text.Encoding]::UTF8.GetString(
        $decrypted,
        0,
        $decryptedByteCount
      )
      $cipher.Clear()
      $stream.SetLength(0)
      $stream.Close()
      $reader.Clear()
      $reader.Close()
      return $decryptedValue
    }
    Catch
    {
      Write-Error $_
    }
  }
}
```

As promised, some more examples.

#### Example 1:
----

```powershell
$pass = Read-Host -AsSecureString
$info = Protect-AesString -String "Keep this super safe!!" -Password $pass

# Decrypt
Unprotect-AesString -CipherInfo $info -Password $pass
```

This example encrypts a string using the default salt, and the pasword specified.
It decrypts the encrypted string using the default salt and the same password.

#### Example 2:
----

```powershell
$password = Read-Host -AsSecureString
$secret = 'Super secret info...'
$cipherInfo = Protect-AesString -String $secret -Password $password -Salt 'MoreSalt'
$cipherInfo | ConvertTo-Json -Compress > ./nothingtosee.json

$info = Get-Content ./nothingtosee.json | ConvertFrom-Json
Unprotect-AesString -String $info.CipherText -Salt $info.Salt -InitializationVector $info.IV -Password $password
```

This example uses a custom salt for encryption. 
It stores the output in a json file for safe keeping and re-use later.
It then decrypts the stored string using the stored info and the password used to encrypt it.
