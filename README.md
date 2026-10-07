# ESPHome Common

Reusable ESPHome configuration, board definitions, buses, peripherals, and
components used by my ESPHome devices.

This repository is intended to be included as a Git submodule in repositories
containing concrete ESPHome device configurations.

## Design Philosophy

> **Inheritance for what the device IS. Composition for what the device HAS.**

The configuration is organized into layers with clearly defined
responsibilities.

### Inheritance — What a Device Is

Base files describe increasingly specific device and platform characteristics.

Example:

    base/base.yaml
        |
        +-- base/esp32-devkit-30pin.yaml
                |
                +-- esp-32-garage.yaml
                +-- esp-32-solarshed.yaml

- `base/base.yaml`
  - Common ESPHome services and behavior.
  - No board-specific or application-specific hardware.

- `base/esp32-devkit-30pin.yaml`
  - Characteristics and defaults for a generic 30-pin ESP32 development board.
  - Inherits `base.yaml`.
  - Describes hardware capabilities and default assignments without allocating
    optional resources.

- Concrete device YAML
  - Inherits the appropriate board base.
  - Defines application-specific behavior.
  - Composes the hardware and capabilities physically present on the device.

## Composition — What a Device Has

Optional hardware is added through packages.

Repository structure:

    esphome-common/
    ├── base/
    │   ├── base.yaml
    │   └── esp32-devkit-30pin.yaml
    │
    ├── buses/
    │   └── i2c.yaml
    │
    ├── peripherals/
    │   ├── bme280.yaml
    │   ├── pca9555.yaml
    │   └── pcf8575.yaml
    │
    ├── components/
    │
    ├── README.md
    └── .gitignore

### Shared Resources

Shared hardware resources such as I2C buses are instantiated exactly once by
the concrete device.

Peripheral packages reference shared resources but do not instantiate them.

For example:

    esp-32-garage.yaml
             |
             +------ I2C Bus ------+
             |                     |
             +-- BME280 -----------+
             +-- PCA9555 ----------+--> main_i2c
             +-- PCF8575 ----------+

ESPHome packages do not deduplicate repeated component definitions by ID.

Therefore, each peripheral must NOT independently include the same I2C bus
package. Doing so would instantiate `main_i2c` multiple times and cause a
duplicate-ID validation error.

Instead, the concrete device explicitly composes the shared bus once:

```yaml
packages:
  device_base: !include common/base/esp32-devkit-30pin.yaml

  i2c_bus: !include common/buses/i2c.yaml
  environment_sensor: !include common/peripherals/bme280.yaml
  input_expander: !include common/peripherals/pca9555.yaml
  output_expander: !include common/peripherals/pcf8575.yaml