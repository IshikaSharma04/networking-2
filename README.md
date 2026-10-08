# BHTTP — HTTP in Binary

> Course Project · HTTP in Binary · Two Tracks, One Protocol

A custom binary-framed HTTP-like protocol implemented from scratch in C, including:
- A **server** (`bserve`) — Track 1
- A **client** (`bcurl`) — Track 2
- A two-page **protocol specification** (`spec.md`)
- An **annotated hexdump** of a real request/response (`hexdump_annotated.md`)

---

## Building

```bash
make
# produces: ./bserve  ./bcurl
```

## Running

**Server** (serves files from `./www` on port 9000):
```bash
./bserve ./www 9000
```

**Client** (fetch index.html with hexdump output):
```bash
./bcurl -v localhost:9000/index.html
```

## Tests

```bash
make test      # 200 OK test: fetches www/index.html, expects exit 0
make test404   # 404 test:    fetches missing file,   expects exit 1
```

## Protocol Summary

BHTTP uses a **fixed 9-byte frame header** (identical field layout to HTTP/2):

```
[Length:24][Type:8][Flags:8][R:1][StreamID:31] [Payload...]
```

| Field          | Width | Notes |
|----------------|-------|-------|
| Payload Length | 24 bit | big-endian |
| Type           | 8 bit  | 0x01=REQUEST, 0x02=HEADERS, 0x03=DATA |
| Flags          | 8 bit  | END_STREAM=0x01, END_HEADERS=0x04 |
| R (reserved)   | 1 bit  | always 0 |
| Stream ID      | 31 bit | odd = client-initiated |

### Key design decisions

| Decision | Rationale |
|----------|-----------|
| 24-bit length (same as HTTP/2) | Avoids 32-bit split-flag complexity; enough for files |
| Static header name table (10 entries) | Covers the 10 most common header names in one byte each |
| Literal fallback index `0xFF` | Any name not in the table can still be sent |
| Skip rule for unknown types | A receiver MUST skip, not drop — version 2 extensibility |
| Keep connection open | Matches HTTP/2 semantics; server never assumes EOF = done |

Full details: see [`spec.md`](spec.md).

## File Layout

```
networking-prorj2/
├── bhttp_protocol.h   # shared protocol constants, structs, wire helpers, parse_headers
├── bserve.c           # Track 1: binary HTTP server
├── bcurl.c            # Track 2: binary HTTP client
├── Makefile           # build + integration tests
├── spec.md            # two-page protocol specification
├── hexdump_annotated.md  # annotated hexdump (submission item 3)
└── www/
    └── index.html     # sample file for testing
```

## What you hand in

1. **Spec** → [`spec.md`](spec.md) ✅
2. **Program** → `bserve.c`, `bcurl.c`, `bhttp_protocol.h`, `Makefile` ✅
3. **Annotated hexdump** → [`hexdump_annotated.md`](hexdump_annotated.md) ✅
