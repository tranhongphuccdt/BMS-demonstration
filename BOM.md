1) What we’re building (v1 goals)

Target use: hands-on training for SOC (coulomb counting + OCV correction) and SOH (capacity + impedance) on a safe, low-voltage pack (4-cell LFP recommended).

Cost bias: pick affordable, widely available parts; lean on open-source where possible.

Scalability: the same stack should scale to 12–16 cells and add passive (v2) and optional active (v3) balancing.

Safety mindset: bench-top, fused, current-limited; keep safety and non-safety concerns separated in software (QM level for training).

2) Recommended hardware (low cost, expandable)
A. Core controller + comms (two good options)

Option A (cheapest + plenty of analog):

MCU board: ST NUCLEO-G474RE (Cortex-M4F, FDCAN, many op-amps/comparators). Typical board price ≈ US$22. 
Newark Electronics

CAN transceiver: TI TCAN1042 (CAN-FD capable, automotive-grade). 
Texas Instruments
Mouser Electronics

USB-CAN to PC: CANable 2.0 (open-source; low-cost). Expect US$35 retail, clones ~US$19. 
Openlight Labs
makerbase3d.com

Option B (automotive toolchain from day 1):

MCU board: NXP S32K144EVB (ASIL-B capable family; CAN-FD). “Low-cost” EVB commonly ~US$49. 
NXP Semiconductors
DigiKey

S32K family targets ISO 26262 up to ASIL-B and has safety-ready drivers (RTD). 
NXP Semiconductors
+1

Pick A if you want the absolute cheapest start; Pick B if you want an ISO-26262-oriented MCU family for training from day one.

B. Cell monitoring & balancing

Best value / open-source path (my top pick):

Battery monitor AFE: TI BQ76952 (3–16 s, high-accuracy, host/autonomous balancing, coulomb counter). 
Texas Instruments

Hardware platform: Libre Solar BMS-C1 (open hardware using BQ76952; 3–16s; CAN; passive balancing; detailed manual + BOM). This is much cheaper than buying the EVM if you can order the PCB and assemble (BOM in repo). 
GitHub
libre.solar

If you don’t want to assemble PCBs:

TI BQ76952EVM (complete board with FETs/NTCs, etc.)—excellent for lab use, but pricier (~US$249–$299). It even includes on-board resistor dividers to simulate cells for bring-up. 
Texas Instruments
+1
DigiKey

Why BQ76952? High-accuracy cell voltages, integrated coulomb counter, multiple thermistors, and built-in passive balancing control—perfect for SOC/SOH research and later balancing work. 
Texas Instruments
+1

C. Pack current & temperature sensing (for SOC/SOH)

Current: TI INA226 (I²C current/power monitor) + low-ohm shunt; great value for coulomb counting and load profiling. 
Texas Instruments
+1

(BQ76952 also has a coulomb counter—use both initially to compare and teach redundancy and calibration.) 
Texas Instruments

Temperature: 10 kΩ NTCs on cells/pack (BQ76952 supports up to nine external thermistors). 
Texas Instruments

D. Safe bench energy + loads

Cells (training): Start with 4 × LFP 18650 in holders (12.8 V nominal) + 10 A fuse near pack positive.

No-battery bring-up: Use resistor-divider cell simulator or the EVM’s built-in dividers to emulate cell voltages (great for first SOC code). Note balancing can perturb divider voltages—TI warns about it and how to debug. 
Texas Instruments
+1

Simple load: MOSFET + power resistor (or an inexpensive DC electronic load) for current-pulse tests (SOH Rint estimation).

3) Development environment (free, robust)

Firmware IDE & SDK

STM32 path: STM32CubeIDE + HAL for FDCAN/I²C/SPI (free). (Board features: FDCAN, many op-amps/comparators.) 
ST
os.mbed.com

NXP path: S32 Design Studio + RTD drivers (AUTOSAR/non-AUTOSAR), aligned with ISO 26262 workflows. 
NXP Semiconductors

PC tools: Python + Jupyter (logging & plotting), pybamm for physics-based battery simulation & OCV curves (super for teaching). 
GitHub
Journal of Open Research Software
pybamm.org

Open datasets (for SOC/SOH algorithm training offline):
NASA Prognostics Center Li-ion dataset; CALCE battery datasets (OCV tests, aging). 
Mouser Electronics
calce.umd.edu

CAN on PC: CANable 2.0 + candlelight/slcand/gs_usb; works with can-utils and Wireshark. 
canable.io
+1

4) Software architecture (teaching-oriented, safety-aware)

Tasks / modules

Drivers/HAL: BQ76952 (SPI/I²C), INA226 (I²C), FDCAN, GPIO, timers, NVM.

Sampler:

Current @ 100 Hz (INA226 + BQ76952 coulomb counter).

Per-cell voltages @ 10 Hz (BQ76952).

Temps @ 1 Hz.

SOC estimator:

