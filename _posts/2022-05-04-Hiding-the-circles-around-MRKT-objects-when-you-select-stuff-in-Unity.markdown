---
layout: post
title: Hiding the circles around MRKT objects when you select stuff in Unity
date: 2022-05-04T00:00:00.0000000+02:00
categories: []
tags: []
featuredImageUrl: https://LocalJoost.github.io/assets/2022-05-04-Hiding-the-circles-around-MRKT-objects-when-you-select-stuff-in-Unity/screenshot1.png
comment_issue_id: 416
---
A *very* short Unity tip this time. Something that has bothered me for quite some time, but never enough to actually find out what the hell it was - until today. And probably something every Unity buff will guffaw about, because I didn't know. Well I don't care looking stupid, and now it's documented for everyone to find :)

If you ever have created something using the MRTK, chances are you will have seen something like this when you select a few objects or an object with lots of objects in it, specifically objects you can interact with:

![](/assets/2022-05-04-Hiding-the-circles-around-MRKT-objects-when-you-select-stuff-in-Unity/screenshot1.png)

A huge number of circles appear, and until shortly I had not idea what they mean. They are some sort of gizmo, but messing around with this had no effect

![](/assets/2022-05-04-Hiding-the-circles-around-MRKT-objects-when-you-select-stuff-in-Unity/gizmo.png)

But what they certainly achieve is creating a lot of visual clutter, and making the stuff you are trying to manipulate quite a lot less visible.

It turns out I was in the right menu. These are, as it turns out, graphical depictions of *AudioSource ranges*. You have to click the Gizmos menu, scroll down - and I mean *really really down*, until you reach "Built-in components"

![](/assets/2022-05-04-Hiding-the-circles-around-MRKT-objects-when-you-select-stuff-in-Unity/screenshot2.png)

Where you can disable the "Audio Source" checkbox. And then all this visual clutter is gone as you can see. 

Of course, in some situations it makes sense to visualize AudioSource ranges. But clearly Unity builders did not take the MRKT into account, where basically every element has it's own little AudioSource for providing the necessary audio feedback. I think this is an option that should be off by default. But anyway, this way I could get rid of this annoying clutter. Finally.
