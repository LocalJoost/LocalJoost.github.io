---
layout: post
title: Finding all script components of a type in a Lens Studio scene
date: 2025-12-06T00:00:00.0000000+01:00
categories: []
tags:
- Lens Studio
- Spectacles
- ScriptComponent
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2025-12-06-Finding-all-script-components-of-a-type-in-a-Lens-Studio-scene/found.png
comment_issue_id: 493
---
A short and small one, but one that is very useful for quick proof of concepts and/orhackatons - and Snap likes to organize those at the moment. If you are working with multiple people at a such high paced short interations project, scene stucture tends to change a lot, breaking references to objects in the scene, and sending your project into havoc at every commit or pull - not to mention your nerves and the team spirit. There are, however, in the Spectacles Interaction Kit, a few functions that allow you to quickly find objects in the scene *from code by name or type* rather than referencing by @input fields, and obtains references that way:
* findSceneObjectByName
* findComponentInChildren
* findAllComponentsInChildren
* findScriptComponentInChildren
* findAllScriptComponentsInChildren
* findComponentInParents
* findAllComponentsInParents
* findScriptComponentInParents
* findComponentInSelfOrParents
* findAllScriptComponentsInSelfOrParents

If you are coming from Unity, you might easily fall into the trap (as I did) of seeing not differences between `Component` and `ScriptComponent`, use `findComponentInChildren` or `findAllComponentsInChildren`, notice that that won't even compile because it wants a "keyof ComponentNameMap" in stead of a type - and waste a lot of time on that. So if you want to find `ScriptComponent`/behaviours that you have placed in the scene and your team mate has moved it *somewhere*, you should rather use the \*findScriptComponent\* methods than the \*findComponent\* methods.

*However*, with the exception of `findSceneObjectByName`, these only work if you provide a top `SceneComponent`. So if your lovely team mates have created multiple root scene objects, good luck finding your scripts. 

So I created this very little helper function that does that for you:  

```typescript
import { findAllScriptComponentsInChildren } 
  from "SpectaclesInteractionKit.lspkg/Utils/SceneObjectUtils";
import { DEFAULT_MAX_CHILD_SEARCH_LEVELS } 
  from "SpectaclesInteractionKit.lspkg/Utils/SceneObjectUtils";

export function findAllScriptComponentsInScene<T extends ScriptComponent>(
  scriptClass: new (...args: any[]) => T,
  maxDepth: number = DEFAULT_MAX_CHILD_SEARCH_LEVELS): T[] | null {
    const foundComponents: T[] = [];
    const rootCount = global.scene.getRootObjectsCount();
    try {
        for (let i = 0; i < rootCount; i++) {
            const so = global.scene.getRootObject(i);
            const components = findAllScriptComponentsInChildren<T>(so, scriptClass, maxDepth);
            if (components && components.length > 0) {
                foundComponents.push(...components);
            }
        }
        return foundComponents.length > 0 ? foundComponents : null;
    } catch (error) {
        print("Error while searching for components in scene: " + error);
        return null;
    }
}
```

I created [a little demo project](https://github.com/LocalJoost/FindAnyScriptComponentInSceneDemo.git) that demonstrates this. For this I made a trivial `TestTarget` ScriptComponent that I sprinkled through the scene:

```typescript
@component
export class TestTarget extends BaseScriptComponent {

    public getInfo(): string {
        return "I am a TestTarget component on " + this.getSceneObject().name + "!";
    }
}
```

And an equally trivial `TestFinder` ScriptComponent that finds them:  
  
```typescript
import { findAllScriptComponentsInScene } from "LocalJoost/Utilities/SceneUtils";
import { TestTarget } from "TestTarget";

@component
export class TestFinder extends BaseScriptComponent {
    onAwake() {
        const foundTargets = findAllScriptComponentsInScene<TestTarget>(TestTarget);
        if (foundTargets) {
            for (const target of foundTargets) {
                print(target.getInfo());
            }
        }
    }
}
```

Result in the Lens Studio Logger panel shows it actually works:

![found](/assets/2025-12-06-Finding-all-script-components-of-a-type-in-a-Lens-Studio-scene/found.png)

A word of warning: this is highly inefficient as you are bascially traversing the *whole scene* to find script components. This can be useful at startup, for connecting the dots initially, especially in the circumstances I described. But do not use this very often, like from an Update loop, because especially in a large scene it can be very resource intensive and slow.