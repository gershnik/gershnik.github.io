---
layout: bare
exclude: true
sitemap:
  exclude: 'yes'
title:  "Tiny Intercom"
description: "Tiny Intercom iOS app"
image: images/intercom.png
date: 2025-11-19 11:00:00 -0700
---

Repurpose an old, unused iPhone/iPad into an audio intercom. Install the app on both your regular phone and the unused one, put the unused device into a remote room in your house and you will be able to talk to whoever is in that room from your phone. No more yelling or walking over!

* TOC
{:toc}

## Usage

The app can run in two modes: a terminal and a controller. The terminal is what you put in a remote room. In this mode the app runs continuously (turning the screen off is ok) and transmits the sound in and out. 
The controller is for your regular phone or tablet. In this mode you can connect to terminals to listen and talk through them. 

You can have as many terminals and controllers as you wish. Multiple controllers can connect to the same terminal at the same time (and speak together) and a controller can connect to multiple terminals (but not at the same time)

Features and requirements:

- Audio only. This app does not support video.
- All devices must be on the same local network to communicate. 
- All communications between controllers and terminals are encrypted
- In order to connect to a terminal a controller must first be paired to it. Pairing code is displayed on the terminal and can be scanned as a QR code or entered manually. Pairing code can be reset if desired.
- There is no “cloud” or “server”. All communications happen solely between controllers and terminals locally on your network and nothing is ever sent outside of it.

### Tips

* You can use [Apple Configurator](https://support.apple.com/guide/apple-configurator-2/welcome/mac) to lock your terminal device into a “single app mode” (aka kiosk mode) to make it run just the intercom terminal and not allow to switch out of it. A good guide on how to do so can be found [here](https://www.howtogeek.com/252670/how-to-put-an-ipad-into-kiosk-mode-restricting-it-to-a-single-app/)

* You can change your controller or terminal name from the default (your iPhone device name) to whatever you wish in app Settings.



## Support

Please report bugs and suggestion to <a id="maillink" data-mail="firmest.privets.34" data-domain="icloud.com" href="#">this e-mail</a>

## Privacy Policy

Tiny Intercom app doesn't collect, store or share any of your data and doesn't track your usage in any way.

If you email the developer for support or other feedback, the emails with email addresses will be retained indefinitely. The email addresses will be used only to reply to the concerns or suggestions raised and will never be used for any marketing purpose.

<script>
let maillink = document.getElementById('maillink')  
maillink.innerHTML = maillink.getAttribute('data-mail')+'@'+ maillink.getAttribute('data-domain')
maillink.href = 'mailto:' + maillink.getAttribute('data-mail')+'@'+ maillink.getAttribute('data-domain')+'?subject=Tiny Intercom Issue'
</script>



