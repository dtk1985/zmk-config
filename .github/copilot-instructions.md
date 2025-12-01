# AI Coding Instructions for ZMK Keyboard Config

This is a **ZMK firmware configuration** repository for a handwired 65% keyboard (Sick68 HW) using a nice!nano v2 controller.

## Project Overview

- **Purpose**: Custom ZMK keyboard firmware build configuration
- **Hardware**: 5×15 keyboard matrix (68 keys) wired to nice!nano v2 nRF52840 microcontroller
- **Build System**: West (Zephyr manifest tool) + GitHub Actions
- **Key Files**:
  - `build.yaml` - CI/CD matrix configuration (board/shield combinations)
  - `config/west.yml` - ZMK project manifest and dependencies
  - `boards/shields/sick68_hw/` - Shield definition and keymap

## Architecture & Key Concepts

### Shield vs Board Structure
- **Shield** (`boards/shields/sick68_hw/`): Device-specific hardware definition for this keyboard
  - `.yaml` metadata (file format, ID, required controller type)
  - `.overlay` (device tree) - GPIO pin mappings for rows/columns
  - `.keymap` (ZMK keymap) - key bindings and layers
  - `Kconfig.*` - Build system configuration
- **Board** (configured in `build.yaml`): The microcontroller (`nice_nano_v2`)

### Hardware Pin Mapping Challenge
The original QMK config used ATmega32u4 pins. Migration to nRF52840 required careful translation:
- QMK ATmega pins → ZMK pro_micro abstraction (pins 0-10, 14-21)
- **Critical Issue**: QMK uses pins B0 and D5 (not on pro_micro), solved by mapping to direct GPIO (P1.01, P1.02)
- See detailed comments in `sick68_hw.overlay` for full mapping reference

### Configuration Hierarchy
1. `build.yaml` - Which board+shield combinations to build
2. `config/sick68_hw.conf` - Global ZMK settings (low power, battery reporting for wireless)
3. `boards/shields/sick68_hw/Kconfig.defconfig` - Shield-specific defaults
4. `sick68_hw.keymap` - Layer definitions (base + function layers)

## Development Workflows

### Building Firmware Locally
```bash
# Install west (Zephyr build tool): pip install west
west init -l config
west update
west build -s config -b nice_nano_v2 -- -DSHIELD=sick68_hw
# Output: build/zephyr/zmk.uf2
```

### CI/CD Build
- Triggered on push/PR/manual dispatch
- Defined in `.github/workflows/build.yml` → calls ZMK's standard build workflow
- Matrix configuration in `build.yaml` (currently: nice_nano_v2 + sick68_hw only)

### Modifying Keymap
1. Edit `sick68_hw.keymap` - Update key bindings or add layers
2. Rebuild and reflash to test: UF2 drag-and-drop to nice!nano in bootloader mode
3. Reference: ZMK keymap documentation for `&kp` (key press), `&mo` (momentary layer), etc.

## Key Patterns & Conventions

### Device Tree Overlay (`.overlay` files)
- Uses Zephyr device tree syntax with ZMK bindings
- `kscan0` defines the keyboard matrix scanner (GPIO pins, diode direction)
- `matrix_transform` defines key position layout (68 keys in 5×15 grid)
- GPIO references like `&pro_micro 0` use pin abstraction layer

### Keymap Layer Definition
```zephyr
default_layer {
    display-name = "Base";
    bindings = < ... >;  // 68 key bindings matching layout
};
```
- `&trans` - pass through to next layer
- `&mo 1` - momentary layer 1 (function layer in this config)
- Rows/columns must match matrix size (5 rows shown as individual rows in keymap)

### Configuration (`.conf` files)
- Key-value format: `CONFIG_NAME=value`
- `CONFIG_ZMK_SLEEP=y` - Enable low power mode for wireless
- `CONFIG_ZMK_BATTERY_REPORTING=y` - Important for nice!nano battery monitoring

## Important Constraints & Gotchas

1. **GPIO Limitations**: 68-key keyboard needs 20 pins (5 rows + 15 cols) but nice!nano/SuperMini only exposes 18. Solution uses direct GPIO test pads.
2. **Diode Direction**: Matrix uses COL2ROW (columns drive rows sense) - must match physical wiring
3. **ZMK Keymap Syntax**: Different from QMK. Uses ZMK behaviors (`&kp`, `&mo`, etc.) not QMK keycodes
4. **Nice!Nano Power**: Firmware config (CONFIG_ZMK_SLEEP) critical for battery life in wireless mode
5. **Zephyr Version**: Locked by `config/west.yml` manifest (v0.3) - do not arbitrarily upgrade without testing

## Files to Know

| File | Purpose |
|------|---------|
| `build.yaml` | Which board/shield to build |
| `config/west.yml` | ZMK project dependencies and version |
| `config/sick68_hw.conf` | Global firmware config flags |
| `boards/shields/sick68_hw/sick68_hw.overlay` | Pin mappings (critical hardware definition) |
| `boards/shields/sick68_hw/sick68_hw.keymap` | Key layouts and layers |
| `.github/workflows/build.yml` | CI/CD pipeline trigger |

## Common Tasks

- **Add a key**: Edit `sick68_hw.keymap`, add binding to appropriate layer
- **Add a layer**: Duplicate layer block in keymap, reference with `&mo N`
- **Change power settings**: Modify `config/sick68_hw.conf`
- **Remap GPIO pins**: Edit row/col GPIO lists in `sick68_hw.overlay` (requires hardware knowledge)
- **Test locally**: `west build` after `west update`
