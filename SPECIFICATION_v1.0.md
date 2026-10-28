# Technical Specification: ESP32-S3-Touch-LCD-1.47-M Smart Switch Panel
**System Version:** Release 1.0.0  
**Target Hardware:** ESP32-S3-Touch-LCD-1.47-M  
**Date:** October 2026  
**Status:** Approved / Production-Ready  

---

## 1. Executive Summary
This document provides the exhaustive engineering specification for the first major release (v1.0.0) of the **ESP32-S3-Touch-LCD-1.47-M Smart Switch Panel**. 

The solution transforms a compact ESP32-S3-Touch-LCD-1.47-M development board featuring a 1.47-inch 172×320 IPS display and capacitive touch sensor into a bidirectional, Home Assistant-integrated 8-gang smart touch switch panel. The firmware is built on the ESPHome framework running on the ESP-IDF 5.5.6 framework, with custom low-level display and touch drivers, smooth PWM backlight control, and power-saving heuristics.

```mermaid
graph LR
    subgraph ESP32-S3-Touch-LCD-1.47-M
        MCU["ESP32-S3 (240MHz, 8MB PSRAM)"]
        LCD["1.47\" IPS LCD (ST7789V / JD9853)"]
        TOUCH["AXS5106L Touch Controller"]
        BL["Backlight LED (LEDC PWM)"]
    end
    
    subgraph Home Assistant Ecosystem
        HA["Home Assistant Core"]
        API["Native Encrypted API"]
        NTP["SNTP Time Server"]
    end
    
    TOUCH -->|I2C 0x63 (Raw Coordinates)| MCU
    MCU -->|SPI 20MHz (Landscape UI)| LCD
    MCU -->|PWM Duty Cycle| BL
    MCU <-->|Bidirectional Sync (8 Switches + Dimmer)| API <--> HA
    NTP -->|SNTP Sync (Europe/Amsterdam)| MCU
```

---

## 2. Hardware Architecture & Pinout Matrix

### 2.1 Component Specifications
| Subsystem | Component | Specifications |
| :--- | :--- | :--- |
| **SoC** | Espressif ESP32-S3 (QFN56) | Dual-core Xtensa LX7 @ 240MHz, 512KB SRAM, 8MB Octal PSRAM, 16MB Flash |
| **Display** | 1.47" IPS LCD Panel | $172 \times 320$ pixels, 262K colors, JD9853 / ST7789V controller |
| **Touch Sensor** | Capacitive Touch (AXS5106L / CST5106L) | Single/Multi-point capacitive touch sensing, I2C interface |
| **PMIC / Power** | ETA6098 PMIC | USB-C 5V input, battery charging support, 3.3V LDO |
| **Backlight** | High-efficiency White LED string | Driven via GPIO46 with hardware LEDC PWM dimming |

### 2.2 GPIO Interfacing Matrix
| Signal Name | ESP32-S3 GPIO | Bus / Protocol | Functional Role | Notes |
| :--- | :--- | :--- | :--- | :--- |
| **LCD_SCL** | `GPIO38` | SPI CLK | Display serial clock | Max 20MHz |
| **LCD_SDA** | `GPIO39` | SPI MOSI | Display serial data | Master-Out-Slave-In |
| **LCD_CS** | `GPIO21` | SPI CS | Display chip select | Active low |
| **LCD_DC** | `GPIO45` | GPIO Output | Display Data / Command selection | Hardware strapping pin |
| **LCD_RST** | `GPIO40` | GPIO Output | Display hardware reset | Active low |
| **LCD_BL** | `GPIO46` | LEDC PWM (Output) | Backlight intensity modulation | Strapping pin, 1kHz PWM |
| **TP_SDA** | `GPIO42` | I2C SDA | Capacitive touch serial data | $400\text{ kHz}$ Fast Mode |
| **TP_SCL** | `GPIO41` | I2C SCL | Capacitive touch serial clock | $400\text{ kHz}$ Fast Mode |
| **TP_RST** | `GPIO47` | GPIO Output | Capacitive touch hardware reset | Active low |
| **TP_INT** | *N/A* | Interrupt | Touch event interrupt | Polling mode active |

---

## 3. Firmware Architecture & Component Drivers

### 3.1 Display Subsystem (`components/st7789v/`)
The physical display is an asymmetric narrow aspect ratio panel ($172 \times 320$ portrait natively). The firmware operates the panel in **Landscape Mode** ($320 \times 172$):
- **Driver Platform:** Customized `st7789v` component.
- **MADCTL Register:** Configured to `0x80` (`MY` row address order reversed) to eliminate horizontal mirroring and match the landscape orientation.
- **Color Format:** `color_order: BGR`, `invert_colors: false`.
- **SPI Bus Frequency:** $20\text{ MHz}$ data rate.
- **Refresh Rate:** 100ms update interval ($10\text{ fps}$) providing responsive state rendering with low CPU overhead.

