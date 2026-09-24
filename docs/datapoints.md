# Breezary Levelle Tuya Datapoints

This document records the datapoints observed while reverse-engineering the
Breezary Levelle ceiling fan.

The descriptions below distinguish between behavior that was physically
confirmed and datapoints whose purpose remains unknown.

## Confirmed Lighting Datapoints

| DP | Type | Observed function |
|---|---|---|
| 20 | Boolean | Tracks overall lighting active/power state |
| 22 | Value | White brightness, observed range 10–1000 |
| 23 | Value | White color-temperature control, 0–1000 |
| 24 | Raw | Reported/live static RGB color |
| 25 | Raw | Reported/live RGB effect |
| 54 | Boolean | RGB lighting active |
| 55 | Enum | RGB operating mode |
| 56 | Raw | Writable/persistent RGB effect |
| 61 | Raw | Writable/persistent static RGB color |
| 63 | Boolean | White lighting active |

### Lighting state

Observed state combinations:

| State | DP20 | DP54 | DP63 |
|---|---:|---:|---:|
| White | ON | OFF | ON |
| RGB | ON | ON | OFF |
| Off | OFF | OFF | OFF |

DP20 is therefore treated as overall lighting activity rather than as the
white-light selector.

### DP55 RGB mode

Observed values:

- `0` — pauses/freezes an effect
- `1` — static RGB
- `2` — RGB effect mode

The ESPHome implementation does not expose the pause state.

## Static RGB Colors

The physical remote cycles through seven static colors.

DP24 reports the currently active color. Direct writes to DP24 did not
reliably reproduce the remote behavior.

DP61 was confirmed as the writable companion datapoint and reproduces the
remote's static-color behavior.

Remote sequence:

| Index | DP61 raw payload |
|---:|---|
| 1 | `00:00:00:0C:00:00:00:03:E8:03:E8` |
| 2 | `00:00:00:0C:00:00:78:03:E8:03:E8` |
| 3 | `00:00:00:0C:00:00:F0:03:E8:03:E8` |
| 4 | `00:00:00:0C:00:00:28:03:E8:03:E8` |
| 5 | `00:00:00:0C:00:00:18:03:E8:03:E8` |
| 6 | `00:00:00:0C:00:00:A8:03:E8:03:E8` |
| 7 | `00:00:00:0C:00:01:4A:03:E8:03:E8` |

The sequence then wraps to the first color.

During physical testing:

- value `40` corresponded to yellow
- value `168` corresponded to cyan
- value `330` corresponded to magenta

No broader interpretation of the raw color encoding is assumed here.

## RGB Effects

DP25 reports the active effect.

DP56 was confirmed as the writable/persistent companion datapoint and
reproduces all 11 factory effects.

### Effect 1

`01:17:03:5E:5E:60:00:00:64:00:38:2F:00:1E:5C:00:D5:45:01:1A:64`

### Effect 2

`01:18:02:64:64:F0:00:00:64:00:B2:39:01:0A:64:01:2D:64:01:3F:64`

### Effect 3

`01:47:05:4D:4D:00:00:00:64:01:03:45:00:C1:43`

### Effect 4

`01:49:07:03:03:00:00:00:64:00:DA:37:01:52:41:00:5C:37`

### Effect 5

`01:4B:09:32:32:00:00:00:64:01:03:45:00:41:3A:00:25:4B:00:5E:42`

### Effect 6

`01:4C:0C:32:32:00:00:00:64:00:D8:4D:00:C1:43:01:03:45:00:5C:37`

### Effect 7

`01:4F:0F:19:19:00:00:00:64:00:BC:64:00:2D:4E:00:00:64:00:64:3C`

### Effect 8

`01:20:0A:55:55:60:00:00:64:00:C2:58:01:3E:33:00:FF:46:01:1D:64`

### Effect 9

`01:24:0A:4B:4B:60:00:00:64:00:BC:26:00:D6:55:01:18:64:00:F9:4D`

### Effect 10

`01:29:02:61:61:E0:00:00:64:00:0B:64:00:D9:64:00:2B:64:00:91:64:00:B9:64`

### Effect 11

`01:00:0A:50:50:00:00:00:64:00:00:64:00:02:64:00:78:64:00:74:64:00:E6:64:00:E6:64`

The sequence then wraps.

## Confirmed Fan Datapoints

| DP | Type | Observed function |
|---|---|---|
| 104 | Enum | Fan direction |
| 105 | Value | Fan speed, 1–6 |
| 107 | Boolean | Fan running/stopped |

### DP104 direction

Physically confirmed:

- `0` — counter-clockwise
- `1` — clockwise

The receiver's native behavior is to start the fan when direction is changed
while the fan is stopped. The ESPHome implementation compensates for this
when issuing a direction command while stopped.

### DP105 speed

Values `1` through `6` correspond directly to the six fan-speed buttons on
the physical remote.

### DP107 fan state

- `false` — fan stopped
- `true` — fan running

Forced datapoint writes are used by the ESPHome buttons so repeated commands
are transmitted even when ESPHome already has the same value cached.

## DP103 — Factory 2-Hour Timer

The physical remote has a dedicated 2-hour timer button.

Pressing it consistently caused the MCU to report:

1. `DP103 = 120`
2. immediately followed by `DP103 = 7200`

The light also flashes as acknowledgement.

Stopping the fan manually causes the MCU to report `DP103 = 0`.

The receiver does not stream a countdown over UART.

Direct ESP writes were also tested:

- `DP103 = 1` followed by `60`
- `DP103 = 120`
- `DP103 = 7200`
- `DP103 = 60`

The receiver accepted and echoed these writes, but they did not reproduce
the physical remote's timed fan shutdown.

The evidence therefore establishes DP103 as timer-associated state/action,
but does not establish a writable arbitrary countdown protocol.

The production ESPHome configuration uses an ESP-local shutdown timer
instead.

## Other Observed Datapoints

These datapoints were observed but their functions were not established:

| DP | Observed value/type |
|---|---|
| 21 | Enum, accepts 0/1/2; no visible effect established |
| 34 | Boolean, observed `false` |
| 57 | Raw, observed empty |
| 102 | Boolean, observed `true` |
| 106 | Enum, observed `0` |
| 108 | Boolean, observed `false` |
| 112 | Enum, observed `0` |
| 113 | Value, observed `22` |
| 116 | Boolean, observed `false` |

DP116 remained off during the lighting tests.

No function is assigned to these datapoints without additional physical
evidence.
