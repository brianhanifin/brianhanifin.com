---
title: "Home Assistant Template Macros: Date and Time"
date: 2020-01-18 21:00:00 -0700
categories: [Code Snippets]
tags: [Home Assistant, Jinja]
seo:
  date_modified: 2022-05-10 19:45:00 -0700
---

Over time I have created a large library of date and time manipulation code which are used in my
automations and scripts. I plan to update this post with the snippets as I add to my library.

## Date

### Standard Examples

```yaml{% raw %}
{% set date = as_timestamp(now())|timestamp_custom("%A %B %-d, %Y") %}
{% set datetime = as_timestamp(now()) | timestamp_custom("%I:%M:%S %p %b/%d/%Y", true) %}
{% set now_string = now().strftime("%Y-%m-%d") %}

{% set month_name = now().strftime("%B") %}
{% set day_name = now().strftime("%A")|lower %}

{% set this_month = now().month %}
{% set this_day = now().day %}
{% set this_year = now().year %}
{% endraw %}```

### timedelta()

```yaml{% raw %}
# Calculate fifteen days ago.
{{ ( now() - timedelta(days=15) ).date() }}

# Calculate one week from today.
{{ ( now() + timedelta(weeks=1) ).date() }}
{% endraw %}```

### dayofweek_number(dayofweek)
ex. Get the number which represents Thursday.
`{% raw %}{{ dayofweek_number("Thursday") }}{% endraw %}`

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
ex. Get the last day of February 2020.
`{% raw %}{{ last_dayofmonth(2, 2020) }}{% endraw %}`

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
{%- macro nth_dayofmonth(nth, dayofweek, month, year=now().year) -%}
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
{# Current time using templates instead of requiring sensor.time. #}
{% set current_time = "%02d:%02d:%02d"|format(now().hour, now().minute, now().second) %}

{# Convert datetime to a timestamp. #}
{% set current_time = as_timestamp(now())|timestamp_custom("%I:%M %p") %}
{% set unix_timestamp = as_timestamp(now())|int %}

{# 24 hour version #}
{% set hour = now().hour %}

{# 12 hour version #}
{% set hour = now().strftime("%I")|int %}
{% set hour = iif(now().hour>12, now().hour-12, now().hour) %}

{% set minutes = now().minute %}
{% set seconds = now().second %}
{% set ampm = now().strftime("%p") %}
{% endraw %}```

### Between hours
```yaml{% raw %}
# Standard way
{% if 6 <= now().hour and now().hour < 12 %}
0.25
{% elif 12 <= now().hour and now().hour < 17 %}
0.40
{% else %}
0.20
{% endif %}

# Shorter way
{% if 6 <= now().hour < 12 %}
0.25
{% elif 12 <= now().hour < 17 %}
0.40
{% else %}
0.20
{% endif %}
{% endraw %}```

### Add or subtract time

```yaml{% raw %}
# 15 minutes ago.
{{ as_timestamp(now()) - timedelta(minutes=15) }}

# 12 hours from now.
{{ as_timestamp(now()) + timedelta(hours=12) }}
{% endraw %}```

### Difference between two times

```yaml{% raw %}
# How long has it been since the last update?
{% set seconds_difference = (as_timestamp(now()) - as_timestamp(last_update)) %}
{% set minutes_difference = (seconds_difference) / 60 %}
{% set hours_difference   = (seconds_difference) / 3600 %}
{% endraw %}```

### Add or subtract time
Pass a negative number to substract minutes instead of adding.
`{% raw %}add_minutes(start_time, -15)]{% endraw %}`

```yaml{% raw %}
{% macro add_minutes(start_time, minutes_to_add) %}
  {%- set now_datetime = now() %}
  {%- set now_time = "%02d:%02d:%02d"|format(now_datetime.hour, now_datetime.minute, now_datetime.second) %}
  {%- set new_datetime = now_datetime | replace(now_time, start_time) %}
  {%- set new_datetime = as_datetime(new_datetime) + timedelta(minutes=minutes_to_add) %}
  {%- set new_time = "%02d:%02d:%02d"|format(new_datetime.hour, new_datetime.minute, new_datetime.second) %}
  {{ new_time }}
{% endmacro %}

{% set minutes_to_add = 5 %}
{% set boys = add_time(states("sensor.boys_room_next_alarm"), minutes_to_add) %}
{% set brian = add_time(states("sensor.wakeup_brian_time"), minutes_to_add) %}
{% endraw %}```

### Bonus: generate a list of dates
I use the template code below to generate a list of dates to add summer break to a school day sensor.

```yaml{% raw %}
{%- macro last_dayofmonth(month, year) -%}
  {%- set daysinmonths = [31,28,31,30,31,30,31,31,30,31,30,31] -%}
  {%- set month = month|default(0)|int -%}
  {%- set year = year|default(0)|int -%}

  {# Simplified leap year calculation. See https://www.mathsisfun.com/leap-years.html #}
  {%- set isleapyear = year % 4 == 0 and (year % 100 != 0 or year % 400 == 0) -%}

  {%- set monthindex = month-1 -%}
  {%- if month == 2 and isleapyear -%}
    {{ daysinmonths[monthindex]+1 }}
  {%- else -%}
    {{ daysinmonths[monthindex] }}
  {%- endif -%}
{%- endmacro -%}

{%- set year = now().year %}
{%- set months = [6,7,8] %}
{%- set days = [1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31] %}
{%- for month in months %}
  {%- set lastday = last_dayofmonth(month, year)|int %}
  {%- for day in days if day <= lastday %}
- "{{ year ~"-"~ "%02d"|format(month) ~"-"~ "%02d"|format(day) }}"
  {%- endfor %}
{%- endfor %}
{% endraw %}```

configuration.yaml snippet

```yaml{% raw %}
binary_sensor:
  platform: workday
  name: School day
  country: US
  province: CA
  excludes: [sat, sun, holiday]
  remove_holidays:
    - Susan B. Anthony Day
  add_holidays:
    - "2022-04-04" # Spring recess
    - "2022-04-05"
    - "2022-04-06"
    - "2022-04-07"
    - "2022-04-08"
    - "2022-04-11"
    - "2022-04-12"
    - "2022-04-13"
    - "2022-04-14"
    - "2022-04-15"
{% endraw %}```
