---
layout: post
title: Positioning QR codes in space with HoloLens 2 - building a 'poor man's Vuforia'
date: 2021-02-03T00:00:00.0000000+01:00
categories: []
tags:
- HoloLens2
- MRKT2
- Unity3d
- Windows Mixed Reality
- QRCode
featuredImageUrl: https://LocalJoost.github.io/assets/2021-01-30-Positioning-QR-codes-in-space-with-HoloLens-2--building-a-'poor-man's-Vuforia'/locationcalculation.png
comment_issue_id: 372
---
In [my previous post](https://localjoost.github.io/Reading-QR-codes-with-an-MRTK2-Extension-Service/) I showed how to read QR codes using native functionality of the HoloLens 2 and the Microsoft.MixedReality.QR package, using a MRTK Extension service to host the actual QR tracking. I also promised to show you how *to position these QR codes in space*, building a kind of 'poor man's Vuforia' - using QR codes in stead of VU markers. And unlike Vuforia, free to use for any purpose, including commercial. This post is the fulfillment of the promise.

What we will build is essentially as analogue to Vuforia as possible. It will recognize a QR code containing the home address of my blog, then put a marker on it - and for good measure, an airplane on top of that marker as well. It will basically be this:

<iframe width="650" height="365" src="https://www.youtube.com/embed/AJ-gc9EalSM" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Scene definition
The general idea is that when a the app recognizes a QR code, it should put a 'marker' on it matching the exact location and rotation in space, and it's apparent physical size. To make that happen, I have defined this game object structure in the Scene hierarchy:

![](/assets/2021-02-03-Positioning-QR-codes-in-space-with-HoloLens-2--building-a-'poor-man's-Vuforia'/scenedef.png)

Important note: *unless otherwise specified* Trackers, it's parent *and* child objects should all have position (0,0,0), rotation (0,0,0) and scale (1,1,1). Otherwise you will make your life possibly a lot more difficult. 

### Tracker1
This is the root object for a tracker. Tracker1 is the first (and only) QR code tracker in this sample, but there is no reason why there should not be more than one.

![](/assets/2021-02-03-Positioning-QR-codes-in-space-with-HoloLens-2--building-a-'poor-man's-Vuforia'/tracker1.png)

It contains only one behaviour: the `QRTrackerController`. This sports two properties you can set via the editor:
* SpatialGraphCoordinateSystemSetter: The `SpatialGraphCoordinateSystemSetter`, the behaviour which will do the actual positioning in space (that I nicked from [the Microsoft sample](https://github.com/chgatla-microsoft/QRTracking/tree/master/SampleQRCodes), then heavily adapted it) - it should be (but does not need to be) on a child object of the tracker object.
* LocationQRValue: the value of the QR code we should check for.

### TrackerHolder
This is the object that is actually going to be positioned in space and scale. It contains only the `SpatialGraphCoordinateSystemSetter` behaviour. 

![](/assets/2021-02-03-Positioning-QR-codes-in-space-with-HoloLens-2--building-a-'poor-man's-Vuforia'/trackerholder.png)

It also has an AudioSource, but this is actually not used by `SpatialGraphCoordinateSystemSetter` but `QRTrackerController`

### Marker
The marker is the thing that is actually displayed when the QR code is recognized.

![](/assets/2021-02-03-Positioning-QR-codes-in-space-with-HoloLens-2--building-a-'poor-man's-Vuforia'/marker.png)

The important thing is that this object is flat and
* has a physical size of 1x1 meter
* its apparent forward direction actually points forward.

I have used a Sprite, and it's apparent size is default 10x10 meter, and default it points *upward*:

![](/assets/2021-02-03-Positioning-QR-codes-in-space-with-HoloLens-2--building-a-'poor-man's-Vuforia'/upward.png)

So it's scaled (0.1,0.1,0.1) and rotated 90 degrees over X

![](/assets/2021-02-03-Positioning-QR-codes-in-space-with-HoloLens-2--building-a-'poor-man's-Vuforia'/forward.png)

## Two behaviours in tandem

Basically we have one behaviour in control, the other being controlled and providing feedback. This basically stems from me initially not understanding the Microsoft Sample at all, and also because I thought it to be a good idea to keep the positioning magic away from the reading of QR codes.
* `QRTrackerController` is the behaviour that is in control. It provides the 'Spatial Graph NodeId' and the physical size of the QR code to:
* `SpatialGraphCoordinateSystemSetter`: this attempts to position the QR code in space, and notifies the `QRTrackerController` of success or failure of trying to do so. 

In this approach things differ a bit from the Microsoft sample. There, the `SpatialGraphCoordinateSystem` continuously tries to follow the QR code and keeps (re)placing the QR code visualizer time and time again. My approach is more the one-shot approach, used to determine fixed points to know where certain key points in reality are. The `QRTrackerController` is control.

## QRTrackerController

This is a behaviour that actually reads the QR codes and checks if it is the desired QR code. 

### Declarations and definitions
It starts as follows:

```csharp
using System;
using Microsoft.MixedReality.Toolkit;
using UnityEngine;

namespace MRTKExtensions.QRCodes
{
    public class QRTrackerController : MonoBehaviour
    {
        [SerializeField]
        private SpatialGraphCoordinateSystemSetter spatialGraphCoordinateSystemSetter;

        [SerializeField]
        private string locationQrValue = string.Empty;

        private Transform markerHolder;
        private AudioSource audioSource;
        private GameObject markerDisplay;
        private QRInfo lastMessage;
        private int trackingCounter;

        public bool IsTrackingActive { get; private set; } = true;

        private IQRCodeTrackingService qrCodeTrackingService;
        
        private IQRCodeTrackingService QRCodeTrackingService
        {
            get
            {
                while (!MixedRealityToolkit.IsInitialized && Time.time < 5) ;
                return qrCodeTrackingService ??
                       (qrCodeTrackingService = 
                         MixedRealityToolkit.Instance.
                          GetService<IQRCodeTrackingService>());
            }
        }
    }
}
```

We see the two properties you can set from the editor, the code to access the QR tracking service, and a couple of fields to hold the components used in the the display process - as well as the last encountered message. And there's `trackingCounter` which I will explain in a bit

The next part is a bit more interesting. 

## Initialization

In the Unity Start method the most important initialization is performed: 
```csharp
private void Start()
{
    if (!QRCodeTrackingService.IsSupported)
    {
        return;
    }
    
    markerHolder = spatialGraphCoordinateSystemSetter.gameObject.transform;
    markerDisplay = markerHolder.GetChild(0).gameObject;
    markerDisplay.SetActive(false);

    audioSource = markerHolder.gameObject.GetComponent<AudioSource>();

    QRCodeTrackingService.QRCodeFound += ProcessTrackingFound;
    spatialGraphCoordinateSystemSetter.PositionAcquired += SetScale;
    spatialGraphCoordinateSystemSetter.PositionAcquisitionFailed += 
        (s,e) => ResetTracking();

    if (QRCodeTrackingService.IsInitialized)
    {
        StartTracking();
    }
    else
    {
        QRCodeTrackingService.Initialized += QRCodeTrackingService_Initialized;
    }
}
```
To recap:
* `markerHolder` is the game object that is going to be placed in space by the `spatialGraphCoordinateSystemSetter`.
* `markerDisplay` contains the 'marker', a graphical element placed on top of the QR code to show its location and orientation. It's immediately set to being invisible at the start because we don't have position for the marker yet.

We set up a few event listeners:
* `QRCodeTrackingService.QRCodeFound` so we know when a QR code is found
* `spatialGraphCoordinateSystemSetter.PositionAcquired` so we know when the position has been successfully determined 
* `spatialGraphCoordinateSystemSetter.PositionAcquisitionFailed` so we know the position could not be determined and we need to try again using the `ResetTracking` method.

The rest of the initialization code is simple and the same as in the previous post:

```csharp
private void QRCodeTrackingService_Initialized(object sender, EventArgs e)
{
    StartTracking();
}

private void StartTracking()
{
    QRCodeTrackingService.Enable();
}
```
Well, except for this little part, that allows us to reset tracking. It can be used from the outside (and it will), but it's also called when the `SpatialGraphCoordinateSystemSetter` fails to determine a position.

```csharp
public void ResetTracking()
{
    if (QRCodeTrackingService.IsInitialized)
    {
        markerDisplay.SetActive(false);
        IsTrackingActive = true;
        trackingCounter = 0
    }
}
```
This code simply hides the marker and enables tracking again.

### Tracking and positioning

The processing of a QR code message happens in `ProcessTrackingFound`:

```csharp
private void ProcessTrackingFound(object sender, QRInfo msg)
{
    if ( msg == null || !IsTrackingActive)
    {
        return;
    }

    lastMessage = msg;

    if (msg.Data == locationQrValue)
    {
        if (trackingCounter++ == 2)
        {
            IsTrackingActive = false;
            spatialGraphCoordinateSystemSetter.SetLocationIdSize(
                msg.SpatialGraphNodeId, msg.PhysicalSideLength);
        }
    }
}
```
It's function is very simple:
* Check if tracking is active and the message is acceptable.
* If so
    * Keep this message for later reference
    * If QR code contains the string we are looking for and has been found twice
        * Disable further tracking until this message is processed
        * Feed the message's `SpatialGraphNodeId` and deteced physical size to the `spatialGraphCoordinateSystemSetter`
        
The `trackingCounter` is a weird thing. I have found out scanning the *first* QR codes always works fine, but the *first* scan of the *second* and following code might return the location of the *previous* scanned code. It does return the correct *size* though. Maybe an error in my code, maybe a lack of understanding from my side - and maybe it's a bug in the Microsoft `QRCodeWatcher`. I can imagine it was was not taken into consideration some dolt - me - might scan the same QR code over and over again on multiple locations close to each other. And it is a preview package after all. At the time of this writing at version 0.5.2103. 

Requiring the user basically scan the same code twice makes it a more difficult to get a good fix (as you might gather from the video), but it's location is now at least reliable. And of course, if you don't show the QR code text displayer all the user may notice is the need to look a bit longer at the same code.
        
Anyway - then it's the `SpatialGraphCoordinateSystemSetter`'s turn. If it fails, it will call `ResetTracking` and the process can start again. If it succeeds, it calls `SetScale`:

```csharp
private void SetScale(object sender, Pose pose)
{
    markerHolder.localScale = Vector3.one * lastMessage.PhysicalSideLength;
    markerDisplay.SetActive(true);
    PositionSet?.Invoke(this, pose);
    audioSource.Play();
}

public EventHandler<Pose> PositionSet;
```
It uses last message's size directly as scale - this is why we needed to keep it, and this also shows why the markerHolder needed to be (1,1,1) and marker needed to be 1x1 meter: the PhysicalSideLength is given in meters - so if you apply that value as the scale, the scaled object will have the same apparent size as the QR code.

Anyway, as you can see it not only applies the scale, it also makes the marker visible, and it calls an event to tell the outside world the correct location and orientation is determined - kind of like it making it behave like Vuforia VU Markers. Also, it plays a sound, but that is just some song and dance - it's not important to the process.

## SpatialGraphCoordinateSystemSetter

### Introduction
This is based upon the [SpatialGraphCoordinateSystem](https://github.com/chgatla-microsoft/QRTracking/blob/unity2018-old/SampleQRCodes/Assets/Scripts/SpatialGraphCoordinateSystem.cs) from the [sample by Microsoft](https://github.com/chgatla-microsoft/QRTracking/tree/master/SampleQRCodes) and I will honestly admit I don't understand some parts of it in detail. I understand it in a black box kind of way: a `Guid` goes in, a location and rotation come out. In global terms I changed a few things
* I made it a one shot affair - it tries to determine a location and it succeeds or fails. It doesn't try to continuously find the location, like the original does. 
* In either case (success or failure) it calls an *event* to notify the outside world of what happened: `PositionAcquired` when it has successfully calculated a position, and `PositionAcquisitionFailed` when it has failed.
* The *location* returned is the *center* of the QR code - not the top (like in the Microsoft sample) - once again making it more behave like Vuforia
* The *rotation* returned has *forward* pointing horizontally from the center to the top of the QR code, and *up* points up from the flat surface of the QR code. 
* Finally, I have moved as much out from the between `#if WINDOWS_UWP` and `#endif` preprocessor directives to have more IntelliSense joy 

The difference difference in location and rotation calculation between the original code (left) and mine (right). 

![](/assets/2021-02-03-Positioning-QR-codes-in-space-with-HoloLens-2--building-a-'poor-man's-Vuforia'/locationcalculation.png)

I have the audacity to claim my version yields a more logical result, but you are of course free to disagree ;)

### Declarations & stuff

Let's start at the top. 
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.MixedReality.Toolkit.Utilities;
using UnityEngine;
using UnityEngine.XR.WSA;
#if WINDOWS_UWP
using Windows.Perception.Spatial;
#endif

namespace MRTKExtensions.QRCodes
{
    public class SpatialGraphCoordinateSystemSetter : MonoBehaviour
    {
        public EventHandler<Pose> PositionAcquired;
        public EventHandler PositionAcquisitionFailed;

        private Tuple<Guid, float> locationIdSize = null;

        private PositionalLocatorState CurrentState { get; set; }
    }
}
```
Here we see the two events that I have already discussed. There is also the Tuple  `Tuple<Guid, float>>.` This is where I store the input provided by the outside world. `CurrentState` I nicked from the Microsoft sample. It's use will become clear in the next section

### Initialization
I basically nicked this from the Microsoft sample, I made it a bit simpler.
```csharp
void Awake()
{
    CurrentState = PositionalLocatorState.Unavailable;
}

void Start()
{
    WorldManager.OnPositionalLocatorStateChanged +=
       WorldManager_OnPositionalLocatorStateChanged;
    CurrentState = WorldManager.state;
}

private void WorldManager_OnPositionalLocatorStateChanged(
                PositionalLocatorState oldState, PositionalLocatorState newState)
{
    CurrentState = newState;
    gameObject.SetActive(newState == PositionalLocatorState.Active);
}
```
What this basically does is setting the value of `CurrentState`, and based upon that state it activates and deactivates the current game object (which is MarkerHolder, remember?)

### Kicking off a localization
This is once again made by yours truly

```csharp
public void SetLocationIdSize(Guid spatialGraphNodeId, float physicalSideLength)
{
    if (locationIdSize == null)
    {
        locationIdSize = new Tuple<Guid, float>(spatialGraphNodeId,
                                                physicalSideLength);
    }
}

void Update()
{
    if (locationIdSize != null)
    {
        UpdateLocation(locationIdSize.Item1, locationIdSize.Item2);
        locationIdSize = null;
    }
}
```

You might remember `SetLocationIdSize` is called by `QRTrackerController` when it detects a QR code. The data is simply put into the `Tuple`. The `Update` method, called 60 times a second by the Unity game loop, will pop it and put it into the `UpdateLocation`. The `== null` check in `SetLocationIdSize` is to prevent race conditions.

### Calculating a location

Like I said, most of this is black magic to me. I have rewritten this part to be a bit simpler and made it call `PositionAcquisitionFailed` when it fails (in the sample goes in another loop) - which apparently can occur in three cases. 

```csharp
private void UpdateLocation(Guid spatialGraphNodeId, float physicalSideLength)
{
    if (CurrentState != PositionalLocatorState.Active)
    {
        PositionAcquisitionFailed?.Invoke(this, null);
        return;
    }

    System.Numerics.Matrix4x4? relativePose = System.Numerics.Matrix4x4.Identity;
    
#if WINDOWS_UWP
    SpatialCoordinateSystem coordinateSystem =
      Windows.Perception.Spatial.Preview.SpatialGraphInteropPreview.
        CreateCoordinateSystemForNode(spatialGraphNodeId);

    if (coordinateSystem == null)
    {
        PositionAcquisitionFailed?.Invoke(this, null);
        return;
    }

    SpatialCoordinateSystem rootSpatialCoordinateSystem =
      (SpatialCoordinateSystem)System.Runtime.InteropServices.Marshal.
          GetObjectForIUnknown(WorldManager.GetNativeISpatialCoordinateSystemPtr());

    // Get the relative transform from the unity origin
    relativePose = coordinateSystem.TryGetTransformTo(rootSpatialCoordinateSystem);
#endif

    if (relativePose == null)
    {
        PositionAcquisitionFailed?.Invoke(this, null);
        return;
    }

    System.Numerics.Matrix4x4 newMatrix = relativePose.Value;

    // Platform coordinates are all right handed and unity uses left handed matrices. 
    // so we convert the matrix
    // from rhs-rhs to lhs-lhs 
    // Convert from right to left coordinate system
    newMatrix.M13 = -newMatrix.M13;
    newMatrix.M23 = -newMatrix.M23;
    newMatrix.M43 = -newMatrix.M43;

    newMatrix.M31 = -newMatrix.M31;
    newMatrix.M32 = -newMatrix.M32;
    newMatrix.M34 = -newMatrix.M34;

    System.Numerics.Vector3 scale;
    System.Numerics.Quaternion rotation1;
    System.Numerics.Vector3 translation1;

    System.Numerics.Matrix4x4.Decompose(newMatrix, out scale, out rotation1, 
                                        out translation1);
    var translation = new Vector3(translation1.X, translation1.Y, translation1.Z);
    var rotation = new Quaternion(rotation1.X, rotation1.Y, rotation1.Z, rotation1.W);
    var pose = new Pose(translation, rotation);

    // If there is a parent to the camera that means we are using teleport and we
    // should not apply the teleport to these objects so apply the inverse
    if (CameraCache.Main.transform.parent != null)
    {
        pose = pose.GetTransformedBy(CameraCache.Main.transform.parent);
    }
```
As far as I can see spatialGraphNodeId, containing a `Guid`, goes in; this returns a `SpatialCoordinateSystem` whatever that may be, then it gets another `SpatialCoordinateSystem` that is the root coordinate system, and then it does some matrix calculus that is a bit over my head - finally resulting in a `Pose` (which holds a rotation and a position). This `Pose` returns the top left point of the QR code like I said, and the rotation which makes an object point upwards out of the QR code.

**NOTE: this code is nicked from the unity2018-old branch from the Microsoft sample, because I am using 'Legacy XR'.** I already mentioned that in the previous blog.

The next part is crystal clear, because I wrote that myself ;)
```csharp
// Rotate 90 degrees 'forward' over 'right' so 'up' is pointing straight up from the QR code
pose.rotation *= Quaternion.Euler(90, 0, 0);

// Move the anchor point to the *center* of the QR code
var deltaToCenter = physicalSideLength * 0.5f;
pose.position += (pose.rotation * (deltaToCenter * Vector3.right) -
                  pose.rotation * (deltaToCenter * Vector3.forward));
gameObject.transform.SetPositionAndRotation(pose.position, pose.rotation);
PositionAcquired?.Invoke(this, pose);
```
I have this simple cheat sheet on my computer:
* X = right
* Y = up
* Z = forward

As you could see in the picture, the airplane was basically upside down pointing upward. So we want to rotate that over the left-to-right axis, forward 90&deg; - *in it's own reference frame*. To rotate forward over the left-to-right axis - that's X, so we simply multiply the rotation found by the mystery algorithm by a `Quaternion.Euler(90, 0, 0)`.

All right, so now the orientation is okay, but it's still sitting on the top `left corner` of the QR code in stead of on the `center`. So we have to move it
* half the size of the QR code to the right
* half the size of the QR code backward. Or actually, minus half the size of the QR code forward as you can see in the code';)

To move an object in its own reference frame, simply add (or subtract) the rotation times a world vector, as shown in the code.

This whole exercise was very enlightening to me and actually improved my knowledge of rotation calculation.

### Recap
* `QRTrackerController` listens to 
    * `QRCodeTrackingService.QRCodeFound`
    * `SpatialGraphCoordinateSystemSetter.PositionAcquired`
    * `SpatialGraphCoordinateSystemSetter.PositionAcquisitionFailed`
* If a QR code is found, `QRTrackerController` calls `SpatialGraphCoordinateSystemSetter.SetLocationIdSize`
* If a position is found, `SpatialGraphCoordinateSystemSetter.PositionAcquired` gets fired and the marker is displayed on the right place and scale
* If a position is *not* found, `SpatialGraphCoordinateSystemSetter.PositionAcquisitionFailed` gets fired and `QRTrackerController` waits for another QR code to give to `SpatialGraphCoordinateSystemSetter`

## Some bits and pieces
`QRCodeTrackingService`, `QRTrackerController` and `SpatialGraphCoordinateSystemSetter` work together and act kind of like Vuforia. Actually making *use* of found simply requires observing the `PositionSet` event of the `QRTrackerController`. This happens in the `JetController`

```csharp
using MRTKExtensions.QRCodes;
using UnityEngine;

public class JetController : MonoBehaviour
{
    [SerializeField]
    private QRTrackerController trackerController;

    private void Start()
    {
        trackerController.PositionSet += PoseFound;
    }

    private void PoseFound(object sender, Pose pose)
    {
        var childObj = transform.GetChild(0);
        childObj.SetPositionAndRotation(pose.position, pose.rotation);
        childObj.gameObject.SetActive(true);
    }
}
```
After all is set up, it's simply the matter of tracking one event of the tracker you are interested in. This behaviour in put on an empty game object with a very much scaled down airplane mode; 

![](/assets/2021-02-03-Positioning-QR-codes-in-space-with-HoloLens-2--building-a-'poor-man's-Vuforia'/jetholder.png)

To be more specific - the Boeing 737 you might recognize from my [AMS HoloATC](https://www.microsoft.com/en-us/p/ams-holoatc/9nblggh52szp) and/or [ATL HoloATC](https://www.microsoft.com/en-us/p/atl-holoatc/9pdj25rkjnqj) app ;)

![](/assets/2021-02-03-Positioning-QR-codes-in-space-with-HoloLens-2--building-a-'poor-man's-Vuforia'/boeing737.png)

And finally there's the hand menu that has no code at all:

![](/assets/2021-02-03-Positioning-QR-codes-in-space-with-HoloLens-2--building-a-'poor-man's-Vuforia'/handmenu.png)

It has only one button - Reset - which calls the public ResetTracking method of the `QRTrackerController` in the Tracker1 game object.

## Conclusion

And that's that. That is all there is to reading and positioning of QR codes with HoloLens 2. I hope separating out the actual reading and positioning makes stuff a bit clearer to you - it did to me - and it certainly makes it a bit more manageable and reusable. 

Lessons learned while testing QR code positioning:
* Make sure the QR code is properly lighted. If your head casts a shadow on it, it's harder for the HoloLens to pick it up
* 10x10cm is a size that works very well. 6x6 cm can work, but takes more more effort to get scanned. Anything below that - like 4x4cm - is just pestering your users and will probably just get you a lot of support calls.  
* The simpler, the better. Less QR code payload = more coarse QR code = easier scanned.
* For continuous tracking of QR codes - like you can do with Vuforia - the internal QR code tracker is simply too slow (still). But for one shot positioning, determining real locations (and rotations) in a 3D environment and aligning your holographic space accordingly, as I have done for a few commercial projects, this is very usable and reliable. 

You can view and download the full code at [here](https://github.com/LocalJoost/QRCodeService/tree/blog2).