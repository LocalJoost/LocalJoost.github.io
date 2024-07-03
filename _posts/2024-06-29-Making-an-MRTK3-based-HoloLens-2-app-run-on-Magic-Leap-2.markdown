---
layout: post
title: Making an MRTK3 based HoloLens 2 app run on Magic Leap 2
date: 2024-06-29T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- Magic Leap 2
- HoloLens 2
- Mixed Reality
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/switchtarget.png
comment_issue_id: 472
---
This is a bit of an unusual post, as it does not have a project attached to it. Instead, it's more like a *recipe* that I followed to get my app "Walk the World", originally developed for HoloLens 2, to run on Magic Leap 2. Or actually, how I did it after retracing all my steps, cutting out all the useless detours, pitfalls, and dead ends I encountered trying it the first time.

## Upgrade MRTK3

First things first: upgrade to the latest version of MRTK3. There is no two ways about it; otherwise, you will run into issues with the Magic Leap MRTK3 package. Check if your app still works on HoloLens 2 after that.

## Switch build target to Android

This is a simple step: switch the build target to Android and close Unity.

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/switchtarget.png)

## Identify and add necessary packages

I found the [MRTK3 setup instructions for Magic Leap 2](https://developer-docs.magicleap.cloud/docs/guides/third-party/mrtk3/mrtk3-overview/) not very clear, and after a failed upgrade, I devised an alternative strategy.

You first need to pull the [template Magic Leap MRTK3 project](https://github.com/magicleap/MixedRealityToolkit-Unity), which is actually a fork of MRTK3 made by Magic Leap, from GitHub.

Then, compare the manifest file of the template project with the manifest file of your project using [the little tool](https://github.com/LocalJoost/UnityManifestCompare/releases/download/1.0.0.0/UnityManifestCompare.zip) I made for [my previous blog post](https://localjoost.github.io/A-little-tool-to-compare-two-Unity-Manifest-files/). You will find the dev template manifest in MixedRealityToolkit-Unity\UnityProjects\MRTKDevTemplate\Packages. Use the Magic Leap MRTK3 manifest file as the first argument, and your manifest file as the second.

Finally, you will need to add the Magic Leap scope and the missing packages as [described and explained in my previous blog post](https://localjoost.github.io/A-little-tool-to-compare-two-Unity-Manifest-files/).

## Set Magic Leap audio settings

Open Unity again, and let it import all the packages we just added in the manifest file.

When it's done, you will see these two message boxes pass by:

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/spatializer.png)

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/ambisonic.png)

Click OK in both cases. In the end, the package manager will pop up, informing you a scope has been added.

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/packagemanageropen.png)

To prevent you from getting those two messages about audio every time you open the project, select "MSA Spatializer" and "MSA Ambisonic Plugin" in the audio tab of the project settings.

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/audiosettings.png)

## Install Magic Leap Project Setup Tool

Now it's time to install the [Magic Leap Project Setup Tool from the Unity Asset Store](https://assetstore.unity.com/packages/tools/integration/magic-leap-setup-tool-194780). At the time of this writing, you will get version 2.0.10.

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/setuptoolpackagemanager.png)

After it's done importing, this box will pop up:

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/sdkchoice.png)

The choice is yours, but I prefer to go with OpenXR, and in any case, the Magic Leap SDK is deprecated, so that's a dead end anyway. This tutorial uses OpenXR, so I'd suggest you click "OpenXR."

The Project Setup Tool will pop up, and all kinds of stuff is being imported into Unity. Wait till it's done. Then it asks *again* the same question:
s
![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/sdkchoice.png)

Click OpenXR again. Then nothing seems to happen for a while (be patient), but finally, the Project Setup Tool shows a bunch of green and yellow buttons.

## Configure settings using the Project Setup Tool

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/setuptool1.png)

It's very tempting to click "Apply All", but my experience is that things usually work out better if you press the buttons one by one and wait for the result.

* "Enable plugin" will pop up the XR Plugin management; put that aside and continue.
* "Use DXT Texture Compression" will open a progress box that will close again.
* "Apply XR validation" will do something, whatever it is, but the button turns green.
* "Change to X86_X64 architecture" changes the architecture; you will see nothing happening but the button turn green.
* "Set Minimum Android API level": Set API level to 29 (currently), no visual clue but green button.
* "Update the Manifest file" will undoubtedly add something to the manifest, although I don't know what - button turns green.
* "Use Vulkan Graphics API" - set the graphic API, button turns green.

At this point, all the buttons should be green.

## Setting necessary permissions.

Now it's time to review the permissions. We know these in HoloLens as *capabilities*. Some capabilities are just granted when they are selected - like internet access - but some require explicit user consent - like microphone access. But all the capabilities for HoloLens 2 are in one list; you can't really see which require consent or not - but in any case, HoloLens 2 will automatically ask for consent when necessary.

Magic Leap 2 knows a similar concept called *permissions*. The difference is: they are *explicitly divided into two groups*: normal and 'dangerous' permissions. Dangerous permissions need explicit user consent. It's neat you can easily see what permissions are considered dangerous - although it's also rather odd spatial maaping is considered a dangerous permission. Magic Leap is a Mixed Reality device, and *the very point and USP* of such a device is the ability to *interact with reality*. It feels a bit like buiding a boat but not putting it into water without special permission. I can't really see how it possibly would open doors to surreptitious activities like, for instance, getting microphone access would - but hey, whoever makes the devices, writes the rules.

