# ESPHome — M5Stack AirQ

Community [ESPHome](https://esphome.io/) configuration for the [M5Stack AirQ](https://docs.m5stack.com/en/app/AirQ): StampS3 (ESP32-S3), Sensirion SEN55, SCD40, and 1.54″ e-ink display.

This example goes beyond a USB-only plug-and-flash config: it latches the battery **HOLD** line, reads pack voltage, shows a SoC gauge on the display, can shut down on low voltage, and switches between **Auto / Max / Eco** power profiles (SEN55 PM bursts + lower duty cycle on battery).

## Display preview

![M5Stack AirQ e-ink display running this ESPHome config](images/display.jpg)

Photo of a unit running this config (e-ink layout with battery % on the bottom bar). Replace [`images/display.jpg`](images/display.jpg) anytime — see [`images/README.md`](images/README.md). Optional wider hardware-only shot: `images/device.jpg`.

## Requirements

- M5Stack AirQ hardware
- ESPHome `2026.7.4` / ESP-IDF on ESP32-S3, `flash_size: 8MB`
- Home Assistant (native API) recommended for time sync and entities

## Quick start

1. Copy `secrets.yaml.example` → `secrets.yaml` and set Wi-Fi, API encryption key, OTA password, and a strong fallback-hotspot password.
2. Edit substitutions in `airq.yaml`, especially the unique `devicename` and short `friendlyname`; set `location`, `fallback_timezone`, `clock_hours`, `display_temperature_scale`, and any other defaults you want to override there. [`package/airq.yaml`](package/airq.yaml) provides the defaults; leave it unchanged to keep Git updates simple.
3. For the first USB flash, power off the AirQ, hold Button A (`G0`), then connect USB. Release the button after power is applied to enter download mode. See M5Stack's [download-mode instructions](https://docs.m5stack.com/en/arduino/m5air_quality/program).
4. Compile and flash (USB serial for first install; OTA afterward):

```bash
esphome run airq.yaml
```

Or use the ESPHome dashboard / your usual builder workflow. Point the builder at this directory so `secrets.yaml` resolves next to `airq.yaml`.

## Notable features

| Feature | Notes |
| --- | --- |
| HOLD latch (`GPIO46`) | Internal `ALWAYS_ON` switch so battery power stays latched early in boot |
| Battery voltage (`GPIO14`) | 1M/1M divider; YAML multiplies by 2 for pack V |
| Battery % + e-ink gauge | Piecewise LiPo curve; shutdown uses **voltage**, not % |
| Power mode select | Auto / Max / Eco — Auto uses pack V + voltage-rate hysteresis |
| Battery saver | Wi‑Fi `HIGH` (fall back to `LIGHT` if HA disconnects), no `web_server`, SCD4x `low_power_periodic`, SEN55 PM bursts |
| Display | Warm-up screen, SCD40 / SEN55 layout, clock (HA time with SNTP fallback) |
| Buzzer (`GPIO9`) | Onboard passive buzzer with a disabled-by-default `Test Buzzer` button and `play_buzzer_alert` API action |

## Substitutions (high level)

See comments at the top of [`airq.yaml`](airq.yaml). Important ones:

- `devicename` — unique hostname for mDNS, OTA, Home Assistant, and the fallback hotspot; use lowercase letters, digits, and hyphens
- `friendlyname` — short Home Assistant and e-ink display label
- `fallback_timezone` — IANA zone for SNTP when the HA API is down (default `Etc/UTC`)
- `clock_hours` — `"24"` or `"12"`
- `display_temperature_scale` — `"C"` or `"F"` for the e-ink Temp row only (HA temperature entity stays °C)
- `battery_shutdown_voltage` — `0` disables auto power-off; otherwise under-load pack volts for HOLD release
- `usb_detect_voltage` — Auto mode threshold between Max and Eco profiles

## Secrets

```yaml
wifi_ssid: "..."
wifi_password: "..."
api_encryption_key: "..."   # ESPHome API encryption key
ota_password: "..."
fallback_ap_password: "..." # At least 8 characters; protects the recovery hotspot
```

Generate an API key with `esphome wizard` / the ESPHome UI encryption key helper as you prefer.


## Clone-and-run vs Git package

**Local / standalone:** this directory is a complete example. Copy `secrets.yaml.example` → `secrets.yaml`, edit substitutions in [`airq.yaml`](airq.yaml), and compile that wrapper. [`package/airq.yaml`](package/airq.yaml) is pulled in locally; ESPHome does not need GitHub at flash time.

**Remote package:** [ESPHome remote packages cannot contain secret lookups](https://esphome.io/components/packages.html). Point `packages:` at `package/airq.yaml` and pass credentials as substitutions from *your* `secrets.yaml` (any key names you already use):

```yaml
substitutions:
  # Must be unique on your network; use lowercase letters, digits, and hyphens.
  devicename: airq-living-room
  # Keep this short: it is shown on the e-ink display.
  friendlyname: AirQ LR
  location: Living Room
  fallback_timezone: "Etc/UTC"
  clock_hours: "12"
  display_temperature_scale: "F"
  wifi_ssid: !secret wifi_ssid
  wifi_password: !secret wifi_password
  api_encryption_key: !secret api_encryption_key
  ota_password: !secret ota_password
  fallback_ap_password: !secret fallback_ap_password

packages:
  airq:
    url: https://github.com/shomanjk/esphome-m5stack-airq
    ref: main
    files: [package/airq.yaml]
    refresh: 1d
```

Pin `ref` to a tag or commit if you do not want to track `main`. `refresh: 1d` caches the clone for a day; use `0s` while testing a moving branch.

## Relationship to devices.esphome.io

The [official device page](https://devices.esphome.io/devices/m5stack-airq/) hosts a catalog-style example. That site's contribution rules require a secrets-free, hardware-focused `config.yaml` (no `!secret`, limited top-level components). This repository is the full, Home Assistant-oriented configuration with battery management.

## License

MIT — see [LICENSE](LICENSE).

Not affiliated with M5Stack or ESPHome; use at your own risk.
