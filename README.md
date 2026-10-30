# fmc-facility-manager-verification
 
# FMC Facility Manager – VHDL Verification and I²C Protocol Emulation

**A VHDL verification environment and dual-role I²C protocol emulation suite for the Facility Manager (FCM), a power-management FPGA within a Flight Management Computer (FMC).**

This project focuses on simulation-based verification of the FCM's power-management functions, multi-device I²C communication, diagnostic memory access, and fault-handling sequences. The verification environment uses modular VHDL testbenches, protocol models, automated test scenarios, and TCL-based coverage analysis.

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

## ⚡ 1. Dual-Role I²C Interface Engineering

The Facility Manager (FCM) supports two distinct I²C operating roles for power-management configuration and diagnostic data access:

### A. Master Mode: LTC Hardware Configuration Array
In Master Mode, the FCM initiates I²C transactions with LTC 0 (0x50) and LTC 1 (0x52), which provide six voltage-monitoring channels each. These devices are distributed across the five hardware building blocks and support power-rail monitoring and configuration.

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
*   **OBZ (Output Back to Zero) State:** Initiates I²C write transactions to program the initial OBZ parameters into the LTC controllers associated with the respective building blocks. If an internal configuration validation fails or if an over-voltage (OV) condition is captured during this window, a fault flag triggers an immediate fallback to `IMMINENT SHUTDOWN`. If configuration passes safely, the state shifts on `OBZ_OK = '1'`.
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
* **IMMINENT SHUTDOWN:** Triggers an interrupt line (`FCM_PS_IRQ_N`) and preserves diagnostic logs in the FMEM flash partition.
* **SHORTCUT / POWERDOWN Staging:** Uses timed intervals (`T_imminent` / `T_shortcut`) to sequence power-rail shutdown and support controlled power-down and memory preservation.

---

### B. Slave Mode: FMEM Diagnostic Logging Interconnect
In Slave Mode, the FCM responds to transactions initiated by an external I²C test controller (`i2c_cnt`). This interface provides register-level access to the FCM and supports read/write transactions for retrieving and updating diagnostic data in the non-volatile Flash Memory (FMEM).

#### Message Frame Format
Data offloading captures black-box records via **Byte/Page Write** and **Byte/Page Read** sequences:
*   **Byte/Page Write Format:**
    `[START] -> [Control Byte + Wr(0)] -> [ACK] -> [Address High Byte] -> [ACK] -> [Address Low Byte] -> [ACK] -> [Data Byte 0] -> [ACK] -> ... -> [Data Byte 127] -> [ACK] -> [STOP]`
*   **Byte/Page Read Format:**
    `[START] -> [Control Byte + Wr(0)] -> [ACK] -> [Address High Byte] -> [ACK] -> [Address Low Byte] -> [ACK] -> [REPEATED START] -> [Control Byte + Rd(1)] -> [ACK] -> [Data Byte (n)] -> [ACK] -> ... -> [Data Byte (n+x)] -> [NACK] -> [STOP]`

#### Technical Explanation
This secondary channel facilitates structural non-volatile diagnostic transfers:
*   **Log Extraction:** Emulates the extraction of Flash Memory (`FMEM`) history words stored during imminent hardware shutdown routines.
*   **Workspace Validation:** The testbench drives diagnostic read/write transactions to verify page alignment, address handling, and data integrity during FMEM access.

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
2.  **Staged Activation:** Verifies sequential delay pacing during power-up. For instance, `BBCP_EN_GRP0` fires first, followed by `GRP1` and `GRP2` after a 4ms delay, and `GRP3` after a 1ms step. This verifies the intended power-up sequencing and staged activation of the hardware building blocks.
3.  **Emergency Log Sequence:** Validates that if an LPF or VLPF state is caught, the system holds voltage gates active long enough for the processor to write diagnostic logs to `FMEM` before memory paths are completely cut off.

### Functional and Code Coverage Automation
*   **Scripted Infrastructure:** Verification execution is driven by **TCL scripting automation** within the simulator environment.
*   **Metric Extraction:** The scripts compile the VHDL design hierarchy, execute automated test scenarios, and collect structural code-coverage metrics and functional coverage results. This ensures coverage measurement against the defined verification objectives.

---

## 🛠️ Toolchain Infrastructure & Verification Metrics

To meet avionics system verification requirements, professional, industry-standard electronic design automation (EDA) tools were utilized throughout the development and validation lifecycle:

*   **RTL Design and Verification Environment:** Developed the VHDL RTL modules, testbenches, and verification scripts using **Microsemi Libero IDE**. Executed functional simulations and waveform analysis in **QuestaSim**, with TCL scripts automating test execution and coverage analysis.
*   **Comprehensive Code Coverage Automation:** Advanced TCL scripting workflows were deployed within **QuestaSim** to systematically measure and report the following structural metrics:
    *   *Line Coverage:* Measures the execution of individual executable lines of RTL code.
    *   *Block Coverage:* Measures the execution of defined code blocks within the RTL hierarchy.
    *   *Branch Coverage:* Checks both `true` and `false` pathways for every conditional switch.
    *   *Condition Coverage:* Evaluates Boolean expression permutations within logical control blocks.
    *   *FSM State Coverage:* Verifies that every distinct operational state within the Finite State Machine core is successfully visited.
    *   *FSM Transition Coverage:* Verifies execution of all defined FSM state transitions.
    *   *Overall Verification Coverage:* Achieved 100% code and functional coverage across the defined verification scope.

---

## 📂 Repository Layout & Project Artifacts

Because the core RTL blocks contain sensitive aerospace IP, this repository focuses exclusively on **architectural verification layout diagrams, timing waveforms, and script assets** used to validate the chip:

*   `facility_manager_bd.png` — Block diagram illustrating the unified system architecture and internal sub-module partitioning.
*   `fmc_fsm.png` — Detailed implementation map layout documenting the core Finite State Machine state transitions.
*   `ufcm_filter_waveform.png` — Simulation trace checking signal assertion filtering against input glitches under custom constraints (`FIL_TIME`).
*   `ddr_manage_waveform.png` — Timing capture confirming correct DDR Cut-Off and low-power Self-Refresh activation handshakes.
*   `ps_management_waveform.png` — Signal log mapping core processing communications, interrupt assertions, and LPF/VLPF boundary checks.
*   `i2c_master_slaves_arch.png` — Hardware mapping showing the I2C Master (FCM) and dual Slaves (LTC 0, LTC 1) bus matrix layout across all 5 building blocks.
*   `i2c_msg_format_ltc.png` — Schematic guide tracking specific start, control, ACK, and stop frame packet layouts for LTC communications.
*   `sim_waveform_bb0_write.png` — Functional simulation trace recording successful bus writes to the `BB0 - BBCP` sub-registers.
*   `sim_waveform_bb0_read.png` — Functional simulation trace tracking bus data loopback reads through the I2C channel.
*   `sim_waveform_bb0_obz_nominal.png` — Waveform trace verifying output-back-to-zero validation routines alongside active nominal configuration status checks.
*   `i2c_cnt_slave_fcm_arch.png` — Architecture blueprint illustrating the second I2C master channel interconnect (`i2c_cnt`) routing data into the FCM acting as a Slave device.
*   `i2c_msg_format_fmem.png` — Bit-level reference document detailing the byte and page formatting protocols required for `FMEM` flash memory offloads.
