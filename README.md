# ha-overnight-load-prediction

A Home Assistant automation **shared as an example** that predicts overnight electricity load and dynamically sets a minimum battery SOC floor before the evening peak export period.

The primary use case is battery energy management — knowing the expected overnight load (base consumption, heating, cooling, and any manual boost) allows the battery export calculator to protect sufficient reserve capacity before discharging during a peak feed-in period.

This is posted as a reference to adapt to your own system. Temperature thresholds, load estimates, climate entities, and battery capacity are all specific to the author's location and household and will need adjusting.

---

## Acknowledgements

This automation was developed with assistance from [Claude](https://claude.ai), Anthropic's AI assistant.

---

## What it does

Triggers at a configurable time each afternoon (and whenever the weather forecast or aircon state changes) and:

1. Fetches the hourly weather forecast for overnight (midnight to 6am)
2. Calculates the minimum overnight outdoor temperature
3. Reads current indoor temperatures from HVAC climate entities
4. Predicts whether heating or cooling will run overnight based on configurable thresholds
5. Calculates a temperature-scaled heating energy estimate
6. Computes an overnight SOC floor from four components:
   - **Outage reserve** — live from the inverter's ESS backup SOC setting
   - **Base load** — fixed estimate for overnight base consumption (lighting, fridge, standby, etc.)
   - **Aircon estimate** — temperature-scaled heating or fixed cooling estimate
   - **Manual boost** — optional top-up via `input_number.overnight_soc_boost`
7. Writes the calculated floor directly to `input_number.battery_minimum_soc_end_of_export`
8. Sets `input_boolean.aircon_expected_overnight` on or off
9. Sends a persistent notification with a full breakdown of the calculation

---

## Why calculate a dynamic SOC floor?

A fixed overnight reserve is a blunt instrument — it either leaves too much battery unused on mild nights, or not enough on cold nights. This automation calculates a floor that reflects the actual expected overnight demand, allowing the export calculator to discharge more aggressively on mild nights and conservatively on cold ones.

The key insight is: **there is no point exporting battery energy at a peak feed-in rate if you have to import from the grid the following morning at a higher rate.**

---

## SOC floor calculation

```
calculated_floor = outage_reserve%
                 + base_load%          (base_load_kwh / battery_capacity_kwh × 100)
                 + aircon%             (aircon_kwh / battery_capacity_kwh × 100)
                 + boost%              (manual top-up)
```

Capped at 100%.

### Temperature-scaled heating estimate

The heating kWh estimate scales with the forecast overnight minimum temperature. These values reflect one system's observed heating load — **adjust the bands and kWh values to match your climate and HVAC system**:

| Forecast overnight min | Heating kWh | Reserve % (32 kWh battery) |
|---|---|---|
| ≥ 12°C (no aircon active) | 0 kWh | 0% |
| ≥ 12°C (aircon already active) | 4 kWh | 13% |
| 10–12°C | 6 kWh | 19% |
| 7–10°C | 10 kWh | 31% |
| 4–7°C | 14 kWh | 44% |
| < 4°C | 18 kWh | 56% |

The cooling estimate (6 kWh) is a fixed value — to be refined with real summer data.

---

## Manual boost

`input_number.overnight_soc_boost` allows a manual top-up for situations where the calculated floor needs supplementing — guest nights, anticipated cold snaps, or any other exceptional circumstance. Set it to a non-zero value (e.g. 10%) before the 5 PM trigger and reset it to 0% afterwards.

---

## Requirements

### Weather integration
A weather integration providing **hourly forecasts** with temperature data. Compatible integrations include:

- [Bureau of Meteorology (BOM)](https://github.com/bremor/bureau_of_meteorology) (Australia)
- [Open-Meteo](https://github.com/natekspencer/hacs-open_meteo)
- [OpenWeatherMap](https://www.home-assistant.io/integrations/openweathermap/)
- Any integration that exposes an hourly forecast weather entity

### HVAC / climate entities
One or more `climate` entities. The automation reads:
- `current_temperature` attribute (current indoor temperature)
- Entity state (`heat`, `cool`, `off`, etc.)

### Sigenergy ESS backup SOC
The outage reserve is read live from `number.sigen_0_plant_ess_backup_soc` — the inverter's configured ESS backup SOC setting. This means the floor automatically adjusts if you change your backup reserve setting. Replace this entity if yours differs.

### Helper entities
Create these in **Settings → Devices & Services → Helpers**:

| Entity | Type | Notes |
|---|---|---|
| `input_boolean.aircon_expected_overnight` | Toggle | Set by this automation; also read by the Battery Export Calculator flow |
| `input_number.battery_minimum_soc_end_of_export` | Number | Written by this automation; read by the Peak Export Control automation |
| `input_number.overnight_soc_boost` | Number | Range 0–30%, step 5%, default 0% — manual top-up |

---

## Installation

1. Create all helper entities listed above.

2. Copy `predict_overnight_aircon_usage.yaml` to your `config/automations/` directory, or paste into the Automation editor → Edit in YAML.

3. Find and replace all `← REPLACE` and `← ADJUST` markers:

| Placeholder | What to replace with |
|---|---|
| `weather.YOUR_WEATHER_ENTITY_HOURLY` | Your hourly weather entity ID (both occurrences) |
| `climate.YOUR_HVAC_ZONE_1` | Your first climate entity |
| `climate.YOUR_HVAC_ZONE_2` | Your second climate entity (or remove) |
| `climate.YOUR_HVAC_MASTER_BED` | Additional zone (or remove) |
| `avg_indoor_temp` divisor | Adjust to match your number of zones |
| `any_heating` / `any_cooling` | Match to your climate entities |
| `battery_capacity_kwh` | Your usable battery capacity in kWh |
| `base_load_kwh` | Your estimated overnight base load in kWh |
| `number.sigen_0_plant_ess_backup_soc` | Your inverter's ESS backup SOC entity (if different) |
| Temperature thresholds | Adjust to your climate |
| Heating kWh bands | Adjust to your HVAC system's observed consumption |
| `"17:00:00"` trigger time | Adjust to run before your peak export period |

4. Reload automations or restart Home Assistant.

---

## Adapting for other overnight loads

The same pattern can be adapted for other predictable overnight loads by adding additional kWh estimates to the floor calculation:

- **EV charging** — add an EV reserve kWh if the car is home and below a target SOC
- **Hot water heat pump** — add a fixed kWh if the heat pump runs overnight
- **Pool pump / filter** — add based on schedule or season

The key principle: predict the load *before* the peak export period so the battery calculator can reserve enough capacity.

---

## How it integrates with battery management

This automation connects to two other components in the [ha-sigenergy-automations](https://github.com/wattmatters/ha-sigenergy-automations) collection:

1. **`input_number.battery_minimum_soc_end_of_export`** — written here, read by the [Peak Export Control](https://github.com/wattmatters/ha-sigenergy-automations/tree/main/peak-export-control) automation to limit how aggressively the battery discharges during the peak window
2. **`input_boolean.aircon_expected_overnight`** — written here, read by the [Battery Export Calculator](https://github.com/wattmatters/ha-sigenergy-automations/tree/main/node-red/battery-export-calculator) Node-RED flow as a cross-check

---

## Known limitations / notes

- The heating kWh estimates were developed for a temperate coastal Australian climate. They will need significant adjustment for other climates.
- The cooling estimate (6 kWh) is a fixed placeholder — real summer data should be used to refine it.
- The overnight temperature extraction looks for the **next occurrence of midnight** in the forecast. Run the automation in the afternoon (e.g. 5 PM) to reliably get tonight's forecast.
- `battery_capacity_kwh` is hardcoded. For a dynamic value, replace it with `states('sensor.sigen_0_inverter_1_rated_battery_capacity') | float(32)` — though note this is the *rated* capacity which may differ slightly from measured usable capacity.
- Tested on **Home Assistant OS 17.3 / Core 2026.5.2** in the NSW NEM region with a Bureau of Meteorology hourly weather entity and iZone ducted HVAC system.

---

## Contributing

Shared as a starting point. If you adapt it for different overnight loads, climate zones, or HVAC systems, feel free to open a PR or issue to share what you changed.

---

## License

MIT — do whatever you like, no warranty implied.
