---
layout: post
title: 'Fix: Dark background when taking pictures or making movies with HoloLens 2 and OpenXR'
date: 2021-12-08T00:00:00.0000000+01:00
categories: []
tags:
- HoloLens2
- MRKT2
- Unity3d
- Windows Mixed Reality
- OpenXR
featuredImageUrl: https://LocalJoost.github.io/assets/2021-12-08-Fix-Dark-background-when-taking-pictures-or-making-movies-with-HoloLens-2-and-OpenXR/dark.png
comment_issue_id: 396
---
## The problem 
A short tip this time. At [Velicus](https://velicus.nl), we are currently working on two apps. One is built with Unity 2019.4 LTS, the other with Unity 2020.3 LTS. The latter uses the the OpenXR plugin. Last week, after hearing complaints from a customer about dark backgrounds when shooting movies of HoloLens apps, I decided to take a look at the latter app, and saw the following at our login screen:

![](/assets/2021-12-08-Fix-Dark-background-when-taking-pictures-or-making-movies-with-HoloLens-2-and-OpenXR/dark.png)

Now compare that to the login screen of our other app:

![](/assets/2021-12-08-Fix-Dark-background-when-taking-pictures-or-making-movies-with-HoloLens-2-and-OpenXR/light.png)

I went back to earlier pictures and movies of the OpenXR based app, and on that one the background was as clear as the other app. So what had changed? We did not upgrade Unity or anything else in the app. We just wrote code and added assets.

## The solution
The culprit turned out to be the Mixed Reality OpenXR plugin:

![](/assets/2021-12-08-Fix-Dark-background-when-taking-pictures-or-making-movies-with-HoloLens-2-and-OpenXR/MRTKFT.png)

In our app, this was set to version 1.0.2, about the first release version of the OpenXR plugin. Once I set this to the latest release version (currently 1.2.1), the background was clear again on pictures and movies. I think (but am not sure) this has something to do with the Open XR *Runtime* on the HoloLens being a separate thing that can be updated out of bounds, like an app, without updating the HoloLens as a whole.

![](/assets/2021-12-08-Fix-Dark-background-when-taking-pictures-or-making-movies-with-HoloLens-2-and-OpenXR/runtime.png)

Maybe something got out of sync between the older plugin and the runtime. However, this is what fixed it. My fellow MVP [Ivana Tilca](https://twitter.com/ivanatilca) pointed me a bit later to [this post in the HoloDevelopers Slack channel](https://holodevelopers.slack.com/archives/C1CQKRQM6/p1638542176458200?thread_ts=1637888306.362900&cid=C1CQKRQM6) where the issue is actually mentioned as an aside in an answer about Azure Remote Rendering. I decided to write a short post about it to make this fix easier to find for other people struggling with it.
