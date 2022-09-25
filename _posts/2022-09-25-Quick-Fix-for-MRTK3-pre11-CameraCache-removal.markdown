---
layout: post
title: Quick Fix for MRTK3 pre.11 CameraCache removal
date: 2022-09-25T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- HoloLens2
- Unity
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2022-09-25-Quick-Fix-for-MRTK3-pre11-CameraCache-removal/thisitheway.png
comment_issue_id: 429
---
A very small and short post for a change. When working on [my previous blog post](https://localjoost.github.io/Running-an-MRTK3-app-on-an-Android-Phone/) I noticed yet another shiny new drop of the MRTK3, I upgraded merrily, and was greeted with a number of syntax errors I had not encountered before. Apparently the trusted old `CameraCache` class, [that has been around since April 2018](https://github.com/microsoft/MixedRealityToolkit-Unity/commits/d5350e87cb8bb81f0274af2aa09b2061f90d2362/Assets/MixedRealityToolkit/_Core/Utilities/Scripts/CameraCache.cs?browsing_rename_history=true&new_path=Assets/MRTK/Core/Utilities/CameraCache.cs&original_branch=releases/2.8.2), was suddenly gone. It has always been 'common knowledge' that calling `Camera.main` directly in Unity resulted in a huge performance hit, making continuous tracking in the Update loop tremendously expensive. You should only do that once, then keep a reference to the camera in a local field. MRTK(2) facilitated this with the `CameraCache` class, basically encapsulating the `Camera` class instance and caching it automatically so you didn't have to worry about that rather common pitfall.

However, [apparently Unity have fixed this performance hit in 2020.2](https://blog.unity.com/technology/new-performance-improvements-in-unity-2020-2), so trusted old `CameraCache` isn't necessary anymore. Consequently, the MRTK team have decided to [remove it from the MRTK3](https://github.com/microsoft/MixedRealityToolkit-Unity/pull/10953) without much further ado. Breaking my code.

When I upgrade to new APIs I usually go in the head-first, taking one app as test case,  go about yanking out the old API, and putting in the new. The first thing I then do is what I call 'making scaffolding code': encapsulating new APIs in old ones that have disappeared or are changed, and bring back stuff that got deleted or is missing. The overarching idea is to at least make apps *compile* again - there will probably be a bazillion syntax errors anyway, but now there will be substantially less. Maybe only half a bazillion ;). In the end, when the app works again, you can always come back to fixing some obsolete calls. In 2019, when the old HoloToolkit was superseded by MRTK (later MRTK2) and the transition broke my code I created [HoloToolkitCompatiblityPack](https://github.com/LocalJoost/HoloToolkitCompatiblityPack) - to do just that: make the hard landing into the new toolkit softer. The first app is where you suffer pain. The next will be a lot easier.

And that's why in the code base of my previous blog you will find [this little class](https://github.com/LocalJoost/MRTK3PhoneInteraction/blob/main/Assets/MRTKExtensions/MRTK2Compatibility/CameraCache.cs):

```csharp
namespace Microsoft.MixedReality.Toolkit
{
    public static class CameraCache
    {
        [Obsolete("CameraCache.Main is deprecated, please use Camera.main instead. Unity have solved the performance hit for that")]
        public static Camera Main => Camera.main;
    }
}
```

It basically does nothing but directly call the camera, yet it is enough to keep the old calls compiling. And it warns the developer politely to consider not calling it anymore.

The MRTK2 -> MTRK3 transition poses enough challenges already as it is; adding in things like this makes it easier. I will probably put out more of these things online, eventually in a single repo, just like last time. Noblesse oblige, as the old English used to say. Or to put it in more contemporary words ;) : 

![](/assets/2022-09-25-Quick-Fix-for-MRTK3-pre11-CameraCache-removal/thisitheway.png)
