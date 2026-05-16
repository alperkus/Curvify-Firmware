# Curvify Firmware

Public distribution channel for the Curvify on-bike lean sensor firmware.

This repository hosts the built `.bin` files and a single `latest.json`
manifest. The Curvify iOS app fetches the manifest, compares the version
string against the device's `FW_VERSION` (read over BLE), and streams the
matching `.bin` to the sensor via BLE OTA.

Source code lives in the main (private) Curvify repository under
`hardware/esp32/curvify_sensor_amoled/`. Each release here corresponds
to an `fw-v<X.Y.Z>` tag in that repo.

## URLs

- **Manifest** (iOS app fetches this):
  `https://raw.githubusercontent.com/alperkus/Curvify-Firmware/main/latest.json`
- **Binary** (URL is in the manifest's `url` field):
  `https://raw.githubusercontent.com/alperkus/Curvify-Firmware/main/releases/curvify_sensor_amoled_v<X.Y.Z>.bin`

Old `.bin` files stay in `releases/` for rollback testing.

## Manifest schema

`latest.json` is a single object:

| Field             | Meaning                                                                          |
|-------------------|----------------------------------------------------------------------------------|
| `schema`          | Manifest format version. iOS app refuses unknown schemas.                        |
| `board`           | Board tag the .bin targets. Must match `BOARD_TAG` reported by the sensor.       |
| `version`         | Firmware semver (same string as `FW_VERSION` in `config.h`).                     |
| `released`        | ISO-8601 date.                                                                   |
| `url`             | Raw GitHub URL to the `.bin`. iOS streams it straight into BLE OTA.              |
| `size`            | Byte count of the `.bin`. iOS uses this for the OTA `BEGIN` packet.              |
| `sha256`          | Hex digest. iOS verifies before flashing.                                        |
| `min_app_version` | Older app builds skip incompatible firmware to avoid bricking older pairings.    |
| `min_battery_pct` | Optional. iOS refuses to start OTA below this. The firmware also enforces it server-side (status 0x04). |
| `notes`           | Short changelog shown in the update prompt.                                      |

## OTA status byte (BLE control characteristic, READ)

| Value | Meaning                                                                                |
|-------|----------------------------------------------------------------------------------------|
| `0`   | Idle                                                                                   |
| `1`   | Receiving                                                                              |
| `2`   | Success (device is rebooting into the new firmware)                                    |
| `3`   | Generic error (bad packet, partition write failed, etc.)                               |
| `4`   | Low battery — BEGIN refused. Device returned to idle. Show "charge first" to the user. |

## Release workflow (manual, until CI is set up)

1. In the main Curvify repo, bump `FW_VERSION` in `hardware/esp32/curvify_sensor_amoled/config.h`.
2. Build: `arduino-cli compile --fqbn esp32:esp32:esp32s3:PSRAM=opi,FlashSize=16M,PartitionScheme=app3M_fat9M_16MB --build-path build .`
3. Copy `build/curvify_sensor_amoled.ino.bin` → this repo's `releases/curvify_sensor_amoled_v<X.Y.Z>.bin`.
4. Compute SHA: `shasum -a 256 releases/curvify_sensor_amoled_v<X.Y.Z>.bin`
5. Update `latest.json` (version, url, size, sha256, released, notes).
6. Commit + push. Tag `v<X.Y.Z>` for archival.

The manifest is the only file the iOS app polls. Older `.bin`s stay
addressable by URL for manual rollback if a release goes bad.
