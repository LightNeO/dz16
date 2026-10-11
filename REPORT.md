# HW-16 - OTA update testing (Smart Lamp, station_WiFi fw_6)

## 0. Test environment

| Item | Value |
|---|---|
| Board | ESP32-S3 DevKit, 2 MB flash, WS2812 RGB LED on GPIO48 |
| Connection | USB-UART (FTDI, COM3), 115200 8N1 |
| Host | Windows 11 Pro, esptool v5.3.1, pyserial 3.5 |
| Wi-Fi | 2.4 GHz phone hotspot `CMF` (WPA3-SAE), password replaced with `***` in all logs |
| Start build | `station_WiFi_v1.3.0_merged.bin`, flashed after `erase-flash` (clean NVS) |
| OTA server | `https://raw.githubusercontent.com/BohdanHorbanych/esp32-ota-files/main/firmware/` (stable = 1.4.0, beta = 1.5.0) |
| Test basis | `fw_6/PRD.md` only. Items marked in the PRD as "not implemented yet" are not bugs |

---

## 1. Checklist by version


### 1.1 Version 1.3.0

| # | Check | PRD | Result | Evidence |
|---|---|---|---|---|
| 1 | Clean boot: `station_WiFi fw v1.3.0 ready`, `App version 1.3.0` | OTA-1 | PASS | `logs/19_v1.3.0_clean_boot.log`, `logs/11_initial_boot_1.3.0.log` |
| 2 | `version` = boot log = `ota check` (1.3.0), partition + `state: valid` | OTA-1 | PASS | `logs/19`, `logs/24` |
| 3 | Defaults on a clean device: white, 50 %, solid, lamp off | FR-L9 | PASS | `logs/19` |
| 4 | `connect`: scan list with RSSI, SSID chosen by number / by name | FR-W1 | PASS | `logs/22_v1.3.0_wifi.log`, `logs/20` |
| 5 | Password < 8 chars -> `password too short (min 8 chars)` | FR-W1 | PASS | `logs/20_v1.3.0_no_wifi_and_connect_negative.log` |
| 6 | Wrong password / missing SSID -> `connection timeout` | FR-W1 | PASS | `logs/20` |
| 7 | Empty SSID with no saved network -> `SSID cannot be empty` | FR-W1/W2 | PASS | `logs/20` |
| 8 | Credentials saved; boot log `saved credentials found but auto-connect disabled`; `connect` + Enter uses saved network | FR-W2, FR-W3 | PASS | `logs/22` |
| 9 | `status` when connected (SSID, IP, RSSI); `scan` with RSSI and channel | FR-W4 | PASS | `logs/22` |
| 10 | `disconnect` -> `status` = `WiFi: disconnected` | FR-W4 | **FAIL (BUG-05)** | `logs/23_v1.3.0_disconnect_status.log` |
| 11 | `lamp on` / `lamp off`, lamp off after reboot | FR-L1 | PASS | `logs/21_v1.3.0_lamp_suite.log`, `logs/01` |
| 12 | Named colours (7) - response and RGB in JSON; `orange`, `RED` -> `unknown color` | FR-L2 | PASS | `logs/21`, `logs/02` |
| 13 | Custom RGB incl. 0 0 0 / 255 255 255; out of range -> `rgb values must be 0-255`, colour unchanged | FR-L3 | PASS | `logs/21`, `logs/03` |
| 14 | Physical LED colour matches the requested colour | FR-L4 | **FAIL (BUG-01)** | `images/BUG-01_1..4.jpg` |
| 15 | Brightness 0 / 30 / 100 accepted | FR-L5 | PASS | `logs/21` |
| 16 | Brightness 150 / 101 / -5 / -1 -> `brightness must be 0-100`, value unchanged | FR-L5 | **FAIL (BUG-02)** | `logs/21`, `logs/05_brightness(BUG-02).log` |
| 17 | Modes solid / blink / breathe; `disco` -> `unknown mode` | FR-L6 | PASS | `logs/21`, `logs/06` |
| 18 | `lamp mode rainbow` -> `not implemented yet` (expected for 1.3.0) | FR-L6 | PASS (not implemented) | `logs/21` |
| 19 | Timer: `lamp is off` error, range errors (5000, -1, 3601), 3600 accepted, `timer 0` cancels, countdown in `timer_s`, expiry in solid/blink/breathe | FR-L7 | PASS | `logs/21`, `logs/07` |
| 20 | Timer is not kept after reboot | FR-L7 | PASS | `logs/21` |
| 21 | `lamp status` JSON format and consistency | FR-L8 | PASS | `logs/21`, `logs/08` |
| 22 | Colour/brightness/mode survive reboot | FR-L9 | PASS | `logs/21`, `logs/09_1.txt` |
| 23 | Scenes -> `scenes not implemented yet` (expected for 1.3.0) | FR-L10 | PASS (not implemented) | `logs/21`, `logs/10` |
| 24 | Unknown `lamp ...` command -> `unknown lamp command. See 'help'` | FR-L11 | PASS | `logs/21` |
| 25 | `ota check` - current / stable / beta, `update available` | OTA-2 | PASS | `logs/24`, `logs/12` |
| 26 | `ota update` 1.3.0 -> 1.4.0: progress every 10 %, `success`, reboot, self-check passed | OTA-3, OTA-5, OTA-9 | PASS | `logs/27_ota_1.3.0_to_1.4.0.log`, `logs/38`, `logs/13` |
| 27 | `ota install 1.4.0` installs the requested version; unknown version -> 404 error, device stays on 1.3.0 | OTA-4 | PASS | `logs/14_ota_install.log`, `logs/24_v1.3.0_ota_commands.log` |
| 28 | Downgrade / same-version protection | OTA-4 | N/A (introduced in 1.4.0; 1.3.0 re-installs 1.3.0 without `force`) | `logs/24` |
| 29 | Wi-Fi loss during download -> `download interrupted`, retry works without reboot | OTA-6 | **FAIL (BUG-03)** | `logs/15_wifi_lost(BUG-03).log` |
| 30 | OTA commands without Wi-Fi (never connected) -> `[OTA] failed: WiFi not connected` | OTA-7 | PASS | `logs/20`, `logs/16` |
| 31 | OTA commands after `disconnect` -> `[OTA] failed: WiFi not connected` | OTA-7 | **FAIL (BUG-06)** | `logs/23`, `logs/24` |
| 32 | Data kept after OTA 1.3.0 -> 1.4.0 (Wi-Fi, colour, brightness, mode) | OTA-8, FR-W3, FR-L9 | PASS | `logs/27` |
| 33 | `ota broken` -> self-check FAILED -> rollback to 1.3.0, `state: valid`, settings kept | OTA-10 | PASS | `logs/26_v1.3.0_ota_broken_rollback.log`, `logs/18` |
| 34 | Invalid image (`ota <url-to-stable.txt>`) -> `invalid image header`, stays on 1.3.0 | OTA-11 | PASS | `logs/24` |
| 35 | `ota <url>` with 404 / malformed URL -> controlled error | OTA-12 | PASS | `logs/24` |
| 36 | `ota-test-server` - fake log, reboot, firmware unchanged | OTA-12 | PASS | `logs/25_v1.3.0_ota_test_server.log` |


