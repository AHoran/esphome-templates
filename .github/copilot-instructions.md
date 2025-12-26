# Copilot Instructions for esphome-templates

This is a public **template repository** for ESPHome tank-level monitoring setups, designed to be reused across multiple projects via GitHub package includes.

## Project Structure

- `templates/tanksensor_base.yaml`: Core template with ultrasonic A02YYUW sensor, battery ADC, and derived tank volume/percentage sensors. Fully parameterized via substitutions.
- `templates/optional_bme280.yaml`: Optional I2C BME280 environmental sensors (temperature, pressure, humidity).
- `tanksensor.yaml`: Example config for ESP32-S3 with BME280 and logger enabled.
- `tanksensor-small.yaml`: Example config for ESP32-C3 without BME280, includes LED strip.
- `README.md`: User-facing guide for importing and customizing templates.

## Key Design Principles

1. **Substitutions over hardcoding**: All pins, timing, geometry, and ADC calibration are substitution-driven.
2. **No secrets in repo**: Wi-Fi, OTA, and API encryption keys use `!secret` references.
3. **Modular packages**: Base template is always included; BME280 is optional.
4. **Battery-first**: Deep sleep, throttled sensors, efficient update intervals (ADC 29s, BME280 30s, ultrasonic 5s throttle).

## Core Substitutions

All per-tank overrides happen in the user's config via `substitutions:`:

- `name`, `friendly_name`: device identity
- `board`: ESP32 variant (default `esp32-s3-devkitc-1` in examples; override for C3, C6, etc.)
- `run_duration_s`, `sleep_duration_s`: deep sleep timings (both in seconds)
- `uart_tx_pin`, `uart_rx_pin`, `uart_baud_rate`: A02YYUW distance sensor
- `adc_pin`, `adc_multiplier`, `adc_attenuation`: battery voltage (multiplier calibrates voltage divider)
- `water_depth_mm`, `sensor_offset_mm`, `capacity_liters`: tank geometry
- `i2c_sda_pin`, `i2c_scl_pin`, `bme280_address`: optional BME280 I2C

## Sensor Derivation

The A02YYUW ultrasonic sensor (distance from sensor to water surface in mm) is converted to:

1. **Water height** (mm from tank bottom): `water_height = water_depth - (distance - sensor_offset)`
2. **Tank percentage**: `percent = (water_height / water_depth) * 100`
3. **Tank volume** (L): `volume = (capacity * percent) / 100`

Derived values are published via template sensors `volume_storage` (device_class) and `percentage_storage` (state_class: measurement).

## Usage Pattern

Users create a config per tank that:

```yaml
substitutions:
  name: kitchen_tank
  board: esp32-c3-devkitm-1
  uart_tx_pin: GPIO21
  # ... override any needed pins/geometry ...

esp32:
  board: ${board}
  framework:
    type: arduino  # or esp-idf

api:
  encryption:
    key: !secret api_encryption_key

ota:
  password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

packages:
  base: github://ahoran/esphome-templates/templates/tanksensor_base.yaml@main
  # Uncomment if tank has BME280:
  # climate: github://ahoran/esphome-templates/templates/optional_bme280.yaml@main
```

The user defines `esp32` block, Wi-Fi, API, OTA, and any board-specific extras (e.g., LED strips). The templates inject the sensor logic.

## Debugging & Development

- **Disable deep sleep** temporarily if doing rapid flashing or debugging logs.
- **Enable logger** in user configs (examples do this) to see sensor values and startup sequence.
- **Test locally** with `!include templates/tanksensor_base.yaml` before pushing to GitHub.
- **Tank geometry verification**: Measure sensor offset (from surface to sensor), water depth (tank bottom to overflow), and total capacity. Adjust substitutions accordingly.

## When Modifying Templates

- Keep substitutions stable (don't rename them without updating examples).
- Test changes on both example configs (S3 + BME280, C3 no BME280).
- Ensure no hardcoded secrets leak into any file.
- Update README if adding new substitutions or changing behavior.
- Increment version (git tag) if making breaking changes.

## Home Assistant Integration

Entities automatically appear in HA after flashing:
- `sensor.<device>_tank_volume` (L, device_class: volume_storage)
- `sensor.<device>_tank_percentage_full` (%)
- `sensor.<device>_battery_voltage` (V)
- `sensor.<device>_temperature`, `_pressure`, `_humidity` (if BME280 included)

No special HA config needed; ESPHome API discovery handles it.
