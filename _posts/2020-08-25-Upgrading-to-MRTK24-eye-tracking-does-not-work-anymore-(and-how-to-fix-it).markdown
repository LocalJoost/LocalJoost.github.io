---
layout: post
title: Upgrading to MRTK2.4-eye tracking does not work anymore (and how to fix it)
date: 2020-08-25T00:00:00.0000000+02:00
categories: []
tags:
- HoloLens2
- MRTK2
- Unity3d
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2020-08-25-Upgrading-to-MRTK24-eye-tracking-does-not-work-anymore-(and-how-to-fix-it)/EyeGazeProvider.png
comment_issue_id: 357
---
Is it possible to write a blog post about one single check box? Gather ye children, around the fire, and I will tell. For an app for my employer I am employing the awesome HoloLens 2 eye tracking. So I implemented it the way I was used to. And I found it did not work. Nothing at all. So to check if I was not not crazy, I took my [Eye Tracing Demo ](https://localjoost.github.io/migrating-to-mrtk2-setting-up-and/) that I wrote in September 2019 - still worked. Until I updated the MRTK (which was still 2.1) to 2.4

## Setting up - how it used to be (and mostly still is)

First, make sure the Eye Gaze Provider is set up in the Input Data Providers

![](/assets/2020-08-25-Upgrading-to-MRTK24-eye-tracking-does-not-work-anymore-(and-how-to-fix-it)/EyeGazeProvider.png)

And for good measure, I always set up set the Eye Simulation under Input Simulation Service.  

![](/assets/2020-08-25-Upgrading-to-MRTK24-eye-tracking-does-not-work-anymore-(and-how-to-fix-it)/InputSimulation.png). 
  
That is also under Input Data Providers

Then, simply set up an Eye Tracking Target

![](/assets/2020-08-25-Upgrading-to-MRTK24-eye-tracking-does-not-work-anymore-(and-how-to-fix-it)/EyeTrackingTarget.png)

And when I run it, nothing happened. I can look at the blue orb all I want, never once the eye target events are fired. WT....H?

## And the culprit is....
After quite some time hunting in the MRTK configuration, I spotted this little checkbox.

![](/assets/2020-08-25-Upgrading-to-MRTK24-eye-tracking-does-not-work-anymore-(and-how-to-fix-it)/Pointers.png)  

Yeah. Under... pointers. So you can define Eye Gazing Providers and capabilities all you want, but this little checkbox is the final yay or nay. So we have to clone the *pointer* profile, check the checkbox and then it works again.

![](/assets/2020-08-25-Upgrading-to-MRTK24-eye-tracking-does-not-work-anymore-(and-how-to-fix-it)/TargetHit.png).

## Conclusion
It may feel like a bit of overkill to write a blog post about one single check box but it took me considerably more time to track this down than to actually write this. And as always: my main goal is to spare some poor sod the agony of having to dig for this on the edge of some deadline - one of those poor sods potentially being future me ;).

I have been able to trace the addition of this option to [this commit](https://github.com/microsoft/MixedRealityToolkit-Unity/commit/5ed0777d786f66e8c93ce9c6581e962b4a116ad8#diff-9cea7f902d081572e00edcb6f9c09e6e) from March 20, 2020. That is indeed after I made my Eye Tracking demo. The property was first called `UseEyeTracking`, later renamed to `IsEyeTrackingEnabled`. Further research learned it was not present yet in MRKT2.3, so it is a 2.4 addition. 

I assume it's *somehere* in the release documents, but it would have been nice to have a kind of warning. I also wonder why the choice was made to put this setting in the *pointer*. I would expect it to be in the Eye Tracking Profile.

I have put the upgraded code in the same GitHub repo as before, [in branch mrtk24upgrade](https://github.com/LocalJoost/EyeTrackingDemo/tree/mrtk24upgrade).