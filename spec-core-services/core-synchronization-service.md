# Core Synchronization Service


<!------------------------------------------------------------------------>


## Introduction

The Synchronization Service manages the time reference (TREF) and the correspondence between system time, radio time, and logical time (based on TU). It lets the layer above Core set and read TREF in system time, or set TREF to the RX timestamp of received frame for high-accuracy synchronization.

The first TREF in a network is established by the synchronization source. Its Core sets TREF through `CORE_Tref_Set_Command` before scheduling its first timeframe. After discovering a valid and synchronizable frame, every other device derives TREF from a received frame through `CORE_Tref_Sync_Command`.

The normative behavior is defined in sections [Commands](#commands) and [Events](#events).


### Related Documents

| Document | Description |
| -------- | ----------- |
| [Introduction](./introduction.md) | Introduction to the Core Services |
| [Core Overview](./core-overview.md) | Defines TREF and the three time bases (terminology in the [OpenSTX Glossary](../glossary.md)). |
| [Execution Service](./core-execution-service.md) | Provides the receive command handle consumed by `CORE_Tref_Sync_Command` |
| [RAL Time Service](../spec-ral-services/ral-time-service.md) | Source of radio time (`RalInstant`) used internally by Core |


<!------------------------------------------------------------------------>


## Commands


### `CORE_Tref_Set_Command`

**Summary:** Sets or modifies TREF from a system time instant supplied by the layer above Core.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `system_time_instant` | `SystemInstant` | — | The system time instant at which TREF is to be associated to. Core MUST treat this instant as TUC 0. |

**Description:**

`CORE_Tref_Set_Command` requests Core to establish or modify TREF. Core MUST capture the radio time that corresponds to `system_time_instant`, record the system time value, and associate TUC 0 at that moment. Core MUST also release the TFC it holds, if any, so that the next `CORE_Tref_Sync_Command` can set it from any received frame.

Core MUST emit exactly one `CORE_Tref_Set_CommandEndEvent` for each `CORE_Tref_Set_Command` processed.

<!-- If a timeframe is active, Core MUST emit `CORE_Tref_Set_CommandEndEvent` with `command_status = ERROR_TIMEFRAME_ACTIVE`, closing the command lifecycle. `CORE_Tref_Set_Command` establishes the initial TREF before the first timeframe. Once a timeframe has been scheduled, reassigning TREF by `CORE_Tref_Set_Command` is not available. Refinement of an established TREF uses `CORE_Tref_Sync_Command`. -->

If `system_time_instant` is not a valid `SystemInstant`, Core MUST emit `CORE_Tref_Set_CommandEndEvent` with `command_status = ERROR_INVALID_VALUE`, closing the command lifecycle.

The new TREF takes effect immediately.


### `CORE_Tref_Get_Command`

**Summary:** Reads the current TREF, returned as a system time value.

**Parameters:** None.

**Description:**

`CORE_Tref_Get_Command` requests the current TREF expressed in system time. Core MUST emit exactly one `CORE_Tref_Get_CommandEndEvent`.

If no TREF is established, Core MUST emit `CORE_Tref_Get_CommandEndEvent` with `command_status = ERROR_TREF_NOT_SET`, closing the command lifecycle.


### `CORE_Tref_Sync_Command`

**Summary:** Sets TREF to the radio time of a frame received by a completed `CORE_Schedule_Rx_Command` identified by its command handle.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_handle` | Unsigned Integer | A valid command handle | Identifies the completed `CORE_Schedule_Rx_Command` whose reception sets the new TREF. Its lifecycle MUST have closed. |

**Description:**

`CORE_Tref_Sync_Command` requests Core to set TREF based on a frame received with a given `CORE_Schedule_Rx_Command`. The command lifecycle MUST have closed. Core reads the RX timestamp (in radio time) of the referenced RX command, and computes TREF as the timestamp minus the TUC value received as part of the Core header, multiplied by the TU duration. The TU duration is derived from `coreTuSize`.

Where the received Core header carries a TFC, Core MUST also set the TFC of the timeframe in which the frame was received to that value, and Core holds a TFC from then on (see [Security](./core-overview.md#security)).

Core MUST emit exactly one `CORE_Tref_Sync_CommandEndEvent` for each `CORE_Tref_Sync_Command` processed.

If `command_handle` does not identify a `CORE_Schedule_Rx_Command` whose lifecycle has closed, Core MUST emit `CORE_Tref_Sync_CommandEndEvent` with `command_status = ERROR_INVALID_ACTION`, closing the command lifecycle.

If the Core header of the referenced RX action carries no TUC value, Core MUST emit `CORE_Tref_Sync_CommandEndEvent` with `command_status = ERROR_NOT_SYNCHRONIZABLE`, closing the command lifecycle.

While `coreSecurityEnabled` is `TRUE`, Core MUST also emit `CORE_Tref_Sync_CommandEndEvent` with `command_status = ERROR_NOT_SYNCHRONIZABLE`, closing the command lifecycle, if the referenced reception did not end with `command_status = SUCCESS`.

The new TREF takes effect immediately.


<!------------------------------------------------------------------------>


## Events


### `CORE_Tref_Set_CommandEndEvent`

**Soliciting command:** `CORE_Tref_Set_Command`

**Summary:** Reports the final outcome of a TREF update request.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_TIMEFRAME_ACTIVE`, `ERROR_INVALID_VALUE` | Outcome of the TREF update request. |
| `tref` | `SystemInstant` | Present only when `command_status = SUCCESS`. | The new TREF, expressed as a system time value. |

**Description:** Core emits `CORE_Tref_Set_CommandEndEvent` after setting or rejecting a TREF update. This event closes the command lifecycle.


### `CORE_Tref_Get_CommandEndEvent`

**Soliciting command:** `CORE_Tref_Get_Command`

**Summary:** Reports the current TREF as a system time value.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_TREF_NOT_SET` | Outcome of the read request. |
| `tref` | `SystemInstant` | Present only when `command_status = SUCCESS`. | The current TREF, expressed as a system time value. |

**Description:** Core emits `CORE_Tref_Get_CommandEndEvent` after reading or rejecting a TREF read. This event closes the command lifecycle.


### `CORE_Tref_Sync_CommandEndEvent`

**Soliciting command:** `CORE_Tref_Sync_Command`

**Summary:** Reports the final outcome of a sync-to-reception request.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_INVALID_ACTION`, `ERROR_NOT_SYNCHRONIZABLE`, `ERROR_INVALID_VALUE` | Outcome of the sync request. |
| `tref` | `SystemInstant` | Present only when `command_status = SUCCESS`. | The new TREF, expressed as a system time value. |

**Description:** Core emits `CORE_Tref_Sync_CommandEndEvent` after setting or rejecting a sync-to-reception. This event closes the command lifecycle.
