# BMS Training Demo — Option B2 Diagrams (GitHub-compatible Mermaid)

This document contains the **System Diagram** for **Option B2** (NXP S32K144EVB + TCAN1042 + Libre Solar BMS‑C1 + INA226) and the **Software Architecture Diagram** for the training platform.

---

## 1) System Diagram — Option B2

```mermaid
flowchart LR
  %% ===== PC / Host Side =====
  subgraph PC["PC / Host"]
    CANable["CANable 2.0 (USB-CAN FD)"]
    Logger["Jupyter / Logger"]
  end

  %% ===== MCU Board =====
  subgraph MCU["S32K144EVB (NXP S32K144)"]
    FDCAN["FDCAN"]
    I2C["I2C1"]
    USB["USB-CDC (optional)"]
  end

  %% ===== Transceiver =====
  TCAN["TCAN1042 (CAN FD Transceiver)"]

  %% ===== Battery Board (Libre Solar) =====
  subgraph BMSC1["Libre Solar BMS-C1 (BQ76952-based)"]
    Cells["4x LFP 18650 -> 3-16s inputs"]
    NTCs["NTC thermistors"]
    FETs["Charge / Discharge FETs + Pre-charge"]
    CC["Coulomb counter and protections"]
  end

  %% ===== Additional Sensors & Power Path =====
  INA["INA226 Current/Power Monitor"]
  Load["Load: MOSFET + Power Resistor or DC Electronic Load"]
  Charger["Charger (optional)"]

  %% ===== CAN Bus =====
  CANbus(("CAN Bus"))

  %% Connections
  PC --- CANable
  CANable <--> CANbus
  CANbus <--> TCAN
  TCAN <--> FDCAN
  I2C <--> BMSC1
  I2C <--> INA
  USB --- Logger
  Cells --> BMSC1
  NTCs --> BMSC1
  BMSC1 --> FETs --> Load
  Charger --> BMSC1
```

**Signals & Interfaces**
- **I2C1** <-> **BMS‑C1** (BQ76952 driver: cell voltages, temperatures, coulomb counter, commands)  
- **I2C1** <-> **INA226** (pack current and bus voltage)  
- **FDCAN** <-> **TCAN1042** <-> **CANable/PC** (telemetry, DTCs, control)  
- **USB‑CDC** (optional) for CSV logging / firmware debug  
- **Cells/NTCs** wired to the **BMS‑C1**; **FETs/pre‑charge** drive the **load/charger**

---

## 2) Software Architecture Diagram

```mermaid
flowchart TB
  subgraph Platform["Platform / Drivers"]
    HAL["HAL Drivers: I2C, FDCAN, GPIO, Timers, ADC, NVM"]
    BQ["BQ76952 Driver"]
    INA["INA226 Driver"]
  end

  subgraph Services["Core Services"]
    Sched["Scheduler (bare-metal or RTOS)"]
    Sampler["Sampler: I 100 Hz, V 10 Hz, T 1 Hz"]
    SOC["SOC Estimator: Coulomb + OCV (rest), EKF/UKF optional"]
    SOH["SOH Estimator: Capacity and R_int pulses"]
    Safety["Safety Manager: OV/UV, OT/UT, OC/SC -> safe state"]
    Balance["Balancing Manager (v2): Autonomous or Host-controlled"]
    Comm["Comms: FDCAN frames; USB-CDC CSV"]
    Diag["Diagnostics / DTC Mapping"]
  end

  subgraph Data["Calibration and Tables"]
    OCV["OCV-SOC Table"]
    Cal["Calibration and Limits in NVM"]
    Logs["CSV Logs"]
  end

  HAL --> BQ
  HAL --> INA
  BQ --> Sampler
  INA --> Sampler
  Sched --> Sampler
  Sampler --> SOC
  Sampler --> SOH
  SOC --> Comm
  SOH --> Comm
  Safety --> Comm
  Comm --> Logs
  OCV --> SOC
  Cal --> SOC
  Cal --> SOH
  Cal --> Safety
  Cal --> Balance
  SOC --> Safety
  SOH --> Safety
  Safety --> BQ
  Balance --> BQ
```

---

## 3) ASCII Fallback (System)

```
 PC/Jupyter  --USB-->  CANable  == CAN ==  TCAN1042  <== FDCAN ==>  S32K144EVB (NXP S32K144)
                                          |
                                I2C1 <----+---->  Libre Solar BMS-C1 (BQ76952)
                                I2C1 <---------->  INA226 (shunt)
 Cells/NTCs --> BMS-C1 --> FETs/Precharge --> Load / Charger
 USB-CDC (optional) <------> PC Logger
```
