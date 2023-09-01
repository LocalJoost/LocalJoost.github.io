---
layout: post
title: Using MRTK3 pre17 on Magic Leap 2
date: 2023-08-13T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- Magic Leap 2
published: true
permalink: 
featuredImageUrl: 
comment_issue_id: 453
---
As [James Ashley](https://www.linkedin.com/in/imaginativeuniversal/) demonstrated in his AWE talk in May, it's perfectly possible to run an MRTK 3 app on [Magic Leap 2](https://www.magicleap.com/magic-leap-2), provided you take some steps - [which are now adequately described in the Magic Leap developer docs](https://developer-docs.magicleap.cloud/docs/guides/third-party/mrtk3/mrtk3-overview). However, what it does *not* say is that this *only works up until the MRTK 3 pre 16 release*. If you try to import the newest (pre-17) package, Unity will crash if you still have it open, and if you re-open the project, you are greeted with a multitude of vague errors. However, on closer look, three of those are very clear:

**Library\PackageCache\com.magicleap.mrtk3@8f7cbd2e78d5\Runtime\MagicLeap\Scripts\MagicLeapControllerHandProximityDisabler.cs(49,59): error CS0266: Cannot implicitly convert type 'Microsoft.MixedReality.Toolkit.Subsystems.IHandsAggregatorSubsystem' to 'Microsoft.MixedReality.Toolkit.Subsystems.HandsAggregatorSubsystem'. An explicit conversion exists (are you missing a cast?)**

## What happened here?

Well, actually, it's pretty simple. Microsoft has changed the return value of `XRSubsystemHelpers.HandsAggregator` from `HandsAggregatorSubsystem` to `IHandsAggregatorSubsystem`. So the com.magicleap.mrtk3-1.0.0-pre.1 package is not compatible anymore with the latest MRTK3. This affects three files in the package. However, that is easy to fix.

## Simple hack fix

* Open com.magicleap.mrtk3-1.0.0-pre.1.tgz (in the project's packages folder) with a suitable tool, for instance [7-Zip](https://www.7-zip.org/).
* Find the three offending files:
  * MagicLeapAuxiliaryHandDevice.cs, 
  * MagicLeapControllerHandProximityDisabler.cs
  * TrackedHandJointVisualizer.cs
* Extract those files to a folder on your disk (using drag & drop)
* In every file, find the line that says something like:

```csharp
HandsAggregatorSubsystem HandSubsystem => XRSubsystemHelpers.HandsAggregator;
```

* Change that into:

```csharp
IHandsAggregatorSubsystem HandSubsystem => XRSubsystemHelpers.HandsAggregator;
```

* Copy the files back into the archive.

And presto. No more errors, no more crashes.

The real lazy people [might also just download the fixed archive](https://www.schaikweb.net/ML2/com.magicleap.mrtk3-1.0.0-pre.1.tgz) from my old website and plonk it directly over the one in packages ;)

## No panic

Yeah, I have a Magic Leap 2 lying here since a few days. No, it's not mine. No, I am not abandoning HoloLens 2. Yes, I do like Magic Leap 2 although it has some clear cons compared to HoloLens 2. Yes, it also has some very clear pros. Anyway, it's just that I am very much a proponent of cross-platform development. You have seen me porting my app to Quest 2/Pro and even Android ARCore phones - and even putting them up for download in their various stores. I think choice is important, cross-platform development is important (and fun, as well as challenging), and I intend to write about some findings and pitfalls I come across. So other people don't need to fall into the same pit. Basically, what I have been doing since 2007.

## So how, why?

In the last week of April, while I was on holiday in northern Germany, I came to talk with James on Slack, who said he was going to do a talk on AWE about porting HoloLens 2 apps to Magic Leap 2. That triggered me to try porting my iconic HoloATC app. Not having a Magic Leap 2 myself, this was a bit of a cumbersome process: try something, send it to James, get feedback on what did not work, rinse and repeat. After four tries, the app worked, and it was done in time for James to actually demonstrate it during his AWE talk. At [about 4:43, he starts showing off my app](https://youtu.be/BkE3y_uSfa8?t=283). And then I was cheeky enough to contact the folks at Magic Leap and ask for a loaner device to make this rough hack into a real port. And to my surprise, after some back and forth, they actually agreed. And here we are ;) It will need to go back to its maker at one point, but in the meantime, I have a new challenging environment to work with.