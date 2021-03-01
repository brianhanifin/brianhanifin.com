---
title: "Home Assistant Template Macros: Date and Time"
date: 2020-01-18 21:00:00 -0700
categories: [Code Snippets]
tags: [Home Assistant, Jinja]
seo:
  date_modified: 2021-02-28 18:30:00 -0700
---

Over time I have created a large library of date and time manipulation code which are used in my
automations and scripts. I plan to update this post with the snippets as I add to my library.

## Date

### Standard Examples

```yaml{% raw %}
{% set date = as_timestamp(now())|timestamp_custom("%A %B %-d, %Y") %}
{% set datetime = as_timestamp(now()) | timestamp_custom("%I:%M:%S %p %b/%d/%Y", true)
{% set now_string = now().strftime("%Y-%m-%d") %}

{% set month_name = now().strftime("%B") %}
{% set day_name = now().strftime("%A")|lower %}
{% set this_year = now().strftime("%Y")|int %}
{% endraw %}```

### timedelta()

```yaml{% raw %}
# Calculate one week from today.
{{ as_timestamp(now()) + timedelta(days=7) }}

# Calculate one month ago.
{{ as_timestamp(now()) - timedelta(month=1) }}
{% endraw %}```

### dayofweek_number(dayofweek)

```yaml{% raw %}
{%- macro dayofweek_number(dayofweek) -%}
  {%- if dayofweek == "Sunday" or dayofweek == "Sun" -%}
    0
  {%- elif dayofweek == "Monday" or dayofweek == "Mon" -%}
    1
  {%- elif dayofweek == "Tuesday" or dayofweek == "Tue" -%}
    2
  {%- elif dayofweek == "Wednesday" or dayofweek == "Wed" -%}
    3
  {%- elif dayofweek == "Thursday" or dayofweek == "Thu" -%}
    4
  {%- elif dayofweek == "Friday" or dayofweek == "Fri" -%}
    5
  {%- elif dayofweek == "Saturday" or dayofweek == "Sat" -%}
    6
  {%- endif -%}
{%- endmacro -%}
{% endraw %}```

### last_dayofmonth(month, year)

```yaml{% raw %}
{%- macro last_dayofmonth(month, year) -%}
  {%- set daysinmonths = [31,28,31,30,31,30,31,31,30,31,30,31] -%}
  {%- set month = month|int -%}
  {%- set year = year|int -%}

  {# Simplified leap year calculation. See https://www.mathsisfun.com/leap-years.html #}
  {%- set isleapyear = year % 4 == 0 and (year % 100 != 0 or year % 400 == 0) -%}

  {%- set monthindex = month-1 -%}
  {%- if month == 2 and isleapyear -%}
    {{ daysinmonths[monthindex]+1 }}
  {%- else -%}
    {{ daysinmonths[monthindex] }}
  {%- endif -%}
{%- endmacro -%}
{% endraw %}```

### nth_dayofmonth(nth, dayofweek, month, year)
ex. Get the nth Monday of May.<br/>
1st: `{% raw %}{{ nth_dayofmonth(1, "Monday", 5) }}{% endraw %}`<br/>
2nd: `{% raw %}{{ nth_dayofmonth(2, "Monday", 5, 2020) }}{% endraw %}`<br/>
Last: `{% raw %}{{ nth_dayofmonth("last", "Monday", 5) }}{% endraw %}`

