# Vlad's Boosted 944 — Saturday Morning Special v2.1

**All-in-one Android Dash & Launcher for a 1986 Porsche 944 with Speeduino Ocelot ECU.**

Head unit: **YT9216BJ** · Android 9.1 · 1280×720

---

## What's in this build

| Screen | What it does |
|--------|-------------|
| Boot | Retro-wave 944 splash (Extended to **7.0 s**) |
| Warning | WELCOME VLAD / ⚠ CAUTION card layout (Extended to **4.0 s**) |
| **DASH** | Big RPM tachometer, Boost+TPS mini gauges, sensor strip, Target Boost ±, AutoTune toggle, Save, Logging |
| MAPS | Google Maps with GPS speed overlay & Location permissions handling |
| **DYNO** | Real-time HP & Torque estimation with live RPM-based power curve plotting |
| **DRAG** | Auto-start Performance Timer: 0-60 mph, 0-100 mph, and 1/4 Mile ET/Trap Speed |
| **RADIO** | Retro 80s Boombox UI with animated cassette reels, VU meters, and Graphic EQ |
| **APPS** | Fully functional App Launcher to browse and launch all installed Android apps |
| FUEL TABLE | Interactive 16×16 VE table — tap cell, +/− edit |
| SPARK TABLE | Interactive 16×16 ignition advance table |
| BOOST CTRL | Live boost gauge + target control |
| AUTOTUNE | VE Analyze Live status + AFR monitoring |
| LOGGING | MPAndroidChart live chart + CSV export |
| SETTINGS | ECU status, vehicle info, app version |

---

## Launcher Mode
The app is now configured as a **Home Launcher**. 
- Pressing the 'Home' button on the head unit allows you to select "Vlad's 944" as the default.
- The **APPS** tab provides a seamless way to switch to other apps (Spotify, Maps, etc.) while keeping the Dash as the core OS experience.

---

## USB / ECU connection

### Hardware
- **Speeduino Ocelot** — Silicon Labs **CP2102N** USB-UART bridge
- Baud: **115200, 8N1**
- Connect USB-C from Ocelot to the head unit's USB-A OTG port
- The app auto-launches when the ECU is plugged in (`USB_DEVICE_ATTACHED` intent filter)

### Protocol
The app sends the Speeduino **0x41 "A" command** at ~20 Hz and parses the
115-byte `outpc[]` realtime data packet.

---

## Build & Install for YT9216BJ

### Prerequisites
- Android Studio Hedgehog or newer
- JDK 17
- Google Maps API key (for the GPS screen)

### Steps
1. **Clone / unzip** this project.
2. Open in Android Studio.
3. Add your Maps API key to `app/src/main/res/values/strings.xml`.
4. Run `./gradlew clean assembleDebug` to generate the universal APK.

### Installation
1. **Uninstall** any previous version of the app from the head unit first.
2. Copy `app/build/outputs/apk/debug/app-debug.apk` to a USB drive.
3. Install via the head unit's File Manager.
4. Set as **Default Home App** by pressing the Home button and selecting "Always".

---

## Architecture

```
MainActivity  →  DashApp (Compose)
                    ├── BootScreen (7s) / WarningScreen (4s)
                    └── MainDash (HorizontalPager + Scrollable Nav Bar)
                         ├── DashboardScreen
                         ├── GpsMapScreen
                         ├── DynoScreen (Live HP/Torque Curve)
                         ├── PerformanceScreen (0-60, 1/4 Mile)
                         ├── RadioScreen (80s Boombox UI)
                         ├── LauncherScreen (Installed Apps Grid)
                         ├── FuelTableScreen / SparkTableScreen
                         ├── BoostControlScreen
                         ├── AutotuneScreen
                         ├── LogsScreen
                         └── SettingsScreen

DashViewModel  ←  EcuSession  ←  UsbSerialPort (mik3y)
                                    └── SpeeduinoParser
```

---

## Safety notes
- **READ-ONLY by default** — the dash only polls telemetry; no writes without user action.
- Table edits are **staged in RAM** — not written to ECU until you implement the burn command.
- Do not burn while engine is above safe idle RPM.
