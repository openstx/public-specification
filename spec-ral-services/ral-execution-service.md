# RAL Execution Service


<!------------------------------------------------------------------------>


## Introduction

The RAL Execution Service is responsible for scheduling radio operations within real-time constraints.

The service has three notable characteristics:

1. **Asynchronous and sequential operation scheduling.** The RAL acknowledges the scheduling of commands, only accepting one at a time, independently of operation completion. The final result is delivered afterwards via a dedicated event.
2. **Decoupled data transfer for transmissions.** Data for transmission may be committed after an operation has already been scheduled. This allows the higher layer to continue data preparation while the operation is pending, reducing the total sequential time budget for the operation.
3. **Early receive indication.** Upon scheduling a reception, the higher layer may request an event as soon as selected PSDU octets are available, enabling earlier reaction when specific information is accessible.

The normative behavior is defined in sections [Commands](#commands) and [Events](#events).


### Related Documents

| Document                                  | Description                                                                                                     |
| --------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| [Introduction](./introduction.md)         | Introduction to the RAL Services                                                                                |
| [Buffer Manager](./ral-buffer-manager-service.md)                 | Outlines the `PacketBuffer` abstraction: its structure, lifecycle, specializations, and behavior aspects. The executor uses `PacketBuffers` to carry the PSDUs it transmits and receives.  |
| [RAL Security Service](./ral-security-service.md) | Specifies the cipher that protects the PSDU of a protected operation, and the key it uses. |


<!------------------------------------------------------------------------>


## Commands


### `RAL_Rx_Schedule_Command`

**Summary:** Schedules the reception of a single PPDU expected on air within a specified time range.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `packet` | `PacketBuffer`, as defined in [§Packet Buffer](./ral-overview.md#packetbuffer) | — | Indicates a memory space into which the received PSDU is to be delivered. |
| `rx_min_at` | `RalInstant` | As defined in [§RalInstant](ral-overview.md#ralinstant) | Earliest point in time, expressed in radio time, at which the PTA is expected on air. |
| `rx_max_at` | `RalInstant` | As defined in [§RalInstant](ral-overview.md#ralinstant) | Latest point in time, expressed in radio time, at which the PTA is expected on air. |
| `frame_timeout` | `RalInstant` | As defined in [§RalInstant](ral-overview.md#ralinstant) | Maximum allowed time interval within which the RAL is expected to complete PPDU reception, measured from the expected PTA reception time. If `frame_timeout` is 0, it MUST be ignored. |
| `scan` | Boolean | — | If `TRUE`, indicates that the radio MUST listen indefinitely until a successful or failed reception, or until the command is cancelled. |
| `psdu_trigger_indexes` | Set of Integers | Unique non-negative octet indexes, lower than `PHY_MAX_PSDU_LENGTH - F`. | Set of octet positions in the frame content upon delivery of which the RAL MUST emit `RAL_Rx_Data_Available_CommandEvent`. |
| `nonce` | Octet String. | Exactly [§nonceSize](./ral-config-service.md#generic-configuration-attributes) octets. | Indicates a memory space containing the nonce value used to authenticate and decrypt the `packet`'s secured regions. |
| `nonce_implicit_commit` | Boolean | — | If `TRUE`, the nonce value is implicitly committed immediately. Otherwise, the nonce MUST be committed explicitly via `RAL_Nonce_Commit_Command`. |

<!--CHECK: `frame_timeout` is a duration, not an instant — `RalInstant` is a placeholder type. -->

**Description:**

`RAL_Rx_Schedule_Command` schedules an RX operation for the reception of a single PPDU. The command specifies a time range within which the PPDU's PTA is expected on air. If no PPDU is detected or received in time, the operation completes with a timeout status.

The command MUST be rejected if any of the following applies:
1. The RAL state is `PENDING` or `RUNNING`.
2. The moment in time expressed by `rx_min_at` is in the past, or too close in the future for the radio to start listening at the requested time, accounting for its PHY-characteristic offset.
3. No partition has been declared for the `packet` through [§RAL_Buffer_Configure_Regions_Command](./ral-buffer-manager-service.md#ral_buffer_configure_regions_command).
4. The `packet`'s `psdu_length` plus the current MIC length exceeds `PHY_MAX_PSDU_LENGTH`, counting `micSize` for a protected partition and zero otherwise.

<!-- REV: Perhaps we should include rejection reasons in the status, e.g.,
`REJECTED_UNEXPECTED_STATE`, `REJECTED_PTA_IN_PAST`, `REJECTED_BUFFER_REGIONS_NOT_CONFIGURED` 
 -->
If the command is rejected, the RAL MUST emit `RAL_CommandStatusEvent` with `command_id = RAL_Rx_Schedule_Command` and `command_status = REJECTED`, closing the command lifecycle.

 If the command is accepted, the RAL MUST:
 - transition to `PENDING` state (see [§State Transitions](ral-overview.md#state-transitions)), 
 - set `RAL_Rx_Schedule_Command` as the [active command](ral-overview.md#active-command),
 - emit `RAL_CommandStatusEvent` with `command_id = RAL_Rx_Schedule_Command`, `command_status = ACCEPTED`, and 
 - make the radio enter listening state in time to correctly receive the PPDU according to the point in time expressed by `rx_min_at`.

 The RAL MUST make the radio enter listening state in advance w.r.t. the point in time expressed by `rx_min_at`, with a PHY-characteristic offset (to account for, e.g., ramp-up time and synchronization header duration). This offset SHOULD NOT be longer than the minimum time required to ensure the radio is ready to correctly receive the PPDU.
 
 When the radio begins to ramp-up for listening, the RAL MUST transition to `RUNNING` state (see [§State Transitions](ral-overview.md#state-transitions)).

If `scan = FALSE` and no incoming frame is detected within a given detection interval, the RAL MUST emit `RAL_Rx_Schedule_CommandEndEvent` with `command_status = ERROR_DETECTION_TIMEOUT`, closing the command lifecycle. The detection interval combines a PHY-characteristic interval appropriate for reliable frame detection, increased by the time difference expressed by `rx_max_at - rx_min_at`.

After successful frame detection, if PTA is not detected by the time expressed by `rx_max_at`, the RAL MUST emit `RAL_Rx_Schedule_CommandEndEvent` with `command_status = ERROR_PTA_TIMEOUT`, closing the command lifecycle. 
<!--CHECK: Not obvious if radios can detect specifically the PTA of the packet; this may need to be softened-->
<!--TODO: Consider whether using `rx_at` and `rx_guard`, replacing min-max, could be more intuitive and more actionable-->

After successful frame detection, if the entire PPDU is not received within the `frame_timeout` interval, measured from the moment in time represented by `rx_max_at`, the RAL MUST emit `RAL_Rx_Schedule_CommandEndEvent` with `command_status = ERROR_FRAME_TIMEOUT`, closing the command lifecycle.

If the radio detects the incoming frame does not comply with the configured rules described in !REF, the RAL SHOULD emit `RAL_Rx_Schedule_CommandEndEvent` with `command_status = ERROR_FRAME_REJECTED`, closing the command lifecycle.
<!--CHECK: We SHOULD revise this to decide if it's a MUST feature of the RAL, or a nice to have-->
<!--TODO: Add the reference where the Config is presented; it would probably be a Structure, one field would be the OpenSTX identifier-->

<!--The RAL MUST verify the CRC of each received PPDU. If the CRC is invalid, the RAL MUST emit `RAL_Rx_Schedule_CommandEndEvent` with `command_status = ERROR_INVALID_CRC`.-->
<!--CHECK: The above would go into CORE, keeping the RAL free from dealing with the frame format.-->
If the RAL identifies uncorrectable data errors in the received PSDU, the RAL SHOULD emit `RAL_Rx_Schedule_CommandEndEvent` with `command_status = ERROR_FRAME_CORRUPTED`, closing the command lifecycle.

If `BUFFER_REGION_AUTHENTICATED` or `BUFFER_REGION_ENCRYPTED` of the `packet` has a non-zero size (as declared via [§RAL_Buffer_Configure_Regions_Command](./ral-buffer-manager-service.md#ral_buffer_configure_regions_command)): 
1. If no key is installed (see [§RAL Security Service](./ral-security-service.md#the-cipher)), the command MUST be rejected at intake, as for the conditions listed above.
2. If `nonce` is undefined, the RAL MUST emit `RAL_Rx_Schedule_CommandEndEvent` with `command_status = ERROR_NONCE_REQUIRED`.
3. The RAL MUST use the `nonce` to check and decrypt the protected octets with the cipher of the [§RAL Security Service](./ral-security-service.md#the-cipher), before it emits `RAL_Rx_Schedule_CommandEndEvent`. 
4. The RAL MUST NOT use a `nonce` value that is not yet committed (implicitly or via `RAL_Nonce_Commit_Command`). The RAL MUST await the commit if necessary, keeping the command's lifecycle open. The caller SHOULD commit the `nonce` as soon as possible.
5. Until the `nonce` is committed, the memory referenced by nonce is owned by the caller and the RAL MUST NOT read it. From the commit until the emission of the `CommandEndEvent` of the scheduling command, the caller MUST NOT modify it.
6. If the authentication check fails, the RAL MUST emit `RAL_Rx_Schedule_CommandEndEvent` with `command_status = ERROR_NOT_AUTHENTICATED`, closing the command lifecycle.
7. If the cipher fails, the RAL MUST emit `RAL_Rx_Schedule_CommandEndEvent` with `command_status = ERROR_DECRYPTION_FAILED`, closing the command lifecycle.
8. If the complete received on-air PSDU has fewer octets than `BUFFER_REGION_UNSECURED`, `BUFFER_REGION_AUTHENTICATED`, the MIC and the FCS take together, the RAL MUST emit `RAL_Rx_Schedule_CommandEndEvent` with `command_status = ERROR_FRAME_REJECTED`, closing the command lifecycle, without waiting for the `nonce` to be committed.
9. If the reception does not end with `command_status = SUCCESS`, the RAL MUST NOT leave in `buffer_data` any octet it decrypted, and it MUST NOT let a received octet it holds elsewhere, such as in a radio buffer, reach the caller or the air afterwards: such octets MUST read, and be transmitted, as zero until the caller writes them. The RAL MAY defer erasing that memory.

If the RAL would emit `RAL_Rx_Schedule_CommandEndEvent` with a `command_status` other than `SUCCESS`, it MUST abort the reception operation before emitting the event.

As described in [§Ownership of Buffer Data](./ral-buffer-manager-service.md#ownership-of-buffer-data), when `RAL_Rx_Schedule_Command` is accepted, the `packet`'s data transitions to `BUFFER_OWNERSHIP_STATE_RAL`. While the data is in that state the caller MUST NOT write them, and any value it reads from them is not stable. The data is transitioned back to `BUFFER_OWNERSHIP_STATE_CALLER` when the execution of the accepted `RAL_Rx_Schedule_Command` ends with the emission `RAL_Rx_Schedule_CommandEndEvent`, regardless of the `command_status`. If the `command_status` is `SUCCESS`, the RAL MUST ensure that the three declared regions of the `packet`, are ready to be read by the caller in their plain, contiguous representation through the [§RAL_Buffer_Ensure_Available_Command](./ral-buffer-manager-service.md#ral_buffer_ensure_available_command). The frame content is undefined if `command_status` is other than `SUCCESS`.

Let `W` be the received PSDU length, including the MIC. If `W < M + F`, or if `W - M` is longer than the declared `psdu_length`, the RAL MUST emit `RAL_Rx_Schedule_CommandEndEvent` with `command_status = ERROR_FRAME_REJECTED`, closing the command lifecycle. Otherwise, on `SUCCESS`, the RAL MUST set the `packet`'s `psdu_length` to `W - M`. The last non-empty region then ends at `psdu_length - F` (see [§PSDULength](./ral-overview.md#psdulength)).

_Note._ If RAL-level encryption is in use, the requirement of providing a "plain representation" of the PSDU implies that encrypted regions of the PSDU have to be decrypted. If a PHY-specific ordering of PSDU octets within the PPDU was applied, the "contiguous representation" implies that the RAL has to reassemble the original, intended PSDU into a contiguous region. 

While the PSDU is being received, the RAL MUST emit a `RAL_Rx_Data_Available_CommandEvent` for each octet position listed in `psdu_trigger_indexes` as early as possible. With the emission of this event, the PSDU range `[0, psdu_index]` becomes available to the caller, and the RAL MUST ensure that this range is ready to be read in its plain, contiguous representation through the [§RAL_Buffer_Ensure_Available_Command](./ral-buffer-manager-service.md#ral_buffer_ensure_available_command). The range does not become owned by the caller, which therefore MUST NOT write it. 

If an immediate emission of the `RAL_Rx_Data_Available_CommandEvent` is not possible due to hardware limitations, the RAL MAY defer these events, but MUST NOT emit any of them later than `RAL_Rx_Schedule_CommandEndEvent`. If one or more events were, or would be, deferred until the emission time of an event with a higher index, the RAL MAY merge those events and only emit the higher-index event.

The RAL provides no integrity or authenticity guarantee for octets returned to the caller before `RAL_Rx_Schedule_CommandEndEvent` has reported `command_status = SUCCESS`. Reading them is permitted, and the caller is responsible for handling the risk of acting on data that may prove corrupted or not authentic.

If reception completes successfully, the RAL MUST emit `RAL_Rx_Schedule_CommandEndEvent` with `command_status = SUCCESS`, closing the command lifecycle.

If this active command is cancelled, the RAL MUST abort the scheduled or ongoing reception operation, and MUST emit `RAL_Rx_Schedule_CommandEndEvent` with `command_status = CANCELLED`, closing the command lifecycle.

When the lifecycle of this command is closed, the RAL MUST transition to `IDLE` state.

### `RAL_Tx_Schedule_Command`

**Summary:** Schedules the transmission of a single PPDU at a specified point in time.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `packet` | `PacketBuffer`, as defined in [§Packet Buffer](./ral-overview.md#packetbuffer) | — | Indicates a memory space containing the PSDU to be transmitted. |
| `tx_at` | `RalInstant` | As defined in [§RalInstant](ral-overview.md#ralinstant) | Point in time at which the PTA of the trasmitted PPDU MUST appear on air. |
| `packet_implicit_commit` | Boolean | — | If `TRUE`, the packet data is implicitly committed immediately. Otherwise, data MUST be committed explicitly via `RAL_Tx_Data_Commit_Command`. |
| `nonce` | Octet string of size `nonceSize` | — | Indicates a memory space containing the nonce value used to authenticate and encrypt the `packet`'s secured regions. |
| `nonce_implicit_commit` | Boolean | — | If `TRUE`, the nonce value is implicitly committed immediately. Otherwise, the nonce MUST be committed explicitly via `RAL_Nonce_Commit_Command`. |

**Description:**

`RAL_Tx_Schedule_Command` schedules the transmission of a single PPDU at the specified point in time. 

The command MUST be rejected if any of the following applies:
1. The RAL state is `PENDING` or `RUNNING`.
2. The moment in time expressed by `tx_at` is in the past, or too close in the future for the radio to start transmitting at the requested time, accounting for its PHY-characteristic offset.
3. No partition has been declared for the `packet` through [§RAL_Buffer_Configure_Regions_Command](./ral-buffer-manager-service.md#ral_buffer_configure_regions_command).
4. The `packet`'s `psdu_length` plus the current MIC length exceeds `PHY_MAX_PSDU_LENGTH`, counting `micSize` for a protected partition and zero otherwise.

<!-- REV: Perhaps we should include rejection reasons in the status, e.g.,
`REJECTED_UNEXPECTED_STATE`, `REJECTED_PTA_IN_PAST`, `REJECTED_BUFFER_REGIONS_NOT_CONFIGURED` 
 -->
If the command is rejected, the RAL MUST emit `RAL_CommandStatusEvent` with `command_id = RAL_Tx_Schedule_Command`, `command_status = REJECTED`, closing the command lifecycle.

If the command is accepted, the RAL MUST:
- transition to `PENDING` state (see [§State Transitions](ral-overview.md#state-transitions)),
- set `RAL_Tx_Schedule_Command` as the [active command](ral-overview.md#active-command),
- emit `RAL_CommandStatusEvent` with `command_id = RAL_Tx_Schedule_Command`, `command_status = ACCEPTED`, and
- transmit the PPDU, starting transmission in time to correctly transmit the PTA at the point in time expressed by `tx_at`.

The RAL MUST ready the radio in advance w.r.t. the point in time expressed by `tx_at`, with a PHY-characteristic offset (to account for, e.g., ramp-up time and synchronization header duration). This offset SHOULD NOT be longer than the minimum time required to ensure the radio is ready to correctly transmit the PPDU.

When the radio begins to ramp-up for the transmission, the RAL MUST transition to `RUNNING` state (see [§State Transitions](ral-overview.md#state-transitions)).

When the transmission completes successfully, the RAL MUST emit `RAL_Tx_Schedule_CommandEndEvent` with `command_status = SUCCESS`, closing the command lifecycle.

<!--All PPDUs transmitted by the RAL MUST include a CRC value for error detection.--> 
<!--TODO: Would be managed by CORE-->

<!-- 
REV: This is a precondition. It might be good to say that explicitly (or even use dedicated structural elements to highlight preconditions and invariants?) 
-->
The `packet`'s `psdu_length` MUST be set to the final PSDU length before `RAL_Tx_Schedule_Command` is issued, and MUST NOT change while the command is `PENDING` or `RUNNING`. Otherwise, the behaviour of the command is unspecified.

<!-- 
TODO: Beside the below paragraph, it might be useful to express the octet deadline requirement quantitatively with a specific constant (e.g. to make it possible for the Core to decide about minimum slot length constraints when using the concrete RAL). The constant could be placed at a common location, e.g. in the Config Service or in a new dedicated doc.
 -->
The RAL MUST NOT transmit uncommitted frame-content octets. Each such octet has a commit deadline, which is the latest time at which the radio can still place it on air at its scheduled time, w.r.t. the PTA. If any octet is not committed by its commit deadline using the [§RAL_Tx_Data_Commit_Command](#ral_tx_data_commit_command), the RAL MUST abort the scheduled or ongoing transmission immediately and emit `RAL_Tx_Schedule_CommandEndEvent` with `command_status = ERROR_TX_DATA_LATE`, closing the command lifecycle. To maximize the time budget available for data preparation, the RAL SHOULD defer the evaluation of whether an octet is committed before its deadline as far as the underlying hardware platform allows.

_Note._ Restricted radios may require all octets to be committed before transmission begins. However, the RAL also supports more capable radios, which accept octets on the fly while preceding octets are still being transmitted.

Prior to issuing the `RAL_Tx_Schedule_Command`, the caller declares the partition of the `packet` through [§RAL_Buffer_Configure_Regions_Command](./ral-buffer-manager-service.md#ral_buffer_configure_regions_command). The RAL MUST ensure that committed data is authenticated and/or encrypted according to that partition before it is transmitted.

If `BUFFER_REGION_AUTHENTICATED` or `BUFFER_REGION_ENCRYPTED` of the `packet` has a non-zero size (as declared via [§RAL_Buffer_Configure_Regions_Command](./ral-buffer-manager-service.md#ral_buffer_configure_regions_command)): 
1. If no key is installed (see [§RAL Security Service](./ral-security-service.md#the-cipher)), the command MUST be rejected at intake, as for the conditions listed above.
2. If `nonce` is undefined, the RAL MUST emit `RAL_Tx_Schedule_CommandEndEvent` with `command_status = ERROR_NONCE_REQUIRED`.
3. The RAL MUST use the `nonce` to encrypt and authenticate the protected octets with the cipher of the [§RAL Security Service](./ral-security-service.md#the-cipher). 
4. The RAL MUST NOT use a `nonce` value that is not yet committed (implicitly or via `RAL_Nonce_Commit_Command`). 
<!-- TODO: Analog to tx_data_commit, we need to quantify the commit-deadline. -->
5. The `nonce` commit deadline is the latest point in time at which the RAL can still start the cryptographic processing needed to place the first secured octet on air w.r.t the PTA. The RAL SHOULD defer the deadline evaluation as far as the hardware allows. If the `nonce` is not committed by its commit deadline, the RAL MUST abort the scheduled or ongoing transmission immediately and emit `RAL_Tx_Schedule_CommandEndEvent` with `command_status = ERROR_NONCE_LATE`, closing the command lifecycle.
6. Until the `nonce` is committed, the memory referenced by nonce is owned by the caller and the RAL MUST NOT read it. From the commit until the emission of the `CommandEndEvent` of the scheduling command, the caller MUST NOT modify it.
7. If the cipher fails, the RAL MUST NOT transmit any octet of the PSDU and MUST emit `RAL_Tx_Schedule_CommandEndEvent` with `command_status = ERROR_ENCRYPTION_FAILED`, closing the command lifecycle.

As described in [§Ownership of Buffer Data](./ral-buffer-manager-service.md#ownership-of-buffer-data), an octet range committed implicitly or through [§RAL_Tx_Data_Commit_Command](#ral_tx_data_commit_command) transitions to `BUFFER_OWNERSHIP_STATE_RAL`. While a range is in that state, the RAL MAY modify or rearrange the corresponding octets of `buffer_data`. The caller MUST NOT write them, and any value it reads from them is undefined. The RAL MUST return all of `buffer_data` to `BUFFER_OWNERSHIP_STATE_CALLER` before emitting `RAL_Tx_Schedule_CommandEndEvent`, regardless of the reported `command_status`. To this end, the RAL MUST restore every frame-content octet it modified within the three regions of the `packet` to the value the caller had provided at the point of commit. The RAL SHOULD return ownership as soon as the octets are no longer required for the transmission, which MUST be signalled by an earlier `RAL_Tx_Buffer_Released_CommandEvent`.

The RAL MUST support the reuse of frame-content data from a previously received `packet`. In particular, the content of untouched ranges within the three regions (i.e. ranges not marked as touched via [§RAL_Buffer_Mark_Touched_Command](./ral-buffer-manager-service.md#ral_buffer_mark_touched_command)) MUST be adopted as they are.

_Implementation Note._ In case of a `GeneralPurposePacketBuffer` (GPPB), the `buffer_data` area is the buffer, so no special actions are required to enable the reuse of received data. In case of a `TightlyCoupledPacketBuffer` (TCPB), touched data MUST be transferred from `buffer_data` (the "shadow buffer") to the associated radio buffer before transmission. In addition, if a TCPB implementation hides separate radio buffers for receive and transmit behind a TCPB, then the implementation MUST ensure that all frame-content octets in the three regions not marked as touched by [§RAL_Buffer_Mark_Touched_Command](./ral-buffer-manager-service.md#ral_buffer_mark_touched_command) are copied from the internal receive buffer to the internal transmit buffer before transmission. Upon successful completion of each [§RAL_Tx_Data_Commit_Command](#ral_tx_data_commit_command), the transfers described above MUST be complete for all octets committed so far.


If this active command is cancelled, the RAL MUST abort the scheduled or ongoing transmission operation, and MUST emit `RAL_Tx_Schedule_CommandEndEvent` with `command_status = CANCELLED`, closing the command lifecycle.

When the lifecycle of this command is closed, the RAL MUST transition to `IDLE` state.


### `RAL_Tx_Data_Commit_Command`

**Summary:** Commits a portion of the PSDU for a scheduled transmission up to a given position, so that the caller can continue preparing data while the transmission is pending or running.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `addressing_mode` | Enumeration      | `ABSOLUTE`, `BUFFER_REGION_UNSECURED`, `BUFFER_REGION_AUTHENTICATED`, `BUFFER_REGION_ENCRYPTED` | Selects how `until` is interpreted. `ABSOLUTE` addresses the psdu directly; the three region values address the named region of the configured `psdu` partition.|
| `until` | Unsigned Integer | `[0, psdu_length - F]` if `addressing_mode = ABSOLUTE`; otherwise `[0, S]`, where `S` is the configured size of the named region. | End offset (exclusive) of the range to be committed. |

**Description:**

The command refers to the `PacketBuffer` passed as `packet` to the active `RAL_Tx_Schedule_Command`.

For an active transmission, the RAL maintains a _commit position_ `c`, expressed as an offset into the `PacketBuffer`'s `psdu`. Let `C = psdu_length - F`, the end of the three regions. Every frame-content octet before `c` is committed, and no frame-content octet at or after `c` is. When `RAL_Tx_Schedule_Command` is accepted, `c` is set to 0, or to `C` if `packet_implicit_commit = TRUE`. The commit position never decreases. Committed octets therefore always form a contiguous prefix of the frame content, and the regions of a partitioned buffer are committed in the order `BUFFER_REGION_UNSECURED`, `BUFFER_REGION_AUTHENTICATED`, `BUFFER_REGION_ENCRYPTED`, incrementally within each region.

The command specifies an absolute _target position_ `t`:
- If `addressing_mode = ABSOLUTE`, then `t = until`.
- Otherwise, `t` is `until` plus the configured sizes of all regions preceding the named region. The RAL performs this translation using the partition declared through [§RAL_Buffer_Configure_Regions_Command](./ral-buffer-manager-service.md#ral_buffer_configure_regions_command).

The RAL MUST evaluate the following conditions. If any of them applies, the commit position MUST remain unchanged.
1. If the RAL state is not `PENDING` or `RUNNING`, or if the active command is not `RAL_Tx_Schedule_Command`, the RAL MUST emit `RAL_Tx_Data_Commit_CommandEndEvent` with `command_status = ERROR_NOT_ALLOWED`.
2. If `until` lies outside its valid range, the RAL MUST emit `RAL_Tx_Data_Commit_CommandEndEvent` with `command_status = ERROR_INVALID_BUFFER_RANGE`.
3. If `t < c`, the RAL MUST emit `RAL_Tx_Data_Commit_CommandEndEvent` with `command_status = ERROR_COMMIT_OUT_OF_ORDER`.

Otherwise, the RAL MUST mark all octets in `[c, t)` as committed, set `c = t`, and emit `RAL_Tx_Data_Commit_CommandEndEvent` with `command_status = SUCCESS`. If `t = c`, the command has no effect and completes with `command_status = SUCCESS`.

<!-- NOTE: we allow an implicit commit through regions because:
- Region-relative addressing then behaves exactly like ABSOLUTE.
- The whole PSDU can be committed with one command.
 -->
_Note._ A target position within a selected region implies that all not yet committed octets of the preceding regions are committed as well. In particular, `addressing_mode = BUFFER_REGION_ENCRYPTED` with `until` equal to the size of that region commits all frame-content octets.

As described in [§Ownership of Buffer Data](./ral-buffer-manager-service.md#ownership-of-buffer-data), committed octets transition to `BUFFER_OWNERSHIP_STATE_RAL`. The caller MUST NOT modify them until the RAL returns their ownership by emitting `RAL_Tx_Buffer_Released_CommandEvent` or `RAL_Tx_Schedule_CommandEndEvent`.

If the `packet` is a `TightlyCoupledPacketBuffer` (TCPB), the RAL MUST ensure, upon successful completion of this command, that the associated radio buffer holds the committed content of every octet before `c`. This is the value in `psdu` for octets marked as touched via [§RAL_Buffer_Mark_Touched_Command](./ral-buffer-manager-service.md#ral_buffer_mark_touched_command), and the retained value for all other octets.

### `RAL_Nonce_Commit_Command`

**Summary:** Declares the nonce referenced by the active scheduling command as committed.

**Parameters:** None.

**Description:**

The lifecycle of the nonce value is coupled to the lifecycle of the associated radio operation. Once committed, the nonce MUST remain in that state until the corresponding `RAL_Rx_Schedule_Command` or `RAL_Tx_Schedule_Command` concludes with a `CommandEndEvent`. 

The RAL MUST NOT retain a nonce value in committed state across commands, even if the caller reuses the same memory space for the nonce.

If the nonce is already committed, the RAL MUST immediately emit a `RAL_Nonce_Commit_CommandEndEvent` with `command_status = ERROR_NONCE_ALREADY_COMMITTED`. 

<!-- NOTE: For simplicity, we do not allow to commit the nonce in advance, i.e., prior to scheduling. In case the caller has a ready, fixed nonce already upon scheduling, the caller can use the `nonce_implicit_commit` parameter of the respective scheduling command. -->
If neither `RAL_Tx_Schedule_Command` nor `RAL_Rx_Schedule_Command` is currently active, the RAL MUST immediately emit a `RAL_Nonce_Commit_CommandEndEvent` with `command_status = ERROR_NOT_ALLOWED`. 

If the command fails for any other reason, the RAL MUST immediately emit a `RAL_Nonce_Commit_CommandEndEvent` whose `command_status` is not `SUCCESS`.

If the command is successful, the RAL MUST acknowledge the nonce as committed, emitting `RAL_Nonce_Commit_CommandEndEvent` with `command_status = SUCCESS`.

### `RAL_Preschedule_Command`

**Summary:** Informs the RAL that a radio operation will be scheduled in the near future.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `command_id` | Enumeration | `RAL_Rx_Schedule_Command`, `RAL_Tx_Schedule_Command` | Forthcoming command. |
| `op_at` | `RalInstant` | As defined in [§RalInstant](ral-overview.md#ralinstant) | Point in time at which the next operation is planned to start. For transmission, it corresponds to `tx_at`. For reception, it corresponds to `rx_min_at`. |

**Description:**

`RAL_Preschedule_Command` is an optional hint that informs the RAL of a forthcoming scheduling command, either `RAL_Tx_Schedule_Command` or `RAL_Rx_Schedule_Command`. This command does not transition the RAL state.

The RAL MUST validate that `op_at` is in the future. 
If this validation fails, the RAL MUST emit `RAL_Preschedule_CommandEndEvent` with `command_status = ERROR_INVALID_PARAMETER`, closing the command lifecycle.

If validation succeeds, the RAL MUST emit `RAL_Preschedule_CommandEndEvent` with `command_status = SUCCESS`, closing the command lifecycle.

The RAL MAY use the hint to improve scheduling efficiency.
<!--TODO: Provide intuition how-->


### `RAL_Cancel_Command`

**Summary:** Cancels the active command, if any, which causes the RAL to abort any scheduled or ongoing radio operation (transmission or reception).

**Parameters:** None.

**Description:**

`RAL_Cancel_Command` requests to terminate the active command and abort the ongoing radio operation, if any.

The RAL MUST validate that a command is active, i.e., the RAL state is `PENDING` or `RUNNING`. If no command is active, the RAL MUST emit `RAL_Cancel_CommandEndEvent` with `command_status = ERROR_ACTIVE_COMMAND_NOT_FOUND`, closing the command lifecycle.

If a command is active, the RAL MUST cancel the [active command](ral-overview.md#active-command).
If the RAL is in `PENDING` state, the RAL MUST abort the scheduled radio operation and emit `RAL_Cancel_CommandEndEvent` with `command_status = SUCCESS`, closing the command lifecycle.
If the operation is in `RUNNING` state, the RAL MUST abort the ongoing radio operation immediately and emit `RAL_Cancel_CommandEndEvent` with `command_status = SUCCESS`, closing the command lifecycle.

If the RAL emits `RAL_Cancel_CommandEndEvent` with `command_status = SUCCESS`, the RAL MUST prevent the emission of events solicited by the cancelled active command,
except for events with `command_status = CANCELLED`.
<!--
[NOTE](https://github.com/openstx/wg-radio-abstraction-layer/pull/67#discussion_r3513163642): We can presume a strict order by associating the 'effective execution' of the Cancel command and the completion of a radio operation with single points in time, respectively. Although the execution of a Cancel command takes some microseconds, I think we can presume that internally there is a point that acts (or can act) like a barrier. If a radio operation finishes before this point (but after calling Cancel), then this should be considered as the radio op completing before Cancel's 'effective execution', i.e. the radio op should finish undisturbed while Cancel returns with ERROR_ACTIVE_COMMAND_NOT_FOUND. If the radio op ends after this point, then Cancel should suppress further events from the radio op (as still being able to successfully complete in a small time window after the cancel point is an artifact of finite-speed program execution) and return with SUCCESS.

We do not need to define the temporal position of the cancel point in the standard, just its existence. This ensures a strict relation beween Cancel's return value and further radio op events:
- If cancel returns with SUCCESS, then there will be no more radio op events.
- If cancel returns with ERROR_ACTIVE_COMMAND_NOT_FOUND, then Cancel has no influence on the events of the previous radio operation. They arrive (or not) as specified, since Cancel didn't 'hit' this radio op. 
-->


Note that upon success, as a result of closing the lifecycle of the active command, this command will cause the RAL to transition to the `IDLE` state (see [§State Transitions](ral-overview.md#state-transitions)), as expressed in the description of the affected commands.

### `RAL_State_Wakeup_Command`

**Summary:** Transitions the RAL from `SLEEP` to `IDLE` state.

**Parameters:** None.

**Description:**

`RAL_State_Wakeup_Command` requests the RAL to transition from `SLEEP` to `IDLE` state, making the radio operational and ready to execute commands.

If the RAL state is not `SLEEP`, the RAL MUST emit `RAL_State_Wakeup_CommandEndEvent` with `command_status = ERROR_NOT_ALLOWED`, closing the command lifecycle.

Otherwise, the RAL MUST transition to `IDLE` state (see [§State Transitions](ral-overview.md#state-transitions)) and MUST emit `RAL_State_Wakeup_CommandEndEvent` with `command_status = SUCCESS`, closing the command lifecycle.


### `RAL_State_Sleep_Command`

**Summary:** Transitions the RAL from `IDLE` to `SLEEP` state.

**Parameters:** None.

**Description:**

`RAL_State_Sleep_Command` requests the RAL to transition from `IDLE` to `SLEEP` state, allowing the radio hardware to enter a power-saving mode.

If the RAL state is not `IDLE`, the RAL MUST emit `RAL_State_Sleep_CommandEndEvent` with `command_status = ERROR_NOT_ALLOWED`, closing the command lifecycle.

Otherwise, the RAL MUST transition to `SLEEP` state (see [§State Transitions](ral-overview.md#state-transitions)) and MUST emit `RAL_State_Sleep_CommandEndEvent` with `command_status = SUCCESS`, closing the command lifecycle.


### `RAL_State_Get_Command`

**Summary:** Returns the current state of the RAL.

**Parameters:** None.

**Description:**

`RAL_State_Get_Command` queries the current state of the RAL. This command does not transition the RAL state.

The RAL MUST emit a `RAL_State_Get_CommandEndEvent` with `ral_state` equal to the RAL state, described in !REF.
<!--TODO: Add reference to RAL state description-->

<!-- The RAL MUST validate that an operation is active. If no operation is active, the RAL MUST emit `RAL_State_Get_CommandEndEvent` with `command_status = ERROR_ACTIVE_COMMAND_NOT_FOUND`. -->
<!-- If the command is accepted, the RAL MUST emit exactly one `RAL_State_Get_CommandEndEvent`. -->
<!--CHECK: Now due to sequential operation, this returns the state of the RAL, not a command-->


<!------------------------------------------------------------------------>


## Events


### `RAL_CommandStatusEvent`

**Soliciting command:** `RAL_Rx_Schedule_Command`, `RAL_Tx_Schedule_Command`

**Summary:** Reports whether the addressed scheduling command was accepted for processing.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `command_id` | Enumeration | `RAL_Rx_Schedule_Command`, `RAL_Tx_Schedule_Command` | Identifies the command invocation being reported. |
| `command_status` | Enumeration | `ACCEPTED`, `REJECTED` | Acceptance or rejection status. `ACCEPTED` is the only acceptance value. |

<!--CHECK: Should we replace `command_id` with `command_id` to be more explicit?-->
<!--CHECK: Should we replace `command_status` with `command_status` to be more explicit?-->

**Description:** The RAL emits `RAL_CommandStatusEvent` after intake validation for the addressed scheduling command. `command_id` identifies the command. If `command_status = ACCEPTED`, the command-specific event sequence continues. Any other `command_status` value rejects the command invocation, closing its lifecycle.


### `RAL_Rx_Schedule_CommandEndEvent`

**Soliciting command:** `RAL_Rx_Schedule_Command`

**Summary:** Reports the final outcome of a scheduled receive operation.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `command_status` | Enumeration | `SUCCESS`, `CANCELLED`, `ERROR_DETECTION_TIMEOUT`, `ERROR_PTA_TIMEOUT`, `ERROR_FRAME_TIMEOUT`, `ERROR_FRAME_REJECTED`, `ERROR_FRAME_CORRUPTED`, `ERROR_NONCE_REQUIRED`, `ERROR_NOT_AUTHENTICATED`, `ERROR_DECRYPTION_FAILED`| Final execution status. |
| `t_rx` | `RalInstant` | As defined in [§RalInstant](ral-overview.md#ralinstant) | Reception timestamp of the PTA. Accuracy: ± `TRX_MEASUREMENT_ACCURACY`. Valid only if `command_status = SUCCESS`. |

**Description:** The RAL emits `RAL_Rx_Schedule_CommandEndEvent` upon conclusion of the accepted receive operation. This event is emitted only after `RAL_CommandStatusEvent` reported `command_status = ACCEPTED` for the same command invocation. It closes the lifecycle opened by `RAL_Rx_Schedule_Command`.


### `RAL_Tx_Schedule_CommandEndEvent`

**Soliciting command:** `RAL_Tx_Schedule_Command`

**Summary:** Reports the final outcome of a scheduled transmit operation.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `command_status` | Enumeration | `SUCCESS`, `CANCELLED`, `ERROR_TX_DATA_LATE`, `ERROR_NONCE_REQUIRED`, `ERROR_NONCE_LATE`, `ERROR_ENCRYPTION_FAILED`| Final execution status. |

**Description:** The RAL emits `RAL_Tx_Schedule_CommandEndEvent` upon conclusion of the accepted transmit operation. This event is emitted only after `RAL_CommandStatusEvent` reported `command_status = ACCEPTED` for the same command invocation. It closes the lifecycle opened by `RAL_Tx_Schedule_Command`.

### `RAL_Tx_Buffer_Released_CommandEvent`

**Soliciting command:** `RAL_Tx_Schedule_Command`

**Summary:** Reports that the packet's `buffer_data` has returned to `BUFFER_OWNERSHIP_STATE_CALLER` before the transmission has concluded.

**Parameters:** None.

**Description:** The RAL emits `RAL_Tx_Buffer_Released_CommandEvent` to signal that all of the active packet's `buffer_data` has returned to `BUFFER_OWNERSHIP_STATE_CALLER` and may be used by the caller again.

### `RAL_Rx_Data_Available_CommandEvent`

**Soliciting command:** `RAL_Rx_Schedule_Command`

**Summary:** Indicates that a PSDU octet at a requested trigger position has been delivered.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `psdu_index` | Unsigned Integer | Octet index within the PSDU | Index of the octet whose reception is being reported. |

**Description:** The RAL emits `RAL_Rx_Data_Available_CommandEvent` each time a PSDU octet at a position specified in `psdu_trigger_indexes` is made available.


### `RAL_Tx_Data_Commit_CommandEndEvent`

**Soliciting command:** `RAL_Tx_Data_Commit_Command`

**Summary:** Reports the final outcome of a data commit request.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_NOT_ALLOWED`, `ERROR_COMMIT_OUT_OF_ORDER`, `ERROR_INVALID_BUFFER_RANGE` | Outcome of the commit request. |

**Description:** The RAL emits `RAL_Tx_Data_Commit_CommandEndEvent` after processing a data commit request, closing the command lifecycle.

### `RAL_Nonce_Commit_CommandEndEvent`

**Soliciting command:** `RAL_Nonce_Commit_Command`

**Summary:** Reports the final outcome of a nonce commit request.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_NOT_ALLOWED`, `ERROR_NONCE_ALREADY_COMMITTED`, or any RAL-specific error status. | Outcome of the commit request. |

**Description:** The RAL emits `RAL_Nonce_Commit_CommandEndEvent` after processing a nonce commit request, closing the command lifecycle.


### `RAL_Preschedule_CommandEndEvent`

**Soliciting command:** `RAL_Preschedule_Command`

**Summary:** Reports that the preschedule hint has been acknowledged.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_INVALID_PARAMETER` | Outcome of the preschedule request. |

**Description:** The RAL emits `RAL_Preschedule_CommandEndEvent` after acknowledging the preschedule hint, closing the command lifecycle.


### `RAL_Cancel_CommandEndEvent`

**Soliciting command:** `RAL_Cancel_Command`

**Summary:** Reports the outcome of an abort request.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_ACTIVE_COMMAND_NOT_FOUND` | Outcome of the abort request. |

**Description:** The RAL emits `RAL_Cancel_CommandEndEvent` after processing a command cancellation request, closing the command lifecycle.


### `RAL_State_Wakeup_CommandEndEvent`

**Soliciting command:** `RAL_State_Wakeup_Command`

**Summary:** Reports the outcome of a request to transition the RAL to `IDLE` state.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_NOT_ALLOWED` | Outcome of the request. |

**Description:** The RAL emits `RAL_State_Wakeup_CommandEndEvent` after processing a `RAL_State_Wakeup_Command` request, closing the command lifecycle.


### `RAL_State_Sleep_CommandEndEvent`

**Soliciting command:** `RAL_State_Sleep_Command`

**Summary:** Reports the outcome of a request to transition the RAL to `SLEEP` state.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_NOT_ALLOWED` | Outcome of the request. |

**Description:** The RAL emits `RAL_State_Sleep_CommandEndEvent` after processing a `RAL_State_Sleep_Command` request, closing the command lifecycle.


### `RAL_State_Get_CommandEndEvent`

**Soliciting command:** `RAL_State_Get_Command`

**Summary:** Reports the current state of a queried radio operation.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `command_status` | Enumeration | `SUCCESS` | Outcome of the state query. |
| `ral_state` | Enumeration | `SLEEP`, `IDLE`, `PENDING`, `RUNNING` | Current RAL state |

<!--CHECK: Should we replace `command_status` with `command_status` to be more explicit?-->

**Description:** The RAL emits `RAL_State_Get_CommandEndEvent` after querying the RAL state, closing the command lifecycle.


<!------------------------------------------------------------------------>



