---
layout: post
title: Getting Eye Tracking to work with OpenXR on HoloLens 2
date: 2021-06-30T00:00:00.0000000+02:00
categories: []
tags:
- HoloLens2
- MRKT2
- Unity3d
- Windows Mixed Reality
- QRCode
- OpenXR
featuredImageUrl: https://LocalJoost.github.io/assets/2021-06-30-Getting-Eye-Tracking-to-work-with-OpenXR-on-HoloLens-2/PlayerSettings.png
comment_issue_id: 383
---
You have moved your Unity project to OpenXR following [Microsoft's Guidelines](https://docs.microsoft.com/en-us/windows/mixed-reality/mrtk-unity/configuration/getting-started-with-mrtk-and-xrsdk?view=mrtkunity-2021-05#configuring-mrtk-for-the-xr-sdk-pipeline) and/[or my blog post](https://localjoost.github.io/Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenXR-plugin/) and upgraded to Unity 2020.3.x. Maybe there were some hiccups along the way, but finally you got deploy your OpenXR powered app to your HoloLens 2 - and you found out Eye Tracking is not working. In fact, you don't even get the consent popup.

## You have checked all the obvious things

The capabilities are set in the Unity Player Settings

![](/assets/2021-06-30-Getting-Eye-Tracking-to-work-with-OpenXR-on-HoloLens-2/PlayerSettings.png)

And to be sure, you have checked these translated neatly into the resulting C++ solution's app manifest:

![](/assets/2021-06-30-Getting-Eye-Tracking-to-work-with-OpenXR-on-HoloLens-2/appmanifest.png)

You have set up the Eye Gaze Provider

![](/assets/2021-06-30-Getting-Eye-Tracking-to-work-with-OpenXR-on-HoloLens-2/XREyeGazeProvider.png)

You have set up a pointer profile to support eye tracking

![](/assets/2021-06-30-Getting-Eye-Tracking-to-work-with-OpenXR-on-HoloLens-2/Pointerprofile.png)

You even made sure there's an input simulation service set up, and you have set the Default Eye Simulation Mode to "Mouse" so you can easily control the gaze cursor in the editor and it all freaking works there

![](/assets/2021-06-30-Getting-Eye-Tracking-to-work-with-OpenXR-on-HoloLens-2/inputsimulationprofile.png)

But not in HoloLens 2. You don't even get a consent popup and if you check in code using the capability tracker, it says eye tracking is not supported.

## Welcome to OpenXR Interaction Profiles - and the solution

I have no idea why I looked here - there's certainly nothing to find about it, but the solution is in the OpenXR 'interaction profiles', whatever they might be. This is what it said in my project:

![](/assets/2021-06-30-Getting-Eye-Tracking-to-work-with-OpenXR-on-HoloLens-2/NoEyeGazing.png)

Then I clicked this pesky little plus sign on the right bottom of the "Interaction Profiles" box and noticed there was an "Eye Gaze Interaction Profile" entry.

![](/assets/2021-06-30-Getting-Eye-Tracking-to-work-with-OpenXR-on-HoloLens-2/EyeGazingAdd.png)

I selected it, and the result was this:

![](/assets/2021-06-30-Getting-Eye-Tracking-to-work-with-OpenXR-on-HoloLens-2/EyeGazingAdded.png)

And lo and behold: I built and deployed the app, I got the eye tracker consent popup and my app responded to the events like it did before the upgrade. That one entry was all it took. Or what was missing. Literally the only thing this action results in, is flipping one zero to a one in a file called Open "XR Package Settings.asset" in Assets\XR\Settings

![](/assets/2021-06-30-Getting-Eye-Tracking-to-work-with-OpenXR-on-HoloLens-2/packagesettings.png)

## Concluding words

Can you write a blog post about essentially two mouse clicks? Turns out, you can. I have no idea why this apparently is no-where to be found on the internet, or why this profile was not added automatically during upgrade. Maybe it's a fluke, because my in my [upgrade of the QR code reading](https://localjoost.github.io/Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenXR-plugin/) the Eye Gazing Interaction Profile *is* selected (although that app does not use eye tracking). However it took me long enough to find, so I decided to document it - so you know what to check when eye tracking goes south with OpenXR.
