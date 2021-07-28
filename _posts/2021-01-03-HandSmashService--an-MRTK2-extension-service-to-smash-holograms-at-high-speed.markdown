---
layout: post
title: HandSmashService - an MRTK2 extension service to smash holograms at high speed
date: 2021-01-03T00:00:00.0000000+01:00
categories: []
tags:
- HoloLens2
- MRTK2
- Unity3d
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2021-01-03-HandSmashService--an-MRTK2-extension-service-to-smash-holograms-at-high-speed/Handdirection.png
comment_issue_id: 370
---
I was (and am) very excited about the HoloLens 2 hand tracking capabilities, and was intrigued by the MRKT2 Hand Physics Service. Yet I quickly found out that service has some limitations: although it's a great way to detect whether or not things are *touched*, it's capabilities for actually manipulating stuff is limited to low speed interactions.

The Hand Physics Service basically attaches a collider around your hand (on finger), which you can use to push holograms a bit around and then let Unity colliders do the work. An elegant solution, but if you try to smash to swat a hologram, you will basically smash right trough it. The hologram halfheartedly moves a little with your hand, but it's not like it moves away like you swatted a ball or something. 

<iframe width="650" height="365" src="https://www.youtube.com/embed/-4WOH9dzkSQ" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


What I envisioned was more something like this:

<iframe width="650" height="365" src="https://www.youtube.com/embed/7L3LnK6SUpo" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


