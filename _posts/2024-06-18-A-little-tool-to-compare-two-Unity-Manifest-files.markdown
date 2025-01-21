---
layout: post
title: A little tool to compare two Unity Manifest files
date: 2024-06-18T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- Unity
- Magic Leap 2
- HoloLens2
published: true
permalink: 
featuredImageUrl: 
dontStripH1Header: false
comment_issue_id: 471
---
I recently published a [Walk the World version for Magic Leap 2](https://www.linkedin.com/feed/update/urn:li:activity:7208123082929025024/), but I would be lying if I said it was as easy this time as when I ported HoloATC. Things have changed massively since Magic Leap fully embraced the OpenXR standard, and the process of re-configuring an existing MRKT3 HoloLens 2 app to run on Magic Leap 2 has become a bit more complex. One of the things I could not really figure out definitively from the documentation was: what extra packages I needed to install, and what versions. At the first try, my app did not show anything; at the second, it crashed at startup.

However, as they state [here](https://developer-docs.magicleap.cloud/docs/guides/third-party/mrtk3/mrtk3-template/#option-1--magic-leap-2-mrtk3-template-project), there is what they call a '[template project](https://github.com/magicleap/MixedRealityToolkit-Unity.git)' that actually runs, but unfortunately, that is a fork of the MRTK3 itself. So it's built upon the sources, not the packages, and therefore not really usable to build your own project on.

## Talk is cheap, show me the manifest

But it can be used as a sample, and more importantly, it has a manifest file and that very definitively showed what I need. Normally this is maintained using the package manager, but in the end, it's only a JSON file that can be edited using any old text editor. Unfortunately, it also tends to be pretty long, and comparing two of those files can be a time-consuming and error-prone process. To make sure I didn't make any (more) mistakes, I wrote a small .NET 8 tool called **UnityManifestCompare** to do that for me. It basically shows:

* What extra scoped registries the first manifest had over the second one
* What extra packages the first manifest had over the second one
* What packages appear in both but have different versions.

## JSON data structure

As I said, the manifest file is simply a JSON structure, so first, I defined two simple data classes to hold that JSON. We start with the manifest structure itself

```csharp
internal class Manifest
{
    [JsonProperty("scopedRegistries")]
    public List<ScopedRegistry> ScopedRegistries { get; set; } = new();

    [JsonProperty("dependencies")]
    public Dictionary<string,string> Dependencies { get; set; } = new();
}
```

And then the scoped registry:

```csharp
internal class ScopedRegistry
{
    [JsonProperty("name")]
    public string Name { get; set; }

    [JsonProperty("url")]
    public string Url { get; set; } = new();

    [JsonProperty("scopes")]
    public List<string> Scopes { get; set; } = new();
}
```

## Main program

The main program itself accepts two arguments. The first one is the source (in my case the Magic Leap project manifest file) and the second one the 'target', the one that should be adapted. It does the three comparison tasks and prints some layout stuff in between:

```csharp
internal class Program
{
    static void Main(string[] args)
    {
        var firstManifest = JsonConvert.DeserializeObject<Manifest>(
           File.ReadAllText(args[0]));
        var secondManifest = JsonConvert.DeserializeObject<Manifest>(
           File.ReadAllText(args[1]));

        ShowExtraScopedRegistries(firstManifest, secondManifest);
        Console.WriteLine();
        Console.WriteLine(
           "===============================================================");
        Console.WriteLine();
        ShowExtraDependencies(firstManifest, secondManifest);
        Console.WriteLine();
        Console.WriteLine(
           "===============================================================");
        Console.WriteLine();
        ShowVersionDifferences(firstManifest, secondManifest);
    }
}
```

## Added scoped registries

This is simply getting all names from the first registry list that are not in the second list, conveniently printing them out in a format that can be added to the manifest file directly:

```csharp
private static void ShowExtraScopedRegistries(Manifest firstManifest, 
                                              Manifest secondManifest)
{
    var uniqueToFirstManifest = firstManifest.ScopedRegistries
        .Where(entry => 
           secondManifest.ScopedRegistries.All(sr => sr.Name != entry.Name))
        .ToList();

    Console.WriteLine("Scoped registries missing from the second manifest:");
    Console.WriteLine("---------------------------------------------------");
    foreach (var scopedRegistry in uniqueToFirstManifest)
    {
        Console.WriteLine($"{JsonConvert.SerializeObject(scopedRegistry, 
          Formatting.Indented)},");
    }
}
```

Result, in my case:
```json
Scoped registries missing from the second manifest:
---------------------------------------------------
{
  "name": "Magic Leap",
  "url": "http://registry.npmjs.org",
  "scopes": [
    "com.magicleap"
  ]
},
```

I added this to my manifest file, which now looks like this:

```json
{
  "scopedRegistries": [
    {
      "name": "OpenUPM",
      "url": "https://package.openupm.com",
      "scopes": [
        "com.atteneder",
        "com.github-glitchenzo.nugetforunity",
        "com.openupm",
        "com.realitycollective"
      ]
    },
    {
      "name": "Magic Leap",
      "url": "http://registry.npmjs.org",
      "scopes": [
        "com.magicleap"
      ]
    }
  ],
```

Do not forget to remove the trailing comma if you add it to the bottom!

## Added packages

Also not rocket science: simply list all the key-value combinations where the key does not appear in the second list.

```csharp
private static void ShowExtraDependencies(Manifest firstManifest, 
                                          Manifest secondManifest)
{
    var uniqueToFirstManifest = firstManifest.Dependencies
        .Where(entry => !secondManifest.Dependencies.ContainsKey(entry.Key))
        .ToDictionary(entry => entry.Key, entry => entry.Value);

    Console.WriteLine("Dependencies missing from the second manifest:");
    Console.WriteLine("----------------------------------------------");

    foreach (var dependency in uniqueToFirstManifest)
    {
        Console.WriteLine($"\"{dependency.Key}\": \"{dependency.Value}\",");
    }
}
```

Result:

```diff
Dependencies missing from the second manifest:
----------------------------------------------
"com.atteneder.ktx": "https://github.com/atteneder/KtxUnity.git#v2.1.2",
"com.magicleap.mrtk3": "1.0.0",
"com.magicleap.soundfield": "3.4.4-231122.68.6849fab",
"com.magicleap.unitysdk": "2.2.0",
"com.microsoft.mixedreality.visualprofiler": "https://github.com/microsoft/VisualProfiler-Unity.git#v2.2.0",
"com.unity.asset-store-validation": "0.5.1",
"com.unity.inputsystem": "1.7.0",
"com.unity.mobile.android-logcat": "1.4.2",
"com.unity.performance.profile-analyzer": "1.2.2",
"com.unity.xr.arcore": "5.1.4",
"com.unity.xr.arfoundation": "5.1.4",
"com.unity.xr.hands": "1.4.1",
"com.unity.xr.interaction.toolkit": "2.5.4",
"com.unity.xr.magicleap": "7.0.0",
"com.unity.xr.management": "4.4.0",
"com.unity.xr.openxr": "1.10.0",
"org.mixedrealitytoolkit.accessibility": "file:../../../org.mixedrealitytoolkit.accessibility",
"org.mixedrealitytoolkit.data": "file:../../../org.mixedrealitytoolkit.data",
```

Now you have to be careful with everything that has "file:" in it as this refers to local files. So if you need those, you should install them from the MRTK Feature Tool. I didn't need them, so I skipped them. The important thing is: now you see all the extra stuff Magic Leap needs for an MRTK3 project, and you can copy it simply into the dependencies section of your manifest. I even added the commas at the end for convenience.

## Different versions:

Basically the same idea: show everything that has the same key but a different value. This is the least interesting part, as this can be fixed mostly with the package manager, and also usually has less impact than missing packages:

```csharp
private static void ShowVersionDifferences(Manifest firstManifest, 
                                           Manifest secondManifest)
{
    var differentVersions = firstManifest.Dependencies
        .Where(entry => secondManifest.Dependencies.ContainsKey(entry.Key) 
          && secondManifest.Dependencies[entry.Key] != entry.Value)
        .ToDictionary(entry => entry.Key, entry => (entry.Value, 
           secondManifest.Dependencies[entry.Key]));

    Console.WriteLine("Dependencies with different versions:");
    Console.WriteLine("-------------------------------------");

    foreach (var dependency in differentVersions)
    {
        Console.WriteLine(
        $"{dependency.Key}: {dependency.Value.Item1} vs {dependency.Value.Item2}");
    }
}
```

Since I installed the MRTK from the MRTK Feature Tool, every package was listed, so I am not going to list the whole output. I just show a few samples that showed some inconsequential version changes:

```diff
Dependencies with different versions:
-------------------------------------
com.atteneder.gltfast: https://github.com/atteneder/glTFast.git#v4.8.3 vs 5.0.4
com.microsoft.mixedreality.openxr: file:../../../ExternalDependencies/com.microsoft.mixedreality.openxr-1.10.0.tgz vs file:MixedReality/com.microsoft.mixedreality.openxr-1.10.1.tgz
com.microsoft.mrtk.graphicstools.unity: https://github.com/microsoft/MixedReality-GraphicsTools-Unity.git?path=/com.microsoft.mrtk.graphicstools.unity#v0.6.6 vs file:MixedReality/com.microsoft.mrtk.graphicstools.unity-0.7.0.tgz
com.microsoft.mrtk.tts.windows: file:../../../ExternalDependencies/com.microsoft.mrtk.tts.windows-1.0.4.tgz vs file:MixedReality/com.microsoft.mrtk.tts.windows-1.0.4.tgz
com.microsoft.spatialaudio.spatializer.unity: file:../../../ExternalDependencies/com.microsoft.spatialaudio.spatializer.unity-2.0.55.tgz vs file:MixedReality/com.microsoft.spatialaudio.spatializer.unity-2.0.55.tgz
com.unity.textmeshpro: 3.0.6 vs 3.0.7
```

## Concluding words

Using this tool, and courtesy of Magic Leap now hosting everything - including their MRTK3 package - on their own registry, there's no need to mess around with the Magic Leap Setup Tool to connect to local copies of the SDK and the Unity Package manager to get everything you need. In my case, Unity even hung twice after importing the SDK - this does not happen now. Please note: you still *do need* the Setup Tool - but for the other settings. And of course, UnityManifestCompare can be used to compare the setup for other purposes than configuring for Magic Leap as well - you can just as easily use it to port an app from Magic Leap 2 to HoloLens 2 - but it was a vital step in my porting process.

By the way: most actual code written by GitHub CoPilot - with some minor additions by me :).

[Project code, as usual, on GitHub.](https://github.com/LocalJoost/UnityManifestCompare). You can also [directly download the built project](https://github.com/LocalJoost/UnityManifestCompare/releases/download/1.0.0.0/UnityManifestCompare.zip) if you don't want to build it yourself. You simply run it from the command line:

```dos
UnityManifestCompare manifest1.json manifest2.json
```

Note: 
* Please make a backup of your manifest.json before you start messing with it
* To run use the latest MRTK3 stuff on Magic Leap, you need the latest MRTK3. More details later.