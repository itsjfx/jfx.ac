---
layout: post
title: Debugging my stupidity with mitmproxy
date: 2025-01-28
tags: [inspection, mitmproxy, neovim]
---

## Introduction

A colleague showed me a Neovim plugin that displayed errors from their LSP on virtual lines below their code, instead of adjacent to the code which is the default in Neovim. This interested me as I'd never thought about customising how errors are displayed in my Neovim.

I opened up ChatGPT and had a quick conversation on which plugins were available for doing customising error messages. To my surprise, ChatGPT suggested the same plugin that my colleague had just shown me. I then asked ChatGPT to tell me how to install the plugin with my plugin manager, Lazy.vim. What followed was a waste of 15 minutes of my day.

## The problem

The plugin my colleague showed me was [lsp_lines.nvim](https://git.sr.ht/~whynothugo/lsp_lines.nvim). It looks like this: {{< image src="/assets/blog/debugging-my-stupidity-with-mitmproxy/lsp_lines.png" >}}

I looked at the README and was convinced to try it out, so I asked ChatGPT to "help me install `lsp-lines.nvim`".

ChatGPT then suggested I add the following to my Lua config for `lazy.nvim`:
```lua
{
    "https://git.sr.ht/~whynothugo/lsp-lines.nvim",
    config = function()
        require("lsp_lines").setup()
    end,
}
```
I eyeballed the README for the project and it looked OK, so I slapped it in my Neovim config and tried it out.

After restarting Neovim, I got a HTTP 403 error during cloning the repository from Sourcehut.

{{< image src="/assets/blog/debugging-my-stupidity-with-mitmproxy/bad-clone.png" >}}

Not what I was expecting. I had no clue why Sourcehut would give me a 403 for a repository that clearly existed! It couldn't be SSH keys as I'd specified a HTTP URL. My first thought was Lazy was appending `.git` to the URL which works on GitHub but not on Sourcehut.

I looked at my colleagues config and his looked the same, and we also had the same version of Lazy installed.

Next, I went to the directory where Lazy clones plugins, copied the URL for the repository from my browser, and ran `git clone` ... and that worked! But when I opened Neovim, it tried cloning the repository and it failed again.

I checked the source code for Lazy and saw it ran `git clone`, but didn't seem to modify the URL, so my `.git` hypothesis was unlikely. I found it odd that Lazy would attempt cloning the repository even though the repository was on my machine, maybe cause I'd cloned it manually?

I needed to get more information on what was happening when it reached out to Sourcehut.

## Enter mitmproxy

I've wrote about [mitmproxy](https://mitmproxy.org/) before [when I was hacking IoT devices](https://jfx.ac/blog/homelab-mikrotik-inspect-network/). It can be used to look at the HTTP requests Lazy makes when pulling the plugin.

First we need to start capturing traffic. I have an alias that runs `mitmdump --flow-detail=4 --showhost --set hardump=/tmp/dump.har`, which outputs traffic to the terminal in real time and will write a [HAR file](https://en.wikipedia.org/wiki/HAR_(file_format)) (essentially a `HTTP` traffic dump in `JSON` format) on exit.

## Enter mitmwrap

Next, we need Neovim to send its traffic through the proxy, and also make sure Neovim trusts the SSL certificate for `mitmproxy`. I have a script [in my dotfiles called mitmwrap](https://github.com/itsjfx/dotfiles/blob/master/bin/mitmwrap) for this.

The usage is `mitmwrap [MITM_ARGS] PROGRAM [ARGS]`. It currently has two modes: setting the `HTTP_PROXY` variables to point to my `mitmproxy`, or using `proxychains` to force an application to use `mitmproxy`. I plan on adding [graftcp](https://github.com/hmgle/graftcp) support later.

The script attempts to [set common environment variables](https://github.com/itsjfx/dotfiles/blob/2e76936d6f4f01cc7d3ab840dac9c2a438d6d03f/bin/mitmwrap#L38) used by programs to look up the systems SSL store. I point these to the `mitmproxy` SSL certificate. Lastly, it runs the program in a lightweight container using [bubblewrap](https://github.com/containers/bubblewrap) , with `/etc/ssl/certs/ca-certificates.crt` mounted to our `mitmproxy` certificate in case the program does not use any of those earlier environment variables.

`bubblewrap` not only has a cool name, but is also a cool program, and is used internally by Flatpak to sandbox applications.

Why are we using bubblewrap? We aren't using it to sandbox our program for security reasons, but cause `/etc/ssl/certs/ca-certificates.crt` is owned by `root`. I don't want to run commands as `root` to change the file when I should be able to do everything in user-land.

I also don't want `mitmproxy`'s certificate in my systems cert store as it affects every program on my system, and is a potential security risk.

Instead `bubblewrap` can be run with `--dev-bind / /` which is [supposedly a no-op](https://wiki.archlinux.org/title/Bubblewrap#No-op), before giving  `--ro-bind MITM_CERT_FILE /etc/ssl/certs/ca-certificates.crt` to bind the SSL file in read-only mode. There's a lot of options that could be set, but I set as little as possible to not break any programs.

The time to launch `bubblewrap` is a few milliseconds which is great for this use case. There's a [detailed blog on bubblewrap performance](https://jvns.ca/blog/2022/06/28/some-notes-on-bubblewrap/) and using it as a Docker replacement if you're curious. 

I've ran my `mitmwrap` script a lot to write CLI wrappers for programs that make API calls. Most recently I used it to craft an API call similar to what the GitHub CLI (`gh`) made to its GraphQL endpoint. It was quicker than reading the docs and figuring out how to craft the query myself.

Bonus from today: I used the script to troubleshoot how [aider](https://github.com/Aider-AI/aider) was communicating with AWS Bedrock as it was failing and there were no helpful logs. I noticed `aider` was hitting `us-west-2` instead of `ap-southeast-2`, and I quickly realised my `AWS_REGION` environment variable was not set, while `AWS_DEFAULT_REGION` was... which Aider does not read. Whoops!

## Getting to the bottom of my cloning issue

`mitmdump` gave pretty output in the console, but I still couldn't figure it out. It looked like the URL  was formed well.

{{< image src="/assets/blog/debugging-my-stupidity-with-mitmproxy/bad.png" >}}

I also ran the `git clone` that worked earlier via `mitmwrap` and captured the traffic, and the request looked identical, except it had a 200 response code.

{{< image src="/assets/blog/debugging-my-stupidity-with-mitmproxy/good.png" >}}

Generally the console output is pretty helpful, but I knew something _had_ to be different, so I decided to look at the HAR file. I finally got to the bottom of it after running this command:

```bash
diff <(cat /tmp/dump.har | jq -rc '.log.entries[0].request | select(.method == "GET")' | gron) <(cat /tmp/dump.har | jq -rc '.log.entries[1].request | select(.method == "GET")' | gron)
```
This command reads the first HTTP GET requests JSON with [jq](https://jqlang.github.io/jq/), and pipes it into [gron](https://github.com/tomnomnom/gron), and does the same for the second HTTP GET request. It then diffs the two outputs.

`jq` is a very powerful CLI JSON processor, and `gron` flattens JSON to make them easier to `grep` or parse. e.g.

```bash
echo '{"hello": "blog", "x": [1, 2]}' | gron
json = {};
json.hello = "blog";
json.x = [];
json.x[0] = 1;
json.x[1] = 2;
```
I find `gron` nice for feeding JSON into `diff` as it flattens it onto new lines.

What was the cause?? Well here is the output:
```diff
30c30
< json.url = "https://git.sr.ht/~whynothugo/lsp-lines.nvim/info/refs?service=git-upload-pack";
---
> json.url = "https://git.sr.ht/~whynothugo/lsp_lines.nvim/info/refs?service=git-upload-pack";
```
The difference is clear as day on my colourised `diff` output. I had been giving my Neovim `lsp-lines.nvim`, when the real URL is `lsp_lines.nvim` with an underscore.

## How did this happen

I never typed the complete URL out manually in my Neovim config, so how did this happen? Well earlier in [#the-problem](#the-problem), I had asked ChatGPT to give me the config for `lsp-lines.nvim`. I had not typed the name of the repository properly, so ChatGPT ran with it and gave me the URL in the same form with a hyphen instead of an underscore.

I also had no coffee for the day.

## Reflecting

The URL in the example given by ChatGPT was incorrect, so I was doomed from the start. I reflected afterward and realised there were red flags and clear signs of why this was happening:
1. The 403 error was likely due to a URL typo
2. Cloning the repository locally
3. Neovim still trying to pull the plugin with the folder existing

At each of these points I should've figured out what was going on. Using `mitmproxy` and all the other tools felt like using a chainsaw to cut butter, but I'd already had these things pre-baked, so consuming them was not costly.

I was very grateful to have my own tooling to help me out here, as the next step was probably to get someone to pair with me and waste their time :) or drop the whole thing :)

I hope this story encourages you to brush up your scripts and keep them around in your dotfiles. You never know when you may need them again.

## The plugin

At the end of the day I decided not to continue using `lsp_lines.nvim`, as in certain repositories it was moving lines around a lot and got annoying. I may use it again in the future with a toggle, but the default diagnostic view from Neovim is fine with me for now. I really wasted my time.

{{< image src="/assets/blog/debugging-my-stupidity-with-mitmproxy/default.png" >}}

## See also

* <https://jvns.ca/blog/2022/06/28/some-notes-on-bubblewrap>
* <https://wiki.archlinux.org/title/Bubblewrap>
* <https://mitmproxy.org/>
* <https://github.com/itsjfx/dotfiles/blob/master/bin/mitmwrap>
