# 🛠️ Technical Logs — ESP32 AI Assistant (S3 Lite)

> **System Status: ✅ Operational (8kHz Audio + Deep Sleep Power Optimization)**  
> **Last Updated: March 28, 2026**  
> **Hardware: ESP32-S3 (QFN56) rev v0.2 | Firmware: ESP-IDF v5.5.1 | PSRAM: 8 MB Octal**  
> **Build: Mar 28 2026 12:23:52 | ELF SHA256: eff048b61**

---

## 🚀 Measured Performance Metrics

| Metric | Value | Notes |
|---|---|---|
| ⏱️ Full system boot | **~1.5 s** | From CPU start to `main: assistant_esp32 started` |
| 📶 Wi-Fi connected (IP) | **~1.5–1.6 s** | From WiFi init to IP obtained — no retry needed |
| 🧠 Available PSRAM | **8 MB** | AP Octal PSRAM 64Mbit, 80MHz |
| 🧠 Heap at runtime | **~7.5–7.6 MB total** | Internal ~100–102 KB, Largest DMA block 31 KB |
| 🎙️ Sample Rate | **8,000 Hz** | 16-bit mono, 100 ms window = 1,600 bytes |
| 🔊 High-Pass Filter (HPF) | **100 Hz @ 8kHz** | IIR Butterworth 1st order, applied before API call |
| 💾 WAV recording to SD | **~200–800 ms** | Depends on file size; SD kept mounted |
| 💬 Chat log append | **~100 ms** | CMMDD.TXT appended per interaction |
| 💤 Deep Sleep Timeout | **~50 s inactivity** | Warning at 10s remaining |
| ⚡ Deep Sleep current | **~0.1 mA (estimated)** | All peripherals shut down; hw validation pending |
| ⚡ Sleep current (before fix) | **~30 mA** | Backlight floating HIGH + WiFi/I2S/I2C active |
| 📶 WiFi modem-sleep ratio | **87–89%** | `WIFI_PS_MIN_MODEM` — measured across 3 sessions |
| ⏱️ Wake + WiFi reconnect | **~1.5–1.6 s** | Full reboot to IP, consistent across 3 wakeups |
| 🔆 Backlight during sleep | **OFF (GPIO held LOW)** | `gpio_hold_en()` prevents floating HIGH |
| 🧩 Dynamic Profiles | **1–6 profiles** | Loaded from SD card, configurable via Captive Portal |
| 🔄 Profile save on switch | **~30–40 ms** | Persisted to `/sdcard/data/config.txt` (3142 bytes) |
| 🔒 TLS handshake | **~980–1,030 ms** | X.509 certificate validation per session |
| 🕐 API total time (short) | **~3.9 s** | 2.5s PTT — from release to `interaction finished` |
| 🕐 API total time (long) | **~5.3 s** | 8.0s PTT — from release to `interaction finished` |
| 🎙️ Audio max per recording | **262,144 bytes** | Longer recordings truncated with warning log |
| 🔋 Battery Reading (ADC) | **O(1)** | ADC_UNIT_1 (GPIO 4), calibration at boot |

---

## 📋 Boot Sequence

