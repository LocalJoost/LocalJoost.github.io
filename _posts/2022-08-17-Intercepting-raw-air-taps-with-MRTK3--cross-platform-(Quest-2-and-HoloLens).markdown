---
layout: post
title: Intercepting raw air taps with MRTK3 - cross platform (Quest 2 and HoloLens)
date: 2022-08-17T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- HoloLens2
- Unity
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2022-08-17-Intercepting-raw-air-taps-with-MRTK3--cross-platform-(Quest-2-and-HoloLens)/riggedhandmeshvisualizer.png
comment_issue_id: 427
---
Some people might start to think I am becoming a bit obsessed with the subject of air taps, as this the third post about this subject in two months. [First I described](https://localjoost.github.io/MRTK2-to-MRTK3-intercepting-a-raw-air-tap-like-with-IMixedRealityPointerHandler/) how I wrote a [Reality Collective Service](https://service-framework.realitycollective.io/docs/get-started/) to intercept air taps, later [Finn Sinclair](https://github.com/Zee2)  of the MRTK team [explained this could be done simpler](https://github.com/LocalJoost/BlogComments/issues/421#issuecomment-1184713692) - and this merited [blog post 2](https://localjoost.github.io/MRTK2-to-MRTK3-a-MUCH-SIMPLER-way-to-intercept-a-raw-air-tap-like-with-IMixedRealityPointerHandler/). But anyway, lately I upgraded AMS HoloATC to MRTK3 pre.9 and found I had full hand control now in both HoloLens 2 and Quest 2, with no extra code, [which I tweeted about proudly](https://twitter.com/LocalJoost/status/1557351986164531200).

Only to discover, some time later, this was not entirely true. The 'raw air tap' did not work. And I need that for the user to confirm the location of the airport. I assumed an issue in MRTK3 and opened one on GithUb, but [as Finn explained to me in a comment](https://github.com/microsoft/MixedRealityToolkit-Unity/issues/10856#issuecomment-1211084417), HoloLens adheres to the OpenXR standard by having an interaction profile that exposes actions performed by the user in a unified way. Unfortunately, the Quest 2 does not (yet). The reason why hand control for Quest 2 still works in MRTK3 is trickery by the MRTK team that *makes* it work regardless of that missing profile. That included everything, except for the raw air tap.

Challenge accepted :) 

## Spelunking into the MRTK3 innards

MRTK3 clearly has a way to *visualize* the hand tracking in Quest 2. So there must be a way to generate an air tap event myself, if necessary completely manually, by analyzing index finger and thumb joint positions. The only thing I had to find out was how they managed to get the data needed for visualization. Time to go spelunking into the MRTK3. Exploratory programming - one of my favorite occupations. :)

