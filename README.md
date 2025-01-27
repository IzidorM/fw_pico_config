# Pico Config

This repository contains a simple implementation of an
application-level command protocol. It supports packet receive
confirmation and features a C reference implementation. The protocol
is easy to use and flexible, allowing users to define mappings between
packet IDs and their corresponding payload parsing functions and
response handlers.


## Key Features
- Reference implementation available in C (and Python)
- Compact and efficient codebase
- BSD licensed for flexible use

## Definition of the pico config packet 

[header (1byte), payload(0-Nbytes)] 

where N is set by the application for every ID. The max payload size
should be defined by the lower layer MTU.

   
### Header

```

|-------------------+---------------------------------+-----------------------------------|
| 7 (bit)           | 6 (bit)                         | 5:0 (bits)                        |
|-------------------+---------------------------------+-----------------------------------|
| packet direction: | if request:                     | id:                               |
| 0: response       | 0 - no response                 | Id describe the message payload.  |
| 1: request        | 1 - send back ack/nack response | They are assigned by application. |
|                   |                                 |                                   |
|                   | if response:                    |                                   |
|                   | 0 - nack                        |                                   |
|                   | 1 - ack                         |                                   |
|-------------------+---------------------------------+-----------------------------------|

```

### Payload 

The Pico config does not provide specific information about the
payload size and structure. Instead, it uses the first byte of the
packet as a header. This header byte contains an ID that Pico config
uses to locate the registered payload handler. It is the
responsibility of the application to register payload handlers for all
the IDs that will be used. If there is no handler for specific ID, the
package is dropped.


## Example of a Pico Config Message Exchange

Every Pico config transfer begins with a request packet. If the
response flag in the request header is set, the receiver must reply
with a response packet using the same ID within the next X
milliseconds (X should be defined according to the specific use
case). While waiting for the response packet, the sender should not
send a new request packet. If there were no response packet within the
defined response time window, the sender should retry sending the
request.  With response, receiver inform the sender about the
successfull reception of the packet. If the reception failed due
receiver unability to perform the requested operation, it should send
back the nack response packet, where the exact error code descriping
the error can be placed in the payload. For more detailes see the
example section.

### Example of the packet structure in request and response packets

Sending a packet with ID = 0x8 and a 3-byte payload: 0xaa 0xbb 0xcc,
with response enabled.

- *TX (Transmit)*: ['0xc8' (request, enable response, ID=0x8), 0xaa,
  0xbb, 0xcc]
- *RX (Receive)*:
  - If ACK: ['0x48' (response flag, ACK, ID), rsp_payload]
  - If NACK: ['0x08' (response flag, NACK, ID), rsp_payload]

_NOTE: The response payload (rsp_payload) is optional._

### Example of Pico Config Packet Usage in an Application

An application wants to query the receiving system for the software
(firmware) version running there. The developers have decided that the
ID for the software version query will be 0x1. They also specify the
payload format for this message. The payload size should be 4 bytes,
representing a software version number as a uint32 in little-endian
format.

- *Example Usage*:
  - The transmitter sends a request with ID 0x1 and an empty payload:
    - TX (Transmit)*: ['0xc1' (request, enable response, ID=0x1)] 
  - The receiver responds with the software version:
    - RX (Receive)*: ['0x41' (response flag, ACK, ID=0x1), 0x01, 0x00, 0x00, 0x00] (e.g., version 1.0.0.0)

If there's no response within the specified time window, the sender
should retry sending the request. If the receiver cannot process the
request, it should send a NACK response packet with an appropriate
error code in the payload.

_ADDITIONAL NOTE: Ensure that the response time window and error codes are clearly defined for robust communication._
