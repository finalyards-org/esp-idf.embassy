# Components for ESP-IDF 6.1

All experiments carried out against `esp32c6` target, ESP-IDF `v6.1.1`, `esp-idf-sys` 0.38.1, `esp-idf-svc` 0.53.0, `esp-idf-hal` 0.47.0.

>✅ **Status: verified by build.** Runs on a device.


## TL;DR

| | All (default) | `ESP_IDF_COMPONENTS` | Change |
|---|---:|---:|---:|
| Components built | 144 | **80** | −44 % |
| Translation units (app) | 956 | **598** | −37 % |
| Compiler-seconds (app) | 63.3 s | **34.6 s** | −45 % |
| ESP-IDF build, wall (compiler busy) | 21.2 s | **12.8 s** | −40 % |
| `cargo build --release`, wall (from clean) | 50.0 s | **36.8 s** | −26 % |
| Binary (`cargo size`: text / data) | 393538 / 8198 | 393538 / 8198 | **none** |

- Trimming components **does not change the binary** at all: same symbols, same sizes. The linker already discards everything unused.
- The benefit is **build time only**.


## How `ESP_IDF_COMPONENTS` works

`esp-idf-sys`'s CMake project (`resources/cmake_project/CMakeLists.txt`) does:

```cmake
set(COMPONENTS $ENV{ESP_IDF_COMPONENTS})
```

ESP-IDF then builds three sets of components:

- the ones listed
- their `REQUIRES` / `PRIV_REQUIRES`, transitively
- the always-on `common_component_reqs`:

	```
	cxx esp_libc freertos esp_hw_support heap log soc hal esp_rom esp_common esp_system esp_stdio riscv
	```

>❗️The separator is **`;`**, not `,`. See `esp-idf-sys` `build/config.rs`: `s.split(';')`, with no trimming of whitespace. In a multi-line TOML string, end each line with `;\`.


## The list

Only the *roots* need to be listed; everything else follows as dependencies.

The first 12 roots form the **minimum** that builds, with 80 components. The figures in the TL;DR above are for this minimum.

`lwip` and `nvs_flash` are added on top, making a **"Rust `std` profile"** with 83 components. This is the set used on the `skinned` branch:

- `std::net` (sockets via `lwip`) works.
- NVS works.
- Wi-Fi, `esp_netif`, `esp_event`, HTTP/TLS etc. stay out.

```toml
ESP_IDF_COMPONENTS = """\
    main;\
    driver;\
    esp_adc;\
    esp_driver_i2c;\
    esp_driver_i2s;\
    esp_driver_ledc;\
    esp_driver_rmt;\
    esp_driver_twai;\
    esp_driver_gptimer;\
    esp_driver_pcnt;\
    fatfs;\
    spiffs;\
    lwip;\
    nvs_flash\
"""
```

| Profile | Components | Binary text / data |
|---|---:|---:|
| Minimum (12 roots) | 80 | 393538 / 8198 |
| **Rust `std` profile** (+`lwip`, `nvs_flash`) | **83** (+`lwip`, `nvs_flash`, `nvs_sec_provider`) | 394466 / 8202 |

- The +928 B comes mostly from a larger `esp_err_msg_table`, now including the `lwip` and NVS error names, and from C++ runtime init code (`eh_globals`; `nvs_flash` is C++).
- No `lwip` code is linked unless sockets are actually used.
- Build time for this profile was not measured. It adds 3 components to the 80.

Implied by the above (no need to list): `pthread`, `vfs`, `esp_timer`, `esp_driver_gpio`, `esp_driver_uart`, `esp_driver_spi`, `esp_driver_usb_serial_jtag`, `esp_driver_sdmmc`, `esp_driver_sdspi`, `sdmmc`, `wear_levelling`, `mbedtls`, …

### Why each root is needed

Each root was found by trial: build, read the error, add the component.

| Root | Required by | Failure without it |
|---|---|---|
| `main` | `esp-idf-sys` CMake project | Linking the throw-away `libespidf.elf` fails: `undefined reference to 'app_main'`. The (empty) `app_main` lives in `main.c`. |
| `driver` | `esp-idf-hal` `can` | bindgen: `'driver/twai.h' file not found`. The legacy TWAI header lives in the legacy `driver` component in 6.x. |
| `esp_adc` | `esp-idf-hal` `adc` | `adc_atten_t`, `adc_unit_t`, … not found |
| `esp_driver_i2c` / `_i2s` / `_ledc` / `_rmt` / `_twai` / `_gptimer` / `_pcnt` | `esp-idf-hal` `i2c`, `i2s`, `ledc`, `rmt`, `can`, `timer`, `pcnt` | Hundreds of missing-symbol errors in exactly those modules (tested by dropping all seven at once) |
| `fatfs`, `spiffs` | `esp-idf-svc` `io::vfs`, `partition` | `esp_vfs_fat_register`, `esp_vfs_spiffs_register`, `wl_mount`, `FATFS`, … not found |


## What the Rust crates require

### `esp-idf-sys` 0.38.1 ✅

- `bindings.h` guards optional headers by `#ifdef ESP_IDF_COMP_<X>_ENABLED`.
- Only `main` is needed, for the CMake project's own link step.

