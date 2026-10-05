# Core Configuration Service


<!------------------------------------------------------------------------>


## Introduction

The Configuration Service reads and modifies the Core configuration attributes. It uses the same attribute model as the [RAL Configuration Service](../spec-ral-services/ral-config-service.md): a single attribute namespace accessed through generic Set and Get commands. The layer above Core never uses attribute-specific commands. The attributes are listed in [Core Configuration Attributes](#core-configuration-attributes).

The normative behavior is defined in sections [Commands](#commands) and [Events](#events).


### Related Documents

| Document | Description |
| -------- | ----------- |
| [Introduction](./introduction.md) | Introduction to the Core Services |
| [Core Overview](./core-overview.md) | Defines the time bases (terminology in the [OpenSTX Glossary](../glossary.md)). |
| [RAL Configuration Service](../spec-ral-services/ral-config-service.md) | The model this service mirrors |
| [RAL Security Service](../spec-ral-services/ral-security-service.md) | Installs and removes the key that `coreSecurityKey` writes |


<!------------------------------------------------------------------------>


## Commands


### `CORE_Config_Set_Command`

**Summary:** Updates the requested Core configuration attribute to the supplied value.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `attribute` | Enumeration | Any supported Core configuration attribute (see [Core Configuration Attributes](#core-configuration-attributes)). | Identifies the configuration attribute to update. |
| `value` | Various | Values valid for the selected `attribute`. | Provides the new value for the selected configuration attribute. |

**Description:**

`CORE_Config_Set_Command` requests a configuration change for the indicated attribute. Core MUST validate the requested value against the constraints of the selected attribute before applying any state change.

Core MUST emit exactly one `CORE_Config_Set_CommandEndEvent` for each `CORE_Config_Set_Command` processed.

Configuration attributes can only be modified while no future slots are scheduled.
If the schedule contains an upcoming action slot, Core MUST emit `CORE_Config_Set_CommandEndEvent` with `command_status = ERROR_NOT_ALLOWED`, closing the command lifecycle.
<!--CHECK: Does this work or do we need to make conf set schedulable as well? Forcing an empty future schedule is meant to keep the scheduler implementation simple in the initial prototypes (if config changes are schedulable, the implementation needs to keep views of the configuration instead of just checking the actual configuration) -->

If the indicated `attribute` is not supported, Core MUST emit `CORE_Config_Set_CommandEndEvent` with `command_status = ERROR_UNSUPPORTED`, closing the command lifecycle.

If the indicated `attribute` is read-only, Core MUST emit `CORE_Config_Set_CommandEndEvent` with `command_status = ERROR_NOT_ALLOWED`, closing the command lifecycle.

If validation fails, Core MUST emit `CORE_Config_Set_CommandEndEvent` with `command_status = ERROR_INVALID_VALUE`, closing the command lifecycle.

The security attributes take effect through the [RAL Security Service](../spec-ral-services/ral-security-service.md):

- A value written to `coreSecurityKey` is installed in the RAL. Core issues `RAL_Crypto_ConfigureKey_Command`. If this fails, Core MUST emit `CORE_Config_Set_CommandEndEvent`, closing the command lifecycle, with `command_status = ERROR_UNSUPPORTED` if the RAL has no cipher, `ERROR_NOT_ALLOWED` if the RAL is busy, and `ERROR_INVALID_VALUE` otherwise, including for a key length the RAL does not support. Core MUST NOT keep a copy of the key once the write has concluded.
- A write of `TRUE` to `coreSecurityEnabled` needs a cipher and an installed key. If `coreSecurityEnabled` is already `TRUE`, Core MUST leave it and the RAL as they are and emit `CORE_Config_Set_CommandEndEvent` with `command_status = ERROR_NOT_ALLOWED`, closing the command lifecycle. If the RAL reports `nonceSize` 0, Core MUST emit `CORE_Config_Set_CommandEndEvent` with `command_status = ERROR_UNSUPPORTED`, closing the command lifecycle. If no key has been written since security was last turned off, or the last one was not installed, Core MUST emit `CORE_Config_Set_CommandEndEvent` with `command_status = ERROR_NOT_ALLOWED`, closing the command lifecycle. Core also writes `coreMicSize` to the RAL's `micSize`. If this fails, Core MUST keep `coreSecurityEnabled` at `FALSE` and emit `CORE_Config_Set_CommandEndEvent`, closing the command lifecycle, with `command_status = ERROR_NOT_ALLOWED` if the RAL is busy, and `ERROR_INVALID_VALUE` otherwise.
- A write to `coreMicSize` while `coreSecurityEnabled` is `TRUE` changes nothing: Core MUST emit `CORE_Config_Set_CommandEndEvent` with `command_status = ERROR_NOT_ALLOWED`, closing the command lifecycle.
- A write of `FALSE` to `coreSecurityEnabled` removes the key: Core issues `RAL_Crypto_RemoveKey_Command`, and security can only be turned on again after a new key is written. On a RAL that has no cipher there is no key to remove. If the RAL refuses the removal because it is busy, Core MUST keep `coreSecurityEnabled` unchanged and emit `CORE_Config_Set_CommandEndEvent` with `command_status = ERROR_NOT_ALLOWED`, closing the command lifecycle.


### `CORE_Config_Get_Command`

**Summary:** Reads the current value of the requested Core configuration attribute.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `attribute` | Enumeration | Any supported Core configuration attribute (see [Core Configuration Attributes](#core-configuration-attributes)). | Identifies the configuration attribute to read. |

**Description:**

`CORE_Config_Get_Command` requests the current value of the selected configuration attribute, without modifying Core state.

Core MUST emit exactly one `CORE_Config_Get_CommandEndEvent` for each `CORE_Config_Get_Command` processed.

If the indicated `attribute` is not supported, Core MUST emit `CORE_Config_Get_CommandEndEvent` with `command_status = ERROR_UNSUPPORTED`, closing the command lifecycle.

If the indicated `attribute` is write-only, Core MUST emit `CORE_Config_Get_CommandEndEvent` with `command_status = ERROR_NOT_ALLOWED`, closing the command lifecycle.


<!------------------------------------------------------------------------>


## Events


### `CORE_Config_Set_CommandEndEvent`

**Soliciting command:** `CORE_Config_Set_Command`

**Summary:** Reports the final outcome of a configuration update request.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_NOT_ALLOWED`, `ERROR_UNSUPPORTED`, `ERROR_INVALID_VALUE` | Outcome of the configuration update request. |
| `attribute` | Enumeration | Any supported Core configuration attribute. | Echoes the attribute targeted by the command. |

**Description:** Core emits `CORE_Config_Set_CommandEndEvent` after validating and either applying or rejecting the requested write on a configuration attribute. This event closes the command lifecycle.


### `CORE_Config_Get_CommandEndEvent`

**Soliciting command:** `CORE_Config_Get_Command`

**Summary:** Reports the final outcome of a configuration read request.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_UNSUPPORTED`, `ERROR_NOT_ALLOWED` | Outcome of the configuration read request. |
| `attribute` | Enumeration | Any supported Core configuration attribute. | Echoes the attribute targeted by the command. |
| `value` | Various | Present only when `command_status = SUCCESS`. | Current value of the requested configuration attribute. |

**Description:** Core emits `CORE_Config_Get_CommandEndEvent` after reading or rejecting the requested read on a configuration attribute. This event closes the command lifecycle.


<!------------------------------------------------------------------------>


## Core Configuration Attributes

### Notation

Read-only attributes are denoted with a "†" mark.

### Generic Configuration Attributes

* _coreTuSize_. Duration of one TU expressed in radio time ticks. Governs the TU to radio time conversion.
* _coreTxOffset_. Distance in TU between the start of a transmit slot and the PTA of the transmitted PPDU.
* _coreShortAddress_. The short address the local node uses in the network.
* _coreExtendedAddress_. The extended address assigned to the local node.
* _corePanId_. Local PAN ID, used for the `Src PAN` field of transmitted frames.
* _coreUpcomingScheduleCapacity_ †. Maximum number of schedule entries that have not yet run to completion (timeframe and pending slots). Constant, read-only.
* _coreExpiredScheduleCapacity_ †. Maximum number of completed slots Core retains in the schedule. Constant, read-only. MUST be at least 1.
* _coreSecurityEnabled_. Whether Core protects the frames it transmits and accepts only protected frames (see [Security](./core-overview.md#security)). `FALSE` until written.
* _coreSecurityKey_. The key of the network: 16 octets for AES-128, or 32 octets for AES-256 where the RAL supports it. Write-only.
* _coreMicSize_. Number of octets of the MIC in the frames Core protects: 4, 8 or 16. 4 until written. Nodes that exchange protected frames need the same value.
