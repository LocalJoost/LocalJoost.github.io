---
layout: post
title: Renaming prefabs in Lens Studio
date: 2025-10-31T00:00:00.0000000+01:00
categories: []
tags:
- Lens Studio
- Spectacles
- Prefabs
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2025-10-31-Renaming-prefabs-in-Lens-Studio/fix.png
comment_issue_id: 492
---
Is this really worth a blog post? Well maybe it is not, but it is something that made me scratch my head. An maybe other people are stymied by it as well, so I thought to write a little something. The issue is this. Suppose I have a prefab MyAwesomePrefab, like this:

![prefab1](/assets/2025-10-31-Renaming-prefabs-in-Lens-Studio/prefab1.png)

Lens Studio unfortunately doesn't know the concept of prefab variants like Unity does, so if I want another prefab that is almost the same, I need to copy it. Let's call it MyOtherPrefab. I add an extra text. The top scene object is still called MyAwesomePrefab, so let's rename that, and hit Apply

![prefab2](/assets/2025-10-31-Renaming-prefabs-in-Lens-Studio/prefab2.png)

You might think we are done, but this is were it gets odd. Because if you drag both MyAwesomePrefab and MyOtherPrefab in the scene...

![prefab3](/assets/2025-10-31-Renaming-prefabs-in-Lens-Studio/prefab3.png)

*They both show up as MyAwesomePrefab*.

This, of course, can be highly confusing. Maybe it's a bug, or maybe I simply not understand how this is supposed to work. However, this is how I fix it. Fortunately, all files a Lens Studio creates are text files. I don't know what you call this format, but somewhere you will find this:

![fix](/assets/2025-10-31-Renaming-prefabs-in-Lens-Studio/fix.png)

Change that to MyAwesomePrefab, save the file, and if you now drag MyAwesomePrefab on the scene...

![prefab4](/assets/2025-10-31-Renaming-prefabs-in-Lens-Studio/prefab4.png)

Important note: *save your project first before starting the rename*. If you haven't saved it, Lens Studio makes a kind of temporay project on a in a temporary location and that may cause issues when you do things like this. Even better - commit your project to Git before doing scary things like this. A prudent developer always makes small steps so it's easy to recover from oopsies. 

No code this time, as there is no code to share ;)