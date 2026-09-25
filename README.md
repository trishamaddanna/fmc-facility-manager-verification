# fmc-facility-manager-verification
Aerospace-grade VHDL verification suite and dual-role I2C protocol emulation for the Power Management FPGA (Facility Manager) in Flight Management Computer (FMC) 
# VHDL Verification & I2C Protocol Interconnect for Avionics Facility Manager

A safety-critical **VHDL testbench architecture and protocol emulation suite** for the 'Facility Manager' (FCM)—a dedicated Power Management block residing within a **Flight Management Computer (FMC)**. Developed and verified under strict aerospace validation standards, this project coordinates autonomous hardware power staging, multi-slave I2C topology mapping, and error-logging memory verification using automated VHDL testbenches.

---

## 📐 System Architecture & Interconnect Topology

The Facility Manager (FCM) functions as the hardware supervisor within the FMC framework. The system is structurally split across the **CPUIO** card (containing the CPU, networking processor, and the FCM) and the **Power IO** card (containing the BBEXT, BBMISC and BBSBE2), joined by a dedicated physical interface interconnect.

The system coordinates **5 distinct Micro-Facility-Manager (uFCM) Building Blocks**:
*   **BBCP (`BB0`):** Main Processing Core + Volatile DDR state saving infrastructure.
*   **BBNT (`BB1`):** Core Avionics Networking and physical channel management.
*   **BBEXT (`BB4`):** Extension Module overseeing peripheral discrete avionics I/Os.
*   **BBMISC (`BB5`):** Miscellaneous structural housekeeping channels.
*   **BBSBE2 (`BB2`):** Primary Avionics Voltage Source and power supply generation core.

---

## ⚡ 1. Dual-Role I2C Interface Engineering

The FCM handles power configuration parameters and system debugging via a dual-role I2C structure, processing two independent physical communication profiles concurrently:

### A. Master Mode: LTC Hardware Configuration Array
In Master Mode, the FCM drives transactions to monitor power supply rails using external Linear Technology controllers (**LTC 0 at address `0x50`**, **LTC 1 at address `0x52`**) embedded across each building block.

#### Message Frame Format
LTC transactions leverage an integrated 16-bit **Write Word** and **Read Word** format containing localized device sub-address command bytes:
*   **Write Word Sequence:**
    `[START] -> [Slave Addr + Wr(0)] -> [ACK] -> [Command Code] -> [ACK] -> [Data Byte Low] -> [ACK] -> [Data Byte High] -> [ACK] -> [STOP]`
*   **Read Word Sequence:**
    `[START] -> [Slave Addr + Wr(0)] -> [ACK] -> [Command Code] -> [ACK] -> [REPEATED START] -> [Slave Addr + Rd(1)] -> [ACK] -> [Data Byte Low] -> [ACK] -> [Data Byte High] -> [NACK] -> [STOP]`

#### Technical Explanation
The Master interface programs parameters directly to slave registers to establish target tracking windows:
*   **Output Back to Zero (OBZ):** Sets voltage and temperature safety parameters to catch hardware variations.
*   **Nominal Configuration:** Defines steady-state voltage values for the rails once the system initializes.

## 🚦 Core FSM State Machine Routing & Operational States

The underlying operational scheduling of the Facility Manager (FCM) is driven by a deterministic Finite State Machine (FSM) that bridges three distinct operational regions: **INIT** (Initialization), **OPER** (Normal Operations/Autotest), and **POWERDOWN** (Sequence Containment).

### 1. Initialization Phase (INIT Loop)
*   **RESET State:** Entered upon a physical low assertion on `FCM_RESET_N`. The system drives default baseline signals (`BBCP_FOZ_ENA = 0`, `BBNT_FOZ_ENA = 0`, `FCM_EN_IP = 0`). Once `FCM_PW_ON = '1'`, the machine routes into the configuration check loops.
*   **OBZ (Out of Bound Zone) State:** Initiates localized I2C write cycles to program the initial OBZ parameters down to the respective building block's LTC controllers. If an internal configuration validation fails or if an over-voltage (OV) condition is captured during this window, a fault flag triggers an immediate fallback to `IMMINENT SHUTDOWN`. If configuration passes safely, the state shifts on `OBZ_OK = '1'`.
*   **WAIT State:** Holds processing execution boundaries pending system status changes. Once the data buffer asserts `EFULL = '1'`, the FSM transitions directly into target initialization.
*   **INITIALIZATION State:** Continues nominal LTC controller configurations. It enforces strict setup rules, watching for proper time windows (`T_initialization`) and validating that `PROGRAM_OK = '1'` before pushing the system out into steady-state monitoring loops.

