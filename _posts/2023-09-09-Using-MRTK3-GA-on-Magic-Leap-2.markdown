---
layout: post
title: Using MRTK3 GA on Magic Leap 2
date: 2023-09-09T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- GA
- Magic Leap 2
published: true
permalink: 
featuredImageUrl: 
comment_issue_id: 455
---
I hope the good folks at Magic Leap don't get sick and tired of me hacking their SDKs to work with software versions they are not intended for, but I found a way to make MRTK3 GA work with the latest drop, that is, the 1.0.0-pre4 package. The procedure is kind of comparable to when [I got MRKT3 pre-17 to work](https://localjoost.github.io/Using-MRTK3-pre17-on-Magic-Leap-2/).

* First, find the package tar file "com.magicleap.mrtk3-1.0.0-pre.4.tgz" in your Magic Leap packages folder.
* Make a backup of it in case something goes wrong.
* Open the tgz file with [7-Zip](https://www.7-zip.org/).
* Pull out the following files and folders into one folder, so they are all next to each other:
  * Editor
  * Runtime
  * Samples
  * package.json
  
* Open the package.json file in a text editor. Replace all instances of "com.microsoft.mrtk" with "org.mixedrealitytoolkit".
* Open the *parent* folder in which the extracted files and folders reside in Visual Studio Code.
* Do a global search & replace, replacing "Microsoft.MixedReality" with "MixedReality".
* Copy the three folders and package.json back into the tgz file, effectively replacing what was there.

You now have a package that is *mostly* compatible with MRTK3 GA. You can proceed to upgrade your Magic Leap 2 project to MRKT3 and install this package instead of theirs. If you work with the controller, everything works as you would expect. If you use hand tracking, you will need to make a new MRKT3 Profile to get the hand tracking to work at all, [as I described in my previous blog post](https://localjoost.github.io/Upgrading-to-MRTK3-GA-from-a-pre-release-some-assembly-required/). And even if you do that, you will notice close interaction works, but far interaction does not. You can press buttons and menus, but the hand ray does not appear, and ['air taps' using this technique](https://localjoost.github.io/Intercepting-raw-air-taps-with-MRTK3-cross-platform-(Quest-2-and-HoloLens)/) won't work either.

And, of course, instead of following this procedure, you can also [just download my hacked version of pre-4](https://www.schaikweb.net/ML2/com.magicleap.mrtk3-1.0.0-pre.4_ga_hacked_jvs.tgz) ;). Or just wait until Magic Leap issues an official package. I guess that won't be long now.
