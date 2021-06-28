---
layout: post
title: "Better Command Line Experience on Windows With ConEmu, Clink and Oh My Posh"
date: 2021-06-20 16:11
comments: true
categories:
 - Development
 - Tools
tags:
 - Command Line
 - ConEmu
 - Clink
 - Oh My Posh
---

Recently, I stumbled upon Brad Wilson’s post - [Anatomy of a Prompt (PowerShell)](https://bradwilson.io/blog/prompt/powershell) and decided that I also want to have a fancy-looking command prompt for a cmd.exe. Fanciness includes but not limited to:

 - A custom prompt that display computer name and current user, git status, and features pretty looking powerlines
 - Persistent commands history
 - Command completion, aliases/macros support + their expansion on demand

This is what my console looks like after all modifications:

{% img center /images/posts/awesome-looking-command-line-prompt.gif 'Awesome looking command line prompt' 'Awesome looking command line prompt' %}

At first, I was planning to give an overview of my current setup, but then the description grew, and now I have a detailed guide about how to improve the look and feel of the CMD.

Table of contents:
 - [Set Command Aliases/Macros For CMD.exe In ConEmu]()
 - [Integrate Clink]()
   - [Configure Clink Completions]()
 - [Change The Prompt With Oh My Posh]()
 - [Conclusions]()

## Set Command Aliases/Macros For CMD.exe In ConEmu

For many years I have been a loyal and happy user of ConEmu. This tool is great - it is reliable, fast, and highly configurable. ConEmu has a portable version, so setup replication is not a problem - I keep it in the ever-growing list of utilities on my cloud storage.

One of the issues with cmd is the absence of persistent user-scoped command aliases or macros. Yes, there is a [DOSKEY](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/doskey) command, but you are required to integrate its invocation into your cmd startup. To get more control over my macros setup, I wrote a utility ([GitHub - manekovskiy/aliaser](https://github.com/manekovskiy/aliaser)) that pulls a list of command aliases from the file and sets them up for the current process.

> ConEmu provides native support for command aliases; refer to [ConEmu | Settings › Environment](https://conemu.github.io/en/SettingsEnvironment.html) page for more details.

My setup is following:

 - Put compiled aliaser.exe and a file containing aliases (my list - [GitHub - aliases.txt](https://gist.github.com/manekovskiy/57e98863e0b04e7ce3b9d3346486b3aa)) to the `%ConEmuBaseDir%\Scripts` folder.
 - Add a batch file containing the invocation of the aliaser utility to the `%ConEmuBaseDir%\Scripts` folder.

{% codeblock setup-aliases.cmd lang:bat %}
@echo off
call "%~dp0aliaser.exe" -f "%~dp0aliases.txt"
{% endcodeblock %}

 - Update ConEmu CMD tasks to include setup-aliases.cmd invocation. Example: `cmd.exe /k setup-aliases.cmd`

> There is no need to provide a path to the batch file because the default ConeEmu setup adds a `%ConEmuBaseDir%\Scripts` folder to the PATH environment variable.

## Integrate Clink

Another great addition to the cmd is a Clink - it augments the command line with many great features like persistent history, environment variable names completion, scriptable keybindings, and command completions.

Again, ConEmu provides integration with Clink (see [ConEmu | cmd.exe and clink](https://conemu.github.io/en/TabCompletion.html#ConEmu_and_clink)). It is important to note that ConEmu works well only with the current active fork of the clink project - [chrisant996/clink](https://github.com/chrisant996/clink).

In short, to install and enable Clink in ConEmu, you should extract the contents of the clink release archive into the `%ConEmuBaseDir%\clink` and check the “Use Clink in prompt” under the Features settings section.

{% img center /images/posts/conemu-settings-features-use-clink-in-prompt.png 'Enable &quot;Use Clink in prompt&quot; configuration setting' 'Enable &quot;Use Clink in prompt&quot; configuration setting' %}

An indicator of successful integration is the text mentioning the Clink version and its authors, similar to the following:

    Clink v1.2.9.329839
    Copyright (c) 2012-2018 Martin Ridgers
    Portions Copyright (c) 2020-2021 Christopher Antos
    https://github.com/chrisant996/clink

### Configure Clink Completions

One of the most powerful features of the Clink is that it is scriptable through Lua. It is possible to add custom command completion logic, add or change the keybindings or even modify the look of the prompt.

On startup, Clink looks for a clink.lua script, which is an entry point for all extension registrations. There are a couple of places where Clink tries to locate the file, one of them is the `%CLINK_INPUTRC%` folder. There should be an empty `clink.lua` file in the `%ConEmuBaseDir%\clink` folder (comes as a part of the Clink release). To make it visible to ConEmu and Clink, add a `CLINK_INPUTRC` variable to the ConEmu Environment configuration: `set CLINK_INPUTRC=%ConEmuBaseDir%\clink`.

{% img center /images/posts/conemu-settings-startup-environment-add-CLINK_INPUTRC-variable.png 'Add &quot;CLINK_INPUTRC&quot; variable to the ConEmu Startup settings' 'Add &quot;CLINK_INPUTRC&quot; variable to the ConEmu Startup settings' %}

Not so long ago, I found that Cmder (a quite opinionated build of ConEmu) distribution already contains Clink and completion files for all super popular command-line utilities. A little bit of search showed that completions in Cmder come from the [GitHub - vladimir-kotikov/clink-completions](https://github.com/vladimir-kotikov/clink-completions) repository.

Download the latest available clink-completions release and unpack it in the `%ConEmuBaseDir%\clink\profile`. I decided to drop the version number from the clink-completions folder name, so I would not have to update the registration script every time I update the completions. I also made the registration of the Clink extensions maximally universal. I went with a convention-based approach to files and folders organization:

 - Each Clink extension should be put into a separate folder under `%ConEmuBaseDir%\clink\profile`. This ensures proper grouping and logical separation of scripts.
 - Each Clink extension group should define a registration script under `%ConEmuBaseDir%\clink\profile`.
 - Code in `clink.lua` should locate all registration scripts under `%ConEmuBaseDir%\clink\profile` and unconditionally execute them.

Example folder structure:

    📂 %ConEmuBaseDir%\clink
      📂 profile
        📁 clink-completions
        📁 extension-x
        📁 oh-my-posh
        📄 clink-competions.lua
        📄 oh-my-posh.lua
        📄 extension-x.lua
      📄 clink.lua

The registration script is simple:

{% codeblock clink.lua lang:lua %}
-- clink.lua

-- Globals
__clink_dir = clink.get_env('ConEmuBaseDir')..'/clink/'
__clink_profile_dir = clink.get_env('ConEmuBaseDir')..'/clink/profile/'

-- Load profile scripts
for _,lua_module in ipairs(clink.find_files(__clink_profile_dir..'*.lua')) do
    local filename = __clink_profile_dir..lua_module
    -- use dofile instead of require because require caches loaded modules
    -- so config reloading using Alt-Q won't reload updated modules.
    dofile(filename)
end
{% endcodeblock %}

The clink-completions registration script is also a barebone minimum. I extracted it from the Cmder's [clink.lua](https://github.com/cmderdev/cmder/blob/946f929eafe0f9017d1f3cb9f8d144c7f61064e0/vendor/clink.lua#L505):

{% codeblock clink-completions.lua lang:lua %}
-- clink-completions.lua
-- Completions scripts taken from from https://github.com/vladimir-kotikov/clink-completions
-- Last updated on 6/5/2021. Version 0.3.7.

local completions_dir = __clink_profile_dir..'clink-completions/'
-- Execute '.init.lua' first to ensure package.path is set properly
dofile(completions_dir..'.init.lua')
for _,lua_module in ipairs(clink.find_files(completions_dir..'*.lua')) do
    -- Skip files that starts with _. This could be useful if some files should be ignored
    if not string.match(lua_module, '^_.*') then
        local filename = completions_dir..lua_module
        -- use dofile instead of require because require caches loaded modules
        -- so config reloading using Alt-Q won't reload updated modules.
        dofile(filename)
    end
end
{% endcodeblock %}

## Change The Prompt With Oh My Posh

If you never heard of it before, [Oh My Posh](https://ohmyposh.dev/) is a command prompt theme engine. It was first created for PowerShell, but now, in V3, Oh My Posh became cross-platform with a universal configuration format, which means that you can use it in any shell or OS.

> ConEmu distribution contains an initialization script file - CmdInit.cmd, which can display the current git branch and/or current user name in the command prompt (see [ConEmu | Configuring Cmd Prompt](https://conemu.github.io/en/CmdPrompt.html) for more details).

When it comes to CMD, Oh My Posh could be integrated through Clink. Clink has a concept of [prompt filters](https://chrisant996.github.io/clink/clink.html#customising-the-prompt) - code that executes when the prompt is being rendered.

Installation and integration with Clink steps are very straightforward:

 1. Download the latest release
 2. Move the executable to the `%ConEmuBaseDir%\clink\profile\oh-my-posh\bin` folder
 3. Add oh-my-posh.lua script to the `%ConEmuBaseDir%\clink\profile` folder
 4. Create a theme file (I named mine amanek.omp.json)

Following is the expected folder structure:

    📂 %ConEmuBaseDir%\clink
      📂 profile
        📂 oh-my-posh
          📂 bin
            📦 oh-my-posh.exe <-- note, that I renamed the executable file to oh-my-posh.exe.
          🎨 amanek.omp.json
        📄 oh-my-posh.lua
      📄 clink.lua

The registration script was inspired by the Clink project readme file:

{% codeblock oh-my-posh.lua lang:lua %}
-- oh-my-posh.lua
-- Taken from https://github.com/chrisant996/clink/blob/master/docs/clink.md#oh-my-posh

local ohmyposh_dir = __clink_profile_dir.."oh-my-posh/"
local ohmyposh_exe = __clink_profile_dir.."oh-my-posh/bin/oh-my-posh.exe"

local ohmyposh_prompt = clink.promptfilter(1)
function ohmyposh_prompt:filter(prompt)
    prompt = io.popen(ohmyposh_exe.." --config "..ohmyposh_dir.."amanek.omp.json --shell universal"):read("*a")
    return prompt, false
end
{% endcodeblock %}

The Oh My Posh comes with a wide variety of prebuilt themes. The customization process is well described in the [Override the theme settings](https://ohmyposh.dev/docs/windows#override-the-theme-settings) documentation section.

Here is the link to my theme file - [GitHub - amanek.omp.json](https://gist.github.com/manekovskiy/70698cc9309a833582bb750221c49e04). It includes the following sections:

 - Indicator of elevated prompt. Displays a lightning symbol if my console instance is running as Administrator.
 - Logged-in user name and computer name. I frequently connect to different machines over RDP, so it is good to know where am I right now 😊
 - Location path
 - Git status

One of the issues I encountered during the command prompt customization process is that ConEmu remaps console colors and replaces them with its own color scheme:

{% img center /images/posts/conemu-features-colors.png 'ConEmu Colors configuration section' 'ConEmu Colors configuration section' %}

As you can see, ConEmu uses a color scheme based on the 16 ANSI colors. Fortunately, in Oh My Posh, it is also possible to specify color using one of the well-known 16 color names (see [standard colors documentation section](https://ohmyposh.dev/docs/configure/#standard-colors)). Here is the “ConEmu color number to Oh My Posh color name” conversion table:

**Number**  | **Color**
----------- | ------
0           | black
1/4         | blue
2           | green
3/6         | cyan
4/1         | red
5           | magenta
6/3         | yellow
7           | white
8           | darkGray
9           | lightBlue
10          | lightGreen
11          | lightCyan
12          | lightRed
13          | lightMagenta
14          | lightYellow
15          | lightWhite

Another thing that did not work for me right away was fonts. To render powerlines and icons, Oh My Posh requires the terminal to use a font that contains glyphs from the [Nerd Fonts](https://github.com/ryanoasis/nerd-fonts). Nerd Fonts readme contain links to the patched and supported fonts with permissive licensing terms.

The font that I prefer to use was not on the list, so I had to patch it manually. The process is following:

1. Clone Nerd Fonts repository
2. Install Font Forge and Python
3. Copy font file to a separate folder
4. In the terminal, navigate to the Nerd Fonts repo root and run the following command

{% codeblock lang:bat %}
<PATH_TO_FONT_FORGE>\fontforge.exe -script font-patcher --windows --complete <PATH_TO_FONT_FILE>
{% endcodeblock %}

More detailed instructions here - [Nerd Fonts | Patch Your Own Font](https://github.com/ryanoasis/nerd-fonts#option-8-patch-your-own-font).

> The Nerd Fonts repository is heavy, and I recommend doing a shallow clone with --depth 1 option.

## Conclusions

The amount of work people put into the open-source projects I mentioned is astounding. I was also pleasantly surprised with the quality of tools and customization options available for CMD. Never before my terminal window was so aesthetically pleasing and functionally rich. Now I feel more inspired to continue experimenting with my setup and hope that this guide helped to improve your console experience.

Good luck and happy hacking!