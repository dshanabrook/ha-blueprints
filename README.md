# RampTemp – Heating Setpoint Ramp Blueprint

## Mitsubishi Integration Requirements

This implementation is for Mitsubishi mini splits in a Home Assistant setting using a esp2366 device:  [ESP hardware described here](https://chrdavis.github.io/hacking-a-mitsubishi-heat-pump-Part-1/) which uses  HA implementation: [gysmo38/mitsubishi2MQTT](https://github.com/gysmo38/mitsubishi2MQTT).

- **Hardware** (ESP8266-based CN105 interface):
  - CN105 5-pin cable (e.g., AliExpress 5P option).
  - Wemos D1 Mini or Adafruit Huzzah ESP8266 (5V tolerant).
  - Soldering iron/solder for wire connections (5V-brown, GND-orange, RX-blue, TX-red).
  - Micro-USB for flashing.

Firmware uses SwiCago/HeatPump Arduino library; flash via Arduino IDE with ESP8266 support, ArduinoJson, PubSubClient.

- **Software**: `gysmo38/mitsubishi2MQTT` integration (exposes Mitsubishi CN105 protocol over MQTT).


---

## Overview

RampTemp is a Home Assistant **automation** blueprint; it ramps the setpoint upward from the current room temperature toward a user-defined target temperature.

This is useful as mini splits perform inefficiently with sudden thermostat increases, it tricks the unit by instead increasing  the temperature slowly in a stepwise fashion.

The blueprint never overwrites the stored target setpoint; it only adjusts the live thermostat setpoint while ramping.

It does this by storing the thermostat setting as target_temp.  Then it send a "fake" setting to the unit which is retained at x degrees above the room temperature until the target is reached.

## Features

- Heating-only ramp logic from current room temperature toward a stored target setpoint.  
- Configurable ramp amount (degrees per step) and internal increment (0.5 ° steps for rounding).  
- Uses a persistent `input_number` to store the user’s final target temperature.  
- Context classification of setpoint changes:
  - Manual changes at the thermostat or UI. **!!not working!!**
  - Automatic ramp steps from this blueprint.
  - “Adjusted” steps (post-ramp tweaks).
  **- NOTE:  Only HA temp changes could accurately be detected so Manual changes are ignored.**
- Diagnostic tracking via `input_text` helpers for:
  - Last trigger parent ID
  - Last trigger user ID
  - Whether a step was an adjusted step.
- `input_boolean` flag to indicate when a ramp is in progress.
- Safety logic to:
  - Start a new ramp when the user changes the setpoint.
  - Continue ramping while current temperature and setpoint are below target.
  - Stop ramping and clear the flag when the target is reached or overshot.

---

## Blueprint Inputs

The blueprint defines the following inputs:

- **Mini-Split Climate Device (`climateentity`)**  
  - Type: `climate` entity  
  - Example: `climate.living_room_mini_split`  
  - Used as the controlled thermostat for all setpoint changes and temperature readings.

- **Desired Target Temperature (`finaltargetinput`)**  
  - Type: `input_number` entity  
  - Example: `input_number.living_room_target_temp`  
  - Stores the persistent “user intent” final setpoint.  
  - On a manual setpoint change, the current setpoint is saved here before ramping begins.

- **Ramp Amount (`rampamount`)**  
  - Type: numeric input  
  - Default: `1.0`  
  - Range: `0.1`–`5.0`, step `0.1`  
  - Amount (in degrees) to increase per ramp step, measured from the current room temperature. 1 degree is recommended!

The blueprint also expects several helper entities (created manually by you) that are referenced directly in the YAML:

- `input_boolean.ramp_in_progress` – indicates whether a ramp is currently active.
- `input_text.minisplit_parent_id` – last trigger parent context ID (for debugging/context).
- `input_text.minisplit_user_id` – last user ID associated with a change.
- `input_text.minisplit_adjusted_step` – tracks whether the last step was an adjusted step.

If you use different entity IDs for these helpers, you will need to edit the blueprint YAML accordingly.

---

## How It Works

1. **Triggers**  
   - On any change of the climate entity’s `temperature` attribute (setpoint change).  
   - On any change of the climate entity’s `current_temperature` attribute (room temp update).

2. **Context and Classification**  
   - Extracts `trigger.tostate.context.parent_id` and `trigger.tostate.context.user_id`.  
   - Classifies each event as:
     - Manual (has `user_id`),
     - Ramp step (has `parent_id` but no `user_id`),
     - Remote/automatic (neither).

3. **Manual Setpoint Change (start ramp)**  
   - When a manual change is detected:
     - Sets `input_boolean.ramp_in_progress` to `on`.  
     - Saves the current setpoint into `input_number` defined as `finaltargetinput`.  
     - Computes a new next setpoint:
       - Base = current room temperature.  
       - Add `rampamount`.  
       - Clamp with a minimum safety temperature (57 °F) and a maximum of the final target.  
       - Round to the internal increment (0.5 °).  
     - Calls `climate.set_temperature` with the computed next setpoint.

4. **Ongoing Ramp Steps**  
   - While the current temperature and current setpoint are still below the final target:
     - Keeps `input_boolean.ramp_in_progress` on.  
     - Repeats the “next setpoint” calculation and applies it as another ramp step.

5. **Stopping the Ramp**  
   - If current temperature or setpoint reaches or exceeds the stored target:
     - Turns `input_boolean.ramp_in_progress` off.  
     - Stops additional ramp adjustments.

---

## Installation

1. **Create/clone the repository**

   - Clone `https://github.com/dshanabrook/ha_blueprints` locally.
   - Ensure there is an `automation` folder inside the repo (e.g. `automation/rampTemp.yaml`).[web:9]

2. **Copy the blueprint file**

   - Place `rampTemp.yaml` into an `automation` or `blueprint` subfolder in the repo, e.g.:  
     - `automation/rampTemp.yaml`  
   - Commit and push to GitHub.[web:9]

3. **Create helper entities in Home Assistant**

   In your Home Assistant config, create:

   - `input_number.<something>` for the final target temperature (and use that as `finaltargetinput`).  
   - `input_boolean.ramp_in_progress`.  
   - `input_text.minisplit_parent_id`.  
   - `input_text.minisplit_user_id`.  
   - `input_text.minisplit_adjusted_step`.

   Adjust entity IDs in the YAML if they differ.

4. **Import the blueprint into Home Assistant**

   - Go to **Settings → Automations & scenes → Blueprints → Import blueprint**.  
   - Paste the raw GitHub URL for `rampTemp.yaml`, for example:  
     `https://raw.githubusercontent.com/dshanabrook/ha_blueprints/main/automation/rampTemp.yaml`  
   - Preview and save.[web:9]

5. **Create an automation from the blueprint**

   - In **Blueprints**, select “RampTemp – Heating-only temperature ramp”.  
   - Choose your mini-split `climate` entity.  
   - Select the `input_number` entity that stores your target setpoint.  
   - Set the desired `Ramp Amount` (e.g., 1.0 °F per step).  
   - Save the automation.[web:3]

---

## Usage Notes

- The blueprint assumes a **heating-only** configuration; for cooling or dual-mode logic, the template would need to be extended.  
- The minimum “safe” base temperature in the ramp calculation is hard-coded as 57 °F; adjust in the YAML if your application differs.  
- All context-tracking helpers are optional from a pure control perspective but are very useful for debugging and ensuring that manual vs automatic changes are correctly classified.
