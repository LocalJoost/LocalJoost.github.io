---
layout: post
title: Snap Spectacles 2024 - my take after a month of spare time development
date: 2025-01-11T00:00:00.0000000+01:00
categories: []
tags:
- Spectacles
- devices
published: true
permalink: 
featuredImageUrl: https://LocalJoost.github.io/assets/2025-01-11-Snap-Spectacles-2024--my-take-after-a-month-of-spare-time-development/spectacles.png
---
## How it started

December 15, 2024, was a kind of an unusual day for me. Not only did I get to meet my old friend [Jesse McCulloch](https://www.linkedin.com/in/jessebmcculloch/) (who I had not seen in almost 5 years), but he also had a package for me - a Snap Spectacles 2024. After a month of playing around with it and developing for it in my spare time, I feel I have something to say about it. For more technical data and review - well, you can find them at several places on the internet (my dear friend and former colleague [Alexander Meijers](https://www.linkedin.com/in/alexandermeijers/) wrote [an excellent review](https://xrpowerhero.com/the-next-generation-wearables-for-the-industry-are-here/), for instance). I just wanted to write down my own experiences and hunches about it.

![](/assets/2025-01-11-Snap-Spectacles-2024--my-take-after-a-month-of-spare-time-development/spectacles.png)

Let me be honest: when I was introduced to SnapChat back in 2017 (I think) I thought it to be a silly and pointless application. Apparently, kids think it's fun to send short takes to their friends that make it look like they are barfing rainbows or having dog ears. For me, that has about 5 minutes of novelty value. During the Covid years, they gained fame (or notoriety) with the Snap Camera - that could make you wear a silly hat or mustache during a video call, famously leading to the "[Potato Boss](https://time.com/5813683/boss-turns-herself-into-a-potato/)" who got her face changed into a talking potato, and, having no idea how to turn it off, had to do a whole Teams meeting looking like a talking potato. Clever software, but once again - not something that was on my radar as a professional XR developer. For the same reason, I never paid attention to the earlier Spectacles either, filed them under "Google Glass-alike", and went on with life.

## Wait, what?

I guess it's no surprise for you to read that when I unpacked the 2024 Spectacles, I was still pretty skeptical. Then I got to play with it, first a tutorial, then some apps (pardon, 'lenses'). The next half hour was one of the weirdest experiences in my life. What I expected to be a silly half-assed device at best, appeared to be a fully functional, WaveGuide-based, see-through Mixed Reality device, capable of hand tracking, gesture recognition, spatial mapping, spatial audio, spatial anchoring, scene recognition, even object recognition... the whole kit and caboodle that made HoloLens such an amazing device almost a decade ago, with a few additional tricks up its sleeve. 

I was like - *what the actual [censored]*? Why couldn't Microsoft build this? Or Meta, or Apple for that matter? And *Snap Inc* could? A company known for projecting silly images on faces? I felt like expecting to walk into a Dollar Store to see some Christmas crackers - and instead, bumping into an actual orbital rocket ready to take off. I have no idea what they have been doing there in Santa Monica, but flying under the radar, they must have been putting the money earned with SnapChat to good use, quietly amassing a lot of talent and knowledge, and doing a *lot* of R&D and product design. Building a fully functional WaveGuide Mixed Reality device is no simple matter, my friends. To be able to cram one into what amounts to thick but pretty normal looking glasses instead of a bulky headset is quite an accomplishment. And then actually bringing it to market (hello Meta Orion?)... to say I was surprised is like saying "Mount Everest is a pretty steep hill".

![](/assets/2025-01-11-Snap-Spectacles-2024--my-take-after-a-month-of-spare-time-development/imagery.png)

## It's not all roses

Okay, okay, let's not get carried away here. This is not a HoloLens 2, but it comes a *lot* closer than I ever would have imagined possible. But it would not be fair just to skip its limitations and drawbacks. So let me address those:

* Its field of view is 46 degrees diagonal, which is a nice way of saying "pretty small".
* Battery life: 45 minutes. That is shorter than any device on the market.
* It can handle considerably fewer triangles than, for instance, HoloLens 2. I have no exact numbers yet, but when I tried to rebuild (and succeeded in doing so) my very first XR app, the super simple "CubeBouncer", I had to reduce the number of cubes to prevent the app from crashing. So you are not going to display a lot of objects, or huge complex models in it.
* The Spatial Map is generated using computer vision alone - no Time Of Flight sensor. This works surprisingly well, is blazing fast - it seems to scan the environment and refresh the spatial map *multiple times per second* - but it is a bit less accurate. It feels like it estimates the wall and floor a bit too close, and my dark purple carpet seems to be 10cm (4") higher than it actually is. But that's software, and I presume that can be improved.
* It does not run WebXR apps, which took me as a bit of a surprise, as I would have expected it to do so. Although to be really honest, WebXR does not get half the love it deserves to have from any of the headsets builders these days.
* I have not looked in depth yet, but I have the feeling the 3D objects drift a bit more than in, for instance, HoloLens 2, but I have to look into more detail to have a definitive opinion on that.
* The subscription model of €110/$99 per month looks cheap, but like with every subscription, my proverbial Dutch frugality kicks in and does the math. After one year, the minimum subscription, you are talking €1320/$1180. After two years you are nearing 5 times the price of a Quest 3 128GB. By no stretch of imagination is this a cheap device.
* The biggest issue: it comes with a bespoke development environment - Lens Studio - that looks deceptively like Unity, but also differs considerably at points. It's actually a pretty good environment, but code is written in JavaScript or TypeScript and it has bespoke APIs and libraries. Effectively this means that apart from some textures, models, and my general knowledge of 3D concepts, I can reuse *nothing at all* from my 8 years of Unity development. Not a single line of code, none of the helper classes I wrote over the years, none of the great frameworks like MRTK3 and Reality Collective's Service Framework - not to mention the countless packages and assets you can download from the asset store. Also, you cannot tap into the years and years of institutional knowledge on the internet. You start from scratch building apps, pardon, *lenses* for it. And initially, it's back to school. All the things I do automatically in Unity, without even thinking anymore - require thinking and tinkering, which can be pretty frustrating sometimes.

## However...

* The smallish FOV is, very cleverly, almost square, even a bit higher than wider, therefore preventing the 'mail slot' view HoloLens 1 was infamously criticized for. This actually gives a much better experience than you would expect based upon the numbers alone.
* The 'holograms' (no idea how Snap calls these) are displayed *very* sharp, and there is hardly any translucency. As a long-time HoloLens fan, I have to admit Spectacles' hologram display is clearly, and notably better.
* In addition, you can turn the glass tint darker or lighter, and it can also do this automatically. So if you step outside, it can dim the light of, for instance, the sun and you can still see your holograms well. This looks a bit like the dimming feature on Magic Leap 2.
* The limited battery life can be mitigated by attaching a power bank to the device via a USB cable, to keep it topped up, and apparently, it's acceptable to have a 'puck' on a cable attached to a headset these days (right Magic Leap? Right Apple?). But unlike the Apple Vision Pro, that pack is hot swappable as the Spectacles have their own internal battery, and if you keep rotating the packs, it can in theory run forever.
* It comes with built-in capabilities for sharing sessions. This is notoriously hard to build and has caused quite some headaches for me in the past.
* It comes with a companion app for your phone that connects over Bluetooth, and that's how you control the device settings. No messing around with settings menus in the device itself, typing in network passwords with a virtual keyboard, and stuff like that.
* This companion app also includes a spectator view. Start the app, point your Spectacles at the phone's display until it sees the tracker image displayed on the screen - boom, you have a third-person view on your phone. This is really genius and makes demoing and instructing users so much easier.
* Having a bespoke development environment actually has a few advantages: like Apple, Snap Inc controls every bit of the stack, both hard- and software, and if something needs to be fixed, Snap Inc can ship updates very fast (and have done so). I can't count number the times I cursed Unity for yet another 'insignificant update' that broke a crucial function or hosed performance in HoloLens, having to downgrade to the last working version, and hoping Their Serene Highnesses would have the grace to fix it in some future version.

## So... verdict?

Let me be honest - I am still confused about this device. I love its capability, but I still can't fathom the why. Why does Snap Inc build and bring to market what amounts to a pretty high-end WaveGuide Mixed Reality device? Who is the intended customer? How will they make money off it? How do they incentivize the developer community? There must be a plan. The technology has matured a bit, so they probably had to invest less than the billions Microsoft allegedly invested in HoloLens - but still the money plowed into this must be a staggering amount.

Can you make money from it as an Indie developer? I have no idea. I work on business XR applications for a living, I don't have to live from whatever I field myself, and I consider those apps as finger exercises, resume items, or just things I do for fun, exploring avenues my work doesn't bring - and generating raw materials for my blog, too.

As to business applications: I think this is certainly viable for training applications. For starters, you don't look like a giant dorky bug wearing this - like with a Quest 3 or an Apple Vision Pro. Whether the imagery is stable enough to be able to work in professional industry or healthcare applications is open for debate and subject to more testing, but for a consumer-facing part of a business system (think of a patient seeing a 3D model of his insides, a customer a new car or piece of furniture, a bunch of C-level executives a new prototype) I think it's certainly viable. If it weren't for the lack of Unity support, I would already have tried. Having a bespoke development environment has its advantages, as I wrote, but it also comes at a high price. Even Apple, (in)famous for their closed environment, and promoting their own Swift-based development environment, supports (or allows) Unity. Grudgingly, no doubt, but still.

There are other nuts to crack, like: how am I going to distribute 'lenses' to business devices? Can I manage devices? Stuff like that. But I do know one thing: if there ever comes a day this device or a successor runs OpenXR apps from Unity, I will deploy some business apps on it the day after that, and if it holds together running them so, I think it will sell like hot cakes.

Whatever Snap Inc is planning for the future, in the midst of tech giants fielding big expensive headsets, they came out of left field, they completely blindsided me with this device, but they definitely made their mark and staked their claim. As an XR professional, seeing my beloved HoloLens slowly being relegated towards history, I *very* much welcome that.

P.S. No code this time. Expect that soon in the next posts :)