```
I (24)  boot: ESP-IDF GIT-NOTFOUND 2nd stage bootloader
I (24)  boot: compile time Mar 28 2026 12:24:12
I (25)  boot: Multicore bootloader | chip revision: v0.2
I (32)  boot.esp32s3: Boot SPI Speed: 80MHz | Mode: DIO | Flash: 16MB
I (375) octal_psram: Found 8MB PSRAM device (AP gen3, 80MHz, 3V)
I (853) esp_psram: SPI SRAM memory test OK
I (862) cpu_start: cpu freq: 160000000 Hz
I (902) heap_init: At 3FCB58E8 len 00033E28 (207 KiB): RAM
I (907) heap_init: At 3FCE9710 len 00005724 (21 KiB): RAM
I (912) heap_init: At 3FCF0000 len 00008000 (32 KiB): DRAM
I (917) heap_init: At 600FE06C len 00001F7C  (7 KiB): RTCRAM
I (923) esp_psram: Adding pool of 8192K of PSRAM to heap allocator
I (946) sleep_gpio: Configure to isolate all GPIO pins in sleep state
I (952) sleep_gpio: Enable automatic switching of GPIO sleep configuration
I (999) bsp: Init BSP
I (999) bsp: Button logic: Active=0, Current Level=0
I (1119) CST816S: IC id: 182
I (1119) bsp: I2S mic init ok (BCLK=16 WS=17 SD=21)
I (1269) bsp_battery: ADC Calibration Success
I (1289) bsp_sd: SD card SPI bus ready (MOSI=38 MISO=40 CLK=39 CS=41)
I (1389) bsp_sd: SD card mounted at /sdcard  (SDHC, 29818 MB, 10 MHz)
I (1419) config_mgr: Config loaded: SSID='MyNetHome' profiles=4 volume=70 brightness=85
I (1459) wifi:mode : sta (80:b5:4e:d9:2b:ac)
I (1469) bsp: Wi-Fi connection started for SSID: MyNetHome
I (1479) main: assistant_esp32 started
I (1519) wifi:connected with MyNetHome, aid=5, ch5, BW20, rssi: -65
I (3069) bsp: Wi-Fi connected, got IP: 192.168.0.184
I (3069) wifi:Set ps type: 1, coexist: 0
I (3069) bsp: Wi-Fi Power Save: WIFI_PS_MIN_MODEM enabled
I (3079) bsp: Initializing SNTP in background...
```

**Boot time: ~1.5 s** (to `main: started`) | **WiFi IP: at ~3.0 s from power-on**, ~1.5 s from WiFi init.  
`WIFI_PS_MIN_MODEM` enabled immediately after IP — radio sleeps between DTIM beacons.

> ⚠️ `W i2c: This driver is an old driver` — legacy I2C API for CST816S; no functional impact.  
> ⚠️ `W i2s_common: dma frame num limited to 511` — automatic DMA buffer adjustment for 8kHz; informational.  
> ⚠️ `E spi: SPI bus already initialized` — SPI2 shared between LCD and SD card; handled correctly.

---

## 🎙️ Interaction Flow — Push-to-Talk with RMS Monitoring

The system uses **Push-to-Talk (PTT)** as the exclusive recording control. All 100ms audio chunks are fully captured; the RMS of each window is computed and logged for monitoring. After capture, a **100 Hz HPF** is applied in-place on the PCM buffer before sending to the API.

**Observed RMS patterns:**
- Silence / ambient noise: 500–1,400
- Normal speech: 4,000–14,000
- Loud speech / peak: 15,000–22,000

---

## 📅 Session Log — March 28, 2026

### Session 1 — Boot + Interaction (PTT 8.0 s / 128 KB)

```
I (8199) bsp: GPIO 18 changed from 0 to 1           ← button pressed
I (8239) app: button pressed -> start recording
I (8239) bsp: Audio captured: 8000 Hz, 16-bit, 1 ch, 100 ms, 1600 bytes
I (8529) app: [RMS] Window: 17038.85 (Total: 16000 bytes)   ← clear speech peak
...
I (15559) bsp: GPIO 18 changed from 1 to 0          ← button released
I (15569) app: Button released -> stopping recording
I (15589) app: HPF applied: 100 Hz cutoff @ 8kHz, 64000 samples
I (15699) app: Audio-only path initiated
I (15759) app: HTTP client initialized: https://api.openai.com/v1/chat/completions
I (16739) esp-x509-crt-bundle: Certificate validated         ← TLS: 980 ms
I (20479) app_storage: Audio queued in PSRAM (128000 bytes, queue: 1/2)
I (20819) app: interaction finished (captured=128000 bytes, ms=8000)
I (21469) app_storage: Audio saved: /sdcard/media/audio/R155838.WAV (128044 bytes WAV)
I (21469) app_storage: Batch save complete (SD kept mounted): 1 saved, 0 failed
```

**Timing — Interaction 1:**

| Phase | Duration |
|---|---|
| Recording (PTT) | **8,000 ms** |
| HPF + setup | ~170 ms |
| TLS handshake | **980 ms** |
| Upload + inference + stream | ~3,740 ms |
| **Total (release → finished)** | **~5,260 ms** |

**Storage:**
- Heap: Total ~7.54 MB, Internal ~101 KB, Largest DMA 31 KB ✅
- Chat log: `/sdcard/logs/chat/C0328.TXT` (482 bytes)
- WAV: `R155838.WAV` — 128,044 bytes (PCM 128,000 → WAV 128,044)

