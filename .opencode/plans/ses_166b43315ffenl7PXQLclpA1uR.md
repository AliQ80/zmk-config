# Plan: Prospector Module — Operator Layout on Dongle with Single Peripheral

> **Status: ✅ IMPLEMENTED** (2026-06-05)

## Summary

Integrate the Prospector ZMK module (`feat/new-status-screens` branch) on a XIAO BLE dongle with Waveshare 1.69" Round LCD display (240×280, ST7789V driver), using the **Operator** status screen layout. The dongle connects to a single Reviung41 unibody peripheral (Nice!Nano v2).

Constraint: `ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=1` via Kconfig so the Operator layout shows a single battery arc ("PRPH" + percentage) instead of split keyboard dual-arcs.

**Key insight from code analysis:** No Prospector source modifications needed. The Operator layout's `battery_circles.c` already has a `PERIPHERAL_COUNT == 1` code path that renders a single battery arc with label and percentage. We only need correct Kconfig.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│  Dongle (XIAO BLE + Waveshare 1.69" Round LCD 240×280)      │
│  CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y                             │
│  CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=1                  │
│  Shield: reviung41_dongle + prospector_adapter               │
│  Layout: Operator                                            │
│                      │ BLE (Central role)                    │
└──────────────────────┼──────────────────────────────────────┘
                       │
┌──────────────────────┼──────────────────────────────────────┐
│  Reviung41 Peripheral (Nice!Nano v2)                         │
│  CONFIG_ZMK_SPLIT=y, CONFIG_ZMK_SPLIT_ROLE_CENTRAL=n         │
│  Battery reports via zmk_peripheral_battery_state_changed    │
└──────────────────────────────────────────────────────────────┘
```

### Event Flow

```
Reviung41 ──[BLE]──► central_status_changed_observer.c  ──► battery_circles.c
(peripheral)          fires {slot=0, connected=true}        source=0 < PERIPHERAL_COUNT(1)? YES
                                                            update arc value, show "PRPH" + "85%"

Reviung41 ──[BLE]──► zmk_peripheral_battery_state_changed ──► battery_circles.c
(peripheral)          {source=0, state_of_charge=85}         animate arc to 85%
```

---

## Hardware

| Device | Board | Controller | Role |
|--------|-------|-----------|------|
| Dongle | `xiao_ble//zmk` | XIAO BLE | ZMK Central + display |
| Keyboard | `nice_nano_v2` | Nice!Nano v2 | ZMK Peripheral |

---

## Files Created / Modified

### New: `boards/shields/reviung41_dongle/Kconfig.shield`

```kconfig
config SHIELD_REVIUNG41_DONGLE
    def_bool $(shields_list_contains,reviung41_dongle)
```

### New: `boards/shields/reviung41_dongle/Kconfig.defconfig`

```kconfig
if SHIELD_REVIUNG41_DONGLE

config ZMK_KEYBOARD_NAME
    default "Reviung41"

config ZMK_SPLIT
    default y

config ZMK_SPLIT_ROLE_CENTRAL
    default y

# Single unibody peripheral (ZMK default is already 1, set explicitly for clarity)
config ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS
    default 1

# 1 peripheral + 5 BT profiles = 6 total connections
config BT_MAX_CONN
    default 6

config BT_MAX_PAIRED
    default 6

endif
```

### New: `boards/shields/reviung41_dongle/reviung41_dongle.conf`

```ini
# ZMK Display enablement
CONFIG_ZMK_DISPLAY=y
CONFIG_ZMK_DISPLAY_STATUS_SCREEN_CUSTOM=y

# Display thread configuration
CONFIG_ZMK_DISPLAY_DEDICATED_THREAD_STACK_SIZE=8192

# Battery level fetching from peripherals
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y

# Proxy peripheral battery levels to BLE HID Battery Service (required by Prospector)
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_PROXY=y

# Prospector status screen - Operator layout
CONFIG_PROSPECTOR_STATUS_SCREEN_OPERATOR=y

# Brightness: manual fixed mode (no ambient light sensor)
CONFIG_PROSPECTOR_USE_AMBIENT_LIGHT_SENSOR=n
CONFIG_PROSPECTOR_FIXED_BRIGHTNESS=80

# Modifier display
CONFIG_PROSPECTOR_SHOW_MODIFIERS=y
CONFIG_PROSPECTOR_SHOW_INACTIVE_MODIFIERS=y
CONFIG_PROSPECTOR_MODIFIER_OS_MAC=y
CONFIG_PROSPECTOR_MODIFIER_ORDER="GACS"
```

### New: `boards/shields/reviung41_dongle/reviung41_dongle.overlay`

Dongle has no physical keys — uses a mock kscan with the Reviung41 matrix transform (13 columns × 4 rows). The `prospector_adapter` shield handles display initialization via its XIAO BLE pin overlay.

### New: `boards/shields/reviung41_dongle/reviung41_dongle.keymap`

