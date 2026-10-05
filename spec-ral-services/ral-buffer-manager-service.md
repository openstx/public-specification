# RAL Buffer Manager Service

## Introduction

The RAL Buffer Manager is responsible for managing the lifecycle of a [§Packet Buffer](./ral-overview.md#packetbuffer), and for the protection disposition of the octets it holds.

The management of packet buffers is meant to accomplish several goals:

1. Adapt to different hardware architectures:
   1. MCU and radio peripherals with substantial DMA capabilities (e.g., integrated in a single SoC).
   2. Separated MCU and radio ICs connected via slower external interfaces with limited or missing DMA capabilities.
2. Minimize data copying / data transfers.
3. Allow the caller to state what protection each octet requires without knowing the on-air frame layout.
<!-- CHECK: commented out for now, as the underlying VIEW-concept is not included yet -> to be discussed  -->
<!-- 3. Unified API (that fails gracefully) across OpenSTX layers for data manipulation. -->

The ultimate goal of the above is to create a buffer abstraction that enables efficient operations on different types of hardware, without requiring the user to handle hardware-specific details.

The normative behavior is defined in sections [Commands](#commands) and [Events](#events).

### Packet Buffer Specializations

As defined in [§Packet Buffer](./ral-overview.md#buffer-specializations), the standard distinguishes two types of `PacketBuffers`:
1. `GeneralPurposePacketBuffer` (GPPB),
2. `TightlyCoupledPacketBuffer` (TCPB).

When scheduling a reception or a transmission (as defined in [§RAL Executor](./ral-execution-service.md)), the caller can choose which type of buffer is used by allocating the preferred one and passing it through the `packet` parameter.

Within this document, the term “radio buffer” refers to a buffer located close to the radio peripheral, regardless of whether it is used explicitly through a TCPB, or implicitly through a GPPB.

The radio buffer associated with a TCPB is retained, with its content available for reuse across radio operations, until `RAL_Buffer_Delete_Command` dissolves the association. By contrast, the radio buffer backing a GPPB (if any) may be freed as soon as its data has been fetched into RAM.

Support for GPPBs is mandatory, whereas support for TCPBs is optional: an implementation whose target platform does not benefit from TCPBs may provide GPPBs only. GPPBs can always be used, even in systems that also support TCPBs (e.g., when there is no need to directly reuse the data in the associated radio buffer).

### PSDU Partitioning and Region Addressing

The frame content of a `PacketBuffer` may be partitioned into three regions, declared through [§RAL_Buffer_Configure_Regions_Command](#ral_buffer_configure_regions_command), excluding the FCS as defined in [§PSDULength](./ral-overview.md#psdulength). Partitioning applies to both buffer specializations.

The regions and their corresponding level of protection (where level refers to data being authenticated, encrypted, or both) are listed in the following table.

The cipher that authenticates and encrypts the regions is specified in [§RAL Security Service](./ral-security-service.md#the-cipher).

| Region Order | Name                          | Authentication | Encryption |
|--------------|-------------------------------|----------------|------------|
| 1st          | `BUFFER_REGION_UNSECURED`     | No             | No         |
| 2nd          | `BUFFER_REGION_AUTHENTICATED` | Yes            | No         |
| 3rd          | `BUFFER_REGION_ENCRYPTED`     | Yes            | Yes        |

Octet ranges within the PSDU may be addressed in two ways, selected through the `addressing_mode` parameter of the respective command:
1. **Absolute** — `from` and `to` as offsets into the PSDU, where offset `0` addresses the first PSDU octet. This mode is used throughout this document for unsecured buffers.
2. **Region-relative** — `from` and `to` as offsets into the _named region_ of the PSDU, where offset `0` addresses the first offset of the given region. The RAL translates region-relative addresses to absolute ones using the configured partition. An offset beyond the configured size of the named region is invalid. The RAL rejects such a range rather than truncating it or mapping any part of it into a neighbouring region, so that the protection level of every addressed octet matches the region named by the caller.

Region-relative addressing exists so that the caller can state what protection each octet requires without knowing where the octet lands on air. The caller addresses regions by name and by offset within the region; it does not address the protocol-specific frame. On a PHY whose on-air field order differs from region order, the RAL performs the necessary rearrangement as part of [§RAL_Tx_Data_Commit_Command](./ral-execution-service.md#ral_tx_data_commit_command).

Absolute addressing remains available on a partitioned buffer, but is **not** equivalent: a range given absolutely is not checked against region boundaries, and on a PHY requiring rearrangement the absolute positions before and after commit differ. A caller that has declared a partition SHOULD use region-relative addressing for octets whose protection matters.

<!-- TODO: rev comment -- > Reads a bit like a definition which maybe was not the goal? Maybe "For cases in which encryption and/or authentication are not necessary, ..." 
-->
An unsecured buffer is declared explicitly, by a partition in which `BUFFER_REGION_AUTHENTICATED` and `BUFFER_REGION_ENCRYPTED` have length zero, so that all octets belong to `BUFFER_REGION_UNSECURED`. Omitting the declaration does not yield an unsecured buffer, as the scheduling commands reject a buffer without a partition.

### Coherence and the point of commit

The Buffer Manager provides a unified interface for accessing and modifying contents of any `PacketBuffer` type. It defines a coherence protocol between `buffer_data` and the radio buffer. Consistent reads are enabled by calling the [§RAL_Buffer_Ensure_Available_Command](#ral_buffer_ensure_available_command). Modifications can be reported using the [§RAL_Buffer_Mark_Touched_Command](#ral_buffer_mark_touched_command), with write-back coherence guaranteed after [§RAL_Tx_Data_Commit_Command](./ral-execution-service.md#ral_tx_data_commit_command) completes successfully. Moreover, this interface allows selectively reading and modifying only small chunks of the PSDU. This could be used for example in Glossy, so that forwarders may only need to modify the relay counter, leaving the rest untouched, or in Chaos to read only the progress bits and act accordingly if possible. Note that the benefits of small-chunk operations are most pronounced in setups where the MCU and radio are connected via a slower external interface.

<!-- TODO: Re 'There is no separate region-commit step ...'
> you mean are sent in plaintext across the RAL interface? If so, I suggest to specify 
-->
On a partitioned buffer, the time between [§RAL_Tx_Data_Commit_Command](./ral-execution-service.md#ral_tx_data_commit_command) and the actual transmission is when the RAL applies security: region rearrangement into on-air order, encryption of `BUFFER_REGION_ENCRYPTED`, and computation of the MIC over `BUFFER_REGION_AUTHENTICATED` and `BUFFER_REGION_ENCRYPTED`. There is no separate region-commit step. The consequence for the caller is that the octets it writes are plaintext, unless encrypted at a higher layer, for their whole observable lifetime.

### Ownership of Buffer Data

While a `PacketBuffer` is used for a radio operation, the RAL takes temporary ownership of octets within `buffer_data`. The caller cannot safely use an octet range while that range is RAL-owned: its contents are neither guaranteed to be valid nor stable, and its layout need not match the partition the caller declared.

**Example.** For a `GeneralPurposePacketBuffer`, the RAL may directly rearrange octets into PHY-specific PPDU order or encrypt them in place, so while a range is RAL-owned the octets the caller wrote are not necessarily the octets it would read back.

<!-- TODO: add link to RAL_Tx_Data_Commit_Command -->
**Ownership transfer on TX.** An octet range enters `BUFFER_OWNERSHIP_STATE_RAL` once it is _committed_ for a transmission initiated with the [§RAL_Tx_Schedule_Command](./ral-execution-service.md#ral_tx_schedule_command). The RAL returns all of `buffer_data` to `BUFFER_OWNERSHIP_STATE_CALLER` at the latest upon [§RAL_Tx_Schedule_CommandEndEvent](./ral-execution-service.md#ral_tx_schedule_commandendevent), and earlier where it emits [§RAL_Tx_Buffer_Released_CommandEvent](./ral-execution-service.md#ral_tx_buffer_released_commandevent). Before returning ownership, the RAL restores every octet it modified within the `psdu` region to the value the caller had provided at the point of commit.

**Ownership transfer on RX.** All of `buffer_data` enters `BUFFER_OWNERSHIP_STATE_RAL` when [§RAL_Rx_Schedule_Command](./ral-execution-service.md#ral_rx_schedule_command) is accepted, and returns to `BUFFER_OWNERSHIP_STATE_CALLER` upon [§RAL_Rx_Schedule_CommandEndEvent](./ral-execution-service.md#ral_rx_schedule_commandendevent). Certain octet ranges may however be made available to the caller for reading before that point, following the emission of a corresponding [§RAL_Rx_Data_Available_CommandEvent](./ral-execution-service.md#ral_rx_data_available_commandevent).


### Related Documents

| Document                           | Description                          |
|------------------------------------| ------------------------------------ |
| [RAL Introduction](./introduction.md)   | Introduction to the Radio Abstraction Layer  |
| [RAL Executor](./ral-execution-service.md)   | Relies on `PacketBuffer`s to carry the PSDUs it transmits and receives. Its `RAL_Tx_Data_Commit_Command` is where region rearrangement, encryption and MIC computation occur. |
| [RAL Security Service](./ral-security-service.md) | Specifies the cipher that protects the regions, and the key it uses. |


## Commands

The command set divides into three groups. The **lifecycle commands** (`Create`, `Delete`) manage `PacketBuffer` instances. The **content commands** (`Ensure_Available`, `Mark_Touched`) move octets between `buffer_data` and the radio buffer. The **region command** (`Configure_Regions`) declares the three-region partition; the partition is used by [§RAL_Tx_Data_Commit_Command](./ral-execution-service.md#ral_tx_data_commit_command).

**Typical secured TX sequence.** `RAL_Buffer_Create_Command` → `RAL_Buffer_Configure_Regions_Command` → (write data to the buffer) → `RAL_Buffer_Mark_Touched_Command` (per region) → `RAL_Tx_Schedule_Command` with the `nonce` → `RAL_Tx_Data_Commit_Command` (incrementally, or once for the entire buffer) → (RAL encrypts and computes the MIC) → `RAL_Tx_Schedule_CommandEndEvent`.

**Typical secured RX sequence.** `RAL_Buffer_Create_Command` → `RAL_Buffer_Configure_Regions_Command` → `RAL_Rx_Schedule_Command` with `nonce_implicit_commit = FALSE` and the index of the last octet the nonce is formed from in `psdu_trigger_indexes` → `RAL_Rx_Data_Available_CommandEvent` (at that octet) → `RAL_Buffer_Ensure_Available_Command` (over the octets the nonce is formed from) → `RAL_Nonce_Commit_Command` → (frame arrives, RAL decrypts and verifies) → `RAL_Rx_Schedule_CommandEndEvent` → `RAL_Buffer_Ensure_Available_Command`.

### `RAL_Buffer_Create_Command`

**Summary:** Creates an instance of a `PacketBuffer`, as defined in [§Packet Buffer](./ral-overview.md#packetbuffer).

**Parameters:**

| Name          | Type             | Valid Range   | Description                        |
| ------------- | ---------------- | ------------- | ---------------------------------- |
| `buffer_type` | Enumeration      | `BUFFER_TYPE_GENERAL_PURPOSE`, `BUFFER_TYPE_TIGHTLY_COUPLED` | Specifies the type of the buffer to be created.  | 
| `buffer_data` | Octet String     | — | Designates the caller-provided octet region to be used for the `PacketBuffer`'s `buffer_data`. For a `GeneralPurposePacketBuffer`, this region holds the authoritative payload. For a `TightlyCoupledPacketBuffer`, it serves as the staging region (or "shadow buffer") into and out of which the RAL transfers octets through commands [Ensure_Available](#ral_buffer_ensure_available_command), [Mark_Touched](#ral_buffer_mark_touched_command), and [Commit_Tx_Data](./ral-execution-service.md#ral_tx_data_commit_command); its contents are defined only within that protocol. |
| `buffer_size` | Unsigned Integer | — | Number of octets that fit into `buffer_data`. |

**Description:**

The RAL MUST create a `PacketBuffer`, associate it with the provided `buffer_data` region, and immediately emit a `RAL_Buffer_Create_CommandEndEvent` with `command_status = SUCCESS` and the `packet_buffer` parameter referring to the created `PacketBuffer` instance.

The RAL MUST support all PSDU lengths (and corresponding minimal `buffer_size`s), the only limitation being the PHY-specific maximal PSDU length.

The RAL MUST create the `PacketBuffer` instance according to the requested `buffer_type`:

- a `GeneralPurposePacketBuffer` (GPPB) if `buffer_type = BUFFER_TYPE_GENERAL_PURPOSE`,
- a `TightlyCoupledPacketBuffer` (TCPB) if `buffer_type = BUFFER_TYPE_TIGHTLY_COUPLED`.

The caller MUST allocate sufficient space for `buffer_data`, covering the PSDU plus the headroom and tailroom required by the RAL, the latter two together sized by the [bufferSpaceMargin](./ral-config-service.md#generic-configuration-attributes) attribute.    

<!-- TODO: First sentence could be misunderstood as if the caller must provide additional space. Should be clarified. -->
_Note._ Additional headroom may be needed, for instance, for a length-field imposed by the PPDU format. The tailroom can be used to accommodate a MIC, when authentication is enabled. 

<!-- TODO: except for temporal PPDU rearrangement (can include adaptation of psdu) -->
The RAL MAY place the PSDU freely within `buffer_data`, provided the PSDU occupies a single contiguous octet region, and it MUST set the `psdu` reference in the returned `packet_buffer` accordingly and MUST NOT change it throughout the `PacketBuffer`'s lifetime. 

<!-- CHECK: Perhaps we should enforce a partitioning on create, duping the parameters from Configure_Regions -->
A newly created `PacketBuffer` has no partition in the sense of [§PSDU Partitioning and Region Addressing](#psdu-partitioning-and-region-addressing). The three region references specified in [§Packet Buffer](./ral-overview.md#packetbuffer) remain undefined until a partition is declared through [§RAL_Buffer_Configure_Regions_Command](#ral_buffer_configure_regions_command). Until then, every command that depends on the partition fails, as specified for that command.

If the command fails, the RAL MUST immediately emit a `RAL_Buffer_Create_CommandEndEvent` whose `command_status` is not `SUCCESS`, using either a status value defined in subsequent paragraphs or a RAL-specific one.

The following rules (1-2) apply if `buffer_type = BUFFER_TYPE_TIGHTLY_COUPLED`:
1. If no sufficient radio buffer space is available, i.e., when the requested `buffer_size` is greater than the value returned by `RAL_Buffer_Get_TCPB_Available_Capacity_Command`, then the RAL MUST immediately emit `RAL_Buffer_Create_CommandEndEvent` with `command_status = ERROR_BUFFER_CREATE_TCPB_NO_SPACE_AVAILABLE`.
2. The RAL MUST ensure that the created TCPB retains exclusive ownership of the underlying radio buffer, until the association dissolves with the completion of the `RAL_Buffer_Delete_Command` for this TCPB.

<!-- CHECK: On one hand, this formulation is quite vague . On the other hand, our goal is not to detail concurrency control mechanisms. -->
If the RAL implementation permits concurrent access, it MUST ensure that this command is atomic.

### `RAL_Buffer_Delete_Command`

**Summary:** Deletes the given `PacketBuffer` instance.

**Parameters:**

| Name            | Type      | Valid Range | Description  |
| --------------- | --------- | ----------- | ------------ |
| `packet_buffer` | [§PacketBuffer](./ral-overview.md#packetbuffer) | -  | The `PacketBuffer` instance to be deleted. |

**Description:**

If the `packet_buffer` is is currently used in an active [§RAL_Rx_Schedule_Command](./ral-execution-service.md#ral_rx_schedule_command) or [§RAL_Tx_Schedule_Command](./ral-execution-service.md#ral_tx_schedule_command), the RAL MUST refuse the command, emitting `RAL_Buffer_Delete_CommandEndEvent` with `command_status = ERROR_BUFFER_NOT_READY`. 

Otherwise, the RAL MUST delete the `packet_buffer` immediately and emit `RAL_Buffer_Delete_CommandEndEvent` with `command_status = SUCCESS`.

If the given the buffer's `buffer_type` is `BUFFER_TYPE_TIGHTLY_COUPLED`, then the RAL MUST release the underlying radio buffer before emitting `RAL_Buffer_Delete_CommandEndEvent`, so that the radio buffer is available for future use.

Deleting a `PacketBuffer` discards any configured region partition. A RAL that has a cipher MUST set every octet of `buffer_data` to zero before releasing it (a received frame leaves its decrypted payload there). A `PacketBuffer` holds no key material: the key is installed in the RAL through [§RAL_Crypto_ConfigureKey_Command](./ral-security-service.md#ral_crypto_configurekey_command), and the nonce is supplied with each scheduling command.

<!-- TODO: rev comment > Reads like a very C-level concern, not relevant here? 
If the point is to ensure no dangling ref with the Core buffer, it is to be handled explicitly (in Core) 
-->
_Note._ After this command's completion, the memory region that was used for `PacketBuffer`'s `buffer_data` (regardless of the buffer type), is no longer guaranteed to contain valid data.

### `RAL_Buffer_Configure_Regions_Command`

**Summary:** Declares the three-region partition of the PSDU.

**Parameters:**

| Name           | Type       | Valid Range      | Description       |
|----------------|------------| -----------------|-------------------|
| `packet_buffer`| Reference to a [§PacketBuffer](./ral-overview.md#packetbuffer) | - | The referenced `PacketBuffer` instance for which the partition is configured. |
| `region_unsecured_size`         | Unsigned Integer | — | Size of `BUFFER_REGION_UNSECURED`. |
| `region_authenticated_size` | Unsigned Integer | — | Size of `BUFFER_REGION_AUTHENTICATED`.|
| `psdu_length` | `PSDULength` | As defined in [§PSDULength](./ral-overview.md#psdulength), and MUST NOT be less than `region_unsecured_size + region_authenticated_size + F` | Total PSDU length; `BUFFER_REGION_ENCRYPTED` ends at `psdu_length - F`. |

**Description:**

`RAL_Buffer_Configure_Regions_Command` declares the partition for the given `packet_buffer` instance. Two parameters fix the region boundaries, and `psdu_length - F` fixes the end of the third region, so any region may be zero-length.

In accordance with the requested partition, the RAL MUST set the `packet_buffer`'s region pointers `psdu_region_unsecured`, `psdu_region_authenticated`, and `psdu_region_encrypted`, and their corresponding sizes `psdu_region_unsecured_size`, `psdu_region_authenticated_size`, and `psdu_region_encrypted_size`. The RAL MUST set the `packet_buffer`'s `psdu_length` to the value of the `psdu_length` parameter. The `psdu` reference is fixed at [§RAL_Buffer_Create_Command](#ral_buffer_create_command) and MUST NOT be modified by this command.

If any octet of the given `PacketBuffer` is in `BUFFER_OWNERSHIP_STATE_RAL`, the RAL MUST refuse the command, emitting `RAL_Buffer_Configure_Regions_CommandEndEvent` with `command_status = ERROR_BUFFER_NOT_READY`, and the previously configured partition MUST remain in effect.

If `psdu_length` exceeds `buffer_size - bufferSpaceMargin` of the given `PacketBuffer`, the RAL MUST refuse the command, emitting `RAL_Buffer_Configure_Regions_CommandEndEvent` with `command_status = ERROR_INVALID_BUFFER_RANGE`, and the previously configured partition MUST remain in effect.

A RAL that has a cipher MUST support every partition. A RAL that has no cipher supports only partitions whose `BUFFER_REGION_AUTHENTICATED` and `BUFFER_REGION_ENCRYPTED` are empty; if the requested partition has a non-empty one, the RAL MUST refuse the command, emitting `RAL_Buffer_Configure_Regions_CommandEndEvent` with `command_status = ERROR_UNSUPPORTED_PARTITION`, and the previously configured partition MUST remain in effect. Whether a key is installed does not change which partitions the RAL supports; the scheduling commands check the key.

If `BUFFER_REGION_AUTHENTICATED` or `BUFFER_REGION_ENCRYPTED` of the partition is non-empty and `psdu_length` plus `micSize` exceeds `PHY_MAX_PSDU_LENGTH`, the RAL MUST refuse the command, emitting `RAL_Buffer_Configure_Regions_CommandEndEvent` with `command_status = ERROR_INVALID_BUFFER_RANGE`, and the previously configured partition MUST remain in effect.

_Note._ A region pointer conveys the _position_ of a region, not the validity of its contents. Only ranges requested through [RAL_Buffer_Ensure_Available_Command](./ral-buffer-manager-service.md#ral_buffer_ensure_available_command) are guaranteed coherent. On a `TightlyCoupledPacketBuffer`, octets outside that range may not yet have been fetched, and reading them through a pointer yields undefined data.

This command MUST be issued before [§RAL_Tx_Schedule_Command](./ral-execution-service.md#ral_tx_schedule_command) on the TX path and before [§RAL_Rx_Schedule_Command](./ral-execution-service.md#ral_rx_schedule_command) on the RX path, since both depend on the region boundaries.

On the RX path, `psdu_length` declares the longest PSDU the reception accepts, not counting the MIC.

If the partition was configured successfully, the RAL MUST emit `RAL_Buffer_Configure_Regions_CommandEndEvent` with `command_status = SUCCESS`.

Reconfiguring a partition after content has been written discards no content, but the octets already written are reinterpreted under the new boundaries. A caller that changes the partition SHOULD rewrite the affected regions.

### `RAL_Buffer_Ensure_Available_Command`

**Summary:** Ensures that the specified octet range is available in the `PacketBuffer`'s `buffer_data`.

**Parameters:**

| Name              | Type             | Valid Range   | Description                                          |
| ----------------- | ---------------- | ------------- | ---------------------------------------------------- |
| `packet_buffer`   | [§PacketBuffer](./ral-overview.md#packetbuffer) | - | The referenced `PacketBuffer` instance. |
| `addressing_mode` | Enumeration | `ABSOLUTE`, `BUFFER_REGION_UNSECURED`, `BUFFER_REGION_AUTHENTICATED`, `BUFFER_REGION_ENCRYPTED` | Selects how `from` and `to` are interpreted. `ABSOLUTE` addresses the `psdu` directly; the three region values address the named region of the configured partition. |
| `from` | Unsigned Integer | `[0, S]`, where `S` is `psdu_length - F` if `addressing_mode = ABSOLUTE`, otherwise the configured size of the named region | Start offset (inclusive) of the requested octet range. |
| `to`              | Unsigned Integer | `[0, S]`, where `S` is as for `from` | End offset (exclusive) of the requested octet range. |

**Description:**

The RAL MUST ensure that the octet range `[from, to)` contains valid data.

Where `addressing_mode` names one of the three regions, the requested range is interpreted within that region and the RAL translates it to absolute positions using the partition declared through [§RAL_Buffer_Configure_Regions_Command](#ral_buffer_configure_regions_command). 

The RAL MUST evaluate the following conditions and complete the command immediately if one of them applies:
1. If no partition has been declared for the `PacketBuffer` through [§RAL_Buffer_Configure_Regions_Command](#ral_buffer_configure_regions_command), the RAL MUST emit `RAL_Buffer_Ensure_Available_CommandEndEvent` with `command_status = ERROR_REGION_NOT_CONFIGURED`, regardless of `addressing_mode`.
2. If `to < from`, or if `to` exceeds the applicable bound, the RAL MUST emit `RAL_Buffer_Ensure_Available_CommandEndEvent` with `command_status = ERROR_INVALID_BUFFER_RANGE`.
3. If any octet in the requested range is in `BUFFER_OWNERSHIP_STATE_RAL`, as detailed in [§Ownership of Buffer Data](#ownership-of-buffer-data), the RAL MUST emit `RAL_Buffer_Ensure_Available_CommandEndEvent` with `command_status = ERROR_DATA_NOT_READY`. The exceptions are ranges whose readiness has been signalled early through the `RAL_Rx_Data_Available_CommandEvent` solicited by [§RAL_Rx_Schedule_Command](./ral-execution-service.md#ral_rx_schedule_command).
<br><br> _Note._ The RAL carries no authentication guarantee for octets returned before the ownership is transferred back to the caller. The caller is responsible for treating such content as untrusted.

Otherwise, the RAL MUST make the range available as specified in the following paragraphs and then emit `RAL_Buffer_Ensure_Available_CommandEndEvent` with `command_status = SUCCESS`.

For `addressing_mode = BUFFER_REGION_ENCRYPTED`, the RAL provides the recovered plain representation of the data, as detailed in [§RAL_Rx_Schedule_Command](./ral-execution-service.md#ral_rx_schedule_command).

For a `TightlyCoupledPacketBuffer`, upon successful completion, the content of the specified range in `buffer_data` MUST be identical to that of the corresponding range in the underlying radio buffer. The RAL MUST fetch every octet in the given range that has not already been fetched from the underlying radio buffer. The RAL SHOULD NOT fetch any octet more than once. The RAL MAY fetch octets outside the given range (e.g. in advance) if this improves overall performance.

<!-- TODO: "may do nothing" (-> maybe dependent on internal GPPB implementation)? -->
For a `GeneralPurposePacketBuffer`, this command does effectively nothing, since the data is already available in RAM.

If the command fails for a reason not specified in this section, the RAL MUST immediately emit `RAL_Buffer_Ensure_Available_CommandEndEvent` with a RAL-specific `command_status` other than `SUCCESS`.

### `RAL_Buffer_Mark_Touched_Command`

**Summary:** Informs the RAL that the specified octet range within the given `PacketBuffer`'s `buffer_data` has been modified.

**Parameters:**

| Name              | Type             | Valid Range   | Description                                          |
| ----------------- | ---------------- | ------------- | ---------------------------------------------------- |
| `packet_buffer`   | [§PacketBuffer](./ral-overview.md#packetbuffer) | - | The referenced `PacketBuffer` instance. |
| `addressing_mode` | Enumeration | `ABSOLUTE`, `BUFFER_REGION_UNSECURED`, `BUFFER_REGION_AUTHENTICATED`, `BUFFER_REGION_ENCRYPTED` | Selects how `from` and `to` are interpreted. `ABSOLUTE` addresses the `psdu` directly; the three region values address the named region of the configured partition. |
| `from` | Unsigned Integer | `[0, S]`, where `S` is `psdu_length - F` if `addressing_mode = ABSOLUTE`, otherwise the configured size of the named region | Start offset (inclusive) of the modified octet range. |
| `to`              | Unsigned Integer | `[0, S]`, where `S` is as for `from` | End offset (exclusive) of the modified octet range. |

**Description:**

This command informs the RAL that the caller has modified the octet range `[from, to)` of the `PacketBuffer`'s `buffer_data`. 

Where `addressing_mode` names one of the three regions, the requested range is interpreted within that region and the RAL translates it to absolute positions using the partition declared through [§RAL_Buffer_Configure_Regions_Command](#ral_buffer_configure_regions_command). 

The RAL MUST evaluate the following conditions and complete the command immediately if one of them applies:
1. If no partition has been declared for the `PacketBuffer` through [§RAL_Buffer_Configure_Regions_Command](#ral_buffer_configure_regions_command), the RAL MUST emit `RAL_Buffer_Mark_Touched_CommandEndEvent` with `command_status = ERROR_REGION_NOT_CONFIGURED`, regardless of `addressing_mode`.
2. If `to < from`, or if from or to lies outside its valid range, the RAL MUST emit `RAL_Buffer_Mark_Touched_CommandEndEvent` with `command_status = ERROR_INVALID_BUFFER_RANGE`.
3. If any octet in the range is in `BUFFER_OWNERSHIP_STATE_RAL`, as detailed in [§Ownership of Buffer Data](#ownership-of-buffer-data), the RAL MUST emit `RAL_Buffer_Mark_Touched_CommandEndEvent` with `command_status = ERROR_DATA_NOT_READY`.

Otherwise, the RAL MUST record the range as touched and emit `RAL_Buffer_Mark_Touched_CommandEndEvent` with `command_status = SUCCESS`.

For a `TightlyCoupledPacketBuffer`, the RAL MAY use the information obtained through this command to transfer the octet range `[from, to)` from `buffer_data` to the corresponding location in the underlying radio buffer, either immediately or by the time [§RAL_Tx_Data_Commit_Command](./ral-execution-service.md#ral_tx_data_commit_command) completes successfully.

<!-- TODO: "may do nothing" (-> maybe dependent on internal GPPB implementation)? -->
For a `GeneralPurposePacketBuffer`, this command does effectively nothing, since the RAL reads modifications directly from `buffer_data` and no write-back to a radio buffer is required.

_Note._ This command can be called multiple times for the same range or parts thereof. The RAL decides internally which data transfers it performs and when, e.g. whether it uses multiple immediate transfers, or marks modified ranges and transfers all data at once. Eventually, coherence between any range within the `PacketBuffer`'s `buffer_data` and its corresponding range in the associated radio buffer is only guaranteed once [§RAL_Tx_Data_Commit_Command](./ral-execution-service.md#ral_tx_data_commit_command) completes successfully for that range, i.e., the RAL MAY defer the data transfer up to this point.

_Note._ Modifying an octet of the `PacketBuffer`'s `buffer_data` leads to undefined behavior in either of the following cases:
1. The modification is made while the octet is in `BUFFER_OWNERSHIP_STATE_RAL`, e.g. after it has been committed for transmission (implicitly or through [§RAL_Tx_Data_Commit_Command](./ral-execution-service.md#ral_tx_data_commit_command)) and before the RAL has returned its ownership (see [§Ownership of Buffer Data](#ownership-of-buffer-data)).
2. The modification is not reported through a `RAL_Buffer_Mark_Touched_Command` covering the octet before the octet is committed.

If the command fails for a reason not specified in this section, the RAL MUST immediately emit `RAL_Buffer_Mark_Touched_CommandEndEvent` with a RAL-specific `command_status` other than `SUCCESS`.

### `RAL_Buffer_Get_TCPB_Available_Capacity_Command`

**Summary:** Returns the number of octets that can be currently allocated to a  `TightlyCoupledPacketBuffer`.

**Parameters:** None.

**Description:**

The RAL MUST immediately emit a `RAL_Buffer_Get_TCPB_Available_Capacity_CommandEndEvent` with `command_status = SUCCESS` and the `capacity` parameter carrying the requested value.

If the RAL does not support `TightlyCoupledPacketBuffers`, the returned `capacity` MUST be 0.

## Events

**Status convention.** Status values follow the Execution Service: `SUCCESS` reports success and every failure is prefixed `ERROR_`, so that a caller can distinguish the two without enumerating the values.

### `RAL_Buffer_Create_CommandEndEvent`

**Soliciting command:** `RAL_Buffer_Create_Command`.

**Summary:** Reports the result of creating a `PacketBuffer`.

**Parameters:**

| Name    | Type    | Valid Range | Description      |
| ------- | ------- | ----------- | ---------------- |
| `command_status`| Enumeration    | `SUCCESS`, `ERROR_BUFFER_CREATE_TCPB_NO_SPACE_AVAILABLE`, or any RAL-specific error status. | Final execution status.   |
| `packet_buffer` | `PacketBuffer` | As defined in [§PacketBuffer](./ral-overview.md#packetbuffer)  | The created `PacketBuffer` instance, valid only if `command_status = SUCCESS`. |

**Description:** The RAL emits `RAL_Buffer_Create_CommandEndEvent` upon conclusion of the `RAL_Buffer_Create_Command`.

### `RAL_Buffer_Delete_CommandEndEvent`

**Soliciting command:** `RAL_Buffer_Delete_Command`.

**Summary:** Reports the result of deleting a `PacketBuffer`.

**Parameters:**

| Name     | Type        | Valid Range                                  | Description             |
| -------- | ----------- | -------------------------------------------- | ----------------------- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_BUFFER_NOT_READY`, or any RAL-specific error status. | Final execution status. |

**Description:** The RAL emits `RAL_Buffer_Delete_CommandEndEvent` upon conclusion of the `RAL_Buffer_Delete_Command`.

### `RAL_Buffer_Configure_Regions_CommandEndEvent`

**Soliciting command:** `RAL_Buffer_Configure_Regions_Command`.

**Summary:** Reports whether the requested region partition was accepted.

**Parameters:**

| Name     | Type    | Valid Range| Description           |
| -------- | ------- | ---------- | --------------------- |
| `command_status`   | Enumeration  | `SUCCESS`, `ERROR_BUFFER_NOT_READY`, `ERROR_UNSUPPORTED_PARTITION`, `ERROR_INVALID_BUFFER_RANGE`, or any RAL-specific error status. | Outcome of the partition request.  |

**Description:** Emitted on conclusion of the `RAL_Buffer_Configure_Regions_Command`.

### `RAL_Buffer_Ensure_Available_CommandEndEvent`

**Soliciting command:** `RAL_Buffer_Ensure_Available_Command`.

**Summary:** Reports the result of the `RAL_Buffer_Ensure_Available_Command`.

**Parameters:**

| Name      | Type  | Valid Range  | Description     |
| --------- | ----- | ------------ | --------------- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_REGION_NOT_CONFIGURED`, `ERROR_INVALID_BUFFER_RANGE`, `ERROR_DATA_NOT_READY`, or any RAL-specific error status. | Final execution status. |

**Description:** The RAL emits `RAL_Buffer_Ensure_Available_CommandEndEvent` upon conclusion of `RAL_Buffer_Ensure_Available_Command`.

### `RAL_Buffer_Mark_Touched_CommandEndEvent`

**Soliciting command:** `RAL_Buffer_Mark_Touched_Command`.

**Summary:** Reports the result of the `RAL_Buffer_Mark_Touched_Command`.

**Parameters:**

| Name     | Type        | Valid Range  | Description             |
| -------- | ----------- | ------------ | ----------------------- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_REGION_NOT_CONFIGURED`, `ERROR_INVALID_BUFFER_RANGE`, `ERROR_DATA_NOT_READY`, or any RAL-specific error status. | Final execution status. |

**Description:** The RAL emits `RAL_Buffer_Mark_Touched_CommandEndEvent` upon conclusion of `RAL_Buffer_Mark_Touched_Command`.

### `RAL_Buffer_Get_TCPB_Available_Capacity_CommandEndEvent`

**Soliciting command:** `RAL_Buffer_Get_TCPB_Available_Capacity_Command`.

**Summary:** Reports the result of the `RAL_Buffer_Get_TCPB_Available_Capacity_Command`.

**Parameters:**

| Name       | Type             | Valid Range | Description     |
| ---------- | -----------------| ----------- | --------------- |
| `command_status`   | Enumeration      | `SUCCESS`   | Final execution status. |
| `capacity` | Unsigned Integer | -           | The queried capacity that is currently available for allocating TCPBs. |

**Description:** The RAL emits `RAL_Buffer_Get_TCPB_Available_Capacity_CommandEndEvent` upon conclusion of `RAL_Buffer_Get_TCPB_Available_Capacity_Command`.
