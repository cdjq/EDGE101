# EDGE101 (DFR0886) ESP32 IoT Programmable Controller — ESP-IDF Development Reference

> **This document is written for AI agents (and developers)**: after reading it, you should be able — without consulting any other material — to select peripherals, confirm pins, pick drivers, and write a complete, buildable, flashable, runnable application firmware for this board based on **ESP-IDF**.
>
> ESP-IDF only (C/C++, FreeRTOS). Arduino is not covered. All pins and electrical parameters have been verified against the official board support package sources (`edge101_sdk/1.0.1/variants/esp32/pins_arduino.h`, `libraries/Ethernet/src/ETH.h`, etc.) and the product Wiki; treat them as authoritative.
>
> **Version note**: Chapters 1–7 are written against ESP-IDF v5.x as the baseline. **ESP-IDF v6 adaptation (component changes, driver migrations, verified results) is covered in Chapter 8**, validated by building the reference project with v6.1.1.
>
> This repository contains a complete ESP-IDF project already validated on this board: `wind_logger/` (RS485 Modbus acquisition + RTC + SD card + Wi-Fi SoftAP + HTTP server). Use it as an application skeleton reference.

---

## Contents

1. Hardware Platform Overview
2. Hardware Features and Industrial Qualifications
3. On-board Resource List
4. Determining Pins: GPIO Assignment Table
5. Determining Drivers: Peripheral → ESP-IDF Driver Selection
6. Caveats (Pitfall Checklist)
7. Writing an Application: Project Structure and Configuration
8. ESP-IDF v6 Adaptation Guide
9. Feasibility Quick Reference for Typical Combinations
10. Reference Index

---

## 1. Hardware Platform Overview

The EDGE101 (DFRobot SKU: DFR0886) is an industrial-grade IoT programmable controller built around the classic ESP32 (not S3/C3/C6). Inside a metal enclosure it integrates the peripherals needed for field deployment — RS485, CAN, Ethernet, RTC, TF card — and supports 4G/NB-IoT expansion via a PCIe-sized slot.

| Item | Specification |
|---|---|
| MCU | ESP32 (Xtensa 32-bit LX6 dual-core, 240 MHz, 600 MIPS) |
| Flash | 16 MB |
| RAM | 520 KB SRAM + 16 KB RTC SRAM, **no PSRAM** |
| Wireless | 2.4 GHz Wi-Fi 802.11 b/g/n (150 Mbps, STA/AP/sniffer); Bluetooth 4.2 BR/EDR + BLE / BLE Mesh |
| Wired | 1× Ethernet RJ45 10/100 Mbps, 1× isolated RS485, 1× isolated CAN (≤1 Mbps) |
| Expansion | 11 native GPIO (Gravity 3P), 3× Gravity I2C, 1× PCIe-sized slot (4G/NB-IoT, e.g. Quectel EG800AK), 1× TF card slot (≤32 GB), 1× Nano SIM slot |
| HMI | 1× reset button, 1× user button, 3× LEDs (power / user-defined / wireless status) |
| USB | Type-C, CH340 USB-UART (VID:PID = 1a86:55d4), for flashing and debugging |
| RTC | On-board PCF8563T (I2C address 0x51) |
| Power | DC 9–26 V terminal input, or USB Type-C 5 V/2 A |
| Enclosure | Metal, 136.7×76×31 mm, 335 g assembled; DIN-rail / wall-ear / flat / vertical mounting |

---

## 2. Hardware Features and Industrial Qualifications

The board is designed to industrial standards and can be deployed directly in the field (electrical rooms, greenhouses, warehouses, rooftops) — it is not a desk-only dev board. You may rely on the following facts when selecting parts and writing drivers:

| Qualification / Feature | Specification | Implication for firmware development |
|---|---|---|
| Power supply | DC 9–26 V wide input, or USB Type-C 5 V/2 A | Can share one 12/24 V supply with RS485 sensors on site |
| Lightning surge protection | 2 kV | Suitable for outdoor long-wire scenarios (RS485/power) |
| ESD protection | 6 kV contact discharge | — |
| Power protection | Reverse-polarity and over-voltage protection | Reversed terminals will not damage the board |
| Communication isolation | Both RS485 and CAN have isolated transceivers | No software handling of ground-potential differences on the bus |
| Termination resistors | 1× 120 Ω each for RS485 and CAN, switchable via DIP on the enclosure | Flip the switch when the board is a bus endpoint; no external resistor needed |
| Operating temperature | -20 ~ +75 ℃ | Suitable for outdoor cabinets |
| Operating humidity | 5–90 %RH (non-condensing) | — |
| Duty | 7×24 h continuous operation | Long-running firmware should enable the TWDT watchdog |
| Idle power | 50 mA @ 12 V terminal; 100 mA @ USB 5 V | Evaluate yourself for battery-powered scenarios |
| Enclosure | Metal, protects against touch, insects, animals | — |

**Antennas**: the front SMA connector carries the 2.4 GHz antenna (Wi-Fi/Bluetooth); a second external antenna position is for the 4G modem when installed.

---

## 3. On-board Resource List

