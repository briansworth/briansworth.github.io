---
layout: post
title: PowerShell - Encrypting Passwords...Again
---

I have covered working with passwords in PowerShell in a previous 
[post](http://codeandkeep.com/Powershell-Read-Password/), 
but wanted to go over some more advanced options.

The 2 methods I will cover are:
  - Secure String Key Encryption
  - Certificate Encryption

### Secure String Key Encryption
---

<p>
  As you probably know, there is a -Key parameter in both the 
  ConvertTo-SecureString and ConvertFrom-SecureString cmdlets. 
  If you have played around with this you have likely figured out how 
  to use this parameter.  
  There is however a more secure way of generating a random byte array 
  for use as the key in these situations.
</p>

<p>
  Here is an example:
</p>

```powershell
$key=New-Object -TypeName Byte[] -ArgumentList 32
$rng=[Security.Cryptography.RNGCryptoServiceProvider]::Create()

$rng.GetBytes($key)
$secureString=Read-Host -AsSecureString -Prompt String

$encrypted=ConvertFrom-SecureString -Key $key -SecureString $secureString

Write-Host "Encrypted String:"
Write-Output $encrypted
```

<p>
  You could use the Get-Random cmdlet to generate the key, 
  however it is more secure to use the RNGCryptoServiceProvider object 
  as it will be more random in its number generation. 
  Of course to be truly random, 
  you would want to be using something less controlled than the CPU's clock, 
  but that would potentially require special hardware and software.
</p>

<p>
  Now to decrypt your string:
</p>

```powershell
$secure=ConvertTo-SecureString -String $encrypted -Key $key
$tempCred=New-Object -TypeName PSCredential -ArgumentList 'temp',$secure

Write-Host "Decrypted String:"
$tempCred.GetNetworkCredential().Password
Remove-Variable tempCred
```

<p>
  I should point out that the -Key parameter will only accept a ByteArray with 
  a bit length of either 128, 192, or 256.
</p>

<p>
  Armed with that simple example; we can make a general function for this:
</p>

```powershell
Function New-RandomByteArray {
  [CmdletBinding()]
  Param(
    [ValidateRange(8,[int]::MaxValue)]
    [Parameter(Position=0)]
    [int]$BitLength=256
  )
  Try{
    if($BitLength%8 -ne 0){
      Write-Error -Message "BitLength must be divisible by 8" `
        -Category InvalidArgument `
        -ErrorAction Stop
    }
    $key=New-Object -TypeName Byte[] -ArgumentList ($BitLength/8)
    $rng=[Security.Cryptography.RNGCryptoServiceProvider]::Create()

    $rng.GetBytes($key)
    Write-Output $key
  }Catch{
    Write-Error -Category $_.CategoryInfo.Category `
      -Exception $_.Exception `
      -Message $_.Exception.Message
  }
}

Function ConvertTo-EncryptedString {
  [CmdletBinding(DefaultParameterSetName='String')]
  Param(
    [Parameter(
      Position=0,
      Mandatory=$true,
      ValueFromPipeline=$true,
      ParameterSetName='String'
    )]
    [String]$String,

    [Parameter(
      Position=0,
      Mandatory=$true,
      ValueFromPipeline=$true,
      ParameterSetName='SecureString'
    )]
    [Security.SecureString]$SecureString,

    [ValidateCount(16,32)]
    [Parameter(Position=1,Mandatory=$true)]
    [Byte[]]$Key
  )
  Begin{}
  Process{
    Try{
      if($PSCmdlet.ParameterSetName -eq 'String'){
        $SecureString=ConvertTo-SecureString -String $String `
          -AsPlainText `
          -Force `
          -ErrorAction Stop
      }
      ConvertFrom-SecureString -SecureString $SecureString `
        -Key $Key
    }Catch{
      Write-Error -Category $_.CategoryInfo.Category `
        -Exception $_.Exception `
        -Message $_.Exception.Message
    }
  }
  End{}
}

Function ConvertFrom-EncryptedString {
  [CmdletBinding()]
  Param(
    [Parameter(Position=0,Mandatory=$true,ValueFromPipeline=$true)]
    [String]$EncryptedString,

    [ValidateCount(16,32)]
    [Parameter(Position=1)]
    [Byte[]]$Key,

    [Parameter(Position=2)]
    [Switch]$Secure
  )
  Begin{
    if(!($PSBoundParameters.ContainsKey('Key'))){
      Write-Error "No key specified. Unable to decrypt string" `
        -Category SecurityError
      break
    }
  }
  Process{
    Try{
      $secureStr=ConvertTo-SecureString -String $EncryptedString `
        -Key $Key `
        -ErrorAction Stop

      if($Secure){
        return $secureStr
      }

      $bstr=[Runtime.InteropServices.Marshal]::SecureStringToBSTR($secureStr)
      [Runtime.InteropServices.Marshal]::PtrToStringAuto($bstr)

    }Catch{
      Write-Error -Category $_.CategoryInfo.Category `
        -Exception $_.Exception `
        -Message $_.Exception.Message
    }
  }
  End{}
}
```

#### Example 1
----

<p>
  Here is a basic example of how to encrypt/decrypt a string with this method:
</p>

```powershell
$key=New-RandomByteArray -BitLength 256
$encrypt=ConvertTo-EncryptedString -String 'SecureMePlz!' -Key $key
Write-Output $encrypt

