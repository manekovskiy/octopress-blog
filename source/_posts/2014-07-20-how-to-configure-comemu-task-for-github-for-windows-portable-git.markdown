---
layout: post
title: "How to Configure ComEmu Task for GitHub for Windows Portable Git"
date: 2014-07-20 11:58
comments: true
categories:
- Development
tags:
- ConEmu
- Git
---

> **2/19/2015 Update:** I've decided that it would be good to propose the change described in this post to the [msysgit](https://github.com/msysgit/msysgit) project. And today it was accepted and merged. 
> It took me only 7 months to come up with idea that the change described below could be included into the official release of the software that I'm using on a daily basis :)

Maybe a year or something ago I switched from [Console2](http://sourceforge.net/projects/console/files/) to [ConEmu](http://code.google.com/p/conemu-maximus5/). One of the reasons behind this switch was a [Task](http://code.google.com/p/conemu-maximus5/wiki/SettingsTasks) concept that ConEmu offered. 

There was only one problem with my tasks setup - I wanted to launch Portable Git which is a part of GitHub for Windows installation inside ConEmu. But launching the `git-cmd.bat` from ConEmu will create a new  window.

> As you may know Portable Git binaries are located in  `%LOCALAPPDATA%\GitHub\PortableGit_054f2e797ebafd44a30203088cd3d58663c627ef\`
> Note that the last part of the directory name is a version string so it could change in future.

The problem lies in the last line of the `git-cmd.bat` file:

{% codeblock git-cmd.bat lang:bat %}
@rem Do not use "echo off" to not affect any child calls.
@setlocal

@rem Get the abolute path to the current directory, which is assumed to be the
@rem Git installation root.
@for /F "delims=" %%I in ("%~dp0") do @set git_install_root=%%~fI
@set PATH=%git_install_root%\bin;%git_install_root%\mingw\bin;%git_install_root%\cmd;%PATH%

@if not exist "%HOME%" @set HOME=%HOMEDRIVE%%HOMEPATH%
@if not exist "%HOME%" @set HOME=%USERPROFILE%

@set PLINK_PROTOCOL=ssh
@if not defined TERM set TERM=msys

@cd %HOME%
@start %COMSPEC%
{% endcodeblock %}

To fix the issue replace last line `@start %COMSPEC%` with `@call %COMSPEC%`.

> This change will **not** break the existing "Open in Git Shell" context action in GitHub application GUI.

The difference between `start` and `call` commands is that `call` runs the batch script inside the same shell instance while `start` creates a new instance. Here is a little fragment from `start` and `call` help:

{% codeblock %}
C:\>call /?
Calls one batch program from another.

C:\>start /?
Starts a separate window to run a specified program or command.
{% endcodeblock %}

That's it! Now following task for ConEmu will work as expected:

`*cmd /k Title Git & "%LOCALAPPDATA%\GitHub\PortableGit_054f2e797ebafd44a30203088cd3d58663c627ef\git-cmd.bat"`