Anyway, if you pull the [MRTK3 source from GitHub](https://github.com/microsoft/MixedRealityToolkit-Unity/tree/mrtk3) and run a random scene which has the MRTKInputSimulator in it, you can see what exactly visualizes the hand:

![](/assets/2022-08-17-Intercepting-raw-air-taps-with-MRTK3--cross-platform-(Quest-2-and-HoloLens)/riggedhandmeshvisualizer.png)

Having a look into `RiggedHandMeshVisualizer`'s code, the `OnEnable` method (hello again Finn) quickly spills some beans:

![](/assets/2022-08-17-Intercepting-raw-air-taps-with-MRTK3--cross-platform-(Quest-2-and-HoloLens)/OnEnable.png)

And just when I thought "well now I have to devise an algorithm to determine whether or not the user is pinching thumb and index finger together", IntelliSense told me kindly not to bother:

![](/assets/2022-08-17-Intercepting-raw-air-taps-with-MRTK3--cross-platform-(Quest-2-and-HoloLens)/trygetpinch.png)

Because Finn's colleague [David](https://github.com/davidkline-ms) already built something for that:

![](/assets/2022-08-17-Intercepting-raw-air-taps-with-MRTK3--cross-platform-(Quest-2-and-HoloLens)/trygetpinchimpl.png)

So it turns out all the pieces I need are already in the MRTK3!

## Building a cross-platform air tap intercepting service

I went back to my first implementation of intercepting a raw air taps - the Reality Collective Service - to make it both intercept air taps on HoloLens and Quest 2, with the goal of making something that would work in such a way that the developer using it would not need to worry about it running on either device.

I adapted the service's Start method to not only look for the controllers, but also the necessary subsystem, storing that in a private field:

```csharp
public override void Start()
{
    GetHandControllerLookup();
    handsAggregatorSubsystem = XRSubsystemHelpers.
      GetFirstRunningSubsystem<HandsAggregatorSubsystem>();
}
```

The `Update` method, which first contained all code, now calls two methods:

```csharp
public override void Update()
{
    if (!TryUpdateByTrigger())
    {
        TryUpdateByPinch();
    }
}
```

And you can see where this is going: the code first tries to check if an air tap is received by the 'regular' method, that is - via the XR Interaction Profile. If returns true it does not need to look further.

```csharp
private bool TryUpdateByTrigger()
{
    var triggeredByTrigger = false;
    var newStatus = GetIsTriggered(LeftHand);
    if (newStatus != leftHandTriggerStatus && LeftHand)
    {
        leftHandTriggerStatus = newStatus;
        LeftHandStatusTriggered.Invoke(leftHandTriggerStatus);
        triggeredByTrigger = true;
    }
    
    newStatus = GetIsTriggered(RightHand);
    if (newStatus != rightHandTriggerStatus)
    {
        rightHandTriggerStatus = newStatus;
        RightHandStatusTriggered.Invoke(rightHandTriggerStatus);
        triggeredByTrigger = true;
    }

    return triggeredByTrigger;
}
```

However, if nothing is received by the first method, it calls the other one, which looks remarkably like the first one:

```csharp
private void TryUpdateByPinch()
{
    if (handsAggregatorSubsystem != null)
    {
        var newStatus = TryUpdateByPinch(LeftHand);
        if (newStatus != leftHandTriggerStatus)
        {
            leftHandTriggerStatus = newStatus;
            LeftHandStatusTriggered.Invoke(leftHandTriggerStatus);
        }
    
        newStatus = TryUpdateByPinch(RightHand);
        if (newStatus != rightHandTriggerStatus)
        {
            rightHandTriggerStatus = newStatus;
            RightHandStatusTriggered.Invoke(rightHandTriggerStatus);    
        }
    }
}
```

And the secret sauce, if it's worth that name, is of course simply calling David's method:

```csharp
private bool TryUpdateByPinch(ArticulatedHandController handController)
{
    var progressDetectable = 
            handsAggregatorSubsystem.TryGetPinchProgress(handController.HandNode,
                                                         out bool isReadyToPinch, 
                                                         out bool isPinching, 
                                                         out float pinchAmount);
    return progressDetectable && isPinching && pinchAmount > PinchTreshold;
}
```

And lo and behold, the raw air tap now also works in Quest 2.

![](/assets/2022-08-17-Intercepting-raw-air-taps-with-MRTK3--cross-platform-(Quest-2-and-HoloLens)/rawairtap.gif)

Developers still only have to subscribe to `LeftHandStatusTriggered` and `RightHandStatusTriggered`

```csharp
private void Start()
{
    leftHand.SetActive(false);
    rightHand.SetActive(false);
    
    findingService = ServiceManager.
      Instance.GetService<IMRTK3ConfigurationFindingService>();
    
    findingService.LeftHandStatusTriggered.AddListener(t=> leftHand.SetActive(t));
    findingService.RightHandStatusTriggered.AddListener(t=> rightHand.SetActive(t));
}
```
and basically do not have to care at all whether their app runs on HoloLens or Quest 2.

## Why back to the old method?

As I described [in the previous post](https://localjoost.github.io/MRTK2-to-MRTK3-a-MUCH-SIMPLER-way-to-intercept-a-raw-air-tap-like-with-IMixedRealityPointerHandler/), I was explained by Finn my method of intercepting air taps is a bit convoluted and could be achieved in a simpler way. While that's certainly true, *this* method is cross platform *now* and not in some future where Meta decides to make a proper interaction profile for Quest 2. And when the proper interaction profile comes, I can just change the implementation of my service - the MonoBehaviours using it won't know the difference and don't need to be adapted.

Because you see, I found out the following interesting thing. In my previous post, I showed you how you could drag Input Actions onto input references, remember?

![](/assets/2022-08-17-Intercepting-raw-air-taps-with-MRTK3--cross-platform-(Quest-2-and-HoloLens)/dragdrop.png)

It turns out that once you have a reference to the *controller* do you need a reference to the InputActionReference, *because you already have it*. Via some IntelliSense sleuthing I found the actions attached to a controller not only have a `value` property, but also a `reference` property:

```csharp
LeftHand.selectActionValue.reference
```

And wouldn't you know it - you can attach to the performed event like this:

```csharp
LeftHand.selectActionValue.reference.action.performed += ProcessContext;
```
Just like a reference that's dragged on your behaviour. So when the interaction profile comes, I can just tap this event in my service and be done with it. To be honest, I am more a proponent of explicit access via code paths than via implicit access via drag & drop Unity references anyway. Moreover, a service like this can be mocked and tested. So for the moment, I think I will stick with accessing input events via this service after all.

Project can be [downloaded from GitHub as usual](https://github.com/LocalJoost/MRKTAirTap/tree/crossplatairtap).