### 1.2 Version 1.4.0

| # | Check | PRD | Result | Evidence |
|---|---|---|---|---|
| 1 | Boot after OTA: `App version 1.4.0`, `PENDING_VERIFY` -> `self-check passed`, `version` = `ota check` = 1.4.0 | OTA-1, OTA-9 | PASS | `logs/27`, `logs/38` |
| 2 | `connect` + Enter (saved), `status`, `scan` with channel | FR-W1, FR-W2, FR-W4 | PASS | `logs/29_v1.4.0_wifi_ota_commands.log` |
| 3 | `disconnect` -> `status` = `WiFi: disconnected` | FR-W4 | **FAIL (BUG-05)** | `logs/29` |
| 4 | `lamp on/off`, lamp off after reboot | FR-L1 | PASS | `logs/28_v1.4.0_lamp_suite.log` |
| 5 | Named colours + invalid names | FR-L2 | PASS | `logs/28` |
| 6 | Custom RGB + out-of-range values | FR-L3 | PASS | `logs/28` |
| 7 | Physical colour matches (BUG-01 fixed) | FR-L4 | PASS (visual) | - |
| 8 | Brightness 0/30/100 accepted; 150/101/-5/-1 rejected, value unchanged (BUG-02 fixed) | FR-L5 | PASS | `logs/28` |
| 9 | Brightness restored from NVS after OTA is within 0-100 | FR-L5, FR-L9 | **FAIL (BUG-08)** | `logs/38_ota_1.3.0_to_1.4.0_invalid_brightness.log` |
| 10 | Modes solid/blink/breathe, `disco` error | FR-L6 | PASS | `logs/28` |
| 11 | `rainbow` -> `not implemented yet` (expected for 1.4.0) | FR-L6 | PASS (not implemented) | `logs/28` |
| 12 | Timer - all positive/negative cases, expiry in each mode, not kept after reboot | FR-L7 | PASS | `logs/28` |
| 13 | `lamp status` JSON | FR-L8 | PASS | `logs/28` |
| 14 | Settings survive reboot | FR-L9 | PASS | `logs/28` |
| 15 | Scenes -> `not implemented yet` (expected for 1.4.0) | FR-L10 | PASS (not implemented) | `logs/28` |
| 16 | Unknown lamp command | FR-L11 | PASS | `logs/28` |
| 17 | `ota check` -> `up to date (stable)` | OTA-2 | PASS | `logs/29`, `logs/30` |
| 18 | `ota update` on 1.4.0 -> `already up to date`, nothing downloaded | OTA-3 | PASS | `logs/29` |
| 19 | `ota install 1.3.0` -> `downgrade blocked: 1.4.0 -> 1.3.0` | OTA-4 | PASS | `logs/29` |
| 20 | `ota install 1.4.0` (same version) without `force` is blocked | OTA-4 | **FAIL (BUG-07)** | `logs/29` |
| 21 | `ota install 9.9.9` -> 404, stays on 1.4.0 | OTA-4 | PASS | `logs/30_v1.4.0_ota_misc.log` |
| 22 | `ota update beta` 1.4.0 -> 1.5.0: progress, success, self-check passed | OTA-3, OTA-5, OTA-9 | PASS | `logs/40_ota_1.4.0_to_1.5.0.log` |
| 23 | Wi-Fi loss during download + retry | OTA-6 | **FAIL (BUG-03)** | manual run, log not saved |
| 24 | OTA commands without Wi-Fi after boot -> `[OTA] failed: WiFi not connected` | OTA-7 | PASS | `logs/29` |
| 25 | OTA commands after `disconnect` | OTA-7 | **FAIL (BUG-06)** | `logs/29` |
| 26 | `ota-test-server` without Wi-Fi -> `[OTA] failed: WiFi not connected...` | OTA-7 | **FAIL (BUG-09)** | `logs/39_v1.4.0_test_server_broken.log` |
| 27 | Data kept after OTA 1.4.0 -> 1.5.0 (Wi-Fi, lamp) | OTA-8 | **FAIL (BUG-04)** | `logs/40`, `logs/30`, `logs/17` |
| 28 | Forced downgrade 1.5.0 -> 1.3.0, then `ota update` back to 1.4.0, data kept | OTA-4, OTA-8 | PASS | `logs/37`, `logs/38` |
| 29 | `ota broken` -> rollback to 1.4.0, `state: valid`, settings kept | OTA-10 | PASS | `logs/39` |
| 30 | Invalid image -> `invalid image header` | OTA-11 | PASS | `logs/30` |
| 31 | `ota-test-server` with Wi-Fi | OTA-12 | PASS | `logs/39` |

