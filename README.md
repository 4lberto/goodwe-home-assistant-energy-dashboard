# goodwe-home-assistant-energy-dashboard

## Why?

Because, even though you installed the Goodwe integration into Home Assistant, it is not straightforward to use the values of the Goodwe inverter in the **Energy** dashboard.
In this guide, you can configure sensors in Home Assistant to display values in the Energy Dashboard.

This is working for a **Goodwe GW5048D-ES** (castaña) inverter with a Goodwe Lynx home u battery. Probably it will work in yours, but maybe you have to adjust the `sensor.on_grid_export_power` Goodwe sensor and use the one provided by your inverter.

Source: https://github.com/mletenay/home-assistant-goodwe-inverter

### Prerequisites

- Have the Goodwe Home Assistant integration installed and working, so all the inverter sensors are read correctly by Home Assistant.
- Edit the `configuration.yaml` file and add the content of the file called `add_to_configuration.yaml`.

> **⚠️ Important:** Home Assistant has deprecated the legacy `sensor: -> platform: template -> sensors:` syntax. It will be removed in **Home Assistant 2026.6**. The configuration below now uses the modern `template:` syntax. If you are upgrading from an older version, please remove the old template sensors before copying the new ones.

---

## 1. Create new Sensors

Energy panel needs:

- **Separated sensors** for buying and selling energy from the grid, and the same for the energy charged or discharged from the battery. Since the Goodwe integration only provides 1 sensor for grid and another for battery with positive and negative values, it is necessary to split them into two different sensors.
- **Sensors that provide data in kWh** and not W. This means that we have to use Riemann sum to incorporate the "h" to the kWh.

### 1.1. Split grid sensor into buy and sell + Split battery (if you have) into charge and discharge

Edit your `configuration.yaml` and add this block **under a single top-level `template:` key**:

> **Note:** If your `configuration.yaml` already contains a `template:` section, do **not** add a second one. Merge the sensors under the existing `template:` key.

```yaml
template:
  - sensor:
      - name: "Energy Buy"
        unique_id: energy_buy
        device_class: power
        unit_of_measurement: "W"
        state: >
          {% if states('sensor.on_grid_export_power')|float < 0 %}
            {{ states('sensor.on_grid_export_power')|float * -1 }}
          {% else %}
            {{ 0 }}
          {% endif %}
        availability: "{{ states('sensor.on_grid_export_power') not in ['unknown', 'unavailable'] }}"

      - name: "Energy Sell"
        unique_id: energy_sell
        device_class: power
        unit_of_measurement: "W"
        state: >
          {% if states('sensor.on_grid_export_power')|float > 0 %}
            {{ states('sensor.on_grid_export_power')|float }}
          {% else %}
            {{ 0 }}
          {% endif %}
        availability: "{{ states('sensor.on_grid_export_power') not in ['unknown', 'unavailable'] }}"

      - name: "Energy Battery Charge"
        unique_id: energy_battery_charge
        device_class: power
        unit_of_measurement: "W"
        state: >
          {% if states('sensor.battery_power')|float < 0 %}
            {{ states('sensor.battery_power')|float * -1 }}
          {% else %}
            {{ 0 }}
          {% endif %}
        availability: "{{ states('sensor.battery_power') not in ['unknown', 'unavailable'] }}"

      - name: "Energy Battery Discharge"
        unique_id: energy_battery_discharge
        device_class: power
        unit_of_measurement: "W"
        state: >
          {% if states('sensor.battery_power')|float > 0 %}
            {{ states('sensor.battery_power')|float }}
          {% else %}
            {{ 0 }}
          {% endif %}
        availability: "{{ states('sensor.battery_power') not in ['unknown', 'unavailable'] }}"

      - name: "House Consumption"
        unique_id: house_consumption
        device_class: power
        unit_of_measurement: "W"
        state: >
          {{
            (states('sensor.pv_power')|float(0))
            + (states('sensor.energy_buy')|float(0))
            + (states('sensor.energy_battery_discharge')|float(0))
            - (states('sensor.energy_sell')|float(0))
            - (states('sensor.energy_battery_charge')|float(0))
          }}
        availability: >
          {{ states('sensor.pv_power') not in ['unknown', 'unavailable']
             and states('sensor.energy_buy') not in ['unknown', 'unavailable']
             and states('sensor.energy_battery_discharge') not in ['unknown', 'unavailable']
             and states('sensor.energy_sell') not in ['unknown', 'unavailable']
             and states('sensor.energy_battery_charge') not in ['unknown', 'unavailable']
          }}
```

