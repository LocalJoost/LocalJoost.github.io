---
layout: post
title: Reading QR codes with an MRTK2 Extension Service
date: 2021-01-28T00:00:00.0000000+01:00
categories: []
tags:
- HoloLens2
- MRTK2
- Unity3d
- Windows Mixed Reality
- QRCode
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2021-01-28-Reading-QR-codes-with-an-MRTK2-Extension-Service/shellviewer.png
comment_issue_id: 371
---
QR codes are a great way to input data into Mixed Reality apps. You can simply hold them in front of the device you are wearing and the app can read the data directly from it. Much easier, faster and less error-prone than typing, even on the awesome HoloLens 2 keyboard. These codes can hold quite some data, and  HoloLens 2 comes with the native capability to read QR codes. You can actually test this by holding a random QR code before your HoloLens while in the shell (that is, not running any app)

![](/assets/2021-01-28-Reading-QR-codes-with-an-MRTK2-Extension-Service/shellviewer.png)

If the QR code contains a web link, it will even show a kind of "play" button on top of it. If you press it, it should open the web link. As you can see it not only is able to *read* the QR code, it also knows its *position in space*. This allows you to build a poor man's Vuforia - using QR codes in stead of VU markers. With the added benefit that it's a built-in capacity - and thus, free for any (including commercial) use. 

