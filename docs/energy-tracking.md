# Energy tracking

Sensors that measure progress toward the power strategy in
`.plans/better-power-usage.md`. They observe only and control nothing.

## Off-grid

- `binary_sensor.grid_importing`: on when grid import (from the meter) is
  above 100 W for 30 seconds. Turns off after 1 minute below. The meter idles
  at around 25 W import, and import blips while the inverter responds to a
  load change are ignored.
- `sensor.last_grid_import`: when import last ended.
- `sensor.off_grid_streak`: hours since import last ended (0 while
  importing).
- `sensor.off_grid_streak_record`: longest streak so far.
- `sensor.self_sufficiency_today`: share of today's house consumption,
  including the car, not supplied by the grid.

Grid charging of the house battery counts as import:
`sensor.grid_import_power_meter` reads the grid meter directly.

## Car solar share

Only charging at home counts. Charging power is Tesla charger voltage x
current, because `sensor.tesla_model_3_charger_power` is reported in whole kW.

Grid import is attributed to the car first, so the solar share is
conservative. House battery discharge counts as solar.

- `sensor.ev_solar_share_session`: since the car was last plugged in at home.
  `automation.ev_charge_session_reset` resets the session counters.
- `sensor.ev_solar_share_month`: calendar month.
- Energy counters: `sensor.ev_solar_energy_session`,
  `sensor.ev_grid_energy_session`, `sensor.ev_solar_energy_monthly`,
  `sensor.ev_grid_energy_monthly`.

## Solar forecast accuracy

- `sensor.solar_forecast_today_morning`: today's Forecast.Solar forecast,
  recorded at 05:00.
- `sensor.solcast_forecast_today_morning`: today's Solcast forecast (p50),
  recorded at 05:00.
- `sensor.solar_actual_today_recorded`: today's generation, recorded at 23:55.
- `sensor.solar_forecast_ratio` and `sensor.solcast_forecast_ratio`: actual /
  forecast for each source. Long-term statistics give the calibration factor.

## Daily counters

`utility_meter.yaml` defines `sensor.grid_import_daily`,
`sensor.grid_export_daily`, `sensor.solar_generation_daily`,
`sensor.battery_charged_daily` and `sensor.battery_discharged_daily`.
