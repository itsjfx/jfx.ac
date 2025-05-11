---
layout: post
title: My KVM is better than yours
date: 2025-04-18
tags: [github, engineering]
---

Damn right, it's better than yours.

## Introduction

My KVM (Keyboard, video, mouse) switch setup lets me use my home desktop and work laptop with ease.

Without giving away _too much_ of the implementation (don't worry, I'll tell you in a bit), the key features:
* USB 3.0 device support
* Can control with an API :)
* Controllable via Stream Deck or keyboard bindings
* Supports HDMI/DisplayPort to any supported resolution and refresh rate of the monitor
    * My 240hz 1080p monitor works great
* No display disconnecting
    * On Windows, if you disconnect a display, windows get moved to other monitors. This is incredibly annoying and does not happen on my setup
* Relatively quick
* Cheap
    * My end costs were $40 AUD

Things I wanted to avoid:
* Software-based KVM solutions - my home and work machines should not be aware of each other. I don't feel comfortable sending keystrokes across machines.
* An external monitor switcher hub device
    * Mainly due to the lack of 240hz support and display disconnecting
* Spending lots of money
    * How do I justify the need to spend $300 on a high quality KVM to my company? or myself?
* Slowness

Here's how I did it

## Handling USB

To switch my USB devices, I got a boring USB 3.0 hub which can switch between two machines from Amazon. It took a day to arrive, and costs about ~$30 AUD on sale. 