| Resource | Qty | Key parameters | GPIOs involved |
|---|---|---|---|
| Ethernet (RJ45) | 1 | 10/100 Mbps, RMII, PHY TLK110/IP101 (address 1) | 0/2/4/13/19/21/22/25/26/27 |
| RS485 (isolated) | 1 | Half-duplex, recommended 9600/115200 bps, DE pin controlled | 16/17/36 |
| CAN (isolated) | 1 | CAN 2.0B (ESP32 built-in TWAI controller), ≤1 Mbps | 32/35 |
| Debug UART | 1 | CH340, 115200 8N1, flashing + logging | 1/3 |
| Gravity GPIO (3P header) | 11 | P5/P12/P14/P15/P18/P23/P33/P34/P37/P38/P39, input/output/ADC | see §4 |
| Gravity I2C | 3 | One shared bus, 100/400 kHz, sockets provide 3V3/5V | 18/23 |
| SD (TF) card slot | 1 | ≤32 GB, SPI mode | 5/12/14/39 |
| RTC | 1 | PCF8563T, I2C 0x51 | 18/23 (shared) |
| PCIe-sized slot | 1 | 4G/NB-IoT modems (EG800AK etc.), UART1 | 33/34 |
| Nano SIM slot | 1 | Wired directly to the 4G modem | — |
| User button | 1 | Pressed = low | 38 |
| Reset button | 1 | Hardware reset (EN) | — |
| User LED | 1 | Active low | 15 |
| Wireless-status LED / power LED | 1 each | Not programmable | — |
| 120 Ω termination | 2 | One each for RS485 and CAN, DIP-switchable | — |
| Earth terminal | 1 | On the back, connect to earth ground | — |

ESP32 resources that exist on-chip but are **not usable on this board**: DAC (GPIO25/26 are taken by Ethernet RMII), PSRAM (not fitted).

---

## 4. Determining Pins: GPIO Assignment Table

### 4.1 Complete pin table (all 40 GPIOs verified)

Pins marked "Exposed" are routed to the Gravity header on the back and are free to use; all other pins are already occupied by on-board peripherals — do not reuse them.

| GPIO | On-board function | Exposed | Notes |
|---|---|---|---|
| 0 | Ethernet RMII 50 MHz reference clock input (external oscillator) | No | Also the BOOT download strapping pin |
| 1 | UART0 TX → CH340 → USB Type-C | No | Debug console, 115200 |
| 2 | Ethernet PHY power/reset control | No | Boot strapping pin |
| 3 | UART0 RX ← CH340 | No | Debug console |
| 4 | Ethernet PHY MDC | No | SMI management interface |
| **5** | SPI CS (on-board SD card chip-select) | ✅ P5 | Internal pull-up; header SPI devices need a different CS |
| 6–11 | On-chip SPI Flash | No | **Forbidden** |
| **12** | SPI MOSI (header SPI-SDO) | ✅ P12 | ADC2_CH5; boot strapping pin (internal pull-down, **never hard pull up externally**) |
| 13 | Ethernet PHY MDIO | No | SMI management interface |
| **14** | SPI SCK (header SPI-CLK) | ✅ P14 | ADC2_CH6 |
| **15** | User LED (active low) | ✅ P15 | ADC2_CH3; boot strapping pin (MTDO) |
| 16 | RS485 direction control DE | No | High = transmit, low = receive; UART driver toggles it via RTS |
| 17 | RS485 TX (UART2 TXD) | No | To RS485 terminals via isolated transceiver |
| **18** | I2C SDA | ✅ P18 | Shared by all 3 Gravity I2C sockets; on-board RTC is on this bus |
| 19 | Ethernet RMII TXD0 | No | Fixed RMII pin |
| 21 | Ethernet RMII TX_EN | No | Fixed RMII pin |
| 22 | Ethernet RMII TXD1 | No | Fixed RMII pin |
| **23** | I2C SCL | ✅ P23 | Same bus as above |
| 25 | Ethernet RMII RXD0 | No | Therefore DAC1 is unavailable |
| 26 | Ethernet RMII RXD1 | No | Therefore DAC2 is unavailable |
| 27 | Ethernet RMII CRS_DV | No | Fixed RMII pin |
| 32 | CAN TX (TWAI CTX) | No | To CAN terminals via isolated transceiver |
| **33** | UART1 TX → PCIe slot (4G modem RXD) | ✅ P33 | Occupied when a 4G modem is installed |
| **34** | UART1 RX ← PCIe slot (4G modem TXD) | ✅ P34 | Input-only, no internal pull-up/down |
| 35 | CAN RX (TWAI CRX) | No | Input-only |
| 36 | RS485 RX (UART2 RXD) | No | Input-only |
| **37** | Free | ✅ P37 | Input-only, no internal pull-up/down; ADC1_CH1 |
| **38** | User button (pressed = low) | ✅ P38 | Input-only; ADC1_CH2 |
| **39** | SPI MISO (header SPI-SDI) | ✅ P39 | Input-only, no internal pull-up/down; ADC1_CH3 |

### 4.2 Exposed-pin capability quick reference

11 Gravity GPIOs: P5, P12, P14, P15, P18, P23, P33, P34, P37, P38, P39

| Capability | Usable pins |
|---|---|
| Push-pull output (digital/PWM/interrupt) | 5, 12, 14, 15, 18, 23, 33 |
| Input-only (no output driver) | 34, 37, 38, 39 |
| Internal pull-up/down | 5, 12 (pull-down), 14, 15, 18, 23, 33; 34/37/38/39 have **no** internal pulls — floating inputs need external resistors |
| ADC1 (Wi-Fi compatible, recommended) | 37 (CH1), 38 (CH2), 39 (CH3) |
| ADC2 (unavailable while Wi-Fi is active) | 12 (CH5), 14 (CH6), 15 (CH3) |
| RTC domain / deep-sleep wakeup | 33, 34, 37, 38, 39 |
| I2C (fixed) | SDA=18, SCL=23 |
| SPI (fixed, shared with SD card) | SCK=14, MOSI=12, MISO=39, CS=5 |
| UART1 (shared with 4G modem) | TX=33, RX=34 |

