# OpenSTX Glossary

This document provides a centralized definition of terms, concepts, acronyms and abbreviations used in the OpenSTX project.
It is a living document intended to establish a common vocabulary for all working group members and contributors.

The terminology is partially inspired by IEEE 802.15.4 (e.g., "services", "primitives").

## Acronyms and Abbreviations

This table provides a list of common acronyms and abbreviations and their full definitions.

| Short form | Definition              |
| :--------- | :---------------------- |
| DMA        | Direct Memory Access    |
| IC         | Integrated Circuit      |
| MCU        | Microcontroller Unit    |
| PHY        | Physical Layer          |
| RAL        | Radio Abstraction Layer |
| RX         | Reception               |
| SoC        | System on a Chip        |
| TX         | Transmission            |

## Glossary of Terms

| Term | Sub-term | Definition | Example | Notes |
| :--- | :--- | :--- | :--- | :--- |
| | | | | |
| **Action** | | A Core-layer operation, which involves a transmit (TX) or receive (RX) radio operation, executed within a **slot**. | A single transmission, a continuous RX window. | An action is the TX or RX operation executed within a slot by an active command. Not to be confused with radio operations, which do not expressly have a duration, as the concept (and the responsibility to maintain time alignment) resides in the Core layer. |
| **Active command** | | An accepted command that schedules a transmission or a reception, which remains active until its lifecycle closes. | An accepted `CORE_Schedule_Tx_Command`, `RAL_Rx_Schedule_Command`. | While active, the command can be cancelled and its events correlate to it. In the RAL at most one active command exists at a time; in the Core several TX or RX scheduling commands may be active at the same time. |
| *(Name to be defined, was Complex Action)* |  | A composite operation governed by a finite state machine (FSM) or logic sequence that orchestrates multiple actions. | A Glossy flood. | The actions that constitute it are not pre-determined: how Glossy unfolds in terms of TX/RX depends on the events occurring during the operation. |
| **Implementation** || The concrete realization of a specification. An implementation consists of the specific code that provides the functionality defined by the OpenSTX **primitives**. | | |
| **Layer** || A functional grouping of **services** that exists to separate concerns within the architecture. The working group is tasked with the definition of two layers, the **core layer** and the **RAL**. | | In IEEE 802.15.4, the MAC layer includes the data and management services. |
|| **Core layer** | The layer above the RAL, whose primitives introduce critical STX functionalities like time-slotting. | | |
|| **Lower/upper layer** | Respectively, a layer that sits below (i.e., provides **services** to) another layer, and a layer that sits above (i.e., uses the **services** of) another layer. | | |
|| **Radio abstraction layer (RAL)** | The lowest layer of the OpenSTX architecture (i.e., does not use the services of any other layer), responsible for providing an interface to various radio hardware, and abstracting platform-specific details. This layer provides the necessary functionalities defined by RAL primitives to support STX, but is only concerned with radio operations (i.e., TX, RX, timestamping, scheduling, or optional features like carrier-based drift estimation, but nothing else). | | |
| **Primitive** || An abstract, *implementation-independent* description of a specific functionality offered by a layer to an upper layer. Primitives define *what* functionality is provided, not *how* it is implemented. |  | Adopted from IEEE 802.15.4. Not to be confused with the term "primitive" often used in the STX community, where it indicates a "protocol", "approach", "system" that constitutes a building block for a more complex solution (e.g., Glossy is often called a "primitive"). To avoid confusion, use the term "primitive" exclusively as defined in this glossary. |
|| **Core primitive** | A primitive exposed by the **Core layer**, to the **Protocol layer** and other **upper layers**. |
|| **RAL primitive** | A primitive exposed by the **Radio abstraction Layer (RAL)** to the **Core layer**. | | |
| **Radio time** | | The time representation of the radio, used in RAL primitives as `RalInstant`. Radio time generally has the highest resolution of the time sources available to the Core layer. | | The primitives of the Core layer only accept time in TU representation, not radio time. |
| **Service** || A set of functionalities offered by one layer to another. A service is defined abstractly by a set of **primitives**. | | Adopted from IEEE 802.15.4. Typically comprises a Data service and a Management service (each associated to a sub-set of the layer primitives). |
| **Skip** | | A no-action **slot** that reserves a TU interval during which the radio is neither transmitting nor receiving from the perspective of the Core layer. The Core layer may put the radio to sleep in a skip slot. | | Used to place gaps in a schedule. Often used to fill the remainder of the protocol-allocated time units after the early termination of the protocol. |
| **Slot** | | A time interval, whose duration is an integer multiple of the **time unit (TU)**, enqueued into the current **timeframe**. Each slot starts at a given **time unit counter (TUC)** value and advances the TUC by its duration. A slot may be used to perform a TX/RX **action**, or be empty (**skip**). | | Introduced by the Core layer. The RAL has no notion of a slot. |
| **System time** | | A monotonic, always-on time base (unaffected by the power state of the radio). The Core layer may read system time from a platform time source supplied by the binding, but the system time source may coincide with the radio time source, provided system time requirements are respected. Upper layers may affect the time alignment of the Core layer by specifying a **time reference (TREF)** in system time. | | |
| **Time reference (TREF)** | | A moment in time used to align **timeframes**. TREF is represented simultaneously as a system time value, a radio time value, and TUC 0. The Core layer holds the correspondence between the three. The decision to set or modify TREF belongs to the layer above the Core layer. | | To enable the *synchronous* in synchronous transmissions, the involved devices must each set TREF values that refer to roughly the same instant in the common time base. |
| **Time unit (TU)** | | The fundamental discrete unit of time used to express durations and to sequence **slots**. | | The Core layer is responsible for the translation between TU intervals and radio time intervals. |
| **Time unit counter (TUC)** | | The scheduling offset within the current **timeframe**, expressed in **time units (TU)**. The TUC starts from 0, where TUC 0 corresponds to **time reference (TREF)**. The TUC is set to 0 when a scheduled timeframe is about to start, and advances by the duration in TU of each **slot** scheduled into it. | | |
| **Timeframe** | | A **time unit (TU)** interval of the schedule. Only one timeframe is active at a time. When a new timeframe is about to start (no pending actions to execute), the Core layer advances **time reference (TREF)** by the current timeframe duration and sets the **time unit counter (TUC)** to 0. | | Introduced by the Core layer. |