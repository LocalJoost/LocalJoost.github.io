---
layout: post
title: MRTK2 to MRTK3 - migrating the QRCode sample
date: 2022-07-02T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- HoloLens2
- Unity
- Windows Mixed Reality
- Service Framework
- Reality Collective
featuredImageUrl: https://LocalJoost.github.io/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/packages.png
comment_issue_id: 423
---
As I [announced in a previous post](https://localjoost.github.io/MRTK2-to-MRTK3-migrating-Extension-Services-to-the-Reality-Collective-Service-Framework/), in which I migrated the extension services to the [Reality Collective Service Framework](https://service-framework.realitycollective.io/docs/get-started), I am now migrating the whole sample to MRTK3. Migrating the services was the ground work because, as I explained in said post, MRTK3 comes without a service host.

## Step 1: remove MRTK2
The easiest way get that done, is via the Package Manager. Remove these packages:
![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/packages.png)

## Step 2: Add MRTK3
Start the MRTK feature tool and add the following MRTK3 packages:
* MRTK Core Definitions
* MRTK Graphic Tools
* MRTK Input
* MRTK Spatial Manipulation
* MRTK Standard Assets
* MRTK UX Components
* MRTK UX Core Scripts

Maybe a few are superfluous, I don't always check for the very minimal stuff. Comes with having a fast computer ;)

## Update code
As far as code is concerned, updates are *very* few. It turns out the static `CameraCache` class has been moved from namespace `Microsoft.MixedReality.Toolkit.Utilities` to namespace `Microsoft.MixedReality.Toolkit` so you only need to change the using statements, in the following classes:
* SpatialGraphCoordinateSystemSetter
* OpenXRSpatialGraphCoordinateSystemSetter

In three other files, the service initialization still waits on the Mixed Reality Toolkit being initialized - but this initialization check has disappeared. Since it's actually waiting for the *service host* being initialized, we need to change:
```csharp
while (!MixedRealityToolkit.IsInitialized && Time.time < 5) ;
```
into
```csharp
while (!ServiceManager.Instance.IsInitialized && Time.time < 5) ;
```
in the following three classes:
* QRCodeDisplayController
* QRTrackingServiceDebugController
* QRTrackerController

And now your project will actually have zero compile errors left. Granted, there's a lot of other stuff broken, but that's all in the scene.

## Fixing SampleScene2
This is the scene which locates the QR code in space and places a model airplane on it. Since this is the most comprehensive scene, so we'll take on this one to migrate.

### Remove old stuff, bring in the new

So the game objects "MixedRealityToolkit" and "MixedRealityPlaceSpace" are remnants of the past. So remove those first.

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/removefromscene.png)

Then, search for "MRTK XR Rig" in your packages and drag that over to your scene

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/XRrig.png)

Finally, drag "MRTK Input Simulator" on top of the MRTK XR Rig. Or where ever else you like it, but this seems to be the usual place

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/inputsimulator.png)

And finally, although I am not sure if this is strictly necessary, I click both of these menu items: 

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/menu.png)

### Fixing missing scripts

If you look at the QRCodeDisplayer 'dialog box', you will notice a) it looks kinda whitish in stead of blue, and b) there are two missing scripts:

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/missingsolver.png)

These are actually Radial View and Solver - familiar from MRTK2, but apparently they got a different asset id now. Anyway, remove the two missing script references and add them anew with the following parameters:

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/radial.png)

### Fixing backplates

In the display box, you will find the Backplate gameobject containing a quad:

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/backplate.png)

Remove Backplate completely, and replace it by a UIBackplate component from MRTK3:

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/uibackplate.png)

You will need it to stretch it a bit horizontally. The text will look a bit janky:

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/janky.png)

But you can fix that by changing the Z position of both the Header and DataText from 0 to -0.009

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/zpos.png)

And done:

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/done.png)

### Fixing hand menu part 1 - scripts

The hand menu looks like it's going to need some more work. Not only scripts are gone, but apparently so are the materials, and the button is also completely kaputt. 

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/handmenu1.png)

However, "things are never quite the way they seem" (bonus points for who gets the quote without googlebinging). Reparing is quite easy. Remove the three missing scripts, then add a "Hand constraint (Palm up)" component. This adds automatically the other two necessary behaviours, "Hand Bounds" and "Solver Handler" - also familiar from the past. Now the only thing left as far as scripts for the hand menu itself are concerned, are configuring the Solver Handler and the Hand constraint (Palm up) behaviours. The first one goes as follows:
* Set "Tracked Hand Type" to "Hand Joint"
* Check if "Tracked Handeness" is set Left and Right
* Set Tracked Hand Joint to is set to "Palm"

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/solver.png)

And for the Hand constraint (Palm up) you need to re-configure the On Hand Activate and Deactivate events, to let the MenuHolder appear and disappear at the right time:

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/handactivatedeactivate.png)

### Fixing hand menu part 2 - button

The button is completely broken, so let's remove that entirely and replace it by a prefab "PressableButton_32x32mm_IconAndText" from MRTK3. 

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/button.png)

Unfortunately, as of yet, these new buttons come without the ButtonConfigHelper that was present on the MRTK2 buttons, so we have to dig a little deeper to change the icon and the text:

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/texticon.png)

The text is simply a TextMeshPro whose contents you can replace like any other TextMeshPro, for icon I took "Icon_Refresh_F.png" from the package com.microsoft.mrtk.standardassets, which gives the following result

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/finishedbutton.png)

And finally, to make the button actually *work*, we have to re-configure the buttons' Pressable Button's OnClick event to Tracker1.QRTrackerController.ResetTracking again.

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/tracker.png)

### Fixing hand menu part 3 - backplate

This is basically a repeat of what we did to the QRCodeDisplayer backplate. Remove the backplate, replace it by a UIBackplate.

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/backplate2.png)

Change size as needed, and in this case, move it a wee bit backward. Net result

![](/assets/2022-07-02-MRTK2-to-MRTK3--migrating-the-QRCode-sample/finishedbuttonwithbackplate.png)

## Conclusion & thoughts

There is much more to do, but now you basically have a fully functional QR tracker again, based on MRTK3, MRTK3 assets and a bit of help from the Service Framework. There is more to do - updating the UI of the Debug Displayer, but that's basically replacing the back plate and re-applying the Radial View again. The other scene (SampleScene) - that only reads QR codes and displays the value - I leave as exercise to the reader. It largely a repeat of things I just described. For those who don't feel like exercising, or people who are just interested in the end result, I pushed the finished update to [the MRTK3 branch of my QRCodeService repo](https://github.com/LocalJoost/QRCodeService/tree/MRTK3).

As you might have noticed, it looks like most of the work migrating to MRTK2 to MRTK3 will consist of largely rebuilding the UI. I find it peculiar that behaviours having the same name and function (SolverHander, RadialView, HandBounds, HandPalmContstraintUp) in MRTK3 as in MRTK2, apparently have a different asset id so you have to re-apply all of them. If the id had been retained, they would have automatically replaced by the new behaviour. But then again, the MRTK3 is still in preview, rough edges like this might be expected.