---
layout: post
title: Fixing HoloLens 2 app build failure indicating il2cppFileRoot.txt and LineNumberMappings.json do not exist
date: 2023-11-11T00:00:00.0000000+01:00
categories: []
tags:
- HoloLens 2
- Unity
- UWP
- DevOps
published: true
permalink: 
featuredImageUrl: 
comment_issue_id: 464
---
This seems to be a very rare case, and one that I have not been able to reproduce. It mostly seems to happen when you upgrade from one major Unity release to another. You generate the C++ application from Unity, then build the app using Visual Studio, and at the very end, it fails with something along the lines of:

Il2CppOutputProject\Source\il2cppOutput\Symbols\il2cppFileRoot.txt does not exist
Il2CppOutputProject\Source\il2cppOutput\Symbols\LineNumberMappings.json does not exist

There is very little to find online; about the only real source of information I found was [this one, from June 2023, in the Unity forums](https://forum.unity.com/threads/uwp-build-crashing.1454554/#post-9140083). Basically, the solution is: edit the "Unity Data.vcxproj" and remove the entries describing those files. This, of course, does not work when you make a fresh build or run into this in a CI/CD pipeline, which was exactly how I learned about this when I upgraded [Augmedit](https://www.augmedit.com/)'s Lumi to a new major Unity solution.

The short version: if you are not interested in the how and why and want this issue fixed and you want it fixed *now*, just download [this file](https://www.schaikweb.net/blog/UWPBuildFixer.cs), put it somewhere in your project, build the project again and be done with it forever.

## Fixing Unity Data.vcxitems

So the problem is: Unity generates links in the Unity Data.vcxitems to files that are simply not there. And as the solution suggested in the Unity forums says, we can do without. So after my colleague [Niek](https://www.linkedin.com/in/niek-eichner-07b79443/) brought the Unity `PostProcessBuild` attribute to my attention, off I went:

```csharp
[PostProcessBuild(1)]
public static void FixVcxItemsFile(BuildTarget target, string pathToBuiltProject)
{
   if (target == BuildTarget.WSAPlayer)
   {
      var vcxItemsFile = 
         Directory.GetFileSystemEntries(pathToBuiltProject, 
            "Unity Data.vcxitems",
            SearchOption.AllDirectories).FirstOrDefault();
      if (vcxItemsFile != null)
      {
         FixVcxItemsFile(vcxItemsFile);
      }
   }
}
```

When the build is done, Unity calls methods in static classes decorated with `PostProcessBuild`. The number indicates the order in which they are to be called, should you have more than one, but since we don't, that number is inconsequential. The method gets a build target and the path where the project is built to as parameters. We then search for the file "Unity Data.vcxitems" and if we find it, we are going to fix it.

The offending lines are pretty long and look like this, at least in Lumi:

```xml
<None Include="$(MSBuildThisFileDirectory)..\Il2CppOutputProject\Source\il2cppOutput\Symbols\il2cppFileRoot.txt">
  <DeploymentContent>true</DeploymentContent>
  <ExcludeFromResourceIndex>true</ExcludeFromResourceIndex>
</None>
<None Include="$(MSBuildThisFileDirectory)..\Il2CppOutputProject\Source\il2cppOutput\Symbols\LineNumberMappings.json">
  <DeploymentContent>true</DeploymentContent>
  <ExcludeFromResourceIndex>true</ExcludeFromResourceIndex>
</None>
```

And they sit inside an "ItemGroup" element. So the trick is to load the document into an XML processing API, find all "None" elements inside the project group that have an Include attribute that ends with either "il2cppFileRoot.txt" or "LineNumberMappings.json". So I asked CoPilot to write me an algorithm, that compiled great and even ran beautifully, but unfortunately didn't *do* anything - and it also did this very inefficiently. But at least it showed me what API to use in this context, and I came to the following code:

```csharp
private static void FixVcxItemsFile(string vcxItemsFile)
{
   var projectFile = XDocument.Load(vcxItemsFile);

   var itemsToDelete = projectFile.Descendants().
      FirstOrDefault(node => node.Name.LocalName == "ItemGroup")?.
      Descendants().Where
      (node => node.Name.LocalName == "None" &&
               (node.IncludeAttributeEndsWith("il2cppFileRoot.txt") ||
                node.IncludeAttributeEndsWith("LineNumberMappings.json")));
   if (itemsToDelete != null)
   {
      foreach (var item in itemsToDelete.ToList())
      {
         item.Remove();
      }

      projectFile.Save(vcxItemsFile);
   }
}
```

So it does exactly what I just wrote - it finds offending items, then tells them to delete themselves, and saves the remaining.

`IncludeAttributeEndsWith` is a simple extension method that I wrote because the Linq statement is already complex enough:

```csharp
private static bool IncludeAttributeEndsWith(this XElement element, string contents)
{
   var attr = element.Attribute("Include");
   if (attr == null) return false;
   return attr.Value.EndsWith(contents);
}
```

And that's all. Should you ever run into this strange error, you can get around it by using this little helper.