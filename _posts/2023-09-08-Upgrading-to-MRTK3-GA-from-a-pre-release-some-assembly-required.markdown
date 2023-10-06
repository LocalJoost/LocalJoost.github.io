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
[Update - October 6, 2023]

Last Wednesday, the XR community received some *great* news: [MRTK3 finally entered the General Availability (GA) state](https://github.com/orgs/MixedRealityToolkit/discussions/452), which means this is the first real (non-pre-release) version. So today, I had some time to experiment with it and upgrade [HoloATC](https://www.microsoft.com/store/apps/9NVHWRS1V9ZH) to it. I was able to deploy and run it successfully on my HoloLens 2. However, let's say the upgrade story is a *bit* more rocky this time. So far, it was simply a matter of selecting the new libraries - new ones were installed, old ones were deleted, and Bob's your uncle. Now, we need to do a bit more work.

## Updating packages

My first surprise was when I opened the MRTK feature tool on my existing project:

![](/assets/2023-09-08-Upgrading-to-MRTK3-GA-from-a-pre-release-some-assembly-required/featuretool.png)

It doesn't show the old packages anymore. I reported my confusion about this, and Microsoft [have since then explained why this is](https://github.com/orgs/MixedRealityToolkit/discussions/452#discussioncomment-6951746). I quickly discovered the consequences of this: if you install the packages anyway, you will be greeted by a giant mass of "GUID xyz conflicts with" errors - because although the MRTK Feature Tool *adds* the GA packages indeed, *it does not remove the preview packages*. So basically, everything is now duplicated.

To fix that, take the following steps:
* Close the Unity project (if you still have it open).
* Open the "manifest.json" file in your project's "Packages" folder with a text editor.
* Remove every line that has "3.0.0-pre" in it.
* Save the manifest.json file.
* Remove all files with "3.0.0-pre" in their name from Packages\MixedReality (this is optional, but otherwise, the preview packages will stay in this folder - and your repo - potentially forever, doing nothing but wasting space).

## Fixing namespaces, part 1

Then came the second surprise: when I opened the project again after this step, Unity still complains about errors, but in my case, there were now only 42, instead of 999+. This is because everything that used to be in the namespace "Microsoft.MixedReality" is now in "MixedReality". They removed the "Microsoft" prefix *everywhere*. The fix is simple: in Visual Studio, do a global search & replace of "Microsoft.MixedReality" with "MixedReality". My project compiled and ran after that. *However*, if you used an API that is used in assemblies that have not changed namespace - like the **MRTK Graphic Tools** (currently in version 0.5.12), which have stuff in the namespace `Microsoft.MixedReality.GraphicsTools`, you will have to revert those. You will notice soon enough because of compile errors.

## Fixing the profile (if necessary)

The third thing you might want to have a *very* good look at. If you have used the *default* MRTK3 profile in Project Settings, everything you have selected will be selected correctly in the new profile. However, if you have created a *custom* profile - for instance, because you wanted to have the speech recognition subsystem enabled, or because there needed to be an additional subsystem, like the people at Magic Leap needed for their hand tracking, you might be in for a nasty surprise:

![](/assets/2023-09-08-Upgrading-to-MRTK3-GA-from-a-pre-release-some-assembly-required/subsystems.png)

Some of the subsystems your profile pointed to have changed names, so now your profile points to systems that no longer exist (see warning at the top), while the subsystems that were selected in the pre-release versions are now no longer selected. This might lead to, for instance, hand tracking no longer working. The trick is to make a new, empty MRTK profile and add whatever you need back to it, or make a fresh copy of the MRTK3 profile and change the settings to what they were before the upgrade.

## Fixing namespaces, part 2

There is a fourth thing you need to check: *prefabs* or even *scenes* might have references to namespaces as well. For instance, suppose you have created a simple default cube while using a pre-release version of the MRTK3, added an ObjectManipulator to that, and turned that into a prefab - and then upgraded, you are in for a surprise. If you open that prefab in an editor - for instance, Visual Studio Code

![](/assets/2023-09-08-Upgrading-to-MRTK3-GA-from-a-pre-release-some-assembly-required/prefab.png)

you will find no less than 10 references to namespaces and even assemblies that have changed names. So I did a global search & replace using Notepad++ to replace all references to "Microsoft.MixedReality" with "MixedReality".

## Fixing assembly references

If you have split up your app into multiple assemblies, references are not made automatically anymore - you have to make them yourself in the .asmdef file. [I wrote about this process almost three years ago](https://localjoost.github.io/Setting-up-unit-tests-in-Unity-to-test-your-Model-Driven-Mixed-Reality-app/), for being able to unit test objects. If you have made a reference to the MRTK assemblies, those references are now broken, and you will still have lots of compiler errors. I got this fixed pretty easily by switching to JetBrains Rider - it found the assemblies containing the now missing namespace automatically and offered to reference them. If you have to do this manually, it's best to do this on an assembly-by-assembly basis on two screens - one showing the unupgraded project in a Unity instance, the other showing the project being upgraded.

## Check suspicious asset file changes

At this point, the app compiled, I was able to deploy and run it. Only - and this was very weird - short-range hand interaction did not work. Far interaction - with a hand ray - fine, but the poke pointer - the circle that appears on your index finger - did not appear, and I could not press buttons. I noticed one of the asset files had changed. I reverted it, rebuilt the app, and then my problem was solved. The issue only is - I don't quite remember *which* asset file. I seem to recall it was **OpenXR Package Settings.asset** (that sits in Assets\XR\Settings), but I am not sure. At that time, I was a bit stressed because it seemed I was going to have defeat snatched from the jaws of victory.

## Concluding thoughts

The namespace issue, of course, is a logical side effect of the MRTK3 [coming under the stewardship of an extended steering committee in which not only Microsoft has a seat at the table but also Qualcomm and Magic Leap](https://techcommunity.microsoft.com/t5/mixed-reality-blog/microsoft-mixed-reality-toolkit-3-mrtk3-moves-to-an-independent/ba-p/3898941), as announced by Robin Seiler, Corporate Vice President and Chief Operating Officer of the Windows + Devices organization, on August 21. However, this required quite a bit of work, with some challenging pitfalls. It may be related to the fact that I skipped the pre-18 version and went directly from pre-17 to GA. I initially attempted to upgrade to pre-18, as suggested on the HoloDevelopers Slack, but that did not work out. So, I started from scratch and this time I got to the other end.

The important thing is: the MRTK is GA, and its future seems secured - an important thing for the future of XR!