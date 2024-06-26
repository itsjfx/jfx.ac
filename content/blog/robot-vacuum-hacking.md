---
layout: post
title: Robot vacuum hacking
date: 2024-06-24
tags: [robot, vacuum, hacking, root]
---

## Important Notice

I'm not writing this to sell the idea of hacking your robot, but to give a lens into robot hacking and the work being done. If you're reading this blog, chances are you might find this interesting, and after doing heavy research may consider to hack your own robot vacuum.

Please do your research and understand the goals of hacking your robot and how to do it before you try.

## Introduction

This is a long post on things I wish I knew about robot vacuums and robot hacking earlier. This is not a replacement for existing documentation or help guides. If you only care about rooting your robot and freeing it for the cloud, and don't care about how the robot works, chances are this post won't be that helpful to you.

Earlier this month, I installed custom firmware and a cloud replacement on a second [Dreame L10s Ultra robot vacuum](https://www.dreametech.com/products/dreamebot-l10s-ultra). Also released this month was root and cloud-replacement support for the new [Dreame X40 Ultra](https://www.dreametech.com/products/dreametech-x40-ultra-robot-vacuum) and [Dreame L10s Pro Ultra Heat](https://robotinfo.dev/detail_dreame.vacuum.r2338a_0.html), so it's certainly still an exciting time to get into robot hacking!

In this post I'll quickly cover about my views on robot vacuums, the robot hardware, the hacking community, the software, and then talk briefly about my experiences. Feel free to skip over my preamble into robot vacuums (or this entire post, haha) if you're not interested.

## Why am I crazy about robot vacuums?

I believe a robot vacuum is a great purchase as you can only vacuum and mop your home so much. Dust builds up very quickly, unless you're regularly vacuuming your home daily you'll find that it'll travel across the house across more surfaces.

Now days robots have mop heads which spin at fast speeds to get your floors clean. They can take in clean water from a tank in a dock which reduces the amount of times you need to fill it up, use detergent, can clean and dry the mop heads, and empty the dirty water into a secondary water tank for you to empty.

The mop means my hardwood floors are shiny which makes me happy. I find mopping tiring so it helps relieve some physical hardship.

I also got a robot for my elderly parents which helps reduce physical effort needed by them to maintain a tidy house. It's a great piece of **luxury hardware** if you can justify spending the money on them.

## Why am I also unhappy with robot vacuums?

One of the main factors is privacy. As a consumer you place a lot of trust on the vendor. You don't have any control of the software running on the robot. It requires internet access, has cameras, sensors, and local AI models equipped.

These aren't crazy concerns to have. If you watch [Dennis Giese & braelynn's talk at 37C3](https://www.youtube.com/watch?v=56N1dYfdVf4), they discuss with a specific vendor some user data is publicly accessible, remains on cloud servers past account deletion, and remains on the robot past factory reset. The same vendors robots software avoids TLS checks as they have shell scripts with  `wget --no-check-certificate` in the firmware. Additionally they showcased an exploit where the live video feed for a robot is accessible under certain conditions as a pin check is enforced client side in the application.

Other than privacy, you may not want a robot due to high cost, messy or uneven flooring, or carpet, which in recent years is less of an issue due to detachable mop vacuums.

## Robot hardware breakdown

Below is a hardware breakdown diagram from the same talk mentioned earlier.

![Hardware Architecture](/assets/blog/robot-vacuum-hacking/hardware-architecture.png)

### The SoC

The system on a chip (SoC) is the application processor which handles navigation, mapping and networking. It typically runs Linux. Custom vendor software runs on the robot as a daemon and does most of the heavy lifting handling cloud communication, OTA, robot control, mapping, and other user facing functions.

Some interesting things I found:
* Some vendors have a lot of shell scripts with poor bash/shell scripting practices which get called on the SoC. These scripts may interact with the OS, handle WiFi (e.g. interact with `wpa_supplicant`), or interact with the custom vendor software.
* As the camera is part of the SoC, on some devices it can be accessed as a video device on `/dev/videoX` .
* Newer robots can be seen having 4 core SoCs, 1-2GB of RAM, 4-8GB of flash storage, and getting more powerful overtime.
* My robot, the Dreame L10s Ultra runs Athena Linux ARM64 on the SoC
    * `Linux r2228_release 4.9.191 #1 SMP PREEMPT Wed Sep 13 15:19:31 CST 2023 aarch64 GNU/Linux`

### The MCU

The micro-controller unit (MCU) handles real-time operations such as motors and sensors. This has it's own firmware separate to the SoC's firmware/OS. [Information on the MCU firmware for the Dreame Z10 is available here](https://github.com/alufers/dreame_mcu_protocol/blob/master/dreame_z10_notes.md) if you'd like to learn about what lives in the MCU for a robot.

MCU interest is not as popular. It requires a different set of skills. There's a lot of secret sauce baked into the MCU, so vendors make it difficult for people to understand what's happening. Most people are interested in the SoC and the cloud integration. I'd suggest only learning more if you're interested in hardware or MCUs.

### Finding hardware information

Dennis Giese has done research on a large number of robots and [published information here](https://robotinfo.dev). Information on the hardware used as well as software & firmware versions, links to rooting methods and custom firmware downloads are available for many robots.

There are sometimes breakdowns of the robot boards themselves. It's an extremely valuable resource to understand what builds up a robot.

## The robot hacking community

The community seems to be broken down to the following groups of people:
* People interested in removing the cloud from their robot
    * This is the average hacker and makes a large portion of the community
    * Mixed level of skills
    * Many sadly incapable of reading documentation/FAQs or following instructions and complain on Telegram or GitHub
    * It's disappointing to see in the chats as it makes approaching the community as an outsider more difficult as such
* People interested in the hardware
* People interested in the firmware/software on the SoC
    * Gaining root
    * Building custom cloud implementations
    * Research
* People interested in the firmware on the MCU

There are many people who contribute greatly in the community, but it boils down to these two people (who are the most public):
* [Dennis Giese](https://dontvacuum.me/), a PhD student at [Northeastern University](https://www.neu.edu)
    * Dennis maintains [Dustbuilder](https://dustbuilder.dontvacuum.me/), a website which allows you to create custom rooted firmware for robot vacuums
    * Dennis also maintains the [Robot Info website](https://robotinfo.dev/)  (mentioned earlier)
    * Dennis and others discover root methods, gain persistence, and verify vendor claims
* [Sören Beye (known as Hypfer)](https://github.com/Hypfer) who is the maintainer of [Valetudo](https://valetudo.cloud), a cloud replacement software for robot vacuums
    * Hypfer is very vocal about his views and vocalising the strug of maintaining a popular open-source project and providing support. See the "On not burning out" section in this release: <https://github.com/Hypfer/Valetudo/releases/tag/2024.06.2>. It's a good read. Don't be like these people if you visit the Telegram chat.
    * FWIW: I found that the [Valetudo/rooting how-to guides](https://valetudo.cloud/pages/general/getting-started.html) and surrounding documentation are simple enough to follow and answer enough questions that you _shouldn't need_ additional support in Telegram.

## Dustbuilder

Once a root method is discovered and published, a [Dustbuilder](https://dustbuilder.dontvacuum.me/) page for a robot is generally created sometime later.

Dustbuilder creates a patched firmware to run on your SoC, and gives you an MCU firmware image. It saves you from dumping and patching the firmware yourself (which may not be possible), and saves you from doing it incorrectly.

It has several patches to allow you to gain persistence on the robot, and disable cloud connectivity, calling home features like telemetry, video recordings. It allows you to use Valetudo, and much more. You can see the diff from the original firmware image which is really helpful in understanding what it does.

If a vendor updates the firmware, it's likely the Dustbuilder firmware will also be updated (once determined stable). This means you can continue to receive vendor updates to the MCU/SoC even when rooting.

## Valetudo

### What is it? (tech wise)

As a preface, I'd suggest reading the [Valetudo Newcomer Guide](https://valetudo.cloud/pages/general/newcomer-guide.html) and come back here. I'm going to write mostly about my observations about tech itself rather than what the software offers and it's goals. Please read the official documentation to understand the project.

As mentioned, Valetudo is a cloud replacement for robot vacuums. It has a frontend written in React + TypeScript, and a backend written in Node.js.

Valetudo is an interesting piece of software as it will run an API, serve a frontend, act as an MQTT client, and implement the robots cloud all as a single binary on your robot.

It's not a firmware replacement unlike other articles on the internet mention. Instead it sits alongside existing firmware on the robot, typically a patched firmware from Dustbuilder. As it does not replace the firmware, _you shouldn't_ lose functionality of the original robot.

Before installing Valetudo I was expecting that I'd have to host it on another machine as it's a cloud replacement, but this is not the case. It has no dependencies on other infrastructure like a server, except for NTP, and has no dependency on a mobile app interface like stock firmware robot vacuums.

Cause of it's architecture, Valetudo won't face any issues due to infrastructure being down or having a conflicting version. It will simply operate as expected as long as it can continue to run on the robot. If you have multiple robots, then they will each run their own instance of Valetudo.

This has the con that the robot has to be powerful enough to run Valetudo as well as its original firmware, however the maintainer has taken a lot of efforts to ensure Valetudo runs with a low hardware footprint.

Valetudo [officially supports 35 different robots](https://valetudo.cloud/pages/general/supported-robots.html) across different vendors. Officially supported means each robot has been tested by Hypfer, with the root method documented for each robot. It's insane that a single piece of software not only works with so many robots, but that the developer has also tested it themselves and continue to hold the software to this standard.

Valetudo has well documented releases every few months with breaking changes documented, and can also update itself in the UI. Newer versions may have UI updates, bug fixes, but more importantly support for newer robots.

### Using it

To use Valetudo, you simply go to it's web UI, which is accessible via your robots private IP, or by port forwarding to the web server on the machine via SSH. The web UI offers you control of your robot like you would in a native app.

Of course it can integrate with systems like Home Assistant via it's MQTT client, which will then allow you to control it via Home Assistant with various integrations. At the moment, [I use this card](https://github.com/PiotrMachowski/lovelace-xiaomi-vacuum-map-card) to control my robot in Home Assistant.

### Why/why not

I'm not going to write about why or why not to use Valetudo. If this got your interest, read the below pages, and the surrounding documentation on the Valetudo website:
* [Why Valetudo](https://valetudo.cloud/pages/general/why-valetudo.html)
* [Why not Valetudo](https://valetudo.cloud/pages/general/why-not-valetudo.html)

## My rooting journey

I followed the Dreame fastboot instructions for both my Dreame L10s Ultra's. The robot is slowly approaching end of life, so I was lucky to get it on sale. I couldn't easily obtain the Valetudo rooting PCB board, so I used some dupont cables and an old USB mouse with a USB 2.0 cable (rip mouse). It worked OK minus some USB 2.0 shenanigans. If I could do it again, I would put the effort in to obtain the PCB board, as I probably wasted 2 hours getting the USB connection to work correctly.

I previously had a Roborock S6 MaxV which I couldn't root, and faced many issues getting the Home Assistant integration working. Occasionally the vendor would rate limit my Home Assistant integration or change their API, meaning I'd have to wait for the maintainer of the integration to address the issue. I've not faced any issues with my L10s Ultra and Valetudo due to its native MQTT client and Home Assistant support. The robot has not skipped a beat and is far more responsive than my previous vendor cloud locked Roborock. I also have no internet calls coming out of my robot (using a NTP server on my LAN), so I'm super pleased with that.

## What's upcoming in the community?

In Dennis Giese's recent talk he covered Ecovacs vacuums. The root method for many models is now public, however there is no Valetudo support for them right now. Please do not ask the maintainers when they will supported. We can only hope there'll be Valetudo support in the future or a cloud replacement for these robots.

## See also

The resources below helped me understand the things I wrote about:
* Dennis Giese's talks (these are a must watch):
    * [DEF CON 29- Dennis Giese - Robots with lasers and cameras but no security Liberating your vacuum](https://www.youtube.com/watch?v=EWqFxQpRbv8)
    * [DEF CON 31 - Vacuum Robot Security & Privacy Prevent yr Robot from Sucking Your Data - Dennis Giese](https://www.youtube.com/watch?v=AfMfYOUYZvc)
    * [37C3 - Sucking dust and cutting grass: reversing robots and bypassing security](https://www.youtube.com/watch?v=56N1dYfdVf4)
* [Valetudo's website and documentation](https://valetudo.cloud)
* [Dennis Giese's website](https://dontvacuum.me)
    * [robotinfo.dev](https://robotinfo.dev)
    * [Dustbuilder](https://builder.dontvacuum.me)
* [alufers/dreame_mcu_protocol](https://github.com/alufers/dreame_mcu_protocol/tree/master)
