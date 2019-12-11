---
title: Self-contained Hue light switch with offline fail-over 
date: 2019-12-11 7:00:00 -0800
categories: [Project]
tags: [ESPHome, Home Assistant]
seo:
  date_modified: 2019-12-11 07:00:01 -0800
---

## Goal

Create a light switch that is decoupled from power delivery so the 9 Hue Bulbs in my Dining Room Chandelier can always be powered, while allowing use of the light switch on/off paddle.

## Concept

I started with the following code fragment which uses the Home Assistant API to toggle the lights on or off when the toggle is flipped. Basically, if the API is connected, then have Home Assistant toggle the lights. However, if the API is NOT connected (because Home Assistant is offline currently) then use the relay to toggle power to the lights.

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
                entity_id: ${hass_light}
        else:
          # When HA is unavailable, toggle the relay.
          - switch.toggle: relay

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
      - script.execute: hass_light_toggle
    on_release:
      - script.execute: hass_light_toggle
```

## Problem

This worked pretty well, but after a several rounds of: repeatedly toggle the switch, restart home assistant, toggle some more… I learned that sometimes the light switch toggle wouldn’t change the light state (it would stay on or off on the first toggle).

## Solution

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
      - script.execute: hass_light_toggle
    on_release:
      - script.execute: hass_light_toggle

switch:
  # Relay is for internal use only. Do not expose to Home Assistant.
  - platform: gpio
    id: relay
    pin: GPIO4
    restore_mode: ALWAYS_ON
```

## More ESPHome code

If you want to see my other ESPHome code, this is my Github Repository.

https://github.com/brianhanifin/esphome-config
