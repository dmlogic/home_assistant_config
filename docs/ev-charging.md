# EV charging controller

## Overview

The EV charging controller coordinates the Tesla Model 3, Octopus Intelligent,
and SolarEdge home battery. It uses off-peak electricity when appropriate and
otherwise limits daytime charging according to home battery state and available
solar power.

There is no global automation enable switch. The current Tesla, Octopus,
schedule, and Charge override states determine controller behavior.

All Tesla, Octopus, and SolarEdge controller actions require
`device_tracker.tesla_model_3_location` to be `home`. Charging elsewhere never
changes Tesla charging, Octopus Boost, or SolarEdge battery settings.

## Dashboard controls

The normal controls are:

- Octopus Smart Charge: `switch.octopus_energy_00000000_0002_4000_8020_0000000bb802_intelligent_smart_charge`
- Charge override: `input_boolean.ev_charge_override`

Octopus Boost is controlled automatically by Charge override. It is not needed
on the dashboard except as a fallback or diagnostic control.

### Octopus Smart Charge

When Smart Charge is on, Octopus controls normal charging start and stop times.
Whenever the Tesla is charging with Smart Charge on, the controller treats it
as grid charging: it sets the Tesla current to 32 A and holds the SolarEdge
battery's starting state of charge. This applies even before the Octopus
dispatch sensor turns on.

When Smart Charge is off, a plugged-in Tesla without a pending Tesla charging
schedule is started by the controller and follows the peak-hour rules below.

### Charge override

Charge override requests an immediate 32 A charge and bypasses Tesla schedules
and solar matching:

- with Smart Charge on, it turns on Octopus Boost;
- with Smart Charge off, it starts the Tesla directly;
- it turns itself off, and turns off Octopus Boost, when the Tesla reaches its
  configured charge limit.
- the house battery is held at its starting state of charge, as for all grid
  charging. During peak hours the house imports at peak rate while the
  override runs.

The controller never changes the Tesla charge limit.

When Charge override is on while the Tesla is away, it has no effect. It remains
on and takes effect only when the Tesla returns home.

## Charging rules

When the Tesla starts charging, the controller evaluates the rules immediately.
It also evaluates changes to Tesla, Octopus, SolarEdge battery, schedule, and
override state. Solar matching runs no more than once every ten minutes.

### Tesla schedule

When `binary_sensor.scheduled_charging_pending` is on, Tesla controls the
charging start and stop time. Charge override is the only exception.

### Off-peak and Intelligent dispatch

Off-peak charging applies when either of these is on:

- `binary_sensor.octopus_energy_electricity_23e5077367_2000016805578_off_peak`
- `binary_sensor.octopus_energy_00000000_0002_4000_8020_0000000bb802_intelligent_dispatching`

While the Tesla is charging in either condition, the controller sets:

- Tesla charge current to 32 A.
- A house battery hold (see Home battery rules).

### Car first or battery first

`binary_sensor.ev_car_first` is on when `sensor.solar_spare_today_p10` is at
least 1.5 kWh: Solcast's pessimistic (p10) forecast for the rest of today,
`sensor.solar_remaining_today_p10`, covers:

- the energy to fill the house battery (allowing 5% charging loss);
- house load until sunset, assumed to be 0.6 kW;
- a 1.5 kWh margin.

Forecast generation above the inverter's 6 kW AC limit counts only towards
filling the battery.

It turns off immediately and turns on after 15 minutes. When it is on, solar
headroom includes the power charging the house battery, so the car can take
solar the battery would otherwise store. When it is off, or Solcast is
unavailable, the car only gets solar that would otherwise be exported.

### Peak hours

Outside off-peak and an Intelligent dispatch, the controller applies these
daytime rules:

- Tesla charge current never exceeds 16 A. Above this, house plus car load
  exceeds the inverter's 6 kW AC output and the excess comes from the grid.
- Tesla current is calculated from available solar power and constrained to
  5-16 A, whatever the house battery's state of charge.
- Below Tesla's 5 A minimum available capacity, the controller stops charging
  to avoid unintended import or home-battery discharge.

Solar headroom is calculated for every sample as net grid export, plus current
Tesla charging power, plus house battery charging power when the car comes
first, minus home-battery discharge and a 250 W safety margin. Tesla charging
power is charger voltage x current, because the Tesla power sensor reports
whole kW.
Grid import and battery discharge are subtracted, so neither can be treated as
solar capacity. The controller uses the ten-minute average of that headroom and
the lower of that average and the current headroom. Adding Tesla draw back to
net export prevents the calculation falling to zero after Tesla begins using
surplus generation.

After a Home Assistant restart, before the average is available, solar
matching uses the current solar-headroom value as a safety fallback. This
allows the controller to reduce or stop charging immediately rather than wait
ten minutes.

