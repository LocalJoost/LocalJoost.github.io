---
layout: post
title: A behaviour to put MRTK buttons in a disabled state
date: 2021-12-24T00:00:00.0000000+01:00
categories: []
tags:
- HoloLens2
- MRTK2
- Unity3d
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2021-12-24-A-behaviour-to-put-MRKT-buttons-in-a-disabled-state/word.png
comment_issue_id: 400
---
Sometimes you want UI elements to show in a disabled state - indicating you can do something, but that's not available right now because the app is in a state where the UI element in not applicable, or the user has to do something first. Exhibit A: this Microsoft Word dialog. Yes, you can save the file, but you have to enter a name first:

![](/assets/2021-12-24-A-behaviour-to-put-MRKT-buttons-in-a-disabled-state/word.png)

Of course, you can also opt to not show the UI element at all, but that might cause confusion at user level, or can can leave 'holes' in your UI (think of a button bar). Since the graphical user interface emerged, grayed out parts became the standard way of conveying to the user something must happen first, or is not available. As far as I can see, something like that is not available in the MRTK, so maybe this little behaviour might be of use. 

## A little demo

This little video shows how it works. If you press the Ok button, the counter will go up. If you disable it by pressing the toggle button, it grays out and basically any form of interaction is gone. Until you press the toggle button again.

![](/assets/2021-12-24-A-behaviour-to-put-MRKT-buttons-in-a-disabled-state/buttondisable.gif)

## A little code

There is actually not much code involved. The first part of the `ButtonStatusController` class is a lot of private variables that are used to store a lot of components involved in the disabling and enabling of components:

```csharp
public class ButtonStatusController : MonoBehaviour
{
    private TextMeshPro textMeshPro;
    private Color textOriginalColor;
    private Color iconOriginalColor;
    private Renderer iconRenderer;
    private List<MonoBehaviour> buttonBehaviours;
    private Transform buttonHighLightComponent;
    private bool isInitialized = false;
```

Then we get the Awake method, that basically grabs all the component we need, as well as the current color of the text and the icon (which are white, if you have stuck to the standard, but they need not be). Since finding component is an expensive operation, those values are stored in the private variables I already mentioned, so we make sure this will happen only once and keep the result. Although I don't think Awake methods are ever called more than once, we explicitly check for that:

```csharp
private void Awake()
{
    if (!isInitialized)
    {
        isInitialized = true;

        var iconParent = transform.Find("IconAndText");
        textMeshPro = iconParent.GetComponentInChildren<TextMeshPro>();
        iconRenderer = iconParent.Find("UIButtonSquareIcon").
           gameObject.GetComponent<Renderer>();
        buttonHighLightComponent =
           transform.Find("CompressableButtonVisuals");
        buttonBehaviours = GetComponents<MonoBehaviour>().ToList();
        textOriginalColor = textMeshPro.color;
        iconOriginalColor = iconRenderer.material.color;
    }
}
```
We find a number of components by name, and extract the necessary items. the `textMeshPro` and the `iconRenderer` items are used to change colors (from default to gray), the buttonHighLightComponent is used to turn off dynamic behaviour, as is the buttonBehaviours list.

The actual dynamics are in the `SetStatus` method. This is what you might call from outside based upon some state:

```csharp
public void SetStatus(bool active)
{
    foreach (var b in buttonBehaviours.Where( p => p!= this))
    {
        b.enabled = active;
    }
    buttonHighLightComponent.gameObject.SetActive(active);
    textMeshPro.color = active ? textOriginalColor : Color.gray;
    iconRenderer.material.color = active ? iconOriginalColor : Color.gray;
}
```

When `active` = `false`, The `foreach` loop rather hamfistedly turns off all top level behaviours in the button (excluding, of course, itself). Then it turns off the components showing the visual interaction events, and finally it sets the color of both the text and the icon to gray. And then basically the only thing left of your button is a completely unresponsive square with a gray text and icon. If you set active to true, everything is brought back to life again and all display items turn back to their original color.

## Conclusion

As you can see, it's not quite rocket science. Just put this behaviour on the button you might want to disable and call the method from some controller. You just have to dig a little into how a button in constructed. This also is a weakness of this approach - if for some reason they MRKT team decides to change the component *names*, this behaviour falls of course flat on it's face. 

[Demo project can be found on GitHub.](https://github.com/localjoost/ButtonDisableDemo)
