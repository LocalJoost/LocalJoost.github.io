---
layout: post
title: App update service for store-less XR devices based on Android (like MetaLense 2)
date: 2026-08-02T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- Service Framework
- MetaLense 2
- Android
published: true
permalink: 
featuredImageUrl: http://localjoost.github.io/assets/2026-08-02-App-update-service-for-store-less-Android-XR-devices-(like-MetaLense-2)/newversion.png
comment_issue_id: 503
---
A while back, I wrote about [a ServiceFramework Service to help update Magic Leap 2 MRTK3 apps](https://localjoost.github.io/A-ServiceFramework-Service-to-help-update-Magic-Leap-2-MRTK3-apps/). Magic Leap 2 has no store, so I built a little service that checks a JSON file online for a newer version, and if found, offers the user a link to download and install it manually. That worked, but it wasn't exactly frictionless: four taps were needed to get from "yes, update" to a running new version.

As you might have noticed I have been [playing around with the MetaLense 2](https://localjoost.github.io/Playing-around-with-the-P&C-MetaLense-2-a-Mixed-Reality-device-from-Korea/). Just like Magic Leap 2, it has no store of any kind: you sideload apps using ADB or P&C's own tooling. So the same problem applies, and the same kind of solution is warranted. But there is no default browser available that opens on `Application.OpenURL(loadedVersionData.Url)`. But because MetaLense 2 is an Android device there is an extra trick available: if you make your app the *Device Owner* of the device, you can let an app download and install *an update of itself*. So I rewrote the service, ported it to MetaLense 2, and added device-owner-based silent installation on top, with some demo code around it.

## What does it look like?

The [demo app](https://github.com/localJoost/MTL2SelfUpdateDemo/) is really basic. If you run it after deploying it, it will say:

![newversion](/assets/2026-08-02-App-update-service-for-store-less-XR-devices-based-on-Android-(like-MetaLense-2)/newversion.png)

This is the 'normal' version, that is: the version that is not the Device Owner. This requires the user to download the app themselves. MetaLense 2 is delivered without a browser that's recognizable for Unity, so you have to type this URL in a browser on your PC or Mac.

If you press your only choice ("Dismiss") the app starts for real, but all it does is show its version number:

![version2](/assets/2026-08-02-App-update-service-for-store-less-XR-devices-based-on-Android-(like-MetaLense-2)/version2.png)

However, if you *have* made the app Device Owner, it shows:

![askinstall](/assets/2026-08-02-App-update-service-for-store-less-XR-devices-based-on-Android-(like-MetaLense-2)/askinstall.png)

If you press "Yes" it will show that it's downloading:

![download](/assets/2026-08-02-App-update-service-for-store-less-XR-devices-based-on-Android-(like-MetaLense-2)/download.png)

If you are lucky you will see the text "Start installing update..." appear or even "Installing update" but most likely the app will just terminate and the launcher will appear. However, if you re-launch the app you don't get a screen that asks you about updating, it will just show 

![version1001](/assets/2026-08-02-App-update-service-for-store-less-XR-devices-based-on-Android-(like-MetaLense-2)/version1001.png)

indicating you have successfully updated. 

## Configuration

The profile is virtually unchanged from the Magic Leap 2 version:

```csharp
[CreateAssetMenu(menuName = "PackageVersionLoaderProfile", fileName = "PackageVersionLoaderProfile",
    order = (int)CreateProfileMenuItemIndices.ServiceConfig)]
public class PackageVersionLoaderProfile : BaseServiceProfile<IServiceModule>
{
    [SerializeField]
    private string packageLocation = string.Empty;
    
    public string PackageLocation => packageLocation;
    
    [SerializeField]
    private string deviceType = string.Empty;
    
    public string DeviceType => deviceType;
}
```

`PackageLocation` points to a JSON file online, and `DeviceType` is (part of) the string that should be found in `SystemInfo.deviceModel` for the version check to actually run. In the demo project, the profile is configured like this:

```
packageLocation: https://www.schaikweb.net/MTL2SelfUpdateDemo/version.json
deviceType: METALENSE
```

The JSON file itself is just as simple as before:

```javascript
{
  "version": "1.0.1.0",
  "url": "https://www.schaikweb.net/MTL2SelfUpdateDemo/MTL2SelfUpdateDemo_1.0.1.0.apk"
}
```

## Version data class

Also unchanged:

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

## Service interface

This is where things start to diverge from the Magic Leap 2 version. Instead of a single `UnityEvent` and a couple of plain properties, I switched to UniRx `ReactiveProperty` objects, because there is now more than one piece of state the UI needs to react to while a silent install is running:

```csharp
public interface IPackageVersionLoader : IService
{
    IReadOnlyReactiveProperty<string> NewVersionFound { get; }
    
    bool CanInstallUpdate { get; }
    
    string PackageLocation { get; }

    void ClearDeviceOwner();

    Task DownloadNewVersion();
    
    IReadOnlyReactiveProperty<string> ProgressUpdateMessage { get; }
    
    IReadOnlyReactiveProperty<bool> HasFailed { get; }
    
    IReadOnlyReactiveProperty<bool> IsInstalling { get; }
    
    void DismissNewVersion();
}
```

* `NewVersionFound` fires with the new version string as soon as a newer version is detected (or `null` if there isn't one)
* `CanInstallUpdate` is `true` when this app is currently the Device Owner - more on that below
* `PackageLocation` is where the new APK can be found - a (shortened) URL
* `ClearDeviceOwner` lets the app give up its Device Owner status again
* `DownloadNewVersion` kicks off the download-and-install process
* `ProgressUpdateMessage`, `HasFailed` and `IsInstalling` let the UI reflect exactly what stage the download/install process is in
* `DismissNewVersion` lets the user say "not now"

## Detecting a new version

The detection logic itself is basically the same as in the Magic Leap 2 version, just rewired to a `ReactiveProperty` instead of a `UnityEvent`:

```csharp
private async Task DetectNewVersion()
{
#if !UNITY_EDITOR
    if (SystemInfo.deviceModel.Contains(profile.DeviceType))
#endif
    {
        var currentApplicationVersion = new Version(Application.version);
        loadedVersionData = await LoadVersionDataFromWeb();

        if (loadedVersionData != null)
        {
            var newVersion = loadedVersionData.ToVersion();
            if (newVersion > currentApplicationVersion)
            {
                PackageLocation = await ShortenUrl(loadedVersionData.Url);
                newVersionFound.Value = loadedVersionData.Version;
                return;
            }
        }
        PackageLocation = null;
        newVersionFound.SetValueAndForceNotify(null);
    }
}
```

One small addition: `PackageLocation` is run through `ShortenUrl`, which calls the free TinyURL API. That's purely cosmetic - it's there for the fallback scenario where the app is *not* Device Owner and has to fall back to showing the user a URL to type or scan, rather than a long one pointing straight at an APK file on my own web server.

## Device Owner: the actual new trick

Device Owner is an Android Enterprise concept, normally used by MDM (Mobile Device Management) software to lock down and manage company-owned devices. It comes with a lot of power: a Device Owner app can silently install and uninstall packages, disable the camera, wipe the device, and more. It can also only be set on a device that has no user accounts configured yet, typically right after a factory reset, and there can be only one Device Owner app per device. You set it via ADB, with the app not even running:

```
adb shell dpm set-device-owner com.metalense.downloaddemo/com.localjoost.services.AdminReceiver
```

`com.metalense.downloaddemo` is this demo app's package name, and `com.localjoost.services.AdminReceiver` is a `DeviceAdminReceiver` that has to be registered for this to work at all. It doesn't need to do anything:

```java
package com.localjoost.services;

import android.app.admin.DeviceAdminReceiver;
import android.content.Context;
import android.content.Intent;

public class AdminReceiver extends DeviceAdminReceiver {
    @Override
    public void onEnabled(Context context, Intent intent) {}

    @Override
    public void onDisabled(Context context, Intent intent) {}
}
```

It just has to exist and be declared in the Android manifest, with the `BIND_DEVICE_ADMIN` permission and a reference to a `device_admin.xml` resource describing which admin policies it wants to use (I use practically none of them, I only need the receiver to exist so `dpm set-device-owner` has something to point at):

```xml
<receiver
    android:name="com.localjoost.services.AdminReceiver"
    android:permission="android.permission.BIND_DEVICE_ADMIN"
    android:exported="true">
    <meta-data
        android:name="android.app.device_admin"
        android:resource="@xml/device_admin" />
    <intent-filter>
        <action android:name="android.app.action.DEVICE_ADMIN_ENABLED" />
    </intent-filter>
</receiver>
```

Once the app is Device Owner, checking that fact from Unity is a one-liner using `DevicePolicyManager`:

```csharp
public bool CanInstallUpdate
{
    get
    {
#if UNITY_ANDROID && !UNITY_EDITOR
        var activity = new AndroidJavaClass("com.unity3d.player.UnityPlayer")
            .GetStatic<AndroidJavaObject>("currentActivity");
        var dpm = activity.Call<AndroidJavaObject>("getSystemService", "device_policy");
        return dpm.Call<bool>("isDeviceOwnerApp", Application.identifier);
#else
        return false;
#endif
    }
}
```

And giving up Device Owner status again - handy for a demo, or when handing the device to someone else - is just as short:

```csharp
public void ClearDeviceOwner()
{
    try
    {
        using var unityPlayer = new AndroidJavaClass("com.unity3d.player.UnityPlayer");
        using var activity = unityPlayer.GetStatic<AndroidJavaObject>("currentActivity");
        using var context = activity.Call<AndroidJavaObject>("getApplicationContext");
        using var dpm = context.Call<AndroidJavaObject>("getSystemService", "device_policy");
        var packageName = context.Call<string>("getPackageName");
        dpm.Call("clearDeviceOwnerApp", packageName);
    }
    catch (Exception ex)
    {
    }
}
```

Note that once cleared, the app can't simply reclaim Device Owner status by itself - that again requires the `adb shell dpm set-device-owner` command, which in turn again requires no accounts to be present on the device. For a demo project that's a fine round trip; for a real deployment you'd typically set Device Owner once, during provisioning, and never clear it again.

## Silently installing the APK

If `CanInstallUpdate` is `true`, `DownloadNewVersion` downloads the APK with a plain `UnityWebRequest`, then hands the file off to a small native helper class:

```csharp
private async Task DownloadAndInstallAsync()
{
    progressMessage.Value = "Downloading update...";
    var downloadPath = System.IO.Path.Combine(Application.persistentDataPath, "update.apk");
    using var www = new UnityWebRequest(loadedVersionData.Url, UnityWebRequest.kHttpVerbGET);
    www.downloadHandler = new DownloadHandlerFile(downloadPath);
    var op = www.SendWebRequest();
    while (!op.isDone) await Task.Yield();
    if (www.result != UnityWebRequest.Result.Success)
    {
        progressMessage.Value = $"Update download failed: {www.error}";
        isInstalling.Value = false;
        hasFailed.Value = true;
        return;
    }

    try
    {
        progressMessage.Value = "Start installing update...";
        var up = new AndroidJavaClass("com.unity3d.player.UnityPlayer");
        var activity = up.GetStatic<AndroidJavaObject>("currentActivity");
        var helper = new AndroidJavaClass("com.localjoost.services.PackageInstallHelper");
        helper.CallStatic("installApk", activity, downloadPath);
        progressMessage.Value = "Installing update...";
    }
    catch (Exception ex)
    {
        progressMessage.Value = $"Update installation failed: {ex.Message}";
        isInstalling.Value = false;
        hasFailed.Value = true;
    }
}
```

`PackageInstallHelper.installApk` is a small Java plugin that uses Android's `PackageInstaller` session API directly:

```java
public static void installApk(final Activity activity, String apkPath) throws Exception {
    PackageInstaller packageInstaller = activity.getPackageManager().getPackageInstaller();
    PackageInstaller.SessionParams params = new PackageInstaller.SessionParams(
        PackageInstaller.SessionParams.MODE_FULL_INSTALL);

    if (android.os.Build.VERSION.SDK_INT >= 31) {
        params.setRequireUserAction(PackageInstaller.SessionParams.USER_ACTION_NOT_REQUIRED);
    }

    File apkFile = new File(apkPath);
    params.setSize(apkFile.length());

    int sessionId = packageInstaller.createSession(params);
    PackageInstaller.Session session = packageInstaller.openSession(sessionId);

    try (InputStream in = new FileInputStream(apkFile);
         OutputStream out = session.openWrite("package", 0, apkFile.length())) {
        byte[] buffer = new byte[65536];
        int read;
        while ((read = in.read(buffer)) != -1) {
            out.write(buffer, 0, read);
        }
        session.fsync(out);
    }
    // ... register a receiver for the install result, then:
    session.commit(pendingIntent.getIntentSender());
    session.close();
}
```

The interesting line is `params.setRequireUserAction(PackageInstaller.SessionParams.USER_ACTION_NOT_REQUIRED)`, available since Android 12 (API 31). This flag is silently ignored - the OS falls back to asking the user - unless the calling app is either the Device Owner or has been granted the `INSTALL_PACKAGES` permission by the system. Since our app *is* Device Owner, the session commits and the new APK gets installed with zero dialogs, zero taps, and (as a side effect) the app gets killed the moment its own new version is being installed.

## Demo UI

The demo has three screens and a hand menu entry, all driven off the same `IPackageVersionLoader` service, retrieved from the `ServiceManager` through a small `StateModel`:

```csharp
public class StateModel : IStateModel
{
    private IPackageVersionLoader packageVersionLoaderService;
    public IPackageVersionLoader PackageVersionLoader
    {
        get
        {
            if (packageVersionLoaderService == null && ServiceManager.IsActiveAndInitialized)
            {
                packageVersionLoaderService = ServiceManager.Instance.GetService<IPackageVersionLoader>();
            }
            return packageVersionLoaderService;
        }
    }
}
```

The screen shown when a new version is found subscribes to all the reactive properties and adapts its buttons depending on whether the app can install silently or not:

```csharp
private void CheckInstallNewVersion(string newVersion)
{
    var isAvailable = !string.IsNullOrEmpty(newVersion);
    visuals.SetActive(isAvailable);
    if (isAvailable)
    {
        text.text = $"New version {newVersion} available!";
    }

    packageVersionLoader.ProgressUpdateMessage.Subscribe(message =>
    {
        prompt.text = message;
    }).AddTo(this);

    packageVersionLoader.IsInstalling.Subscribe(isInstalling =>
    {
        if (isInstalling)
        {
            installButton.gameObject.SetActive(false);
            dismissButton.gameObject.SetActive(false);
            okButton.gameObject.SetActive(false);
        }
    }).AddTo(this);

    packageVersionLoader.HasFailed.Subscribe(hasFailed =>
    {
        if (hasFailed)
        {
            installButton.gameObject.SetActive(false);
            dismissButton.gameObject.SetActive(false);
            okButton.gameObject.SetActive(true);
        }
    }).AddTo(this);

    installButton.gameObject.SetActive(packageVersionLoader.CanInstallUpdate);
    dismissButton.gameObject.SetActive(packageVersionLoader.CanInstallUpdate);
    okButton.gameObject.SetActive(!packageVersionLoader.CanInstallUpdate);

    prompt.text = packageVersionLoader.CanInstallUpdate
        ? "Would you like to install it now? App will abort during \ninstallation, you will have to restart it manually."
        : $"Download new version from\n{packageVersionLoader.PackageLocation}";
}
```

If the app is Device Owner, the user gets "Yes" and "No" buttons, and a warning that the app is about to be killed by the OS mid-install (which it will be - you can't overwrite your own running APK without that happening, silent install or not). If it isn't, the user just gets a single "Dismiss" and the shortened URL to go download the update by hand.

A small hand menu entry lets you clear Device Owner status at will, as already discussed:

```csharp
public void ClearDeviceOwner()
{
    packageVersionLoader.ClearDeviceOwner();
}
```

And because Device Owner status can, in principle, change underneath a running app (someone clearing it via another admin app or `adb`, for example), there's a tiny controller that just polls `CanInstallUpdate` every frame and pops up a brief "Device Owner status was removed" confirmation toast if it flips from `true` to `false`:

```csharp
private async Task Update()
{
    if (isDeviceOwner && !packageVersionLoader.CanInstallUpdate)
    {
        isDeviceOwner = false;
        await Task.Delay(300);
        visuals.transform.localScale = Vector3.one;
        await Task.Delay(4000);
        visuals.transform.localScale = Vector3.zero;
    }
}
```

## Caveats

### About some weird implementations

The reason why the code sometimes contains some weird implementations - like the Update loop above, that does not disable or enable UI elements but only manipulates their scale - is because removing the device owner is something that apparently throws the system quite off kilter. As in: the app crashes, and it crashes so hard it even crashes the MetaLense 2 launcher, falling back to a small 2D default launcher. It took me quite some time to arrive at a state that simply shows this screen, suggesting you leave the app. This might be an Android thing, this might be a MetaLense 2 thing. But this method is pretty stable, albeit a bit kludgy.  

### Missing packages because I don't want to need a lawyer
Also: this app ships without two packages that it really needs to run on a MetaLense 2:
* com.qualcomm.qcht.unity.interactions-4.1.14.tgz
* com.qualcomm.snapdragon.spaces-1.0.4.tgz

This is because it's apparently illegal to redistribute these packages. You can [download these from the Qualcomm developer portal](https://data.spaces.qualcomm.com/spaces-sdk/VRMR_Snapdragon_Spaces_SDK_1_0_4_for_Unity.zip) but you will need to create an account for that. Be aware that Qualcomm is apparently migrating their Spaces to Android XR (at least, that is what the download page suggests); I guess at one point P&C Solution will do so as well - either for the MetaLense 2 itself or a successor device.

## Conclusion

Although this sample is specifically targeted toward the MetaLense 2, and takes into account some of its particular quirks (especially around disabling the Device Ownership), it is a principle usable for all kinds of Android XR devices that come without a store of some kind. 

This obviously isn't something you'd want for just any consumer app - Device Owner is a fairly heavy hammer, and only one app on the device can hold it. But for exactly the kind of purpose-built, store-less, often single-purpose-device scenario that devices like Magic Leap 2 and MetaLense 2 are used for - kiosks, line-of-business apps, devices that are enrolled and managed by whoever deploys them - it's a good fit. As always, you can find the full sample project, including the Android plugin sources, on [GitHub](https://github.com/LocalJoost/MTL2SelfUpdateDemo).

Credits go to [Anthropic Claude](https://claude.com/) for helping me brainstorm, and particularly with the Java code, as Java is not quite my forte anymore since I abandoned it in 2002 to move to C#.