# Home-Assistant-Weather-dependent-car-heater
For some reason I could not get the automatic car heaters found online to work. While figuring out how they work, i decided to make my own with quite simple logic.

## First set up some helpers
Create a time helper to set up your desired departure time:

![image](https://github.com/user-attachments/assets/e39b9b19-3c52-45ae-ba27-85657e6dd6fa)

Create a toggle helper to enable your car heater:

![image](https://github.com/user-attachments/assets/60ecf4d4-5038-47cd-b4bb-c0a9da18997d)


To set up the time when the heater should begin to warm up your car, create a template helper:
```jinja2
{% set time_start_delta = timedelta(minutes=(state_attr('weather.forecast_koti','temperature') -10)*5.5 | int) %}
{% set time_departure = today_at(timedelta(hours=state_attr('input_datetime.lahtoaika', 'hour'), minutes=state_attr('input_datetime.lahtoaika', 'minute'))) %}
{% set time_start = time_start_delta + time_departure %}
{{ time_start }}
```
in this `state_attr('weather.forecast_koti','temperature'` is your outside temperature in degrees of celsius.
`input_datetime.lahtoaika` is your departure time helpers name.
Change accordingly if needed.
Like this: if you have a temperature sensor like a ruuvitag
```jinja2
{% set time_start_delta = timedelta(minutes=((states('sensor.ruuvitag_2b8c_temperature') | float) -10)*5.5 | int) %}
{# Calculate the time delta based on the current temperature from the Ruuvitag sensor.
   - Convert the sensor value to a float.
   - Subtract 10 from the temperature.
   - Multiply the result by 5.5 to calculate the offset in minutes.
   - Cast the result to an integer to ensure it's a valid input for timedelta.
   - Create a timedelta object with the calculated number of minutes. #}

{% set time_departure = today_at(timedelta(hours=state_attr('input_datetime.lahtoaika', 'hour'), minutes=state_attr('input_datetime.lahtoaika', 'minute'))) %}
{# Determine the departure time by retrieving the hours and minutes from the input_datetime entity `lahtoaika`.
   - Use `today_at()` to create a datetime object for today with the given hour and minute attributes.
   - This represents the scheduled departure time for today. #}

{% set time_start = time_start_delta + time_departure %}
{# Add the calculated time delta to the departure time to determine the start time.
   - This represents when the process (e.g., preheating) should start. #}

{{ time_start }}
{# Output the calculated start time. #}

```
After a morning when i didn't get temperature reading of the ruuvitag and it's value was "unavailable" i made a number helper (number.viimeisin_ulkolampotila) to store last known temperature. Extra added redundancy was implemented in the helper in form of few if-statements
```jinja2
{% if states('sensor.ruuvitag_2b8c_temperature') != 'unavailable' %}
  {# Check if the Ruuvitag sensor value is available (not 'unavailable'). #}
  {{ states('sensor.ruuvitag_2b8c_temperature') }}
  {# If available, use the temperature value from the Ruuvitag sensor. #}
{% elif state_attr('weather.forecast_koti', 'temperature') is not none %}
  {# If the Ruuvitag sensor value is unavailable, check if the temperature attribute 
      of the weather forecast entity ('weather.forecast_koti') is not None. #}
  {{ state_attr('weather.forecast_koti', 'temperature') }}
  {# If it's not None, use the temperature from the weather forecast entity. #}
{% else %}
  {# If neither the Ruuvitag sensor nor the weather forecast entity provides a usable value, 
      fall back to the value of 'number.viimeisin_ulkolampotila'. #}
  {{ states('number.viimeisin_ulkolampotila') }}
{% endif %}
```
This template helper sets time for the heater to shut off, whit included 30 minutes delay if you are running late in the morning:
```jinja2
{% set time_departure = today_at(timedelta(hours=state_attr('input_datetime.lahtoaika', 'hour'), minutes=state_attr('input_datetime.lahtoaika', 'minute'))) %}
{% set time_offset_stop = timedelta(minutes=30) %}
{% set time_stop = time_departure + time_offset_stop %}
{{ time_stop  }}
```
Again, `input_datetime.lahtoaika` is your departure time helpers name.

## Then few automations

Create an automation that turns car heater on:
```yaml
alias: Ford lämmityksen hallinta
description: Controls the heating based on start and end times.
triggers:
  - value_template: |
      {{ now() >= as_datetime(states('sensor.ford_lammityksen_aloitus')) or
         now() >= as_datetime(states('sensor.ford_lammitys_pois')) }}
    trigger: template
conditions:
  - condition: or
    conditions:
      - condition: state
        entity_id: input_boolean.ford_vain_arkipaivisin
        state: "off"
      - condition: time
        weekday:
          - mon
          - tue
          - wed
          - thu
          - fri
actions:
  - choose:
      - conditions:
          - condition: template
            value_template: >
              {{ now() >= as_datetime(states('sensor.ford_lammityksen_aloitus'))
              and
                 now() < as_datetime(states('sensor.ford_lammitys_pois')) }}
          - condition: state
            entity_id: input_boolean.ford_lammitin_paalla
            state: "on"
        sequence:
          - condition: device
            type: is_off
            device_id: 8a0fde4688ab3134ee939e17a6bd1b1f
            entity_id: e2ac351a439b47cfa18fdfea25130c18
            domain: switch
          - target:
              entity_id: switch.shellyplug_s_eafe05
            action: switch.turn_on
            data: {}
      - conditions:
          - condition: template
            value_template: |
              {{ now() >= as_datetime(states('sensor.ford_lammitys_pois')) }}
          - condition: state
            entity_id: input_boolean.ford_lammitin_paalla
            state: "on"

        sequence:
          - condition: device
            type: is_on
            device_id: 8a0fde4688ab3134ee939e17a6bd1b1f
            entity_id: e2ac351a439b47cfa18fdfea25130c18
            domain: switch
          - target:
              entity_id: switch.shellyplug_s_eafe05
            action: switch.turn_off
            data: {}
    default: []
mode: single

```

Create an automation that turns the power off if there is no car plugged in or you unplug the car:
```yaml
alias: Sammuta auton lämmitys jos ei kulutusta
description: ""
triggers:
  - type: turned_on
    device_id: 8230786f881a0cb5c19de5507a45dd80
    entity_id: f4f0421d354b4f28c32ef492ea565100
    domain: switch
    trigger: device
    for:
      hours: 0
      minutes: 1
      seconds: 0
  - type: power
    device_id: 8230786f881a0cb5c19de5507a45dd80
    entity_id: 8ec3aab09417c87e20935387024781ba
    domain: sensor
    trigger: device
    below: 1
    for:
      hours: 0
      minutes: 1
      seconds: 0
conditions:
  - type: is_power
    condition: device
    device_id: 8230786f881a0cb5c19de5507a45dd80
    entity_id: 8ec3aab09417c87e20935387024781ba
    domain: sensor
    below: 1
actions:
  - type: turn_off
    device_id: 8230786f881a0cb5c19de5507a45dd80
    entity_id: f4f0421d354b4f28c32ef492ea565100
    domain: switch
mode: single
```
