---
layout: post
title: Using Passwords in PowerShell
---

Working with passwords and credentials does not have to be scary.

I will discuss some useful tricks when working with passwords and credentials.
### PSCredential tricks
----
<br>
PowerShell has made it incredibly easy and secure to specify and use credentials with the Get-Credential command.
```powershell
$credential=Get-Credential
```
![_config.yml]({{ site.baseurl }}/images/getCredential.PNG)

<br>
The `Get-Credential` cmdlet returns a PSCredential object.

You may not know this but there is a method for this object type that will allow you to the stored password as plain text.

```powershell
$credential=Get-Credential
$credential.GetNetworkCredential()

$credential.GetNetworkCredential() | FL
```

![_config.yml]({{ site.baseurl }}/images/getNetworkCredentials.PNG)

If you need to quickly validate your credentials, this would be quite useful.
<br>

You can build a PSCredential object without using the `Get-Credential` cmdlet like so:
```powershell
$pass=ConvertTo-SecureString 'P@ssw0rd1' -AsPlainText -Force
$cred=New-Object -TypeName PSCredential -ArgumentList @('codeAndKeep\Administrator',$pass)
```
Not a great way to do it in terms of security, but in a dev environment this can be handy when automating tasks.

<br>
### Working with Passwords
----
Sometimes, there is no need to use a full PSCredential object, and all you need is a password.

Generally, when specifying a password in PowerShell, the parameters only accept secure strings (this is a good thing).
This can however get to be tedious always having to convert your password into that data type.
Here are the most common ways of making secure strings in PowerShell.
```powershell
$secure=Read-Host -AsSecureString -Prompt "Password"
# or
$secure=ConvertTo-SecureString -String "P@ssw0rd" -AsPlainText -Force
```
There are pros and cons to both methods.
<br>
**Read-Host approach**

Pros:
1. Easy to remember
2. Password is never in command history as clear text

Cons:
1. Requires user input when run
2. Cannot validate if there was a typo
<br>

**ConvertTo-SecureString approach**

Pros:
1. Can be used a script without user input
2. Easily validate if typed correctly
Cons:
1. Password is in clear text in command history

<br>
In the ActiveDirectory world, you can reset a user's password in a one-liner (as long use you have the AD cmdlets available).
```powershell
Set-ADAccountPassword -Identity user -Reset -NewPassword (Read-Host -AsSecureString -Prompt "Password")
```
<br>

### Function time!
----

I am a fan of the Read-Host method of capturing passwords because it is more secure than converting from plain text.
I'm also a fan of not mistyping passwords and accidentally locking myself out of accounts.

So here's a nice way of getting out clear text from a secure string.
```powershell
$secure=Read-Host -AsSecureString -Prompt Password
$bstr=[Runtime.InteropServices.Marshal]::SecureStringToBSTR($secure)
$plain=[Runtime.InteropServices.Marshal]::PtrToStringAuto($bstr)
Write-Output $plain
```
Now we can use this to prompt for a password twice, and validate that they match exactly.

```powershell
$pass=Read-Host -Prompt 'Enter a Password' `
  -AsSecureString 
$pass2=Read-Host -Prompt 'Re-type Password' `
  -AsSecureString
$bstr=[Runtime.InteropServices.Marshal]::SecureStringToBSTR($pass)
$plain=[Runtime.InteropServices.Marshal]::PtrToStringAuto($bstr)
[Runtime.InteropServices.Marshal]::ZeroFreeBSTR($bstr)
$bstr2=[Runtime.InteropServices.Marshal]::SecureStringToBSTR($pass2)
$plain2=[Runtime.InteropServices.Marshal]::PtrToStringAuto($bstr2)
[Runtime.InteropServices.Marshal]::ZeroFreeBSTR($bstr2)
if($plain -cne $plain2){
  Write-Error -Message "Passwords don't match...Try again."
}else{
  Write-Output $pass
}
```

Now to make this into a function.

```powershell
function Read-Password {
  $pass=Read-Host -Prompt 'Enter a Password' `
    -AsSecureString 
  $pass2=Read-Host -Prompt 'Re-type Password' `
    -AsSecureString 
  $bstr=[Runtime.InteropServices.Marshal]::SecureStringToBSTR($pass)
  $plain=[Runtime.InteropServices.Marshal]::PtrToStringAuto($bstr)
  [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($bstr)
  $bstr2=[Runtime.InteropServices.Marshal]::SecureStringToBSTR($pass2)
  $plain2=[Runtime.InteropServices.Marshal]::PtrToStringAuto($bstr2)
  [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($bstr2)
  [bool]$pValid=$true
  $builder=New-Object -TypeName Text.StringBuilder
  if ($plain -cne $plain2){
    $pValid=$false
    [void]$builder.Append('Passwords do not match. ')
  }
  if ($plain.Length -lt 8){
    $pValid=$false
    [void]$builder.Append('Password too short. ')
  }
  if($plain.Length -gt 127){
    $pValid=$false
    [void]$builder.Append('Password too long. ')
  }
  if($plain.ToLower() -ceq  $plain){
    $pValid=$false
    [void]$builder.Append('Must have an upper case letter. ')
  }
  if($plain.ToUpper() -ceq $plain){
    $pValid=$false
    [void]$builder.Append('Must have a lower case letter. ')
  }
  if($plain -notmatch '[\W\d]'){
    $pValid=$false
    [void]$builder.Append('Must contain a special or numeric character. ')
  }
  if($pValid -eq $false){
    Write-Error -Message $builder.ToString()
  }else{
    Write-Output $pass
  }
}
```
<br>
This is pretty rudimentary.  It has some basic password requirements hard coded into it (it would be nice to make them parameters in the future).

<br>
Hopefully you find this to be a useful tool to keep in your inventory.

*Thanks for reading,*

PS> exit
