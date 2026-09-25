\# OTA Protocol v1



Status: frozen alongside the GATT spec. Changing it requires changing

ebike-firmware AND ebike-app in lockstep.



\## Characteristic

OTA  0000EB05-7A5C-4C7D-9E42-3B1F00000001

Service 0000EB01-7A5C-4C7D-9E42-3B1F00000001 (existing)

Properties: Write, Write Without Response, Notify

Control opcode 0x30 remains reserved and UNUSED.

All integers little-endian unless stated.



\## Signed block (46 bytes)

| Off | Size | Field                                   |

|-----|------|-----------------------------------------|

| 0   | 4    | magic "EBFW" = 45 42 46 57              |

| 4   | 1    | blockFormat = 0x01                      |

| 5   | 1    | hwRev (0x01 SuperMini bench, 0x02 rev-0.1 PCB; 0x00 invalid) |

| 6   | 1    | fwMajor                                 |

| 7   | 1    | fwMinor                                 |

| 8   | 1    | fwPatch                                 |

| 9   | 1    | reserved, must be 0x00                  |

| 10  | 4    | imageSize (u32 LE)                      |

| 14  | 32   | SHA-256 of the image bytes              |



Signature: ECDSA P-256 with SHA-256 over the 46-byte block

(i.e. ECDSA\_sign(SHA-256(block))). Encoded as 64 bytes raw r||s,

each 32 bytes big-endian. Public key: 65-byte uncompressed point (04||X||Y).



Version order: (major, minor, patch) lexicographic.

Anti-downgrade: reject version < running. Equal is allowed (reinstall).



\## App -> dongle (byte 0 = command)

| Cmd  | Name     | Payload                          | Write type    |

|------|----------|----------------------------------|---------------|

| 0x01 | INFO     | none                             | with response |

| 0x02 | BEGIN    | block(46) + signature(64)        | with response |

| 0x03 | DATA     | offset u32 + 1..(ATT\_MTU-3-5) bytes | no response |

| 0x04 | END      | none                             | with response |

| 0x05 | ACTIVATE | none                             | with response |

| 0x06 | ABORT    | none                             | with response |



\## Dongle -> app (notify, byte 0 = response)

| Rsp  | Name       | Payload |

|------|------------|---------|

| 0x81 | INFO       | fwMajor, fwMinor, fwPatch, hwRev, state u8, maxImageSize u32, bootState u8 |

| 0x82 | BEGIN\_OK   | windowBytes u16 |

| 0x83 | PROGRESS   | committedBytes u32 |

| 0x84 | END\_OK     | none (image verified and staged, NOT active) |

| 0x85 | ACTIVATING | none (reboot \~250 ms after this notification) |

| 0x8F | ERROR      | code u8, detail u32 |



state: 0 IDLE, 1 RECEIVING, 2 STAGED

bootState: 0 confirmed, 1 pending verification, 2 last update rolled back



\## Error codes

0x01 BAD\_STATE · 0x02 BAD\_LENGTH · 0x03 BAD\_FORMAT · 0x04 HW\_MISMATCH

0x05 DOWNGRADE · 0x06 TOO\_LARGE · 0x07 BAD\_SIGNATURE · 0x08 OFFSET\_MISMATCH

(detail = expected offset) · 0x09 HASH\_MISMATCH · 0x0A FLASH\_ERROR

0x0B TIMEOUT · 0x0C BUSY · 0x0D REFUSED\_MODE



\## Rules

1\. Signature, magic, blockFormat, reserved byte, hwRev, version and size

&#x20;  are all checked at BEGIN, BEFORE any flash is erased.

2\. The session is bound to the BLE connection that sent BEGIN.

&#x20;  Disconnect -> session aborted, running image untouched.

3\. Flow control: the app never has more than windowBytes sent beyond the

&#x20;  last PROGRESS.committedBytes. The dongle sends PROGRESS at least every

&#x20;  4096 committed bytes and at the final byte.

4\. DATA offsets must be contiguous. On mismatch the dongle sends ONE

&#x20;  OFFSET\_MISMATCH (detail = expected offset) and silently drops DATA

&#x20;  until one arrives at the expected offset. The app resumes from detail.

5\. 15 s without DATA while RECEIVING -> abort, TIMEOUT.

6\. END: received == imageSize AND streamed SHA-256 == block SHA-256,

&#x20;  then image validation. Boot partition is NOT changed at END.

7\. ACTIVATE: only in STAGED, only from the same connection. Sets the boot

&#x20;  partition, notifies ACTIVATING, reboots. STAGED expires after 10 min -> IDLE.

8\. The keepalive task is never paused, gated or deprioritised by OTA.

9\. While RECEIVING or STAGED, Control opcodes 0x10 and 0x20 are rejected;

&#x20;  0x01 and 0x02 still work.

10\. A new image boots as pending verification and marks itself valid only

&#x20;   after: baseline round-trip check passed, keepalive task running, BLE

&#x20;   up, 30 s uptime. Otherwise the previous image is restored.



\## Release manifest (public host, untrusted)

{

&#x20; "manifestFormat": 1,

&#x20; "releases": \[

&#x20;   { "hwRev": 2, "version": "1.3.0", "url": "https://.../fw.bin",

&#x20;     "size": 812345, "sha256": "<hex>", "signedBlock": "<hex, 46 bytes>",

&#x20;     "signature": "<hex, 64 bytes>", "notes": "..." }

&#x20; ]

}

Trust derives ONLY from the signature. Every manifest field must match the

verified signed block, or the entry is rejected.

