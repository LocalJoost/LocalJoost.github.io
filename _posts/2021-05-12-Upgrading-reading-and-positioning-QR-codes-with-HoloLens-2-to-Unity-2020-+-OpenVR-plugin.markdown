---
layout: post
title: Upgrading reading and positioning QR codes with HoloLens 2 to Unity 2020 + OpenVR plugin
date: 2021-05-12T00:00:00.0000000+02:00
categories: []
tags:
- HoloLens2
- MRKT2
- Unity3d
- Windows Mixed Reality
- QRCode
featuredImageUrl: https://LocalJoost.github.io/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/mrft1.png
comment_issue_id: 380
---
In a previous posts I showed [how to read QR codes with HoloLens 2](https://localjoost.github.io/Reading-QR-codes-with-an-MRTK2-Extension-Service/), and how to use those to [get a position in space](https://localjoost.github.io/Positioning-QR-codes-in-space-with-HoloLens-2-building-a-'poor-man's-Vuforia'/) - using that position to align a Hologram to. I used the recommended Unity 2019 LTS, combined with built-in XR. But the writing is on the wall: Unity's 'Long' Term Service support is not very long, and soon there will come a time when we need to move beyond Unity 2019. In that version, Unity already called the built-in XR support 'legacy' and in 2020 and beyond it's *gone*. And although Microsoft are still recommending Unity 2019.4 LTS and the built-in XR support for HoloLens 2 development, it's definitely time to look forward and identify possible roadblocks. 

In this post I am not conveying much new information. Most can be found on various Microsoft web pages. I just compiled it into a single post with a step-by-step explanation of how *I* upgraded my *own* project doing this specific thing.

## First, a choice, but not really
Moving to 2020 means going for the plugin structure. Right now, you can choose between two plugins: the Windows XR Plugin, or the Mixed Reality OpenXR plugin. On [this page](https://docs.microsoft.com/en-us/windows/mixed-reality/develop/unity/choosing-unity-version) Microsoft provides some guidance, which can be summarized as follows:
* The Windows XR plugin suffers from some at least three issues, one of them being 'severe hologram instability'
* The Windows XR plugin will be deprecated *as well* soon, in Unity 2021.2
* Later this year, Unity 2020.3 LTS and the OpenXR plugin will become the recommended Unity configuration.
* The OpenXR plugin will be the only path forward from Unity 2021.2

So the current choice is between a plugin with severe issues that will be deprecated soon, or a plugin that is not done yet but will be the only part forward. If you are looking into shipping a project soonish, the guidance is simple: stop right here. Neither plugins are airworthy at this time. But if you want to do some experiments with your product to see if there are roadblocks ahead for future versions, there is only one option: forget about the Windows XR plugin, and go for OpenXR. Make a branch of your code and see how far you can get, and please report any issues to Microsoft.  

## A little heads-up
After I wrote my last post about QR codes, I made a few little improvements in the code, which are in the branch [slightly_improved_version](https://github.com/LocalJoost/QRCodeService/tree/slightly_improved_version). This will be the starting point for the upgrade path. 

## Step 1: install the Mixed Reality Feature Tool
Microsoft now offers the [Mixed Reality Feature Tool for Unity](https://docs.microsoft.com/en-us/windows/mixed-reality/develop/unity/welcome-to-mr-feature-tool), which is a new way for installing Mixed Reality features into Unity. It has definite advantages over the Unity package manager - most notably showing packages available, their dependencies, and showing upgrade suggestions. Although it still says "beta" in the version label at the moment, it works perfectly fine for me.

## Step 2: upgrade to the newest MRKT
My app was built using the MRTK, like all my apps, and although some people do not seem to like it because it is allegedly complex and opinionated, I happen to like it very much - and it proves a nice level of cross platform features and abstracts that insulate you from the underlying platform implementation. This latter part has proven to be a huge boon when upgrading for me as I hardly had to change *anything* in my own code and setup. 

Enough of that - start the Mixed Reality Feature Tool (don't start Unity yet) and point it to your project:

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/mrft1.png)

Hit the discover button, and on the next screen expand the "Mixed Reality Toolkit" panel. 

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/mrft2.png)

And here we see the current project the MRTK 2.5.3 installed (using the Unity Package Manager still, by the way). The tool shows me There's a new version, but there is also shows a host of *other* stuff that is available, which I encourage you to explore - but now right now ;). Anyway, select all the stuff that says "Version 2.5.3 is currently installed"

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/mrft3.png)  

and hit "Get Features"

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/mrft4.png)

and finally "Approve"

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/mrft5.png)

And you are done. Unity will now proceed to load the features. I personally always close Unity before I use the Mixed Reality Feature Tool and re-open it after I am done.

## Step 3: Opening the project in Unity, initial settings
Use the version of Unity *the project was created in* for this step, in for this project that's Unity 2019.4.17f1. When you open the project in Unity, you will be greeted by this

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/shaderupgrade.png)

