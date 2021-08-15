---
layout: post
title: Showing water in HoloLens 2 using the (deprecated) standard assets - in BOTH eyes
date: 2021-08-14T00:00:00.0000000+02:00
categories: []
tags:
- HoloLens2
- MRTK2
- Unity3d
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2021-08-14-Showing-water-in-HoloLens-2-using-the-(deprecated)-standard-assets--in-BOTH-eyes/water.png
comment_issue_id: 386
---
A pretty short tip this time. Recently I had the requirement to show water (that is, a fluid) in a HoloLens 2 app. Something like this:

![](/assets/2021-08-14-Showing-water-in-HoloLens-2-using-the-(deprecated)-standard-assets--in-BOTH-eyes/water.png)

Fortunately, from the Assets store, you can download the Unity Standard Assets, and one of those include water. 

![](/assets/2021-08-14-Showing-water-in-HoloLens-2-using-the-(deprecated)-standard-assets--in-BOTH-eyes/standardassets.png)

This asset pack comes with a warning it is no longer maintained, but it can't hurt to try, right? So I imported the standard assets, choosing only the water part.

![](/assets/2021-08-14-Showing-water-in-HoloLens-2-using-the-(deprecated)-standard-assets--in-BOTH-eyes/onlywater.png)

Then I found the prefab "WaterProDaytime" and put this in the scene. I tweaked one setting of it's "Water" script, the water mode

![](/assets/2021-08-14-Showing-water-in-HoloLens-2-using-the-(deprecated)-standard-assets--in-BOTH-eyes/watermode.png)

and tweaked some settings in the transform to make the puddle about 1m in diameter, and appear 1.5 meter below your viewpoint, about 3 meters away. To my surprise it works great. A nice rippling water surface before you. Although... there was something funny with the way things showed up. I could not really put my finger on it, but it looked like it was neatly stuck to the floor, and it was somewhat tiring to my eyes to watch 'water' in an actual HoloLens 2. Only when I came closer I noticed what was wrong - when I turned my head to look a bit left of the water, the whole puddle flashed out of existence, while it was still clearly in the FOV. It turned out it was only rendered in the *left* HoloLens 2 'screen', not in the right. If I closed my left eye, the puddle was not visible. 

That explained why it was a bit tiring to look at the puddle - my brain had clearly trouble making sense of something that should be visible in both eyes, but was not. This clearly would not do in a production environment.

