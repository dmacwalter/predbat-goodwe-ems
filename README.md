# Predbat control of a GoodWe ET over EMS mode: Home Assistant scripts

These are the scripts and automations running on a GoodWe GW9.99K-EH (ET series, 10 kW AC, 5.4 kW grid export limit, about 22.4 kWh battery). The inverter is on the mletenay GoodWe integration, local only. Predbat drives it through the Service API. ARM/BMU firmware is 27 Oct 2026 (inverter 04094-02-S10).

They have been in daily use since September 2026. Treat them as a working reference, not a drop-in template: several values and helper entities are specific to this site (listed below).

## Files

| File | What it is |
|---|---|
| `scripts/predbat_goodwe.yaml` | `predbat_charge`, `predbat_freeze`, `predbat_freeze_export`, `predbat_export`, `predbat_stop`. Paste into `scripts.yaml` or a package |
| `custom_templates/predbat_control.jinja` | Shared decision macros, imported by the scripts and the follow-through. Goes in `config/custom_templates/` |
| `automations/predbat_goodwe.yaml` | Follow-through between cycles (needed), sell_power export-limit watchdog (strongly recommended), battery taper guard (optional) |

## apps.yaml service mapping

```yaml
  charge_start_service: script.predbat_charge
  charge_freeze_service: script.predbat_freeze
  charge_stop_service: script.predbat_stop
  discharge_start_service: script.predbat_export
  discharge_freeze_service: script.predbat_freeze_export
  # predbat_freeze also checks predbat.status, so a Freeze exporting call that lands on it is handled too
  discharge_stop_service: script.predbat_stop
```

Use whichever service format your Predbat version expects; the [docs](https://springfall2008.github.io/batpred/apps-yaml/#service-api) show the `service:` / `entity_id:` form. Each service entry also has `repeat: true` (start/freeze) or `repeat: false` (stop) on my system. Predbat's charge and discharge rates are written to `input_number.predbat_charge_rate` and `input_number.predbat_discharge_rate`, and the scripts read them from there. The reserve is written to `input_number.predbat_reserve`. Turn on `switch.predbat_set_export_freeze` if you want Freeze exporting.

## Entities the scripts use

GoodWe (mletenay integration):
- `select.goodwe_ems_mode`
- `number.goodwe_ems_power_limit` (Xset)
- `number.goodwe_grid_export_limit`
- `select.goodwe_inverter_operation_mode` (must be `general`)
- `sensor.battery_state_of_charge`, `sensor.pv_power`, `sensor.house_consumption`
- `sensor.grid_power` (+ = import), `sensor.battery_power` (+ = discharge)
- `sensor.battery_charge_limit` (BMS current limit, A) and `sensor.total_power` (inverter AC output)

Some of these entity names come from my own template sensors and renames, so map them to yours.

Predbat: `predbat.status` (state plus its `detail` and `debug` attributes), `predbat.load_energy`, `select.predbat_mode`, `switch.predbat_set_read_only` and `switch.predbat_active`.

Forecast: Solcast (`sensor.solcast_pv_forecast_power_now` and `sensor.solcast_pv_forecast_forecast_today`).

Helpers to create:
- `input_boolean.predbat_guard_held_charge_at_target`
- `input_boolean.predbat_guard_stopped_export_at_target`
- `input_boolean.ems_test_in_progress` (I set it during manual EMS tests so nothing interferes)
- `input_boolean.taper_guard_holding` (taper guard only)

My own helper sensors, which you'd need to create or swap for something equivalent:
- `sensor.pv_surplus_kw_smoothed` (smoothed PV minus house, kW)
- `sensor.home_goodwe_pv_power_mean_15_min` (statistics, 15-min mean)
- `sensor.house_consumption_30min_average`

## Site-specific things to change

- Battery capacity `22.4` kWh and `rate_max` 9600 W in `custom_templates/predbat_control.jinja`.
- `960` (16:00) in the jinja. I run a "battery full by 16:00" rule ahead of a 16:00-21:00 peak tariff, and `late_fill_ok()`, `solar_margin_kwh()` and `freeze_export_mode()` all use it. If you don't want that rule, make `charge_pace_w()` skip `late_fill_ok()`, and have `freeze_export_mode()` ignore the margin.
- The export constants in `script.predbat_export` (AC ceiling 10 kW, 9.6 kW PV ceiling, 5.4 kW export limit read from `number.goodwe_grid_export_limit`, ~5 kW battery discharge, 15 A taper limit) are for this inverter and network.
- The evcc / car entities (`switch.evcc_goodwe_charger_ha_battery_boost`, `sensor.evcc_goodwe_charger_ha_charge_power`, `input_boolean.car_may_share_battery_solar`, `input_select.evcc_battery_mode`, `input_boolean.evcc_battery_control`) are optional. If they don't exist they read off / 0.
- `script.predbat_stop` starts with a block that yields to a separate PV-curtailment automation I run. Delete that block if you don't have one.
- Rule I of the follow-through checks `input_boolean.demand_export_first_active`, a separate export-first automation of mine. Remove that condition if you don't use one.

## Behaviour notes the logic depends on (tested day and night, retested after the firmware)

- Write order is auto -> Xset -> mode, and nothing is written if the inverter is already in the target mode at about the same Xset. Scripts are `mode: queued`, because `charge_stop` and `discharge_stop` arrive together.
- buy_power holds grid import at Xset and leaves PV free. That's used for daytime grid top-up instead of charge_battery.
- charge_battery and battery_standby are the same mode: battery power held at Xset, and Xset 0 is a hard hold.
- discharge_pv never charges the battery and respects the export limit. Xset 0 is a hold with the surplus exported.
- sell_power holds grid export at Xset but ignores the export limit (8.5 kW seen with a full battery), hence the Xset cap and the watchdog.
- export_ac curtails PV. conserve, charge_pv and import_ac run the house from the grid in daylight. None of them are used.
- Asking for more AC than about 10 kW minus the house load pins the output at ~9.95 kW, curtails PV and drains the battery, so export setpoints are capped at 10 kW - house - 300 W.
- Grid-only charge ceiling is about 7.05 kW. PV (DC) charging reaches about 10 kW.
- The follow-through is needed because Predbat sends one command per state change, expects the inverter to stop at the target by itself and won't resend an identical command, and GoodWe EMS has no target SoC.

## Changes

Dated changes are recorded in the git history. Discussion is in [springfall2008/batpred#1000](https://github.com/springfall2008/batpred/issues/1000).
