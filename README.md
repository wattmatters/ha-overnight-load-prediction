# ha-overnight-load-prediction

A Home Assistant automation **shared as an example** that predicts overnight electricity load by setting a boolean flag based on weather forecasts, indoor temperatures, and current HVAC usage patterns.

The primary use case is battery energy management — knowing whether the overnight load will be high (e.g. aircon running for heating or cooling) allows a battery export calculator to protect more reserve capacity before discharging during a peak feed-in period.

This is posted as a reference to adapt to your own system. The temperature thresholds, climate entities, and trigger time are specific to the author's location and household and will need adjusting.

---

## Acknowledgements

This automation was developed with assistance from [Claude](https://claude.ai), Anthropic's AI assistant.

---

## Revision history

### v1.1 — indoor temperature averaging and forecast logic fix

Prompted by a real overnight battery-drain incident traced back to two separate issues in v1.0. Both are fixed in this revision; noting them here in case you're running an older copy or hit something similar.

**1. Indoor temperature now averages multiple real per-room sensors per HVAC zone, with failure tolerance.**

v1.0 read a single `current_temperature` attribute per zone, typically from a `climate` entity representing a whole ducted controller. In production this turned out to be unreliable for one zone: the controller's reported `current_temperature` tracked a smooth, perfectly linear ~0.5°C-per-8-minutes decline for hours, inconsistent with every real room sensor in that zone (which showed normal, noisy, physically plausible readings 8-10°C lower). This pattern is consistent with a ducted controller falling back to a synthesized/estimated value — some systems do this when a zone's temperature sensor isn't properly bound or has dropped out — rather than reporting a live probe reading.

v1.1 instead reads a list of real per-room sensors per zone, filters out anything `unavailable`/`unknown`/non-numeric before averaging, and averages the zones together (rather than a flat average of individual sensors) so a zone that loses most of its sensors isn't outvoted by one that hasn't. If you don't have per-room sensors, a single-entity list still works — see "Known limitations" below for a caution about trusting a ducted controller's own reported temperature at face value.

**2. `expect_heating`'s forecast/indoor-temp check changed from AND to OR.**

v1.0 only predicted heating (when aircon wasn't already running) if *both* the forecast overnight minimum was cold *and* the current indoor temperature was already cold. This AND condition can silently fail on exactly the nights it matters most: a day that runs unseasonably warm, followed by a forecast cold, clear night. At the time this automation needs to make its reserve decision (before the evening peak-export window), the house may still be sitting well above the indoor threshold purely because it hasn't had time to cool down since sunset — regardless of how cold the forecast says the night will get. In the incident that prompted this fix, the indoor-temp condition didn't clear until roughly 90 minutes *after* the export window had already closed, so the correct reserve was calculated far too late to protect the battery during export.

v1.1 changes this to OR: a cold forecast alone is now sufficient to predict heating, without waiting for the house to actually feel cold. The indoor-temp check is kept as an independent trigger too, so a house that's unexpectedly cold despite a mild forecast (e.g. an overcast day, doors left open) still gets flagged. This is a deliberate asymmetric-risk choice: over-predicting heating costs a modest amount of export revenue on a night that turns out mild; under-predicting it risks depleting the battery and importing at peak rates the next morning. The bands in `heating_kwh` are also relatively low at the top end (4 kWh for the mildest "expected" case), so a false-positive trigger is a small cost.

If you're relying on the `any_heating AND tonight_min < 14` branch for confirmed-running aircon, that logic is unchanged — it was never the problem.

---

## What it does

Triggers at a configurable time each afternoon (and whenever the weather forecast or aircon state changes) and:

1. Fetches the hourly weather forecast for overnight (midnight to 6am)
2. Calculates the minimum overnight outdoor temperature
3. Reads current indoor temperatures from real per-room sensors, grouped and averaged by HVAC zone
4. Applies configurable thresholds to predict whether heating or cooling will run overnight
5. Sets `input_boolean.aircon_expected_overnight` on or off accordingly
6. Logs the decision to the Home Assistant system log and a persistent notification

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

One or more `climate` entities representing your heating/cooling system, used only for **mode detection** (`heat`/`cool`/`off`).

### Indoor temperature sensors

One or more real per-room temperature sensors per HVAC zone, used for the **indoor temperature** calculation. Prefer these over a ducted controller's own `current_temperature` attribute where available — see "Known limitations" below.

### Helper entity

Create this in **Settings → Devices & Services → Helpers**:

| Entity                                    | Type   | Notes                                                         |
| ------------------------------------------ | ------ | ------------------------------------------------------------- |
| `input_boolean.aircon_expected_overnight` | Toggle | Set by this automation; read by the battery export calculator |

---

## Installation

1. Create the `input_boolean.aircon_expected_overnight` helper.

2. Copy `predict_overnight_aircon_usage.yaml` to your `config/automations/` directory, or paste into the Automation editor → Edit in YAML.

3. Find and replace all `← REPLACE` and `← ADJUST` markers:

| Placeholder                          | What to replace with                                          |
| ------------------------------------- | --------------------------------------------------------------- |
| `weather.YOUR_WEATHER_ENTITY_HOURLY` | Your hourly weather entity ID                                  |
| `climate.YOUR_HVAC_ZONE_1` / `_2`    | Your climate entity/entities (used for mode detection only)     |
| `zone_a_temps`, `zone_b_temps`       | Lists of your real per-room temperature sensors, grouped by zone; add/remove zones and rooms to match your system |
| `avg_indoor_temp` calculation        | Add another `{% if %}` block if you add a zone beyond A/B       |
| `any_heating` / `any_cooling`        | Match to your climate entities                                  |
| Temperature thresholds                | Adjust to your climate and comfort preferences                  |
| `"17:00:00"` trigger time            | Adjust to run before your peak export period                    |

4. Reload automations or restart Home Assistant.

---

## Temperature threshold guide

The default thresholds reflect a temperate coastal Australian climate:

| Threshold               | Default | Meaning                                                |
| ------------------------ | ------- | -------------------------------------------------------- |
| `cold_threshold`        | `10°C`  | Outdoor overnight min below this = potentially cold    |
| `indoor_cold_threshold` | `20°C`  | Indoor temp below this = heating likely (independent of forecast) |
| `hot_threshold`         | `24°C`  | Outdoor overnight min above this = cooling likely      |
| `indoor_hot_threshold`  | `26°C`  | Indoor temp above this = cooling likely                |

Adjust these significantly if you live in a different climate. For a continental climate with cold winters you might lower `cold_threshold` to 5°C; for a tropical climate you might raise `hot_threshold` to 28°C.

---

## How it integrates with battery management

This automation is referenced by the [ha-sigenergy-automations](https://github.com/wattmatters/ha-sigenergy-automations) collection as a supporting input to the Battery Export Calculator Node-RED flow. The flow reads `input_boolean.aircon_expected_overnight` and adds an extra battery reserve (`AIRCON_ADDITIONAL_RESERVE_KWH`) when the flag is set, ensuring enough capacity remains for overnight aircon load.

The same pattern works for any battery management system that needs to account for overnight loads — replace the aircon logic with whatever overnight loads are significant in your household.

---

## Known limitations / notes

- **A ducted controller's reported `current_temperature` isn't always trustworthy.** If you don't have per-room sensors and are using a `climate` entity's `current_temperature` attribute instead, be aware some ducted systems substitute a synthesized/estimated value when a zone's actual sensor isn't bound or has failed, rather than reporting `unavailable`. If a zone's temperature looks implausible against your other rooms, check its sensor binding on the HVAC controller itself, not just in Home Assistant.
- The forecast/indoor-temp heating check is OR'd, not AND'd (see Revision history v1.1) — this deliberately biases toward over-predicting heating rather than under-predicting it, given the asymmetric cost of a peak-rate grid import versus a little forgone export revenue.
- The overnight temperature extraction looks for the **next occurrence of midnight** in the forecast. If run after midnight, it may look at the following night. Run the automation in the afternoon (e.g. 5 PM) to reliably get tonight's forecast.
- The automation uses a 48-hour forecast window to find midnight. If your weather integration provides fewer hours, the `overnight_temps` list may be empty and the fallback temperature of `20°C` will be used.
- The heating/cooling kWh bands are fixed estimates based on one system's observed load, not a regression against actual consumption. If you have historical HVAC consumption data, it's worth eventually comparing actual overnight kWh against forecast temperature, cloud cover, and the resulting SOC trajectory to check whether the bands are well calibrated for your system — clear, cold nights draw very differently from overcast nights at a similar average temperature, and this automation currently only accounts for temperature.
- Tested on **Home Assistant OS 17.3 / Core 2026.5.2** in the NSW NEM region with a Bureau of Meteorology hourly weather entity and iZone ducted HVAC system.

---

## Contributing

Shared as a starting point. If you adapt it for different overnight loads or climate conditions, feel free to open a PR or issue.

---

## License

MIT — do whatever you like, no warranty implied.