### 1.3 Version 1.5.0

| # | Check | PRD | Result | Evidence |
|---|---|---|---|---|
| 1 | Boot after OTA: `App version 1.5.0`, self-check passed, `version` = `ota check` = 1.5.0 | OTA-1, OTA-9 | PASS | `logs/40`, `logs/30` |
| 2 | Saved Wi-Fi network available after OTA | FR-W2, FR-W3, OTA-8 | **FAIL (BUG-04)** | `logs/40`, `logs/31_v1.5.0_post_ota_wifi.log` |
| 3 | `connect` by name / number, `status`, `scan` with channel, Enter uses saved network | FR-W1, FR-W2, FR-W4 | PASS | `logs/31`, `logs/37` |
| 4 | `disconnect` -> `status` = `WiFi: disconnected` | FR-W4 | **FAIL (BUG-05)** | `logs/31` |
| 5 | `lamp on/off` | FR-L1 | PASS | `logs/32_v1.5.0_lamp_suite.log` |
| 6 | Named colours, custom RGB, invalid values | FR-L2, FR-L3 | PASS | `logs/32` |
| 7 | Physical colour matches | FR-L4 | PASS (visual) | - |
| 8 | Brightness valid / invalid values | FR-L5 | PASS | `logs/32` |
| 9 | Modes solid / blink / breathe / **rainbow**; `disco` error | FR-L6 | PASS | `logs/32` |
| 10 | Timer - all cases incl. rainbow mode | FR-L7 | PASS | `logs/32` |
| 11 | `lamp status` JSON | FR-L8 | PASS | `logs/32` |
| 12 | Settings survive reboot | FR-L9 | PASS | `logs/32`, `logs/33` |
| 13 | Settings survive OTA | FR-L9, OTA-8 | **FAIL (BUG-04)** | `logs/40`, `logs/36` |
| 14 | Scenes: save/load 1-3, overwrite, values restored exactly, load while lamp off | FR-L10 | PASS | `logs/33_v1.5.0_scenes.log` |
| 15 | Scenes: `save 0/4`, `load -1` -> `scene must be 1-3`; empty scene -> `scene N is empty` | FR-L10 | PASS | `logs/32`, `logs/33` |
| 16 | Scenes survive reboot and rollback | FR-L10 | PASS | `logs/33`, `logs/35` |
| 17 | Scenes survive OTA | FR-L10, OTA-8 | **FAIL (BUG-04)** | `logs/36_v1.5.0_test_server_reinstall.log` |
| 18 | Unknown lamp command | FR-L11 | PASS | `logs/32`, `logs/33` |
| 19 | `ota check` | OTA-2 | PASS | `logs/34_v1.5.0_ota_commands.log` |
| 20 | `ota update beta` on 1.5.0 -> `already up to date` | OTA-3 | PASS | `logs/34` |
| 21 | `ota install 1.4.0` / `1.3.0` / `ota update` (stable 1.4.0) -> `downgrade blocked` | OTA-4 | PASS | `logs/34` |
| 22 | `ota install 1.5.0` (same version) without `force` is blocked | OTA-4 | **FAIL (BUG-07)** | `logs/36` |
| 23 | `ota install 1.3.0 force` -> downgrade to 1.3.0, self-check passed | OTA-4, OTA-9 | PASS | `logs/37_ota_1.5.0_to_1.3.0_force.log` |
| 24 | Data kept after forced downgrade (Wi-Fi, colour, brightness, mode) | OTA-8 | PASS | `logs/37` |
| 25 | Wi-Fi loss during download (K3 at 40 %) -> `download interrupted`, device stays on 1.5.0, settings kept | OTA-6 | PASS | `logs/42_v1.5.0_ota_wifi_loss.log` |
| 25a | After reconnect the next OTA attempt works without reboot | OTA-6 | **FAIL (BUG-03)** | `logs/42` |
| 25b | K3 works during OTA download (disconnects Wi-Fi) | FR-W5 | PASS | `logs/42` |
| 25c | K3 when not connected -> `[K3] wifi reconnect (saved credentials)`, connects | FR-W5 | PASS | `logs/43_v1.5.0_K3_disconnect_stuck.log` |
| 25d | K3 after a K3 disconnect -> reconnect | FR-W5 | **FAIL (BUG-10)** | `logs/43` |
| 25e | LED indication (lamp off, LED auto): dim green when connected, dim red when not | FR-W6 | **FAIL (BUG-10)** (stays green after disconnect) | `logs/43`, visual |
| 26 | OTA commands without Wi-Fi | OTA-7 | PASS | `logs/31` |
| 26a | OTA commands after `disconnect` | OTA-7 | **FAIL (BUG-06)** | `logs/31` |
| 27 | `ota-test-server` without Wi-Fi -> `[OTA] failed: ...` | OTA-7 | **FAIL (BUG-09)** | `logs/41_v1.5.0_misc_commands.log` |
| 28 | `ota broken` -> rollback to 1.5.0, `state: valid`, Wi-Fi + lamp + scenes kept | OTA-10 | PASS | `logs/35_v1.5.0_ota_broken_rollback.log` |
| 29 | Invalid image -> `invalid image header` | OTA-11 | PASS | `logs/34` |
| 30 | `ota-test-server` with Wi-Fi | OTA-12 | PASS | `logs/36` |
| 31 | Other commands: `help`, unknown command, `sysinfo`, `temp`, `sensor status`, `relay`, `led`, `distance` | §6 | PASS | `logs/41` |

