# Reverse Engineering the Breezary Levelle

This document describes the hardware investigation and protocol
reverse-engineering used to replace the factory Tuya Wi-Fi firmware with
ESPHome while retaining the original fan receiver and RF remote.

## Hardware

The tested fan is a Breezary Levelle 52-inch ceiling fan with:

- DC fan motor
- six fan speeds
- reversible direction
- white CCT lighting
- dimming
- RGB static colors
- RGB effects
- Tuya / Smart Life support
- RF remote control

The fan receiver contains the actual fan and lighting control electronics.
The removable Tuya module provides the Wi-Fi interface.

The RF remote communicates with the receiver itself, not directly with the
Wi-Fi module. This is important because replacing the Wi-Fi firmware does
not eliminate use of the original remote.

## Tuya Module

The tested module is:

- T1-2S-NL
- BK7238 / T1 family
- 2 MiB flash

Accessible module pads:

`3V3 | GND | RX1 | TX1 | P9 | P24`

The module communicates with the receiver MCU through UART.

## Factory Firmware Backup

Before modifying the module, two complete 2 MiB flash reads were made.

Both produced the same SHA-256:

`6d4e1116fdf3983288f2f55bac36de3c6d93845438623b6ddfb5bd4f24ae7940`

The factory image itself is intentionally not distributed by this project.

Observed factory information included:

- bootloader: `BK7238_T1_2_0_0`
- flash ID: `85 20 15`
- TuyaOS: `tuyaos-iot_3.11.11_T1_wifi_`

The factory firmware build information observed during investigation was
approximately March 2025.

## Flash Access

The BK7238 bootloader was accessed using a 3.3 V USB-to-UART interface.

Connections:

| USB-UART | T1-2S-NL |
|---|---|
| 3.3 V | 3V3 |
| GND | GND |
| TX | RX1 |
| RX | TX1 |

The bootloader handshake was obtained by starting the flashing tool and then
momentarily removing and restoring 3.3 V power to the module.

`bk7231tools` successfully communicated with the BK7238 using the FULL
protocol.

## ESPHome Platform

The working ESPHome configuration uses:

```yaml
bk72xx:
  board: generic-bk7238-tuya

Testing was performed with ESPHome 2026.9.0 and a LibreTiny build identifying
itself as 1.13.0+sha.6514b26.

Receiver UART

An important distinction was discovered between the bootloader UART behavior
and the receiver’s normal runtime protocol.

The working receiver MCU UART configuration is:
uart:
  id: tuya_uart
  rx_pin: RX1
  tx_pin: TX1
  baud_rate: 9600

The runtime UART is:

* 9600 baud
* 8 data bits
* no parity
* 1 stop bit
* non-inverted

The factory firmware had also produced boot/runtime observations at other
stages of investigation, but 9600 baud is the configuration that successfully
communicates with the fan receiver under ESPHome.

Tuya Protocol

The receiver uses the standard Tuya MCU framing beginning with:

55 AA

A normal ESPHome heartbeat is:

55 AA 00 00 00 00 FF

A receiver heartbeat response observed during testing was:

55 AA 03 00 00 01 01 04

The product information reported during investigation included:
{"p":"tiqsjo1pie3b0pmc","v":"1.0.0","m":2,"low":1}

A startup command using command 0x25 was also observed. ESPHome may log this
as an invalid/unsupported Tuya command, but it did not prevent operation of
the fan.

Discovering the Datapoints

The physical RF remote was used as the reference implementation.

Individual remote actions were performed while observing MCU datapoint
reports. This made it possible to correlate physical behavior with Tuya
datapoints without assuming meanings from generic Tuya fan implementations.

Confirmed fan controls ultimately mapped to:

* DP104 — direction
* DP105 — speed
* DP107 — running/stopped

Confirmed lighting controls included:

* DP20 — overall lighting active state
* DP22 — white brightness
* DP23 — white temperature
* DP54 — RGB active
* DP63 — white active

RGB required additional investigation because the datapoints reported by the
receiver were not necessarily the datapoints that should be written.

See datapoints.md for the complete map.

RGB Static Color Discovery

DP24 reports the active static RGB color.

Initially it appeared to be the natural datapoint to write. Direct DP24
writes, however, produced incorrect or transient behavior.

Further testing identified DP61 as the writable/persistent companion.

The resulting architecture is therefore:
DP24 -> reported/live static color
DP61 -> writable/persistent static color

Writing the seven captured factory payloads to DP61 reproduced the physical
remote’s static-color sequence.

RGB Effect Discovery

The same pattern appeared with effects.

DP25 reports the currently active effect, while direct writes did not
reliably reproduce the remote behavior.

DP56 was identified as the writable/persistent companion:
DP25 -> reported/live effect
DP56 -> writable/persistent effect

Writing the captured factory payloads to DP56 reproduced all 11 effects.

Why ESPHome’s Native Light Component Was Not Used

A native ESPHome light: implementation was tested during development.

The receiver’s feedback behavior caused unwanted state interactions,
including white-light activation while manipulating RGB behavior.

The final configuration intentionally exposes the receiver’s functions using
template switches, numbers, and buttons instead.

This more closely represents the receiver’s actual protocol and preserves
the behavior of the physical remote.

Forced Datapoint Writes

ESPHome normally avoids transmitting a datapoint when its cached state already
matches the requested value.

That is useful for state-oriented controls, but it caused problems with the
fan’s button-like behavior. For example, pressing Speed 1 when DP105 was
already cached as 1 could result in no UART transmission.

The final implementation therefore uses ESPHome’s forced Tuya write methods
where a physical command must always be sent.

Examples include:

* force_set_integer_datapoint_value
* force_set_boolean_datapoint_value
* force_set_enum_datapoint_value
* force_set_raw_datapoint_value

This allows repeated fan-speed, stop, direction, RGB-color, and RGB-effect
commands to behave more like the original remote.

Fan Direction Behavior

The receiver itself starts the fan when a direction change is requested while
the fan is stopped.

That behavior originates in the receiver MCU rather than ESPHome.

The production configuration compensates when issuing a direction command
while stopped:

1. write the requested direction
2. wait briefly for the receiver to process it
3. force DP107 back to OFF

This permits the direction setting to be changed without intentionally
leaving a previously stopped fan running.

Factory 2-Hour Timer

The physical remote contains a dedicated 2-hour timer button.

Pressing it consistently produced:
DP103 = 120
DP103 = 7200

The receiver also flashes the light as acknowledgement.

However, reproducing those DP103 writes from ESPHome did not start the same
internal countdown.

Additional values were tested, including 60 seconds, without producing an
arbitrary writable countdown.

The evidence suggests that the RF receiver’s physical timer handler performs
additional internal work and then reports its state through DP103.

For that reason, the production ESPHome firmware does not attempt to emulate
the factory timer through DP103.

Instead, it implements a local ESP timer and sends DP107 OFF when the selected
time expires.

ESP-Local Fan Timer

The production firmware exposes:

Fan Timer Minutes

with a range of 0–240 minutes.

The countdown runs on the ESP itself after being started. Home Assistant does
not need to remain connected for the timer to expire.

The timer is intentionally volatile:

* it does not survive ESP reboot or loss of power
* setting it to zero cancels it
* fan-control activity can cancel it according to the production logic
* lighting operation is independent of the timer

When the countdown expires, only the fan is stopped. Lighting state is left
unchanged.

What This Project Does Not Claim

Several datapoints remain unidentified.

They are documented as observations rather than assigned speculative
functions.

Likewise, the raw RGB payload structures have not been fully decoded. The
project reproduces the factory payloads known to work rather than assigning
unsupported meanings to every byte.

The goal of this work is a reproducible local-control implementation based on
observed hardware behavior.
