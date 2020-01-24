---
title: "Inovelli Z-Wave Dimmer Status LED in Home Assistant"
date: 2020-01-21 19:45:00 -0800
categories: [Project]
tags: [Home Assistant, Jinja, Z-Wave]
seo:
  date_modified: 2020-01-24 15:00:00 -0800
---

<div style="float: right"><img src="//ir-na.amazon-adsystem.com/e/ir?t=brianhanifi0d-20&l=am2&o=1&a=B07S1BMMGH"><img src="//ws-na.amazon-adsystem.com/widgets/q?_encoding=UTF8&MarketPlace=US&ASIN=B07S1BMMGH&ServiceVersion=20070822&ID=AsinImage&WS=1&Format=_SL160_&tag=brianhanifi0d-20"></div>Recently I ordered [Inovelli (Red Series) Z-Wave Dimmer][amazon-inovelli-dimmer] from Amazon. I primarily got it because for the ability to take actions based on up to 5 taps up (and down). I set it up to toggle one lamp with one tap, and toggle all three lamps with a double tap.

This dimmer has a really slick led strip embedded. As it turns they can preform animations and change colors. I am experimenting with using it as a secondary status notifier. So far I'm using Cyan to indicate an unlocked front door and Purple to indicate an open Garage Door.

Unfortunately the LED feature requires some mathmatical gymnastics to pull off. Luckly I found [a discussion on the Inovell forum][discuss] which provided me with two amazing resources. 1.) @nathanfiscus's amazing [Inovelli Toolbox][calc] and 2.) Inovelli's [Google Spreadsheet][spreadsheet]. I ended up taking the calculations from the spreadsheet and converting them into Jinja code to do the calculations.

Anyway, here is the script I created to control the Status LED.

## Tip

If you get an error with the `entity_id` @flyingsubs suggests you may need to shorten the name of your `zwave.{really_long_light_switch_name_and_model_blah_blah_blah}` entity. Apparently too long of an entity_name can cause a problem.

## Scripts

{% gist 9dcac14f7b05d7ccb62383626eac5a21 %}

## Automations

{% gist 7464297a1e7a96839cc439695968bcca %}

[amazon-inovelli-dimmer]: https://www.amazon.com/gp/product/B07S1BMMGH/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=B07S1BMMGH&linkCode=as2&tag=brianhanifi0d-20&linkId=e9eab63e3bd22d91d96bc880228508a1

[calc]: https://nathanfiscus.github.io/inovelli-notification-calc/
[discuss]: https://community.inovelli.com/t/home-assistant-2nd-gen-switch-rgb-working/168/62
[spreadsheet]: https://docs.google.com/spreadsheets/u/1/d/1SGJrJHCUtz8AzznWL_mLCTJjjr2U0IpltcUkRr7N_6M/edit?usp=sharing
