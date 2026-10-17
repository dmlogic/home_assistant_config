# House battery mode

Home Assistant steers the SolarEdge battery with Remote Control commands. The
inverter's storage control mode is `Remote Control` with default mode
`Maximize Self Consumption`. Commands time out and the inverter falls back to
the default. If Home Assistant stops, the battery carries on as normal, except
during a hold: the 0 W discharge limit also applies to the default mode, so the
battery stays held (extra grid import, never a drained battery) until Home
Assistant returns.

`sensor.battery_mode` (`templates/battery_mode.yaml`) says which mode the
battery should be in: Grid top-up, Hold, Car lock, Fill late or Normal.
`automation.battery_mode_controller` turns that into an inverter command and
charge/discharge limits:

| Mode | Command | Limits | Timeout |
|---|---|---|---|
| Grid top-up | Charge from Solar Power and Grid | charge = `grid_charge_rate` | 1 h |
| Hold, Car lock | Maximize Self Consumption | discharge 0 W | 7 h |
| Fill late | Charge from Clipped Solar Power | | 1 h |
| Normal | Maximize Self Consumption | | |

Grid top-up and Fill late are sent again when the command read back from the
inverter differs, which covers hourly timeouts. Hold and Car lock use the same
command as Normal, so the read-back can't show them expiring. They are sent on
entry (and whenever the timeout read back isn't 7 hours) and renewed hourly.
They end when Normal restores the discharge limit. Registers are written only
when they differ, and each mode change is logged to the logbook as "Battery
mode".

Don't use "Charge from Solar Power" for a hold. It sends solar to the battery
before the house: on 2026-10-17 the battery took 3.7 kW while the house and car
imported 4 kW.

## Modes

Nothing here is seasonal: each mode follows the forecast and the tariff, so
grid top-ups happen mostly on winter nights and filling late mostly on long
days, without any date rules. In priority order:

1. **Grid top-up**. Between 23:30 and
   05:30, during off-peak, when the state of charge is more than 2 points below
   the 23:00 plan's target, that is, when tomorrow's forecast won't cover the
   morning and evening. It continues until the target is reached. The rate
   (`grid_charge_rate` attribute) is set to finish about 04:30, between 500 and
   2500 W. That is gentle on the battery and leaves supply headroom while the
   car charges.
2. **Hold**. When `binary_sensor.house_battery_hold` is on: the car is
   charging from the grid, or the overnight plan says hold. The battery does
   not discharge; solar serves the house first and any surplus still charges
   the battery. See [ev-charging.md](ev-charging.md).
3. **Car lock** (applied as Hold). While the car may be solar-charging
   (`binary_sensor.ev_may_be_charging`) outside off-peak, unless the battery
   has energy to spare: at least 90% (92% to unlock once locked), or
   `binary_sensor.ev_car_first` is on. Otherwise the battery would cover the
   car's shortfall while the Tesla's 10-minute-old data catches up. When it's
   locked, a shortfall shows as grid import, which the car controller sees
   within seconds.
4. **Fill late**. When Solcast's
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
   - the sun is up;
   - the state of charge is at least 50% to start (47% to continue), so there
     is stored energy to fall back on;
   - the house is not importing, and hasn't for 10 minutes. This mode doesn't
     discharge, so passing cloud would otherwise mean grid import.
5. **Normal**.

## Overnight plan

`sensor.battery_overnight_plan` (`templates/battery_plan.yaml`) is made at 23:00,
or on a `battery_plan_refresh` event. Its state is the state of charge needed at
05:30 to cover the peak-rate morning (2.3 kWh) and evening (7.0 kWh), after
tomorrow's forecast solar surplus (midway between Solcast p10 and p50, less
5.5 kWh daytime house load). It is at least the morning load.

`hold` is true when stored energy minus 2.0 kWh overnight load is below the
target. That includes nights with a grid top-up, so the top-up isn't run down
again before 05:30. During the off-peak tariff window, the battery is then held so
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
- Car may be drawing power: `binary_sensor.ev_may_be_charging`
- Solcast p10 remaining today: `sensor.solar_remaining_today_p10` (attribute
  `above_6kw`)
- Spare solar: `sensor.solar_spare_today_p10`
- Controller: `automation.battery_mode_controller`
- Overnight plan: `sensor.battery_overnight_plan`
- Mode check: `automation.solaredge_storage_mode_check`
- Inverter: `select.solaredge_multi_i1_storage_control_mode`,
  `select.solaredge_multi_i1_storage_command_mode`,
  `number.solaredge_multi_i1_storage_command_timeout`,
  `number.solaredge_multi_i1_storage_charge_limit`,
  `number.solaredge_multi_i1_storage_discharge_limit`

The Remote Control entities are unavailable unless the storage control mode is
Remote Control.

## Flash memory

SolarEdge's Power Control Protocol treats the command mode, timeout and
charge/discharge limits as dynamic commands, designed to be renewed often. The
storage control mode, default mode and backup reserve are initial
configuration, so they are set once.
