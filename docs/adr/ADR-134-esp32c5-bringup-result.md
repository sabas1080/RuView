# ADR-134 — ESP32-C5 bring-up result + 5 GHz CSI feasibility outcome

**Status:** Accepted
**Date:** 2026-05-27
**Owner:** Sabas (Electronic Cats)
**Related:** ADR-110 (C6 capabilities), spec `docs/superpowers/specs/2026-05-27-esp32c5-target-design.md`, plan `docs/superpowers/plans/2026-05-27-esp32c5-target.md`

## Context

ADR-110 added the ESP32-C6 as the first RISC-V research target. The Electronic Cats ESP32-C5 board adds dual-band Wi-Fi 6 (2.4 + 5 GHz) on top of the same Wi-Fi 6 / 802.15.4 / LP-core feature set. This ADR records the bring-up outcome on real hardware.

## Hardware under test

- Chip: ESP32-C5, silicon revision **v1.0 (eco2)**, MAC `3c:dc:75:9f:e1:c4`
- Board: Electronic Cats / placa propia, 8 MB embedded flash, no PSRAM
- Host link: native USB-Serial/JTAG (`/dev/ttyACM1`, vendor `303a:1001`)

## ESP-IDF version constraint

ESP-IDF **v5.5** is required. v5.4 was released before eco2 silicon shipped:

- v5.4 esptool's chip detection emits `WARNING: This chip doesn't appear to be a ESP32-C5 (chip magic value 0x5fd1406f)` and the stub fails verification.
- v5.4 bootloader image bakes `max chip rev = v0.99`. The ROM bootloader on eco2 hardware refuses to boot with:
  ```
  E boot_comm: Image requires chip rev <= v0.99, but chip is v1.0
  E boot: No bootable app partitions in the partition table
  ```
- v5.5 ships the eco2-aware bootloader (`Min chip rev: v1.0, Max chip rev: v1.99`) and the matching esptool/RISC-V toolchain.

Both v5.4 and v5.5 still list C5 as preview support — `idf.py --preview set-target esp32c5` is required (subsequent commands need no flag).

## API drift v5.4 → v5.5

`esp_now_send_cb_t` signature changed:
- v5.4: `void (*)(const uint8_t *mac, esp_now_send_status_t)`
- v5.5: `void (*)(const wifi_tx_info_t *info, esp_now_send_status_t)`

Resolved in `c6_sync_espnow.c` with a `#if ESP_IDF_VERSION >= ESP_IDF_VERSION_VAL(5, 5, 0)` guard so both versions compile.

## Wi-Fi STA association on v5.5

The default `WIFI_AUTH_WPA2_PSK` threshold path silently rejects every AP with `STA_DISCONNECTED reason=211 (NO_AP_FOUND_IN_RSSI_THRESHOLD)` on v5.5 because:
1. `wifi_sta_config_t.threshold.rssi` defaults to **0 dBm** under designated-initializer construction. STAs only see APs above this, which is never.
2. Modern AP setups expect PMF and SAE/H2E even on WPA2-PSK.

C5-only fix (gated by `CONFIG_IDF_TARGET_ESP32C5`, no impact on S3/C6):
```c
wifi_config.sta.threshold.authmode = WIFI_AUTH_WPA2_PSK;
wifi_config.sta.threshold.rssi     = -127;
wifi_config.sta.sae_pwe_h2e        = WPA3_SAE_PWE_BOTH;
wifi_config.sta.pmf_cfg.capable    = true;
wifi_config.sta.pmf_cfg.required   = false;
```

Verified: with the patch, the node associates with a Wi-Fi 6 AP (Electronic Cats `EC_HQ`) in ~7.5 s and reaches `Got IP: 192.168.0.116`.

## 5 GHz CSI feasibility — Outcome A (works natively)

The probe Kconfig `CONFIG_C5_5GHZ_CSI_EXPERIMENTAL` exists to force a UNII-1 channel after promiscuous mode is enabled and observe whether the driver delivers `wifi_csi_info_t`. In practice, on hardware:

- EC_HQ is on **5 GHz channel 157** (5785 MHz, UNII-2e).
- The firmware's auto-detect of the connected AP's channel (`esp_wifi_sta_get_ap_info`) returned `157`.
- `wifi_csi_rx_cb_t` fires immediately after CSI registration with `ch=157`, `len=106` bytes, RSSI between -27 and -79 dBm.

This is **outcome A** without ever toggling the experimental flag: the v5.5 C5 driver delivers CSI in the 5 GHz band natively. The probe block remains in the codebase as a forced-channel diagnostic for sites where the connected AP is 2.4 GHz but the operator wants 5 GHz sensing.

## ADR-110 features active on C5

The following compile-time features inherited from C6 are confirmed running on C5 via the gate-widening done in the bring-up commits:

- `c6_timesync`: 802.15.4 leader/follower election, ~10 Hz TS_BEACON, mesh epoch propagation. Observed: `c6_ts: tx#N is_leader=1` from boot, zero failures over a 70 s observation window.
- `c6_lp_core`: `ulp_embed_binary` links cleanly under v5.5 RISC-V LP toolchain.
- `c6_twt`: Wi-Fi 6 iTWT setup attempted post-connect. EC_HQ replies `Connected AP does not support setup individual TWT agreement` — graceful, expected.
- `c6_softap_he`: not enabled in the C5 overlay (matches C6 default).

## Decision

Adopt ESP32-C5 (eco2) as a supported target of `firmware/esp32-csi-node`, pinned to **ESP-IDF v5.5** in CI, with the SAE/PMF/RSSI patch and the `c6_sync_espnow` `on_send` callback shim. The `CONFIG_C5_5GHZ_CSI_EXPERIMENTAL` knob stays as a diagnostic for forced-channel cases.

## Out of scope (follow-ups)

- ADR-110 v2: rename `c6_*` files/symbols to capability-named once a third HE-capable target lands.
- ESP-IDF bump path for S3/C6 from v5.4 to v5.5 (currently kept on v5.4 for stability).
- Remove `continue-on-error` from the `c5-8mb` CI job after a green run on the v5.5 image.
- Flash baud-rate ceiling on USB-Serial/JTAG: 460800 fails ("No serial data received"); 115200 works. Investigate whether a v5.5 esptool upgrade unlocks higher rates on eco2.
- main.c `target_name` and `led_gpio` are still gated on `CONFIG_IDF_TARGET_ESP32C6`; C5 currently falls through to `"ESP32"` and an invalid `led_gpio=38`. Cosmetic only — the LED init fails silently and the log target string is generic. Worth an `#elif defined(ESP32C5)` branch once the C5 board's actual LED GPIO is confirmed.
