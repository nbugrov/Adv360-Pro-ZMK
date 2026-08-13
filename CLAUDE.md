# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

ZMK firmware **config** for the Kinesis Advantage 360 Pro — a split, wireless ergonomic keyboard. This repo contains only the keymap, board definitions, and Kconfig; the firmware itself is built by pulling in an external ZMK source tree via `west`. It does not build against upstream ZMK but against a Kinesis/ReFil fork pinned in `config/west.yml` (`remote: refil`, `revision: adv360-z3.5-2`), which adds Advantage 360 Pro-specific features (indicator RGB LEDs, etc.).

## Building

Two paths produce the same `left.uf2` / `right.uf2` artifacts:

- **GitHub Actions** — any push, PR, or manual dispatch triggers `.github/workflows/build.yml`. It builds two variants: "Legacy" (`firmware-no-clique`) and "Clique" (`firmware-clique`, built with `-DCONFIG_ZMK_STUDIO=y`). Download the artifact from the run.
- **Local (Docker/Podman)** — via `Makefile`:
  - `make` — build **both** halves
  - `make left` — build the left half only (faster iteration)
  - `make clean_firmware` — delete `firmware/*.uf2`
  - `make clean_image` — remove the built docker image
  - `make clean` — both of the above
  Output lands in `firmware/`, named `<timestamp>-<commit>-<side>-clique.uf2`. Podman is preferred if both podman and docker are present. On non-macOS, SELinux `:z` volume flags are added automatically.

The actual build commands live in `bin/build.sh` (run inside the container). Note the asymmetry: the **left** half is built with `-S studio-rpc-usb-uart -DCONFIG_ZMK_STUDIO=y` (ZMK Studio support); the **right** half is not. Building "both" from the Makefile is controlled by the `BUILD_RIGHT` env var passed to the container.

There is no local test suite — "does it compile" is the test. To verify a keymap change locally, run `make` (or `make left`) and confirm a `.uf2` is produced.

## Layout of the config

- `config/adv360.keymap` — **the real keymap**: all layers, the `hm` homerow-mods hold-tap behavior, and `#include`s of `macros.dtsi` + `version.dtsi`. Layers currently: Base, Kp (keypad), Fn, Mod.
- `config/adv360_left.keymap` / `config/adv360_right.keymap` — thin shims that just `#include "adv360.keymap"`. Do **not** put per-side keymap content here; edit `adv360.keymap`.
- `config/macros.dtsi` — macro behavior definitions referenced by the keymap.
- `config/keymap.json` / `config/info.json` — metadata consumed by external keymap editors (Nick Coutsos's editor, Kinesis Clique). Keep in sync with the keymap if you change layout structure.
- `config/boards/arm/adv360/` — board definition: `*.dts` / `*.dtsi` (device tree, matrix, pinctrl), `*_defconfig` (Kconfig per side), `Kconfig*`, `board.cmake`. Feature toggles like NKRO extended report (`CONFIG_ZMK_HID_KEYBOARD_EXTENDED_REPORT`), BLE battery reporting (`CONFIG_BT_BAS`), and indicator LED color (`CONFIG_ZMK_RGB_UNDERGLOW_MOD_COLOR`) live in the `_defconfig` files — settings that must apply to both sides need editing in **both** `adv360_left_defconfig` and `adv360_right_defconfig`.

## Key positions & combos

Combos and other position-dependent features need exact matrix key indices. These are documented (image + text) in `assets/key-positions.md` — consult it before writing combos.

## Versioning macro (generated, do not hand-edit)

`config/version.dtsi` is **auto-generated** by `bin/get_version_local.sh` (local builds) and `bin/get_version.sh` (CI). It encodes the build date, branch, commit hash, and clique flag into a ZMK macro (accessible on the keyboard via Mod+V). The Makefile regenerates it before building and runs `git checkout config/version.dtsi` afterward to discard the change — so it normally shows as empty/unchanged in git. Don't edit it manually and don't commit a populated version.

## Branch note

The default/working branch is `V3.0` (not `main`/`master`). Upgrading a fork from V2.0 to V3.0 can cause git conflicts and build failures — see `UPGRADE.md`; a `make clean` is often needed after such an upgrade.
