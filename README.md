# BQ76920 Low-Side Battery Management System (3S / 5S)

---

## Overview

This repository contains the KiCad schematic and design files for a Battery Management System (BMS) based on the Texas Instruments **BQ76920** Analog Front End (AFE). While the schematic is designed to accommodate up to a 5-series (5S) lithium-ion cell configuration, the current component selection is heavily optimized for a **3S battery pack** architecture.

This project started as an ambitious attempt at an active BMS for a larger battery pack, but as a student with limited experience I scaled it down to something I could actually design, validate, and learn from. The design is heavily referenced from TI's BQ76920 evaluation modules. There may be mistakes — the plan is to build a prototype and validate the design through real-world testing.

---

## System Architecture: Low-Side Switching

This design uses a **low-side switching topology** for charge and discharge control.

While high-side switching is generally preferred in some EV and backup applications (to maintain a continuous system ground), low-side switching was selected here to align directly with the native architecture of the BQ76920. TI specifically designed this AFE to drive low-side N-channel MOSFETs without the need for external charge pumps. Implementing high-side switching with this IC introduces unnecessary complexity, and TI recommends a different AFE family for high-side applications.

---
## Component Selection & Design Calculations

### Power Path MOSFETs

**Selected Part: IRFB3206PbF (60V, ~3mΩ, 120A)**

For a 3S pack (nominal 11.1V, max 12.6V), TI recommends a MOSFET voltage rating of approximately 10V per cell, requiring at least a **30V-rated device** for 3S. A **50–60V rated MOSFET** is needed to safely support a 5S upgrade.

The design targets a continuous current of up to **30A**. To ensure thermal stability without excessive heatsinking, maximum acceptable power dissipation per FET was calculated using a worst-case $R_{DS(on)}$ of 5mΩ:

$$P = I^2 \times R_{DS(on)} = 30^2 \times 0.005 = 4.5\text{ W per MOSFET}$$

At 4.5W per FET (9W total for the charge/discharge pair), thermal dissipation is manageable.

The IRFB3206PbF was selected for additional safety margin — it offers a 60V breakdown voltage and a very low $R_{DS(on)}$ of 3mΩ, reducing dissipation further to ~2.7W per FET at 30A. While the datasheet claims 120A continuous drain current (which should be treated with skepticism in real-world thermal environments without aggressive cooling), it is more than sufficient for the 30A continuous target. This part was also chosen for its high availability and cost-effectiveness in the local component market.

#### 3S vs 5S MOSFET Requirements

| Configuration | Max Pack Voltage | Min Vds Rating |
|---------------|-----------------|----------------|
| 3S (current)  | 12.6V           | 30V            |
| 5S (upgrade)  | 21.0V           | 50–60V         |

> For 5S operation, swap Q1 and Q2 for a 60V+ rated N-MOSFET with comparable Rds(on). The rest of the design remains unchanged.

---

### Current Sense Resistor

**Selected Value: R4 = 1mΩ**

The BQ76920 monitors system current via the voltage drop across a shunt resistor, triggering hardware protections based on internal thresholds:

| Protection | Threshold |
|------------|-----------|
| OCD — Overcurrent in Discharge | ~100mV |
| SCD — Short Circuit in Discharge | ~200mV |

By selecting a 1mΩ sense resistor, the system aligns cleanly with these hardware limits for high-drain applications. A 100A current spike generates exactly 100mV across the shunt, triggering OCD protection; a 200A dead-short hits the 200mV threshold for immediate SCD shutoff.

> **Layout note:** At 1mΩ, even a few mΩ of PCB trace resistance can affect measurement accuracy. Kelvin connections directly at the resistor pads are essential.

---

## Design Decisions

### Microcontroller & Power Architecture

**Selected MCU: STM32G030K6T6 (ARM Cortex-M0+)**

The STM32G030 was specifically selected for its highly efficient, low-power profile. A primary design constraint for this BMS was minimizing parasitic quiescent current to prevent the board from slowly draining the battery pack during prolonged storage.

To achieve this, the system eliminates the need for an external buck converter or dedicated low-dropout linear regulator (LDO). Instead, the MCU is powered directly from the BQ76920's internal 3.3V `REGOUT` pin.

- The internal `REGOUT` LDO of the BQ76920 is rated to supply up to **20mA**
- The STM32G030 typically draws around **~3mA** in its active running state

This leaves a highly comfortable current margin for the MCU and standard I²C communication, while keeping the overall footprint and standby power consumption of the BMS as low as physically possible.

---

### Charge Path Protection & Gate Drive Topology

> **Reference:** TI Application Figure 8-2

At first glance, the gate drive circuitry for the Charge FET (Q1) appears overly complex — utilizing an auxiliary P-Channel MOSFET (Q3) and a diode network instead of a direct connection to the BQ76920 `CHG` pin. This specific topology was selected to provide critical system protection against negative voltage transients on the `PACK-` terminal, which frequently occur during reverse-charger connections or highly inductive load disconnection.

If the `PACK-` voltage is driven aggressively negative relative to system ground (`VSS` / `BAT-`):

**1. Prevents Accidental Conduction** — Without Q3, the negative potential could pull the gate of Q1 low relative to its source, accidentally biasing the N-channel FET into conduction during a fault state.

**2. Protects the AFE** — The `CHG` pin on the BQ76920 cannot survive large negative voltages. The P-Channel FET (Q3), configured with a grounded gate, acts as a high-voltage blocking switch. When `CHG` is inactive, Q3 isolates the sensitive AFE silicon from the negative potentials on `PACK-`.

The parallel 1MΩ resistor ensures Q1 is safely clamped OFF when the `CHG` signal is removed, preventing floating gate states.

---

## Features

- 3S Li-ion cell monitoring (expandable to 5S with MOSFET swap)
- Passive cell balancing via BQ76920 internal balancing switches
- Hardware overcurrent (OCD) and short-circuit (SCD) protection
- Temperature monitoring via NTC thermistor (TH1, 10kΩ)
- I²C communication between BQ76920 and STM32
- ALERT line from AFE to MCU for fault notification
- TVS diode (SMCJ24A, 1500W) on Pack+ for transient surge protection
- SWD debug/programming header (J1) for STM32

---

## Project Status

- [x] Schematic design complete
- [x] Component selection and BOM
- [ ] PCB layout
- [ ] Prototype fabrication
- [ ] Hardware validation and testing
- [ ] Firmware development

---

## References

- [BQ76920 Datasheet — Texas Instruments](http://www.ti.com/lit/ds/symlink/bq76920.pdf)
- TI BQ76920 Evaluation Module (BQ76920EVM)
- [STM32G030K8T6 Datasheet — STMicroelectronics](https://www.st.com/resource/en/datasheet/stm32g030c6.pdf)
- IRFB3206PbF Datasheet — Infineon/International Rectifier

---

## Author

**Sreyas** — Student project. Feedback and suggestions welcome.

> This design has not yet been validated on hardware. Use as a reference at your own risk.
