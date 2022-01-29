---
layout: post
title: Finding the Floor with HoloLens 2 and MRTK 2.7.3
date: 2022-01-29T00:00:00.0000000+01:00
categories: []
tags:
- HoloLens2
- MRKT2
- Unity3d
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2022-01-29-Finding-the-Floor-with-HoloLens-2-and-MRTK-273/floorfinder.gif
comment_issue_id: 403
---
Time flies... this blog is now past it's 14th anniversary, in internet time this means parts of it are ancient - and indeed, it contains it's share of posts that are outdated or even completely irrelevant. But time flies so fast I had not realized even my earlier *HoloLens* related blog posts (the first one is [from June 1st, 2016](https://localjoost.github.io/default-hololens-toolkit-occlusion/)) - are starting to get outdated. And so - when I got [the request](https://github.com/LocalJoost/FloorFinder/issues/1) on how to update floor finding code [I had blogged about in (gulp) November 2017](https://localjoost.github.io/finding-floor-and-displaying-holograms/) to HoloLens 2 and MRTK 2.7.3 - I decided to make an update.

## How the app works
* The app will prompt you to look at the floor
* If it finds a floor, it will show an indicator where it thinks the floor is, and ask you to confirm
* If you say "no" it will try again
* If you say "yes" it will show a the model Cessna 152 [from my previous blog post](https://localjoost.github.io/Some-Blender-tricks-for-the-impatient-Mixed-Reality-HoloLens-developer-(2)/).

![](/assets/2022-01-29-Finding-the-Floor-with-HoloLens-2-and-MRTK-273/floorfinder.gif)

## Base software and configuration
I made the new solution using:
* Unity 2020.3.18f1
* MRKT 2.7.3
* OpenXR

Using the [MRTK Feature tool](https://docs.microsoft.com/en-us/windows/mixed-reality/develop/unity/welcome-to-mr-feature-tool), I installed the following components in the project:
* Mixed Reality Toolkit Foundation 2.7.3
* Mixed Reality Toolkit Standard Assets 2.7.3
* Microsoft Spatializer 1.0.246
* Mixed Reality OpenXR Plugin 1.2.1

I'm going to assume you know how to set up a project using these components - in any case, the MRTK does its best to guide you through the process. If that doesn't work for you, let me know, and I will write another blog post detailing the process.

## Attack of the cloning
If you have worked with the MRKT before, you know the drill - if you want to configure stuff, you will need to do a lot of profile cloning and adapting. 
* First, clone the DefaultHoloLens2ConfigurationProfile (I've called mine AppConfigurationProfile)
* Select Diagnostics at top level, Disable diagnostics so we don't get the profiler float before our eyes by default
* Select Spatial Awareness at top level, and check the Enable Spatial Awareness checkbox
* Clone the DefaultMixedRealitySpatialAwarenessProfile (I have called mine AppSpatialAwarenessSystemProfile)
* Expand the OpenXR Spatial Mesh Observer pane, clone the DefaultMixedRealitySpatialAwarenessMeshObserverProfile - I have called mine AppOpenXRSpatialAwarenessMeshObserverProfile  
* Scroll down to the penXR Spatial Mesh Observer Display Settings, select "Occlusion" at "Display Option"
* Select Input at top level, clone the DefaultHoloLens2InputSystemProfile (I have called mine AppInputSystemProfile)
* Open the Speech Commands pane
* Clone the DefaultMixedRealitySpeechCommandsProfile (I have called mine AppSpeechCommandsProfile)
* Add two speech commands: Yes and No

## Interacting with the Spatial Map
[In August 2019 I already detailed](https://localjoost.github.io/migrating-to-mrtk2interacting-with/) how you can retrieve the Spatial Map layer mask from the MRTK configuration, and how you can find a point *on* that Spatial Map using a ray cast from the head (i.e. the camera) forward. In the folder Assets/MRKTExtensions/Utilities you will find good old `LookingDirectionHelpers` sporting the static method `GetPositionOnSpatialMap` that I usually (and in this sample) just simply call with a number indicating the number of meters the app should search for a spatial map point. 

## The actual looking at and accepting the floor

In the hierarchy, we find this:

![](/assets/2022-01-29-Finding-the-Floor-with-HoloLens-2-and-MRTK-273/floorfinder.png)

On the FloorFinder there's a behaviour called "FloorFinder" as well. This basically comes down to this private method that's called from the `Update` method

```csharp
private void CheckLocationOnSpatialMap()
{
    if (foundPosition == null && Time.time > _delayMoment)
    {
        foundPosition = LookingDirectionHelpers.GetPositionOnSpatialMap(maxDistance);
        if (foundPosition != null)
        {
            if (CameraCache.Main.transform.position.y - foundPosition.Value.y > 1f)
            { 
                lookPrompt.SetActive(false);
                confirmPrompt.transform.position = foundPosition.Value;
                confirmPrompt.SetActive(true);
                locationFoundSound.Play();
            }
            else
            {
                foundPosition = null;
            }
        }
    }
}
```
When no position is found yet and a certain time is passed, try to find a point on the spatial map. If that is at least 1 meter *below* the camera (i.e. the user's head), we assume this is the floor and report the floor is found at this position. It does so by disabling the prompt that asks the user to look at the floor, and enables the prompt that ask the user if this is, indeed, the floor. Both prompts are game objects below the actual FloorFinder game object - one is a simple text saying "Look at the floor", one a rotating arrow with a text "Is this the floor?" above it.

Then there are two public methods, that should be called by speech commands:

```csharp
public void Reset()
{
    _delayMoment = Time.time + 2;
    foundPosition = null;
    lookPrompt.SetActive(true);
    confirmPrompt.SetActive(false);
}

public void Accept()
{
    if (foundPosition != null)
    {
        locationFound?.Invoke(foundPosition.Value);
        lookPrompt.SetActive(false);
        confirmPrompt.SetActive(false);
        gameObject.SetActive(false);
    }
}
```

`Reset` - well, resets the whole behaviour and gives you 2 seconds to actually look at another place, and `Accept` effectively disables both the prompts and the FloorFinder game object itself, and fires a Unity event with the found location to interested parties. If any.

## Placing an object on the found floor location

This, now, is trivial at this point. In HologramCollection, there is this little behaviour:

```csharp
public class ObjectPlacer : MonoBehaviour
{
    [SerializeField]
    private GameObject objectToPlace;

    public void PlaceObject(Vector3 location)
    {
        var obj = Instantiate(objectToPlace, gameObject.transform);
        obj.transform.position = location;
    }
}
```

Whose only public method `PlaceObject` is hooked up to the `FloorFinder`'s LocationFound event like this:

![](/assets/2022-01-29-Finding-the-Floor-with-HoloLens-2-and-MRTK-273/locationfound.png)

to place a model airplane

![](/assets/2022-01-29-Finding-the-Floor-with-HoloLens-2-and-MRTK-273/modelairplane.png)

## Speech commands

This is also completely standard: we have defined the speech command in the profile, and we can simply activate them by putting a `SpeechInputHandler` next to FloorFinder and hook up the speech commands to `Accept` and `Reset`

![](/assets/2022-01-29-Finding-the-Floor-with-HoloLens-2-and-MRTK-273/speechcommands.png)

Just make sure the "Is Focus Required" checkbox is off.

## Bonus content

By default, the DefaultMixedRealitySpatialAwarenessProfile contains three observers, of which only one is actually active in this app - the OpenXR Spatial Mesh Observer. You might as well delete the other two. However, if you look at the observers in this app, you will actually see *four* observers.

![](/assets/2022-01-29-Finding-the-Floor-with-HoloLens-2-and-MRTK-273/objectmeshobserver.png)

I have added the Spatial Object Mesh Observer. This creates a fake spatial mesh *but only in the Unity editor*. This enables you to run the whole application in play mode in the editor, as the ray cast also works on this 'spatial mesh'. [I already blogged about this in May 2020](https://localjoost.github.io/migrating-to-mrtk2-using-spatial-mesh/) but not much people seem to know about it, so I thought it might help if I mention it again.

## Conclusion

Looking for the floor via this method of ray casting did not change that much, chiefly it is more configuring in the MRKT profiles and knowing how to get the layer mask from it. Otherwise it's still pretty simple. I hope I did help Ivan getting started - and any other developer joining the fast-growing ranks of Mixed Reality developers as well.

You can find the [complete code in the original repo, branch "hl2mrkt273".](https://github.com/LocalJoost/FloorFinder/tree/hl2mrkt273)