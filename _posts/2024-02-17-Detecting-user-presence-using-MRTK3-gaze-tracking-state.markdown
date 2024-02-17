---
layout: post
title: Detecting user presence using MRTK3 gaze tracking state
date: 2024-02-17T00:00:00.0000000+01:00
categories: []
tags:
- MRTK3
- HoloLens 2
- Unity
- Mixed Reality
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2024-02-17-Detecting-user-presence-using-MRTK3-gaze-tracking-state/Presence.gif
comment_issue_id: 468
---
Sometimes it's necessary to know whether or not the user is actually wearing the headset while running your app. For instance, this might be because the user is doing an important task that may not be stopped until completed, or because you want to pause critical, performance-heavy, or battery-draining processes on the device when the user takes it off for a few minutes. I have found MRTK3 gaze tracking state to be a reliable way to detect user presence, and wrote a little [ServiceFramework](https://service-framework.realitycollective.io/) Service to utilize that.

![](/assets/2024-02-17-Detecting-user-presence-using-MRTK3-gaze-tracking-state/Presence.gif)

## Profile

(Almost) everything in the ServiceFramework starts with a profile, and so does this service. 

```csharp
public class UserPresenceServiceProfile : BaseServiceProfile<IServiceModule>
{
    [SerializeField]
    private InputActionReference gazeTrackingState;
    
    [SerializeField]
    private float userAwayWaitTime = 3.0f;

    [SerializeField]
    private float userPresentWaitTime = 0.5f;
    
    public InputActionReference GazeTrackingState => gazeTrackingState;
    public float UserAwayWaitTime => userAwayWaitTime;
    public float UserPresentWaitTime => userPresentWaitTime;
}
```
`userAwayWaitTime` is the time (in seconds) the service needs to detect the continued user absence before it signals the outside world. This is to prevent events being triggered when users so much as blink, or the eye tracking loses track for a few moments.
`userPresentWaitTime` is the time (also in seconds) the service needs to detect the continued *return* of the user after absence. 
`gazeTrackingState` is a reference to the Tracking State input action of the MRTK Default Input Actions. This is used to get the actual gaze state from the Interaction Manager in Unity's XR Interaction Toolkit. 

![](/assets/2024-02-17-Detecting-user-presence-using-MRTK3-gaze-tracking-state/gazestateref.png)

## Public interface

The service exposes only two items: the current user presence, and an event to tell the outside world the presence has changed. 

```csharp
public interface IUserPresenceService : IService
{
    public bool IsUserPresent { get; }
    public UnityEvent<bool> UserPresenceChanged { get; } 
}
```

## The service itself

I will omit declarations and most of the constructor, as most of it is fairly standard for a Service Framework Service. The only thing worth noting is this line, where we pick up the reference to the gaze tracking state from the profile:

```csharp
gazeTrackingState = profile.GazeTrackingState;
```

## Enabling event reading

When the service is actually enabled, it only sets up a listener to the gaze tracking state's `action.performed` event. Note, this is actually not MRTK specific anymore - `gazeTrackingState` is an `InputActionReference` and that's part of Unity's Input System.

```csharp
public override void Enable()
{
    if (isInitialized)
    {
        return;
    }

    isInitialized = true;
    gazeTrackingState.action.performed += GazeTrackingStateChanged;
}

private void GazeTrackingStateChanged(InputAction.CallbackContext ctx)
{
    gazeStateResult = ctx.ReadValue<int>();
}
```

Now the thing to keep in mind is - if you run this in the editor, `GazeTrackingStateChanged` gets called a few times and that's it. When you run this on HoloLens 2, `GazeTrackingStateChanged` gets really *hammered* with events. It seems like the eye tracker is firing this event all the time. That's why the only thing we do is put the value in a field, and let the service's Update loop handle the logic.

## Interpreting values and handling timeouts

In the Update loop, we first check the state value. I have seen that "3" means "eye tracking detected". I also have seen value "0" when I took off the device, so I have chosen to interpret "3" as "user present" and anything else as "not present"

```csharp
public override void Update()
{
    var newState = gazeStateResult == 3;
```

First step: if the detected new state is the same as the current state, remember that last requested state, and exit the method
 
```csharp
    if (newState == IsUserPresent)
    {
        lastRequestedState = newState;
        return;
    }
```

Second step: if there is, however, a new state, the method runs to the second if. If the `lastRequestedState` does not match the `newState` yet, that means a state change has taken place. This is registering noting the time the state change has taken place.

```csharp
    if( newState != lastRequestedState)
    {
        lastRequestedState = newState;
        lastStateChangeTime = Time.time;
    }
```

So. The first if doesn't do anything anymore, as `newState` and `IsUserPresent` are not equal. But `newState` and `lastRequestedState` *are* equal, so `lastStateChangeTime` stays fixed. Now the clock starts ticking in the last part start:

```csharp
    if( Time.time - lastStateChangeTime > (lastRequestedState ? 
         profile.UserPresentWaitTime : profile.UserAwayWaitTime))
    {
        IsUserPresent = lastRequestedState;
        UserPresenceChanged.Invoke(IsUserPresent);
    }
}
```
If the user does not do anything that makes the state flip again (at which point the first if kicks in again and 'stops the clock'), `IsUserPresent` is set and and the event is called. The time to wait before the event indicating state change is determined by whether the presence changes from true to false, or the other way around. It's not really rocket science. 

## Some demo code to go with it

I have added a demo scene with a simple behaviour `UserPresenceDisplayer` that shows how you might use this. The interesting parts are this:

```csharp
public class UserPresenceDisplayer : MonoBehaviour
{
    private async Task Start()
    {
        audioSource = GetComponent<AudioSource>();
        await ServiceManager.WaitUntilInitializedAsync();
        userPresenceService = 
           ServiceManager.Instance.GetService<IUserPresenceService>();
        userPresenceService.UserPresenceChanged.AddListener(OnUserPresenceChanged);
    }

    private void OnUserPresenceChanged(bool currentPresence)
    {
        displayText.text = $"User is {(currentPresence ? "present" : "away")}";
        audioSource.PlayOneShot(currentPresence ? userPresentClip : userAwayClip);
    }
}
```
It waits for the Service Manager to be ready, then gets a reference to the service. When events are received, it shows an appropriate message on a floating text, plays a high note when the user presence changes from away to present, and a low note when it changes from present to away.

## Concluding words

I have found it is best to set a slightly longer wait time (3-5 seconds) before firing the "user away" event before pausing whatever you want to pause, as false positives can be really annoying to the user. However, if the user *returns*, you want your app's functionality back up to speed ASAP, so that is usually a shorter time. However, I can also imagine scenarios where a quick "user away" event is necessary, for instance when an app is used to monitor an exam or something. You can simply make multiple profiles for that without needing to change any code. That's the beauty of the Service Framework.

Note: so far, this has been tested on HoloLens 2 only. For that, it will require the Eye Gaze Interaction profile being set in the OpenXR Eye Tracking interaction profiles.  

![](/assets/2024-02-17-Detecting-user-presence-using-MRTK3-gaze-tracking-state/eyegazeprofile.png)

And it will, obviously, only work with devices that actually *support* eye tracking.  I will conduct experiments with Magic Leap 2 soon.

[Fully working demo, as always, on GitHub](https://github.com/LocalJoost/MRTK3UserPresence.git
).
