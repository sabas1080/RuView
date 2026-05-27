# ESP32-C5 target for `esp32-csi-node` firmware

**Date:** 2026-05-27
**Status:** Design draft — pending user review
**Owner:** Sabas (Electronic Cats)
**Related ADRs:** ADR-110 (C6 capabilities), ADR-018 (CSI node firmware), ADR-045 (display variant — partition origin)

## 1. Goal

Add **ESP32-C5** as a first-class build target of the `esp32-csi-node` firmware, with two outcomes:

1. **Production parity with C6** on 2.4 GHz — CSI capture, streamer, ESP-NOW sync, 802.15.4 timesync, LP-core wake-on-motion, Wi-Fi 6 TWT, SoftAP HE. All existing behaviour reused unchanged.
2. **Feasibility probe for 5 GHz CSI** — a gated experimental flag that switches capture to UNII-1 channels and reports whether `wifi_csi_rx_cb_t` actually fires in the 5 GHz band on ESP-IDF v5.4. This is the unique differentiator of C5 vs S3/C6 (both 2.4 GHz only).

Non-goals (deferred): rename of `c6_*` symbols to capability names, antenna-switch GPIO, full 5 GHz mesh, OTA cross-target images, display variant on C5.

## 2. Hardware

- **Chip:** ESP32-C5 (single-core RISC-V @ 240 MHz, ~400 KB SRAM, dual-band Wi-Fi 6 2.4 + 5 GHz, BLE 5, 802.15.4, LP-core).
- **Board:** Electronic Cats / placa propia.
- **Flash:** 8 MB embedded.
- **PSRAM:** none.
- **Host link:** USB-Serial/JTAG nativo del C5 (GPIO12/13). No external USB-UART chip.

## 3. Toolchain

- **ESP-IDF v5.4** (matches `espressif/idf:v5.4` used by CI in `.github/workflows/firmware-ci.yml:44` and per `firmware/esp32-csi-node/README.md`). C5 support in v5.4 is **preview** — sufficient for bring-up; 5 GHz CSI may not be plumbed yet.

## 4. Design — Approach B (Minimal expansion)

Three other approaches were considered and rejected for this iteration:

- **A (capability refactor):** rename `c6_*.{c,h}` → capability-named files (`wifi_he_twt.c`, `lp_core_motion.c`, ...) and migrate every gate from `IDF_TARGET_*` to `SOC_*`. Cleanest long-term, but a 10-file diff with risk to the working C6 build. Deferred as ADR-110 v2.
- **C (parallel `c5_*` mirrors):** clone `c6_*.{c,h}` into `c5_*.{c,h}` siblings sharing helpers. Maximum isolation, 2× maintenance. Rejected.
- **B (selected):** keep filenames and symbol prefixes as `c6_*`, widen the three external gates (CMakeLists, Kconfig menu title+depends, per-file `#if`). Smallest diff, zero risk to S3 and C6 paths.

### 4.1 New files

| Path | Purpose |
|---|---|
| `firmware/esp32-csi-node/sdkconfig.defaults.esp32c5` | Target overlay, auto-applied by `idf.py set-target esp32c5`. |

The overlay mirrors `sdkconfig.defaults.esp32c6` with these deltas:

```
CONFIG_IDF_TARGET="esp32c5"

# Reuse the S3 8 MB layout verbatim — partitions_display.csv is chip-agnostic
# CSV; the "display" in the filename is historical (ADR-045). Rename to
# partitions_8mb.csv is a follow-up.
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions_display.csv"
CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y
CONFIG_ESPTOOLPY_FLASHSIZE="8MB"

CONFIG_ESP_WIFI_CSI_ENABLED=y
CONFIG_ESP_WIFI_ENABLE_WPA3_SAE=y

CONFIG_IEEE802154_ENABLED=y
CONFIG_OPENTHREAD_ENABLED=n
CONFIG_C6_TIMESYNC_CHANNEL=26   # symbol name keeps the C6_ prefix per Approach B

CONFIG_ULP_COPROC_ENABLED=y
CONFIG_ULP_COPROC_TYPE_LP_CORE=y
CONFIG_ULP_COPROC_RESERVE_MEM=8192

CONFIG_COMPILER_OPTIMIZATION_SIZE=y
CONFIG_BOOTLOADER_LOG_LEVEL_WARN=y
CONFIG_LOG_DEFAULT_LEVEL_INFO=y

CONFIG_LWIP_SO_RCVBUF=y
CONFIG_ESP_MAIN_TASK_STACK_SIZE=8192
CONFIG_FREERTOS_TIMER_TASK_STACK_DEPTH=8192

# C5 raises the ceiling from C6's 160 MHz to 240 MHz.
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_240=y
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ=240

# 5 GHz CSI feasibility probe (Section 5). Default off.
# CONFIG_C5_5GHZ_CSI_EXPERIMENTAL is not set
```

