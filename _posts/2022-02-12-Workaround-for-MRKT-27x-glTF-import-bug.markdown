---
layout: post
title: Workaround for MRTK 2.7.x glTF import bug
date: 2022-02-12T00:00:00.0000000+01:00
categories: []
tags:
- HoloLens2
- MRKT2
- Unity3d
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2022-02-12-Workaround-for-MRKT-27x-glTF-import-bug/cessna1.png
comment_issue_id: 406
---
Not-so-hypothetical situation: your co-worker, who is a designer, sends you a nice glTF or glb file, or you made one yourself after getting inspired by [my last Blender posts](https://localjoost.github.io/Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/). You use this model in one of your HoloLens apps. All nice and fine. 

Until your other co-worker pulls your project from Git, opens the project in Unity, and in stead of the beautifully crafted Cessna 152 looking like this:

![](/assets/2022-02-12-Workaround-for-MRKT-27x-glTF-import-bug/cessna1.png)

It looks like this:

![](/assets/2022-02-12-Workaround-for-MRKT-27x-glTF-import-bug/cessna2.png)

And it appears thus in the HoloLens app as well - looking completely crappy.

## What is going on?

Well, you see: Unity, by itself, does not understand glTF or glb *at all*. You can see this easily for yourself when you try to use a glTF or glb model in a bog standard Unity project, so without the MRTK configured in it. You can't drag in the scene. It's not even recognized as a model. It's 'just a file'. 

What is happening, behind the scenes, is code from the *MRKT* kicking in, doing the import in the background. If you look at the console, you will see a couple of warnings indicating the Mixed Reality Toolkit Standard Shaders are not available:

![](/assets/2022-02-12-Workaround-for-MRKT-27x-glTF-import-bug/warnings.png)

And on closer inspection, you can see the `ConstructGltf` class failing

![](/assets/2022-02-12-Workaround-for-MRKT-27x-glTF-import-bug/fail.png)

What happens, my friends, is pretty easy to explain - the MRKT importer kicks in *before Unity is done importing the MRKT itself*.

## A simple workaround

First, you disable the MRKT glTF asset importer, like this

![](/assets/2022-02-12-Workaround-for-MRKT-27x-glTF-import-bug/disable.png)

Then, you re-import your assets and wait till Unity is done

![](/assets/2022-02-12-Workaround-for-MRKT-27x-glTF-import-bug/reimport.png)

And finally, you enable the MRKT glTF asset importer again.

![](/assets/2022-02-12-Workaround-for-MRKT-27x-glTF-import-bug/enable.png)

Now the importer runs *after* the project and the MRTK itself are properly imported and initialized, and hey presto  
  
![](/assets/2022-02-12-Workaround-for-MRKT-27x-glTF-import-bug/cessna1.png)

## Final thoughts

The fact that the option to enable and disable the importer like this exists at all, makes me think the original developer was kind of aware this problem might occur, but did not know how to tackle it - or just never came around to implement a more robust solution. However, I noticed, when I tried this for the first time, I actually got a change in the ProjectSettings/ProjectSettings.asset

![](/assets/2022-02-12-Workaround-for-MRKT-27x-glTF-import-bug/projectsettings.png)

So *if* your MRTK project contains glTF assets, and the project or branch goes out of active development, I think you might consider committing the ProjectSettings.asset this way (so with the importer disabled) in the final commit, and add a warning in the readme of the project. After all, it's quite likely people will notice a completely *missing* assets and go hunting for a solution. A more or less mangled asset that is noticed too late might cause other, more difficult to fix and/or integrate problems.

This will of course not work during active development, but in that case this problem won't occur so easily. 

As I am writing no code here, there is no project to go with this blog post.