### 4.3 Decision procedure for pin assignment (for agents)

1. List every peripheral/signal your application needs.
2. Check §3: signals using on-board peripherals (RS485/CAN/Ethernet/SD/RTC/4G) have **fixed** pins — take them from the corresponding table in §5; they cannot be changed.
3. All remaining signals must be assigned to the 11 exposed GPIOs, filtered by the capability constraints in §4.2:
   - Need output → choose only from 5/12/14/15/18/23/33 (note 18/23 are also I2C);
   - Need ADC while using Wi-Fi → choose only from 37/38/39;
   - Buttons/open-drain inputs needing internal pull-ups → avoid 34/37/38/39.
4. Use the §9 quick-reference table to rule out conflicts (e.g. 4G modem vs. P33/34, SD card vs. header SPI CS).

---

## 5. Determining Drivers: Peripheral → ESP-IDF Driver Selection

> Default baseline is ESP-IDF v5.x; v4.x differences are noted. **ESP-IDF v6 driver migrations are consolidated in Chapter 8** (Ethernet PHY, TWAI, I2C etc. changed significantly in v6). The build target is always `esp32`.

### 5.1 USB debug UART (UART0)

- Hardware: CH340, TX=GPIO1, RX=GPIO3, 115200 8N1.
- Nothing to configure: default `CONFIG_ESP_CONSOLE_UART_NUM=0`; `idf.py flash monitor` works out of the box.
- If flashing fails (board does not enter download mode automatically): hold GPIO0 low manually (button next to reset) and reset, then run `idf.py flash` again (GPIO0 doubles as the Ethernet 50 MHz clock input, so the auto-reset circuit is occasionally unreliable).

### 5.2 RS485 (isolated) → UART driver (RS485 half-duplex mode)

| Signal | GPIO |
|---|---|
| TX | 17 |
| RX | 36 |
| DE (direction enable) | 16 (wired to UART RTS) |

Recommended 9600/115200. The ESP-IDF UART driver natively controls DE via RTS — no manual direction toggling:

```c
uart_config_t cfg = {
    .baud_rate = 115200,
    .data_bits = UART_DATA_8_BITS,
    .parity    = UART_PARITY_DISABLE,
    .stop_bits = UART_STOP_BITS_1,
    .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
};
uart_param_config(UART_NUM_2, &cfg);
uart_set_pin(UART_NUM_2, 17, 36, 16, UART_PIN_NO_CHANGE);  // TX, RX, RTS(=DE)
uart_driver_install(UART_NUM_2, 512, 512, 0, NULL, 0);
uart_set_mode(UART_NUM_2, UART_MODE_RS485_HALF_DUPLEX);
```

- Modbus RTU: use the `espressif/esp-modbus` component (freemodbus) with the same UART parameters.
- Termination: when the board is a bus endpoint, switch on the RS485 120 Ω DIP on the enclosure.
- This UART API is unchanged in v6 (only the `UART_FIFO_LEN` macro was renamed to `UART_HW_FIFO_LEN`); on v6, declare an explicit CMake dependency on `esp_driver_uart` (see §8.2).

### 5.3 CAN (isolated) → TWAI driver

| Signal | GPIO |
|---|---|
| CTX | 32 |
| CRX | 35 |

An isolated transceiver is already on board; software only drives the ESP32 built-in TWAI (CAN 2.0B) controller, up to 1 Mbps:

```c
#include "driver/twai.h"

twai_general_config_t g = TWAI_GENERAL_CONFIG_DEFAULT(GPIO_NUM_32, GPIO_NUM_35, TWAI_MODE_NORMAL);
twai_timing_config_t  t = TWAI_TIMING_CONFIG_500KBITS();   // macros exist for 12.5K~1M
twai_filter_config_t  f = TWAI_FILTER_CONFIG_ACCEPT_ALL();

ESP_ERROR_CHECK(twai_driver_install(&g, &t, &f));
ESP_ERROR_CHECK(twai_start());
```

> On v4.x this driver is `driver/can.h` (`can_driver_install`) with the same API shape. For CANopen, CANopenNode can be ported (ESP-IDF provides an example).
>
> **ESP-IDF v6**: the legacy driver above still works but raises a deprecation warning (suppressible via `CONFIG_TWAI_SUPPRESS_DEPRECATE_WARN`); Espressif recommends migrating to the new driver introduced in v5.5 (`esp_twai.h` + `esp_twai_onchip.h`, event-callback based) — see §8.4 for the new-style code. The old and new drivers **must not be mixed**.

### 5.4 Ethernet (RMII) → esp_eth + IP101/TLK110 PHY

| Signal | GPIO | Notes |
|---|---|---|
| PHY address | — | 1 (hardwired) |
| MDC | 4 | SMI clock |
| MDIO | 13 | SMI data |
| PHY power/reset | 2 | Active high |
| REF_CLK | 0 | **External 50 MHz oscillator input** (`EMAC_CLK_EXT_IN`) |
| TXD0/TXD1/TX_EN | 19/22/21 | Fixed RMII pins |
| RXD0/RXD1/CRS_DV | 25/26/27 | Fixed RMII pins |

