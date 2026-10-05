# Core Overview

This section describes the main features of the Core layer, how Core builds on the Radio Abstraction Layer (RAL), and the services Core exposes to its upper layers.

Core sits above the RAL and below the Protocol layer.

```
+-------------------------------+
| Protocol Layer                |
+-------------------------------+
| Core Layer                    |
+-------------------------------+
| Radio Abstraction Layer       |
+-------------------------------+
| PHY / RF Hardware             |
+-------------------------------+
```

While RAL is concerned only with radio operations and hardware abstraction, Core introduces the common time base, timeframes, slot scheduling, synchronization, and the common OpenSTX header. Note that Core provides scheduling primitives for the fundamental building blocks of OpenSTX, such as individual TX/RX actions. Multi-TX/RX sequences generally belong to the Protocol layer.

The operational model described here underpins the normative behavior specified in Core Services. It introduces the terminology used in command descriptions. Canonical definitions are in the [OpenSTX Glossary](../glossary.md).



## Active TX/RX commands and handles

An accepted `CORE_Schedule_Tx_Command` or `CORE_Schedule_Rx_Command` remains active until its TX/RX operation completes. Such a command is an **active command**. The upper layer may cancel it through `CORE_Cancel_Command`, and associates the resulting events to it. Core translates the TX/RX operation of each active command into RAL operations expressed in radio time and submits them through the [RAL Execution Service](../spec-ral-services/ral-execution-service.md). Core MAY hold several active commands at the same time, one per scheduled slot.

An action is the TX or RX operation executed within a slot by an active command.


### Command handles

Core allocates a **command handle** when a TX or RX scheduling command is accepted and reports it in `CORE_CommandStatusEvent`. The handle identifies the active command in `CORE_Cancel_Command` and correlates the events of that command. The handle of a transmit scheduling command is retired when the lifecycle of that command closes. The handle of a receive scheduling command remains valid after its lifecycle closes, so the upper layer can use it with `CORE_Tref_Sync_Command`. Core guarantees that at least the handle of the most recently completed receive command remains valid whenever a new entry is scheduled.


## Time Concepts

This section provides the necessary introduction to the basic time concepts adopted in OpenSTX. Further details on slots, schedules, and actions, are provided in dedicated sections.

Core works with three time counters in three time bases.
The OpenSTX time shared across nodes is based on the **time unit (TU)**, and its associated counter is the **time unit counter (TUC)**.
The other two time bases, associated to physical clocks, are **system time** and **radio time**.
All counters (for system time, radio time, and TUC itself) are expected to be increasing monotonically, except upon their known wraparound.


### Time unit (TU) and time unit counter (TUC)

The time unit (TU) provides the common time base in OpenSTX.
TU values depend on the OpenSTX time mode, but in most cases it corresponds to the period of a 32.768 kHz clock (roughly 30.52 microseconds).
<!--NOTE:Modes will be defined later-->

The time unit counter (TUC) is used by Core to identify a specific moment in time within a timeframe. Each TUC increment corresponds to 1 TU.

Through TUs, Core abstracts hardware-based time for all OpenSTX devices, orchestrating system time and radio time. 
The fundamental difference between system time and radio time is the requirement on their availability. 


### System time

System time is read from an always-on source. The system time source MUST be readable while the radio is asleep. Core may read system time from a platform time source supplied by the binding. However, the system time source may coincide with the radio time source, provided system time requirements are respected. 
This always-available time source allows the Core to maintain time alignment with other OpenSTX devices when radio time is not available.

The binding of Core to a concrete system time source is not specified in this normative text.

#### `SystemInstant`
The system time type, `SystemInstant`, is an Unsigned Integer whose width is defined by the binding. Its tick frequency is provided by the binding as the constant `SYSTEM_TIME_FREQUENCY`. `SystemInstant` values are monotonically non-decreasing, except for known wraparounds.


### Radio time

Radio time is the time representation of the radio, used in RAL primitives as `RalInstant`. Its associated counter is expected to be readable, and monotonically increasing, only while the radio is not in `SLEEP` state.
Radio time generally has the highest resolution of the time sources available to Core. The standard never requires radio time values to be exposed to upper layers.


