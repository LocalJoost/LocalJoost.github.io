---
layout: post
title: Model driven Mixed Reality apps using UniRx and a MRKT extension service, part 1
date: 2020-11-20T00:00:00.0000000+01:00
categories: []
tags:
- HoloLens2
- MRTK2
- Unity3d
- Windows Mixed Reality
- UniRx
featuredImageUrl: https://LocalJoost.github.io/assets/2020-11-20-Model-driven-Mixed-Reality-apps-using-UniRx-and-a-MRKT-extension-service,-part-1/spagetti.png
comment_issue_id: 362
---
Unity, my preferred platform for making Mixed Reality apps, is a gaming engine. The mindset of people using platforms like this is definitely different from those of us who grew up in Enterprise environments. The stuff I saw in Unity tutorials, or apps developed by Unity aficionados, looked to me, frankly, a lot like this:
![](/assets/2020-11-20-Model-driven-Mixed-Reality-apps-using-UniRx-and-a-MRKT-extension-service,-part-1/spagetti.png)
Now this may just be my lack of skill or imagination, but I frequently lost track of the flow of events, what state was kept where - even in my own apps. When I became the lead Senior Mixed Reality Architect at [Velicus](https://velicus.nl/) and needed to build apps that are a magnitude larger and more complex than my own store apps, I knew I had to find a radical new way to keep on top of things.

## Requirements
1. I need to have a mechanism that kept the state of the app in a central, easily accessible way - kind of like I built my XAML apps using MVVMLight.
2. To make this testable, I should be able to use a form of dependency injection
3. I need to have a kind of pub/sub mechanism like INotifyPropertyChanged, that allowed me to respond to property changes
4. I need a kind of messenger that allows me to send messages between separate parts of the model that have no knowledge of each other.

## Analysis 
Requirement 1 was easy enough to fill: the Mixed Reality Toolkit sports the idea of Extension Services. Using the service locator pattern, I could easily define a service to keep the state alive over scenes during the lifetime of the app. And since you can obtain a reference to MRTK extension services by *interface* name, I could use that for simple testing or mocking. A messenger I [already created myself once ](https://localjoost.github.io/using-messenger-to-communicate-between/) and that was easy enough to make into a service as well. But how about INotifyPropertyChanged?

Enter UniRx - [Reactive Extensions for Unity](https://github.com/neuecc/UniRx), created by MVP [Yoshifumi Kawai](https://twitter.com/neuecc), who I am not sure I have had the honor of meeting.

## Simple demo
<iframe width="650" height="365" src="https://www.youtube.com/embed/GrNxpIr0cps" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
You can see the following:
* If I select one toggle, nothing special happens
* If I select both toggles, the square on top of the menu turns green
* If I click the reset button, both toggles spring back to unselected - and the square on top becomes red again.

There is no interaction between UI components at all. Every UI element *only* interacts with the model. So how does this work?

## One model service to rule them all
I created a simple MRTK service that is the hat stand for my application's models. Right now, it hosts only one model, the TwoButtonModel:

```csharp 
namespace ReactNativeDemo.State
{
    [Serializable]
    [MixedRealityExtensionService(...stuff omitted...)]
    public class StateService : BaseExtensionService, IStateService,
                                IMixedRealityExtensionService
    {
        private StateServiceProfile stateServiceProfile;

        public StateService(string name,  uint priority, 
                            BaseMixedRealityProfile profile) :
            base(name, priority, profile) 
        {
            stateServiceProfile = (StateServiceProfile)profile;
        }

        public ITwoButtonModel ButtonModel { get; } = new TwoButtonModel();
    }
}
```
That model itself is very simple. It has only two properties:
```csharp
using UniRx;

namespace ReactNativeDemo.State
{
    public class TwoButtonModel : ITwoButtonModel
    {
        public BoolReactiveProperty SelectOne { get; } = 
           new BoolReactiveProperty();
        public BoolReactiveProperty SelectTwo { get; } = 
           new BoolReactiveProperty();
    }
}
```
Notice both the service and this model in the service are only accessible via an interface. Now notice the properties we have here - they are not simple booleans, but `BoolReactiveProperty`. They look like they are read-only, but they are not. You can set them using the Value property, kind of like `Nullable<T>` works: 
```csharp
SelectOne.Value = false; 
```

## Subscriptions
Now the fun thing is, you can *subscribe* to a ReactiveProperty of any type and get notified of any changes. We can see this happen in the ToggleButton1Controller, that is attached to Toggle 1:

```csharp
using Microsoft.MixedReality.Toolkit.UI;
using UniRx;
using ReactNativeDemo.Controllers.Base;

namespace ReactNativeDemo.Controllers
{
    public class ToggleButton1Controller : BaseController
    {
        public void SetState()
        {
            AppState.ButtonModel.SelectOne.Value =
               GetComponent<Interactable>().IsToggled;
        }

        private void OnEnable()
        {
            if(!isInitialized)
            {
                isInitialized = true;
                AppState.ButtonModel.SelectOne.Subscribe(
                    p=> GetComponent<Interactable>().IsToggled = p).AddTo(this);
            }
        }
    }
}
```
If you click the button, SetState is called. This basically only makes the value of the model's SelectOne property equal to the Interactable's IsToggled property. Interactable is a script is a standard feature of the MRKT buttons.

In the OnEnable method we *subscribe* to any changes of SelectOne and populated the value to the Interactable's IsToggled. So SetState populates the UI state to the model, and the Subscribe populated any model changes back to the UI. Since the subscribe only fires when a value *changes*, this prevents a circular event loop. Lacking data binding like we had in XAML, this is the way I connect models to UI, and vice versa. There is an almost identical class for Toggle 2, only it targets the SelectTwo property. There's definitely room for improvement as far as duplicate code goes, but I didn't want to obscure too much detail in base classes.

As a final notice one this class, take note of the .AddTo(this): a subscription is something that might stay alive even after the behaviour itself is destroyed, and by using AddTo(this) the subscription will be destroyed automatically when the behaviour is destroyed.

## BaseController

The base class for all controllers is `BaseController`. It only contains a shortcut property to the State service. Notice the dirty trick with the tight loop to make *sure* the Mixed Reality Toolkit itself is initialized.

```csharp
using Microsoft.MixedReality.Toolkit;
using ReactNativeDemo.State;
using UnityEngine;

namespace ReactNativeDemo.Controllers.Base
{
    public abstract class BaseController : MonoBehaviour
    {
        protected bool isInitialized;
        private IStateService state;

        protected IStateService AppState
        {
            get
            {
                while (!MixedRealityToolkit.IsInitialized && Time.time < 1);
                return state ?? (state =
                  MixedRealityToolkit.Instance.GetService<IStateService>());
            }
        }
    }
}
```
## Combining two property values

As stated, the square on top in only green when both toggles are selected. This can be done using `CombineLatest`:
```csharp
using UniRx;
using UnityEngine;
using ReactNativeDemo.Controllers.Base;

namespace ReactNativeDemo.Controllers
{
    public class SelectionStateController : BaseController
    {
        private void OnEnable()
        {
            if(!isInitialized)
            {
                isInitialized = true;
                AppState.ButtonModel.SelectOne.CombineLatest(
                   AppState.ButtonModel.SelectTwo,
                     (one, two) => one && two).Subscribe(
                         r => GetComponent<Renderer>().material.color = r?
                           Color.green: Color.red).
                    AddTo(this);
            }
        }
    }
}
```

This may be a bit hard to read at first, but simply try to read it from front to back. `CombineLatest` fired when any of properties listed changes. Then we see a lambda `(one, two) => one && two)`.
* `SelectOne` gets populated into `one`
* `SelectTwo` gets populated into `two`
* one && two is populated pushed into the `Subscribe` "r" parameter

And after that the color of the renderer is set to green or red, depending on whether r is true of false.

Note that if you change the initial lines to
```csharp
AppState.ButtonModel.SelectTwo.CombineLatest(
   AppState.ButtonModel.SelectOne
```
this makes no difference whatsoever. You can also combine more than two properties. Just pick one to calls the CombineLatest, and add the rest as parameters:

```csharp
AppState.ButtonModel.SelectOne.CombineLatest(
  AppState.ButtonModel.SelectTwo, SomeOtherPropety, YetAnotherProperty,
  (one, two, someOther, yetAnother) => one && two && someOther &&
   yetAnother).Subscribe(...)
```

## A little schematic 
Basically, it all boils down to this:
![](/assets/2020-11-20-Model-driven-Mixed-Reality-apps-using-UniRx-and-a-MRKT-extension-service,-part-1/schema.png)
* UI element asks MRKT state service reference
* UI element accesses the state service, gets the model the and sets a value to a property
* Some other UI element is subscribed to that property and gets notified
* There is *never* direct connection between independent UI components 

## Conclusion
This on only a very basic introduction to UniRx and MRTK services for making Model Driven Mixed Reality apps, and why this is such a useful approach. I anticipate to be blogging quite some more about UniRx. You can find the [code for this project here](https://github.com/LocalJoost/unirxmodeldrivenmr/tree/blog1).

## Thanks
I would not be writing this without fellow MVP [András Velvart](https://twitter.com/vbandi) - who had [wrestled with the same problem earlier](https://github.com/vbandi/MVPToolkit-Unity) and pointed me to UniRx. I am building partially on his ideas but took it in a somewhat different direction.