```c
eth_mac_config_t mac_cfg = ETH_MAC_DEFAULT_CONFIG();
eth_esp32_emac_config_t emac = ETH_ESP32_EMAC_DEFAULT_CONFIG();
emac.smi_gpio.mdc_num  = 4;
emac.smi_gpio.mdio_num = 13;
emac.clock_config.rmii.clock_mode = EMAC_CLK_EXT_IN;   // external 50 MHz via GPIO0
emac.clock_config.rmii.clock_gpio = EMAC_CLK_IN_GPIO;  // GPIO0
esp_eth_mac_t *mac = esp_eth_mac_new_esp32(&emac, &mac_cfg);

eth_phy_config_t phy_cfg = ETH_PHY_DEFAULT_CONFIG();
phy_cfg.phy_addr       = 1;
phy_cfg.reset_gpio_num = 2;                            // on-board PHY power/reset pin
esp_eth_phy_t *phy = esp_eth_phy_new_ip101(&phy_cfg);  // v5 also offers esp_eth_phy_new_tlk110

esp_eth_handle_t eth = NULL;
esp_eth_config_t cfg = ETH_DEFAULT_CONFIG(mac, phy);
ESP_ERROR_CHECK(esp_eth_driver_install(&cfg, &eth));
ESP_ERROR_CHECK(esp_netif_attach(esp_netif_new(ESP_NETIF_DEFAULT_ETH()),
                                esp_eth_new_netif_glue(eth)));
```

- PHY model: the official board package defaults to `ETH_PHY_TLK110` and notes that the IP101 is register-compatible (`ETH_PHY_IP101 = ETH_PHY_TLK110`). On v5 the two correspond to `esp_eth_phy_new_tlk110` / `esp_eth_phy_new_ip101`; try either in practice.
- **ESP-IDF v6**: all PHY drivers were moved out of ESP-IDF (the TLK110 driver was dropped by Espressif). Run `idf.py add-dependency "espressif/ip101"` and include `esp_eth_phy_ip101.h`; for zero external dependencies, the built-in generic PHY `esp_eth_phy_new_generic()` is available. See §8.3.
- Configure the RMII clock in code as shown above (v6 removed the corresponding Kconfig options; code configuration is the only way).
- Wi-Fi and Ethernet can coexist (multiple netifs) — well suited to an "Ethernet primary, Wi-Fi provisioning fallback" product design.
- The RMII clock occupies GPIO0, so GPIO0 cannot be used for anything else.

### 5.5 SD card (TF slot) → SDSPI host + FatFS

| Signal | GPIO |
|---|---|
| CS | 5 |
| MOSI | 12 |
| MISO | 39 |
| SCK | 14 |

The SD card is wired in SPI mode (not SDMMC); in ESP-IDF this is the **SDSPI host**:

```c
spi_bus_config_t bus = {
    .mosi_io_num = 12, .miso_io_num = 39, .sclk_io_num = 14,
    .quadwp_io_num = -1, .quadhd_io_num = -1,
};
spi_bus_initialize(SPI2_HOST, &bus, SPI_DMA_CH_AUTO);

sdspi_device_config_t dev = SDSPI_DEVICE_CONFIG_DEFAULT();
dev.gpio_cs = 5;
dev.host_id = SPI2_HOST;

sdmmc_host_t host = SDSPI_HOST_DEFAULT();
esp_vfs_fat_sdmmc_mount("/sdcard", &host, &dev, &mount_cfg, &card);
```

- The header SPI and the SD card share **one bus**: external SPI devices must use a separate CS and re-apply bus parameters before transactions as needed.
- Enable long file names in menuconfig: `CONFIG_FATFS_LFN_HEAP=y`, `CONFIG_FATFS_MAX_LFN=255` (as `wind_logger` does).
- This usage is unchanged in v6 (on v6, declare explicit dependencies on `esp_driver_spi` and `esp_driver_sdspi`, see §8.2).

### 5.6 I2C bus and on-board RTC → i2c_master

| Signal | GPIO |
|---|---|
| SDA | 18 |
| SCL | 23 |

One bus routed to 3 Gravity sockets (providing 3V3/5V); the on-board RTC sits on the same bus:

- **RTC: PCF8563T, 7-bit address 0x51** (8-bit write 0xA2 / read 0xA3).
- Registers: 0x00/0x01 control/status, 0x02 seconds, 0x03 minutes, 0x04 hours, 0x05 days, 0x06 weekdays, 0x07 months, 0x08 years — all BCD-encoded.
- Since ESP-IDF v5 the recommended driver is `driver/i2c_master.h`; **the legacy `driver/i2c.h` is marked EOL in v6 (scheduled for removal in v7)** — new projects should always use `i2c_master`.
- A complete, working implementation to copy from: `wind_logger/main/rtc_pcf8563.c` (read/write, power-on time check; based on the legacy API, still compiles on v6 — migration advice in §8.5).
- Before attaching Gravity I2C sensors, scan the bus to confirm no address clash with 0x51.

### 5.7 PCIe-slot 4G modem (EG800AK) → UART1 / esp_modem

| Signal | GPIO |
|---|---|
| ESP32 TX → modem RXD | 33 |
| ESP32 RX ← modem TXD | 34 |

