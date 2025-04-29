# goodwe-home-assistant-energy-dashboard

## Why?

Because, even though you installed the Goodwe integration into Home Assistant, it is not straightforward to use the values of the Goodwee inverter in the "Energy" dashboard.
In this guide, you can configure sensors in Home Assistant to display values in the Energy Dashboard

This is working for a **Goodwe GW5048D-ES** (castaña) inverter with a Goodwe Lynx home u battery. Probably it will work in yours, but maybe you have to adjust the "sensor.on_grid_export_power" Goodwe sensor and use the one provided by your inverter.

Source: https://github.com/mletenay/home-assistant-goodwe-inverter

### Prerequisites:
- Have the Goodwe Home Assistant integration installed and working, so all the inverter sensors are read correctly by Home Assitant.
- Edit the configuration/configuration.yaml file and add the content of the file called "add_to_configuration.yaml" 


## 1. Create new Sensors

Energy panel needs:
- Sepparated sensors for buying and selling energy from the grid, and the same for the energy charged or discharged from the battery. Since Goodwe integration only provides 1 sensor for grid and other of battery with positive and negative values, it is neccesary to split them in two different sensors.
- Sensors that provide data in kWh and not kW. This means that we have to use Riemann sum to incorporate the "h" to the kWh

### 1.1. Split grid sensor into buy and sell + Split battery (if you have) into charge and discharge

Just edit your configuration.yaml and add this:

Note: The new energy sensors are based on **sensor.on_grid_export_power** for my GW5048D-ES inverter, maybe yours has a different sensor. Same for battery.

```
sensor:
  - platform: template
    sensors:
      # Template sensor for values of energy bought (active_power < 0)
      energy_buy:
        device_class: power
        friendly_name: "Energy Buy"
        unit_of_measurement: 'W'
          #        state_class: measurement
        value_template: >-
          {% if states('sensor.on_grid_export_power')|float < 0 %}
            {{ states('sensor.on_grid_export_power')|float * -1 }}
          {% else %}
            {{ 0 }}
          {% endif %}
        availability_template: "{{ states('sensor.on_grid_export_power') not in ['unknown', 'unavailable'] }}"
      # Template sensor for values of energy sold (active_power > 0)
      energy_sell:
        device_class: power
        friendly_name: "Energy Sell"
        unit_of_measurement: 'W'
          #        state_class: measurement
        value_template: >-
          {% if states('sensor.on_grid_export_power')|float > 0 %}
            {{ states('sensor.on_grid_export_power')|float }}
          {% else %}
            {{ 0 }}
          {% endif %}
        availability_template: "{{ states('sensor.on_grid_export_power') not in ['unknown', 'unavailable'] }}"
      energy_battery_charge:
        device_class: power
        friendly_name: "Energy Battery Charge"
        unit_of_measurement: 'W'
        value_template: >-
          {% if states('sensor.battery_power')|float < 0 %}
            {{ states('sensor.battery_power')|float * -1 }}
          {% else %}
            {{ 0 }}
          {% endif %}
        availability_template: "{{ states('sensor.battery_power') not in ['unknown', 'unavailable'] }}"
      # Template sensor for values of energy sold (active_power > 0)
      energy_battery_discharge:
        device_class: power
        friendly_name: "Energy Battery Discharge"
        unit_of_measurement: 'W'
        value_template: >-
          {% if states('sensor.battery_power')|float > 0 %}
            {{ states('sensor.battery_power')|float }}
          {% else %}
            {{ 0 }}
          {% endif %}
        availability_template: "{{ states('sensor.battery_power') not in ['unknown', 'unavailable'] }}"
      house_consumption:
        device_class: power
        friendly_name: "House Consumption"
        unit_of_measurement: 'W'
        value_template: >-
          {{
            (states('sensor.pv_power')|float(0))
            + (states('sensor.energy_buy')|float(0))
            + (states('sensor.energy_battery_discharge')|float(0))
            - (states('sensor.energy_sell')|float(0))
            - (states('sensor.energy_battery_charge')|float(0))
          }}
        availability_template: >-
          {{ states('sensor.pv_power') not in ['unknown', 'unavailable']
             and states('sensor.energy_buy') not in ['unknown', 'unavailable']
             and states('sensor.energy_battery_discharge') not in ['unknown', 'unavailable']
             and states('sensor.energy_sell') not in ['unknown', 'unavailable']
             and states('sensor.energy_battery_charge') not in ['unknown', 'unavailable']
          }}
```

Now **RESTART** Home assistant (Developer Tools -> Restart)

### 1.2. Create sensors that transforms from kW to kWh 

Continue editing configuration.yaml and add this code to the sensor parte:


```
#Continue from the code block above... Please, keep the indentation

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
  - platform: integration
    source: sensor.house_consumption
    name: house_consumption_sum
    unit_prefix: k
    round: 1
    method: left

```


### 1.3 Add to Utility meters

From the kWh sensors we can create other calculations that can be useful in other Home Assistant places

```
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
    source: sensor.pv_power_sum
    cycle: daily
  pv_energy_monthly:
    source: sensor.pv_power_sum
    cycle: monthly
```


### 1.4. Restart Home Assitant.

Yes, you have to RESTART Home Assistant to apply the changes, so the sensors appear in the options.

## 2. Configure Energy Panel

Just go to **Settings > Dashboard > Energy** and use the new sensors.

Here is my configuration:

<img src="https://github.com/4lberto/goodwe-home-assistant-energy-dashboard/blob/main/energy_panel.png?raw=true">

Maybe you want to include the energy price, both for buying and selling from the grid. You have to do it in the specific configuration of each measurement, like this:

<img src="https://github.com/4lberto/goodwe-home-assistant-energy-dashboard/blob/main/energy_grid_config.png?raw=true" width="400">

This is the final result:

<img src="https://github.com/4lberto/goodwe-home-assistant-energy-dashboard/blob/main/energy_panel_working.png?raw=true">



