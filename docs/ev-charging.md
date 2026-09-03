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
The controller still sets the Tesla current to 32 A and preserves 50% SolarEdge
battery reserve while the Tesla is charging in an Octopus dispatch or an
off-peak window.

When Smart Charge is off, a plugged-in Tesla without a pending Tesla charging
schedule is started by the controller and follows the peak-hour rules below.

### Charge override

Charge override requests an immediate 32 A charge and bypasses Tesla schedules
and solar matching:

- with Smart Charge on, it turns on Octopus Boost;
- with Smart Charge off, it starts the Tesla directly;
- it turns itself off, and turns off Octopus Boost, when the Tesla reaches its
  configured charge limit.

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
- SolarEdge backup reserve to 50%.

### Peak hours

Outside off-peak and an Intelligent dispatch, the controller applies these
daytime rules:

- Tesla charge current never exceeds 16 A.
- Above 50% SolarEdge battery state of charge, Tesla charge current is 16 A.
- At or below 50% battery state of charge, Tesla current is calculated from
  available solar power and constrained to 5-16 A.
- Below Tesla's 5 A minimum available capacity, the controller stops charging
  to avoid unintended import or home-battery discharge.

Available solar power is the ten-minute average of grid export plus the Tesla's
current charging power, less a 250 W safety margin. Adding current Tesla draw
back to export prevents the calculation falling to zero after Tesla begins
using surplus generation.

## Home battery rules

During off-peak or Intelligent dispatch charging, the SolarEdge backup reserve
is set to 50%.

When the Tesla is not charging, the controller restores the backup reserve to
0%, allowing normal battery discharge. It does not restore the reserve when
SolarEdge storage control mode is `Time of Use`.

During peak-hour charging, the controller also leaves SolarEdge storage policy
unchanged when `Time of Use` is selected. Off-peak charging always sets a 50%
reserve, including when Time of Use is selected.

When the Tesla leaves home, the controller sends no Tesla or Octopus commands.
It restores SolarEdge backup reserve to 0% as local cleanup, unless SolarEdge
storage control mode is `Time of Use`.

## Control precedence

Each controller run applies the first matching state:

1. Tesla away from home: make no Tesla or Octopus changes; restore SolarEdge
   reserve to 0% unless it is in Time of Use mode.
2. Charge override: immediate 32 A charging; use Octopus Boost when Smart
   Charge is enabled.
3. Tesla scheduled charging: leave start and stop timing to Tesla.
4. Smart Charge while Tesla is not charging: leave start timing to Octopus.
5. Tesla charging in off-peak or Intelligent dispatch: 32 A and 50% reserve.
6. Tesla charging in peak hours: apply the 16 A and solar-matching rules.
7. Tesla plugged in, not scheduled, and Smart Charge off: start Tesla according
   to the applicable off-peak or peak-hour rule.
8. Tesla unplugged or not charging: restore the reserve to 0%, unless SolarEdge
   is in Time of Use mode.

## Components

The controller is implemented by:

- Automation: `automation.ev_charging_controller`
- Reconciliation script: `script.ev_charge_reconcile`
- 10-minute export average: `sensor.ev_grid_export_10_minute_average`
- Available solar calculation: `sensor.ev_available_solar_power`
- Off-peak window: `binary_sensor.ev_off_peak_charging_window`

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

## Operational checks

Use these entities to check controller behavior:

- `sensor.ev_grid_export_10_minute_average` should populate after enough grid
  export samples have been collected following a restart.
- `sensor.ev_available_solar_power` shows the power used for low-battery solar
  matching.
- `binary_sensor.ev_off_peak_charging_window` identifies 32 A / 50% reserve
  conditions.
- `number.tesla_model_3_charge_current` shows the latest requested Tesla
  current.
- `number.solaredge_multi_i1_backup_reserve` is 50% during off-peak Tesla
  charging and normally 0% when the Tesla is not charging.

If Tesla entities are unavailable, the reconciliation script exits without
sending Tesla commands. This is intentional fail-safe behavior.
