---
title: "RGB Shades"
date: 2015-06-21T12:22:22Z
---
A few months ago I received my pair of [RGB Shades](https://www.kickstarter.com/projects/macetech/rgb-led-shades) from [@macetech](https://twitter.com/macetech). Basically a pair of shutter shades with 68 addressable LED's and an Arduino based micro-controller.

Out of the box it ships with a few pretty cool animations, but doesn't respond to any external inputs such as sound which is what I really wanted. Inspired by a few other LED / EL projects I set about using the MSGEQ7 in a similar fashion to [this article](http://makezine.com/2014/07/18/hacking-the-macetech-rgb-shades/) on makezine.

{{< image alt="Prototyping on a breadboard" src="breadboard.jpg" >}}

What I found pretty quickly was that with the limited tools I had to my disposal packaging the electronics up in a tidy form factor was going to be somewhat more difficult.

The next thing I tried was doing the audio analysis in software, but the 16Mhz controller struggled to do anything useful in real-time. I'm now using a [Teensy 3.1](https://www.pjrc.com/teensy/teensy31.html) which has substantially more memory and runs at 96Mhz. It also has a tiny form factor at just 35x18mm.

I also found a max9814 electret mic + pre-amp combo from [Adafruit](https://www.adafruit.com/products/1713). Rather than me trying to pack up the DIN package pre-amp I was using this board uses surface mount equivalents and I don't need to try and awkwardly solder SMD components.

Once everything was working and tested on a breadboard I packaged the whole thing up in neat little form factor using a bit of strip-board. All up the finished size is around 38x22x26mm - small enough to easily fit in your pocket.

{{< image alt="Packaged up with striboard." src="striboard.jpg" >}}

I removed the original controller off the side off the glasses and the ribbon cable goes and is directly driving the LED's. Overall I think that since you needed a battery pack in your pocket anyway, the end result is cleaner with no controller on the glasses and just a single cable that I can run down the inside of my shirt.

<style>.embed-container { position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%; } .embed-container iframe, .embed-container object, .embed-container embed { position: absolute; top: 0; left: 0; width: 100%; height: 100%; }</style><div class='embed-container'><iframe src="https://www.youtube.com/embed/T2LpZwZxsmo?rel=0&amp;showinfo=0" frameborder="0" allowfullscreen></iframe></div>

I'll go into some more detail around the software side of the visualization in the video and share the code in another post soon.

Parts list:

* Teensy 3.1 - £19.80 from [proto-pic](http://proto-pic.co.uk/teensy-3-1/)
* MAX9814 - £6.72 from [proto-pic](http://proto-pic.co.uk/electret-microphone-amplifier-max9814-with-auto-gain-control/)
* Random bits of stripboard and jumper wire I had lying around.

*You can get most of these parts cheaper if you're in the US or have the patience for them to ship internationally.*


