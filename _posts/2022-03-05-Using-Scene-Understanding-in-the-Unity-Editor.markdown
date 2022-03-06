---
layout: post
title: Using Scene Understanding in the Unity Editor
date: 2022-03-05T00:00:00.0000000+01:00
categories: []
tags:
- HoloLens2
- MRTK2
- Unity3d
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2022-03-05-Using-Scene-Understanding-in-the-Unity-Editor/sceneunderstanding1.png
comment_issue_id: 410
---
A short tip this time, but nevertheless one that merits its own post. 

Scene Understanding is an awesome feature, but it would be nice if you could use a scan from an real environment *in the editor*, just like you can do with a spatial mesh, like [I described in early 2017](https://localjoost.github.io/using-hololens-scanned-room-inside-your/). Developing and testing in the Unity Editor is way faster than deploying an app to the HoloLens every time, and is especially desirable if testing needs an environment you don't have access to all the time (like a client's factory or a ship).

This is perfectly possible. And in fact, attentive readers might have seen something peculiar in a screenshot in my [previous post about Scene Understanding](https://localjoost.github.io/Making-Surface-Magnetism-work-on-specific-surfaces-by-sorting-Scene-Understanding-surfaces-into-separate-layers/). 

![](/assets/2022-03-05-Using-Scene-Understanding-in-the-Unity-Editor/sceneunderstanding1.png)

That, in fact, is a Scene Understanding scan of my home office.


## How to obtain a Scene Understanding scan for use in the editor

1. Clone the [Mixed Reality Toolkit from GitHub ](https://github.com/microsoft/MixedRealityToolkit-Unity)
2. Open the project *with Unity 2019.4.x*. I used 2019.4.20f1.
3. Install the Scene Understanding Package using the [MRTK Feature Tool](https://docs.microsoft.com/en-us/windows/mixed-reality/develop/unity/welcome-to-mr-feature-tool), [as explained in my previous post.](https://localjoost.github.io/Making-Surface-Magnetism-work-on-specific-surfaces-by-sorting-Scene-Understanding-surfaces-into-separate-layers/#a-few-minor-things)
4. Open the SceneUnderstandingSample scene. It's in Assets\MRTK\Examples\Experimental\SceneUnderstanding\Scenes
5. Build the project with this scene as default scene 0 and deploy to a HoloLens as usual

![](/assets/2022-03-05-Using-Scene-Understanding-in-the-Unity-Editor/build.png)

6. Start the app MixedRealityToolkit on your HoloLens, accept all consent popups. 

![](/assets/2022-03-05-Using-Scene-Understanding-in-the-Unity-Editor/screen.png)

7. Turn on all Surface types (World Mesh is default off).
8. Select "Get Quads".
9. Tap "Save Scene".
10. Connect the HoloLens to your computer and open the device portal.
11. Using the File Explorer (in Views/System), browse to LocalAppData\Microsoft.MixedReality.Toolkit_2.8.0.0_arm64__xxxxxxxxxxxxx\LocalState. the x-es stand for the package name the app got assigned. This may vary.
12. You will see a file a couple of files:

![](/assets/2022-03-05-Using-Scene-Understanding-in-the-Unity-Editor/bytes.png)

And the Scene_[date]_[time].bytes file or files is what you are looking for. Copy this file to your computer, drag it in your project, drag it into the "Serialized Scene" field in the Windows Scene Understanding Observer. Press play. 

And presto.

![](/assets/2022-03-05-Using-Scene-Understanding-in-the-Unity-Editor/scan.png)

You can see this result [in the previous post's project](https://github.com/LocalJoost/SpatialUnderstandingDemo). Just run the app in Editor and you will seen green walls and red floors, just like when you deploy the app on a HoloLens.
