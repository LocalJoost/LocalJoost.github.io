---
layout: post
title: An MRTK3 KeywordRecognitionSubsystem for Magic Leap 2
date: 2023-11-01T00:00:00.0000000+01:00
categories: []
tags:
- MRTK3
- Magic Leap 2
- Subsystems
- KeywordRecognition
- Unity
published: true
permalink: 
featuredImageUrl: http://localjoost.github.io/assets/2023-10-31-An-MRTK3-KeywordRecognitionSubsystem-for-Magic-Leap-2/handtracking2.png
comment_issue_id: 463
---
As I have shown already, MRTK3 is very versatile and very well suited for cross-platform development. The architecture is extensible by the usage of Unity *subsystems*. At Magic Leap, they have used that to implement a version of the Hands subsystem - that allows hand tracking to work - and you can use that by simply selecting a different implementation of the subsystem in the Android MRTK3 profile:

![](/assets/2023-11-01-An-MRTK3-KeywordRecognitionSubsystem-for-Magic-Leap-2/handtracking2.png)

For contrast, this is the HoloLens 2 profile. Here the OpenXR Hands API is selected instead of the Magic Leap one.

![](/assets/2023-11-01-An-MRTK3-KeywordRecognitionSubsystem-for-Magic-Leap-2/handtracking.png)

Magic Leap 2 also supports keyword recognition - there is a custom API for that in their SDK. It's things like this that make cross-platform development difficult. Therefore I could not get the speech command in my port of [Augmedit](https://www.augmedit.com/)'s Lumi product to work. If only someone had thought of making a KeywordRecognitionSubsystem implementation so you could just as easily use keyword recognition as on HoloLens 2...

![](/assets/2023-11-01-An-MRTK3-KeywordRecognitionSubsystem-for-Magic-Leap-2/keywordrecog.png)

Well, good news. Someone just did. Yours truly.

## Subsystems in MRTK3

I will not even start to pretend I completely understand how subsystems work, but I have a kind of an idea now. In theory, they can consist of 5 classes and an interface:
* The subsystem itself - this is the class that actually registers itself as a subsystem
* A 'provider' (that does the actual work). Both subsystem and provider implement the same interface, and the subsystem basically forwards all actual work to the provider. 
* A 'descriptor' (that is used for registering the class)
* A configuration class (allows you to configure parameters of the subsystem)
* A 'Cinfo' class whose purpose is not entirely clear to me, but it is necessary for registering the subsystem.
* An interface that describes the public methods of the subsystem (and the provider)

## KeywordRecognitionSubsystem

For KeywordRecognitionSubsystems, there's already a lot of heavy lifting done. We actually only need to provide a subsystem and a provider and can do so by extending existing base classes for keyword recognition. The entire subsystem therefore looks like this: 

```csharp
[Preserve]
[MRTKSubsystem(
    Name = "MRTKExtensions.MagicLeap.SpeechRecognition",
    DisplayName = "MRTK MagicLeap KeywordRecognition Subsystem",
    Author = "LocalJoost",
    ProviderType = typeof(MagicLeapKeywordRecognitionProvider),
    SubsystemTypeOverride = typeof(MagicLeapKeywordRecognitionSubsystem))]
public class MagicLeapKeywordRecognitionSubsystem : 
                KeywordRecognitionSubsystem
{
#if MAGICLEAP
    [RuntimeInitializeOnLoadMethod(
        RuntimeInitializeLoadType.SubsystemRegistration)]
    static void Register()
    {
        var cinfo = XRSubsystemHelpers.
            ConstructCinfo<MagicLeapKeywordRecognitionSubsystem,
            KeywordRecognitionSubsystemCinfo>();

        if (!Register(cinfo))
        {
            Debug.LogError($"Failed to register the {cinfo.Name} subsystem.");
        }
    }
#endif
}
```

The `MRTKSubsystem` describes the subsystem and indicates which Subsystem and Provider are to be connected together. Optionally you can also configure a ConfigType in this attribute. The subsystem itself then only needs a static method decorated with a `RuntimeInitializeOnLoadMethod` attribute to actually get launched on startup. It uses `KeywordRecognitionSubsystemCinfo` cinfo class - which is part of the MRTK3, so we don't have to create this ourselves. As I said before, I don't know why this is necessary, but sometimes you simply have to ~~do some cargo cult programming~~ follow existing patterns.

