---
layout: post
title: 'Short Unity tip: keeping values changed in play mode'
date: 2020-12-06T00:00:00.0000000+01:00
categories: []
tags:
- Unity3d
- HoloLens2
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2020-12-06-Short-Unity-tip-keeping-values-changed-in-play-mode/editinplaymode.png
comment_issue_id: 368
---
A common annoyance and mistake: the only way to position an object on the correct place, in the correct scale and/or rotation in Unity is in play mode. 

![](/assets/2020-12-06-Short-Unity-tip-keeping-values-changed-in-play-mode/editinplaymode.png)


And then you need either to make a screenshot of the new values, copy them down, or - in the worst case - you forget all that, stop play mode and poof - away is your work.

![](/assets/2020-12-06-Short-Unity-tip-keeping-values-changed-in-play-mode/goneisit.png)

Been there, done that. There is a logical reason not to retain play mode values, because animations could play havoc with your carefully crafted scene if all the values they change are stored in the scene. But sometimes, just sometimes you would very much like to retain these values. There appears to be a very simple way to do that, at least on component level.

Start play mode, do whatever you need to do to your poor game object (a cube in my case), then press the little three dots to the right of the Transform

![](/assets/2020-12-06-Short-Unity-tip-keeping-values-changed-in-play-mode/copycomponent.png)

... and click Copy Component. Stop play mode (and poof goes your work). But then select the three dots again

![](/assets/2020-12-06-Short-Unity-tip-keeping-values-changed-in-play-mode/pastcomponentvalues.png)

...and click Past Component Values

![](/assets/2020-12-06-Short-Unity-tip-keeping-values-changed-in-play-mode/copiedvalues.png)

And there's your play mode work, in edit mode. Of course, it's limited to this one component but still, this little trick saved me a lot of work. I guess it's Unity 102, but a surprising number of developers coming from an enterprise background - like myself - don't know this, so I thought it prudent to blog about it with an easy-to-find title.
