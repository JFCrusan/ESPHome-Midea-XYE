# ACiQ Extreme+ XYE Field Map

This document records the current enthusiast-grade map for the ACiQ Extreme+ ducted heat-pump family, built from live captures, Home Assistant history, controlled mode/fan tests, and cross-checks against Midea-family service/protocol material.

Tested hardware:

- ACIQ-48-PAH air handler
- ACIQ-48-HPD outdoor unit
- XYE / RS-485, 4800 baud, 8N1

The goal is usefulness, not pretending every internal Midea name is proven. Raw byte positions are retained so interpretations can be corrected later without losing the underlying evidence.

## Confidence legend

- **Very high**: repeated direct correlation with a commanded/physical state.
- **High**: repeatable behavior plus strong protocol/refrigeration evidence.
- **Medium**: plausible and useful, but not yet independently pinned down.
- **Unknown**: preserve raw value only.

## C0 basic query payload

Absolute receive-frame positions are shown below. Payload is bytes 6..29.

| Byte | Current interpretation | Observed ACiQ behavior | Confidence |
|---|---|---|---|
| B06 | Unknown/configuration | Commonly 48 on tested unit | Unknown |
| B07 | Capabilities | Commonly 20 on tested unit | Medium |
| B08 | Operating mode | OFF=0x00, FAN=0x81, DRY=0x82, HEAT=0x84, COOL=0x88, AUTO observed as 0x98 | Very high |
| B09 | Actual fan state | Bit 7=AUTO; low nibble: 0=off, 1=high, 2=medium, 3/4=low | Very high |
| B10 | Target temperature | Raw Celsius setpoint on tested system | Very high |
| B11 | T1 room temperature | `(raw - 40) / 2` °C | Very high |
| B12 | T2A indoor coil/refrigerant sensor | `(raw - 40) / 2` °C | Very high |
| B13 | T2B indoor coil/refrigerant sensor | `(raw - 40) / 2` °C | Very high |
| B14 | T3 outdoor coil sensor | `(raw - 40) / 2` °C | High |
| B15 | Current field / unsupported sentinel | 0xFF on tested ACiQ; do not interpret as 255 A | High |
| B16 | Unknown/reserved | 0 on tested unit | Unknown |
| B17 | Start timer | Usually 0 in captures | Medium |
| B18 | Stop timer | Usually 0 in captures | Medium |
| B19 | Compressor running | 0=idle, 1=running; aligns with independent compressor power | Very high |
| B20 | Mode flags | Usually 0 in current captures | Medium |
| B21 | Operation flags | Usually 0 in current captures | Medium |
| B22-23 | Error flags, little-endian | 0 in normal operation | High |
| B24-25 | Protection flags, little-endian | 0 in normal operation | High |
| B26 | CCM communication/error field | 0 in current captures | Medium |
| B27 | Unknown/configuration | 0 in current captures | Unknown |
| B28-29 | **EEV position, little-endian steps** | Dynamic ~177..480; smooth 16-bit transitions across 255/256; parks at 480; stepwise movement at startup/shutdown | **High** |

### EEV notes

`EEV = B28 | (B29 << 8)`

Evidence on the tested Extreme+ system:

- Observed range in captured operation: roughly 177..480 steps.
- Crossing the 255/256 boundary is smooth, strongly indicating a single 16-bit value rather than two independent fields.
- Compressor shutdown repeatedly drives the value toward 480 in ~62/63-step increments.
- Cooling startup moves from 480 toward a lower operating target before the compressor-running flag becomes active.
- During long cooling runs the value makes smaller control corrections and participates in periodic refrigeration housekeeping events.

The interpretation **EEV position** is considered useful enough to ship. Which physical expansion valve is represented remains provisional until another bus or service tool provides an independent matching value.

## C4 extended query payload

