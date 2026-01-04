---
layout: post
title: MRTK3 apps on Apple Vision Pro - fixing the stuck to your head issue (and more)
date: 2026-01-04T00:00:00.0000000+01:00
categories:
- Apple
- MRTK3
- Unity
- Vision Pro
tags: []
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2026-01-04-MRTK3-apps-on-Apple-Vision-Pro--fixing-the-stuck-to-your-head-issue-(and-more)/Headlock.gif
dontStripH1Header: false
comment_issue_id: 498
---
Ah, Unity. The company that was instrumental in me being able to venture into Mixed Reality, the very embodiment of the Silicon Valley motto "move fast and break things"... with unfortunately, *emphasis on the second part*.

Early HoloLens days, I learned an important lesson quickly: one of the scariest things you could do is upgrade your apps to a new Unity version - even just a point release could hose your app. As fellow Mixed Reality developer [Oliver Vasvari]() noted in [comments](https://localjoost.github.io/Adapting-MRTK3-apps-for-Apple-Vision-Pro/) on my [August blog about running MRTK3 apps on Apple Vision Pro](https://localjoost.github.io/Adapting-MRTK3-apps-for-Apple-Vision-Pro/), my solution (based on Unity 6000.0.49) was still working on 6000.0.53, but fell apart on 6000.0.62. Specifically, the XR camera view was 'stuck' to your head - that is, every virtual object was moving along with your head on the Vision Pro.

![](/assets/2026-01-04-MRTK3-apps-on-Apple-Vision-Pro--fixing-the-stuck-to-your-head-issue-(and-more)/Headlock.gif)

This kind of thing is unfortunately part of the life of a Unity Mixed Reality developer. So I set out to find if I could revive my solution. Spoiler - *I could*.

Starting point was [my last Vision Pro MRTK port](https://github.com/LocalJoost/QuestBouncer/tree/AppleVisionPro). I went for broke and did not just upgrade to the version where Oliver noticed the problem, but jumped to the newest Unity 6.3 version (which was 6000.3.2f1 at the point I wrote that).

## Upgrade project to 6.3

Straightforward enough: open the solution with 6000.3.2f1. This apparently needs an URP upgrade:

![](/assets/2026-01-04-MRTK3-apps-on-Apple-Vision-Pro--fixing-the-stuck-to-your-head-issue-(and-more)/upgradeurp.png)

Click OK, and then you will be greeted by another issue:

![](/assets/2026-01-04-MRTK3-apps-on-Apple-Vision-Pro--fixing-the-stuck-to-your-head-issue-(and-more)/package_error.png)

Click open Package Manager and you will notice my hacked 2.2.4 Vision OS XR Plugin is no longer accepted.

![](/assets/2026-01-04-MRTK3-apps-on-Apple-Vision-Pro--fixing-the-stuck-to-your-head-issue-(and-more)/hackedpackage.png)

Uninstall it, and Unity will proceed to install the newest Vision Pro packages:

![](/assets/2026-01-04-MRTK3-apps-on-Apple-Vision-Pro--fixing-the-stuck-to-your-head-issue-(and-more)/newvisonpropackages.png)

## Upgrade MRTK3 (optional)

In [my previous Vision Pro post](https://localjoost.github.io/Adapting-MRTK3-apps-for-Apple-Vision-Pro/) I used MRTK3 4.0.0-pre.1, but in the meantime 4.0.0-pre.2 has been released. It would be a shame not to use all the good work done by [Kurtis Eveleigh](https://www.linkedin.com/in/kurtie/), the proud MRTK custodian. The MRTK Feature Tool has been abandoned, and never worked on the Mac anyway, so the only way to upgrade I can think of is to [download the packages manually](https://github.com/MixedRealityToolkit/MixedRealityToolkit-Unity/releases/tag/core-v4.0.0-pre.2), and put them in the Packages/MixedReality folder. The quickest way to effectuate the upgrade is to open the manifest.json in a text editor, do a search & replace on pre.1, and replace it with pre.2. You can then, of course, remove the pre.1 tgz files.

Unity will then popup this warning:

![](/assets/2026-01-04-MRTK3-apps-on-Apple-Vision-Pro--fixing-the-stuck-to-your-head-issue-(and-more)/warningsig.png)

This you can safely ignore.

## Fix nothing visible

The app can now be deployed on the Vision Pro, but when yourun it, you will most likely see nothing. That is because they are created 1.6m above your head. To fix this, you will need to change the Y value of the Camera Offset in the MRTK XR Rig to 0.

![](/assets/2026-01-04-MRTK3-apps-on-Apple-Vision-Pro--fixing-the-stuck-to-your-head-issue-(and-more)/yoffset.png)

This is normally not a problem, but apparently it is now. If it's Unity does not like this, the Vision Pro packages, or something else, I don't know.

## Fix view being stuck to camera

If you now run the app, the cubes will appear all right, but if you move your head, all the cubes move with your head instead of hanging in space after the initial spawn, as displayed on the gif a the start of this post. This is one of the most peculiar changes we need to make:

![](/assets/2026-01-04-MRTK3-apps-on-Apple-Vision-Pro--fixing-the-stuck-to-your-head-issue-(and-more)/trackedposedriver.png)

As you can see, I have disabled the original Tracked Pose Driver and added a second one. This one has some hard-coded Action definitions in stead of an Action *References*. I got this wisdom by studying the [Vision OS template v 3.0.2](https://drive.google.com/drive/folders/1Oe-6bBCCmk7okbK832HWiYFbM8mV0XrZ). Why I didn't use Action References, and made a new Input Actions asset, like the default MRTK Default Input Actions, tailored for Vision Pro? Believe me, *I tried*, but either I don't understand those Input Action assets correctly, or it simply doesn't work in the Unity 6.3/Vision Pro combo. However, *this* works. But this leads to a new problem.

## Fix cubes appearing on floor

The cubes are now no longer stuck to your head, but instead of appearing in the view, they appear on the *floor* and even partially *in* the floor, with only one or two rows visible. The rest is invisible due to occlusion being applied to the Spatial Map. This issue, although seemingly very simple, took me quite some time to figure out. My best guess is: whatever layer Unity is putting on top of the Apple stuff - it apparently needs some time to initialize and figure out where the headset actually is before it properly sets the main camera's transform position.

The simplest way to fix that was to change my own startup code in `CubeManager` from:

```csharp
private void Start()
{
    audioSource = GetComponent<AudioSource>();
    CreateGrid();
}
```

to

```csharp
private async Task Start()
{
    audioSource = GetComponent<AudioSource>();
    await Task.Delay(1000);
    CreateGrid();
}
```

Simply wait a second till the Unity Player get it's act together. I don't know if this is optimal, but for my demo app, it does the trick.

## Fix hand menu

Now the only thing missing is the hand menu - that does not appear when you hold up your hand. The root cause is still the same: Vision Pro apparently does not track your *palm*, and that's what the hand menu (and other things in the MRTK) apparently rely on. So I went back to the trick [Guillaume Vauclin from EADS, France](https://www.linkedin.com/in/guillaume-vauclin-51592021/) created - adapt the code of the VisionOSHandProvider.cs file so that it no longer reports the palm cannot be tracked, but returns the wrist `Pose` instead. There is only one thing - I could not get Unity to accept a hacked version of the Apple Vision OS XR Plugin with a changed file inside it. So I went back to an old trick [I used in November 2022 for patching the MRTK2](https://localjoost.github.io/Fixing-hand-models-spawning-when-hand-tracking-is-lost-in-MRTK-28x/) - replace the file in the Library folder after Unity has unpacked all the library files.

You will find the fixed VisionOSHandProvider.cs in the Patches folder, together with a small shell script I created (actually I asked Claude to write it) that simply takes all C# files in the current folder, finds the location of each correspondingey named file in the Library folder, and replaces it:

```bash
#!/bin/bash

# Get the current directory
CURRENT_DIR="$(pwd)"
LIBRARY_DIR="$(pwd)/../Library"

# Check if Library directory exists
if [ ! -d "$LIBRARY_DIR" ]; then
    echo "Error: $LIBRARY_DIR directory not found"
    exit 1
fi

# Find all C# files in current directory
for cs_file in *.cs; do
    # Skip if no .cs files found
    if [ ! -f "$cs_file" ]; then
        echo "No C# files found in current directory"
        exit 0
    fi
    
    echo "Processing: $cs_file"
    
    # Find all matching files in Library and subdirectories
    while IFS= read -r target_file; do
        echo "  Replacing: $target_file"
        cp "$cs_file" "$target_file"
    done < <(find "$LIBRARY_DIR" -type f -name "$cs_file")
done

echo "Done!"
```

Make sure the project is opened by Unity at least once, wait until it is fully done loading and installing all packages. Then open the Patches folder in Terminal, and run the following commands:

```bash
chmod +x replace_files.sh
./replace_files.sh
```

The first command you will need to run only the first time. Also, remember to run this command in your CI build scripts. In my previous jobs, I wrote CI scripts that did the following steps when a patch like this was needed:
* Pull code
* Open in Unity
    * basically do nothing
    * quit
* Run patch script
* Open in Unity again
    * run tests
    * perform actual build
    * etc

## Concluding words

It's remarkable how flexible and malleable MRTK3 is - I have got it to run on all Mixed Reality headsets I managed to get my hands on, with minor tweaks. This makes for a very consistent experience and retains as much of your investments in Mixed Reality as possible. Which is an important thing, given the HoloLens 2 deprecation.

Updated QuestBouncer project for Apple Vision Pro can [be found here](https://github.com/LocalJoost/QuestBouncer/tree/AppleVisionPro_Unity63).