I discussed this with a colleague and he mentioned something about Single Pass rendering, and this made me [stumble on this page.](https://docs.unity3d.com/Manual/SinglePassStereoRenderingHoloLens.html).

## So what is going on?
The problem is that MRTK standard setup sets the Stereo rendering method for Windows Holographic to Single Pass Instanced. 

![](/assets/2021-08-14-Showing-water-in-HoloLens-2-using-the-(deprecated)-standard-assets--in-BOTH-eyes/singlepass.png)

While this is an excellent idea from a performance point of view, it actually disables the stereo rendering of the water puddle. The simplest solution is to set the Stereo Rendering Mode to multi pass, but this doubles the CPU work load. This might - and most likely will - have a negative effect on your overall performance, just for this one puddle. 

The other option is to adapt the shader

![](/assets/2021-08-14-Showing-water-in-HoloLens-2-using-the-(deprecated)-standard-assets--in-BOTH-eyes/herebedragons.png)

## Adapting the shader
I will shamelessly admit that I know next to nothing about shaders. They are the closest thing to black art I know - I can see what they do, but not how or why, turning their knobs sometimes yield very unexpected result, and if they don't work or break I am a little less clueless than the avarage fashion influencer reading Assembler code. 

If we want to get somewhere, we need we need to make five changes [according to the Unity manual](https://docs.unity3d.com/Manual/SinglePassStereoRenderingHoloLens.html).

First we have to find a `struct appdata` and add `UNITY_INSTANCE_ID` to it. Easy enough

<div class="highlight"><pre class="highlight"><code>struct appdata<br/>
{<br/>
&nbsp;&nbsp;float4 vertex : POSITION;<br/>
&nbsp;&nbsp;float2 uv : TEXCOORD0;<br/>
&nbsp;&nbsp;<b>UNITY_INSTANCE_ID</b><br/>
};</code></pre></div>

Then we have to find `struct v2f` and add `UNITY_INSTANCE_ID` and `UNITY_VERTEX_OUTPUT_STEREO`. Well okay, the struct looks quite bit bigger than in the sample, but let's try it anyway:

<div class="highlight"><pre class="highlight"><code>struct v2f {<br/>
&nbsp;&nbsp;float4 pos : SV_POSITION;<br/>
&nbsp;&nbsp;#if defined(HAS_REFLECTION) || defined(HAS_REFRACTION)<br/>
&nbsp;&nbsp;&nbsp;&nbsp;float4 ref : TEXCOORD0;<br/>
&nbsp;&nbsp;&nbsp;&nbsp;float2 bumpuv0 : TEXCOORD1;<br/>
&nbsp;&nbsp;&nbsp;&nbsp;float2 bumpuv1 : TEXCOORD2;<br/>
&nbsp;&nbsp;&nbsp;&nbsp;float3 viewDir : TEXCOORD3;<br/>
&nbsp;&nbsp;#else<br/>
&nbsp;&nbsp;&nbsp;&nbsp;float2 bumpuv0 : TEXCOORD0;<br/>
&nbsp;&nbsp;&nbsp;&nbsp;float2 bumpuv1 : TEXCOORD1;<br/>
&nbsp;&nbsp;&nbsp;&nbsp;float3 viewDir : TEXCOORD2;<br/>
&nbsp;&nbsp;#endif<br/>
&nbsp;&nbsp;UNITY_FOG_COORDS(4)<br/>
<b>&nbsp;&nbsp;UNITY_INSTANCE_ID<br/>
&nbsp;&nbsp;UNITY_VERTEX_OUTPUT_STEREO</b><br/>
};</code></pre></div>

The third step involves adding three... well let's call it statements, for lack of a better word - to something that looks like a function, called `v2f vert(appdata v)`. I presume this is a function called `vert`, with a parameter `v` and a `v2f` return value.
<div class="highlight"><pre class="highlight">
<code>v2f vert(appdata v)<br/>
{<br/>
&nbsp;&nbsp;v2f o;<br/>
<br/>
<b>&nbsp;&nbsp;UNITY_SETUP_INSTANCE_ID(v);<br/>
&nbsp;&nbsp;UNITY_TRANSFER_INSTANCE_ID(v, o);<br/>
&nbsp;&nbsp;UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);<br/></b>
&nbsp;&nbsp;o.pos = UnityObjectToClipPos(v.vertex);<br/>
</code></pre></div>


Step 4 is apparently just adding a declaration 'somewhere'. I put it below the vert function
<div class="highlight"><pre class="highlight"><code><b>UNITY_DECLARE_SCREENSPACE_TEXTURE(_MainTex);</b></code></pre></div>

The final step instructs you to add `UNITY_SETUP_INSTANCE_ID(i);` to `fixed4 frag (v2f i) : SV_Target`. The problem is - there is no function(?) `fixed4 frag (v2f i) : SV_Target`. There is a `half4 frag( v2f i ) : SV_Target` though. I was like, heck, it's nearly the same let's try it anyway

<div class="highlight"><pre class="highlight"><code>half4 frag( v2f i ) : SV_Target<br/>
{<br/>
<b>&nbsp;&nbsp;UNITY_SETUP_INSTANCE_ID(i);</b><br/>
&nbsp;&nbsp;i.viewDir = normalize(i.viewDir);<br/>
&nbsp;&nbsp;<br/>
&nbsp;&nbsp;// combine two scrolling bumpmaps into one<br/>
&nbsp;&nbsp;half3 bump1 = UnpackNormal(tex2D( _BumpMap, i.bumpuv0 )).rgb;<br/>
&nbsp;&nbsp;half3 bump2 = UnpackNormal(tex2D( _BumpMap, i.bumpuv1 )).rgb;</code></pre></div>
When I saved this file, Unity apparently did not like `UNITY_INSTANCE_ID` because it auto-upgraded both instances to `UNITY_VERTEX_INPUT_INSTANCE_ID` and duly notified me with an upgrade comment on top of the shader

`// Upgrade NOTE: replaced 'UNITY_INSTANCE_ID' with 'UNITY_VERTEX_INPUT_INSTANCE_ID'`

## Cargo cult programming FTW
It's of course impossible to show in a picture that shows a single 2D rendering of something what should be viewed in 3D with two eyes, but to my great satisfaction the water now showed in *both* yes and the weird eye-straining effect had gone. It was like a kind of ['Darmok and Jalad at Tanagra'](https://en.wikipedia.org/wiki/Darmok) experience - I can see it's a C-like development language, I recognize some words and constructs, but I am not very sure what they actually mean and do. I also don't know if this is the best way to show water in HoloLens 2, but  the important thing is: in the end it worked, and unlike at [El-Adrel](https://memory-alpha.fandom.com/wiki/El-Adrel_system), no-one had to die for that ;) - a bit of pattern recognition and educated guesses were enough in this case.

You can download the [demo project](https://github.com/LocalJoost/HoloWater) with the already adapted shader [here](https://github.com/LocalJoost/HoloWater).