# Decrypt
ConvertFrom-EncryptedString -EncryptedString $encrypt -Key $key
```

<p>
  You will only be able to decrypt this string if you have the key that 
  was used to encrypt it.  
  You can try to generate a new random key and test the results.
</p>

#### Example 2
----

<p>
  For additional security, you can opt to encrypt/decrypt the string and 
  using SecureString objects.
</p>

```powershell
$key=New-RandomByteArray
$secure=Read-Host -AsSecureString -Prompt Password

$encryptedSecure=ConvertTo-EncryptedString -SecureString $secure -Key $key
Write-Output $encryptedSecure

# Decrypt to securestring using -Secure
$decrypted=ConvertFrom-EncryptedString -EncryptedString $encryptedSecure `
  -Secure `
  -Key $key

Write-Output $decrypted

# Now to prove it is decrypted correctly
$tempCred=New-Object -TypeName PSCredential `
  -ArgumentList 'temp',$decrypted
$tempCred.GetNetworkCredential().Password
Remove-Variable tempCred
```


### Encrypt Passwords with Certificates
----

I made a [post recently](http://codeandkeep.com/Dsc-Encrypting-Credentials/) 
that discussed how to encrypt credentials with Desired State Configuration 
(DSC).  
It just so happens that if you create a certificate using either 
of the methods used in that post, 
you can encrypt your own passwords using that certificate. 

<p>
  To complete the following example, 
  you will need to be running PowerShell as an Administrator. 
  Here's an example using the self-signed certificate example from that post:
</p>

#### Create the certificate

```powershell
# Create and import certificate to the LocalComputer\My certificate store
$certFolder='C:\cert'
$certStore='Cert:\LocalMachine\My'
$pubCertPath=Join-Path -Path $certFolder -ChildPath SelfSigned.cer
$expiryDate=(Get-Date).AddYears(2)

# You may want to delete this file after completing
$privateKeyPath=Join-Path -Path $ENV:TEMP -ChildPath SelfPrivKey.pfx

$privateKeyPass=Read-Host -AsSecureString -Prompt "Private Key Password"

if(!(Test-Path -Path $certFolder)){
    New-Item -Path $certFolder -Type Directory | Out-Null
}

$cert=New-SelfSignedCertificate -Type DocumentEncryptionCertLegacyCsp `
  -DnsName 'TestSelfSigned' `
  -HashAlgorithm SHA512 `
  -NotAfter $expiryDate `
  -KeyLength 4096 `
  -CertStoreLocation $certStore

$cert | Export-PfxCertificate -FilePath $privateKeyPath `
  -Password $privateKeyPass `
  -Force

$cert | Export-Certificate -FilePath $pubCertPath 

Import-Certificate -FilePath $pubCertPath `
  -CertStoreLocation $certStore

Import-PfxCertificate -FilePath $privateKeyPath `
  -CertStoreLocation $certStore `
  -Password $privateKeyPass | Out-Null
```

#### Encrypt using the certificate public key

```powershell
$certificate=Get-Item -Path "Cert:\LocalMachine\My\$($cert.Thumbprint)"
$encryptMe='Encrypt all the things!'
$encodeBytes=[Text.Encoding]::UTF8.GetBytes($encryptMe)

# Encrypt
[byte[]]$encryptBytes=$certificate.PublicKey.Key.Encrypt($encodeBytes, $true)
$encrypted=[Convert]::ToBase64String($encryptBytes)
Write-Output $encrypted
```

#### Decrypt using certificate private key

```powershell
$encryptedBytes=[Convert]::FromBase64String($encrypted)
if($certificate.PrivateKey){
  $bytes=$certificate.PrivateKey.Decrypt($encryptedBytes, $true)
  $decrypted=[Text.Encoding]::UTF8.GetString($bytes)
  Write-Output $decrypted
}else{
  Write-Error "The Certificate does not have a private key accessible!" `
    -ErrorAction Stop
}
```

### Functions for Certificate Encryption
----

```powershell
Function CreateX509CertFromFile {
  [CmdletBinding()]
  Param(
    [ValidateScript({Test-Path -Path $_ -PathType Leaf})]
    [String]$filePath
  )
  Try{
    $pathResolve=Resolve-Path -Path $filePath -ErrorAction Stop
    if($pathResolve.Provider.Name -eq 'Certificate'){
      Get-Item -Path $filePath
    }else{
      $cert=[Security.Cryptography.X509Certificates.X509Certificate2]::CreateFromCertFile(
        $filePath
      )
      [Security.Cryptography.X509Certificates.X509Certificate2]$cert
    }
  }Catch{
    Write-Error -Category $_.CategoryInfo.Category `
      -Exception $_.Exception `
      -Message $_.Exception.Message
  }
}

