---
layout: post
title: Using ARMeshManager for Spatial Awareness with MRTK3 on HoloLens 2
date: 2023-06-05T00:00:00.0000000+02:00
categories: []
tags:
- Unity
- Mixed Reality
- HoloLens2
- MRTK3
- Spatial Mesh
- Spatial Awareness
featuredImageUrl: https://LocalJoost.github.io/assets/2023-06-05-Using-ARMeshManager-for-Spatial-Awareness-with-MRTK3-on-HoloLens-2/spatialmeshconfig.png
comment_issue_id: 446
---
The transition from MRTK2 to MRTK3 leaves some people confused, especially less experienced developers, as a lot of GoogleBinging for "how to do things" leads to MRTK2 documentation that is not appliccable anymore to MRTK3. Recently someone asked me how Spatial Awareness worked, after discovering [the MRTK2 samples](https://learn.microsoft.com/en-us/windows/mixed-reality/mrtk-unity/mrtk2/features/spatial-awareness/spatial-awareness-getting-started?view=mrtkunity-2022-05) don't work anymore. Now there is [a sample in the MRTK3 repo](https://github.com/microsoft/MixedRealityToolkit-Unity/blob/mrtk3/UnityProjects/MRTKDevTemplate/Assets/Scenes/SpatialMappingExample.unity), but apparently that is not very easy to find. Hence this little sample, that basically comes down to this one picture:

![](/assets/2023-06-05-Using-ARMeshManager-for-Spatial-Awareness-with-MRTK3-on-HoloLens-2/spatialmeshconfig.png)

* Expand your MRTK XR Rig to the Camera Offset
* Add an empty gameobject "ARMeshManager" to it
* Drag an ARMeshManager script on it
* Drag a mesh prefab to the "Mesh Prefab" field. In the sample, I use the well-know wireframe material, leading to this view in the HoloLens:

![](/assets/2023-06-05-Using-ARMeshManager-for-Spatial-Awareness-with-MRTK3-on-HoloLens-2/wireframe.png)

And you are done. The Spatial Mapping prefab I have nicked from the MRTK3 sample, and created three variants of it, showing either occlusion, wireframe, or a transparent spatial map. I also - and I recommend you do that too - added Layer 31 as "Spatial Mesh" to the project and made sure the spatial mapping prefabs are in that as well.

![](/assets/2023-06-05-Using-ARMeshManager-for-Spatial-Awareness-with-MRTK3-on-HoloLens-2/layerconfig.png)

Ready-to-steal [demo project here](https://github.com/LocalJoost/MRTK3SpatialMesh).
