# ESP32 NES Retro Game Console 🎮

[![ESP32](https://img.shields.io/badge/Platform-ESP32-blue.svg)](https://www.espressif.com/en/products/socs/esp32)
[![Language](https://img.shields.io/badge/Language-C%20%2F%20C%2B%2B%20%2F%20Arduino-orange.svg)]()
[![Display](https://img.shields.io/badge/Display-ST7789%20SPI%20(240x240)-green.svg)]()
[![License: GPL-3.0](https://img.shields.io/badge/License-GPL%203.0-blue.svg)](https://opensource.org/licenses/GPL-3.0)
[![Institution](https://img.shields.io/badge/Institution-VNIT%20Nagpur-red.svg)](https://vnit.ac.in)

A full-featured, portable **Nintendo Entertainment System (NES) emulator** engineered for the **ESP32 microcontroller**. The system simulates the MOS Technology 6502 CPU and the NES Picture Processing Unit (PPU) in real time, driving a 1.3" ST7789 TFT display using high-speed SPI with Direct Memory Access (DMA), loading `.nes` ROMs directly from a FAT32 MicroSD card, and supporting physical gamepad buttons, PS/2 keyboards, and I2S DAC audio.

Developed as an academic and embedded systems project at **Visvesvaraya National Institute of Technology (VNIT), Nagpur**.

---

## 🏛️ Academic Information & Project Team

**Department of Electronics and Communication Engineering**  
**Visvesvaraya National Institute of Technology (VNIT), Nagpur**

| Name | Enrollment Number | Role |
| :--- | :--- | :--- |
| **Aditya Kinjawadekar** | BT23ECE064 | Emulation Core & Memory Optimization |
| **Vinay Shelke** | BT23ECE065 | Hardware Interfacing & System Integration |
| **Siddhant Jinturkar** | BT23ECE066 | Display Pipeline, SPI DMA & Controls |
| **Akshat Shevde** | BT23ECE067 | SD Card VFS & File Management |

---

## 📑 Table of Contents

- [Key Features](#-key-features)
- [Demonstration](#-demonstration)
- [System Architecture](#-system-architecture)
  - [MOS 6502 CPU Emulation](#1-mos-6502-cpu-emulation)
  - [NES PPU Emulation](#2-nes-ppu-emulation)
  - [Display Driving via SPI DMA](#3-display-driving-via-spi-dma)
  - [MicroSD File System & ROM Loading](#4-microsd-file-system--rom-loading)
  - [Extensive Cartridge Mapper Support](#5-extensive-cartridge-mapper-support)
- [Hardware Specifications](#-hardware-specifications)
- [Bill of Materials (BOM)](#-bill-of-materials-bom)
- [Wiring & Pinout Configuration](#-wiring--pinout-configuration)
  - [ST7789 TFT Display (SPI)](#1-st7789-tft-display-spi)
  - [MicroSD Card Module (SPI)](#2-microsd-card-module-spi)
  - [Gamepad Push Buttons (8-Button Layout)](#3-gamepad-push-buttons-8-button-layout)
  - [Optional I2S DAC Audio (PCM5102) & PS/2 Keyboard](#4-optional-i2s-dac-audio-pcm5102--ps2-keyboard)
- [System Execution Flow](#-system-execution-flow)
- [Directory & File Structure](#-directory--file-structure)
- [Getting Started](#-getting-started)
  - [Prerequisites](#prerequisites)
  - [MicroSD Card Preparation](#microsd-card-preparation)
  - [Flashing Firmware via Arduino IDE](#flashing-firmware-via-arduino-ide)
- [Controls & Keymap](#-controls--keymap)
- [Future Enhancements](#-future-enhancements)
- [Uploading to GitHub](#-uploading-to-github)
- [License & Acknowledgments](#-license--acknowledgments)

---

## ⚡ Key Features

- **Real-Time 8-bit NES Emulation**: Accurate software simulation of the MOS Technology 6502 CPU instruction set (all 56 opcodes, official addressing modes) and cycle-accurate interrupt handling (NMI, IRQ, RESET).
- **Scanline-Based PPU Rendering**: Full graphics pipeline supporting pattern tables (CHR ROM/RAM), name & attribute tables (VRAM), 64-color master palette, and up to 64 hardware sprites (8 per scanline) with Sprite 0 hit detection.
- **Hardware-Accelerated Display**: Driven over SPI at high clock frequencies (40–80 MHz) using ESP32 **SPI Direct Memory Access (DMA)** to offload CPU rendering pipelines and eliminate frame tearing.
- **On-the-Fly ROM Browser**: Reads and lists all `.nes` game files placed in the `/NES/` directory on a FAT32-formatted MicroSD card via SPI.
- **iNES Format Parsing**: Automatically extracts PRG-ROM (program code), CHR-ROM (graphics tiles), mapper IDs, and mirroring configuration.
- **Wide Mapper Compatibility**: Supports Mapper 0 (NROM), Mapper 1 (MMC1), Mapper 2 (UxROM), Mapper 3 (CNROM), Mapper 4 (MMC3), Mapper 7 (AxROM), and dozens of extended ASIC mappers.
- **Multiple Input Methods**:
  - Dedicated 8-button tactile gamepad (D-Pad: UP/DOWN/LEFT/RIGHT, Action: A/B, Functions: START/SELECT).
  - PS/2 Keyboard interface via level-shifted GPIO.
  - Experimental Bluetooth gamepad connectivity (DualShock / HID server).
- **Audio Output Ready**: Integrated I2S digital audio generation compatible with PCM5102 DAC modules.
- **Composite Video Support**: Optional PAL composite TV output via ESP32 DAC channel (GPIO25/26).

---

## 📹 Demonstration

A video demonstration is included in this repository:
- **Video File**: [`live_demo.mp4`](live_demo.mp4) showcasing hardware boot, SD card ROM menu navigation, and real-time NES gameplay on the ST7789 display.

---

## 🧠 System Architecture

```mermaid
flowchart TD
    subgraph Storage ["Storage (SPI)"]
        SD["MicroSD Card (FAT32)"] -->|Load .nes ROM| VFS["Virtual File System / iNES Parser"]
    end

    subgraph EmulationCore ["ESP32 Dual-Core Xtensa LX6 (240MHz)"]
        VFS -->|PRG Code| CPU["MOS 6502 CPU Core<br/>56 Opcodes | 64KB Address Space"]
        VFS -->|CHR Tiles / Mapper| PPU["NES PPU (Picture Processing Unit)<br/>VRAM | OAM | Palettes | Scanline Renderer"]
        CPU <-->|Bus Sync (~1:3 Ratio)| PPU
        PPU -->|RGB565 Scanlines| FB["Frame Buffer (256x240 / 240x240)"]
    end

    subgraph Peripherals ["Peripherals & Output"]
        FB -->|SPI DMA (40-80MHz)| TFT["ST7789 1.3' TFT Display (240x240)"]
        INPUT["8 Tactile Buttons / PS2 / BT"] -->|GPIO / Poll / ISR| CPU
        CPU -->|I2S Audio Pipeline| DAC["PCM5102 I2S DAC / PAM8403 Amplifier"]
    end
```

### 1. MOS 6502 CPU Emulation
- **Registers Emulated**: Accumulator (`A`), Index Registers (`X`, `Y`), Program Counter (`PC`), Stack Pointer (`S` mapped to `$0100–$01FF`), and Processor Status (`P` flags: Carry, Zero, Interrupt Disable, Decimal, Break, Overflow, Negative).
- **Memory Map Emulation**:
  - `$0000–$07FF`: 2KB Internal Work RAM (mirrored up to `$1FFF`).
  - `$2000–$2007`: PPU I/O registers (mirrored up to `$3FFF`).
  - `$4000–$4017`: APU and I/O registers (controller ports at `$4016` and `$4017`).
  - `$4020–$5FFF`: Expansion ROM.
  - `$6000–$7FFF`: Battery-backed Cartridge SRAM (Save RAM).
  - `$8000–$FFFF`: PRG-ROM bank-switched space.

### 2. NES PPU Emulation
- Simulates the NES Picture Processing Unit scanline by scanline:
  - **Pattern Tables ($0000–$1FFF)**: Two 4KB tables holding 8×8 pixel graphic tiles (2 bits per pixel).
  - **Nametables & Attribute Tables ($2000–$3EFF)**: 32×30 tile matrices with 2×2 tile palette attributes (horizontal/vertical mirroring supported).
  - **Palette RAM ($3F00–$3F1F)**: 32 bytes defining 4 background and 4 sprite palettes from a 64-color master palette.
  - **Object Attribute Memory (OAM)**: 256 bytes storing 64 sprites (Y-pos, Tile index, Attributes, X-pos) loaded via `$4014` OAMDMA.
  - **Synchronization**: Precise NMI generation upon vertical blanking (VBlank) interval and hardware Sprite 0 hit timing.

### 3. Display Driving via SPI DMA
- The emulated 256×240 pixel frame is scaled or cropped to the ST7789 display's 240×240 resolution in 16-bit **RGB565** format.
- Hardware SPI DMA transfers whole frame buffers or scanline bursts asynchronously, eliminating CPU blocking and delivering smooth gameplay at up to 60 FPS.

### 4. MicroSD File System & ROM Loading
- Standard FAT32 microSD cards are mounted over SPI.
- The emulator traverses `/NES/`, parses 16-byte iNES headers (`NES<EOF>`), checks PRG/CHR ROM dimensions, allocates memory, registers the matching mapper, and sets the 6502 reset vector.

### 5. Extensive Cartridge Mapper Support
Cartridge mappers extend memory banking and IRQ triggers beyond 32KB PRG / 8KB CHR limits:
- **Mapper 0 (NROM)**: *Super Mario Bros., Donkey Kong, Pac-Man*
- **Mapper 1 (MMC1)**: *The Legend of Zelda, Metroid, Mega Man 2*
- **Mapper 2 (UxROM)**: *Castlevania, Contra, Duck Tales*
- **Mapper 3 (CNROM)**: *Gradius, Cybernoid*
- **Mapper 4 (MMC3)**: *Super Mario Bros. 3, Kirby's Adventure, Mega Man 3-6*
- **Additional Mappers**: 5, 7, 8, 9, 11, 15, 16, 18, 19, 21-25, 32-34, 40-42, 46, 50, 64-66, 70, 73, 75, 78, 79, 85, 87, 93, 94, 99, 160, 229, 231.

---

## 🛠️ Hardware Specifications

| Component | Specification |
| :--- | :--- |
| **Microcontroller** | ESP32-WROOM-32 / ESP32-WROVER (Dual-core Xtensa LX6 @ 240 MHz) |
| **Internal Memory** | 520 KiB SRAM, 448 KiB ROM, 4MB / 8MB SPI Flash |
| **External Memory** | Optional 4MB / 8MB PSRAM (supported on WROVER modules) |
| **Display** | 1.3-inch ST7789 IPS TFT LCD (240×240 pixels, 65K RGB565 colors) |
| **Display Bus** | High-speed SPI with DMA (Clock: 40 MHz – 80 MHz, Mode 3) |
| **Storage** | MicroSD Card Reader module (SPI mode, FAT32 format) |
| **Input Devices** | 8 Tactile Push Buttons with 10kΩ pull-down resistors / PS/2 Keyboard |
| **Audio** | I2S Digital Audio (PCM5102 DAC) + PAM8403 Amplifier + 1W Speaker |
| **Power Supply** | 5V via USB Micro-B/Type-C or 3.7V LiPo Battery with TP4056 + 3.3V LDO |

---

## 💰 Bill of Materials (BOM)

*Indicative component costs from the project engineering report:*

| # | Component | HSN Code | Qty | Unit Price (₹) | Total (₹) |
| :-: | :--- | :---: | :-: | :---: | :---: |
| 1 | ESP-WROOM-32 Dev Board (v1.1) | 84733010 | 1 | 300.85 | 355.00 |
| 2 | MB102 Breadboard (830 Points) | 84799090 | 1 | 40.68 | 48.00 |
| 3 | 1.3" ST7789 SPI TFT Display (240x240) | 85249110 | 1 | 253.39 | 299.00 |
| 4 | MicroSD Card Reader Module (SPI) | 85439000 | 1 | 30.51 | 36.00 |
| 5 | 10kΩ Metal Film Resistors (1/4W) | 85331000 | 8 | ~0.83 | ~6.64 |
| 6 | Tactile Push Buttons | - | 8 | ~2.00 | ~16.00 |
| 7 | Breadboard Jumper Wires (M-M / M-F) | - | 1 Set | ~50.00 | 50.00 |
| 8 | MicroSD Card (8GB - 32GB FAT32) | - | 1 | ~200.00 | 200.00 |
| **Total** | **Estimated Hardware Subtotal** | | | | **~₹910.64** |

---

## 🔌 Wiring & Pinout Configuration

### 1. ST7789 TFT Display (SPI)

| Display Pin | Description | ESP32 GPIO (Project Doc) | ESP32 GPIO (`NesCat.ino`) | Notes |
| :--- | :--- | :---: | :---: | :--- |
| **VCC** | Power Supply (3.3V) | 3.3V | 3.3V | Connect to 3.3V rail |
| **GND** | Ground | GND | GND | Common ground |
| **SCL / SCK** | SPI Clock | **GPIO 18** | **GPIO 18** | High-speed SPI clock |
| **SDA / MOSI**| SPI Data (MOSI) | **GPIO 23** | **GPIO 23** | Shared SPI data line |
| **RES / RST** | Display Reset | **GPIO 4** | **GPIO 19** | Active LOW reset |
| **DC** | Data / Command Select | **GPIO 2** | **GPIO 5** | LOW = Cmd, HIGH = Data |
| **CS** | Chip Select | **GPIO 5** | Not tied (`-1`) / GND | Optional if dedicated |
| **BLK** | Backlight Control | 3.3V / PWM | 3.3V / Optional | Tie to 3.3V for full brightness |

### 2. MicroSD Card Module (SPI)

| SD Module Pin | Description | ESP32 GPIO (Project Doc) | ESP32 GPIO (`NesCat.ino`) | Notes |
| :--- | :--- | :---: | :---: | :--- |
| **VCC** | Power (3.3V / 5V) | 3.3V / 5V | 3.3V / 5V | Ensure 3.3V logic levels |
| **GND** | Ground | GND | GND | Common ground |
| **MISO** | SPI Master In Slave Out | **GPIO 19** | **GPIO 13** | SD Data Out |
| **MOSI** | SPI Master Out Slave In | **GPIO 23** | **GPIO 12** | SD Data In |
| **SCK** | SPI Clock | **GPIO 18** | **GPIO 14** | SPI Bus Clock |
| **CS** | SD Chip Select | **GPIO 13** | **GPIO 22** | Dedicated CS line |

### 3. Gamepad Push Buttons (8-Button Layout)

Connect each button between the ESP32 GPIO and **3.3V** (or GND with pull-ups). Use **10kΩ pull-down resistors** connected to GND to avoid floating inputs:

| NES Button | Action / Function | ESP32 GPIO (Project Doc) | ESP32 GPIO (`NesCat.ino`) |
| :--- | :--- | :---: | :---: |
| **UP** | D-Pad Direction Up | **GPIO 32** | **GPIO 2** |
| **DOWN** | D-Pad Direction Down | **GPIO 33** | **GPIO 35** |
| **LEFT** | D-Pad Direction Left | **GPIO 25** | **GPIO 39** |
| **RIGHT** | D-Pad Direction Right | **GPIO 26** | **GPIO 34** |
| **A** | Primary Action Button | **GPIO 27** | **GPIO 21** |
| **B** | Secondary Action Button | **GPIO 14** | **GPIO 15** |
| **START** | Start / Pause / ROM Select | **GPIO 12** | **GPIO 25** |
| **SELECT** | Select / Options | **GPIO 15** | **GPIO 36** |

> **Note:** Button mappings are fully customizable in software via macro definitions at the top of `NesCat.ino`.

### 4. Optional I2S DAC Audio (PCM5102) & PS/2 Keyboard

| Peripheral | Signal | ESP32 Pin | Description |
| :--- | :--- | :---: | :--- |
| **I2S Audio** | **BCK** (Bit Clock) | **GPIO 26** | PCM5102 BCK |
| **I2S Audio** | **WS / LCK** (Word Select) | **GPIO 32** | PCM5102 LCK |
| **I2S Audio** | **DO / DIN** (Data Out) | **GPIO 27** | PCM5102 DIN |
| **PS/2 Keyboard** | **DATA** (-D) | **GPIO 4** | Via 1kΩ protective resistor |
| **PS/2 Keyboard** | **CLK** (+D) | **GPIO 0** | Via 1kΩ protective resistor |
| **Composite Video** | **VIDEO_OUT** | **GPIO 25 / 26**| ESP32 DAC Channel 2 PAL Out |

---

## 🔄 System Execution Flow

```
+-------------------------------------------------------------+
| 1. Power On / Hardware Boot                                  |
|    - Xtensa core clocks set to 240 MHz                      |
|    - Initialize Serial debugging, GPIOs, and SPI Bus        |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
| 2. Peripherals Initialization                                |
|    - ST7789 TFT initial commands & clear display            |
|    - Mount FAT32 File System on MicroSD Card                |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
| 3. ROM Selection Menu                                       |
|    - Scan directory '/NES/' for all '.nes' files            |
|    - Render list onto ST7789 Display                        |
|    - Navigate via UP/DOWN; Confirm with A or START          |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
| 4. Load & Parse ROM                                         |
|    - Open selected file, parse 16-byte iNES header          |
|    - Allocate SRAM / PSRAM buffers for PRG-ROM & CHR-ROM    |
|    - Instantiate corresponding Cartridge Mapper             |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
| 5. Initialize Emulation Core                                |
|    - Reset 6502 CPU registers and read Reset Vector         |
|    - Initialize PPU registers, VRAM, and clear OAM          |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
| 6. Emulation Execution Loop (Repeat each frame)             |
|    a. Poll GPIO / Keyboard inputs into $4016/$4017 bytes    |
|    b. Step CPU instructions & sync PPU cycles (~1:3 ratio)  |
|    c. Render scanlines, detect Sprite 0 hit, trigger NMI    |
|    d. At VBlank: dispatch frame buffer via SPI DMA          |
|    e. Check Hotkey (START + SELECT) -> return to ROM menu   |
+-------------------------------------------------------------+
```

---

## 📁 Directory & File Structure

```
ESP-32-NES-Emulator/
├── Adafruit_BusIO/               # Hardware I2C/SPI abstraction library
├── Adafruit_GFX_Library/         # Core 2D graphics primitive library
├── SdFat-MODIFIED_config/        # Optimized high-speed SD SPI FAT driver
├── SdFat-MODIFIED_config.zip     # Compressed archive of custom SdFat library
├── debug.h                       # Diagnostic logging and serial debug macros
├── hci_server.cpp / .h           # Bluetooth HCI transport implementation
├── hci_transport_esp32.cpp / .h  # Low-level ESP32 HCI UART/controller bindings
├── hid_server.cpp / .h           # Bluetooth HID gamepad driver
├── mappers.h                     # Extensive NES cartridge memory mappers (0-231)
├── NesCat.ino                    # Primary firmware sketch, CPU/PPU loops, main flow
├── PrintHex8.h                   # Helper functions for formatted hex serial output
├── Ps2KbdBle.ino                 # Secondary sketch for PS2/Bluetooth keyboard test
├── PS2Keyboard.cpp / .h          # Scancode decoder for physical PS/2 keyboards
├── Retro8x16.c                   # 8x16 bitmap retro font for UI & ROM menu
├── video.h                       # Video output configurations & PAL composite driver
├── live_demo.mp4                 # Video demonstration of the emulator in action
├── ESP32 NES Retro Game Console Project Document.pdf  # Comprehensive project report
├── .gitignore                    # Git tracking ignore file
└── README.md                     # Project documentation
```

---

## 🚀 Getting Started

### Prerequisites

1. **Hardware**:
   - ESP32 Development Board (ESP32-WROOM-32 or ESP32-WROVER recommended).
   - 1.3" ST7789 SPI TFT display (240×240).
   - MicroSD SPI module + FAT32 formatted MicroSD card.
   - 8 tactile buttons + 8× 10kΩ resistors.
   - Micro-USB cable and jumper wires.

2. **Software**:
   - [Arduino IDE](https://www.arduino.cc/en/software) (version 1.8.19 or 2.x) or [ESP-IDF](https://docs.espressif.com/projects/esp-idf/).
   - **ESP32 Board Package**: Install via Arduino Board Manager (`esp32` by Espressif).

### MicroSD Card Preparation

1. Format your MicroSD card (up to 32GB) with **FAT32**.
2. Create a root directory named `NES`:
   ```
   [SD Card Root]/
   └── NES/
       ├── SuperMarioBros.nes
       ├── Contra.nes
       ├── Pacman.nes
       └── LegendOfZelda.nes
   ```
3. Place valid `.nes` ROM files inside the `/NES/` folder.

### Flashing Firmware via Arduino IDE

1. Clone or download this repository.
2. In Arduino IDE, go to **File -> Preferences** and ensure the ESP32 board manager URL is added:
   ```
   https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
   ```
3. Install or copy the libraries inside this repo (`Adafruit_GFX_Library`, `Adafruit_BusIO`, and `SdFat-MODIFIED_config`) into your Arduino `libraries/` directory.
4. Open [NesCat.ino](file:///c:/Users/shelk/Music/ESP%2032%20NES%20Emulator/NesCat.ino) in Arduino IDE.
5. Configure board settings under **Tools**:
   - **Board**: `ESP32 Dev Module`
   - **CPU Frequency**: `240MHz (WiFi/BT)`
   - **Flash Frequency**: `80MHz`
   - **Flash Mode**: `QIO`
   - **Partition Scheme**: `Huge APP (3MB No OTA / 1MB SPIFFS)` or `Minimal SPIFFS`
   - **PSRAM**: `Enabled` (if using ESP32-WROVER)
   - **Upload Speed**: `921600`
6. Select your COM port and click **Upload**.

---

## 🎮 Controls & Keymap

### Tactile Gamepad Controls

| Hardware Button | Menu Navigation | NES Gameplay Action |
| :---: | :---: | :---: |
| **UP** | Scroll Up | D-Pad Up |
| **DOWN** | Scroll Down | D-Pad Down |
| **LEFT** | Previous Page | D-Pad Left |
| **RIGHT** | Next Page | D-Pad Right |
| **A** | Select / Launch Game | NES A Button (Jump / Primary) |
| **B** | Back / Cancel | NES B Button (Attack / Secondary) |
| **START** | Select / Launch Game | NES Start (Pause / Start Game) |
| **SELECT** | — | NES Select |
| **START + SELECT** | — | **Exit Game & Return to ROM Menu** |

### PS/2 Keyboard Mapping (When Enabled)

| Keyboard Key | NES Function |
| :---: | :---: |
| `W` / `Up Arrow` | D-Pad UP |
| `S` / `Down Arrow` | D-Pad DOWN |
| `A` / `Left Arrow` | D-Pad LEFT |
| `D` / `Right Arrow` | D-Pad RIGHT |
| `K` / `Z` | A Button |
| `J` / `X` | B Button |
| `Enter` | START |
| `Space` / `Right Shift` | SELECT |
| `Escape` | Reset / Menu |

---

## 🔮 Future Enhancements

- [ ] **I2S APU Audio Synthesis**: Full hardware emulation of NES pulse waves, triangle wave, noise channel, and DMC via I2S DAC.
- [ ] **Real-Time Save States**: Freeze and restore CPU/PPU/RAM registers directly to SD card files during gameplay.
- [ ] **3D-Printed Custom Handheld Enclosure**: Ergonomic housing accommodating an integrated 18650 Li-Ion battery, TP4056 USB-C charging module, speaker, and controls.
- [ ] **Custom PCB Design**: Transition from breadboard to a compact dual-layer PCB handheld console.
- [ ] **Wi-Fi & IoT Features**: Over-the-air (OTA) ROM upload through a local Web UI and Bluetooth wireless multiplayer.

---

## 📤 Uploading to GitHub

To push this repository to your personal GitHub account, follow these simple steps in your terminal or PowerShell:

```bash
# 1. Initialize git in the project root
git init

# 2. Stage all project files (including .gitignore and README.md)
git add .

# 3. Create initial commit
git commit -m "Initial commit: ESP32 NES Retro Game Console complete project"

# 4. Rename default branch to main
git branch -M main

# 5. Add your GitHub remote repository (replace with your GitHub URL)
git remote add origin https://github.com/<your-username>/<your-repo-name>.git

# 6. Push firmware and documentation to GitHub
git push -u origin main
```

---

## 📜 License & Acknowledgments

- **License**: Released under the [GNU General Public License v3.0](https://opensource.org/licenses/GPL-3.0).
- **Core Engine**: Based on `NesCat` by Nathalis (v0.5), adapted and enhanced for modern ESP32 architectures.
- **Academic Mentor & Department**: Department of Electronics and Communication Engineering, **Visvesvaraya National Institute of Technology (VNIT), Nagpur**.
- Special thanks to the open-source emulation and ESP32 developer communities.
