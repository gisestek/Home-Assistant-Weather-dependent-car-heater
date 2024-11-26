# Home-Assistant-Weather-dependent-car-heater

Timer helper, template:
{% set time_start_delta = timedelta(minutes=(state_attr('weather.forecast_koti','temperature') -10)*5.5 | int) %}
{% set time_departure = today_at(timedelta(hours=state_attr('input_datetime.lahtoaika', 'hour'), minutes=state_attr('input_datetime.lahtoaika', 'minute'))) %}
{% set time_start = time_start_delta + time_departure %}
{{ time_start }}


Timer helper, template:
{% set time_departure = today_at(timedelta(hours=state_attr('input_datetime.lahtoaika', 'hour'), minutes=state_attr('input_datetime.lahtoaika', 'minute'))) %}
{% set time_offset_stop = timedelta(minutes=30) %}
{% set time_stop = time_departure + time_offset_stop %}
{{ time_stop  }}


Automation:

alias: Auto lämpenee
description: ""
triggers:
  - trigger: time
    at: sensor.heater_start_time
  - trigger: state
    entity_id:
      - binary_sensor.auto1_heater_helper
    to: "on"
conditions:
  - condition: state
    entity_id: input_boolean.auton_lammityksen_ajastus_paalla
    state: "on"
  - condition: or
    conditions:
      - condition: state
        entity_id: input_boolean.only_weekdays
        state: "off"
      - condition: time
        weekday:
          - mon
          - tue
          - wed
          - thu
          - fri
actions:
  - type: turn_on
    device_id: 8230786f881a0cb5c19de5507a45dd80
    entity_id: f4f0421d354b4f28c32ef492ea565100
    domain: switch
mode: single
