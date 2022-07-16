---
layout: post
title: MRTK2 to MRTK3 - a MUCH SIMPLER way to intercept a raw air tap like with IMixedRealityPointerHandler
date: 2022-07-16T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- HoloLens2
- Unity
- Windows Mixed Reality
- XR Interaction Toolkit
featuredImageUrl: https://LocalJoost.github.io/assets/2022-07-16-MRTK2-to-MRTK3--a-MUCH-SIMPLER-way-to-intercept-a-raw-air-tap-like-with-IMixedRealityPointerHandler/input1.png
comment_issue_id: 424
---
Sometimes the gods themselves descend from the Olympos - or maybe in this case, [Mount Rainier](https://en.wikipedia.org/wiki/Mount_Rainier) maybe - [to explain to you things can be done a lot simpler ;)](https://github.com/LocalJoost/BlogComments/issues/421#issuecomment-1184713692). When [I wrote this](https://localjoost.github.io/MRTK2-to-MRTK3-intercepting-a-raw-air-tap-like-with-IMixedRealityPointerHandler/) I already had the feeling there *must* be a simpler way than looking up the hand controllers, continually checking values using the update loop in a service, etc. - but I could not find it. I will cheerfully admit I am still very much in the process of tying to get to grips the Unity XR Interaction Toolkit. I can find lots of long rambling videos about it, as well as Unity's nearly sample-free API documentation - that is nearly useless, as Intellisense works just as well for showing available API calls. But complete, simple, how-to-samples? Almost none to be found. This I why I write them, as I have been doing for nearly 15 years now. I don't mind looking like an idiot in the process, doing things the long way around, as long as I can help people getting forward and get things done. 

Case in point: I got explained getting an air tap using a way simpler method, but not *how*. It took me a while searching around and piece a bit of code together. And my immediate reflex is than: make a short sample, blog it, be done with it forever and I (and the rest of the world) finally have a - hopefully - easy to find sample.

## Enough talk, let's code

After this rather long introduction to a very short piece of code: here is "Intercepting an air tap, the XR Interaction Toolkit way". That is, the first part:

```csharp
public class DirectInputAirTapDisplayer : MonoBehaviour
{
    [SerializeField]
    private GameObject leftHand;
    [SerializeField]
    private GameObject rightHand;

    [SerializeField]
    private InputActionReference leftHandReference;
    [SerializeField]
    private InputActionReference rightHandReference;
    
    private void Start()
    {
        leftHand.SetActive(false);
        rightHand.SetActive(false);
        leftHandReference.action.performed += ProcessLeftHand;
        rightHandReference.action.performed += ProcessRightHand;
    }
}
```

So I now have *four* editor configurable fields, the two new ones  are actually `InputActionReference` just as [Finn](https://github.com/Zee2) explained. What I had to find out myself, and good luck doing so: it turns out, those have an `action` property, which in turn have a `performed` event that you can trap. Which brings us to the next part:

```csharp
private void ProcessRightHand(InputAction.CallbackContext ctx)
{
    ProcessHand(ctx, rightHand);
}

private void ProcessLeftHand(InputAction.CallbackContext ctx)
{
    ProcessHand(ctx, leftHand);
}

private void ProcessHand(InputAction.CallbackContext ctx, GameObject g)
{
    g.SetActive(ctx.ReadValue<float>() > 0.95f);
}
```

The event handler gets a `InputAction.CallbackContext` parameter that allows you to read information about the event. Now if you use the `Select` input action as Finn suggest, you can't - that's just an event. I need the `SelectValue` event. And in case you have no idea what the hell I am talking about and where to find that - like my yesterday self:
* Search for "Input Actions" in Packages,

![](/assets/2022-07-16-MRTK2-to-MRTK3--a-MUCH-SIMPLER-way-to-intercept-a-raw-air-tap-like-with-IMixedRealityPointerHandler/input1.png)

* Select "MRTK Default Input Actions"
* After you have selected that, click the little cross all the way to the right in the select box to cancel the search
* Expand the MRTK Default Input Actions, then drag the "MRTK LeftHand/Select Value" node on top of "Left Hand Reference", and the "MRTK RightHand/Select Value" node on top of "Right Hand Reference"

![](/assets/2022-07-16-MRTK2-to-MRTK3--a-MUCH-SIMPLER-way-to-intercept-a-raw-air-tap-like-with-IMixedRealityPointerHandler/input2.png)

All very evident - *once you know how, and once you know this stuff exists at all*. 

Of course, I could have done this all with two little lambdas attached to the event handler, but when I see event handler attachments, the "memory leak warning" light in my brain flashes on immediately - and although I never was with the boy scouts, I like to behave like one, and leave no litter laying around:

```csharp
private void OnDestroy()
{
    leftHandReference.action.performed -= ProcessLeftHand;
    rightHandReference.action.performed -= ProcessRightHand;
}
```

## Concluding words

As I said, I am still trying to wrap my head around the XR Interaction Toolkit. I hope I don't sound condescending about or angry on [Finn](https://github.com/Zee2) for 'not properly explaining' - I hugely appreciate reaching out to me, putting me on the right track, and not beating around the bush about documentation still leaving a lot to be desired. And in any case, this is the place where I like to be most - trail blazing, finding out how - even when I sometimes take a roundabout way to get somewhere -, running into and filing bugs...  and [sometimes even fixing them](https://github.com/microsoft/MixedRealityToolkit-Unity/pull/10632) before the team themselves have found them. To boldly go where no-one (or at least very few) have gone before ;) 

(Yeah, I am Star Trek fan. Although a moderate one. You won't find me learning Klingon or dressing up like one).

Anyhew, you can find the MRTKAirTap with the `DirectInputAirTapDisplayer` [here](https://github.com/LocalJoost/MRKTAirTap/tree/using_inputreference).
