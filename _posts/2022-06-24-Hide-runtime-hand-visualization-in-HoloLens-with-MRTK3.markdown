---
layout: post
title: Hide runtime hand visualization in HoloLens with MRTK3
date: 2022-06-24T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- HoloLens2
- Unity
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2022-06-24-Hide-runtime-hand-visualization-in-HoloLens-with-MRTK3/visualizer.png
comment_issue_id: 422
---
A rather simple thingy this time, but one that was slightly irksome while I was playing with the MRTK3 private preview. As you may have seen in the 'video' [in my previous post](https://localjoost.github.io/MRTK2-to-MRTK3-intercepting-a-raw-air-tap-like-with-IMixedRealityPointerHandler/), by default MRTK3 shows a hand visualization. This is of course very handy while simulating hand gestures in the Unity editor, but not always desirable run time - where users can actually see their own real hands.

Poking a bit around in the MRTK XR Rig prefab, I quickly found this:

![](/assets/2022-06-24-Hide-runtime-hand-visualization-in-HoloLens-with-MRTK3/visualizer.png)

The Hand Visualizer. If you disable that script, the hand won't show up in the HoloLens anymore - but unfortunately, it also does not show up in the Unity editor - which makes testing in said editor a bit difficult. But that, of course, is easily fixed by this script:

```csharp
public class HandVisualizationController : MonoBehaviour
{
    [EnumFlags]
    [SerializeField]
    private VisualizationEnvironment environment =
               VisualizationEnvironment.Editor;
    
    private void Start()
    {
        var hands = GetComponentsInChildren<HandVisualizer>();
#if UNITY_EDITOR
        SetHandDisplayForEnvironment(hands, 
          VisualizationEnvironment.Editor);
#else
        SetHandDisplayForEnvironment(hands, 
          VisualizationEnvironment.RunTime);
#endif
    }
    
    private void SetHandDisplayForEnvironment(HandVisualizer[] hands, 
        VisualizationEnvironment requestedEnvironment)
    {
        for(var i = 0; i < hands.Length; i++)
        {
            hands[i].enabled = (environment & requestedEnvironment) != 0;
        }
    }
}
```

This you simply put on top of the MRTK XR Rig prefab:

![](/assets/2022-06-24-Hide-runtime-hand-visualization-in-HoloLens-with-MRTK3/location.png)

and you are done. In the default configuration, hands visualization will show up in the editor, but not in the HoloLens. The environments in which the hands can show up are defined in this accompanying enum:

```csharp
[Flags]
public enum VisualizationEnvironment
{
    Editor = 1 << 1,
    RunTime = 1 << 2,
}
```
Which allows you to change the selected environments to your liking:

![](/assets/2022-06-24-Hide-runtime-hand-visualization-in-HoloLens-with-MRTK3/enum.png)

## Conclusion

It's not always rocket science that I post, and I always say that people who want to start blogging but don't know what about: just start somewhere. Write what you learned. Some desperate fellow developer might just need the little thing you just discovered or made. ;). And if not, then at least you have an easy reference for your future self. 

Anyway, I was too lazy to set up a complete new project for this, so I incorporated this into the MRTKAirTap project I used last time, [only in a different branch](https://github.com/LocalJoost/MRKTAirTap/tree/handvisualizationcontroller).