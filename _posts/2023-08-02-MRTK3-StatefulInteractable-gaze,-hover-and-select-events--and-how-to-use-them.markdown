---
layout: post
title: MRTK3 StatefulInteractable gaze, hover and select events - and how to use them
date: 2023-08-02T00:00:00.0000000+02:00
categories: []
tags:
- MRTK3
- Eye tracking
- HoloLens2
- Unity
published: true
permalink: 
featuredImageUrl: http://localjoost.github.io/assets/2023-08-02-MRTK3-StatefulInteractable-gaze,-hover-and-select-events--and-how-to-use-them/events.png
comment_issue_id: 450
---
When my colleague [Sophie](https://www.linkedin.com/in/sophiechenyang/) asked me if I had a sample for [something I had written before](https://localjoost.github.io/migrating-to-mrtk2-setting-up-and/), but then for MRTK3, I had to admit that I did not. So I set out to *create* that sample for MRTK3 - after all, I had already used gaze interaction with MRTK3 in my cross-platform [HoloATC](https://apps.microsoft.com/store/detail/holoatc/9NVHWRS1V9ZH) app. I ended up exploring quite a few possibilities of the MRTK3 `StatefulInteractable` component and found out that there are a lot more events that can be dealt with.

![](/assets/2023-08-02-MRTK3-StatefulInteractable-gaze,-hover-and-select-events--and-how-to-use-them/events.png)

## Configuring the StatefulInteractable Component

The eye target is once again a simple sphere, now adorned with a `StatefulInteractable` component. In the "Advanced StatefulInteractable Settings" pane, I changed the settings until they looked like this:

![](/assets/2023-08-02-MRTK3-StatefulInteractable-gaze,-hover-and-select-events--and-how-to-use-them/settings.png)

Note that I intentionally set a ridiculously user-unfriendly 3 seconds of dwell time to make the differences between the events that go off immediately and the events that take some time very clear.

## Events, Events, and More Events

After configuring the basic settings, it's time to have a look at the actual events. There are quite a lot: 10 in total. As far as pure gaze-related events on `StatefulInteractable` are concerned, they fall into two categories. The first two can be found under MRTK Events/MRTKBaseInteractable Events:

![](/assets/2023-08-02-MRTK3-StatefulInteractable-gaze,-hover-and-select-events--and-how-to-use-them/mrtkevents.png)

The second category is in "XRI Interactable Events" and is a pretty long list:

![](/assets/2023-08-02-MRTK3-StatefulInteractable-gaze,-hover-and-select-events--and-how-to-use-them/xri_events.png)

As with my previous example, every event has its own red dot that turns green when the corresponding event is called, and it automatically turns red after some time (now 0.75 seconds).

The SingleShot controller that actually processes a dot's color is pretty simple:

```csharp
public class SingleShotController : MonoBehaviour
{
    [SerializeField]
    private Color activatedColor = Color.green;

    [SerializeField]
    private float resetTime = 0.75f;

    private Color originalColor;

    private Material material;

    private float timeActivated = float.MinValue;

    public void ShowActivated()
    {
        timeActivated = Time.time;
    }

    private void Start()
    {
        material = GetComponent<Renderer>().material;
        originalColor = material.color;
    }

    void Update()
    {
        var desiredColor = Time.time - timeActivated > resetTime ?
            originalColor : activatedColor;
        if (material.color != desiredColor)
        {
            material.color = desiredColor;
        }
    }
}
```

The table below shows which dots light up at what event:

| Dot Text | Event Name                   |
| -------- | ---------------------------- |
| MGHN     | MRTK Is Gaze Hovered Entered |
| MGHX     | MRTK Is Gaze Hover Exited    |
| XFHN     | XRI First Hover Entered      |
| XLHX     | XRI Last Hover Exited        |
| XHN      | XRI Hover Entered            |
| XHX      | XRI Hover Exited             |
| XFSN     | XRI First Select Entered     |
| XLSX     | XRI Last Select Exited       |
| XSN      | XRI Select Entered           |
| XSX      | XRI Select Exited            |

And then I ran a test in the editor, which resulted in the following:

![](/assets/2023-08-02-MRTK3-StatefulInteractable-gaze,-hover-and-select-events--and-how-to-use-them/BasicEvents.gif)

As you can see:

- MRTK Is Gaze Hovered Entered, XRI First Hover Entered, and XRI Hover Entered fire simultaneously.
- MRTK Is Gaze Hover Exited and XRI Hover Entered fire simultaneously.
- MRTK Is Gaze Hover Exited, XRI Last Hover Exited, and XRI Hover Exited fire simultaneously.
- MRTK Is Gaze Hover Exited, XRI Last Hover Exited, and XRI Hover Exited fire simultaneously.
- XRI First Select Entered and XRI Select Entered fire simultaneously.
- XRI Last Select Exited and XRI Select Exited fire simultaneously.

## Duplicity? Think Again...

So why are all these events here? Most of them go off at the same time. This may seem like there's quite a bit of duplicity, right? Wrong - because there are other things that can hover and select:

![](/assets/2023-08-02-MRTK3-StatefulInteractable-gaze,-hover-and-select-events--and-how-to-use-them/Advanced.gif)

What we saw here was:
- A *hand ray* activates XRI First Hover Entered, XRI First Select Entered, and (after 3 seconds) XRI Select Entered as well, but *not* MRTK Is Gaze Hovered Entered or MRTK Is Gaze Hover Exited.
- If you point to the sphere with one hand and then add a second hand, XRI Hover Entered fires at the second hand, but *not* XRI First Hover Entered.
- You can also fire the select events directly by either air tapping while a hand ray is hovering over the sphere or poking the sphere, and by saying "select".
- And by the way, you can also look at the sphere and air tap with a hand just anywhere in view of your device. You have been able to do this long before Apple announced this 'revolutionary' way of selecting things.

So why are the MRTK gaze events there at all when the XRI events cover all the events anyway? Basically, only for one reason: to be able to distinguish that the hover and select are *specifically* caused by *looking* at your target. For instance, to show some specific visual effect.

## Conclusion

So these are a lot of the events, and this is how you can use them. As always, you can find [a demo project on GitHub](https://github.com/LocalJoost/MRTK3StatefulInteractableEvents).
