# AirMsg

**AirMsg** — Long-range hybrid messaging over LoRa/APRS with MQTT integration and peer-to-peer forwarding.

AirMsg is a lightweight, APRS-compatible protocol designed for SMS-style messaging. It supports multi-fragment messages, optional compression, peer-to-peer forwarding, and seamless MQTT bridging for Internet-connected devices.

---

## Features

- **APRS-compatible:** Printable ASCII, fits APRS message size limits.  
- **Hybrid RF + MQTT:** RF messages can be bridged to MQTT topics for logging or apps.  
- **Peer forwarding:** Multi-hop forwarding among AirMsg nodes without relying on APRS digipeaters.  
- **Fragmented messaging:** Supports multi-fragment reassembly.  
- **Compression support:** Optional binary mode with Base91 + heatshrink compression.  
- **Virtual channels:** Logical separation of chats over a single physical channel.  
- **ACKs & reliability:** Optional acknowledgment system.  
- **Compact metadata:** Message ID, fragment index/count, channel, hops, and flags packed into 5 bytes.  

---

## Protocol Overview

AirMsg messages have **two modes**:

- **Text mode (`AMT`)** — plain ASCII fragments.  
- **Binary/compressed mode (`AMB`)** — compressed and Base91-encoded payload.  

All messages start with a **compact 5-byte binary header** (Base91-encoded for APRS compliance).  

---

## Compact Binary Header (5 bytes)

**Bit allocation:**

| Field       | Bits | Description |
|------------|------|-------------|
| `MSGID`    | 16   | Unique per sender, wraps at 65535 |
| `FRAGIDX`  | 4    | Fragment index (1–16) |
| `FRAGCOUNT`| 4    | Total fragments (1–16) |
| `CHANNEL`  | 4    | Virtual channel ID (0–15) |
| `HOPS`     | 3    | Current hop count (0–7) |
| `FLAGS`    | 4    | 1-bit each: ACK, compressed, encrypted, reserved |
| **Padding** | 5   | Unused, set to 0 |

**Byte layout:**

Byte 0: MSGID[15:8]   
Byte 1: MSGID[7:0]   
Byte 2: FRAGIDX[3:0] | FRAGCOUNT[3:0]   
Byte 3: CHANNEL[3:0] | HOPS[2:0] | FLAG[0]   
Byte 4: FLAGS[3:0] | padding[3:0]   


### Encoding

- 5-byte header → 7–8 Base91 characters → prepended to payload.     
- Example APRS message (binary mode):   

```
AMB|qK9a@xGHello from the summit
```   


- Receiver Base91-decodes first 7–8 chars → extracts metadata → reassembles message.   

---

## Fragmentation

- Each message is broken into fragments ≤67 APRS chars (including header).  
- Fragment size depends on payload mode:
  - Text mode: ~50–55 chars payload  
  - Binary mode: ~42–55 chars depending on compression  

- Each fragment carries:
  - Message ID  
  - Fragment index / count  
  - Virtual channel  
  - Hops / TTL  
  - Flags  

---

## Forwarding & Peer Mesh

AirMsg nodes implement **peer-to-peer forwarding**:

- Forward only AirMsg-labeled messages (`AMT`/`AMB`).  
- Increment `HOPS` before forwarding.  
- Stop forwarding when `HOPS >= TTL` (default TTL = 2).  
- Suppress duplicates via LRU cache of `MSGID + FRAGIDX`.  
- Respect airtime and APRS norms — no flooding.  
- Virtual channel is respected: nodes only forward messages for channels they handle.

---

## Virtual Channels

- Logical separation within a single physical channel.  
- `CHANNEL` field in header (4 bits → 16 channels).  
- Nodes can subscribe to a subset of channels for multi-chat operation.  
- MQTT mapping: each channel → separate topic (`airmsg/chan01`, `airmsg/chan02`, …).

---

## MQTT Integration

AirMsg nodes can bridge to MQTT:

**Topics:**
- airmsg/gateway/<call>/rx # RF fragments received
- airmsg/gateway/<call>/tx # Messages to send via RF
- airmsg/decoded/incoming # Fully reassembled messages
- airmsg/raw/rf # Raw APRS AirMsg fragments
- airmsg/nodes/<call>/status # Node status



**Decoded JSON message example:**

```
{
  "origin": "DL1ABC-9",
  "msgid": "1A2B",
  "channel": 1,
  "text": "Hello from the summit",
  "received_at": "2025-11-22T12:34:56Z",
  "via": "rf/gateway/DL1GATE",
  "hops": 1
}
```

