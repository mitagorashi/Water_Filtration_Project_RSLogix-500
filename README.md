# 💧 Automated Self-Cleaning Water Filtration System
### PLC + HMI Software-in-the-Loop Simulation | RSLogix 500 · RSLogix Emulate 500 · EasyBuilder 5000

<p align="center">
  <img src="docs/assets/gif/system_overview_demo.gif" alt="System Overview Demo" width="90%"/>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/PLC-RSLogix%20500-red?style=for-the-badge&logo=data:image/png;base64,iVBORw0KGgo=" />
  <img src="https://img.shields.io/badge/HMI-EasyBuilder%205000-blue?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Protocol-DF1%20Full%20Duplex-green?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Logic-Ladder%20(IEC%2061131--3)-orange?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Simulation-Software--in--the--Loop-purple?style=for-the-badge" />
</p>

---

## 📋 Table of Contents

- [Executive Summary](#-executive-summary)
- [System Architecture](#-system-architecture)
- [Software Stack & Tools](#-software-stack--tools)
- [PLC Program Structure](#-plc-program-structure)
- [HMI Design](#-hmi-design)
- [Communications Integration](#-communications-integration)
- [Simulation Setup Guide](#-simulation-setup-guide)
- [Screenshot Gallery](#-screenshot-gallery)
- [Demo Video](#-demo-video)
- [Key Engineering Challenges](#-key-engineering-challenges)
- [Project File Structure](#-project-file-structure)

---

## 🧠 Executive Summary

This project is an end-to-end industrial automation simulation of an **Automated Self-Cleaning Water Filtration System**. It demonstrates complete control system integration from PLC ladder logic to a fully operational HMI — without any physical hardware.

A **Software-in-the-Loop (SIL)** simulation environment was engineered by bridging a virtual PLC (RSLogix Emulate 500) with the HMI (EasyBuilder 5000) over a virtual null-modem RS-232 link using **com0com** and the **DF1 Full Duplex** protocol — successfully working around the data-sharing limitations of RSLinx Classic Lite.

**Core engineering areas demonstrated:**
- IEC 61131-3 Ladder Logic — state machines, safety interlocks, and HOA control
- Analog signal scaling — 4–20 mA transmitters to engineering units (PSI, %)
- Automated backwash sequencing triggered by differential pressure logic
- 8-screen HMI with role-based access control, animated P&ID, and real-time trending
- Industrial serial communications troubleshooting and virtual port bridging

---

## 🏗 System Architecture

<p align="center">
  <img src="docs/assets/thumbnails/4.png" alt="Full System Visualization" width="85%"/>
</p>

The physical process draws raw water through a media filter driven by a single pump (P-1) and routes it via six electrically actuated solenoid valves (SV-1 through SV-6).

### Process States

| State | Description | Active Devices |
|---|---|---|
| **Normal Flow (Filtration)** | Water drawn through SV-1 → P-1 → Filter → SV-5 → Storage Tank | P-1, SV-1, SV-5 |
| **Automated Backwash** | Reverse flow flushes filter when ΔP exceeds setpoint | P-1, SV-2, SV-4, SV-6 |
| **Standby / High Tank** | Pump inhibited when tank level is at HH setpoint | All OFF |
| **Fault / Interlock** | Low-flow detected — pump tripped to prevent cavitation | P-1 OFF, Alarm Active |

### Process Mode Diagrams

<p align="center">
  <img src="docs/assets/thumbnails/1.png" alt="Fill Mode Diagram" width="32%"/>
  &nbsp;
  <img src="docs/assets/thumbnails/2.png" alt="Drain Mode Diagram" width="32%"/>
  &nbsp;
  <img src="docs/assets/thumbnails/3.png" alt="Backwash Mode Diagram" width="32%"/>
</p>
<p align="center"><em>Left: Fill Mode — P-1 pumping to storage tank &nbsp;|&nbsp; Centre: Drain Mode &nbsp;|&nbsp; Right: Backwash Mode — reverse flow through filter</em></p>

### Instrumentation

| Tag | Type | Function | PLC Address |
|---|---|---|---|
| LT-1 | Analog Input (4–20 mA) | Storage tank level (%) | F8:2 |
| PT-1 | Analog Input (4–20 mA) | Filter inlet pressure (PSI) | F8:0 |
| PT-2 | Analog Input (4–20 mA) | Filter outlet pressure (PSI) | F8:1 |
| FS-1 | Digital Input | Flow switch — cavitation protection | B3:0/0 |
| ΔP Calc | Internal Float | PT-1 − PT-2 (computed in PLC) | F8:3 |

---

## 🛠 Software Stack & Tools

| Tool | Version | Role |
|---|---|---|
| RSLogix Micro Starter Lite | — | Ladder logic development environment |
| RSLogix Emulate 500 | — | Virtual SLC 500 / MicroLogix soft PLC |
| RSLinx Classic Lite | — | RSLogix ↔ Emulator programming link |
| EasyBuilder 5000 | — | HMI runtime and screen design (primary) |
| Wonderware InTouch (AVEVA) | — | HMI runtime and screen design (secondary) |
| com0com | — | Virtual null-modem serial port emulator |

> **No licensed OPC server was used.** The integration was achieved via native DF1 Full Duplex serial communication between the Emulator and HMI — a deliberate constraint that required engineering a custom virtual serial bridge.

---

## 📂 PLC Program Structure

The program is organized into **12 ladder rungs (LADs)** with strict separation of concerns:

```
MicroLogix Program
├── LAD 2  — Main Logic
├── LAD 3  — (Reserved / Expansion)
├── LAD 4  — Alarms
├── LAD 5  — Control (CTRL)
├── LAD 6  — HOA (Hand-Off-Auto)
├── LAD 7  — Simulation (SIM)
├── LAD 8  — Mode Selection
├── LAD 9  — Backwash Sequence
├── LAD 10 — Level Control
├── LAD 11 — Hourmeter
└── LAD 12 — Review (Diagnostic bit forcing)
```

---

<p align="center">
  <img src="docs/assets/screenshots/plc_program_overview.png" alt="RSLogix Program File List" width="75%"/>
</p>

> For the full rung-by-rung logic, see [`docs/plclogic.pdf`](docs/plclogic.pdf) and [`docs/address_symbol_database.pdf`](docs/address_symbol_database.pdf).

---

**LAD 2 — Main (Dispatcher):** The entry point. Contains no process logic itself — it is a clean JSR (Jump to Subroutine) dispatcher that calls every other LAD in sequence each scan: IO → Alarms → CTRL → HOA → SIM → MODE → Backwash → Level → Hourmeter → Review. A single overflow trap rung (`S:5/0`) prevents a math fault from halting the processor. This architecture keeps each function completely isolated and makes the program easy to navigate.

**LAD 3 — IO:** Maps virtual I/O to real process tags. Because the program runs entirely in Emulate with no physical wiring, real `O:` and `I:` addresses are not used. Instead, output bits in `B3:0` and `B9:0` act as virtual field devices, and `N10:0–2` holds the raw analog counts. This ladder performs the SCP (Scale with Parameters) instruction to convert the 0–16383 raw integer range of each analog input into engineering units: PT-1 and PT-2 scale to 0–20 PSI (`F8:0`, `F8:1`), and LT-1 scales to 0–100% (`F8:2`). Clamping logic using LES/GRT+MOV instructions guards against out-of-range values. Differential pressure is then continuously computed as `|F8:0 − F8:1|` using SUB and ABS instructions into `F8:3`.

**LAD 4 — Alarms:** Implements all four system alarms using a consistent TON-gated, ONS-latched pattern — a 5-second confirmation delay filters out transient spikes before a fault is declared. Each alarm has two bits: a latching **Alarm Bit** (the fault memory) and a **Notification Bit** (the HMI indicator, which can be silenced via `B3:3/10` without clearing the fault). A composite `Critical Alarm Present` bit (`B3:4/15`) is set whenever the High Pressure or Low Flow alarm bits are active — this single bit is used downstream by MODE and BACKWASH to safely inhibit the system.

| Alarm | Trigger Condition | Notification Bit |
|---|---|---|
| High Pressure | PT-1 or PT-2 > `F8:5` (15 PSI) for 5 s | `B3:4/8` `PRES_HH_ALARM` |
| Low Flow | P-1 running but FS-1 open for 5 s | `B3:4/10` `LOW_FLOW_ALARM` |
| Tank Low-Low | LT-1 < `F8:6` (10%) for 5 s | `B3:4/12` `LEVEL_LL_ALARM` |
| Tank High-High | LT-1 > `F8:7` (90%) for 5 s | `B3:4/14` `LEVEL_HH_ALARM` |

**LAD 5 — CTRL:** Resolves HOA status to actual output bits. Each device rung reads its N7 status register with EQU instructions and drives the corresponding `B3:0` output bit. The logic for SV-1, SV-2, SV-4, and SV-6 uses `HOA = HAND (1)` OR the system/backwash running bits to energize — meaning AUTO mode defers entirely to the process logic in MODE and BACKWASH. SV-3 and SV-5 additionally allow AUTO energization outside backwash. The pump rung is the most complex: it allows HAND, or AUTO with the `Tank Filling` bit (`B3:5/4`) or the `Backwash Running` bit — while `System Running` (`B3:3/8`) must be present in all AUTO paths.

**LAD 6 — HOA:** Manages the HOA status registers for all 7 devices (SV-1 through SV-6 + P-1) stored in `N7:0–N7:6`. Each device has three pushbutton inputs (OFF/HAND/AUTO mapped to `B3:0–B3:1` bits) that use ONS (One-Shot Rising) to trigger a single MOV instruction writing 0, 1, or 2 into the N7 register. A `First Pass` contact initializes all devices to OFF (0) on controller power-up. The final rung checks whether all 7 devices are in AUTO (value = 2) and sets `B3:5/5` (All HOAs in AUTO) — a prerequisite the MODE ladder requires before the system is allowed to start.

**LAD 7 — SIM:** Generates dynamic, physically plausible analog signals so the rest of the logic behaves as it would with real instrumentation. A 500 ms repeating TON timer (`T4:9`) gates all simulation calculations. On each tick: the raw level count in `N10:2` increases (+80 counts) when the pump is running in normal mode (tank filling), decreases (−12) during backwash, and drains (−17) when the system is running without the tank filling. A separate float register `F8:10` accumulates filter resistance (+6.0 per tick when running, −22.0 during backwash, with a floor of 400.0). Pressure transmitter raws in `N10:0` and `N10:1` are computed from the resistance accumulator to simulate differential pressure building realistically across the filter. A 2-second flow switch close delay (`T4:10`) simulates the real-world lag between pump start and flow confirmation.

**LAD 8 — MODE:** Controls the `System Running` bit (`B3:3/8`) using a standard latch/unlatch pattern. A STOP pushbutton (`B3:3/2`) or `First Pass` sets a RUN Interrupt bit; a START pushbutton (`B3:3/3`) sets a RUN Trigger bit. The main latch rung energizes `SYS_RUNNING` when the trigger is set, and holds it through its own contact — but drops out immediately if the RUN Interrupt fires, if `All HOAs in AUTO` (`B3:5/5`) is not true, or if `Critical Alarm Present` (`B3:4/15`) is active. This means the system cannot start with any device in HAND or OFF, and any critical fault drops the system out of RUN immediately.

**LAD 9 — BACKWASH:** The most complex ladder in the program. Backwash can be locked out or enabled via HMI pushbuttons (`B3:6/0`, `B3:6/1`). A pending backwash is triggered when differential pressure exceeds the setpoint `F8:4` (5.0 PSI) for 5 seconds — or manually via `MAN_BW_PB` (`B3:5/14`). The pending state latches in `B3:4/5` and only converts to an active backwash (`B3:4/6` `BACKWASH_RUNNING`) once the tank level confirms it is above the high setpoint `F8:9` (80%) — ensuring there is sufficient water volume to complete the flush. The backwash runs for a fixed 60-second TON (`T4:6`), with remaining time computed for HMI display (`N7:7 = 60 − T4:6.ACC`). A critical fault during the cycle immediately drops `BACKWASH_RUNNING`. Cycle count is tracked in `N7:8` and can be reset from the HMI behind a 5-second hold (`T4:11`). A 500 ms pulse bit (`B3:6/7` `LOG_BW_DATA`) is generated on each backwash start for external data logging.

<p align="center">
  <img src="docs/assets/gif/backwash_sequence.gif" alt="Backwash Sequence Live" width="80%"/>
</p>
<p align="center"><em>Live: Backwash sequence triggering in RSLogix — rung highlighting as ΔP threshold is crossed</em></p>

**LAD 10 — LEVEL:** Controls tank fill cycles by toggling `Tank Filling` (`B3:5/4`). When LT-1 drops below the low setpoint `F8:8` (20%) and the system is running, a 2-second confirmation delay (`T4:7`) triggers the `Fill Start` latch. When LT-1 rises above the high setpoint `F8:9` (80%), a 2-second delay (`T4:8`) unlatches `Tank Filling`. This bit is the signal CTRL uses to run the pump in AUTO mode during normal filtration.

**LAD 11 — HOURMETER:** Tracks total system runtime using a retentive timer RTO (`T4:13`, 60-second preset). The `Log Time` bit (`B3:6/8`) enables the RTO only when the system is running and no backwash is active — counting pure filtration hours. Each time the RTO reaches 60 seconds it increments `N7:10` (minutes) and resets. When minutes reach 60, `N7:11` (hours) increments and minutes subtract 60. When hours reach 1000, `N7:12` (kilo-hours) increments. Reset zeroes all four registers and resets the RTO — protected by a 5-second hold timer (`T4:14`) to prevent accidental resets.

**LAD 12 — REVIEW:** Contains no control logic. It consolidates key system bits and registers into a single visible rung so the programmer can monitor and force values during testing without navigating between ladders. The comment in the program file states it explicitly: *"This ladder doesn't do ANYTHING."* In a production deployment this file would be removed.

---

## 🖥 HMI Design

Developed as an 8-screen interface in **EasyBuilder 5000**, engineered around operator UX and functional safety principles.

### Screen Overview

| # | Screen Name | Key Features |
|---|---|---|
| 1 | Welcome / Main Dashboard | Animated flow direction, ΔP readout, dynamic tank fill visual |
| 2 | HOA Control Matrix | Multi-State Indicators tied to N7 registers — Hand/Off/Auto per device |
| 3 | Security & Access Control | 3-tier user authentication — Operator / Maintenance / Engineering |
| 4 | Runtime & Maintenance | Hourmeter display, backwash cycle count, credential-gated reset buttons |
| 5 | Configuration & Setpoints | Live setpoint editing for tank levels (F8:8, F8:9) and backwash SP (F8:4) |
| 6 | Monitoring / P&ID Diagram | Animated valve states, color-coded pipeline flow paths, live process values |
| 7 | Alarm Management | Alarm banner, event log, Silence (B3:3/10) and Reset (B3:3/9) controls |
| 8 | Real-Time Trending | 4-channel trend: ΔP, Tank Level, PT-1, PT-2 |

### Screen Screenshots

<p align="center">
  <img src="docs/assets/screenshots/hmi_01_dashboard.png" alt="HMI Main Dashboard" width="45%"/>
  &nbsp;
  <img src="docs/assets/screenshots/hmi_06_pid.png" alt="HMI P&ID Screen" width="45%"/>
</p>
<p align="center"><em>Left: Main Dashboard with animated flow & tank level | Right: P&ID Monitoring Screen</em></p>

<p align="center">
  <img src="docs/assets/screenshots/hmi_07_alarms.png" alt="HMI Alarm Screen" width="45%"/>
  &nbsp;
  <img src="docs/assets/screenshots/hmi_08_trending.png" alt="HMI Trending Screen" width="45%"/>
</p>
<p align="center"><em>Left: Alarm Management with event log | Right: Real-Time 4-Channel Trend</em></p>

<p align="center">
  <img src="docs/assets/screenshots/hmi_02_hoa.png" alt="HMI HOA Matrix" width="45%"/>
  &nbsp;
  <img src="docs/assets/screenshots/hmi_05_config.png" alt="HMI Configuration Screen" width="45%"/>
</p>
<p align="center"><em>Left: HOA Control Matrix | Right: Setpoint Configuration (role-protected)</em></p>

<p align="center">
  <img src="docs/assets/gif/hmi_pid_animated.gif" alt="HMI P&ID Live Animation" width="80%"/>
</p>
<p align="center"><em>Live: HMI P&ID screen — valve states and pipeline flow paths updating in real time</em></p>

---

## 🖥 HMI Design — Wonderware (AVEVA) InTouch

The same 8-screen interface was independently rebuilt in **Wonderware InTouch** (AVEVA), demonstrating that the control system design is platform-agnostic and not tied to any single HMI vendor. All screens, tag bindings, alarm workflows, HOA interactions, and setpoint configurations are identical in behaviour — only the development environment differs.

> Both HMI projects communicate with the same RSLogix Emulate 500 program via the same DF1 Full Duplex virtual serial bridge. No PLC logic was changed between implementations.

**▶ [Full Wonderware Workflow Demo on YouTube](https://youtu.be/E_H0Ll977O4)**

### Screen Gallery

<p align="center">
  <img src="docs/assets/screenshots/ww_01_dashboard.png" alt="Wonderware Dashboard" width="45%"/>
  &nbsp;
  <img src="docs/assets/screenshots/ww_06_pid.png" alt="Wonderware P&ID" width="45%"/>
</p>
<p align="center"><em>Left: Main Dashboard | Right: P&ID Monitoring Screen</em></p>

<p align="center">
  <img src="docs/assets/screenshots/ww_07_alarms.png" alt="Wonderware Alarms" width="45%"/>
  &nbsp;
  <img src="docs/assets/screenshots/ww_08_trending.png" alt="Wonderware Trending" width="45%"/>
</p>
<p align="center"><em>Left: Alarm Management | Right: Real-Time Trending</em></p>

<p align="center">
  <img src="docs/assets/screenshots/ww_02_hoa.png" alt="Wonderware HOA" width="45%"/>
  &nbsp;
  <img src="docs/assets/screenshots/ww_05_config.png" alt="Wonderware Config" width="45%"/>
</p>
<p align="center"><em>Left: HOA Control Matrix | Right: Setpoint Configuration</em></p>

<p align="center">
  <img src="docs/assets/screenshots/ww_03_security.png" alt="Wonderware Security" width="45%"/>
  &nbsp;
  <img src="docs/assets/screenshots/ww_04_maintenance.png" alt="Wonderware Maintenance" width="45%"/>
</p>
<p align="center"><em>Left: Access Control | Right: Runtime & Maintenance</em></p>

<p align="center">
  <img src="docs/assets/gif/ww_workflow.gif" alt="Wonderware Full Workflow" width="80%"/>
</p>
<p align="center"><em>Live: Full system workflow in Wonderware InTouch</em></p>

---

## 🔌 Communications Integration

<p align="center">
  <img src="docs/assets/screenshots/comms_architecture.png" alt="Communications Architecture" width="75%"/>
</p>

### The Challenge

**RSLinx Classic Lite** does not expose a DDE or OPC server interface. Third-party HMI software — including EasyBuilder 5000 — cannot pull PLC tags through it. This is a hard software limitation, not a configuration issue.

### The Solution — Virtual Serial Bridge

```
RSLogix Emulate 500          com0com             EasyBuilder 5000
  (Virtual SLC 500)       (Virtual Cable)           (HMI Runtime)
        │                       │                        │
     COM5 ──────────────── COM5 ↔ COM6 ────────────── COM6
        │                                               │
   DF1 Full Duplex                              AB Serial Driver
   19200 Baud                                  19200 Baud
   Even Parity                                 Even Parity
   Station 1 (Node)                            Station 1 (Node)
   CRC Error Check                             CRC Error Check
```

**RSLinx Classic Lite** is used exclusively for the **RSLogix ↔ Emulator** programming connection — it plays no role in the HMI communication path.

**Critical configuration details that must match exactly on both sides:**

| Parameter | Value |
|---|---|
| Protocol | DF1 Full Duplex |
| Baud Rate | 19200 |
| Parity | Even |
| Stop Bits | 1 |
| Node Address | Station 1 |
| Error Check | CRC |

---

## ⚙️ Simulation Setup Guide

Follow these steps in order to replicate the simulation environment.

### Prerequisites

- RSLogix Micro Starter Lite (installed, no license restrictions for simulation)
- RSLogix Emulate 500
- RSLinx Classic (any version — Lite is sufficient)
- EasyBuilder 5000
- [com0com](https://sourceforge.net/projects/com0com/) virtual serial driver

### Step 1 — Install & Configure com0com

1. Install com0com and open **Setup Command Prompt**
2. Create a virtual COM port pair:
   ```
   install PortName=COM5 PortName=COM6
   ```
3. Verify both ports appear in **Device Manager → Ports (COM & LPT)**

### Step 2 — Configure RSLogix Emulate 500

1. Open RSLogix Emulate 500
2. Create a new controller slot
3. Set **Channel 0** configuration:
   - Driver: **DF1 Full Duplex**
   - COM Port: **COM5**
   - Baud: **19200**, Parity: **Even**, Stop Bits: **1**
   - Node: **1**, Error Check: **CRC**
4. Start the controller

### Step 3 — Connect RSLogix & Download Program

1. Open RSLinx Classic — configure an **RS-232 DF1** driver pointing to the same COM5
2. Open the project `.RSS` file in RSLogix Micro
3. Go Online → Download to the Emulate 500 controller
4. Place controller in **RUN** mode

### Step 4 — Configure EasyBuilder 5000

1. Open the HMI project `.exob` file in EasyBuilder 5000
2. Go to **System Parameters → Device Settings**
3. Select device: `Allen-Bradley` → Driver: `AB_SLC500_DF1`
4. Set COM Port: **COM6**
5. Match baud/parity/node settings exactly as listed above
6. Compile and launch **Off-line Simulation** (or download to HMI hardware)

### Step 5 — Verify Live Connection

- Confirm green communication status in EasyBuilder
- Navigate to the **P&ID screen** — process values should be live
- Use **LAD 12 (Review)** in RSLogix to force test bits and observe HMI response in real time

---

## 📸 Screenshot Gallery

> Screenshots should be captured at 1920×1080. Use the naming convention below when adding files.

| File Name | Description |
|---|---|
| `plc_program_overview.png` | RSLogix Program File List showing all 13 files |
| `hmi_01_dashboard.png` | HMI Screen 1 — Main dashboard with animated tank |
| `hmi_02_hoa.png` | HMI Screen 2 — HOA Control Matrix |
| `hmi_03_security.png` | HMI Screen 3 — Login / Access Control |
| `hmi_04_maintenance.png` | HMI Screen 4 — Runtime hours & backwash counter |
| `hmi_05_config.png` | HMI Screen 5 — Setpoint configuration |
| `hmi_06_pid.png` | HMI Screen 6 — Animated P&ID diagram |
| `hmi_07_alarms.png` | HMI Screen 7 — Alarm management & event log |
| `hmi_08_trending.png` | HMI Screen 8 — Real-time 4-channel trend |
| `comms_architecture.png` | Communications bridge diagram (com0com topology) |
| `architecture_diagram.png` | Full system architecture overview |

---

## 🎬 Demo Video

<p align="center">
  <a href="YOUR_VIDEO_LINK_HERE">
    <img src="docs/assets/thumbnails/4.png" alt="Watch Demo Video" width="60%"/>
  </a>
</p>

> **▶ [Click to Watch Full Demo on YouTube](YOUR_VIDEO_LINK_HERE)**

The demo walkthrough covers:
1. RSLogix Emulate 500 running in RUN mode
2. Live ladder rung highlighting during normal filtration
3. Backwash sequence trigger — automatic ΔP-based activation
4. HMI P&ID screen with real-time animated valves and pipeline states
5. Alarm trigger and acknowledgment workflow
6. HOA override — forcing a device to HAND from the HMI
7. Setpoint change via Configuration screen and live PLC response

---

## 🔧 Key Engineering Challenges

### 1. Bypassing RSLinx Classic Lite Limitations
RSLinx Classic Lite has no OPC/DDE server capability, making direct HMI tag bridging impossible through the standard path. The solution was routing DF1 Full Duplex communication directly between the Emulator and EasyBuilder via a virtual null-modem cable — completely bypassing RSLinx for the HMI data path while retaining it for RSLogix programming.

### 2. Analog Simulation Without Hardware
With no physical 4–20 mA transmitters, LAD 7 was engineered to generate mathematically consistent simulated values that respond dynamically to pump and valve states. This required careful float register management to ensure the backwash trigger logic, level control, and trending all received realistic data.

### 3. Non-Interruptible Backwash Sequencing
Implementing a safe, non-interruptible step sequence using only TON timers and latch bits (no SFC) in RSLogix 500 Ladder requires careful use of OSR (One-Shot Rising) instructions to advance steps without re-triggering. The sequence must not allow partial execution — a half-completed backwash risks media damage in a real system.

### 4. HOA Architecture with Fault Override
The HOA structure needed to allow HAND mode to energize a device for maintenance while still being tripped by hard safety faults. This was implemented as: `HAND bit OR AUTO bit` in series with a `NOT fault bit` — meaning no HOA state can override a live safety interlock.

---

## 📁 Project File Structure

```
water-filtration-plc-hmi/
│
├── README.md
│
├── plc/
│   └── water_filtration_system.RSS           # RSLogix 500 project file
│
├── hmi/
│   ├── easybuilder/
│   │   └── water_filtration_hmi.exob         # EasyBuilder 5000 project file
│   └── wonderware/
│       └── water_filtration_hmi.aap          # Wonderware InTouch project file
│
└── docs/
    ├── plclogic.pdf                           # Full RSLogix Micro project report (all rungs)
    ├── address_symbol_database.pdf            # Complete tag/symbol reference
    └── assets/
        ├── screenshots/
        │   ├── plc_program_overview.png       # RSLogix program file list
        │   ├── hmi_01_dashboard.png           # EasyBuilder — Dashboard
        │   ├── hmi_02_hoa.png                 # EasyBuilder — HOA Matrix
        │   ├── hmi_03_security.png            # EasyBuilder — Access Control
        │   ├── hmi_04_maintenance.png         # EasyBuilder — Maintenance
        │   ├── hmi_05_config.png              # EasyBuilder — Setpoints
        │   ├── hmi_06_pid.png                 # EasyBuilder — P&ID
        │   ├── hmi_07_alarms.png              # EasyBuilder — Alarms
        │   ├── hmi_08_trending.png            # EasyBuilder — Trending
        │   ├── ww_01_dashboard.png            # Wonderware — Dashboard
        │   ├── ww_02_hoa.png                  # Wonderware — HOA Matrix
        │   ├── ww_03_security.png             # Wonderware — Access Control
        │   ├── ww_04_maintenance.png          # Wonderware — Maintenance
        │   ├── ww_05_config.png               # Wonderware — Setpoints
        │   ├── ww_06_pid.png                  # Wonderware — P&ID
        │   ├── ww_07_alarms.png               # Wonderware — Alarms
        │   ├── ww_08_trending.png             # Wonderware — Trending
        │   └── comms_architecture.png
        ├── gif/
        │   ├── backwash_sequence.gif
        │   ├── hmi_pid_animated.gif
        │   └── ww_workflow.gif                # Wonderware full workflow
        └── thumbnails/
            ├── 1.png                          # Fill mode diagram
            ├── 2.png                          # Drain mode diagram
            ├── 3.png                          # Backwash mode diagram
            └── 4.png                          # Full system visualization
```

---

## 📄 License

This project is shared for portfolio and educational purposes.

---

<p align="center">
  <strong>Mohamed</strong> — Automation & Controls Engineer<br/>
  <em>MSc Electrical & Electronics Engineering (Automation Specialization)</em>
</p>
