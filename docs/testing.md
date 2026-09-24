# Validation and Testing

This document records the physical validation performed against the Breezary Levelle receiver, original RF remote, and the final ESPHome implementation.

The goal was not only to identify Tuya datapoints, but to verify that ESPHome reproduced the actual behavior of the factory controls.

---

## Test Environment

Validation was performed using:

- Breezary Levelle 52-inch ceiling fan
- Original RF remote
- T1-2S-NL module
- BK7238
- ESPHome 2026.9.0
- LibreTiny 1.13.0+sha.6514b26
- Tuya MCU UART at 9600 baud
- Home Assistant for observing and issuing ESPHome controls

---

## White Lighting

### Main Light Toggle

**Result: PASS**

Verified behavior:

- Turning the ESPHome `Light` control ON activates white lighting.
- Turning it OFF disables lighting.
- The receiver reports the expected state through the Tuya datapoints.

Observed white-light state:

```text
DP20 = ON
DP54 = OFF
DP63 = ON
```

Observed OFF state:

```text
DP20 = OFF
DP54 = OFF
DP63 = OFF
```

---

## White Brightness

**Result: PASS**

DP22 was confirmed as white brightness.

Observed working range:

```text
10 to 1000
```

Changing brightness through ESPHome reproduces the receiver's white-brightness control.

Brightness changes select white-light mode if RGB is currently active.

---

## White Temperature

**Result: PASS**

DP23 was confirmed as white color-temperature control.

Observed range:

```text
0 to 1000
```

Observed direction:

```text
0    -> warmest
1000 -> coolest
```

Exact Kelvin values were not determined.

Changing white temperature through ESPHome selects white-light mode if RGB is currently active.

---

## RGB Toggle

**Result: PASS**

Turning RGB ON through ESPHome reproduces the receiver's static RGB mode.

Observed RGB state:

```text
DP20 = ON
DP54 = ON
DP63 = OFF
```

Turning RGB OFF disables lighting.

The receiver recalls the previously selected static RGB color when RGB is enabled again.

---

## Static RGB Colors

**Result: PASS**

The original RF remote's RGB color button was used as the reference.

Seven static color states were captured.

Testing established:

```text
DP24 -> reported/live static RGB color
DP61 -> writable/persistent static RGB color
```

Writing the captured raw payloads to DP61 reproduced the original remote behavior.

The ESPHome `RGB Color` button cycles through the same seven-color sequence.

Physical remote changes are reported through DP24 and are used to keep the ESP's internal color index synchronized.

---

## RGB Effects

**Result: PASS**

The original RF remote exposes 11 RGB effects.

Testing established:

```text
DP25 -> reported/live RGB effect
DP56 -> writable/persistent RGB effect
```

All 11 captured factory effect payloads were written through DP56 and physically verified.

The ESPHome `RGB Effect` button reproduces the original remote's effect cycle.

Physical remote changes are reported through DP25 and are used to keep the ESP's internal effect index synchronized.

---

## Native ESPHome `light:` Testing

**Result: REJECTED FOR PRODUCTION**

A native ESPHome `light:` component was tested during development.

Receiver feedback caused unwanted state behavior, including transitions back to white lighting while working with RGB controls.

The production configuration therefore uses discrete template entities instead of ESPHome's native light abstraction.

This behavior was repeatable enough that the native light implementation was removed rather than worked around.

---

# Fan Validation

## Fan OFF

**Result: PASS**

DP107 was confirmed as fan running/stopped state.

Writing:

```text
DP107 = false
```

stops the fan.

Repeated OFF commands were also tested.

ESPHome's normal Tuya write behavior can suppress a same-value write when the datapoint is already cached as OFF, so the production button uses:

```text
force_set_boolean_datapoint_value(107, false)
```

This ensures every press sends an actual command.

---

## Fan Speed 1

**Result: PASS**

DP105 value `1` selects Speed 1.

A caching issue was found during testing: if DP105 was already cached as `1`, a normal write could be suppressed.

The production implementation therefore:

1. force-writes DP105 = 1
2. waits approximately 150 ms
3. force-writes DP107 = ON

This reliably starts the fan at Speed 1.

---

## Fan Speeds 2–6

**Result: PASS**

The following DP105 values were physically verified:

| DP105 | Physical speed |
|---:|---|
| 1 | Speed 1 |
| 2 | Speed 2 |
| 3 | Speed 3 |
| 4 | Speed 4 |
| 5 | Speed 5 |
| 6 | Speed 6 |

