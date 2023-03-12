---
layout: post
title: Passthrough transparency with MRTK2 and 3 on Quest 2/Pro
date: 2023-03-11T00:00:00.0000000+01:00
categories: []
tags:
- MRKT3
- Quest2
- HoloLens
featuredImageUrl: https://LocalJoost.github.io/assets/2023-03-11-Passthrough-transparency-with-MRTK2-and-3-on-Quest-2Pro/QuestSeeThrough.gif
comment_issue_id: 440
---
Recently I had an informal talk with a fellow Mixed Reality developer who was convinced was it was not possible to use Quest passtrough transparency with MRTK. However, that is perfectly possible - as you can see below, on my Quest 2

![](/assets/2023-03-11-Passthrough-transparency-with-MRTK2-and-3-on-Quest-2Pro/QuestSeeThrough.gif)

I was a bit surpised, I thought this was common knowlegde after [Davide Zordan](https://twitter.com/DavideZordan) [tweeted about it](https://twitter.com/DavideZordan/status/1586315761227661313) in October 2022 and although that was not the complete story, there was very little missing. However, I decided to write a little blog post about it, if only because that's easier for myself to find back someone else's tweet from 5 months ago.

## Set up an MRTK project

First, set up an MRTK project normally. I used MRTK3, but 2 would work just as fine. I added a little floating cube (kept in view by an MRTK RadialView and a Solver) to it to have something to look at. Check if it works on the Quest. Make sure the camera background is solid black (so no skybox)

## Import Oculus integration package

Import [Oculus integration package](https://assetstore.unity.com/packages/tools/integration/oculus-integration-82022) from the asset store by clicking "Open in Unity":

![](/assets/2023-03-11-Passthrough-transparency-with-MRTK2-and-3-on-Quest-2Pro/assetstore.png)

That will open the package managager

![](/assets/2023-03-11-Passthrough-transparency-with-MRTK2-and-3-on-Quest-2Pro/packagemanager.png)

Click Import. You probably don't need to import everything, but I have not taken the trouble yet to find out the minimal import.

After the import, it asks you a couple of questions:
* Update the OVR plugin? Choose upgrade 
* Choose OpenXR? Yep
* Upgrade the spatializer? Indeed
* Restart the editor? Yes
* After the restart, it will ask you to enable OculusXR Feature. Choose upgrade

## Add passtrough to your scene

I usually make an empty gameobject named "Oculus PassThrough" and add these scripts to it:
* OVR Manager
* OVR Passthrough Layer

In the OVR manager, you will need to scroll all the way to the bottom and make these two settings:

![](/assets/2023-03-11-Passthrough-transparency-with-MRTK2-and-3-on-Quest-2Pro/ovrmanager.png)

And on the OVR PassTrough Layer, set the Opacity to 0.3

![](/assets/2023-03-11-Passthrough-transparency-with-MRTK2-and-3-on-Quest-2Pro/passtroughmgr.png)

Make sure the Placement is *Overlay*, otherwise it won't work. Now this may look odd, because you would like to have the transparent background as, well, background, so you would be inclined to set it to Underlay. Funny thing is, then you will simply get a black background. It probably conflicts with the way the MRTK camera works. However, by using a partly transparent *overlay* on top, you will get a very HoloLens-like effect: the reality a bit darker than with passthrough on the Quest home itself, and the virtual objects are just that little bit transparent - but that enables you to navigate through reality with confidence while still being able to the virtual objects. The key difference is reality look *really* cr*p compared to HoloLens - grainy and in black and white - but for situations where this is not important, this solution might just be 'good enough'. Choose the transparency value that suits your use case. I found 0.3 to be pretty good for most cases.

The amusing thing is: even after adding all the Oculus stuff, *the app still works on HoloLens*. I usually make a script that disables the "Oculus PassThrough" gameobject when the app is on a HoloLens, to make sure it does not interfere or uses resources unnecessarily.

A little [sample project can be found here](https://github.com/LocalJoost/MRTKQuestSeeThrough). The branch [baseversion](https://github.com/LocalJoost/MRTKQuestSeeThrough/tree/baseversion) will shows you the base project without the Quest integration package and configuration.