---
layout: post
title: A cross-platform Service Framework service to write Unity log messages into files on-device
date: 2023-10-24T00:00:00.0000000+02:00
categories: []
tags:
- Service FrameWork
- HoloLens 2
- Magic Leap
- Quest
- Unity
published: true
permalink: 
featuredImageUrl: http://localjoost.github.io/assets/2023-10-24-A-cross-platform-Service-Framework-service-to-write-Unity-log-messages-into-files-on-device/portal.png
comment_issue_id: 461
---
HoloLens  applications conveniently dump all the Unity log messages in a text file - including all run time errors. This file can be very useful for tracking down weird occasional errors that only happen at run time. Getting to the file from HoloLens is a bit cumbersome, you have to go via the device portal and you really have to know where to look, it is always called UnityPlayer.log and at least you can *get* to it.

![](/assets/2023-10-24-A-cross-platform-Service-Framework-service-to-write-Unity-log-messages-into-files-on-device/portal.png)

When I started developing for Quest and Magic Leap 2 as well, I noticed no such file was available. When I was experimenting with getting Lumi - [Augmedit](http://www.augmedit.com/)'s flagship product for brain surgery - to run on the Magic Leap 2, there were some things that only occasionally went wrong, and only at run time. How helpful it would be if that app also would dump its log messages in a file, so I could check after the fact. Indeed. And so I wrote a little ServiceFramework service that does exactly that.

## Profile

The service sports a profile that allows you to set a few properties:
* App file prefix
* Filter phrases - log messages that contain these will not be written
* Log file types to log - Unity provides 5 types of log messages
* How many days log files will be retained - default is 4 days
* Whether or not the service will auto start logging

This all can be conveniently set via the inspector.

## Public interface

The service sports only two methods, that you don't actually even need to use - provided you set the AutoStart profile property to true, but hey, you can do it all yourself if you want

```csharp
public interface IFileLoggerService : IService
{
    public void StartLogging();
    public void StopLogging();
}
```

## The service itself

It's a bog standard ServiceFramework Service. Messages are dumped in a queue whenever they arrive, and written in the log file one by one by the `Update` method, to prevent overloading the system with write actions - and keeping the messages in order.

```csharp
public class FileLoggerService : BaseServiceWithConstructor, IFileLoggerService
{
    private readonly string nl = Environment.NewLine;
    private readonly FileLoggerServiceProfile serviceProfile;
    private StreamWriter currentLogFile = null;
    private readonly Queue<string> logMessages = new();

    public FileLoggerService(string name, uint priority, 
      FileLoggerServiceProfile profile)
        : base(name, priority)
    {
        serviceProfile = profile;
    }
}
```

`nl` is just shorthand for `Environment.NewLine` which we will see back later. Next up is the basic startup/shutdown logic that really doesn't show much surprises:

```csharp
public override void Start()
{
    if (serviceProfile.AutoStart)
    {
        StartLogging();
    }
}

public override void Destroy()
{
    StopLogging();
    base.Destroy();
}

public void StartLogging()
{
    if (currentLogFile == null)
    {
        Application.logMessageReceivedThreaded += LogListener;
    }
}

public void StopLogging()
{
    Application.logMessageReceivedThreaded -= LogListener;
    if (currentLogFile != null)
    {
        currentLogFile.Close();
        currentLogFile = null;
    }
}
```

It also shows the crux of the whole story: the event `Application.logMessageReceivedThreaded` is used to intercept everything Unity wants to log - this is a rather standard trick - and pass it to the `LogListener` method. This method formats any log message that is not filtered either by phrase or type and dumps the result in the `logMessages` queue...

```csharp
private void LogListener(string logString, string stacktrace, LogType lType)
{
    if ( ShouldLog(logString, lType))
    {
        logMessages.Enqueue(GetLogString(logString, stacktrace, lType));
    }
}

private string GetLogString(string message, string stacktrace, LogType lType)
{
    var timeStamp = DateTimeOffset.UtcNow.ToString("yyyy-MM-dd HH:mm:ss.fff");
    var trace = !string.IsNullOrEmpty(stacktrace) ? 
        $"StackTrace: {stacktrace}{nl}" : string.Empty;
    return
        $"Time: {timeStamp}{nl}Log: {lType}{nl}Msg: {message}{nl}{trace}====={nl}";
}
```


... and like I wrote before, those log messages are written in the log file by the `Update` method, that is called once every frame by the Service Framework.

```csharp
public override void Update()
{
    base.Update();
    if (logMessages.Any())
    {
        LogFile.WriteLine(logMessages.Dequeue());
    }
}
```

The `LogFile` property creates an auto flushing `StreamWriter` or returns the existing one for append:

```csharp
private StreamWriter LogFile
{
    get
    {
        if (currentLogFile == null)
        {
            var logFilePath = Path.Combine(Application.persistentDataPath,
                $"{serviceProfile.LogFilePrefix}{DateTimeOffset.UtcNow:yyyyMMddHHmmss}.log");
            currentLogFile = new StreamWriter(
                new FileStream(logFilePath, FileMode.Append, FileAccess.Write));
            currentLogFile.AutoFlush = true;
        }

        return currentLogFile;
    }
}
```

Cleaning up the old log files, as we saw being called from the `Initialize` method, is simply done thus:

```csharp
private void DeleteOldLogs()
{
    foreach (var file in GetOldLogFilesToDelete())
    {
        try
        {
            File.Delete(file);
        }
        catch (Exception e)
        {
            Debug.LogError($"Log {file} could not be deleted: {e}");
        }
    }
}

private IEnumerable<string> GetOldLogFilesToDelete()
{
    return Directory.GetFiles(Application.persistentDataPath,
        $"{serviceProfile.LogFilePrefix}*.log").Where(file =>
        Math.Abs((DateTimeOffset.Now - File.GetCreationTime(file)).Days) >
        serviceProfile.RetainDays);
}
```

All files older than `RetainDays` are deleted. Or at least, we try to - and if that fails, it will also be logged in the *current* log file. How meta ;) 