### 2. Operational Phase (OPER & AUTOTEST Loops)
*   **NORMAL State:** The steady-state runtime window for the avionics unit. The FCM periodically updates internal registers and continuously sweeps background status reads from the active LTC rails over the shared I2C channels.
*   **AUTOTEST State:** Triggered when software routines assert `CONTROL_AUTEST.LTC_AUTOTEST = '1'`. The FCM suspends baseline tasks to execute critical **BITS (Built-in Test Sequences)** over a designated duration (`T_Autotest`). If background parameter readings fail continuously (exceeding a maximum counter of 3 attempts), the FSM forces execution back to previous recovery loops.

### 3. Graceful Containment Phase (POWERDOWN Staging)
The FSM continuously guards active lines against critical failures. An immediate transition to the `IMMINENT SHUTDOWN` node is executed if any of the following hardware triggers trip:
*   A drop in primary input supply lines lasting over 200ms (`FCM_PW_ON = '0' for >200ms`).
*   Direct hardware over-voltage or over-temperature flag signals (`FCM_OVP_N` or `FCM_OTP_N` drive low).
*   Active manual hardware reset signals held for over 500ms (`FCM_MANUAL_RESET_N = '0' for 500ms`).
*   Repeated write anomalies where writing the LTC configuration parameters fails three consecutive times.

#### Adaptive Emergency Power-Down Staging paths:
*   **IMMINENT SHUTDOWN:** Instantly triggers an interrupt line (`FCM_PS_IRQ_N`) and freezes vital logs out into the `FMEM` flash partition block.
*   **SHORTCUT / POWERDOWN Staging:** Stepped intervals systematically collapse voltage lines based on failure severity timelines (`T_imminent` / `T_shortcut`), sequentially isolating peripheral components to safeguard the master processing core fabric from residual back-power damage.

---

### B. Slave Mode: FMEM Diagnostic Logging Interconnect
When executing ground troubleshooting or downloading failure metrics, the FCM functions as an **I2C Slave**, responding to queries initiated by an external I2C test controller (`i2c_cnt`).

#### Message Frame Format
Data offloading captures black-box records via **Byte/Page Write** and **Byte/Page Read** sequences:
*   **Byte/Page Write Format:**
    `[START] -> [Control Byte + Wr(0)] -> [ACK] -> [Address High Byte] -> [ACK] -> [Address Low Byte] -> [ACK] -> [Data Byte 0] -> [ACK] -> ... -> [Data Byte 127] -> [ACK] -> [STOP]`
*   **Byte/Page Read Format:**
    `[START] -> [Control Byte + Wr(0)] -> [ACK] -> [Address High Byte] -> [ACK] -> [Address Low Byte] -> [ACK] -> [REPEATED START] -> [Control Byte + Rd(1)] -> [ACK] -> [Data Byte (n)] -> [ACK] -> ... -> [Data Byte (n+x)] -> [NACK] -> [STOP]`

#### Technical Explanation
This secondary channel facilitates structural non-volatile diagnostic transfers:
*   **Log Extraction:** Emulates the extraction of Flash Memory (`FMEM`) history words stored during imminent hardware shutdown routines.
*   **Workspace Validation:** The testbench drives configuration values (baud rate, etc.) to evaluate page write alignment, ensuring critical diagnostics save accurately without data mismatch errors during emergency power severs.

---

## 🛠️ Subblock Descriptions & Power Management Logic

### Explanation of Building Blocks (uFCM Core Elements)
The Facility Manager splits its supervisor duties among independent operational sub-modules:
*   **FCM_MANAGER (`B0`):** The primary FSM engine that schedules chip-wide power-up, auto-tests, and power-down states based on CIDS criteria.
*   **uFCM Cells (`B1` to `B5`):** Individual logic blocks assigned to handle unique platform interfaces, processing independent rail flags for their respective hardware units.
*   **Debug/Memory Interfaces (`B7`, `B10`):** Specialized tracking bridges that control data lines leading to diagnostic loops and `FMEM` channels.

