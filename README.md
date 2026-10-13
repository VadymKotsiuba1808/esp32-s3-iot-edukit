# ESP32-S3 IoT EduKit

The ESP32-S3 IoT EduKit is an educational development board built around the **ESP32-S3-WROOM-1 (N16R8)** module. It combines Wi-Fi/Bluetooth LE connectivity with wired Ethernet, a color LCD, a camera connector, I²S audio input and output, environmental and power sensing, and storage on a single board. It is intended for learning embedded programming, IoT networking, sensor data acquisition, and audio/vision experiments.

> **Status:** this repository currently contains the board's electrical schematic and this README. Firmware examples, PCB layout files, and a bill of materials are not included yet.

## Contents

- [Hardware Overview](#hardware-overview)
- [Hardware Documentation](#hardware-documentation)
- [Pin Mapping](#pin-mapping)
- [I²C Devices](#i2c-devices)
- [Power Architecture](#power-architecture)
- [Getting Started](#getting-started)
- [Development Resources](#development-resources)
- [Contributing](#contributing)
- [License](#license)

## Hardware Overview

All components below are taken from the schematic (revision 1.0, dated 2025-07-22).

| Function | Component | Ref. | Interface |
| --- | --- | --- | --- |
| Main controller | ESP32-S3-WROOM-1 (N16R8): 16 MB flash, 8 MB octal PSRAM | U3 | — |
| Display | 1.77-inch SPI LCD (N177-1216TCWPG01-H14) | LCD1 | SPI |
| Camera | 24-pin FPC connector for an OV2640 module (AFC01-S24FCA-00) | FPC1 | 8-bit DVP + SCCB (I²C) |
| Ethernet | W5500 controller with HR911105A RJ45 jack (integrated magnetics) | U1, J3 | SPI |
| Environmental sensor | BME280 (temperature, humidity, pressure) | U16 | I²C |
| Current/power monitor | INA219 with 0.1 Ω shunt on the battery path | U7 | I²C |
| Microphone | INMP441 I²S MEMS microphone | U8 | I²S |
| Audio amplifier | MAX98357A I²S class-D amplifier, 8 Ω 1 W speaker output | U14, SPK1 | I²S |
| Storage | microSD card socket | U12 | SPI |
| RGB LED | WS2812B addressable LED | LED9 | 1-wire |
| GPIO expander | TCA9535 16-bit I²C GPIO expander | U4 | I²C |
| USB (native) | USB-C connector wired to ESP32-S3 USB D+/D− | J1 | USB |
| USB-to-UART | USB-C connector with CP2102N bridge and DTR/RTS auto-programming circuit | J2, U6 | UART0 |
| Battery charging | MCP73812 Li-ion/Li-Po charger, 2-pin battery connector | U5, J10 | — |
| Voltage regulation | 4 × SPX3819 3.3 V LDO, XC6206 1.2 V and 2.8 V LDOs for the camera | U10, U13, U15, U17, U9, U11 | — |
| Buttons | Reset, Boot (IO0), two user buttons, power-control button | KEY1–KEY5 | — |
| Touch input | Capacitive touch pad | J11 | Touch GPIO |
| LEDs | 3 power LEDs, 1 status LED, 4 user LEDs | LED1–LED8 | — |
| Expansion headers | SPI, I²C, UART, GPIO, power, and GND headers | J4–J9 | — |

## Hardware Documentation

[Download Hardware Schematic](hardware/ESP32-S3-IoT-EduKit-Schematic.pdf)

The schematic is the authoritative source for pin assignments, power nets, and component values. If this README and the schematic disagree, the schematic takes precedence.

## Pin Mapping

### ESP32-S3 GPIO assignments

| GPIO | Net name | Function |
| --- | --- | --- |
| EN | `RST` | Chip enable/reset; shared with LCD `RES`, W5500 `RSTn`, and camera `RESET` |
| IO0 | `IO0` | Boot strapping pin, Boot button (KEY2), auto-programming circuit |
| IO1 | `SCL` | I²C clock (4.7 kΩ pull-up) |
| IO2 | `SDA` | I²C data (4.7 kΩ pull-up) |
| IO3 | `ST_LED` | Status LED (LED4) |
| IO4 | `TOUCH_BTN` | Capacitive touch pad (J11) |
| IO5 | `I2S_SCK` | I²S bit clock (microphone and amplifier) |
| IO6 | `I2S_WS` | I²S word select (microphone and amplifier) |
| IO7 | `I2S_MIC` | I²S data in from INMP441 |
| IO15 | `I2S_SPK` | I²S data out to MAX98357A |
| IO16 | `LCD_DC` | LCD data/command (`A0`) |
| IO17 | `SPI_SCK` | Shared SPI clock |
| IO18 | `SPI_MISO` | Shared SPI MISO |
| IO8 | `SPI_MOSI` | Shared SPI MOSI |
| IO19 | `ESP_DN` | Native USB D− (J1) |
| IO20 | `ESP_DP` | Native USB D+ (J1) |
| IO43 (TXD0) | `TX0` | UART0 TX to CP2102N and UART header J9 |
| IO44 (RXD0) | `RX0` | UART0 RX from CP2102N and UART header J9 |
| IO42 | `RGB_NEO` | WS2812B data in |
| IO41 | `EP_INT` | TCA9535 interrupt output (`INT#`, 10 kΩ pull-up) |
| IO48 | `CSI_D0` | Camera Y2 |
| IO39 | `CSI_D1` | Camera Y3 |
| IO40 | `CSI_D2` | Camera Y4 |
| IO38 | `CSI_D3` | Camera Y5 |
| IO47 | `CSI_D4` | Camera Y6 |
| IO14 | `CSI_D5` | Camera Y7 |
| IO13 | `CSI_D6` | Camera Y8 |
| IO11 | `CSI_D7` | Camera Y9 |
| IO21 | `CSI_PCLK` | Camera pixel clock |
| IO12 | `CSI_MCLK` | Camera master clock (`XCLK`) |
| IO9 | `CSI_VSYNC` | Camera VSYNC |
| IO10 | `CSI_HSYNC` | Camera HREF |

Not connected in the schematic: IO35, IO36, IO37 (used internally by the octal PSRAM on N16R8 modules), IO45, IO46.

### Signals routed through the TCA9535 expander

Several chip-select, interrupt, and user I/O signals are **not** connected to ESP32-S3 pins. They are driven or read through the TCA9535 over I²C, so firmware must configure the expander before using these peripherals. For example, the LCD driver must toggle `LCD_CS` and `LCD_LED` through the expander rather than through a native GPIO.

| Expander pin | Net name | Function |
| --- | --- | --- |
| P00 | `uSD_CS` | microSD chip select |
| P01 | `LCD_CS` | LCD chip select |
| P02 | `ETH_CS` | W5500 chip select (`SCSn`) |
| P03 | `PWR_CTRL` | Power-switching circuit control |
| P04 | `LCD_LED` | LCD backlight enable (via BC847 transistor Q1) |
| P05 | `UCS_1` | Chip select 1 on SPI header J4 |
| P06 | `UCS_2` | Chip select 2 on SPI header J4 |
| P07 | `ETH_INT` | W5500 interrupt (`INTn`, 10 kΩ pull-up) |
| P10 | `ULED_1` | User LED 1 (LED5) |
| P11 | `ULED_2` | User LED 2 (LED6) |
| P12 | `ULED_3` | User LED 3 (LED7) |
| P13 | `ULED_4` | User LED 4 (LED8) |
| P14 | `UGPIO_1` | GPIO header J8 |
| P15 | `UGPIO_2` | GPIO header J8 |
| P16 | `BTN1` | User button KEY3 (10 kΩ pull-up) |
| P17 | `BTN2` | User button KEY4 (10 kΩ pull-up) |

### Shared buses

| Bus | Devices |
| --- | --- |
| SPI (IO17/IO18/IO8) | LCD, W5500, microSD, SPI header J4 |
| I²C (IO1/IO2) | TCA9535, BME280, INA219, camera SCCB, I²C header J6 |
| I²S (IO5/IO6) | INMP441 (data on IO7), MAX98357A (data on IO15) |

<a id="i2c-devices"></a>

## I²C Devices

Addresses as listed in the schematic's I²C address table (7-bit unless noted).

| Ref. | Device | Address | Notes |
| --- | --- | --- | --- |
| U4 | TCA9535 GPIO expander | `0x20` | A0–A2 tied to GND |
| U7 | INA219 current/power monitor | `0x40` | |
| U16 | BME280 environmental sensor | `0x76` | |
| FPC1 | OV2640 camera (SCCB) | `0x60` write / `0x61` read | 8-bit form as given in the schematic; equals 7-bit `0x30` |

## Power Architecture

| Rail | Source | Supplies |
| --- | --- | --- |
| `VCC_5V` | USB VBUS from J1 or J2 (through DSS34 Schottky diodes) | Charger, power header J7 |
| `V_BAT` | Battery on J10, charged by MCP73812 | Power switching, INA219 shunt |
| `V_REG` | `VCC_5V` or `V_BAT` via power-switching circuit (AO3401A, KEY5, `PWR_CTRL`) | All 3.3 V LDOs, WS2812B |
| `V_MCU` | SPX3819 3.3 V (U10) | ESP32-S3, TCA9535, W5500 (`V_ETH` through ferrite bead) |
| `V_TFT` | SPX3819 3.3 V (U13) | LCD |
| `V_OTH` | SPX3819 3.3 V (U15) | BME280, microSD, camera I/O, 1.2 V/2.8 V camera LDOs |
| `V_AUDIO` | SPX3819 3.3 V (U17) | INMP441, MAX98357A |
| `CSI_1.2V`, `CSI_2.8V` | XC6206 (U9, U11) | Camera core and analog supplies |

## Getting Started

Requirements:

- A USB-C cable and a computer with a supported OS (Windows, Linux, or macOS).
- [ESP-IDF](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/get-started/index.html) installed with ESP32-S3 as the target (`idf.py set-target esp32s3`).
- For the CP2102N port (J2): the [Silicon Labs CP210x VCP driver](https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers) if your OS does not include it.

Connecting the board:

- **J2 (CP2102N):** provides UART0 as a serial port. The DTR/RTS auto-programming circuit drives `EN` and `IO0`, so standard ESP-IDF tooling can enter download mode without pressing buttons.
- **J1 (native USB):** connects directly to the ESP32-S3 USB peripheral (IO19/IO20).
- **Manual download mode:** hold the Boot button (IO0), press and release Reset, then release Boot.

Before writing firmware, check the [Pin Mapping](#pin-mapping): the LCD, microSD, and W5500 chip selects require the TCA9535 to be initialized first.

No firmware examples are provided in this repository yet, and the steps above have not been verified on physical hardware as part of this documentation.

## Development Resources

- [ESP32-S3 Datasheet](https://www.espressif.com/sites/default/files/documentation/esp32-s3_datasheet_en.pdf)
- [ESP32-S3-WROOM-1 Datasheet](https://www.espressif.com/sites/default/files/documentation/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf)
- [ESP32-S3 Technical Reference Manual](https://www.espressif.com/sites/default/files/documentation/esp32-s3_technical_reference_manual_en.pdf)
- [ESP-IDF Programming Guide (ESP32-S3)](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/)
- [esp32-camera driver](https://github.com/espressif/esp32-camera)
- [W5500 documentation (WIZnet)](https://docs.wiznet.io/Product/Chip/Ethernet/W5500)
- [TCA9535 datasheet (Texas Instruments)](https://www.ti.com/product/TCA9535)
- [INA219 datasheet (Texas Instruments)](https://www.ti.com/product/INA219)
- [BME280 datasheet (Bosch Sensortec)](https://www.bosch-sensortec.com/products/environmental-sensors/humidity-sensors-bme280/)
- [MAX98357A datasheet (Analog Devices)](https://www.analog.com/en/products/max98357a.html)

## Contributing

Contributions are welcome through pull requests:

- **Hardware documentation:** cite the schematic sheet or reference designator for every pin or value you add or change.
- **Examples:** state the ESP-IDF version, the peripherals used, and whether the example was tested on real hardware.
- **Corrections:** open an issue or pull request describing the discrepancy and its source.

Keep one logical change per pull request.

## License

This repository does not currently include a license. Until the maintainer adds one, all rights are reserved by the author; do not assume the schematic or documentation may be redistributed under an open-source license.

Schematic author: VadymKotsiuba (as credited in the schematic title block).
