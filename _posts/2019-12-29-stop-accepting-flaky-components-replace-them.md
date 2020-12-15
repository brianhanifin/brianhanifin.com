---
title: Smart Home Confidence Crisis
date: 2019-12-29 09:15:00 -0700
categories: [General]
tags: [Home Assistant]
seo:
  date_modified: 2020-12-15 09:00:00 -0700
# sticky: true
---

## How to stop accepting flaky components, and start replacing them

A few months ago [I posted a message in the Home Assistant Community](https://community.home-assistant.io/t/138066)
outlining the problems that were driving down the acceptance of my Home Automation hobby.

If 9 out of 10 devices work as they should, that 10th flaky device, which I think works most of the time but
sometimes doesn't work, is the thing my family remembers. My family has gone from being pleasantly accepting 
of most things (and some automations have delighted everyone) to becoming more and more annoyed.

Examples of automations that delight:

* Kitchen cabinet LEDs light up when someone enters the room.
* Notifications when the garage door is closed.
* Colorful lightshow in our 9 bulb dining room chandelier.

Examples of flakiness that irritates:

* Drip watering controller: was battery powered and her plants have died too many times from it dying without
me realizing.
* Bedroom light switch: just wouldn't turn off almost every evening for about a month!

Anyway, my family approval of my hobby was awfully low. I couldn't blame them as I find those things annoying
as well... especially when they kill one of our outdoor plants.

There were several things stopping me from replacing components like these.

* **Stubbornness.** I don't feel like I should have to replace a thing. It is supposed to work, so it should
just work better!
* **Cost.** I don't want to replace one expensive thing, with another thing.
* **Uncertainty.** Who knows if the thing I replace it with will work any better?

I was finally able to convince myself the money I was saving for a CNC machine would be better spent on
making our smart home less frustrating and ... well, less brain dead.

### Making a list, checking it twice

I made a list of things that are not working reliably so I could tackle them one at a time. Here is my list
of problems and the solutions I implemented.

| Problem | Solution |
| ------- | ---------- |
| Unreliable Wink Hub | Replace Wink Hub with Z-Wave USB Stick and Lutron Pro Hub |
| Smart Smoke Detectors, Wink only | Replace with Z-Wave equivelent |
| Unreliable Lutron switch | Replace Wink with a Lutron Pro Hub |
| Unreliable watering controller | Install an outlet and build my own ESPHome based controller (In Progress) |
| Node-Red automations | I am currently migrating my Node-Red code to native Home Assistant code so I have one place to maintain my code. |

The advantage of the Lutron Pro hub is it allows Home Assistant to intercept the Pico Remote's button press
events. Here is how I took advantage of that feature to create Lutron Pico "Scene" remotes.

* **Dining Room**: The primary 4 buttons control Hue bulbs. The center button toggles a secondary Family Room
floor lamp which previously had no physical switch.
* **Play Room**: I was controlling Insteon lamp modules with my wall mounted Pico Remote. I decided to simplify
my system and replaced all Insteon lamp modules with refurbished white Hue bulbs. The center button controls
a secondary floor lamp which previously had not physical switch.

### Conclusion

My family and I are much happier now that everything is working as expected much more of the time! I can spend
more time create new automations, gadgets, and articles. Let me know if you have had the courage to replace
unreliable modules in the comments below.
