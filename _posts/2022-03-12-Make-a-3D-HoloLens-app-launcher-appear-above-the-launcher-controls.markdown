---
layout: post
title: Make a 3D HoloLens app launcher appear above the launcher controls
date: 2022-03-12T00:00:00.0000000+01:00
categories: []
tags:
- HoloLens2
- Mixed Reality
- Blender
- MRTK
featuredImageUrl: https://LocalJoost.github.io/assets/2022-03-12-Make-a-3D-HoloLens-app-launcher-appear-above-the-launcher-controls/mrtk1.png
comment_issue_id: 412
---
A very, very small tip this time. In HoloLens 2 - or actually Windows Mixed Reality - you can create these awesome 3D launchers that look a lot better than the default floating windows. These can be simple glb files, that you can create with [Blender](https://www.blender.org/). Have a look at this beautiful logo: that's from the MRTK demo app itself.

![](/assets/2022-03-12-Make-a-3D-HoloLens-app-launcher-appear-above-the-launcher-controls/mrtk1.png)

Also, since Unity 2020 or some version of the MRTK - I am not quite sure what made it happen - you don't have to add manually edit the Package.appxmanifest in the generated C++ solution, you can just assign the logo in Unity:

![](/assets/2022-03-12-Make-a-3D-HoloLens-app-launcher-appear-above-the-launcher-controls/assigninunity.png)

But there's something that annoys me - if you look at the MRTK logo you can see the bottom controls (right pointing arrow, slanted rectangle and X) are being projected *over* the bottom of the logo. Smart *ss that I was, I tried to import the logo into Blender and move it up a little from the global origin

![](/assets/2022-03-12-Make-a-3D-HoloLens-app-launcher-appear-above-the-launcher-controls/blender1.png)

Of course, that did not work. My 2nd attempt was a bit smarter: I [joined all the parts together](https://localjoost.github.io/Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/#joining-multiple-objects-into-one-in-blender) and [moved the resulting object's pivot point](https://localjoost.github.io/Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/#joining-multiple-objects-into-one-in-blender) down using the 'world cursor':

![](/assets/2022-03-12-Make-a-3D-HoloLens-app-launcher-appear-above-the-launcher-controls/blender2.png)

Nope.

What *does* work, is my kinda stupid hack: draw a minute cube (0.001 x 0.001 x 0.001) a bit below the center of the actual logo, like so: 

![](/assets/2022-03-12-Make-a-3D-HoloLens-app-launcher-appear-above-the-launcher-controls/blender3.png)

Tadaaa! 

![](/assets/2022-03-12-Make-a-3D-HoloLens-app-launcher-appear-above-the-launcher-controls/mrtk2.png)

The cube is so small is stays invisible, but it stretches the object area far enough to pull the controls down below the actual logo. The right distance between logo and the invisible cube depends on the shape and size of the actual logo - you can find this with a bit of fiddling. Unfortunately every test requires a deployment to HoloLens, so best try this with an as small as possible test app.

There are two important details to observe. First and foremost: the cube, and basically *every* element in the logo, *must have a material*. Default your cube has no material, which will make the import in Unity (by the MRTK) fail. 

If you have created the cube in the right spot, select the little beach ball icon:

![](/assets/2022-03-12-Make-a-3D-HoloLens-app-launcher-appear-above-the-launcher-controls/materialicon.png)

hit the large "New" button shown at the bottom of this image below:

![](/assets/2022-03-12-Make-a-3D-HoloLens-app-launcher-appear-above-the-launcher-controls/new.png)

And then it will show this, and you are done. You don't have to change any material settings, as the cube is so small you won't see in anyway.

![](/assets/2022-03-12-Make-a-3D-HoloLens-app-launcher-appear-above-the-launcher-controls/material.png)

The second important thing to observe is that the entire logo file *may not contain more than 10000 triangles*. It says so here in [this blog post by Microsoft](https://docs.microsoft.com/en-us/windows/mixed-reality/distribute/creating-3d-models-for-use-in-the-windows-mixed-reality-home#triangle-counts-and-levels-of-detail-lods?WT.mc_id=WDIT-MVP-4034925):

> The Windows Mixed Reality home doesn't support models with more than 10,000 triangles.

and I have empirically ascertained this is true. 10000 triangles is really not that much, so you can't just plonk in any model, and if you try to decimate it using Blender it will probably end up looking very bad. So best find a skilled modeler, explain the limits, and have them made something "low poly" that still looks nice.

No repo this time, as there is no code or any other useful file to share.