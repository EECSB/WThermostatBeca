# Temperature Offset / Calibration — Changelog

Adds a persistent, configurable temperature calibration offset with **active
control compensation**, so the thermostat not only *displays* a corrected
temperature but actually *heats to* it.

## Summary

- **Range:** −5.0 … +5.0 °C, step 0.1 °C
- **Persistent:** stored in EEPROM, survives reboot, applied immediately (no reboot)
- **Interfaces:** local web UI, MQTT, Home Assistant MQTT Discovery (Number entity)
- **Active calibration:** the setpoint sent to the Tuya MCU is compensated
  (`sent = userTarget − offset`) for manual, schedule and eco values, while the
  UI keeps showing the user's chosen numbers.

## How it works

The ESP is only a bridge; the Tuya MCU owns the sensor and the heating control.
Two symmetric transforms are applied at the serial boundary:

- **From MCU → user (add offset):** actual temperature, echoed manual target,
  schedule temperatures, eco target — everything the UI/MQTT/HASS reports is
  `corrected = raw + offset`.
- **User → MCU (subtract offset):** manual setpoint and schedule setpoints are
  sent as `raw = userValue − offset`, so the MCU (which controls against its own
  raw sensor) drives the room to the corrected value.

When the offset itself changes, the cached schedule bytes are re-derived and the
manual setpoint is re-sent so behaviour stays correct without re-entering values.

## Files changed

- `WThermostat/WBecaDevice.h` — all feature logic (see `temperature-offset.patch`)
- `platformio_override.ini` — added (local build config; required by `extra_configs`)
- `lib/WAdapter/` — populated from the pinned submodule (was empty in the zip)

## Detailed changes in `WBecaDevice.h`

1. **Setting/property** `temperatureOffset` (`DOUBLE`) added. It is persisted in
   the 8 bytes previously reserved as `PROP_BECARES1..8` (a `double` is exactly
   8 bytes), so the **EEPROM layout is unchanged** and old configs still load.
   A NaN/out-of-range guard (`clampOffset`) handles fresh flash and upgrades.
2. **Corrected temperature reporting** — offset added when parsing the actual
   temperature (MCU cmd `0x03`); the raw value is cached for live offset changes.
3. **Active compensation:**
   - Manual target sent to MCU (`targetTemperatureManualModeToMcu`) uses
     `compensatedTargetForMcu()` = `userTarget − offset`, clamped to 5…45 °C.
   - Manual target echoed by MCU (cmd `0x02`) is converted back with `+ offset`.
   - Schedule temps: `+ offset` on display/JSON/target, `− offset` on input.
   - Eco target display uses `+ offset`.
4. **`onChangeTemperatureOffset()`** — clamps, re-derives cached schedule bytes,
   resends schedules + manual setpoint to the MCU, refreshes the corrected
   actual-temperature display, and persists (auto-saved as a setting).
5. **Web UI** — new "Temperature Calibration" page with a −5…+5 slider that
   applies instantly (no reboot), following the existing Schedules-page pattern.
6. **Home Assistant** — new `MQTT_HASS_AUTODISCOVERY_NUMBER` template published to
   `homeassistant/number/<id>_tempoffset/config` (min −5, max 5, step 0.1,
   mode slider).
7. **MQTT** — because `temperatureOffset` is a normal device property, it is
   published in `.../stat/things/thermostat/properties` and is settable via
   `.../cmnd/things/thermostat/properties/temperatureOffset` automatically.

## Known limitations

- **Large offsets vs. MCU limits:** the compensated setpoint is clamped to
  5…45 °C. A very large offset combined with an extreme target can hit that clamp,
  so active control cannot follow beyond the MCU's physical band (the *display*
  offset still applies). Range is ±5 °C.
- **Eco mode:** the MCU runs eco against its own internal eco setpoint, which this
  firmware does not expose, so eco is display-corrected but its control point is
  the MCU's. Manual and schedule modes are fully compensated.
- **Floor sensor** (`0x66`) is left uncorrected — it is a separate sensor from the
  calibrated room sensor.

## Build

Built locally with PlatformIO:

    pio run -e wthermostat

- Output: `build_output/firmware/wthermostat-1.23-tempoffset-dev.bin`
- Size: Flash 43.9% (449 587 B), RAM 41.6% — well within the 1 MB / 80 KB budget.
