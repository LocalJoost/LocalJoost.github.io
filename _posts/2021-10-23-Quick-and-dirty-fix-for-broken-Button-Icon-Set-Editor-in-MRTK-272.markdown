---
layout: post
title: Quick and dirty fix for broken Button Icon Set Editor in MRTK 2.7.2
date: 2021-10-23T00:00:00.0000000+02:00
categories: []
tags:
- HoloLens2
- MRTK2
- Unity3d
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2021-10-23-Quick-and-dirty-fix-for-broken-Button-Icon-Set-Editor-in-MRTK-272/error.png
comment_issue_id: 392
---
People who, like me, use the heck out of Mixed Reality Toolkit as the base for all their Mixed Reality projects and dutifully upgraded to the latest version - 2.7.2 - may have run into a nasty surprise when they try to create or edit a button icon set in Unity 2020:

![](/assets/2021-10-23-Quick-and-dirty-fix-for-broken-Button-Icon-Set-Editor-in-MRTK-272/error.png)

If you know where to look for, you can find it mentioned in [issue 9999 on the MRTK GitHub repo](https://github.com/microsoft/MixedRealityToolkit-Unity/pull/9999) with the added comment "Unfortunately this PR wasn't included in the 2.7.2 release (it barely missed the window). We'll be sure to get it in for the next one!"

That was in July 22, 2021 and since then no new version of the MRTK have been released - 2.7.2 is still the current stable version of of today according to the MRTK Feature Tool:

![](/assets/2021-10-23-Quick-and-dirty-fix-for-broken-Button-Icon-Set-Editor-in-MRTK-272/featuretool.png)

So three months later, the stable version still has a broken Button Icon Set Editor. You have two options: either go for a development version of the MRTK and pull it in as source, which I would recommend for curious persons, but not necessarily for production releases - or you follow this dirty little band aid trick:
* [Click here](https://raw.githubusercontent.com/microsoft/MixedRealityToolkit-Unity/f9644bae57958f6e297cf38fa668476779aef14a/Assets/MRTK/SDK/Features/UX/Scripts/Buttons/ButtonIconSet.cs) to get to the raw version of the fixed file
* Save this file in [yourproject]\Library\PackageCache\com.microsoft.mixedreality.toolkit.foundation@0a1b6508371c-1633336231433\SDK\Features\UX\Scripts\Buttons

And lo and behold:

![](/assets/2021-10-23-Quick-and-dirty-fix-for-broken-Button-Icon-Set-Editor-in-MRTK-272/fixed.png)

*Mind you* this is a band aid, a little fix to be able to edit icons, and nothing more. It won't 'stick' - if you reinstall the MRTK it will be overwritten, and you will have to apply this on every machine you and you co-workers develop on. But it's a way to move forward while still using the stable version of the MRTK - while we wait for an official fix release.