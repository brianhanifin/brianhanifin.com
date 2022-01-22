---
title: Voice Event Reminders with a User Interface
date: 2020-01-06 21:00:00 -0700
categories: [Project]
tags: [Home Assistant, Alexa, Lovelace]
seo:
  date_modified: 2022-01-22 09:00:00 -0700
---

Some of my favorite "quality of life" automations are: 1.) an LED strip that turns on when anyone
enters the kitchen, 2.) bedside lights that help us wake up in the morning, and 3.) Alexa voice alerts
that remind us when it is about time to leave for events (school, water polo, circus class, etc).

I got caught up on the Node-Red hype train for a short while, but that ride didn't last long. To
begin the new decade I decided to retire Node-Red for good. To do so required me to move my many
Event Reminders over to YAML and I wanted to really push my skills with this project.

## Updates

* **2022-01-22**
  * Fix broken links to my Github repository.

## Project Requirements

[![event-reminder1-thumbnail]{:style="float:right; margin-left: 1em;"}][event-reminder1]

1. Schedule Alexa to alert us when an event draws near.
2. Provide a User Interface (UI) with the following features for each event:
  - Enable/Disable button
  - "Skip Next" button
  - Days of the Week selection
  - First and Second Announcement times
  - First and Second Announcement text
  - List of Echo devices and device groups the announcement should be heard on
3. The Automations must be reliable! The school principal doesn't seem to understand that my kids are late because automated voice assistant didn't remind me to take them to school. ;)

### User Interface

Before I wrote the automations, I started to lay out the UI so I knew what entities I needed to create.
You can see the final results of this planning in the image above. I ended up using the following six
custom Lovelace plugins, all installed via the [Home Assistant Community Store (HACS)][hacs].

1. [custom:**decluttering-card**][custom-declutter]
2. [custom:**text-input-row**][custom-text]
3. [custom:**button-card**][custom-button]
4. [custom:**button-entity-row**][custom-button-row]
5. [custom:**fold-entity-row**][custom-fold]
6. [custom:**text-divider-row**][custom-divider]

I consider the first two to be necessary for this project to be feasible. The last four could be
omitted if you prefer. The decluttering card makes a template for each of my (currently) 10 events,
while the text card allows the text field to span the entire width of the parent card.

If you'd like to see the code behind the UI, here are the files in my repository to look at.

- [/lovelace/views/05_event_reminders.yaml][lovelace-view]
- [/lovelace/templates/event_reminder.yaml][lovelace-event_reminder]
- [/lovelace/templates/heading.yaml][lovelace-heading]
- [/lovelace/templates/button_row7.yaml][lovelace-button_row7]
- [/lovelace/templates/button.yaml][lovelace-button]
- [/lovelace/templates/button_pill2.yaml][lovelace-button_pill2]

The detail on the UI design beyond the intended scope of this article. However, I will share that
the Enable button simply enables and disables the automation directly. As always feel free to ask
questions in the comment section below, and who knows I may do a follow up article detailing the UI.

### Automation

Here is the base automation I wrote to trigger the Alexa voice reminders. First I trigger the
automation when either the First or Second time equal the current time. This saves a lot of code
duplication as you will see. Then I check if the alert is supposed to be spoken on the current day,
Finally I call `script.event_reminder_announce` to pass the First or Second Announcement text along
to be spoken.

```yaml{% raw %}
automation:
  - alias: event_reminder_1
    trigger:
      platform: template
      value_template: >-
        {{ is_state('sensor.event_reminder_1_1',states('sensor.time'))
        or is_state('sensor.event_reminder_1_2',states('sensor.time')) }}
    condition:
      condition: and
      conditions:
        # Is today one of the selected days?
        - condition: template
          value_template: >
            {% set   day_name = now().strftime("%A")|lower -%}
            {%- if   day_name == 'monday'    and is_state('input_boolean.event_reminder_1_mon','on') -%}
              {{true}}
            {%- elif day_name == 'tuesday'   and is_state('input_boolean.event_reminder_1_tue','on') -%}
              {{true}}
            {%- elif day_name == 'wednesday' and is_state('input_boolean.event_reminder_1_wed','on') -%}
              {{true}}
            {%- elif day_name == 'thursday'  and is_state('input_boolean.event_reminder_1_thu','on') -%}
              {{true}}
            {%- elif day_name == 'friday'    and is_state('input_boolean.event_reminder_1_fri','on') -%}
              {{true}}
            {%- elif day_name == 'saturday'  and is_state('input_boolean.event_reminder_1_sat','on') -%}
              {{true}}
            {%- elif day_name == 'sunday'    and is_state('input_boolean.event_reminder_1_sun','on') -%}
              {{true}}
            {%- else -%}
              {{false}}
            {%- endif %}
    action:
      - service: script.event_reminder_announce
        data_template:
          media_player: "{{ states('input_select.event_reminder_1_echo') }}"
          message: >-
            {% if is_state('sensor.event_reminder_1_1',states('sensor.time')) %}
              {{ states('input_text.event_reminder_1_1') }}
            {% else %}
              {{ states('input_text.event_reminder_1_2') }}
            {% endif %}
{% endraw %}```

There is a little more to the final code that handles when the Skip Next button is enabled. But that just
complicates things a little bit.

### Announcement Scripts

### `script.event_reminder_announce`

This script replaces the friendly media player name with the entity_id of the target Echo device, and
passes the message along to `script.say`.

