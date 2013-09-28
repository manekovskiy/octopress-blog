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
- Powershell
---

Every time I need to install and configure developer environment on a fresh OS (either on real or virtual machine) I feel irritated by the fact that I need to spend almost all my day just clicking around various installation dialogs confirming destination folders, accepting user agreements (that I can bet no one even tried to read fully) and performing other repetitive and almost pointless tasks.

I'm developer, I'm creating things (or at least trying to), so why would I waste my time doing dull and pointless work?! Ah, and why should I keep in mind (or in notepad, "installs" folder, etc.) a list of my tools and installation packages?

But honestly, I just cannot resist or give a single reason why this shouldn't be automated. Said it - did it. And here are my adventures.

## Let's get Chocolatey? ##

Maybe you've heard about [Chocolatey](http://chocolatey.org/ "Chocolatey website"). In short this tool is like [apt-get](http://en.wikipedia.org/wiki/Advanced_Packaging_Tool "Wiki page for apt-get") but for Windows and it is built on top of NuGet.

> For those who are not familiar with NuGet and all variety of tools around it take a look at [An Overview of the NuGet Ecosystem](http://www.codeproject.com/Reference/628210/An-Overview-of-the-NuGet-Ecosystem) article by Xavier Decoster.

> For a quick Chocolatey overview I can recommend Scott Hanselman's post [Is the Windows user ready for apt-get](http://www.hanselman.com/blog/IsTheWindowsUserReadyForAptget.aspx)?

As of time of writing Chocolatey had 1,244 unique packages which is pretty cool - it is really hard to find package that does not exist there.

After a little search it appeared that I can even install Visual Studio with Chocolatey. Okay, cool, let's do this.

### No Battle Plan Survives Contact With the Enemy ###

I tried to install my first package on a fresh Windows 8 virtual machine and failed on the very first step. Jumping ahead of the story partially that was my fail but let's roll on.

I wanted no more no less - install [Visual Studio 2013 Ultimate Preview](http://chocolatey.org/packages/VisualStudio2013Ultimate "VS 2013 chocolatey package") and see its new shining features for web devs. As described on site I installed Chocolatey and run `cinst VisualStudio2013Ultimate` command. Package downloaded, and .NET 4.5.1 installation started. Boom! I got my first error:

{% codeblock %}
[ERROR] Exception calling "Start" with "1" argument(s): "The operation was canceled by the user"
{% endcodeblock %}

{% img center /images/posts/win8-chocolatey-net451-error.png 'Chocolatey .NET 4.5.1 installation error' 'Chocolatey .NET 4.5.1 installation error' %}

After some research it appeared that by default Windows 8 processes are not launched with administrator privileges (even if current user is member of Administrator group) and because of silent installation mode (read "non-UI mode") UAC prompt was not showed and attempt to elevate rights was cancelled by default. To fix this issue I had to disable UAC notifications. I have spent quite time searching the cause of my issue and decided to table VS 2013 for now and proceed with installation of the Visual Studio 2012 instead.

To install 90 day trial of Visual Studio 2012 Ultimate I run `cinst VisualStudio2012Ultimate` command and after a little pause and some blinking of standard installation dialog another crazy error appeared: 

{% codeblock %}
blah-blah-blah. Exit code was '-2147185721'
{% endcodeblock %}

{% img center /images/posts/win8-chocolatey-vs2012-error-1.png 'Chocolatey VS 2012 installation error' 'Chocolatey VS 2012 installation error' %}

Thankfully, I have experience with silent installations of Visual Studio and I have a link to [Visual Studio Administrator Guide](http://msdn.microsoft.com/en-us/library/vstudio/ee225238.aspx "Visual Studio Administrator Guide on MSDN") in my bookmarks which contains a list of exit codes for installation package. `-2147185721` code is "*Incomplete - Reboot Required*". That sounded logically. `/NoRestart` switch in VS chocolatey install script setup was automatically cancelled and returned non-zero value which was treated as error. Okay, rebooted the machine.

But this was not my last error :). After reboot using `-force` parameter I resumed installation process of the Visual Studio and got my next error (extracted from installation log `vs.log` file):

{% codeblock %}
[0824:0820][2013-09-14T12:56:04]: Applied execute package: vcRuntimeDebug_x86, result: 0x0, restart: None
[082C:09C4][2013-09-14T12:56:04]: Registering dependency: {ae17ae9b-af38-40d2-a194-6102c56ed502} on package provider: Microsoft.VS.VC_RuntimeDebug_x86,v11, package: vcRuntimeDebug_x86
[082C:0850][2013-09-14T12:56:12]: Error 0x80070490: Failed to find expected public key in certificate chain.
{% endcodeblock %}

The last words from "chocolatey gods" were `Exit code was '1603'`.

This time nothing came to my mind except trying to install updates on Windows first (words "certificate chain" lead me to this idea). As it turned out that was the case and my great mistake not to install updates first.

> Moral: never try to install something serious unless you have all updates for your OS installed.

After all these errors I decided to rollback my virtual machine back to the initial state and start from scratch. This time I installed all Windows updates and after I finished all Chocolatey packages were installed without any errors.

## Share all the scripts! ##

{% img right /images/posts/share-all-the-scripts.jpg 'Share all the scripts!' 'Share all the scripts!' %}

After I finished with my journey I decided that it would be great to keep my scripts in one place and have a possibility to share them. I cannot find any better service for this but Github. Now I can share my scripts, update them, have a history of changes, make tags and special branches for some specific setups. Isn't this great and how it should be?

Go fork [my repository](https://github.com/manekovskiy/devenv-setup-scripts "devenv-setup-scripts project page") and start making your life easier!

## Conclusions ##

Here I did only first steps on the road to the bright future of the automated environment setup. And while we can use Chocolatey to save time with installations we still need to configure the stuff. Of course if you are using default settings this is not a problem but unfortunatelly this is not my case ;)

I think in my next post I will share my experience in automated configurations trasferring.