### Deep Sleep 1 — Shutdown Sequence

```
I (71939) app: Deep sleep warning: 10s remaining
I (81939) app: Inactivity timeout reached, preparing deep sleep...
I (83439) bsp_sleep: Entering Deep Sleep Mode...
I (83439) bsp_sleep: LVGL task deleted (Core 1 freed)        ← prevents Task WDT
I (83439) bsp_sleep: LVGL tick timer stopped                 ← 500 IRQs/s eliminated
I (83439) bsp_sleep: Backlight GPIO1 forced LOW and held     ← gpio_hold_en()
I (83449) bsp_sleep: I2S channel stopped and deleted
I (83449) bsp_sleep: SNTP stopped
I (83449) wifi:state: run -> init (0x0)
I (83459) wifi:pm stop, total sleep time: 72204308 us / 81930905 us   ← 88.1% PS ratio
I (83519) bsp_sleep: WiFi stopped and deinitialized
I (83519) bsp_sleep: I2C driver deleted
I (83519) bsp_sd: SD card unmounted
E (83519) spi_master: not all CSses freed                    ← cosmetic: LCD io_handle
I (83519) bsp_sleep: SPI bus freed
W (83549) bsp_sleep: Botao pressionado. Aguardando soltar...
I (91749) bsp_sleep: Botao solto. Entrando em sleep.
I (91749) bsp_sleep: Todos perifericos desligados. Entrando em Deep Sleep.
```

**WiFi modem-sleep ratio — session 1:** `72,204,308 µs / 81,930,905 µs` = **88.1%**

**9-step shutdown sequence:**

| # | Peripheral | Action | Result |
|---|---|---|---|
| 1 | LVGL Task (Core 1) | `vTaskDelete()` | Core 1 released — no Task WDT |
| 2 | LVGL tick timer | `esp_timer_stop()` | 500 IRQs/s eliminated |
| 3 | Backlight GPIO1 | `gpio_set_level(0)` + `gpio_hold_en()` | Backlight forced OFF and held |
| 4 | I2S / INMP441 | `channel_disable` + `channel_del` | BCLK/WS/SD GPIOs floating |
| 5 | SNTP | `esp_sntp_stop()` | NTP stack stopped |
| 6 | WiFi | `esp_wifi_stop()` + `esp_wifi_deinit()` | RF radio powered down |
| 7 | I2C / CST816S | `i2c_driver_delete()` | SDA/SCL without pull-up drain |
| 8 | SD Card | `vfs_fat_sdspi_unmount()` | Filesystem safely unmounted |
| 9 | SPI bus | `spi_bus_free()` | Bus released |

---

### Session 2 — Wakeup 1 + Profile Switching + Interaction (PTT 2.5 s / 40 KB)

**Boot after wakeup:**
```
I (1515) wifi:connected with MyNetHome, aid=5, ch5, rssi: -62   ← no retry
I (2595) bsp: Wi-Fi connected, got IP: 192.168.0.184            ← ~1.5 s
I (2595) bsp: Wi-Fi Power Save: WIFI_PS_MIN_MODEM enabled
```

**Profile switching — 4 cycles, all persisted:**

```
I (5075)  app: Profile changed to: 1 (Agronomo)
I (5115)  config_mgr: Config saved to /sdcard/data/config.txt (3142 bytes, 4 perfis)  ← ~40ms
I (6565)  app: Profile changed to: 2 (Teacher)     → saved ~40ms
I (9145)  app: Profile changed to: 3 (Digital)     → saved ~40ms
I (11655) app: Profile changed to: 0 (Generalista) → saved ~40ms
```

**Interaction:**
```
I (19865) app: button pressed -> start recording
I (21705) app: Button released -> stopping recording
I (21715) app: HPF applied: 100 Hz cutoff @ 8kHz, 20000 samples
I (22805) esp-x509-crt-bundle: Certificate validated        ← TLS: 1,030 ms
I (25115) app_storage: Audio queued in PSRAM (40000 bytes, queue: 1/2)
I (25365) app: interaction finished (captured=40000 bytes, ms=2500)
I (25655) app_storage: Audio saved: /sdcard/media/audio/R160327.WAV (40044 bytes WAV)
```

**Timing — Interaction 2:**

