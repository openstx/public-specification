# Introduction

The Radio Abstraction Layer (RAL) forms the foundation of the OpenSTX architecture, providing a common interface for radio and hardware functionality. By abstracting away hardware-specific details, it allows higher layers to remain portable across different radio platforms.

This document provides an overview of the internal model of the RAL, and the specification of RAL Services, provided by the RAL to the upper layer.


## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

Additionally, this document uses terminology from the [OpenSTX Glossary](../glossary.md) in the table [RAL Services Terminology](#table-ral-terminology)

<a name="table-ral-terminology"></a>

<b>Table</b>: RAL Services Terminology. 

| Term or Abbreviation |Definition                                                                                          
| -------------------- | ------------------------------------------------ |
| PPDU                 | PHY Protocol Data Unit (as in IEEE 802.15.4, often called “frame”) |
| PSDU                 | PHY Service Data Unit (as in IEEE 802.15.4, often called “PHY payload”) |
| PPDU Timing Anchor (PTA) | Reference point inside a PPDU used for scheduling transmissions and receptions, and for time synchronization. The PPDU Timing Anchor is defined as the beginning of the PSDU on air (independent of the underlying physical layer). <br>All corresponding timestamps refer to this anchor, independent of PHY-specific markers like SFD, RMARKER, and hardware-specific events or interrupt signals. <br>This enables generic, PHY-independent RAL interfaces, and ensures interoperability between different hardware platforms.|                                                                                                                                               


## Specification Documents

- [Overview](./ral-overview.md)
- [Execution Service](./ral-execution-service.md)
- [Time Service](./ral-time-service.md)
- [Configuration Service](./ral-config-service.md)
- [Security Service](./ral-security-service.md)