- Officially supports the Quectel **EG800AK** (4G Cat.1), pure AT-command interaction, default 115200 (auto-baud 4800–1M). The Nano SIM slot is wired directly to the modem.
- **P33/P34 are the same UART1 pair used by the 4G modem**: with a modem installed, those header pins cannot be used for anything else.
- Two driver routes:
  - Simple scenarios (SMS, signal query, modem-built-in protocol stacks): `uart_driver_install(UART_NUM_1, ...)` + `uart_set_pin(UART_NUM_1, 33, 34, -1, -1)`, implement your own AT state machine.
  - PPP dial-up Internet: the `espressif/esp_modem` component; DTE on UART1, pins 33/34, flow control off.
- The modem has no dedicated power-control GPIO; it runs as soon as powered.

### 5.8 Button and LEDs → GPIO / LEDC

| Device | GPIO | Active level |
|---|---|---|
| User LED | 15 | **Active low** |
| User button | 38 | **Pressed = low** (input-only, external circuit on board) |
| Reset button | EN | Hardware reset |

- The LED can be a PWM breathing light via LEDC; example parameters 5 kHz / 13-bit resolution.
- Debounce the button with 20–50 ms software debouncing or an `esp_timer` scan.

### 5.9 Wi-Fi / Bluetooth → esp_wifi / esp_bt

Standard ESP32 wireless, no board-specific handling:

- Wi-Fi: STA / SoftAP / sniffer, 2.4G b/g/n. SoftAP reference: `wind_logger/main/wifi_ap.c` (open AP, 192.168.4.1).
- Bluetooth: BT 4.2 (BR/EDR) + BLE; stacks: Bluedroid or NimBLE; BLE Mesh supported.
- **While Wi-Fi is active, ADC2 (P12/P14/P15) cannot do analog reads** — use ADC1 (P37/P38/P39) instead.

### 5.10 Other on-chip peripherals

| Peripheral | ESP-IDF driver | Availability on this board |
|---|---|---|
| PWM | `driver/ledc.h` (16 channels) | Output-capable pins: 5/12/14/15/18/23/33 |
| ADC | `esp_adc/adc_oneshot.h`, `adc_continuous.h` | See §4.2; ADC1 recommended |
| High-resolution timer | `esp_timer` | Available |
| Hardware timer groups | `driver/gptimer.h` | Available |
| Watchdog | `esp_task_wdt.h` (TWDT) | Recommended for 7×24 applications |
| Pulse counter | `driver/pulse_cnt.h` (PCNT) | Available |
| DAC | — | **Unavailable** (GPIO25/26 taken by Ethernet) |
| Deep sleep / RTC wakeup | `esp_sleep.h` | RTC GPIOs 33/34/37/38/39 available |

> v6 notes: the legacy ADC (`driver/adc.h`), timer, PCNT, RMT and DAC drivers were all removed in v6 — every entry above already names the new API. The deep-sleep GPIO wakeup API `gpio_deep_sleep_wakeup_enable()` was renamed to `gpio_wakeup_enable_on_hp_periph_powerdown_sleep()` in v6. See §8.6.

---

## 6. Caveats (Pitfall Checklist)

Ordered by likelihood of being hit:

1. **ADC2 and Wi-Fi are mutually exclusive**: P12/P14/P15 cannot sample analog values while Wi-Fi runs. Always prefer ADC1 (P37/P38/P39) for analog acquisition.
2. **GPIO12 boot strapping pin**: its internal pull-down guarantees 3.3 V flash operation — **attached devices must never hard pull it up**, or the board will not boot.
3. **GPIO0 is occupied**: it is the Ethernet 50 MHz clock input; do not reuse it. For flashing failures, enter download mode manually per §5.1.
4. **Input-only pins have no internal pulls**: floating inputs on P34/P37/P38/P39 need external resistors, or readings will drift.
5. **P33/P34 and the 4G modem are mutually exclusive**: with the modem installed those two ports are unavailable, and vice versa.
6. **Header SPI shares the bus with the SD card**: external SPI devices need their own CS (any spare output pin works), with attention to bus arbitration.
7. **There is only one I2C bus**: the 3 Gravity sockets plus the on-board RTC (0x51) share SDA=18/SCL=23. Multiple I2C devices must have unique addresses; if you repurpose 18/23 as plain GPIO, the RTC is lost.
8. **Do not toggle the RS485 DE pin manually**: leave it to the UART driver's RS485 half-duplex mode (automatic RTS control); calling `gpio_set_level(16, ...)` yourself conflicts with the driver.
9. **No DAC**: for analog output, approximate with LEDC PWM + RC filtering.
10. **Boot strapping pins overview** (levels at power-on affect boot): GPIO0 (Ethernet clock), GPIO2 (PHY power, must be low/floating to enter download mode), GPIO5 (internal pull-up, harmless as SD CS), GPIO12 (internal pull-down, do not pull up), GPIO15 (MTDO, pull low to silence ROM boot logs).
11. **GPIO6–11 are the on-chip flash** — never use them under any circumstance.
12. The metal enclosure attenuates radio signals; always use the SMA antenna brought out of the case for Wi-Fi/Bluetooth.
13. **IDF version differences**: v6 removed the Ethernet PHY drivers, the `json` component, and a batch of legacy drivers — read Chapter 8 before upgrading a v5 project.

---

## 7. Writing an Application: Project Structure and Configuration

### 7.1 Create the project

```bash
. ~/esp/esp-idf/export.sh        # adjust to your IDF install path
idf.py create-project my_app
cd my_app
idf.py set-target esp32          # classic ESP32, not S3/C3
```

### 7.2 Board pin header file

