---
layout: post
title: Fixing Quest 2 controllers not showing up in MRTK3 app
date: 2023-01-12T00:00:00.0000000+01:00
categories:
- MRTK3
- Quest2
- Unity
tags: []
featuredImageUrl: https://LocalJoost.github.io/assets/2023-01-12-Fixing-Quest-2-controllers-not-showing-up-in-MRTK3-app/nocontrollers.png
comment_issue_id: 436
---
A very quick and small tip. When I deployed my new HoloATC app to the Quest 2, I noticed hand models showed up when I was using hand control, but no controller models were showing up when I used controllers. 

![](/assets/2023-01-12-Fixing-Quest-2-controllers-not-showing-up-in-MRTK3-app/nocontrollers.png)

[Finn Sinclair](https://github.com/Zee2), one of the Microsoft developers working on the [MKTK3](https://github.com/microsoft/MixedRealityToolkit-Unity/tree/mrtk3), told me I needed to import [glTFast](https://github.com/atteneder/glTFast) to make the controllers show up. It seems the MRTK3 has a kind of soft dependency to it. Why it is not installed automatically, is explained in this [MRTK3 GitHub Issue. ](https://github.com/microsoft/MixedRealityToolkit-Unity/issues/11294)

However, after I installed gltFast [using the package installer link on it's GitHub page](https://package-installer.glitch.me/v1/installer/OpenUPM/com.atteneder.gltfast?registry=https%3A%2F%2Fpackage.openupm.com&scope=com.atteneder), I still didn't see the controllers in my app. After peeking in the MRTK3 development project samples, I found there's *another* package by the same author installed. It's called "KTX/Basis Universal Texture". The easiest way to install it, is as follows:

* Open the package manager
* Select "My Registries"

![](/assets/2023-01-12-Fixing-Quest-2-controllers-not-showing-up-in-MRTK3-app/registries.png)

* Select "KTX/Basis Universal Texture"

![](/assets/2023-01-12-Fixing-Quest-2-controllers-not-showing-up-in-MRTK3-app/ktx.png)

* Hit the install button on the bottom right on the Package Manager Windows

And voilà

![](/assets/2023-01-12-Fixing-Quest-2-controllers-not-showing-up-in-MRTK3-app/controllers.png)

Be aware the version of gltFast apparently *must be* 4.8.3 and no other. 

![](/assets/2023-01-12-Fixing-Quest-2-controllers-not-showing-up-in-MRTK3-app/version.png)

By default, the package installer installs 5.0.0 and when I tried that, it didn't work. The controllers did not show up. My app is build using MRTK3 1.0.0-pre.13 so the may change in the future.
