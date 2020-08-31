---
layout: post
title: Generic MRKT2 Extension Service Inspector
date: 2020-08-30T00:00:00.0000000+02:00
categories: hololens2 mrtk2 unity3d windows-mixed-reality
featuredImageUrl: https://LocalJoost.github.io/assets/2020-08-30-Generic-MRKT2-Extension-Service-Inspector/ServiceDisplay.png
comment_issue_id: 358
---
I make extensive use of the MRTK2 extension service feature and its dependency injection capabilities. Unfortunately it's not easy to see the internal state of such a service in the Unity editor while it's running, unless you are willing to write a custom inspector for it using the Unity Editor API. This API is, to put it mildly, not very intuitive. Writing editor UIs is a cumbersome and error-prone process. If your extension service grows and holds more complex data, maintaining it becomes a chore as for *every single field* you have to add UI code. 

## The lazy developer's approach

So I started thinking: wouldn't it be possible to write something that just iterates over the properties of a service using reflection and draw a UI on the fly? And then using the property type info to determine a field type - and fall back to a `ToString()` value in a text field. It turned out I could... although I had to solve the slight matter of recursively diving into child objects. I use a rather primitive approach to determine whether a property is a complex type or a simple type: if its `ToString()` value equals its type's `FullName`, it's a complex type and I have to dive into render it's properties in a nested way. If not, it's just a field. That works for *almost* everything. QED.

![](/assets/2020-08-30-Generic-MRKT2-Extension-Service-Inspector/ServiceDisplay.png)

## Using BaseGenericServiceInspector
If you create a new service and opt for creating a Service Inspector, you get handed a class that looks like this:
```csharp
#if UNITY_EDITOR
using System;
using Microsoft.MixedReality.Toolkit.Editor;
using Microsoft.MixedReality.Toolkit;
using UnityEngine;
using UnityEditor;

namespace YourNameSpace
{    
    [MixedRealityServiceInspector(typeof(IYourService))]
    public class YourServiceInspector : BaseMixedRealityServiceInspector
    {
        public override void DrawInspectorGUI(object target)
        {
            YourService service = (YourService)target;
            
            // Draw inspector here
        }
    }
}
#endif
```
In stead of adding inspector fields yourself, literally all you have to do is:
* Add `using MRTKExtensions.ServiceExtensions.Editor` on top
* Delete the `DrawInspectorGUI` override
* Replace the `BaseMixedRealityServiceInspector` base class by  `BaseGenericServiceInspector`


## The heart of the matter

Basically, in `BaseGenericServiceInspector`, this routine does most of the work:

```csharp
private void RenderObjectFields(object target)
{
    if (target == null)
    {
        return;
    }
    foreach (var prop in target.GetType().GetProperties(
        BindingFlags.Public | BindingFlags.Instance))
    {
        if (prop.Name == nameof(BaseExtensionService.ConfigurationProfile))
        {
            continue;
        }
        var propVal = prop.GetValue(target);

        if (propVal?.ToString() == propVal?.GetType().FullName ||
            prop.PropertyType.GetCustomAttribute(
                   typeof(InspectorExpandAttribute)) != null)
        {
            keyCounter++;
            RenderFoldout(prop.Name, () =>
            {
                using (new EditorGUI.IndentLevelScope())
                {
                    RenderObjectFields(propVal);
                }
            }, keyCounter.ToString());
        }
        else
        {
            DrawField(prop.Name, propVal);
        }
    }
}
```
It simply iterates over every property, and draws a field for it. Unless the property is an object with more properties - or when the object is decorated with a `[InspectorExpand]` attribute.

