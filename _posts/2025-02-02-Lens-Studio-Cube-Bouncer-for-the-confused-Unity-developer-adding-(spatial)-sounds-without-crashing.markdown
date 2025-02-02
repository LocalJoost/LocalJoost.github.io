---
layout: post
title: 'Lens Studio Cube Bouncer for the confused Unity developer: adding (spatial) sounds without crashing'
date: 2025-02-02T00:00:00.0000000+01:00
categories: []
tags:
- Lens Studio
- Spectacles
- TypeScript
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2025-02-02-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-adding-(spatial)-sounds-without-crashing/dragsound.png
comment_issue_id: 484
---
With apologies for the somewhat pedantic title, but of all things, *sound* has given me quite some headaches. This will be a bit of an unusual blog, as it also shows a few things that did turn out *not* to work or worked differently than I thought. But of course, I will also show how I solved it.

## Objective

What I wanted to do was, just like in the original HoloLens cube bouncer, play four sounds:
* when the grid is (re)created
* when a cube bounces against another cube
* when a cube bounces with something else
* when the cubes are moved back to the original position.

To this end, I have used four audio files, duly converted to mono mp3 files:
* Ready.mp3
* BounceCube.mp3
* BounceWall.mp3
* Return all.mp3

So I added those to the Audio folder. Let the fun begin.

## Adapting CubeManager.

First, we add two input items at the top, of type "`AudioComponent`"

```typescript
@input private readySound: AudioComponent
@input private revertAllSound: AudioComponent
```

At the end of the method `createNewGrid`, we simply add:

```typescript
this.readySound.play(1);
```

And at the end of the method `revertAll`:

```typescript
this.revertAllSound.play(1);
```

This means: play the sound one time. A bit weird, because why would I want to play it multiple times, but whatever.

## Assigning sound

Here we run into something weird as well, that surely made me feel the confused Unity developer. You would think you can drag your audio file onto the AudioComponent like this:  

![dragsound](/assets/2025-02-02-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-adding-(spatial)-sounds-without-crashing/dragsound.png)

But you cannot. Because it needs an Audio Component, not an Audio *Track* Component. Next pitfall I fell into: I dragged the audio track over like this (1)

![dragsound2](/assets/2025-02-02-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-adding-(spatial)-sounds-without-crashing/dragsound2.png)

And that will indeed create an Audio component, but you cannot drag it over from the Inspector (2) to the Read Sound Audio field, like you can in Unity.

\*Sigh\*

Next thing I tried was this:

![dragsound3](/assets/2025-02-02-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-adding-(spatial)-sounds-without-crashing/dragsound3.png)

And that works, *provided you have only one Audio component on your HologramCollection*. But I have two sounds. I can add a second Audio Component, but I cannot drag a single *component* in, only a Scene object *containing that component*. Result: I got the first sound twice.

![frustrated](/assets/2025-02-02-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-adding-(spatial)-sounds-without-crashing/frustrated.png)

So I took a step back. Apparently there can only be one Audio Component on a Scene Object. The solution then is, logically, to *create scene objects for every Audio Component*. In practice:

- Create an empty scene object, for instance under HologramCollection. I named it Ready, so I could see what AudioTrack belonged to it
- Drag the AudioTrack "Ready" on top of it:

![ready1](/assets/2025-02-02-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-adding-(spatial)-sounds-without-crashing/ready1.png)

- Select the HologramCollection Scene object in the Scene Hierarchy, and drag the "Ready" *Scene object* from the scene on top of the "Ready Sound" box.

![ready2](/assets/2025-02-02-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-adding-(spatial)-sounds-without-crashing/ready2.png)

- Repeat for the return all sound. Net result should be:

![allsounds](/assets/2025-02-02-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-adding-(spatial)-sounds-without-crashing/allsounds.png)

If you now start the app, or press "Create grid" on the hand menu, you hear the "prinnggg" sound that has been the hallmark of literally all my XR apps since my very first HoloLens 1 trial app, and a kind of "tadadaaa" when you press the "Revert all" button.

Long story short: if you need two audio components, you will need separate Scene Objects. Confusing and weird for a Unity developer, but that's the way it is.

## Spatial sounds: some preparations

If you have looked a bit deeper into the Audio Component, you might have seen this:

![audiocomponent](/assets/2025-02-02-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-adding-(spatial)-sounds-without-crashing/audiocomponent.png)

