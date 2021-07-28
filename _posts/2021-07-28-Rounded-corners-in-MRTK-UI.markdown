---
layout: post
title: Rounded corners in MRTK UI
date: 2021-07-28T00:00:00.0000000+02:00
categories: []
tags:
- HoloLens2
- MRKT2
- Unity3d
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2021-07-28-Rounded-corners-in-MRTK-UI/rounded2.png
---
Rounded corners are suddenly hip again and all the rage, even in Microsoft land, as is clearly visible in [Windows 11.](https://blogs.windows.com/windowsexperience/2021/06/24/introducing-windows-11/)

![](/assets/2021-07-28-Rounded-corners-in-MRTK-UI/Windows11.png) 

Unfortunately, HoloLens and the [Mixed Reality Toolkit](https://docs.microsoft.com/en-us/windows/mixed-reality/develop/unity/mrtk-getting-started) are *seemingly* lagging a bit behind as far as this fashion is concerned. A typical MRTK user interface looks like this:

![](/assets/2021-07-28-Rounded-corners-in-MRTK-UI/Windows11.jpg)

Square buttons galore. However, you you look very carefully, you will see neither the buttons nor the slate is actually completely square.

![](/assets/2021-07-28-Rounded-corners-in-MRTK-UI/corner.png)

What you want (or most likely, what your *designer* wants ;) ) is something like this:

![](/assets/2021-07-28-Rounded-corners-in-MRTK-UI/rounded1.png)

Including, of course, cool rounded 3D visual effects like these when hit boxes are activated.

![](/assets/2021-07-28-Rounded-corners-in-MRTK-UI/rounded2.png)

... or pressed

![](/assets/2021-07-28-Rounded-corners-in-MRTK-UI/rounded3.png)

I am a great proponent of using COTS where ever possible - I would *not* like to build my own UI stack as the MRKT already offers so much, especially not for some minor visual effects. For the same reason I don't want to *fork* the MRTK because that will probably come back to haunt me when in some future the time comes to upgrade. It's best just to find out how we can fix this without too much work and too much things breaking -  staying as close to the original as possible. As the screen shots probably tell you, this is entirely doable.

## Analysis
To my great surprise and joy I found *this* in the Mixed Reality Toolkit Standard Shader that is used is almost all (if not all) MRKT materials, under Fluent options. 

![](/assets/2021-07-28-Rounded-corners-in-MRTK-UI/shader.png)

So what we need, is *already prepared* in the MRTK. This is the material "HolographicBackPlate", that is used for the slate and for the button background. If you move the slider "Unit radius" all the way to the right, it will move to 0.5 and you will immediately get this effect: 

![](/assets/2021-07-28-Rounded-corners-in-MRTK-UI/rounded1.png)

The problem with this: it does not 'stick'. You are basically adapting a material that is *part of and inside the MRTK*. And since that comes in a couple of .tgz packages file in Packages\MixedReality nothing about this material is actually in your source files. The material in question is unpacked in Library\PackageCache\com.microsoft.mixedreality.toolkit.foundation@a05dc7578209-1624820429212\SDK\Features\UX\Interactable\Materials. Everything in Library is stuff that is generated from source and packages - and this you typically *don't* commit to source control (you don't, do you? Please tell me you don't). If your co-worker pulls your solution from source control all the buttons and slates will be square again. And even if you were committing this to source control - if you later were to upgrade to a *newer* version of MRTK, you will loose your changes.

The best way to go about this, is making a copy of the material *outside* of the MRKT, adapt it, and *replace* it in the UI elements concerned. 

## Finding and adapting materials

There are actually three materials involved:
1. HolographicBackPlate 
2. HolographicButtonContentCageProximity
3. MRTK_PressableInteractablesButtonBox

In a button, they are used in these places:

![](/assets/2021-07-28-Rounded-corners-in-MRTK-UI/materiallocations.png)

The files live in various place in the MRKT, and are easy to find. Just, for instance, click the HighLightPlate in the hierarchy, then click the material in the Mesh plate's MeshRender, this will make it show up in the assets pane - right-click it, then select "Show in Explorer" and you have it selected in a File Explorer Window.

![](/assets/2021-07-28-Rounded-corners-in-MRTK-UI/FindMaterial.png)

Now of course you might go about and change the three materials in every UI element in your app. Apart from this being a lot of work, this might cause other problems if the material in the MRKT is updated, and your material starts to lag behind - then you will have to repeat the adapt and replace routine for all UI elements *again*. Also, I am a bit lazy, as you might know, so I tend to solve things like that with code.

I first copied the three materials from the MRTK into my Assets. I renamed the copies - they all got the prefix "Round".

![](/assets/2021-07-28-Rounded-corners-in-MRTK-UI/roundmaterials.png)

For all three materials I made sure Rounded Corners are selected, then I set Unit Radius all the way to the maximum. The it's time for code. Finally.

![](/assets/2021-07-28-Rounded-corners-in-MRTK-UI/roundedselected.png)

## One behaviour to change them all

The amount of code is surprisingly small:

```csharp
using System.Collections.Generic;
using System.Linq;
using UnityEngine;

public class MaterialAdapter : MonoBehaviour
{
    [SerializeField]
    private List<Material> replacementMaterials = new List<Material>();

    [SerializeField]
    private string replacementPrefix = "Round";
    
    void Start()
    {
        var replacementMaterialsDictionary = 
            replacementMaterials.ToDictionary(material => 
                material.name.Replace(replacementPrefix, string.Empty));

        var renderers = gameObject.GetComponentsInChildren<MeshRenderer>(true);
        foreach (var r in renderers)
        {
            if (r.sharedMaterial == null) continue;
            if (replacementMaterialsDictionary.ContainsKey(r.sharedMaterial.name))
            {
                r.sharedMaterial =   
                  replacementMaterialsDictionary[r.sharedMaterial.name];
            }
        }
    }
}
```
You are supposed to assign the replacement materials in the editor like this:

![](/assets/2021-07-28-Rounded-corners-in-MRTK-UI/behaviourconfig.png)

So the first line in Start actually creates a dictionary with the material names to replace as key, with the materials they should be replaced by as a value. This, of course, supposes the names of the replacement materials are identical to the original names, just having "Round" (or actually the value of `replacementPrefix`) as prefix. The next line finds all MeshRenderers, and the rest loops over them, checks if the MeshRenderer currently uses one of the materials, and if so, replaces it. It you put this behaviour on top of a menu...

![](/assets/2021-07-28-Rounded-corners-in-MRTK-UI/behaviouruse.png)

... it will swap all the materials as soon as the scene is loaded, and effectively make all corners rounded to the max.

## Why not an editor script?

It would be also possible to create an *editor* script for this that makes the change permanent. There are advantages to this, as well as disadvantages. It's easier to configure a behaviour, and make that configuration stick. Also, the stuff in the scene just stays stock MRTK. That is also it's main *dis*advantage: no design time view of the final result - and of course, a bit of a performance penalty at the start. I took what I thought to be an easy and quick fix, that does nothing permanent to MRKT components.

## Concluding words

It's nice to see the MRTK was actually already preparing for the Era of the Rounded Corners, and that is was so easy so make this adaption without large scale adaptions, using a different UI stack, or doing a lot of work. As any developer knows, laziness can be a virtue in our profession ;)

[Demo project can be found here.](https://github.com/LocalJoost/RoundedCorners)