> **Note:** The new energy sensors are based on `sensor.on_grid_export_power` for my GW5048D-ES inverter; maybe yours has a different sensor. Same for battery.

### 1.2. Create sensors that transform from W to kWh

Add these integration sensors under the **existing top-level `sensor:` key** in your `configuration.yaml`:

```yaml
sensor:
  - platform: integration
    source: sensor.energy_buy
    name: energy_buy_sum
    unit_prefix: k
    round: 1
    method: left

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

### 1.3. Add Utility Meters

From the kWh sensors we can create other calculations that can be useful in other Home Assistant places:

```yaml
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

### 1.4. Restart Home Assistant

**Restart Home Assistant** to apply the changes, so the sensors appear in the options.

---

## 2. Configure Energy Panel

Go to **Settings > Dashboard > Energy** and use the new sensors.

Here is my configuration:

<img src="https://github.com/4lberto/goodwe-home-assistant-energy-dashboard/blob/main/energy_panel.png?raw=true">

You can also include the **energy price** for buying and selling from the grid in the specific configuration of each measurement, like this:

<img src="https://github.com/4lberto/goodwe-home-assistant-energy-dashboard/blob/main/energy_grid_config.png?raw=true" width="400">

This is the final result:

<img src="https://github.com/4lberto/goodwe-home-assistant-energy-dashboard/blob/main/energy_panel_working.png?raw=true">

---

## 3. Savings, Amortisation & Cost Tracking (Optional)

A common request is to calculate how much money you **save** per day by self-consuming your solar production, or how much you **earn** by selling energy back to the grid.

Home Assistant's Energy Dashboard already provides cost tracking if you set the **static or dynamic prices** in the grid configuration (see step 2).
If you want **additional helper sensors** (e.g. for custom cards or automations), you can add the following optional templates under the same `template:` key:

```yaml
template:
  - sensor:
      - name: "Grid Buy Cost"
        unique_id: grid_buy_cost
        device_class: monetary
        unit_of_measurement: "EUR"
        state: "{{ states('sensor.energy_buy_daily')|float(0) * 0.20 }}"
        availability: "{{ states('sensor.energy_buy_daily') not in ['unknown', 'unavailable'] }}"

      - name: "Grid Sell Revenue"
        unique_id: grid_sell_revenue
        device_class: monetary
        unit_of_measurement: "EUR"
        state: "{{ states('sensor.energy_sell_daily')|float(0) * 0.05 }}"
        availability: "{{ states('sensor.energy_sell_daily') not in ['unknown', 'unavailable'] }}"

      - name: "PV Self Consumption Savings"
        unique_id: pv_self_consumption_savings
        device_class: monetary
        unit_of_measurement: "EUR"
        state: >
          {{
            (
              states('sensor.energy_buy_daily')|float(0)
              + states('sensor.house_consumption_daily')|float(0)
              - states('sensor.energy_sell_daily')|float(0)
            ) * 0.20
          }}
        availability: "{{ states('sensor.energy_buy_daily') not in ['unknown', 'unavailable'] }}"
```

> **How it works:**
> - `Grid Buy Cost`: energy bought from the grid × your buy price.
> - `Grid Sell Revenue`: energy sold to the grid × your sell price.
> - `PV Self Consumption Savings`: the amount of energy you did **not** have to buy because you produced it yourself, multiplied by your buy price.
>
> **Adjust the prices** (`0.20`, `0.05`, etc.) to match your local tariffs and currency.

These sensors can then be added to any Lovelace card or used in automations to build your own amortisation dashboard.

---

## 4. Troubleshooting / FAQ

### Q: I see warnings about `device_class: energy` in the logs
**A:** Make sure your power template sensors use `device_class: power` (not `energy`). The integration sensors that convert W → kWh will automatically get `device_class: energy`.

### Q: Home Assistant says the template platform is deprecated
**A:** You are using the old `sensor: -> platform: template -> sensors:` syntax. Migrate to the modern `template:` syntax shown in section 1.1. Remove the old definitions first, then copy the new ones.

### Q: The new template sensors do not appear after a restart
**A:** Check that you do **not** have duplicate top-level `template:` keys in your `configuration.yaml`. There can only be one. Merge all template sensors under a single `template:` key.
