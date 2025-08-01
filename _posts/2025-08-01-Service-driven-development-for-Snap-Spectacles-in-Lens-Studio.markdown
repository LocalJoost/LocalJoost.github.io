---
layout: post
title: Service driven development for Snap Spectacles in Lens Studio
date: 2025-08-01T00:00:00.0000000+02:00
categories: []
tags:
- Lens Studio
- Spectacles
- Services
- Architecture
published: true
permalink: 
featuredImageUrl: http://localjoost.github.io/assets/2025-07-30-Service-driven-development-for-Snap-Spectacles-in-Lens-Studio/handmenu.png
dontStripH1Header: false
comment_issue_id: 488
---
Regular readers may have noticed my blog production has dropped dramatically over the past few months, to an unprecedented low of only *two posts in nearly half a year*. I blame this entirely on [Jesse McCulloch](https://www.linkedin.com/in/jessebmcculloch/) 😁, community manager at [Snap, Inc.](https://www.linkedin.com/company/snap-inc-co), for getting me hooked on [Snap Spectacles 2024](https://newsroom.snap.com/sps-2024-spectacles-snapos) development in late 2024. The result, [HoloATC for Snap Spectacles](https://www.spectacles.com/lens/1e9933cbc6df44ceb936b154b0cb7b78?type=SNAPCODE&metadata=01), has been published recently. I was so engrossed in this project that I essentially forgot to blog, and now I have a substantial backlog of topics to share - both for Unity and Lens Studio. So let’s get started.

Warning: this blog does not showcase flashy features. It is essentially a dense piece of software architecture, demonstrating how you can organize your code in a professional, reusable, and maintainable way by decoupling logic from UI.

## Services! We Need Services!

Entering Lens Studio and TypeScript made me miss many things we take for granted in Unity and C#, but most of all I missed the powerful combination of [Reactive Extensions for Unity (UniRx)](https://github.com/neuecc/UniRx) and the outstanding work by [Reality Collective](https://www.linkedin.com/company/realitycollective/), particularly the [Service Framework](https://github.com/realitycollective/com.realitycollective.service-framework), created by (among others) my close friend and fellow MVP [Simon 'DarkSide' Jackson](https://www.linkedin.com/in/xrconsultant/).

## Eh... why do we need this again?

Services are an excellent way to decouple business logic from display logic. As developers painfully learned in the early days of Visual Basic (yes, I am *that* old), mixing these two leads to a tangled mess where even a minor UI change can break the entire application. Unity took this to another level by allowing business logic to be embedded in components (behaviours), scattered across the scene. Components could reference each other directly by name or hierarchy, leading to chaos when someone “reorganized the scene a bit” - behavior A could no longer find behavior B because it was no longer two levels down but three, causing the app to crash due to null reference exceptions.

Lens Studio draws significant inspiration from Unity. If you don’t look closely, you might assume they are the same. While they are not, Lens Studio inherits the concept of components (called “behaviours” in Unity), and thus also inherits the risk of embedding application logic directly in the scene. Logic that can break with even a minor scene change.

With the help of various models in GitHub Copilot, I set out to create - or find - something similar. This is the result.

## Reactive properties and events

A service component must support at least:
- A kind of reactive property that allows components to subscribe to value changes - typically indicating a state change in the app.
- A command, event, or similar fire-and-forget mechanism where subscribers can listen, but which may or may not carry data.
- Both must come in variants that can be externally modified or invoked (from multiple locations), as well as read-only variants where the service itself controls the change or firing.

In [UniRx](https://github.com/neuecc/UniRx), these are `ReactiveProperty` and `ReactiveCommand`. There is also `IReadOnlyReactiveProperty` for read-only access, and `ReactiveCommand` includes a `CanExecute` property to prevent unintended execution. We need equivalent constructs in Lens Studio TypeScript.

For events - fire-and-forget mechanisms - there is already a class in the Spectacles Interaction Kit (SIK, or how *not* to name a component 😁) called `Event<T>`, which supports subscription, unsubscription, and invocation. It also includes an interface `PublicApi<T>` that allows callers to subscribe and unsubscribe but not invoke the event - a “read-only event.”

In the [Spectacles Samples](https://github.com/Snapchat/Spectacles-Sample) repository, there [used to be an `Observable<T>` class](https://github.com/Snapchat/Spectacles-Sample/blob/2263e8ab3b56a748c7a29aae7fe521d11ddfb01e/HighFive/Assets/SpectaclesSyncKit/Utils/Observable.ts) that provided basic `ReactiveProperty` functionality - until it was [apparently deleted on May 15th](https://github.com/Snapchat/Spectacles-Sample/commit/67ee3423df34e084a570ff3d26958a8f82f085f7#diff-4ca688defae1c1fbb1deb466617168cceda5357ea13fe53d27afade744382ed6). However, by then I had already copied it into my project folder, and I still use it - both in my app and in this sample.

The `Observable` class also includes a read-only interface called `PublicObservable`, and together they form a minimal equivalent of `ReactiveProperty` and `IReadOnlyReactiveProperty`.

The `Observable` code is as follows:

```typescript
import Event, {PublicApi} from "SpectaclesInteractionKit.lspkg/Utils/Event"

/*
 * An PublicObservable is a value that allows a consumer to subscribe to
 * OnChanged callbacks. This interface is read-only.
 */
export interface PublicObservable<T> {
  // The current value
  readonly value: T

  onChanged: PublicApi<T>
}

// A value that allows consumers to register for OnChanged callbacks
// that are triggered whenever the value is set.
export default class Observable<T> implements PublicObservable<T> {
  private val: T
  private onChangedEvent = new Event<T>()
  public onChanged = this.onChangedEvent.publicApi()

  // Gets the current value
  public get value(): T {
    return this.val
  }

  // Sets the current value. Will trigger an onChanged callback if
  // the value has changed.
  public set value(val: T) {
    this.set(val)
  }

  // Sets the current value. Will trigger an onChanged callback if
  // the value has changed.
  public set(val: T): boolean {
    if (this.val !== val) {
      this.val = val

      this.onChangedEvent.invoke(val)
      return true
    }
    return false
  }

  constructor(val: T) {
    this.val = val
  }
}
```

To be absolutely clear: *I did not write this code*. I copied it from the [Spectacles Samples](https://github.com/Snapchat/Spectacles-Sample), where it resided until last May.

## A very bare-bones service manager

By “bare bones,” I mean *truly* bare bones. In no way does this compete with the original Reality Collective version - but it handles the basics. With some concessions to the quirks of TypeScript.

```typescript
export class ServiceManager {
  private static instance: ServiceManager;
  private services: Map<any, any> = new Map();

  public static getInstance(): ServiceManager {
    if (!ServiceManager.instance) {
      ServiceManager.instance = new ServiceManager();
    }
    return ServiceManager.instance;
  }

  private constructor() {}

  public register<I, C>(interfaceType: I, implementationType: C): void {
    let serviceName = 'Unknown Service';
    if (typeof interfaceType === 'function') {
      serviceName = interfaceType.name || 'Anonymous Implementation';
    }
    if (this.services.has(interfaceType)) {
      print(`Service for ${interfaceType} is being overwritten`);
    }

    print(`Registering service for ${serviceName}`);
    this.services.set(interfaceType, implementationType);
  }

  public get<T>(interfaceType: Function): T | null {
    return this.services.get(interfaceType) as T || null;
  }

  public remove(interfaceType: Function): boolean {
    return this.services.delete(interfaceType);
  }

  public clear(): void {
    this.services.clear();
  }
}
```

## Registering and using a service

I typically do this at startup, in a component called `BootStrapper`. In its `Awake` method, it contains something like:

```typescript
var serviceManager = ServiceManager.getInstance();
var myService = new MyService();
serviceManager.register(IMyService, myService);
```

Then elsewhere, you can access the service like this:

```typescript
private myService: IMyService;
myService = serviceManager.get(IMyService);
```

## That Is All Very Nice, Mr. Architect - How About an Example?

Ah yes, of course. I created [a simple example](https://github.com/localJoost/SpectaclesServiceSample). It features [a hand menu](https://localjoost.github.io/Lens-Studio-Cube-Bouncer-for-the-confused-Unity-developer-add-a-hand-menu/) that allows you to spawn boxes in red, green, or blue. It’s admittedly minimal in functionality, but it demonstrates the core principle.

![Blockdemo](/assets/2025-08-01-Service-driven-development-for-Snap-Spectacles-in-Lens-Studio/Blockdemo.gif)

We have the following business rules:
1. You can only spawn boxes after selecting a color - red, green, or blue. If the color is white, you cannot create a block.
2. A color may be any value, but not `null`.

We also have these functional requirements:
1. If you change the color after spawning boxes, all existing boxes should update to the new color.
2. If you tap any spawned box, all boxes turn white (which is not a selectable color, so rule #1 disables the "Create Block" button).

Essentially, the color can be changed from multiple locations, yet there must be centralized logic to keep everything synchronized. I know this is a contrived example, but if you implement it traditionally, every box must reference the menu, and the menu must reference every box. With a service, that problem disappears. All components communicate with the service, which holds the business logic. The view components only need to listen to and talk with the service.

## Service implementation

Let’s start at the top:

```typescript
export class BlockControlService implements IBlockControlService {

    private scriptComponent: ScriptComponent = null;
    private readonly createBlockRequestedInternal = new Event<void>();
    private readonly blockColorInternal = new Observable<vec4>(Colors.WHITE);
    private readonly canAddBlockInternal = new Observable<boolean>(false);

    public constructor(scriptComponent: ScriptComponent) {
        this.scriptComponent = scriptComponent;
        this.setCanAddBlockStatus();
    }

    private setCanAddBlockStatus(): void {
        this.canAddBlockInternal.set(this.blockColorInternal.value !== Colors.WHITE);
    }
```

Here we see internal properties: an event for block creation, an `Observable` for the current color, and an `Observable` indicating whether a block can be added. The constructor accepts a `ScriptComponent` - I always pass this in so I can hook into scene events like `Update`, though it’s not used in this sample. In `setCanAddBlockStatus`, we implement rule #1: you can only add a block when the color is not white. Initially, since the color is white, `canAddBlockInternal` is set to `false`.

To expose the color externally in a read-only way, we define a `PublicObservable` - a read-only version of the internal `Observable`, which only the service can modify:

```typescript
public get blockColor(): PublicObservable<vec4> {
    return this.blockColorInternal;
}
```

To allow external code to set the color (e.g., from the menu), we define a method that implements rule #2: a color must not be `null`.

```typescript
public setColor(color: vec4): void {
    if (!color) {
        return;
    }
    this.blockColorInternal.set(color);
    this.setCanAddBlockStatus();
}
```

Finally, we handle block creation:

```typescript
get createBlockRequested(): PublicApi<void> {
    return this.createBlockRequestedInternal.publicApi();
}

public addBlock(): boolean {
    if (this.canAddBlockInternal.value) {
        this.createBlockRequestedInternal.invoke();
    }
    return this.canAddBlockInternal.value;
}

public canAddBlock(): PublicObservable<boolean> {
    return this.canAddBlockInternal;
}
```

External listeners can subscribe to `createBlockRequested`, which fires when `addBlock` is called - but only if `canAddBlockInternal.value` is `true`. The read-only `canAddBlock` observable can be checked first to determine whether calling `addBlock` makes sense - e.g., to disable UI elements that trigger it. That’s exactly what we’ll do. But first, the service setup.

## Initializing and registering a service

As mentioned, I use a `Bootstrapper` component:

```typescript
@component
export class Bootstrapper extends BaseScriptComponent {

    onAwake() {
        var blockControlService = new BlockControlService(this);
        ServiceManager.getInstance().register(
           IBlockControlService, blockControlService);
    }   
}
```

## The service interface

If you open `IBlockControlService.ts`, you’ll notice something peculiar:

```typescript
export interface IBlockControlService {
    blockColor: PublicObservable<vec4>;
    setColor(color: vec4): void;
    createBlockRequested: PublicApi<void>;
    canAddBlock(): PublicObservable<boolean>;
    addBlock(): boolean;
}

// Create a constructor function with the same name
export function IBlockControlService() {}
```

Besides the expected interface, you’ll also see a constructor function with the same name. If you’re coming from C#, like me, this is confusing - but here’s the point: in TypeScript, an interface is not a type and does not exist at runtime. It cannot be stored in the `Map<any, any>` used by `ServiceManager`. This trick makes it *appear* to be a type so you can register it via its interface. It’s a workaround.

This has a notable side effect: if you try to retrieve the service like this:

```typescript
var blockService = ServiceManager.getInstance().get(IBlockControlService);
```

`blockService` will be of type `Any`, and you won’t be able to call any `IBlockControlService` methods on it.

You must either use this:
```typescript
private blockControlService: IBlockControlService; 
var blockService = ServiceManager.getInstance().get(IBlockControlService);
```

or this:

```typescript
var blockService = ServiceManager.getInstance().
   get<IBlockControlService>(IBlockControlService);
```

This is a perfect example of the [Law of Leaky Abstraction](https://en.wikipedia.org/wiki/Leaky_abstraction). TypeScript is syntactic sugar over JavaScript, and under the hood, it’s just as weakly typed. TypeScript provides guardrails - but they’re more like rubber bands than steel girders.

## HandMenuController

It begins like this:

```typescript
export class HandMenuController extends BaseScriptComponent {

    @input addblockButton: Interactable;
    @input redButton: ToggleButton;
    @input greenButton: ToggleButton;
    @input blueButton: ToggleButton;
    blockControlService: IBlockControlService;

    private onAwake(): void {
        this.blockControlService = ServiceManager.getInstance().get(IBlockControlService);
        const delayedEvent = this.createEvent("DelayedCallbackEvent");
        delayedEvent.bind(() => {
            this.initializeEvents();
        });
        delayedEvent.reset(1);
        this.createEvent("OnEnableEvent").bind(() => this.onEnabled());
    }

    private onEnabled(): void {
        this.handleAllOff();
        this.handleColorChange(this.blockControlService.blockColor.value);
        this.setAddButtonStatus();
    }
```

This component has inputs for the four buttons. It schedules an event to bind callbacks after one second. It also binds a callback to the `OnEnableEvent`. This is necessary because a hand menu can appear or disappear at any time, and the component may be disabled accordingly. When re-enabled, we must sync with the central logic - the service - again.

We continue with initialization:

```typescript
initializeEvents(): void {
	this.addblockButton.onTriggerEnd(this.addBlock.bind(this));
	this.setAddButtonStatus();

	this.redButton.onStateChanged.add((isOn: boolean) => {
		if (isOn) {
			this.blockControlService.setColor(Colors.RED);
		}
		else {
			this.handleAllOff();
		}
	});

    // Same for blue and green button, omitted

	this.blockControlService.blockColor.onChanged.add((color: vec4) => {
		this.handleColorChange(color);
	});

	this.blockControlService.canAddBlock().onChanged.add((canAdd: boolean) => {  
		this.setAddButtonStatus();
	});
}
```

Here we see:
- Clicking the “Add Block” button calls `addBlock`.
- When the red button is toggled on, it sets the color in the service to red; otherwise, it handles the “all off” case. The green and blue buttons work similarly (code omitted).
- If the block color changes in the *service*, the buttons update accordingly - because tapping a block can change the color from elsewhere.
- The “Add Block” button is enabled or disabled based on the current color.

Implementation:

```typescript
private addBlock(): void {
    this.blockControlService.addBlock();
}

private handleAllOff(): void {
    if (!this.redButton.isToggledOn && !this.greenButton.isToggledOn && !this.blueButton.isToggledOn) {
        this.blockControlService.setColor(Colors.WHITE);
    }
}

private handleColorChange(color: vec4): void {
    this.redButton.isToggledOn = Colors.isEqual(color, Colors.RED);
    this.greenButton.isToggledOn = Colors.isEqual(color, Colors.GREEN);
    this.blueButton.isToggledOn = Colors.isEqual(color, Colors.BLUE);
}

private setAddButtonStatus(): void {
    this.addblockButton.getSceneObject().enabled = 
      this.blockControlService.canAddBlock().value;
}
```

## But what is creating the blocks?

This simple component - what Unity developers like me would call a “behaviour”:

```typescript
export class BlockBuilderController extends BaseScriptComponent {

    @input private blockPrefab: ObjectPrefab;

    private blockControlService: IBlockControlService;
    private camera = WorldCameraFinderProvider.getInstance().getTransform();

    onAwake() {
        this.blockControlService = ServiceManager.getInstance().get(IBlockControlService);
        this.blockControlService.createBlockRequested.add(() => this.createBlock());
        this.createEvent("OnDestroyEvent").bind(() => {
            this.onDestroy();
        });
    }

    private createBlock(): void {
        const forward = this.camera.forward;
        const rotation = this.camera.getWorldRotation();
        const distance = Math.random() * 200 | 0 + 30;
        const clone = this.blockPrefab.instantiate(this.getSceneObject());
        clone.getTransform().setWorldPosition(
            this.camera.getWorldPosition().sub(forward.mult(VectorUtils.scalar3(distance)));
        clone.getTransform().setWorldRotation(rotation);
    }

    onDestroy() {
        this.blockControlService.createBlockRequested.remove(this.createBlock);
    }
}
```

When the service’s `createBlockRequested` event fires, it instantiates a `blockPrefab` at a random distance between 30 and 230 cm in front of the camera, using the camera’s rotation. That’s it.

## Controlling a block

The final piece: every `blockPrefab`, in addition to a `RenderMeshVisual`, has a `PhysicsBody` and an `Interactable`.

![](/assets/2025-08-01-Service-driven-development-for-Snap-Spectacles-in-Lens-Studio/block.png)

And this component:

```typescript
@component
export class BlockController extends BaseScriptComponent {

    private blockControlService: IBlockControlService;
    @input  blockMesh: RenderMeshVisual;

    onAwake() {
        this.blockControlService = ServiceManager.getInstance().get(IBlockControlService);
        this.setBlockColor(this.blockControlService.blockColor.value);
        this.blockControlService.blockColor.onChanged.add((color: vec4) => {
            this.setBlockColor(color);
        });
        this.getSceneObject().getComponent(Interactable.getTypeName()).onTriggerEnd.add(() => {
            this.blockControlService.setColor(Colors.WHITE);
        });
    }

    private setBlockColor(color: vec4): void {
        const material = this.blockMesh.mainMaterial;
        const clone = material.clone();
        clone.mainPass.baseColor = color;
        this.blockMesh.mainMaterial = clone;
    }
}
```

It listens to changes in the service’s `blockColor` and updates the `RenderMeshVisual` accordingly. If the `Interactable` is tapped, it sets the color in the service to white. Then all blocks - including this one - turn white. Note it does not set its own color to white; instead, it tells the service to change the color. The service then fires the `blockColorChanged` event. Action initiation is in the UI, display is handled by the UI, and logic resides in the service.

## Concluding words

Congratulations and compliments if you made it this far, understood what I’m describing, and see why this matters - even for non-trivial apps. Software architecture may seem boring and tedious, but it’s like the wiring in your house: when done right, it’s nearly invisible and works flawlessly. When done poorly, you end up with issues like the fridge turning off when you switch on the attic light.

Though TypeScript isn’t ideal for this kind of architecture, it’s absolutely possible to build a solid, well-structured solution - just with more discipline than in C#, which by virtue of being strongly typed offers stronger safety nets.

As usual, [the sample project is available on GitHub](https://github.com/localJoost/SpectaclesServiceSample).