| Byte | Current interpretation | Observed ACiQ behavior | Confidence |
|---|---|---|---|
| B06 | Indoor fan PWM protocol slot | 0 despite large real blower-power changes | Low/usefulness |
| B07 | Indoor fan tach protocol slot | 0 despite large real blower-power changes | Low/usefulness |
| B08 | Unknown/fixed field | 0x80 across compressor on/off states | Unknown |
| B09 | ESP/profile field | 0x30 on tested unit | Medium |
| B10 | Unknown/fixed/protection-style field | 0x8C across normal state changes | Unknown |
| B11 | Coil inlet protocol slot | 0 on tested unit | Unused here |
| B12 | Coil outlet protocol slot | 0 on tested unit | Unused here |
| B13 | Discharge-temperature protocol slot | 0 on tested unit | Unused here |
| B14 | Expansion-valve protocol slot | 0 on tested unit | Unused here |
| B15 | Reserved | 0 on tested unit | Unknown |
| B16 | Operating mode/state | OFF=0x01, FAN=0x81, DRY=0x82, HEAT=0x84, COOL=0x88, AUTO=0x90; 0x08 briefly observed while disabling COOL | Very high |
| B17 | Target/commanded fan | Same family of fan encodings as C0; stays at requested speed while actual fan can differ | Very high |
| B18 | Target temperature | On tested Fahrenheit-configured unit: `raw - 135 = °F` | Very high |
| B19-20 | Fixed metadata/signature | 0xBCD6 / 48342 in all current ACiQ captures; also seen in related Midea-family traffic | High that it is not live telemetry |
| B21 | T4 outdoor ambient | `(raw - 40) / 2` °C | High |
| B22 | Reserved | 0 | Unknown |
| B23 | Reserved | 0 | Unknown |
| B24 | Static-pressure/profile protocol field | 0 on tested unit; did not reflect large filter-pressure/blower-power changes | Low as measured static pressure |
| B25 | Reserved | 0 | Unknown |
| B26 | Unknown/static field | 0x80 in all captured ACiQ states so far | Unknown |
| B27 | Unknown/static field | 0x80 in all captured ACiQ states so far | Unknown |
| B28 | Unknown/static field | 0x80 in all captured ACiQ states so far | Unknown |
| B29 | Unknown/static field | 0x80 in all captured ACiQ states so far | Unknown |

## Fan behavior

C0 B09 reports the actual running fan state.

- bit 7 (`0x80`) indicates automatic fan control
- low nibble reports physical speed
- 0x01 = HIGH
- 0x02 = MEDIUM
- 0x03 or 0x04 = LOW
- 0x00 = OFF

Examples observed on the ACiQ:

- `0x84` = AUTO + LOW
- `0x82` = AUTO + MEDIUM
- `0x81` = AUTO + HIGH
- bare `0x04`, `0x02`, `0x01` = manual LOW/MEDIUM/HIGH

C4 B17 is the target/commanded fan setting and can remain AUTO while C0 shows the physical speed selected underneath AUTO.

## Periodic Housekeeping

Long cooling runs repeatedly showed a distinct **Housekeeping** event at approximately 120.5 minutes after compressor start, and then again about 121.5 minutes later when the run continued long enough.

Observed signature:

- cooling continues rather than changing to a defrost operating state
- EEV position opens abruptly by roughly 57..81 steps
- independent compressor electrical power rises markedly
- the EEV then returns toward its prior modulation range

Midea literature describes oil/lubricant-return routines with similar long-runtime timing, compressor-frequency changes, and EEV movement. For this project the user-facing name is simply **Housekeeping**. "Likely oil return" is an engineering interpretation, not required for the map to remain useful.

No dedicated Housekeeping flag has yet been identified on C0/C4, so the clean ESPHome profile does not synthesize a binary Housekeeping entity at this time.

## Filter / blower experiment

A large real-world airflow-resistance change did **not** move C4 B06, B07, or B24 on the tested air handler:

- no filter: blower about 65 W
- disposable MERV 11: about 80 W
- restrictive washable MERV 11: about 150 W

The C4 fields remained effectively unchanged. This suggests the indoor ECM performs substantial local torque/RPM/static compensation that is not represented by these particular XYE fields on this air handler.

## Package layout

Use `examples/aciq-extreme-plus.yaml` for the clean Home Assistant sensor set.

For ongoing reverse engineering, optionally add:

- `examples/aciq-diagnostics.yaml` - known/raw C0/C4 fields and readable companions
- `examples/aciq-byte-watch.yaml` - temporary observer for remaining payload bytes
- `examples/aciq-snapshot.yaml` - one-click capture helper

The raw packages are deliberately separate so a normal installation can keep the useful ACiQ sensor table without carrying the laboratory instrumentation forever.
