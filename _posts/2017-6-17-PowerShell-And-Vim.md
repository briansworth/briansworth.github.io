---
layout: post
title: PowerShell and Vim
---
<p>
  If you are familiar with Linux or come from a Unix background,
  you probably know about Vim.
  For those of us that started and stay mostly in the realm of Windows however;
  Vim may be a foreign thing.
  I was fortunte enough to be exposed to vim, and see what it can do;
  now there's no turning back.
</p> 

### Vi on the Command Line
----

<p>
  There has been a module for PowerShell for a while now that allows you to use
  a vi editor on the command line.  Previously you would have had to
  copy the code from GitHub, but now - thanks to the PowerShell NuGet package manager -
  you can install it easily straight from PowerShell.
  Better yet, if you are on Windows 10, or server2016, it comes pre-installed.
</p>
<p>
  You can check if you have it with one command.
</p>

```powershell
Get-Module -Name PSReadline -ListAvailable
```
<p>
  If you don't get any results then it's not installed.
</p>


If you have a version number 1.1 or prior, 
then you will need to upgrade the version to 1.2 or higher:
<br>
![_config.xml]({{ site.baseurl }}/images/psReadlineVersion.png)
<br>
<p>
  You can install it like so:
</p>

```powershell
# Search for it
Find-Package -Name PSReadline

# install it with -Force to overwrite the older version if you have it
Install-Package -Name PSReadline -Force
```
<p>
  If you haven't setup NuGet previously, you can follow the onscreen instructions
  to get it setup and configured.
</p>

<p>
  Now that you have it, you can setup your command line editor in vi mode:
</p>

```powershell
Set-PSReadlineOption -EditMode vi -BellStyle None
# I like to also add -BellStyle 'None' because I hate the bell sound
```

<p>
  If you want this set by default you can add this to your profile 
  so it loads automatically.
</p>

#### If you don't have it
----

If you are on anything before win10 or server 2016, 
you will need to some prep work:

* You need to have .Net 4.5 installed.
  * Download [here](https://www.microsoft.com/en-ca/download/details.aspx?id=30653)
* Minimum Windows Management Framework version 3 (WMF 3), but I recommend WMF 5.1.
  * [WMF3](https://www.microsoft.com/en-ca/download/details.aspx?id=34595),  [WMF5.1](https://www.microsoft.com/en-us/download/details.aspx?id=54616)
* You need the PackageManagement PowerShell module
  * Download [here](https://www.microsoft.com/en-us/download/details.aspx?id=51451)

With these prerequisites installed, 
you can follow the steps above to install PSReadline.

### Vim on Windows
----

Now that you have your command line setup with vi, now you can setup the real thing.
You can download Vim for Windows [here](www.vim.org/download.php), or
with code (working at the time of writing this post):

```powershell
# This should download a file (gvim80-586.exe) to your Downloads folder
Invoke-WebRequest -Uri https://ftp.nluug.nl/pub/vim/pc/gvim80-586.exe `
  -Outfile ~\Downloads\gvim80-586.exe
```

<p>
  This is a self installing exe, so all you need to do is run it and customize
  it as you will.
  I typically just use the default settings (next, next, finish).
</p>

#### Configure Vim for PowerShell
----

<p>
  By default, vim is installed here 'C:\Program Files (x86)\Vim'.
  The executable that you will want to run will be here: 
  'C:\Program Files (x86)\Vim\vim80\vim.exe'.
  You can add the base path of the above to your PATH, but since I will only be using 
  the one executable here (vim.exe), and I will only be using it via PowerShell,
  I simply create an alias in my PowerShell profile.
</p>

```powershell
# Add these lines to your $PROFILE
New-Alias -Name vi -Value 'C:\Program Files (x86)\vim\vim80\vim.exe'
New-Alias -Name vim -Value 'C:\Program Files (x86)\vim\vim80\vim.exe'

# Include this if you like a vim command line experience
Set-PSReadlineOption -EditMode vi -BellStyle None
```

Now next time you want to edit a file, instead of `notepad '.\test.txt'`, 
do `vi test.txt`.

### Customize your vimrc file
-----
<p>
  There should be a a file named "_vimrc" in your user directory.  
  If there isn't you can go ahead and create it.
</p>

```powershell
$vimrc="~\_vimrc"
# Make it if it's not already there
if(!(Test-Path -Path $vimrc)){
  New-Item -Path $vimrc -ItemType File
} 
# Open it up in vi
vi $vimrc
```

<p>
  Now I'm still relatively new with using Vim, but these are the settings
  that I have come accross that work well for me with PowerShell.
  The biggest problem for me was finding a color scheme that worked with 
  PowerShell's color scheme.
</p>


```vim
" Enable syntax highlighting
syntax on

" Change tabs to 2 spaces
set expandtab 
set tabstop=2
set shiftwidth=2

" Automatically indent when starting new lines in code blocks
set autoindent

" Add line numbers
set number

" shows column, & line number in bottom right 
set ruler

" Color scheme I found that works best with PowerShell
colorscheme shine

" helpful if using 'set ruler' and 'colorscheme shine', makes lineNumbers grey
" Same example from http://vim.wikia.com/wiki/Display_line_numbers
highlight LineNr term=bold cterm=NONE ctermfg=DarkGrey ctermbg=NONE gui=NONE guifg=DarkGrey guibg=NONE

" Disable bell sounds 
set noerrorbells visualbell t_vb=
```

<p>
  This is a start; hopefully you can build on this and improve on it.
</p>

*SIDE NOTE:*
If you want to write PowerShell code using Vim, I would recommend using VisualStudio Code.
If has extensions for both PowerShell and Vim.
The PowerShell extension allows for 'Intellisense' and syntax highlighting for PowerShell;
and the Vim extension will allow you to use Vim shortcuts for editting your code.
