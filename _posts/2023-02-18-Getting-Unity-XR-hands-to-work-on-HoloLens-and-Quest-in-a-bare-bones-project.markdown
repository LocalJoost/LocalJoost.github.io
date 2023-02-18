---
layout: post
title: Getting Unity XR hands to work on HoloLens and Quest in a bare bones project
date: 2023-02-18T00:00:00.0000000+01:00
categories: []
tags:
- Unity
- Mixed Reality
- XRHands
- HoloLens2
- Quest
featuredImageUrl: https://LocalJoost.github.io/assets/2023-02-13-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/handtracking.gif
comment_issue_id: 438
---
Recently I saw someone (I think on LinkedIn) post a screenshot about [Unity XR Hands](https://docs.unity3d.com/Packages/com.unity.xr.hands@1.0/manual/index.html) - this basically claims hand tracking via a standard Unity API, using a plugin architecture. There is very little information in Unity documentation about how to get it to work (\*cough\* nothing new *there* \*cough\*), and it's all in preview. I wondered how hard it would be to set it up in a bare bones project (so without MRTK or any other toolkit) and having it to work on HoloLens 2 *and* Quest. Turns out, this is quite doable, although for Quest (Android) there's a strange hoop you have to jump trough. But it works!

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/handtracking.gif)

## The hand package
The documentation is, as I said, a bit limited and at points contradictory. At one point it states you should use Unity 2021.2+. I took Unity 2021.3.16f1 because I happened to have that one handy. I am going to assume you have both the UWP and Android components installed.

Once the project is set up, I first installed the hands package. You can [see on this page](https://docs.unity3d.com/Packages/com.unity.xr.hands@1.0/manual/project-setup/install-xrhands.html) where to get it. If you click the download link on that page, it will open a package in the Unity package manager:

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/packagemanager.png)

Click "Add". Unity will then ask to restart: agree.

After that - as the Unity documentation says - you should also import the samples:

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/samples.png)

## Installing Microsoft Mixed Reality OpenXR plugin

For that, we need the [MRTK feature tool](https://aka.ms/MRFeatureTool). The only thing we are going to install is the OpenXR plugin

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/openxrplugin.png)

## Setting up for HoloLens

First, go to File/Build settings, then select "Universal Windows Platform", perform the settings as displayed for UWP and click Switch Platform. Make sure the HandVisualizer sample scene (that we imported two steps back) is your main scene and included in the build

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/buildsettings1.png)


Then, click Edit/Project settings. Select the UWP tab (with the Window logo). Proceed by clicking XR Plug-in Management, then check the OpenXR and Microsoft HoloLens feature group checkboxes:

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/XRpluginUWP1.png)

Select the OpenXR plugin, and make sure settings look like this. Note: **Hand Tracking is off, Hand Tracking *Subsystem* is on.**

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/XRpluginUWP2.png)

Now click MixedReality/Project/Apply recommended project settings for HoloLens 2

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/projectsettings.png)

This will result in a number of warnings. Click "Fix all" on this screen, and most errors should go away. There is one about the camera not having a black background which will stay, but it has an edit button - click that button, and it will go away too.

Now simply build the app as normal, deploy to HoloLens 2, and hey presto:


![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/hlhand.gif)


## Setting up for Quest

First of all, of course, we switch to Android:

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/buildsettings2.png)

First order of business is to enable OpenXR for Android as well:

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/Android1.png)

And then here as well the hand tracking *subsystem* (not the ordinary hand tracking), and Oculus Quest support.

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/Android2.png)

However: we can see there is a problem. 

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/Openxr153.png)

Now the 2nd one we can easily do away with by clicking the "Fix" button. Right. But Unity only wants to serve me the OpenXR plugin 1.5.3 - and shows no way to update it 

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/Openxr153packagemanager.png)

I hope there is a simpler and smarter way to do it, but the only workaround this here simple developer could find was open the manifest.json file in the Packages folder of my project, locate the OpenXR package:

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/manifestjson.png)

And change 1.5.3. into 1.6.0. This will result in more, but different errors:

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/Openxr160.png)

But these are nice enough to go away by clicking "Fix all"

Then the screen looks a bit different: "Oculus" has been changed into "Meta". Make sure it looks like this before proceeding:

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/Android3.png)

Build the project, deploy the APK with SideQuest, and once again: hey presto

![](/assets/2023-02-18-Getting-Unity-XR-hands-to-work-on-HoloLens-and-Quest-in-a-bare-bones-project/questhand.gif)

## Great, now what is the use?

Well, to be honest, at the moment not very much yet. This Unity sample only *visualizes* the hands. But this shows an exiting way forward for utilizing hand tracking using pure Unity functionality and OpenXR, in stead of custom code for every platform or an extensive piece of code to abstract to into something being generally available - cross platform. But no doubt you could already use this to manipulate stuff around if you want. Standardization is what takes a platform forward. Unity clearly believes hand tracking will become a big thing - and rightly so, IMHO.

You can [download the resulting project here](https://github.com/LocalJoost/UnityXRHands). It should be able to deploy to both HoloLens and Quest without any changes - you only have to change the build target. It's been tested on a Quest 2, that's were the screenshots comes from.