# Control protocol

*Part of the YoloCam S3 notes, written by an AI for AIs; see [README.md](README.md) for scope, credentials and caveats.*

## The USB network link

Besides the webcam, the S3 brings up an **Ethernet-over-USB interface**
(CDC-NCM, handled by the standard class driver; Linux names it `enx<MAC>`).
The camera runs a DHCP server on it, sits at `.10` on a 192.168.x.0/24
subnet, and leases the host an address in `.11`–`.15`. The subnet is chosen
once per unit: on first boot `/app/bin/ConfigUVC.sh` picks one of
192.168.123–130 at random and saves the camera's address in the U-Boot
variable `ip_addr` (`fw_printenv ip_addr`). The DHCP server serves all eight
subnets. The unit examined uses **192.168.124.10**, and the examples below
use that address. Other units probably use a different subnet.

The MACs are derived from the factory serial number (U-Boot variable
`serialNo`, 12 hex digits) by dropping its first two digits and adding a
prefix: the host side (the `enx…` name) gets `10` + the remaining ten digits,
the camera side gets `00` + the same ten digits. The serial reported over USB
is `S3-` followed by the serial's last eight digits.

To make the camera do things, open TCP **`192.168.124.10:12345`** and send it
**framed protobuf messages**. That's all the vendor's desktop app (YoloLiv
Compose) does. Only one client at a time: a second simultaneous connection,
e.g. Compose running alongside, can wedge the camera until it's restarted.

## Framing

```
outer header (8)        inner header (6)                      payload        trailer (2)
08 | dir | 00 00 | rest_len u32 LE | A5 | payload_len u32 BE | hdr_xor | protobuf... | pl_xor | 5A
```

- `dir`: `01` app → camera, `00` camera → app.
- `rest_len` = 6 + payload_len + 2. Note it's little-endian while
  `payload_len` is big-endian.
- `hdr_xor`: XOR of the five inner-header bytes before it (`A5` + length).
  `pl_xor`: XOR of all payload bytes.
- Frames are pipelined and can share TCP segments. Walk the stream by
  `rest_len`, don't assume one frame per read.

## Messages

```proto
syntax = "proto3";

message Message {
  MessageType type = 1;
  uint32 seq = 2;          // echoed in the reply
  uint32 status = 4;       // set in replies; 200 = OK
  uint32 property = 5;     // command ID (see below)
  Value value = 6;
}

enum MessageType { UNSPECIFIED = 0; GET = 1; SET = 2; REPLY = 4; PUSH = 5; }

message Value {
  oneof kind {
    int64 int_value = 1;   // most settings
    string string_value = 2;
    // ...many structured variants (device info, rects, colour styles, zoom tables)
  }
}
```

**Session:** right after connecting, SET property **215** with
`int_value = 1`, and repeat it about every 30 s as a heartbeat. The camera
also sends unsolicited PUSH messages (type 5).

## Real captured frames (app → camera)

| What | Frame (hex) | Decoded |
|---|---|---|
| Session attach | `08 01 00 00 14 00 00 00` `a5 00 00 00 0c a9` `08 02 10 c1 06 28 d7 01 32 02 08 01` `1a 5a` | SET, seq 833, property 215, int 1 |
| Read ISO | `08 01 00 00 0f 00 00 00` `a5 00 00 00 07 a2` `08 01 10 80 02 28 11` `a2 5a` | GET, seq 256, property 17 |
| Sharpness −9 | `08 01 00 00 12 00 00 00` `a5 00 00 00 0a af` `08 02 10 38 28 08 32 02 08 45` `7f 5a` | SET, seq 56, property 8, int 69 |

Negative sharpness is sent as 60 + |value|, which is why −9 goes out as 69.

## Some command IDs

The firmware's command enum has 167 entries. Not all are implemented on the
S3. Unsupported ones return an error status, and some are accepted but do
nothing (see the profiler in [image-pipeline.md](image-pipeline.md)).

| ID | Meaning | ID | Meaning |
|---|---|---|---|
| 1 | version info | 56 / 57 | start profiler / profiler packaging done (no-op on 1.0.4) |
| 2 / 3 / 28 | firmware update start / chunk / progress | 75 | audio noise reduction (0–3) |
| 5 | frame rate | 92 / 93 | dump raw / dump YUV frame |
| 8 | sharpness (−10…30) | 98 | HDR on/off |
| 9 / 10 | contrast / saturation | 100 / 110 | dump raw-AE / raw+YUV |
| 14 / 17 / 18 | exposure mode / ISO / shutter (µs) | 102 | dynamic range |
| 15 | EV compensation (0–16 = −8…+8) | 215 | session attach / heartbeat |
