# BMS Training Demo — Option A1 (Enhanced Diagrams)

These diagrams are styled for clarity and next‑step execution. Colors differentiate **inputs**, **outputs**, **processing/software**, **hardware elements**, **communications**, **bus**, **safety**, and **data/storage**. Render in any Mermaid‑enabled Markdown viewer (e.g., GitHub).

---

## Legend

```mermaid
flowchart LR
  In[Input]:::input --> Out[Output]:::output
  Proc[Processing / Software]:::processing --> Saf[Safety Fn]:::safety
  HW[Hardware Element]:::hw --> Comm[Comm Interface]:::comm --> Bus((Bus)):::bus
  Store[Data / Storage]:::storage

  classDef input fill:#FFF3E0,stroke:#EF6C00,stroke-width:1px;
  classDef output fill:#FCE4EC,stroke:#AD1457,stroke-width:1px;
  classDef processing fill:#E8F5E9,stroke:#2E7D32,stroke-width:1px;
  classDef safety fill:#FFEBEE,stroke:#C62828,stroke-width:1px;
  classDef hw fill:#E3F2FD,stroke:#1565C0,stroke-width:1px;
  classDef comm fill:#E0F7FA,stroke:#006064,stroke-width:1px;
  classDef bus fill:#EDE7F6,stroke:#6A1B9A,stroke-width:2px;
  classDef storage fill:#F5F5F5,stroke:#424242,stroke-width:1px;
```
---

## 1) System Diagram — Option A1 (STM32 + TCAN1042 + BQ76952EVM + INA226)

```mermaid
flowchart LR
  %% ===== Classes =====
  classDef input fill:#FFF3E0,stroke:#EF6C00,stroke-width:1px;
  classDef output fill:#FCE4EC,stroke:#AD1457,stroke-width:1px;
  classDef processing fill:#E8F5E9,stroke:#2E7D32,stroke-width:1px;
  classDef safety fill:#FFEBEE,stroke:#C62828,stroke-width:1px;
  classDef hw fill:#E3F2FD,stroke:#1565C0,stroke-width:1px;
  classDef comm fill:#E0F7FA,stroke:#006064,stroke-width:1px;
  classDef bus fill:#EDE7F6,stroke:#6A1B9A,stroke-width:2px;

  %% ===== Host Side =====
  subgraph PC["PC / Host"]
    Logger["Logger (Jupyter / Python)"]:::processing
    CANable["CANable 2.0 (USB-CAN FD)"]:::comm
  end

  %% ===== MCU Board =====
  subgraph MCU["STM32 NUCLEO-G474RE"]
    FDCAN["FDCAN"]:::comm
    I2C["I2C1"]:::comm
    USB["USB-CDC (optional)"]:::comm
  end

  %% ===== Transceiver =====
  TCAN["TCAN1042 (CAN FD Transceiver)"]:::comm

  %% ===== Battery AFE / EVM =====
  subgraph AFE["TI BQ76952EVM"]
    Cells["4x LFP 18650 (training)"]:::input
    NTCs["NTC Thermistors"]:::input
    FETs["Charge / Discharge FETs + Pre-charge"]:::hw
    CC["Coulomb Counter and Protections"]:::safety
  end

  %% ===== Additional Sensors & Power Path =====
  INA["INA226 Current/Power Monitor"]:::hw
  Load["Load: MOSFET + Power Resistor or DC E-Load"]:::output
  Charger["Charger (optional)"]:::input

  %% ===== CAN Bus =====
  CANbus(("CAN Bus")):::bus

  %% ===== Connections =====
  Logger -- USB --> CANable
  CANable <--> CANbus
  CANbus <--> TCAN
  TCAN <--> FDCAN
  I2C <--> AFE
  I2C <--> INA
  USB -. optional .- Logger
  Cells --> AFE
  NTCs --> AFE
  AFE --> FETs --> Load
  Charger --> AFE
```
**Key notes**
- **I2C1** links the MCU to **BQ76952EVM** and **INA226** for measurement/control.  
- **FDCAN → TCAN1042 → CANable → PC** forms the telemetry and control path.  
- **Safety** actions (OV/UV/OT/OC/SC) are primarily enforced in the AFE (**CC** block).

---

## 2) Software Architecture — Training Scope

```mermaid
flowchart TB
  %% ===== Classes =====
  classDef input fill:#FFF3E0,stroke:#EF6C00,stroke-width:1px;
  classDef output fill:#FCE4EC,stroke:#AD1457,stroke-width:1px;
  classDef processing fill:#E8F5E9,stroke:#2E7D32,stroke-width:1px;
  classDef safety fill:#FFEBEE,stroke:#C62828,stroke-width:1px;
  classDef hw fill:#E3F2FD,stroke:#1565C0,stroke-width:1px;
  classDef comm fill:#E0F7FA,stroke:#006064,stroke-width:1px;
  classDef storage fill:#F5F5F5,stroke:#424242,stroke-width:1px;

  %% ===== Platform / Drivers =====
  subgraph Platform["Platform / Drivers"]
    HAL["HAL: I2C, FDCAN, GPIO, Timers, ADC, NVM"]:::processing
    BQdrv["BQ76952 Driver"]:::processing
    INAdrv["INA226 Driver"]:::processing
  end

  %% ===== Services =====
  subgraph Services["Core Services"]
    Scheduler["Scheduler (bare-metal or RTOS)"]:::processing
    Sampler["Sampler: I 100 Hz, V 10 Hz, T 1 Hz"]:::processing
    SOC["SOC Estimator: Coulomb + OCV (rest); EKF/UKF optional"]:::processing
    SOH["SOH Estimator: Capacity and R_int pulses"]:::processing
    Safety["Safety Manager: OV/UV, OT/UT, OC/SC -> safe state"]:::safety
    Balance["Balancing Manager (v2): Autonomous or Host-controlled"]:::processing
    Comm["Comms: FDCAN frames; USB-CDC CSV"]:::comm
    Diag["Diagnostics / DTC Mapping"]:::processing
  end

  %% ===== Data / Storage =====
  subgraph Data["Calibration and Tables"]
    OCV["OCV-SOC Table"]:::storage
    Cal["Calibration and Limits (NVM)"]:::storage
    Logs["CSV Logs"]:::storage
  end

  %% ===== Flows =====
  HAL --> BQdrv
  HAL --> INAdrv
  BQdrv --> Sampler
  INAdrv --> Sampler
  Scheduler --> Sampler
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
  Safety --> BQdrv
  Balance --> BQdrv
```
**Reading the colors**
- **Green** = algorithms/services; **Blue** = hardware elements; **Cyan** = comms; **Orange** inputs; **Rose** outputs; **Red** safety; **Gray** storage/calibration.

---

## 3) Suggested Next Steps (from these diagrams)

1. **Bring-up:** verify I2C comms to **BQ76952EVM** and **INA226**; stream raw frames over **FDCAN**.  
2. **Sampler timing:** lock **100 Hz** (current), **10 Hz** (cell V), **1 Hz** (temps).  
3. **SOC v1:** coulomb counting + rest‑only OCV correction; CSV logs for replay.  
4. **SOH v1:** current‑pulse **R_int**; cycle‑based capacity tracking.  
5. **Safety thresholds:** configure OV/UV/OT/OC/SC; validate **safe state** (open FETs).  
6. **Balancing (v2):** top‑of‑charge bleed with hysteresis and idle guards.

