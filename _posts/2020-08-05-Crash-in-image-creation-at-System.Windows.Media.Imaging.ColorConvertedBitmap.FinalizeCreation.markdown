---
layout: post
title: Crash in image creation at System.Windows.Media.Imaging.ColorConvertedBitmap.FinalizeCreation
date: 2020-08-05T00:00:00.0000000+02:00
tags: .net
featuredImageUrl: https://LocalJoost.github.io/assets/2020-08-05-Crash-in-image-creation-at-System.Windows.Media.Imaging.ColorConvertedBitmap.FinalizeCreation/CamtasiaCrash.png
comment_issue_id: 355
---
For my blog videos I tend to use [Camtasia](https://www.techsmith.com/store/camtasia) and I was not very amused when the 2019 version stopped working on my big development box - while it kept running perfectly on my laptop. Along came the 2020 version, and that did not run either. 

![](/assets/2020-08-05-Crash-in-image-creation-at-System.Windows.Media.Imaging.ColorConvertedBitmap.FinalizeCreation/CamtasiaCrash.png)

A prolonged conversation with Tech Support followed, but nothing we tried worked. It just crashed at startup.
The error message I found, digging into log files and the event viewer, came deep from within .NET


> The process was terminated due to an unhandled exception.
Exception Info: System.ArithmeticException
Exception Info: System.OverflowException
at System.Windows.Media.Imaging.ColorConvertedBitmap.FinalizeCreation()
at System.Windows.Media.Imaging.BitmapImage.FinalizeCreation()
at System.Windows.Media.Imaging.BitmapImage.EndInit()

To make matters worse, when I re-installed a Unity support plugin for [Resharper](https://www.jetbrains.com/resharper/) - I got the *same* error message. 

## Analysis in two parts
First, I had my wife login on the machine and had her start Camtasia. Flawless start. So it *had* to do with some setting in my profile. Second, ignoring the suggestion to delete my profile and start again, I got in touch with fellow MVP [Rafael Rivera](https://twitter.com/withinrafael) who analyzes crash dumps for fun while having breakfast :wink:. He could find nothing special, but he made one crucial remark - about color profiles. He was not entirely correct - but very nearly so. He told me to remove device specific color profiles, but that did not help. And then I found [this post](https://forum.affinity.serif.com/index.php?/topic/118653-photo-crash-on-startup-even-installer-windows/), handling about an entirely different product ('Affinity Photo'), but with the same error message.

## The culprit
It turns out, you have to look at the *advanced* tab:

![](/assets/2020-08-05-Crash-in-image-creation-at-System.Windows.Media.Imaging.ColorConvertedBitmap.FinalizeCreation/ProfileSetting.png)

Rafe's remark made me look on the first tab, but this post made me look on the second tab. I am not quite sure what I had selected - but if I have the selection displayed on the image above, Camtasia and the ReSharper plugin crash. If I take the Affinity forum suggestion and set it to System Default:
![](/assets/2020-08-05-Crash-in-image-creation-at-System.Windows.Media.Imaging.ColorConvertedBitmap.FinalizeCreation/Fixed.png)
The problem does not occur

## Conclusion
Never, ever mess with color profiles. I have GoogleBinged for error solutions for quite some time, but only when I added "color profiles" to the query string this popped up. TechSmith tech support was not familiar with it either - it's apparently a very dark corner of Windows where not much people are familiar with. So if this error occurs in an application - either a third party app or one made by yourself - you know where to look. Because it was so hard to find, I decided to write a blog post about it.
