---
layout: post
title: Getting components by their base class name in Lens Studio
date: 2025-12-07T00:00:00.0000000+01:00
categories: []
tags:
- Lens Studio
- Spectacles
- ScriptComponent
- TypeScript
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2025-12-07-Getting-components-by-their-base-class-name-in-Lens-Studio/logger.png
comment_issue_id: 494
---
Yet another small thing, this time to solve something that annoyed the heck out of me since day one. I really dislike this construction that you have to use when you have to get a reference to a component from a `SceneObject`

```typescript
someSceneObject.getComponent(MyComponentClass.getTypeName()) as MyComponentClass;
```

And what is even *worse*, it does not work *at all* when when you need to find the reference but you only have the component's *base class*. Suppose I have this trivial base controller component:

```typescript
@component
export class MyBaseClass extends BaseScriptComponent {

    public hello(): void {
        print("Hello from MyBaseClass");
    }
}
```

However, I have multiple prefabs types with a slightly different child class suited for their particular purpose, for instance:

```typescript
import { MyBaseClass } from "./MyBaseClass";

@component
export class MyChildClass extends MyBaseClass {

    public hello(): void {
        print("Hello from MyChildClass");
    }
}
```

If I add this child class to my prefab, then try to find a reference to its controlling component using the `MyBaseClass` class

```typescript
myPrefab.getComponent(MyBaseClass.getTypeName()) as MyBaseClass;
```

I wil get a null value result! This is highly confusing for someone coming from Unity, and rather limiting as well. 

However, there is a solution. I have extended my SceneUtils with this little method:

```typescript
export function getComponent<T>(
    sceneObject: SceneObject,
    myClassType: new (...args: any[]) => T): T | null {
    for (const component of sceneObject.getComponents("Component") as Component[]) {
        if (component instanceof myClassType) {
            return component as T
        }
    }
    return null;
}
```

I made a little test script [in the demo project](https://github.com/LocalJoost/GetComponentLikeUnity.git) to show how it works:

```typescript
import { MyBaseClass } from "./MyBaseClass";
import { getComponent } from "LocalJoost/Utilities/SceneUtils";

@component
export class TestScript extends BaseScriptComponent {
    @input prefab : ObjectPrefab;
   
    onAwake() {
        var instance = this.prefab.instantiate(this.getSceneObject());
        var getComponentResult = instance.getComponent(MyBaseClass.getTypeName()) as MyBaseClass;
        print("getComponentByClass: " + (getComponentResult != null));

        var findScriptComponentResult = getComponent<MyBaseClass>(instance, MyBaseClass);
        print("findScriptComponentByClass: " + (findScriptComponentResult != null));
        findScriptComponentResult.hello();
    }
}
```

Here's the logger output that actually *proves* it works

![logger](/assets/2025-12-07-Getting-components-by-their-base-class-name-in-Lens-Studio/logger.png)

Although I search using the *base* class, I get a reference to a *child* class object. Which is exactly what you would expect. Or at least what *I* would expect. It's only five lines of code, so I wonder why Snap didn't implement it this way in their own `getComponent`. Maybe they wanted to leave in some challenges for grumpy old skool software engineers to solve 😁. However, this works and it easy to use.