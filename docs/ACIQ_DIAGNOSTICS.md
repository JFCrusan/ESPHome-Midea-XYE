# ACiQ XYE Diagnostic Capture

This branch adds a raw-first Home Assistant diagnostic profile for investigating the XYE/CCM bus used by an ACiQ Extreme+ ducted heat pump.

The diagnostic profile is intentionally non-invasive: it does not change frame generation, polling, climate control, or the protocol parser. It observes the component's most recent validated receive frame and publishes selected absolute byte positions to Home Assistant.

## Why raw-first?

Several XYE fields vary by Midea-derived product family. Publishing uncertain fields under confident names makes later reverse engineering harder, so unknown values remain labelled by command and absolute byte position until controlled captures validate their meaning.

## Enable it

Point ESPHome at this branch while testing:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/JFCrusan/ESPHome-Midea-XYE
      ref: aciq-diagnostics
    components: [midea_xye]
```

Give the existing climate component this ID:

```yaml
climate:
  - platform: midea_xye
    id: heatpump_xye
    name: Heatpump
    # keep the rest of the existing configuration
```

Then merge the `sensor:`, `globals:`, and `interval:` sections from `examples/aciq-diagnostics.yaml` into the ESPHome node.

The diagnostic interval samples the component's last validated receive frame every 200 ms and computes an FNV-1a fingerprint over bytes 0-29. A frame is only republished when its payload changes, which avoids flooding Home Assistant with duplicate states.

## First-pass fields

### C0 QUERY

The profile exposes:

- byte 6 unknown/model field
- byte 7 capabilities
- byte 8 operation mode
- byte 9 raw actual fan mode
- decoded AUTO fan bit and physical fan-speed nibble
- byte 10 raw target-temperature field
- byte 15 raw current field
- byte 16 unknown
- byte 19 compressor-running candidate flag
- byte 20 mode flags
- byte 21 operation flags
- bytes 22-23 raw error flags
- bytes 24-25 raw protection flags
- byte 26 CCM error flags
- bytes 27, 28, 29 unknown fields
- bytes 28-29 combined little-endian as `Candidate EEV Position LE`

The candidate EEV sensor is a research hypothesis, not a confirmed mapping.

### C4 QUERY_EXTENDED

The profile exposes:

- byte 6 indoor fan PWM raw
- byte 7 indoor fan tach raw
- byte 8 compressor flags
- byte 9 ESP profile
- byte 10 protection/status flags
- bytes 11-13 temperature-related raw fields
- byte 14 expansion-valve-related raw field
- byte 16 system status
- byte 17 commanded fan raw
- byte 18 target-temperature raw
- bytes 19-20 combined big-endian engineering word
- byte 21 outdoor-temperature raw
- byte 24 static-pressure raw
- bytes 26-29 subsystem status bytes

The C4 bytes 19-20 value remains deliberately named `Engineering Word BE`; do not assign compressor-Hz or fan-RPM units until controlled captures establish which quantity it tracks on this hardware.

## Suggested capture sequence

Record ESPHome logs and Home Assistant sensor history through distinct operating states:

1. OFF for several minutes
2. FAN_ONLY with LOW, MEDIUM, HIGH, and AUTO fan selections
3. COOL startup from idle
4. COOL under high demand (setpoint well below room temperature)
5. COOL approaching setpoint and compressor ramp-down
6. COOL satisfied / compressor idle
7. HEAT startup and steady operation
8. HEAT approaching setpoint and idle
9. AUX / emergency heat if available
10. A natural defrost event when conditions allow

Make only one deliberate change at a time where practical. The useful evidence is correlation: which raw fields move immediately with a user command, which track compressor loading over seconds/minutes, and which remain hardware constants.

## Initial ACiQ observations

From the first supplied capture:

- C0 byte 9 = `0x84`, consistent with AUTO fan enabled while the physical fan is presently LOW.
- C0 bytes 28-29 = `0x44 0x01`, which combine to 324 little-endian. This is plausible as an EEV step count but remains unconfirmed.
- C4 bytes 19-20 = `0xBC 0xD6` (48342 big-endian). The existing protocol code treats this as an engineering quantity whose exact meaning is not yet established.
- C4 bytes 26-29 are all `0x80` in the supplied running capture, consistent with the currently documented subsystem-OK interpretation.
