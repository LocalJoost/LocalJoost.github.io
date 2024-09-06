---
layout: post
title: Cross-platform spatial QR code tracking for HoloLens 2 and Magic Leap 2 with a ServiceFramework Service, part 2
date: 2024-09-06T00:00:00.0000000+02:00
categories: []
tags:
- HoloLens 2
- Magic Leap 2
- cross-platform
- Service Framework
comment_issue_id: 474
---
[As promised on August 18](https://localjoost.github.io/Cross-platform-spatial-QR-code-tracking-for-HoloLens-2-and-Magic-Leap-2-with-a-ServiceFramework-Service,-part-1/), this blog contains a more detailed description of the inner workings of cross platform QR code tracking for HoloLens 2 and Magic Leap 2.

## General procedure for building cross-platform HL2 & ML2 apps.

Whenever I intend to write something that is supposed to work on both HoloLens 2 and Magic Leap 2, I usually follow the following procedure:
1. Create a piece of code that works on HoloLens 2, and commit this to a repo
2. Create a branch of the HoloLens code, 
3. Adapt the project up the project for Magic Leap 2 [per this procedure](https://localjoost.github.io/Making-an-MRTK3-based-HoloLens-2-app-run-on-Magic-Leap-2/)
4. Create code that is specific for Magic Leap 2
5. If changes affect how the HoloLens 2 code is set up, back port that code to the HoloLens 2 branch, then merge the HoloLens 2 branch back into the Magic Leap 2 branch 

## Shared code

### Data classes

As we have seen in the previous post, the services return an ITrackedMarkerChangedEventArgs. This, of course, has an implementation as well:

```csharp
public class TrackedMarkerArgs : ITrackedMarkerChangedEventArgs
{
    public TrackedMarkerArgs()
    {
        MarkersInternal = new List<ITrackedMarker>();
    }
    
    public IReadOnlyList<ITrackedMarker> Markers => MarkersInternal;
    
    internal List<ITrackedMarker> MarkersInternal { get; set; }
}
```

This basically only obscures the `MarkersInternal` property from the outside world. 

Something similar goes for `ITrackedMarker`, this also has an implementation `TrackedMarker`, but this has properties with *setters*, unlike `ITrackedMarker`.

```csharp
public class TrackedMarker : ITrackedMarker
{
    public string Id { get; set; }
    public string Payload { get; set; }
    public Pose Pose { get; set; }
    public Vector2 Size { get; set; }
    public float LastSeenTime { get; set; }
    public bool IsTracked { get; set; }
}
```

### The base service

The class `BaseMarkerTrackingService` contains, not surprisingly, the base class for a tracking service.

```csharp
public abstract class BaseMarkerTrackingService : BaseServiceWithConstructor
{
    protected readonly TrackedMarkerServiceProfile Profile;
    protected readonly Dictionary<string, TrackedMarker> KnownMarkers = new();
    protected float LastEventUpdate;

    public event Action<ITrackedMarkerChangedEventArgs> MarkersChanged;
    
    protected BaseMarkerTrackingService(string name, uint priority, TrackedMarkerServiceProfile profile)
        : base(name, priority)
    {
        Profile = profile;
        LastEventUpdate = -profile.TrackingLostTimeOut;
    }
```
It is a rather bog standard implementation of a Service Framework Service - albeit an abstract one. Notice its `KnownMarkers` dictionary, used to keep track of previously seen markers, and the `LastEventUpdate` float that will be particularly useful for *removing* markers that we have not see for a while.

```csharp
public override void Update()
{
    if (Time.time - LastEventUpdate > Profile.TrackingLostTimeOut)
    {
        NotifyMarkersChanged(CreateMarkerArgs());
    }
}

protected void NotifyMarkersChanged(TrackedMarkerArgs args)
{
    MarkersChanged?.Invoke(args);
    LastEventUpdate = Time.time;
}
```

The service's `Update` method is used to notify changes at least every `Profile.TrackingLostTimeOut` seconds. This is because although both devices clearly indicate markers are first detected or updated, the actual disappearance of markers, especially on HoloLens where the process is event-driven, is not so very eagerly announced. This is effectuated in the next two methods:

```csharp
protected TrackedMarkerArgs CreateMarkerArgs()
{
    PruneUntrackedMarkers();

    var result = new TrackedMarkerArgs();
    result.MarkersInternal.AddRange(KnownMarkers.Values);
    return result;  
}

private void PruneUntrackedMarkers()
{
    foreach( var marker in KnownMarkers.Values)
    {
        if( marker.LastSeenTime < Time.time - Profile.TrackingLostTimeOut)
        {
            marker.IsTracked = false;
        }
    }
}
```

First, we remove markers that have not been updated during `Profile.TrackingLostTimeOut` seconds, assuming tracking is lost. Then we simply add all known markers left to the event. This, of course, begs the question - when are markers actually *added* to, or updated in the `KnownMarkers` dictionary. That now, is the task of the actual implementation classes.

## HoloLens 2 implementation

HoloLens 2 uses an `ARMarkerManager` behaviour from the Microsoft Mixed Reality OpenXR implementation. First step, therefore, is to create an empty game object "MarkerManager" and add an `ARMarkerManager` behaviour to it. But that needs an `XROrigin` to be present, so we add that first.

```csharp
public class HL2QrCodeTrackingService : BaseMarkerTrackingService, IMarkerTrackingService
    {
        public HL2QrCodeTrackingService(string name, uint priority, TrackedMarkerServiceProfile profile)
            : base(name, priority, profile)
        {
        }
        
#if UNITY_WSA        
        private ARMarkerManager arMarkerManager;

        public override void Enable()
        {
            if (arMarkerManager == null)
            {
                var markerManagerGameObject = new GameObject
                {
                    name = "MarkerManager"
                };
                markerManagerGameObject.AddComponent<XROrigin>();
                arMarkerManager = markerManagerGameObject.AddComponent<ARMarkerManager>();
                arMarkerManager.defaultTransformMode = TransformMode.Center;
            }
            arMarkerManager.markersChanged += OnMarkersChanged;
        }
```
When all objects and behaviours are created, we set the `arMarkerManager`'s default transform mode to center, making sure the returned marker position matches the center of the QR code. And then a attach an event listener to it, and are good to go.

The `OnMarkersChanged` is very simple:

```csharp
private void OnMarkersChanged(ARMarkersChangedEventArgs changedEventArgs)
{
    NotifyMarkersChanged(ToMarkerArgs(changedEventArgs));
}
```

it basically converts the HoloLens 2 specific marker data to our general marker data structure, using `ToMarkerArgs`:

```csharp
private TrackedMarkerArgs ToMarkerArgs(ARMarkersChangedEventArgs args)
{
    foreach (var marker in args.added.Concat(args.updated))
    {
        var id = marker.trackableId.ToString();
        if( KnownMarkers.TryGetValue(id, out var knownMarker))
        {
            UpdateMarker(marker, knownMarker);
        }
        else
        {
            KnownMarkers.Add(id, CreateNewMarker(marker, id));
        }
    }

    return CreateMarkerArgs();
}
```
The HoloLens `ARMarkersChangedEventArgs` basically has three properties of interest:
* added
* updated
* removed

I have not reliably been able to determine when markers actually appear in the `removed` list, so I have chosen to ignore this and implement my own mechanism, the `PruneUntrackedMarkers` method in the base class. As a bonus, this makes both device work in a consistent manner. So basically we process both added and updated markers as a single list, comparing them against the `KnownTrackers` dictionary in the base class. For `Id` we choose the id HoloLens 2 supplies - I have not checked what it actually is, but it is apparently unique. After we either created or updated markers, we call the base class' `CreateMarkerArgs` to notify the outside world.

`CreateMarker` creates - duh - a new marker for every new marker, calling the `GetDecodedString` to extract payload...

```csharp
private TrackedMarker CreateNewMarker(ARMarker arMarker, string id)
{
    var marker = new TrackedMarker
    {
        Id = id,
        Payload = arMarker.GetDecodedString(),
        IsTracked = true
    };
    UpdateMarker(arMarker, marker);
    return marker;
}
```

... then uses UpdateMarker to set or update it's position in space:

```csharp
private void UpdateMarker(ARMarker marker, TrackedMarker trackedMarker)
{
    trackedMarker.Pose = new Pose(marker.transform.position, 
        marker.transform.rotation * Quaternion.Euler(90, 0, 0));
    trackedMarker.Size = marker.size;
    trackedMarker.LastSeenTime = marker.lastSeenTime;
    trackedMarker.IsTracked = true;
}
```

We can extract position and rotation from the `ARMarker`, but for some reason we have to rotate the position 90 degrees over X to get a marker that actually shows in the right rotation. HoloLens 2 is nice enough to supply us with the last time it has seen the tracker, we update `IsTracked` to true, and we are done. Our app can now track QR codes.

Note, trackers that are removed are ignored. The thing is - if no trackers are updated, apparently the `ArMarkerManager.markersChanged` does not always fire. This would cause markers that are seen once but never updated to be viewed as tracked at the last position almost indefinitely. But remember, the deletion of markers the device has lost track of are handled by base class' `Update` method, that after `Profile.TrackingLostTimeOut` seconds sets a marker's `IsTracked` property to false. So although it's remains in the KnownMakers dictionary for the duration of the app's run. It's up to the calling code to take action on that - like deleting a visualizing game object, as the samples in my previous blog post do.

Some minor details: when the service is disabled or destroyed, clean up the mess:

```csharp
public override void Disable()
{
    if(arMarkerManager != null)
    {
        arMarkerManager.markersChanged -= OnMarkersChanged;
    }
}

public override void Destroy()
{
    if(arMarkerManager != null)
    {
        arMarkerManager.markersChanged -= OnMarkersChanged;
        arMarkerManager.gameObject.Destroy();
    }
}
```

## Magic Leap 2 implementation

This is, in all honesty, simply [a variant of Magic Leap's Marker Tracking Sample](https://developer-docs.magicleap.cloud/docs/guides/unity/marker-tracking/marker-tracker-example/) changed to fit into this my own little cross-platform architecture.

It has a few private member of is own:

```csharp
private MagicLeapMarkerUnderstandingFeature markerFeature;
private bool isInitialized;
private XROrigin xrOrigin;
private MarkerDetectorSettings detectorSettings;
```

`MagicLeapMarkerUnderstandingFeature` is Magic Leap's implementation of the OpenXR marker tracking feature, while `MarkerDetectorSettings` is a structure needed to tell the tracker what markers need to be tracked, and set options for that. As I said before: unlike HoloLens, Magic Leap can track more than QR codes natively, and although my code supports only two of them, it can easily be extented to support most of the other ones.

The whole initialization and configuration of the tracking process happens in the service's `OnEnable` method, which is pretty long, so I will address it in two parts. It starts pretty simple, with code 1:1 stolen from the sample:

```csharp
public override void Enable()
 {
     if (isInitialized)
     {
         return;
     }

     markerFeature = OpenXRSettings.Instance.GetFeature<MagicLeapMarkerUnderstandingFeature>();
     xrOrigin = Object.FindAnyObjectByType<XROrigin>();
     if (xrOrigin == null)
     {
         throw new MissingComponentException("No XR Origin Found, markers will not work.");
     }

     if (markerFeature == null || !markerFeature.enabled)
     {
         throw new MissingComponentException(
             "The Magic Leap 2 Marker Understanding OpenXR Feature is missing or disabled.");
     }
```
The only addition by me is the prevention of re-initialization. Note that this code does not add an XROrigin but tries to find an existing one, and since the MRTK3 camera rig has one, it will find it allright.

Next part is the actual configuration:

```csharp
detectorSettings.MarkerDetectorProfile = Profile.DetectorProfile;
detectorSettings = new MarkerDetectorSettings();
if (Profile.EnableQR)
{
    detectorSettings.QRSettings.EstimateQRLength = true;
    detectorSettings.MarkerType = MarkerType.QR;
    markerFeature.CreateMarkerDetector(detectorSettings);
}

if (Profile.EnableAruco)
{
    detectorSettings.ArucoSettings.EstimateArucoLength = true;
    detectorSettings.ArucoSettings.ArucoType = ArucoType.Dictionary_5x5_50;
    detectorSettings.MarkerType = MarkerType.Aruco;
    markerFeature.CreateMarkerDetector(detectorSettings);
}

isInitialized = true;
```

Based upon service's configuration, it sets the Magic Leap 2 [detector profile](https://developer-docs.magicleap.cloud/docs/guides/unity/marker-tracking/marker-tracker-api/#marker-tracker-profile), then proceeds to configure the desired markers to be tracked. As said before, my code only supports QR and Aruco, but other trackers can be added easily in this code, although like I also says, that breaks the cross platform philosophy a bit. 

Unlike HoloLens, Magic Leap does not provide an event that indicate markers are tracked, updated or deleted: it's a polling approach, depending on the calling code *asking* for marker data. That, of course  driven by the Update method:

```csharp
public override void Update()
{
    if (!isInitialized || !IsEnabled)
    {
        return;
    }

    var shouldNotify = false;
    markerFeature.UpdateMarkerDetectors();

    foreach (var detector in markerFeature.MarkerDetectors)
    {
        if (detector.Status == MarkerDetectorStatus.Ready && ProcessDetector(detector))
        {
            shouldNotify = true;
        }
    }

    if (shouldNotify)
    {
        NotifyMarkersChanged(CreateMarkerArgs());
    }
    else
    {
        base.Update();
    }
}
```

Basically: if any new or updated markers are found, shouldNotify becomes through the outside world is notified. If *not*, the base method is called, which was, for reference:

```csharp
public override void Update()
{
    if (Time.time - LastEventUpdate > Profile.TrackingLostTimeOut)
    {
        NotifyMarkersChanged(CreateMarkerArgs());
    }
}
```

So at least, if nothing changes, every `Profile.TrackingLostTimeOut` seconds the `NotifyMarkersChanged` method is called, to remove possible lost trackers. 

Detecting and updating tracker data internally is done by the `ProcessDetector` method, as you have already seen above, which is simply this:

```csharp
private bool ProcessDetector(MarkerDetector detector)
{
    var detectedMarkers = 0;
    foreach (var data in detector.Data)
    {
        var id = GetId(data, detector.Settings.MarkerType);
        if (data.MarkerPose.HasValue)
        {
            detectedMarkers++;
            if (!KnownMarkers.TryGetValue(id, out var trackedMarker))
            {
                trackedMarker = CreateNewMarker(detector, data);
                KnownMarkers.Add(id, trackedMarker);
            }
            else
            {
                UpdateMarker(data, trackedMarker);
            }
        }
    }

    return detectedMarkers > 0;
}
```

Note: apparently Magic Leap can detect and report codes that are seen, but not have a position determined yet. In that case, we simply ignore it. And since the API does not provide an ID like HoloLens does, we make one ourselves

```csharp
private string GetId(MarkerData arMarker, MarkerType markerType)
{
    return markerType switch
    {
        MarkerType.Aruco => arMarker.MarkerNumber.ToString(),
        MarkerType.QR => arMarker.MarkerString,
        _ => null
    };
}
```
Simply based upon the payload of the QR code, or the marker number for an Aruco marker.

Anyway, all that's left now is the `CreateMarker` and `UpdateMarker` methods, which are very similar to the HoloLens implementation, they just call slightly different APIs. `CreateMarker` simply loads the data from the MarkerData object, and sets id and payload to be equal:

```csharp
private TrackedMarker CreateNewMarker(MarkerDetector detector, MarkerData markerData)
{
    var id = GetId(markerData, detector.Settings.MarkerType);
    var marker = new TrackedMarker
    {
        Id = id,
        Payload = id,
        IsTracked = true
    };
    UpdateMarker(markerData, marker);
    return marker;
}
```

UpdateMarker is slightly more elaborate than the HoloLens equivalent: 

```csharp
private void UpdateMarker(MarkerData markerData, TrackedMarker trackedMarker)
{
    var originTransform = xrOrigin.CameraFloorOffsetObject.transform;
    trackedMarker.Pose = new Pose(
        originTransform.TransformPoint(markerData.MarkerPose.Value.position),
        (originTransform.rotation * markerData.MarkerPose.Value.rotation) * Quaternion.Euler(-90, 0, 0));
    trackedMarker.Size = new Vector2(markerData.MarkerLength, markerData.MarkerLength);
    trackedMarker.LastSeenTime = Time.time;
    trackedMarker.IsTracked = true;
}
```

We apparently have to transform both rotation and with repect to the `xrOrigin.CameraFloorOffsetObject`, and then - just like in HoloLens - we have to rotate the transform over X to get the right rotation, but in this case -90 degrees, in stead of the +90 degrees we needed in HoloLens. Don't ask me why - in both cases.

Finally, the Destroy method needs to clear up some stuff as well, just as in HoloLens, only *different* stuff:

```csharp
public override void Destroy()
{
    base.Destroy();
    if (markerFeature != null)
    {
        markerFeature.DestroyAllMarkerDetectors();
    }
}
```

## Concluding words

I am happy to see the XR providers embrace OpenXR, if only because it makes a lot of things simpler and the availability of features becoming more standard. And I love to see the power of the [Service Framework](https://serviceframework.realitycollective.net/docs/get-started/) making it possible to smooth over API differences to a point where it really does not matter what device you use. Ever since [I discovered it's predecessor in late 2018](https://localjoost.github.io/mixed-reality-toolkit-vnextdependency/), part of was then still called "MRTK-vNext" and what was to become MRTK 2, it has been a favorite and often-used way for me to build robust, reusable, easy to use and now cross-platform pieces of code. If you have never used Service Framework before, I urge you to take a look at it. They also provide [a simple sample project](https://github.com/realitycollective/ServiceFramework-Navigator-Sample-Project). 

Note: the code has not actually been tested with Aruco codes, just QR codes, as these are my primary use case.
