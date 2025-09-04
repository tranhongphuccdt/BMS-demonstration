# BMS Training Demo — Option A1 Diagrams

This document contains the **System Diagram** for **Option A1** (STM32 + TCAN1042 + BQ76952EVM + INA226) and the **Software Architecture Diagram** for the training platform.

---

## 1) System Diagram — Option A1

```mermaid
flowchart LR
  %% ===== PC / Host Side =====
  subgraph PC[PC / Host]
    CANable[CANable 2.0<br/>USB–CAN FD]
    Logger[Jupyter / Python Logger]
  end

  %% ===== MCU Board =====
  subgraph MCU[NUCLEO‑G474RE (STM32G474)]
    FDCAN[FDCAN]
    I2C[I2C1]
    USB[USB‑CDC (optional)]
    GPIO[GPIO / PWM]
  end

  %% ===== Transceiver =====
  TCAN[TCAN1042<br/>CAN FD Transceiver]

  %% ===== Battery AFE / EVM =====
  subgraph AFE[BQ76952EVM (AFE + FETs + NTCs + cell dividers)]
    Cells[4× LFP 18650 (training)<br/>→ 3–16s inputs]
    NTCs[NTC thermistors (2–9)]
    FETs[Charge / Discharge FETs<br/>+ Pre‑charge path]
    CC[Coulomb counter & protections<br/>(OV/UV/OT/OC/SC)]
  end

  %% ===== Additional Sensors & Power Path =====
  INA[INA226 Current/Power Monitor]
  Load[Load: MOSFET + Power Resistor<br/>or DC Electronic Load]
  Charger[Charger (optional)]

  %% ===== CAN Bus =====
  CANbus((CAN Bus))

  %% Connections
  PC --- CANable
  CANable <--> CANbus
  CANbus <--> TCAN
  TCAN <--> FDCAN
  I2C <--> AFE
  I2C <--> INA
  USB --- MCU
  Cells --> AFE
  NTCs --> AFE
  AFE --> FETs --> Load
  Charger --> AFE
```

**Signals & Interfaces**
- **I²C1** ↔ **BQ76952EVM** (cell voltages, temperatures, coulomb counter, commands)  
- **I²C1** ↔ **INA226** (pack current & bus voltage)  
- **FDCAN** ↔ **TCAN1042** ↔ **CANable/PC** (telemetry, DTCs, control)  
- **USB‑CDC** (optional) for CSV logging / firmware debug  
- **Cells/NTCs** wired to the **EVM**; **FETs/pre‑charge** drive the **load/charger**

---

## 2) Software Architecture Diagram

```mermaid
flowchart TB
  subgraph Platform[Platform / Drivers]
    HAL[HAL Drivers: I²C, FDCAN, GPIO, Timers, ADC, NVM]
    BQ[BQ76952 Driver]
    INA[INA226 Driver]
  end

  subgraph Services[Core Services]
    Sched[Scheduler (bare‑metal or RTOS)]
    Sampler[Sampler<br/>I:100 Hz, V:10 Hz, T:1 Hz]
    SOC[SOC Estimator<br/>Coulomb Count + OCV (rest)<br/>EKF/UKF optional]
    SOH[SOH Estimator<br/>Capacity & R_int pulses]
    Safety[Safety Manager<br/>OV/UV, OT/UT, OC/SC → safe state]
    Balance[Balancing Manager (v2)<br/>Autonomous/Host‑controlled]
    Comm[Comms<br/>FDCAN frames; USB‑CDC CSV]
    Diag[Diagnostics/DTC Mapping]
  end

  subgraph Data[Calibration & Tables]
    OCV[(OCV–SOC Table)]
    Cal[(Calibration & Limits in NVM)]
    Logs[(CSV Logs)]
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
  SOC -- plausibility/limits --> Safety
  SOH -- trend flags --> Safety
  Safety -- open FETs / inhibit charge --> BQ
  Balance -- bleed commands --> BQ
```

---

## 3) ASCII Fallback (System)

```
 PC/Jupyter  --USB-->  CANable  ==CAN==  TCAN1042  <== FDCAN ==>  STM32G474 (NUCLEO)
                                         |
                               I2C1 <----+---->  BQ76952EVM (cells, NTCs, FETs, coulomb)
                               I2C1 <---------->  INA226 (shunt)
  Cells/NTCs --> BQ76952EVM --> FETs/Precharge --> Load / Charger
  USB‑CDC (optional) <------> STM32G474 (CSV logging)
```
