---
layout: post
title:  "Converting battery powered Somfy blinds to wired connection"
date: 2024-03-21 11:23:00 -0700
tags: diy
---

* TOC
{:toc}

<!-- References -->

[somfy]: https://www.somfysystems.com/en-us/products/shades-blinds-curtains/motorized-blinds-shades
[power]: https://www.amazon.com/gp/product/B08C513N9P
[cable]: https://www.amazon.com/gp/product/B08JV8QJ9L
[adapters]: https://www.amazon.com/gp/product/B07C61434H
[jst-connectors]: https://www.amazon.com/gp/product/B06XRBKSY2
[wire-clips]: https://www.amazon.com/gp/product/B07BKMVYNZ

<!-- -->

## Intro

A couple of years ago I decided to convert my battery-powered automatic [Somfy][somfy] blinds to a wired connection.

Why not use batteries? Well despite what the blinds salesperson claimed (duh!) the batteries don't last that long. With a twice-daily use (open in the morning, close in the evening) the battery "wand" lasted for only a few months. 

The blinds are very high up and not convenient to access at all. Going through the process of rearranging the furniture, setting up a ladder and mucking with the "wand" to replace the batteries while perched on it quickly became an incredible annoyance.

Conversion turned out to be not that hard conceptually but of course required lots of tedious work running the wire all the way up to the blinds.

This article describes what I did in hope that it might help somebody in the same situation.

Note that most of it is only applicable if you blinds are powered with a 12V battery wand that looks like this:

![wand](/images/somfy/wand.jpg){: width="600" style="display:block; margin-left:auto; margin-right:auto"}

If you have a different wand or a different connector you _might_ be able to use the same ideas but you are on your own.

## General idea

In order to replace the battery wand with a wire I needed:

A power supply that produces 12V output. I wanted a single power supply to power all 3 blinds (the windows are next to each other). I preferred a power supply that sits on the floor rather than trying to hide it inside the wall somehow. First any power supply is a fire risk, even if small, so I wanted to have an easy access to it to be able to see how it is doing. Second, I might decide to add something else to be powered by it in the future. Third, aesthetically having a "brick" on the floor was fine in my case since it is hidden behind some furniture and there are already some power strips and other equipment in that area.

A wire to run from the power supply to the blinds. I decided to run the wire on the outside along the windows edges, not inside the wall. Partly because I wanted to avoid the hassle of setting up wall plates and partly because my windows have a big inside "edge" that hides the cables rather well. I also wanted a separate wire from the power supply to each window not a single wire that splits 3 ways. With separate wires I can disconnect and reconnect each window individually.

A way to connect the wire to Somfy motor. Most long DC wires either come without a connector or have cylindrical connectors like this

![cylindrical connector](/images/somfy/wire-connector.jpg){: width="250" style="display:block; margin-left:auto; margin-right:auto"}
  
while Somfy motors have a 2 Pin JST connector like this 

![2-pin male jst](/images/somfy/2-pin-male-jst.jpg){: width="250" style="display:block; margin-left:auto; margin-right:auto"}
  
Some kind of adapter that needs to be small to fit on top windowsill edge was needed here.

## Parts

For power supply I ended up getting [this one][power] which **I do not recommend**. Its AC cable doesn't sit tightly in its socket and disconnects under slightest provocation. You'd be better off getting a different one.

At the time I bought it it came with a splitter cable like this

![splitter](/images/somfy/splitter.jpeg){: width="250" style="display:block; margin-left:auto; margin-right:auto"}

but I see that now, at the time of this writing, this is no longer the case. If your DC adapter doesn't have a splitter you can easily find one online. Search for "dc power splitting cable"

Knowing the end result I would probably use a different cable but originally I wasn't sure how exactly I am going to run it and how long it needs to be so I ended up getting [this][cable]:

[![cable](/images/somfy/cable.jpg){: width="250" style="display:block; margin-left:auto; margin-right:auto"}][cable]

If I were to redo it today I would get a plain cable without connectors, measure the amount I need and add the connectors myself.

To connect this cable to the motor I ended up using [these adapters][adapters]

[![adapters](/images/somfy/adapters.jpg){: width="250" style="display:block; margin-left:auto; margin-right:auto"}][adapters]

though I wish I could find ones in white color to blend better with the window edge. The windows are high and usually nobody sees their tops so this is ok.

Finally to connect the adapters to 2 Pin JST of the motor I used [these][jst-connectors]

[![JST connectors](/images/somfy/jst-connectors.jpg){: width="250" style="display:block; margin-left:auto; margin-right:auto"}][jst-connectors]


You can easily split their ends and insert them into the cylindrical adapters above. I wish these also came in white but I couldn't find any other colors. It seems that the main market for JST 2-pin connections are in printed board electronics where bright colors are the norm. Make sure to use _female_ connectors since the motor has a male end.

## Putting it all together

First visualize connectors sex. In my case it looked like this:

![schema](/images/somfy/schema.png){: width="500" style="display:block; margin-left:auto; margin-right:auto"}

That is, the female cable connector is at the floor level and the male on top. You will need the female cylindrical connector and female JST connector for the motor.

Build the motor connector. Make sure to connect <font color="red">red</font> JST cable to the ⊕ terminal of the adapter. 

Set up the power brick, lay out the cable but don't fasten it yet. Use painter tape or something similar to hold its top in place near the motor.
Disconnect the battery wand from the motor. Depending on how the installers arranged the cables this could be easy or cumbersome.

Connect the power cable instead. Test the blind to see that everything works. If you connected something wrong this is a great time to fix it before everything is fully attached.

Once everything is clear you can then fasten the cable to the wall using something like these [wire clips][wire-clips]:

![wire-clips](/images/somfy/wire-clips.jpg){: width="250" style="display:block; margin-left:auto; margin-right:auto"}

Now you can remove the power wand (don't forget to dispose of the batteries inside) and you are done.

Hope it helps!













