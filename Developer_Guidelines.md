# AirMsg Developer Guidelines / Quick Reference

This document summarizes the essential details for encoding, decoding, forwarding, and reassembling AirMsg fragments.  
Use it while implementing firmware, gateways, or tooling.

---

## 1. Fragment Structure (in TX order)



[ Base91(Header 5 bytes) ]
[ Base91(Position 9 bytes) ] (optional, only if position flag set)
[ Payload (text or compressed) ]


- Entire fragment must fit in APRS message body (≈67 chars).
- `AMT|` or `AMB|` prefix identifies AirMsg mode.

---

## 2. Header (5 bytes, before Base91 encoding)

### Bit Allocation

| Field        | Bits | Range           |
|--------------|------|-----------------|
| MSGID        | 16   | 0–65535         |
| FRAGIDX      | 4    | 0–15            |
| FRAGCOUNT    | 4    | 0–15            |
| CHANNEL      | 4    | 0–15            |
| HOPS         | 3    | 0–7             |
| FLAGS        | 4    | ACK / CMP / ENC / POSI |
| PADDING      | 5    | unused (0)      |

### Byte Layout



Byte 0: MSGID[15:8]
Byte 1: MSGID[7:0]
Byte 2: (FRAGIDX << 4) | FRAGCOUNT
Byte 3: (CHANNEL << 4) | (HOPS << 1) | FLAG0
Byte 4: (FLAGS << 4) | padding


**FLAGS bits:**
- Bit 3: Position included  
- Bit 2: Encrypted  
- Bit 1: Compressed  
- Bit 0: ACK requested  

---

## 3. Optional Position Block (9 bytes, before payload)

Only present if position flag is set.

### Encoding (fixed-point)



lat_enc = int((lat + 90.0) * 10000) # 24 bits
lon_enc = int((lon + 180.0) * 10000) # 25 bits
alt_enc = altitude_meters & 0xFFFF # 16 bits


### Layout



Bytes 0–2 : LAT[23:0]
Bytes 3–6 : LON[24:0]
Bytes 7–8 : ALT[15:0]


Base91-encode these bytes immediately after the encoded header.

---

## 4. Payload

Two modes:

### **AMT** — Text
- UTF-8 / ASCII text  
- No compression  
- Max usable payload: ~50–55 chars  

### **AMB** — Binary/Compressed
- Compress using heatshrink or similar
- Base91-encode afterward  
- Max usable payload depends on compression ratio  

---

## 5. Building a Fragment (TX-side)



header_bytes = build_header(...)
header_b91 = base91_encode(header_bytes)

pos_bytes = build_position(...) if flags.position else ""
pos_b91 = base91_encode(pos_bytes)

payload = raw_text or compressed_data
payload_b91 = (text as-is for AMT) or Base91(compressed) for AMB

fragment = MODE_PREFIX + header_b91 + pos_b91 + payload_b91


APRS message body = `fragment`.

---

## 6. Parsing a Fragment (RX-side)



read MODE_PREFIX (AMT or AMB)
decode Base91(header_bytes)
parse header fields
if position_flag:
decode Base91(position_bytes)
decode lat/lon/alt
payload = rest of message
if compressed_flag:
payload = decompress(Base91-decode(payload))


Then feed fragment into reassembly buffer.

---

## 7. Reassembly Logic

Key = `(origin, MSGID)`

Steps:
1. Create buffer on first fragment.  
2. Store fragment payload indexed by `FRAGIDX`.  
3. When `all FRAGIDX from 0..FRAGCOUNT` present:
   - Concatenate payloads  
   - Emit decoded message (to app/MQTT)  
4. Drop incomplete buffers on timeout (default 180s).  
5. Optionally send ACK if requested.

---

## 8. Forwarding Rules (mesh mode)

- Increase `HOPS` by 1 before forwarding.  
- Drop if `HOPS >= TTL`.  
- Suppress duplicates via cache key `(origin, MSGID, FRAGIDX)`.  
- Forward only if node subscribes to the fragment's `CHANNEL`.  
- Do not modify any other header fields.

---

## 9. Virtual Channels

- CHANNEL (0–15) lets multiple logical chats share the same RF channel.  
- Nodes choose which channels they handle.  
- MQTT mapping:  
  - `airmsg/chanNN/decoded`  
  - `airmsg/chanNN/raw`  

---

## 10. Recommended Defaults

| Parameter | Value |
|-----------|--------|
| TTL (max hops) | 2 |
| Max fragments | 16 |
| Reassembly timeout | 180 seconds |
| Duplicate cache | LRU 64–256 entries |
| Precision | 0.0001° + 1 m altitude |

---

## 11. Minimal Example Fragment



AMT|>3cA:9PGHello world
^ ^^^^^ ^^^^^^^^^
| | |
| | +-- Payload
| +----------- Base91(header)
+---------------- Mode prefix


---

## 12. Developer Checklist

- [ ] Implement header struct + bit packing  
- [ ] Implement Base91 encode/decode  
- [ ] Implement fixed-point position codec  
- [ ] Implement fragment construction  
- [ ] Implement fragment parser  
- [ ] Reassembly buffer with timeout  
- [ ] Duplicate suppression  
- [ ] Forwarding with hop increment  
- [ ] MQTT bridge (optional)  
- [ ] UI / device application logic  

---

