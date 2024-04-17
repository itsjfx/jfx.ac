---
layout: post
title: Logging into Entra ID via FIDO2 with Firefox on Linux
date: 2024-04-02
tags: [fido2, entra, azure, firefox, linux]
---

## Introduction

Entra ID (formerly Azure AD) is a widely used IDP in many enterprise
environments. Despite [Mozilla enabling FIDO2 support by default (as of
114.0)](https://www.mozilla.org/en-US/firefox/114.0/releasenotes), you cannot
login to Entra ID with Firefox on Linux.

The two are listed as incompatible [in Microsoft's
documentation](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-fido2-compatibility#web-browser-support),
but Chromium on Linux works fine. After some digging, I've found that Firefox
works with simple workarounds.

## Workarounds

### 1) Set your user agent to a Chromium browser

* You can test this quickly by setting `general.user.agent.override` in
  `about:config` to a Chromium user-agent
    * e.g. `Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like
      Gecko) Chrome/123.0.0.0 Safari/537.36`
* Or you can install a [User Agent Switcher
  addon](https://addons.mozilla.org/en-US/firefox/addon/user-agent-string-switcher/)

This works but means you must always spoof your user-agent, at least for Entra
ID. I was glad to find this, but I wanted to figure out _why_ this works.

### 2) Use my user-script

After tracing the JavaScript, I figured out why being on Chromium makes a
difference, and how to fix the problem on Firefox.

My solution is to patch 4 variables in the global JavaScript `window` which
dictate what login flow you face when logging into Entra.
* My gist is [located
  here](https://gist.github.com/itsjfx/e9e63130ba17a180a2e42294a2d955d5)
* and a [raw link
  here](https://gist.github.com/itsjfx/e9e63130ba17a180a2e42294a2d955d5/raw/75157271fae2e7f89b13e8ec43e2037ac673c187/azure_login_fido2_fix.user.js)
  to install with a userscript manager, e.g.
  [Tampermonkey](https://addons.mozilla.org/en-US/firefox/addon/tampermonkey) or
  [Greasemonkey](https://addons.mozilla.org/en-US/firefox/addon/greasemonkey/)

I've been using this almost everyday since Firefox 114.0 was released in June,
and I've not faced any problems.

## Why does the user-script work?

When I discovered a user-agent change was needed, I was motivated to figure out
a better way to fix the problem, and also understand how Microsoft disabled it
for Firefox.

I eyeballed the JavaScript and noticed a couple of checks for FIDO which checked
browser APIs such as
[navigator.credentials.get()](https://developer.mozilla.org/en-US/docs/Web/API/CredentialsContainer/get)
as well as checks from a global `window` variable, `$Config` to check if
[WebAuthn](https://en.wikipedia.org/wiki/WebAuthn) is supported.

I knew from reversing the <https://login.microsoftonline.com> site in the past
that `$Config` is set by the backend in the HTML. It stores flags and state
variables for the client.

A quick diff of the `$Config` JSON keys with `gron` between Firefox and
Chromium, showed that Firefox was missing a `fIsFidoSupported` variable which
Chromium had.

```bash
diff <(gron firefox.json | sed 's/ =.*//' | sort -u) <(gron chromium.json | sed 's/ =.*//' | sort -u)
```

gave:
```diff
> json.browser.Chrome
16d16
< json.browser.Firefox
20c20
< json.browser._M124
---
> json.browser._M123
24c24
< json.browser.RE_Gecko
---
> json.browser.RE_WebKit
100a101
> json.fIsFidoSupported
179a181,182
> json.oAppCobranding.signinDescription
> json.oAppCobranding.signinTitle
543a547,548
> json.urlFidoHelp
> json.urlFidoLogin
562a568,569
> json.urlPostAad
> json.urlPostMsa
```

I tried setting `fIsFidoSupported` and `urlFidoLogin`, but still received an
error:

![Trouble signing in
error](/assets/blog/entra-id-fido2-firefox-linux/passkey-error.png)

I wasn't expecting this to work the first time cause I was taking shortcuts and
troubleshooting backwards. I wanted to do the right thing and step through it,
but the code was difficult to trace cause most of it is event based. I felt I
was close and had time to burn so I continued being reckless and trying things.

Missing `$Config` variables were a good idea, but I never tried injecting the
entire working Chromium `$Config` into Firefox. I should've tried this first.

That sort of worked... I got the FIDO2 prompt, logged in, but then got a
different error page. In my books a different error is a good error. I suspect
this error occurred cause I was using variables from a separate login session.

![Trouble signing in
error](/assets/blog/entra-id-fido2-firefox-linux/token-error.png)

At this stage I had a pretty big hunch that I was on the right path, I'd just
need to figure out what variables are needed. It could be a combination of
variables being missing, or variables having different values (e.g. a `false`
on Firefox, but a `true` on Chromium).

The `$Config` JSON was 629 lines long, but not difficult to brute-force.
Inspired by [git bisect](https://git-scm.com/docs/git-bisect) and binary search
I started replacing variables of the JSON like a maniac until I received no
errors, then cut it down even more.

After a gruelling ten minutes, the missing offenders were `urlPostAad` and
`urlPostMsa`. They were in the missing `diff` list from earlier. I could've
avoided this time-consuming exercise!

It was clear why it was failing after checking the JS. The `urlPost` variables
are used in a `POST` request to the `urlFidoLogin` URL mentioned earlier. I
could've picked this up diffing the `POST` requests.

```javascript
var _postUrl = _serverData.urlPost;
var _aadPostUrl = _serverData.urlPostAad;
// ...
var postParams =
{
    allowedIdentities: _allowedIdentities,
    canary: _fidoChallenge,
    serverChallenge: _fidoChallenge,
    postBackUrl: _postUrl,
    postBackUrlAad: _aadPostUrl,
    postBackUrlMsa: _msaPostUrl,
    cancelUrl: _loginUrl,
    resumeUrl: _resumeUrl || _loginUrl,
    correlationId: _correlationId,
    credentialsJson: _allowList,
    ctx: _originalRequest,
    username: _username
};
```

Now I had all the pieces and I could finally login with Firefox.

## End

FIDO2 is awesome and I'm always trying to find ways to increase adoption. People
in my team using Linux were reluctant to use FIDO2 on their account cause they
could only use Chromium, but now with these workarounds they can use their
favourite browser and things such like
[Containers](https://support.mozilla.org/en-US/kb/containers). Logging in is
also a lot quicker than TOTP/Authenticator and more secure, so everybody is a
lot happier.

I'm not sure why Microsoft has chosen to do this... all I know is I'm happy
logging in with my key.