Function ConvertTo-CertificateEncryptedString {
  [CmdletBinding(DefaultParameterSetName='CertFile')]
  Param(
    [Parameter(Position=0,Mandatory=$true,ValueFromPipeline=$true)]
    [String]$String,

    [Parameter(
      Position=1,
      Mandatory=$true,
      ValueFromPipeline=$true,
      ParameterSetName='Cert'
    )]
    [Security.Cryptography.X509Certificates.X509Certificate2]$Certificate,

    [Parameter(
      Position=1,
      Mandatory=$true,
      ParameterSetName='CertFile'
    )]
    [ValidateScript({Test-Path -Path $_ -PathType Leaf})]
    [String]$CertFilePath
  )
  Begin{}
  Process{
    Try{
      if($PSCmdlet.ParameterSetName -eq 'CertFile'){
        $Certificate=CreateX509CertFromFile -filePath $CertFilePath `
          -ErrorAction Stop
      }
      $encodeBytes=[Text.Encoding]::UTF8.GetBytes($String)
      
      [byte[]]$encryptBytes=$Certificate.PublicKey.Key.Encrypt($encodeBytes, $true)
      [Convert]::ToBase64String($encryptBytes)
    }Catch {
      Write-Error -Message $_.Exception.Message `
        -Exception $_.Exception `
        -Category $_.CategoryInfo.Category
    }
  }
  End{}
}

Function ConvertFrom-CertificateEncryptedString {
  [CmdletBinding(DefaultParameterSetName='CertFile')]
  Param(
    [Parameter(Position=0,Mandatory=$true,ValueFromPipeline=$true)]
    [String]$EncryptedString,

    [Parameter(
      Position=1,
      Mandatory=$true,
      ValueFromPipeline=$true,
      ParameterSetName='Cert'
    )]
    [Security.Cryptography.X509Certificates.X509Certificate2]$Certificate,

    [Parameter(
      Position=1,
      Mandatory=$true,
      ParameterSetName='CertFile'
    )]
    [ValidateScript({Test-Path -Path $_})]
    [String]$CertFilePath
  )
  Begin{}
  Process{
    Try{
      if($PSCmdlet.ParameterSetName -eq 'CertFile'){
        $Certificate=CreateX509CertFromFile -filePath $CertFilePath `
          -ErrorAction Stop
      }
      if(!$Certificate.PrivateKey){
        $privKeyException=[Management.Automation.PropertyNotFoundException](
          "Certificate [$($Certificate.Thumbprint)] Private Key not found."
        )
        Write-Error -Category ResourceUnavailable `
          -Exception $privKeyException `
          -Message $privKeyException.Message `
          -ErrorAction Stop
      }
      $encryptBytes=[Convert]::FromBase64String($EncryptedString)
      $bytes=$Certificate.PrivateKey.Decrypt($encryptBytes, $true)
      [Text.Encoding]::UTF8.GetString($bytes)
    }Catch{
      Write-Error -Category $_.CategoryInfo.Category `
        -Exception $_.Exception `
        -Message $_.Exception.Message
    }
  }
  End{}
}
```

#### Example 1
----

<p>
  Use a certificate from the certificate store
</p>

```powershell
# Encrypt
$certPath='Cert:\LocalMachine\My\3B99C626C216A9626193D85C61587132C42FBCC2'
$secret=ConvertTo-CertificateEncryptedString -String 'Testing Encryption' `
  -CertFilePath $certPath
Write-Output $secret

# Decrypt
$secret | ConvertFrom-CertificateEncryptedString -CertFilePath $certPath
```

#### Example 2
----

<p>
  Use a certificate object
</p>

```powershell
# Encrypt 
$cert=ls Cert:\LocalMachine\My\ | Where {$_.Subject -eq 'CN=TestSelfSigned'}
$encrypted=$cert | ConvertTo-CertificateEncryptedString -String 'Hello World!'
Write-Output $encrypted

# Decrypt
$cert | ConvertFrom-CertificateEncryptedString -EncryptedString $encrypted
```
