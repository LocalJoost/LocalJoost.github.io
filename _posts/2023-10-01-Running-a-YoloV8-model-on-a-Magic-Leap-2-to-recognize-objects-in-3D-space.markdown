---
layout: post
title: Running a YoloV8 model on a Magic Leap 2 to recognize objects in 3D space
date: 2023-10-01T00:00:00.0000000+02:00
categories: []
tags:
- Artificial Intelligence
- ONNX
- Machine Learning
- MRTK
- Magic Leap 2
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2023-10-01-Running-a-YoloV8-model-on-a-Magic-Leap-2-to-recognize-objects-in-3D-space/YoloMLHolo.gif
comment_issue_id: 459
---
After I successfully got to run [YoloV8 models on HoloLens 2 to recognize the model aircraft I made as a teenager and locate them in space](https://localjoost.github.io/HoloLens-AI-training-a-YoloV8-model-locally-on-custom-pictures-to-recognize-objects-in-3D-space/) - using the Unity Barracuda inference engine to process the model - I thought it would be fun to try this on the Magic Leap 2 as well. The short version of this experiment is - it worked, as this 4x sped up video shows:

![](/assets/2023-10-01-Running-a-YoloV8-model-on-a-Magic-Leap-2-to-recognize-objects-in-3D-space/YoloMLHolo.gif)

However, this was ... quite a venture.

## Camera access

When I [wrote about using the Spatial Map on Magic Leap 2](https://localjoost.github.io/Using-a-Spatial-Mesh-with-MRTK3-on-Magic-Leap-2/), I was surprised to learn Magic Leap hadn't implemented ARMeshManager. However, using a simple behaviour filled that void. That was nothing compared to what I ran into when I wanted to do something I *assumed* to be easy: capturing the camera image. After all, I needed it to feed the model. On HoloLens 2, all you need to do to get a webcam camera view is this:

```csharp
var webCamTexture = new WebCamTexture(requestCameraSize.x, requestCameraSize.y, 
                                      cameraFPS);
webCamTexture.Play();
```

When I tried this on Magic Leap 2, nothing happened. I could actually retrieve cameras and camera sizes using the `WebCamTexture` API, but I did not get any image. After conferring with a Magic Leap engineer, I got more or less the same message as with the Spatial Map: this part of the Unity stuff is not implemented (yet), so I also had to use Magic Leap specific code here as well. He was kind enough to point me to the ["Simple Camera Example" in the Magic Leap 2 developer docs](https://developer-docs.magicleap.cloud/docs/guides/unity/camera/ml-camera-example/). The name is a bit misleading because there are actually two samples there. I used the simplest one of the two. That 'Simple Camera Example' is 300 lines long.

I repeat: that 'Simple Camera Example' is *300 lines long*.

It would also require me to completely rewrite the logic of the app. This, of course, would not do.

## Image acquiring service

I am not a software architect just for the showy name, so I put on my 'architect hat' and called the Reality Collective Service Framework to the rescue. If there are two or more very different APIs aiming to basically achieve the same goal, I try to define a *service* handling those differences. Instead of relying on either WebCamTexture or the Magic Leap camera API, I made myself an Image Acquiring Service, implementing one interface, with two implementations: one for HoloLens, and one for Magic Leap 2.

The interface is hilariously simple:

```csharp
public interface IImageAcquiringService : IService
{
    void Initialize(Vector2Int requestedImageSize);
    Vector2Int ActualCameraSize { get; }
    Task<Texture2D> GetImage();
}
```

The main `YoloObjectLabeler` behaviour first held all the settings and did the image processing: now the settings have all moved to service profiles and it just gets a reference to an `IImageAcquiringService`, sends it the Yolo model image size, loads the actual provided camera size, and then repeatedly calls `GetImage()` to get the latest image.

## Profile

To be able to give every implementation its own settings, I have defined a profile with the following settings:

```csharp
public class ImageAcquiringServiceProfile : BaseServiceProfile<IServiceModule>
{
    [SerializeField]
    private int cameraFPS = 4;

    [SerializeField]
    private Vector2Int requestedCameraSize = new(896, 504);

    public int CameraFPS => cameraFPS;
    
    public Vector2Int RequestedCameraSize => requestedCameraSize;
}
```

The FPS is only functional on the HoloLens 2 (or any other platform that implements `WebCamTexture`). The default value shown here is for HoloLens 2 - Magic Leap 2 actually provides lots more different camera sizes, and I have set it to 640x480, as this is closest to the 320x256 the Yolo V8 model in the app uses.

## HoloLens 2 implementation

Although this is about running it on Magic Leap 2, I wanted to show you how a WebCamTexture using implementation looks like, if only to show how simple and clear it is:
```csharp
public class ImageAcquiringService : BaseServiceWithConstructor, IImageAcquiringService
 {
     private readonly ImageAcquiringServiceProfile profile;
     private WebCamTexture webCamTexture;
     private RenderTexture renderTexture;
     
     public ImageAcquiringService(string name, uint priority, 
                                  ImageAcquiringServiceProfile profile)
         : base(name, priority)
     {
         this.profile = profile;
     }
   
     public void Initialize(Vector2Int requestedImageSize)
     {
         renderTexture = new RenderTexture(requestedImageSize.x, requestedImageSize.y, 
                             24);
         webCamTexture.Play();
     }

     public override void Start()
     {
         webCamTexture = new WebCamTexture(profile.RequestedCameraSize.x, 
                                profile.RequestedCameraSize.y, profile.CameraFPS);
         ActualCameraSize = new Vector2Int(webCamTexture.width, webCamTexture.height);
     }

     public Vector2Int ActualCameraSize { get; private set; }

     public async Task<Texture2D> GetImage()
     {
         if (renderTexture == null)
         {
             return null;
         }
         Graphics.Blit(webCamTexture, renderTexture);
         await Task.Delay(32);

         var texture = renderTexture.ToTexture2D();
         return texture;
     }
 }
```

`Start` is called by the ServiceFramework, creates the `WebCamTexture` with the requested size, then retrieves the actual size (this might differ - I can ask HoloLens for 640x480 but I won't get it - it will default to the closest camera size).

`Initialize` sets the `RenderTexture`'s initial size (320x256 for this model) and starts generating images.

And then, by calling `GetImage()` I can get the latest image from the WebCamTexture. Mind you, this generates a new `Texture2D`. It's the caller's responsibility to destroy that after it's done with it (this already was the case in the previous sample, but then it all happened in one class)

## Magic Leap 2 implementation

I will limit myself to a few snippets, as the complete service is 267 lines long. I have basically taken the "Simple Camera Sample", removed a lot of the global variables and changed them into parameters. At the top of the service, it says this:
```csharp
public void Initialize(Vector2Int requestedImageSize)
{
    this.requestedImageSize = requestedImageSize;
}

public override void Start()
{
    StartCameraCapture();
}
```

Basically the same as the HoloLens 2 implementation, only it now fires off a lot of Magic Leap specific code, instead of a `WebCamTexture.Play()`. This kicks off a whole batch of things. As far as I can see it:
* Waits for a camera to become available
* Connects to and configures the camera
* Starts the video capture and connects a listener method `OnCaptureRawVideoFrameAvailable` to the `OnRawVideoFrameAvailable`
* `OnCaptureRawVideoFrameAvailable` then calls `UpdateRGBTexture`
* This loads, for every event, the camera frame into a `Texture2D` "`videoTexture`", which is about the only global variable left. It also sets the `ActualCameraSize` (this is available only after the first frame).

The GetImage implementation for Magic Leap 2 remarkably looks like that of the HoloLens 2:

```csharp
public async Task<Texture2D> GetImage()
{
    if (videoTexture == null)
    {
        return null;
    }

    if (renderTexture == null)
    {
        renderTexture = new RenderTexture(imageSize.x, imageSize.y, 24);
    }
    Graphics.Blit(videoTexture, renderTexture);
    await Task.Delay(32);
    return FlipTextureVertically(renderTexture.ToTexture2D());
}
```

... but for one crucial detail: the image is flipped *upside down*.

So there's this final method that actually takes care of that by flipping it. [I nicked it off StackOverflow, of course](https://stackoverflow.com/questions/54623187/unity-flip-texture-using-c-sharp)

```csharp
private static Texture2D FlipTextureVertically(Texture2D original)
{
    var originalPixels = original.GetPixels();
    var newPixels = new Color[originalPixels.Length];

    var width = original.width;
    var rows = original.height;

    for (var x = 0; x < width; x++)
    {
        for (var y = 0; y < rows; y++)
        {
            newPixels[x + y * width] = originalPixels[x + (rows - y - 1) * width];
        }
    }

    original.SetPixels(newPixels);
    original.Apply();
    return original;
}
```

## Conclusion

Apart from the refactoring of the image aquistion into a service, the app itself is largely unchanged. So how does it work in real life on the Magic Leap 2? Well, it recognizes the airplanes, more or less in the right place, just like HoloLens 2. It also recognizes the DC3 Dakota, even though I never added a picture of that to the training set - but I guess this is more a Unity Barracuda accomplishment than a Magic Leap 2 one. As performance goes on this one, I would say, about on par with HoloLens 2. This surprised me, as I expected it to be *faster*, having a more beefy processor, but it actually seems to be just that bit *slower* and have a lower recognition rate than HoloLens 2.

Don't get me wrong - there's definitely room for improvement here - in my code, that is. I will not even begin to pretend I understand this device as well as I do HoloLens 2. I notice the image I get is a bit grainier than HoloLens 2 delivers. Also, the FOV of the webcam seems to be bigger, which gives some more distortion at the edges - in the sense that the 3D object and the image don't exactly overlap anymore, and therefore my rather crude approach to locating objects in 3D space based upon the location in a 2D image doesn't work that well anymore.

I am excited to see Magic Leap 2 can do computer vision in the way as well, which in the light of the current AI wave is a very important feature. In this regard, I think it would be nice if the Magic Leap SDK would implement the WebCamTexture as well. However, since this is a bit of an unusual use case, I can also imagine this having not the highest priority.

A branch of the YoloHolo for Magic Leap 2 can be downloaded [here](https://github.com/LocalJoost/YoloHolo/tree/MagicLeap2)