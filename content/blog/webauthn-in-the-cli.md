---
layout: post
title: WebAuthn in the CLI
date: 2024-12-20
tags:
  - fido2
  - webauthn
  - python
---

## Introduction

The first thing that often comes to mind about FIDO2 WebAuthn clients is their integration with modern web browsers. Typically, users are prompted on a website to verify their identity by entering a PIN and interacting with a physical security key. This user-friendly flow is designed to to be fast, secure, and reduce reliance on passwords.

But what if you need to implement this functionality outside of a browser? What if your application runs in a command-line interface (CLI)? The good news is this is possible, but it's not well documented or commonly implemented. I was motivated to implement this as leaving a terminal and waiting for a page to load to tap a device seemed wasteful.

This post will guide you through the process of implementing a FIDO2 WebAuthn client in Python. In our example, we wrap <https://webauthn.io> and cover some of the main concepts along the way. With enough focus this approach could be used to wrap any IDP or service.

## Deep dive into WebAuthn

### Summary

Mozilla has a detailed [explanation here](https://developer.mozilla.org/en-US/docs/Web/API/Credential_Management_API/Credential_types#web_authentication_assertions) . I will try summarise below.

A typical WebAuthn authentication process in a browser looks like this:
1. The "relying party" (the website) sends user and server information to the web app handling the registration process, along with a "challenge"
2. The browser asks the authenticator device (e.g. YubiKey) to sign the challenge via a [`navigator.credentials.get()`](https://developer.mozilla.org/en-US/docs/Web/API/CredentialsContainer/get "navigator.credentials.get()") call. This process is called getting an assertion.
3. If the authenticator contains one of the given credentials and is able to successfully sign the challenge, it returns a signed assertion to the web app after receiving user consent
4. The web app forwards the signed assertion to the relying party server to validate
5. If everything checks out, the relying party will return a successful response to the web app

{{< image src="/assets/blog/webauthn-in-the-cli/Passwordless_Web_Authentication.svg.png" >}}

*(from [Trscavo](https://en.wikipedia.org/wiki/User:Trscavo) at English Wikipedia [available here](https://commons.wikimedia.org/wiki/File:Passwordless_Web_Authentication.svg))*

### Input

Drilling further into `navigator.credentials.get()`, it expects the following fields as parameters/inputs:
* [`challenge`](https://developer.mozilla.org/en-US/docs/Web/API/PublicKeyCredentialRequestOptions#challenge) - a value originating from the relying party's server and used as a [cryptographic challenge](https://en.wikipedia.org/wiki/Challenge%E2%80%93response_authentication)
* [`rpId`](https://developer.mozilla.org/en-US/docs/Web/API/PublicKeyCredentialRequestOptions#rpid) (optional) - a string that specifies the relying party's identifier
    * This is used for identifying which keys to use and for preventing cross-origin attacks
    * an example value may be `login.microsoft.com`
* [`allowCredentials`](https://developer.mozilla.org/en-US/docs/Web/API/PublicKeyCredentialRequestOptions#allowcredentials) (optional) - A restricted the list of acceptable credentials. An empty array indicates that any credential is acceptable.
    * An empty list is typically used when the relying party is unaware of your username
    * For example, [GitHub sends an empty allowCredentials list](https://notes.jfx.ac/fido2/github/#multiple-accounts-on-a-single-fido2-device) when you press "Sign in with a passkey".
    * If you have multiple credentials registered under the relying party, your browser will ask you to select a credential to authenticate with
* [`timeout`](https://developer.mozilla.org/en-US/docs/Web/API/PublicKeyCredentialRequestOptions#timeout) (optional) - a hint indicating the time the relying party is willing to wait for the retrieval operation to complete
* [`userVerification`](https://developer.mozilla.org/en-US/docs/Web/API/PublicKeyCredentialRequestOptions#userverification) (optional) - A enumerated string specifying the relying party's requirements for user verification of the authentication process
    * e.g. whether or not to do PIN verification

More detail on each option is available:
* In the [Mozilla developer documentation](https://developer.mozilla.org/en-US/docs/Web/API/PublicKeyCredentialRequestOptions)
* and in the [WebAuthn specification](https://www.w3.org/TR/webauthn-2/#dictionary-assertion-options)

Most fields are optional, however if specified by the relying party they are important and should not be omitted.

### Output

The function then returns [an object with type PublicKeyCredential](https://developer.mozilla.org/en-US/docs/Web/API/PublicKeyCredential). Typically an identity provider (IDP) will expect this entire object to be sent back, or a subset of the fields under the `response` key which [has type AuthenticatorAssertionResponse](https://developer.mozilla.org/en-US/docs/Web/API/AuthenticatorAssertionResponse).

The important fields are:
* [`authenticatorData`](<https://developer.mozilla.org/en-US/docs/Web/API/AuthenticatorAssertionResponse/authenticatorData>) - contains information about which relying party and credential was used
    * your device may set an interesting field, [`signCount`](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API/Authenticator_data#signcount), a [signature counter](https://www.imperialviolet.org/2023/08/05/signature-counters.html) used to prevent device cloning
    * if `signCount` is set, then repeated assertions will not return an identical `authenticatorData` value, thus each `signature` will be a unique value
* [`clientDataJSON`](https://developer.mozilla.org/en-US/docs/Web/API/AuthenticatorResponse/clientDataJSON) - contains the `challenge` and  the origin used to prevent cross-origin attacks
* [`signature`](https://developer.mozilla.org/en-US/docs/Web/API/AuthenticatorAssertionResponse/signature) - the combined signature of `authenticatorData` and `clientDataJSON`, signed with the private key of the selected credential associated with the relying party

Once a server receives these values, it can verify the signature was created by the correct credential, and can validate the contents of `authenticatorData` and `clientDataJSON`.

## Inspecting webauthn.io

<https://webauthn.io> is a public service for testing passkeys. This makes it the perfect test bed for understanding the flow for a WebAuthn client and testing our CLI client. We may create a bunch of malformed requests, so it's better to do this to a test service and not a real IDP as we may get locked out of our real account.

I won't be covering registering passkeys in a CLI cause this is only completed once per security key. For the curious, this done in the browser via [navigator.credentials.create()](https://developer.mozilla.org/en-US/docs/Web/API/CredentialsContainer/create).

Assuming we have already registered our passkey to a username, we can open the network explorer and observe a dual request log in flow:
1. A POST call to `https://webauthn.io/authentication/options`
    * With a JSON request body: `{"username":"jfxac","user_verification":"preferred"}`
    * And a JSON response body: `{"challenge": "BASE64URL_CHALLENGE", "timeout": 60000, "rpId": "webauthn.io", "allowCredentials": [{"id": "BASE64URL_ID", "type": "public-key", "transports": ["usb"]}], "userVerification": "preferred"}`
        * Luckily these are the exact key value pairs [used by `navigator.credentials.get()`](#input)
2. A POST call to `https://webauthn.io/authentication/verification`
    * With a JSON request body: containing the [full response of `navigator.credentials.get()`](#output)
    * And a JSON response body: `{"verified": true}`

With all the information required, we can write some code to call the APIs and feed the values into a WebAuthn client.

## Writing some code

Yubico have a created a [Python package called fido2](https://github.com/Yubico/python-fido2) which has a [WebAuthn client](https://developers.yubico.com/python-fido2/API_Documentation/autoapi/fido2/client/index.html). They also provided [an example](https://github.com/Yubico/python-fido2/blob/main/examples/credential.py) which acts as both a server & client and allows for registering a credential and getting an assertion. We will use this code as a base for our webauthn.io wrapper.

The psuedocode is roughly:
1. Initialise a FIDO2 WebAuthn client
2. Call <https://webauthn.io/authentication/options> with username provided by command-line arguments
3. Pass the assertion options to the client, and wait for the user to complete the perform the actions
4. Once complete, call <https://webauthn.io/authentication/verification> with the values returned by the WebAuthn client
5. If successful, write something to the terminal and exit successfully - - otherwise write any errors to the terminal and exit unsuccessfully

A POST request to webauthn.io to kick off a log in:
```python
import requests

COOKIES = {
    'csrftoken': generate_random_string(32),
    'sessionid': generate_random_string(32),
}

def get_options(username):
    data = {
        'username': username,
        'user_verification': 'required',
    }

    response = requests.post('https://webauthn.io/authentication/options', cookies=COOKIES, json=data)
    return response.json()

print(get_options(username))
```

This will give our `challenge`, `rpId`, `allowCredentials`, and `userVerification` values.

<br>
We can then call our WebAuthn client, passing in the above key value pairs:

I've removed the client initialisation code for this snippet but it's available here: <https://github.com/Yubico/python-fido2/blob/main/examples/exampleutils.py#L70>

```python
from fido2.utils import websafe_decode
...
...
options = get_options()
client = get_client()
request_options = {
    'rpId': options['rpId'],
    'challenge': websafe_decode(options['challenge']),
    'timeout': 600000,
    'allowCredentials': [PublicKeyCredentialDescriptor(type=cred['type'], id=websafe_decode(cred['id']), transports=cred['transports']) for cred in options['allowCredentials']],
    'userVerification': options['userVerification'],
}
result = client.get_assertion(request_options)
result = result.get_response(0)
```

`result` will contain our signed assertion, which can be sent to webauthn.io

Note the `websafe_decode` method used to give the FIDO2 library the raw challenge. A typical mistake is to give the WebAuthn client the challenge incorrectly encoded. In that case, the client will still sign an assertion, but the IDP will return a failure as the challenge is incorrect.

<br>

With the assertion from our device, we can make a POST request to webauthn.io with the fields populated:
```python
from fido2.utils import websafe_decode
import requests

def respond(username, result):
    data = {
        'username': username,
        'response': {
            'authenticatorAttachment': 'cross-platform',
            'clientExtensionResults': {},
            'id': websafe_encode(result['credentialId']),
            'rawId': websafe_encode(result['credentialId']),
            'type': 'public-key',
            'response': {
                'clientDataJSON': websafe_encode(result['clientDataJSON']),
                'authenticatorData': websafe_encode(result['authenticatorData']),
                'signature': websafe_encode(result['signature']),
                'userHandle': websafe_encode(result['userHandle']),
            },
        },
    }

    response = requests.post('https://webauthn.io/authentication/verification', cookies=COOKIES, json=data)
    return response.json()
```

Note using `websafe_encode` to encode many of the fields. The data needs to be encoded as it's in raw bytes. Typically Base64 URL encoded strings are used, but some IDPs may encode values differently. Make sure to identify the correct encoding.

## Piecing it all together

With all the pieces found we can write a complete CLI tool. Here is a demo:

{{< image src="/assets/blog/webauthn-in-the-cli/cli.gif" >}}

And here is the complete code that [I published as a gist](https://gist.github.com/itsjfx/b4612d90c210d0a1c77de606a87311ad)

Most of the code is initialising the FIDO2 WebAuthn client based on the [example code mentioned earlier](https://github.com/Yubico/python-fido2/blob/main/examples/credential.py). The complexity lies in signing the assertion correctly and creating web requests exactly the way webauthn.io expects. This includes cookies and headers. It may take a bit of trial and error to get everything perfect.

{{< gist itsjfx b4612d90c210d0a1c77de606a87311ad >}}

## Handling WSL

The following works great on Linux, Mac, and on native Windows, but does not work on WSL. Windows does not pass-through USB devices to WSL natively.

There are a few solutions however:
1. Install Python + `fido2` package on Windows, call `python.exe` and capture the assertion (e.g. via `stdout`)
2. Follow [this guide by Microsoft](https://learn.microsoft.com/en-us/windows/wsl/connect-usb) to expose the USB device to WSL
    * With enough elbow grease you could automate this process

## Conclusion

I enjoy working with hardware keys and have a keen interest in how FIDO2 works, so this was a really fun post to write. If you don't write a CLI application with WebAuthn, I hope you still learnt something about how it works.

## See also

* [Adam Langley's book "A Tour of WebAuthn"](https://www.imperialviolet.org/tourofwebauthn/tourofwebauthn.html)
    * I wish existed when I first learnt about WebAuthn
* [The WebAuthn spec](https://www.w3.org/TR/webauthn-2)
* <https://en.wikipedia.org/wiki/WebAuthn>
* <https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API>
