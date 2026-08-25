# Phillips Brothers LED Controller

A physical 9-button LED strip controller built around an ESP32-S3-Zero, designed to control a Zigbee RGBW LED strip via Home Assistant and Node-RED.

## Overview

The controller exposes 9 buttons to Home Assistant via ESPHome. All logic (LED strip control, effects, colour presets) is handled in Node-RED, communicating with the LED strip via Zigbee2MQTT.

The ESP32-S3-Zero also acts as a Bluetooth proxy for Home Assistant, providing BLE coverage in the room.

## Hardware

> **Affiliate disclosure:** as an Amazon Associate I earn from qualifying purchases.

| Title | Description | Qty | Affiliate Link |
|-------|-------------|-----|----------------|
| Waveshare ESP32-S3-Zero | ESP32-S3FH4R2 MCU — 4MB flash, 2MB PSRAM, WiFi + BLE 5 | 1 | https://amzn.to/4wRNUeB Waveshare 2pc <br> https://amzn.to/4wIOBa7 Clone 1pc <br>  https://amzn.to/45HKDnj Clone 3pc|
| Cherry MX switches | Plate-mount mechanical keyboard switches | 9 | https://amzn.to/4wIDi1H |
| Zigbee RGBW LED strip | Z2M exposes state, brightness, colour, colour temp and effect | 1 | https://amzn.to/4wKnqeY |
| Custom PCB | KiCad design, 100x82mm, 4x M3 mounting holes | 1 | See [`gerber/`](gerber/) |
| Panel-mount USB-C connector | Remote 5V input, wired back to the PCB | 1 | https://amzn.to/46jsM6f |

## Button Functions

| Button | GPIO | Function |
|--------|------|----------|
| Power | GPIO5 | Toggle strip on/off |
| Bright Up | GPIO4 | +10% brightness |
| Bright Down | GPIO6 | -10% brightness (floor 10%) |
| Rainbow | GPIO2 | Colorloop effect |
| Random | GPIO1 | Random RGB colour |
| White Cycle | GPIO7 | Cycle cool / 50-50 / warm white |
| Red | GPIO8 | Solid red |
| Green | GPIO9 | Solid green |
| Blue | GPIO10 | Solid blue |

## Software Stack

- **ESPHome** — firmware, exposes buttons as binary sensors to HA
- **Home Assistant** — integration layer
- **Node-RED** — button logic and LED strip control via `light.turn_on` service calls
- **Zigbee2MQTT** — Zigbee LED strip integration

## Node-RED

All button logic lives in [`node-red/flows.json`](node-red/flows.json). Import it from the Node-RED menu (Import → select the file); it creates an "LED Controller" tab.

**It will not work until you repoint the entity IDs to your own.** The flow ships with placeholders:

| Placeholder | What it is |
|-------------|------------|
| `light.underbed_led_strip` | The Zigbee RGBW strip |
| `binary_sensor.led_controller_*_button` | The nine ESPHome buttons |

The Home Assistant server node also needs pointing at your instance on first import.

### Notes

- Colour buttons use `hs_color` rather than `rgb_color`, bypassing HA's stale-state dedup in xy colour space — pressing red twice would otherwise do nothing the second time. Random is the exception and sends `rgb_color`, but three fresh random bytes practically never collide with the current colour, so dedup never bites.
- Brightness buttons reapply an active effect (colorloop) after adjustment, tracked in a `currentEffect` flow variable, since the Zigbee strip doesn't report effect status back.
- Bright up uses `brightness_step_pct`, which is atomic and capped at 100 by HA. Bright down instead reads current brightness and sets `brightness_pct`, because a negative step would walk to 0 and switch the strip off — it needs the 10% floor.
- White cycle steps `color_temp_kelvin` through 6500K / 3250K / 2000K via a flow index variable.

## ESPHome Setup

1. Copy `yaml/led-controller.yaml` to your ESPHome config directory.
2. Add the following to your `secrets.yaml`:
```yaml
api_encryption_key: "your_key_here"
ota_password: "your_password_here"
wifi_ssid: "your_ssid"
wifi_password: "your_password"
ap_password: "fallback_hotspot_password"
```
3. Flash via USB on first install, OTA thereafter.
4. Use DHCP with a router-side IP reservation rather than `manual_ip` on this board.

### Recovery and diagnostics

- If the configured WiFi network is unreachable, the device starts a fallback hotspot (`LED Controller Setup`) with a captive portal, so credentials can be corrected without a USB reflash.
- A web UI is served on port 80 at the device IP, exposing all entities and a restart control.
- `WiFi Signal` and `ESP32 Temperature` sensors are available for diagnosing connectivity faults, since all button logic lives remotely in Node-RED.

## CAD

> ⚠️ **Case design is WIP** — STEP files will be added once the enclosure is finalised.

```
cad/
├── full-assy/    # Full assembly STEP (PCB + enclosure) — coming soon
└── pcb/          # PCB STEP export
```

## PCB

Designed in KiCad. Gerbers in [`gerber/`](gerber/). Manufactured at 100x82mm, 1.6mm FR4, HASL finish.

## Licence

Fully reciprocal — anyone who distributes a modified version, or manufactures and sells a modified board, must publish their changes under the same terms.

- **Firmware** (`yaml/`) — [GPL-3.0-or-later](LICENSE)
- **Hardware** (`gerber/`, `cad/`) — [CERN-OHL-S v2](LICENSE-HARDWARE)

Two licences because GPL covers software and CERN-OHL-S is its hardware equivalent; neither one properly covers the other kind of work.

```
Copyright (c) 2026 Mr-P1nky
This source describes Open Hardware and is licensed under the CERN-OHL-S v2.
```
