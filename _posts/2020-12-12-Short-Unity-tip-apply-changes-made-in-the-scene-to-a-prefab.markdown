---
layout: post
title: 'Short Unity tip: apply changes made in the scene to a prefab'
date: 2020-12-12T00:00:00.0000000+01:00
categories: []
tags:
- Unity3d
- HoloLens2
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2020-12-12-Short-Unity-tip-apply-changes-made-in-the-scene-to-a-prefab/context.png
comment_issue_id: 369
---
Common problem: you have created a prefab which you want to modify. Of course you can open the prefab and edit that, but when you want to edit a prefab in the context of a scene you can be in for a cumbersome process switching between scene and prefab editing. There are some simple options to make that easier.

## Context
So assume we have a prefab "DemoObject" that has two instances of another prefab, "SphereView". 

![](/assets/2020-12-12-Short-Unity-tip-apply-changes-made-in-the-scene-to-a-prefab/context.png)

## Changing a few things
So I kind of horribly mangled the second sphere. I moved it a little downward - so far so good, but I also changed the color to cheese yellow and changed the X scale so it becomes a very elongated egg.

![](/assets/2020-12-12-Short-Unity-tip-apply-changes-made-in-the-scene-to-a-prefab/changed.png)

## Applying changes to the prefab - method 1

Now if you select the top prefab in the scene, you will see a button "Overrides". If you click that, you will see a kind of pull down that shows you exactly what has changed where

![](/assets/2020-12-12-Short-Unity-tip-apply-changes-made-in-the-scene-to-a-prefab/override1.png)

You can even drill down into the exact changes:

![](/assets/2020-12-12-Short-Unity-tip-apply-changes-made-in-the-scene-to-a-prefab/override2.png)

And if you hit "Apply All" it will push down all changes to the prefab

![](/assets/2020-12-12-Short-Unity-tip-apply-changes-made-in-the-scene-to-a-prefab/apply1.png)

## Applying changes to the prefab - method 2

The previous method was top-down from the prefab down to components, this is the other way round. If you click the three-dots-menu to the changed component whose properties you want to push to the prefab, you can select "Modified Component", which will give you the options displayed in the image below: 

![](/assets/2020-12-12-Short-Unity-tip-apply-changes-made-in-the-scene-to-a-prefab/apply2.png)

In the image I have selected the Mesh Renderer, whose material (and thus color) I have changed. Important is now to understand the difference between both Apply options:
* Apply to Prefab 'SphereView' will make *all* SphereViews yellow
* Apply to Override in Prefab 'DemoObject' will *only* make the bottom SphereView yellow - in the DemoObject prefab 

## Conclusion

I hope these little tips will make your work as an MR developer coming from an enterprise background like myself. 

The second method I found myself, but I have to have extend sincere thanks to my colleague [Timmy Kokke aka @sorskoot](https://twitter.com/sorskoot) for the first method. I really didn't know this. He is not so much a blogger, so I decide to blog this. 

I recommend you watch [his Twitch stream](https://www.twitch.tv/sorskoot), especially if you are into WebXR.
