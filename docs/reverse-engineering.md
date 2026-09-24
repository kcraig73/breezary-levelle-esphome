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