Speeds 2 through 6 were verified to transmit and operate correctly.

Forced DP105 writes are used to ensure repeated button presses still transmit.

---

## Fan Direction

**Result: PASS**

DP104 was physically verified as direction:

```text
0 -> Counter-Clockwise
1 -> Clockwise
```

The original receiver has an important native behavior:

> Changing direction while the fan is stopped starts the fan.

This behavior originates in the receiver MCU.

The production ESPHome implementation compensates when issuing a direction command while the fan is stopped:

1. send the new direction
2. wait approximately 500 ms
3. force DP107 back to OFF

This allows direction to be changed without intentionally leaving the fan running.

Physical remote direction changes are also reported back through DP104.

---

# Factory Timer Investigation

## Physical 2-Hour Timer Button

**Result: OBSERVED**

Pressing the physical remote's 2-hour timer button consistently produced:

```text
DP103 = 120
DP103 = 7200
```

The lighting also flashes briefly as acknowledgement.

Repeated presses produce the same sequence rather than behaving as a simple toggle.

Stopping the fan manually results in:

```text
DP103 = 0
```

No periodic countdown updates were observed over UART.

---

## Direct DP103 Writes

**Result: DID NOT REPRODUCE FACTORY TIMER**

The following direct ESP writes were tested:

```text
DP103 = 1
DP103 = 60
DP103 = 120
DP103 = 7200
```

Additional sequence testing included:

```text
1 -> 60
```

The receiver accepted and echoed the writes, and some writes caused the same lighting acknowledgement flash.

However, they did not start the same internal timed shutdown as the physical remote.

A clean `DP103 = 60` test was allowed to run well beyond 60 seconds and the fan remained running.

The factory timer therefore appears to involve additional receiver-side logic beyond a simple writable countdown value.

---

# ESP-Local Fan Timer

Because the factory DP103 behavior could not be reproduced reliably through UART writes, the final firmware implements the shutdown timer locally on the ESP.

The exposed entity is:

```text
Fan Timer Minutes
```

Range:

```text
0 to 240 minutes
```

---

## Local Timer Expiration

**Result: PASS**

Test procedure:

1. Start fan at Speed 1.
2. Set `Fan Timer Minutes` to 1.
3. Wait for expiration.

Observed result:

- approximately 60 seconds later, ESPHome sent DP107 OFF
- receiver stopped the fan
- timer value returned to 0

The timer therefore operates independently of Home Assistant once started.

---

## Timer Cancellation by ESPHome Speed Command

**Result: PASS**

Test procedure:

1. Start the fan.
2. Set a 1-minute local timer.
3. Issue Speed 2 before expiration.

Observed result:

- timer value returned to 0
- fan continued running
- fan did not stop at the original expiration time

---

## Timer Cancellation by Physical Remote Speed Change

**Result: PASS**

Test procedure:

1. Start the local timer.
2. Change fan speed using the original RF remote.

Observed result:

- the new DP105 speed report was received
- the ESP-local timer was canceled
- timer value returned to 0

---

## Lighting Interaction with Fan Timer

**Result: PASS**

While a local fan timer was active, the following lighting actions were tested:

- white lighting controls
- RGB toggle
- static RGB color changes
- RGB effect changes

Observed result:

- lighting actions did not cancel the timer
- when the timer expired, the fan stopped
- lighting remained active

This confirms that the ESP-local timer controls fan shutdown only.

---

## Direction Interaction with Fan Timer

During final physical testing, changing fan direction did **not** cancel the active timer.

The timer continued to expiration and stopped the fan normally.

This is recorded as the observed production behavior.

The current ESPHome configuration also contains timer-cancellation logic associated with direction handling, so this specific interaction should be treated as an observed result rather than a generalized protocol claim.

---

# Final Functional Status

The following functions were physically reproduced through ESPHome:

| Function | Status |
|---|---|
| White light ON/OFF | PASS |
| White brightness | PASS |
| White temperature | PASS |
| RGB ON/OFF | PASS |
| Seven static RGB colors | PASS |
| Eleven RGB effects | PASS |
| Fan OFF | PASS |
| Fan Speed 1 | PASS |
| Fan Speeds 2–6 | PASS |
| Counter-clockwise direction | PASS |
| Clockwise direction | PASS |
| Physical remote synchronization | PASS |
| ESP-local fan timer | PASS |
| Fan-only shutdown at timer expiration | PASS |
| Lighting preserved at timer expiration | PASS |

The factory DP103 timer protocol remains only partially understood and is intentionally not used for the production timer implementation.
