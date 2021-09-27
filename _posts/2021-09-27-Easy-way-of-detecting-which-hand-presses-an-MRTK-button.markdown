---
layout: post
title: 'Easy way of detecting which hand presses an MRTK button '
date: 2021-09-27T00:00:00.0000000+02:00
categories: []
tags:
- HoloLens2
- MRKT2
- Unity3d
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2021-09-27-Easy-way-of-detecting-which-hand-presses-an-MRTK-button/eventhookup.png
comment_issue_id: 391
---
Some UI elements cannot be neutral to the fact whether or not someone is right-handed or left-handed. For instance, if someone is left handed, you might want to place the props a user needs to choose from  on the left side of the user, in stead of default taking the right side. Inclusivity is the word these days, after all. Of course, you can make the user express an explicit choice for right or left handedness - but I decided to see if I could infer this from what hand the user uses to operate the main menu. This menu was only buttons, and those buttons are pretty neutral to left- or right handedness.

Turned out, this is *really* easy. I devised a little script that you can drag on any odd MRKT button, that exposes a click event just like the ordinary button click - but with a parameter: the hand that was used to tap the button 

## A wee bit of code

Like I said, it's really easy to do so I give the whole script in one go:
```csharp
[RequireComponent(typeof(Interactable))]
public class HandedClickEventController : MonoBehaviour,
                                          IMixedRealityTouchHandler
{
    private Handedness lastTouch;
    public UnityEvent<Handedness> OnClick = new UnityEvent<Handedness>();
    
    private void Start()
    {
        GetComponent<Interactable>().OnClick.AddListener(HandleClick);    
    }
    
    public void OnTouchStarted(HandTrackingInputEventData eventData)
    {
        lastTouch = eventData.Handedness;
    }

    public void OnTouchCompleted(HandTrackingInputEventData eventData)
    {
    }

    public void OnTouchUpdated(HandTrackingInputEventData eventData)
    {
    }

    private void HandleClick()
    {
        OnClick.Invoke(lastTouch);
    }
}
```
The key trick here is: the script implements `IMixedRealityTouchHandler` and this receives a OnTouchStarted event. This event 1) exposes the hand that was used to touch and 2) fires *before* the `Interactable`, that's on the button, fires *its* click event. The script retains the last touch event, and passes that on to it's own (typed) `OnClick` event when an `Interactable`'s `OnClick` event is detected.

## Demo script

The script that shows this actually works, is even simpler:
```csharp
public class MenuController : MonoBehaviour
{
    [SerializeField]
    private TextMeshPro displayText;
    
    public void ButtonOnePressed(Handedness handedness)
    {
        displayText.text = $"Button 1 pressed - {handedness}";
    }
    
    public void ButtonTwoPressed(Handedness handedness)
    {
        displayText.text = $"Button 2 pressed - {handedness}";
    }
}
```

## Hooking it up

All that's left now is hooking up the `HandedClickEventController`'s click event to the `MenuController`'s methods

![](/assets/2021-09-27-Easy-way-of-detecting-which-hand-presses-an-MRTK-button/eventhookup.png)

And Bob's your uncle.

## Proof of the pudding

<iframe width="650" height="365" src="https://www.youtube.com/embed/w2zTr7hMEzU" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


The demo project can be found [here](https://github.com/LocalJoost/HandPressDetection).