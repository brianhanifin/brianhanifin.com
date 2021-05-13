---
title: Inovelli Red Series Status LED Script Update
date: 2020-09-18 15:00:00 -0700
categories: [Project]
tags: [Home Assistant, Jinja, Z-Wave]
seo:
  date_modified: 2020-09-18 15:00:00 -0700
#image: /assets/img/YYYY-MM-DD-article/image.png
---

This is a follow up to my previous article [Inovelli Z-Wave Dimmer Status LED in Home Assistant][part1]. If you are
interested in the basics of how to use this script to send notification light animations to your
[Inovelli (Red Series) Z-Wave Dimmer][amazon-inovelli-dimmer] start there. In this article I will explain the
improvements made to support all three Red Series in wall switches.

## Added compatibility

Here is an example call to `script.inovelli_led` with the new `model` parameter set to `dimmer` (valid options are switch, 
dimmer, combo_light, or combo_fan). This information allows one script to support the 
[Red Series Dimmer][amazon-inovelli-dimmer], the [Red Series Switch][amazon-inovelli-dimmer], as well as the [Red Series Light & Fan controller][amazon-inovelli-fan]. (Note: the OZW integration does not add zwave objects, so I am passing the light object instead.)

```{%raw%}
- service: script.inovelli_led
  data:
    entity_id: light.family_room
    model: dimmer
    color: purple
    duration: 10 seconds
    effect: blink
{%endraw%}```

## Advanced scripting using `choose` & `variables`

