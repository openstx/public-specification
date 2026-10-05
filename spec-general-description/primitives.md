# Primitives and Layer Interaction

Each layer exposes services as sets of primitives.
A primitive is an implementation-independent description of a functionality offered by one layer to an upper layer.
A primitive defines what information is exchanged and what behavior is expected at the service boundary; it does not prescribe the programming model, calling convention, callback structure, or any other implementation mechanism.

At a service boundary, the interaction follows a command/event paradigm:
- A command is issued by an upper layer to request an operation from a lower layer.
- An event is emitted by a lower layer to report a command result or a spontaneous indication to an upper layer.

The standard exploits the following nomenclature for primitive roles to express the lifecycle of a command with its related events:
- `<LAYER_NAME>_<COMMAND_NAME>_Command`: a command that requests an operation.
- `<LAYER_NAME>_CommandStatusEvent`: a layer-wide event that reports whether a command was accepted for processing. This event carries a `command_id` identifying the command, a `command_status`, and, when applicable, a `command_handle` identifying the accepted command instance. `command_status = ACCEPTED` indicates that command processing has started.
Any other `command_status` value expresses the rejection reason. If a command is rejected in `<LAYER_NAME>_CommandStatusEvent`, no other event follows for that command invocation.
- `<LAYER_NAME>_<EVENT_NAME>_CommandEvent`: a command-specific intermediate event emitted when the command produces intermediate results.
- `<LAYER_NAME>_<EVENT_NAME>_CommandEndEvent`: an event that indicates the termination of the associated command and reports the command's final outcome. No more than one `*_CommandEndEvent` can be emitted for a given command.
- `<LAYER_NAME>_<EVENT_NAME>_Event`: an unsolicited event not bound to command lifecycle.

Each command defines which events it solicits. A command may be associated to only a subset of these event types.
For example, the command `RAL_Conf_Set_Command` causes the RAL to emit `RAL_Conf_Set_CommandEndEvent` and no other event.





