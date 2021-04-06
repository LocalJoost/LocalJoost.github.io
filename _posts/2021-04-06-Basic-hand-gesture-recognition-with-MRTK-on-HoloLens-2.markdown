---
layout: post
title: Basic hand gesture recognition with MRTK on HoloLens 2
date: 2021-04-06T00:00:00.0000000+02:00
categories: []
tags:
- HoloLens2
- MRKT2
- Unity3d
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2021-04-06-Basic-hand-gesture-recognition-with-MRTK-on-HoloLens-2/hero.png
comment_issue_id: 376
---
For a little POC I was working on, I wanted to be able to detect if a user was grabbing or pinching on or near specific locations. I did not need the elaborate functionality of the `Interactable`, because I was planning on doing something completely different, which I will talk about in the next two blog posts.

What I wanted to achieve, initially, was something like this where I could distinguish between hand and gesture, specifically pinch and grab:

<iframe width="650" height="365" src="https://www.youtube.com/embed/-cGkgOiLo_E" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

As you can see:
* The cylinder on the left accepts grab and pinch by both hands
* The capsule accept only pinch, and only by the right hand
* The cube accepts only grab, and only by the left hand

## Detecting grabbing and pinching

In the `Microsoft.MixedReality.Toolkit.Utilities` namespace there is a little hidden gem called `HandPoseUtils`. It has a few nice utility methods for detecting grab - but for some reason I don't get the result I want. So I wrote my own set of little helper utilities around it.

```csharp
using Microsoft.MixedReality.Toolkit.Utilities;

namespace MRKTExtensions.Gesture
{
    public static class GestureUtils
    {
        private const float PinchThreshold = 0.7f;
        private const float GrabThreshold = 0.4f;
        
        public static bool IsPinching(Handedness trackedHand)
        {
            return HandPoseUtils.CalculateIndexPinch(trackedHand) > PinchThreshold;
        }

        public static bool IsGrabbing(Handedness trackedHand)
        {
            return !IsPinching(trackedHand) &&
                   HandPoseUtils.MiddleFingerCurl(trackedHand) > GrabThreshold &&
                   HandPoseUtils.RingFingerCurl(trackedHand) > GrabThreshold &&
                   HandPoseUtils.PinkyFingerCurl(trackedHand) > GrabThreshold &&
                   HandPoseUtils.ThumbFingerCurl(trackedHand) > GrabThreshold;
        }
    }
}
```

Detecting a pinch is the easiest - you simply call the calculate `CalculateIndexPinch` and check it against a threshold. Grabbing is a bit more difficult, because it's based upon detecting the curling of your finger. But if you pinch, you also tend to curl your fingers. So I check for the right finger curling but specifically check for *not* pinching.

The values here are based on nothing but empirical data and you might want to change them for your purposes.

## And now for a demo, thank you

I pride myself in always creating a working demo, but in this case actually demoing the code in a kind-of useful way is considerably much more work than the actual code. However, noblesse oblige so here we go with the `DyiHandManipulation` behaviour, basically a poor man's way to move and rotate objects. I would not suggest using this in a real app (use the MRKT components instead) but it's instructive in showing how you can use the above extension methods in a working context.

The first part simply is the settings you can do: 
* which hand joint we track, 
* what the distance is for the grab/pinch to be activated,
* what hand to track,
* whether we should track pinch, grab, or both.

```csharp
namespace DyiPinchGrab
{
    public class DyiHandManipulation : MonoBehaviour
    {
        [SerializeField]
        private TrackedHandJoint trackedHandJoint = TrackedHandJoint.IndexMiddleJoint;

        [SerializeField]
        private float grabDistance = 0.1f;

        [SerializeField]
        private Handedness trackedHand = Handedness.Both;

        [SerializeField]
        private bool trackPinch = true;

        [SerializeField]
        private bool trackGrab = true;
    }
}
```
There's also these private fields we need to access the HandJointService and to keep track of where the hands were previously

```csharp
private IMixedRealityHandJointService handJointService;

private IMixedRealityHandJointService HandJointService =>
    handJointService ??
    (handJointService = CoreServices.GetInputSystemDataProvider<IMixedRealityHandJointService>());

private MixedRealityPose? previousLeftHandPose;

private MixedRealityPose? previousRightHandPose;
```

