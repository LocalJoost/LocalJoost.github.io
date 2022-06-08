---
layout: post
title: MRTK2 to MRTK3 - tapping / selecting holograms
date: 2022-06-08T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- HoloLens2
- Unity
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2022-06-08-MRTK2-to-MRTK3--tapping-selecting-holograms/inspector.png
comment_issue_id: 419
---
I have a confession to make: as a Microsoft MVP I got a bit earlier in on MRTK3 than the rest of the world. I've spent the time porting [AMS HoloATC](https://www.microsoft.com/store/productId/9P6SVQQCP2SQ) over to MRTK3 - banging my head against all kinds of things I did not see coming and dropping into pitfalls I didn't know they were there - basically running into all kinds of stuff I did not understand - so I could learn and document things, so you, dear reader, would not have to suffer half the pain I did. 

You are *welcome*. ;)

This is the first of probably a *long* series of post. Years ago I started the "Migrating to MRTK2" series when the HoloToolkit was sunsetted, and now we move over to MRTK3. I start with very simple things. Be aware I am mostly stumbling in the dark yet, and my way may not be the smart way or even the way *intended* by the MRTK team. 

## How it was

Using MRTK2, you had made this simple cube tapper that made by implementing  `IMixedRealityPointerHandler`

```csharp
using Microsoft.MixedReality.Toolkit.Input;
using UnityEngine;

public class ColorToggler : MonoBehaviour, IMixedRealityPointerHandler
{
    public void OnPointerDown(MixedRealityPointerEventData eventData)
    {
        gameObject.GetComponent<Renderer>().material.color = Color.green;
    }

    public void OnPointerDragged(MixedRealityPointerEventData eventData)
    {
    }

    public void OnPointerUp(MixedRealityPointerEventData eventData)
    {
        gameObject.GetComponent<Renderer>().material.color = Color.red;
    }

    public void OnPointerClicked(MixedRealityPointerEventData eventData)
    {
    }
}
```

You also put in a `NearInteractionTouchable` and set the EventToReceive to "Pointer":

![](/assets/2022-06-08-MRTK2-to-MRTK3--tapping-selecting-holograms/inspector.png)

And the net result was that when you either touched the cube or air tapped (using the hand ray) it, it would turn green when you tapped, and red when you released the tap.

![](/assets/2022-06-08-MRTK2-to-MRTK3--tapping-selecting-holograms/Toggler.gif)

## Moving to MRTK3

If you simply plonk the scene and the assets in an MRTK3 project, remove the MRTK2 things from the scene and drag in a MRTK XR Rig prefab, you will notice everything is broken:

![](/assets/2022-06-08-MRTK2-to-MRTK3--tapping-selecting-holograms/brokeh.png)

Your ColorToggler script is broken, and the NearInteractionTouchable is gone. So let's remove that, and try to fix ColorToggler.

So the simplest way to get this working again, is not implement IMixedRealityPointerHandler but to subclass `MRTKBaseInteractable`, and override `OnSelectEntered` and `OnSelectExited`

```csharp
using Microsoft.MixedReality.Toolkit;
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

public class ColorToggler : MRTKBaseInteractable
{
    protected override void OnSelectEntered(SelectEnterEventArgs args)
    {
        gameObject.GetComponent<Renderer>().material.color = Color.green;
    }

    protected override void OnSelectExited(SelectExitEventArgs args)
    {
        gameObject.GetComponent<Renderer>().material.color = Color.red;
    }
}
```
![](/assets/2022-06-08-MRTK2-to-MRTK3--tapping-selecting-holograms/MRTK3tap.gif)

And as you can see, about the only difference you see is the way the simulated hand is displayed. Incidentally, to get hand simulation at all, you will need to have need to have the MRTKInputSimulator prefab in your scene

![](/assets/2022-06-08-MRTK2-to-MRTK3--tapping-selecting-holograms/handsimulator.png)

What you also can see is that we apparently need a little less code. 

## Caveat Emptor

The MRTK3 is *in public preview*. Code may change dramatically. This is just me documenting some simple things.

## Concluding words

You can find the code [of this sample here](https://github.com/LocalJoost/MRKT3TapThings).

If you want to see what the MRTK3 version of AMS HoloATC looks like, here's a little YouTube video

<iframe width="650" height="365" src="https://www.youtube.com/embed/3zW8ba6Qbkw" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