### 3.2 Capacitive Touch Driver (`components/axs5106/`)
The touch controller operates at **7-bit I2C address `0x63`**:
- **Protocol Timing:** Requires writing register pointer `0x01`, waiting a mandatory $45\text{ \mu s}$ settling time, and reading 14 bytes raw data without evaluating return ACK codes (due to non-standard ACK implementation on the AXS5106/CST5106L silicon).
- **Coordinate Transformation:**
  - `swap_xy: true`: Converts native vertical sensor geometry to landscape coordinates.
  - `mirror_x: true`: Synchronizes touch X with display landscape horizontal orientation.
  - `mirror_y: false`: Retains standard vertical downward orientation.
- **Calibration Bounds:** $X \in [0, 320]$, $Y \in [0, 172]$.

---

## 4. User Experience & Graphical Interface Specification

```
+-------------------------------------------------------------------+
|  Switches                                                    1/4  |  <-- Header (y: 0..36, Title & Page)
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
|  192.168.1.147                                           22:45:00 |  <-- Footer (y: 152, IP & Time)
+-------------------------------------------------------------------+
```

### 4.1 Boot Sequence
- **Duration:** 3.0 seconds elapsed from initial power-on.
- **Visuals:** Full-screen solid color cycling across Red $\rightarrow$ Green $\rightarrow$ Blue (1 second per primary hue).
- **Branding:** Centered full-screen text **"TECHN8"** rendered with a double-pass drop shadow (Black shadow offset $+2\text{px}$, White foreground, font size 46).
- **Auto-Transition:** Automatically hands off to Screen 1 upon completion.

### 4.2 Screen & Page Layout (Screens 1 to 4)
The interface is structured into four distinct screens, each hosting two dedicated switches:
- **Screen 1:** Switch 1 & Switch 2 (Pagination: `1/4`)
- **Screen 2:** Switch 3 & Switch 4 (Pagination: `2/4`)
- **Screen 3:** Switch 5 & Switch 6 (Pagination: `3/4`)
- **Screen 4:** Switch 7 & Switch 8 (Pagination: `4/4`)

#### 4.2.1 Header Zone ($y = 0 \dots 36$)
- **Title:** Left-aligned `"Switches"` at $(x=16, y=18)$, rendered in white (`#FFFFFF`) with 20px font (`font_title`).
- **Page Counter:** Right-aligned `"1/4"`, `"2/4"`, `"3/4"`, `"4/4"` at $(x=304, y=18)$, rendered in soft gray (`color_light_gray`) with consistent 20px font (`font_info`).
- **Divider Line:** Solid horizontal rule spanning $(x=16 \dots 304)$ positioned strictly at **$y = 36$** (providing a clean $8\text{px}$ clearance below header text).

#### 4.2.2 Switch Buttons Zone ($y = 50 \dots 130$)
Each screen hosts two symmetrically positioned, minimalist push buttons:
- **Button Geometry:** Width $114\text{ px}$, Height $80\text{ px}$.
- **Horizontal Positioning:**
  - Left Button: $x \in [30, 144]$ (Center $x = 87$).
  - Central Inter-Button Gap: $32\text{ px}$.
  - Right Button: $x \in [176, 290]$ (Center $x = 233$).
  - Outer Margins: Exactly $30\text{ px}$ on both left and right edges.
- **Vertical Positioning:** $y \in [50, 130]$ (Center $y = 90$), providing $14\text{ px}$ upper clearance to the divider line and $22\text{ px}$ lower clearance to the footer text.
- **Button Labeling:** Large, bold single digit (1 to 8) centered at $(87, 90)$ or $(233, 90)$ using size 38 font (`font_huge`).
- **State Representation:**
  - **State ON:** Solid emerald background (`rgb(10%, 65%, 25%)`), crisp white outer boundary, neon green accent border, white digit.
  - **State OFF:** Solid dark slate background (`rgb(14%, 16%, 20%)`), muted border (`rgb(35%, 38%, 45%)`), light gray digit.

#### 4.2.3 Footer Zone ($y = 152$)
- **Device IP Address:** Left-aligned at $(x=16, y=152)$ using 20px font (`font_info`) in accent green. Automatically extracts DHCP/Static STA IP address.
- **Synchronized Time:** Right-aligned at $(x=304, y=152)$ using 20px font (`font_info`) in bright white (`HH:MM:SS`). Displays `--:--:--` in amber until SNTP synchronization completes.

---

## 5. Touch Gestures & Interaction State Machine

### 5.1 Swipe Navigation
- **Next Page (Forward):** Swipe from **Right to Left** ($\Delta X < -30\text{ px}$, $|\Delta Y| < 80\text{ px}$). Cycles $1 \rightarrow 2 \rightarrow 3 \rightarrow 4 \rightarrow 1$.
- **Previous Page (Backward):** Swipe from **Left to Right** ($\Delta X > +30\text{ px}$, $|\Delta Y| < 80\text{ px}$). Cycles $4 \rightarrow 3 \rightarrow 2 \rightarrow 1 \rightarrow 4$.

