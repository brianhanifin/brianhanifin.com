---
title: Alexa Alarm Wake Up Light
date: 2019-12-21 11:00:00 -0800
categories: [Project]
tags: [Home Assistant]
seo:
  date_modified: 2019-12-21 11:00:00 -0800
---

Thanks to the wonderful [Alexa Media Player][alexa_media_player] custom component,
Home Assistant can make announcements and I can perform actions with my voice. I even have setup custom routines that
are context aware. For example I can say "Alexa, turn on the lights" and depending on the room I am in, those lights
will turn off! But, that's content for another article.

My 13 year old has been having more and more trouble waking up in the morning. I decided to help him wake up by turning
a bedside lamp on at the same time as his Echo alarm. Luckily Alexa Media Player generates a `next_alarm` sensor for us.

### Sensor

While we could just have the light turn on when the alarm goes off, I have found it helpful to gradually increase the
brightness starting 10 minutes before the alarm. So I created a [template sensor][template_sensor] to calculate a wake
up light start time of 10 minutes before the next alarm.

```yaml{% raw %}
platform: template
sensors:
  wakeup_boys_10min_early:
    friendly_name: Light Fade-in Start
    icon_template: mdi:lightbulb-on-outline
    entity_id: sensor.boys_room_next_alarm
    value_template: >-
      {%- set next_alarm = states('sensor.boys_room_next_alarm') -%}
      {%- if next_alarm != "unknown" -%}
        {%- set wakeup_time = next_alarm.split("T")[1].split("-")[0] -%}
      {%- else -%}
        {%- set wakeup_time = "06:00" -%}
      {%- endif -%}
      {%- set wakeup_hour = wakeup_time.split(':')[0] -%}
      {%- set wakeup_minutes = wakeup_time.split(':')[1] -%}
      {%- if (wakeup_minutes | int >= 10) -%}
        {{ "%0.02d:%0.02d"|format(wakeup_hour|int, wakeup_minutes|int -10) }}
      {%- else -%}
        {{ "%0.02d:%0.02d"|format(wakeup_hour|int -1, wakeup_minutes|int +50) }}
      {%- endif -%}
{% endraw %}```

### Automation

This automation is triggered when the current time matches `sensor.wakeup_boys_10min_early`.

```yaml{% raw %}
alias: wakeup_boys_10min_early
trigger:
  platform: template
  value_template: >
    {%- set next_alarm = states("sensor.boys_room_next_alarm") -%}
    {%- if next_alarm != "unknown" -%}
      {%- set wakeup_date = next_alarm.split("T")[0] -%}
    {%- else -%}
      {%- set wakeup_date = states('sensor.date') -%}
    {%- endif -%}
    {%- set start_time = states("sensor.wakeup_boys_10min_early") -%}

    {%- if next_alarm != "unknown" -%}
      {{ wakeup_date == states('sensor.date') and start_time == states('sensor.time') }}
    {%- else -%}
      false
    {%- endif -%}
action:
  - service: script.turn_on
    data_template:
      entity_id: script.wakeup_boys_light_start
{% endraw %}```

### Script

The script is straight forward. It turns on the light and immediately dims the light to 20%. After a minute delay
the light brightness is increased by 10%. The brightness is increased by 10% for 10 minutes until we are at 100%
brightness! I know there are fancy python scripts to gradually increase the light. But 10% increments are good
enough for me.

(I shortened the code below for brevity. You can see the full script on my [Github Repository][wakeup_boys_light_start].)

```yaml
wakeup_boys_light_start:
  sequence:
    # Start lights at ~20% and increment by ~10% every minute for 9 minutes.
    - service: light.turn_on
      data:
        entity_id: light.boys_wakeup

    - service: light.turn_on
      data:
        entity_id: light.boys_wakeup
        brightness: 50

    - delay: 00:01:00
    - condition: state
      entity_id: light.boys_wakeup
      state: 'on'
    - service: light.turn_on
      data:
        entity_id: light.boys_wakeup
        brightness: 75

    .
    .
    .

    - delay: 00:01:00
    - condition: state
      entity_id: light.boys_wakeup
      state: 'on'
    - service: light.turn_on
      data:
        entity_id: light.boys_wakeup
        brightness: 255
```

### Lovelace User Interface

This represents the information I need at a glance.

![My Alexa Alarm Lovelace UI](/assets/img/2019-12-20-alexa-alarm.png)

[alexa_media_player]: https://github.com/custom-components/alexa_media_player
[template_sensor]: https://www.home-assistant.io/integrations/template/
[wakeup_boys_light_start]: https://github.com/brianhanifin/Home-Assistant-Config/blob/master/scripts/wakeup/wakeup_boys_light_start.yaml
