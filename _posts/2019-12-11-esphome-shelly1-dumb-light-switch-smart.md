---
title: Making a dumb light switch smart with Shelly1 and ESPHome and Home Assistant
date: 2019-12-11 7:00:00 -0800
categories: [Project]
tags: [ESPHome, Home Assistant]
seo:
  date_modified: 2019-12-11 09:10:00 -0800
---

## Project: Self-contained Hue light switch with offline fail-over 

### Goals

1. Create a light switch that is decoupled from power delivery so the 9 Hue Bulbs in my Dining Room Chandelier can always be powered, while allowing use of the light switch on/off paddle.
2. Provide a fail-over mechanism that allows the switch to operate even when Home Assistant is unavailable.

### Goal #1: Make the dumb light smart

#### Concept

I started with the following code fragment which uses the Home Assistant API to toggle the lights on or off when the toggle is flipped.

```yaml
script:
  - id: hass_light_toggle
    then:
      # Have Home Assistant toggle the light.
      - homeassistant.service:
          service: light.toggle
          data:
            entity_id: ${hass_light}

binary_sensor:
  - platform: gpio
    pin:
      number: GPIO5
      inverted: True
    name: ${upper_devicename} Button
    id: button
    filters:
      - delayed_on: 10ms
      - delayed_off: 10ms
    # Light switch is up.
    on_press:
      - script.execute: hass_light_toggle
    # Light switch is down.
    on_release:
      - script.execute: hass_light_toggle
```

#### Problem

This worked pretty well, but after a several rounds of: repeatedly toggle the switch, restart home assistant, toggle some more… I learned that sometimes the light switch toggle wouldn’t change the light state (it would stay on or off on the first toggle).

#### Solution

I settled on a reasonably simple solution of turning the relay back on, and turing the light on in Home Assistant if the relay was off when the switch was next toggled.

```yaml
# Shelly 1 Power Module
#
substitutions:
  devicename: shelly1_01
  upper_devicename: Shelly1 01
  hass_light: light.dining_room

esphome:
  name: ${devicename}
  platform: ESP8266
  board: esp01_1m
  board_flash_mode: dout

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_pass

# Enable Home Assistant API
api:

ota:
  safe_mode: True

logger:

binary_sensor:
  - platform: gpio
    pin:
      number: GPIO5
      inverted: True
    name: ${upper_devicename} Button
    id: button
    filters:
      - delayed_on: 10ms
      - delayed_off: 10ms
    on_press:
      # Have Home Assistant toggle the light.
      - homeassistant.service:
          service: light.toggle
          data:
            entity_id: ${hass_light}
    on_release:
      # Have Home Assistant toggle the light.
      - homeassistant.service:
          service: light.toggle
          data:
            entity_id: ${hass_light}

switch:
  # Relay is for internal use only. Do not expose to Home Assistant.
  - platform: gpio
    id: relay
    pin: GPIO4
    restore_mode: ALWAYS_ON
```

### Goal #2: Fail-over

Luckily ESPHome provides a condition for checking if Home Assistant is connected: `api.connected`.

```yaml
if:
  condition:
    api.connected:
  then:
```

Next, we check to see if the relay is on, and if it is off turn it on. We need the relay to be on before Home Assistant can turn our Home Assistant controlled smart lights on (in this case Phillips Hue).

> **Note**
> 
> In theory it should always be on. However there is one scenerio, which will be revealed soon, in which the relay would be off.

```yaml
    - if:
        condition:
          switch.is_off: relay
        then:
          - switch.turn_on: relay

          - homeassistant.service:
              service: light.turn_on
              data:
                entity_id: ${hass_light}
```

If the relay is already on then ask Home Assistant to toggle the light on or off.

```yaml

    else:
      - homeassistant.service:
          service: light.toggle
          data:
            entity_id: ${hass_light}
```

Now we come to the "fail-over" bit. This is the *else* condition for when `api.connected` is false, meaning that ESPHome has lost connection to Home Assistant!

```yaml
else:
  - switch.toggle: relay
```

This code fragment simply turns the Shelly1's internal relay from *on to off* or *off to on* when the physical light switch is toggled.

### Solution

Replace the `script:` section in the above code with this code fragment to enable to switch to continue to turn the lights on and off when Home Assistant is not available.

```yaml
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
                      entity_id: ${hass_light}
              else:
                # Have Home Assistant toggle the light.
                - homeassistant.service:
                    service: light.toggle
                    data:
                      entity_id: ${hass_light}
        else:
          # When HA is unavailable, toggle the relay.
          - switch.toggle: relay
```

### Feedback

Thank you for visiting my new blog! This is my first article, and I would really appreciate your feedback. Please stick around as I will be
 posting articles at least once a week for the next few months. I already have ideas set aside for 29 more articles on 
 [Home Assistant](/tags/home-asssistant/) and [ESPHome](/tags/esphome/), not to mention potential for tips on creating designs for laser cutters!

To hold you until my next article, visit my ESPHome Github Repository.

https://github.com/brianhanifin/esphome-config
