# CarScreen – BMW F20 118i (N13, MEVD17.2.x)

ESP32-based in-car display with Normal and Drift modes, warnings and a rolling 5-minute log.

## 1. Architecture

```
OBD2 port (pin 6 CAN-H, 14 CAN-L, 16 +12V, 4/5 GND)
  └─ open-wire OBD2 cable ─ fuse ─ Waveshare ESP32-S3-Touch-LCD-4.3B (onboard CAN, 7–36V in)
                                     ├─ LVGL UI (Normal / Drift / Warning)
                                     ├─ RAM ring buffer (5 min) → save clip to SD / flash
                                     ├─ WiFi AP + web page → download clips
                                     └─ buttons: short = mode, long = save clip
```

- 500 kbit/s diagnostic CAN behind the ZGW gateway. Nothing is broadcast here, so the ESP32 has to poll.
- MHD adapter isn't needed for the screen. MHD uses ENET, and its earlier captures are a DID source.

## 2. Parts list

| # | Part | Purpose | Approx. price | Status |
|---|------|---------|---------------|--------|
| 1 | Waveshare **ESP32-S3-Touch-LCD-4.3B** (800×480 touch, 16 MB flash, 8 MB PSRAM, CAN + RS485 terminal, TF slot, wide-range DC input) | Main unit | ~37–45 USD | buy |
| 2 | Same board with the **-BOX** case, if you want a finished housing | Optional | +~10 USD | optional |
| 3 | OBD2 16-pin male open-wire / pigtail cable | Car connection | ~8 € | buy |
| 4 | Inline blade fuse holder + 2 A fuse | Protect the +12 V feed | ~3 € | buy |
| 5 | microSD card, 8–32 GB | Log clip storage | ~5 € | buy |
| 6 | Momentary push buttons (2×), ideally on a cable to the gear-lever area | Mode / save clip | – | have |
| 7 | Voltage divider (100 kΩ / 22 kΩ + 100 nF) on +12 V into an ADC pin | Engine-running detect (>13.2 V = alternator charging) so the car isn't kept awake | <1 € | buy |
| 8 | IMU module (ICM-42688 or MPU-6050) | Drift backup: yaw rate and lateral g, in case DSC wheel speeds can't be read | ~3–10 € | recommended |
| 9 | Mount: 3D-printed bracket or vent/phone mount | Install | – | later |
| – | AZ-Delivery ESP32 DevKitC V4 + CAN transceiver (model unconfirmed), SSD1306 OLED | Prototype for phase 1 | – | have |

Notes:
- Check in the Waveshare wiki that the board's input range really is 7–36 V before connecting it to 12 V directly. If it isn't, add a 12→5 V buck converter (~5 €).
- Keep the CAN stub from the OBD port to the board short (<30 cm), or twist the pair.

## 3. Logging

- Ring buffer in PSRAM: all channels at 20 Hz × 300 s, about 0.5–1 MB.
- **Save clip:** long-press the button, or it triggers automatically on a warning. The last 5 minutes go to `/clips/YYYYMMDD_HHMMSS.csv` on SD, with flash as fallback.
- **Download:** the ESP32 runs a WiFi AP (`CarScreen`). A phone browser at `192.168.4.1` can list, download and delete clips.
- Clips stay on the device until you delete them. This works.

## 4. Data sources / IDs

> Status legend: **works** = confirmed on the car with the ESP32 in earlier chats · **MHD** = seen in MHD ENET captures, not tested on the ESP32 · **lead** = forum or estimate only.
> Earlier findings are unproven and unreliable. Verify everything again in phase 1.

### 4.1 Addressing
- **0x7DF → 0x7E8** (functional): **works** for OBD Mode 01 and for UDS `22 44 02`.
- **0x7E0 → 0x7E8** (physical DME): this is what the ZGW uses internally (from ENET traffic). Not yet tested from the ESP32.
- **0x6F1 + target byte → 0x600+addr** (BMW extended addressing over D-CAN): untested. **DME 0x12** is confirmed as the DME address in the ENET captures. **DSC 0x29** is a standard BMW address that hasn't been seen on this car.
- The ZGW filters PT-CAN. Passive listening only sees **0x130** (on E-series that is terminal/ignition status, which could serve as an ignition-on signal; unverified).
- MHD talks **ENET** (DoIP-like, 192.168.10.10:6801, tester 0xF4 → DME 0x12), not CAN. A CAN sniffer can't see MHD traffic, so use the ENET captures for that.
- ISO-TP: single-frame only so far. Multi-frame handling is needed for F300 and other long DIDs.

### 4.2 Signals