* [0.113's choose feature][0.113-choose] allowed me to separate the calls to the Z-Wave and OZW services.
* [0.115's variables feature][0.115-variables] allowed me to avoid duplicating the the Inovelli Math calculation for each 
service call.

## `variables`: Part 1

This section of variables are defined before the `sequence` section.

```yaml{%raw%}
variables:
  model: |
    {% if model is string %}
      {{ model }}
    {%- elif state_attr(entity_id, 'product_name') is string %}
      {%- if 'LZW31' in state_attr(entity_id, 'product_name') %}
        dimmer
      {%- elif 'LZW36' in state_attr(entity_id, 'product_name') %}
        combo_light
      {%- else %}
        switch
      {%- endif %}
    {%- else %}
      dimmer
    {%- endif %}
  parameters:
    dimmer: 16
    combo_light: 24
    combo_fan: 25
    switch: 8
  color: |
    {%- if color is not number %}
      {{ color|default("Yellow") }}
    {%- else %}
      {{ color|int }}
    {% endif %}
  # 1-10
  level: '{{ level|default(4) }}'
  duration: '{{ duration|default("Indefinitely") }}'
  effect: '{{ effect|default("Blink") }}'
  colors:
    "Off": 0
    "Red": 1
    "Orange": 21
    "Yellow": 42
    "Green": 85
    "Cyan": 127
    "Teal": 145
    "Blue": 170
    "Purple": 195
    "Light Pink": 220
    "Pink": 234
  durations:
    "Off": 0
    "1 Second": 1
    "2 Seconds": 2
    "3 Seconds": 3
    "4 Seconds": 4
    "5 Seconds": 5
    "6 Seconds": 6
    "7 Seconds": 7
    "8 Seconds": 8
    "9 Seconds": 9
    "10 Seconds": 10
    "15 Seconds": 15
    "20 Seconds": 20
    "25 Seconds": 25
    "30 Seconds": 30
    "35 Seconds": 35
    "40 Seconds": 40
    "45 Seconds": 45
    "50 Seconds": 50
    "55 Seconds": 55
    "60 Seconds": 60
    "2 Minutes": 62
    "3 Minutes": 63
    "4 Minutes": 64
    "10 Minutes": 70
    "15 Minutes": 75
    "30 Minutes": 90
    "45 Minutes": 105
    "1 Hour": 120
    "2 Hours": 122
    "Indefinitely": 255
  effects_dimmer:
    "Off": 0
    "Solid": 1
    "Chase": 2
    "Fast Blink": 3
    "Slow Blink": 4
    "Blink": 4
    "Pulse": 5
    "Breath": 5
  effects_switch:
    "Off": 0
    "Solid": 1
    "Fast Blink": 2
    "Slow Blink": 3
    "Blink": 3
    "Pulse": 4
    "Breath": 4
{%endraw%}```


## `variables`: Part 2

Variables can also be defined or redefined in the `sequence` section as well. Here I am assigning the final values,
and using those values to preform the Inovelli Math.


```yaml{%raw%}
sequence:
  # Preform the Inovelli math.
  - variables:
      parameter: '{{ parameters[model|default("dimmer")] }}'
      color: '{{ colors[color|default("purple")|title] }}'
      duration: '{{ durations[duration|default("10 Seconds")|title] }}'
      effect: |
        {% if model == "switch" %}
          {{- effects_switch[effect|default("Blink")|title]|int }}
        {%- else %}
          {{- effects_dimmer[effect|default("Blink")|title]|int }}
        {% endif %}
      inovelli_math: |
        {%- if effect|int > 0 %}
          {{ color|int + (level|int * 256) + (duration|int * 65536) + (effect|int * 16777216) }}
        {%- else %}
          0
        {% endif %}
{%endraw%}```

## `choose` between the Z-Wave or OZW Integration

Finally, the service calls. If the entity_id begins with `zwave` then use the `zwave.set_config_parameter` service.
Otherwise use the `ozw.set_config_parameter` service.

```yaml{%raw%}
  - choose:
    # The Z-wave integration requires this service call.
    - conditions:
        - '{{ entity_id.split(".")[0] == "zwave" }}'
      sequence:
        # Clear the previous effect.
        - service: zwave.set_config_parameter
          data_template:
            node_id: '{{ state_attr(entity_id,"node_id") }}'
            parameter: '{{ parameter }}'
            size: 4
            value: '0'

        # Start the new effect.
        - service: zwave.set_config_parameter
          data_template:
            node_id: '{{ state_attr(entity_id,"node_id") }}'
            parameter: '{{ parameter }}'
            size: 4
            value: '{{ inovelli_math }}'

    # The OZW integration requires this service call.
    default:
      # Clear the previous effect.
      - service: ozw.set_config_parameter
        data_template:
          node_id: '{{ state_attr(entity_id,"node_id") }}'
          parameter: '{{ parameter }}'
          value: '0'

        # Start the new effect.
      - service: ozw.set_config_parameter
        data_template:
          node_id: '{{ state_attr(entity_id,"node_id") }}'
          parameter: '{{ parameter }}'
          value: '{{ inovelli_math }}'
{%endraw%}```

## Fixing reliability

I found that sending an "off" command to the service ensures the new effect is applied reliably. I did not discover
this issue until I was testing these new changes and I could not figure out why nothing would happen sometimes and
other times it would work just fine. As it turns out a new effect has to be assigned to the switch before it can
show the previous effect again. This must have begun occuring after my switch to the OZW integration. I suppose that
makes sense, OZW is storing the parameters in MQTT. Anyway, this simple change has made new effects reliable every time!


[part1]: /posts/inovelli-dimmer-status-led-home-assistant/
[0.113-choose]: https://www.home-assistant.io/blog/2020/07/22/release-113/#automations--scripts-chooser
[0.115-variables]: https://www.home-assistant.io/blog/2020/09/17/release-115/#variables
[amazon-inovelli-dimmer]: https://www.amazon.com/gp/product/B07T26MVYC/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=B07T26MVYC&linkCode=as2&tag=brianhanifi0d-20&linkId=f70336e0578d04d9c39812db7ced730a
[amazon-inovelli-fan]: https://www.amazon.com/gp/product/B08665WJ2B/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=B08665WJ2B&linkCode=as2&tag=brianhanifi0d-20&linkId=2f2e7797c8b657345aab59d1c90c57ae