### `esp-idf-svc` 0.53 🟨

- Most optional modules are gated by `cfg(esp_idf_comp_*_enabled)`: Wi-Fi, netif, HTTP, NVS, MQTT, …
- **Exceptions** that are gated only by `vfs` / `esp_partition`, not by the component they actually use:
	- `io::vfs` SPIFFS mount → needs `spiffs`
	- `io::vfs` FAT mount → needs `fatfs`
	- `partition::EspWlPartition` → needs `wear_levelling` (pulled in by `fatfs`)

>Upstream suggestion: gate these by `esp_idf_comp_spiffs_enabled`, `esp_idf_comp_fatfs_enabled` and `esp_idf_comp_wear_levelling_enabled`.

### `esp-idf-hal` 0.47 🟨

- Driver modules (`adc`, `can`, `i2c`, `i2s`, `ledc`, `rmt`, `pcnt`, `timer`, …) are compiled **unconditionally**. They are gated only by SoC capabilities (`esp_idf_soc_*_supported`), not by component presence.
- `can` still uses the legacy `driver/twai.h`, which drags in the legacy `driver` component.

>Upstream suggestion: gate driver modules by `esp_idf_comp_esp_driver_*_enabled`, as `esp-idf-svc` does. That would let an app drop the drivers it doesn't use.

### Rust `std`

- Needs `pthread` and `vfs`. Both come in transitively (via `esp_driver_*`, `fatfs`, …).


## What got dropped

64 components in the minimum (61 in the `std` profile, which keeps `lwip`, `nvs_flash` and `nvs_sec_provider`), among them:

```
esp_wifi wpa_supplicant esp_phy esp_coex esp_netif esp_netif_stack esp_event
esp-tls esp_http_client esp_http_server esp_https_ota esp_https_server tcp_transport http_parser
protocomm esp_local_ctrl protobuf-c console
esp_lcd esp_driver_mcpwm esp_driver_sdm esp_driver_tsens esp_driver_ana_cmpr esp_driver_cam esp_driver_isp ...
ieee802154 openthread bt esp_hid
unity cmock app_trace espcoredump esp_gdbstub ulp
```

- Wi-Fi alone accounted for 108 translation units, 8.3 compiler-seconds and ~11 % of the ESP-IDF build wall time.


## What stays

- **`mbedtls`**: 148 translation units, 9.2 compiler-seconds, now **27 %** of the remaining app build. It is pulled in by `bootloader_support`, which is in turn pulled in by the common components.

	Q: Is `bootloader_support` → `mbedtls` needed when secure boot / flash encryption are disabled?

- **`fatfs`, `spiffs`, `wear_levelling`, `sdmmc`, and 7 drivers**: forced by `esp-idf-svc` / `esp-idf-hal`, see above. Together about 5–6 compiler-seconds.


## Trade-offs

- Neither (??) profile is suitable as-is for the `main` branch template. Apps using Wi-Fi would add e.g. `esp_wifi;esp_netif;esp_event`.


## Method

**Dependency closure.** Computed from `project_description.json`:

```
build_component_info[*].reqs + priv_reqs + common_component_reqs
```

The prediction (80 components) matched the `Built components:` line of `cargo build -vv` exactly.

**Timing.** Measured from a clean build in a separate `CARGO_TARGET_DIR`, with a CMake compiler launcher (`CMAKE_{C,CXX,ASM}_COMPILER_LAUNCHER`) that logs start and end time, exit code, working directory and output of every compiler invocation.

- The host is `linux/aarch64`, where `esp-idf-sys` uses Unix Makefiles, so there is no `.ninja_log`.
- "Compiler-seconds" is the sum of per-translation-unit durations.
- "Wall (compiler busy)" is the time at least one compiler was running.
- Lima VM (2.3.0), 4 cores. One run per configuration.

**Baseline.** "All" is the same ESP-IDF 6.1.1, built with `ESP_IDF_COMPONENTS` set empty in the environment (which overrides `.cargo/config.toml`).

**Binary comparison.** `llvm-nm -S` symbol lists of the two binaries are identical: names and sizes, excluding ROM `A` symbols.

