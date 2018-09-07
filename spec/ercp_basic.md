# ERCP Basic

## Generalities

ERCP Basic is the first protocol from the ERCP suite. It aims to provide simple
and easy-to-implement yet reliable communication with embedded systems.

## Frame format

Frames **MUST** have the format described below.

* The frame reads top to bottom
* Each cell represents an octet
* Quoted values are ASCII characters

```
|---------|
|   'E'   |
|---------|
|   'R'   |
|---------|
|   'C'   |
|---------|
|   'P'   |
|---------|
|   'B'   |
|---------|
|   Type  |
|---------|
|  Length |
|---------|
|         |
    ...
|         |
|  Value  |
|         |
    ...
|         |
|---------|
|   CRC   |
|---------|
|  'EOT'  |
|---------|
```

### Fields

* **Type:** The command type. Several commands are
    [built-in](#built-in-commands) and the protocol is made to be extended by
    application commands handled by user callbacks.
* **Length:** The length of the *Value* field.
* **Value:** A field containing additional data. May be 0 to 255-bit long.
* **CRC:** CRC for the *Command*, *Length* and *Value* fiels combined. **Its
    computation is yet to be determined.**

### Implementation

A device receiving an incorrectly formatted frame **MUST** ignore it.

A device receiving a correctly formatted frame **MUST** compute its CRC. If the
CRC is valid, the device **MUST** call the command callback corresponding to the
*Type*. If the CRC is invalid or the command type is not implemented, the device
**MUST** send a [`Nack(reason)`](#nackreason).

A device receiving an incomplete frame... **To be determined**.

A device **CAN** set a maximum acceptable *Length* inferior to 255. In the case
a too long frame is received, the device **MUST** send a `Nack(TOO_LONG)`.

## Built-in commands

In the following subsections, commands are referred as functions taking zero or
more arguments. Each command comes with a description, a type, a length and a
value if appropriate. Then follows its expected behaviour.

All built-in commands **MUST** be implemented.

### `Ping()`

* **Description:** Tests connectivity.
* **Type:** `0x00`
* **Length:** 0

This command **CAN** be used to test connectivity.

A device receiving this command **MUST** reply with an `Ack()`.

### `Ack()`

* **Description:** Acknowledgement.
* **Type:** `0x01`
* **Length:** 0

This command **SHOULD** be sent to acknowledge a previously received command.

This command **CAN** be sent immediately after receiving the command or after
doing some processing.

This command **MUST NOT** be sent if not replying to a command.

A device receiving this command **MUST NOT** reply.

### `Nack(reason)`

* **Description:** Negative acknowledgement.
* **Type:** `0x02`
* **Length:** 1
* **Value:**
    * `reason` on 8 bits

The following values for `reason` **MUST** be implemented:

* `0x00`: `NO_REASON`
* `0x01`: `TOO_LONG`
* `0x02`: `INVALID_CRC`
* `0x03`: `UNKNOWN_COMMAND`
* `0x04`: `INVALID_ARGUMENTS`

Other values for `reason` **CAN** be implemented.

This command **MUST** be sent after a command with an invalid CRC has been
received. In this case, `reason` **MUST** be `INVALID_CRC`.

This command **MUST** be sent after an unknown command has ben received. In this
case, `reason` **MUST** be `UNKNOWN_COMMAND`.

This command **SHOULD** be sent to negatively acknowledge a previously received
command. A custom `reason` **CAN** be used.

This command **CAN** be sent immediately after receiving the command or after
doing some processing. In the second case, it **SHOULD** mean an problem has
occured while processing the command.

This command **MUST NOT** be sent if not replying to a command.

A device receiving this command **MUST NOT** reply.

### `Reset()`

* **Description:** Resets the device.
* **Type:** `0x03`
* **Length:** 0

This command **CAN** be used to reset the device.

A device receiving this command **CAN** permorm a reset.

If the expected behaviour is not implemented, a device receiving this command
**MUST** send a `Nack(UNKNOWN_COMMAND)`.

### `Protocol()`

* **Description:** Gets the protocol version.
* **Type:** `0x04`
* **Length:** 0

This command **CAN** be used to get the protocol version implemented by the
other end.

A device receiving this command **MUST** reply with a `Protocol_Reply(major,
minor, patch)`.

### `Protocol_Reply(major, minor, patch)`

* **Description:** Replies to `Protocol()`.
* **Type:** `0x05`
* **Length:** 3
* **Value:**
    * `major` on 8 bits: the major version
    * `minor` on 8 bits: the minor version
    * `patch` on 8 bits: the patch version

This command **MUST** be sent after a `Protocol()` command has been received.
Its value **MUST** state the version of the protocol implemented by the device.

This command **MUST NOT** be sent if not replying to `Protocol()`.

A device receiving this command without waiting for it **SHOULD** reply with a
`Nack(UNKNOWN_COMMAND)`.

### `Version(component)`

* **Description:** Gets the version of a software component.
* **Type:** `0x06`
* **Length:** 1
* **Value:**
    * `component` on 8 bits

The following values for `component` **SHOULD** be implemented:

* `0x00`: *Firmware Version*
* `0x01`: *ERCP Library Version*

Other values for `component` **CAN** be implemented. Custom values **MUST NOT**
replace the ones listed above.

This command **CAN** be used to get the version of a software component.

A device receiving this command **MUST** reply with a `Version_Reply(version)`.

### `Version_Reply(version)`

* **Description:** Replies to `Version(component)`
* **Type:** `0x07`
* **Length:** *Variable*
* **Value:**
    * `version` on *Length* bytes: a string of characters. It **SHOULD NOT** be
        null-terminated.

This command **MUST** be sent after a `Version(component)` command has been
received. If the `component` is unknown or not implemented, `version` **SHOULD**
be `"unknown_component"`. If the `component` is known, `version`:

* **CAN** start with the name of the component if appropriate,
* **SHOULD** end with the version of the component.

For instance, while the *Firmware Version* could be `"1.0.0-rc.1"`, the *ERCP
Library Version* could state its name, like in `"ercp_rust 0.1.0"`, to contrast
with `"ercp_cpp 0.1.1"`.

This command **MUST NOT** be sent if not replying to `Version(component)`.

A device receiving this command without waiting for it **SHOULD** reply with a
`Nack(UNKNOWN_COMMAND)`.

### `Max_Length()`

* **Description:** Gets the maximum acceptable length.
* **Type:** `0x08`
* **Length:** 0

This command **SHOULD** be used to get the maximum length accepted by the other
end.

A device receiving this command **MUST** reply with a
`Max_Length_Reply(max_length)`.

### `Max_Length_Reply(max_length)`

* **Description:** Replies to `Max_Length()`
* **Type:** `0x09`
* **Length:** 1
* **Value:**
    * `max_length` on 8 bits: the maximum acceptable length.

This command **MUST** be sent after a `Max_Length(max_length)` command has been
received. `max_length` **MUST** be the maximum *Value* length accepted by the
device.

This command **MUST NOT** be sent if not replying to `Max_Length(max_length)`.

A device receiving this command without waiting for it **SHOULD** reply with a
`Nack(UNKNOWN_COMMAND)`.
