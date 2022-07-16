---
layout: post
title: MRTK2 to MRTK3 - intercepting a raw air tap like with IMixedRealityPointerHandler
date: 2022-06-19T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- HoloLens2
- Unity
- Windows Mixed Reality
- Service Framework
- Reality Collective
featuredImageUrl: https://LocalJoost.github.io/assets/2022-06-19-MRTK2-to-MRTK3--intercepting-a-raw-'air-tap'-like-with-IMixedRealityPointerHandler/controllerlookup.png
comment_issue_id: 421
---
**NOTE: there is a [simpler way of doing this](../MRTK2-to-MRTK3-a-MUCH-SIMPLER-way-to-intercept-a-raw-air-tap-like-with-IMixedRealityPointerHandler/)**

In ye olden days it was simple: you created a behaviour that implemented `IMixedRealityPointerHandler` and you got this methods to play with:

```csharp
public interface IMixedRealityPointerHandler : IEventSystemHandler
{
  void OnPointerDown(MixedRealityPointerEventData eventData);

  void OnPointerDragged(MixedRealityPointerEventData eventData);

  void OnPointerUp(MixedRealityPointerEventData eventData);

  void OnPointerClicked(MixedRealityPointerEventData eventData);
}
```

## So how does this work now

The new MRTK3 works with Interactables and Interactors, which is all very fine, but what if you want to have the user perform a regular air tap as a confirmation gesture, without anything specific to tap *on*? Now `IMixedRealityPointerHandler` is gone, this is not straightforward.

Basic procedure is as follows:
* You have to find a ControllerLookup in your scene
* This has both a LeftHandController and a RightHandController property of type `ArticulatedHandController`
* If you look at `controller.currentControllerState.selectInteractionState.value` you get a value between 0 and 1 that gives you an idea if the user is performing an air tap, and if to, how far they are with it.

If you feel that's a bit complicated for just simply determining whether or not an air tap is being performed, I tend to agree. So I decided to write a little [Reality Collective Service Framework](https://service-framework.realitycollective.io/) service to make this a wee bit easier. It's provisionally called `MRTK3ConfigurationFindingService` because it's primary thing was finding the MRTK3 Configuration in a central spot. 

## Finding the configuration

To actually get hold of the hands, you first have to find the ControllerLookup. Unlike the MRTK2, things like this are not hosted in a service, but in game object that sits in your scene, inside the MRTK XR Rig:

![](/assets/2022-06-19-MRTK2-to-MRTK3--intercepting-a-raw-air-tap-like-with-IMixedRealityPointerHandler/controllerlookup.png)

To find a reference to it, I use this piece of code, nicked and adapted from the MRTK re-implementation of the `Solver` class

```csharp
public override void Start()
{
    GetHandControllerLookup();
}

#region Nicked from Solver

private ControllerLookup controllerLookup;

public ControllerLookup ControllerLookup => controllerLookup;

private void GetHandControllerLookup()
{
    if (controllerLookup == null)
    {
        ControllerLookup[] lookups =
            GameObject.FindObjectsOfType(typeof(ControllerLookup)) as
              ControllerLookup[];
        if (lookups.Length == 0)
        {
            Debug.LogError(
                "Could not locate an instance of the ControllerLookup ...");
        }
        else if (lookups.Length > 1)
        {
            Debug.LogWarning(
                "Found more than one instance of the ControllerLookup ....");
            controllerLookup = lookups[0];
        }
        else
        {
            controllerLookup = lookups[0];
        }
    }
}

#endregion
```

And yes, `GameObject.FindObjectsOfType` is pretty expensive from a performance standpoint, but this way, it's done once and only once, in a single service, at startup. And there is no chance of race conditions.

## Obtaining a reference to the hands
This is pretty simple. Once you have gotten hold of the ControllerLookup, you will notice it actually has properties for left and right hand. To get access to the full hand properties and events, I cast them to their actual type - `ArticulatedHandController` 

```csharp
public ArticulatedHandController LeftHand => 
    (ArticulatedHandController)controllerLookup.LeftHandController;
public ArticulatedHandController RightHand => 
    (ArticulatedHandController)controllerLookup.RightHandController;

```

## Intercepting the air tap

Now, to actually check what the hand is doing, we need this code:

```csharp
public override void Update()
{
    var newStatus = GetIsTriggered(LeftHand);
    if (newStatus != leftHandTriggerStatus)
    {
        leftHandTriggerStatus = newStatus;
        LeftHandStatusTriggered.Invoke(leftHandTriggerStatus);
    }

    newStatus = GetIsTriggered(RightHand);
    if (newStatus != rightHandTriggerStatus)
    {
        rightHandTriggerStatus = newStatus;
        RightHandStatusTriggered.Invoke(rightHandTriggerStatus);
    }
}

private bool GetIsTriggered(ArticulatedHandController hand)
{
    return hand.currentControllerState.selectInteractionState.value > 0.95f;
}
```

Apparently, the hand's `currentControllerState.selectInteractionState.value`is an indication to how far the user has completed the air tap gesture. 0 is none at all (open hand, or at least no tap) and 1 is a full pinch-like gesture. Now this:

```csharp
private bool leftHandTriggerStatus;
private bool rightHandTriggerStatus;

public UnityEvent<bool> LeftHandStatusTriggered { get; } = 
  new UnityEvent<bool>();
public UnityEvent<bool> RightHandStatusTriggered { get; } = 
  new UnityEvent<bool>();
```
the fields that are used keep track of whether or not the status flips, and the events are used to tell the world what happened: the hand gesture is passing the  the pinch threshold - or falling back back under it. These events correspond to `IMixedRealityPointerHandler`'s `OnPointerUp` and `OnPointerDown`. You can now simply add a listener to these events to get notified of what the hands are doing:

## The proof of the pudding 

So I snitched some assets from MRTK2, and created this little behaviour to show off it actually works:

```csharp
public class AirTapDisplayer : MonoBehaviour
{
    [SerializeField]
    private GameObject leftHand;
    
    [SerializeField]
    private GameObject rightHand;
    
    
    private IMRTK3ConfigurationFindingService findingService;
    private void Start()
    {
        leftHand.SetActive(false);
        rightHand.SetActive(false);
        
        findingService =
          ServiceManager.
            Instance.GetService<IMRTK3ConfigurationFindingService>();
        
        findingService.LeftHandStatusTriggered.AddListener(t=>
          leftHand.SetActive(t));
        findingService.RightHandStatusTriggered.AddListener(t=>
          rightHand.SetActive(t));
    }
}
```

And on HoloLens it shows like this:

![](/assets/2022-06-19-MRTK2-to-MRTK3--intercepting-a-raw-air-tap-like-with-IMixedRealityPointerHandler/airtap.gif)

## Concluding words

Note: I found out `currentControllerState.selectInteractionState` is actually a property of `XRBaseController` already, so technically I don't even have to cast it to `ArticulatedHandController`. However, this exposes the full properties of the hand controller so I thought it nicer to let it stay that way, so this server - or clients using it - could easily access the hand controller without rummaging through the scene, trying to find it.

Once again - no idea if this is *the* way of doing it, like the MRTK team intended it be - but it works.

[Sample project here.](https://github.com/LocalJoost/MRKTAirTap)