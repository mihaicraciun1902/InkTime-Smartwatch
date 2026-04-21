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

Updated from Fusion Hub partlist export (Project: **Proiect TSC**, Board: **PCB**, Date: **21/04/2026 4:52 PM**).

| Qty | Value / MPN | Device / Package | Parts | Description |
|-----|-------------|------------------|-------|-------------|
| 1 | ESP32_C6_LIBRARY_7_JUMPER_SJ | ESP32_C6_LIBRARY_7_JUMPER_SJ | SJ1 | SMD solder jumper |
| 3 | 0 / CPF0201D7K68C1 | CPF0201D7K68C1_0201 | R2, R3, R4 | 7.68k thin film resistor, 0201 |
| 2 | 0.1uF / GRM011R60J152KE01L | CAPC0201X13N | C23, C42 | Murata capacitor |
| 3 | 0.1uF | NORDIC_NRF_3_RESC0201_L | C27, C34, EPD_C5 | Generic chip capacitor |
| 1 | 0.47 / CPF0201D7K68C1 | CPF0201D7K68C1_0201 | R1_EP_DR | 0201 thin film resistor |
| 1 | 1.0uF | NORDIC_NRF_3_RESC0201_L | C15 | Generic chip capacitor |
| 5 | 100nF | NORDIC_NRF_3_RESC0201_L | C5, C7, C8, C12, C19 | Generic chip capacitor |
| 1 | 100pF | NORDIC_NRF_3_RESC0201_L | C11 | Generic chip capacitor |
| 2 | 10K / CPF0201D7K68C1 | CPF0201D7K68C1_0201 | R2_EP_DR, R_PWR_EPD | 0201 thin film resistor |
| 4 | 10k / CPF0201D7K68C1 | CPF0201D7K68C1_0201 | R5, R7, R8, R9 | 0201 thin film resistor |
| 3 | 10uF | NORDIC_NRF_3_RESC0201_L | C1-EP-DR, C24, C39 | Generic chip capacitor |
| 1 | 10uH | NORDIC_NRF_2_RESC0402_L | L2 | Generic chip inductor |
| 4 | 12pF | NORDIC_NRF_3_RESC0201_L | C1, C2, C17, C18 | Generic chip capacitor |
| 1 | 15nH | NORDIC_NRF_2_RESC0402_L | L3 | Generic chip inductor |
| 2 | 1pF | NORDIC_NRF_3_RESC0201_L | C3, C4 | Generic chip capacitor |
| 2 | 1uF / GRM011R60J152KE01L | CAPC0201X13N | C37, C38 | Murata capacitor |
| 9 | 1uF | NORDIC_NRF_3_RESC0201_L | EPD_C1, EPD_C2, EPD_C6, EPD_C7, EPD_C8, EPD_C9, EPD_C10, EPD_C11, EPD_C12 | Generic chip capacitor |
| 4 | 1uF | NORDIC_NRF_3_RESC0402_L | C29, C30, C31, C32 | Generic chip capacitor |
| 1 | 2.2 / CPF0201D7K68C1 | CPF0201D7K68C1_0201 | R_TYPE_SEL | 0201 thin film resistor |
| 3 | EVP-AKE31A | SW_EVP-AKE31A_PAN | SW_DN, SW_ENT, SW_UP | Panasonic tactile switch |
| 1 | DMG2305UX-7 | SOT23-3 | Q1 | P-channel MOSFET (20V/4.2A/52mΩ) |
| 2 | 22uF | NORDIC_NRF_3_RESC0201_L | C25, C33 | Generic chip capacitor |
| 1 | 3.9nH | NORDIC_NRF_2_RESC0201_M | L1 | Generic chip inductor |
| 1 | 9HT10-32.768KDZF-T | XTAL330X1051X90N | X2 | 32.768kHz crystal |
| 1 | 32MHz | NORDIC_NRF_BT-XTAL_2016_N | X1 | 32MHz crystal |
| 2 | 3k3 / CPF0201D7K68C1 | CPF0201D7K68C1_0201 | R17, R18 | 0201 thin film resistor |
| 6 | 4.7uF | NORDIC_NRF_3_RESC0201_L | C2-EP-DR, C6, C14, C20, C21, C43 | Generic chip capacitor |
| 1 | 2450AT18B100E | ANTC3216X140N | ANT1 | 2.45GHz chip antenna |
| 1 | 47nF | NORDIC_NRF_3_RESC0201_L | C16 | Generic chip capacitor |
| 1 | 744043680 | IND_4828-WE-TPC_WRE | L5 | Wurth inductor, 48uH |
| 1 | 503480-2400 | 5034802400 | J1 | 24-pin 0.5mm FPC connector |
| 2 | 5k1 / CPF0201D7K68C1 | CPF0201D7K68C1_0201 | R1_USB, R2_USB | 0201 thin film resistor |
| 1 | 820pF | NORDIC_NRF_3_RESC0201_L | C9 | Generic chip capacitor |
| 1 | BMA423 | BMA423_BMA423 | IC3 | 3-axis accelerometer |
| 1 | BQ25180YBGR | BGA8C40P2X4_100X155X50 | IC1 | Li-ion/LiFePO4 charger IC |
| 1 | FTC252012SR47MBCA | INDC2016X100N | L7 | 0.47uH inductor |
| 14 | HECTOR_WATCH_1_TPTP20R | HECTOR_WATCH_1_TP20R | TP_3.3V, TP_3V3, TP_BAT_GND, TP_GND, TP_ON, TP_OP, TP_RESET, TP_SCL, TP_SDA, TP_SWDIO, TP_SWD_CLK, TP_SWO, TP_VBAT, TP_VREG | Test pads |
| 1 | KH-TYPE-C-16P | KH-TYPE-C-16P_KINGHELM_KH-TYPE-C-16P | J4 | USB-C 16-pin connector |
| 1 | MAX17048G+T10 | SON50P200X200X80-9N | U3 | 1-cell/2-cell fuel gauge |
| 3 | MBR0530 | SOD3716X135N | D2, D4, D5 | Schottky diode 30V 0.5A |
| 3 | N.C. | NORDIC_NRF_3_RESC0201_L | C10, C13, C22 | DNI capacitor footprints |
| 1 | NRF52840_QF | AQFN50P700X700X85_HS-74N | U1 | nRF52840 SoC |
| 1 | RT6160AWSC | BGA15C40P3X5_140X230X60 | IC9 | Buck-boost regulator |
| 1 | DRV2605YZFR | BGA9C50P3X3_144X144X62 | IC2 | Haptic driver |
| 1 | SI1308EDL-T1-GE3 | SOT65P210X110-3N | Q3 | N-channel MOSFET |
| 1 | TC2030-IDC | TC2030IDC | J2 | 6-pos cable adapter |
| 1 | USBLC6-2SC6Y | SOT95P280X145-6N | D3 | USB ESD TVS protection array |

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
