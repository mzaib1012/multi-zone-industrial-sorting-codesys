# Multi-Zone Industrial Sorting System Factory Simulation

An industrial-grade control application designed in **CODESYS V3.5** implementing the **IEC 61131-3** standard. This project simulates a multi-conveyor material handling and automated routing network using structured, modular software architecture. It features sequential state-machine execution, defensive fault tracking, debounced sensor monitoring, and a dynamic Human-Machine Interface (HMI).

---

## 🏗️ System Architecture & Program Structure

The codebase is organized following modern ISA-88 modular design paradigms to separate logical domains, simplify debugging, and minimize local deployment overhead:

```text
MultiZone_Sorting.project
├── 01_Core_Logic
│   └── PLC_PRG (PRG)          # Core Structured Text (ST) Sequential State Machine
├── 02_Data_Structures
│   └── Global_Variables        # System I/O, Timer Instances, Counter Registers
└── 03_Visualizations
    └── Visualization (VISU)    # Dynamic Operator HMI Control Deck Interface
