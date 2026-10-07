# ESPHome Common

Reusable ESPHome configuration, board definitions, buses, peripherals, and
components used by my ESPHome devices.

This repository is intended to be included as a Git submodule in repositories
containing concrete ESPHome device configurations.

## Design Philosophy

> **Inheritance for what the device IS. Composition for what the device HAS.**

The configuration is organized into layers with clearly defined
responsibilities.

---

## Inheritance — What a Device Is

Base files describe increasingly specific device and platform characteristics.

Example:

    base/base.yaml
        |
        +-- base/base.esp32-devkit-30pin.yaml
                |
                +-- esp-32-garage.yaml
                +-- esp-32-solarshed.yaml

### `base/base.yaml`

Defines behavior common to all devices.

Examples include:

- ESPHome device identity
- Logging
- Home Assistant API
- OTA updates
- Wi-Fi
- Web server
- Safe mode
- Diagnostic sensors
- Common lifecycle behavior

The generic base should not allocate optional hardware resources.

### `base/base.esp32-devkit-30pin.yaml`

Defines characteristics and defaults for a generic 30-pin ESP32 development
board.

It inherits `base.yaml` and provides board-specific configuration such as:

- ESP32 platform and framework
- Status LED
- Default I2C pins
- Other board-level defaults

A board base may define default pins or capabilities, but should not instantiate
optional resources merely because the board supports them.

### Concrete Device Configuration

Concrete device YAML files represent actual deployed devices.

Examples:

    esp-32-garage.yaml
    esp-32-solarshed.yaml

A concrete device:

- Inherits the appropriate board base.
- Defines application-specific behavior.
- Composes the buses and peripherals physically present on the device.
- Owns allocation of shared resources.
- Implements required lifecycle hooks such as `setup_script`.

---

## Composition — What a Device Has

Optional hardware and capabilities are added through packages.

Repository structure:

    esphome-common/
    ├── base/
    │   ├── base.yaml
    │   └── base.esp32-devkit-30pin.yaml
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
    ├── LICENSE
    ├── .gitattributes
    └── .gitignore

---

## Package Responsibilities

### `base/`

Defines what a device **is**.

Base packages contain common behavior and progressively more specific
board/platform definitions.

Base packages may define defaults and capabilities, but should avoid
instantiating optional hardware resources.

---

### `buses/`

Defines shared hardware resources such as:

- I2C
- SPI
- UART
- OneWire

Shared resources are instantiated exactly once by the concrete device that
requires them.

For example:

```yaml
packages:
  i2c_bus: !include common/buses/i2c.yaml
```

---

### `peripherals/`

Defines reusable hardware attached to a shared resource.

Examples include:

- BME280 environmental sensor
- PCA9555 GPIO expander
- PCF8575 GPIO expander

Peripheral packages reference shared resources but do not instantiate them.

For example, an I2C peripheral may contain:

```yaml
i2c_id: main_i2c
```

but should not independently include the I2C bus package.

---

### `components/`

Reserved for reusable higher-level capabilities that combine behavior and
hardware into a functional unit.

Possible future examples include:

- Heater control
- Fan control
- Pump control
- Battery monitoring
- Environmental control

Components should remain reusable and should not contain configuration that is
specific to one deployed device.

---

## Shared Resources

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

ESPHome packages do not deduplicate repeated component definitions merely
because they use the same component ID.

Therefore, each peripheral must NOT independently include the same I2C bus
package. Doing so would instantiate `main_i2c` multiple times and cause a
duplicate-ID validation error.

Instead, the concrete device explicitly composes the shared bus once:

```yaml
packages:
  device_base: !include common/base/base.esp32-devkit-30pin.yaml

  i2c_bus: !include common/buses/i2c.yaml
  environment_sensor: !include common/peripherals/bme280.yaml
  input_expander: !include common/peripherals/pca9555.yaml
  output_expander: !include common/peripherals/pcf8575.yaml
```

This makes ownership of shared resources explicit.

---

## Device Lifecycle Contract

The common base defines a common device initialization lifecycle.

During boot, `base.yaml` executes:

```yaml
esphome:
  on_boot:
    priority: -100
    then:
      - script.execute: setup_script
```

Every concrete device is therefore required to implement a script with the ID:

```yaml
setup_script
```

