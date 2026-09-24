# Breezary Levelle ESPHome

Local ESPHome firmware and reverse-engineering documentation for the Breezary Levelle ceiling fan using the Tuya T1-2S-NL / BK7238 Wi-Fi module.

This project replaces the factory Tuya Wi-Fi firmware while retaining the original fan receiver, RF remote, fan controls, white lighting, RGB colors, and RGB effects.

## Status

The tested implementation reproduces the fan's primary functions locally through ESPHome:

- White light ON/OFF
- White brightness
- White color temperature
- RGB ON/OFF
- Seven factory static RGB colors
- Eleven factory RGB effects
- Fan OFF
- Fan speeds 1–6
- Clockwise / counter-clockwise direction
- Original RF remote synchronization
- ESP-local fan shutdown timer

The configuration has been physically tested against the original Breezary Levelle receiver and remote.

---

## Hardware

Tested hardware:

| Item | Value |
|---|---|
| Fan | Breezary Levelle 52-inch ceiling fan |
| Wi-Fi module | T1-2S-NL |
| MCU | BK7238 / T1 family |
| Flash | 2 MiB |
| ESPHome board | `generic-bk7238-tuya` |
| Receiver UART | 9600 baud |
| ESPHome tested | 2026.9.0 |
| LibreTiny tested | 1.13.0+sha.6514b26 |

Accessible module pads:

```text
3V3 | GND | RX1 | TX1 | P9 | P24
```

The original RF remote communicates with the fan receiver rather than directly with the Wi-Fi module, so the remote continues to function after replacing the Tuya firmware.

---

## ESPHome Configuration

The production configuration is:

[`breezary-levelle.yaml`](breezary-levelle.yaml)

Copy the example secrets file:

```text
secrets.example.yaml
```

to:

```text
secrets.yaml
```

and populate your own values.

Required secrets:

```yaml
wifi_ssid: "YOUR_WIFI_SSID"
wifi_password: "YOUR_WIFI_PASSWORD"

api_encryption_key: "YOUR_ESPHOME_API_ENCRYPTION_KEY"
fallback_ap_password: "CHANGE_ME"
```

The real `secrets.yaml` file is excluded by `.gitignore`.

---

## UART Configuration

The working Tuya MCU connection is:

```yaml
uart:
  id: tuya_uart
  rx_pin: RX1
  tx_pin: TX1
  baud_rate: 9600

tuya:
  id: tuya_mcu
  uart_id: tuya_uart
```

Although other serial behavior was observed while investigating the factory firmware and bootloader, **9600 baud** is the runtime configuration confirmed to work between ESPHome and the receiver MCU.

---

## Fan Controls

The receiver exposes the fan through three confirmed datapoints:

| DP | Function |
|---|---|
| 104 | Direction |
| 105 | Speed 1–6 |
| 107 | Running / stopped |

Direction was physically confirmed as:

```text
0 -> Counter-Clockwise
1 -> Clockwise
```

The ESPHome interface intentionally uses:

- `Fan OFF`
- `Fan Speed 1`
- `Fan Speed 2`
- `Fan Speed 3`
- `Fan Speed 4`
- `Fan Speed 5`
- `Fan Speed 6`
- Direction selector

rather than a conventional fan ON/OFF entity.

This more closely matches the behavior of the original remote and receiver.

---

## Lighting Controls

Confirmed lighting datapoints include:

| DP | Function |
|---|---|
| 20 | Overall lighting active state |
| 22 | White brightness |
| 23 | White color temperature |
| 24 | Reported/live static RGB color |
| 25 | Reported/live RGB effect |
| 54 | RGB active |
| 55 | RGB operating mode |
| 56 | Writable/persistent RGB effect |
| 61 | Writable/persistent static RGB color |
| 63 | White active |

### White mode

Observed state:

```text
DP20 = ON
DP54 = OFF
DP63 = ON
```

### RGB mode

Observed state:

```text
DP20 = ON
DP54 = ON
DP63 = OFF
```

### Lighting off

Observed state:

```text
DP20 = OFF
DP54 = OFF
DP63 = OFF
```

---

## RGB Report / Write Datapoint Pairs

One of the more important findings during reverse engineering was that the datapoint reported by the receiver was not always the datapoint that should be written.

### Static RGB

```text
DP24 -> reported/live static color
DP61 -> writable/persistent static color
```

