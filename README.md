# Vlad's Boosted 944 — Saturday Morning Special v2

**Single-module Android dash app for a 1986 Porsche 944 with Speeduino Ocelot ECU.**

Head unit: **YT9216BJ** · Android 9.1 · 1280×720

---

## What's in this build

| Screen | What it does |
|--------|-------------|
| Boot | Retro-wave 944 splash (3.5 s) |
| Warning | WELCOME VLAD / ⚠ CAUTION card layout |
| **DASH** | Big RPM tachometer, Boost+TPS mini gauges, sensor strip, Target Boost ±, AutoTune toggle, Save, Logging |
| MAPS | Google Maps with GPS speed overlay |
| FUEL TABLE | Interactive 16×16 VE table — tap cell, +/− edit |
| SPARK TABLE | Interactive 16×16 ignition advance table |
| BOOST CTRL | Live boost gauge + target control |
| AUTOTUNE | VE Analyze Live status + AFR monitoring |
| LOGGING | MPAndroidChart live chart + CSV export |
| SETTINGS | ECU status, vehicle info, app version |

---

## USB / ECU connection

### Hardware
- **Speeduino Ocelot** — Silicon Labs **CP2102N** USB-UART bridge
- Baud: **115200, 8N1**
- Connect USB-C from Ocelot to the head unit's USB-A OTG port
- The app auto-launches when the ECU is plugged in (`USB_DEVICE_ATTACHED` intent filter)

### Protocol
The app sends the Speeduino **0x41 "A" command** at ~20 Hz and parses the
115-byte `outpc[]` realtime data packet:

| Byte(s) | Field | Scaling |
|---------|-------|---------|
| 0 | Status1 flags | — |
| 6–7 | RPM | hi<<8\|lo |
| 9 | Battery | × 0.1 V |
| 11 | CLT | raw − 40 → °C |
| 13 | IAT | raw − 40 → °C |
| 16–17 | MAP | / 10 → kPa |
| 19 | TPS | / 2.55 → % |
| 21 | O2/AFR | × 0.1 |
| 23 | Ignition advance | × 0.5 − 64 → ° |
| 31 | Gear | 0–6 |
| 34 | Knock retard | × 0.5 → ° |
| 40–41 | Loop time | µs |
| 47 | Corrections | % |
| 49 | Boost duty | % |

Return codes handled:
- `0x00` OK → parse payload
- `0xDE` CRC error → retry ≤3 with backoff
- `0xBB` Busy → exponential backoff
- `0xEE` Range error → log, stop retrying

When no USB device is found the app falls back to **smooth simulation** so
the UI works without hardware.

---

## Build instructions

### Prerequisites
- Android Studio Hedgehog or newer
- JDK 17
- Google Maps API key (for the GPS screen)

### Steps

1. **Clone / unzip** this project
2. Open in Android Studio
3. Add your Maps API key to `app/src/main/res/values/strings.xml`:
   ```xml
   <string name="google_maps_key">YOUR_API_KEY_HERE</string>
   ```
   And add to `AndroidManifest.xml` inside `<application>`:
   ```xml
   <meta-data android:name="com.google.android.geo.API_KEY"
              android:value="@string/google_maps_key" />
   ```
4. `Build → Build APK(s)` or `Run` on the YT9216BJ head unit via ADB

### ADB side-load
```bash
adb connect <head-unit-ip>:5555
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

Or copy the APK to a USB stick and install via the head unit's file manager.

---

## Screens reference

### Dashboard layout (matches render exactly)

```
┌─────────────────────────────────────────────────────────────────┐
│  12:47 PM  APR 24, 2026  │ ECU CONNECTED ●● LIVE DATA │  13.8V  │  ← 40px status bar
├──────────┬──────────────────────────────┬────────────────────────┤
│  BOOST   │                              │   TARGET BOOST         │
│ 18.7 PSI │      BIG RPM TACHOMETER      │   − ████ 22.0 PSI +   │
│          │         4820 RPM             │                        │
│  TPS     │           [3]                │   AUTOTUNE  [ON ●]     │
│  74 %    │          GEAR                │                        │
│          │                              │   [SAVE]  [LOGGING ●]  │
├──────────┴──────────────────────────────┴────────────────────────┤
│  CLT 182°F  │  IAT 96°F  │  AFR 11.8  │  GEAR 3  │  IGN 24.5°   │  ← 48px sensor strip
├─────────────────────────────────────────────────────────────────┤
│ ⏱DASH │ 🗺MAPS │ ⛽FUEL TABLE │ ⚡SPARK TABLE │ 💨BOOST │ 🎯AUTOTUNE │ 📊LOGGING │ ⚙SETTINGS │  ← 70px nav
└─────────────────────────────────────────────────────────────────┘
```

---

## Safety notes

- **READ-ONLY by default** — the dash only polls telemetry; no writes without user action
- Table edits are **staged in RAM** — not written to ECU until you implement the burn command
- Always create a tune backup before writing/burning any changes
- Do not burn while engine is above safe idle RPM

---

## Architecture

```
MainActivity  →  DashApp (Compose)
                    ├── BootScreen / WarningScreen
                    └── MainDash (HorizontalPager + nav bar)
                         ├── DashboardScreen
                         ├── GpsMapScreen
                         ├── FuelTableScreen
                         ├── SparkTableScreen
                         ├── BoostControlScreen
                         ├── AutotuneScreen
                         ├── LogsScreen
                         └── SettingsScreen

DashViewModel  ←  EcuSession  ←  UsbSerialPort (mik3y)
                                    └── SpeeduinoParser
```

All serial I/O runs on `Dispatchers.IO`. UI collects `StateFlow` on the main thread.
Telemetry decimation: raw poll ~20 Hz, UI updates capped by Compose recomposition.
