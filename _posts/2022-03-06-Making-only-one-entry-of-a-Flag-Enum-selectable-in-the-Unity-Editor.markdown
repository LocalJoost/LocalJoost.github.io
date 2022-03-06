---
layout: post
title: Making only one entry of a Flag Enum selectable in the Unity Editor
date: 2022-03-06T00:00:00.0000000+01:00
categories: []
tags:
- HoloLens2
- MRTK2
- Unity3d
- Windows Mixed Reality
featuredImageUrl: https://LocalJoost.github.io/assets/2022-03-06-Making-only-one-entry-of-a-Flag-Enum-selectable-in-the-Unity-Editor/maskselector.png
comment_issue_id: 411
---
Another small trick, easily reusable. Attentive users might have noticed something odd in my `recent blog post about Scene Understanding`. You can assign a Surface Type to a Layer. But a the `SpatialAwarenessSurfaceTypes` 
has the [`Flags`] attribute - so this is a *bit mask*, and by default Unity draws an editor for a bit mask like this:

![](/assets/2022-03-06-Making-only-one-entry-of-a-Flag-Enum-selectable-in-the-Unity-Editor/maskselector.png)

This is not what I wanted - I did not want the developer to select more than one layer. So I used a little trick that  allows the user to select only one entry from the bit mask:

![](/assets/2022-03-06-Making-only-one-entry-of-a-Flag-Enum-selectable-in-the-Unity-Editor/singleselect.gif)

To make this possible, the field that's used to store the surface type in has a special attribute

```csharp
[Serializable]
public class SpatialUnderstandingLayer
{
    [SerializeField,SingleEnumFlagSelect(EnumType = typeof(SpatialAwarenessSurfaceTypes))]
    [Tooltip("The surface type to act on")]
    private SpatialAwarenessSurfaceTypes surfaceType
```

This attribute, `SingleEnumFlagSelect`, is defined like this:

```csharp
public class SingleEnumFlagSelectAttribute : PropertyAttribute
{
    private Type enumType;

    public Type EnumType
    {
        get => enumType;
        set
        {
            if (value == null)
            {
                Debug.LogError($"{GetType().Name}: EnumType cannot be null");
                return;
            }
            if (!value.IsEnum)
            {
                Debug.LogError($"{GetType().Name}: EnumType is {value.Name} this is not an enum");
                return;
            }
            enumType = value;
            IsValid = true;
        }
    }
    
    public bool IsValid { get; private set; }
}
```

This accepts an `Enum` and `Enum` only, as you can see. And this attribute triggers this little editor script:

```csharp
[CustomPropertyDrawer(typeof(SingleEnumFlagSelectAttribute))]
public class SingleEnumFlagSelectAttributeEditor : PropertyDrawer
{
    public override void OnGUI(Rect position, 
      SerializedProperty property, GUIContent label)
    {
        var singleEnumFlagSelectAttribute = 
          (SingleEnumFlagSelectAttribute)attribute;
        if (!singleEnumFlagSelectAttribute.IsValid)
        {
            return;
        }
        var displayTexts = new List<GUIContent>();
        var enumValues = new List<int>();
        foreach (var displayText in 
          Enum.GetValues(singleEnumFlagSelectAttribute.EnumType))
        {
            displayTexts.Add(new GUIContent(displayText.ToString()));
            enumValues.Add((int)displayText);
        }

        property.intValue = EditorGUI.IntPopup(position, label, property.intValue,
            displayTexts.ToArray(), enumValues.ToArray());
    }
}
```

This replaces the default editor behavior by a single select popup with the display names of the bit masked `Enum` as texts, and their corresponding `int` value as value. Net result: you can only pick one value, exactly as I wanted.

I kind of nicked the idea from the `PhysicsLayerAttributeDrawer` class [in the MRTK](https://docs.microsoft.com/en-us/dotnet/api/microsoft.mixedreality.toolkit.physics.editor.physicslayerattributedrawer?view=mixed-reality-toolkit-unity-2020-dotnet-2.7.0) that does the same for layers, but for layers only. This is basically generically applicable to every flagged `Enum`

The code is [in the Scene Understanding project from two blog posts ago](https://github.com/LocalJoost/SpatialUnderstandingDemo).