Anyway, we need (or want) to use the app with hand tracking, and we also require spatial mapping as we want to be able to put the map on the floor:

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/permissions1.png)

Also different is that permissions need to be set twice, in two different places - on the Magic Leap level, and the MRTK3 level. These are not always nicely in sync. Make sure they are in sync before proceeding.

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/permissions2.png)

Notice the "Auto Request Dangerous Permissions". That used to work nicely, making the experience on par with how HoloLens 2 handles it, but with the SDK 2.2.0 I used, that does not seem to work anymore. So you will need to ask for permissions in code, which I will come to later.

## Setting the MRTK 3 profile

The MRTK3 cannot work without a profile. So we need to select one.

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/profile.png)

If you click the tempting button "Assign MRTK default," it looks like you are done with it - but alas, you will set the MRTK default profile, which is wrong. You see, Magic Leap 2 uses different subsystems. So to be sure you don't get any nasty surprises, you need to select the little dot on the right (marked 1) and then the eye icon on the right of the popup window - and you will see no less than 3 MRTK profiles. As you might have guessed, we need the MRTKProfile-MagicLeap-OpenXR.

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/mrtksettings1.png)

## Configuring Magic Leap OpenXR settings

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/openxrsettings.png)

I added the two interaction profiles in the red square by clicking the little plus icon indicated by the arrow top right, and turned on the Hand Tracking System. I turned *off* the Magic Leap Spatial Anchor Storage because I noticed a lot of errors in the Unity Application Log when I checked that using my [ServiceFramework-based FileLoggerService](https://localjoost.github.io/A-cross-platform-Service-Framework-service-to-write-Unity-log-messages-into-files-on-device/). The Pixel sensor thing was already off by default, so I left it off. I am not sure whether you can turn off other things and still have it work, nor what kind of advantages that might have. I haven't gone deep into this.

## Turning off the old Unity Input System

If you build and get this error:

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/inputsystem.png)

Change this in Other settings (in Player settings
comment_issue_id: 472)

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/inputsystems.png)

and restart the editor.

You will then have to check if you have a nifty help function like this one, that shows the hand menu in the editor by simply pressing the "H" key. Because this particular one is between `#if UNITY_EDITOR` preprocessor tags, it will not hurt the app, but by golly, does it keep the Unity console busy with errors. So best remove functions using stuff like this:

```csharp
#if UNITY_EDITOR       
private void Update()
{
    if (Input.GetKeyDown(KeyCode.H))
    {
        var menu = visuals.transform.GetChild(0);
        menu.gameObject.SetActive(!menu.gameObject.activeSelf);
        visuals.transform.position = 
           LookingDirectionHelpers.CalculatePositionDeadAhead(0.3f);
        visuals.transform.LookAt(Camera.main.transform);
        visuals.transform.Rotate(Vector3.up, 180);
    }
}
#endif
```

## Add code to ask for permissions

Automatic dangerous permission did not work, so I had to add explicit code to ask for permissions. I assume this to be a bug (I reported it as such), so this requirement will probably go away.
```csharp
public override void Initialize()
{
#if MAGICLEAP   
    Permissions.RequestPermissions(new[]{
        MLPermission.SpatialMapping} ,
        OnPermissionGranted, OnPermissionDenied, OnPermissionDenied);
#endif
    model.Initialize();
}
private void OnPermissionDenied(string permission)
{
    // Do something
}

private void OnPermissionGranted(string permission)
{
    // Do something
}
```
## Change some input settings

These are not mandatory but *highly* recommended. Go to Player Settings/InputSettings/Other settings and change these settings:

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/identification.png)

You never saw this before if you only did HoloLens development, but in Unity for Android, you need to set a package name. Don't use the default, but enter a real package name.

Then we have this thing. In HoloLens 2 apps (or rather, UWP apps), the name of the product is not necessarily the text that appears on the label behind the icon. In Magic Leap 2 apps (or rather, Android apps) it is, so if you have some random project name, please change it into something that looks nice, like I did.

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/player1.png)

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/player2.png)

## Upgrade packages

Also not mandatory but recommended: after you have confirmed the app runs, you might also consider looking at the package manager to see if any packages can or need to be upgraded. My package manager says some packages have slightly new versions indeed. I upgraded most of them, and it still works. Not sure what I gained.

![](/assets/2024-06-29-Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/packagemanagerupgrade.png)

## Concluding words

After you have completed the mandatory steps, your app should simply be able to compile, and you should be able to deploy and run it on Magic Leap 2. Of course, your mileage may vary if you use very specific HoloLens 2 functionality, like iris login or tracking QR codes.

There is only one thing I could not get to work: while the hand tracking works perfectly, I could *not* get the Magic Leap 2 *controller* to work. It fails in a very weird way. First of all, it does not show a ray. But the really weird thing is this: if you point the controller at a button (which is a bit tricky without a ray, but if you point carefully, you can do it) and press the trigger - the button animates as if it is pressed, the click sound is audible, but the `OnClick` event in the `StatefulInteractable` is not fired. So apart from that animation and sound, nothing happens. Very odd. I assume this to be a bug, and I reported it.

Enjoy your cross-platform XR development. If you use this 'recipe', let me know how it worked out for you.