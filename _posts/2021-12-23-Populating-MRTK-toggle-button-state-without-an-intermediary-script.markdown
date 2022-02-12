---
layout: post
title: Populating MRTK toggle button state without an intermediary script
date: 2021-12-23T00:00:00.0000000+01:00
categories: []
tags:
- HoloLens2
- MRTK2
- Unity3d
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2021-12-23-Populating-MRTK-toggle-button-state-without-an-intermediary-script/demo.gif
comment_issue_id: 399
---
Another short tip: Usually when you want to act on a toggle button press - for instance on a PressableButtonHoloLens2Toggle - you connect a script method to it's Interactable's OnClick event, and read the Interactable's IsToggled event afterward. But you can do this directly, as you can see if to download [this demo project](https://github.com/localjoost/DirectToggleButtonControl). The text disappears and reappears again without a helper script, as shown in this short video.

![](/assets/2021-12-23-Populating-MRTK-toggle-button-state-without-an-intermediary-script/demo.gif)

The trick is in the Interactable's Receivers. This is a panel that is right below the Events

![](/assets/2021-12-23-Populating-MRTK-toggle-button-state-without-an-intermediary-script/events.png)

And by default, it's collapsed. However, if you expand it, you will notices there's all kind of stuff set. But what we look for is at the end: the "Add Event" button. 

![](/assets/2021-12-23-Populating-MRTK-toggle-button-state-without-an-intermediary-script/receivers.png)

If you press the "Add Event" button, an "InteractableOnPressReceiver" is added. Change that into an "InteractableOnToggleReceiver"

![](/assets/2021-12-23-Populating-MRTK-toggle-button-state-without-an-intermediary-script/addevent.png)

This has two events, very obscurely called "OnSelect" and "OnDeselect". The name does not really suggest it, but the "OnSelect" event is triggered when the toggle button is toggled, the "OnDeselect" when the button is untoggled (is that even English? ;) ).

![](/assets/2021-12-23-Populating-MRTK-toggle-button-state-without-an-intermediary-script/togglereceiver.png)

 Anyway, we can add an event to the OnDeselect as well by clicking the + button underneath it, and then we can configure it like this:

![](/assets/2021-12-23-Populating-MRTK-toggle-button-state-without-an-intermediary-script/eventconfigured.png)

Drag the text component on the object (below both "Runtime Only" dropdowns), the select "GameObject.SetActive" on both occasions, and select the checkmark only at the "OnSelect" event. 

And this will make the text show when the toggle button is toggled, and hide it when it's untoggled. How's that for low code ;) ? This feature is especially handy when you have option menus and stuff that will pop out sub menus or something like that, and it saves you a lot of little scripts.

As written before, in the [demo project](https://github.com/localjoost/DirectToggleButtonControl) you can see it all put together, although this is literally all there is, and there is literally no code at all in it.