| Signal | Request | Formula | Status |
|---|---|---|---|
| Engine load | OBD 01 04 | A·100/255 % | works |
| Coolant temp | OBD 01 05 | A−40 °C | works |
| MAP | OBD 01 0B | A kPa abs | works |
| IAT | OBD 01 0F | A−40 °C | works |
| Baro | OBD 01 33 | A kPa | works |
| Boost (derived) | MAP − Baro | kPa → ×0.145 psi | works (derived) |
| RPM | OBD 01 0C | (256A+B)/4 | untested, standard |
| Throttle | OBD 01 11 | A·100/255 % | untested, standard |
| Lambda / AFR | OBD 01 24 / 34 / 44 | λ=(256A+B)·2/65536, AFR=λ·14.7 | untested. Check support with PID 00/20/40 |
| Oil temp (OBD) | OBD 01 5C | – | **not supported** |
| **Oil temp** | UDS 22 44 02 | (A·256+B)−64 °C **or** raw·0.7037−43.74 | works (reply `05 62 44 02 00 42`). The formula is unresolved, so compare with MHD at hot oil |
| Coolant (BMW) | UDS 22 43 00 | raw·0.7333−46.4 | MHD |
| IAT (BMW) | UDS 22 58 0B | raw16·0.04762−1217 | MHD, 2-point fit |
| Boost manifold | UDS 22 4A B0 | raw16·0.000576−14.93 psi | MHD, 2-point fit |
| Oil pressure | UDS 22 58 6F | unknown (A ≈ 9–12) | lead |
| Boost act/target, oil T/P, RPM, timing corr. | UDS 22 D9 0B / D9 06 / D9 6E / D9 6F / F4 0C / D0 12–16 | estimates | lead |
| IAT / oil (MHD session start) | UDS 22 F1 50 / 58 6A | – | MHD |
| F300 composite | 2C 01 define + 22 F3 00 | layout changes per session | MHD only, avoid. Poll individual DIDs instead |
| Wheel speeds ×4 | DSC 0x29 via 6F1, DID TBD | – | **not found** |
| Steering / yaw | 0x130? / DSC | – | not found |

### 4.3 Known issues from earlier attempts
- **ESP and TPMS warnings after driving**, which cleared after unplugging. Likely causes are high-rate UDS requests on the functional 0x7DF (every ECU on the gateway gets them) and no rate limiting. Fixes to try, in order:
  1. Physical 0x7E0 (or 6F1/0x12) instead of 0x7DF.
  2. A lower total poll rate (start at ≤10 requests/s).
  3. TesterPresent `3E 80` every 2 s, only if a session is used.
  4. Reading DTCs to see which ECU complains.
- **Parser bug:** `requestUSDData` checks `data[0]==0x04`, but the 2-byte DID reply has `data[0]==0x05`. It should check the length per DID instead.
- **ISTA over ENET doesn't connect** ("cable unplugged"). Possibly OBD pins, the fuse or the ZGW. Separate issue.
- No NRCs were ever logged. The scanner should log `7F 22 xx`.

### 4.4 Phase 1 plan (discovery)
1. Fix the parser and switch to physical 0x7E0. Confirm that the working signals still work and that no ESP/TPMS warnings appear after a 30 min drive.
2. Supported-PID scan (01 00/20/40/60) to find RPM, throttle and lambda support.
3. Test 6F1 addressing: `6F1: 12 03 22 44 02` → expect reply on 0x612. If that works, try DSC `6F1: 29 03 22 F1 90` (VIN) → 0x629.
4. If DSC answers: a slow DID sweep on 0x29 with the wheels lifted or rolling, to find the wheel-speed DID. Measure the achievable rate.
5. Resolve the oil temp formula and the 586F oil pressure DID against MHD readings, warm and cold.
6. **Drift fallback:** if no wheel speeds can be read at ≥10 Hz, use an IMU (yaw rate vs. speed gives the slip angle) plus RPM/speed ratio for wheelspin. One-wheel versus two-wheel detection then isn't possible, and only "drift angle" plus "wheelspin" can be shown.

## 5. Modes

- **Normal:** oil temp, coolant, oil pressure (if available), AFR, boost, load. Peak/min markers on each.
- **Drift:**
  - `front = avg(FL, FR)`, `slipRL = RL − front`, `slipRR = RR − front`.
  - One rear above threshold → **ONE-WHEEL** (shows which side). Both above → **TWO-WHEEL**.
  - Big slip gauge, peak slip, drift timer. Temps go to a small strip at the bottom.
  - The car is RWD, so the fronts are a valid reference speed.
- **Warnings (all modes):** configurable thresholds, e.g. oil > 130 °C, coolant > 112 °C, AFR lean under boost. Full-screen red flash plus an automatic clip save.

## 6. Phases

1. **Discovery:** see 4.4. Scanner on the existing ESP32, with no new parts needed.
2. **Data layer:** poll scheduler (fast: wheel speeds, RPM, MAP, about 20 Hz; slow: temps, 1–2 Hz), engine-running detection, sleep.
3. **Logger:** PSRAM ring buffer, clip save, WiFi AP + download page.
4. **UI:** LVGL Normal / Drift / Warning screens on the 4.3B.
5. **Install:** mount, buttons, fused wiring, threshold tuning.

Stack: PlatformIO, Arduino-ESP32 (ESP-IDF TWAI driver), LVGL 8/9, SD/LittleFS, ESPAsyncWebServer.

## 7. References
- MEVD17.2 test plans (diagnostic background): https://www.1addicts.com/forums/attachment.php?attachmentid=3372432&d=1706186165
- F20 N13 oil temp via Torque (source of DID 0x4402): https://www.bimmerforums.co.uk/threads/torque-app-f20-n13-now-displays-oil-temperature.590040/
- BMW F-series UDS/ISO-TP addressing: https://www.projectgus.com/2022/06/bmw-f-series-gear-selector-part-two-breakthrough/
- E90 DBC (0x6F1 / 0x0CE): https://github.com/nberlette/canbus/blob/main/dbc/bmw-e90.dbc
- Racelogic F30 CAN DB (wheel speeds): https://www.racelogic.co.uk/_downloads/vbox/Vehicles/Other/Docs/BMW-3%20Series%20(F30)%202012-.pdf
- Waveshare 4.3B wiki: https://docs.waveshare.com/ESP32-S3-Touch-LCD-4.3B
