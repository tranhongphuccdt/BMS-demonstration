# BMS Training Demo — Option A1 Diagrams (Fixed Mermaid)

This document contains the **System Diagram** for **Option A1** (STM32 + TCAN1042 + BQ76952EVM + INA226) and the **Software Architecture Diagram** for the training platform.

> **Note:** To maximize compatibility with GitHub’s Mermaid renderer, this version avoids non‑ASCII punctuation, HTML line breaks, and edges that target subgraph IDs.

---

## 1) System Diagram — Option A1 (GitHub‑compatible Mermaid)

```mermaid
flowchart LR
  %% ===== PC / Host Side =====
  subgraph PC["PC / Host"]
    CANable["CANable 2.0 (USB-CAN FD)"]
    Logger["Jupyter / Logger"]
  end

  %% ===== MCU Board =====
  subgraph MCU["NUCLEO-G474RE (STM32G474)"]
    FDCAN["FDCAN"]
    I2C["I2C1"]
    USB["USB-CDC (optional)"]
    GPIO["GPIO / PWM"]
  end

  %% ===== Transceiver =====
  TCAN["TCAN1042 (CAN FD Transceiver)"]

  %% ===== Battery AFE / EVM =====
  subgraph AFE["BQ76952EVM (AFE + FETs + NTCs + dividers)"]
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
  I2C <--> AFE
  I2C <--> INA
  USB --- Logger
  Cells --> AFE
  NTCs --> AFE
  AFE --> FETs --> Load
  Charger --> AFE
```

**Signals & Interfaces**
- **I2C1** <-> **BQ76952EVM** (cell voltages, temperatures, coulomb counter, commands)  
- **I2C1** <-> **INA226** (pack current and bus voltage)  
- **FDCAN** <-> **TCAN1042** <-> **CANable/PC** (telemetry, DTCs, control)  
- **USB-CDC** (optional) for CSV logging / firmware debug  
- **Cells/NTCs** wired to the **EVM**; **FETs/pre-charge** drive the **load/charger**

---

## 2) Software Architecture Diagram (GitHub‑compatible Mermaid)

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
 PC/Jupyter  --USB-->  CANable  == CAN ==  TCAN1042  <== FDCAN ==>  STM32G474 (NUCLEO)
                                          |
                                I2C1 <----+---->  BQ76952EVM (cells, NTCs, FETs, coulomb)
                                I2C1 <---------->  INA226 (shunt)
 Cells/NTCs --> BQ76952EVM --> FETs/Precharge --> Load / Charger
 USB-CDC (optional) <------> PC Logger
```