The seven captured factory color payloads reproduce the original remote's color sequence when written to DP61.

### RGB effects

```text
DP25 -> reported/live effect
DP56 -> writable/persistent effect
```

The eleven captured factory effect payloads reproduce the original remote's effect sequence when written to DP56.

See [`docs/datapoints.md`](docs/datapoints.md) for the captured payloads.

---

## Why This Does Not Use ESPHome's Native `light:` Component

A native ESPHome `light:` implementation was tested during development.

The receiver's feedback behavior caused unwanted interactions, including RGB operations resulting in white-light state changes.

The final firmware therefore models the receiver using discrete template switches, numbers, buttons, and a direction selector rather than forcing the hardware into ESPHome's standard light abstraction.

This preserves the receiver's native behavior more reliably.

---

## Forced Tuya Writes

ESPHome normally suppresses writes when its cached datapoint value already matches the requested value.

That caused issues with controls that behave like physical buttons. For example, pressing Speed 1 again might not transmit anything if DP105 was already cached as `1`.

The production firmware therefore uses forced Tuya writes where commands must always reach the receiver.

Examples include:

```text
force_set_integer_datapoint_value
force_set_boolean_datapoint_value
force_set_enum_datapoint_value
force_set_raw_datapoint_value
```

---

## Fan Timer

The original RF remote has a factory 2-hour timer.

Physical testing showed:

```text
DP103 = 120
DP103 = 7200
```

when the remote timer button is pressed.

Directly writing those values from ESPHome did not reproduce the receiver's internal shutdown timer, so DP103 is not used by the production implementation.

Instead, ESPHome provides a local:

```text
Fan Timer Minutes
```

control with a range of:

```text
0–240 minutes
```

Once started, the countdown runs on the ESP itself and does not require Home Assistant to remain connected.

When the timer expires:

- the fan stops
- lighting remains unchanged
- the timer returns to zero

The timer does not survive an ESP reboot or loss of power.

---

## Flashing

The BK7238 bootloader was successfully accessed with `bk7231tools`.

The tested application image begins at:

```text
0x11000
```

An example direct-write command is:

```bash
bk7231tools write_flash \
  -d /dev/cu.usbserial-XXXX \
  --timeout 30 \
  -s 0x11000 \
  breezary-levelle-image_bk7238_app.0x011000.rbl
```

Replace the serial-device path and image filename with the values for your system.

The bootloader handshake was obtained by starting the flashing command and then momentarily removing and restoring 3.3 V power to the module.

### Wiring

| USB-UART | T1-2S-NL |
|---|---|
| 3.3 V | 3V3 |
| GND | GND |
| TX | RX1 |
| RX | TX1 |

The factory firmware should be backed up before flashing replacement firmware.

The original factory image is **not distributed** by this repository.

---

## Factory Firmware Backup Reference

Two complete factory 2 MiB reads were made before modification.

Both produced:

```text
SHA-256
6d4e1116fdf3983288f2f55bac36de3c6d93845438623b6ddfb5bd4f24ae7940
```

Observed factory information included:

```text
Bootloader: BK7238_T1_2_0_0
Flash ID:   85 20 15
TuyaOS:     tuyaos-iot_3.11.11_T1_wifi_
```

This hash is provided only as a reverse-engineering reference.

---

## Documentation

Detailed project notes are available here:

- [`docs/datapoints.md`](docs/datapoints.md) — confirmed and observed Tuya datapoints, RGB payloads, and DP103 investigation
- [`docs/reverse-engineering.md`](docs/reverse-engineering.md) — hardware, UART, flashing, and protocol-discovery process
- [`docs/testing.md`](docs/testing.md) — physical validation and final test results

---

## Known Limitations

The project intentionally does not assign functions to datapoints that were not physically established.

Several datapoints remain unidentified.

The raw RGB payload format has also not been fully decoded. The implementation reproduces known-good factory payloads rather than claiming unsupported meanings for every byte.

The exact Kelvin values represented by DP23 were not established.

The factory DP103 timer protocol remains only partially understood.

---

## Safety

This project involves modification of electronics installed in a mains-powered ceiling fan.

Programming and UART work should be performed with the Wi-Fi module electrically isolated from mains power and powered only from an appropriate low-voltage programming interface.

---

## License

This project is released under the MIT License.

See [`LICENSE`](LICENSE).