### uFCM Filter Block
*   **Functionality:** Designed to counter electrical noise and filter out signal bounces on raw voltage rail indicators.
*   **Mechanics:** Evaluates input line transitions against strict parameter bounds (`CLK_FREQ = 1000kHz`, `FIL_TIME = 1ms`). Transient spikes or glitches below these limits are blocked, keeping control loops stable.

### DDR Manager Block
*   **Functionality:** Manages volatile data preservation states for the processor's external memory.
*   **Mechanics:** Controls transitions between **Cut-Off Mode** (which safely clears out active blocks) and **Self-Refresh Mode** (which supplies low-level background power to preserve memory registers during unexpected supply losses).

### PS Management (Power Supply Management)
*   **Functionality:** Constantly checks primary voltage parameters and manages core communication links with the processor.
*   **Mechanics:** Emulates health parameters and generates system-wide interrupts (`FCM_PS_IRQ_N`) if faults occur. It distinguishes between a **Long Power Failure (LPF > 200ms)** and a **Very Long Power Failure (VLPF > 5s)** to adaptively trigger power-down staging.

---

## 🚦 Verification Scenarios & Coverage Automation

### Failure Testing & Power-Off Sequencing
The structural VHDL environment subjects the FSM to severe power rail drops to verify critical containment routines:
1.  **Reset Phase:** Asserts `FCM_MANUAL_RESET_N = 1` and systematically monitors internal voltage lines.
2.  **Staged Activation:** Verifies sequential delay pacing during power-up. For instance, `BBCP_EN_GRP0` fires first, followed by `GRP1` and `GRP2` after a 4ms delay, and `GRP3` after a 1ms step. This confirms the system prevents high current inrush loads.
3.  **Emergency Log Sequence:** Validates that if an LPF or VLPF state is caught, the system holds voltage gates active long enough for the processor to flash operating logs to `FMEM` before memory paths are completely cut off.

### Functional and Code Coverage Automation
*   **Scripted Infrastructure:** Verification execution is driven by **TCL scripting automation** within the simulator environment.
*   **Metric Extraction:** The scripts compile VHDL design hierarchies, inject automated test vectors, and compile full structural code coverage and functional feature checklists. This ensures complete compliance with strict aerospace reliability definitions.

---

## 📂 Repository Layout & Project Artifacts

Because the core RTL blocks contain sensitive aerospace IP, this repository focuses exclusively on **architectural verification layout diagrams, timing waveforms, and script assets** used to validate the chip:

*   `facility_manager_bd.jpg` — Block diagram illustrating the unified system architecture and internal sub-module partitioning.
*   `fmc_fsm.jpg` — Detailed implementation map layout documenting the core Finite State Machine state transitions.
*   `ufcm_filter_waveform.jpg` — Simulation trace checking signal assertion filtering against input glitches under custom constraints (`FIL_TIME`).
*   `ddr_manage_waveform.jpg` — Timing capture confirming correct DDR Cut-Off and low-power Self-Refresh activation handshakes.
*   `ps_management_waveform.jpg` — Signal log mapping core processing communications, interrupt assertions, and LPF/VLPF boundary checks.
*   `i2c_master_slaves_arch.jpg` — Hardware mapping showing the I2C Master (FCM) and dual Slaves (LTC 0, LTC 1) bus matrix layout across all 5 building blocks.
*   `i2c_msg_format_ltc.jpg` -> Schematic guide tracking specific start, control, ACK, and stop frame packet layouts for LTC communications.
*   `sim_waveform_bb0_write.jpg` — Functional simulation trace recording successful bus writes to the `BB0 - BBCP` sub-registers.
*   `sim_waveform_bb0_read.jpg` — Functional simulation trace tracking bus data loopback reads through the I2C channel.
*   `sim_waveform_bb0_obz_nominal.jpg` — Waveform trace verifying output-back-to-zero validation routines alongside active nominal configuration status checks.
*   `i2c_cnt_slave_fcm_arch.jpg` — Architecture blueprint illustrating the second I2C master channel interconnect (`i2c_cnt`) routing data into the FCM acting as a Slave device.
*   `i2c_msg_format_fmem.jpg` — Bit-level reference document detailing the byte and page formatting protocols required for `FMEM` flash memory offloads.
