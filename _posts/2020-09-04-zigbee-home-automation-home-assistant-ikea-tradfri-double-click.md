---
title: "IKEA Trådfri Remote: ZHA Double (and Triple) Click"
date: 2020-09-04 20:00:00 -0800
categories: [Project]
tags: [Home Assistant, Zigbee, Jinja]
seo:
  date_modified: 2020-09-04 20:10:00 -0800
image: /assets/img/2020-09-04-zha-double-click/ikea-remote.jpg
---

I recently picked up an [IKEA Trådfri Remote](https://www.ikea.com/us/en/p/tradfri-remote-control-00443130/) to try out. 
My computer is in the Living Room and I thought it would be nice to have a multi function remote to manage the fan and 
lights around me.

## 5 button remote

The remote has five buttons to do whatever I want with. Using Home Assistant this become a custom "Scene Controller".
Initially I setup the large power button in the middle to toggle the fan on and off. The top and bottom "brightness"
buttons turn the primary light group on and off, and the right and left arrow buttons turn the secondary lamp on and off.

My wife asked me if she could adjust the light brightness with the remote as well. That led me to discover that `zha_event`
captures hold events for the 4 light control buttons. A long press on one of those buttons either raises or lowers the
assigned light's brightness by 20%.

Nice! That brings me up to 9 "scenes" that I can control with this remote.

## Double click?

Well, I thought it would be nice to be able to toggle the TV on and off. After checking though `zha_event` does not
capture double taps of any of the IKEA remote's buttons. Darn.

One thought kept nagging at the back of my mind though. Although `zha_event` didn't provide a unique command for a
"double click" event, I noticed that double the center power button did fire two events in quick succession. I knew there
was probably a way to exploit that.

## How to store the information needed for later comparison?

After a few hours (and a couple of breaks) I figured it out! Storing the previous click's timestamp to compare it with the
current click's timestamp. The trick was to find a way to store the data. At first I used the custom_component 
[hass-variables](https://github.com/rogro82/hass-variables) to store the timestamp of the current click, device_ieee, 
command, and the difference in time between the previous click and the current click. While that worked great and gave my
data structure, I wanted to make it easier for everyone to use.

I created a "Helper" entity named `input_text.zha_click`. In it I store a comma separate list of the values needed.
Here is a table with some captured click data:

| device_ieee | command | previous_click | click_count | *click_delta* |
| :---: | :---: | :---: | :---: | :---: |
| ec:1b:bd:ff:fe:23:9c:ee | toggle | 1599275643.719181 | 1 | *6.6513881683349609* |
| ec:1b:bd:ff:fe:23:9c:ee | toggle | 1599276764.579644 | 2 | *0.2137620449066162* |
| ec:1b:bd:ff:fe:23:9c:ee | toggle | 1599276776.025289 | 3 | *0.16803312301635742* |

The "click_delta" has been remarked out from my current code, but it was very helpful to help me understand the timing
during testing! For example it told me 1 second not only felt too long to wait for my fan to turn on, but it also
felt like too long to wait for a second click. I found half of a second to be a pretty sweet spot between the two.

## Example Automation

Below is an example of the automation I use. In this section of the code that handles the large button in the middle.
Theoretically it can have an unlimited number of clicks. I have held it to 3 for now: 1x - Toggle Fan, 2x - Toggle TV,
3x - Toggle Front Door Lock.

<blockquote>
<i>Note:</i> I chose `mode: restart` so a second click within the allotted delay time (0.5 seconds) would cancel the default
single click action.
</blockquote>

```yaml{% raw %}
automation:
  alias: "ZHA: IKEA Remote Click Handler"
  id: zha_ikea_remote_click
  initial_state: true
  mode: restart # Force restart to pickup double click
  trigger:
    platform: event
    event_type: zha_event
    event_data:
      data.device_ieee: "ec:1b:bd:ff:fe:23:9c:ee"
  action:
    - choose:
        # Middle Button: "Power"
        - conditions:
            - condition: template
              value_template: '{{ trigger.event.data.command == "toggle" }}'
          sequence:
            # Store the previous click values and decide if this click was a double click or not.
            - service: script.zha_store_click
              data_template:
                device: '{{ trigger.event.data.device_ieee }}'
                command: '{{ trigger.event.data.command }}'

            - choose:
                # Double click
                - conditions:
                    # Click count: returned by 4th value
                    - condition: template
                      value_template: |
                        {% set click_count = states("input_text.zha_click").split(",")[3]|int %}
                        {{ click_count == 2 }}
                  sequence:
                    # Delay 0.5 second to allow for a second click to cancel the second action.
                    - delay:
                        seconds: 0.5
                    
                    - service: switch.toggle
                      entity_id: switch.tv_family_room

                # Triple click
                - conditions:
                    # Click count: returned by 4th value
                    - condition: template
                      value_template: |
                        {% set click_count = states("input_text.zha_click").split(",")[3]|int %}
                        {{ click_count == 3 }}
                  sequence:
                    # Delay 0.5 second to allow for a second click to cancel the third action.
                    - delay:
                        seconds: 0.5
                    
                    - choose:
                        # The door us locked
                        - conditions:
                            - condition: template
                              value_template: '{{ states("lock.front_door")|lower|trim == "locked" }}'
                          sequence:
                            - service: lock.unlock
                              entity_id: lock.front_door
                            
                            - service: input_boolean.turn_on
                              entity_id: input_boolean.leave_unlocked

                      # The door is unlocked
                      default:
                        - service: input_boolean.turn_off
                          entity_id: input_boolean.leave_unlocked
                        
                        - service: lock.lock
                          entity_id: lock.front_door

              # Single Click
              default:
                # Delay 0.5 second to allow for a second click to cancel the first action.
                - delay:
                    seconds: 0.5

                # Toggle the fan.
                - service: switch.toggle
                  entity_id: switch.family_room_fan

      ##########################################################
      # Omitting handlers for the other 4 buttons for brevity. #
      ##########################################################
{% endraw %}```

## Reusable logic in a script

```{% raw %}
# The click data is stored in CSV fomat in the order: device_ieee, command, click, click_count
#
# Note: you can extract the data from the input_text list like so.
# {% set data = states("input_text.zha_click").split(",") %}
# {% set previous_device  = data[0] %}
# {% set previous_command = data[1] %}
# {% set previous_click   = data[2] %}
# {% set click_count      = data[3] %}
script:
  zha_store_click:
  mode: queued
  sequence:
    # Store the previous click values, and whether or not the current click is a double click.
    #   * second click must occur within 0.5 second of the first click
    #   * must be the same remote
    #   * must be sending the same command
    - service: input_text.set_value
      data_template:
        entity_id: input_text.zha_click
        # CSV Order: device_ieee, command, click, click_count
        value: >-
          {%- set data = states("input_text.zha_click").split(",") %}
          {%- set previous_device = data[0]|trim %}
          {%- set previous_command = data[1]|trim %}
          {%- set click = as_timestamp(now())|float %}
          {%- set previous_click = data[2]|float %}
          {%- set click_delta = click - previous_click %}
          {%- set click_count = data[3]|int %}
          {{- device|trim }},
          {{- command|trim }},
          {{- click }},
          {%- if click_delta <= 0.5 and previous_device == device|trim and previous_command == command %}
            {{- click_count + 1 }}
          {%- else %}
            {{- 1 }}
          {%- endif %}
#          ,{{ click_delta }}

# ^^^ Unremark the last line to show the timing between clicks during debugging.
{% endraw %}```

## Source
You can find full versions of my automation and script at the links below. My full automation handles multiple
remotes instead of just the one in the below example.
 * [automation.zha_button_click](https://github.com/brianhanifin/Home-Assistant-Config/blob/0b8fe58881201655cce7a56e4317d5b221b869b3/automations/buttons/zha_button_click.yaml)

 * [script.zha_store_click](https://github.com/brianhanifin/Home-Assistant-Config/blob/0b8fe58881201655cce7a56e4317d5b221b869b3/scripts/buttons/zha_store_click.yaml)

## Discuss

Post any questions [on the Home Assistant Community post](https://community.home-assistant.io/t/ikea-tradfri-remote-zha-double-and-triple-click/224535) or below and I will be happy to answer any questions.