I am too dumb to customize shaders, so I always hit "Yes". Next up is the MRKT configurator that wants to do some settings:

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/projectconfig.png)

Although it's not strictly necessary, I usually click "apply" (as this is a demo app anyway). Now is also a good moment to check if the build setting for your app are set to UWP (File/Build Settings)

## Step 4: Install and select plugin management

Hit Edit/Project Settings and select "XR Plugin Management". 

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/pluginmanagement1.png)

This will show you this rather ominous message. I hate doing things manually that apparently can be done automatically for me, so I hit Ok

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/pluginmanagement2.png)

This shows another dialog box indicating you are removing Windows Mixed Reality.

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/pluginmanagement3.png)

I thought I already said Ok, but apparently they want to make sure you really really want to do this. I really really did, so I hit Ok again

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/pluginmanagement4.png)

Then you get this screen. You can now only select Windows Mixed Reality plugin. Select this for now. 

## Step 5: Upgrade to Unity 2020.3 LTS

At the moment this is 2020.3.7, so you might have to install a new version first - or just use the 2020.3 version you have available. I have currently 11 different Unity versions on my development PC - welcome to the wonderful world of XR development with Unity ;)

If you open the the project with the a Unity 2020.3 LTS version you will be greeted by this.

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/ignore.png)

Hit ignore, we will solve this in step 10. Let Unity perform the upgrade and import everything a new. Then close it again

## Step 6: Install the OpenXR plugin

For this we need the Mixed Reality Feature Tool again.

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/installplugin.png)

Expand tab "Platform Support", then "Mixed Reality OpenXR Plugin" and proceed as in step 2 by importing the package.

## Step 7 Apply changed  settings in Unity

Open Unity again. This will first ask you about enabling native platform backends for the new input system.

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/inputprompt.png)

Hit "Yes". Unity will indeed close and restart

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/ignore.png)

Hit "Ignore" again. The MRKT project configuration now will suggest enabling the input simulation system. This is mightily handy for development inside the Unity editor, so hit "Yes". Unity will restart *again*...

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/projectconfig2.png)

... and again ask us about safe mode, which we will ignore again

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/ignore.png)

## Step 8: Select and configure Open XR plugin

Now we are to the real thing: selecting the OpenXR plugin. Select File/Build Settings. Then hit the "Player Settings button:

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/playersettings_openxr.png)

Select the check boxes before "OpenXR" and "Microsoft HoloLens feature set". You will notice a small orange exclamation sign in right of "OpenXR". Click that, and that will show this popup:

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/fixopenvrsettings.png)

Click "Fix All"

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/openxr_issues_fixed.png)

The plugin is now succesfully configured.

## Step 9: Update MRTK profile settings

Microsoft are providing some excellent step-by-step guidance on how to configure the MRKT for use with the OpenXR plugin, explaining how to set up a new project as well as updating an existing configuration. I don't quite feel like repeating that. [Just follow the steps here.](https://docs.microsoft.com/en-us/windows/mixed-reality/mrtk-unity/configuration/getting-started-with-mrtk-and-xrsdk#configuring-mrtk-for-the-xr-sdk-pipeline). Updating is a bit more complex than starting anew, but is certainly advisable if you have a lot of custom profile settings (which I usually have, as I am using the [censored] out of the MRKTs configuration options). Do be careful to follow the instructions to the letter - instructions for the soon-to-be-deprecated Windows Mixed Reality XR plugin and the OpenXR plugin are provided *in the same text*, and mixing up is easy - with very undesirable results. It's not hard, just double check every setting to make sure it's accurate. 

Also, *make sure to follow the tip about link.xml at the very end of the section*.

## Step 10: Finally fixing those annoying compiler errors

Up until know we have done *nothing whatsoever* that has *anything* to do with QR code reading specifically - everything up to this step might as well be a generic upgrade guide to OpenXR. However, ever since step 5 you have been pestered by Unity wanting to open the project in Safe Mode due to compiler errors, and you may have seen this appear in the console window:

**SpatialGraphCoordinateSystemSetter.cs(34,67): error CS0246: The type or namespace name 'PositionalLocatorState' could not be found (are you missing a using directive or an assembly reference?).**

This is because the code in `SpatialGraphCoordinateSystemSetter` is using an API that is no longer available now we have disabled Legacy XR. To fix this, I have refactored `SpatialGraphCoordinateSystemSetter` to a an abstract base class with two concrete child classes: `LegacySpatialGraphCoordinateSystemSetter` and `OpenXRSpatialGraphCoordinateSystemSetter` 