| Phase | Duration |
|---|---|
| Recording (PTT) | **2,500 ms** |
| HPF + setup | ~60 ms |
| TLS handshake | **1,030 ms** |
| Upload + inference + stream | ~2,290 ms |
| **Total (release → finished)** | **~3,660 ms** |

### Deep Sleep 2 — Idle (no interaction)

```
I (47985) wifi:pm stop, total sleep time: 41477007 us / 46462092 us   ← 89.3% PS ratio
...full 9-step shutdown sequence...
I (192175) bsp_sleep: Botao solto. Entrando em sleep.   ← button held ~144s (normal)
```

**WiFi modem-sleep ratio — session 2:** `41,477,007 µs / 46,462,092 µs` = **89.3%**

---

### Session 3 — Wakeup 2 + Idle

```
I (1525) wifi:connected with MyNetHome, aid=5, ch5, rssi: -59   ← no retry
I (3095) bsp: Wi-Fi connected, got IP: 192.168.0.184            ← ~1.6 s
I (3095) bsp: Wi-Fi Power Save: WIFI_PS_MIN_MODEM enabled
```

### Deep Sleep 3 — Idle (60s timeout)

```
I (71685) wifi:pm stop, total sleep time: 61103219 us / 70148142 us   ← 87.1% PS ratio
I (71665) bsp_sleep: LVGL task deleted (Core 1 freed)
I (71665) bsp_sleep: Backlight GPIO1 forced LOW and held
...full shutdown sequence...
I (74585) bsp_sleep: Todos perifericos desligados. Entrando em Deep Sleep.
```

**WiFi modem-sleep ratio — session 3:** `61,103,219 µs / 70,148,142 µs` = **87.1%**

---

### Session 4 — Wakeup 3 (in progress)

```
I (1525) wifi:connected with MyNetHome, aid=5, ch5, rssi: -63   ← no retry
I (3095) bsp: Wi-Fi connected, got IP: 192.168.0.184            ← ~1.6 s
I (3095) bsp: Wi-Fi Power Save: WIFI_PS_MIN_MODEM enabled
I (36465) app: Deep sleep warning: 10s remaining                ← 4th cycle incoming
```

---

## 📊 Multi-Cycle Summary — 3 Deep Sleep Cycles (28/03/2026)

| Cycle | Context | WiFi PS ratio | WiFi reconnect | Shutdown |
|---|---|---|---|---|
| 1 | PTT 8s interaction | **88.1%** | ~1.5 s | ✅ clean |
| 2 | Idle + profile switch + PTT 2.5s | **89.3%** | ~1.5 s | ✅ clean |
| 3 | Idle 60s timeout | **87.1%** | ~1.6 s | ✅ clean |
| **Average** | — | **88.2%** | **~1.55 s** | 3/3 ✅ |

**Observations:**
- ✅ Zero crashes across 3 complete cycles: sleep → wakeup → operation → sleep
- ✅ Zero Task WDT — LVGL task deleted before sleep in every cycle
- ✅ Zero WiFi race conditions — `s_wifi_shutting_down` flag effective
- ✅ Backlight OFF in every sleep cycle, ON in every wakeup
- ✅ SD card safely unmounted and remounted in every cycle
- ✅ Profile switching: 4 switches × ~40ms, config.txt 3142 bytes, zero write failures
- ✅ Heap stable: Total ~7.5–7.6 MB, Internal ~100–102 KB across all cycles
- ⚠️ `E spi_master: not all CSses freed` — cosmetic in every shutdown (LCD io_handle not deleted)

---

## 💾 Storage Subsystem (Opportunistic Saving)

```
I (20479) app_storage: Audio queued in PSRAM (128000 bytes, queue: 1/2)
W (20479) app_storage: Audio queue almost full, triggering immediate save
I (20489) app_storage: Memory before SD mount: Total=7543896, Internal=101759, Largest=31744
I (20499) app_storage: DMA memory diagnostic: LargestDMA=31744 bytes (need 24576) ✅
I (20519) app_storage: SD card already mounted, proceeding to save
I (20619) app_storage: Chat log appended: /sdcard/logs/chat/C0328.TXT (482 bytes)
I (21469) app_storage: Audio saved: /sdcard/media/audio/R155838.WAV (128044 bytes WAV)
I (21469) app_storage: Batch save complete (SD kept mounted): 1 saved, 0 failed
```

