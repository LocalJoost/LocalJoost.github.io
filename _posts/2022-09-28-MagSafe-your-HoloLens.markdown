---
layout: post
title: MagSafe your HoloLens
date: 2022-09-28T00:00:00.0000000+02:00
categories: []
tags: []
featuredImageUrl: https://LocalJoost.github.io/assets/2022-09-28-MagSafe-your-HoloLens/scratch.jpg
---
In 2006, Apple introduced the concept of using powerful magnets to a connect a power supply to a device with the MagSafe connector, that made its debut with the first Intel based MacBook Pro. This concept found its way onto other devices: Microsoft, for instance, has been using magnetic power connectors since the first Surface devices in 2012. Most smaller and wearable devices come with physical connectors - MicroUSB or (lately) USB-C. HoloLens 1 sports the first one, HoloLens 2 the second. The trouble with these connectors it that they are small and relatively vulnerable. If you are not careful, a damaged power port might turn your expensive device into a paperweight. And even if you *are* careful, eventually wear and tear will take it's toll. When you are in a develop/test/deploy cycle, and especially when you are testing things that only work on a device (\*cough\* Bluetooth \*cough\*), you  easily plug and unplug a device 50 times or more a day while deploying test apps over the USB cable - this is by far the fastest option, and our apps at [Velicus](https://velicus.nl/) are easily half a gigabyte, the biggest well over 2GB.

After over two years of intensive use, your HoloLens will at least get scratched, however careful you are:

![](/assets/2022-09-28-MagSafe-your-HoloLens/scratch.jpg)

Recently, [I saw this post by Kerwin Kassulke](https://www.linkedin.com/posts/kerwink_hololens-productivity-activity-6978166583688925184-_9ZV) where he showed *magnetic USB-C* connectors. Now the tricky part with HoloLens 2 is that if you use anything but the USB-C cable that comes with the device, speed throughput will quickly drop dramatically - up to a point where deploying our 2+GB app to the HoloLens takes well over 15 minutes in stead of less than a minute - unless you find a very high quality cable. This is of course an unacceptable time loss in a development scenario, so I was a bit weary, yet decided to take the plunge and test with [this little plug I found on Amazon](https://www.amazon.nl/gp/product/B09KY8M4FM/):

![](/assets/2022-09-28-MagSafe-your-HoloLens/dynafunk.jpg)

The results were stunning. Not only power came trough at full speed, but data as well. The magnets, however small they must be, are quite strong: if you hold both parts closer together than this, they will already pull together ;)

![](/assets/2022-09-28-MagSafe-your-HoloLens/parts.jpg)

On your HoloLens, it will look like this. You can still pry it out easily if you need to, but it's stuck fairly solid - it won't drop out just like that.

![](/assets/2022-09-28-MagSafe-your-HoloLens/ports.jpg)

And it attaches very easily and solidly, as Kerwin already showed, and with this particular type, a light comes on if the connection is successful indeed:

![](/assets/2022-09-28-MagSafe-your-HoloLens/magsafe.gif)

Using something like this, which come at a price of about 0.5% of an actual HoloLens, can help a developer save time, hassle and most importantly: keep the device safe and most likely in a better state, even under heavy develop/deploy/test use. I highly recommend it, I thought it important enough to merit a separate blog post. I would recommend going for one (like this) with a high data throughput. If you are only using the HoloLens in production or for demo purposes. a bit cheaper device may work as well, but compared to the list price of a HoloLens this is penny shaving anyway.

Once again thanks to [Kevin Kassulke](https://www.linkedin.com/in/kerwink/) for this awesome tip!