Coulomb counting (primary), corrected by OCV-SOC table when at rest; optional EKF/UKF blend. (OCV-SOC modeling references & pitfalls—especially flat LFP OCV). 
MDPI
SpringerOpen
zitara.com

SOH estimator:

Capacity from full/partial cycles (coulombs over ΔSOC).

Impedance (Rint) from current-pulse tests (ΔV/ΔI at short windows).

Balancing manager (passive, v2):

Top-of-charge bleed with hysteresis + min idle current + min cell voltage thresholds; schedule to avoid fighting charger. BQ76952 supports autonomous or host-controlled balancing. 
Texas Instruments

Safety manager (training scope):

Over/under-voltage, over/under-temp, over-current, short-circuit => open MOSFETs to safe state (BQ76952 implements many protections in hardware). 
Texas Instruments

Comms + logging:

FDCAN frames (cells, pack, SOC, SOH, faults), CSV logging over USB CDC.

For LFP, voltage is flat across much of the SOC range—combine coulomb counting with OCV only near the “knees”; ensure ADC/AFE resolution is good (5 mV steps can be limiting). 
zitara.com

5) Concrete starter BOM (approx; pick one MCU path)

Minimum viable lab kit (Option A, ultra-low cost):

STM NUCLEO-G474RE dev board — ~US$22. 
Newark Electronics

TI TCAN1042 CAN transceiver (small breakout or your own board). 
Texas Instruments

CANable 2.0 USB-CAN — US$35 retail (clones ≈ US$19). 
Openlight Labs
makerbase3d.com

TI INA226 + shunt (e.g., 2–5 mΩ). 
Texas Instruments

BQ76952 platform:

Best value: fabricate Libre Solar BMS-C1 PCB and populate (BOM + manual provided). 
GitHub
libre.solar

Simple (costlier): buy BQ76952EVM (US$249–$299). 
Texas Instruments
DigiKey

4× LFP 18650 cells + holders; 10 A fuse; NTCs; MOSFET+resistor load.

(If you prefer the NXP toolchain, swap the NUCLEO for S32K144EVB.) 
NXP Semiconductors

6) Bring-up plan (safe & fast)

No-battery mode (bench): Power board; use resistor-divider harness to feed realistic per-cell voltages; verify BQ76952 comms, cell readings, thermistors. (TI app note notes caveats when testing balancing with dividers.) 
Texas Instruments
+1

SOC v1: enable INA226 + BQ76952 coulomb counter; integrate current @ 100 Hz; implement rest-detection, then OCV correction (OCV tables from PyBaMM or dataset-fit). 
PyBaMM Documentation

SOH v1: run scripted current pulses into your resistor load; compute Rint; track delivered capacity on deep cycles.

Logging + plots: Python notebooks to compare SOC truth vs estimation; use NASA/CALCE datasets to validate filters offline. 
Mouser Electronics
calce.umd.edu

Real 4S LFP pack: add fuse, low-current limits; repeat tests; compare INA226 vs BQ76952 coulomb counter drift and temperature dependence.

7) Expansion roadmap

v2 (12–16s + passive balancing):
Use BQ76952 balancing (autonomous or host-controlled). Add balancing strategy at top-of-charge and/or long idle, with guardbands to avoid charger fights. 
Texas Instruments

v3 (active balancing option):
Add ADI LTC3300-1 active balancer modules for advanced coursework (note: evaluation boards are expensive, but the IC itself is reasonable). 
Analog Devices
+1
mexico.newark.com

v4 (toward ISO 26262 processes):
Move to S32K + safety SBC (watchdog, undervoltage), redundant current sense (INA226 + ADC path), brownout/clock monitors, SW partitioning, safe-state & DTC reporting. 
NXP Semiconductors

8) Practical ISO 26262-friendly habits (even for a demo)

Hazard analysis (bench scope): thermal runaway (mitigate by LFP, fusing, current limits), shorts (fast cutoff via BQ76952), wiring mistakes (color-coded harness + keyed connectors). 
libre.solar

Safety mechanisms: independent over-current/short protection in AFE, periodic sensor plausibility (range & rate checks), watchdogs, safe state = open FETs. 
Texas Instruments

Verification: unit tests on estimators, “golden log replays” from NASA/CALCE, HIL with cell-sim harness before touching real cells. 
Mouser Electronics
calce.umd.edu

TL;DR (my pick)

Cheapest solid start: NUCLEO-G474RE + CANable 2.0 + INA226 + Libre Solar BMS-C1 (self-built) on a 4S LFP pack.

If you won’t assemble PCBs: substitute BQ76952EVM (higher cost).

Dev tools: STM32CubeIDE (or NXP S32DS), Python/Jupyter, PyBaMM, CAN-utils.

You’ll be ready to implement and demo SOC/SOH today and add balancing algorithms (passive now, active later) without re-buying the core stack.

If you want, I can sketch the firmware folder structure and a first SOC estimator (coulomb + OCV with rest detection) next.