Empty keymap — dongle has no physical keys. Satisfies the build system.

### Modified: `config/west.yml`

```yaml
manifest:
  defaults:
    revision: main
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: carrefinho
      url-base: https://github.com/carrefinho
  projects:
    - name: zmk
      remote: zmkfirmware
      import: app/west.yml
    - name: prospector-zmk-module
      remote: carrefinho
      revision: feat/new-status-screens
      import: false
  self:
    path: config
```

Key changes: ZMK `revision: main` (required by Prospector's Zephyr 4.1), added Prospector module with `import: false`.

### Modified: `config/reviung41.conf`

Added `CONFIG_ZMK_SPLIT=y`. Peripheral role (`CONFIG_ZMK_SPLIT_ROLE_CENTRAL=n`) is forced via cmake-args in build.yaml.

### Modified: `build.yaml`

```yaml
---
include:
  # Dongle with Prospector operator screen
  - board: xiao_ble//zmk
    shield: reviung41_dongle prospector_adapter
    artifact-name: reviung41_dongle_firmware
  # Dongle settings reset
  - board: xiao_ble//zmk
    shield: settings_reset
    artifact-name: settings_reset_dongle
  # Reviung41 as peripheral
  - board: nice_nano_v2
    shield: reviung41
    cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
    artifact-name: reviung41_keyboard_firmware
  # Keyboard settings reset
  - board: nice_nano_v2
    shield: settings_reset
    artifact-name: settings_reset_keyboard
```

---

## Kconfig Resolution Order

1. **ZMK core** (`main` branch) — `ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS` defaults to 1
2. **`reviung41_dongle/Kconfig.defconfig`** — sets `ZMK_SPLIT=y`, `ZMK_SPLIT_ROLE_CENTRAL=y`, `ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=1`
3. **Prospector module** `Kconfig.defconfig` — sets `ZMK_DISPLAY=y`, `ZMK_DISPLAY_STATUS_SCREEN_CUSTOM=y`, LVGL settings
4. **Prospector module** `Kconfig` — provides `PROSPECTOR_STATUS_SCREEN_OPERATOR` choice
5. **`reviung41_dongle.conf`** — selects Operator layout, enables battery proxy, sets brightness
6. **`build.yaml` cmake-args** — overrides `CONFIG_ZMK_SPLIT_ROLE_CENTRAL=n` for the Reviung41 peripheral build

---

## Build Outputs

| Artifact (`.uf2`) | Board | Shield(s) | Flash to |
|---|---|---|---|
| `reviung41_dongle_firmware.uf2` | xiao_ble//zmk | reviung41_dongle + prospector_adapter | Dongle XIAO BLE |
| `settings_reset_dongle.uf2` | xiao_ble//zmk | settings_reset | Dongle (before flashing) |
| `reviung41_keyboard_firmware.uf2` | nice_nano_v2 | reviung41 | Reviung41 |
| `settings_reset_keyboard.uf2` | nice_nano_v2 | settings_reset | Reviung41 (before flashing) |

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| ZMK `main` introduces changes from v0.3.0 | Test both firmware builds thoroughly |
| Prospector `central_status_changed_observer.c` vs ZMK core split handling | Totem project already combined these successfully |
| RAM overflow (`region 'RAM' overflowed`) | Add `CONFIG_LV_Z_VDB_SIZE=25` to dongle `.conf` if needed |
| Unibody battery display | `ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=1` ensures only slot 0 tracked |

---

## Flashing Procedure

1. Flash `settings_reset_dongle.uf2` to dongle
2. Flash `settings_reset_keyboard.uf2` to Reviung41
3. Flash `reviung41_dongle_firmware.uf2` to dongle
4. Flash `reviung41_keyboard_firmware.uf2` to Reviung41
5. Power on dongle first, then Reviung41 — pair automatically

## Verification Checklist

- [ ] Build succeeds for all 4 firmware targets
- [ ] Dongle display shows Operator status screen on boot
- [ ] Battery arc updates when Reviung41 connects (single "PRPH" + percentage)
- [ ] Layer name changes when switching layers on keyboard
- [ ] WPM meter registers when typing
- [ ] BLE profile selector shows connected profile
- [ ] Modifier indicators respond to Ctrl/Cmd/Alt/Shift

---

## Sources Verified

- [Prospector ZMK Module README](https://github.com/carrefinho/prospector-zmk-module/tree/feat/new-status-screens)
- [Prospector Bill of Materials](https://github.com/carrefinho/prospector) — Waveshare 1.69" Round LCD, XIAO BLE
- [ZMK Split Configuration docs](https://zmk.dev/docs/config/split) — confirms `ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS` default 1, battery proxy symbols
- [ZMK Dongle docs](https://zmk.dev/docs/hardware-integration/dongle) — confirms mock kscan, Kconfig pattern, cmake-args, settings_reset
- [Your working Totem project](https://github.com/AliQ80/zmk-totem) — proven shield structure and build pattern
