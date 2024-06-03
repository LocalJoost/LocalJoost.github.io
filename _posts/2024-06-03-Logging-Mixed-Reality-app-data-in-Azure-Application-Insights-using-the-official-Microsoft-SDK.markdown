---
layout: post
title: Logging Mixed Reality app data in Azure Application Insights using the official Microsoft SDK
date: 2024-06-03T00:00:00.0000000+02:00
categories: []
tags:
- Unity
- HoloLens 2
- Mixed Reality
- Service Framework
- telemetery
- Application Insights
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2024-06-02-Logging-Mixed-Reality-app-data-in-Azure-Application-Insights-using-the-official-Microsoft-SDK/banner.png
comment_issue_id: 470
---
If you are supporting any kind of professional business app, you don't want to rely on users reporting issues. They typically concentrate on doing their job, which in their opinion usually does *not* include detailed descriptions, reproduction paths, or even just the error message itself (especially if it's longer than five words). So you want the app to 'phone home' itself. Microsoft have been doing that for years, if not decades, and so have I since I started putting Mixed Reality apps out into the wild. To this end, I have been successfully using the [UnityApplicationInsights](https://github.com/Unity3dAzure/UnityApplicationInsights) project from GitHub to log telemetry in [Azure Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview). It simply uses the instrumentation key, you plonk a behaviour in your scene, and you are good to go. However, this code has not seen an update in 6 years, and recently when I looked at some [Application Insights documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/app/asp-net-core?tabs=netcorenew) I noticed this banner:

![](/assets/2024-06-03-Logging-Mixed-Reality-app-data-in-Azure-Application-Insights-using-the-official-Microsoft-SDK/banner.png)

So I tried to get the official SDK to work with Unity on a HoloLens 2, and it works. There is some setup required though, and I have made it into a simple reusable [RealityCollective Service](https://service-framework.realitycollective.io/)

## TelemetryService functions

The main functions of the service are:
* Log an event
* Log an exception
* Automatically log an event when a keyword appears in the Unity log. How this works will be explained later

## Using the TelemetryService

In [the demo project](https://github.com/localJoost/TelemetryService), there is a simple menu with four buttons. It's driven by the class `DemoMenuHandler` that initializes the service, and various other buttons that start code that almost physically hurt to write, so bad is it, but it is intended to actually generate errors.

![](/assets/2024-06-03-Logging-Mixed-Reality-app-data-in-Azure-Application-Insights-using-the-official-Microsoft-SDK/menu.png)

### Initializing the service

On top of the `DemoMenuHandler`, you see what begins like a pretty standard Service Framework initialization:

```csharp
private async void Start()
{
    await ServiceManager.WaitUntilInitializedAsync();
    telemetryService = ServiceManager.Instance.GetService<ITelemetryService>();
    telemetryService.Initialize("your_connection_string_here");
}
```

Assuming you have already set up an Application Insights resource in Azure, you will need to change "your_connection_string_here" to the thing that appears in "CONNECTION STRING." Typically it looks like this: "InstrumentationKey=_*some_guid_here*;IngestionEndpoint=https://*something*.applicationinsights.azure.com/;
LiveEndpoint=https://*something*.livediagnostics.monitor.azure.com/;ApplicationId=*some_other_guid*"

So why didn't I put this into the service profile? Well, *because you don't bake secrets into an app*. Make sure you download this from a secure location where you can change it easily, preferably after the user has logged in, from an authenticated API. Although having those keys doesn't give bad actors access to your app, they might use it just to spam your application insights with nonsense just to bug you. It pays to be paranoid these days.

![](/assets/2024-06-03-Logging-Mixed-Reality-app-data-in-Azure-Application-Insights-using-the-official-Microsoft-SDK/portal.png)

### Using the service from code

The TelemetryService has a surprisingly simple public API, as defined by ITelemetryService:

```csharp
public interface ITelemetryService : IService
{
    void Initialize(string connectionString, string userId = null);
    public void TrackEvent(string eventName, Dictionary<string, string> properties = null);
    public void TrackEvent(string eventName, 
       params string[] properties);
    public void TrackException(Exception exception);
}
```

* We already have seen `Initialize` in action: this sets the connection string, and an optional user id to appear in every call.
* The first `TrackEvent` method allows you to send a simple custom event to the ApplicationInsights **customEvents** collection. Anything you put into the optional `Dictionary` will be added as *custom properties* (I will show that later).
* The second `TrackEvent` does the same, but allows the developer who is too lazy to define a `Dictionary` to supply the properties and values as string parameters (see sample code).
* `TrackException` sends a detailed exception log to the ApplicationInsights **exceptions** collection

Initialization sample:

```csharp
await ServiceManager.WaitUntilInitializedAsync();
telemetryService = ServiceManager.Instance.GetService<ITelemetryService>();
telemetryService.Initialize("your_connection_string_here");
```

Tracking an event with some properties:

```csharp
telemetryService.TrackEvent("Hello world button clicked", 
    "prop1", "value1", "prop2", "value2");
```

  
Doing the same, but using the Dictionary method:

```csharp
telemetryService.TrackEvent("Hello world button clicked", 
    new Dictionary<string, string>
    {
        {"prop1", "value1"},
        {"prop2", "value2"}
    });
```

Tracking an exception:

```csharp
try
{
   // Do someting
}
catch (Exception e)
{
    telemetryService.TrackException(e);
    throw;
}
```

## Automatically trapping application events

Unity is a pretty chatty platform that dumps all kinds of information in its application log. Those who have read my [blog about the FileLoggerService](https://localjoost.github.io/A-cross-platform-Service-Framework-service-to-write-Unity-log-messages-into-files-on-device/) have seen this before. You can tap into this log by subscribing to the `Application.logMessageReceivedThreaded` event. The TelemetryService can do just that for you, check for whatever keyword or phrase you want, and log this to Application Insights as a custom event. The default DefaultTelemetryServiceProfile looks like this:

![](/assets/2024-06-03-Logging-Mixed-Reality-app-data-in-Azure-Application-Insights-using-the-official-Microsoft-SDK/config.png)

So whenever, wherever in the app something like this happens - you will know. This is a very good way to surface hidden errors that may not have immediate results like the app misbehaving or even crashing, but are a good indication of where some of the 'Heisenbugs' originate that typically plague any piece of software - yes, yours too. Admit it.

As you might have noticed, the configuration also offers a checkbox that shows the events and exceptions it sends in the debug log.

## Setting up the project from scratch

Assuming you have set up a proper MRTK 3.0 project, the way to go is this:
* Install the [Reality Collective Service Framework](https://openupm.com/packages/com.realitycollective.service-framework/) from OpenUPM. For the moment, I would suggest version 1.0.7, the latest build that is not in preview
* Install [NuGet for Unity](https://github.com/GlitchEnzo/NuGetForUnity?tab=readme-ov-file#how-do-i-install-nugetforunity)
* Using NuGet for Unity, install package **Microsoft.ApplicationInsights**

![](/assets/2024-06-03-Logging-Mixed-Reality-app-data-in-Azure-Application-Insights-using-the-official-Microsoft-SDK/nuget.png)

* As of today, this will create three folders in your Assets/Packages folder 
    * Microsoft.ApplicationInsights.2.22.0
    * System.Diagnostics.DiagnosticSource.5.0.0 
    * System.Runtime.CompilerServices.Unsafe.5.0.0 
    
    As ApplicationInsights apparently has some fancypants reflection-based initialization code, you will need to place a link.xml in the Packages folder to prevent the compiler stripping away an 'unused' constructor that is used anyway. The link.xml will need to contain this code:
    
```xml
<linker>
  <assembly fullname="Microsoft.ApplicationInsights" preserve="all"/>
</linker>
```
* Copy the four files in Assets/LocalJoost/Services/Telemetry into your project
* Add a Service Manager Instance to your scene (Tools/Service Framework/Add to Scene)
* Configure the TelemetryService to run on the platform(s) you want, you can use the DefaultTelemetryServiceProfile as configuration if you want - or create your own, or use none at all
* On a HoloLens project, don't forget to assign the "Internetclient" capability

And you are good to go.

## How it looks like in Azure

If you go to your Azure portal, select the Application Insights resource, then the "Logs" option, you can write queries on the Application Insights customEvents collection. It looks like this:

![](/assets/2024-06-03-Logging-Mixed-Reality-app-data-in-Azure-Application-Insights-using-the-official-Microsoft-SDK/AI1.png)

You can also see where the app has been used.

If you expand the "customEvent" property, you can, for instance, see the detailed issue of a tracked NullReferenceException text appearing in the log

![](/assets/2024-06-03-Logging-Mixed-Reality-app-data-in-Azure-Application-Insights-using-the-official-Microsoft-SDK/customEvent.png)

In fact, you can see quite some detailed info about the device and the location that Application Insights apparently collects automatically.

![](/assets/2024-06-03-Logging-Mixed-Reality-app-data-in-Azure-Application-Insights-using-the-official-Microsoft-SDK/customEvent2.png)

You can also, as I said before, query the "exceptions" collection that is actually more suited for displaying exception information

![](/assets/2024-06-03-Logging-Mixed-Reality-app-data-in-Azure-Application-Insights-using-the-official-Microsoft-SDK/exception.png)

It shows very detailed information about the exception:

![](/assets/2024-06-03-Logging-Mixed-Reality-app-data-in-Azure-Application-Insights-using-the-official-Microsoft-SDK/exception2.png)

So although the keyword tracking option is a nice fallback, actually trapping the exception yourself and sending info is much more helpful. But then again, you first need to be aware things go wrong.

## So how does this all work?

### Initialization

The service creation is pretty standard, we stick the profile in a private field

```csharp
public TelemetryService(string name, uint priority, TelemetryServiceProfile profile)
    : base(name, priority)
{
    serviceProfile = profile;
}
```

The Initialize method can only be called once. It configures the `TelemetryClient` from the Microsoft SDK, adds data to the telemetry context (see below), then adds the keywords it should track from the log (even further below)

```csharp
public void Initialize(string connectionString, string userId = null)
{
    if(telemetryClient != null)
    {
        return;
    }
    user = userId;
    var config = new TelemetryConfiguration();
    config.ConnectionString = connectionString;
    telemetryClient = new TelemetryClient(config);
    AddTelemetryContext(telemetryClient.Context);
    TrackUnityLogKeywords();
}
```

### Adding context

Context are the standard (that is, not custom) properties of telemetry. You can either add those to an event or to the client itself. If you use the latter (like me) the client will automatically add those property values to every event. This is what happens at the bottom of the service:

```csharp
private void AddTelemetryContext(TelemetryContext context)
{
    context.Component.Version = GetAppVersion();
    context.Device.Id = SystemInfo.deviceUniqueIdentifier;
    context.Device.OperatingSystem = SystemInfo.operatingSystem;
    context.Device.Type = SystemInfo.deviceModel;
    if (user != null)
    {
        context.User.Id = user;
    }
}

private string GetAppVersion()
{
#if WINDOWS_UWP
    var version = Windows.ApplicationModel.Package.Current.Id.Version;
    return $"{version.Major}.{version.Minor}.{version.Build}.{version.Revision}";
#else
    return Application.version;
#endif        
}
```

Note that the application version for HoloLens apps is retrieved differently than when using other platforms. This is because the 'other platforms' are usually Android-based, which Unity generates in one go, while HoloLens uses the two-stage process that first generates a C++ UWP project - and the version is ultimately decided by what's in that project's manifest, which may be (and usually is) different from what is defined by Unity's project settings.

### Tracking keywords

The setup is pretty easy: it copies all the keywords to track into a list, removing duplicates. If there are any keywords to track, the service attaches to the `Application.logMessageReceivedThreaded` event.

```csharp
private void TrackUnityLogKeywords()
{
    if (serviceProfile == null)
    {
        return;
    }
    if (serviceProfile.LogKeywordsToTrack.Any())
    {
        Application.logMessageReceivedThreaded += OnLogMessageReceived;
    }

    foreach (var keyword in serviceProfile.LogKeywordsToTrack)
    {
        if (!logKeywordsToTrack.Contains(keyword))
        {
            logKeywordsToTrack.Add(keyword);
        }
    }
}
```

The code that listens to the event is actually pretty simple:

```csharp
private void OnLogMessageReceived(string condition, string stacktrace, LogType type)
{
    var keywordFound = logKeywordsToTrack.FirstOrDefault(condition.Contains);
    if (keywordFound != null)
    {
        TrackEvent($"Keyword tracked: {keywordFound}", 
            "message", condition, "stacktrace", stacktrace);
    }
}
```

### Tracking an event in code

The `TrackEvent` simply passes the event and its properties to Application Insights, and optionally logs it to the debug log. Note it refuses to do that for keyword tracked events: otherwise, we would get a kind of endless feedback log loop.

```csharp
public void TrackEvent(string eventName, 
                       Dictionary<string, string> properties = null)
{
    if( telemetryClient == null )
    {
        return;
    }
    telemetryClient.TrackEvent(eventName, properties);
    telemetryClient.Flush();
    if(serviceProfile != null && serviceProfile.LogToConsole && !eventName.Contains("Keyword tracked"))
    {
        var propertiesString = properties != null
            ? string.Join(", ", 
            properties.Select(p => $"{p.Key}: {p.Value}"))
            : string.Empty;
        Debug.Log($"Telemetry event: {eventName} {propertiesString}");
    }
}
```
The other `TrackEvent` method and the `TrackException` method I leave as an exercise for the reader.

## Concluding words

Application Insights is an awesome and simple-to-use telemetry solution. However, as you have seen, it logs quite some detailed information. You might want to think if you actually want and need all that information, considering GDPR and such. You should think about what you need to log and when, consider a retention strategy, decide when you delete stuff and what. Note, however, it does show a bogus IP address, and the location presented by XR devices like HoloLens is where my internet provider hooks up to the internet - so it does show Amsterdam instead of my actual location in Amersfoort.

If your app is authenticated, you can even provide a user id at `Initialize` which can help you track down issues by specific user, and link entries to user error reports. That's all fine, but never provide the direct user name or some id that is relatable to an actual person. Instead, use some token or id that can only be related to data inside your backend - like an encrypted Entra id.

[A demo project, as always, can be found on GitHub.](https://github.com/localJoost/TelemetryService)
