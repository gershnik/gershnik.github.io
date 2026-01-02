---
layout: bare
exclude: true
title:  "Tiny Intercom"
description: "Tiny Intercom app"
image: images/intercom.png
date: 2025-11-19 11:00:00 -0700
---

Repurpose an old, unused iPhone/iPad/Android device or even a Mac (Apple Silicone only) into an audio intercom. Install this app on both your regular devices and the unused one, put the unused device into a remote room in your house and you will be able to talk to whoever is in that room from your phone. No more yelling or walking over!

* TOC
{:toc}

## Basics 

- There is no "cloud" or "server", no accounts to create and no service to sign up for. All communications happen locally, on your network, between your devices. Nothing is ever sent outside of your network. Nobody can connect to your devices from outside (and that includes yourself when you are not on your home network). This is an "intercom", not a general chat service. 
- Connecting to a device requires pairing using an access code that is only visible on the device. 
- All devices must be on the same local network to communicate. 
- All communications between devices are encrypted. 
- The app is audio only. There is no support for video and no plans to add it.
- The app is free (as in beer) and there are no in-app purchases or ads of any kind. 


The app can run in two modes: a terminal and a controller. The terminal is what you put in a remote room. In this mode the app runs continuously (turning the screen off is ok) and transmits the sound in and out. 

The controller is for your regular phone, tablet or Mac. In this mode you can connect to terminals to listen to and talk through them. 

You can have as many terminals and controllers as you wish. Multiple controllers can connect to the same terminal at the same time (and speak together) and a controller can connect to multiple terminals (but not at the same time)

In order to connect to a terminal a controller must first be paired to it. Pairing code is displayed on the terminal and can be scanned as a QR code or entered manually. Pairing code can be reset if desired.


## Usage

Specific platform usage notes below:

### iOS and macOS

The app is available from AppStore: [https://apps.apple.com/us/app/tiny-intercom/id6755145518](https://apps.apple.com/us/app/tiny-intercom/id6755145518)

Both iPhone and iPad are supported. The iPad version can also be run on Apple Silicone Macs.
(watchOS companion app is coming soon!)

The app on the AppStore supports iOS 10.3 or above. It can run on iPhone 5s or above and iPad Air (1st generation)/iPad Mini 2 or above. 

The name the app uses to identify the controller or terminal can be changed in either of these ways:
- In iOS Settings for the Tiny Intercom app
- On the terminal screen directly (tap the pencil icon next to the device name)

You can use [Apple Configurator](https://support.apple.com/guide/apple-configurator-2/welcome/mac) to lock your terminal device into a “single app mode” (aka kiosk mode) to make it run just the intercom terminal and not allow to switch out of it. A good guide on how to do so can be found [here](https://www.howtogeek.com/252670/how-to-put-an-ipad-into-kiosk-mode-restricting-it-to-a-single-app/)

### Android

The app APK can be downloaded [from this link](https://www.gershnik.com/apps/Intercom/Intercom.apk). You need to manually install it (download on your device and open).

Currently, due to Google's hostile policies for independent developers[^1], there are no plans to make the app available on the Play Store.

Minimum Android version supported is 23 (Marshmallow). The app supports arm64-v8a, armeabi-v7a, x86_64 and x86 hardware architectures. The app doesn't use or depend on any Google services.

When running in a terminal mode it is important to make Android not turn off the Wi-Fi for the entire device or the app after a period of inactivity. How to do so depends on Android version. On more recent ones you need to:

- Go to Settings > Apps > See all apps and find Tiny Intercom
- Tap Battery and select Unrestricted.

On Pixel devices you will also need to:
- Go to Settings > Network & internet.
- Select Adaptive connectivity (on the bottom of the screen)
- Toggle it OFF.

On earlier Android versions there is a "Keep Wi-Fi on during sleep" toggle under Settings > Wi-Fi > Advanced.

If you don't have any of these settings or your terminal keeps disconnecting from Wi-Fi you will need to consult your device documentation and/or internet search and/or friendly AI to find how to prevent network disconnect.

## Support

Please report bugs and suggestion to <a id="maillink" data-mail="firmest.privets.34" data-domain="icloud.com" href="#">this e-mail</a>. 
All bug reports, comment and suggestions, constructive or not, are welcome! 

## Privacy Policy

Tiny Intercom **app** doesn't collect, store or share any of your data and doesn't track your usage in any way.

If you **email** the developer for support or other feedback, the emails with email addresses will be retained indefinitely. The email addresses will be used only to reply to the concerns or suggestions raised and will never be used for any marketing purpose.

This **website** (not the app!) uses Google Analytics to understand how visitors engage with it. This information will also never be
used for any marketing purpose. 

Google Analytics own privacy policy is available at [How Google uses information from sites or apps that use our services](https://policies.google.com/technologies/partner-sites)

You can prevent Google Analytics from recognizing you on return visits to this site by disabling cookies in your browser or by installing the [Google Analytics Opt-out Browser Add-on](https://tools.google.com/dlpage/gaoptout)

<script>
let maillink = document.getElementById('maillink')  
maillink.innerHTML = maillink.getAttribute('data-mail')+'@'+ maillink.getAttribute('data-domain')
maillink.href = 'mailto:' + maillink.getAttribute('data-mail')+'@'+ maillink.getAttribute('data-domain')+'?subject=Tiny Intercom Issue'
</script>


---

## Footnotes

[^1]: Google now requires developers to have [12 testers run the app for 14 days](https://support.google.com/googleplay/android-developer/answer/14151465) for new apps and _any major updates_. This is a free app made by a single developer. I have no 12 testers available and no intention to pay for one of many services that proliferated after this requirement was made.