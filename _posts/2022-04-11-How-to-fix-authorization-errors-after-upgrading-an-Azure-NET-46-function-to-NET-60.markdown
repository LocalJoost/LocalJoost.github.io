---
layout: post
title: 'How to fix authorization errors after upgrading an Azure .NET 4.6 function to .NET 6.0 '
date: 2022-04-11T00:00:00.0000000+02:00
categories: []
tags:
- .NET 6
- Azure
- Functions
featuredImageUrl: https://LocalJoost.github.io/assets/2022-04-10-How-to-fix-authorization-errors-after-upgrading-an-Azure-NET-46-function-to-NET-60/function1.png
comment_issue_id: 415
---
Okay, one more Azure functions post. [After upgrading my .NET 2.0 standard function](https://localjoost.github.io/Upgrading-a-NET-standard-20-Azure-function-using-Table-Storage-to-a-NET6-function/) for [AMS HoloATC](https://www.microsoft.com/store/apps/9NBLGGH52SZP) I discovered the back end function for my HoloLens app [Walk the World](https://www.microsoft.com/store/productid/9P6SVQQCP2SQ) was on an even older tech stack: .NET4.6. I actually had to upgrade the source project to 4.8 to even be able to open the project at all. 

The actual upgrade was actually was fairly easy. This was the function header:
```csharp
public static class WhateverFunction
{
    [FunctionName("WhateverFunction")]
    public static async Task<HttpResponseMessage> Run(
       [HttpTrigger(AuthorizationLevel.Function, "get", 
          Route = null)]HttpRequestMessage req,
        TraceWriter log, ExecutionContext context, 
        [Table("WhateverTable", "AzureWebJobsStorage")]
        CloudTable whateverTable)
```

And I only had apply what I [learned in my previous post](https://localjoost.github.io/Upgrading-a-NET-standard-20-Azure-function-using-Table-Storage-to-a-NET6-function/) to change that to this:

```csharp
public static class WhateverFunction
{
    [FunctionName("WhateverFunction")]
    public static async Task<IActionResult> Run(
       [HttpTrigger(AuthorizationLevel.Function, "get",
          Route = null)]HttpRequestMessage req,
        ILogger log, ExecutionContext context, 
        [Table("WhateverTable", Connection = "AzureWebJobsStorage")]
        TableClient whateverTable)
```

And I only had to change the end from 
```csharp
return req.CreateResponse(HttpStatusCode.OK, result);
```
to

```csharp
return new OkObjectResult(result);
```

And I was basically done. I tested the function locally - worked fine. Then I deployed it, started up Walk the World - and nothing happened. I then ran some test code calling the deployed function running on Azure - and I got a 401 (Unauthorized) error. Oops. And there are quite some installations of Walk the World out there being used regularly. Fortunately I could quickly revert to the old function, and that still worked.

It took me quite some time to figure out what went wrong. Something peculiar happens in the function configuration in Azure. If you work with Azure functions, I assume you are familiar with the following interface:

![](/assets/2022-04-11-How-to-fix-authorization-errors-after-upgrading-an-Azure-NET-46-function-to-NET-60/function1.png)

And if you click "Get Function URL" you get this piece of UI

![](/assets/2022-04-11-How-to-fix-authorization-errors-after-upgrading-an-Azure-NET-46-function-to-NET-60/function2.png)

To be able to call the function, you have to supply the "code=[some_long_value]" parameter as a kind of security code, to prevent everyone else from calling your function willy-nilly. 

Apparently - when I deployed my function as .NET 6.0 function, *all available values for codes changed* and the old one *was no longer valid*. 

The solution is very simple: you can add the old code as a custom function key. This is pretty easy. First, you click on "Function keys":

![](/assets/2022-04-11-How-to-fix-authorization-errors-after-upgrading-an-Azure-NET-46-function-to-NET-60/function3.png)

Then, you then click on "New function key":

![](/assets/2022-04-11-How-to-fix-authorization-errors-after-upgrading-an-Azure-NET-46-function-to-NET-60/function4.png)

And then you add the old code key back by adding a key with name "code", and for  value the code that used to be valid before the upgrade:

![](/assets/2022-04-11-How-to-fix-authorization-errors-after-upgrading-an-Azure-NET-46-function-to-NET-60/function5.png)

If you go back to the "Get Function URL" panel, you will now see five entries in stead of four:

![](/assets/2022-04-11-How-to-fix-authorization-errors-after-upgrading-an-Azure-NET-46-function-to-NET-60/functions5.png)

And you can breathe a sigh of relief, like me, for now all your existing clients can call this function using the existing code they have, and they won't notice a thing of the completely upgraded back end.

I have no idea if behavior this intentional or a bug. It feels like the latter, as this problem did not occur when I upgraded my .NET standard 2.0 function to .NET 6.0. Maybe this was caused by the jump in technology stack being a lot bigger than the previous upgrade, and maybe no-one ever tested this particular aspect of this particular transition. There might be a lesson in that: never put off upgrading for too long, or it might hurt you in unexpected ways.However, should it hit you too, this is how you can mitigate it.

No code this time, as this is about configuring the function and not about writing code.