# ⌚ InkTime: Open-Source Hackable Smartwatch

**InkTime** is an affordable, open-source smartwatch powered by the Nordic nRF52840 SoC and an ultra-low-power e-paper display. It is designed from the ground up to be highly hackable, easy to manufacture, and power-efficient.

This repository contains all Engineering Validation (EVT) and Design Validation (DVT) files, including schematics, PCB layouts, 3D mechanical models, and manufacturing files.

---

## 🏗️ 1. System Architecture (Block Diagram)

The following diagram illustrates the core components of the InkTime hardware and their data/power relationships:

```text
                     +---------------------------------------------------+
                     |               nRF52840 SoC (U$1)                  |
                     |          ARM Cortex-M4F @ 64 MHz                  |
                     |    BLE 5.0 | 1MB Flash | 256KB RAM | USB 2.0      |
                     +---------------------------------------------------+
                              |                 |                  |
       [ SPI & GPIO ]---------+                 |                  +--------[ I2C Bus ]
       |                                        |                  |
+-------------------+                           |           +-------------------+
| E-Paper Display   |                           |           | BQ25180 (Charger) |<-- USB VBUS
| 24-pin FPC (J2)   |                           |           +-------------------+
+-------------------+       [ Push Buttons ]----+           | MAX17048 (Fuel)   |<-- LiPo Cell
                            | SW_UP (P0.13)     |           +-------------------+
+-------------------+       | SW_ENT (P0.14)    |           | BMA423 (IMU)      |
| Power Management  |       | SW_DN (P1.02)     |           +-------------------+
| RT6160A 3.3V Reg. |       +-------------------+           | DRV2605 (Haptic)  |
+-------------------+                                       +-------------------+
```

## 🧩 2. Hardware Implementation & Functionality

The hardware is designed on a 1mm thick, 2-layer PCB to fit strictly within the mechanical constraints of the provided case. All component placement is restricted to the TOP layer, with dual ground planes and extensive via stitching.

### Core Processing & Connectivity (nRF52840)

The heart of the watch is the nRF52840-QIAA. It handles all BLE 5.0 communication, USB 2.0 Full-Speed interfacing, and peripheral management.

- **Clocks:** Requires a 32 MHz crystal (X1) for the RF HFXO and a 32.768 kHz crystal (X2) for the RTC/LFXO deep sleep timekeeping.
- **RF Layout:** The 2.4 GHz chip antenna (Johanson 2450AT18B100E) is edge-mounted. Per Nordic guidelines, the PCB is physically cut out directly beneath the antenna, with a strict keepout zone (no copper, no routing).

### Display System

We utilize an e-paper display connected via a 24-pin 0.5mm FPC connector.

- **Interface:** 4-wire SPI + GPIO control (DC, RST, BUSY).
- **Power:** Driven by an independent 3.3V boost converter (L1). Power is gated by a P-channel MOSFET (Q1) via the `EPD_PWR` pin, ensuring zero current draw when the display is not refreshing.

### Power Management & Battery (BQ25180 + RT6160A)

- **Charging:** The BQ25180 manages charging from the USB-C VBUS. It is compact and entirely I2C-configurable. The charge current is hardware-set to 100mA via a 10kΩ resistor.
- **Regulation:** The RT6160A buck-boost converter takes the variable LiPo voltage (3.0V - 4.2V) and provides a stable 3.3V system rail.
- **Clearance hack:** To meet strict Z-axis constraints of the case, the JST connector was omitted; the LiPo cell is soldered directly to `TP_VBAT` and `TP_BAT_GND`.

### Sensors & Actuators

- **IMU (BMA423):** A 3-axis accelerometer with a built-in hardware step counter and wrist-tilt detection. It uses hardware interrupts (`IMU_INT1`, `IMU_INT2`) to wake the nRF from deep sleep, saving processing power.
- **Haptics (DRV2605):** Drives an ERM/LRA vibration motor. The `HAPTIC_EN` pin physically disables the driver during sleep to eliminate quiescent current leakage.

## 📌 3. nRF52840 Pin Mapping

Pins were carefully selected to group peripheral buses (like SPI and I2C) physically close to their target ICs, minimizing trace lengths.

