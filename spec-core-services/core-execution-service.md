# Core Execution Service


<!------------------------------------------------------------------------>


## Introduction

The Execution Service schedules timeframes, single TX and RX actions, and skips. The layer above Core schedules a timeframe declaring a TU budget, then fills it with TX actions, RX actions, and skips to consume the budget fully, then schedules the next timeframe. Core sequences scheduled slots using the TUC, which advances by each slot duration. Core may hold several active commands at the same time in its local schedule representation, each identified by a command handle (see [Active TX/RX commands and handles](./core-overview.md#active-txrx-commands-and-handles)).
For each TX or RX action, Core translates the request into RAL operations and submits them through the RAL Execution Service.

The normative behavior is defined in sections [Commands](#commands) and [Events](#events).


### Related Documents

| Document | Description |
| -------- | ----------- |
| [Introduction](./introduction.md) | Introduction to the Core Services |
| [Core Overview](./core-overview.md) | Defines Core time and scheduling concepts |
| [Packet Manager Service](./core-packet-manager-service.md) | Provides the CorePackets bound to TX and RX actions |
| [Synchronization Service](./core-synchronization-service.md) | Establishes the TREF of timeframes |
| [RAL Execution Service](../spec-ral-services/ral-execution-service.md) | Executes the radio operations Core translates actions into |
| [RAL Security Service](../spec-ral-services/ral-security-service.md) | Provides the cipher that protects the frames of protected actions |


<!------------------------------------------------------------------------>


## Commands


### `CORE_Schedule_Timeframe_Command`

**Summary:** Schedules a timeframe of a given TU duration, starting at the current TREF.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `duration` | Unsigned Integer | Greater than 0 | Duration of the timeframe in TU. |

**Description:**

`CORE_Schedule_Timeframe_Command` requests Core to open a new timeframe. The timeframe starts at the current TREF.

Core MUST emit exactly one `CORE_Schedule_Timeframe_CommandEndEvent` for each `CORE_Schedule_Timeframe_Command` processed.

A new timeframe may be scheduled as soon as the previous timeframe is fully allocated with slots (the TUC of the last slot plus the slot duration has reached the timeframe duration), even if the radio is still executing earlier actions.
If a timeframe is already active and not fully allocated with slots, Core MUST emit `CORE_Schedule_Timeframe_CommandEndEvent` with `command_status = ERROR_TIMEFRAME_ACTIVE`, closing the command lifecycle.

If no TREF is established, Core MUST emit `CORE_Schedule_Timeframe_CommandEndEvent` with `command_status = ERROR_TREF_NOT_SET`, closing the command lifecycle.

If the schedule has reached `coreUpcomingScheduleCapacity`, Core MUST emit `CORE_Schedule_Timeframe_CommandEndEvent` with `command_status = ERROR_SCHEDULE_FULL`, closing the command lifecycle.

If the command succeeds, Core MUST eventually advance TREF by the duration of the previous timeframe (if any) and set the TUC to 0, without disruption to the current timeframe, and before the start of the scheduled timeframe. The new timeframe starts at the updated TREF. Core MUST emit `CORE_Schedule_Timeframe_CommandEndEvent` with `command_status = SUCCESS` and the updated TREF.


### `CORE_Schedule_Tx_Command`

**Summary:** Schedules a TX action of a given TU duration, provided a CorePacket that holds the data to transmit.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `duration` | Unsigned Integer | Greater than 0 | Duration of the transmit action in TU. |
| `core_packet` | [`CorePacket`](./core-overview.md#table-core-packet-parameters) | — | The object carrying the payload to be transmitted. |
| `dst_addr_mode` | Enumeration | `NONE`, `SHORT`, `EXTENDED` | Destination addressing mode for this frame. `NONE` indicates broadcast. |
| `dst_addr` | — | As implied by `dst_addr_mode` | Destination address. Omitted when `dst_addr_mode = NONE`. |
| `dst_pan_id` | Unsigned Integer | `0x0000..0xffff` | Destination PAN ID for this frame. `0xffff` is the broadcast PAN accepted by all PANs. |
| `src_addr_mode` | Enumeration | `NONE`, `SHORT`, `EXTENDED` | Source addressing mode for this frame. `NONE` indicates no source address. |
| `aux_sec_present` | Boolean | `FALSE` while `coreSecurityEnabled` is `FALSE` | Whether the frame carries the Aux Sec field. |
| `src_addr` | Unsigned Integer | A short address. Optional. | Source address to use in the Src Addr field where `src_addr_mode` is `SHORT`, regardless of whether security is enabled. On a protected frame, also supplies the short source address in the nonce, even when the header omits it. |
| `aux_sec` | Octet String | 4 octets. Optional; given only while `coreSecurityEnabled` is `TRUE`. | IV seed to use in the nonce and, where `aux_sec_present` is `TRUE`, in the Aux Sec field. Core does not check where the supplied seed comes from. |

**Description:**

`CORE_Schedule_Tx_Command` requests Core to schedule a TX action within the current timeframe, allocating a slot of size `duration`, starting at the first unallocated TU in the timeframe. Core assembles the Core header of the transmitted PSDU from `dst_pan_id`, `dst_addr_mode`/`dst_addr`, `src_addr_mode`, local configuration attributes `corePanId`/`coreShortAddress`/`coreExtendedAddress`, plus time fields `TUC = TUC_slot + coreTxOffset` and `TFC` timeframe counter. On a protected frame it adds the Aux Sec field where `aux_sec_present` asks for it. The optional `src_addr` and `aux_sec` override their respective values independently; a relay may pass the values of the frame it relays, and an unprotected relay may pass `src_addr` alone. Core does not require either value to come from a received frame. The payload comes from the provided `CorePacket`.
<!--NOTE: Use TUC for the TX/RX timestamp in logical time, do not use it for the schedule-->
<!--NOTE: Key selection per transmission is still open (see the key slot TODO in the RAL Security Service). Per-TX PHY parameters (data rate, channel, TX power) are persistent RAL configuration attributes at the moment.-->

Core MUST emit `CORE_CommandStatusEvent` with `command_id = CORE_Schedule_Tx_Command`.

If no timeframe is active, Core MUST reject the command in `CORE_CommandStatusEvent` with `command_status = REJECTED`, closing the command lifecycle.

If `duration` would overrun the remaining duration of the current timeframe, Core MUST reject the command in `CORE_CommandStatusEvent` with `command_status = REJECTED`, closing the command lifecycle.

If the schedule has reached `coreUpcomingScheduleCapacity`, Core MUST reject the command in `CORE_CommandStatusEvent` with `command_status = REJECTED`, closing the command lifecycle.

Core MUST also reject the command in `CORE_CommandStatusEvent` with `command_status = REJECTED`, closing the command lifecycle, if `src_addr` is given and `src_addr_mode` is `EXTENDED`, since the supplied short address does not identify an extended source address, or if `aux_sec` is given while `coreSecurityEnabled` is `FALSE`.

While `coreSecurityEnabled` is `TRUE`, if a draw of a new IV seed that this command needs fails, as described below, Core MUST reject the command in `CORE_CommandStatusEvent` with `command_status = REJECTED`, closing the command lifecycle. Core then discards its IV seed.

While `coreSecurityEnabled` is `TRUE`, Core MUST set `Security Enabled` in the header, and when Core accepts the command, it fixes the header and the nonce of the frame; a later change of the TFC does not change them. Core MUST build the nonce as described in [Nonce](./core-overview.md#nonce): from `src_addr` where supplied and otherwise `coreShortAddress`, from `aux_sec` where supplied and otherwise its own IV seed, and from the TFC of the timeframe and the TUC of the frame, `TUC_slot + coreTxOffset`. When `aux_sec` is absent, Core MUST first draw a new IV seed through `RAL_Crypto_Iv_Seed_Command` if it holds none, or if the frame's TFC and TUC, as the nonce holds them and compared TFC first, are not greater than those of the previous frame it used its seed for. A supplied `aux_sec` does not replace or advance Core's own IV seed state. The upper layer remains responsible for the nonce-uniqueness rules when it supplies either value. If Core holds no TFC because no `CORE_Tref_Sync_Command` has set one, it holds one from its first protected transmission.

If the command is accepted, Core MUST allocate the slot, create a handle, and report `command_status = ACCEPTED` with the handle in `CORE_CommandStatusEvent`. 

Core then translates the action into a RAL TX operation in radio time and submits it through the RAL Execution Service, through the following procedure:

1. At acceptance, Core computes the point in time of the PTA in radio time as `tx_at = TREF_radio + (TUC_slot + coreTxOffset) * coreTuSize`, where `TUC_slot` is the TUC of the allocated slot and `TREF_radio` is TREF expressed in radio time.
3. At acceptance, Core prepares the PSDU in the `PacketBuffer` associated with the `CorePacket`.
Core determines the Core header of the frame and its size `H`. The header carries the addressing fields from the command parameters and the `coreShortAddress` or `coreExtendedAddress` configuration attribute, the network identifier from the `corePanId` configuration attribute, and the TUC equal to `TUC_slot + coreTxOffset`, the position of the PTA. The Core header begins with control flags that indicate, for each optional field, whether it is present. Core omits every optional field that is absent for the transmission (for example, the destination address when `dst_addr_mode = NONE`, or the source address when `src_addr_mode = NONE`).
4. Core issues `RAL_Buffer_Configure_Regions_Command` with `psdu_length = H + data_len + F`. If `Security Enabled` is `1`, Core MUST place the Core header in the authenticated and not encrypted region and the payload in the authenticated and encrypted region, by setting `region_unsecured_size = 0` and `region_authenticated_size = H`. If `Security Enabled` is `0`, Core MUST declare an unsecured partition, by setting `region_unsecured_size = H + data_len` and `region_authenticated_size = 0`.
5. Core writes the Core header in the appropriate regions of `core_packet`, issuing `RAL_Buffer_Mark_Touched_Command`, with parameters `from = 0`, and `to = H + data_len`.
   __Note.__ The payload is already in place, since the upper layer wrote it before the `CorePacket` was committed. Declaring the partition does not move or discard it.
   <!--In the future the Core may be responsible for writing (or often, leaving as-is) the payload. This will require more careful wording here.-->
6. If the RAL is in `SLEEP` state, Core wakes it with `RAL_State_Wakeup_Command` so that the RAL is in `IDLE` state sufficiently early before the transmission start time.
7. Core issues `RAL_Tx_Schedule_Command` providing the `PacketBuffer` and `tx_at` parameters. Core MUST issue it only after the lifecycle of the RAL command of the previous slot has closed.
   If `Security Enabled` is `1`, Core supplies the nonce it fixed at acceptance as the `nonce` parameter. Core MAY commit it with the command (`nonce_implicit_commit = TRUE`) or later through `RAL_Nonce_Commit_Command`.
<!--TODO: Add reference to frame format in step 2.-->
<!--NOTE: v1.0 transmits with the data fully prepared at acceptance; the RAL commit strategy may change when late data producers are introduced.-->

If any command that Core issues within the execution of this command to the RAL Buffer Manager Service fails, or if the RAL rejects `RAL_Tx_Schedule_Command`, Core MUST emit `CORE_Schedule_Tx_CommandEndEvent` with `command_status = ERROR_TX_FAILED`, closing the command lifecycle.

Upon `RAL_Tx_Schedule_CommandEndEvent`, Core MUST emit `CORE_Schedule_Tx_CommandEndEvent`, closing the command lifecycle, with `command_status = SUCCESS` if the RAL reports `SUCCESS`, `command_status = CANCELLED` if the RAL reports `CANCELLED`, and `command_status = ERROR_TX_FAILED` otherwise.

If this command is cancelled while its `RAL_Tx_Schedule_Command` is the RAL [active command](../spec-ral-services/ral-overview.md#active-command), Core MUST issue `RAL_Cancel_Command`. If the cancellation occurs after Core has issued `RAL_Tx_Schedule_Command` but before the RAL has accepted or rejected it, Core MUST issue `RAL_Cancel_Command` as soon as the command is accepted. If Core has not yet issued `RAL_Tx_Schedule_Command`, Core MUST cease the execution of this command emit `CORE_Schedule_Tx_CommandEndEvent` with `command_status = CANCELLED`, closing the command lifecycle.


### `CORE_Schedule_Rx_Command`

**Summary:** Schedules an RX action of a given TU duration, bound to a CorePacket that is to receive the PSDU.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `duration` | Unsigned Integer | Greater than 0 | Duration of the receive action in TU. |
| `core_packet` | [`CorePacket`](./core-overview.md#table-core-packet-parameters) | — | The object into which the received payload is delivered. |
| `frame_control` | Unsigned Integer | `0x0000..0xffff` | The Frame Control field of the protected frame this action expects. Used only while `coreSecurityEnabled` is `TRUE`. |
| `src_addr` | Unsigned Integer | A short address. Optional. | The short address to use in the nonce when the expected frame carries no short source address. |
| `aux_sec` | Octet String | 4 octets. Optional. | The IV seed to use in the nonce when the expected frame carries no Aux Sec field. |

<!--CRITICAL: We cannot SCAN via Core right now. `scan` option was not added here because what happens when using `scan = TRUE` is not obvious w.r.t. timeframes and scheduled actios.
Suggestion: Introduce a CORE_Schedule_Scan_Command in place of the `scan` option. Whether this should be done in this version or the next is to be decided. 
CORE_Schedule_Scan_Command would take no duration and is only usable when no Timeframe is active (`immediate` option) or, if within a timeframe (and a start aligned to a TU) only if there are no other operations scheduled afterwards (and new ones cannot be scheduled either). 
-->

**Description:**

`CORE_Schedule_Rx_Command` requests Core to schedule an RX action within the current timeframe, allocating a slot of size `duration`, starting at the first unallocated TU in the timeframe.

Core MUST emit `CORE_CommandStatusEvent` with `command_id = CORE_Schedule_Rx_Command`.

If no timeframe is active, Core MUST reject the command in `CORE_CommandStatusEvent` with `command_status = REJECTED`, closing the command lifecycle.

If `duration` would overrun the remaining duration of the current timeframe, Core MUST reject the command in `CORE_CommandStatusEvent` with `command_status = REJECTED`, closing the command lifecycle.

If the schedule has reached `coreUpcomingScheduleCapacity`, Core MUST reject the command in `CORE_CommandStatusEvent` with `command_status = REJECTED`, closing the command lifecycle.

If the command is accepted, Core MUST allocate the slot, create a handle, and report `command_status = ACCEPTED` with the handle in `CORE_CommandStatusEvent`.

Core then translates the action into a RAL RX operation in radio time and submits it through the RAL Execution Service, through the following procedure:

1. Core computes the expected point in time of the PTA of an incoming frame in radio time as `TREF_radio + (TUC_slot + coreTxOffset) * coreTuSize`, where `TUC_slot` is the TUC of the allocated slot. It then computes `rx_min_at`, `rx_max_at`, starting from the expected point in time of the PTA and reducing/increasing it by the maximum expected anticipation/delay of the transmitter (due to, e.g., clock drift). <!--TODO: Mandate calculation? It is commonly done by setting the ppm tolerance and multiplying by the time since the last synchronization. -->
2. If the RAL is in `SLEEP` state, Core wakes it with `RAL_State_Wakeup_Command` so that the RAL is in `IDLE` state sufficiently early before the reception start time.
3. Before it issues `RAL_Rx_Schedule_Command`, Core declares the partition of the `PacketBuffer` via `RAL_Buffer_Configure_Regions_Command`. While `coreSecurityEnabled` is `FALSE`, the partition is unsecured, with `psdu_length` the largest PSDU the `PacketBuffer` accepts and `region_unsecured_size = psdu_length - F`. Otherwise, the header that `frame_control` describes, of size `H`, is the authenticated and not encrypted region and the payload is the authenticated and encrypted region: `region_unsecured_size = 0`, `region_authenticated_size = H` and `psdu_length = H + max_data_len + F`, where `max_data_len` is that of the `CorePacket`. If the declaration fails, Core MUST NOT issue `RAL_Rx_Schedule_Command`, and MUST emit `CORE_Schedule_Rx_CommandEndEvent` with `command_status = ERROR_RX_FAILED`, closing the command lifecycle. Core issues `RAL_Rx_Schedule_Command`. Its `packet` parameter SHOULD be set to the `PacketBuffer` associated with `core_packet`. `rx_min_at`, `rx_max_at` are provided as computed in previous steps. The `frame_timeout` parameter is set equal to the maximum on-air duration of the part of the PPDU that follows the PTA after `rx_max_at`, for the largest complete on-air PSDU the `PacketBuffer` accepts, including the MIC when protected. Core MUST issue it only after the lifecycle of the RAL command of the previous slot has closed.
4. While `coreSecurityEnabled` is `TRUE`, Core supplies the nonce described in [Nonce](./core-overview.md#nonce) as the `nonce` parameter. Core MUST build it from the TFC of the timeframe and the TUC of the frame, `TUC_slot + coreTxOffset`, if it holds a TFC, and otherwise from those the frame carries; and from the short source address and IV seed the frame carries, and otherwise from `src_addr` and `aux_sec`. If it knows every value of the nonce before the frame arrives, it MAY commit it with the command (`nonce_implicit_commit = TRUE`). Otherwise it adds the last octet of the header, `H - 1`, to `psdu_trigger_indexes`, and at the `RAL_Rx_Data_Available_CommandEvent` for that octet it reads the header with `RAL_Buffer_Ensure_Available_Command`, builds the nonce and commits it through `RAL_Nonce_Commit_Command`.
<!--CRITICAL: Neither Core nor the RAL currently exposes the on-air duration of a PSDU of a given length, which Core needs to compute `frame_timeout`. The same quantity is important to quatify the minimum `duration` of an RX slot (`coreTxOffset`, plus the maximum expected delay of the transmitter, plus `frame_timeout`).-->

If the RAL rejects `RAL_Rx_Schedule_Command`, Core MUST emit `CORE_Schedule_Rx_CommandEndEvent` with `command_status = ERROR_RX_FAILED`, closing the command lifecycle.

Upon `RAL_Rx_Schedule_CommandEndEvent`, Core MUST emit `CORE_Schedule_Rx_CommandEndEvent`, closing the command lifecycle, with:
- `command_status = CANCELLED` if the RAL reports `CANCELLED` for a cancellation of this command,
- `command_status = ERROR_DETECTION_TIMEOUT` if the RAL reports `ERROR_DETECTION_TIMEOUT` or `ERROR_PTA_TIMEOUT`,
- `command_status = ERROR_RX_FAILED` if the RAL reports `ERROR_FRAME_TIMEOUT`, `ERROR_FRAME_REJECTED`, `ERROR_FRAME_CORRUPTED`, `ERROR_NONCE_REQUIRED`, `ERROR_NOT_AUTHENTICATED`, or `ERROR_DECRYPTION_FAILED`, or if it reports `CANCELLED` for a reception Core cancelled because its header failed the checks below,
- the outcome of the procedure below if the RAL reports `SUCCESS`.

If the RAL reports `SUCCESS`, Core issues `RAL_Buffer_Ensure_Available_Command` with parameters `from = 0` and `to = psdu_length - F`. If that command succeeds, Core parses the Core header and determines its size `H`. If the command fails, the header is malformed, `psdu_length < H + F`, or the header check below refuses the frame, Core MUST emit `CORE_Schedule_Rx_CommandEndEvent` with `command_status = ERROR_RX_FAILED`, closing the command lifecycle. Otherwise, Core sets `CorePacket.data` to the octet that follows the Core header and `CorePacket.data_len` to `psdu_length - H - F`, then MUST emit `CORE_Schedule_Rx_CommandEndEvent` with `command_status = SUCCESS`, closing the command lifecycle. The RX timestamp (`t_rx`) is captured internally in the state of the completed RX command, where it is available to `CORE_Tref_Sync_Command` via the command handle.
<!--TODO: Revise when implementing Core-level FastSTX operations-->

While `coreSecurityEnabled` is `TRUE`, Core checks the header of every received frame, at the `RAL_Rx_Data_Available_CommandEvent` of step 4 if it requested one, and otherwise after the RAL reports `SUCCESS`. Core MUST refuse the frame if:
- its Frame Control field differs from `frame_control`,
- Core holds a TFC and the frame carries a TFC or TUC that differs from its own,
- a value the nonce needs is neither in the frame nor available to Core, or
- Core cannot read the header or commit the nonce.

At the data event, Core refuses the frame by issuing `RAL_Cancel_Command`. While `coreSecurityEnabled` is `FALSE`, Core MUST refuse a frame whose `Security Enabled` bit is set. A refused frame ends with `command_status = ERROR_RX_FAILED` and yields no `CorePacket`.

If this command is cancelled while its `RAL_Rx_Schedule_Command` is the RAL [active command](../spec-ral-services/ral-overview.md#active-command), Core MUST issue `RAL_Cancel_Command`. If the cancellation occurs after Core has issued `RAL_Rx_Schedule_Command` but before the RAL has accepted or rejected it, Core MUST issue `RAL_Cancel_Command` as soon as the command is accepted. If Core has not yet issued `RAL_Rx_Schedule_Command`, Core MUST cease the execution of this command and emit `CORE_Schedule_Rx_CommandEndEvent` with `command_status = CANCELLED`, closing the command lifecycle.


### `CORE_Schedule_Skip_Command`

**Summary:** Schedules a skip of a given TU duration. A skip is a slot that reserves an idle interval for its duration, and may authorize Core to put the radio to sleep.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `duration` | Unsigned Integer | Greater than 0 | Duration of the skip in TU. |
| `allow_sleep` | Boolean | — | If `TRUE`, Core is allowed to change the radio state to `SLEEP` during the skip slot. |

**Description:**

`CORE_Schedule_Skip_Command` requests Core to schedule a skip within the current timeframe, allocating a slot of size `duration`, starting at the first unallocated TU in the timeframe.

Core MUST emit exactly one `CORE_Schedule_Skip_CommandEndEvent` for each `CORE_Schedule_Skip_Command` processed.

If no timeframe is active, Core MUST emit `CORE_Schedule_Skip_CommandEndEvent` with `command_status = ERROR_NO_TIMEFRAME`, closing the command lifecycle.

If `duration` would overrun the remaining duration of the current timeframe, Core MUST emit `CORE_Schedule_Skip_CommandEndEvent` with `command_status = ERROR_OVERRUN`, closing the command lifecycle.

If the schedule has reached `coreUpcomingScheduleCapacity`, Core MUST emit `CORE_Schedule_Skip_CommandEndEvent` with `command_status = ERROR_SCHEDULE_FULL`, closing the command lifecycle.

If the command succeeds, Core MUST allocate the slot and emit `CORE_Schedule_Skip_CommandEndEvent` with `command_status = SUCCESS`. A skip allocates no handle. When `allow_sleep = TRUE`, Core MAY put the radio to sleep for the duration of the skip. If it does, Core MUST wake the radio in time for the next scheduled action.


### `CORE_Cancel_Command`

**Summary:** Cancels the active TX or RX command identified by its handle and aborts its underlying RAL operation.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_handle` | Unsigned Integer | A valid command handle | Identifies the active command to cancel. |

**Description:**

`CORE_Cancel_Command` requests Core to cancel an active TX or RX command and abort its underlying RAL operation.

Core MUST emit exactly one `CORE_Cancel_CommandEndEvent` for each `CORE_Cancel_Command` processed.

If `command_handle` does not identify an active TX or RX command, Core MUST emit `CORE_Cancel_CommandEndEvent` with `command_status = ERROR_ACTIVE_COMMAND_NOT_FOUND`, closing the command lifecycle.

Otherwise, Core MUST cancel the identified command and emit `CORE_Cancel_CommandEndEvent` with `command_status = SUCCESS`, closing the command lifecycle.

__Note.__ If the command succeeds, Core may issue `RAL_Cancel_Command` for the underlying active command
following the procedures in `CORE_Schedule_Tx_Command` and `CORE_Schedule_Rx_Command`.


<!------------------------------------------------------------------------>


## Events


### `CORE_CommandStatusEvent`

**Soliciting command:** `CORE_Schedule_Tx_Command`, `CORE_Schedule_Rx_Command`

**Summary:** Reports whether the addressed scheduling command was accepted for processing.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_id` | Enumeration | `CORE_Schedule_Tx_Command`, `CORE_Schedule_Rx_Command` | Identifies the command invocation being reported. |
| `command_status` | Enumeration | `ACCEPTED`, `REJECTED` | Acceptance or rejection status. `ACCEPTED` is the only acceptance value. |
| `command_handle` | Unsigned Integer | Present only when `command_status = ACCEPTED`. | Handle allocated for the accepted scheduling command. |

**Description:** Core emits `CORE_CommandStatusEvent` after intake validation for the addressed scheduling command. `command_id` identifies the command. If `command_status = ACCEPTED`, the command-specific event sequence continues, correlated by `command_handle`. Any other `command_status` value rejects the command invocation, closing its lifecycle without a terminal event.


### `CORE_Schedule_Timeframe_CommandEndEvent`

**Soliciting command:** `CORE_Schedule_Timeframe_Command`

**Summary:** Reports the final outcome of a timeframe scheduling request.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_TIMEFRAME_ACTIVE`, `ERROR_TREF_NOT_SET`, `ERROR_SCHEDULE_FULL` | Outcome of the timeframe scheduling request. |
| `tref` | `SystemInstant` | Present only when `command_status = SUCCESS`. | The TREF the timeframe starts at, expressed as a system time value. |

**Description:** Core emits `CORE_Schedule_Timeframe_CommandEndEvent` after processing a timeframe scheduling request. This event closes the command lifecycle.


### `CORE_Schedule_Tx_CommandEndEvent`

**Soliciting command:** `CORE_Schedule_Tx_Command`

**Summary:** Reports the final outcome of the scheduled transmit operation.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_handle` | Unsigned Integer | — | Handle of the transmit command. |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_TX_FAILED`, `CANCELLED` | Final execution status. |

**Description:** Core emits `CORE_Schedule_Tx_CommandEndEvent` upon conclusion of the accepted transmit operation. This event is emitted only after `CORE_CommandStatusEvent` reported `command_status = ACCEPTED` for the same command invocation. It closes the command lifecycle.


### `CORE_Schedule_Rx_CommandEndEvent`

**Soliciting command:** `CORE_Schedule_Rx_Command`

**Summary:** Reports the final outcome of the scheduled receive operation.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_handle` | Unsigned Integer | — | Handle of the receive command. |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_DETECTION_TIMEOUT`, `ERROR_RX_FAILED`, `CANCELLED` | Final execution status. A frame received with a failed authentication check is reported as `ERROR_RX_FAILED`. |
| `core_packet` | `CorePacket` | Present only when `command_status = SUCCESS`. | The CorePacket carrying the received payload. |
| `src_addr_mode` | Enumeration | `NONE`, `SHORT`, `EXTENDED` | Source addressing mode the received Core header held; `NONE` where it held no source address. Valid only if `command_status = SUCCESS`. |
| `src_addr` | Unsigned Integer | As implied by `src_addr_mode` | Source address the received Core header held. Valid only if `command_status = SUCCESS` and `src_addr_mode` is not `NONE`. |
| `aux_sec` | Octet String | 4 octets. Present only when `command_status = SUCCESS` and the header carried `Aux Sec`. | `Aux Sec` the received Core header held. |

**Description:** Core emits `CORE_Schedule_Rx_CommandEndEvent` upon conclusion of the accepted receive operation. This event is emitted only after `CORE_CommandStatusEvent` reported `command_status = ACCEPTED` for the same command invocation. It closes the command lifecycle. The reception radio time is captured internally in the state of the completed RX command and is available to `CORE_Tref_Sync_Command`. Core never exposes radio time in this event.


### `CORE_Schedule_Skip_CommandEndEvent`

**Soliciting command:** `CORE_Schedule_Skip_Command`

**Summary:** Reports the final outcome of a skip scheduling request.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_NO_TIMEFRAME`, `ERROR_OVERRUN`, `ERROR_SCHEDULE_FULL` | Outcome of the skip scheduling request. |

**Description:** Core emits `CORE_Schedule_Skip_CommandEndEvent` after processing a skip scheduling request. A skip is a slot without TX/RX operations, so this event is emitted at intake. This event closes the command lifecycle.


### `CORE_Cancel_CommandEndEvent`

**Soliciting command:** `CORE_Cancel_Command`

**Summary:** Reports the outcome of a cancellation request.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_ACTIVE_COMMAND_NOT_FOUND` | Outcome of the cancellation request. |

**Description:** Core emits `CORE_Cancel_CommandEndEvent` after processing a cancellation request. This event closes the command lifecycle.


### `CORE_Wakeup_Event`

**Soliciting command:** *(none, unsolicited)*

**Summary:** Indicates that the radio woke from sleep for the next scheduled operation.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_handle` | Unsigned Integer | — | Handle of the next TX or RX command the radio woke for. |

**Description:** Core emits `CORE_Wakeup_Event` when it wakes the radio from sleep to execute a scheduled operation. The event is emitted only when the radio actually transitioned to sleep during a skip and is now waking, not merely when sleep was authorized. It is not emitted for the initial radio bring-up. This event is unsolicited. It lets the layer above Core track radio duty cycling.
