# ACiQ Extreme+ packages

The ACiQ work is split into a clean Home Assistant layer and optional reverse-engineering tools.

## Recommended Home Assistant setup

The generic `midea_xye` component already provides the temperature, compressor,
defrost, error/protection, and fan-speed sensors. Configure those directly in
the climate block so Home Assistant gets one clean copy of each value.

Recommended friendly names:

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

    # Optional diagnostic ordinal: 0=off, 1=low, 2=medium, 3=high.
    fan_speed:
      name: Fan Speed

    compressor_aware_action: true
    sync_fan_mode_from_device: true
```

`current:` is intentionally omitted on the tested ACiQ because C0 B15 reports
`0xFF`, which behaves as an unsupported/sentinel value rather than 255 A.

## ACiQ-specific add-on

`aciq-extreme-plus.yaml` is intended to stay installed alongside the climate
component. It only adds useful ACiQ-specific entities that are not already
provided by the generic component:

- **EEV Position**
- **Actual Fan Mode**
- **Target Fan Mode**
- **Bus Mode** (diagnostic)
- **Operating State** (diagnostic)

This avoids the duplicate `ACiQ ACiQ ...` / T1/T2/T3-style entities from the
earlier research profile.

Example package configuration:

```yaml
packages:
  aciq_extreme_plus:
    url: https://github.com/JFCrusan/ESPHome-Midea-XYE
    ref: aciq-diagnostics
    files:
      - examples/aciq-extreme-plus.yaml
    refresh: 5min
```

While this work is under review, load the external component from the same branch:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/JFCrusan/ESPHome-Midea-XYE
      ref: aciq-diagnostics
    components:
      - midea_xye
```

## Research / debug packages

These are optional and can be removed without removing the useful ACiQ layer.

### `aciq-diagnostics.yaml`

Publishes established raw C0/C4 fields and decoded diagnostic companions. Some
internal IDs and older raw labels are intentionally stable so Home Assistant
history collected during the reverse-engineering campaign remains usable.

### `aciq-byte-watch.yaml`

Temporary passive observer for payload bytes not individually exposed by
`aciq-diagnostics.yaml`. Home Assistant history becomes a long-running protocol
flight recorder.

### `aciq-snapshot.yaml`

One-click diagnostic snapshot helper. Requires `aciq-diagnostics.yaml`.

## Full lab configuration

During active reverse engineering:

```yaml
packages:
  aciq_packages:
    url: https://github.com/JFCrusan/ESPHome-Midea-XYE
    ref: aciq-diagnostics
    files:
      - examples/aciq-extreme-plus.yaml
      - examples/aciq-diagnostics.yaml
      - examples/aciq-byte-watch.yaml
      - examples/aciq-snapshot.yaml
    refresh: 5min
```

For a normal installation, remove the three research packages and keep only
`aciq-extreme-plus.yaml`.

See [`../docs/ACIQ_EXTREME_PLUS.md`](../docs/ACIQ_EXTREME_PLUS.md) for the
field map, validation metrics, real-log graphics, and the reasoning behind the
promoted fields.
