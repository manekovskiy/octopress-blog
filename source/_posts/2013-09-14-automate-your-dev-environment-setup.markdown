---
layout: post
title: "Automate your dev environment setup"
date: 2013-09-14 19:03
comments: true
categories:
- Development
tags:
- Chocolatey
- NuGet
- Scripting
- 
---

# Why? #

Because every time I need to install and configure developer environment on a fresh OS (either on real or virtual machine) I feel irritated by the fact that I need to spend almost all my day just clicking around various installation dialogs confirming destination folders, accepting user agreements (that I can bet no one even tried to read fully) and performing other repetitive and pointless things.

I'm developer, I'm creating things (or at least trying to), so why should I waste my time doing dull and pointless work?! Ah, and why should I keep in mind (or in notepad, "installs" folder, etc.) a list of my tools and installation packages?

But honestly, I just cannot resist or give a single reason why this shouldn't be automated. Said it - did it. And here are my adventures.

# Let's get chocolatey? #

Maybe you've heard about [Chocolatey](http://chocolatey.org/ "Chocolatey website"). In short this tool is like [apt-get](http://en.wikipedia.org/wiki/Advanced_Packaging_Tool "Wiki page for apt-get") but for Windows and it is built on top of NuGet. This means that you can "silently" install your favorite software using command line.

As of time of writing Chocolatey had 1,244 unique packages which is pretty cool - it is really hard to find package that does not exist there.

After a little search it appeared that I can even install Visual Studio with Chocolatey. Okay, cool, lets do this.

## No Battle Plan Survives Contact With the Enemy ##

I tried to install my first package on a fresh Windows 8 virtual machine and failed on the very first step. Jumping ahead of the story partially that was my fail but lets roll on.

I wanted no more no less - intall [Visual Studio 2013 Ultimate Preview](http://chocolatey.org/packages/VisualStudio2013Ultimate "VS 2013 chocolatey package") and see its new shining features for web devs. As described on site I installed Chocolatey and run `cinst VisualStudio2013Ultimate` command. Package downloaded, and .NET 4.5.1 installation started. Boom! I got my first error:

>  `[ERROR] Exception calling "Start" with "1" argument(s): "The operation was canceled by the user"`

{% img center /images/posts/win8-chocolatey-net451-error.png 'Chocolatey .NET 4.5.1 installation error' 'Chocolatey .NET 4.5.1 installation error' %}

After some research it appeared that by default Windows 8 processes are not launched with administrator priviledges (even if current user is member of Administrator group) and because of silent installation mode (read "non-UI mode") UAC prompt was not showed and attempt to elevate rights was cancelled by default. To fix this issue I had to disable UAC notifications. I have spent quite time searching the cause of my issue and decided to table VS 2013 for now and proceed with Visual Studio 2012 instead.

To install 90 day trial of Visual Studio 2012 Ultimate I run `cinst VisualStudio2012Ultimate` and after a little pause and some blinking of standard installation dialog another wild error appeared: 

> `blah-blah-blah. Exit code was '-2147185721'`

{% img center /images/posts/win8-chocolatey-vs2012-error-1.png 'Chocolatey VS 2012 installation error' 'Chocolatey VS 2012 installation error' %}

Thankfully, I have experience with silent installations of Visual Studio and I have a link to [Visual Studio Administrator Guide](http://msdn.microsoft.com/en-us/library/vstudio/ee225238.aspx "Visual Studio Administrator Guide on MSDN") in my bookmarks which contains a list of exit codes for installation package. `-2147185721` code is "*Incomplete - Reboot Required*". That sounded logically. `/NoRestart` switch in VS chocolatey install script setup was automatically cancelled and returned non-zero value which was treated as error. Okay, rebooted the machine.

But this was not my last error :). After reboot using `-force` parameter I resumed installation process of the Visual Studio and got my next error (extracted from installation log `vs.log` file):

{% blockquote %}
[0824:0820][2013-09-14T12:56:04]: Applied execute package: vcRuntimeDebug_x86, result: 0x0, restart: None
[082C:09C4][2013-09-14T12:56:04]: Registering dependency: {ae17ae9b-af38-40d2-a194-6102c56ed502} on package provider: Microsoft.VS.VC_RuntimeDebug_x86,v11, package: vcRuntimeDebug_x86
[082C:0850][2013-09-14T12:56:12]: Error 0x80070490: Failed to find expected public key in certificate chain.
{% endblockquote %}

The exit code from chocolatey was "Exit code was '1603'".

This time nothing came to my mind except trying to install updates on Windows first (words "certificate chain" lead me to this idea). As it turned out that was the case and my great mistake not to install updates first.

> Moral: never try to install something serious unless you have all updates for your OS installed.

After all these errors I decided to rollback my virtual machine back to the initial state and start from scratch. This time I installed all Windows updates and after I finished all Chocolatey packages were installed without any errors.

# Conslusions

While you can use Chocolatey to save your time with installations you cannot use it to configure the stuff you installed. But I don't think it should. There 

# Share all the scripts! #

After I finished with my journey I decided that it would be great to keep my scripts in one place and have a possibility to share them. I cannot find any better service for this but Github. Now I can share my scripts with folks, update them, have a history of changes, make tags and special branches for some specific setups. Isn't this what social coding means?

Go [fork it](https://github.com/manekovskiy/devenv-setup-scripts "devenv-setup-scripts project page") and make you life easier!