---
layout: post
title: Removing the 'hiccup' from repeating Unity animations
date: 2022-01-30T00:00:00.0000000+01:00
categories: []
tags:
- Unity
- HoloLens2
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2022-01-30-Removing-the-'hiccup'-from-repeating-Unity-animations/rotation1.gif
comment_issue_id: 403
---
This is one of those trivial things 'everyone' knows - every Unity game developer, that is - except for wierdos like me who come from a enterprise development background and learned Unity as his or her front-end of choice for HoloLens development. 

## 'Hiccuping' rotation

[In November 2018(!) I already wrote](https://localjoost.github.io/adjusting-and-animating-hololensmixed/) how you can create a simple endless animation making - in that particular case - a helicopter's rotor spin. In the video that goes with it you can see I am struggling at this point, because the rotor animates not that smoothly. It seems to 'hiccups' while spinning. Kind of like this: 

![](/assets/2022-01-30-Removing-the-'hiccup'-from-repeating-Unity-animations/rotation1.gif)

This is what the animation I created for [my blog post from yesterday](https://localjoost.github.io/Finding-the-Floor-with-HoloLens-2-and-MRTK-273/) looked immediately after I created it. But in the mean time, I had learned what the culprit was: curves.

## Curves? What curves

The Animation pane has another tab: Curves

![](/assets/2022-01-30-Removing-the-'hiccup'-from-repeating-Unity-animations/curves.png)

And if you click that, you see how Unity tries to be helpful. In the natural world, a physical object wants to change speed needs to overcome it's own *inertia*. When it starts moving it doesn't instantly reach it's speed - it *accelerates*. and when it stops moving, it slows down first before coming to a full stop. So this makes stuff that moves, move more naturally. Unless, of course, if you want to make a repeating animation like a rotating arrow - then it messes it up completely.

So what you need to do is, *carefully* click the point top left in the curve, right click, then select "Auto"

![](/assets/2022-01-30-Removing-the-'hiccup'-from-repeating-Unity-animations/point1.png)  

And repeat that process for the point bottom right

![](/assets/2022-01-30-Removing-the-'hiccup'-from-repeating-Unity-animations/point2.png)

So now the 'curve' is a straight line

![](/assets/2022-01-30-Removing-the-'hiccup'-from-repeating-Unity-animations/flat.png)

And the rotation smoothly repeats itself.

![](/assets/2022-01-30-Removing-the-'hiccup'-from-repeating-Unity-animations/rotation2.gif)

I really can't fathom why the menu entries are named this way - the 'Auto' setting results in a flat line, and 'Clamped Auto' a curve. The words don't really make a connection in my brain to what's happening in the animation or how the resulting line look like. I assume it's game developer's lingo, or may Unity is just plain weird at this point. It would definitely not be the only weird point ;).

## Conclusion

This is one of the rare occasions where I don't provide a specific repo to go with the blog post, although of course you can have a look at the[ repo accompanying yesterday's blog](https://github.com/LocalJoost/FloorFinder/tree/hl2mrkt273) where you will find an animation very much like this. 

Shout out to my colleague and fellow MVP [Timmy Kokke](https://twitter.com/sorskoot) who, after some hunting on the interwebz, found the answer somewhere last year when we stumbled upon this in one of our company HoloLens apps.