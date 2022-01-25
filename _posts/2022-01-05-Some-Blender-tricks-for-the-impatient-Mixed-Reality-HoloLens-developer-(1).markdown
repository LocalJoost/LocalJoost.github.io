---
layout: post
title: Some Blender tricks for the impatient Mixed Reality / HoloLens developer (1)
date: 2022-01-05T00:00:00.0000000+01:00
categories: []
tags:
- HoloLens2
- Mixed Reality
- Blender
featuredImageUrl: https://LocalJoost.github.io/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/blenderstart.png
comment_issue_id: 401
---
Ideally, when you are working on HoloLens or other Mixed Reality projects, you have a full time designer on call in your team handling all the nastiness around models, and who knows how to handle Maya or some other high end design tool - preferably with valid licenses. Otherwise, you are in the same boat as me. You simply download models, and change them to your need *yourself*, using [Blender](https://www.blender.org/).

Blender can do everything and the kitchen sink, is open source *and* free. Beggars can't be choosers. But if you are like me - a developer set on reaching goals, and not a graphic artist *at all*, Blender's UI looks like a Rube Goldberg machine crashed into a wall at high speed. There's little buttons *everywhere*, with no apparent logical grouping based on task or flow. In addition, most of the tutorials are long videos by well meaning designers babbling endlessly while pressing hot keys 'everyone knows' without explaining what keys they are, and since the UI was recently redesigned, most of it is outdated, too.

I don't aspire to become a graphic artist or to get to understand Blender on a deep level, I just want to do some basic stuff. If you are like me - a developer just needing to do some small things with models - this article is for you. I am just explaining a few simple how-tos, without magic keys - step by step. There will be more, later.

For this blog post, I am going to use a sample model from my HoloLens app - a helicopter - that had several problems, which I am going to repair with Blender. Blender 3.0, that is. If you are wondering why every chapter has "in Blender" behind it - that's to make it easier to find by search engines. I have spent too much time searching for stuff. 

## Some definitions first
When Blender opens, it shows like this:

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/blenderstart.png)

I have no idea how these things are actually called in Blender lingo, but in my mind I call them this like in the image. If you are coming from Unity, like me, things on the right and the middle look more or less familiar and I nicked part of the naming too. 
* Mode selector: a drop down almost fully top left that can change between 6 modes. I use only Object Mode and Edit Mode, and have no clue what the rest is for.
* Top menu: if you value your sanity, leave this in Layout
* Edit menu: shows all kinds of functions. Contents depend on Mode Selector. [Aside - two menu's on top of each other - who comes up with things like that?]
* Tool bar: all kinds of functions appear there, depending what is selected in the Top or Edit menu
* Scene - the graphic stuff, like in Unity
* Hierarchy - like in Unity, a tree-like representation of the graphic elements
* Inspector: a rather complex jumble of all kinds of properties of the currently selected object.

## Zooming / rotating/ panning / moving in Blender
* Zoom in/out: mouse wheel
* Rotating: keep mouse wheel pressed and move mouse
* Moving / panning: keep mouse wheel and shift button pressed, and move mouse

## Cleaning out the Blender scene
* Make sure the Mode Selector is in Object Mode
* Click Select/All in the Edit Menu
* Hit the delete key on your keyboard

## Importing an existing model into Blender
This is an easy one: click File/Import in the Edit menu and choose the type. Like I wrote before, I imported an helicopter model - an FBX - I used in AMS HoloATC that has multiple problems. 

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/helicopter.png)

* It's origin is not correct (it's positioned off the center)
* It's rotation is wrong (it should be pointing to the left)
* The left pointing blade of the rotor is incorrectly positioned
* It has two tail rotors
* The pivot point of the rotor is off center (as we will see later)
* The color of the helicopter is all very gray or black.

## Moving an object in Blender
* Make sure the Mode Selector is in Object Mode
* Click Select/All in the Edit Menu
* From the Tool bar (left), select the Move tool. That's this button now marked blue:  

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/movetools.png)

The helicopter will now be adorned with three arrows. 

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/threearrows.png)

Now left-click and hold the green arrow an carefully move the mouse so the helicopter moves to the right, then left-click and hold the red arrow to move it backward till it falls exactly over the origin (the crossing point of the red and green line on the 'floor').

## Rotating an object in Blender
* Make sure the Mode Selector is in Object Mode
* Click Select/All in the Edit Menu
* From the Tool bar (left), select the Rotate tool. That's this button now marked blue:  

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/rotatetool.png)

The helicopter will now be adorned with three circles:

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/rotate.png)

* Blue for yaw (left-right direction the nose is pointing)
* Red for roll (side to side)
* Green for Yaw (up/down direction the nose is pointing)

