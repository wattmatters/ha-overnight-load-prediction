# ha-overnight-load-prediction

A Home Assistant automation **shared as an example** that predicts overnight electricity load by setting a boolean flag based on weather forecasts, indoor temperatures, and current HVAC usage patterns.

The primary use case is battery energy management — knowing whether the overnight load will be high (e.g. aircon running for heating or cooling) allows a battery export calculator to protect more reserve capacity before discharging during a peak feed-in period.

This is posted as a reference to adapt to your own system. The temperature thresholds, climate entities, and trigger time are specific to the author's location and household and will need adjusting.

---

## Acknowledgements

This automation was developed with assistance from [Claude](https://claude.ai), Anthropic's AI assistant.

---

## What it does

Triggers at a configurable time each afternoon (and whenever the weather forecast or aircon state changes) and:

1. Fetches the hourly weather forecast for overnight (midnight to 6am)
2. Calculates the minimum overnight outdoor temperature
3. Reads current indoor temperatures from HVAC climate entities
4. Applies configurable thresholds to predict whether heating or cooling will run overnight
5. Sets `input_boolean.aircon_expected_overnight` on or off accordingly
6. Logs the decision to the Home Assistant system log

---

## Adapting for other overnight loads

The same pattern can be adapted for other predictable overnight loads:

- **EV charging** — set the flag if the car is home and below a target SOC
- **Hot water heat pump** — set the flag based on a schedule or temperature sensor
- **Pool pump / filter** — set based on schedule or season
- **Any scheduled high load** — combine multiple flags into a composite overnight load estimate

The key principle is the same: predict the load *before* the peak export period so the battery calculator can reserve enough capacity.

---

## Requirements

### Weather integration
A weather integration providing **hourly forecasts** with temperature data. The `weather.get_forecasts` action with `type: hourly` is used. Compatible integrations include:

- [Bureau of Meteorology (BOM)](https://github.com/bremor/bureau_of_meteorology) (Australia)
- [Open-Meteo](https://github.com/natekspencer/hacs-open_meteo)
- [OpenWeatherMap](https://www.home-assistant.io/integrations/openweathermap/)
- Any integration that exposes an hourly forecast weather entity

### HVAC / climate entities
One or more `climate` entities representing your heating/cooling system. The automation reads:
- `current_temperature` attribute (current indoor temperature)
- Entity state (`heat`, `cool`, `off`, etc.)

### Helper entity
Create this in **Settings → Devices & Services → Helpers**:

| Entity | Type | Notes |
|---|---|---|
| `input_boolean.aircon_expected_overnight` | Toggle | Set by this automation; read by the battery export calculator |

---

## Installation

1. Create the `input_boolean.aircon_expected_overnight` helper.

2. Copy `predict_overnight_aircon_usage.yaml` to your `config/automations/` directory, or paste into the Automation editor → Edit in YAML.

3. Find and replace all `← REPLACE` and `← ADJUST` markers:

| Placeholder | What to replace with |
|---|---|
| `weather.YOUR_WEATHER_ENTITY_HOURLY` | Your hourly weather entity ID |
| `climate.YOUR_AIRCON_CONTROLLER_1` | Your first climate entity |
| `climate.YOUR_AIRCON_CONTROLLER_2` | Your second climate entity (or remove) |
| `indoor_temp_1`, `indoor_temp_2` | Match to your climate entities; add/remove zones |
| `avg_indoor_temp` calculation | Adjust divisor if you have more/fewer zones |
| `any_heating` / `any_cooling` | Match to your climate entities |
| Temperature thresholds | Adjust to your climate and comfort preferences |
| `"17:00:00"` trigger time | Adjust to run before your peak export period |

4. Reload automations or restart Home Assistant.

---

## Temperature threshold guide

The default thresholds reflect a temperate coastal Australian climate:

| Threshold | Default | Meaning |
|---|---|---|
| `cold_threshold` | `10°C` | Outdoor overnight min below this = potentially cold |
| `indoor_cold_threshold` | `20°C` | Indoor temp below this + cold outside = heating likely |
| `hot_threshold` | `24°C` | Outdoor overnight min above this = cooling likely |
| `indoor_hot_threshold` | `26°C` | Indoor temp above this = cooling likely |

Adjust these significantly if you live in a different climate. For a continental climate with cold winters you might lower `cold_threshold` to 5°C; for a tropical climate you might raise `hot_threshold` to 28°C.

---

## How it integrates with battery management

This automation is referenced by the [ha-sigenergy-automations](https://github.com/wattmatters/ha-sigenergy-automations) collection as a supporting input to the Battery Export Calculator Node-RED flow. The flow reads `input_boolean.aircon_expected_overnight` and adds an extra battery reserve (`AIRCON_ADDITIONAL_RESERVE_KWH`) when the flag is set, ensuring enough capacity remains for overnight aircon load.

The same pattern works for any battery management system that needs to account for overnight loads — replace the aircon logic with whatever overnight loads are significant in your household.

---

## Known limitations / notes

- The overnight temperature extraction looks for the **next occurrence of midnight** in the forecast. If run after midnight, it may look at the following night. Run the automation in the afternoon (e.g. 5 PM) to reliably get tonight's forecast.
- The automation uses a 48-hour forecast window to find midnight. If your weather integration provides fewer hours, the `overnight_temps` list may be empty and the fallback temperature of `20°C` will be used.
- Tested on **Home Assistant OS 17.3 / Core 2026.5.2** in the NSW NEM region with a Bureau of Meteorology hourly weather entity and iZone ducted HVAC system.

---

## Contributing

Shared as a starting point. If you adapt it for different overnight loads or climate conditions, feel free to open a PR or issue.

---

## License

MIT — do whatever you like, no warranty implied.
