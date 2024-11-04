# Protobuf Framing 
## Overview
The Protobuf protocol does not have a built in framing feature. When sending a stream of bytes the protocol does not provide any information of when a packet starts and ends or what type the packet is. This library aims to fix this issue.

The protobuf framing library allows for multiple use cases. If the underlying transport layer has error correction/detection then the base format may be used. If the transport stream is a raw string of bytes then a packet format with start bytes and a checksum are available. 

## Defining Message Format
In order to serialize data each .proto file needs to have a message id associated with it and each message needs to have an id associated with it. In keeping with the core principles of protobuf, any depcrated proto file IDs or message IDs should be reserved. 

### Defining a Main Proto File
A main proto file needs to be defined. It should import all the required proto files. It needs to define an enum called ProtoFileIds. Each element of ProtoFileIds corresponds with a .proto file. Each element is the name of the proto file in uppercase.


```
syntax = "proto3";
package messages;

import public "vehicle_control.proto";
import public "camera_control.proto";

enum ProtoFileIds {
  VEHICLE_CONTROL = 0;
  RESERVED_1 = 1; // Previously used .proto file that is not longer used
  CAMERA_CONTROL = 1;
}
```

### Format of Message Proto File
A regular .proto file may be used. They main change that makes the proto file compatible is to add a MsgIds enum to the .proto file. The name of each element should be the message name in uppercase.

```
syntax = "proto3";
package camera_control;

enum MsgIds {
  SHUTTER = 0;
  RESERVED_1 = 1; // Previously defined message that is no longer used
  POSITION = 2;
}

message shutter {
  boolean trigger = 1;
}

message position {
  double lat = 1;
  double lon = 2;
  float alt = 3;
}
```

## Packet Framing Format
There are two packet framing formats:

The base format may be used when the data is already packetized. The base format can be used when sending a websocket, or tcp packet.

The serial format may be used for a raw serial stream. It includes a header byte and checksum

### Base Format 1
The base format of the protobuf framer (3 bytes overhead):

| byte 0 | byte 1 | byte 2 | byte 3 - byte (3 + N - 1) |
| --- | --- | --- | --- |
| FILE ID | MSG ID | LENGTH | PAYLOAD  
| 0x | 0x | N | DATA |

### Base Format 2
The base format of the protobuf framer with system ID field (4 bytes overhead):

| byte 0 | byte 1 | byte 2 | byte 3 | byte 4 - byte (4 + N - 1) |
| --- | --- | --- | --- | --- |
| SYS ID | FILE ID | MSG ID | LENGTH | PAYLOAD  
| 0x | 0x | 0x | N | DATA |

### Serial Format 1
This serial format of the protobuf framer has 7 bytes overhead:

| byte 0 | byte 1 | byte 2 | byte 3 | byte 4 | byte 5 - byte (5 + N - 1) | byte (6 + N) | byte (7 + N) |
| --- | ---| --- | --- | --- | --- | --- | --- |
| START BYTE 0 | START BYTE 1  | FILE ID | MSG ID | LENGTH | PAYLOAD  | CHECKSUM 1  | CHECKSUM 2  
| 0xA2 | 0x90 | 0x | 0x | N | DATA | 0x | 0x |

### Serial Format 2
This serial format of the protobuf framer adds a system ID to differentiate messages from different senders. This method has 8 bytes overhead:

| byte 0 | byte 1 | byte 2 | byte 3 | byte 4 | byte 5 | byte 6 - byte (6 + N - 1) | byte (7 + N) | byte (8 + N) |
| --- | ---| --- | --- | --- | --- | --- | --- | --- |
| START BYTE 0 | START BYTE 1  | SYS ID | FILE ID | MSG ID | LENGTH | PAYLOAD  | CHECKSUM 1  | CHECKSUM 2  
| 0xA2 | 0x91 | 0x0 | 0x | 0x | N | DATA | 0x | 0x |

## Language Implentation

### Embedded Proto

To send a message first include its header file.
```
include "messages/camera_control_framer.h"

WriteInterface writeBuffer;
camera_control::shutter msg;

// encode base format
// camera_control::shutter_msg_encode_base(msg, writeBuffer);

// encode with serial format 1
// camera_control::shutter_msg_encode(msg, writeBuffer);

// encode with serial format 2
uint8_t system_id = 1;
camera_control::shutter_msg_encode(system_id, msg, writeBuffer);


sendData(writeBuffer);
```

To parse out a message:

```
ProtobufFramer<256> protobuf_framer;

void process_camera_control_message(){
  switch (result.msgId)
  {
  case camera_control::MsgIds::shutter:
    camera_control::shutter shtr;
    shtr.deserialize(protobuf_framer.data)
    break;

  case camera_control::MsgIds::position:
    camera_control::position pos;
    pos.deserialize(protobuf_framer.data)
    break;

  default:
    printf("Unknown message id")
    break;
  }
}

protobuf_framer_result = protobuf_framer.parse(c);
if (result.success){
  switch (result.fileId)
  {
  case messages::ProtoFileIds::VEHICLE_CONTROL:
    process_vehicle_control_message(result);
    break;

  case messages::ProtoFileIds::CAMERA_CONTROL:
    process_camera_control_message(result);
    break;

  default:
    printf("Unknown proto file id")
    break;
  }
}


```