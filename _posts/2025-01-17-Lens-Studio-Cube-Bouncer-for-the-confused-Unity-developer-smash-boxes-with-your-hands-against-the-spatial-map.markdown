---
layout: post
title: 'Lens Studio Cube Bouncer for the confused Unity developer: smash boxes with your hands against the spatial map'
date: 2025-01-17T00:00:00.0000000+01:00
categories: []
tags:
- Lens Studio
- Spectacles
- TypeScript
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2025-01-17-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-smash-boxes-with-your-hands-against-the-spatial-map/handsmasheradded.png
comment_issue_id: ''
---
In this blog, I take you, the confused Unity developer trying to understand Lens Studio, by the hand and introduce you to doing hand tracking, sphere casting, smashing things away, and how to let them bounce off the spatial map. This is a continuation of [my previous post](https://localjoost.github.io/Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-set-up,-create-box,-create-a-grid-of-boxes/). If you have not read that, I strongly suggest you do; otherwise, it will make little sense. At the end of this blog, the app will do this:

![bounce2](/assets/2025-01-17-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-smash-boxes-with-your-hands-against-the-spatial-map/bounce2.gif)

## Smashing virtual stuff with your hand

This is a 1:1 translation of my original [MRTK2 Hand Smash Service](https://localjoost.github.io/HandSmashService-an-MRTK2-extension-service-to-smash-holograms-at-high-speed/) that I created in January 2021 and recently re-implemented for [MRKT3 and the Reality Collective Service Framework](https://localjoost.github.io/CubeBouncer-revisited-setting-up-Mixed-Reality-for-Quest-3-with-MRTK3-and-Unity-6/). It's basically one script "HandSmasher.ts" that you can add to your scene; it will track your hands, do a sphere cast from your hand position, and apply an amount of force depending on the speed of your hand to any object it encounters while doing so.

![handsmasheradded](/assets/2025-01-17-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-smash-boxes-with-your-hands-against-the-spatial-map/handsmasheradded.png)

The principle is still the same, and I quote my past self:

"Assuming the hand will continue to travel in the same direction and with the same speed 

* I can perform a ray cast from its current position in the direction where the hand will be most likely next 
* If that ray cast hits an object, apply force to it proportionally to the speed of the hand."

I used this little image to explain back then:

![handtravel](/assets/2025-01-17-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-smash-boxes-with-your-hands-against-the-spatial-map/handtravel.png)

## Smash script

### Imports
We start, as often, with a number of imports. We once again use a bunch of stuff from the SIK. Mind the relative path thing I warned for already [in my previous blog](https://localjoost.github.io/Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-set-up,-create-box,-create-a-grid-of-boxes/#deconstructing-cubemanager):

```typescript
import { VectorUtils } from "./VectorUtils";
import { HandInputData } from "../../SpectaclesInteractionKit/Providers/HandInputData/HandInputData";
import { HandType } from "../../SpectaclesInteractionKit/Providers/HandInputData/HandType";
import TrackedHand from "../../SpectaclesInteractionKit/Providers/HandInputData/TrackedHand"
```

### Component setup with members (and already some initialization)

Then the actual component definition, with some input parameters (= SerializeField) that you can set for from the editor

```typescript
@component
export class HandSmasher extends BaseScriptComponent {
    @input forceMultiplier: number = 100;
    @input smashAreaSize: number = 10;
    @input projectionDistanceMultiplier: number = 2;
```

These parameters mean:
* forceMultiplier: multiplication of the force applied by the vector. Default I use is 100, so you get a quite nice fast effect. 
* smashAreaSize: the radius of the sphere cast being used. A 10cm radius is a 20cm diameter, a bit bigger even than my hand, but at least it makes hitting stuff easier
* projectionDistanceMultiplier - tweaks the distance of the sphere cast. The current value of 2 makes it twice as long, so if your hand has traveled 10 cm in a frame, it will actually make a sphere cast of 20 cm. Playing with this factor allows you to tackle the situation where an object is just a few millimeters from the projected future position of the hand - which might result in swatting right through the object in the next frame.

The next piece of code already shows some interesting new things:

```typescript
private handProvider: HandInputData = HandInputData.getInstance()
private leftHand = this.handProvider.getHand("left" as HandType);
private rightHand = this.handProvider.getHand("right" as HandType);
private probe = Physics.createGlobalProbe();

private previousLeftPosition: vec3;
private previousRightPosition: vec3;
```

* The first line gets a reference to the hand tracker. Why it must be done this way, no idea.
* The second and third lines get references to the actual tracked hands. Notice the use of hardcoded strings. Using these instead of things like constants or enums are a recipe for pain, but apparently, This Is The Way.
* The `probe` is an object we can use to make ray, sphere, and other casts.

Note: you can already can use non-static members and methods during initialization, this is not possible in C#, so you can do declaration and initialization in one go. As for `previousLeftPosition` and `previousRightPosition`: we will need those for tracking the hand movement through time.

### Lifecycle event or how to update

And now we get to another weirdness of Lens Studio scripting. In Unity, you are used to implementing methods like Start, OnEnable, Update, OnDisable, etc., for hooking into lifecycle events. In Lens Studio, things work a little differently: although there are lifecycle events, you have to explicitly hook up an event to be able to use that. The only exception is the onAwake method, which is always called automatically.

```typescript
onAwake() {
    this.createEvent("UpdateEvent").bind(() => {
        this.onUpdate();
    })
}

onUpdate() {
    this.previousLeftPosition = this.applySmashMovement(this.leftHand, this.previousLeftPosition);
    this.previousRightPosition = this.applySmashMovement(this.rightHand, this.previousRightPosition);
}
```

But without the explicit hook up of `onUpdate` with the `createEvent` call in `onAwake`, `onUpdate` will never get called. This also means you can hook up multiple methods to the UpdateEvent and make an interesting hard-to-debug mess, especially since you can't set breakpoints, as I mentioned before.

### The actual smashing bit, part 1

```typescript
private applySmashMovement(hand: TrackedHand, previousPosition: vec3): vec3 {
    const currentPosition = hand.indexKnuckle.position;
    this.tryApplyForceFromVectors(previousPosition, currentPosition);
    return currentPosition != null ? currentPosition : previousPosition
}
```

Pretty simple: it retrieves the knuckle of the index finger (I seem to recall having read somewhere that was the best tracked point, but I don't remember where), then tries to "apply force" from the previous location to the current location

### The actual smashing bit, part 2: the return of the sphere cast

```typescript
private tryApplyForceFromVectors(previousPosition: vec3, currentPosition: vec3) {
    if (previousPosition == null || currentPosition == null) {
        return;
    }
    const handVector = currentPosition.sub(previousPosition);
    this.probe.sphereCastAll(this.smashAreaSize, currentPosition,
        currentPosition.add(handVector.mult(VectorUtils.scalar3(this.projectionDistanceMultiplier))),
        (hits: RayCastHit[]) => {
            for (const hit of hits) {
                const objectHit = hit.collider.getSceneObject();
                const bodyComponent = objectHit.getComponent("Physics.BodyComponent") as BodyComponent
                if (bodyComponent != null) {
                    const force = handVector.mult(
                      VectorUtils.scalar3(handVector.length * this.forceMultiplier));
                    bodyComponent.addForce(force, Physics.ForceMode.Impulse);
                }
            }
        });
}
```

We calculate the direction the hand has traveled in, then do a sphere cast using the `probe` object we created earlier from the current position to where we think the next position will be based upon the hand travel direction. In the callback, we get the scene objects that were hit, try to get a reference to its `BodyComponent`, and if that exists, we add force. And the cube moves.

Notice: I used the "Physics.BodyComponent" as a stringto get a reference to the `BodyComponent`. You might wonder why didn't I use the nice `getTypeName()` method, like in the `CubeManager`, to prevent the use of a hardcoded string

```typescript
var cubeController = clone.getComponent(CubeController.getTypeName()) as CubeController;
```

The answer is - well, I tried, and I got this:

![error](/assets/2025-01-17-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-smash-boxes-with-your-hands-against-the-spatial-map/error.png)

Apparently, this method is only available for some objects, and not all objects. Which is quite annoying, but that's the way it is.

## Adding a spatial map the cubes can bounce off

If you deploy this to your Spectacles, you can now actually smash the cubes away, they bounce off each other but slowly continue to disappear into the blue yonder, not being bothered by walls, floors, or anything else for what matters. This is of course not fun in a Mixed Reality app. How do you get a Spatial Map with a collider and occlusion, just like you were used to doing in HoloLens or Magic Leap 2? Well, there's an easy answer for that, and a slightly more complicated one.

### The easy answer

In my [demo project](https://github.com/LocalJoost/CubeBouncerDemoLens/tree/blog2), there's a folder "Occluded Collider World Mesh" 

![assetbrowser](/assets/2025-01-17-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-smash-boxes-with-your-hands-against-the-spatial-map/assetbrowser.png)

Copy that folder over to your own project using File Explorer, drag the prefab "Occluded Collider World Mesh" somewhere in your scene, for example on root level

![projectocclusion](/assets/2025-01-17-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-smash-boxes-with-your-hands-against-the-spatial-map/projectocclusion.png)

...aaand you are done. You have a spatial map with a collider and occlusion, so things will bounce off real stuff and appear behind real stuff. Just like in HoloLens and Magic Leap 2. 

### The slightly harder answer

If you want to do it the hard way:
* In the Asset Browser, right-click, then "Create Asset/Meshes/World Mesh"
* In the Scene Hierarchy, right-click, then "Create Scene Object"
* To that Scene Object, in the Inspector, add a "Render Mesh Visual" and a Physics Body
* In the Render Mesh Visual, choose the World Mesh you just created as Mesh:

![rendermesh](/assets/2025-01-17-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-smash-boxes-with-your-hands-against-the-spatial-map/rendermesh.png)

* In the Physics Body, Select type "Mesh" from the dropdown, and also select the World Mesh as Mesh

![physicsbody](/assets/2025-01-17-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-smash-boxes-with-your-hands-against-the-spatial-map/physicsbody.png)

Now the only thing we miss is the occluder material and occluder shader. I'll be honest with you: that I nicked myself, from a Snap Package. Or Asset bundle, or whatever they call it. Simply click the Asset Library top left, search for "World Mesh"

![package](/assets/2025-01-17-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-smash-boxes-with-your-hands-against-the-spatial-map/package.png)

Hover your mouse over "Spawn Object at World Mesh" (the right one) and click the import button that appears

And if you search for occluder, you will now find these items:

![occluder](/assets/2025-01-17-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-smash-boxes-with-your-hands-against-the-spatial-map/occluder.png)

I tend to do these kinds of things, where I need only a tiny part of a package, in a throwaway project, then only copy over the stuff I need using File Explorer. In this case we only the material and its shader. Don't forget to include the .meta file as well - although I guess you, coming from a Unity background, probably already did that without even thinking.

And then you only have to assign the occluder material to the Render Mesh Visual by clicking the "+ Choose Material" button.

So you see - it's simply a prefab rendering some mesh apparently created by the Spectacles, we put a PhysicsBody on it which also creates a collider - apparently, and then we only have to add an "occluder" material. Kind of like the ARMeshManager in Unity, but a bit simpler.

## Conclusion

The further you venture into Lens Studio, the more things start to make sense, and by now you are - I hope - a considerably less confused Unity developer. Although I will admit some things are different and downright weird, and there are things that make you go "what the [bleep]" were they thinking here" (and there's more to come). But you have to keep in mind they came into this from a totally different angle - not a game engine, but a project-funny-pictures-on-your-face angle. Also, I thought "what the [bleep]" were they thinking here" of Unity as well, a lot of times, with pretty long and pretty NSFW bleeps sometimes ;)

Anyway, you can [find the app so far on GitHub, branch blog2](https://github.com/LocalJoost/CubeBouncerDemoLens/tree/blog2). And as of January 15, [even us Dutchies can now get their own Spectacles](https://www.linkedin.com/feed/update/urn:li:activity:7285207073137557505/) without having to resort to nefarious ways so if you want to try, you know where to get your Shiny New Toy ;)