We want to rotate the nose to the left, so we carefully position the mouse cursor over the blue circle (x marks the spot), press the left mouse button and slowly move the mouse. If you press the CTRL key while doing that, you can snap to small increments and you get this compass rose ring:

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/rotate2.png)

## Deleting part of an object in Blender

Assuming the object consists out of separate parts, this is very easy.
* Make sure the Mode Selector is in Object Mode
* Click the part you want to delete (The right tail rotor)
* Hit the delete button on your keyboard

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/tailrotor.png)

## Changing the pivot point of an object in Blender

The left pointing rotor is pretty badly aligned. We want to rotate it, unfortunately, it's pivot point (the orange dot) is about halfway the blade. 

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/bladepivot.png)

This makes putting it in the right place a cumbersome procedure of moving and rotating. It's easier to change the pivot point first.

* Make sure the Mode Selector is in Object Mode
* Select the rotor blade
* Put the Mode Selector in *Edit Mode*. The view changes a little:

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/editmode.png)

* At the Tool Bar, select the Cursor button. That's this button now marked blue:  

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/cursorbutton.png)

Now carefully zoom in to the point where you want the new pivot point to appear - the location where the rotor is attached to the rotor axis. Click at this location in the middle of the rotor:

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/middlerotor.png)

And you should see the '3D cursor', as it's called, appear in that location.

* Put the Mode Selector *back into Object Mode again*
* From the Edit menu, select Object/Set Origin/Origin to 3D cursor

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/origin.png)

And boom. The pivot point is set.

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/originchanged.png)

Now if you zoom out a little, and select the Rotate tool, you can easily make the rotor look less warped:

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/lesswarped.png)

## Joining multiple objects into one in Blender

We want to make the rotor to be able to spin, but now it's three separate object.

* Make sure the Mode Selector is in Object Mode
* At the Tool Bar, select the select button. That's this button now marked blue:  

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/select.png)

Hold down the shift button, then click all three rotor blades and the center axis:

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/selectall.png)

* From the Edit Menu, select Object/Join

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/join.png)

And presto. Unfortunately, the pivot point is way off again

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/pivotrotor.png)

But I have already explained how you can fix that.

## Changing color of objects in Blender

Nearly the whole helicopter is black. We want to changes this into a more gray color.

* Make sure the Mode Selector is in Object Mode
* In the properties area, select the Material Pane. To that end, on the left side of the scene, almost all the way to the bottom, you should click a button that looks like a small pinkish beach ball

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/materialbutton.png)

This will show something like this in the Properties pane:

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/complexcolors.png)

I always press "Use Nodes" as this makes the UI simpler and colors easier to set. Don't ask me what it means:

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/simplecolors.png)

I have annotated a few things of interest
1. is the actually selected material
2. shows a drop down of all other materials (this thing actually has 4)
3. deletes the current material (not recommended)
4. shows a preview of the current material
5. if you click this, you get a color picker to change the color
6. creates a new material - or actually, a copy of an existing one

So I clicked the color picker (5), clicked "RGB" because I am too simple a man to understand what HSV is, put the slider a little bit up and the whole helicopter turned battleship gray - except for the windows.

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/grey.png)

Now if you want to give for instance the landing gear a different color. Select one of the sled and then you click the button marked 6 - the double pages symbol. This will create material "wire_000000000.001" but you can simply rename it. I renamed it 'sled'

Then I simply started to change to color by moving the mouse cursor over the color picker and the sled of the landing gear became a bit purplish
![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/colorgear.png)

If you want to have the other sled to have the same color, simply select the other sled (I have rotated the helicopter to make that easier), press the dropdown (button I marked 2 a bit above, the white beachball) and select the new material 

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/otherslde.png)

## Turning stuff off temporarily in Blender

Sometimes it's particularly hard to select one part of the object you want to change or delete. That's when the hierarchy window comes in handy. Consider this problem: there's stuff from the inside sticking out of the helicopter. I have selected the fuselage of the helicopter. That part is, according to the Hierarchy window, called "Box001"

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/fuselage1.png)

There is a little eye icon next to it. If you click that, it turns the whole fuselage invisible, without deleting it. This shows the person who designed this helicopter, apparently found it necessary to design part of the interior as well. 

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/fuselage2.png)

We can now safely delete this without having to worry about the fuselage:

![](/assets/2022-01-05-Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(1)/fuselage3.png)

And then turn on the fuselage again with the eye-icon in the hierarchy.

## Conclusion

There's more stuff I could share, but this is a nice start to get you going doing some basic stuff in Blender. This is as good as my notebook as your source of information. If you would like to see the files I played with during this post or maybe try to do this yourself, [you can download them here.](https://github.com/LocalJoost/HeliSample)