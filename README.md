# ESP32-S3-Touch-LCD-1.47-M — 8-Gang Smart Switch Panel

[![ESPHome Version](https://img.shields.io/badge/ESPHome-2024.6%2B-blue)](https://esphome.io/)
[![Hardware](https://img.shields.io/badge/Hardware-ESP32--S3--Touch--LCD--1.47--M-orange)](https://www.waveshare.com/wiki/ESP32-S3-Touch-LCD-1.47)
[![Integration](https://img.shields.io/badge/Integration-Home%20Assistant-41BDF5)](https://www.home-assistant.io/)
[![Release](https://img.shields.io/badge/Release-v1.0.0-brightgreen)](SPECIFICATION_v1.0.md)

Production-ready ESPHome firmware for the **ESP32-S3-Touch-LCD-1.47-M** development board. Transforms the 1.47" $172 \times 320$ IPS touchscreen into a bidirectional 8-gang smart touch switch panel integrated with **Home Assistant**.

---

## 📸 Overview & Layout

The user interface operates in landscape orientation ($320 \times 172\text{ px}$) across 4 swipable pages:

```
+-------------------------------------------------------------------+
|  Switches                                                    1/4  |  <-- Header: Title & Page (20px)
|-------------------------------------------------------------------|  <-- Divider Line (y: 36)
|                                                                   |
|       +-------------------+       +-------------------+           |
|       |                   |       |                   |           |
|       |                   |       |                   |           |
|       |         1         |       |         2         |           |  <-- Buttons (114x80 px, y: 50..130)
|       |                   |       |                   |           |
|       |                   |       |                   |           |
|       +-------------------+       +-------------------+           |
|                                                                   |
|  192.168.1.147                                           22:45:00 |  <-- Footer: IP & SNTP Clock (20px)
+-------------------------------------------------------------------+
```

- **Screen 1 (Page `1/4`)**: Switches 1 & 2
- **Screen 2 (Page `2/4`)**: Switches 3 & 4
- **Screen 3 (Page `3/4`)**: Switches 5 & 6
- **Screen 4 (Page `4/4`)**: Switches 7 & 8

---

## ✨ Features

- **Boot Sequence (3 Seconds)**:
  - Cycles full-screen Red $\rightarrow$ Green $\rightarrow$ Blue hues with full-screen centered text **`TECHN8`** and drop shadow.
  - Automatically transitions to Screen 1.
- **8 Bidirectional Switches**:
  - Tapping a button toggles its state immediately on the LCD and updates Home Assistant in real time.
  - Toggling any switch from Home Assistant immediately updates the corresponding button on the display.
  - State **ON**: Bright emerald background with accent border and white digit.
  - State **OFF**: Dark slate background with subtle border and light gray digit.
- **Swipe & Touch Navigation**:
  - **Swipe Right $\rightarrow$ Left**: Next screen ($1 \rightarrow 2 \rightarrow 3 \rightarrow 4 \rightarrow 1$).
  - **Swipe Left $\rightarrow$ Right**: Previous screen ($4 \rightarrow 3 \rightarrow 2 \rightarrow 1 \rightarrow 4$).
  - Tapping top header or bottom footer navigates between pages.
- **Auto-Dimming & Power Saving**:
  - Dims display backlight smoothly to 0% after 60 seconds of inactivity.
  - Touching the dimmed screen smoothly wakes it up without accidentally toggling any switch.
  - Screen backlight is exposed to Home Assistant as a dimmable light entity (`light.screen_backlight`, 0–100%).
- **Header & Footer**:
  - Header: Left title "Switches" and right pagination (`1/4`, `2/4`, `3/4`, `4/4`) in 20px typography.
  - Footer: Left device IP address and right live synchronized SNTP clock (`HH:MM:SS`) in matching 20px typography.

---

## 📌 Verified Hardware Pin Mapping (ESP32-S3-Touch-LCD-1.47-M)

| Peripheral | Signal | ESP32-S3 GPIO | Interface / Role |
| :--- | :--- | :--- | :--- |
| **LCD Display (ST7789V / JD9853)** | **LCD_SCL** | **GPIO 38** | SPI Clock (20MHz) |
| | **LCD_SDA** | **GPIO 39** | SPI MOSI Data |
| | **LCD_CS** | **GPIO 21** | Chip Select |
| | **LCD_DC** | **GPIO 45** | Data / Command |
| | **LCD_RST** | **GPIO 40** | Display Hardware Reset |
| | **LCD_BL** | **GPIO 46** | Backlight LEDC PWM (1kHz) |
| **Touch Controller (AXS5106L)** | **TP_SDA** | **GPIO 42** | I²C Data line (400kHz, Addr `0x63`) |
| | **TP_SCL** | **GPIO 41** | I²C Clock line |
| | **TP_RST** | **GPIO 47** | Touch Reset |

---

## 🚀 Quick Setup & Flashing

### 1. Configure Secrets
Copy `secrets.yaml.example` to `secrets.yaml` and configure your Wi-Fi and Home Assistant credentials:
```bash
cp secrets.yaml.example secrets.yaml
```

Edit `secrets.yaml`:
```yaml
wifi_ssid: "Your_WiFi_SSID"
wifi_password: "Your_WiFi_Password"
fallback_password: "FallbackPassword"
api_encryption_key: "YOUR_32_BYTE_BASE64_KEY"
ota_password: "YourOTAPassword"
```

### 2. Compile and Upload
Using ESPHome with `uv`:
```bash
# Compile firmware
uv --system-certs run --with esphome --with pip-system-certs esphome compile esp32-s3-touch-147.yaml

# Flash to COM port (e.g. COM19)
uv --system-certs run --with esphome --with pip-system-certs esphome upload esp32-s3-touch-147.yaml --device COM19
```

### 3. Home Assistant Integration
The device is automatically discovered via mDNS (`esp32s3-touch-147m.local`). Add the ESPHome integration in Home Assistant and enter the encryption key configured in `secrets.yaml`.

---

## 📄 Documentation
For the complete technical specification, timings, hitbox calculations, and protocol details, see [`SPECIFICATION_v1.0.md`](SPECIFICATION_v1.0.md).
