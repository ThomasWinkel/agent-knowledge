---
title: GZipStream in .NET Framework
description: Read when net48 code packs or unpacks GZip (System.IO.Compression.GZipStream) and must detect damaged data or handle empty input.
---

# GZipStream in .NET Framework

Measured with .NET Framework 4.8 (`System.IO.Compression.GZipStream`), unit tests.

| Case | Behavior |
|---|---|
| Changed byte in the data, the CRC32 or the length field | `InvalidDataException` on read |
| Data cut off (end missing: 1, 3, 8 or 20 bytes) | no error; the text just ends early |
| Write of an empty input (no `Write` with data) | 0 bytes output: no header, not valid GZip |
| Input packed with another `CompressionLevel` | reads fine (level is not part of the format) |

## Detect cut-off data

The last 4 bytes of a GZip member are the length of the unpacked data (mod 2^32, little-endian). Compare after reading:

```csharp
if (packed.Length < 18 || BitConverter.ToUInt32(packed, packed.Length - 4) != (uint)unpacked.Length)
    throw new InvalidDataException("Cut off.");
```

- 18 = 10 header + 8 trailer bytes: anything shorter has no trailer (the smallest real member has 20, see below).
- Assumes one GZip member, as `GZipStream` writes. Wrong for concatenated members.

## Empty input

Write the standard GZip of an empty text instead of the empty output; any decoder (also `GZipStream`) reads it:

```csharp
static readonly byte[] EmptyGZip = { 0x1F, 0x8B, 0x08, 0, 0, 0, 0, 0, 0, 0x0B, 0x03, 0, 0, 0, 0, 0, 0, 0, 0, 0 };
// packed.Length > 0 ? packed.ToArray() : EmptyGZip
```

## Limit the unpacked size

`GZipStream` has no size limit: a few KB can unpack to gigabytes. Read in a loop and stop above your cap before writing the next buffer.
