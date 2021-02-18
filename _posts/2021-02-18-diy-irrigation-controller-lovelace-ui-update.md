---
title: "DIY Irrigation Controller: Home Assistant Integration"
date: 2021-02-18 9:45:00 -0700
categories: [Project]
tags: [Lovelace, Home Assistant, ESPHome]
seo:
  date_modified: 2021-02-18 9:45:00 -0700
image: /assets/img/2021-02-18-irrigation-controller/ui.png
---

*Articles in this series:*
1. *[Hardware, Electronics, and ESPHome code](../diy-irrigation-controller-esphome-home-assistant/)*
2. *[Lovelace User Interface](../diy-irrigation-controller-lovelace-user-interface-home-assistant/)*
3. *Entities & Simplified User Interface*

## Home Assistant Entities & Simplified User Interface

Your many questions helped me realize that I totally dropped the ball when I attempted to share how I
created the Lovelace User Interface (UI) for my DIY Irrigation Project. Not only did I overly complicate
the Lovelace code with custom components, but I also omitted how I created the necessary entities in
Home Assistant.

I now present you with the **simplified and complete** code to properly integrate Home Assistant with
my ESPHome DIY Irritation Controller project.

### Add Supporting Entities

#### Using Helpers Configuration UI

Home Assistant's **Helpers Configuration UI** [introduced in 2020](https://www.home-assistant.io/blog/2020/03/18/release-107/#helpers-configuration-panel)
is the easiest way to add these `input_text` and `input_number` entities. If you choose to go this route the YAML code
below will be a useful reference.

#### Using a "Package" File

I have chosen instead to split my Irrigation Controller entities and automation into a "package file".

1. First we need to create new file named `irrigation.yaml` in your Home Assistant `/config` folder.
2. Copy and paste the following code into this file to get started.

<blockquote>
<i>Note:</i> In this example you will see that I have two irrigation zones configured. If you have more than two zones
you will need to add enough `input_text` and `input_number` entities to accommodate your additional zones.
</blockquote>

```yaml
---
# Lovelace UI to set a list of irrigation cycle times.
input_text:
  irrigation_zone1_times:
    name: List of Times (eg. 08:00,12:00,15:00)
    icon: mdi:clock-outline
  irrigation_zone2_times:
    name: List of Times (eg. 11:00,15:30)
    icon: mdi:clock-outline

# Lovelace UI to set the duration of each irrigation cycle.
input_number:
  irrigation_zone1_duration:
    name: Duration in Minutes
    icon: mdi:timer-sand
    min: 0
    max: 60
    step: 1
    unit_of_measurement: "minutes"

  irrigation_zone2_duration:
    name: Duration in Minutes
    icon: mdi:timer-sand
    min: 0
    max: 60
    step: 1
    unit_of_measurement: "minutes"

### Optional offline notifications. Uncomment this automation if you'd like an notification 
### should the device be disconnected from the network for two hours!
# automation:
#   # Warn me if the system ever goes offline for more than two hours!
#   - alias: irrigation_system_offline
#     initial_state: true
#     trigger:
#       - platform: state
#         entity_id: binary_sensor.irrigation_controller_status
#         to: 'off'
#         for: '02:00:00'
#     action:
#       - service: persistent_notification.create
#         data:
#           title: "Irrigation System Offline"
#           message: "The Irrigation System has been offline for 2 hours!"
#           notification_id: "offline"
#       - service: notify.mobile_app_iphone_brian
#         data:
#           title: "Irrigation System Offline"
#           message: "The Irrigation System has been offline for 2 hours"
```

#### Include the "Package" file in configuration.yaml

<blockquote>
<i>Note:</i> You can skip this section if you are adding the above entities to Home Assistant another way.
</blockquote>

See the [Home Assistant Packages Documentation](https://www.home-assistant.io/docs/configuration/packages/)
for a deeper explanation.

```yaml
homeassistant:
  packages:
    irrigation: !include irrigation.yaml
```

### Simplified Lovelace User Interface

This version of the interface simply uses the built in Entities Card and optionally a Glance Card.

#### Create Irrigation Zone User Interfaces

You will need to add an Entities Card for each Irrigation Zone.

```yaml
type: entities
entities:
  - entity: switch.irrigation_zone1
  - entity: sensor.irrigation_zone1_next
    name: Next 🕑
  - entity: sensor.irrigation_zone1_remaining
    name: Remaining ⏳
  - entity: input_text.irrigation_zone1_times
  - entity: input_number.irrigation_zone1_duration
title: Zone 1
show_header_toggle: false
```

#### Optional: Create Irrigation Controller Status Card

```yaml
type: glance
entities:
  - entity: binary_sensor.irrigation_controller_status
    name: Status
  - entity: sensor.irrigation_controller_uptime
    name: Uptime
  - entity: sensor.irrigation_controller_wifi_signal
    name: WiFi Signal
title: Controller Status
```