### 5.2 Button Tap Detection
- **Touch Duration Threshold:** Gestures released under $600\text{ ms}$ qualify as taps.
- **Vertical Hitbox:** $y \in [38, 142]$ (encompassing button bounds $50 \dots 130$ with generous tolerance).
- **Horizontal Hitboxes:**
  - Left Button Hitbox: $x \in [15, 155]$
  - Right Button Hitbox: $x \in [165, 305]$
- **Action:** Immediately executes `id(switch_X).toggle()`, reflecting on the LCD display on the next 100ms render cycle and dispatching the state change to Home Assistant over the native API.

### 5.3 Auxiliary Tap Navigation
- Tapping the upper header area ($y < 36$) or the lower footer area ($y > 144$) navigates:
  - Right side ($x > 160$): Advance to Next Screen.
  - Left side ($x \le 160$): Return to Previous Screen.

### 5.4 Inactivity Dimming & Wake-Up Protocol
- **Timer:** Monitored via a 1-second background interval checking `millis() - last_activity_time`.
- **Auto-Dimming:** After **60 seconds (1 minute)** of zero touch activity, the backlight dims to 0% over a 1.0-second smooth transition (`call.set_transition_length(1000)`).
- **Touch Wake-Up:** Touching the dimmed panel restores the backlight to 100% within $300\text{ ms}$. 
- **Accidental Click Prevention:** When waking up from dimmed state, `gesture_handled = true` is set immediately, intentionally discarding the wake touch so no button switch is inadvertently toggled.

---

## 6. Home Assistant Integration & Security

### 6.1 Native API Configuration
The device integrates directly into Home Assistant via the native ESPHome API:
- **Port:** 6053 (TCP)
- **Encryption Key:** Configured via `api_encryption_key` in `secrets.yaml` (32-byte base64)
- **Discovery:** Zero-configuration mDNS broadcasting (`esp32s3-touch-147m.local`).

### 6.2 Exposed Entities
| Entity ID | Domain | Integration Type | Description |
| :--- | :--- | :--- | :--- |
| `switch.switch_1` to `switch.switch_8` | `switch` | Template Switch | 8 independent on/off switches with bidirectional sync |
| `light.screen_backlight` | `light` | Monochromatic PWM | Dimmable light entity allowing 0–100% brightness control from HA |
| `sensor.wifi_signal` | `sensor` | Wi-Fi Signal | Real-time signal strength (dBm) |
| `sensor.uptime` | `sensor` | System | Device uptime counter |
| `button.restart` | `button` | System | Remote firmware soft reboot |

### 6.3 Time Synchronization
- **Platform:** SNTP (`Europe/Amsterdam` timezone).
- **NTP Servers:** `0.pool.ntp.org`, `1.pool.ntp.org`, `time.google.com`.
- **Real-time Clock:** Formatted as `%H:%M:%S` with automatic daylight saving time adjustment.

---

## 7. Build, Flashing & Maintenance Guide

### 7.1 Compilation Toolchain
The project uses the `uv` Python package runner with isolated virtual environments and system SSL certificates:

```powershell
# 1. Compile Firmware
uv --system-certs run --with esphome --with pip-system-certs esphome compile esp32-s3-touch-147.yaml

# 2. Flash to Device via USB-Serial/JTAG on COM19
uv --system-certs run --with esphome --with pip-system-certs esphome upload esp32-s3-touch-147.yaml --device COM19

# 3. View Real-time Device Logs
uv --system-certs run --with esphome --with pip-system-certs esphome logs esp32-s3-touch-147.yaml --device COM19
```

### 7.2 OTA (Over-The-Air) Updates
Once installed on the local Wi-Fi network (`Faveo-WiFi-IoT`), future updates can be flashed wirelessly over the network:
```powershell
uv --system-certs run --with esphome --with pip-system-certs esphome run esp32-s3-touch-147.yaml
```

---

## 8. Release Checklist & Verification Summary
- [x] Boot animation: 3-second RGB cycle with centered "TECHN8" and drop shadow.
- [x] Display rendering: High-contrast black background, zero color inversion, correct BGR color order.
- [x] Touch calibration: AXS5106L I2C address `0x63`, `mirror_x: true`, exact button-to-switch mapping.
- [x] Gestures: Right-to-left swipe advances page; left-to-right swipe retreats page.
- [x] Button layout: $114 \times 80\text{ px}$ centered buttons with 38px numbers and no text overlap.
- [x] Typography: Consistent 20px font for Page indicator, IP address, and Time.
- [x] Divider line: Positioned at $y = 36$ to prevent title overlap.
- [x] Power management: 60-second auto-dim to 0% with smooth wake-on-touch protection.
- [x] Home Assistant: 8 switches and 1 dimmable light entity fully synchronized over encrypted API.
