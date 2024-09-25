---
layout: post
title: My workflow philosophy
date: 2024-05-12
tags: [workflow]
---

## Introduction

This is a overview of my "workflow", a really long rant on how I deal with
technology choices related to my workflow, and why I do things "my way". I'm
hoping somebody will find this useful in making their choices.

I'm also writing this as a preface for my future blog posts where I discuss part
of my workflow. I'll keep the below updated with links as they come out:
* Working anywhere with SSH, FIDO2, and Git (coming _soon_)

## Summary of my workflow

At the moment I use 3 hosts (almost) daily:
* my desktop (Linux)
    * Windows dual booted for gaming
* my personal laptop (Linux)
* my work laptop (Linux)

On each hosts, some key technologies:
* Arch Linux encrypted with LUKS + btrfs (see [Linux
  Install](https://notes.jfx.ac/linux/install))
* Encrypted self-hosted WireGuard VPN to access most of my hosts, and TailScale
  for others
* I use SSH via FIDO2 to connect to my hosts, copy files
  * Alternative "personal" SSH Agent on my work machine to not leak keys from
    work
* Git operations to GitHub, commit signing, all with my FIDO2 key. Works over an
  SSH connection from another one of my hosts.
* File copying with `SCP` & `rsync`
* VS Code with the [Remote SSH
  extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)
    * Can do all the operations listed above in the built-in terminal
    * and [a ton of other
      extensions](https://github.com/itsjfx/dotfiles/blob/master/.config/extman/extensions.yaml)
      to help with my workflow
* Neovim
* tmux
    * Using my [custom tmux status
      bar](https://github.com/itsjfx/zsh-tmux-smart-status-bar) to give me
      context of my shell without flooding my prompt
    * if I'm working heavily in a shell, then I'll work in a tmux session over
      SSH, which means I can resume work from any one of my machines later
* Obsidian for note taking
    * Notes synced across devices
* Same base config and bootstrap across machines, through [my dot
  files](https://github.com/itsjfx/dotfiles), all managed in Git
    * the same [i3](https://i3wm.org/)/`x11` setup, key bindings, etc
* as well as a ton of other programs and utilities
    * more mentioned below

I will try write blog posts on each thing above. In the meantime, most of it is
in [my dot files](https://github.com/itsjfx/dotfiles), or I scribbled about it
in [my public notes](https://notes.jfx.ac) if you're bold enough to fish them
out.

## Principles

Generally I try to follow these principles in regards to my workflow. I also
follow some of these when I approach new technologies in system design, but not
all are mutually exclusive:

### 1) secure

This is a no-brainer.

Keep in mind, this is rated number one in importance, but not in practice.
There's a fine balance to strike as things must be secure, but still practical
and useable.

If I _really_ wanted things to be truly secure, then I would have no computer or
smart phone, and only pay in cash. Of course this is not practical.

Some relevant rules I try to follow are:
* limited public facing services (e.g. limit exposure of SSH to the internet)
* adopt hardware keys where supported (and where not lazy)
* where hardware keys are not possible, use a password manager, and other forms
  of 2 factor
* long passwords

These rules are important cause they dictate how I'm able to work. Now in order
to work, I _should_  have easy access to hardware keys, and easy access to my
non-public facing services.

### 2) simple and boring

I use a lot of boring technologies. If it's not boring, then I've likely had a
good reason to choose it, and have a decent understanding on how it works. I
likely understand how the "boring" alternative works in case the "cool" thing
I'm using is no longer "cool".

It's easy to bet on a bad technology that will no longer be supported or
maintained, but it's even easier to pick a good one. Use proven software that
has already solved the problem and has stood the test of time. Software like
Linux and GNU.

I'm also generally a slow adopter of new "cool" technology due to being cynical
and lazy. Unless I try something for the first time and really think, "wow this
is insane, I can't believe nobody else is using this!", I'll probably keep an
eye on it and see how it stands the test of time.

If I'm also getting all I need out of something, and I don't wish to put more
time into improving it, why would I risk my current workflow to move to
something "cooler"? I call this tactical laziness.

Adopting new technologies takes time and effort. That said, it's low risk and
effort to try something and see what life is like on the other side.

An example is [Obsidian](https://obsidian.md/). A colleague showed it to me, I
was impressed, then it took me a year to switch from my existing workflow with
Joplin and VS Code. Why? Cause I had everything set up great already. Linting
rules, note sync working, spell checking, and settings I felt were sensible.
Everything was _just right_.

To switch to Obsidian, I needed to migrate my notes, get my sync working, get my
settings right, plus learn whatever default / new features are applied by
Obsidian and flick things on or off. And the final thing (which will never stop)
is to learn how to optimise my workflow with all the new things I can do in
Obsidian.

The move to Obsidian has paid off though. It's made me take more notes and feel
better doing it. However, in the year between my colleague showing me Obsidian
and when I finally switched to Obsidian, I probably dealt with _some_ more
important things short-term things, but I still wish I'd switched sooner.

See also: <https://boringtechnology.club>

### 3) stable, but also have workarounds

Reality is things will catastrophically fail at some stage. If I cannot access
to my network via my VPN, do I have another point of access or some form of
break-glass?

This cascades from the last point: _generally_ simple things are stable, or
simple to troubleshoot and resolve.

### 4) use my preferred technology where possible

Some examples are:
* Linux as a desktop instead of Windows or Mac
* Firefox, with a container based workflow so I can continue using the same
  browser session
    * e.g. to login to multiple accounts
    * this means i retain my open tabs, hotkeys, extensions, and history
    * i may use profiles if different Firefox settings are needed
* CLI / terminal
    * I love using CLI tools or a terminal over a GUI as you can be more
      efficient. GUIs are opinionated, can differ between application, and are
      generally heavier and slower than using a terminal.
* open-source tooling over proprietary tooling (where applicable)
    * I still use Microsoft's Visual Studio Code build over the open-source one
* VS Code or `vim` to edit files, and finding ways to integrate more into these
  editors
* Hardware keys instead of 2FA
* Unix programs:
    * `bash`
    * `grep`
    * `cut`
    * `sed`
    * `jq`
    * `fzf` to find things

### 5) fast

Building from my previous point. I want to be fast when working on the computer.
If you do things faster, then you can do more things. It's a superpower, and
addicting. Once you get used to being fast you can't go back to being slow.

Having good, quick tooling, shortcuts / quicker ways of gaining access to
something or obtaining information will make you faster than most people.

This doesn't mean a bleeding edge, fast machine, or a gigabit internet
connection. It means spending less time doing or thinking about how to do "crap"
(cause you've automated it, or figured out you don't need to do it), and
spending that time on more meaningful things.

An example is `git` aliases, or tools like
[bash-my-aws](https://github.com/bash-my-aws/bash-my-aws). It seems like a
gimmick at first, but overall you save a lot of time not typing out the same
verbose commands 20 times a day. Once you get used to them, you'll find yourself
using them all the time instead of GUIs, and wanting to use GUIs less.

### 6) resumable

If I'm working on something I'd like to be able to resume it at any given time on the same machine or another one. This helps me not skip a beat as I may drop what I'm doing to do something else, or move somewhere (e.g. the couch) and want to continue what I was doing quickly.

Examples:
* my shell sessions all run `tmux`, so if I use a different machine I can SSH back in and attach the `tmux` session and pick up from where I was
* my Obsidian vault is synced live across my machines using [obsidian-livesync](https://github.com/vrtmrz/obsidian-livesync) so I can write things on my phone and get it real-time on my machines (and vice-versa).
* my `cmus` sessions resume from where I last used `cmus` by setting `resume=true`. See [man cmus](https://linux.die.net/man/1/cmus)

### 7) seamless experience between machines

I find it frustrating when devices or software across machines behave
differently. Of course it takes _some_ effort to set up a "settings sync" (built
into many modern applications now days), or [dot
files](https://github.com/itsjfx/dotfiles), but it pays off.

This is why I like hotkeys and config to be consistent across devices. It means
I don't need to think about how I'm using one of my machines, and I just use it.

Working across fundamental barriers (such as operating systems) makes this
challenging, but I've tried to keep things as seamless as possible.

For example, my [i3](https://i3wm.org/) setup uses tools like
[alttab](https://github.com/sagb/alttab) and
[i3-automark](https://github.com/lincheney/i3-automark) which make it _feel_ a
lot like Windows, which I used as my daily desktop operating system until 2022.
This makes it easy for me to use a Windows machine again as I've not lost all my
habits. I also don't think I've made myself slower by doing this. I like to
think that I've taken the best from Windows and brought it to my [tiling window
manager](https://en.wikipedia.org/wiki/Tiling_window_manager) workflow.

I also bootstrap my machines the same way with [my dot
files](https://github.com/itsjfx/dotfiles) and my scripts. [Even on
Windows](https://github.com/itsjfx/Win10-Initial-Setup-Script).

### 8) consistent experience across applications

I prefer using vim bindings where possible so I don't need to learn application
specific hotkeys. This enables me to use the keyboard more and be more
efficient.

Other basic things, like CTRL/ALT+Number to select tabs, CTRL + T to open tabs.
I want things to be the same.

### 9) version controlled

If I've set something up I like, it should be in some form version control. And
if not version controlled, then documented so I can revisit in the future.
