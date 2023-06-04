---
layout: post
title: Using ARPlaneManager as an alternative to Scene Understanding with MRTK3 on HoloLens 2
date: 2023-06-04T00:00:00.0000000+02:00
categories: []
tags:
- Unity
- Mixed Reality
- HoloLens2
- MRTK3
- Plane Finding
- Scene Understanding
featuredImageUrl: https://LocalJoost.github.io/assets/2023-06-04-Using-ARPlaneManager-as-an-alternative-to-Scene-Understanding-with-MRTK3-on-HoloLens-2/planetracking.gif
comment_issue_id: 445
---
Recently someone asked how Scene Understanding could be done with MRTK3 - the samples don't work anymore as they are targeted towards MRTK2. The short answer is - it cannot be done. This is now covered by Unity itself - using the an ARPlaneManager. I made a little demo that tracks both horizontal and vertical planes in an MRTK3 HoloLens application using just that.

![](/assets/2023-06-04-Using-ARPlaneManager-as-an-alternative-to-Scene-Understanding-with-MRTK3-on-HoloLens-2/planetracking.gif)

## Creating the basic app
I have created a normal 3D core project using Unity 2021.3.16f1. In the current sources it shows the MRTK team are using 2021.3.21f1 - *do not use that*. It causes your app to crash on startup. It took me the better part of a Sunday to find out what the problem was. 2021.3.16f1 works fine.

Then I added:
* Basically everything that is listed under MRTK right now (pre.16 versions ATM)
* OpenXR plugin 1.7.0 (as this is also used in the MRTK sources)
* Microsoft Spatializer 1.0.246.

Then we get the whole rigamarole of configuring OpenXR, althought *most* of that goes automatic. To finish off the process, don't forget to remove the main camera from the scene and replace it by an **MRTK XR Rig**.

## ARManagers and Planes

The process is actually fairly well [described in the Unity documentation](https://docs.unity3d.com/Packages/com.unity.xr.arfoundation@5.1/manual/features/plane-detection.html) but as I noticed so often - the tiniest bit of working code brings more insight than a thousand pages of API explanation.

### Create a plane

You need to create an AR plane object by right-clicking the hierarchy, then choose "XR/Create Default Plane"  

What I did next was:
* Rename "Default AR Plane to "ARHorizontalPlane"
* Change the Mesh renderer's material to "MRTK_Standard_TransparentOrange"
* Created a layer HorizontalPlanes and added this object to it (optional)
* Set the "Detection mode" to Horizontal.

### Create AR Plane Manager

I created an empty gameobject, called it "ARPlaneManager". Then  
* I added an "AR Plane Manager" component to the ARPlaneManager object. This automatically adds an AR origin too.
*  I dragged the MRTK XR Rig's camera into the XR origin's "Camera GameObject" field
* I dragged the ARHorizontalPlane into the "Plane Prefab" field.

![](/assets/2023-06-04-Using-ARPlaneManager-as-an-alternative-to-Scene-Understanding-with-MRTK3-on-HoloLens-2/camera.png)

## Add an AR Session

This actually took me the most time, as I apparently I missed that little part was neccesary to make it work at all. The step is simple: pick a game object, and add an AR Session script to it. I usually make a simple prefab with its name, which makes it easily visible in the hierarchy.

![](/assets/2023-06-04-Using-ARPlaneManager-as-an-alternative-to-Scene-Understanding-with-MRTK3-on-HoloLens-2/arsession.png)

## About the demo project

As you can see in the movie above, it either shows orange transparent horizontal planes, or green transparent vertical planes. It's a bit odd, but *apparently* there can be only one ARPlaneManager in the scene. If you set it to horizontal, you can't add a second one that adds vertical planes. All you get to see are horizontal planes - green and orange. So I created a script that alternates between these two. An ARPlaneManager can track both, but it is right off the bat not able show them visually different, or make them appear in different layers. If you need to separately detect horizontal and vertical planes you would need to write some code yourself. This makes [sorting horizontal and vertical planes into separate layers - like I wrote about before](https://localjoost.github.io/Making-Surface-Magnetism-work-on-specific-surfaces-by-sorting-Scene-Understanding-surfaces-into-separate-layers/) - a bit more tricky. I need to dive deeper into this.

Another thing to keep in mind is Plane Finding is considerably more limited than Scene Understanding. I finds horizontal and/or vertical planes. Period. I does not tell you a horizontal plane is either a floor, a ceiling or a platform - it's a horizontal plane, and good luck with it. 

It does come with a very big benefit: *it is completely cross platform* , I have used in HoloATC, that runs on Hololens, Android ARCore phones and even on Magic Leap. It presumably even runs on ARKit, although, lacking Apple hardware, I have not been able to verify that - I do seem to recall a community member saying something to that effect to me. 

You can [download the demo project here](https://github.com/LocalJoost/MRTK3PlaneFinding.git).
