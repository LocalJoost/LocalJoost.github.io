---
layout: post
title: A ServiceFramework Service to help update Magic Leap 2 MRTK3 apps
date: 2023-09-15T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- Service Framework
- Magic Leap 2
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2023-09-15-A-ServiceFramework-Service-to-help-update-Magic-Leap-2-MRTK3-apps/MLUpdater.gif
comment_issue_id: 456
---
When I talked to the folks at Magic Leap about getting a loaner Magic Leap 2, my first plan was to port [HoloATC](https://localjoost.github.io/holoatcml2/) to their device. My second plan was to get it ready for whatever Store Magic Leap uses for their device and publish it there. They thought that was a great idea, but there was a tiny issue with part two of the plan: they don't have a Store of any kind for Magic Leap 2. You just put your app out somewhere where people can download it, and they install it using the Magic Leap Device Bridge (a desktop program). They don't do walled gardens. There are certain advantages to that, like not having to create accounts, deal with developer certificates, etc. There are also drawbacks, one of them being: how are users going to learn about new versions and how are they going to install them? I made a little something to solve that problem, using a Service Framework Service. It looks like this:

![](/assets/2023-09-15-A-ServiceFramework-Service-to-help-update-Magic-Leap-2-MRTK3-apps/MLUpdater.gif)

Please note that I have sped up the parts where you would be waiting for processes to complete. I didn't think it would be particularly interesting to show Unity loading screens or moving progress bars.

## Configuration

Most services have a configuration, and so does this one. It has only two parameters: a string containing (part of) the device type name (according to Unity's `SystemInfo.deviceModel`), and a location where to find version info.

```csharp
[CreateAssetMenu(menuName = "LocalJoost/PackageVersionLoaderProfile", 
    fileName = "PackageVersionLoaderProfile",
    order = (int)CreateProfileMenuItemIndices.ServiceConfig)]
public class PackageVersionLoaderProfile : BaseServiceProfile<IServiceModule>
{
    [SerializeField]
    private string versionInfoLocation = string.Empty;
    
    public string VersionInfoLocation => versionInfoLocation;

    [SerializeField]
    private string deviceType = "Magic Leap";
    
    public string DeviceType => deviceType;
}
```
Note that the deviceType is "Magic Leap" by default, but in principle, this can be used for spreading updates to any device you want, bypassing a store (although if there's a store, why not use it and let someone else handle the hassle of the process). Anyway, VersionInfoLocation should point to a simple JSON file somewhere online. In my sample, it points to https://www.schaikweb.net/VersionDemo/version.json. This file looks like this:

```javascript
{
  "version": "1.0.1.0",
  "url": "https://www.schaikweb.net/VersionDemo/MLUpdater_1.0.1.0.apk"
}
```

Very simple: the new version number, and the location where to find it. You will find a copy of this file in the root of the demo project. This is all the info we need for alerting the user to a new version and offering to install and update the version.

## Version data class

```csharp
[Serializable]
public class VersionData
{
    [JsonProperty("version")]
    public string Version { get; set; }
    
    [JsonProperty("url")]
    public string Url { get; set; }

    public Version ToVersion()
    {
        return new Version(Version);
    }
}
```

To be able to easily use the configuration data, I created this very simple data class, featuring a small helper method to convert the version string into a real `System.Version` object. This allows us to easily compare versions without having to resort to all kinds of string comparison hoopla.

## Service Interface 

The service has only 4 public members:

```csharp
public interface IPackageVersionLoader : IService
{
    UnityEvent LatestVersionDataLoaded { get; }
    
    bool IsNewVersionAvailable { get; }
    
    string LatestVersion { get; }

    void DownloadNewVersion();
}
```

* The event is fired as soon as the version data loading process is finished (whatever the result is)
* IsNewVersionAvailable becomes true when indeed a newer version has been found
* LatestVersion gets the result of the version data loading process (if any)
* DownloadNewVersion is a method that can be called when the user indeed wants to download a new version

## Loading version data

This is actually a fairly simple piece of code.

```csharp

private VersionData loadedVersionData;
public UnityEvent LatestVersionDataLoaded { get; } = new ();
public string LatestVersion => loadedVersionData?.Version;
public bool IsNewVersionAvailable { get; private set; }

private async Task DetectNewVersion()
{
#if !UNITY_EDITOR
   if (SystemInfo.deviceModel.Contains(profile.DeviceType))
#endif
    {
        var currentApplicationVersion = new Version(Application.version);
        loadedVersionData = await DownloadVersionData();

        if (loadedVersionData != null)
        {
            var newVersion = loadedVersionData.ToVersion();
            IsNewVersionAvailable = newVersion > currentApplicationVersion;
        }
        
        LatestVersionDataLoaded.Invoke();
    }
}
```

`DetectNewVersion` is called from the service's `Start` method. It creates a `System.Version` object from the app's current version (`Application.version`), downloads the version data from the location in the profile into the field `loadedVersionData`, compares it to the app version, and puts the result of that comparison in `IsNewVersionAvailable`. After that, the `LatestVersionDataLoaded` is fired. Note the device type check is between a preprocessor directive (`#if !UNITY_EDITOR`), I did this specifically to make it easily testable in the Unity editor because in Play Mode, the device type check is skipped.

The `DownloadVersionData` method uses a fairly standard way of downloading JSON data and deserializing it

```csharp
private async Task<VersionData> DownloadVersionData()
{
    try
    {
        using (var client = new HttpClient())
        {
            var response = await client.GetAsync(profile.VersionInfoLocation);
            if (response.IsSuccessStatusCode)
            {
                var result = JsonConvert.DeserializeObject<VersionData>(
                    await response.Content.ReadAsStringAsync());
                return result;
            }
        }

        return null;
    }
    catch (Exception ex)
    {
        return null;
    }
}
```

And that is basically all. You put a ServiceManager in your scene, create a profile, configure the PackageVersionLoader in the ServiceManager, and then there is the tiny detail of handling all the logic and showing an appropriate user interface. I made a rough demo to show how it can be used in action:

## Demo code

The demo features two simple screens. The first one is shown when there is an update indeed, 

![](/assets/2023-09-15-A-ServiceFramework-Service-to-help-update-Magic-Leap-2-MRTK3-apps/newversion.png)

and uses this little behavior as a driver:

```csharp
public class HandleNewVersionBehaviour : MonoBehaviour
{
    [SerializeField]
    private GameObject menuRoot;
    
    [SerializeField]
    private TextMeshProUGUI newVersionText;

    private IPackageVersionLoader packageLoadingService;

    void Start()
    {
        menuRoot.SetActive(false);
        packageLoadingService = 
          ServiceManager.Instance.GetService<IPackageVersionLoader>();
        packageLoadingService.LatestVersionDataLoaded.
          AddListener(ProcessVersionData);
        if (packageLoadingService.LatestVersion != null)
        {
            ProcessVersionData();
        }
    }

    private void ProcessVersionData()
    {
        if (packageLoadingService.IsNewVersionAvailable)
        {
            newVersionText.text = 
              $"New version {packageLoadingService.LatestVersion} available!";
            menuRoot.SetActive(true);
        }
        else
        {
            menuRoot.SetActive(false);
        }
    }
}
```
It initially turns its insides off, then gets a reference to the `IPackageVersionLoader` service. It subscribes to the `LatestVersionDataLoaded` event, and, for good measure, also checks the `LatestVersion` property. After all, a Service typically starts pretty fast, so the version data might have been loaded already before this code even has gotten the chance to subscribe to the event. In case there is a new version, it shows the available version and turns on its insides.

It has two more methods, that are connected to the "Yes" and "No" buttons:

```csharp
public void Dismiss()
{
    Destroy(gameObject);
}

public void DownloadNewVersion()
{
    packageLoadingService.DownloadNewVersion();
    menuRoot.SetActive(false);
}
```

I assume these last two methods to be pretty self-explanatory.

The second screen is displayed when there is no new version:

![](/assets/2023-09-15-A-ServiceFramework-Service-to-help-update-Magic-Leap-2-MRTK3-apps/nonewversion.png)

And its behavior is rather trivial:

```csharp
public class HandleNewNoVersionBehaviour : MonoBehaviour
{
    [SerializeField]
    private GameObject menuRoot;

    private IPackageVersionLoader packageLoadingService;

    void Start()
    {
        menuRoot.SetActive(false);
        packageLoadingService = 
          ServiceManager.Instance.GetService<IPackageVersionLoader>();
        packageLoadingService.LatestVersionDataLoaded.
          AddListener(ProcessVersionData);
        if (packageLoadingService.LatestVersion != null)
        {
            ProcessVersionData();
        }
    }

    private void ProcessVersionData()
    {
        menuRoot.SetActive(!packageLoadingService.IsNewVersionAvailable);
    }

    public void Dismiss()
    {
        Destroy(gameObject);
    }
}
```
It turns its insides on when there is no new version, and the only button calls the Dismiss method, effectively closing the screen.

## Conclusion

Using this service and a little bit of UI, you can now easily let users update their apps on Magic Leap 2. For enterprise-level apps, a more adequate security measure other than just putting it on a public site might be necessary, but for less stringent cases, this is an easy way to get around the 'no store' issue. However, it's not quite hassle-free: users have to perform four clicks to download and install the app:
* First, click "Yes" on my homegrown screen
* Then, click the small icon at the bottom left of the browser that opens automatically
* After that, click the downloaded APK in the file manager that pops open
* Finally, click "Yes" on the OS confirmation screen.

But it works for me. As always, you can download a [sample project with all code from GitHub](https://github.com/LocalJoost/MLUpdater). For those who just want to try out the sample and see how the process works, you can also simply download and install v1.0.0.0, which will then ask for 1.0.1.0, just like in the video at the beginning of this post.