| Pin                     | Signal            | Destination      | Function & Justification |
|-------------------------|-------------------|------------------|--------------------------|
| P0.02                   | SCK               | EPD (J2)         | SPI Clock. SPIM is remappable; chosen for routing ease. |
| P0.03                   | MOSI              | EPD (J2)         | SPI Data Out. Grouped with SCK to minimize trace length. |
| P0.05                   | EPD_CS            | EPD (J2)         | SPI Chip Select for e-paper. |
| P0.06                   | SDA               | I2C Bus          | I2C Data. Placed close to BQ25180 & BMA423 to keep traces short. |
| P0.07                   | SCL               | I2C Bus          | I2C Clock. |
| P0.08                   | IMU_INT1          | BMA423           | Hardware wake interrupt from accelerometer. |
| P0.11                   | PMIC_INT          | BQ25180          | Interrupt for battery charge state changes. |
| P0.12                   | HAPTIC_EN         | DRV2605          | GPIO output to hard-disable haptic driver during sleep. |
| P0.13 / P0.14 / P1.02   | SW_UP / ENT / DN  | Tactile Switches | GPIO inputs with hardware debouncing (10kΩ pull-up + 1µF low-pass). |
| P1.01                   | EPD_PWR           | Q1 Gate          | EPD power rail enable switch. |

## 🔋 4. Power Consumption Analysis

The system is highly optimized for deep sleep. Based on component datasheets, estimated battery life on a 180mAh LiPo cell is approximately 5-7 days.

| Mode                  | nRF52840 | Display | Sensors/Other | Total |
|-----------------------|----------|---------|---------------|-------|
| System OFF (Sleep)    | 2.5 µA   | 0 mA    | 0.5 µA        | ~3 µA |
| BLE Advertising (1s)  | ~15 mA   | 0 mA    | 0.13 mA       | ~15 mA |
| Display Refresh       | ~3 mA    | ~30 mA  | 0.13 mA       | ~33 mA |
| Haptic Feedback       | ~5 mA    | 0 mA    | ~60 mA        | ~65 mA |

## 🧾 5. Bill of Materials (BOM)

> Note: All resistors/capacitors are 0201 packages unless otherwise specified (e.g., >100nF uses 0402).

| Ref  | Qty | Part Number      | Description             | Where to Buy | Datasheet |
|------|-----|------------------|-------------------------|--------------|-----------|
| U$1  | 1   | nRF52840-QIAA    | Main BLE 5.0 SoC        | JLC C190794  | PDF |
| IC1  | 1   | BQ25180YBGR      | I2C LiPo Charger        | JLC C2678360 | PDF |
| IC2  | 1   | BMA423           | 3-axis accelerometer    | JLC C379441  | PDF |
| IC3  | 1   | DRV2605YZFR      | Haptic Driver           | JLC C89025   | PDF |
| U1   | 1   | MAX17048G+T10    | 1-cell fuel gauge       | JLC C112139  | PDF |
| IC9  | 1   | RT6160AWSC       | Buck-boost 3.3V Reg     | JLC C389054  | PDF |
| ANT1 | 1   | 2450AT18B100E    | 2.45 GHz chip antenna   | JLC C89939   | PDF |
| J3   | 1   | KH-TYPE-C-16P    | USB-C 16-pin            | JLC C2765186 | PDF |

## 🛠️ 6. Design Log & Constraint Compliance (Review Notes)

This design strictly follows the DVT guidelines and mechanical constraints. Below are key design decisions and ignored rule exceptions intended for the reviewer:

- **1mm PCB thickness:** Reduced from standard 1.6mm to ensure vertical mechanical clearance inside the provided STEP case.
- **TOP-layer-only placement:** All SMDs and ICs are strictly placed on the TOP layer. Both layers feature grounded copper pours, heavily via-stitched (especially near the RF path).
- **Trace widths & geometry:** Power nets (`3V3`, `VBAT`, `VBUS`, `VREG`) are routed at 0.3mm. Data signals use 0.15mm. Trace neck-down is only utilized under BGA pads. 90-degree angles were avoided; all corners use 45-degree chamfers.
- **Decoupling proximity:** Every 100nF decoupling capacitor is placed immediately adjacent (within one pad length) to its respective IC power pin on the TOP layer.
- **RF keepout & substrate:** The area under the 2450AT18B100E antenna is entirely void of copper on both layers, and features a physical PCB cutout per Nordic reference design.
- **Accepted ERC warning:** "Only INPUT pins on NET ID" is present but ignored as a known false-positive in the CAD tool, approved by project guidelines.
- **Accepted DRC errors:** Dimension errors near the three tactile buttons and the USB-C aperture are intentional structural slots required for casing alignment.
- **Silkscreen integrity:** Test pads (`TP_3V3`, `TP_SWDIO`, etc.) are clearly labeled in silkscreen. Component values are hidden to maintain a clean, readable layout.
