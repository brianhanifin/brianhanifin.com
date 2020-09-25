---
title: Outdoor Illuminance Template Sensor
date: 2020-09-25 10:00:00 -0700
categories: [Project]
tags: [Home Assistant, Jinja]
seo:
  date_modified: 2020-09-25 10:00:00 -0700
image: /assets/img/2020-09-25-outdoor-illuminance/screenshot.png
---

Phil Bruckner (@pnbruckner) created a terrific outdoor [Illuminance sensor][ha-illuminance] component. Unfortunately it seemed 
that every time he has added support for a new weather service, support for that service eventually get pulled for one reason 
or another.

## Introducing the *Outdoor Illuminance Educated Guessor*

I took a look at the code and decided it would be a nice challenge to convert his Python code into a Home Assistant sensor 
template. So far I believe I have succeeded. I made use of Phil's [sun2 component][sun2] to gather today's sunrise and sunset 
values. I manually attempted to create a fairly universal `condition_factors` table and used some replacement filters in an 
attempt to normalize the conditions between weather services.

<blockquote>
<i>Note:</i> The intent of this code is to give all of you (that aren't comfortable with Python) the ability to customize the 
condition factors and even the sun factor in any way that works best for you. If you find a weather condition that this code 
doesn't cover, add it to your personal template!
</blockquote>

## Code

The template code is performing very well! I started this as a personal challenge just to see if I could do it, and I did. 
I don't plan on updating the code much in the future. Consider this your starting point!

By the way, I ran it side-by-side with [nkm8's ha-illuminance fork][ha-illuminance-nkm8] and the above chart was the result 
overnight! The reason for the slight difference in timing is ha-illuminance estimates dawn and dusk, while I am getting, what 
I assume is a more accurate time from @pnbruckner's sun2 component. Due to this comparison I modified the template code to 
keep the lowest lx value at 10 instead of 0.

### sensor.outdoor_illuminance - template sensor
```{%raw%}
---
# pnbruckner's sensor component as a template.
# https://github.com/pnbruckner/ha-illuminance/blob/master/custom_components/illuminance/sensor.py
platform: template
sensors:
  outdoor_illuminance:
    friendly_name: Outdoor Illuminance Educated Guessor
    icon_template: mdi:brightness-auto
    unit_of_measurement: lx
    value_template: |
      {%- set factors = namespace(condition='',sun='') %}

      {#- Retrieve the current condition and normalize the value #}
      {%- set current_condition = states("weather.accuweather") %}
      {%- set current_condition = current_condition|lower|replace("partly cloudy w/ ","")|replace("mostly cloudy w/ ","")|replace("freezing","")|replace("and","")|replace(" ", "")|replace("-", " ")|replace("_", " ")|replace("(","")|replace(")","") %}
      
      {#- Assign a seemingly arbitrary number to the condition factor #}
      {%- set condition_factors = {
        "10000": ("clear", "clearnight", "sunny", "windy", "exceptional"),
        "7500": ("partlycloudy", "partlysunny", "mostlysunny", "mostlyclear", "hazy", "hazysunshine", "intermittentclouds"),
        "2500": ("cloudy", "mostlycloudy"),
        "1000": ("fog", "rainy", "showers", "snowy", "snowyheavy", "snowyrainy", "flurries", "chanceflurries", "chancerain", "chancesleet", "drearyovercast", "sleet"),
        "200": ("hail", "lightning", "tstorms")
      } %}
      {%- for factor in condition_factors if current_condition in condition_factors[factor] %}
        {%- set factors.condition = factor %}
      {%- endfor %}
      
      {#- Compute Sun Factor #}
      {%- set right_now = states.sensor.time.last_updated.timestamp() %}
      {%- set sunrise = states("sensor.sunrise") | as_timestamp %}
      {%- set sunrise_begin = states("sensor.dawn") | as_timestamp %}
      {%- set sunrise_end = sunrise + (40 * 60) %}
      {%- set sunset = states("sensor.sunset") | as_timestamp %}
      {%- set sunset_begin = sunset - (40 * 60) %}
      {%- set sunset_end = states("sensor.dusk") | as_timestamp %}
      {%- if sunrise_end < right_now and right_now < sunset_begin %}
        {%- set factors.sun = 1 %}
      {%- elif sunset_end < right_now or right_now < sunrise_begin %}
        {%- set factors.sun = 0 %}
      {%- elif right_now <= sunrise_end %}
        {%- set factors.sun = (right_now - sunrise_begin) / (60*60) %}
      {%- else %}
        {%- set factors.sun = (sunset_end - right_now) / (60*60) %}
      {%- endif %}
      {%- set factors.sun = 1 if factors.sun > 1 else factors.sun %}
      
      {# Take an educated guess #}
      {%- set illuminance = (factors.sun|float * factors.condition|float) | round %}
      {%- set illuminance = 10 if illuminance < 10 else illuminance %}
      {{ illuminance }}
{%endraw%}```

### sun2 sensor config
```{%raw%}
---
platform: sun2
monitored_conditions:
  - sunrise
  - sunset
  - dawn
  - dusk
{%endraw%}```

## Optional: Multiple Weather Condition Sensors
If your primary weather source goes “unavailable” sometimes (I'm looking at you AccuWeather), you can modify a few lines of code to add one or more backup sensors.

### current_condition: original lines
```{%raw%}
{%- set factors = namespace(condition='',sun='') %}

{#- Retrieve the current condition and normalize the value #}
{%- set current_condition = states("weather.accuweather") %}
{%- set current_condition = current_condition|lower|replace("partly cloudy w/ ","")|replace("mostly cloudy w/ ","")|replace("freezing","")|replace("and","")|replace(" ", "")|replace("-", " ")|replace("_", " ")|replace("(","")|replace(")","") %}
{%endraw%}```

### current_condition: new lines (with 2 backup condition sensors)
```{%raw%}
{%- set factors = namespace(condition='',sun='',current_condition='') %}

{#- Retrieve the current condition and normalize the value #}
{%- set weather_sensors = [
  "weather.accuweather",
  "sensor.openweathermap_condition",
  "sensor.cc_climacell_weather_condition"
] %}
{%- for sensor in weather_sensors if states(sensor) != "unknown" %}
  {%- set factors.current_condition = states(sensor) if factors.current_condition == "" else "" %}
{%- endfor %}
{%- set current_condition = factors.current_condition|lower|replace("partly cloudy w/ ","")|replace("mostly cloudy w/ ","")|replace("freezing","")|replace("and","")|replace(" ", "")|replace("-", " ")|replace("_", " ")|replace("(","")|replace(")","") %}
{%endraw%}```

[ha-illuminance]: https://github.com/pnbruckner/ha-illuminance
[ha-illuminance-nkm8]: https://github.com/nkm8/ha-illuminance
[sun2]: https://github.com/pnbruckner/ha-sun2
