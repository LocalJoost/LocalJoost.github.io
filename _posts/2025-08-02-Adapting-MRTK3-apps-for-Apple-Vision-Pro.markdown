---
layout: post
title: Adapting MRTK3 apps for Apple Vision Pro
date: 2025-08-02T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- Unity
- Apple
- Vision Pro
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2025-08-02-Adapting-MRTK3-apps-for-Apple-Vision-Pro/Darthvader.gif
---
"In fair annum of Lady Hopper, the hundred and nineteenth, did Joost of Local - valiant champion of Microsoft, and sworn foe to fruity contraptions - receive with trembling hand a Mac most miniature, and a vision'd orb of Apple make. Forsooth, the priests of Mount Cupertino were appeased, their altar heavy with tributes vast and rare. Then spake Joost's liege lords, their voices thunderous as tempest: “Thou shalt breathe life into MRTK3, that it may dance upon these devices with grace and fire.”

Thus girded he his resolve, and forth did he stride into the mire of code and challenge. And lo! With craft and cunning, he did bend the task to his will, and the deed was wrought!""

Yes, I have an Apple Vision Pro here. Yes, and a Mac too. It finally happened.

![Darthvader](/assets/2025-08-02-Adapting-MRTK3-apps-for-Apple-Vision-Pro/Darthvader.gif)

I am sorry, I could not resist. Both devices are my employer's property. Given my proficiency and experience with cross-platform development, and the fact that I already had some experience, I was asked to research how Unity MRTK3 apps can be ported to multiple devices, Vision Pro among them. And since Vision Pro, just like other devices, might be a lifeboat for HoloLens refugees (depending on the application field), I thought it fitting to write a blog about it. That is all. In the end, it's just another Mixed Reality device, like the others I worked with - and in the end, it needs to do my bidding - just like the others did. So let us now dispense with the nonsense and get down to business.

## URP is mandatory

For Vision Pro, the Universal Render Pipeline is mandatory. If your app uses custom legacy internal pipeline shaders, you are in for a world of pain, although Unity offers some internal tools for that, and if they fail, Gemini 2.5 Pro is *pretty* good at translating them. However, the Quest Cube Bouncer was already URP, so I did not have that problem.

## Starting point