### Conversions

To convert between the hardware time bases and TUs, Core needs the TU size expressed in radio time ticks and in system time ticks. The TU size is a Core configuration attribute. A system time source is mandatory for every Core implementation. Conversion between radio time and system time assumes a nominally fixed ratio given by the radio time tick frequency, which Core reads from the RAL configuration attribute `radioTimeFrequency`. Any relative drift of the radio clock from this nominal ratio introduces conversion error. The bound on this drift is PHY-dependent.


## Slots

A slot is a time interval whose duration is an integer multiple of the TU. The layer above Core requests slots in the order it wants them executed. Core enforces the boundaries of every slot: each slot starts at the current TUC, and Core advances the TUC by the slot's duration. The layer above Core never specifies a start time for a slot, since TUC is managed by Core.

A slot is either dedicated to an action or a skip: 
- An action slot contains an action, i.e., a TX or RX operation. At the physical level, the TX/RX operation is fully contained within the time interval of the slot.
- A skip slot contains no TX/RX operations; Core may request the radio to enter the `SLEEP` state for its duration.

Every slot belongs to the current timeframe. Scheduling a timeframe is mandatory before any slot can be scheduled.


## Timeframe

A timeframe is a bounded interval of the schedule, with a duration in TU. The Core lets the upper layer declare a TU budget for the timeframe and then fill it with slots, as long as timeframe duration is not exceeded. 

### Time reference (TREF)