First thing in every project: create `main/board.h` pinning down the board facts (you can start by copying `wind_logger/main/board.h`):

```c
// Board pin definitions for DFRobot EDGE101 (DFR0886)
#pragma once

// RS485 (isolated), UART2, DE driven via RTS in half-duplex mode
#define PIN_RS485_TX  17
#define PIN_RS485_RX  36
#define PIN_RS485_DE  16

// I2C bus (3x Gravity connectors + onboard RTC)
#define PIN_I2C_SDA   18
#define PIN_I2C_SCL   23
#define RTC_PCF8563_ADDR 0x51

// SD card (SPI mode, shares bus with header SPI pins)
#define PIN_SD_CS     5
#define PIN_SD_MOSI   12
#define PIN_SD_MISO   39
#define PIN_SD_SCK    14

// CAN (TWAI, isolated)
#define PIN_CAN_TX    32
#define PIN_CAN_RX    35

// 4G modem on PCIe slot (UART1, shared with header P33/P34)
#define PIN_MODEM_TX  33
#define PIN_MODEM_RX  34

// User peripherals
#define PIN_LED       15   // active low
#define PIN_BUTTON    38   // active low, input only
```

### 7.3 sdkconfig baseline

Freeze the board's hardware facts in `sdkconfig.defaults` (see `wind_logger/sdkconfig.defaults` for a real example):

```
CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y
CONFIG_ESPTOOLPY_FLASHMODE_DIO=y
CONFIG_ESPTOOLPY_FLASHFREQ_80M=y
CONFIG_ESP32_DEFAULT_CPU_FREQ_240=y
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
CONFIG_PARTITION_TABLE_FILENAME="partitions.csv"
```

Without Ethernet nothing extra is needed; with Ethernet, configure the RMII clock in code (`eth_esp32_emac_config_t`, see §5.4 — code configuration applies to both v5 and v6; v6 removed the corresponding Kconfig options, while v4.x used menuconfig `CONFIG_ETH_RMII_CLK_INPUT` + `CONFIG_ETH_RMII_CLK_IN_GPIO=0`).

### 7.4 Partition table

16 MB flash is generous. Two recommended schemes:

- Simple single app (used by `wind_logger`): 3 MB app + 1 MB FATFS

```
# Name,   Type, SubType, Offset,  Size,       Flags
nvs,      data, nvs,     0x9000,  0x6000,
phy_init, data, phy,     0xf000,  0x1000,
factory,  app,  factory, 0x10000, 0x300000,
storage,  data, fat,     0x310000,0x100000,
```

- With OTA: replace `factory` with `ota_0`/`ota_1` at 3 MB each plus an `otadata` partition; roughly 9 MB remains for FAT/LittleFS.

### 7.5 Recommended application skeleton

Split by peripheral — one `.c` file + FreeRTOS task each; `app_main` only initializes and spawns tasks. `wind_logger` is a validated structure:

```
main/
├── board.h          # Board pin definitions (§7.2)
├── main.c           # app_main: init modules + create acquisition/heartbeat tasks
├── wind_sensor.c    # UART2 RS485 half-duplex Modbus RTU
├── rtc_pcf8563.c    # I2C RTC driver (PCF8563T, 0x51)
├── storage.c        # SD card SDSPI mount + FatFS writes
├── settings.c       # NVS parameter persistence
├── wifi_ap.c        # Wi-Fi SoftAP
├── httpd.c          # esp_http_server REST API + static pages
└── www/             # Front-end assets embedded into firmware via EMBED_FILES
```

### 7.6 Build, flash, debug

```bash
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor     # CH340 usually enumerates as ttyUSB0
```

- If entering download mode automatically fails: hold GPIO0 low and reset manually before flashing (see §5.1).
- Logs go to UART0 at 115200 by default.
- For production/field flashing, `esptool.py` works too; take addresses from `build/flasher_args.json`.
- For long-running deployments: enable the `esp_task_wdt` watchdog and feed it from core tasks; implement reconnect state machines for networked applications.

---

## 8. ESP-IDF v6 Adaptation Guide

This chapter is based on hands-on verification with **ESP-IDF v6.1.1** (released 2026): the reference project `wind_logger` was built successfully on this board (target `esp32`). v6 is largely compatible with v5, but a number of components were removed or moved out — work through this chapter item by item. For the complete official list, see the [ESP-IDF 6.x Migration Guides](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/migration-guides/index.html).

### 8.1 Installation and target support

- v6 **still supports the classic `esp32`** (Xtensa LX6); the board's target is unchanged. The toolchain is upgraded to GCC 15.2 (xtensa-esp-elf).
- Install (Linux/macOS):

```bash
git clone -b v6.1.1 --depth 1 --recurse-submodules --shallow-submodules \
    https://github.com/espressif/esp-idf.git ~/esp/esp-idf-v6
cd ~/esp/esp-idf-v6 && ./install.sh esp32     # ESP32-only toolchain: faster, smaller
. ~/esp/esp-idf-v6/export.sh
```