## Home battery rules

The house battery never supports grid charging of the car.
`binary_sensor.house_battery_hold` is on when the Tesla is at home and either:

- it is charging with Charge override, off-peak or an Intelligent dispatch, or
  Smart Charge on; or
- it is plugged in and below its charge limit during off-peak or a dispatch,
  so the hold is in place before the car reports that it is charging.

It is also on overnight when the 23:00 battery plan says the battery would
otherwise run down before the morning. See [battery-mode.md](battery-mode.md)
for the plan and the inverter modes.

In SolarEdge Remote Control (normal), `automation.battery_mode_controller`
applies the hold as the inverter command "Charge from Solar Power": the battery
does not discharge, and solar can still charge it. This controller then keeps
the backup reserve at 0%.

If the storage control mode is anything else (for example after a change in
the SolarEdge app), the controller falls back to the backup reserve: it sets
the reserve to the battery's whole-percentage state of charge when the hold
starts, and back to 0% afterwards. The inverter lets the state of charge dip
slightly under the reserve, so the reserve is lowered only when it is more
than 2 points above the state of charge (for example after a BMS
recalibration). Lowering it on every dip would ratchet the battery down. The
backup reserve is stored in inverter flash memory, so it is written only when
it must change. `automation.solaredge_storage_mode_check` raises a
notification if the mode is not Remote Control for 5 minutes.

## Control precedence

The backup reserve is set first, as described above. Each controller run then
applies the first matching state:

1. Tesla away from home: make no Tesla or Octopus changes.
2. Charge override and Tesla target reached: clear the override and Octopus
   Boost.
3. Charge override: immediate 32 A charging; use Octopus Boost when Smart
   Charge is enabled.
4. Tesla scheduled charging, target reached or unplugged: leave the Tesla
   alone.
5. Tesla grid charging: 32 A.
6. Smart Charge while Tesla is not charging: leave start timing to Octopus.
7. Tesla charging in peak hours: solar matching at 5-16 A.
8. Tesla plugged in, not charging, in off-peak: start at 32 A.
9. Tesla plugged in, not charging, in peak hours: start when solar headroom
   allows 5 A.

## Components

The controller is implemented by:

- Automation: `automation.ev_charging_controller`
- Reconciliation script: `script.ev_charge_reconcile`
- Instantaneous solar headroom: `sensor.ev_solar_headroom`
- 10-minute solar headroom average: `sensor.ev_solar_headroom_10_minute_average`
- Available solar calculation: `sensor.ev_available_solar_power`
- Off-peak window: `binary_sensor.ev_off_peak_charging_window`
- Car first or battery first: `binary_sensor.ev_car_first`
- Solcast p10 remaining today: `sensor.solar_remaining_today_p10`
- Storage mode check: `automation.solaredge_storage_mode_check`
- House battery hold: `binary_sensor.house_battery_hold`
  (`templates/battery_mode.yaml`), applied by
  `automation.battery_mode_controller`

## Key entities

- Tesla charging state: `sensor.tesla_model_3_charging`
- Tesla control: `switch.tesla_model_3_charge`
- Tesla current: `number.tesla_model_3_charge_current`
- Tesla charge limit: `number.tesla_model_3_charge_limit`
- Tesla charging power: `sensor.tesla_model_3_charger_power`
- Tesla location: `device_tracker.tesla_model_3_location`
- Tesla schedule status: `binary_sensor.scheduled_charging_pending`
- SolarEdge battery state of charge: `sensor.solaredge_battery1_state_of_charge`
- SolarEdge grid export: `sensor.power_grid_export`
- SolarEdge backup reserve: `number.solaredge_multi_i1_backup_reserve`
- SolarEdge storage mode: `select.solaredge_multi_i1_storage_control_mode`
- SolarEdge battery capacity: `sensor.solaredge_multi_i1_b1_available_energy`
- Solcast forecast: `sensor.solcast_pv_forecast_forecast_today`

## Operational checks

Use these entities to check controller behavior:

- `sensor.ev_solar_headroom_10_minute_average` should populate after enough
  solar-headroom samples have been collected following a restart.
- `sensor.ev_available_solar_power` shows the power used for solar matching.
- `binary_sensor.ev_car_first` shows whether the car may take solar that would
  otherwise charge the house battery.
- `binary_sensor.ev_off_peak_charging_window` identifies 32 A conditions.
- `binary_sensor.house_battery_hold` shows whether the house battery is held.
- `number.tesla_model_3_charge_current` shows the latest requested Tesla
  current.
- `number.solaredge_multi_i1_backup_reserve` is 0% in Remote Control. In the
  fallback it holds the battery's starting state of charge during a hold.

If Tesla entities are unavailable, the reconciliation script exits without
sending Tesla commands. This is intentional fail-safe behavior.
