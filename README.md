# ESPHome Common

Reusable ESPHome configuration, board definitions, buses, peripherals, and
components used by my ESPHome devices.

This repository is intended to be included as a Git submodule in repositories
containing concrete ESPHome device configurations.

## Background

I started this library because I was tired of updating the same settings and
behavior across so many ESPHome YAML files. I wanted a more object-oriented
approach: inheritance for the common behavior and board characteristics a
device builds on, and composition for the buses and peripherals it actually
uses.

I found inspiration in these two examples and built on their ideas:

- [How I structure my ESPHome configuration files](https://simplyexplained.com/blog/how-i-structure-my-esphome-config-files/?form=MG0AV3)
- [Let's build a room sensor — Part 0 configuration](https://github.com/homeautomatorza/esphome/blob/main/Lets_build_a_room_sensor/Part%200/code.yaml)

As I developed the library, I refined that approach around the limitations of
ESPHome's package merging, validation, and build process. I use inheritance and
composition as organizing ideas, while keeping resource ownership and lifecycle
hooks explicit: each concrete device composes its shared buses once and provides
its own `setup_script` implementation. This avoids relying on repeated component
IDs to behave like object-oriented overrides or automatically deduplicate shared
resources.

My goal is to keep common changes in one place, make each device's configuration
easier to follow, and work within the ESPHome builder's constraints as the
library grows.

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
        +-- base/base.esp32-wroom-30pin.yaml
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

### `base/base.esp32-wroom-30pin.yaml`

Defines characteristics and defaults for a 30-pin ESP32-WROOM development
board.

The name identifies the WROOM module family and the board's 30-pin layout.
ESP32-WROOM identifies the module, while the carrier board determines which
pins and onboard features are exposed. A 38-pin board can use the same module
family while requiring a separate board package.

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
    │   └── base.esp32-wroom-30pin.yaml
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
    ├── examples/
    │   └── esp32-example.yaml
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

### `examples/`

Contains complete example concrete-device configurations demonstrating how to
consume the common library.

Examples are documentation and reference implementations. They are not
automatically included by the common base.

For example:

    examples/esp32-example.yaml

demonstrates:

- ESP32 board inheritance
- I2C bus composition
- BME280 peripheral composition
- Device substitutions
- The required `setup_script` lifecycle hook

Because examples reside inside this repository, their relative `!include`
paths differ from those used by a consuming repository.

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
  device_base: !include common/base/base.esp32-wroom-30pin.yaml

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
  device_base: !include common/base/base.esp32-wroom-30pin.yaml
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

## Consuming This Library as a Git Submodule

The recommended way to consume `esphome-common` is as a Git submodule inside
the `esphome/` directory of a Home Assistant configuration repository.

This keeps reusable ESPHome infrastructure in its own repository while allowing
the consuming repository to pin a specific known-good version.

### 1. Add the Submodule

From the root of the consuming repository:

```bash
git submodule add https://github.com/miesch1/esphome-common.git esphome/common
```

This creates:

    hass/
    ├── .gitmodules
    └── esphome/
        ├── common/
        │   ├── base/
        │   ├── buses/
        │   ├── peripherals/
        │   ├── components/
        │   └── examples/
        │
        └── esp32-example.yaml

The parent repository does not store the contents of `esphome-common`.
Instead, it stores a reference to a specific commit of the submodule.

### 2. Create a Concrete Device

A concrete device configuration in the consuming repository can use the common
packages through the submodule.

For example:

```yaml
packages:
  device_base: !include common/base/base.esp32-wroom-30pin.yaml
  i2c_bus: !include common/buses/i2c.yaml
  environment_sensor: !include common/peripherals/bme280.yaml

substitutions:
  device_internal_name: esp32-example
  device_internal_id: esp32_example
  device_friendly_name: ESP32 Example
  device_ip_address: 192.168.1.100
  device_sampling_time: 30s
  esphome_project_name: esp32.Example
  esphome_project_version: 1.0.0

script:
  - id: setup_script
    then:
      - logger.log: "ESP32 Example ESPHome Boot Process Started"
```

The important distinction is that the include paths are relative to the
concrete device configuration.

A device located directly under `esphome/` therefore uses:

    common/base/...
    common/buses/...
    common/peripherals/...

The example stored inside the `esphome-common` repository itself instead uses
paths such as:

    ../base/...
    ../buses/...
    ../peripherals/...

### 3. Validate the Device

After adding the common packages, validate the concrete ESPHome configuration
before deploying it.

The base configuration requires every concrete device to implement
`setup_script`.

If no device-specific initialization is required, use an explicit no-op:

```yaml
script:
  - id: setup_script
    then: []
```

### 4. Commit the Submodule to the Parent Repository

After adding the submodule and validating the device:

```bash
git add .gitmodules esphome/common
git commit -m "Add ESPHome common submodule"
git push
```

The parent repository now records the exact `esphome-common` commit that should
be used.

---

## Cloning a Repository That Uses the Submodule

When cloning the parent repository onto a new system, initialize its submodules
at the same time:

```bash
git clone --recurse-submodules <parent-repository-url>
```

If the parent repository has already been cloned without its submodules, run:

```bash
git submodule update --init --recursive
```

This checks out the exact `esphome-common` revision recorded by the parent
repository.

---

## Updating `esphome-common`

The parent repository remains pinned to its currently recorded common-library
revision until explicitly updated.

To update the common library:

```bash
cd esphome/common
git pull
```

Validate affected ESPHome devices after pulling the new common configuration.

Then return to the parent repository:

```bash
cd ../..
git add esphome/common
git commit -m "Update ESPHome common submodule"
git push
```

The parent commit records the new submodule revision.

---

## Developing `esphome-common` From the Submodule

The submodule is itself a Git repository.

Changes to common configuration should therefore be committed from inside the
submodule:

```bash
cd esphome/common

git status
git add .
git commit -m "Update ESPHome common configuration"
git push
```

Then return to the parent repository:

```bash
cd ../..
git status
```

Git will show the submodule as changed, for example:

    modified: esphome/common (new commits)

This does **not** mean the common files need to be committed again in the
parent repository. The parent repository only needs to record the new
submodule commit:

```bash
git add esphome/common
git commit -m "Update ESPHome common submodule"
git push
```

Other modified files in the parent repository do not need to be staged or
committed.

---

## Submodule Workflow Summary

The relationship is:

    hass repository
        |
        +-- esphome/
              |
              +-- esp-32-garage.yaml
              +-- esp-32-solarshed.yaml
              |
              +-- common/ --------------------+
                    esphome-common repository |
                                              |
                    base/                     |
                    buses/                    |
                    peripherals/              |
                    components/               |
                    examples/                 |
                                              |
                    Git commit <-------------+

`esphome-common` owns the reusable configuration.

The parent repository owns:

- Concrete device configurations
- Device-specific behavior
- Secrets
- The particular `esphome-common` revision used by those devices

---

## Secrets

Start with [secrets.example.yaml](secrets.example.yaml). It lists every secret
required by the shared base, with comments explaining each field. All values
are placeholders; replace them before use.

Copy the template **beside the concrete device YAML**, not inside the common
submodule. For a consumer using `esphome/common`, run from its repository root:

```powershell
# Run only when esphome/secrets.yaml does not already exist.
Copy-Item ./esphome/common/secrets.example.yaml ./esphome/secrets.yaml
```

If you already have `secrets.yaml`, add only missing fields to that file rather
than overwriting existing credentials. When using the example in this repository,
place its secrets file at `examples/secrets.yaml`.

Add these rules to the **consuming repository's** `.gitignore`:

```gitignore
secrets.yaml
*.secrets.yaml
```

The common submodule's ignore rules do not protect files outside the submodule.
Git ignores untracked files; adding a tracked file to `.gitignore` does not stop
Git tracking its edits. That is why the published template has a different name.
Avoid force-adding the real secrets file.

### Generate the API key

The API encryption key must encode 32 random bytes in base64. Generate one with
Python:

```shell
python -c "import secrets,base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"
```

With ESPHome Device Builder's bundled Python on Windows PowerShell:

```powershell
& "$env:LOCALAPPDATA\io.esphome.builder\python\python.exe" -c "import secrets,base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"
```

Paste the result as the quoted `api_key` value. Home Assistant uses this key to
connect to the device, and this common base reuses it for encrypted OTA uploads.
Keep the existing key when updating an installed device unless intentionally
rotating credentials. Do not generate a new key for every build.

The fallback AP password protects the ESP's recovery hotspot. The web username
and password protect its local web interface. These serve different purposes
from your home Wi-Fi credentials. Use a literal web username such as `esp-admin`.

See the [ESPHome API documentation](https://esphome.io/components/api/) for key
requirements and the [secrets guide](https://esphome.io/guides/faq/#tips-for-using-esphome)
for configuration details.

---

## Versions and Releases

`base/base.yaml` defines the shared default `esphome_project_version`. Board
packages inherit it without repeating it. Changing the root default therefore
flows through every inheritance layer that does not explicitly override it.

Concrete devices can override `esphome_project_version` to identify their own
firmware releases. The included example demonstrates that override. ESPHome's
project version and the Program Version sensor then show the device version,
not the common-library version.

Use a Git tag and GitHub release on this repository to identify a common-library
release independently of device firmware versions. Match the tag to the root
version, prefixed with `v`, and record the tested common commit in each consumer's
submodule. Updating this repository alone does not update consumer submodule pins
or installed firmware.

The existing four-part version format is retained. It works as an ESPHome version
string; tools requiring Semantic Versioning expect three numeric components.
Do not duplicate the root version in board packages or maintain commented-out
version values, which can become stale.

---

## OTA Encryption

The common base requires encrypted ESPHome OTA updates and reuses the consuming
installation's `api_key` secret. ESPHome 2026.9.0 or newer is required.

Before adopting this configuration on an existing device, install firmware built
with ESPHome 2026.9.0 or newer that has API encryption enabled. Keep the existing
OTA password for that initial upgrade. Confirm its OTA log says
`Encryption: offered, plaintext accepted`, then update to the encrypted common
base. After installation, confirm the OTA log says `Encryption: required`.

The `ota_password` secret is no longer used by the common base. Remove any
additional ESPHome OTA password declarations in consuming packages or device
files, because OTA password authentication cannot be combined with required
encryption.

Encryption applies to the `esphome` OTA platform. A separately enabled
`web_server` OTA platform remains a separate, unencrypted upload path.

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