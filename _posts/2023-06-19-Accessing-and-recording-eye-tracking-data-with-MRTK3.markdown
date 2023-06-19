---
layout: post
title: Accessing and recording eye tracking data with MRTK3
date: 2023-06-19T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- HoloLens2
- Eye tracking
- Unity
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2023-06-19-Accessing-and-recording-eye-tracking-data-with-MRTK3/eytracking.gif
comment_issue_id: 449
---
Recently a community member asked me how to record eye tracking data into a CSV file. I decided to turn the answer into a blog post, so it can benefit the whole community and not only the person asking the question.

I created a little script that uses the MRTK3 Gaze Interactor to shoot a ray from the eyes in the gaze direction until it hits the object of interest, visualizes that point with a little green sphere, and writes the relative coordinates into a csv file on the HoloLens. If you know how to to it, it's extremely simple, but then again - so is almost everything. 

![](/assets/2023-06-19-Accessing-and-recording-eye-tracking-data-with-MRTK3/eytracking.gif)

## EyeTracker script

### Setup 

The setup is pretty easy - in the `Awake` method, the script creates the output file in `Application.persistentDataPath`. Then the prefab that is used to show where the user is looking at is instantiated.



```csharp
private void Awake()
{
    var trackerDataPath = Path.Combine(Application.persistentDataPath,
                                       "eyetracking.csv");
    trackerData = new StreamWriter(trackerDataPath);
    trackerData.AutoFlush = true;
}

private void Start()
{
    hitPointDisplayer = Instantiate(hitPointDisplayPrefab);
}
```

### The actual eye tracking code

The `gazeInteractor` has a property `rayOriginTransform` that basically uses a point more or less between your eyes as  origin, and the `forward` property holds the direction the user is looking at. So then it is simply a matter of shooting a ray ahead 3 max meter from that point the forward direction and see if it actually hits the object of interest. If so, the `hitPointDisplayer`'s origin is set to the place where the ray hits.

```csharp
private void Update()
{
    var ray = new Ray(gazeInteractor.rayOriginTransform.position, 
                      gazeInteractor.rayOriginTransform.forward * 3);
    if (Physics.Raycast(ray, out var hit))
    {
        if (hit.collider.gameObject == objectOfInterest)
        {
            hitPointDisplayer.transform.position = hit.point;
            WriteTrackingPoint(hit.point);
        }
    }
}
```

### Logging the hit point

`WriteTrackingPoint` turns the hit point into a coordinate relative to the object of interest's origin, and then the resulting data is simply written to a file, which is closed off at the moment you stop the app.

```csharp
private void WriteTrackingPoint(Vector3 hitPoint)
{
    var relativePoint = 
        objectOfInterest.transform.position - hitPoint;
    trackerData.WriteLine(FormattableString.Invariant(
        $"{relativePoint.x},{relativePoint.y},{relativePoint.z}"));
}

private void OnDestroy()
{
    trackerData.Close(); 
}
```

## Scene settings

 The EyeTracker script needs access to the MRTK3 gaze interactor, it needs to know what the area of interest is (in the sample: a 40x40x40cm cube rotated by 45° over X and Y), and it needs a 'visualizer', a prefab that appears at the spot where the user is looking at.
 
 ![](/assets/2023-06-19-Accessing-and-recording-eye-tracking-data-with-MRTK3/eyetrackersetup.png)
 
 Note: I have turned off the visualizer's sphere collider, so it does not block the ray to the cube. You can also make this more advanced by putting the object of interest into a separate layer and letting the ray intersect with that layer only, but this works as well.
 
 ![](/assets/2023-06-19-Accessing-and-recording-eye-tracking-data-with-MRTK3/collider.png)

## Getting the tracking file

Access the HoloLens' device portal either by WiFi or by connecting the HoloLens to your PC using a USB cable, go to the File Manager, then browse to 
*LocalAppData\MRTK3EyeTracking_1.0.0.0_arm64__9w1r0z46k1jc2\LocalState*
and there you will find eyetracking.csv.

![](/assets/2023-06-19-Accessing-and-recording-eye-tracking-data-with-MRTK3/filemanager.png)

Mind you, this will be overwritten every time you start the app, and it will only be closed when you explicity close the app. You might want to change that behaviour (for instance, open the file in OnEnable and close in OnDisable). And maybe you want to add a time stamp to the record, and/or limit the number of lines written per time unit - as now you potentially get 60 lines per second, which might be a bit much. But this is the basic general idea.

As I already stated, this is very easy, it's basically a variant of [an age old technique I used to find points on the Spatial Map](https://localjoost.github.io/migrating-to-mrtk2interacting-with/). You can find the sample project [here](https://github.com/LocalJoost/MRTK3EyeTracking).