This behaves conceptually like an abstract method or interface requirement:

    Base Device
        |
        +-- requires setup_script()
                       |
              Concrete Device
                       |
                implements it

The common base deliberately does **not** provide a default implementation.

This makes the lifecycle contract explicit and allows ESPHome validation to
detect a concrete device that fails to provide the required hook.

### Device With Initialization

A device that requires initialization implements the necessary actions:

```yaml
script:
  - id: setup_script
    then:
      - logger.log: "Device ESPHome Boot Process Started"
      - # additional initialization actions
```

### Device With No Initialization

A device that requires no initialization still explicitly satisfies the
contract with a no-op implementation:

```yaml
script:
  - id: setup_script
    then: []
```

A diagnostic boot log is also an acceptable minimal implementation:

```yaml
script:
  - id: setup_script
    then:
      - logger.log: "Device ESPHome Boot Process Started"
```

### Why There Is No Default `setup_script`

ESPHome package lists are merged rather than overridden by matching component
IDs.

Testing with ESPHome 2026.9.1 established that defining a default
`setup_script` in a common package and defining another `setup_script` in a
concrete device produces a duplicate-ID validation error.

Therefore, the common base declares the lifecycle requirement by calling
`setup_script`, while the concrete device owns its implementation.

This behavior is consistent with the resource-allocation rule used elsewhere
in this architecture: definitions with the same component ID should not be
assumed to override or deduplicate one another.

---

## Resource Allocation

Resource ownership should remain explicit.

The general rule is:

> **The concrete consumer owns shared-resource allocation.**

For example, a board base may define:

```yaml
substitutions:
  default_i2c_sda_pin: GPIO21
  default_i2c_scl_pin: GPIO22
```

but does not instantiate an I2C bus.

A concrete device that requires I2C composes:

```yaml
packages:
  i2c_bus: !include common/buses/i2c.yaml
```

The bus package then uses the board-provided defaults.

This separates:

- Board capability
- Resource allocation
- Peripheral implementation
- Application behavior

---

## Example Device

A simple environmental sensor device might contain:

```yaml
packages:
  device_base: !include common/base/base.esp32-devkit-30pin.yaml
  i2c_bus: !include common/buses/i2c.yaml
  environment_sensor: !include common/peripherals/bme280.yaml

substitutions:
  device_internal_name: esp-32-solarshed
  device_internal_id: esp_32_solarshed
  device_friendly_name: ESP-32_SolarShed
  device_ip_address: 192.168.200.158
  device_sampling_time: 30s
  esphome_project_name: esp32.SolarShed
  esphome_project_version: 1.0.0

script:
  - id: setup_script
    then:
      - logger.log: "Solar Shed ESPHome Boot Process Started"
```

---

## Using This Repository as a Git Submodule

A Home Assistant configuration repository can include this repository at:

    esphome/common

Example:

```bash
git submodule add https://github.com/miesch1/esphome-common.git esphome/common
```

The resulting structure is:

    hass/
    └── esphome/
        ├── common/                 <-- esphome-common submodule
        ├── esp-32-garage.yaml
        └── esp-32-solarshed.yaml

Concrete ESPHome configurations can then include packages using paths such as:

```yaml
packages:
  device_base: !include common/base/base.esp32-devkit-30pin.yaml
  i2c_bus: !include common/buses/i2c.yaml
  environment_sensor: !include common/peripherals/bme280.yaml
```

---

## Updating the Common Configuration

The parent repository records a specific commit of the `esphome-common`
submodule.

To update the common repository:

```bash
cd esphome/common
git pull
```

After testing the updated configuration, return to the parent repository and
commit the new submodule revision:

```bash
cd ../..
git add esphome/common
git commit -m "Update ESPHome common configuration"
```

This allows deployed devices to remain pinned to a known-good revision of the
common configuration.

---

## Goals

This repository is intended to:

- Keep common ESPHome behavior in one place.
- Separate board characteristics from optional hardware.
- Make shared-resource ownership explicit.
- Keep peripheral-specific configuration inside peripheral packages.
- Allow concrete devices to compose only the hardware they actually contain.
- Define explicit lifecycle contracts between the base and concrete devices.
- Avoid duplicate shared-resource definitions.
- Keep device-specific application logic out of reusable packages.
- Make ESPHome configurations easier to understand, test, and maintain.
- Allow Home Assistant repositories to pin known-good common configurations.