I will let you in on a little secret: everything under "Spatial Audio" will do zilch, nothing, nada at all, whatever check boxes you check, whatever values you set... unless you add an Audio Listener Component to the Camera in the Scene, as was [told to me by a Product Team Member](https://www.reddit.com/r/Spectacles/comments/1hqn97i/comment/m4vd2wr/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button). So you go to the Camera in the Scene Hierarchy:

![camara](/assets/2025-02-02-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-adding-(spatial)-sounds-without-crashing/camara.png)

You click "+ Add Component" at the bottom, search for "Audio Listener", and select it

![audiolistener](/assets/2025-02-02-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-adding-(spatial)-sounds-without-crashing/audiolistener.png)

## Adapting CubeController

Again we add two `AudioComponent` inputs at the top:

```typescript
@input private bounceCubeSound: AudioComponent
@input private bounceOtherSound: AudioComponent
```

And another private member:

```typescript
private cubeId : number;
```

At the end of the initialize method, we add:
```typescript
this.cubeId = id;
```

and we add this public method:

```typescript
public getID() : number {
    return this.cubeId;
}
```

Handling collisions, and determining what sound to play, is handled by the `handleCollisionEnter` method

```typescript
private handleCollisionEnter(eventArgs: CollisionEnterEventArgs) : void {
    var otherCubeController = eventArgs.collision.collider.getSceneObject().
        getComponent(CubeController.getTypeName());
    if (otherCubeController != null) {
        if( otherCubeController.getID() > this.cubeId ) {
            this.bounceCubeSound.play(1)
        }
    } else {
        this.bounceOtherSound.play(1)
    }
}
```

* When a collision is detected, we check if the other game object is a cube too
* If it is a cube, we determine if we have the highest id. If so, we will play the cube bounce sound (otherwise the other cube will - making the same sound twice does not really make sense)
* If we collided with something else (that most likely being a wall, floor, or piece of furniture) we play the bounce other sound.

As far as code is concerned, the only thing left now is to hook up the `onCollisionEnter` event to our `handleCollisionEnter` method

```typescript
this.bodyComponent.onCollisionEnter.add(this.handleCollisionEnter.bind(this));
```

## Configure stuff in the editor

We add the sounds to the cube prefab in the same manner we did for the other two sounds:

![soundtocube](/assets/2025-02-02-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-adding-(spatial)-sounds-without-crashing/soundtocube.png)

For both sounds, we set these spatial sound settings. I have messed around with values and settings until I found these to be working for me.

![setspatialsound](/assets/2025-02-02-Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-adding-(spatial)-sounds-without-crashing/setspatialsound.png)

And this works fine. Until you start swatting the boxes around a bit too enthusiastically so there's a lot of collision -  or when you click "Drop all". Then you will hear a cacophony of sound, and then...

\*Boop\* "An error occurred while running this lens". Your lens simply stops. Or rather, crashes.

## What's up?

*Apparently* the Spectacles don't take very well to playing a lot of sounds simultaneously. According to [someone from the product team the maximum is 32](https://www.reddit.com/r/Spectacles/comments/1hsps87/comment/m57ulbz/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button), but it should not crash a Lens. Unfortunately, it does crash a lens indeed, and pretty hard as well. Try [this specific branch of the demo project](https://github.com/LocalJoost/CubeBouncerDemoLens/tree/blog5a_crash) - which shows the app so far - you will get a crash indeed.

## Preventing crashing on too many sounds

The name of the game is to prevent the poor Spectacles from getting flooded with requests to play sound until it dies. For this, I created a helper class `AudioComponentPlayManager`

### AudioComponentPlayManager

This component handles requests to play sounds, and it allows only 20 sounds to be played simultaneously. Excess sounds are simply ignored. It's very simple:

```typescript
export class AudioComponentPlayManager{
    private audioComponents: AudioComponent[] = [];
    private MAX_AUDIO_COMPONENTS = 20;

    public addAudioComponent(audioComponent: AudioComponent): void {

        if(this.audioComponents.length < this.MAX_AUDIO_COMPONENTS){
            this.audioComponents.push(audioComponent);
            audioComponent.setOnFinish(() =>  
                this.audioComponents.splice(this.audioComponents.indexOf(audioComponent), 1));
            audioComponent.play(1);
        }
    }
}
```

If you want to play a sound, you call `addAudioComponent` on this class, and it will play its sound provided there are no more than 20 `AudioComponent`s playing already. When the `AudioComponent` is done playing, it removes itself from the list. As said before: if the list is full, the request to play will simply be ignored. Since the cubes produced either one of two sounds, we sure won't miss a few that get silently discarded, and in any case, we sure as hell ain't going to miss our app crashing on it. Maybe you can try to use a higher number than 20, but I like to stay on the safe side. 

### Adapting CubeController

To make this work, we add an import to `CubeController`:

```typescript
import { AudioComponentPlayManager } from "./AudioComponentPlayManager";
```

As well as a private member, along with a member `lastPlayTime` that we will use to reduce the number of sounds even more:

```typescript
private audioComponentPlayManager: AudioComponentPlayManager;
private lastPlayTime: number = 0;
```

The `CubeController`'s initialize method's signature is extended to get the `AudioComponentPlayManager` from the `CubeController`

```typescript
public initialize (id: number, initialPosition: vec3, initialRotation: quat, 
                   audioComponentPlayManager:AudioComponentPlayManager) : void {
    if (this.isInitialized) {
        return;
    }
    this.audioComponentPlayManager = audioComponentPlayManager;
```

And the `handleCollisionEnter` method then becomes this:

```typescript
private handleCollisionEnter(eventArgs: CollisionEnterEventArgs) : void {
    if (getTime() - this.lastPlayTime < 1) {
        return;
    }
    this.lastPlayTime = getTime();
    var otherCubeController = eventArgs.collision.collider.getSceneObject().
        getComponent(CubeController.getTypeName());
    if (otherCubeController != null) {
        if(otherCubeController.cubeId < this.cubeId) {
            this.audioComponentPlayManager.addAudioComponent(this.bounceCubeSound);
        }
    } 
    else {
        this.audioComponentPlayManager.addAudioComponent(this.bounceOtherSound);
    }
}
```

Simply put: if this component has played a sound less than 1 second before, ignore. Otherwise, the logic is the same as before, but instead of `CubeController` playing the collision sounds directly, `AudioComponentPlayManager` is asked to do that, and will do so if there's less than 20 sounds playing.

### Adapting CubeManager

Something has to provide that `AudioComponentPlayManager`, and that something is `CubeManager`.

It has gotten an extra import for `AudioComponentPlayManager` as well:

```typescript
import { AudioComponentPlayManager } from "./AudioComponentPlayManager";
```

At the start of `createNewGrid`, a new `AudioComponentPlayManager` is created:

```typescript
public createNewGrid() :void{
    for (const cube of this.cubes) {
        cube.getSceneObject().destroy();
    }

    var audioComponentPlayManager = new AudioComponentPlayManager();
```

... and that is passed on to `createCube` method who in turn passes it on to the previously adapted `cubeController.initialize`

```typescript
private createCube(id: number, position: vec3, rotation: quat, 
                   audioComponentPlayManager: AudioComponentPlayManager) {
    const clone = this.cubePrefab.instantiate(this.getSceneObject());
    var cubeController = clone.getComponent(
      CubeController.getTypeName()) as CubeController;
    cubeController.initialize(id++, position, rotation, 
      audioComponentPlayManager);
    this.cubes.push(cubeController);
}
```

Note: every call to `createNewGrid` creates a new `AudioComponentPlayManager` - because at grid creation, all previously existing cubes get destroyed, including their audio components, so it makes no sense keeping a reference to it.

## Some thoughts about (Spatial) Sound on Spectacles

Spatial sound has been a kind of a thing for me since the early days of HoloLens 1, and I think I was one of the very few to actually utilize it, even way beyond its original intentions, to the point of having 1:1 calls with the erstwhile Spatial Sound PM at Microsoft. I tend to use it whenever and wherever available. So I am inclined to think I have a kind of expert opinion on this, and my opinion is that while the graphics part of Spectacles and Lens Studio are pretty well fleshed out, the sound part apparently has gotten less attention and can definitely benefit from some more TLC. Sound playing is obviously hard on the device (you can see that making a lot of sound actually degrades optical performance) and being able to make an app *crash* by playing too much *sound* is something that should not be able to happen, I feel. There are more puzzling things - if you play Spatial Sounds in the Lens Studio WebCam Preview, you will be 'rewarded' by horrible screeching sounds, and the only way to make that stop is by actually exiting Lens Studio (and believe me, you will - very fast). Another thing I ran into: `AudioComponent.isPlaying` returns true even though nothing is playing. So here's work to do.

However, with some metaphorical duct tape, I have been able to at least generate the illusion that it's working like it should be, and I would be the last one to deny that it's actually fun and challenging to run into such a weird thing and find a way to deal with it.

## Conclusion

The CubeBouncer app is now complete, so this marks the end of my "Lens Studio for the confused Unity developer" series - at least for the time being. If you have followed the road along with me, you should by now no longer be confused. Although there is still enough uncovered Lens Studio ground for seasoned Unity developers - you are now at the same point as I am, and probably able to continue by yourself.

I will still be sharing things I will discover in the future - both in Lens Studio, Unity, or other things I am tinkering with or environments I am dabbling in. I am not sure in what context - but it will most likely take the form of a blog with a project - which I have been doing since 2007.

The final project [can be found here](https://github.com/LocalJoost/CubeBouncerDemoLens).