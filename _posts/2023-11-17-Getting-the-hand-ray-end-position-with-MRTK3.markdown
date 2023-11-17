---
layout: post
title: Getting the hand ray end position with MRTK3
date: 2023-11-17T00:00:00.0000000+01:00
categories: []
tags:
- MRTK3
- HoloLens2
- Unity
- Service Framework
- HandRay
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2023-11-17-Getting-the-hand-ray-end-position-with-MRTK3/handraypos.gif
comment_issue_id: 465
---
Like I wrote before, I sometimes feel like I am Microsoft’s one-man Mixed Reality Q&A department, judging by the number of questions I get. I guess it's becoming common knowledge that I have the tendency to actually *answer* a lot of those questions ;). 

Anyway, after showing [how to get the position of the hand while doing an air tap](https://localjoost.github.io/Getting-raw-air-taps-and-their-positions-with-MRTK3/), I thought I was done on this subject. Nope: two different developers wanted to know if I could tell them how to get where the *hand ray was projecting on*. 

Well, I don't know if I have found the *right* way, or even the *best* way, but I at least have found *a* way. I modified my previous sample a bit (again), so now it not only shows for each hand where the hand itself is during a tap, but also where the end of the hand ray is. This is actually a pretty simple adaptation of the previous code. The start is more or less the same:

```csharp
private void Start()
{
    handsAggregatorSubsystem =
      XRSubsystemHelpers.GetFirstRunningSubsystem<IHandsAggregatorSubsystem>();
    leftHand.SetActive(false);
    rightHand.SetActive(false);
    findingService = ServiceManager.Instance.
      GetService<IMRTK3ConfigurationFindingService>();
```

but then comes the interesting part. See, my `MRTK3ConfigurationFindingService` does not only provide events to check if left or right hands are triggering in some way, but also direct access to the hands themselves. And the hands have a `LineRender` component in their children:

```csharp
var rightLineRenderer = findingService.RightHand.
  gameObject.GetComponentInChildren<LineRenderer>(true);
var leftLineRenderer = findingService.LeftHand.
      gameObject.GetComponentInChildren<LineRenderer>(true);
 ```

which happens to be the hand ray. And if you want to know where the end of the ray is: simply ask it the position of its last point like this:

```csharp
var rayPos = leftLineRenderer.
              GetPosition(leftLineRenderer.positionCount -1);
```

The whole thing that is triggered when you do an air tap with your left hand:

 ```csharp
findingService.LeftHandStatusTriggered.AddListener(t=>
{
    leftHand.SetActive(t);
    if (t)
    {
        var rayPos = leftLineRenderer.
          GetPosition(leftLineRenderer.positionCount -1);
        textMesh.text = $"Left hand position: {GetPinchPosition(findingService.LeftHand)}";
        textMesh.text +=
            $"{Environment.NewLine} Left hand ray position: {rayPos}";
        leftHand.transform.position = rayPos;
    }
});
```

- It sets the hand display active (like it already did)
- It shows the left hand position (like it already did) *and* the end position of the left hand ray (new)
- It moves the hand display to the hand ray end position (new)

The code for the right hand is omitted, as it's nearly identical. On a HoloLens 2, it looks like this:

![](/assets/2023-11-17-Getting-the-hand-ray-end-position-with-MRTK3/handraypos.gif)

To make sure the ray also hits physical objects (i.e., the spatial map), I have added an [ARMeshManager to the project's camera, as I described here](https://localjoost.github.io/Using-ARMeshManager-for-Spatial-Awareness-with-MRTK3-on-HoloLens-2/). The caveat is - this hits *everything* with a collider - not only the spatial map, but also the cube floating in the air. If you want to distinguish with that, you will have to do ray casts along the direction of the LineRender yourself.

Demo project can be downloaded from [this MRTKAirTap project branch](https://github.com/LocalJoost/MRKTAirTap/tree/crossplatairtap-rayendposition).