Reference: [bennadel.com](https://www.bennadel.com/blog/1446-getting-the-nth-occurrence-of-a-day-of-the-week-for-a-given-month.htm)
```yaml{% raw %}
{%- macro nth_dayofmonth(nth, dayofweek, month, year=now().strftime("%Y")) -%}
  {%- set dayofweek = dayofweek_number(dayofweek)|int -%}
  {%- set firstdateofmonth = strptime(year ~"-"~ month ~"-1", "%Y-%m-%d") -%}
  {%- set firstdayofmonth = dayofweek_number(firstdateofmonth.strftime("%A"))|int -%}

  {# Determine the first occurrence of the day. #}
  {%- if firstdayofmonth == 1 -%}
    {%- set firstoccurrence = dayofweek -%}
  {%- elif firstdayofmonth < dayofweek -%}
    {%- set firstoccurrence = (dayofweek - dayofweek_number(firstdayofmonth)) -%}
  {%- else -%}
    {%- set firstoccurrence = (7 - firstdayofmonth + dayofweek) + 1 -%}
  {%- endif -%}

  {%- if nth is number -%}
    {# Determine the nth occurrence of the dayofweek. #}
    {%- set nthoccurrence = firstoccurrence + 7 * (nth-1) -%}
  {%- else -%}
    {#
    Determine the LAST occurrence of the dayofweek.

    Reference: https://cflib.org/udf/GetLastOccOfDayInMonth
    #}
    {%- set lastdayofmonth = last_dayofmonth(month, year)|int -%}
    {%- set lastdayname = strptime(year ~"-"~ month ~"-"~ lastdayofmonth, "%Y-%m-%d").strftime("%A") -%}
    {%- set lastdaynumber = dayofweek_number(lastdayname)|int -%}
    {%- set daydifference = lastdaynumber - dayofweek -%}

    {# Add a week if the result is negative. #}
    {%- if daydifference < 0 -%}
      {%- set daydifference = daydifference + 7 -%}
    {%- endif -%}

    {%- set nthoccurrence = lastdayofmonth - daydifference -%}
  {%- endif -%}

  {# Return the day with the month and year so it can be useful. #}
  {{ strptime(month ~"/"~ nthoccurrence ~"/"~ year, "%m/%d/%Y") }}
{%- endmacro -%}
{% endraw %}```

## Time

### Standard Examples

```yaml{% raw %}
{% set time = as_timestamp(now())|timestamp_custom("%h:%M %p") %}

{% set hour = now().strftime("%I")|int %}
{% set minutes = now().strftime("%M")|int %}
{% set seconds = now().strftime("%S")|int %}
{% set ampm = now().strftime("%p")|int %}
{% set unix_timestamp = as_timestamp(now())|int %}
{% endraw %}```

### Between hours

```yaml{% raw %}
{% if now().strftime("%H")|int < 12 and now().strftime("%H")|int > 6%}
0.25
{% elif now().strftime("%H")|int > 12 and now().strftime("%H")|int < 17%}
0.40
{% else %}
0.20
{% endif %}
{% endraw %}```

### timedelta()

```yaml{% raw %}
# 15 minutes ago.
{{ as_timestamp(now()) - timedelta(minutes=15) }}

# 12 hours from now.
{{ as_timestamp(now()) + timedelta(hours=12) }}
{% endraw %}```

### Time difference

```yaml{% raw %}
# How long has it been since the last update?
{% set seconds_difference = (as_timestamp(now()) - as_timestamp(last_update)) %}
{% set minutes_difference = (seconds_difference) / 60 %}
{% set hours_difference   = (seconds_difference) / 3600 %}
{% endraw %}```

### `add_time()`

```yaml{% raw %}
{% macro add_time(time, add_minutes) %}
  {% if time|lower != "unavailable" %}
    {% set time = time.split(":") %}
    {% set hour = time[0]|int %}
    {% set minutes = time[1]|int %}
    {% if (minutes + add_minutes) < 60 %}
      {{ "%0.02d:%0.02d"|format(hour, minutes + add_minutes) }}
    {% else %}
      {{ "%0.02d:%0.02d"|format(hour + 1, (minutes + add_minutes) - 60) }}
    {% endif %}
  {% endif %}
{% endmacro %}


# Trigger the automation 5 minutes after the wakeup time.
{% set add_minutes = 5 %}
{% set current_time = states("sensor.time") %}
{% set boys = add_time(states("sensor.boys_room_next_alarm"),add_minutes) %}
{% set brian = add_time(states("sensor.wakeup_brian_time"),add_minutes) %}
{% set nerene = add_time(states("sensor.wakeup_nerene_time"),add_minutes) %}

{{ current_time in [boys,brian,nerene] }}
{% endraw %}```

### `subtract_time()`

```yaml{% raw %}
{% macro subtract_time(time, subtract_minutes) %}
  {% if time|lower != "unavailable" %}
    {% set time = time.split(":") %}
    {% set hour = time[0]|int %}
    {% set minutes = time[1]|int %}
    {% if (minutes|int >= subtract_minutes) %}
      {{ "%0.02d:%0.02d"|format(hour, minutes - subtract_minutes) }}
    {% else %}
      {{ "%0.02d:%0.02d"|format(hour - 1, minutes + (60-subtract_minutes)) }}
    {% endif %}
  {% endif %}
{% endmacro %}

# Trigger the automation 30 minutes before the wakeup time.
{% set subtract_minutes = 30 %}
{% set current_time = states("sensor.time") %}
{% set boys = subtract_time(states("sensor.boys_room_next_alarm"),subtract_minutes) %}
{% set brian = subtract_time(states("sensor.wakeup_brian_time"),subtract_minutes) %}
{% set nerene = subtract_time(states("sensor.wakeup_nerene_time"),subtract_minutes) %}

{{ current_time in [boys,brian,nerene] }}
{% endraw %}```
