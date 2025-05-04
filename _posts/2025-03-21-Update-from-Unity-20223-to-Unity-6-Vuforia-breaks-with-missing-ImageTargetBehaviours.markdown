---
layout: post
title: 'Update from Unity 2022.3 to Unity 6: Vuforia breaks with missing ImageTargetBehaviours'
date: 2025-03-21T00:00:00.0000000+01:00
categories: []
tags:
- Unity
- Vuforia
- Mixed Reality
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2025-03-21-Update-from-Unity-20223-to-Unity-6-Vuforia-breaks-with-missing-ImnageTargetBehaviours/error1.png
comment_issue_id: 486
---
When I upgraded an existing solution from 2022.3.40f1 to Unity 6 (6000.35.f1) something very peculiar happened. First, I noticed a lot of errors in the console:

![error1](/assets/2025-03-21-Update-from-Unity-20223-to-Unity-6-Vuforia-breaks-with-missing-ImageTargetBehaviours/error1.png)

I also noticed all my ImageTargetBehaviours in the Inspector now showed this

![error2](/assets/2025-03-21-Update-from-Unity-20223-to-Unity-6-Vuforia-breaks-with-missing-ImageTargetBehaviours/error2.png)

In stead of this:

![unit2022](/assets/2025-03-21-Update-from-Unity-20223-to-Unity-6-Vuforia-breaks-with-missing-ImageTargetBehaviours/unit2022.png)

The weird thing is, that if I dragged an ImageTargetBehaviour directly onto a [ game in a test scene, it showed like you would expect. 

Now what?

Using WinMerge, I noticed something peculiar:

![guidchanges](/assets/2025-03-21-Update-from-Unity-20223-to-Unity-6-Vuforia-breaks-with-missing-ImageTargetBehaviours/guidchanges.png)

Left is a prefab with a broken ImageTargetBehaviour, right with a working one. They have a different GUID. And this is the class or file identity. So using Notepad++, I did a "Replace in files" from the top level and replaced in every .prefab file bab6fa851cf5a1a4bba3cec5f191cb8e with 8a9a760f95896c34689febc965510927. And hey presto, it worked!

Now why this happened to my project, I have no idea. The folks at Vuforia could not repro the problem. The thing is, all the ImageTargetBehaviours the project in question were in separate assemblies, maybe this is were the automatic upgrade failed. 

However, should you run in this problem, this is how to fix it.