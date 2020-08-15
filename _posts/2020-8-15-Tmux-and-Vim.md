---
layout: post
title: Tmux and Vim
description: Configure Tmux with VIM keys. Works on Windows and Linux
---

Tmux is one of the most useful applications to use on Linux, 
and if you read my [last post]({{ site.baseurl }}/Tmux-on-Windows/), 
you know how to get Tmux up and running on Windows too.
As a VIM user (addict), I need to use the VIM keys everywhere possible, 
and in this post, we will configure Tmux to do exactly that.

## Overview

----

If you are already experienced with Tmux, 
you can grab my config files here (otherwise keep reading):
- Linux: [https://gist.github.com/briansworth/9da664f15e51ca48ab5d7a0ac4a73cb2](https://gist.github.com/briansworth/9da664f15e51ca48ab5d7a0ac4a73cb2)
- Windows: [https://gist.github.com/briansworth/b12f28f9a9e7bd42d9d7b67160079188](https://gist.github.com/briansworth/b12f28f9a9e7bd42d9d7b67160079188)


The final product:

![_config.yml]({{ site.baseurl }}/images/tmux-vim.png)


## Prerequisites

----

You will need Tmux installed and working.

**Linux**: Install as you would any other package

```bash
# Example for Debian based distros
sudo apt install tmux
```

**Windows**: Follow the guide in my 
[previous post]({{ site.baseurl }}/Tmux-on-Windows/) and then continue from here.


#### Disclaimer

----

<p>
  This guide will do more than just add VIM keys to Tmux;
  it will change the default Tmux prefix <code>Ctrl + b</code> to the backtick <code>`</code>. 
  Additionally, the new configuration references some useful Tmux plugins 
  (we will install these in this guide).
</p>
  I would recommend backing up your existing Tmux.conf file before continuing:

```bash
# Backup your existing config if necessary
cp ~/.tmux.conf ~/.tmux.conf-backup
```

## Set the tmux.conf file

----

I have a pre-set Tmux.conf file stored as a Gist in GitHub for Linux and Windows.

#### Linux

Here is the link for the [Linux Tmux config](https://gist.github.com/briansworth/9da664f15e51ca48ab5d7a0ac4a73cb2).

Run the following to quickly install it on Linux (or copy it manually).

```bash
TMUX_CONF=~/.tmux.conf
GIST_URL=https://gist.githubusercontent.com/briansworth/9da664f15e51ca48ab5d7a0ac4a73cb2/raw/ea2f7da743887e345dbddeccd7eedb1fd2271ba6/.tmux.conf

# Backup the existing tmux.conf if it exists before overwriting
if [ -f $TMUX_CONF ]; then
  cp $TMUX_CONF ~/.tmux.conf.bak_CodeAndKeep
fi
curl $GIST_URL -o $TMUX_CONF
```

#### Windows

Here is the link for the [Windows Tmux config](https://gist.github.com/briansworth/b12f28f9a9e7bd42d9d7b67160079188).

Run the following to quickly install it on Windows (or copy it manually).

*NOTE: You will need to be in a bash shell on your WSL Distro*

```bash
TMUX_CONF=~/.tmux.conf
GIST_URL=https://gist.githubusercontent.com/briansworth/b12f28f9a9e7bd42d9d7b67160079188/raw/61bcca39aed0b68fb2891c26ac4bf92c20733bcd/windows.tmux.conf

# Backup the existing tmux.conf if it exists before overwriting
if [ -f $TMUX_CONF ]; then
  cp $TMUX_CONF ~/.tmux.conf.bak_CodeAndKeep
fi
curl $GIST_URL -o $TMUX_CONF
```

### Install Tmux plugins

----

The Tmux configuration file we just installed references some plugins.
The configuration will still work without installing them, 
but I would recommend installing them. 
More details to follow about how to use them.

```bash
# Download TPM (Tmux plugin manager) from github
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
```

You now have the Tmux plugin manager that allows for easy plugin installation.
Next step is to launch Tmux, and install the plugins referenced in the config file.

```bash
tmux
```

<p>
Press <code>` + I</code> to install the plugins. 
It will indicate when the install is completed.
</p>

If you see an error code 127, 
ensure you cloned the tmux-plugins/tpm repo to the correct directory (steps above).

At this point, the configuration is completed.
The rest of the guide will explain the changes.


## What's New

----

**The Prefix:**
<p>
  Most importantly, the prefix is now the backtick <code>`</code> 
  (instead of the default <code>Ctrl + b</code>).
</p>

**Panes:**

Split panes vertically: <code>` + |</code> (backtick + pipe)

Split panes horizontally: <code>` + -</code> (backtick + minus)

Move to pane:
- Up:    <code>` + k</code>
- Down:  <code>` + j</code>
- Left:  <code>` + h</code>
- Right: <code>` + l</code>

Resize pane:
- Up:    <code>` + K</code>
- Down:  <code>` + J</code>
- Left:  <code>` + H</code>
- Right: <code>` + L</code>


**Windows:**

Create a new window the same way: <code>` + c</code>

Cycle through windows:
- Left:  `Ctrl + h`
- Right: `Ctrl + l`

**Copy Mode:**

Get into copy mode as usual: <code>` + [</code>

Use the VIM navigation keys while in the buffer.

Begin text selection: `v`

Yank text selection: `y`

(with the Tmux-yank plugin,
this will also copy the selection to the system clipboard)

**Plugins:**

***Tmux Yank:***
As indicated above, when copying text from the copy-mode buffer,
the text will automatically be copied to the system clipboard. 

Works on Linux and Windows (WSL); very useful.


***Tmux Resurrect:***
Provides the ability to save and restore tmux sessions. 
Even after a reboot or crash.

It will save all panes and windows (in order) 
and even certain programs that are running in said panes.

Save session: <code>` Ctrl + s</code>

*Press the prefix first, then Ctrl and s keys together.*

Restore session: <code>` Ctrl + r</code>

*Press the prefix first, then Ctrl and r keys together.*


**Appearance:**

There are several changes to the default appearance of Tmux using this config.

The most significant is the status bar being on top of the window.
As for the other changes, you can see them. 

You can always read through the tmux.conf file for comments on all changes,
and tweak as needed.