The only thing left then is the filtering, in which we see some simple keyword filtering and some very funky bit shifting stuff.

```csharp
private bool ShouldLog(string logString, LogType lType)
{
    var logTypeFlagInt = 1 << (int)lType;
    return ((int)serviceProfile.LogTypes & logTypeFlagInt) != 0 &&
           !serviceProfile.FilterPhrases.Any(logString.Contains);
}
```

So what is the deal with that? Well, Unity has defined the log levels as such in an Enum:
```csharp
namespace UnityEngine
{
  public enum LogType
  {
    Error,
    Assert,
    Warning,
    Log,
    Exception,
  }
}
```

The issue with that is that if you define a property of type `LogType`, you can only select *one value at a time in the inspector*, so you could trap only one type of log to trap. While what I want was this: being able to choose any combination of log types. 

![](/assets/2023-10-24-A-cross-platform-Service-Framework-service-to-write-Unity-log-messages-into-files-on-device/logtypes.png)

So I defined my own Enum 

```csharp
[Flags]
public enum LogTypeFlags
{
    Error     = 1,
    Assert    = 1 << 1,
    Warning   = 1 << 2,
    Log       = 1 << 3,
    Exception = 1 << 4
}
```

And using the fact that an Enum can be cast to an int, and bit shifting a 1 by the int value of `LogType`, I can compare my own Enum with the Unity one. This, of course, entirely depends on the order of the Unity Enum and the assumption (or hope, whatever you want to call it) they will never change this. 

## Result

If you connect your Magic Leap 2, Quest or other android device, you can now simply find logs. Every session gets its own log file. On Magic Leap 2, you can find it at   
**Internal shared storage\Android\data\\[your.package.name.com]\files**

I have created [a little demo project](https://github.com/LocalJoost/UnityFileLogger) that shows how it works, using an extremely lame piece of demo code:

```csharp
public class DemoLogger : MonoBehaviour
{
    private int logFrameCount;
    void Update()
    {
        if (logFrameCount++ % 240 == 0)
        {
            Debug.Log($"DemoLogger Update {logFrameCount}");
        }
    }
}
```
It basically dumps a debug message in the log every 240 frames (which is about 4 seconds). This must be about the least visually compelling demo I ever created, as it shows absolutely nothing. The only effect is the appearance of log files ;). After a run, you will see log files appear on the device, in this case Magic Leap 2:

![](/assets/2023-10-24-A-cross-platform-Service-Framework-service-to-write-Unity-log-messages-into-files-on-device/logfiles.png)

## Concluding words

This service is very useful in finding those weird 'Heisenbugs' that sometimes occur, and actually - it's useful on HoloLens 2 as well, because how often does a user restart the app after something has gone wrong - overwriting the log file immediately. Of course, you can also use Application Insights, AppCenter logging or some other remote logging service - but this always works, connection or not, is simple to use, and has very little impact. Be aware, however, that excessive logging can also have an impact on performance. 

Anyway, as indicated before, [all code involved can be found here](https://github.com/LocalJoost/UnityFileLogger).