And went on to actually *build* it. Meet HandSmashService, an MRKT2 extension service just like the Hand Physics Service, but then for high speed interactions - specifically, swatting holograms. For this I reached way back into my own blog, to one of my very first HoloLens apps, the [CubeBouncer](https://localjoost.github.io/hololens-cubebouncer-application-part-3/), from which I borrowed some principles - specifically how to apply directed force to holograms - that I learned in (checks notes) July 2016. Wow, time flies when you are having fun.

## The basic principle


* The code tracks hands using the standard MRKT2 `IMixedRealityHandJointService` every update. It keeps the previous position, so I know how far the hand moved during one frame. This gives me a measure of speed and direction.
* Assuming the hand will continue to travel in the same direction and with the same speed, I can
    * Perform a ray cast from it's *current* position in the direction where hand will be most likely next 
    * If that ray cast hits an object, apply force to it proportionally to the speed of the hand 

In a picture: 
![](/assets/2021-01-03-HandSmashService--an-MRTK2-extension-service-to-smash-holograms-at-high-speed/Handdirection.png)

The result you can see in the 2nd video above - a more realistic response from something that's swatted away.

## HandSmashingService explained

### General setup
The first part shows the general setup of the HandSmashing Service as a MRTK2 extension service. Basically the only thing noteworthy is the fact this shows it's piggybacking onto the standard MRTK2 Hand Joint Service:

```cs
using Microsoft.MixedReality.Toolkit.Utilities;
using Microsoft.MixedReality.Toolkit;
using Microsoft.MixedReality.Toolkit.Input;
using UnityEngine;

namespace MRTKExtensions.Manipulation
{
    [MixedRealityExtensionService(..stuff omitted...)]
    public class HandSmashingService : BaseExtensionService, IHandSmashingService
    {
        private HandSmashingServiceProfile handSmashingServiceProfile;
        private IMixedRealityHandJointService handJointService;

        public HandSmashingService(string name, uint priority,
                                   BaseMixedRealityProfile profile) : 
        base(name, priority, profile) 
        {
            handSmashingServiceProfile = (HandSmashingServiceProfile)profile;
        }

        private IMixedRealityHandJointService HandJointService
            => handJointService ?? (handJointService =
            CoreServices.
              GetInputSystemDataProvider<IMixedRealityHandJointService>());
    }
}
```
### The Update method 
This method is called every frame, so 60 times a second (if your app is running properly, that is). It does not show much, apart the `ApplySmashMovement` method is called for each hand - and that the result is retained

```cs
private Vector3? lastRightPosition;
private Vector3? lastLeftPosition;

public override void Update()
{
    base.Update();
    if (HandJointService == null)
    {
        return;
    }

    lastRightPosition = ApplySmashMovement(Handedness.Right, lastRightPosition);
    lastLeftPosition = ApplySmashMovement(Handedness.Left, lastLeftPosition);
}
```
### The ApplySmashMovement method 
Now it gets a bit more interesting. This method  checks if the current requested hand should be tracked at all according to the settings in the profile (see later), and if so, it actually gets the palm position. Upon securing that, it passes both old and new hand positions to `TryApplyForceFromVectors` which is where the real action happens
```cs
private Vector3? ApplySmashMovement(Handedness handedness, Vector3? previousHandLocation)
{
    Vector3? currentHandPosition = null;
    if ((handSmashingServiceProfile.TrackedHands & handedness) == handedness)
    {
        if (HandJointService.IsHandTracked(handedness))
        {
            currentHandPosition = 
                HandJointService.RequestJointTransform(
                        TrackedHandJoint.Palm, handedness)
                .position;
            TryApplyForceFromVectors(previousHandLocation, currentHandPosition);
        }
    }

    return currentHandPosition;
}
```
### The ApplySmashMovement method
This is the heart of the matter:
```cs
private void TryApplyForceFromVectors(Vector3? previousHandLocation, 
                                      Vector3? currentHandPosition)
{
    if (previousHandLocation != null && currentHandPosition != null)
    {
        var handVector = currentHandPosition.Value - previousHandLocation.Value;
        var distanceMoved = Mathf.Abs(handVector.magnitude);
        if (Physics.SphereCast(currentHandPosition.Value, 
                             handSmashingServiceProfile.SmashAreaSize, 
                             handVector, out var hitInfo, 
                             distanceMoved *
                             handSmashingServiceProfile.
                               ProjectionDistanceMultiplier))                
        {
            if (hitInfo.rigidbody != null)
            {
                hitInfo.rigidbody.AddForceAtPosition(
                    handVector * 
                    handSmashingServiceProfile.ForceMultiplier,
                    hitInfo.transform.position);
            }
        }
    }
} 
```
This is the concrete implementation of the principle I described above.
This method
* Calculates a vector from the previous location to the current location. This describes the direction in which the hand is moving, the magnitude is a measure of the speed the hand is traveling. 
* Performs a sphere cast from the current direction of the same length check if something is hit
* Checks if something is hit 
* Applies force to the detected object on the in point, the same direction as the hand is traveling and proportional to the hand speed

Effectively the object is pushed aside as if it's swatted away.

You might notice some values from the profile being factored in along the way. These are values you can set to tweak the process and are explained below:

## The service profile

```cs
using UnityEngine;
using Microsoft.MixedReality.Toolkit;
using Microsoft.MixedReality.Toolkit.Utilities;

namespace MRTKExtensions.Manipulation
{
	[MixedRealityServiceProfile(typeof(IHandSmashingService))]
	[CreateAssetMenu(fileName = "HandSmashingServiceProfile", 
        menuName = "MRTKExtensions/HandSmashingService Configuration Profile")]
	public class HandSmashingServiceProfile : BaseMixedRealityProfile
    {
        [SerializeField]
        private float forceMultiplier = 100;

        public float ForceMultiplier => forceMultiplier;

        [SerializeField]
        private float smashAreaSize = 0.02f;

        public float SmashAreaSize => smashAreaSize;

        [SerializeField]
        private float projectionDistanceMultiplier = 1.1f;

        public float ProjectionDistanceMultiplier => projectionDistanceMultiplier;

        [SerializeField] 
        private Handedness trackedHands = Handedness.Both;

        public Handedness TrackedHands => trackedHands;
    }
}
```

* ForceMultiplier: multiplication of the force applied by the vector. Default I use a 100, so you get a quite nice fast effect.
* SmashAreaSize: the radius of the sphere cast being used. 2 cm radius is 4 cm diameter, so about half the diameter of my hand palm. 
* ProjectionDistanceMultiplier - tweaks the distance of the sphere cast. The current value of 1.1 makes it 10% longer, so if you hand has traveled 10 cm in a frame, it will actually make a sphere cast of 11 cm. Playing with this factor allows you to tackle the situation where an object is *just* a few millimeters from the projected future position of the hand - which might result in swatting right trough the object in the next frame.
* TrackedHands - which hand(s) actually will be tracked - 'both' makes the most sense, but that's up to you.

## Conclusion
I think this is a nice addition to the rich set of hand tracking features the MRKT2 already features. This is meant to be part of a little game I was developing for HoloLens2 just to explore a few avenues I haven't been able to explore so far, but well, it's been a bit busy so this kind of has been sitting on the shelf for a while. You can find it in [this demo project](https://github.com/LocalJoost/HandSmashService) if you would like to play with it.