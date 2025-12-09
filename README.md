# thermia-genesis-calibra

Information meant for integration of Thermia heatpumps (genesis platform) in Home Asssistant;
Modbus BMS must be enabled on your Thermia system (I use modbus TCP).

DISCLAIMER: use at your own risk, don't complain if it eats your dog or burns your house ....

For the Calibra, I managed to use the https://github.com/CJNE/thermiagenesis Home Assistant integration, which works fine for most parameters.
Unfortunately, this integration currently misses some interesting registers:
 - BMS supplied outside temperature
 - Energy performance parameters (COP today for heat and tapwater)
 - Condenser pump continuous operation
 - Comfort wheel with finer 0.1 °C granularity (limited and with faulty scaling for room sensor)
 - ...

As I currently have no time to contribute to the mentioned integration, I added these modbus entities manually.
These missing registers can be enabled by adding a modbus section to your config/configuration.yaml file:


```
modbus:
 - name: "Thermia Calibra"
   type: tcp
   host: "10.x.y.z"
   port: 502
   switches:
    - name: "BMS controlled temperature"
      address: 117
      unique_id: "thermia_bms_controlled_temperature_switch"
      write_type: holding
      command_on: 1
      command_off: 0
      verify:
        input_type: holding
        address: 117
        state_on: 1
        state_off: 0
    - name: "Condenser Pump Continuous Operation"
      address: 59
      unique_id: "condenser_pump_continuous_operation"
      write_type: coil
      command_on: 1
      command_off: 0
      verify:
        input_type: coil
        address: 59
        state_on: 1
        state_off: 0
   climates:
    - name: "BMS outdoor temperature"
      address: 13
      unique_id: "thermia_bms_outdoor_temperature_climate"
      input_type: input
      #count: 1
      data_type: int16
      max_temp: 50
      min_temp: -20
      offset: 0
      precision: 1
      scale: 0.01
      #structure: ">h"
      target_temp_register: 118
      target_temp_write_registers: false
      temp_step: 0.1
      temperature_unit: C
    - name: Thermia comfort wheel
      address: 121
      input_type: input
      max_temp: 25
      min_temp: 10
      precision: 2
      scale: 0.01
      temp_step: 0.1
      target_temp_register: 5
      temperature_unit: C
   sensors:
    - name: Heat energy consumed today
      address: 331
      device_class: energy
      unit_of_measurement: 'kWh'
      input_type: input
      scan_interval: 120
      slave: 1
      unique_id: Thermia_heat_energy_consumed_today
    - name: Heat energy delivered today
      address: 332
      device_class: energy
      unit_of_measurement: 'kWh'
      input_type: input
      scan_interval: 120
      slave: 1
      unique_id: Thermia_heat_energy_delivered_today
    - name: Heat COP today
      address: 333
      unit_of_measurement: state
      scale: 0.1
      precision: 1
      input_type: input
      scan_interval: 120
      slave: 1
      unique_id: Thermia_heat_COP_today
    - name: Heat energy from natural source today
      address: 334
      device_class: energy
      unit_of_measurement: 'kWh'
      input_type: input
      scan_interval: 120
      slave: 1
      unique_id: Thermia_heat_energy_delivered_today
# following entries are guesses - not confirmed and probably wrong 
    - name: Heat COP year
      address: 354
      unit_of_measurement: state
      scale: 0.1
      precision: 1
      input_type: input
      scan_interval: 120
      slave: 1
      unique_id: Thermia_heat_COP_year
    - name: Tapwater energy consumed today
      address: 336
      device_class: energy
      unit_of_measurement: 'kWh'
      input_type: input
      scan_interval: 120
      slave: 1
      unique_id: Thermia_tapwater_energy_consumed_today
    - name: Tapwater energy delivered today
      address: 337
      device_class: energy
      unit_of_measurement: 'kWh'
      input_type: input
      scan_interval: 120
      slave: 1
      unique_id: Thermia_tapwater_energy_delivered_today
    - name: Tapwater COP today
      address: 338
      unit_of_measurement: state
      scale: 0.1
      precision: 1
      input_type: input
      scan_interval: 120
      slave: 1
      unique_id: Thermia_tapwater_COP_today
    - name: Tapwater energy from natural source today
      address: 339
      device_class: energy
      unit_of_measurement: 'kWh'
      input_type: input
      scan_interval: 120
      slave: 1
      unique_id: Thermia_tapwater_energy_delivered_today
    - name: Mix valve 1 position
      address: 298
      input_type: holding
      scan_interval: 120
      slave: 1
      unique_id: Mix_valve_1_position
`# end of guesses
    - name: Thermia Comfort wheel setting
      address: 5
      input_type: holding
      scan_interval: 120
      slave: 1
      unique_id: thermia_comfort_wheel_setting
      scale: 0.01
      precision: 1
      unit_of_measurement: '°C'

```

As some of these registers are not (yet) documented, I have to issue a severe disclaimer: use at your own risk, don't complain if it eats your dog or burns your house !!


This has been tested on both Calibra Cool 7 and Calibra Cool Eco 8 systems running software version 16 and 17.

Installation: Add the content of this section to your config/configuration.yaml file, edit to provide the correct IP address and restart your system
Add the new entities to your preferred dashboard.

<img width="394" alt="image" src="https://github.com/user-attachments/assets/ef86d751-f1ab-494d-9057-7dd207164135">

I use the BMS controlled temperature to use the 4 hour offset temperature forecast, as my floor heating has almost a 4 hour delay.

<img width="433" alt="image" src="https://github.com/user-attachments/assets/0916b08a-05db-4fae-a5c6-0138bc0bc766">


The 4-hour forecast is derived from the met.no weather integration and is polled every hour with following section in config/configuration.yaml

```
template:
  - trigger:
      - platform: time_pattern
        hours: /1
      - platform: event
        event_type: event_template_reloaded
    action:
      - variables:
          w: weather.forecast_home # you may have named it differently
      - service: weather.get_forecasts
        data:
          type: hourly
        target:
          entity_id: '{{ w }}'
        response_variable: hourly
    sensor:
      - name: "temperature forecast 4h"
        unique_id: temperature_forecast_4h
        unit_of_measurement: "°C"
        device_class: temperature
        state: "{{ hourly[w].forecast[3].temperature }}"

```
An automation is used to propagate changes in this forecast to the Thermia outside temperature:

```
alias: Thermia buitentemperatuur forecast
description: ""
triggers:
  - trigger: state
    entity_id:
      - sensor.temperature_forecast_4h
  - trigger: state
    entity_id:
      - switch.bms_controlled_temperature
conditions: []
actions:
  - action: climate.set_temperature
    metadata: {}
    data:
      temperature: "{{ states('sensor.temperature_forecast_4h') }}"
    target:
      entity_id: climate.bms_outdoor_temperature
mode: single

```
