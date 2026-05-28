# Multi-Zone Industrial Sorting System Factory Simulation

An industrial-grade control application designed in **CODESYS V3.5 (IEC 61131-3)** simulating an automated multi-zone conveyor routing and material handling network. This project demonstrates modular, production-ready PLC programming paradigms, covering sequential state machine development, defensive timing routines for fault tolerance, signal edge analysis, and dynamic Human-Machine Interface (HMI) integration.



## 🛠️ Deep-Dive Control Engineering Implementation

### 1. Sequential Master Control & Safety Latching
The system utilizes a dual-button latching routine to manage the global system authorization state (`xSystemActive`). This variable serves as an absolute software interlock; if an emergency stop or an active system fault trips, all physical downstream actuator states are dropped immediately.

* **Core Latching Expression:**
  ```pascal
  // Master control loop logic with hard stop override prioritization
  xSystemActive := (xStartButton OR xSystemActive) AND NOT xStopButton AND NOT xZone2JamAlarm;



### 2. Cascading Downstream Conveyor Sequencing

To prevent material handling pile-ups, part collisions, and mechanical damage at hardware junctions, the zone conveyors utilize cascading interlock logic. Upstream zones are strictly dependent on downstream clearance line readiness.

* **Zone 1 (Main Conveyor):** Operates exclusively if the system is activated and Zone 2 is clear.
* **Zone 2 (Sorting Conveyor):** Serves as the primary inspection zone; automatically stops if an downstream jam condition is flagged.

### 3. Defensive Jam Detection Logic (TON Framework)

To account for real-world field anomalies such as package slippage, physical blockages, or optical sensor fouling, a specialized non-blocking industrial timer block (`TON`) monitors target progression times across the photoeyes.

* **Operation:** If a box breaks the beam of the Zone 2 sensor (`xSensor_Zone2`) and stays stagnant for longer than **5.0 seconds**, the timer completes its cycle and sets a sticky system error flag.

```pascal
// Anti-jam surveillance block setup
fbZone2JamTimer(IN := xSensor_Zone2 AND xSystemActive, PT := T#5s);
xZone2JamAlarm := fbZone2JamTimer.Q;

```

### 4. Deterministic Pneumatic Divert Actuation (TP Framework)

A standard mistake in sorting applications is firing a divert gate using raw sensor state, which can lead to rapid valve chattering or clipping the rear edge of long packages. This architecture solves that by routing the target trigger through an IEC Time-Pulse (`TP`) timer block.

* **Operation:** Upon detecting the rising edge of an object at the sorting junction, the pneumatic cylinder (`xSortGateActuator`) is driven out and locked into position for a reliable window of exactly **2.0 seconds**, completely ignoring sensor fluctuations or trailing edge variations.

```pascal
// Pulse timing calculation for clean material diversion
fbGatePulseTimer(IN := xSensor_Zone2, PT := T#2s);
xSortGateActuator := fbGatePulseTimer.Q AND xSystemActive AND NOT xZone2JamAlarm;

```

### 5. High-Speed Production Telemetry Tracking

Throughput metrics are captured dynamically from the physical field layer via edge-detection function blocks (`R_TRIG`) paired with standard IEC Up-Counters (`CTU`).

* **Operation:** The system isolates the exact millisecond a box exits the final clearance gate, preventing false multiple-counts from a single package, and stores the live value directly into a Double-Integer (`DINT`) data register (`diTotalBoxes`).

---

## 🖥️ Human-Machine Interface (HMI) Control Deck

The system includes a fully dynamic supervisory visualization dashboard that interfaces directly with the live memory registers of the virtual runtime controller:

| HMI Element | Visual Object Type | Underlying Variable Mapping | Dynamic State Behavior |
| --- | --- | --- | --- |
| **START** | Button Input | `PLC_PRG.xStartButton` | Latches system into active operation on momentary tap. |
| **STOP** | Button Input | `PLC_PRG.xStopButton` | Master override; instantly forces system to safe state. |
| **RUN BEACON** | Animated Rectangle | `PLC_PRG.xSystemActive` | Shifting Fill Color:<br>

<br>• **Dark Gray** = System Idle/Off<br>

<br>• **Bright Green** = Line Operational |
| **FAULT BEACON** | Animated Rectangle | `PLC_PRG.xZone2JamAlarm` | Shifting Fill Color:<br>

<br>• **Dark Gray** = Nominal Operation<br>

<br>• **Bright Red** = Material Jam / Safety Fault |
| **GATE ACTUATOR** | Animated Ellipse | `PLC_PRG.xSortGateActuator` | Shifting Fill Color:<br>

<br>• **Light Gray** = Valve Retracted<br>

<br>• **Bright Orange** = Pneumatics Actuated (2s Pulse) |
| **THROUGHPUT METER** | Interpolated Text Block | `PLC_PRG.diTotalBoxes` | String variable interpolation (`Total Packets: %d`) printing live throughput directly on screen. |

---

## 🔍 Engineering Log & HMI Compiler Troubleshooting

During the design and testing phases of the graphic HMI control deck, two notable CODESYS compilation bugs were encountered and resolved:

### 1. Token Parse Errors in Color Variables (`C0009: Unexpected token 'Gray' found`)

* **Problem:** When attempting to hardcode background states directly into the element property field (e.g., typing `Dark Gray` or `Bright Green`), the compiler threw syntax exceptions.
* **Cause:** CODESYS property sheets do not accept unquoted string literals or custom plain-text descriptors directly inside the *Color Variables -> Toggle Color / Fill Color* configuration properties.
* **Resolution:** Static colors must be defined natively via the style dropdown list or configured via actual mapped program variables, while dynamic variations are handled cleanly by utilizing the explicit element properties menu structure.

### 2. Type Mismatch Exceptions (`C0032: Cannot convert type 'DINT' to 'BOOL'`)

* **Problem:** Mapping the live throughput counter `diTotalBoxes` to color-toggle control triggers caused standard type verification loops to crash out.
* **Cause:** The color-state visualization variables expect strict Boolean values (`TRUE`/`FALSE`) to flip between baseline and alarm colors, whereas the throughput registry holds a numerical Double Integer.
* **Resolution:** Decoupled data monitoring structures. Transferred numeric visualization tasks explicitly to text block properties using text variable interpolation modifiers (`%d`), leaving color change conditions tied strictly to discrete binary system tracking flags.

---

## 💻 Compilation, Environment, & Simulation Verification

* **IDE Environment:** CODESYS Development System V3.5 SP22 Patch 2
* **Language Compliance:** Structured Text (ST) & Function Block Integration (IEC 61131-3)
* **Target Architecture Engine:** CODESYS Control Win V3 x64 (Local Virtual PLC Instance)

### Execution & Verification Workflow:

1. Open the project within the CODESYS IDE and press **`F11`** to compile the codebase (**0 errors, 0 warnings**).
2. Initialize **Simulation Mode** inside the online configuration engine.
3. Perform an **Online -> Login** (`Alt + F8`) command with a download to write the compiled binaries to the local virtual memory registers.
4. Press **`F5`** to transition the PLC state machine into **`RUN`** mode.
5. Launch the `Visualization` tab to operate the control room deck interface. Toggle sensors manually within the watch tables to stress-test the anti-jam safety loops and the pneumatic timing sequences live on the graphic terminal canvas.

---

## 🏗️ System Architecture & IDE Project Tree

The logical software components are engineered inside a structured directory hierarchy following modern ISA-88 control system design principles to cleanly isolate sequencing logic, data declarations, and operator interfaces:

```text
Device (CODESYS Control Win V3 x64)
└── PLC Logic
    └── Application
        ├── 01_Core_Logic
        │   └── PLC_PRG (PRG)          # Core Structured Text (ST) Sequential Logic
        ├── 02_Data_Structures
        │   └── Global_Variables        # Global I/O Maps, Timers, and Telemetry Registers
        └── 03_Visualizations
            └── Visualization (VISU)    # Dynamic Operator Supervisory Control Panel

```

```

```