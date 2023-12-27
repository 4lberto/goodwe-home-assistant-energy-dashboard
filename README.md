# goodwe-home-assistant-energy-dashboard
Sensors configuration in Home Assistant to display values in the Energy Dashboard

* This is working for a Goodwe GW5048D-ES inverter with a Goodwe Lynx home u battery

## Sensors creation
Prerequisites:
- Have the Goodwe Home Assistant extension installed and working, so all the inverter sensors are read correctly by Home Assitant.
- Edit the configuration/configuration.yaml file and add the content of the file called "add_to_configuration.yaml" 


```
sensor:
  - platform: template
    sensors:
      # Template sensor for values of energy bought (active_power < 0)
      energy_buy:
        device_class: energy
        friendly_name: "Energy Buy"
        unit_of_measurement: 'W'
          #        state_class: measurement
        value_template: >-
          {% if states('sensor.on_grid_export_power')|float < 0 %}
            {{ states('sensor.on_grid_export_power')|float * -1 }}
          {% else %}
            {{ 0 }}
          {% endif %}
      # Template sensor for values of energy sold (active_power > 0)
      energy_sell:
        device_class: energy
        friendly_name: "Energy Sell"
        unit_of_measurement: 'W'
          #        state_class: measurement
        value_template: >-
          {% if states('sensor.on_grid_export_power')|float > 0 %}
            {{ states('sensor.on_grid_export_power')|float }}
          {% else %}
            {{ 0 }}
          {% endif %}

      energy_battery_charge:
        device_class: energy
        friendly_name: "Energy Battery Charge"
        unit_of_measurement: 'W'
        value_template: >-
          {% if states('sensor.battery_power')|float < 0 %}
            {{ states('sensor.battery_power')|float * -1 }}
          {% else %}
            {{ 0 }}
          {% endif %}
      # Template sensor for values of energy sold (active_power > 0)
      energy_battery_discharge:
        device_class: energy
        friendly_name: "Energy Battery Discharge"
        unit_of_measurement: 'W'
        value_template: >-
          {% if states('sensor.battery_power')|float > 0 %}
            {{ states('sensor.battery_power')|float }}
          {% else %}
            {{ 0 }}
          {% endif %}

  # Sensor for Riemann sum of energy bought (W -> Wh)
  - platform: integration
    source: sensor.energy_buy
    name: energy_buy_sum
    unit_prefix: k
    round: 1
    method: left
  # Sensor for Riemann sum of energy sold (W -> Wh)
  - platform: integration
    source: sensor.energy_sell
    name: energy_sell_sum
    unit_prefix: k
    round: 1
    method: left
  - platform: integration
    source: sensor.energy_battery_charge
    name: energy_battery_charge_sum
    unit_prefix: k
    round: 1
    method: left
  # Sensor for Riemann sum of energy sold (W -> Wh)
  - platform: integration
    source: sensor.energy_battery_discharge
    name: energy_battery_discharge_sum
    unit_prefix: k
    round: 1
    method: left
  - platform: integration
    source: sensor.pv_power
    name: pv_power_sum
    unit_prefix: k
    round: 1
    method: left

utility_meter:
  energy_buy_daily:
    source: sensor.energy_buy_sum
    cycle: daily
  energy_buy_monthly:
    source: sensor.energy_buy_sum
    cycle: monthly
  energy_sell_daily:
    source: sensor.energy_sell_sum
    cycle: daily
  energy_sell_monthly:
    source: sensor.energy_sell_sum
    cycle: monthly
  house_consumption_daily:
    source: sensor.house_consumption_sum
    cycle: daily
  house_consumption_monthly:
    source: sensor.house_consumption_sum
    cycle: monthly

  energy_battery_charge_daily:
    source: sensor.energy_battery_charge_sum
    cycle: daily
  energy_battery_charge_monthly:
    source: sensor.energy_battery_charge_sum
    cycle: monthly
  energy_battery_discharge_daily:
    source: sensor.energy_battery_discharge_sum
    cycle: daily
  energy_battery_discharge_monthly:
    source: sensor.energy_battery_discharge_sum
    cycle: monthly

  pv_energy_daily:
    source: sensor.pv_power
    cycle: daily
  pv_energy_monthly:
    source: sensor.pv_power
    cycle: monthly
```

