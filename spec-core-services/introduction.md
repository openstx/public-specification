# Introduction

The Core layer sits above the Radio Abstraction Layer (RAL) in the OpenSTX architecture. It introduces the common time base, the time reference (TREF), the time unit (TU) abstraction, and the scheduling of single TX and RX actions and skips inside timeframes. The Core lets the layer above it compose synchronous-transmission schedules without dealing with radio time or hardware-specific details.

This document provides an overview of the internal model of the Core, and the specification of Core Services, provided by the Core to the upper layer.


## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

This document uses terminology from the [OpenSTX Glossary](../glossary.md).


## Specification Documents

- [Core Overview](./core-overview.md)
- [Synchronization Service](./core-synchronization-service.md)
- [Execution Service](./core-execution-service.md)
- [Configuration Service](./core-configuration-service.md)
- [Packet Manager](./core-packet-manager-service.md)