In `SpatialGraphCoordinateSystemSetter`, `UpdateLocation` is now an abstract method. There is now an `CalculatePosition` method, that contains the matrix magic that I nicked from [the original Microsoft sample](https://github.com/chgatla-microsoft/QRTracking/tree/master/SampleQRCodes) - what used to be the middle part of `UpdateLocation`. What used to be the *last* part of `UpdateLocation` is now `MovePoseToCenter` - the part that calculates the center of the QR code and provides an - IHMO - more logical orientation using this little piece of code I devised myself.

```csharp
protected void MovePoseToCenter(Pose pose,float physicalSideLength)
{
    // Rotate 90 degrees 'forward' over 'right' so 'up' is pointing 
    // straight up from the QR code
    pose.rotation *= Quaternion.Euler(90, 0, 0);

    // Move the anchor point to the *center* of the QR code
    var deltaToCenter = physicalSideLength * 0.5f;
    pose.position += (pose.rotation * (deltaToCenter * Vector3.right) -
                      pose.rotation * (deltaToCenter * Vector3.forward));
    CheckPosition(pose);
}
```
The other methods are still unchanged.

In `LegacySpatialGraphCoordinateSystemSetter` the coordinate systems are still retrieved using `SpatialGraphInteropPreview.CreateCoordinateSystemForNode` and `WorldManager.GetNativeISpatialCoordinateSystemPtr()`, then `CalculatePosition` - the matrix magic - is called, which in turn calls `MovePoseToCenter`. I have added a `#if !UNITY_2020_1_OR_NEWER` preprocessor statement to make sure this does not bother us in newer versions of Unity. But you can still use it in *old* versions of Unity but already setup your migration path this way. 

The most interesting of course is `OpenXRSpatialGraphCoordinateSystemSetter`, which is surprisingly simple:
```csharp
public class OpenXRSpatialGraphCoordinateSystemSetter : SpatialGraphCoordinateSystemSetter
{
    protected override void UpdateLocation(Guid spatialGraphNodeId, 
                                           float physicalSideLength)
    {
        var node = spatialGraphNodeId != Guid.Empty ? 
           SpatialGraphNode.FromStaticNodeId(spatialGraphNodeId) : null;
        if (node != null && node.TryLocate(FrameTime.OnUpdate, out Pose pose))
        {
            if (CameraCache.Main.transform.parent != null)
            {
                pose = pose.GetTransformedBy(CameraCache.Main.transform.parent);
            }
            MovePoseToCenter(pose,physicalSideLength);
        }
        else
        {
            PositionAcquisitionFailed?.Invoke(this, null);
        }
    }
}
```
This I mainly nicked from [this new Microsoft sample](https://github.com/yl-msft/QRTracking/blob/main/SampleQRCodes/Assets/Scripts/SpatialGraphNodeTracker.cs). As you can see - hand waving with native coordinate system pointers and complex transformation calculations are no long necessary, which is a big plush IMHO. But it still give the top left point of the QR code, which I still find not logical, but my code in `MovePoseToCenter` fixes that.

## Step 11: apply the new fixed scripts

Since `SpatialGraphCoordinateSystemSetter` is now an abstract class, it can no longer be used in a scene, which Unity pretty clearly shows:

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/missingscript.png)

Remove this missing script reference, and replace is by `OpenXRSpatialGraphCoordinateSystemSetter`

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/fixedscript.png)

One level above, we will see a missing script reference in Tracker1

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/missingreference.png)

which will need to be fixed as well by dragging the TrackerHolder object now sporting the `OpenXRSpatialGraphCoordinateSystemSetter` behaviour.

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/fixedreference.png)

And voila, if you deploy this to a HoloLens it will work. It will work exactly as the same - it will show an airplane on top of the QR code that I included in [the first blog post.](https://localjoost.github.io/Reading-QR-codes-with-an-MRTK2-Extension-Service/). The only difference you will see is that the splash screen is bigger and is more close by.

## Step 11: Update QR code related NuGet packages (optional)

For good measure, I also updated the NuGet package needed to read QR codes - which seem to have made *some* progress judging by the version numbers, but I didn't notice any difference, and it worked already before I did, so this is optional.

![](/assets/2021-05-12-Upgrading-reading-and-positioning-QR-codes-with-HoloLens-2-to-Unity-2020-+-OpenVR-plugin/updateqrpackages.png)

## Concluding words
The refactoring of `SpatialGraphCoordinateSystemSetter` might look a bit heavy handed, as it has a lot of code that will only be used in Legacy XR. This is because I made an implementation for the Windows Mixed Reality Plugin as well, which uses a lot more of that code. If have not published that in this article to prevent confusion, but should someone require that code, I will be happy to supply it. 

Once again: the OpenXR plugin is not ready for flight yet, Microsoft clearly states so. But the OpenXR plugin is making exiting progress - and it can successfully read and position QR codes, which is an important feature for an app I am working on. An important page to keep watching is [this one](https://docs.microsoft.com/en-us/windows/mixed-reality/develop/unity/openxr-supported-features#whats-supported), which keeps you informed about the progress of the OpenXR plugin.

You can [pull the completed upgrade from the same project as before, using this branch](https://github.com/LocalJoost/QRCodeService/tree/openxr).