## Samples available
There is [documentation by Microsoft](https://docs.microsoft.com/en-us/windows/mixed-reality/develop/platform-capabilities-and-apis/qr-code-tracking) explaining how to set up things. It points to this [sample](https://github.com/chgatla-microsoft/QRTracking/tree/master/SampleQRCodes). It may be just me, but I found article and code a bit hard to understand. To understand what's going on, I dissected the code. And as am I called an 'architect' these days, of course I had to put it back together again in another form ;) - both to make it more understandable for myself and hopefully for you as well.

This blog will be in two parts: the first part (this one) will describe how to read QR codes and process the data. This is useful if you only want to use QR codes to easily input data in your app. The next article will concentrate on the positioning of QR codes in space in a way that makes sense (to me) and closely resembles the way Vuforia works.

## A little peek-ahead

A behaviour using the service to show the internal working of QR tracking (useful for debugging purposes, and a behaviour that actually reads and displays QR code payload - close to what you would want to do in real life - as demonstrated in this video:

<iframe width="650" height="365" src="https://www.youtube.com/embed/lrheywlIRtg" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Setting up the project
I used Unity 2019.4.17f1 and - mind you - 'Legacy XR settings'. 

![](/assets/2021-01-28-Reading-QR-codes-with-an-MRTK2-Extension-Service/legacyxrsettings.png). 

After I created the project, I changed the platform to "Universal Windows" and I proceeded with:

* importing the MRTK2 using the [Unity Package Manager procedure](https://microsoft.github.io/MixedRealityToolkit-Unity/Documentation/usingupm.html). 
* installing the [NuGet for Unity](https://github.com/GlitchEnzo/NuGetForUnity/releases)
* using NuGet for Unity to install the **Microsoft.MixedReality.QR** package. This will automatically pull in the **Microsoft.VCRTForwarders.14 package** as well
* making sure the WebCam capability is selected in player settings

![](/assets/2021-01-28-Reading-QR-codes-with-an-MRTK2-Extension-Service/nugetpackages.png)

Finally, of course, I added the MRTK2 to the scene - using the drop down menu in Unity that appeared after we imported the MRTK2. The project is now ready to be set up.

## QRCodeService - the general idea
The intention of this service is to be a simple wrapper around `QRCodeWatcher`, the class we got access to when we imported the Microsoft.MixedReality.QR package. This class has some peculiar properties. First, it uses the *tracking cameras* to read the QR code, and not the webcam - unlike Vuforia. This allows you - also unlike Vuforia - to actually stream or film the actual process of acquiring and reading without the webcam being hijacked and the stream or movie being stopped. 

## Interface and description
Since the QRCodeService is an MRTK2 service, it needs to implement a child interface of `IMixedRealityExtensionService`. Our interface is called `IQRCodeTrackingService`: 
```cs
public interface IQRCodeTrackingService : IMixedRealityExtensionService
{
    event EventHandler Initialized;
    event EventHandler<string> ProgressMessageSent;
    event EventHandler<QRInfo> QRCodeFound;
    bool InitializationFailed { get;}
    string ErrorMessage { get; }
    string ProgressMessages { get; }
    bool IsSupported { get; }
    bool IsTracking { get; }
    bool IsInitialized { get; }
}
```
* Initialized is fired when the service has been successfully initialized
* The ProgressMessageSent event sends messages that can be used to debug the internal workings of the service
* QRCodeFound is fired when a QR code is seen. This happens rather continuously, as we will see later
* InitializationFailed is fired when initialization fails and cannot be recovered 
* ErrorMessage then contains a description what went wrong
* IsSupported is set to true when tracking is supported
* IsTracking set to true when the tracker is actually tracking
* IsInitialized is set to true after successful initialization (useful for when you missed the event)

Note, also that `IMixedRealityExtensionService` descends from `IMixedRealityService`. From this interface we inherit a couple of other methods, from which we are going to implement the following:
* Initialize
* Update
* Enable
* Disable

## Declaration, members, properties and events
The top of the service looks like this:
```cs
[MixedRealityExtensionService(SupportedPlatforms.WindowsUniversal)]
public class QRCodeTrackingService : BaseExtensionService, IQRCodeTrackingService
{
    private QRCodeTrackingServiceProfile profile;
    public QRCodeTrackingService(string name, uint priority, 
       BaseMixedRealityProfile profile) : base(name, priority, profile)
    {
        this.profile = (QRCodeTrackingServiceProfile) profile;
    }

    public event EventHandler Initialized;
    public event EventHandler<QRInfo> QRCodeFound;
    public event EventHandler<string> ProgressMessageSent;

    public bool InitializationFailed { get; private set; }
    public string ErrorMessage { get; private set; }
    public bool IsSupported { get; private set; }
    public bool IsTracking { get; private set; }
    public bool IsInitialized { get; private set; }
    public string ProgressMessages { get; private set; }

    private QRCodeWatcher qrTracker;
    private QRCodeWatcherAccessStatus accessStatus;

    private int initializationAttempt = 0;

    private readonly List<string> messageList = new List<string>();
}
```
After the public events and properties, which I already described, you see a couple private fields.
* qrTracker is the actual internal class from the `Microsoft.MixedReality.QR` package
* accessStatus holds the last access status (that is, if the user gave consent to use the camera)
* initializationAttempt is for debugging/logging purposes only
* messageList holds the debug/progress messages issued by this class

## Initialization
This is decidedly a weird piece of code, which may come from the fact that I try to run it as a service. 

```cs
public override void Initialize()
{
    base.Initialize();
    _ = InitializeTracker();
}

private async Task InitializeTracker()
{
    try
    {
        IsSupported = QRCodeWatcher.IsSupported();
        if (IsSupported)
        {
            SendProgressMessage(
              $"Initializing QR tracker attempt {++initializationAttempt}");

            var capabilityTask = QRCodeWatcher.RequestAccessAsync();
            await capabilityTask.AwaitWithTimeout(profile.AccessRetryTime, 
                ProcessTrackerCapabilityReturned,
             () => _ = InitializeTracker());
        }
        else
        {
            InitializationFail("QR tracking not supported");
        }
    }
    catch (Exception ex)
    {
        InitializationFail($"QRCodeTrackingService initialization failed: {ex}");
    }
}
```
What happens here is:
* The code creates a `Task` asking for access
* It launches the task *together* with a timeout (default 5 seconds)
* If 5 seconds pass, it tries again
* If access is granted, `ProcessTrackerCapabilityReturned` is called

I have not been able to find out why this is necessary in this context, but it solves a problem at very first app launch. If you put `QRCodeWatcher.RequestAccessAsync()` in `MonoBehavior` (like in the sample), the camera consent popup appears and the task is blocked until consent is either granted or denied. In a *service*, the task stays blocked no matter what, *even if you give consent* - the code will never get access. You will then only get access when you stop the app and start it again after granting access at the first run. So I made this little weird thing that restarts the asking for code in the app itself..

`AwaitWithTimeout`, is a little extension method to `Task`, that I [nicked from Stack Overflow](https://stackoverflow.com/questions/35645899/awaiting-task-with-timeout) and adapted it to make it able to pass a parameter. It looks this:
```cs
public static class TaskExtensions
{
    public static async Task AwaitWithTimeout<T>(this Task<T> task, 
           int timeout, Action<T> success, Action error)
    {
        if (await Task.WhenAny(task, Task.Delay(timeout)) == task)
        {
            success?.Invoke(task.Result);
        }
        else
        {
            error?.Invoke();
        }
    }
}
```
Mind you, this whole timeout-and-try-again thing is *only* used the *first time you run the app* - when it needs to get consent. Once it has it, is sails right trough it the next time.

## Setting up tracking
The next three methods actually set up and enable tracking:
```cs
private void ProcessTrackerCapabilityReturned(QRCodeWatcherAccessStatus ast)
{
    if (ast != QRCodeWatcherAccessStatus.Allowed)
    {
        InitializationFail($"QR tracker could not be initialized: {ast}");
    }
    accessStatus = ast;
}

public override void Update()
{
    if (qrTracker == null && accessStatus == QRCodeWatcherAccessStatus.Allowed)
    {
        SetupTracking();
    }
}

private void SetupTracking()
{
    qrTracker = new QRCodeWatcher();
    qrTracker.Updated += QRCodeWatcher_Updated;
    IsInitialized = true;
    Initialized?.Invoke(this, new EventArgs());
    SendProgressMessage("QR tracker initialized");
}
```
`ProcessTrackerCapabilityReturned` is called when access is successfully granted or denied, but at least is no longer blocked. It only sets the field `accessStatus` to the parameter value it got supplied. The `Update` method then, that is called every frame just like in a `MonoBehaviour`, checks `accessStatus` and if the `qrTracker` is not yet initialized. In that case it, creates the tracker, connects to its `Updated` event and does some housekeeping to notify the outside world of its status. 

Note, I only observe the `Updated` event. There are also  `Added` and `Removed` events in the `QRCodeWatcher` but I have not been able to reliably make use of those. Basically `Updated` is (continuously) being fired as soon as the QR code comes into view.

## Notifying the outside world
Little code, but still something more to tell about it:
```cs
private void QRCodeWatcher_Updated(object sender, QRCodeUpdatedEventArgs e)
{
    SendProgressMessage($"Found QR code {e.Code.Data}");
    QRCodeFound?.Invoke(this, new QRInfo(e.Code));
}
```
Seems like it's only logging the payload and then passing it on. Which is more or less the case, but for one detail. We don't send a `Microsoft.MixedReality.QR.QRCode` to the outside world, but our own `QRInfo`. This has exactly the same properties as QRCode and the constructor simply copies the properties over one by one. 

The reason for this is that after long debugging, I have ascertained that you apparently don't get a *new* `QRCode` instance when a QR code moves (or your head moves), but a *reference to the same one over and over again*. I presume for the sake of efficiency. This is all very nice, but if you want to compare a location or the `LastDetectedTime` with a previous occurrence, you will find the properties are always the same. This is, of course because the new occurrence *references the same object* as the previous one. In my code, I need to actually be able to compare the previous occurrence with the current one. So, we create our own object, wrapping the data of the internal event object and bypassing the issue of the re-used reference

## Enabling and disabling
Overriding the standard Enable and Disable methods of the service's base class,
we simply start and stop the tracker and do some housekeeping
```cs
public override void Enable()
{
    base.Enable();
    if (!IsInitialized)
    {
        return;
    }

    try
    {
        qrTracker.Start();
        IsTracking = true;
        SendProgressMessage("Enabled tracking");
    }
    catch (Exception ex)
    {
        InitializationFail(
        $"QRCodeTrackingService starting QRCodeWatcher Exception: {ex}");
    }
}

public override void Disable()
{
    base.Disable();
    if (IsTracking)
    {
        IsTracking = false;
        qrTracker?.Stop();
        SendProgressMessage("Disabled tracking");
    }
}
```
**Notice** the service will be active but not actually *doing* anything before its `Enable` method is called! 

What's left now are some logging methods you can [view in the demo project](https://github.com/LocalJoost/QRCodeService/tree/blog1).

## Profile
So that's the service. It should be configured by a `QRCodeTrackingServiceProfile` that has three properties:
* `AccessRetryTime` - number of milliseconds to retry acquiring camera access in `InitializeTracker`
* `ExposedProgressMessages` - true shows and logs progress messages in 
* `DebugMessages` - the number of progress messages to keep before removing the oldest one

![](/assets/2021-01-28-Reading-QR-codes-with-an-MRTK2-Extension-Service/serviceprofile.png)

-------
# Using and enabling the service from a behaviour
Typically, you use something like this pattern:
```cs

private IQRCodeTrackingService qrCodeTrackingService;

private IQRCodeTrackingService QRCodeTrackingService
{
    get
    {
        while (!MixedRealityToolkit.IsInitialized && Time.time < 5);
        return qrCodeTrackingService ??
               (qrCodeTrackingService =
                MixedRealityToolkit.Instance.
                  GetService<IQRCodeTrackingService>());
    }
}

private void Start()
{

    if (!QRCodeTrackingService.IsSupported)
    {
        return;
    }

    if (QRCodeTrackingService.IsInitialized)
    {
        StartTracking();
    }
    else
    {
        QRCodeTrackingService.Initialized += QRCodeTrackingService_Initialized;
    }
}

private void QRCodeTrackingService_Initialized(object sender, System.EventArgs e)
{
    StartTracking();
}

private void StartTracking()
{
    QRCodeTrackingService.QRCodeFound += QRCodeTrackingService_QRCodeFound;
    QRCodeTrackingService.Enable();
}

private void QRCodeTrackingService_QRCodeFound(object sender, QRInfo codeReceived)
{
    // Do something with code
}
```
Here you see the typical pattern from top to bottom:  
* Get the service reference
* Start: 
    * Check for support
    * If already initialized, call `StartTracking` directly
    * If not initialized, wait for the Initialized event that calls `StartTracking`
* In StartTracking, start observing event `QRCodeFound` and enable the service.


## Showing off 1 - QRDebugDisplayer  
In the [demo project](https://github.com/LocalJoost/QRCodeService/tree/blog1), in the HologramCollection there are two game object that can show off the service in action. 

![](/assets/2021-01-28-Reading-QR-codes-with-an-MRTK2-Extension-Service/hierarchy.png)

Preferably you don't activate them during the same run or they will show on top of each other, and probably make you cross-eyed from looking from one to the other. The first one is `QRDebugDisplayer`, which is controlled by `QRTrackingServiceDebugController`. This basically only dumps the progress messages on a floating screen. It does not even listen to the `QRCodeFound` event. This is the behaviour shown off in the first part of the video above

## Showing off 2 - QRCodeDisplayer
The second game object, QRCodeDisplayer, is controlled by `QRCodeDisplayController` and is a bit more interesting - for instance, it only shows its UI after the tracker is successfully initialized.

```cs

private void StartTracking()
{
    menu.SetActive(true);
    QRCodeTrackingService.QRCodeFound += QRCodeTrackingService_QRCodeFound;
    QRCodeTrackingService.Enable();
}

```
Also, it displays the QR code payload and plays a sound only if there's a code detected different from the last one.

```cs
private QRInfo lastSeenCode;

private void QRCodeTrackingService_QRCodeFound(object sender, QRInfo codeReceived)
{
    if (lastSeenCode?.Data != codeReceived.Data)
    {
        displayText.text = $"code observed: {codeReceived.Data}";
        if (confirmSound.clip != null)
        {
            confirmSound.Play();
        }
    }
    lastSeenCode = codeReceived;
}
``` 
And even more so in the update method. If the last seen code has not been seen for `qrObservationTimeOut` milliseconds (in the [demo project](https://github.com/LocalJoost/QRCodeService/tree/blog1) 3500 ms) the app will clear the text from the screen and assume it to be a new code again (and thus play the sound again). This shows the use of using `QRInfo` in stead of directly referencing `QRCode`, so we can actually *perform* this timeout check.

```cs
private void Update()
{
    if (lastSeenCode == null)
    {
        return;
    }
    if (Math.Abs(
        (lastSeenCode.LastDetectedTime.UtcDateTime -
          DateTimeOffset.UtcNow).TotalMilliseconds) >
            qrObservationTimeOut)
    {
        lastSeenCode = null;
        displayText.text = string.Empty;
    }
}
``` 
## Conclusion

QR code reading has it weirdness-es, but I hope I brought a bit more clarity into this. I also hope to have provided an easy to deploy, configure, debug and deploy object. Those who want to read my blog using their HoloLens 2 can use this QR code ;) '

![](/assets/2021-01-28-Reading-QR-codes-with-an-MRTK2-Extension-Service/blogqrcode.png)

But of course you can also just navigate to [this link](https://github.com/LocalJoost/QRCodeService/tree/blog1) for the demo project