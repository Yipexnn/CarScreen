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
- MHD adapter isn't needed for the screen. It's only used in phase 1 for sniffing (see 4.3).

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
| 8 | OBD2 Y-splitter (1 male → 2 female) | Phase 1: sniff MHD traffic in parallel | ~10 € | buy |
| 9 | IMU module (ICM-42688 or MPU-6050) | Optional drift backup: yaw rate / lateral g, works even if DSC data is slow | ~3–10 € | optional |
| 10 | Mount: 3D-printed bracket or vent/phone mount | Install | – | later |
| – | Existing ESP32 + CAN transceiver | Bench prototype / sniffer during phase 1 | – | have |

Notes:
- Check in the Waveshare wiki that the board's input range really is 7–36 V before connecting it to 12 V directly. If it isn't, add a 12→5 V buck converter (~5 €).
- Keep the CAN stub from the OBD port to the board short (<30 cm), or twist the pair.

## 3. Logging

- Ring buffer in PSRAM: all channels at 20 Hz × 300 s, about 0.5–1 MB.
- **Save clip:** long-press the button, or it triggers automatically on a warning. The last 5 minutes go to `/clips/YYYYMMDD_HHMMSS.csv` on SD, with flash as fallback.
- **Download:** the ESP32 runs a WiFi AP (`CarScreen`). A phone browser at `192.168.4.1` can list, download and delete clips.
- Clips stay on the device until you delete them. This works.

## 4. Data sources / IDs

### 4.1 Addressing (BMW UDS over ISO-TP, "extended addressing")
- Request CAN ID **0x6F1**, first data byte = target ECU address.
- Response CAN ID **0x600 + ECU address**, first byte = 0xF1.
- ECU addresses: **DME 0x12** (response 0x612), **DSC 0x29** (response 0x629). These are the standard BMW addresses and need confirming on the car.
- Standard OBD Mode 01 functional request: **0x7DF**, physical **0x7E0**, response from DME **0x7E8**.

### 4.2 Candidate signals

| Signal | Source | Request | Formula | Confidence |
|---|---|---|---|---|
| RPM | OBD 01 | PID 0x0C | (256A+B)/4 rpm | high |
| Coolant temp | OBD 01 | PID 0x05 | A−40 °C | high |
| Intake air temp | OBD 01 | PID 0x0F | A−40 °C | high |
| MAP / boost | OBD 01 | PID 0x0B | A kPa abs (boost = MAP − baro) | high |
| Baro | OBD 01 | PID 0x33 | A kPa | high |
| Calc. load | OBD 01 | PID 0x04 | A·100/255 % | high |
| Throttle | OBD 01 | PID 0x11 | A·100/255 % | high |
| Lambda / AFR (wideband) | OBD 01 | PID 0x24 or 0x34 (bank 1 sensor 1) | λ = (256A+B)·2/65536; AFR = λ·14.7 | medium (depends on which PIDs are supported, read PID 0x00/0x20) |
| Vehicle speed | OBD 01 | PID 0x0D | A km/h | high |
| **Oil temp** | DME UDS | **22 44 02** (DID 0x4402) | A−64 °C (community Torque setup for F20 N13) | medium, verify |
| Oil pressure | DME UDS | TBD, find in phase 1 | – | low. I'm not sure the N13 reports a real pressure value |
| Boost target / actual (BMW) | DME UDS | TBD (sniff MHD) | – | low |
| Wheel speeds ×4 | DSC 0x29 UDS | TBD DID, find via scan / sniff | – | low |
| Wheel speeds ×4 (alt.) | PT-CAN broadcast | E90 uses ID 0x0CE; the F-series ID is TBD (Racelogic F30 CAN DB lists wheel speeds) | – | low, only if needed |
| Steering angle / yaw | DSC / PT-CAN | TBD | – | low |

### 4.3 How to find the TBD IDs (phase 1)
1. **Supported-PID scan:** Mode 01 PIDs 0x00/0x20/0x40/0x60 tell you exactly which standard PIDs the DME supports.
2. **Sniff MHD:** put the Y-splitter on the OBD port, connect the MHD adapter plus the ESP32 in listen-only mode, and log in the MHD app with oil temp, boost, lambda and so on. The ESP32 records every 0x6F1 request and the replies, which shows which DIDs or memory reads MHD uses.
3. **DID sweep:** in a controlled sweep of `22 xx xx` against DME 0x12 and DSC 0x29 (ignition on, engine off), log the positive responses and correlate them by changing conditions (warm-up, turning wheels, rolling).
4. **Fallback if DSC polling is below 10 Hz:** tap PT-CAN for broadcast wheel speeds, or use the IMU for drift detection.

## 5. Modes

- **Normal:** oil temp, coolant, oil pressure (if available), AFR, boost, load. Peak/min markers on each.
- **Drift:**
  - `front = avg(FL, FR)`, `slipRL = RL − front`, `slipRR = RR − front`.
  - One rear above threshold → **ONE-WHEEL** (shows which side). Both above → **TWO-WHEEL**.
  - Big slip gauge, peak slip, drift timer. Temps go to a small strip at the bottom.
  - The car is RWD, so the fronts are a valid reference speed.
- **Warnings (all modes):** configurable thresholds, e.g. oil > 130 °C, coolant > 112 °C, AFR lean under boost. Full-screen red flash plus an automatic clip save.

## 6. Phases

1. **Discovery:** PID/DID scanner + sniffer on the existing ESP32. Fill the TBDs in 4.2 and measure poll rates.
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
