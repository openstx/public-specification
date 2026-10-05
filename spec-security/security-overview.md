# Security Overview

This document gives an overview of frame security in OpenSTX. It explains what is protected, how the work is split between the RAL, Core and the upper layer, and which risks remain. The rules themselves are in the RAL and Core documents it links to.


## What Is Protected

OpenSTX protects each frame on its own, with AES-CCM and a key that all nodes of a network share. The key is 16 octets long (AES-128), or 32 octets (AES-256) on a network whose nodes all support it. In a protected frame:

- the Core header is authenticated but sent in clear, so that a receiver can read it before it has checked the frame,
- the payload is encrypted and authenticated, and
- a message integrity code (MIC) of 4, 8 or 16 octets follows the payload.

A receiver drops a frame whose MIC does not match. Only a node that has the key can produce a frame that passes the check, and only such a node can read the payload.


## Security Responsibilities across Layers

- **The RAL** holds the cipher and the key. It encrypts and authenticates the protected regions before sending them, and checks and decrypts them after receiving them, according to the partition of its `PacketBuffer`. See the [RAL Security Service](../spec-ral-services/ral-security-service.md), the [Buffer Manager](../spec-ral-services/ral-buffer-manager-service.md) and the [Execution Service](../spec-ral-services/ral-execution-service.md).
- **Core** applies the security settings the upper layer writes, installs the key, draws the IV seed, builds the nonce of every frame and checks the header of every received frame. See [Security](../spec-core-services/core-overview.md#security) in the Core Overview and the [Core Execution Service](../spec-core-services/core-execution-service.md).
- **The upper layer** writes the key and chooses the lengths of the key and the MIC. It decides which optional header fields each frame carries, where Core offers the choice, supplies the values a receiver cannot read from a frame, and takes care of joining and key distribution.


## Optional Frame Header Fields

The nonce of a frame is built from the IV seed, the short source address, the TFC and the TUC. A frame may carry all or some of these values. A receiver that holds a TFC uses its own timeframe and slot values; otherwise it takes the time values from the frame. It takes the short source address and IV seed from the frame, or from values supplied by the upper layer when those fields are absent.

- **Self-contained frames** carry all header fields needed to build the nonce. This costs 13 octets of optional header fields per frame. A joining node that has the key but does not yet hold all information required to build the nonce (e.g., the TFC) can extract all such information from a self-contained frame and synchronize to the rest of the network. In contrast, a receiver that already holds a TFC requires the received frame's time values to match its own. 
- **Compact frames** do not carry all header fields and instead leave out what the receivers are expected to know. Missing information is supplied by the upper layer, for example by making use of a protocol that sends regular self-contained synchronization frames carrying all nonce components, or by providing missing information out of band. Whenever a new IV seed is drawn, receivers that fill it in themselves must first learn it, for example from a frame that carries it. A receiver that holds a TFC uses its own TFC and TUC, so a frame replayed in another slot fails the acceptance check (subject to the residual risks described below). 

A protocol can mix self-contained and compact frames as needed, for example by sending compact data frames interleaved with self-contained synchronization frames once in a while.


## Possible Conclusions from Successful Reception of Secured Frames

When a protected frame passes the check, the receiver can rely on the following, within the limits of the residual risks below:
- The frame was protected by a node holding the key. This does not identify the node: the source address is not proof of identity. The frame may also be a replay of an earlier protected frame.
- No part of the header or payload was modified after protection by a key holder.
- The payload is readable only by key holders. The header is integrity-protected but not confidential.
- If the receiver holds a TFC, the frame was sent for the slot and timeframe in which it arrived. Consequently, a single frame can advance the receiver's TREF by at most its receive window. Restarts, TFC wrap-around, keys shared between networks and arrival times below describe the exceptions.


## Residual Risks

- **Guessing the MIC.** A forged frame passes the check with a probability of 2^-32 per attempt with a 4-octet MIC, 2^-64 with 8 octets and 2^-128 with 16, and nothing counts failed attempts. One forged broadcast frame is an attempt at every node that listens in that slot. A network that expects many attempts can choose a longer MIC through `coreMicSize`.
- **Replay in the same slot.** A recorded frame sent again in its own slot and timeframe passes the check, because it cannot be told apart from a relay of the same frame.
- **Joining nodes.** A node that holds no TFC takes the TFC and TUC from the frame, so a recorded frame can mislead it. If it then relays that frame, it builds the nonce from the original sender's IV seed and source address and the old TFC; where the protocol relays in a slot in which that sender protected a different payload under that TFC, the nonce repeats. Protecting the join is left to a later onboarding mechanism.
- **Restarts.** When the node that originates the TFC restarts and counts from a lower value, frames recorded before the restart can carry a TFC and TUC that the network uses again. Replayed in their own slot, they pass the check. An originator that starts above every TFC it used before avoids this; that needs state that survives a restart, such as a stored TFC, or else a new key.
- **TFC wrap-around.** The TFC has 2^32 values. Once the TFC comes back to a value it has taken before, a frame recorded then passes the check again in its own slot. With one step per timeframe this happens after about 50 days with timeframes of 1 ms and about 500 days with timeframes of 10 ms.
- **One key per network.** A node whose key is captured can read and forge every frame protected with that key, until the key is replaced or removed. This version defines no key distribution.
- **Keys shared between networks.** The nonce does not identify the network. Networks that use the same key can accept each other's frames when their TFC and TUC agree. Each network needs its own key.
- **Readable headers.** Addresses, time fields and the IV seed are sent in clear, and the IV seed links the frames a node sends between two restarts.
- **Equal IV seeds.** Two random IV seeds a node draws are equal with a probability of 2^-32. If that happens after its TFC went back, the node can repeat a nonce. The risk grows with every return of the TFC, to about n^2/2^33 after n draws, which is why the TFC should not come back to earlier values.
- **Weak random numbers.** After a restart, the TFC counts through values it has used before, and only a new IV seed keeps the node's nonces from repeating. A random number generator that returns a value it already returned before the restart, for example because it has not yet gathered enough entropy, makes the node repeat nonces.
- **Broken rules.** If the upper layer breaks one of its rules, for example by giving two nodes the same short address or by letting a relay change a payload, two different payloads can be protected with the same nonce, which reveals information about both.
- **Relays.** Core builds the nonce of a relay from the source address and IV seed that the upper layer passes, without checking where they come from. If a relay uses them under a TFC and TUC at which the original sender, or another relay, protects a different payload, the nonce repeats. This cannot happen where every slot has one sender. Where several nodes send in the same slot, as in concurrent-transmission floods, the protocol has to relay a frame only within the flood it belongs to.
- **TFC changes after scheduling.** A transmission scheduled before `CORE_Tref_Sync_Command` changes the TFC still carries the old one, and receivers may refuse it.
- **Decrypted octets before the check.** Octets that the RAL makes readable during a reception, through `RAL_Rx_Data_Available_CommandEvent`, have not been checked yet. Core reads only the header this way, which is sent in clear. Acting on decrypted payload octets before the reception has ended successfully would let an attacker who sends a forged frame in a slot learn the payload of the genuine frame of that slot.
- **Plaintext in radio memory.** A radio that encrypts or decrypts in its own buffer may keep plaintext there, in the radio buffer of a `TightlyCoupledPacketBuffer`, until the buffer is reused, even after the key has changed.
- **Arrival times.** The check covers the octets of a frame, not the time at which it arrives. An attacker who holds a frame back and sends it again later in the receive window, or who sends its predictable start early and the rest as it arrives, shifts the TREF of a receiver that synchronizes from it. Each such frame moves TREF by no more than the receive window, but the next window is placed around the moved TREF, so a sequence of such frames can move it without limit. A node that drifts this way transmits out of step with its neighbours and disturbs the concurrent transmissions around it. Limiting how far one synchronization may move TREF, for example to the drift expected since the last one, is left to the upper layer.
- **Telling attacks from lost synchronization.** Core cannot tell a node that has lost synchronization from one under attack: both see frames that fail the check. An upper layer that synchronizes again after failed receptions can be made to do so by anyone who sends junk frames. If it releases its TFC to do so, the node becomes a joining node that a recorded frame can set to an old TFC and TREF. When Core supplies its own seed, a return to earlier time values makes it draw a new IV seed that receivers of its frames without Aux Sec do not know yet.
- **Energy.** A frame that looks valid until its MIC is checked keeps a receiver on for the whole reception and costs a check. A joining node, which listens longest, is affected most.


## Terms

| Term | Meaning |
| ---- | ------- |
| IV seed | A 4-octet value that keeps the nonces of a node unique across restarts (see IV seed in the Core Overview). |
| MIC | The message integrity code AES-CCM computes over the protected part of a frame, 4, 8 or 16 octets long. |
| Nonce | The 13-octet value AES-CCM uses together with the key. It must never repeat under one key. It is built from the IV seed, the short source address, the TFC and the TUC, and it is never sent. |
| Protected region | `BUFFER_REGION_AUTHENTICATED` or `BUFFER_REGION_ENCRYPTED` of a `PacketBuffer` partition. |
| Installed key | The key the RAL uses, from a successful `RAL_Crypto_ConfigureKey_Command` until it is removed, replaced or withdrawn. |

<!--TODO: Move to glossary-->
