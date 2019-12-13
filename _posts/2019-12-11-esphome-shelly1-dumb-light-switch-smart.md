---
title: Making a dumb light switch smart with ESPHome and Home Assistant
date: 2019-12-11 7:00:00 -0800
categories: [Project]
tags: [ESPHome, Home Assistant]
seo:
  date_modified: 2019-12-13 11:50:00 -0800
---

## Project: Smart light switch with offline fail-over

### Goals

1. Create a light switch that is decoupled from power delivery so the 9 Hue Bulbs in my Dining Room Chandelier can always be powered, 
while allowing use of the light switch on/off paddle.
2. Provide a fail-over mechanism that allows the switch to operate even when Home Assistant is unavailable.

### Purchased Supplies

1. [Shelly1][shelly1] (or a Sonoff Basic, or other ESPHome compatible board).
2. [Electrical wire][romex] (Romex 12/2, solid wire, 2 covered conductors plus one ground conductor).
3. Wire Nuts, [Non-twist Connectors][non-twist-connectors], or [Wago Connectors][wago].

*The above links include my Amazon Affilite id.*

### Goal #1: Make the dumb light switch talk to Home Assistant

I started with the following code fragment which uses the Home Assistant API to toggle the smart bulbs on or off when the toggle is flipped.

```yaml
script:
  - id: hass_light_toggle
    then:
      if:
        condition:
          api.connected:
        then:
          # Have Home Assistant toggle the light.
          - homeassistant.service:
              service: light.toggle
              data:
                entity_id: light.dining_room
        else:
          # When HA is unavailable, toggle the relay.
          - switch.toggle: relay

binary_sensor:
  - platform: gpio
    pin:
      number: GPIO5
      inverted: True
    name: Wall Switch
    id: button
    filters:
      - delayed_on: 10ms
      - delayed_off: 10ms
    on_press:
      - script.execute: hass_light_toggle
    on_release:
      - script.execute: hass_light_toggle

switch:
  platform: gpio
  id: relay
  pin: GPIO4
  restore_mode: ALWAYS_ON
```

### Goal #2: Fail-over

We need a way to check to see if Home Assistant is connected, so we can toggle the light with the relay instead. Luckily ESPHome provides a condition for checking if Home Assistant is connected: `api.connected`.

```yaml
if:
  condition:
    api.connected:
  then:
```

Next, we check to see if the relay is on, and if it is off turn it on. We need the relay to be on before Home Assistant can turn on our smart bulbs on.

> **Note**
>
> In theory the relay should always be on. However there is one scenerio, which will be revealed soon, in which the relay would be off.

```yaml
    - if:
        condition:
          switch.is_off: relay
        then:
          - switch.turn_on: relay

          - homeassistant.service:
              service: light.turn_on
              data:
                entity_id: light.dining_room
```

If the relay is already on then ask Home Assistant to toggle the light on or off.

```yaml

    else:
      - homeassistant.service:
          service: light.toggle
          data:
            entity_id: light.dining_room
```

Now we come to the "fail-over" bit. This is the *else* condition for when `api.connected` is false, meaning that ESPHome has lost connection to Home Assistant!

```yaml
else:
  - switch.toggle: relay
```

This code fragment simply turns the Shelly1's internal relay from *on to off* or *off to on* when the physical light switch is toggled.

### Final Code

```yaml
---
# Shelly 1 Power Module
# Location: Dining Room Light Switch
#
substitutions:
  project: Shelly1 01
  id: shelly1_01
  button_gpio: GPIO5
  relay_gpio: GPIO4
  hass_light: light.dining_room

esphome:
  name: $id
  platform: ESP8266
  board: esp01_1m

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_pass

ota:
  safe_mode: True

logger:

# Home Assistant API
api:

script:
  - id: hass_light_toggle
    then:
      if:
        condition:
          api.connected:
        then:
          - if:
              condition:
                switch.is_off: relay
              then:
                # Turn the relay back on and turn on the light.
                - switch.turn_on: relay

                - homeassistant.service:
                    service: light.turn_on
                    data:
                      entity_id: $hass_light
              else:
                # Have Home Assistant toggle the light.
                - homeassistant.service:
                    service: light.toggle
                    data:
                      entity_id: $hass_light
        else:
          # When HA is unavailable, toggle the relay.
          - switch.toggle: relay

binary_sensor:
  - platform: gpio
    pin:
      number: $button_gpio
      inverted: True
    name: $project Wall Switch
    id: button
    filters:
      - delayed_on: 10ms
      - delayed_off: 10ms
    on_press:
      - script.execute: hass_light_toggle
    on_release:
      - script.execute: hass_light_toggle

switch:
  platform: gpio
  id: relay
  pin: $relay_gpio
  restore_mode: ALWAYS_ON
```

### It's not a bug, its a feature

The relay can only be turned off when Home Assistant is offline when the light was turned off by the wall switch. So, when Home Assistant comes back online that smart light will be unavailable until the wall switch is flipped again.

Let's call this an intended side-effect. If we made the relay somehow turn back on automatically when it connects to Home Assistant again, the smart bulb would turn on! I can tell you from experience wives really don't like your Home Automation hobby when the bedroom lights turn on in the middle of the night!

### Special Thanks

Thank you to Mauricio Bonani of [bonani.tech][bonanitech] for encouraging me to create a blog as a knowledge dump that I can refer back to in the future. I hope others find some of my articles as useful and I have found Mauricio's articles.

### Feedback

Thank you for visiting my new blog! This is my first article, and I would really appreciate your feedback. Please stick around as I will be
 posting articles at least once a week for the next few months. I already have ideas set aside for 29 more articles on [Home Assistant](/tags/home-assistant/) and [ESPHome](/tags/esphome/), not to mention potential tips on creating designs for laser cutters!

To hold you until my next article please visit my [ESPHome Github Repository][esphome-config].

[esphome-config]: https://github.com/brianhanifin/esphome-config
[bonanitech]: https://bonani.tech/

[shelly1]: https://www.amazon.com/gp/product/B07G33LNDY?ie=UTF8&tag=brianhanifi0d-20&camp=1789&linkCode=xm2&creativeASIN=B07G33LNDY
[romex]: https://www.amazon.com/gp/product/B0069F4CXQ?ie=UTF8&tag=brianhanifi0d-20&camp=1789&linkCode=xm2&creativeASIN=B0069F4CXQ
[non-twist-connectors]: https://www.amazon.com/gp/product/B07DW1QZF5/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=B07DW1QZF5&linkCode=as2&tag=brianhanifi0d-20&linkId=b9b4a1708258bbb1a213949082a0eb84
[wago]: https://www.amazon.com/gp/product/B01N5JXOVF?ie=UTF8&tag=brianhanifi0d-20&camp=1789&linkCode=xm2&creativeASIN=B01N5JXOVF