---

## 2. OTA-8 - data persistence across OTA

| Transition | Wi-Fi before -> after | Lamp before -> after | Scenes | Result | Evidence |
|---|---|---|---|---|---|
| 1.3.0 -> 1.4.0 (`ota update`) | saved -> saved | (255,0,128) 75 % breathe -> same | N/A | PASS | `logs/27` |
| 1.3.0 -> 1.4.0 (`ota update`, 2nd run) | saved -> saved | (0,128,255) **150 %** breathe -> same, invalid value kept | N/A | FAIL (BUG-08) | `logs/38` |
| 1.4.0 -> 1.5.0 (`ota update beta`) | saved -> **lost** | (160,0,255) 35 % breathe -> **white 50 % solid** | N/A | **FAIL (BUG-04)** | `logs/40` |
| 1.4.0 -> 1.5.0 (`ota update beta`, earlier run) | saved -> **lost** | (10,200,30) 70 % breathe -> **white 50 % solid** | N/A | **FAIL (BUG-04)** | `logs/30`, `logs/31` |
| 1.5.0 -> 1.5.0 (`ota install 1.5.0`) | saved -> **lost** | (0,255,255) 5 % breathe -> **white 50 % solid** | 3 saved -> **all empty** | **FAIL (BUG-04)** | `logs/36` |
| 1.5.0 -> 1.3.0 (`ota install 1.3.0 force`) | saved -> saved | (0,128,255) 60 % rainbow -> same | N/A in 1.3.0 | PASS | `logs/37` |
| `ota broken` rollback on 1.3.0 / 1.4.0 / 1.5.0 | saved -> saved | kept | kept (1.5.0) | PASS | `logs/26`, `logs/39`, `logs/35` |

## 3. Wi-Fi loss and reconnect during OTA (OTA-6)