The time reference (TREF) is the start of the current timeframe, that is, the start of its first slot. The TUC counts TUs from TREF, so TREF always corresponds to TUC 0. When a new timeframe starts, TREF refers to the start of the new timeframe and the TUC restarts from 0 (see [Timeframe transition procedure](#timeframe-transition-procedure)).

The TREF is established and updated by the Synchronization Service. TREF is a moment in time that Core can express in system time or radio time.

Each device sets a TREF that refers to roughly the same instant. Two ways to set TREF are supported, with different accuracy.

- **By system time.** The layer above Core supplies a system time value through `CORE_Tref_Set_Command`. Core captures the matching radio time and associates TUC 0. Accuracy is bounded by the system time tick granularity.
- **By reception.** The layer above Core asks Core to set TREF from a received frame through `CORE_Tref_Sync_Command`. The received action carries the frame's radio time timestamp. The layer above Core supplies the TU value it decoded from the frame header, which marks the transmission's position in the sender's TU sequence. Core back-computes TREF as the reception radio time minus the received TU multiplied by the TU duration. Because this uses the radio timestamp directly, it avoids the quantization of the slower system time clock. This mirrors how TSCH corrects a node's time base from the measured arrival time of a received frame.

A device MAY start a timeframe unsynchronized with respect to any network, to receive and attempt to find another node. It does so by setting a coarse local TREF through `CORE_Tref_Set_Command`, scheduling a timeframe, and scheduling a receive action inside it. Discovery procedures themselves are not part of the current standard.

### Timeframe transition procedure

These preconditions MUST hold before the first timeframe can be scheduled:
- A system time source is attached. Its tick frequency is provided by the binding as `SYSTEM_TIME_FREQUENCY`.
- `coreTuSize` is configured. It is writable only while no timeframe is active.
- A TREF is established by `CORE_Tref_Set_Command`. The value may be a purely local coarse system-time instant.

The first `CORE_Schedule_Timeframe_Command` takes effect at TREF and starts the schedule.

Starting from the second timeframe, Core automatically handles the transition.

The Core is expected to ready the system for the upcoming timeframe during the current timeframe, without disruption to the current slotframe.

Radio time must be available in advance w.r.t. the start of the timeframe. To ensure this requirement is respected, RAL exposes to Core (via a read-only configuration attribute) the time interval the radio takes to wake from sleep. 

However, TREF is not updated for the upcoming timeframe unless the current timeframe is fully allocated with slots.


## Schedule

The Core allows upper layers to schedule multiple slots, even if preceding skip or action slots have not yet been executed.
Therefore, several TX or RX commands may be active at the same time. 

### Slot order

Slot requests are managed by Core in FIFO order.
Note that upper layers cannot specify the starting TUC of a slot, therefore they cannot influence the ordering.

If an action slot is canceled (through `CORE_Cancel_Command`), Core internally treats the remainder of the canceled slot as a skip slot (the associated TU interval cannot be reused).
<!--CHECK: Possibly too rigorous, but simplest; we may consider allowing the replacement of a canceled slot if: (i) it was the most recent slot, AND (ii) it was not executed yet, for example-->

### Schedule capacity

An upper layer scheduling too many entries may exhaust Core resources. Core therefore enforces two independent limits, exposed as constant read-only configuration attributes.

The first limit is `coreUpcomingScheduleCapacity`: the maximum number of schedule entries that have not yet run to completion, where an entry is a slot (including skips) or a timeframe start. A skip generally counts toward `coreUpcomingScheduleCapacity`, but Core MAY optimize its internal schedule representation by removing a skip as soon as possible, as long as the removal does not affect schedule execution or upper layer logic. Entries that have not yet run to completion cannot be removed to make room for a new entry. When `coreUpcomingScheduleCapacity` is reached, Core MUST reject the new scheduling request.

The second limit is `coreExpiredScheduleCapacity`: the maximum number of completed slots Core retains in the schedule so their command handles stay valid (a receive command handle for `CORE_Tref_Sync_Command`, and CorePacket read-back). `coreExpiredScheduleCapacity` MUST be at least 1. Core MUST guarantee that, whenever a new entry is scheduled, the handle of the most recently completed TX or RX command is still valid. Core therefore retains at least the most recently completed slot and evicts older completed slots oldest-first when `coreExpiredScheduleCapacity` is exceeded, rendering the associated handle invalid.


## Frame format

Core introduces the common OpenSTX header. If necessary for broad compliance with existing PHY layers and radios, Core directly manages PHY-dependent header variants.

The OpenSTX packet carries a Core header followed by the payload from the upper layer. The Core header optionally carries the TUC value that marks the transmission position in the sender's schedule: the position of the PTA, that is, the TUC of the slot plus `coreTxOffset`. On reception, the Synchronization Service uses this value to align timeframes (see [Time reference (TREF)](#time-reference-tref)).

When `coreSecurityEnabled` is `TRUE`, Core sets `Security Enabled` and the frame is protected as described in [Security](#security).


```
       ┌────┬─────────┬──────────┬─────────┬──────────┬─────┬─────┬─────────┬─────────┬─────────┬───────┐
Field: │ FC │ Dst PAN │ Dst Addr │ Src PAN │ Src Addr │ TFC │ TUC │ Aux Sec │ Payload │   MIC   │  FCS  │
Octets:│ 2  │   0/2   │  0/2/8   │   0/2   │  0/2/8   │ 0/4 │ 0/3 │   0/4   │    D    │ 0/4/8/16│ 0/2/4 │
       └────┴─────────┴──────────┴─────────┴──────────┴─────┴─────┴─────────┴─────────┴─────────┴───────┘
             ◄------------ addressing ---------------►
        ◄----------------------- Core header H ----------------------------►
```
On a protected frame, the MIC follows the payload and precedes the FCS. In the following procedures, `F` is the RAL's `fcsLength`, the number of FCS octets the PSDU carries (see [§PSDULength](../spec-ral-services/ral-overview.md#psdulength)).


### Frame Control (FC) field

  * `bits 0-2 Frame Type` — `001`
  * `bit 3 Security Enabled` — `1` if the frame is protected (see [Security](#security)), `0` otherwise
  * `bit 4 TFC Present` — `1` if Timeframe Counter (TFC) present, `0` otherwise
  * `bit 5 TUC Present` — `1` if TU Counter (TUC) present, `0` otherwise
  * `bit 6 PAN ID Compression` — `1` if neither `Dest Addr Mode` or `Src Addr Mode` is `NONE` and `Dst PAN` is the same as `Src PAN`, `0` otherwise
  * `bit 7 Aux Sec Present` — `1` if Aux Sec present, `0` otherwise
  * `bit 8` — reserved
  * `bit 9` — reserved
  * `bits 10-11 Dest Addr Mode` — `0b00` for `NONE`, `0b10` for `SHORT`, `0b11` for `EXTENDED`
  * `bits 12-13 Src Addr Mode` — same as `Dest Addr Mode`
  * `bits 14-15 Frame Version` — `0b00`

#### PAN ID Compression field

This bit indicates whether the source PAN ID field is present (`1`) or omitted (`0`). 
Its value is `1` if and only if:
- `Dest Addr Mode` is not `NONE`, and
- `Src Addr Mode` is not `NONE`, and 
- `Dst PAN` is the same as `Src PAN`.
Its value is `0` otherwise.

### Addressing fields

  * `Dest PAN` — from parameter `CORE_Schedule_Tx_Command.dst_pan_id` (per-frame)
  * `Dest Addr` — from parameter `CORE_Schedule_Tx_Command.dst_addr` (per-frame)
  * `Src PAN` — from configuration attribute `corePanId` (local PAN), may be omitted following PAN ID Compression rules.
  * `Src Addr` — from `CORE_Schedule_Tx_Command.src_addr` where it is given and `src_addr_mode` is `SHORT`, otherwise from configuration attribute `coreShortAddress` or `coreExtendedAddress`, as `src_addr_mode` names

### Time fields

  * `TUC` — from `TUC_slot + coreTxOffset`, where `TUC_slot` indicates the TUC of the slot
  * `TFC` — the current timeframe number

### Auxiliary Security (Aux Sec) field

The Aux Sec field carries the security information that is not already included in the rest of the Core header, analogous to the Auxiliary Security Header of IEEE 802.15.4. It is present if and only if `Aux Sec Present` is `1`, and carries the four-octet IV seed used for the frame (see [Security](#security)).


## Security

Core can protect the frames it sends with the cipher of the [RAL Security Service](../spec-ral-services/ral-security-service.md). The upper layer enables this protection by writing a network key to `coreSecurityKey` and then setting `coreSecurityEnabled` (see the [Configuration Service](./core-configuration-service.md)). 
From then on, Core sets `Security Enabled` in every frame it transmits: the Core header is authenticated but sent in clear, so that a receiver can read it before the frame is checked, and the payload is encrypted and authenticated. The MIC that follows the payload is `coreMicSize` octets long. 
While security is enabled, Core accepts only protected frames.
The [Security Overview](../spec-security/security-overview.md) explains how the RAL, Core and the upper layer share security responsibilities.

Protection does not require any optional header field. A frame that carries the TFC, the TUC, a short source address and the Aux Sec field supplies every nonce component to a joining node that has the key and holds no TFC. A frame that omits some components is shorter, but only a node that knows the missing values can check it; the mechanism by which a node learns them, for example from a regular synchronization frame, is a responsibility of the protocol above Core.

### Nonce

AES-CCM requires a nonce that is never used twice with the same key. Core builds a 13-octet nonce from four values: the short address, the TFC and the TUC are written least significant octet first, and the IV seed as its four octets in order:

| Nonce octets | Value |
| ------------ | ----- |
| 0 to 3 | IV seed used for the frame |
| 4 to 5 | Short source address used for the frame |
| 6 to 9 | TFC of the timeframe in which the frame is sent |
| 10 to 12 | TUC of the frame, `TUC_slot + coreTxOffset` |

The nonce itself is never sent. The sender and the receiver each build it from values they know:

- The sender uses the supplied `src_addr` and `aux_sec` independently, falling back to its own short address and IV seed for each missing value, and the TFC of the timeframe and the TUC, `TUC_slot + coreTxOffset`, of the frame it transmits. A relay can supply the values of the frame it relays.
- A receiver that holds a TFC uses its own TFC and TUC. A receiver that holds none uses the TFC and TUC the frame carries.
- A receiver takes the IV seed and the short source address from the frame. Where the frame carries no IV seed or no short source address, it uses the values the upper layer supplied with `CORE_Schedule_Rx_Command`.

[`CORE_Schedule_Tx_Command`](./core-execution-service.md#core_schedule_tx_command) and [`CORE_Schedule_Rx_Command`](./core-execution-service.md#core_schedule_rx_command) state these rules normatively.

Because a receiver that holds a TFC uses its own TFC and TUC, a frame that is replayed in another slot or timeframe does not pass the check, unless the TFC has come back to the value it had then, and one frame can move the receiver's TREF by no more than its receive window.

A node holds a TFC once `CORE_Tref_Sync_Command` has set it from a received frame. The node that originates the TFC values of a network holds its TFC from its first protected transmission. `CORE_Tref_Set_Command` releases the TFC, so that the node can synchronize again from any frame.

### IV seed

When the TFC of a node starts again from a lower value, for example because the node or the whole network restarted, the TFC and TUC alone would repeat earlier nonces. This is prevented through the following mechanism.
When Core supplies its own seed, it draws a random 4-octet IV seed through `RAL_Crypto_Iv_Seed_Command` before its first use, and a new one before any frame whose TFC and TUC do not follow those of the previous frame it used that seed for (see [`CORE_Schedule_Tx_Command`](./core-execution-service.md#core_schedule_tx_command)).


### Rules for the upper layer

Protection works as described only if the upper layer follows these rules on a network whose frames are protected:

- The upper layer MUST give every node that transmits a short address that no other node of the network uses.
- The upper layer MUST let only one node of the network originate TFC values. Every other node MUST take its TFC from `CORE_Tref_Sync_Command` before it transmits.
- Frames whose nonces are built from the same source address, IV seed, TFC and TUC MUST be the same frame. Relays of one flood meet this rule.
- The upper layer SHOULD NOT let the TFC of a network come back to a value it has taken under the same key. When Core supplies a node's seed, each return makes it draw a new IV seed, and the node repeats a nonce once two of its draws are equal. The upper layer must prevent repetition itself when it supplies the seed.


## The `CorePacket` object

A `CorePacket` is a Core object used by the upper layer for data exchange. The upper layer exchanges frame data with Core exclusively through `CorePacket`, and never manipulates a `PacketBuffer` object directly.

Each CorePacket is associated with exactly one `PacketBuffer`, from its creation to its deletion. `CorePacket.data` refers to the payload octets of the PSDU in that `PacketBuffer`, or to a copy of them (see [From `CorePacket` to `PacketBuffer`](#from-corepacket-to-packetbuffer)).

Each `CorePacket` is characterized by the parameters below:

<a name="table-core-packet-parameters"></a>

**Table**: `CorePacket` Parameters.

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `data` | Octet String | — | Memory region that hosts the payload exchanged with the upper layer. |
| `data_len` | Unsigned Integer | — | Number of octets that carry information in `data`. |


### From `CorePacket` to `PacketBuffer`

The Core header and payload are partitioned into up to three contiguous regions, in order:

- not authenticated and not encrypted (optional),
- authenticated and not encrypted,
- authenticated and encrypted.

There is no region that is encrypted but not authenticated.

`CorePacket` hides the region layout from the upper layer. The upper layer only provides the payload to transmit, or reads back the payload that was received. The Core header sits at the start of the PSDU, followed by the payload. The size of the header varies with the header options it carries (for example the TUC). Which region each of them occupies depends on `Security Enabled`, as described in [Frame format](#frame-format).

`CorePacket.data` begins immediately after the Core header within the PSDU, and holds `CorePacket.data_len` payload octets. Core SHOULD use for `CorePacket.data` a sub-portion of the `PacketBuffer.buffer_data` memory space, to avoid unnecessary data copy operations.


## Core Services

<a name="table-core-specification-contents"></a>

**Table**: Core Services.

| Service | Description |
| ------- | ----------- |
| [Synchronization Service](./core-synchronization-service.md) | Set and read TREF (in system time), and align TREF based on RX timestamps for high-accuracy synchronization. |
| [Execution Service](./core-execution-service.md) | Schedule a timeframe, single TX and RX actions, or skip ahead for TU duration. Provides command handles for cancellation. Hosts the layer-wide `CORE_CommandStatusEvent`. |
| [Packet Manager Service](./core-packet-manager-service.md) | Create, fill, read, and delete `CorePacket` objects, each associated with a RAL `PacketBuffer`. |
| [Configuration Service](./core-configuration-service.md) | Read and modify the Core configuration attributes: TU size, upcoming schedule capacity, and expired schedule capacity. |
