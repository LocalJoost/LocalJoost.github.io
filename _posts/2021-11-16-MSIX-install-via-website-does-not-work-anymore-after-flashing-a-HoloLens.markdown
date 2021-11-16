---
layout: post
title: MSIX install via website does not work anymore after flashing a HoloLens
date: 2021-11-16T00:00:00.0000000+01:00
categories: []
tags:
- HoloLens2
- MSIX
- AppInstaller
featuredImageUrl: https://LocalJoost.github.io/assets/2021-11-16-MSIX-install-via-website-does-not-work-anymore-after-flashing-a-HoloLens/appinstaller%20error.png
comment_issue_id: 393
---
## "Failed to launch ms-appinstaller..."
For some reason you had to flash your HoloLens 2. For instance,  because you lived on the edge and [it started to misbehave after an Insider's update](https://docs.microsoft.com/en-us/hololens/hololens-insider#known-issue---some-users-may-encounter-an-update-failure-with-insider-build-203461466). Or for some other reason - it happens. No sweat here, but at [Velicus](https://velicus.nl/) we distribute apps to our customers using a QR code pointing to a website hosting our MSIX. And after flashing we - and our customers - started to hit the following issue: the appinstaller did not start at all after tapping the install button.

My colleague [Timmy Kokke](https://twitter.com/sorskoot) suggested to look into the Edge developer tools and then we got to see the most peculiar error:

![](/assets/2021-11-16-MSIX-install-via-website-does-not-work-anymore-after-flashing-a-HoloLens/appinstallererror.png)

"**Failed to launch ms-appinstaller:?source= [..]because the scheme does not have a registered handler**". Say *WHAT*? The *appinstaller* scheme is unknown?

## Workaround/solution

After struggling with this for some time I contacted my friend [Matteo Pagani](https://twitter.com/qmatteoq), who had one suggestion that did not work, and then one that - weirdly enough - did work!

The workaround is as follows:
* Connect the HoloLens to your computer
* Drag an msix - *any msix* - on your HoloLens using the File Explorer, for instance in the "Downloads" folder

![](/assets/2021-11-16-MSIX-install-via-website-does-not-work-anymore-after-flashing-a-HoloLens/downloads.png)

* Open the File Explorer on the *HoloLens*, navigate to the Downloads folder

![](/assets/2021-11-16-MSIX-install-via-website-does-not-work-anymore-after-flashing-a-HoloLens/20211116_183509_HoloLens.jpg)

* Tap the MSIX until the app installer opens

![](/assets/2021-11-16-MSIX-install-via-website-does-not-work-anymore-after-flashing-a-HoloLens/20211116_183526_HoloLens.jpg)

* You can now cancel the AppInstaller.

You have now 'repaired' your HoloLens 2 AppInstaller. If you now go back to your MSIX hosting website, open the app's website, and hit the install button, the AppInstaller will work as it should. No more missing registered handlers. We have done this on two HoloLenses, and it seems to be consistent. Apparently starting one MSIX registered the scheme, and now your HoloLens can install from the web again.

Whatever Microsoft are paying Matteo, it can't be enough ;)