### 4.2 Gates to widen (3 locations)

**CMakeLists.txt** (`firmware/esp32-csi-node/main/CMakeLists.txt`):

- Line 46: `if(IDF_TARGET STREQUAL "esp32c6")` → `if(IDF_TARGET STREQUAL "esp32c6" OR IDF_TARGET STREQUAL "esp32c5")`
- Line 75: same widening for the `ulp_embed_binary` block.

**Kconfig.projbuild** (`firmware/esp32-csi-node/main/Kconfig.projbuild`):

- Line 290: rename menu title from `"ESP32-C6 capabilities (ADR-110)"` → `"Wi-Fi 6 / 802.15.4 / LP-core capabilities (ADR-110)"`.
- Line 291: `depends on IDF_TARGET_ESP32C6` → `depends on IDF_TARGET_ESP32C6 || IDF_TARGET_ESP32C5`.
- Internal `config C6_*` entries unchanged (Approach B: no symbol rename).

**Per-file `#if` gates** to widen from `defined(CONFIG_IDF_TARGET_ESP32C6)` to `(defined(CONFIG_IDF_TARGET_ESP32C6) || defined(CONFIG_IDF_TARGET_ESP32C5))`:

| File | Lines |
|---|---|
| `main/c6_lp_core.c` | 26, 196 |
| `main/c6_lp_core.h` | 27 |
| `main/c6_timesync.c` | 21, 265 |
| `main/c6_timesync.h` | 29 |
| `main/c6_twt.c` | 19, 155 |
| `main/c6_twt.h` | 28, 71 |
| `main/c6_softap_he.c` | 25, 177 |
| `main/c6_softap_he.h` | 30 |
| `main/main.c` | 121, 166, 179, 204, 215, 266 |
| `main/csi_collector.c` | 226 |

Capability-based gates (`SOC_WIFI_HE_SUPPORT`, `SOC_IEEE802154_SUPPORTED`, `ULP_COPROC_TYPE_LP_CORE`) are **not touched** — they already auto-resolve correctly on C5.

### 4.3 Untouched

- `c6_*` filenames and `CONFIG_C6_*` Kconfig symbol names — explicitly kept for diff minimisation.
- ADR-110 document — a follow-up note will be added pointing to "ADR-110 v2" for the rename.
- S3, C6, mock, and QEMU builds — no functional change, byte-identical output expected.

## 5. 5 GHz CSI feasibility probe

The only C5-exclusive capability worth exercising in this iteration.

**Kconfig** (added inside the ADR-110 menu, in `Kconfig.projbuild` after the existing `C6_*` entries):

```
config C5_5GHZ_CSI_EXPERIMENTAL
    bool "Probe CSI capture on 5 GHz (C5 only, experimental)"
    default n
    depends on IDF_TARGET_ESP32C5
    help
        Switches the CSI capture channel to a UNII-1 channel
        (default 36) after promiscuous mode is enabled. Reports
        whether wifi_csi_rx_cb_t fires in the 5 GHz band on the
        currently installed ESP-IDF. Falls back to the configured
        2.4 GHz WIFI_CHANNEL if the driver returns ESP_ERR_NOT_SUPPORTED
        or ESP_ERR_INVALID_ARG.

config C5_5GHZ_PROBE_CHANNEL
    int "5 GHz probe channel"
    default 36
    range 36 48
    depends on C5_5GHZ_CSI_EXPERIMENTAL
    help
        UNII-1 channels: 36, 40, 44, 48.
```

**Runtime** (in `csi_collector.c`, in the channel-set block):

