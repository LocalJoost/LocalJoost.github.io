---
layout: post
title: Why a freshly cloned Spectacles projects seems to contain errors in Lens Studio
date: 2025-02-09T00:00:00.0000000+01:00
categories: []
tags:
- Lens Studio
- Spectacles
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2025-02-09-Why-a-freshly-cloned-Spectacles-projects-seems-to-contain-errors-in-Lens-Studio/error.png
comment_issue_id: 485
---
Once again I learned that however clear I think I have explained things, there are always little things I apparently stepped over just too quickly and [left people confused anyway](https://github.com/LocalJoost/CubeBouncerDemoLens/issues/1#issue-2840409461). Here's the thing: if you open a freshly cloned Lens Studio project targeted towards Spectacles, you will immediately be greeted by a number of errors

![error](/assets/2025-02-09-Why-a-freshly-cloned-Spectacles-projects-seems-to-contain-errors-in-Lens-Studio/error.png)

Although the error is clear enough for the *experienced* user; if you are new to Lens Studio this message might be a bit confusing. First you first have to drag around a bit in the UI to enlarge the Logger in the rather busy Lens Studio user interface to actually be able to *read* the message. In addition, you need to understand that Lens Studio was originally targeted towards Snap Chat lenses, that Spectacles projects are more or less bolted upon it, that there's something like different simulation modes, and that you actually have to set that using this pesky little thing that's *all the way down at the right of the UI*, that says "No Simulation":

![nosimulation](/assets/2025-02-09-Why-a-freshly-cloned-Spectacles-projects-seems-to-contain-errors-in-Lens-Studio/nosimulation.png)

You have to set it like this:

![simulation](/assets/2025-02-09-Why-a-freshly-cloned-Spectacles-projects-seems-to-contain-errors-in-Lens-Studio/simulation.png)

And then the issue is gone. 

Lens Studio seems to remember the last simulation setting, but apparently does not store this knowledge in a file that can be committed to source control. Unity has the same annoying behaviour, opening every project default as a Windows Standalone project (unless you pre-select the project type in the Hub, that is your intelligence, not that of the application). Personally I feel this should simply be stored in the Lens Studio project file.