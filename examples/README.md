# ACiQ Extreme+ packages

The ACiQ work is intentionally split into a clean Home Assistant profile and optional reverse-engineering tools.

## Normal / useful profile

`aciq-extreme-plus.yaml`

This is the package intended to stay installed. It exposes the currently validated ACiQ Extreme+ sensor table:

- room temperature T1
- indoor coil T2A / T2B
- outdoor coil T3
- outdoor ambient T4
- EEV position in steps
- compressor running
- actual fan state
- target fan state
- bus operating mode
- C4 operating state
- raw error/protection words

It requires the `midea_xye` climate component to have the ID `heatpump_xye`.

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

The external component should come from the same branch while this profile remains under development:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/JFCrusan/ESPHome-Midea-XYE
      ref: aciq-diagnostics
    components:
      - midea_xye
```

And the climate entry must include:

```yaml
climate:
  - platform: midea_xye
    id: heatpump_xye
    name: Heat Pump
```

## Research / debug packages

These are optional and can be removed without removing the clean ACiQ sensor table.

### `aciq-diagnostics.yaml`

Publishes the established raw C0/C4 fields and readable diagnostic companions. Some older entity names deliberately retain provisional labels so existing Home Assistant history is not needlessly broken by research-driven renames.

### `aciq-byte-watch.yaml`

Temporary passive observer for payload bytes not individually exposed by `aciq-diagnostics.yaml`. Intended to let Home Assistant history act as a long-running protocol flight recorder.

### `aciq-snapshot.yaml`

One-click diagnostic snapshot helper. Requires `aciq-diagnostics.yaml`.

## Full lab configuration

During active reverse engineering the package list can contain all four:

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

When the byte hunt is over, remove the last three files and leave `aciq-extreme-plus.yaml` installed.

See [`../docs/ACIQ_EXTREME_PLUS.md`](../docs/ACIQ_EXTREME_PLUS.md) for the observed byte map, confidence levels, and the reasoning behind the promoted fields.