| Version | Scenario | Result | Evidence |
|---|---|---|---|
| 1.3.0 | Wi-Fi dropped at 40 % -> `[OTA] failed: download interrupted (ESP_FAIL)`, device stays on 1.3.0; after reconnect `ota update` -> `[OTA] error: update already in progress` | **FAIL (BUG-03)** | `logs/15_wifi_lost(BUG-03).log` |
| 1.4.0 | Same scenario, same result | **FAIL (BUG-03)** | manual run, log not saved |
| 1.5.0 | `ota install 1.5.0 force`, K3 pressed at 40 % -> `[OTA] failed: download interrupted (ESP_FAIL)`, device stays on 1.5.0, settings kept; after `connect` every OTA command (`ota install 1.5.0 force`, `ota install 9.9.9`, `ota broken`) -> `[OTA] error: update already in progress`; after `reboot` OTA works again | **FAIL (BUG-03)** | `logs/42_v1.5.0_ota_wifi_loss.log` |

---

## 4. Bug reports

### BUG-01 - Named lamp colours are shown with red and green channels swapped

- **Found in:** 1.3.0
- **Reproduces in:** 1.3.0 only (fixed in 1.4.0 and 1.5.0)
- **Severity:** Major
- **Priority:** High
- **Preconditions:** Firmware 1.3.0, UART console open.
- **Steps:**
  1. `lamp on`
  2. `lamp color red`, observe the LED
  3. `lamp color green`, observe the LED
  4. `lamp color cyan`, `lamp color purple`, observe the LED
- **Expected (PRD FR-L2, FR-L4):** The physical LED colour matches the requested colour: red is red, green is green, cyan is cyan, purple is purple.
- **Actual:** The firmware reports the correct RGB values (`color set: green (0,255,0)`), but the LED shows another colour: red -> green, green -> red, cyan -> magenta, purple -> blue. The red and green channels appear swapped (GRB order on WS2812).
- **Evidence:** `images/BUG-01_1.jpg` (purple -> blue), `images/BUG-01_2.jpg` (cyan -> magenta), `images/BUG-01_3.jpg` (green -> red), `images/BUG-01_4.jpg` (red -> green); `logs/02_named_colors.log`
- **Reproducibility:** 3/3

### BUG-02 - `lamp brightness` accepts values outside 0-100

- **Found in:** 1.3.0
- **Reproduces in:** 1.3.0 only (fixed in 1.4.0 and 1.5.0)
- **Severity:** Major
- **Priority:** High
- **Preconditions:** Firmware 1.3.0, lamp on.
- **Steps:**
  1. `lamp brightness 150`
  2. `lamp brightness -5`
  3. `lamp status`
- **Expected (PRD FR-L5):** `error: brightness must be 0-100`, brightness value does not change.
- **Actual:** `[LAMP] brightness set: 150%`, `[LAMP] brightness set: -5%`; `lamp status` shows `"brightness":-1` after `-1`. Very large numbers overflow (`-99999999999999999999` -> `-1661992959%`). The invalid value is stored in NVS (see BUG-08).
  ```
  I (373396) wifi station: [LAMP] brightness set: 150%
  I (376876) wifi station: [LAMP] brightness set: -5%
  {"lamp":"on","color":[10,20,30],"brightness":-1,"mode":"solid","timer_s":0}
  ```
- **Evidence:** `logs/21_v1.3.0_lamp_suite.log`, `logs/05_brightness(BUG-02).log`, `logs/09_1.txt` (brightness 99999 kept after reboot)
- **Reproducibility:** 3/3

### BUG-03 - OTA cannot be retried after Wi-Fi was lost during download: `update already in progress`

- **Found in:** 1.3.0
- **Reproduces in:** 1.3.0, 1.4.0, 1.5.0
- **Severity:** Critical
- **Priority:** High
- **Preconditions:** Wi-Fi connected, firmware 1.3.0.
- **Steps:**
  1. `ota install 1.4.0` (or `ota update`)
  2. At ~40 % progress switch off Wi-Fi (K3 or hotspot off)
  3. Wait for `[OTA] failed: download interrupted`
  4. Restore Wi-Fi, wait for `got ip`
  5. `ota update`
- **Expected (PRD OTA-6):** The device stays on the current version, and the next OTA attempt after Wi-Fi is back works immediately, without reboot.
- **Actual:** Step 3 is handled correctly (`download interrupted (ESP_FAIL)`, device stays on the current version, settings kept), but every new OTA command (`ota update`, `ota install <ver>`, `ota broken`) returns `[OTA] error: update already in progress`. OTA works again only after a reboot.
  ```
  E (139654) wifi station: [OTA] failed: download interrupted (ESP_FAIL)
  ...
  I (210834) wifi station: [OTA] stable channel: 1.4.0, current: 1.3.0
  E (210834) wifi station: [OTA] error: update already in progress
  ```
  1.5.0:
  ```
  I (197636) wifi station: [K3] wifi disconnect
  E (197716) wifi station: [OTA] failed: download interrupted (ESP_FAIL)
  I (219866) wifi station: successfully connected to SSID:CMF
  E (230406) wifi station: [OTA] error: update already in progress
  E (268886) wifi station: [OTA] error: update already in progress   <- ota install 9.9.9
  E (273136) wifi station: [OTA] error: update already in progress   <- ota broken
  ```
