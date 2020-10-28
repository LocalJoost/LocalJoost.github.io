---
layout: post
title: Generic MRKT2 Extension Service Inspector for MRKT 2.5
date: 2020-10-25T00:00:00.0000000+02:00
categories: []
tags:
- HoloLens2
- MRKT2
- Unity3d
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2020-10-25-Generic-MRKT2-Extension-Service-Inspector-for-MRKT-25/afterupgrade.png
comment_issue_id: 360
---
If the title of this blog post seems a bit familiar, you are right. About two months ago [I proudly wrote](https://localjoost.github.io/Generic-MRKT2-Extension-Service-Inspector/) about a Generic MRTK2 extension service inspector I created. This allowed you to see the inner status of your extension service in the editor. And then MRKT 2.5 was released and the whole thing came crashing down because apparently someone decided Service Inspectors where not a thing anymore. The code that created the infrastructure on which my code piggybacked was deleted or made defunct, and in stead of a representation of my DemoService I only saw this:

![](/assets/2020-10-25-Generic-MRKT2-Extension-Service-Inspector-for-MRKT-25/afterupgrade.png)

If you close the DemoService node, you can see it's now only an empty "ServiceFacade"

![](/assets/2020-10-25-Generic-MRKT2-Extension-Service-Inspector-for-MRKT-25/afterupgrade2.png)

And all the other service 'nodes' inside the Mixed Reality are just empty shells now too. 

## You can complain....
As I wrote two months ago, I use extension services extensively - pun not intended - so this was kind of a problem for me, not in the least because this inspector is playing an important role [in my upcoming talk for the Global XR Bootcamp 2020](https://www.globalxrbootcamp.com/events/model-driven-mixed-reality-apps/). So I did a little investigating on how the service inspector in the MRTK2.4 where implemented, and kind of had to agree the way the where implemented could have some pretty nasty side effects indeed. 

## ... or you can fix it
I was able to lift my code off the BaseMixedRealityInspector class and turn it into a proper *UnityEditor*. While I was at it, I also dramatically improved it, so that it now also could show list contents and the properties of objects in that list. It felt a bit like [when I brought Behaviors back into Windows 8 apps](https://localjoost.github.io/attached-behaviors-for-windows-8-metro/), way back in the beginning of me being an MVP ;)

## How it works now
The MRTK2.4 used the principle of the ServiceFacade, but I did not dare to simply write a new editor for that. The whole Service Inspector idea is apparently deprecated, presumably the ServiceFacade as well, and I don't want to build on top of something that might disappear. Better make a self-contained solution. And I did just that on that. 

If you now want to see the contents of a service, you simply do the following:
* Create an empty GameObject somewhere in the scene (I would advise the root)
* Give it the name of the service you want to see the name of. This should be the name (without namespace) of the class implementing the service
* Add my new ServiceDisplayHook behaviour to it

And then you are done. The result, for instance, is something like this:
![](/assets/2020-10-25-Generic-MRKT2-Extension-Service-Inspector-for-MRKT-25/fullservicedisplay.png)

## The ServiceDisplayHook
I proudly present you the simplest behaviour in the world ;)  
```csharp
using UnityEngine;

namespace MRTKExtensions.ServiceExtensions
{
    public class ServiceDisplayHook : MonoBehaviour
    {
    }
}
```
It *literally does nothing*. It's only a placeholder to trigger Unity in using the new ServiceDisplayEditor, which remarkably looks like the old inspector.

## The ServiceDisplayEditor
Since I already described most of the inner workings of the editor in the previous post, I am just going to point you to some interesting new parts. First of all - the class definition. As I said, it's not a BaseMixedRealityInspector child anymore, but it now derives from a Unity Editor. And it's now a simple CustomEditor. Hence the need for the `ServiceDisplayHook` behaviour - it simply needs to have something you can select - and this will trigger it to show the innards of an extension service:
```csharp
namespace MRTKExtensions.ServiceExtensions.Editor
{
    [CustomEditor(typeof(ServiceDisplayHook))]
    [ExecuteAlways]
    public class ServiceDisplayEditor : UnityEditor.Editor
    {
```

## Starting point for UI drawing
The start point for any editor UI of course is the `OnInspectorGUI` method. This simply gets the editor target - which is of course a ServiceDisplayHook because that's in the `CustomEditor` attribute - and finds a service with the same name as the (empty) game object that it is sitting in.
```csharp
public override void OnInspectorGUI()
{
    if (service == null)
    {
        var hook = target as ServiceDisplayHook;
        if (hook != null)
        {
            serviceName = hook.gameObject.name;
            if (MixedRealityToolkit.IsInitialized)
            {
                service = MixedRealityServiceRegistry.GetAllServices()
                    .FirstOrDefault(p => p.Name == serviceName);
            }
        }
    }

    if (service != null)
    {
        DrawInspectorGUI(service, service.GetType().FullName);
    }
    else
    {
        DrawHeader($"No service with name {serviceName} found");
    }
}
```
`service` and `servicename` are private fields. This editor is called pretty often when active, so in order not to query the actual MixedRealityService every update loop I retain those values once retrieved. And then it's simply a matter of calling DrawInspectorGUI again, only now with a header text so we can actually show the name of the service in the editor in stead of the name of the ServiceDisplayHook behaviour:

## Creating a header showing the service class name
```csharp
protected void DrawInspectorGUI(object targetObject, string header)
{
    keyCounter = 0;
    DrawHeader(header);
    RenderObjectFields(targetObject);
}
```

DrawHeader I simply stole from the MRTK2.4, thank your Microsoft:
```csharp
private void DrawHeader(string header)
{
    // Draw a rect over the top of the existing header label
    var labelRect = EditorGUILayout.GetControlRect(false, 0f);
    labelRect.height = EditorGUIUtility.singleLineHeight;
    labelRect.y -= labelRect.height - headerYOffet;
    labelRect.x = headerXOffset;
    labelRect.xMax -= labelRect.x * 2f;

    EditorGUI.DrawRect(labelRect, EditorGUIUtility.isProSkin ? proHeaderColor : defaultHeaderColor);
    EditorGUI.LabelField(labelRect, header, EditorStyles.boldLabel);
}
```
The effect is that in stead of the name of the behavior (ServiceDisplayHook) the name of the implementing *class* is displayed on top of the panel when you expand it:

![](/assets/2020-10-25-Generic-MRKT2-Extension-Service-Inspector-for-MRKT-25/implementingclass.png)

## Support for 'value' types and collections
While this was basically enough to get the thing working again, for my talk I needed more: support for types that have Value property in stead of an actual value (like nullables). I also needed to be able to show collections and their contents. So in  the DrawField method I have added two more calls. One to the new method `DrawValueType`, which simply checks if this object has a "Value" property and then passes that value back to DrawField

```csharp
private bool DrawValueType(string name, object propVal)
{
    if (propVal != null)
    {
        var propertyToFind = propVal.GetType().GetProperties().
            FirstOrDefault(p => p.Name == "Value");
        if (propertyToFind != null)
        {
            DrawField(name, propertyToFind.GetValue(propVal));
            return true;
        }
    }

    return false;
}
```

And DrawCollection, of which I am rather proud for some reason
```csharp
private bool DrawCollection(string name, object propVal)
{
    if (propVal is ICollection collection)
    {
        RenderFoldout(name, () =>
        {
            using (new EditorGUI.IndentLevelScope())
            {
                var objCount = 0;
                foreach (var obj in collection)
                {
                    keyCounter++;
                    RenderFoldout($"{obj.GetType().Name}[{objCount++}]", () =>
                    {
                        using (new EditorGUI.IndentLevelScope())
                        {
                            RenderObjectFields(obj);
                        }
                    }, keyCounter.ToString());
                }
            }
        }, (++keyCounter).ToString());


        return true;
    }
    return false;
}
```
This creates yet another foldout for the object, then a foldout for every object in the list, then renders once again the object fields.

All in all this whole editor has become a nice piece of recursive programming. 

## A final piece of weirdness
When I was preparing my talk, I noticed the UI did not get updated unless I specifically clicked on the editor pane it. Probably something that changed between Unity 2018 and 2019. Now this may or may not be a problem in debugging circumstances, but for me giving a demo, it was. So I added this rather dirty piece of code:
```csharp
private bool doRepaint;
void OnEnable()
{
    doRepaint = true;
    RepaintLoop();
}

private async Task RepaintLoop()
{
    while (doRepaint)
    {
        await Task.Delay(100);
        Repaint();
    }
}

private void OnDisable()
{
    doRepaint = false;
}
```
This forces the UI to repaint every 100ms in an endless loop. If you don't need this, simply delete it from the code as you copy it from GitHub.

## Conclusion 
Although it was a bit of nasty surprise halfway in preparing a talk, the resulting editor is a lot more powerful. Also, my new approach has the added benefit that you don't have to subclass the BaseGenericServiceInspector for every service you create - simply adding an empty game object with the right name and throwing a ServiceDisplayHook behaviour is sufficient.

I hope this will be useful to help you build and debug your Mixed Reality apps. Of course you can download the code, as always, from GitHub. I have put it the [same project as last time, in a new MRKT2.5 branch](https://github.com/LocalJoost/GenericServiceInspector/tree/MRKT2.5).
