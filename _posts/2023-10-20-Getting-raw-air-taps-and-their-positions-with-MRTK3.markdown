---
layout: post
title: Getting raw air taps and their positions with MRTK3
date: 2023-10-20T00:00:00.0000000+02:00
categories: []
tags:
- HoloLens2
- MRTK3
- Service Framework
- Unity
featuredImageUrl: https://LocalJoost.github.io/assets/2023-10-20-Getting-raw-air-taps-and-their-positions-with-MRTK3/airtappos.gif
comment_issue_id: 460
---
Over the last few months, I sometimes feel like I am Microsoft's one-man Mixed Reality Q&A department, as I get lots of questions for tips, samples, and guidance. Apparently, many students and scientists are trying to get to grips with Mixed Reality. I try to answer them all, but sometimes it takes a while, as I attempt to answer them in order of appearance - and I also have a day job as an MR developer. Anyway, this week a [developer asked me on GitHub](https://github.com/LocalJoost/BlogComments/issues/421#issuecomment-1763662620) how you could not only get a raw air tap in MRTK3, but also how you could get the tap *position*.

So I took a look at the repo from [my blog post about this from June 2022](https://localjoost.github.io/MRTK2-to-MRTK3-intercepting-a-raw-air-tap-like-with-IMixedRealityPointerHandler/) - that was still on some old MRTK3 Pre-release - and first updated it. Then I added the requested functionality. The way to do it turns out to be very simple. First, you have to get a reference to a hands aggregator subsystem:

```csharp
handsAggregatorSubsystem = 
   XRSubsystemHelpers.GetFirstRunningSubsystem<IHandsAggregatorSubsystem>();
```

And then you get the actual pinch position like this:

```csharp
private Vector3 GetPinchPosition(ArticulatedHandController handController)
{
    return handsAggregatorSubsystem.TryGetPinchingPoint(handController.HandNode, 
                                                        out var jointPose)
        ? jointPose.Position
        : Vector3.zero;
}
```

The only piece of the puzzle you then need is finding the `ArticulatedHandController` objects for left and right hand, which you can do using my updated `MRTK3ConfigurationFindingService`.

All things combined, you can use it like this:

```csharp
findingService = 
   ServiceManager.Instance.GetService<IMRTK3ConfigurationFindingService>();

findingService.LeftHandStatusTriggered.AddListener(t=>
{
    leftHand.SetActive(t);
    if (t)
    {
        textMesh.text = $"Left hand position: {GetPinchPosition(findingService.LeftHand)}";
    }
});

findingService.RightHandStatusTriggered.AddListener(t=>
{
    rightHand.SetActive(t);
    if (t)
    {
        textMesh.text = $"Right hand position: {GetPinchPosition(findingService.RightHand)}";
    }
});
```

It still works like before, on HoloLens 2 and Quest, but it now also shows the position on which you actually click, and with what hand, in text.

![](/assets/2023-10-20-Getting-raw-air-taps-and-their-positions-with-MRTK3/airtappos.gif)

You can find the updated code and new functions [in this repo, branch crossplatairtap-position ](https://github.com/LocalJoost/MRKTAirTap/tree/crossplatairtap-position)