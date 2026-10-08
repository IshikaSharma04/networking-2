# BHTTP — Binary HTTP Protocol Specification
**Version 1.0 · Course Project: HTTP in Binary**

---

## 1. Overview

BHTTP is a compact, binary-framed application protocol that carries HTTP semantics over a persistent TCP connection. A single TCP connection is opened and **never closed by the client**; all request/response cycles flow over that connection as typed frames.

The design principle is *simple enough to hand-parse, complete enough to carry real HTTP semantics*. Unknown frame types are **skipped cleanly**, leaving room for a version 2.

---

## 2. Frame Format

Every frame begins with a **fixed 9-byte header**, followed by a variable-length payload.

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Payload Length (24)           |   Type (8)    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Flags (8)   |R|              Stream ID (31)                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Payload (0...2^24-1)                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| Field          | Width | Description |
|----------------|-------|-------------|
| Payload Length | 24 bit| Byte count of the payload that follows this header |
| Type           | 8 bit | Frame type identifier (see section 3) |
| Flags          | 8 bit | Type-specific bit flags |
| R              | 1 bit | Reserved. MUST be 0 on send; ignored on receive |
| Stream ID      | 31 bit| Stream identifier. Odd = client-initiated; 0 = connection-level |

All multi-byte integers are **big-endian** (network byte order).

---

## 3. Frame Types

| Code | Name      | Direction       | Description |
|------|-----------|-----------------|-------------|
| 0x01 | REQUEST   | Client to Server| One full HTTP request |
| 0x02 | HEADERS   | Server to Client| Response status + headers |
| 0x03 | DATA      | Server to Client| Response body chunk |
| 0xFF | (unknown) | any             | Must be skipped cleanly |

### 3.1 Skip Rule (Versioning Guarantee)
A receiver encountering a frame type it does not recognize MUST read and discard exactly Payload Length bytes, then continue reading. It MUST NOT close the connection.

This is the sole mechanism by which a version 2 may add new frame types.

---

## 4. REQUEST Frame (0x01)

Flags: END_HEADERS (0x04)
Stream ID: Client-assigned odd integer, starting at 1, incrementing by 2.

Payload layout:

```
+----------+-----------+------------------+--------------+-----------+
| Method(1)| Path Len(2)|   Path (var)    | Hdr Count(1) | Headers.. |
+----------+-----------+------------------+--------------+-----------+
```

- **Method** (1 byte): 0x01 = GET. Other values reserved.
- **Path Length** (2 bytes, big-endian): byte length of the path string (UTF-8, no NUL).
- **Path** (variable): the request path, e.g. /index.html.
- **Header Count** (1 byte): number of header entries that follow.

### 4.1 Header Encoding (Indexed + Length-Prefixed)

Each header is encoded as:

If name index is 0x00..0x09 (indexed):
  [ Name Index (1 byte) ][ Value Length (2 bytes) ][ Value (var) ]

If name index is 0xFF (literal name):
  [ 0xFF ][ Name Length (2 bytes) ][ Name (var) ][ Value Length (2 bytes) ][ Value (var) ]

**Static Header Name Table** (indices 0-9):

| Index | Name |
|-------|------|
| 0x00  | host |
| 0x01  | user-agent |
| 0x02  | accept |
| 0x03  | content-type |
| 0x04  | content-length |
| 0x05  | authorization |
| 0x06  | cache-control |
| 0x07  | accept-encoding |
| 0x08  | connection |
| 0x09  | referer |

Names not in the table use index 0xFF (literal). This mirrors HPACK's indexed header name + literal value mechanisms.

---

## 5. HEADERS Frame (0x02)

Flags: END_HEADERS (0x04)
Stream ID: Mirrors the request's stream ID.

Payload layout:

```
+----------------+--------------+-----------+
| Status Code(2) | Hdr Count(1) | Headers.. |
+----------------+--------------+-----------+
```

- **Status Code** (2 bytes, big-endian): e.g. 0x00C8 = 200, 0x0194 = 404.
- **Header Count** + **Headers**: same encoding as section 4.1.

---

## 6. DATA Frame (0x03)

Flags: END_STREAM (0x01) -- set on the last DATA frame for a stream.
Stream ID: Mirrors the request's stream ID.

Payload: raw body bytes. A response may span multiple DATA frames; the last one carries END_STREAM.

---

## 7. Error Handling

| Condition | Response |
|-----------|----------|
| Unknown frame type | Skip payload; continue |
| Malformed REQUEST payload | Send HEADERS(400); close stream |
| Path not found under root | Send HEADERS(404); no DATA |
| Successful GET | Send HEADERS(200) + DATA(END_STREAM) |

The server **keeps the TCP connection open** after every response.

---

## 8. Connection Lifecycle

```
Client                          Server
  |--- TCP SYN ---------------------->|
  |<-- TCP SYN-ACK ------------------|
  |--- REQUEST (stream 1) ----------->|
  |<-- HEADERS (200, stream 1) -------|
  |<-- DATA (END_STREAM, stream 1) ---|
  |--- REQUEST (stream 3) ----------->|
  |<-- HEADERS (404, stream 3) -------|
  |  ... connection stays open ...    |
```

The client opens **exactly one TCP connection** for all requests. The server must never assume the connection will close after a single request.

---

End of specification. If you cannot annotate your own bytes, the spec is not finished.
