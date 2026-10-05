# RAL Configuration Service


<!------------------------------------------------------------------------>


## Introduction

The following subsection describes the primitives that allow to read and modify radio configuration attributes.
The normative behavior is defined in sections [Commands](#commands) and [Events](#events).


### Related Documents

| Document                                  | Description                                             |
| ----------------------------------------- | ------------------------------------------------------- |
| [Introduction](./introduction.md)         | Introduction to the RAL Services                        |


<!------------------------------------------------------------------------>


## Commands


<a name="ral-config-service-ral_conf_set_cmd"></a>

### `RAL_Conf_Set_Cmd`

**Summary:** Updates the requested radio configuration attribute to the supplied value.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `attribute` | Enumeration | Any supported configuration attribute (see [Configuration Attributes](#configuration-attributes)). | Identifies the configuration attribute to update. |
| `value` | Various | Values valid for the selected `attribute` | Provides the new value for the selected configuration attribute. |

**Event Pattern:** Single-Response (`RAL_Conf_Set_Cmd -> RAL_Conf_Set_CmdEndEvt`)

**Description:** 

`RAL_Conf_Set_Cmd` requests a configuration change for the indicated attribute. The RAL MUST validate the requested value against the constraints of the selected attribute before applying any state change.

The RAL MUST emit exactly one `RAL_Conf_Set_CmdEndEvt` for each `RAL_Conf_Set_Cmd` processed. 

If an action is `ACTIVE` or `PENDING`, the RAL MUST emit `RAL_Conf_Set_CmdEndEvt` immediately, with `command_status = NOT_ALLOWED`, and perform no other activity. 

At most one `RAL_Conf_Set_Cmd` may be active at any time. If a second `RAL_Conf_Set_Cmd` is issued before the first one has produced its terminal event, the RAL MUST emit `RAL_Conf_Set_CmdEndEvt` for the second `RAL_Conf_Set_Cmd` immediately, with `command_status = NOT_ALLOWED`, and perform no other activity.

If the indicated `attribute` is not supported, the RAL MUST emit `RAL_Conf_Set_CmdEndEvt` immediately, with `command_status = NOT_ALLOWED`, and perform no other activity. 

If validation fails, the RAL MUST generate `RAL_Conf_Set_CmdEndEvt` with `command_status = INVALID_VALUE` and MUST take no other action.


<a name="ral-config-service-ral_conf_get_cmd"></a>

### `RAL_Conf_Get_Cmd`

**Summary:** Reads the current value of the requested radio configuration attribute.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `attribute` | Enumeration | Any supported configuration attribute (see [Configuration Attributes](#configuration-attributes)). | Identifies the configuration attribute to read. |

**Event Pattern:** Single-Response (`RAL_Conf_Get_Cmd -> RAL_Conf_Get_CmdEndEvt`)

**Description:** 

`RAL_Conf_Get_Cmd` requests the current value of the selected configuration attribute (without modifying the radio state). 

The RAL MUST emit exactly one `RAL_Conf_Get_CmdEndEvt` for each `RAL_Conf_Get_Cmd` processed. 

If the RAL state (see [§RAL States](./ral-overview.md#ral-states)) is `PENDING` or `RUNNING`, the RAL MUST emit `RAL_Conf_Set_CmdEndEvt` immediately, with `status = NOT_ALLOWED`, and perform no other activity.

At most one `RAL_Conf_Get_Cmd` may be active at any time. If a second `RAL_Conf_Get_Cmd` is issued before the first one has produced its terminal event, the RAL MUST emit `RAL_Conf_Get_CmdEndEvt` for the second `RAL_Conf_Get_Cmd` immediately, with `command_status = NOT_ALLOWED`, and perform no other activity.

If the indicated `attribute` is not supported, the RAL MUST emit `RAL_Conf_Get_CmdEndEvt` immediately, with `command_status = NOT_ALLOWED`, and perform no other activity. 

If validation fails, the RAL MUST generate `RAL_Conf_Get_CmdEndEvt` with `command_status = INVALID_VALUE` and MUST take no other action.


<!------------------------------------------------------------------------>


## Events


### `RAL_Conf_Set_CmdEndEvt`

**Soliciting command:** `RAL_Conf_Set_Cmd`

**Summary:** Reports the final outcome of a configuration update request.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `command_status` | Enumeration | `SUCCESS`, `NOT_ALLOWED`, `INVALID_VALUE` | Outcome of the configuration update request. |
| `attribute` | Enumeration | Any supported configuration attribute (see [Configuration Attributes](#configuration-attributes)). | Echoes the attribute targeted by the command. |

**Description:** The RAL emits `RAL_Conf_Set_CmdEndEvt` after validating and either applying or rejecting the requested write on a configuration attribute.


### `RAL_Conf_Get_CmdEndEvt`

**Soliciting command:** `RAL_Conf_Get_Cmd`

**Summary:** Reports the final outcome of a configuration read request.

**Parameters:**

| Name | Type | Valid Range | Description |
| --- | --- | --- | --- |
| `command_status` | Enumeration | `SUCCESS`, `NOT_ALLOWED` | Outcome of the configuration read request. |
| `attribute` | Enumeration | Any supported configuration attribute (see [Configuration Attributes](#configuration-attributes)). | Echoes the attribute targeted by the command. |
| `value` | Various | Present only when `command_status` is `SUCCESS`. | Current value of the requested configuration attribute. |

**Description:** The RAL emits `RAL_Conf_Get_CmdEndEvt` after reading or rejecting the requested read on a configuration attribute. This event provides the retrieved value when the read succeeds.


<!------------------------------------------------------------------------>


## Configuration Attributes

Generic and PHY-specific attributes are presented below.
Unless otherwise noted, the following attributes match the PHY PIB attributes of the same name in IEEE 802.15.4-2024 (12.3.2, 12.3.7, 12.3.12, from page 600 onwards).
Different semantics, type or value ranges may be specified.
<!-- Unless otherwise noted, IEEE mandatory values for these attributes MUST also be supported by the OpenSTX RAL. -->
Note that, among IEEE PHY PIB attributes, only those listed here are to be considered OpenSTX radio configuration attributes.


### Notation

Attributes that are introduced by OpenSTX, and are not among IEEE PHY PIB attributes, are denoted with a "+" mark.  
Read-only attributes are denoted with a "†" mark.


### Generic Configuration Attributes

* _phyBroadcastTxPower_
* _phyCurrentChannelInfo_
* _phyMaxPacketSize_
* _phyMaxTxPower_ †
* _phyRxRmarkerOffset_
* _phyTxPower_
* _phyTxRmarkerOffset_
* _phyCurrentCode_

* _bufferCapacityTCPB_ +† — total number of octets available for the allocation of `TightlyCopledPacketBuffers` (TCPBs), as defined in [§Packet Buffer](./ral-overview.md#packetbuffer) and [§Buffer Management](./ral-buffer-manager-service.md). MUST be 0 if the RAL does not support TCPBs.
* _bufferSpaceMargin_ +† — total number of octets that constitute the RAL-specific headroom and tailroom within a [§Packet Buffer](./ral-overview.md#packetbuffer). The RAL fixes it, since only the RAL knows the framing its PHY adds and the MIC its cipher inserts. On a RAL that has a cipher, it includes 16 octets for the MIC, the largest `micSize`. The FCS is already counted in `psdu_length`.
* _fcsLength_ +† — number of octets of the frame check sequence (FCS) that ends the PSDU on the configured PHY, and 0 where the PSDU carries none (see [§PSDULength](./ral-overview.md#psdulength)).
* _micSize_ + — number of octets of the MIC the RAL's cipher computes: 4, 8 or 16 on a RAL that has a cipher, and 0 on a RAL that has none (see [§RAL Security Service](./ral-security-service.md#the-cipher)). On a RAL that has a cipher, it reads 4 until written. A write of any other value, and every write on a RAL that has no cipher, ends with `command_status = INVALID_VALUE`.
* _nonceSize_ +† — number of octets that constitute the nonce used in cryptographic operations: 13 on a RAL that has a cipher, and 0 on a RAL that has none (see [§RAL Security Service](./ral-security-service.md#the-cipher)).
* _radioTimeFrequency_ +† — frequency of the radio time tick, expressed in hertz.
<!--NOTE: Core reads this attribute, together with the binding-provided SYSTEM_TIME_FREQUENCY constant, to convert radio-time durations into system-time durations.-->

### UWB Configuration Attributes

* _phyHrpUwbCcConstraintLength_
* _phyHrpUwbCurrentPulseShape_
* _phyHrpUwbDataRatesSupported_ †
* _phyHrpUwbLcpDelay2_
* _phyHrpUwbLcpDelay3_
* _phyHrpUwbLcpDelay4_
* _phyHrpUwbLcpWeight1_
* _phyHrpUwbLcpWeight2_
* _phyHrpUwbLcpWeight3_
* _phyHrpUwbLcpWeight4_
* _phyHrpUwbPhrA0_
* _phyHrpUwbPhrA1_
* _phyHrpUwbPhrDataRate_
  * DRMDR is not a valid value (due to the different approach in handling data and TX/RX in OpenSTX)
* _phyHrpUwbPsduSize_
* _phyHrpUwbPsr_
* _phyHrpUwbSfdSelector_

<!-- * _phyHrpUwbScanBinsPerChannel_  TODO: INCLUDE IF/WHEN SCANNING IS INTRODUCED -->


<!------------------------------------------------------------------------>
