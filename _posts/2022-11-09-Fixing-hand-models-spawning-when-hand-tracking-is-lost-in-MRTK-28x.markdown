---
layout: post
title: Fixing hand models spawning when hand tracking is lost in MRTK 2.8.x
date: 2022-11-09T00:00:00.0000000+01:00
categories: []
tags:
- MRTK2.8
- HoloLens2
- Unity
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2022-11-09-Fixing-hand-models-spawning-when-hand-tracking-is-lost-in-MRTK-28x/hand.gif
comment_issue_id: 432
---
Wait, what? MRTK 2.x? Yeah, sometimes older apps need to be maintained, you know. Recently I upgraded an app to a newer Unity version and MRTK 2.8.2, and noticed that sometimes the hand models I previously only saw in the editor sometimes flickered close before my face. After a bit of research I found out this happened at 0,0,0 - that is the place where you head was when the app started - and when the hand tracking was just about lost. And in fact, as one of my colleagues discovered, it was a [known issue in MRTK 2.8.0 reported in June 2022.](https://github.com/microsoft/MixedRealityToolkit-hereUnity/issues/10646) 

![](/assets/2022-11-09-Fixing-hand-models-spawning-when-hand-tracking-is-lost-in-MRTK-28x/hand.gif)

It was [reported fixed here](https://github.com/microsoft/MixedRealityToolkit-Unity/issues/10646), then [unfixed again here](https://github.com/microsoft/MixedRealityToolkit-Unity/pull/10831) because "At this point in MRTK2's lifecycle, we're trying to avoid breaking changes, especially to interfaces that might be implemented in any number of other projects". Apparently the fix included undesirable breaking changes.

In the mean time, if you have a HoloLens applications that still runs on MRTK 2.x and you need to use hand mesh display - you have either the choice of using MRTK2.7.x, live with this error - or apply the brute-force fix I created.

## The Culprit

As the original GitHub issue mentions, the RiggedHandMesh appears erratically in the HoloLens. This is, incidentally, exactly the same object appearing in the Unity editor when you use simulated hands. It sports a script RiggedHandsMesh.cs, which features the following method:

```csharp
protected override bool UpdateHandJoints()
{
    using (UpdateHandJointsPerfMarker.Auto())
    {
        // The base class takes care of updating all of the joint data
        _ = base.UpdateHandJoints();

        // Exit early and disable the rigged hand model if we've gotten a hand mesh from the underlying platform
        if (ReceivingPlatformHandMesh || MixedRealityHand.IsNull())
        {
            HandRenderer.enabled = false;
            return false;
        }

        IMixedRealityInputSystem inputSystem = CoreServices.InputSystem;
        MixedRealityHandTrackingProfile handTrackingProfile = inputSystem?.InputSystemProfile != null ? inputSystem.InputSystemProfile.HandTrackingProfile : null;

        // Only runs if render hand mesh is true
        bool renderHandmesh = handTrackingProfile != null && handTrackingProfile.EnableHandMeshVisualization && MixedRealityHand.TryGetJoint(TrackedHandJoint.Palm, out _);
        HandRenderer.enabled = renderHandmesh;
        if (renderHandmesh)
        {
```

This code does the following: when the underlying platform gives a hand mesh, the display is handled elsewhere, so exit out with false. But when it does *not* give a hand mesh, show the rigged hand mesh and try to position it (the rest of the method, not displayed here, is all about positioning the hand). I assume this is meant for cross-platform support. Unfortunately, on HoloLens, not getting a hand mesh most likely means hand tracking is *lost*, no position can be found, and the result is that the hand is *not positioned*. But the mesh is already displayed *before*  that is ascertained. Result: rigged hand mesh appears on 0,0,0. This is highly annoying and distracting, especially when the hands flash right before your face.

## Brute force solution
My first thought was to dig in all the if-then-else constructions in the code below what I showed above, but then it occurred to me - I can never test if I don't break anything on other platforms, and in any case: we don't need all this on HoloLens, or at least *I* don't need it on HoloLens. So I made a very small change:

```csharp
protected override bool UpdateHandJoints()
{
    using (UpdateHandJointsPerfMarker.Auto())
    {
        // The base class takes care of updating all of the joint data
        _ = base.UpdateHandJoints();

        // Exit early and disable the rigged hand model if we've gotten a hand mesh from the underlying platform
        if (ReceivingPlatformHandMesh || MixedRealityHand.IsNull())
        {
            HandRenderer.enabled = false;
            return false;
        }
        
#if WINDOWS_UWP
        // We don't need the rest in HoloLens
        return false;
#endif        
```

Basically: when we are on HoloLens, forget about the showing and juggling with hand meshes. If the HoloLens itself can't do it, forget about the rest. On other platforms, it still works as it did before. However that is supposed to be. ;)

## Patching the MRTK

MRTK packages are delivered as compressed .tgz files, which makes distributing easy, but patching a bit tricky. Now Unity decompresses these files in your Library/PackageCache folder, where they - unfortunately - get random postfixes, like com.microsoft.mixedreality.toolkit.foundation@a56f93c7e9ce-1666612515153. These postfixes differ from project to project and computer to computer. I have no idea what what determines the postfix, but I do know it is not possible to know *in advance* where Unity will put the files in the end. So help me Hopper, I wrote a *PowerShell* script to get it in the right place:

```powershell
$scriptpath = $MyInvocation.MyCommand.Path | Split-Path
$filepath = $scriptpath + "/../Library/PackageCache";

$fileToPatch = Get-Childitem -Path $filepath -Recurse RiggedHandVisualizer.cs;
$patchFile = $scriptpath + "/RiggedHandVisualizer.cs"

Copy-Item $patchFile $fileToPatch.fullname -force  
echo "Replaced " + $fileToPatch.fullname + " by "  $patchFile;

pause
```

This will basically find where the file is unpacked, then copy my patch over it. I assume PowerShell experts will have a laughing fit over my primitive and verbose attempt. It's also horribly inefficient as it searches the whole PackageCache folder, but hey - it only needs to run very occasionally, it *works*, and it solves the problem.

## Procedure

It's important to follow the right procedure:
* Open the project in Unity at least once after loading the MRTK2.8.x packages into your project with the MRTK Feature tool
* Close the Unity project
* Apply the patch by running the PowerShell script. 
* Open Unity again and go work on your project

I have prepared a zip file with both the patched file and the script, [you can download it here.](https://www.schaikweb.net/dotnetbyexample/MRTK280patch.zip). Copy it to the root of your project, unzip it there (it will create a MRTK280Patch folder), then run the PowerShell script and you are good to go.

## Concluding thoughts

First of all, remember to re-run the patch if you upgrade, re-import the project, or open it on a different computer. And sometimes it apparently reverts automatically. The script makes re-patching easy.

In addition, if you are using some DevOps solution to automatically build your deliverables, like GitHub Actions (and you do, *right*?) you need to build this into your build script. This requires your build script *also* to run Unity twice on the same project for every build
* Open the project and do nothing, just quit when Unity is done
* Patch the MRTK
* Open the project with Unity and build the player.

Yes, it's inefficient, yes, it takes longer. You can also unzip the tgz file and patch in permanently, hoping you won't forget it in other projects, there will never be an MRTK2.8.3, or no-one will re-attach the MRTK for some reason. This works. Albeit inefficiently.
