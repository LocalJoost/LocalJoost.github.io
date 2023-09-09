---
layout: post
title: 'Upgrading to MRTK3 GA from a pre-release: some assembly required'
date: 2023-09-08T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- GA
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2023-09-08-Upgrading-to-MRTK3-GA-from-a-pre-release-some-assembly-required/featuretool.png
comment_issue_id: 454
---
Last Wednesday, the XR community received some *great* news:[ MRTK3 finally entered the General Availability (GA) state](https://github.com/orgs/MixedRealityToolkit/discussions/452), which means this is the first real (non-pre-release) version. So today, I had some time to experiment with it and upgrade [HoloATC](https://www.microsoft.com/store/apps/9NVHWRS1V9ZH) to it. I was able to deploy and run it successfully on my HoloLens 2. However, let's say the upgrade story is a *bit* more rocky this time. So far, it was simply a matter of selecting the new libraries - new ones were installed, old ones were deleted, and Bob's your uncle. Now, we need to do a bit more work.

My first surprise was when I opened the MRTK feature tool on my existing project:

![](/assets/2023-09-08-Upgrading-to-MRTK3-GA-from-a-pre-release-some-assembly-required/featuretool.png)

I doesn't show the old packages anymore. I reported my confusion about this, and Microsoft [have since then explained why this is](https://github.com/orgs/MixedRealityToolkit/discussions/452#discussioncomment-6951746). I quickly discovered the consequences of this: if you install the packages anyway, you will be greeted by a giant mass of "GUID xyz conflicts with" errors - because although the MRTK Feature Tool *adds* the GA packages indeed, *it does not remove the preview packages*. So basically, everything is now duplicated.

To fix that, take the following steps:
* Close the Unity project (if you still have it open).
* Open the "manifest.json" file in your project's "Packages" folder with a text editor.
* Remove every line that has "3.0.0-pre" in it.
* Save the manifest.json file.
* Remove all files with "3.0.0-pre" in their name from Packages\MixedReality (this is optional, but otherwise, the preview packages will stay in this folder - and your repo - potentially forever, doing nothing but wasting space).

Then came the second surprise: when I opened the project again after this step, Unity still complains about errors, but in my case, there were now only 42, instead of 999+. This is because everything that used to be in the namespace "Microsoft.MixedReality" is now in "MixedReality". They removed the "Microsoft" prefix *everywhere*. The fix is simple: in Visual Studio, do a global search & replace of "Microsoft.MixedReality" with "MixedReality". My project compiled and ran after that.

The third thing you might want to have a *very* good look at. If you have used the *default* MRKT3 profile in Project Settings, everything you have selected will be selected correctly in the new profile. However, if you have created a *custom* profile - for instance, because you wanted to have the speech recognition subsystem enable, of because there needed to be an additional subsystem, like the people at Magic Leap needed for their hand tracking, you might be in for a nasty surpise:

![](/assets/2023-09-08-Upgrading-to-MRTK3-GA-from-a-pre-release-some-assembly-required/subsystems.png)

Some of the subsystems your profile pointed to have changed names, so now your profile points to systems that no longer exist (see warning at the top), while the subsystems that were selected in the pre-release versions are now no longer selected. This might lead to, for instance, hand tracking no longer working. The trick is to make a new, empty MRTK profile and add whatever you need back to it, or make a fresh copy of the MRTK3 profile and change the settings to what they were before the upgrade

## Concluding thoughts

The namespace thing, of course, is a logical side-effect from the MRTK3 [coming under the stewardship of an extended steering committee in which not only Microsoft has a seat at the table, but also Qualcomm and Magic Leap](https://techcommunity.microsoft.com/t5/mixed-reality-blog/microsoft-mixed-reality-toolkit-3-mrtk3-moves-to-an-independent/ba-p/3898941), as Robin Seiler, Corporate Vice President and Chief Operating Officer of the Windows + Devices organization, announced on August 21. I feel it might have been a good idea to release a preview first, with a change as radical as this - but then again, I am just a developer looking from the outside, I don't have an overview of all the details involved that resulted in implementing this directly into the GA.

The important thing is: the MRTK is GA, and its future seems secured - an important thing for the future of XR!