- **Evidence:** `logs/15_wifi_lost(BUG-03).log` (1.3.0), `logs/42_v1.5.0_ota_wifi_loss.log` (1.5.0)
- **Reproducibility:** 1.3.0: 1/1 (`logs/15`), 1.4.0: 1/1 (log not saved), 1.5.0: 1/1 (`logs/42`)

### BUG-04 - Installing 1.5.0 wipes all settings: saved Wi-Fi, lamp settings and scenes

- **Found in:** 1.5.0
- **Reproduces in:** 1.5.0 (every OTA install of 1.5.0: from 1.4.0 and re-install of 1.5.0). Not present in 1.3.0 / 1.4.0.
- **Regression:** Yes - OTA-8 passes on 1.3.0 -> 1.4.0.
- **Severity:** Critical
- **Priority:** Highest (blocks promotion to stable)
- **Preconditions:** Firmware 1.4.0, Wi-Fi `CMF` saved and connected, lamp set to purple / 35 % / breathe.
- **Steps:**
  1. `lamp status` -> `{"lamp":"on","color":[160,0,255],"brightness":35,"mode":"breathe",...}`
  2. `ota update beta`, wait for `self-check passed`
  3. `lamp status`
  4. `connect`, press Enter
- **Expected (PRD OTA-8, FR-W3, FR-L9, FR-L10):** Saved Wi-Fi network, lamp colour / brightness / mode and scenes are kept after any successful OTA update; no re-configuration needed.
- **Actual:** On first boot 1.5.0 runs `[CFG] migrating settings to v2 format...` and all data is lost: no `saved credentials found` in the boot log, `connect` has no "Enter to use saved" option (`SSID cannot be empty`), lamp is reset to defaults, all scenes are empty after re-install of 1.5.0. The migration runs again on each new flash of 1.5.0.
  ```
  I (504) wifi station: [CFG] migrating settings to v2 format...
  I (644) wifi station: [CFG] migration done
  I (884) wifi station: [OTA] self-check passed, firmware marked VALID
  {"lamp":"off","color":[255,255,255],"brightness":50,"mode":"solid","timer_s":0}
  Enter SSID or number from list:
  I (25214) wifi station: SSID cannot be empty
  E (22365) wifi station: error: scene 1 is empty      <- after ota install 1.5.0
  ```
- **Evidence:** `logs/40_ota_1.4.0_to_1.5.0.log`, `logs/30_v1.4.0_ota_misc.log` + `logs/31_v1.5.0_post_ota_wifi.log`, `logs/36_v1.5.0_test_server_reinstall.log`, `logs/17_persistence.log`
- **Reproducibility:** 4/4 (`logs/17`, `logs/30`, `logs/36`, `logs/40`)

### BUG-05 - After `disconnect`, `status` reports `WiFi: connected` with empty SSID and IP 0.0.0.0

- **Found in:** 1.3.0
- **Reproduces in:** 1.3.0, 1.4.0, 1.5.0
- **Severity:** Major
- **Priority:** Medium
- **Preconditions:** Wi-Fi connected (`status` shows SSID `CMF`, IP).
- **Steps:**
  1. `disconnect`
  2. `status` (also after 10 s)
- **Expected (PRD FR-W4):** `status` -> `WiFi: disconnected`.
- **Actual:** `status` keeps reporting a connected state with empty data. Also `disconnect` prints a misleading `connect to the AP fail`.
  ```
  I (31340) wifi station: disconnected
  I (31340) wifi station: connect to the AP fail
  I (34040) wifi station: WiFi: connected
  I (34040) wifi station:   SSID:
  I (34040) wifi station:   IP:   0.0.0.0
  I (34040) wifi station:   RSSI: 0 dBm
  ```
- **Evidence:** `logs/23_v1.3.0_disconnect_status.log`, `logs/29_v1.4.0_wifi_ota_commands.log`, `logs/31_v1.5.0_post_ota_wifi.log`, `logs/42_v1.5.0_ota_wifi_loss.log`, `logs/43_v1.5.0_K3_disconnect_stuck.log`
- **Reproducibility:** 3/3 on 1.3.0, 1/1 on 1.4.0, 3/3 on 1.5.0

### BUG-06 - After `disconnect`, OTA commands try to download instead of `[OTA] failed: WiFi not connected`

