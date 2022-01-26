---
layout: post
title: Some Blender tricks for the impatient Mixed Reality / HoloLens developer (2)
date: 2022-01-26T00:00:00.0000000+01:00
categories: []
tags: []
featuredImageUrl: https://LocalJoost.github.io/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/onegroup.png
comment_issue_id: 402
---
[As I wrote in my previous post ](https://localjoost.github.io/Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/)about this subject - a Mixed Reality / HoloLens developer sometimes just needs to tinker a bit with models, but doesn't have the time nor the desire to become a fully fledged graphic artist. Finding out how to do a few basic things in [Blender](https://www.blender.org/) - a free 3D tool - is quite a challenge. So I compiled a few (more) things I learned.

## Importing a WaveFront Object File and keeping individual components in Blender

Blender sometimes is a bit too helpful. When you import for instance the model of a Cessna 152 that I use in this blog post, by default it imports it as one object:

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/onegroup.png)

which might be desirable, but not when you want to further edit or animate parts of the model. In that case, it's better to use this option that's cleverly hidden in the import dialog:

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/splitbyobject.png)

In which case you will see the object being imported like this:

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/splitbygroup.png)

## Scaling down huge objects or objects with incorrect units with Blender

Note: for the whole procedure Blender needs to be in ["object mode"](https://localjoost.github.io/Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/#some-definitions-first). 

For some reasons, some objects are excessively huge. I am not sure, but I think this has something to with modelers using incorrect unit settings, or apps interpreting the unit settings wrong. Anyway, I got this WaveFront Object (.obj) files that clearly contained something, but did not show anything

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/nothingtosee.png)

For some reason Blender seems to think (I later found out) the model has a wingspan of about 18.3 *kilometer*, which (\*checks notes\*) is not quite according to manufacturer's specs. Unfortunately, Blender gets a bit confused by this, and basically shows nothing, unless you zoom out *very* far, and then still only you see parts of the plane 'flash by'. However, as we have seen, in the hierarchy it shows this, so clearly *something* is imported:

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/splitbygroup.png)

Select the *bottom* one (Group_014). Now will you see the vertical button bar on the right side of your scene view expand, and you will notice a button with an orange icon that looks like a block in brackets

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/objectprops.png)

Clicking it shows the "object properties". If you expand the "transform" pane

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/transformpane.png)

you will notice the "Scale X", "Y" and "Z" properties. I started entering lower and lower scales into these values, until suddenly, at scale 1:1000 (0.001):

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/firstplane.png)

Now we are getting somewhere. However, if you rotate the view forward (move mouse while holding scroll wheel) you will see it's still a bit big for a this kind of airplane (one square on the grid is 1x1 m)  

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/forward1.png)

Click the round thingy the red arrow points to you get a 'front orthographic' view you get a nice view dead ahead which a clear grid. 

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/frontortho.png)

If you count the blocks, this shows the wingspan is about 18 meters now. But [according to Wikipedia](https://en.wikipedia.org/wiki/Cessna_152) the wingspan of a Cessna 152 is 10.16 meters. This means the scale needs to be 10.16 / 18 * 0.001 = 0.000564 and indeed, if we enter in all three scale fields, and look from the top:

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/orthorightsize.png)

The wingspan is about 10 and a bit meters now, so this is good enough. Note: Blender only shows only *four* decimals in the UI, however, it stores all of them, and you can see that when you click a field again:

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/decimals.png)

So now if you select all the groups of the airplane one by one and enter 0.000564 in all three scale fields of every component the whole plane appears, correctly scaled (I haven't found a way to do this in one go for all groups yet).

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/correctlyscaled.png)

Although not strictly necessary if you just want to export the model right now, the final step is to actually *apply* the scaling, so the individual object measurements make sense. This, fortunately, we can do for all components at once. The procedure is as follows:
* Make sure the mode selector top left is in "Object Mode"
* Click Select/All
* Click Object/Apply/Scale

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/applyscale.png)

You will see the scale number jump back to 1.000 again but the airplane model keeps the same apparent size.

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/scaleback.png)

## Showing the number of polygons in Blender

Without going into details: 3D models exist out of polygons. Simply put: more polygons is more detail, less polygons is better performance. Especially when I [recently dabbled in WebXR](https://holoatcbf3a.z6.web.core.windows.net/) I got into trouble pretty fast as my models appeared to have a lot op polygons. In Blender, you can see the model's complexity by looking at the *scene statistics*. These are, for some reason, off by default - and very cleverly hidden, too. If you right-click at the  nearly empty bar at the very bottom of the screen, like here:

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/rightclick.png)

A little popup shows up. The nearly completely useless option showing Blender version is on by default, and the very useful option "Scene Statistics" is off. Go figure. Anyway, if you click the check box left of "Scene Statistics" you get a set of numbers. The Verts number (short for vertices - the sides of polygons) tells you something about the model's complexity. 

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/statistics.png)

## Reducing the number of polygons in Blender

I am not quite versed in the details of models or modeling, but like I wrote above: from a performance standpoint, less is better. So the trick is to reduce the vertices numbers as much as you can, without making the model ugly. 

The procedure is as follows:
* Make sure the mode selector is in "Object Mode"
* Select all objects by clicking Select/All
* Put the mode selector in "Edit Mode"
* Click Mesh/Clean Up/Decimate Geometry

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/decimate.png)

This will open an almost hidden UI at the left bottom of the screen:

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/closeddecimation.png)

If you click the > sign, it will open a small panel: 

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/decimateopen.png)

Decimating now is a matter of making the number smaller than 1. Then it's just a matter of trying. If you make the number 0.01, that's clearly too much:

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/toomuch.png)

The number of vertices is very low, but the airplane now looks like kind of impressionist's view of an airplane. However if you make the number to 0.4:

![](/assets/2022-01-26-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/decimatedokay.png)

the airplane pretty much looks like itself, but now we only have 5520 vertices in stead of the 14415 we started with. This might have a significant effect on the performance of your app, depending on how much of these airplanes are in your app. There are some artifacts now, especially around the landing gear, but since we imported the groups separately, these all individual components. So, you could opt for decimating *only specific components*, and for instance exclude the landing gear from decimation. It takes some fiddling to see how far you can go. 

## Conclusion

The only thing now that is left to do is change the pivot point of the propeller - that is, if you are planning on animating that in Unity, for instance but [I already showed how to do that in my previous post](https://localjoost.github.io/Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/#changing-the-pivot-point-of-an-object-in-blender) so I leave that as an exercise for the reader ;)

You can [download the model and the files that I used in this blog here](https://github.com/LocalJoost/CessnaSample).
