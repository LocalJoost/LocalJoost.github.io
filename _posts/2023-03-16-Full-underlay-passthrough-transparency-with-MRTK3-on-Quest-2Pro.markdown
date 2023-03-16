---
layout: post
title: Full underlay passthrough transparency with MRTK3 on Quest 2/Pro
date: 2023-03-16T00:00:00.0000000+01:00
categories: []
tags:
- MRKT3
- Quest2
- HoloLens
featuredImageUrl: https://LocalJoost.github.io/assets/2023-03-16-Full-underlay-passthrough-transparency-with-MRTK3-on-Quest-2Pro/enable.png
comment_issue_id: 441
---
The fun thing with giving to the community is - sometimes the community starts *giving back* to you. 

Recently [I described how to get passthrough transparency](https://localjoost.github.io/Passthrough-transparency-with-MRTK2-and-3-on-Quest-2Pro/) with MRTK 2 and 3. The key point was, you needed to import the Oculus Integration package, add an OVR Manager and an OVR Passthrough Layer, then set the OVR Passthrough Layer to *overlay* and partially transparent. However, it turns out that with just a little change you can actually get full underlay transparency.

## 'Enable Unpremultiplied Alpha'

Say what? Yeah, the key trick is very simple: in addition to the already mentioned scripts you add one more, with the rather weird name "Enable Unpremultiplied Alpha". This is also part of the Oculus integration package. 

![](/assets/2023-03-16-Full-underlay-passthrough-transparency-with-MRTK3-on-Quest-2Pro/enable.png)

Then you can set the OVR Passthrough layer's placement to Underlay and the Opactity back to 1:

![](/assets/2023-03-16-Full-underlay-passthrough-transparency-with-MRTK3-on-Quest-2Pro/passthrough.png)

And the result is this. I added an MRTK3 panel to it for fun.

![](/assets/2023-03-16-Full-underlay-passthrough-transparency-with-MRTK3-on-Quest-2Pro/passthrough2.gif)

You can see the background is brighter than the transparent overlay solution I posted earlier, and the virtual elements are not a even a bit transparent, while with the overlay option (the second one), they clearly are.

![](/assets/2023-03-16-Full-underlay-passthrough-transparency-with-MRTK3-on-Quest-2Pro/QuestSeeThrough.gif)

## Credits

Where did I get that wisdom? Well, because [Yerio Janssen](https://www.linkedin.com/in/yerio-janssen-a20980239/), game student at Deltion College, started tinkering with the code in my previous post and contacted me when he stumbled upon the Enable Unpremultiplied Alpha script, which he found in the Quest PassThrough sample scenes. He informed me about his findings, and graciously allowed me to write a blog post about it. I also would like to [link to his sample GitHub project](https://github.com/DazzTDuck/Quest_Passthrough_MRTK3) here. 

This is what I mean by "the community starts giving back to you."

## A word of warning

When you use this underlay option in combination with Enable Unpremultiplied Alpha, your virtual elements get *completely opaque*, blocking *any* visual clue of reality, including obstacles. As reality is already a bit distorted by virtue of it being a reprojected camera image (in black & white in Quest 2 to boot), your users may get disoriented and/or easier miss obstacles. So use wisely, and make sure you don't endager your users by blocking too much of reality!

Also - I have not been able to get this to work with MRTK2 yet, and maybe it simply won't work. 

I have added the changes shown in the first movie to my own sample, [in this branch](https://github.com/LocalJoost/MRTKQuestSeeThrough/tree/underlay).