```c
#if defined(CONFIG_C5_5GHZ_CSI_EXPERIMENTAL)
    esp_err_t e = esp_wifi_set_channel(CONFIG_C5_5GHZ_PROBE_CHANNEL,
                                       WIFI_SECOND_CHAN_NONE);
    if (e == ESP_OK) {
        ESP_LOGI(TAG, "[C5 5GHz] channel %d set OK", CONFIG_C5_5GHZ_PROBE_CHANNEL);
    } else {
        ESP_LOGW(TAG, "[C5 5GHz] set_channel failed (%s); falling back to 2.4 GHz ch %d",
                 esp_err_to_name(e), CONFIG_WIFI_CHANNEL);
        esp_wifi_set_channel(CONFIG_WIFI_CHANNEL, WIFI_SECOND_CHAN_NONE);
    }
#endif
```

**Success criterion:** `wifi_csi_rx_cb_t` fires with `rx_ctrl.channel >= 36` within 5 s of subscribe. Result (works / no callbacks / set_channel ENOTSUP) is documented in a follow-up ADR (next free slot, e.g. ADR-119).

## 6. Build, flash, monitor

```bash
cd firmware/esp32-csi-node
idf.py set-target esp32c5
idf.py build
idf.py -p /dev/ttyACM0 flash monitor

# Provisioning (unchanged — uses generic esptool/serial)
python provision.py --port /dev/ttyACM0 \
  --ssid "YourWiFi" --password "secret" --target-ip 192.168.1.20
```

## 7. CI

Add a job `build-c5` to `.github/workflows/firmware-ci.yml` modeled on the existing `build-c6` job:

- `image: espressif/idf:v5.4`
- `TARGET: esp32c5`
- Initially `continue-on-error: true` until v5.4 + C5 preview build is confirmed green.

If v5.4 cannot build C5 even in preview, the job is pinned to `espressif/idf:release-v5.4` (which tracks the v5.4.x branch including C5 preview backports). If that still fails, the job is removed from the matrix and we open a discussion to bump the project to v5.5.

## 8. Manual test plan (acceptance)

1. `idf.py build` for target `esp32c5` produces a binary, no errors.
2. `idf.py flash monitor` boots; `[csi_collector] subscribed` appears; the mesh reaches `state=READY`.
3. `provision.py` succeeds; node publishes CSI frames to the streamer endpoint.
4. Existing S3 8 MB build is byte-identical to pre-change build (sanity check: nothing on the C6/S3 path moved).
5. Existing C6 build is byte-identical to pre-change build.
6. Toggle `CONFIG_C5_5GHZ_CSI_EXPERIMENTAL=y`, reflash, observe whether 5 GHz callbacks arrive. Report `works` / `no callbacks` / `ENOTSUP` in the follow-up ADR.

## 9. Risks and open questions

1. **IDF v5.4 + C5 preview.** Some C5 APIs may be absent (notably 5 GHz `esp_wifi_set_channel`). Mitigated by the runtime fallback in §5. If 2.4 GHz CSI itself does not work on v5.4 + C5, this design needs a v5.5 bump before it can ship.
2. **LP-core embed for C5.** `ulp_embed_binary` is documented for C6 but is expected to work for C5 (same LP-core architecture). If linking fails, `C6_LP_CORE_ENABLE=n` for the C5 build and move to follow-up.
3. **Unused spiffs partition.** The reused 8 MB layout includes a `spiffs` partition that the C5 firmware does not mount. Wastes 1.875 MB of flash but is otherwise inert. Boot may log `spiffs not formatted` — confirm any spiffs init in the codebase is gated by `CONFIG_DISPLAY_ENABLE`.
4. **Antenna selection.** If the Electronic Cats board exposes both U.FL and PCB antennas via a GPIO-controlled RF switch, an additional Kconfig + init step is required. Out of scope for this iteration; noted for the bring-up follow-up.
5. **Cosmetic debt.** `CONFIG_C6_*` symbols and `c6_*` filenames now also apply to C5, which is confusing. Documented as ADR-110 v2.

## 10. Out of scope (explicit follow-ups)

- ADR-110 v2: rename `c6_*` → capability-named, migrate gates to `SOC_*`.
- Rename `partitions_display.csv` → `partitions_8mb.csv` (and update S3 sdkconfig.defaults to point at the new name).
- New ADR documenting the result of the 5 GHz CSI probe.
- Antenna-switch driver, if applicable to the Electronic Cats board.
- Updating the "Supported Hardware" table in the top-level `CLAUDE.md` to list C5 once acceptance is met.