```yaml{% raw %}
script:
  event_reminder_announce:
    sequence:
      - service: script.say
        data_template:
          message: "{{ message }}"
          media_player: >
            {%- set room = media_player|lower|replace(' ','_')|replace('[','')|replace(']','')  %}
            {%-
              set alexa = {
                "bedroom" : "media_player.master_bedroom",
                "boys_bedroom": "media_player.boys_room",
                "downstairs": "media_player.downstairs",
                "kitchen/garage": "group.alexa_welcome",
                "garage" : "media_player.garage",
                "kitchen" : "media_player.kitchen",
                "family_room" : "media_player.family_room",
                "play_room": "media_player.play_room",
                "upstairs": "media_player.upstairs",
                "upstairs_bathroom": "media_player.upstairs_bathroom"
              }
            -%}
            {{ alexa[room] }}
{% endraw %}```

### `script.say`

The following script is an ultra simplified version of my speech script, which uses the custom
[Alexa Media Player][custom-alexa] [voice notification feature][custom-alexa-notify]. This component
is amazing and can also be installed with HACS.


```yaml{% raw %}
  say:
    sequence:
      - service: notify.alexa_media
        data_template:
          data:
            type: announce
            method: all
          title: >
            {%- if title is not string -%}
              Home Assistant
            {%- else -%}
              {{ title }}
            {%- endif -%}
          message: "{{ message }}"
          target: "{{ media_player }}"
{% endraw %}```

## Supporting Entities

The tedious part was creating all 15 of the `input_boolean`, `input_datetime`, `input_select`,
`input_text`, and `sensor` entities. After creating my first two reminders, I realized it was going
to take forever to do this 10 times! I am a big fan of the way @frenck organizes his configuration
files (essentially, each entity gets a separate file). That's when I realized I needed to package
all of the automations, scripts, and entities together... well, in a [package][packages].

### Packages

You can find all of the packages for this project in the [/integrations/event_reminders][package-folder]
folder. The newest revision of the code I shared above can be found in
[event_reminder_1_package.yaml][package1]. The speech scripts can be found in
[event_reminder_common_package.yaml][package-common].

## Conclusion

After using this code reliably for a few days I am really proud of how it turned out. I tried to keep
this article relatively brief so I didn't bore you with every detail. If you enjoyed this overview
of my most ambitious project to date, please leave a comment below. Questions are also welcomed
on the [Home Assistant Community][forum] forum, [Twitter][twitter], [my GitHub repo][repo] or comments
on this blog.


[repo]: https://github.com/brianhanifin/Home-Assistant-Config
[forum]: https://community.home-assistant.io/u/brianhanifin
[twitter]: https://twitter.com/brianhanifin

[event-reminder1]: /assets/img/2020-01-06/event-reminder1.png
[event-reminders-1-3]: /assets/img/2020-01-06/event-reminders-1-3.png
[event-reminder1-thumbnail]: /assets/img/2020-01-06/event-reminder1-thumbnail.png
[event-reminders-1-3-thumbnail]: /assets/img/2020-01-06/event-reminders-1-3-thumbnail.png

[hacs]: https://github.com/hacs/integration
[custom-button]: https://github.com/custom-cards/button-card
[custom-button-row]: https://github.com/custom-cards/button-entity-row
[custom-declutter]: https://github.com/custom-cards/decluttering-card
[custom-divider]: https://github.com/custom-cards/text-divider-row
[custom-fold]: https://github.com/thomasloven/lovelace-fold-entity-row
[custom-text]: https://github.com/gadgetchnnel/lovelace-text-input-row

[lovelace-view]: https://github.com/brianhanifin/Home-Assistant-Config/blob/ce419ffc8dfb861c4310be9239d2949e6edd70bc/lovelace/views/05_event_reminders.yaml
[lovelace-button]: https://github.com/brianhanifin/Home-Assistant-Config/blob/ce419ffc8dfb861c4310be9239d2949e6edd70bc/lovelace/templates/button.yaml
[lovelace-button_pill2]: https://github.com/brianhanifin/Home-Assistant-Config/blob/ce419ffc8dfb861c4310be9239d2949e6edd70bc/lovelace/templates/button_pill2.yaml
[lovelace-button_row7]: https://github.com/brianhanifin/Home-Assistant-Config/blob/ce419ffc8dfb861c4310be9239d2949e6edd70bc/lovelace/templates/button_row7.yaml
[lovelace-event_reminder]: https://github.com/brianhanifin/Home-Assistant-Config/blob/ce419ffc8dfb861c4310be9239d2949e6edd70bc/lovelace/templates/event_reminder.yaml
[lovelace-heading]: https://github.com/brianhanifin/Home-Assistant-Config/blob/ce419ffc8dfb861c4310be9239d2949e6edd70bc/lovelace/templates/heading.yaml

[custom-alexa]: https://github.com/custom-components/alexa_media_player
[custom-alexa-notify]: https://github.com/custom-components/alexa_media_player/wiki/Configuration%3A-Notification-Component

[packages]: https://www.home-assistant.io/docs/configuration/packages
[package-folder]: https://github.com/brianhanifin/Home-Assistant-Config/blob/ce419ffc8dfb861c4310be9239d2949e6edd70bc/integrations/event_reminders/
[package1]: https://github.com/brianhanifin/Home-Assistant-Config/blob/ce419ffc8dfb861c4310be9239d2949e6edd70bc/integrations/event_reminders/event_reminder_1_package.yaml
[package-common]: https://github.com/brianhanifin/Home-Assistant-Config/blob/ce419ffc8dfb861c4310be9239d2949e6edd70bc/integrations/event_reminders/event_reminder_common_package.yaml
