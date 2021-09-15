---
layout: post
title: Correct setup of OpenXR in a HoloLens 2 project including Holographic Remoting
date: 2021-09-15T00:00:00.0000000+02:00
categories: []
tags:
- HoloLens2
- MRKT2
- Unity3d
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2021-09-15-Correct-setup-of-OpenXR-in-a-HoloLens-2-project-including-Holographic-Remoting/windowpluginmgt.png
comment_issue_id: 389
---
This is a short tip, and an easy checklist reference (not only for you, but also for myself). Although [Microsoft does have a guide for this](https://docs.microsoft.com/en-us/windows/mixed-reality/develop/unity/unity-play-mode?tabs=openxr), it's missing some screenshots (IMHO) and it took me more time to get it right than I think it should. This is not so much a problem when you set up a new project, but if you are migrating, things can be a bit more rough. So anyway, I have a correctly set up project [here](https://github.com/LocalJoost/HandCollider/tree/openxrremotingsetup) that you can check for settings. I am not sure these are all required, but this is what works for *me*.

## Unity project settings
Using Unity 2020.3.6f1 or some other 2020.3 LTS version, click Edit/Project Settings and select "XR Plug-in Management". 
The desktop tab (with the monitor on it) should say this:  

![](/assets/2021-09-15-Correct-setup-of-OpenXR-in-a-HoloLens-2-project-including-Holographic-Remoting/windowpluginmgt.png)

The UWP tab (with the Windows flag on it) should show this:

![](/assets/2021-09-15-Correct-setup-of-OpenXR-in-a-HoloLens-2-project-including-Holographic-Remoting/uwppluginmgt.png)

Now select "OpenXR" below "XR Plug-in Management". The desktop tab should show this:

![](/assets/2021-09-15-Correct-setup-of-OpenXR-in-a-HoloLens-2-project-including-Holographic-Remoting/windowopenxr.png)

And the UWP tab should say this:

![](/assets/2021-09-15-Correct-setup-of-OpenXR-in-a-HoloLens-2-project-including-Holographic-Remoting/uwpopenxr.png)

Now if you click Mixed Reality/Remoting/Holographic Remoting for Play Mode:

![](/assets/2021-09-15-Correct-setup-of-OpenXR-in-a-HoloLens-2-project-including-Holographic-Remoting/click.png)

You should get this little screen: 

![](/assets/2021-09-15-Correct-setup-of-OpenXR-in-a-HoloLens-2-project-including-Holographic-Remoting/remoting.png)

Fill in the IP address your HoloLens 2 is listening on, hit enable and you are good to go. 

*Don't forget to disable it again if you want to test in the editor again!*

## Bonus tip: easily determine the ip address of your HoloLens 2 USB port
The [Holographic Remoting Player App](https://www.microsoft.com/en-us/p/holographic-remoting-player/9nblggh4sv40) conveniently shows you the IP address the HoloLens 2 is using on your network.  

![](/assets/2021-09-15-Correct-setup-of-OpenXR-in-a-HoloLens-2-project-including-Holographic-Remoting/ipwifi.png)

In this case, 192.168.1.21. But this might change the next time you boot up, there might be port blocks on your (corporate) firewall, and in any case - it will probably be very slow and result in a poor performance. It's better to use the IP address of the USB port. Now you can of course look that up in the network settings, but I am a lazy ******* so I usually do this:

* Make sure the Holographic Remoting Player app is not running. If there's a 3D icon around, tap the X on the menu near it.
* Turn off WiFi

![](/assets/2021-09-15-Correct-setup-of-OpenXR-in-a-HoloLens-2-project-including-Holographic-Remoting/wifioff.png)

* Make sure the HoloLens 2 is connected to your PC using the USB cable
* Start up the Holographic Remoting Player app again

![](/assets/2021-09-15-Correct-setup-of-OpenXR-in-a-HoloLens-2-project-including-Holographic-Remoting/ipusb.png)

Tadaaa!

A demo project doing noting much but with all the right settings [can be found here](https://github.com/LocalJoost/HandCollider/tree/openxrremotingsetup).
