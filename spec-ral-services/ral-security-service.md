# RAL Security Service


<!------------------------------------------------------------------------>


## Introduction

The RAL Security Service manages the cipher the RAL uses to protect frames. It installs and removes the network key, and draws the random IV seed that Core puts into every nonce.

The cipher itself runs as part of each radio operation. When a `PacketBuffer` has a partition with protected regions, the [Execution Service](./ral-execution-service.md) encrypts and authenticates its PSDU before transmission, and checks and decrypts it after reception.

Core issues the commands of this service. The upper layer does not use them directly; it configures security through Core (see [§Security](../spec-core-services/core-overview.md#security) in the Core Overview).

The normative behavior is defined in sections [The Cipher](#the-cipher), [Commands](#commands) and [Events](#events).


### Related Documents

| Document | Description |
| -------- | ----------- |
| [Introduction](./introduction.md) | Introduction to the RAL Services |
| [Buffer Manager](./ral-buffer-manager-service.md) | Declares the partition that says which octets are protected |
| [Execution Service](./ral-execution-service.md) | Encrypts, authenticates and checks the PSDU of each radio operation |
| [Configuration Service](./ral-config-service.md) | Defines `nonceSize` and `micSize`, which tell whether the RAL has a cipher |
| [Security Overview](../spec-security/security-overview.md) | Explains how the RAL, Core and the upper layer share the work of frame security |


<!------------------------------------------------------------------------>


## The Cipher

A RAL that has a cipher MUST protect frames with AES-CCM as specified in NIST SP 800-38C, with the formatting and counter generation functions of its Appendix A, using the installed key, a nonce of `nonceSize` octets and a MIC of `micSize` octets. The key is 16 octets long for AES-128 or 32 octets long for AES-256; a RAL that has a cipher MUST support AES-128 and MAY support AES-256. The octets of `BUFFER_REGION_AUTHENTICATED` are the associated data: they are authenticated, but sent in clear. The octets of `BUFFER_REGION_ENCRYPTED` are encrypted and authenticated. The octets of `BUFFER_REGION_UNSECURED` are not protected.

The RAL sends the MIC after the frame content and before the FCS, where present. The MIC is not counted in `psdu_length`, and the RAL removes it from a received frame once it has checked it. The FCS lies outside the protected regions and is calculated over the frame after security processing on transmission and checked before security processing on reception. A RAL that has a cipher MUST support a `micSize` of 4, 8 and 16 octets.

A RAL that has no cipher reports `nonceSize` and `micSize` 0.

A key is installed from a successful [`RAL_Crypto_ConfigureKey_Command`](#ral_crypto_configurekey_command) until it is removed, replaced by another key, or withdrawn because a later install failed. Whenever the RAL withdraws, replaces or removes a key, it MUST set to zero every copy of that key it holds, before it emits the event that reports the change.

<!-- TODO: Key slots. A later revision adds versions of these commands that name a key slot; the commands below then act on slot 0. -->


<!------------------------------------------------------------------------>


## Commands

The RAL MUST emit exactly one `*_CommandEndEvent` for each command of this service, immediately, before the command returns. On a RAL that has no cipher, every command of this service MUST end with `command_status = ERROR_UNSUPPORTED`, whatever the RAL state, closing the command lifecycle.


### `RAL_Crypto_ConfigureKey_Command`

**Summary:** Installs the network key.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `key` | Octet String | 16 or 32 octets | The network key: 16 octets for AES-128, 32 octets for AES-256. |

**Description:**

`RAL_Crypto_ConfigureKey_Command` requests the RAL to install the network key, replacing any key installed before.

If the RAL state is `PENDING` or `RUNNING`, the RAL MUST keep the installed key as it is and MUST emit `RAL_Crypto_ConfigureKey_CommandEndEvent` with `command_status = ERROR_NOT_ALLOWED`, closing the command lifecycle.

If the RAL does not support a key of the given length, it MUST keep the installed key as it is and MUST emit `RAL_Crypto_ConfigureKey_CommandEndEvent` with `command_status = ERROR_UNSUPPORTED_KEY_LENGTH`, closing the command lifecycle.

If the key cannot be installed, the RAL MUST withdraw any installed key and MUST emit `RAL_Crypto_ConfigureKey_CommandEndEvent` with `command_status = ERROR_KEY_INSTALL_FAILED`, closing the command lifecycle.

Otherwise, the RAL MUST install the key, keep its own copy of it, and emit `RAL_Crypto_ConfigureKey_CommandEndEvent` with `command_status = SUCCESS`, closing the command lifecycle.


### `RAL_Crypto_RemoveKey_Command`

**Summary:** Removes the network key.

**Parameters:** None.

**Description:**

`RAL_Crypto_RemoveKey_Command` requests the RAL to remove the installed key.

If the RAL state is `PENDING` or `RUNNING`, the RAL MUST keep the installed key as it is and MUST emit `RAL_Crypto_RemoveKey_CommandEndEvent` with `command_status = ERROR_NOT_ALLOWED`, closing the command lifecycle.

Otherwise, the RAL MUST remove the installed key, if there is one, and emit `RAL_Crypto_RemoveKey_CommandEndEvent` with `command_status = SUCCESS`, closing the command lifecycle. Without a key, the RAL rejects every scheduling command whose partition has a non-empty `BUFFER_REGION_AUTHENTICATED` or `BUFFER_REGION_ENCRYPTED`, until a key is installed again.


### `RAL_Crypto_Iv_Seed_Command`

**Summary:** Draws a random IV seed.

**Parameters:** None.

**Description:**

`RAL_Crypto_Iv_Seed_Command` requests the RAL to draw a 4-octet IV seed from the platform's cryptographically secure random number generator, or from a source that never returns the same value twice under the installed key, such as a counter kept across restarts.

If the source is unavailable or fails, the RAL MUST NOT return a fixed or zero value instead, and MUST emit `RAL_Crypto_Iv_Seed_CommandEndEvent` with `command_status = ERROR_CSPRNG_UNAVAILABLE`, closing the command lifecycle.

Otherwise, the RAL MUST emit `RAL_Crypto_Iv_Seed_CommandEndEvent` with `command_status = SUCCESS` and the drawn value in `iv`, closing the command lifecycle. The RAL MUST NOT mask or fix any bit of a value drawn from the random number generator.


<!------------------------------------------------------------------------>


## Events

### `RAL_Crypto_ConfigureKey_CommandEndEvent`

**Soliciting command:** `RAL_Crypto_ConfigureKey_Command`

**Summary:** Reports whether the network key was installed.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_UNSUPPORTED`, `ERROR_NOT_ALLOWED`, `ERROR_UNSUPPORTED_KEY_LENGTH`, `ERROR_KEY_INSTALL_FAILED` | Outcome of the install. `SUCCESS` if and only if the key was installed. |

**Description:** The RAL emits `RAL_Crypto_ConfigureKey_CommandEndEvent` after processing a key install request, closing the command lifecycle.


### `RAL_Crypto_RemoveKey_CommandEndEvent`

**Soliciting command:** `RAL_Crypto_RemoveKey_Command`

**Summary:** Reports whether the network key was removed.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_UNSUPPORTED`, `ERROR_NOT_ALLOWED` | Outcome of the removal. `SUCCESS` if and only if no key is installed afterwards. |

**Description:** The RAL emits `RAL_Crypto_RemoveKey_CommandEndEvent` after processing a key removal request, closing the command lifecycle.


### `RAL_Crypto_Iv_Seed_CommandEndEvent`

**Soliciting command:** `RAL_Crypto_Iv_Seed_Command`

**Summary:** Reports whether an IV seed could be drawn.

**Parameters:**

| Name | Type | Valid Range | Description |
| ---- | ---- | ----------- | ----------- |
| `command_status` | Enumeration | `SUCCESS`, `ERROR_UNSUPPORTED`, `ERROR_CSPRNG_UNAVAILABLE` | Outcome of the draw. |
| `iv` | Octet String | 4 octets. Present only when `command_status = SUCCESS`. | The drawn IV seed. |

**Description:** The RAL emits `RAL_Crypto_Iv_Seed_CommandEndEvent` after processing a draw request, closing the command lifecycle.


<!------------------------------------------------------------------------>