## Actually drawing fields
Now this is pretty easy in itself. There's actually two methods drawing fields. One draws most of the common fields, falling back to a `ToString()` value in a text field for none-specific fields or text:
```csharp
protected void DrawField(string name, object propVal)
{
    // Check if there's a custom field drawer first
    if (DrawCustomField(name, propVal))
    {
        return;
    }

    switch (propVal)
    {
        case bool boolVal:
            EditorGUILayout.Toggle(name, boolVal,
                Array.Empty<GUILayoutOption>());
            break;
        case Vector2 v2Val:
            EditorGUILayout.Vector2Field(name, v2Val,
                Array.Empty<GUILayoutOption>());
            break;
        case Vector3 v3Val:
            EditorGUILayout.Vector3Field(name, v3Val,
                Array.Empty<GUILayoutOption>());
            break;
        case Color colorVal:
            EditorGUILayout.ColorField(name, colorVal,
                Array.Empty<GUILayoutOption>());
            break;
        default:
            EditorGUILayout.TextField(name, propVal.ToString(),
                Array.Empty<GUILayoutOption>());
            break;
    }
}

public virtual bool DrawCustomField(string name, object propVal)
{
    return false;
}
```
The `DrawCustom` method allow you do make draw more specific fields in addition to the five types I implemented. This is particularly useful for custom frameworks. 

## Showing nested objects and properties

The weirdness of the Unity editor UI API really shows in `RenderFoldout`. I stole - and adapted - this from `BaseMixedRealityProfileInspector` and I engaged in quite a bit of cargo cult programming before I had an idea what was actually going on. As you can see in `RenderObjectFields` (up above), when it encounters a complex object, it calls `RenderFoldout` with call to itself as an `Action` but now with the property value (holding an object) as parameter. 

To understand how RenderFoldout works, it's vital to understand the UI is *not drawn once*, and only then when it's updated - this code is *continually* called. Time and time again the UI is rendered in the editor. And you really have to store state - any state - elsewhere.  the good writers of `BaseMixedRealityProfileInspector` used  the '`SessionState`' object object for that, and I dutifully copied that approach.

```csharp
private void RenderFoldout(string title, Action renderContent, string preferenceKeyPostFix)
{
    EditorGUILayout.BeginVertical(EditorStyles.helpBox);

    var preferenceKey = $"{preferenceKeyPrefix}_{preferenceKeyPostFix}";
    var storedState = SessionState.GetBool(preferenceKey, false);
    var currentState = EditorGUILayout.Foldout(storedState, title, true,
                    MixedRealityStylesUtility.BoldFoldoutStyle);

    if (currentState != storedState)
    {
        SessionState.SetBool(preferenceKey, currentState);
    }

    if (currentState)
    {
        renderContent();
    }

    EditorGUILayout.EndVertical();
}
```
So what happens is:
* A key is created to look into the `SessionState`, which apparently is a key-value bag that keeps it state over UI render calls
* The value of the key, determining if the foldout is expanded or not, is retrieved - with false as default value
* The FoldOut is rendered. Mind you, only the header with the arrow. *Not* the contents. The state parameter apparently only determines whether the arrow next to the label is pointing down (open) or right (closed). 
* By some magic the fact whether or not the user has clicked to open or close the FoldOut magically appears in `currentState`. By what event - beats me.
* The updated state is stored back into the `SessionState` so it can be used in the next render call
* And if `currentState` is true, the actual contents are rendered. This is an action, and if you look back it's simply `RenderObjectFields`'s call to itself.

## [InspectorExpand]'s use and usage

I made a passing remark already: I use a rather crude approach as to determining whether or not a property is a complex object that merits nested rendering of it's properties. In the [demo project](https://github.com/LocalJoost/GenericServiceInspector), whose inspector display is featured in the image on top, I created a convoluted structure of objects to show off it works. 

In the `ToStringOverriddenObject` I had to add the `[InspectorExpand]` attribute because I intentionally have overridden the `ToString()`. If it weren't for the `[InspectorExpand]` it would look like this:

![](/assets/2020-08-30-Generic-MRKT2-Extension-Service-Inspector/InspectorExpand.png)

Commenting out the attribute will demonstrate this issue - and why this attribute is useful.

## Conclusion

Although this code is editor code and by it's very nature will never run on and actual HoloLens, I think it might be a useful addition if you are using MRKT2's Extension Services. I might even make it into a pull request one day ;). I hope people who actually understand writing Editor code - like my esteemed colleague - won't have to laugh to hard about this. 

Demo project can be found [here](https://github.com/LocalJoost/GenericServiceInspector).