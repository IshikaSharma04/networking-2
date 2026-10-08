# Annotated Hexdump — One Complete BHTTP Request/Response

## Setup
```
Server: ./bserve ./www 9000
Client: ./bcurl -v localhost:9000/index.html
```

---

## REQUEST Frame (Client → Server)

```
00 00 2e 01 04 00 00 00  01 ...
|------ | -- -- |------|
   |      |  |     |
   |      |  |     Stream ID = 0x00000001 (stream 1, client-initiated, odd)
   |      |  Flags = 0x04 (END_HEADERS set)
   |      Type = 0x01 (REQUEST)
   Payload Length = 0x00002E = 46 bytes
```

Full 55-byte frame (9-byte header + 46-byte payload):

```
Offset  Hex                                      ASCII
------  -----------------------------------------------
0000    00 00 2e 01 04 00 00 00  01              ........ .

        ^^^^^^^^^  Frame Header (9 bytes)
        00 00 2e   Payload Length = 46
        01         Type = REQUEST (0x01)
        04         Flags = END_HEADERS (0x04)
        00 00 00 01  Stream ID = 1

0009    01                                       .
        ^^ Method = 0x01 (GET)

000a    00 0b                                    ..
        ^^^^^ Path Length = 11 bytes

000c    2f 69 6e 64 65 78 2e 68  74 6d 6c        /index.html
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Path = "/index.html" (11 bytes)

0017    03                                       .
        ^^ Header Count = 3

        --- Header 0: host ---
0018    00                                       .
        ^^ Name Index = 0x00 (host, from static table)
0019    00 09                                    ..
        ^^^^^ Value Length = 9
001b    6c 6f 63 61 6c 68 6f 73  74              localhost
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ "localhost"

        --- Header 1: user-agent ---
0024    01                                       .
        ^^ Name Index = 0x01 (user-agent)
0025    00 09                                    ..
        ^^^^^ Value Length = 9
0027    62 63 75 72 6c 2f 31 2e  30              bcurl/1.0
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ "bcurl/1.0"

        --- Header 2: accept ---
0030    02                                       .
        ^^ Name Index = 0x02 (accept)
0031    00 04                                    ..
        ^^^^^ Value Length = 4
0033    2a 20 2f 2a                              * /*
        ^^^^^^^^^^^^ "* /*"
```

---

## HEADERS Frame (Server → Client)

```
00 00 14 02 04 00 00 00  01 00 c8 02 04 00 02 33
|------|  -- -- |------|  |----| -- --  |------|...
   |    |  |      |          |    |  |     |
   |    |  |      |          |    |  |     Header count = 2
   |    |  |      |          |    |  Flags = (END_HEADERS)
   |    |  |      |          |    Name Index = 0x04 (content-length)
   |    |  |      |          Status = 0x00C8 = 200
   |    |  |      Stream ID = 1
   |    |  Flags = 0x04 (END_HEADERS)
   |    Type = 0x02 (HEADERS)
   Payload Length = 0x000014 = 20 bytes
```

Full 29-byte frame:

```
Offset  Hex                                      ASCII
------  -----------------------------------------------
0000    00 00 14 02 04 00 00 00  01              .........

        Frame Header (9 bytes):
        00 00 14   Payload Length = 20
        02         Type = HEADERS (0x02)
        04         Flags = END_HEADERS
        00 00 00 01  Stream ID = 1

        Payload (20 bytes):
0009    00 c8                                    ..
        ^^^^^ Status Code = 200 (0x00C8)

000b    02                                       .
        ^^ Header Count = 2

        --- Header 0: content-length ---
000c    04                                       .
        ^^ Name Index = 0x04 (content-length)
000d    00 02                                    ..
        ^^^^^ Value Length = 2
000f    33 39                                    39
        ^^^^^ "39" (the file is 39 bytes)

        --- Header 1: content-type ---
0011    03                                       .
        ^^ Name Index = 0x03 (content-type)
0012    00 09                                    ..
        ^^^^^ Value Length = 9
0014    74 65 78 74 2f 68 74 6d  6c              text/html
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ "text/html"
```

---

## DATA Frame (Server → Client)

```
00 00 27 03 01 00 00 00  01 <body>
|------|  -- -- |------|  ^^^^^^^^^
   |    |  |      |       body bytes
   |    |  |      Stream ID = 1
   |    |  Flags = 0x01 (END_STREAM — last frame for this stream)
   |    Type = 0x03 (DATA)
   Payload Length = 0x000027 = 39 bytes
```

Full 48-byte frame:

```
Offset  Hex                                      ASCII
------  -----------------------------------------------
0000    00 00 27 03 01 00 00 00  01              ..'......

        Frame Header (9 bytes):
        00 00 27   Payload Length = 39
        03         Type = DATA (0x03)
        01         Flags = END_STREAM (last data for stream 1)
        00 00 00 01  Stream ID = 1

        Payload (39 bytes — the file contents):
0009    3c 68 74 6d 6c 3e 3c 62  6f 64 79 3e    <html><body>
0015    48 65 6c 6c 6f 20 42 48  54 54 50 21    Hello BHTTP!
0021    3c 2f 62 6f 64 79 3e 3c  2f 68 74 6d    </body></htm
002d    6c 3e 0a                                l>.
```

---

## Annotations Summary

| Byte(s) | Meaning |
|---------|---------|
| `00 00 2e` | Payload Length = 46 (REQUEST) |
| `01` | Frame type = REQUEST |
| `04` | END_HEADERS flag |
| `00 00 00 01` | Stream ID = 1 |
| `01` | Method = GET |
| `00 0b` | Path is 11 bytes |
| `2f 69 6e 64 65 78 2e 68 74 6d 6c` | "/index.html" |
| `03` | 3 request headers follow |
| `00` / `01` / `02` | Static name indices: host, user-agent, accept |
| `00 c8` | Status 200 in response HEADERS |
| `00 00 27` | Payload Length = 39 (DATA body) |
| `01` (DATA flags) | END_STREAM — connection stays open but stream is done |

*Every byte accounted for. The spec is finished.*