- Since v6, more drivers ship as standalone components on the [ESP Component Registry](https://components.espressif.com); `idf.py add-dependency` needs network access on first build.

### 8.2 Build-system and component changes (affect every project)

1. **The `json` component was removed** → use the Registry component instead:
   ```bash
   idf.py add-dependency "espressif/cjson"
   ```
   `#include "cJSON.h"` and the API are unchanged; only the CMake dependency name changes from `json` to `cjson` in `main/CMakeLists.txt`.
2. **The legacy `driver` component was slimmed down**: it no longer publicly re-exports the `esp_driver_*` components and now only keeps the legacy I2C / TWAI / touch drivers itself. Any file including `driver/uart.h`, `driver/gpio.h`, `driver/spi_master.h`, `driver/ledc.h`, `driver/sdspi_host.h`, etc. requires explicit dependencies in `main/CMakeLists.txt`, e.g.:
   ```
   PRIV_REQUIRES driver esp_driver_uart esp_driver_gpio esp_driver_spi \
                 esp_driver_sdspi esp_driver_ledc ...
   ```
   Otherwise the build fails with `fatal error: driver/uart.h: No such file or directory` (verified).
3. **Compiler warnings are errors by default**: clean up warnings when upgrading an old project; as a temporary escape hatch you may set `CONFIG_COMPILER_WARNINGS_AS_ERRORS=n` (not recommended long-term).

### 8.3 Ethernet adaptation (the biggest v6 change)

- **All PHY drivers were moved out of ESP-IDF** into the [esp-eth-drivers](https://github.com/espressif/esp-eth-drivers) repository / the Component Registry. `esp_eth_phy_new_ip101()` and friends are no longer built in.
- **The TLK110 driver was dropped by Espressif** (it is not in esp-eth-drivers). Since the official board package states that the IP101 is register-compatible with the TLK110, use the IP101 driver on v6:
  ```bash
  idf.py add-dependency "espressif/ip101"
  ```
  ```c
  #include "esp_eth_phy_ip101.h"                          // header name changed
  esp_eth_phy_t *phy = esp_eth_phy_new_ip101(&phy_cfg);   // API name unchanged
  ```
- If you want zero external dependencies, use the built-in **generic 802.3 PHY**: `esp_eth_phy_new_generic(&phy_cfg)` (`esp_eth_phy.h`) — usually sufficient for basic communication and handy as a fallback.
- **The RMII clock Kconfig options were removed** (`CONFIG_ETH_RMII_CLK_INPUT`, `CONFIG_ETH_RMII_CLK_IN_GPIO`, etc.): the clock can now only be configured in code via `eth_esp32_emac_config_t`. The §5.4 snippet (`EMAC_CLK_EXT_IN` + `EMAC_CLK_IN_GPIO`) is already the only correct form on v6 — no code change needed.
- The MAC side (`esp_eth_mac_new_esp32`, `esp_eth_driver_install`, netif glue) is unchanged.

### 8.4 CAN/TWAI adaptation

- The legacy `driver/twai.h` still works on v6 but raises a deprecation warning (suppressible with `CONFIG_TWAI_SUPPRESS_DEPRECATE_WARN=y`); **old and new drivers must not be mixed** — mixing triggers startup warnings and may even cause reboots.
- Recommended: migrate to the new driver introduced in v5.5 (component `esp_driver_twai`, headers `esp_twai.h` + `esp_twai_onchip.h`). New-style code with this board's pins:

```c
#include "esp_twai.h"
#include "esp_twai_onchip.h"

twai_node_handle_t node = NULL;
twai_onchip_node_config_t cfg = {
    .io_cfg.tx = GPIO_NUM_32,        // CTX
    .io_cfg.rx = GPIO_NUM_35,        // CRX
    .bit_timing.bitrate = 500000,    // plain bitrate value, ≤1 Mbps
    .tx_queue_depth = 5,
};
ESP_ERROR_CHECK(twai_new_node_onchip(&cfg, &node));
ESP_ERROR_CHECK(twai_node_enable(node));

// Transmit
uint8_t data[8] = {0};
twai_frame_t tx = { .header.id = 0x123, .buffer = data, .buffer_len = sizeof(data) };
ESP_ERROR_CHECK(twai_node_transmit(node, &tx, 0));
ESP_ERROR_CHECK(twai_node_transmit_wait_all_done(node, -1));

// Receive: register a callback and call twai_node_receive_from_isr() inside it
static bool rx_cb(twai_node_handle_t h, const twai_rx_done_event_data_t *e, void *ctx)
{
    uint8_t buf[8];
    twai_frame_t rx = { .buffer = buf, .buffer_len = sizeof(buf) };
    if (twai_node_receive_from_isr(h, &rx) == ESP_OK) { /* handle */ }
    return false;
}
twai_event_callbacks_t cbs = { .on_rx_done = rx_cb };
ESP_ERROR_CHECK(twai_node_register_event_callbacks(node, &cbs, NULL));
```

- Key differences from the legacy driver: reception is event-callback based (no blocking `twai_receive`); there is no `twai_read_alerts` (use the `on_state_change` / `on_error` callbacks); bitrate is a plain number; filtering via `twai_node_config_mask_filter()`; bus-off recovery via `twai_node_recover()`.

### 8.5 I2C adaptation

- The legacy `driver/i2c.h` is marked **EOL in v6, scheduled for removal in v7**; it still compiles on v6. New projects should always use `driver/i2c_master.h` (`i2c_new_master_bus` → `i2c_master_bus_add_device` → `i2c_master_transmit` / `i2c_master_transmit_receive`).
- The new driver returns `ESP_ERR_INVALID_RESPONSE` on bus NACK (the legacy one returned `ESP_ERR_INVALID_STATE`) — adjust error-handling code accordingly.
- `wind_logger/main/rtc_pcf8563.c` currently uses the legacy API: it compiles as-is on v6, but consider migrating later. The PCF8563 register layout (seconds at 0x02, BCD) and wiring (SDA=18/SCL=23, address 0x51) are version-independent.

### 8.6 Other peripheral and networking changes at a glance

| Item | v6 change | Impact on this board |
|---|---|---|
| ADC | legacy `driver/adc.h` removed entirely | Use `esp_adc/adc_oneshot.h` (already recommended); no further impact |
| UART | API largely unchanged; macro `UART_FIFO_LEN` → `UART_HW_FIFO_LEN` | §5.2 RS485 code works as-is |
| LEDC | `ledc_timer_set()`, `LEDC_APB_CLK_HZ` etc. removed | Normal `ledc_timer_config` / `ledc_channel_config` usage unchanged |
| DAC | legacy driver removed | No DAC on this board anyway |
| Timer / PCNT / RMT | legacy `driver/timer.h`, `driver/pcnt.h`, `driver/rmt.h` removed | Use `gptimer` / `pulse_cnt` / `rmt_tx/rx` (already the names recommended here) |
| Deep-sleep wakeup | `gpio_deep_sleep_wakeup_enable()` removed | Use `gpio_wakeup_enable_on_hp_periph_powerdown_sleep()` |
| SPI | Deprecated HSPI/VSPI IOMUX pin macros removed | Use `SPI2_HOST` etc.; §5.5 code unchanged |
| mbedTLS | Upgraded to 4.x; some v3 APIs removed | Usually transparent for projects only using `esp-tls` / HTTPS |
| cJSON | Built-in `json` component removed | See §8.2 |
| Ping | `esp_ping.h` / `ping.h` removed | Use `ping/ping_sock.h` |
| esp_netif | `esp_netif_next()` removed | Use `esp_netif_next_unsafe()` or `esp_netif_find_if()` |
| DHCP server | No longer advertises itself as DNS by default | Configure explicitly with `esp_netif_dhcps_option()` + `ESP_NETIF_DOMAIN_NAME_SERVER` if needed |
| USB / touch_element | Moved to the Registry | Typically unused on this board |

### 8.7 Verified result (wind_logger project, v6.1.1 + esp32)

On v6.1.1, `wind_logger` builds successfully with **only two adaptations**:

1. Run `idf.py add-dependency "espressif/cjson"` and change `json` to `cjson` in `main/CMakeLists.txt`;
2. Add `esp_driver_uart esp_driver_gpio esp_driver_spi esp_driver_sdspi esp_driver_ledc` to `PRIV_REQUIRES` (keep `driver` for the legacy I2C driver).

Everything else (RS485 Modbus, PCF8563, SDSPI, FatFS, SoftAP, HTTP server, NVS, MCP) builds **with zero code changes**. The project does not use Ethernet or CAN; for new projects that do, follow §8.3 / §8.4.

---

## 9. Feasibility Quick Reference for Typical Combinations

| Scenario | Feasible? | Notes |
|---|---|---|
| Wi-Fi + ADC acquisition | ✅ use P37/38/39 (ADC1) | ADC2 conflicts with Wi-Fi |
| Wi-Fi + Ethernet | ✅ | Dual netif coexistence; good for "wired primary, wireless provisioning" |
| RS485 + CAN + Ethernet | ✅ | Pins do not conflict; all can run together |
| 4G modem + header P33/P34 | ❌ | Same UART1 pin pair |
| SD card + header SPI device | ⚠️ | Shared bus; needs a separate CS |
| Ethernet + all 11 header GPIOs | ✅ | RMII pins 19/21/22/25/26/27 are not routed to the header |
| RS485 + 4G simultaneously | ✅ | UART2 and UART1 are independent |
| DAC output | ❌ | GPIO25/26 taken by Ethernet |
| Deep-sleep RTC wakeup | ✅ | RTC GPIOs 33/34/37/38/39 available |
| Multiple Gravity I2C sensors | ✅ | Unique addresses required; do not repurpose 18/23 |

---

## 10. Reference Index

- Product page: https://www.dfrobot.com.cn/goods-4126.html
- Wiki (Chinese): https://wiki.dfrobot.com.cn/SKU_DFR0886_ESP32_IoT_可编程控制器
- Wiki (English, structured examples): https://wiki.dfrobot.com/dfr0886/
- Verification sources in this repository:
  - `edge101_sdk/1.0.1/variants/esp32/pins_arduino.h` — UART0, I2C, SPI, LED pins
  - `edge101_sdk/1.0.1/libraries/Ethernet/src/ETH.h` — Ethernet PHY address/model/SMI/power pins
  - `edge101_sdk/1.0.1/libraries/ESP32CAN/` — CAN pins 32/35
  - `edge101_sdk/1.0.1/libraries/ArduinoRS485/` — RS485 pins 16/17/36
  - `edge101_sdk/1.0.1/libraries/RTC/PCF8563T.cpp` — RTC register layout
  - `wind_logger/` — complete ESP-IDF reference project validated on this board (pins, sdkconfig, partition table, driver usage; built successfully on both v5.4.1 and v6.1.1)
- ESP-IDF documentation: https://docs.espressif.com/projects/esp-idf/en/stable/esp32/
- ESP-IDF 6.x migration guides: https://docs.espressif.com/projects/esp-idf/en/stable/esp32/migration-guides/index.html
- esp-eth-drivers (Ethernet PHY/SPI module drivers moved out of IDF v6): https://github.com/espressif/esp-eth-drivers
- ESP Component Registry: https://components.espressif.com