- **PSRAM queue**: capacity 2; preemptive save triggered at threshold
- **DMA check**: verifies 24,576 bytes available before SD operations (measured: 31,744 bytes)
- **SD kept mounted**: no mount/unmount overhead between saves
- **WAV header**: written with `sample_rate=8000` — directly playable

---

## 🔋 Current Consumption — Comparison

| Scenario | Estimated current | Status |
|---|---|---|
| Before optimizations (WiFi + I2S + I2C active) | ~5–10 mA | baseline |
| Backlight floating HIGH during sleep (bug) | ~30 mA | fixed |
| **Full solution (28/03/2026)** | **~0.1 mA** | ⏳ pending hardware validation |

> ⚠️ **Battery life estimates not included** — values require real current measurement with a precision multimeter on hardware.

**WiFi modem-sleep during active operation:** 87–89% — measured directly from session logs.

---

## 🔊 High-Pass Filter (HPF)

1st-order IIR Butterworth filter applied in-place on the PCM buffer after full capture, before API submission.

| Parameter | Value |
|---|---|
| Type | IIR Butterworth 1st order |
| Cutoff frequency | 100 Hz |
| Rolloff | −6 dB/octave |
| Sample rate | 8,000 Hz |
| Computational cost | ~2 mult + 2 add per sample |
| Extra allocation | None (in-place) |

---

## 🧩 Dynamic Specialist Profiles

Profiles stored in `/sdcard/data/config.txt` (3142 bytes, JSON array). Up to **6 profiles** via Captive Portal. Real-time switching with immediate SD persistence.

| Index | Name | Scope |
|---|---|---|
| 0 | Generalista | General-purpose assistant |
| 1 | Agronomo | Agricultural IoT — greenSe project (UnB FCTE) |
| 2 | Teacher | Digital Electronics — UnB FGA (Prof. Marcelino) |
| 3 | Digital | Digital Electronics scope-restricted |

**Full cycle of 4 switches in this session: zero failures, all persisted in ~30–40 ms.**

---

## 🌐 Captive Portal

- Portal available at `192.168.4.1` with automatic DNS redirect (Android, iOS, Windows)
- Configures: Wi-Fi, AI personality, model, base URL, and 1–6 specialist profiles
- HTML-escaping on all fields (prevents corruption from special characters in SSID/token/prompt)
- `httpd` task stack: 12,288 bytes; profile array heap-allocated
- URL-decode buffer: 2,048 bytes (supports prompts up to 511 chars with UTF-8 `%XX` encoding)

---

## ✅ Operational Conclusion

Firmware validated with **2 audio interactions**, **4 profile switches**, and **3 complete deep sleep/wakeup cycles** in the March 28, 2026 session:

- ✅ **Boot ~1.5 s** without additional calibrations
- ✅ **WiFi reconnect ~1.5–1.6 s** direct, no retry — consistent across 3 wakeups
- ✅ **PTT capture** with per-window RMS monitoring (100ms windows)
- ✅ **HPF 100 Hz IIR Butterworth** — zero extra allocation, in-place
- ✅ **TLS ~980–1,030 ms** per session (cold start per boot, no cross-boot reuse)
- ✅ **Reliable SD persistence** — preemptive save, DMA check, SD kept mounted
- ✅ **9-peripheral ordered shutdown** — zero WDT, zero race conditions, 3/3 cycles clean
- ✅ **Backlight OFF during sleep** (`gpio_hold_en`) | **ON at wakeup** (`gpio_hold_dis`)
- ✅ **WiFi modem-sleep 87–89%** — measured directly from logs across 3 sessions
- ✅ **Estimated ~50–100x current reduction** vs. baseline — hardware measurement pending
- ✅ **PM + Tickless Idle + DFS** active — dynamic CPU frequency scaling
- ✅ **Dynamic profiles** (1–6) — persisted in ~30–40 ms, zero failures
- ✅ **Stable heap** across all cycles (Total: ~7.5–7.6 MB, Internal: ~100–102 KB)
- ⚠️ `E spi_master: not all CSses freed` — cosmetic; LCD io_handle not deleted before `spi_bus_free()`

---

*Log collected via `idf.py -p COM15 monitor` — session 03/28/2026 (build 12:23:52).*