- **Found in:** 1.3.0
- **Reproduces in:** 1.3.0, 1.4.0, 1.5.0
- **Severity:** Minor
- **Priority:** Medium
- **Preconditions:** Wi-Fi connected, then `disconnect`.
- **Steps:**
  1. `disconnect`
  2. `ota check`, `ota update`, `ota install 1.4.0`, `ota broken`
- **Expected (PRD OTA-7):** Every OTA command -> `[OTA] failed: WiFi not connected. Use 'connect' first.`
- **Actual:** The Wi-Fi check is skipped (same wrong "connected" state as BUG-05); the device starts HTTPS requests and fails with DNS/network errors.
  ```
  E (129130) esp-tls: couldn't get hostname for :raw.githubusercontent.com: getaddrinfo() returns 202
  E (129140) wifi station: [OTA] check failed: cannot read stable.txt
  E (136280) wifi station: [OTA] failed: cannot start download (ESP_ERR_HTTP_CONNECT)
  ```
- **Evidence:** `logs/24_v1.3.0_ota_commands.log`, `logs/23`, `logs/29`, `logs/31`
- **Reproducibility:** 3/3 (one per version, plus repeated on 1.3.0)

### BUG-07 - `ota install <current version>` without `force` re-installs the firmware

- **Found in:** 1.4.0
- **Reproduces in:** 1.4.0, 1.5.0 (1.3.0 - N/A, protection was introduced in 1.4.0)
- **Severity:** Minor (on 1.5.0 it triggers BUG-04 -> user data loss)
- **Priority:** Medium
- **Preconditions:** Firmware 1.4.0, Wi-Fi connected.
- **Steps:**
  1. `ota install 1.4.0`
- **Expected (PRD OTA-4):** Re-installing the same version is allowed only with `force` (`ota install <ver> force` - "allows downgrade and re-installing the same version"); without `force` the firmware is not changed.
- **Actual:** The device only prints a warning and flashes the same version.
  ```
  I (101770) wifi station: [OTA] new version: 1.4.0
  W (101770) wifi station: [OTA] new version is the same as current - flashing anyway
  ```
- **Evidence:** `logs/29_v1.4.0_wifi_ota_commands.log`, `logs/36_v1.5.0_test_server_reinstall.log`
- **Reproducibility:** 2/2 (1.4.0, 1.5.0)

### BUG-08 - 1.4.0 restores an out-of-range brightness from NVS after update from 1.3.0

- **Found in:** 1.4.0
- **Reproduces in:** 1.4.0 (on 1.5.0 not observable because BUG-04 resets all settings)
- **Severity:** Minor
- **Priority:** Low
- **Preconditions:** Firmware 1.3.0 with BUG-02, Wi-Fi saved.
- **Steps:**
  1. On 1.3.0: `lamp brightness 150` (accepted because of BUG-02)
  2. `ota update` -> 1.4.0
  3. `lamp status`, `lamp on`
- **Expected (PRD FR-L5, FR-L9):** Brightness is always in 0-100 %; the fixed firmware should not load or use an invalid value (clamp or fall back to default).
- **Actual:** 1.4.0 loads and uses `brightness: 150`. The value is fixed only after the user sets a valid brightness manually.
  ```
  {"lamp":"off","color":[0,128,255],"brightness":150,"mode":"breathe","timer_s":0}
  I (14063) wifi station: [LAMP] on
  {"lamp":"on","color":[0,128,255],"brightness":150,"mode":"breathe","timer_s":0}
  ```
- **Evidence:** `logs/38_ota_1.3.0_to_1.4.0_invalid_brightness.log`
- **Reproducibility:** 1/1

### BUG-09 - `ota-test-server` without Wi-Fi prints a wrong error format

- **Found in:** 1.4.0
- **Reproduces in:** 1.4.0, 1.5.0 (1.3.0 not checked)
- **Severity:** Trivial
- **Priority:** Low
- **Preconditions:** Wi-Fi not connected (after boot).
- **Steps:**
  1. `ota-test-server`
- **Expected (PRD OTA-7):** `[OTA] failed: WiFi not connected. Use 'connect' first.`
- **Actual:** `OTA failed: WiFi not connected. Use 'connect' first.` - the `[OTA]` tag is missing, unlike all other OTA commands.
- **Evidence:** `logs/39_v1.4.0_test_server_broken.log`, `logs/41_v1.5.0_misc_commands.log`
- **Reproducibility:** 2/2

### BUG-10 - After K3 disconnects Wi-Fi, the next K3 press disconnects again instead of reconnecting; LED stays green

- **Found in:** 1.5.0
- **Reproduces in:** 1.5.0 (1.3.0 / 1.4.0 not checked)
- **Severity:** Major
- **Priority:** Medium
- **Preconditions:** Wi-Fi network saved, lamp off, LED in auto mode, Wi-Fi not connected.
- **Steps:**
  1. Press K3 -> `[K3] wifi reconnect (saved credentials)`, `got ip` (LED dim green)
  2. Press K3 -> `[K3] wifi disconnect`
  3. Press K3 again
  4. `status`
