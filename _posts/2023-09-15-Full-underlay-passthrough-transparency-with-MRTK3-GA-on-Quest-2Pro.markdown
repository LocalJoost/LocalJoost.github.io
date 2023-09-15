---
layout: post
title: Full underlay passthrough transparency with MRTK3 GA on Quest 2/Pro
date: 2023-09-15T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- GA
- Quest2
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2023-09-15-Full-underlay-passthrough-transparency-with-MRTK3-GA-on-Quest-2Pro/color.png
comment_issue_id: 457
---
Hey, didn't I write about this before? [Indeed, I did](https://localjoost.github.io/Full-underlay-passthrough-transparency-with-MRTK3-on-Quest-2Pro/), but last Monday I got a report from [José Rocha](https://github.com/l4hacks) that my sample didn't work anymore after he followed [my upgrade tutorial for MRTK3 GA](https://localjoost.github.io/Upgrading-to-MRTK3-GA-from-a-pre-release-some-assembly-required/). After trying that myself on the sample, I had to agree he was right: instead of full transparency, I got to see the default Unity Skybox. Ugh.

The solution turns out to be extremely simple. After some plodding around in the MRTK3 GA sources, searching for the word "SkyBox" found me the `CameraSettingsManager` behavior. Where it comes from, I don't know, but it is certainly the solution.

To get transparency back, follow these simple steps:
* Add a component "CameraSettingsManager" somewhere in the scene. I added it to the "Camera Offset" in the MRTK XR Rig, because that looked like a logical place to put it, but I think you can put it anywhere
* Expand the Opaque section
* Change "SkyBox" into "Color"

![](/assets/2023-09-15-Full-underlay-passthrough-transparency-with-MRTK3-GA-on-Quest-2Pro/color.png)

* Click the color box, drag the Alpha slider all the way to the left. There should be four zeroes in a row now.

![](/assets/2023-09-15-Full-underlay-passthrough-transparency-with-MRTK3-GA-on-Quest-2Pro/alpha.png)

... and you are done. I have [updated the underlay branch of my sample project](https://github.com/LocalJoost/MRTKQuestSeeThrough/tree/underlay) to reflect this change.