---
layout: post
title: "How To Configure ComEmu Task For GitHub For Windows Portable Git"
date: 2014-07-20 11:58
comments: true
categories:
- Development
tags:
- ConEmu
- Git
---

Maybe a year or something ago I switched from [Console2](http://sourceforge.net/projects/console/files/) to [ConEmu](http://code.google.com/p/conemu-maximus5/). One of the reasons behind this switch was a [Task](http://code.google.com/p/conemu-maximus5/wiki/SettingsTasks) concept that ConEmu offered. 

I had only one problem with my tasks setup - I wanted to use Portable Git which is a part of GitHub for Windows installation. 

As you may know Portable Git binaries are located in  `%LOCALAPPDATA%\GitHub\PortableGit_054f2e797ebafd44a30203088cd3d58663c627ef\`. The last part of directory name is a version string so it could change in future.

If you will try to launch the `git-cmd.bat` from ConEmu the interception will not work and you will see new window. The problem lies in the last line of the `git-cmd.bat` file:

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

Last line `@start %COMSPEC%` should be replaced with `@call %COMSPEC%` so ConEmu will be able to intercept the launch of the new shell.

The difference between `start` and `call` commands is that `call` runs the batch script inside the same shell instance (allowing to change environment variables using `SET`) while `start` creates a new instance. Here is a little fragment from `start` and `call` help:

{% codeblock %}
C:\>call /?
Calls one batch program from another.

C:\>start /?
Starts a separate window to run a specified program or command.
{% endcodeblock %}

That's it! Now you are free to create and use a new task for ConEmu:

`*cmd /k Title Git & "%LOCALAPPDATA%\GitHub\PortableGit_054f2e797ebafd44a30203088cd3d58663c627ef\git-cmd.bat"`