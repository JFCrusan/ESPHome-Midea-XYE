# ACiQ Extreme+ + ESPHome / Home Assistant

This is the **normal-user setup** for ACiQ Extreme+ equipment using the ESPHome `midea_xye` component.

If you have compatible ACiQ Extreme+ hardware and want useful Home Assistant telemetry without the reverse-engineering/debug entities, start here.

## Tested hardware

This profile was developed and validated on:

- **ACIQ-48-PAH** air handler
- **ACIQ-48-HPD** outdoor unit
- Midea-style **XYE / RS-485** bus
- **4800 baud, 8 data bits, no parity, 1 stop bit (8N1)**

Other ACiQ Extreme+ sizes using the same control platform may also work, but the models above are the ones directly tested.

## What you get

The base `midea_xye` component already provides the useful standard HVAC sensors, including:

- thermostat / room temperature
- indoor coil inlet temperature
- indoor coil outlet temperature
- outdoor coil temperature
- outdoor temperature
- compressor running
- defrost state
- error flags
- protection flags
- fan speed

The ACiQ Extreme+ add-on profile adds the fields that were useful on this equipment but are not already exposed cleanly by the generic component:

- **EEV Position**
- **Actual Fan Mode**
- **Target Fan Mode**
- **Bus Mode**
- **Operating State**

The goal is one useful Home Assistant entity for each value, not a wall of raw protocol bytes.

## 1. Load the component

Point ESPHome at this fork from your device YAML:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/JFCrusan/ESPHome-Midea-XYE
      ref: main
    components:
      - midea_xye
```

## 2. Configure the XYE UART

Example for the M5Stack Atom wiring used during development:

```yaml
uart:
  rx_pin: GPIO22
  tx_pin: GPIO19
  baud_rate: 4800
  data_bits: 8
  stop_bits: 1
  parity: NONE
```

Use the RX/TX pins appropriate for your own ESP32 board and RS-485 interface.

## 3. Configure the climate component

A clean example:

```yaml
climate:
  - platform: midea_xye
    id: heatpump_xye
    name: Heat Pump
    period: 1s
    timeout: 100ms
    use_fahrenheit: true

    internal_current_temperature:
      name: Thermostat Temp

    temperature_2a:
      name: Inside Coil Inlet Temp

    temperature_2b:
      name: Inside Coil Outlet Temp

    temperature_3:
      name: Outside Coil Temp

    outdoor_temperature:
      name: Outside Temp

    compressor_active:
      name: Compressor Running

    defrost:
      name: Defrost Active

    error_flags:
      name: Error Flags

    protect_flags:
      name: Protection Flags

    fan_speed:
      name: Fan Speed

    compressor_aware_action: true
    sync_fan_mode_from_device: true
```

### A note about current

Do **not** enable the component's `current:` sensor on the tested ACiQ system. Its C0 current byte reports `0xFF`, which behaves as an unsupported/sentinel value rather than a real 255 A reading.

## 4. Add the ACiQ Extreme+ profile

Add this package to the same ESPHome YAML:

```yaml
packages:
  aciq_extreme_plus:
    url: https://github.com/JFCrusan/ESPHome-Midea-XYE
    ref: main
    files:
      - examples/aciq-extreme-plus.yaml
    refresh: 1d
```

The package expects the climate component to use:

```yaml
id: heatpump_xye
```

After compiling and flashing, Home Assistant should gain the ACiQ-specific entities listed above in addition to the normal `midea_xye` sensors.

## What you do NOT need

For a normal installation you do **not** need:

- `aciq-diagnostics.yaml`
- `aciq-byte-watch.yaml`
- `aciq-snapshot.yaml`

Those files are research tools used while reverse-engineering the XYE protocol. They intentionally expose much more raw data and are not part of the recommended everyday configuration.

## Validation and technical details

The promoted mappings were checked against long-running Home Assistant history, independent temperature sensors, compressor power measurements, operating-mode changes, and repeated refrigeration cycles.

For the field map, confidence levels, validation plots, EEV evidence, and the recurring refrigeration Housekeeping Routine, see:

- [ACiQ Extreme+ field map and validation](../docs/ACIQ_EXTREME_PLUS.md)
- [Clean ACiQ Extreme+ package](aciq-extreme-plus.yaml)

The focused contribution is also under review upstream in HomeOps/ESPHome-Midea-XYE:

- [Upstream PR #164: Add ACiQ Extreme+ XYE profile and validated field map](https://github.com/HomeOps/ESPHome-Midea-XYE/pull/164)

## If your system behaves differently

ACiQ equipment can share control electronics across multiple capacities and model variants, but do not assume every byte means the same thing on every unit.

If you find a difference, the most useful report includes:

- indoor and outdoor model numbers
- commanded HVAC mode
- what Home Assistant reports
- the specific value that looks wrong

The research tooling remains available in this repository if deeper capture work is needed.
