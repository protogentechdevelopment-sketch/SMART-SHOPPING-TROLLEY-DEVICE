# 🛒 SMART SHOPPING TROLLEY DEVICE

> An embedded IoT system for automated retail billing using RFID/NFC product detection, weight verification, and real-time TFT display — built on STM32 + ESP32.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [System Architecture](#system-architecture)
- [Hardware Components](#hardware-components)
- [Software Tools](#software-tools)
- [Communication Protocols](#communication-protocols)
- [Circuit & Pin Connections](#circuit--pin-connections)
- [How It Works](#how-it-works)
- [Results](#results)
- [Future Scope](#future-scope)

---

## Overview

Traditional supermarket checkout relies on manual barcode scanning — causing long queues, billing delays, and human errors. This project eliminates that with an **automated, in-trolley billing system** built on embedded hardware.

When a product is placed into the trolley, an NFC/RFID reader instantly identifies it, a load cell verifies its weight, and the total bill updates on a touchscreen — **no cashier, no queue, no manual scanning**.

---

## Features

- ✅ Automatic product identification via RFID/NFC (RC522 reader, 13.56 MHz)
- ✅ Weight verification using load cell + HX711 24-bit ADC
- ✅ Real-time billing on 3.5" TFT touch display (480×320, ILI9488)
- ✅ Dual-MCU architecture — STM32 (sensor layer) + ESP32 (display/UI layer)
- ✅ Buzzer alert on weight mismatch or billing error
- ✅ Standalone operation — no centralized server or internet required
- ✅ Scalable for cloud/IoT integration via ESP32 Wi-Fi
- ✅ NFC tag detection latency: 200–300 ms | Total per-item processing: ~0.5–1 s

---

## System Architecture

```
┌───────────────────────────────────────────────────────────┐
│  INPUT LAYER       Load Cell → HX711 ADC                  │
│                    NFC Tag  → RC522 RFID Reader           │
├───────────────────────────────────────────────────────────┤
│  PROCESSING LAYER  STM32F103C6T6 (Slave — Sensor MCU)     │
│                    HAL-based firmware (SPI + UART + ADC)  │
├───────────────────────────────────────────────────────────┤
│  MASTER LAYER      ESP32 (Master — Display/UI MCU)        │
│                    UART from STM32 → TFT render           │
├───────────────────────────────────────────────────────────┤
│  OUTPUT LAYER      3.5" TFT Touch Display                 │
│                    Buzzer (error feedback)                │
└───────────────────────────────────────────────────────────┘
```

### Block Diagram

```
Load Cell ──► HX711 ──────────────────────────┐
                                               ▼
NFC Tag ──► RC522 RFID Reader ──► STM32F103C6T6 ──► ESP32 ──► 3.5" TFT Display
                                        │
                                        └──► Buzzer (GPIO)
                                        │
                              +5V Power Supply (LM2596 Buck)
```

---

## Hardware Components

| Component | Model | Interface | Role |
|---|---|---|---|
| Slave MCU | STM32F103C6T6 (ARM Cortex, 72MHz) | — | Sensor acquisition, control logic |
| Master MCU | ESP32 (Xtensa LX6, 240 MHz, Wi-Fi/BT) | — | Display management, UI |
| RFID Reader | MFRC522 (RC522) | SPI | NFC tag detection at 13.56 MHz |
| RFID Tags | NTAG213 (144-byte EEPROM) | NFC | Product ID + price storage |
| Load Cell| 50 kg Strain Gauge Load Cell | Analog | Measures product weight |
| ADC Module | HX711 (24-bit, PGA ×128) | 2-wire serial | Amplifies + digitizes load cell signal |
| Display | 3.5" TFT (ILI9488, 480×320) | SPI | Real-time product + billing UI |
| Programmer | ST-LINK V2 | SWD | STM32 firmware flash + debug |

---

## Software Tools

| Tool | Purpose |
|---|---|
| **STM32CubeMX** | Peripheral configuration, HAL code generation |
| **STM32CubeIDE** | Embedded C firmware development + debugging |
| **STM32CubeProgrammer** | Flash firmware to STM32 via ST-LINK (SWD) |
| **Arduino IDE** | ESP32 programming (TFT display + UART handling) |

---

## Communication Protocols

### SPI — STM32 ↔ RC522 RFID Reader

STM32 acts as SPI Master; RC522 as SPI Slave.

| STM32 Pin | RC522 Pin | Signal |
|---|---|---|
| PA5 | SCK | Serial clock |
| PA6 | MISO | Data: RC522 → STM32 |
| PA7 | MOSI | Data: STM32 → RC522 |
| PA4 | SDA (SS) | Chip select |
| PB0 | RST | Reset |
| GND | GND | Ground |
| 3.3V | VCC | Power |

### 2-Wire Serial — STM32 ↔ HX711

| STM32 Pin | HX711 Pin | Signal |
|---|---|---|
| PA0 | DT (DOUT) | 24-bit data output |
| PA1 | SCK | Clock input |
| 5V | VCC | Power |
| GND | GND | Ground |

### UART — STM32 ↔ ESP32

- Baud: **115200**, 8N1
- STM32 transmits: product UID, weight, price data
- ESP32 receives and renders to TFT display

### SPI — ESP32 ↔ TFT Display

- Library: `TFT_eSPI`
- Display controller: ILI9488
- Touch controller: XPT2046 (resistive touch via T_CS, T_CLK, T_DIN, T_DO, T_IRQ)

---

## Circuit & Pin Connections

### Load Cell → HX711 (Wheatstone Bridge)

| Load Cell Wire | HX711 Pin | Function |
|---|---|---|
| Red | E+ | Excitation + |
| Black | E− | Excitation − |
| Green | A+ | Signal + |
| White | A− | Signal − |

### Weight Calculation Formula

```
Offset   = Average of N raw readings (no load)
Weight   = (Raw Value − Offset) / Calibration Factor

Example:
  Raw = 8385000, Offset = 8350200, Factor = 210
  Weight = (8385000 − 8350200) / 210 ≈ 165g
```

---

## How It Works

1. **Boot** — STM32 initializes SPI (RC522), GPIO (HX711, Buzzer), UART (ESP32); ESP32 initializes TFT + UART.
2. **RFID Scan** — RC522 detects NFC tag using 13.56 MHz RF field. Anti-collision resolves multiple tags. UID extracted via SPI.
3. **Weight Read** — HX711 amplifies load cell output (Gain ×128, Channel A). STM32 reads 24-bit value via DT/SCK pins. Weight calculated after offset calibration.
4. **Sensor Fusion** — STM32 matches RFID UID to product database. Cross-checks weight against expected range. Triggers buzzer on mismatch.
5. **Data Transfer** — STM32 sends product name, price, weight, item count over UART at 115200 baud.
6. **Display Update** — ESP32 renders product info + running total on TFT in real time. Touch input allows cart review and checkout.

### RFID Data Flow (STM32 ↔ RC522)

```
STM32 initializes SPI
  └─► Sends REQA command via RC522
        └─► Tag responds with ATQA
              └─► Anti-collision → UID extracted (e.g., 04 A3 B2 9F)
                    └─► UID used as Product ID key
```

### STM32 Firmware Snippet (RFID detection)

```c
MFRC522_Init(&RFID);

while (1) {
    if (!cardPresent) {
        if (waitcardDetect(&RFID) == STATUS_OK) {
            if (MFRC522_ReadUid(&RFID, uid) == STATUS_OK) {
                cardPresent = 1;
                // Transmit UID to ESP32 via UART
                HAL_UART_Transmit(&huart1, uid, uidSize, HAL_MAX_DELAY);
            }
        }
    } else {
        if (waitcardRemoval(&RFID) == STATUS_OK) {
            cardPresent = 0;
            HAL_Delay(300);
        }
    }
}
```

### ESP32 Display Snippet (Arduino IDE)

```cpp
#include <SPI.h>
#include <TFT_eSPI.h>

TFT_eSPI tft = TFT_eSPI();
HardwareSerial STM(2); // UART2: RX=16, TX=17

void setup() {
    tft.init();
    tft.setRotation(1);
    STM.begin(115200, SERIAL_8N1, 16, 17);
    drawHomePage();
}
```

---

## Results

| Metric | Traditional (Barcode) | Proposed System |
|---|---|---|
| Per-item scan delay | 2–3 seconds | 0.5–1 second |
| 15–20 item checkout | 40–90 seconds | Near-instant (parallel) |
| NFC tag detection | — | 200–300 ms |
| Weight verification | — | 300–500 ms |
| STM32→ESP32 latency | — | < 50 ms |
| TFT display update | — | ~100 ms |
| Transaction time reduction | Baseline | **60–80% faster** |

- Dual-sensor verification (RFID + weight) significantly reduced false billing events.
- System operates fully standalone — no network, no external server required.

---

## Future Scope

- **IoT / Cloud** — Push purchase data to AWS IoT / ThingSpeak via ESP32 Wi-Fi for inventory analytics
- **Mobile App** — Real-time cart monitoring, item list, and total bill on customer smartphone
- **Computer Vision** — Replace NFC tags with camera + ML-based product recognition (eliminates tag requirement)
- **Digital Payments** — Integrate QR-code payment, NFC Pay, or mobile wallet — checkout without a counter
- **Anti-theft** — Weight + authentication algorithm to detect unauthorized item removal
- **Battery Operation** — Power optimization for fully wireless, battery-powered trolley deployment

> *Automating retail checkout with embedded intelligence — faster billing, zero queues.*
