---
layout: post
title: 'Short Unity / HoloLens tip: change package name before deploying'
date: 2021-04-02T00:00:00.0000000+02:00
categories: []
tags: []
featuredImageUrl: https://LocalJoost.github.io/assets/2021-04-02-Short-Unity-HoloLens-tip-change-package-name-before-deploying/createproject.png
comment_issue_id: 375
---
Let me first show up where this shows up in Unity: 
* Create a new, empty 3D Unity project
![](/assets/2021-04-02-Short-Unity-HoloLens-tip-change-package-name-before-deploying/createproject.png)
* When it's done creating hit File/Build Settings, select "Universal Windows Platform" and hit the "Switch Platform" button.
![](/assets/2021-04-02-Short-Unity-HoloLens-tip-change-package-name-before-deploying/buildsettings.png)

This will make some settings visible in the Player Settings that are normally disabled or invisible because they are only applicable to the UWP platform. But this is the platform HoloLens and Windows Mixed Reality headset apps run on.

So hit the "Player Settings..." button. Then expand the "Publishing Settings" panel

![](/assets/2021-04-02-Short-Unity-HoloLens-tip-change-package-name-before-deploying/playersettings.png)

Notice the Package name text field. This *always* has the text **Template3D** as value. Now if you generate the C++ solution from this, this will indeed end up in the app manifest.

![](/assets/2021-04-02-Short-Unity-HoloLens-tip-change-package-name-before-deploying/manifest.png)

And here you have a potential problem. All apps you create in Unity this way essentially *get the same package name*. So someone who is not aware of this - or someone who downloads and deploys multipe samples from GitHub made by a somewhat *cough* distracted developer *cough* who sometimes commits files with this default setting, might deploy multiple apps from Visual Studio *with the same package name*. 

Some pretty weird things might happen. I have seen apps simply not show up in the app list - because the update process supposed it was an update to an app already on the HoloLens and put that update in the already deployed app but did not change the name. I also have seen the app appear in the app list, but not working correctly - or not at all. In all cases apps (can) get mangled during the deploy - a pretty undesirable situation. 

The moral of this story: always always always *change the name in the Package name text field* after creating the project. Choose whatever you like, something unique per project, and never, ever commit a project with default values to a GitHub rep. Especially not in a public repo. Unless you to have a lot of weird inexplicable issues opened at your name

Take it from me ;)
