---
title: "DIY Irrigation Controller: Lovelace User Interface"
date: 2020-04-28 13:00:00 -0700
categories: [Project]
tags: [Lovelace, Home Assistant, ESPHome]
seo:
  date_modified: 2021-02-18 9:45:00 -0700
image: /assets/img/2020-02-13-irrigation-controller/ui1t.png
---

*Articles in this series:*
1. *[Hardware, Electronics, and ESPHome code](../diy-irrigation-controller-esphome-home-assistant/)*
2. *Lovelace User Interface*
3. *[Entities & Simplified User Interface](../diy-irrigation-controller-lovelace-ui-update/)*

## DIY Irrigation Controller Part 2

While my the Irrigation Controller can be manually started and stopped with physical buttons, the primary user interface was always intended to be built in Lovelace.

### Reliability Update

Before I describe how I created the interface I have great news! In the 2 months since I put the controller into service it has been rock solid! At the moment the Uptime is up to 248 hours since the last time it restarted. The only times it has been restarted has been once to upload a tweak to my ESPHome code, and the second one was for a neighborhood-wide power outage!!! :)

### Building A User Interface

Each Zone consists of the following:

1. Manual on/off switch.
2. Status Display showing: Is It Running, Next Scheduled Time, and Time Remaining.
3. Scheduler: field to enter a comma separated list of times, and a slider to select the runtime in minutes.

```yaml
type: vertical-stack
cards:
  - type: markdown
    style: |
      ha-card {
        background: none;
        box-shadow: none;
        letter-spacing: 0.06em;
        margin: 0 0 -1em 0;
        padding: 0;
      }
    content: |-
      **🌱 Drip System**

- type: entities
  show_header_toggle: false
  entities:
    - type: custom:paper-buttons-row
      buttons:
        - entity: switch.irrigation_zone1
          name: Start Cycle
          state_icons:
            "off": mdi:play
            "on": mdi:stop
          style:
            button:
            background: lightgray
            border-radius: 9999px
            font-weight: bold
          state_styles:
            "off":
              button:
                color: red
              ripple:
                color: red
            "on":
              button:
                color: green
              ripple:
                color: green

    - type: custom:text-divider-row
      text: Status

    - type: custom:multiple-entity-row
      entity: binary_sensor.irrigation_zone1
      name: 💧 Watering
      icon: "[[icon]]"
      show_state: false
      secondary_info:
        entity: binary_sensor.irrigation_zone1
        name: ""
      entities:
        - entity: sensor.irrigation_zone1_next
          name: ⏭️ Next
        - entity: sensor.irrigation_zone1_remaining
          name: ⏳ Remaining
          unit: min

        - type: custom:fold-entity-row
          head:
            type: custom:text-divider-row
            text: Scheduler
          entities:
            - type: custom:text-input-row
              entity: input_text.irrigation_zone1_times
            - entity: sensor.irrigation_zone1_duration
              name: Duration in Minutes
              icon: mdi:timer-sand
            - type: custom:slider-entity-row
              entity: input_number.irrigation_zone1_duration
              full_row: true
              hide_state: true
```

### Controller Status

At the bottom I include "glance" sensors displaying the status of the controller.

```yaml
  - type: glance
    entities:
      - entity: binary_sensor.irrigation_controller_status
        name: Status
      - entity: sensor.irrigation_controller_uptime
        name: Uptime
      - entity: sensor.irrigation_controller_wifi_signal
        name: WiFi Signal
```

Let me know if you have any questions.

<div style="margin-top:3em;">
<style type="text/css">
div#amzn-native-ad-0 {
  left: 0 !important;
}
</style>
<script type="text/javascript">
amzn_assoc_placement = "adunit0";
amzn_assoc_search_bar = "true";
amzn_assoc_tracking_id = "brianhanifi0d-20";
amzn_assoc_ad_mode = "manual";
amzn_assoc_ad_type = "smart";
amzn_assoc_marketplace = "amazon";
amzn_assoc_region = "US";
amzn_assoc_title = "Components mentioned in this article";
amzn_assoc_linkid = "37a4f3f0677f4d551439a4f5d4e1c92b";
amzn_assoc_asins = "B0793NYYPZ,B0007N5LJK,B000VYGMF2,B001H1NGOI";
</script>
<script src="//z-na.amazon-adsystem.com/widgets/onejs?MarketPlace=US"></script>
</div>

---

[github-issue]: https://github.com/brianhanifin/Home-Assistant-Config/issues/37
[github-esphome]: https://github.com/brianhanifin/esphome-config
[github-ha]: https://github.com/brianhanifin/Home-Assistant-Config
