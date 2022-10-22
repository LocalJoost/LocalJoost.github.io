---
layout: post
title: 'Fix: after setting up a new MRTK3 project Newtonsoft.Json is missing'
date: 2022-10-22T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- HoloLens2
- Unity
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2022-10-22-Fix-after-setting-up-a-new-MRTK3-project-NewtonsoftJson-is-missing/dataval.png
comment_issue_id: 431
---
If you are like me, you never even think about it. Newtonsoft.Json is like the hammer in your toolbox, the knife in your kitchen drawer: it's always there in MRTK projects - and it's a long time since I even thought about that. Until I recently set up a new MRTK3 project (pre-11) and it was not. Why?

## That's why
My initial thought was a breaking change in the MRTK3 dependency chain, but I was wrong. You see, apparently the *only* MRKT3 module - or feature as the MRTK Feature tool calls it - that has a dependency on Newtonsoft.Json is the "MRTK Data Binding and Theming Early Preview" (com.microsoft.mrtk.data).

![](/assets/2022-10-22-Fix-after-setting-up-a-new-MRTK3-project-NewtonsoftJson-is-missing/dataval.png)

The MRTK Feature Tool actually tells you so, if you click the blue "(Details)" link and scroll down:

![](/assets/2022-10-22-Fix-after-setting-up-a-new-MRTK3-project-NewtonsoftJson-is-missing/details.png)

This wisdom you can also find in the generated file packages-lock.json in the packages folder of your product, after you installed the "MRTK Data Binding and Theming Early Preview"

![](/assets/2022-10-22-Fix-after-setting-up-a-new-MRTK3-project-NewtonsoftJson-is-missing/packlock.png)

This is the only occurrence of the com.unity.nuget.newtonsoft-json package. So, 
Long story short: if you don't install "MRTK Data Binding and Theming Early Preview" package, you don't get com.unity.nuget.newtonsoft-json and you can - just like me - spend an aggravating fifteen minutes trying to think why the Newtonsoft.Json namespace does not get resolved in your code.

## Solution 1

Install "MRTK Data Binding and Theming Early Preview", even if you are not using it. Done. A bit of a heavy-handed approach, but it works.

## Solution 2

If you don't plan on using the "MRTK Data Binding and Theming Early Preview", the most efficient way, of course, is to install the Newtonsoft.Json yourself. This is where the fun begins. Where Visual Studio has NuGet, Unity has the package manager. I sometimes think the Unity Package Manager is especially designed to drive developers insane. First of all, you have to give a scope where to search:

![](/assets/2022-10-22-Fix-after-setting-up-a-new-MRTK3-project-NewtonsoftJson-is-missing/packman.png)

Why there no "everywhere" option for search, you ask? Yeah, beats me too. But now comes the real kicker: if you search in all four options, *none of them yield anything when you search for NewtonSoft.Json*.

So in the end I got sick and tired of it, and manually added one line to the dependencies in the manifest.json (also in your project's Packages folder) to see if that would work.

![](/assets/2022-10-22-Fix-after-setting-up-a-new-MRTK3-project-NewtonsoftJson-is-missing/add.png)

And lo and behold, *now* it appears in the package manager and it even helpfully tells you there's a newer version:

![](/assets/2022-10-22-Fix-after-setting-up-a-new-MRTK3-project-NewtonsoftJson-is-missing/packman2.png)

## Concluding words

Maybe I am missing something here, but I have not found a more intuitive way to do this (or any other way, for what matters). I love Unity, but sometimes things are very illogical or hard to find. Anyway, no project to go with this blog as there is no code to share.