## The Subsystem provider

The provider does the actual work. The provider derives from the abstract class `KeywordRecognitionSubsystem.Provider`, which requires us to implement the following methods:
* `UnityEvent CreateOrGetEventForKeyword(string keyword);`
* `void RemoveKeyword(string keyword);`
* `void RemoveAllKeywords();`
* `IReadOnlyDictionary<string, UnityEvent> GetAllKeywords();`

Also, there are several life cycle methods we can override coming from `KeywordRecognitionSubsystem.Provider` - methods that are largely the same as in a behaviour. 

### Starting/initializing the provider

In the `Start` method override, I have implemented some code to initialize the voice recognition. Attentive readers will notice that this - and a lot of the following code - is basically a modified version of the [runtime voice intents sample on the Magic Leap developer docs](https://developer-docs.magicleap.cloud/docs/guides/unity/input/voice-intents/runtime-voice-intents-example/) - with a few additions by me to take care of some idiosyncrasies I ran into.

```csharp
[Preserve]
internal class MagicLeapKeywordRecognitionProvider :
  KeywordRecognitionSubsystem.Provider
{
    private int commandId = 0;
    private MLVoiceIntentsConfiguration voiceConfiguration;

    public override void Start()
    {
        base.Start();
        if (voiceConfiguration == null)
        {
            voiceConfiguration = 
                ScriptableObject.CreateInstance<MLVoiceIntentsConfiguration>();
            voiceConfiguration.VoiceCommandsToAdd = 
                new List<MLVoiceIntentsConfiguration.CustomVoiceIntents>();
            voiceConfiguration.AllVoiceIntents = 
                new List<MLVoiceIntentsConfiguration.JSONData>();
            voiceConfiguration.SlotsForVoiceCommands = 
                new List<MLVoiceIntentsConfiguration.SlotData>();
        }

        if (!running)
        {
            MLVoice.OnVoiceEvent += OnVoiceEvent;
        }
    }
}
```

The line `voiceConfiguration.SlotsForVoiceCommands` = ... is one of those additions that proved to be necessary - if I didn't add that, I got a null reference error. Note that `running` is a read-only base class property that is set internally.

### Adding or getting keywords

You can see that events and keywords are both added to the internal dictionary, as well as 'intent' to the Magic Leap API. `GetAllKeywords` simply returns the keyword/event dictionary

```csharp
public override UnityEvent CreateOrGetEventForKeyword(string keyword)
{
    if (!keywordDictionary.ContainsKey(keyword))
    {
        keywordDictionary.Add(keyword, new UnityEvent());
        AddIntentForKeyword(keyword);
        SetupVoiceIntents();
    }
    return keywordDictionary[keyword];
}

public override IReadOnlyDictionary<string, UnityEvent> GetAllKeywords()
{
    return keywordDictionary;
}
```
`keywordDictionary`, by the way, is once again a base class property. It is a simple `Dictionary<string, UnityEvent>`. 

## Removing keywords

Here the same pattern: remove from the internal dictionary, remove intent. Oh, and disconnect any listeners from the event before we toss it out.

```csharp
public override void RemoveKeyword(string keyword)
{
    if(keywordDictionary.TryGetValue(keyword, out var eventToRemove))
    {
        eventToRemove.RemoveAllListeners();
        keywordDictionary.Remove(keyword);
        voiceConfiguration.AllVoiceIntents.Remove(
            voiceConfiguration.AllVoiceIntents.First(k=> k.value == keyword));
        SetupVoiceIntents();
    }
}

public override void RemoveAllKeywords()
{
    foreach( var eventToRemove in keywordDictionary.Values)
    {
        eventToRemove.RemoveAllListeners();
    }
    keywordDictionary.Clear();
    voiceConfiguration.AllVoiceIntents.Clear();
    SetupVoiceIntents();
}
```

### Stop/destroy

If the subsystem is halted or destroyed, we need to handle things in the `Stop` and `Destroy` life cycles method, which are now pretty easy to make:

```csharp
public override void Stop()
{
    base.Stop();
    MLVoice.OnVoiceEvent -= OnVoiceEvent;
}

public override void Destroy()
{
    base.Destroy();
    RemoveAllKeywords();
    Stop();
}
```

### Internal workings

Now we have set up the framework, there's some little implementation details left. In the `Start` method, we connected the `OnVoiceEvent` method to the  `MLVoice.OnVoiceEvent` event. The implementation is pretty simple: we simply check if a keyword is recognized and if so find the event belonging to it - and then invoke that event, notifying possible external listeners.

```csharp
private void OnVoiceEvent(in bool wasSuccessful, 
                          in MLVoice.IntentEvent voiceEvent)
{
    if (wasSuccessful)
    {
        if (keywordDictionary.TryGetValue(voiceEvent.EventName, 
                out var value))
        {
            value?.Invoke();
        }
    }
}
```

`AddIntentForKeyword` creates the actual intent. The `commandId` needs to be unique, so we simply use an incrementing integer

```csharp
private void AddIntentForKeyword(string keyword)
{
    var newIntent = new MLVoiceIntentsConfiguration.CustomVoiceIntents
    {
        Value = keyword,
        Id = (uint)commandId++
    };
    voiceConfiguration.VoiceCommandsToAdd.Add(newIntent);
}
```

The last method is quite weird. In all methods, you can see that after adding or removing intents, the method `SetupVoiceIntents` is called. This is because for any change of intents to be recognized, `MLVoice.SetupVoiceIntents` needs to be called. At least, that seems to be the case. Here's another idiosyncrasy I found: if an `MLVoiceIntentsConfiguration` contains zero intents, MLVoice.SetupVoiceIntents throws an exception. So if it's empty, I make sure there is at least one dummy intent - without an event

```csharp
private void SetupVoiceIntents()
{
    if (!voiceConfiguration.AllVoiceIntents.Any())
    {
        AddIntentForKeyword("dummyxyznotempty");
    }
    MLVoice.SetupVoiceIntents(voiceConfiguration);
}
```

## Sample code

To show it actually works, I have added a small piece of demo code that allows you to verify this actually works. If you have configured `MagicLeapKeywordRecognitionSubsystem` as your keyword recognition system in the MRTK3 settings and deploy it to the Magic Leap, you can use see this user interface:

![](/assets/2023-11-01-An-MRTK3-KeywordRecognitionSubsystem-for-Magic-Leap-2/ui.png)

By default, it recognizes three phrases. You can add a "Hello there", remove it again, remove all, restore the initial commands, and toggle on/off the whole recognizer. Adding commands goes like this:

```csharp
public void InitStandardPhrases()
{
    RemoveAll();
    keywordRecognitionSubsystem.CreateOrGetEventForKeyword("Good morning").
        AddListener(() => ShowRecognizedCommand("Good morning"));
    keywordRecognitionSubsystem.CreateOrGetEventForKeyword("Nice weather").
        AddListener(() => ShowRecognizedCommand("Nice weather"));
    keywordRecognitionSubsystem.CreateOrGetEventForKeyword("Mixed Reality is cool").
        AddListener(() => ShowRecognizedCommand("Mixed Reality is cool"));
    UpdateRecognizedCommands();
}
```

Removing keywords goes like this:

```csharp
public void RemoveHello()
{
    keywordRecognitionSubsystem.RemoveKeyword("Hello there");
    UpdateRecognizedCommands();
}

public void RemoveAll()
{
    keywordRecognitionSubsystem.RemoveAllKeywords();
    UpdateRecognizedCommands();
}
```

and controlling the keyword recognizer like this:

```csharp
public void ToggleKeywordRecognition()
{
    if (keywordRecognitionSubsystem.running)
    {
        keywordRecognitionSubsystem.Stop();
    }
    else
    {
        keywordRecognitionSubsystem.Start();
    }
}
```

## Concluding words

Using this keyword recognizer, you can use exactly the same MRTK3 API for creating and responding to keywords as you were used to using on HoloLens, with the `WindowsKeywordRecognitionSubsystem`. The Magic Leap API is neatly encapsulated, and as far as speech control for your app goes, it doesn't matter whether it's running on a HoloLens 2 or a Magic Leap 2. As a consequence,[Augmedit Lumi](https://www.augmedit.com/) now *does* support speech recognition on Magic Leap - without any code change for that.

[Full code, as (nearly) always, on GitHub.](https://github.com/LocalJoost/MLMRTK3KeywordRecognitionSubsystem.git)