I jump down a little, otherwise this is going to be a bit confusing, to this little method:
```csharp
private MixedRealityPose? GetHandPose(Handedness hand, bool hasBeenGrabbed)
{
    if ((trackedHand & hand) == hand)
    {
        if (HandJointService.IsHandTracked(hand) &&
            ((GestureUtils.IsPinching(hand) && trackPinch) ||
             (GestureUtils.IsGrabbing(hand) && trackGrab)))
        {
            var jointTransform =
               HandJointService.RequestJointTransform(trackedHandJoint, hand);
            var palmTransForm =
               HandJointService.RequestJointTransform(TrackedHandJoint.Palm, hand);
            
            if(hasBeenGrabbed || 
               Vector3.Distance(gameObject.transform.position,
                 jointTransform.position) <= grabDistance)
            {
                return new MixedRealityPose(jointTransform.position,
                                            palmTransForm.rotation);
            }
        }
    }

    return null;
}
```
What this does is:
* First check if the current hand should be tracked at all
* Then checks if this hand is grabbing or pinching, together with if we should track pinch or grab at all for this hand
* Get the position of the tracked joint of the hand (default: `TrackedHandJoint.IndexMiddleJoint`)
* If said joint is within grabbing distance ***or*** the object has been grabbed before, return the current tracked joint's pose.
* And the important thing - if it does ***not*** meet all those requirements, return null.

In short, check if the current hand has a valid grabbing/pinching solution according to settings and previous data, and return a pose (or null) accordingly.

This method is called from the Update method, which is as follows:
```csharp
private void Update()
{
    var leftHandPose = GetHandPose(Handedness.Left, 
                                   previousLeftHandPose != null);
    var rightHandPose = GetHandPose(Handedness.Right, 
                                    previousRightHandPose != null);
    {
        var jointTransform =
           HandJointService.RequestJointTransform(trackedHandJoint, 
                                                  trackedHand);
        if (rightHandPose != null && previousRightHandPose != null)
        {
            if (leftHandPose != null && previousLeftHandPose != null)
            {
                // fight! pick the closest one
                var isRightCloser = 
                  Vector3.Distance(rightHandPose.Value.Position,
                     jointTransform.position) <
                  Vector3.Distance(leftHandPose.Value.Position,
                      jointTransform.position);

                ProcessPoseChange(
                    isRightCloser ? previousRightHandPose : previousLeftHandPose,
                    isRightCloser ? rightHandPose : leftHandPose);
            }
            else
            {
                ProcessPoseChange(previousRightHandPose, rightHandPose);
            }
        }
        else if (leftHandPose != null && previousLeftHandPose != null)
        {
            ProcessPoseChange(previousLeftHandPose, leftHandPose);
        }
    }
    previousLeftHandPose = leftHandPose;
    previousRightHandPose = rightHandPose;
}
```

It may look a bit intimidating, but it really is simple. It first gets poses for both hands. Remember, it gets null from `GetHandPose` when there is no valid hand grab or pinch pose that meets the requirements. So if there was a valid hand pose the last time, we apparently already have grabbed or pinched the object in a previous call of `Update` and this goes back to this line of `GetHandPose`
```csharp
if(hasBeenGrabbed || 
   Vector3.Distance(gameObject.transform.position,
     jointTransform.position) <= grabDistance)
{
    return new MixedRealityPose(jointTransform.position,
                                palmTransForm.rotation);
}
```
it then bypasses the distance check. Once you know that, the rest of the method simply checks
* if there is a right pose, check if there is also a left pose
* if there is indeed a left pose as well, then take the closest one
* if there was only a right pose, try a right pose
* if there was no right pose in the first test, then check a left pose
* if there is a left pose, take the left pose

Then there is finally this method:

```csharp
private void ProcessPoseChange(
  MixedRealityPose? previousPose, 
  MixedRealityPose? currentPose)
{
    var delta = currentPose.Value.Position - previousPose.Value.Position;
    var deltaRotation = Quaternion.FromToRotation(
                            previousPose.Value.Forward, 
                            currentPose.Value.Forward);
    gameObject.transform.position += delta;
    gameObject.transform.rotation *= deltaRotation;
}
```
It simply determines how the pose of the grabbing/pinching hand has changed relatively to the last time, and rotates the object accordingly. I learned this way about the existence of `Quaternion.FromToRotation` which basically gives you a *delta rotation*. And I learned that unlike position - where you *add* a delta vector to move, you actually have to *multiply* a quaternion with a delta quaternion to a delta rotate. 

## Final notes
To make things like this easier to test in the Unity editor, I usually make these settings in the Input Simulation Service

![](/assets/2021-04-06-Basic-hand-gesture-recognition-with-MRTK-on-HoloLens-2/handprofile.png)

You will find this setting in the Mixed Reality Toolkit Settings under Input/Input Data Providers/Input Simulation Service and then scroll a bit down. You will have to clone the Toolkit profile, the Input System Profile and the Input Simulation Profile first to be able to change this settings at all. These profile settings are included in the demo project. This [demo project can be downloaded here](https://github.com/LocalJoost/DYIPinchGrab).