I took the newest version of [CubeBouncer that I wrote about in November 2024](https://localjoost.github.io/CubeBouncer-revisited-setting-up-Mixed-Reality-for-Quest-3-with-MRTK3-and-Unity-6/) to test out the Quest 3 Mixed Reality capabilities, opened that in Unity 6000.0.49f1, and proceeded to hack out the Meta stuff again:

* Remove the QuestSettings from the main scene

![removeruk](/assets/2025-08-02-Adapting-MRTK3-apps-for-Apple-Vision-Pro/removeruk.png)

* Enable HologramCollection in the scene
* Disable MenuContent-Canvas in HandMenu

![Handmenu](/assets/2025-08-02-Adapting-MRTK3-apps-for-Apple-Vision-Pro/Handmenu.png)

* Delete the Oculus folder
* Delete prefab OcclusionMeshBuildingBlock from the Prefabs folder
* Delete script RoomMeshDetector from the Scripts folder
* Uninstall all three Meta packages:  
   * Meta XR Core SDK
   * Meta MR Utility SDK
   * Meta XR Audio SDK

This you can still do on a PC. Now, for actually building the app, it's best to move to a Mac.

## Unity settings

If you haven't done so already, you will need to install the VisionOS component:

![UnityVisionOS](/assets/2025-08-02-Adapting-MRTK3-apps-for-Apple-Vision-Pro/UnityVisionOS.png)

And you will need a Unity Pro license as well. Unity, cash-strapped as they always are, figured that if you can pay for a Vision Pro *and* a Mac, you are wealthy enough to fork over some of your apparent surplus cash to them as well 😉.

## Configuring MRTK3

Nothing unusual here. Just make sure a valid MRTK3 profile is selected in the Vision OS tab:

![M R T K3](MRTK3.png)

MRTK3 complains about a whole host of missing stuff - just ignore that; it's no problem.

## Plugin settings

Alas, no OpenXR here; standards are for plebs, not for the Cupertino Crew. Select the Apple visionOS plugin. Fortunately, Unity takes care of (most of the) pain.

![X R Plugin](XRPlugin.png)

## Install polyspatial packages

Search for "polyspatial" in the Unity registry and install these two packages. If you don't have at least a Pro license, you will get a warning and they will be uninstalled again.

![polyspatial packages](polyspatial_packages.png)

## Fix settings

Go to Project Validation and select Fix all.

![fixall](/assets/2025-08-02-Adapting-MRTK3-apps-for-Apple-Vision-Pro/fixall.png)

# Vision OS settings

Make sure all the settings in this panel are as they are in the image. Most likely they will be like this, but you will need to fill in "Hand tracking Usage Description" and "World Sensing Usage Description". "World Sensing" is Apple-lingo for what the rest of the world calls Spatial Meshing or Spatial Mapping.

![visionossettings](/assets/2025-08-02-Adapting-MRTK3-apps-for-Apple-Vision-Pro/visionossettings.png)

## Where are we now?

Well wouldn't you know it - if you deploy the app now, it will actually run, show blocks, and you can swipe them away with your hands too. However, there are some things that still don't work. First of all, you will notice that the app does ask if it can use hand tracking, but does not ask for World Sensing. Second, if you swipe the block to a wall or floor, it will go right through it. Third, the hand menu does not appear. The World Sensing one is easy to fix; the hand menu is a bit tricky.

## Getting World Sensing aka Spatial Mapping to work

Although I think this is more Unity's doing than Apple's, this actually works the 'standard' way, [using an ARMeshManager, which I already described for HoloLens 2 over two years ago](https://localjoost.github.io/Using-ARMeshManager-for-Spatial-Awareness-with-MRTK3-on-HoloLens-2/). Make sure it's on the Camera Offset of the MRTK XR Rig, as shown here:

![armeshmanager](/assets/2025-08-02-Adapting-MRTK3-apps-for-Apple-Vision-Pro/armeshmanager.png)

The SpatialMeshPrefab comes directly from HoloLens; it uses the standard VR/SpatialMapping/Occlusion shader as material.

![vrmaterial](/assets/2025-08-02-Adapting-MRTK3-apps-for-Apple-Vision-Pro/vrmaterial.png)

See the [demo project](https://github.com/LocalJoost/QuestBouncer/tree/AppleVisionPro) for details.

## Getting the hand menu to work

The Unity package has a bug, deficiency, or whatever you want to call it that prevents MRTK3 near and far interactions from working properly. The plugin apparently *does* provide joint tracking - but does not provide a *palm pose*, which breaks all interactions. For that to work, you will have to make a change deep in `VisionOSHandProvider`. This is the `com.unity.xr.visionos-2.2.4` package. A rough fix is to use the wrist pose, which is pretty close by in any case:

```csharp
// use wrist Pose for palm pose
if (jointID == XRHandJointID.Palm)
{
    trackingState = XRHandJointTrackingState.Pose;
    jointArray[index] = CreateJoint(handedness, trackingState, jointID, pose);
    var appleTrackingStates = VisionOSHandExtensions.GetVisionOSTrackingStates(handedness);
    appleTrackingStates[index] = true;
    var appleRotations = VisionOSHandExtensions.GetVisionOSRotations(handedness);
    appleRotations[index] = wristRotation;                    
    pose = rootPose;
}
```

It is quite a hassle. First, you will need to download the tgz from Unity, then fish out the file, make the change, put it in the tgz again, then make a link to that local file again.

Or, you can just nick the fixed file from my project. In the Packages folder, you will find **com.unity.xr.visionos-2.2.4-patched.tgz** with the patched `VisionOSHandProvider`. Also, you will see that in the manifest.json of my project, instead of

```json
"com.unity.xr.visionos" : "2.2.4"
```

it says

```json
"com.unity.xr.visionos" : "file:com.unity.xr.visionos-2.2.4-patched.tgz",
```

However, the actual patched file is also in the sources - that is `VisionOSHandProvider.cs` in the project root - should you so be inclined to study it (or improve it).

And then... it works. It all just works. 

<iframe width="768" height="480" src="/assets/2025-08-02-Adapting-MRTK3-apps-for-Apple-Vision-Pro/CubeBounceAVP.mp4" 
frameborder="0" allowfullscreen></iframe>


As you can see, it's still called QuestCubeCouncer, so it probably won't pass the Apple Store, but that's not the point.

## Acknowledgements

There is a wisdom that says it takes a village to raise a child; likewise, I would like to say it often takes a community of wise people to gather true wisdom. First, I would like to express thanks to my awesome friend, fellow MVP, and (alas) former colleague [Alexander Meijers](https://www.linkedin.com/in/alexandermeijers/) who initiated me in the very basics of Apple Vision Pro development [some eight months ago](https://www.linkedin.com/posts/joostvanschaik_apple-visionpro-mrtk3-activity-7268297115359928320-Bepf?utm_source=share&utm_medium=member_desktop&rcm=ACoAAABj6_sBYdWHIQTMX67hYiivEva2MtavpaE), allowing me to get a feel for what is necessary, which gave me a running start when the device finally arrived at my doorstep.

I also am very thankful to [Guillaume Vauclin from EADS, France](https://www.linkedin.com/in/guillaume-vauclin-51592021/). On May 12, 2025, he contacted me via LinkedIn, asking if I could help him get off the ground with MRTK3 apps for Vision Pro. I did not have a device nor a Mac and could not quite remember what we had done in November, but fortunately, Alexander still had the project on his Mac, zipped it up, and made it available to both me and Guillaume. I told Guillaume about the limitations: no decent hand tracking. On May 23, Guillaume came back to me: he had used my sample to get off the ground but also found a solution for the hand tracking, which he shared back to me: the patch to `VisionOSHandProvider.cs`. Which I could use to my great advantage a few months later, and now share with you, with Guillaume's permission.

## Concluding words

What the acknowledgements show, my friends, is why sharing developer knowledge with the community is so vitally important. *Sometimes the community starts giving back*, otherwise known as "what comes around goes around". So now I have a full MRTK3 app working with hand tracking, spatial mapping, and full interaction, and the knowledge I acquired working with HoloLens is also usable on Apple Vision Pro, after it already proved to be usable on Magic Leap 2 and Quest 3.

To end in the same style as I started 😉:

![force](/assets/2025-08-02-Adapting-MRTK3-apps-for-Apple-Vision-Pro/force.png)

The demo project for Apple Vision Pro [can be found here](https://github.com/LocalJoost/QuestBouncer/tree/AppleVisionPro).