- **Expected (PRD FR-W5, FR-W6):** Step 2 disconnects and the LED turns dim red; step 3 reconnects to the saved network (`[K3] wifi reconnect (saved credentials)`); `status` matches the real state.
- **Actual:** Step 3 prints `[K3] wifi disconnect` again, the device stays disconnected and K3 can no longer reconnect (only `connect` from the console helps). The LED stays dim green, and `status` reports `WiFi: connected` with empty SSID. Probably the same stale "connected" flag as BUG-05.
  ```
  I (46186) wifi station: [K3] wifi disconnect
  I (46196) wifi station: disconnected
  I (55396) wifi station: button 3 pressed (GPIO39)
  I (55396) wifi station: [K3] wifi disconnect        <- expected: wifi reconnect
  I (132356) wifi station: WiFi: connected
  I (132356) wifi station:   SSID:
  I (132356) wifi station:   IP:   0.0.0.0
  ```
- **Evidence:** `logs/43_v1.5.0_K3_disconnect_stuck.log`; LED colour observed visually
- **Reproducibility:** 2/2 (second and third press in the same run)

### Observations (not bugs according to PRD)

- 1.3.0 has no downgrade / same-version protection (`ota install 1.3.0` re-installs). Expected: protection is listed as a 1.4.0 feature (PRD §1).
- After a forced downgrade 1.5.0 -> 1.3.0 the stored mode `rainbow` is kept and shown by `lamp status` in 1.3.0, where this mode is not supported. Not described in PRD; LED behaviour was not checked for this case.
- `lamp color 10 20 30 40` is accepted as `(10,20,30)` - the extra argument is ignored.
- `ota install` without a version and `ota foo` are treated as URLs (`downloading: install`) and end with `Error parse url`. Controlled error, no crash.
- On 1.5.0 `ota update` tries stable 1.4.0 and ends with `downgrade blocked` instead of `already up to date`. PRD does not define this case.

---

## 5. Bug matrix

| Bug | Title | Severity | 1.3.0 | 1.4.0 | 1.5.0 | Regression |
|---|---|---|---|---|---|---|
| BUG-01 | R/G channels swapped on LED | Major | Yes | Fixed | Fixed | No |
| BUG-02 | Brightness outside 0-100 accepted | Major | Yes | Fixed | Fixed | No |
| BUG-03 | OTA retry after Wi-Fi loss -> `update already in progress` | Critical | Yes | Yes | Yes | No (never fixed) |
| BUG-04 | 1.5.0 install wipes Wi-Fi, lamp settings, scenes | Critical | No | No | Yes | **Yes, new in 1.5.0** |
| BUG-05 | `status` after `disconnect` = connected, empty SSID | Major | Yes | Yes | Yes | No (never fixed) |
| BUG-06 | OTA after `disconnect` skips OTA-7 check | Minor | Yes | Yes | Yes | No (never fixed) |
| BUG-07 | Same-version install without `force` | Minor | N/A | Yes | Yes | No (feature added in 1.4.0) |
| BUG-08 | Invalid brightness from NVS used by 1.4.0 | Minor | N/A (root cause BUG-02) | Yes | N/A (masked by BUG-04) | No |
| BUG-09 | `ota-test-server` error text without `[OTA]` | Trivial | Not checked | Yes | Yes | No |
| BUG-10 | K3 cannot reconnect after K3 disconnect, LED stays green | Major | Not checked | Not checked | Yes | Unknown |

---

## 6. Conclusion

**1.5.0 must not be promoted from beta to stable.**

- **BUG-04 (Critical, regression):** every update to 1.5.0 erases the saved Wi-Fi network, lamp settings and scenes. Every stable user on 1.4.0 would lose the Wi-Fi configuration and have to set up the device again. This breaks OTA-8, the main OTA requirement.
- **BUG-03 (Critical):** after a Wi-Fi loss during download, no OTA command works until the device is rebooted. It is present in all three versions, including 1.5.0, so OTA-6 is not met.
- The new 1.5.0 features work: scenes (FR-L10) and `rainbow` (FR-L6). Lamp behaviour, downgrade protection, forced downgrade and rollback (OTA-4, OTA-10) also pass, and the 1.3.0 bugs BUG-01 / BUG-02 stay fixed.
- Remaining Wi-Fi state bugs (BUG-05 / BUG-06 / BUG-10) and BUG-07 are not blockers, but should be fixed. In 1.5.0, BUG-07 also lets a user wipe their own settings by re-installing the same version.

Re-test needed after fixes: OTA-8 for 1.4.0 -> 1.5.0 and 1.5.0 re-install, OTA-6 on all versions, FR-W4 / FR-W5 / FR-W6 / OTA-7 after `disconnect` (console and K3).
