---
layout: post
title: A behaviour to move an object back to its initial location after moving it manually
date: 2025-05-04T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- Unity
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2025-05-04-A-behaviour-to-move-an-object-back-to-its-initial-location-after-moving-it-manually/bounceback.gif
comment_issue_id: 487
---
Recently I was approached by someone who uses MRTK3 to make basic Mixed Reality applications - without coding skills. He couldn't find a specific functionality in the MRTK3 and this wasn't a surprise, as it simply wasn't there. It was a simple task for me, so I wrote it in a few minutes just for fun. The functionality he asked for was: when you stop manipulating an object, you want it to be moved back to its original location.

So like this:

![bounceback](/assets/2025-05-04-A-behaviour-to-move-an-object-back-to-its-initial-location-after-moving-it-manually/bounceback.gif)

I made it a bit more elaborate - it doesn't simply put the object back but **moves** it smoothly, also takes into account rotation, and as a little cherry on top I made the timeout before the object starts moving back and the speeds at which it does so configurable. 

The behaviour isn't exactly rocket science, but I thought it was still fun enough to dedicate a little blog post to it.

The start is simple enough, first with two editor-configurable parameters I already mentioned, and some private members we will need along the way:

```csharp
[RequireComponent(typeof(ObjectManipulator))]
[DisallowMultipleComponent]
public class BounceBackBehaviour : MonoBehaviour
{
    [SerializeField]
    [Min(0.01f)]
    private float bounceBackTime = 1f;

    [SerializeField]
    [Min(0.01f)]
    private float bounceBackDelay = 0.5f;

    private Pose originalPose = Pose.identity;
    private CancellationTokenSource cancellationTokenSource;
    private bool initialLocationObtained = false;

    private ObjectManipulator objectManipulator;
```

The Awake method finds the `ObjectManipulator` and wires methods to the events that signal the beginning and ending of manual manipulation:

```csharp
void Awake()
{
    objectManipulator = GetComponent<ObjectManipulator>();
    objectManipulator.firstSelectEntered.AddListener(OnFirstSelectEntered);
    objectManipulator.lastSelectExited.AddListener(OnLastSelectExited);
}
```

When `OnFirstSelectEntered` is called, it first cancels any processes still running, and when it's called for the first time, obtains the original position of the object. Note it takes the `ObjectManipulator`s HostTransform object, not its own game object. Although the ObjectManipulator's HostTransform defaults to the game object it's placed on, it can be actually changed to manipulate a different object.

```csharp
private void OnFirstSelectEntered(SelectEnterEventArgs _)
{
    cancellationTokenSource?.Cancel();
    cancellationTokenSource?.Dispose();
    cancellationTokenSource = null;
    if (!initialLocationObtained)
    {
        initialLocationObtained = true;
        originalPose = new Pose(objectManipulator.HostTransform.position, transform.rotation);
    }
}
```

`OnLastSelectExited` makes a cancellation token, waits for the designated time, and if it still has not been canceled (so the object is not grabbed again) it bounces back the object to its original position.

```csharp
private async void OnLastSelectExited(SelectExitEventArgs _)
{
    cancellationTokenSource = new CancellationTokenSource();
    await Task.Delay((int)(bounceBackDelay * 1000), cancellationTokenSource.Token);
    if (!cancellationTokenSource.IsCancellationRequested)
    {
        await BounceBack();
    }
}
```

BounceBack is a rather standard lerp-based routine that moves and rotates the object back to its original position and orientation: 

```csharp
private async Task BounceBack()
{
    var startPos = objectManipulator.HostTransform.position;
    var startRot = objectManipulator.HostTransform.rotation;
    var i = 0f;
    var rate = 1.0f / bounceBackTime;
    while (i <= 1 && cancellationTokenSource is { IsCancellationRequested: false })
    {
        i += Time.deltaTime * rate;
        objectManipulator.HostTransform.SetPositionAndRotation(
            Vector3.Lerp(startPos, originalPose.position, Mathf.SmoothStep(0f, 1f, i)),
            Quaternion.Lerp(startRot, originalPose.rotation, Mathf.SmoothStep(0f, 1f, i)));
        await Task.Yield();
    }

    if (cancellationTokenSource is { IsCancellationRequested: false })
    {
        transform.SetPositionAndRotation(originalPose.position, originalPose.rotation);
    }
}
```

Note here as well the HostTransform is used, and the lerp loop can be interrupted at any point by the user grabbing the object, and thus canceling the cancellation token. The final lines of code make sure the original pose and rotation are set, as Unity sometimes makes little rounding errors while lerping, and then the object ends up just slightly from its original pose.

You can see the behaviour in action in [the demo project](https://github.com/LocalJoost/BounceBackBehaviour) - in the sample scene you can move the coffee cup that I nicked from the MRTK3 hand interaction demo scene to see that it actually moves back. That's where the little video earlier in this project comes from.