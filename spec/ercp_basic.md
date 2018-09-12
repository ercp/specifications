# ERCP Basic

## Table of contents

* [Generalities](#generalities)
* [Frame format](#frame-format)
    * [Fields](#fields)
    * [Implementation](#implementation)
* [Build-in commands](#built-in-commands)
    * [`Ping()`](#ping)
    * [`Ack()`](#ack)
    * [`Nack(reason)`](#nackreason)
    * [`Reset()`](#reset)
    * [`Protocol()`](#protocol)
    * [`Protocol_Reply(major, minor, patch)`](#protocol_replymajor-minor-patch)
    * [`Version(component)`](#versioncomponent)
    * [`Version_Reply(version)`](#version_replyversion)
    * [`Max_Length()`](#max_length)
    * [`Max_Length_Reply()`](#max_length_replymax_length)

## Generalities

ERCP Basic is the first protocol from the ERCP suite. It aims to provide simple
and easy-to-implement yet reliable communication with embedded systems.

## Frame format

Frames **MUST** follow the format described below.

* *The frame reads top to bottom*
* *Each cell represents an octet*
* *Quoted values are ASCII characters*

```
|---------|--
|   'E'   |  |
|---------|  |
|   'R'   |  |
|---------|  |
|   'C'   |  |- Start sequence
|---------|  |
|   'P'   |  |
|---------|  |
|   'B'   |  |
|---------|--
|   Type  |  |
|---------|  |- Header
|  Length |  |
|---------|--
|         |  |
    ...      |
|         |  |
|  Value  |  |- Payload
|         |  |
    ...      |
|         |  |
|---------|--
|   CRC   |  |
|---------|  |- Footer
|  'EOT'  |  |
|---------|--
```

### Fields

* **Type:** The command type. Several commands are
    [built-in](#built-in-commands) and the protocol is made to be extended by
    application commands handled by user callbacks.
* **Length:** The length of the *Value* field.
* **Value:** A field containing additional data. **MUST** be *Length*-bytes
    long.
* **CRC:** CRC for the *Type*, *Length* and *Value* fiels combined. **Its
    computation is yet to be determined.**

### Implementation

A device receiving an incorrectly formatted frame **MUST** ignore it.

A device receiving a correctly formatted frame **MUST** compute its CRC. If the
CRC is valid, the device **MUST** call the command callback corresponding to the
*Type*. If the CRC is invalid or the command type is not implemented, the device
**MUST** send a [`Nack(reason)`](#nackreason).

A device receiving an incomplete frame... **To be determined**.

A device **MAY** set a maximum acceptable *Length* inferior to 255. In the case
a too long frame is received, the device **MUST** send a
[`Nack(TOO_LONG)`](#nackreason).

## Built-in commands

  Type | Command
------:|:-----------------
`0x00` | [`Ping()`](#ping)
`0x01` | [`Ack()`](#ack)
`0x02` | [`Nack(reason)`](#nackreason)
`0x03` | [`Reset()`](#reset)
`0x04` | [`Protocol()`](#protocol)
`0x05` | [`Protocol_Reply(major, minor, patch)`](#protocol_replymajor-minor-patch)
`0x06` | [`Version(component)`](#versioncomponent)
`0x07` | [`Version_Reply(version)`](#version_replyversion)
`0x08` | [`Max_Length()`](#max_length)
`0x09` | [`Max_Length_Reply()`](#max_length_replymax_length)

In the following subsections, commands are referred as functions taking zero or
more arguments. Each command comes with a description, a type, a length and a
value if appropriate. Then follows its expected behaviour.

All built-in commands **MUST** be implemented if not stated otherwise.

### `Ping()`

* **Description:** Tests connectivity.
* **Type:** `0x00`
* **Length:** 0

This command **MAY** be used to test connectivity.

A device receiving this command **MUST** reply with an [`Ack()`](#ack).

### `Ack()`

* **Description:** Acknowledgement.
* **Type:** `0x01`
* **Length:** 0

This command **SHOULD** be sent to acknowledge a previously received command.

This command **MAY** be sent immediately after receiving the command or after
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

Values between `0x05` and `0x0F` included are reserved for future use. They
**MUST NOT** be used by application code.

Other values for `reason` **MAY** be implemented.

This command **MUST** be sent after a command with an invalid CRC has been
received. In this case, `reason` **MUST** be `INVALID_CRC`.

This command **MUST** be sent after an unknown command has ben received. In this
case, `reason` **MUST** be `UNKNOWN_COMMAND`.

This command **SHOULD** be sent to negatively acknowledge a previously received
command. A custom `reason` **MAY** be used.

This command **MAY** be sent immediately after receiving the command or after
doing some processing. In the second case, it **SHOULD** mean an problem has
occured while processing the command.

This command **MUST NOT** be sent if not replying to a command.

A device receiving this command **MUST NOT** reply.

### `Reset()`

* **Description:** Resets the device.
* **Type:** `0x03`
* **Length:** 0

This command **MAY** be used to reset the device.

A device receiving this command **MAY** permorm a reset.

If the expected behaviour is not implemented, a device receiving this command
**MUST** send a [`Nack(UNKNOWN_COMMAND)`](#nackreason).

### `Protocol()`

* **Description:** Gets the protocol version.
* **Type:** `0x04`
* **Length:** 0

This command **SHOULD** be used to check the protocol version implemented by the
other end.

A device receiving this command **MUST** reply with a [`Protocol_Reply(major,
minor, patch)`](#protocol_replymajor-minor-patch).

### `Protocol_Reply(major, minor, patch)`

* **Description:** Replies to [`Protocol()`](#protocol).
* **Type:** `0x05`
* **Length:** 3
* **Value:**
    * `major` on 8 bits: the major version
    * `minor` on 8 bits: the minor version
    * `patch` on 8 bits: the patch version

This command **MUST** be sent after a [`Protocol()`](#protocol) command has been
received. Its value **MUST** state the version of the protocol implemented by
the device.

This command **MUST NOT** be sent if not replying to [`Protocol()`](#protocol).

A device receiving this command without waiting for it **SHOULD** reply with a
[`Nack(UNKNOWN_COMMAND)`](#nackreason).

### `Version(component)`

* **Description:** Gets the version of a software component.
* **Type:** `0x06`
* **Length:** 1
* **Value:**
    * `component` on 8 bits

The following values for `component` **SHOULD** be implemented:

* `0x00`: *Firmware Version*
* `0x01`: *ERCP Library Version*

Values between `0x02` and `0x0F` are reserved for future use. They **MUST NOT**
be used by application code.

Other values for `component` **MAY** be implemented. Custom values **MUST NOT**
replace the ones listed above.

This command **MAY** be used to get the version of a software component.

A device receiving this command **MUST** reply with a
[`Version_Reply(version)`](#version_replyversion).

### `Version_Reply(version)`

* **Description:** Replies to [`Version(component)`](#versioncomponent)
* **Type:** `0x07`
* **Length:** *Variable*
* **Value:**
    * `version` on *Length* bytes: a string of characters. It **SHOULD NOT** be
        null-terminated.

This command **MUST** be sent after a [`Version(component)`](#versioncomponent)
command has been received. If the `component` is unknown or not implemented,
`version` **SHOULD** be `"unknown_component"`. If the `component` is known,
`version`:

* **MAY** start with the name of the component if appropriate,
* **SHOULD** end with the version of the component.

For instance, while the *Firmware Version* could be `"1.0.0-rc.1"`, the *ERCP
Library Version* could state its name, like in `"ercp_rust 0.1.0"`, to contrast
with `"ercp_cpp 0.1.1"`.

This command **MUST NOT** be sent if not replying to `Version(component)`.

A device receiving this command without waiting for it **SHOULD** reply with a
[`Nack(UNKNOWN_COMMAND)`](#nackreason).

### `Max_Length()`

* **Description:** Gets the maximum acceptable *Length*.
* **Type:** `0x08`
* **Length:** 0

This command **SHOULD** be used to get the maximum *Length* accepted by the
other end.

A device receiving this command **MUST** reply with a
[`Max_Length_Reply(max_length)`](#max_length_replymax_length).

### `Max_Length_Reply(max_length)`

* **Description:** Replies to [`Max_Length()`](#max_length)
* **Type:** `0x09`
* **Length:** 1
* **Value:**
    * `max_length` on 8 bits: the maximum acceptable length.

This command **MUST** be sent after a [`Max_Length()`](#max_length) command has
been received. `max_length` **MUST** be the maximum *Length* accepted by the
device.

This command **MUST NOT** be sent if not replying to
[`Max_Length()`](#max_length).

A device receiving this command without waiting for it **SHOULD** reply with a
[`Nack(UNKNOWN_COMMAND)`](#nackreason).
