# OpenSTX Public Specification

This repository holds the public draft of the **OpenSTX specification**, produced by the OpenSTX Radio Abstraction Layer (RAL) working group.

OpenSTX defines a portable, hardware-independent interface for low-power wireless systems that use **synchronous transmissions (STX)**, where nodes coordinate their radio activity against a common time base. The specification describes what each layer of the stack offers to the layer above it, so that protocols and applications can run unchanged across different radios and chips.

> **Draft status:** This is a working draft and is subject to change. Interfaces, names and behavior may be revised as the working group incorporates feedback.

## Architecture at a glance

```
+-------------------------------+
| Protocol Layer                |
+-------------------------------+
| Core Layer                    |   common time base, timeframes, slot scheduling,
|                               |   synchronization, OpenSTX header
+-------------------------------+
| Radio Abstraction Layer (RAL) |   radio operations, timestamping, hardware abstraction
+-------------------------------+
| PHY / RF Hardware             |
+-------------------------------+
```

Each layer exposes its functionality as a set of implementation-independent **primitives** (commands and events). The specification defines *what* each primitive does, not *how* it is implemented.

## What's in this repository

| Section | Contents |
| :--- | :--- |
| [General Description](spec-general-description/primitives.md) | Cross-cutting concepts: primitives, the command/event model and naming conventions. |
| [RAL Services](spec-ral-services/introduction.md) | The Radio Abstraction Layer: [overview](spec-ral-services/ral-overview.md), [execution](spec-ral-services/ral-execution-service.md), [time](spec-ral-services/ral-time-service.md), [configuration](spec-ral-services/ral-config-service.md), [buffer manager](spec-ral-services/ral-buffer-manager-service.md) and [security](spec-ral-services/ral-security-service.md) services. |
| [Core Services](spec-core-services/introduction.md) | The Core layer: [overview](spec-core-services/core-overview.md), [synchronization](spec-core-services/core-synchronization-service.md), [execution](spec-core-services/core-execution-service.md), [configuration](spec-core-services/core-configuration-service.md) and [packet manager](spec-core-services/core-packet-manager-service.md) services. |
| [Security](spec-security/security-overview.md) | How frames are protected (AES-CCM) and how security responsibilities are split across the RAL, Core and upper layers. |
| [Glossary](glossary.md) | Shared definitions of the terms and abbreviations used throughout the specification. |

Normative requirements use the key words MUST, SHOULD, MAY and so on, as defined in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Who it's for

Silicon vendors, module makers, protocol stack developers, integrators, researchers, and anyone interested in portable synchronous-transmission networking.

## Giving feedback

Feedback on the draft is welcome and is how the specification improves. To share it:

1. Search the [existing issues](https://github.com/openstx/public-specification/issues) to see whether your point has already been raised. If it has, add a comment or a 👍 there.
2. Otherwise, [open a new issue](https://github.com/openstx/public-specification/issues/new/choose) using the **Specification Feedback** template. It asks which section your feedback relates to, the type of feedback and, optionally, a suggested change.
3. Keep to one topic per issue, and quote or link the relevant section where you can.

Useful feedback includes technical errors, ambiguous wording, inconsistencies between sections, missing functionality, interoperability or implementability concerns, terminology, and editorial fixes.

**Please do not report security vulnerabilities in a public issue.** Design-level security comments are welcome as issues, but if you believe you have found an exploitable vulnerability, contact the OpenSTX Foundation privately via <https://openstxfoundation.org/contact>.

Do not include confidential or proprietary information in issues. By submitting feedback you agree that the working group may use it, including any suggested text, to improve the specification.

## Acknowledgements

This specification is developed by the OpenSTX RAL working group. Thanks to everyone who contributed to its content, including
[@m-rzepka](https://github.com/m-rzepka),
[@davide-vecchia](https://github.com/davide-vecchia),
[@schuschu](https://github.com/schuschu),
[@mjb04](https://github.com/mjb04),
[@mbaddeley](https://github.com/mbaddeley),
[@c-at-work](https://github.com/c-at-work) and
[@seanmcilroy29](https://github.com/seanmcilroy29).

## License

This specification is licensed under the [Apache License 2.0](LICENSE).
