---
layout: post
title: Deploying on Spectacles from Windows using an USB cable
date: 2025-08-23T00:00:00.0000000+02:00
categories: []
tags:
- Lens Studio
- Specacles
- Windows
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2025-08-23-Deploying-on-Spectacles-from-Windows-using-an-USB-cable/adbtools.png
comment_issue_id: 491
---
Apparently I am a bit of an oddball in the Snap Spectacles community, as apparently everyone else - including the Snap folks themselves - uses Macs for development. Although I do have a Mac Mini, it's only being used when I'm working with Apple devices (specifically, Vision Pro). For the rest, I use Windows. I grew up on Windows, used it for over three decades, know it inside and out, it runs like the wind on this machine, and I have a great toolset I'd dearly miss going elsewhere - it's just my thing.

However, this posed a few problems when I was trying to deploy on Spectacles from Windows using the relatively new "wired connection" option. It became pretty clear no one at Snap used it much - few probably even tried - because it caused some headaches trying to get it to work. Still, this is where Snap's dedication to their product shows. After posting my woes on the [Spectacles Reddit](https://www.reddit.com/r/Spectacles/), I was contacted by one of their engineers, [Pasha Antonenko](https://www.linkedin.com/in/pavelantonenko/), and we had a well-over-an-hour-long video call (at a rather ungodly early hour for Pasha) in which we established the baseline for successfully connecting and deploying a Spectacles app to a Spectacles using a USB cable from Windows.

In short - the list of requirements is this:

1. Whenever you connect a Spectacles to a Windows machine, you get the "USB device malfunctioned" error. Always. However confusing that is, you can actually *ignore* that. You might first see a message that it set up a "composite USB device" first.  
2. You *must* have the newest version of the [ABD tools](https://developer.android.com/tools/releases/platform-tools) and have that set up in Lens Studio to get a stable connection between Lens Studio and the Spectacles. Do not -  I repeat, *do not* - be tempted to re-use an ADB tools install from some Unity installation, as this developer initially tried because it seemed the logical thing to do.  
3. Double-check that wired connection is turned on in the settings. I was sure they were, but they were not. Maybe a result of the recent upgrade on my phone. I use Android, and I guess the Snap folks are primarily on iOS too 😉.  
4. You can only connect to a *native USB-C port*, so you need a cable with USB-C plugs on both sides. A converter cable with a USB-A plug on one side will not work properly, regardless of how fast the port or cable are rated.

Ad 2: do not forget to actually point to the place where you unpacked the ADB tools in preferences (Lens Studio / Preferences):

![adbtools](/assets/2025-08-23-Deploying-on-Spectacles-from-Windows-using-an-USB-cable/adbtools.png)

Ad 3: you need to set this in the Spectacles app on your phone:

![app](/assets/2025-08-23-Deploying-on-Spectacles-from-Windows-using-an-USB-cable/app.png)

Only when you see this in your Logger is the Spectacles successfully connected to your Windows computer, and the USB cable can be used for deployment.

![connected](/assets/2025-08-23-Deploying-on-Spectacles-from-Windows-using-an-USB-cable/connected.png)

No demo project this time, because there is no code to share.