I bought the popular [UGREEN USB 3.0 Switch Selector](https://www.amazon.com.au/UGREEN-Computers-Peripheral-Switcher-One-Button/dp/B01N6GD9JO). I've tried about 3 other ones and none were as good as this for the price, performance, and USB 3.0 support.

The USB hub comes with a toggle button on the top and no API. This worked well for a while, but I got tired of leaning across the table to press the button. I wanted it to have an API so I could switch it via my [Stream Deck](https://www.elgato.com/ww/en/s/welcome-to-stream-deck).

I bought a second USB hub, pulled it apart, and saw the electronics are simple. There's a tactile switch which grounds a circuit momentarily to "toggle" which machine the device is serving USB to. Playing around with a multi-meter I was able to toggle the connected machine easily.

{{< image src="/assets/blog/kvm-better/disassembled.png" >}}

I had a Raspberry Pico lying around ($10 AUD) so I soldered a cable to the PCB in the spot to ground. I then wrote a simple program to quickly ground a pin on the Pico and connected the soldered cable to the Pico.

To my amazement the program actually worked:

{{< image src="/assets/blog/kvm-better/trial.gif" >}}

Next I wrote a dirty HTTP server, connected it to my WiFi, and I was able to do the same via `curl`. For $40 (Pico + USB hub), I had a USB 3.0 hub I could control with an API.

## Handling Displays

I'd tried an external device for switching between displays, but due to my monitors high refresh rate and Windows freaking out when displays were disconnected I found it not suitable. Luckily, I discovered that it's possible to control displays with software.


There's a not well-known protocol called [Display Data Channel (DDC)](https://en.wikipedia.org/wiki/Display_Data_Channel) developed by the Video Electronics Standards Association (VESA). DDC is essentially a bus that allows for digital communication between a computer display and video card.

Separately, there's another protocol called [Monitor Control Command Set (MCCS)](https://en.wikipedia.org/wiki/Monitor_Control_Command_Set) which allows for controlling the properties of a monitor. Using DDC as a data channel, you can send MCCS messages bidirectionally between a computer and monitor.

With MCCS, you can do things like adjust the brightness or contrast of your monitor - but even more useful, you can set the current input of the monitor! The same as pressing the input button on your monitor yourself.

There's a ton of software based implementations of MCCS via DDC on the internet. I decided to use [monitorcontrol](https://github.com/newAM/monitorcontrol) cause it was written in Python and worked across many platforms.

With a simple shell command, I could set my monitor to HDMI for my work laptop: `monitorcontrol --monitor=2 --set-input-source HDMI1`

The best part is this is completely free, so you do not need to pay for another device. You only need cables for each display and device.

## Integrating USB and Display

I wanted to add custom buttons to my Stream Deck which allowed switching between my devices. To do this, I use this [third-party Stream Deck Python library](https://github.com/abcminiuser/python-elgato-streamdeck) as Elgato's Stream Deck software doesn't work on Linux (shakes fist).

I added two buttons on my StreamDeck, one for switching to my laptop, and one for switching to my desktop. Both buttons effectively:
* Send a web request to my Raspberry PI Pico to toggle the switch
* Run `monitorcontrol` to set the device appropriately

I also created a third button that toggles only the USB device.

The reason why I wanted a button for each machine and not a toggle button - is cause displays get disconnected due to device sleep or cables slipping, so I had the need to explicitly select which device to switch to.

Unfortunately, I had no way of keeping track of which USB device my KVM connected to. This is cause the Pico has no state and grounds the pin whenever it receives a web request. I thought about tapping into the light on my KVM and using that for state management, then I realised there's a simpler solution.

My Stream Deck is always directly connected to my desktop (not via KVM), and my desktop machine is always on. Due to this I could cheat a little.

When I press the button on my Stream Deck, my script runs `lsusb` and checks whether the USB hub is listed as a connected device. If I press the button to switch to my desktop, and the USB hub is connected, then there's no need to call my Pico API to toggle the USB hub as the USB hub is connected to the correct device. If the USB hub did not appear in `lsusb`, then this would mean my USB hub was connected to my laptop, so my Stream Deck code correctly makes a web request to my Pico API to toggle the USB hub.

Cause of this cheat, I had a way to reliably detect whether the USB device was connected to the right PC or not, and only switch when appropriate. I combined this with `monitorcontrol`, and with a single button press I have a way to switch my display and USB devices.

{{< image src="/assets/blog/kvm-better/final.gif" >}}

## A simpler alternative

I reflected on my KVM setup and I realised that there is a fairly simple alternative which does not require soldering or taking apart the UGREEN KVM.

Using [udev rules](https://en.wikipedia.org/wiki/Udev), you can detect when a device is connected or disconnected from your Linux machine and run a shell command. You can create these rules on a single "main" machine - and listen to events on the USB hub device. When disconnected, you could run `monitorcontrol` to switch the display input to your other machine - and when connected switch back to the main machines display input.

For the UGREEN KVM, here are two rules which implement the above:

```bash
ACTION=="add", SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ENV{PRODUCT}=="5e3/610/663", RUN+="/bin/su -c '/home/jfx/.local/bin/monitorcontrol --monitor=2 --set-input-source DP1' jfx"

ACTION=="remove", SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ENV{PRODUCT}=="5e3/610/663", RUN+="/bin/su -c '/home/jfx/.local/bin/monitorcontrol --monitor=2 --set-input-source HDMI1' jfx"
```

This is what works for me as I have `monitorcontrol` installed in my local users Python installation.

This setup requires you to press the button on the KVM, but saves you from needing to switch your monitor input yourself. It also saves you from soldering!

This will also only work on Linux due to `udev`, but I'm sure there's a similar implementation for other operating systems.

## Conclusion

A big factor of working from home is being comfortable, so it pays to have a setup to switch between your devices. You don't need to perform brain surgery on your USB hub to be happy. If your company gives you an allowance for equipment, definitely try out a switchable USB hub with a button. I highly recommend the UGREEN one.

There is no financial cost of using free software like `monitorcontrol` - the only cost is your time. Try it out on your display and perhaps create a key-binding or two to switch between your display inputs.

I also enjoyed the opportunity to learn more about electronics and use a Pico to solve a problem. I'm not the strongest electronics person so this was a great learning project. Thank you to all the friends who helped out.

## The code

The code for my Stream Deck is available here: <https://github.com/itsjfx/dotfiles/blob/24a6e418244909d059a4d057645af0bb1f6996bc/lib/streamdeck.py#L144-L156>

The code for my Pico API is available in this Gist: <https://gist.github.com/itsjfx/647cb9cb142cb4eb3233214ec6a50229>

{{< gist itsjfx 647cb9cb142cb4eb3233214ec6a50229 >}}
