# Core Packet Manager Service


<!------------------------------------------------------------------------>


## Introduction

The Packet Manager manages the lifecycle of a [§CorePacket](./core-overview.md#the-corepacket-object). Each CorePacket is associated with a RAL `PacketBuffer`. The upper layer exchanges frame data with Core exclusively through CorePackets, and never touches a `PacketBuffer` directly.

The upper layer reads and writes the `data` and `data_len` fields of a CorePacket directly. Buffer coherence toward the RAL is handled by Core: when a CorePacket is committed for transmission, Core reports the full frame range through the [RAL Buffer Manager](../spec-ral-services/ral-buffer-manager-service.md). The upper layer never issues RAL buffer commands.
<!--NOTE: In the future we may restrict the upper layer to read/write through primitives only, to control when these operations are allowed. Not necessary for the standard, but good practice to avoid implementation issues.-->

A CorePacket is **committed** when a `CORE_Schedule_Tx_Command` associated to it is accepted. A committed CorePacket is immutable: its contents can no longer be written. The commit is released when the lifecycle of the associated TX command closes, after which the CorePacket may be modified or deleted. A CorePacket bound to an RX command whose lifecycle has not closed, or associated a TX command whose lifecycle has not closed, is **in use**.

The normative behavior is defined in sections [Commands](#commands) and [Events](#events).


### Related Documents

| Document | Description |
| -------- | ----------- |
| [Introduction](./introduction.md) | Introduction to the Core Services |
| [Core Overview](./core-overview.md) | Defines the [CorePacket](./core-overview.md#the-corepacket-object) abstraction and its parameters |
| [RAL Buffer Manager](../spec-ral-services/ral-buffer-manager-service.md) | Provides the `PacketBuffer` associated with each CorePacket |
| [RAL Overview](../spec-ral-services/ral-overview.md) | Defines [`PacketBuffer`](../spec-ral-services/ral-overview.md#packetbuffer) |


<!------------------------------------------------------------------------>


## Commands


### `CORE_Packet_Create_Command`

**Summary:** Creates a CorePacket and its associated `PacketBuffer`.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `max_data_len` | Unsigned Integer | Greater than 0, and at most `PHY_MAX_PSDU_LENGTH - L - M - F` (see [§PSDULength](../spec-ral-services/ral-overview.md#psdulength)), where `L` is the size of the largest Core header the frame format allows, and `M` is `coreMicSize` on a RAL that has a cipher and 0 otherwise | Maximum number of payload octets the CorePacket can hold. |

**Description:**

`CORE_Packet_Create_Command` requests Core to create a `CorePacket`. Core MUST allocate a `PacketBuffer` through `RAL_Buffer_Create_Command` and associate it with the CorePacket. Core MUST provide a `buffer_data` of at least `L + max_data_len + F + bufferSpaceMargin` octets, and selects the `buffer_type`. The created CorePacket starts with `data_len = 0`.

Core MUST emit exactly one `CORE_Packet_Create_CommandEndEvent` for each `CORE_Packet_Create_Command` processed.

If `max_data_len` is outside its valid range, Core MUST emit `CORE_Packet_Create_CommandEndEvent` with `command_status = ERROR_INVALID_VALUE`, closing the command lifecycle.

Core MUST emit `CORE_Packet_Create_CommandEndEvent` with `command_status = ERROR_BUFFER_ALLOC_FAILED`, closing the command lifecycle, if either of the following applies:
1. Core cannot allocate a memory region of at least `L + max_data_len + F + bufferSpaceMargin` octets for `buffer_data`.
2. `RAL_Buffer_Create_CommandEndEvent` reports a `command_status` other than `SUCCESS`.

If the command succeeds, Core MUST emit `CORE_Packet_Create_CommandEndEvent` with `command_status = SUCCESS` and the created CorePacket.


### `CORE_Packet_Delete_Command`

**Summary:** Deletes a CorePacket and releases its associated `PacketBuffer`.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `core_packet` | [`CorePacket`](./core-overview.md#the-corepacket-object) | — | The CorePacket to delete. |

**Description:**

`CORE_Packet_Delete_Command` requests Core to delete a CorePacket and release its associated `PacketBuffer` through the RAL Buffer Manager.

Core MUST emit exactly one `CORE_Packet_Delete_CommandEndEvent` for each `CORE_Packet_Delete_Command` processed.

If `core_packet` does not identify a valid CorePacket, Core MUST emit `CORE_Packet_Delete_CommandEndEvent` with `command_status = ERROR_PACKET_NOT_FOUND`, closing the command lifecycle.

If the CorePacket is in use, Core MUST emit `CORE_Packet_Delete_CommandEndEvent` with `command_status = ERROR_PACKET_IN_USE`, closing the command lifecycle.

If the command succeeds, Core MUST release the associated `PacketBuffer` and emit `CORE_Packet_Delete_CommandEndEvent` with `command_status = SUCCESS`. After deletion, the octets `CorePacket.data` are no longer guaranteed to be valid.


<!------------------------------------------------------------------------>


## Events


### `CORE_Packet_Create_CommandEndEvent`

**Soliciting command:** `CORE_Packet_Create_Command`

**Summary:** Reports the result of creating a CorePacket.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_INVALID_VALUE`, `ERROR_BUFFER_ALLOC_FAILED` | Outcome of the create request. |
| `core_packet` | [`CorePacket`](./core-overview.md#the-corepacket-object) | Present only when `command_status = SUCCESS`. | The created CorePacket. |

**Description:** Core emits `CORE_Packet_Create_CommandEndEvent` upon conclusion of `CORE_Packet_Create_Command`. This event closes the command lifecycle.


### `CORE_Packet_Delete_CommandEndEvent`

**Soliciting command:** `CORE_Packet_Delete_Command`

**Summary:** Reports the result of a CorePacket delete request.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ----------- | ----------- | ----------- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_PACKET_IN_USE`, `ERROR_PACKET_NOT_FOUND` | Outcome of the delete request. |

**Description:** Core emits `CORE_Packet_Delete_CommandEndEvent` upon conclusion of `CORE_Packet_Delete_Command`. This event closes the command lifecycle.
