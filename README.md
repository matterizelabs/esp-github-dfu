# esp-github-dfu

Reference ESP-IDF application for the [`matterizelabs/esp_gh_ota`](https://components.espressif.com/components/matterizelabs/esp_gh_ota) component. Connects to WiFi, starts the `/api/config` HTTP endpoint, and polls a GitHub Releases feed for firmware updates — automatically flashing a new `.bin` asset when one is detected. An onboard LED (GPIO 8, WS2812) breathes purple normally and blinks yellow during OTA.

## How it works

1. Device boots, connects to WiFi, brings up the HTTP config API on port 80.
2. A background task polls `https://api.github.com/repos/<owner>/<repo>/releases` every 5 minutes (configurable).
3. When a release newer than the running firmware is found (strict `vX.Y.Z` semver), the `.bin` asset is downloaded and flashed.
4. On next boot the new image takes over; the old image stays as fallback.
5. Runtime config (owner / repo / token) is stored in NVS and editable at `POST /api/config`.

## Features

- Partial HTTP download — firmware fetched over HTTP Range requests, survives TLS renegotiation windows.
- OTA resumption — if power is lost mid-flash, OTA resumes from last checkpoint on next boot.
- Anti-rollback — software check rejects older tags; hardware check rejects images with `secure_version` below the eFuse value (`CONFIG_BOOTLOADER_APP_ANTI_ROLLBACK`).
- Runtime configuration API — no recompile to switch repos or add a token.

## Requirements

- **ESP-IDF** v5.4+ (tested with v6.0.2)
- **ESP32-C3** (primary target; set with `idf.py set-target esp32c3`)
- **WiFi** credentials (SSID + password)
- A GitHub repo with tagged releases (`v1.0.0`, `v2.1.3`, etc.) containing `.bin` assets

## Quick start

```bash
git clone https://github.com/matterizelabs/esp-github-dfu.git
cd esp-github-dfu

# Set WiFi credentials
cp sdkconfig.secrets.example sdkconfig.secrets
# Edit sdkconfig.secrets with your SSID and password

idf.py set-target esp32c3
idf.py build
idf.py flash monitor
```

On first boot, no owner/repo is configured — the device logs a warning and waits. Set the target repo:

```bash
curl -X POST http://<esp-ip>/api/config -H 'Content-Type: application/json' \
  -d '{"owner":"matterizelabs","repo":"esp-github-dfu","token":"ghp_xxx"}'
```

The poller wakes immediately and checks for releases.

## File layout

```
esp-github-dfu/
├── CMakeLists.txt              # project root
├── main/
│   ├── CMakeLists.txt          # component registration + dependencies
│   ├── app_main.c              # startup (NVS, netif, WiFi retry, LED, OTA events)
│   └── idf_component.yml       # pulls esp_gh_ota + led_strip from registry
├── .github/workflows/
│   └── build.yml               # CI: build on tag, create GitHub Release with .bin
├── sdkconfig.defaults          # component config + partitioning
└── sdkconfig.secrets.example   # WiFi credentials template
```

## sdkconfig.defaults

| Setting | Purpose |
|---|---|
| `PARTITION_TABLE_TWO_OTA_LARGE` | Two OTA app slots (1.8 MB each), 4 MB flash |
| `ESP_GH_OTA_ENABLE_PARTIAL_DOWNLOAD` | Download via HTTP Range requests |
| `ESP_GH_OTA_ENABLE_RESUMPTION` | Survive power loss mid-OTA |
| `CONFIG_BOOTLOADER_APP_ANTI_ROLLBACK` | Reject firmware with lower `secure_version` (disabled by default) |
| `ESP_GH_OTA_BUF_SIZE=1024` | OTA buffer size |
| `ESP_GH_OTA_RECV_TIMEOUT=30000` | OTA receive timeout (ms) |

**Anti-rollback warning**: `CONFIG_BOOTLOADER_APP_ANTI_ROLLBACK` burns eFuses one-way. Disable it during development unless you manage `secure_version` per release.

## Runtime configuration

```bash
# Set owner + repo (required on first boot)
curl -X POST http://<esp-ip>/api/config \
  -H 'Content-Type: application/json' \
  -d '{"owner":"matterizelabs","repo":"esp-github-dfu"}'

# Add a token for private repos
curl -X POST http://<esp-ip>/api/config \
  -H 'Content-Type: application/json' \
  -d '{"owner":"matterizelabs","repo":"private-firmware","token":"ghp_xxx"}'

# Inspect current config + running firmware version
curl http://<esp-ip>/api/config
```

A config change wakes the poller immediately — no need to wait for the next interval.

## Release workflow

1. Push a strict `vX.Y.Z` semver tag to the firmware repo.
2. CI builds the firmware and creates a GitHub Release with the `.bin` asset attached.
3. The device polls `GET /repos/<owner>/<repo>/releases`, compares the tag version against the running firmware.
4. If newer, downloads the asset and flashes it.

Tags like `v1.0.0-rc1`, `v1.0.0-beta`, or `release` are ignored.

## Wiring into your own project

Don't want to clone this repo? Add the component to any ESP-IDF project:

**1. Declare dependency** — `main/idf_component.yml`:

```yaml
dependencies:
  matterizelabs/esp_gh_ota: "^1.0.1"
  protocol_examples_common:
    path: ${IDF_PATH}/examples/common_components/protocol_examples_common
```

**2. Register requirements** — `main/CMakeLists.txt`:

```cmake
idf_component_register(SRCS "app_main.c"
                       INCLUDE_DIRS "."
                       PRIV_REQUIRES esp_gh_ota nvs_flash esp_netif esp_event esp_wifi
                                     app_update esp_https_ota)
```

**3. Boot sequence** — `main/app_main.c`:

```c
#include "github_ota.h"

void app_main(void) {
    // ... NVS init, netif, event loop, WiFi connect ...

    ESP_ERROR_CHECK(github_config_api_start());
    github_poller_start();
}
```

## Kconfig options

Set in `sdkconfig.defaults` or via `idf.py menuconfig` under **ESP GitHub OTA**:

| Option | Default | Description |
|---|---|---|
| `ESP_GH_OTA_POLL_INTERVAL_SEC` | 300 | Seconds between GitHub API checks |
| `ESP_GH_OTA_RECV_TIMEOUT` | 30000 | Max time for firmware download (ms) |
| `ESP_GH_OTA_BUF_SIZE` | 1024 | OTA transfer buffer size |
| `ESP_GH_OTA_DEFAULT_OWNER` | (empty) | Default GitHub owner/org |
| `ESP_GH_OTA_DEFAULT_REPO` | (empty) | Default GitHub repo |
| `ESP_GH_OTA_ENABLE_PARTIAL_DOWNLOAD` | n | HTTP Range request download |
| `ESP_GH_OTA_ENABLE_RESUMPTION` | n | Resume OTA across reboots |

## Dependencies

- [`matterizelabs/esp_gh_ota`](https://components.espressif.com/components/matterizelabs/esp_gh_ota) — OTA from GitHub Releases
- `espressif/cjson` — JSON parsing (transitive via esp_gh_ota)
- `espressif/led_strip` — WS2812 LED driver (breathe/blink indicator)
- `protocol_examples_common` — WiFi connect helper (from ESP-IDF examples tree)

## License

MIT — see [LICENSE](./LICENSE) of the component.
