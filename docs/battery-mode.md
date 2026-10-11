# House battery mode

Home Assistant steers the SolarEdge battery with Remote Control commands. The
inverter's storage control mode is `Remote Control` with default mode
`Maximize Self Consumption`. Every command times out after an hour and the
inverter falls back to the default, so if Home Assistant stops, the battery
carries on as normal.

`sensor.battery_mode` (`templates/battery_mode.yaml`) says which mode the
battery should be in. `automation.battery_mode_controller` sends it to the
inverter when the command read back from the inverter differs. That covers mode
changes and hourly timeouts. Registers are written only when they differ, and
each change is logged to the logbook as "Battery mode".

## Modes

In priority order:

Nothing here is seasonal: each mode follows the forecast and the tariff, so
grid top-ups happen mostly on winter nights and filling late mostly on long
days, without any date rules.

1. **Charge from Solar Power and Grid** (grid top-up). Between 23:30 and
   05:30, during off-peak, when the state of charge is more than 2 points below
   the 23:00 plan's target, that is, when tomorrow's forecast won't cover the
   morning and evening. It continues until the target is reached. The rate
   (`grid_charge_rate` attribute) is set to finish about 04:30, between 500 and
   2500 W. That is gentle on the battery and leaves supply headroom while the
   car charges.
2. **Charge from Solar Power** (hold). When `binary_sensor.house_battery_hold`
   is on: the car is charging from the grid, or the overnight plan says hold.
   The battery does not discharge; solar can still charge it. See
   [ev-charging.md](ev-charging.md).
3. **Charge from Clipped Solar Power** (fill late). When Solcast's
   pessimistic (p10) forecast for the rest of today covers filling the battery
   (allowing 5% loss), the house until sunset (0.6 kW) and a margin, the
   battery stops charging for now. That margin is
   `sensor.solar_spare_today_p10`. Solcast is set to the array's 8.8 kW, but
   the inverter's AC output is limited to 6 kW, so forecast generation above
   6 kW counts only towards filling the battery. Solar beyond the house goes
   to the car, the immersion or export. The battery still takes solar above
   6 kW, which would otherwise be lost. As the day goes on, the remaining
   forecast shrinks, the margin runs out and the battery fills, roughly as
   late as the forecast safely allows, instead of sitting full from
   mid-morning. It
   applies only while:
   - the margin is at least 3 kWh to start (1.5 kWh to continue, the same as
     `binary_sensor.ev_car_first`);
   - the state of charge is at least 50% to start (47% to continue), so there
     is stored energy to fall back on;
   - the house is not importing, and hasn't for 10 minutes. These modes don't
     discharge, so passing cloud would otherwise mean grid import.
4. **Maximize Self Consumption**: normal.

## Overnight plan

`sensor.battery_overnight_plan` (`templates/battery_plan.yaml`) is made at 23:00,
or on a `battery_plan_refresh` event. Its state is the state of charge needed at
05:30 to cover the peak-rate morning (2.3 kWh) and evening (7.0 kWh), after
tomorrow's forecast solar surplus (midway between Solcast p10 and p50, less
5.5 kWh daytime house load). It is at least the morning load.

`hold` is true when stored energy minus 2.0 kWh overnight load is below the
morning load. During the off-peak tariff window, the battery is then held so
the house runs on cheap grid power. When the battery comfortably lasts the
night, it is not held, keeping off-grid nights. Plans older than 12 hours are
ignored. House loads are 75th percentiles from a year of history, excluding EV
charging. `automation.battery_overnight_plan_log` logs each plan's reason.

## Other effects

- The immersion surplus rule treats a battery filling late as full, so the
  heater uses solar before it is exported.
- Car solar headroom ignores battery charging while filling late, because
  clipped solar can't reach the car.

## Components

- Desired mode: `sensor.battery_mode`
- Hold: `binary_sensor.house_battery_hold`
- Solcast p10 remaining today: `sensor.solar_remaining_today_p10` (attribute
  `above_6kw`)
- Spare solar: `sensor.solar_spare_today_p10`
- Controller: `automation.battery_mode_controller`
- Overnight plan: `sensor.battery_overnight_plan`
- Mode check: `automation.solaredge_storage_mode_check`
- Inverter: `select.solaredge_multi_i1_storage_control_mode`,
  `select.solaredge_multi_i1_storage_command_mode`,
  `number.solaredge_multi_i1_storage_command_timeout`,
  `number.solaredge_multi_i1_storage_charge_limit`

The Remote Control entities are unavailable unless the storage control mode is
Remote Control.

## Flash memory

SolarEdge's Power Control Protocol treats the command mode, timeout and
charge/discharge limits as dynamic commands, designed to be renewed often. The
storage control mode, default mode and backup reserve are initial
configuration, so they are set once.
