---
title: Book Event Log Persistence
description: A custom append-only event log persistence layer optimized for Urbit's sequential access patterns.
author: Matthew LeVan <matthew@coeli.network>
status: Draft
type: Standards Track
category: Kernel
created: 2026-01-20
---

## Abstract

This UIP specifies the Book API, a custom append-only event log persistence layer designed to replace LMDB for storing Urbit's event log. Book provides a simple file-based storage format optimized for sequential writes and reads, with built-in crash recovery through CRC32 checksums and length trailers. The format consists of two files: `book.log` for event data and `meta.bin` for ship metadata.

## Motivation

Urbit's event log has a strictly append-only access pattern that does not benefit from LMDB's general-purpose features:

1. **Unnecessary overhead**: LMDB's B+tree structure, random access capabilities, and transaction semantics add complexity without benefit for sequential event logs.

2. **Storage inefficiency**: LMDB's architecture prevents log file size reduction and caps maximum value sizes at approximately 4GB.

3. **Performance**: A purpose-built sequential storage layer can achieve faster write speeds by eliminating the overhead of maintaining B+tree indices.

4. **Owned code**: Book is ~1000 lines of our own code whereas LMDB is a third-party dependency.

Book addresses these limitations with a minimal file format specifically designed for append-only event storage with O(1) appends and O(n) sequential reads.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### File Structure

Book uses two files stored in the pier's data directory:

- `book.log`: Contains the event log header followed by event records (deeds)
- `meta.bin`: Contains fixed-size ship metadata (256 bytes)

### Header Format

The `book.log` file MUST begin with a 16-byte header:

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 4 bytes | `mag_w` | Magic number: `0x424F4F4B` ("BOOK") |
| 4 | 4 bytes | `ver_w` | Format version: `1` |
| 8 | 8 bytes | `fir_d` | First event number in file |

The header is immutable after creation, except for `fir_d` which is set once when the first event is written.

Implementations MUST reject files where:
- `mag_w` does not equal `0x424F4F4B`
- `ver_w` does not equal `1`
- File size is less than 16 bytes

### Deed Format

Each event is stored as a "deed" with the following structure:

**Deed Header (12 bytes):**

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 8 bytes | `len_d` | Total payload size (mug + jam data) |
| 8 | 4 bytes | `mug_l` | Event mug/hash |

**Payload:**

| Size | Field | Description |
|------|-------|-------------|
| `len_d - 4` bytes | `jam_y` | Jammed event data |

**Deed Tail (12 bytes):**

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 4 bytes | `crc_w` | CRC32 checksum over header + payload |
| 4 | 8 bytes | `let_d` | Length trailer (MUST equal `len_d`) |

The total on-disk size of a deed is: `12 + (len_d - 4) + 12 = len_d + 20` bytes.

### Metadata Format

The `meta.bin` file contains a fixed 256-byte structure:

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 16 bytes | `who_d` | Ship identity (as `c3_d[2]`) |
| 16 | 4 bytes | `ver_w` | Metadata format version |
| 20 | 4 bytes | `lif_w` | Lifecycle length |
| 24 | 1 byte | `fak_o` | Fake security flag |
| 25 | 231 bytes | `pad_y` | Reserved for future use |

Implementations MUST support the following metadata keys:
- `"version"`: 4 bytes
- `"who"`: 16 bytes
- `"fake"`: 1 byte
- `"life"`: 4 bytes

### CRC32 Computation

The CRC32 checksum MUST be computed over the concatenation of:
1. The 8-byte `len_d` field
2. The 4-byte `mug_l` field
3. The `jam_y` payload bytes

Implementations MUST use the standard CRC32 algorithm (as provided by zlib).

### Event Contiguity

Events MUST be stored contiguously without gaps:

1. For an empty log, the first event number MUST equal `epoch + 1`, where epoch is the last committed event from a previous epoch (or 0 for a new ship).

2. For a non-empty log, each new event number MUST equal `last_event + 1`.

Implementations MUST reject writes that violate contiguity.

### Crash Recovery

On initialization, implementations MUST validate the event log and recover from crashes:

1. **Reverse Scan**: Starting from file end, read `let_d` trailers to locate the last valid deed. Validate each deed's CRC and `len_d == let_d`.

2. **Forward Scan (Fallback)**: If reverse scan fails, scan forward from header, validating each deed.

3. **Truncation**: Remove any bytes after the last valid deed and update cached state.

The dual-scan approach enables O(1) recovery for clean shutdowns while handling corruption gracefully.

### API Functions

Implementations MUST provide the following operations:

#### Lifecycle

- `u3_book_init(path)`: Open or create event log at path. Returns handle or NULL on failure.
- `u3_book_exit(book)`: Close event log and free resources.

#### Event Operations

- `u3_book_gulf(book, &low, &high)`: Read first and last event numbers.
- `u3_book_save(book, eve, len, buffers, sizes, epoch)`: Save `len` events starting at `eve`. Events are provided as arrays of mug+jam buffers.
- `u3_book_read(book, ctx, eve, len, callback)`: Read `len` events starting at `eve`, invoking callback for each.

#### Iterator

- `u3_book_walk_init(book, &iter, start, end)`: Initialize iterator over event range [start, end].
- `u3_book_walk_next(&iter, &len, &buf)`: Read next event. Returns `c3y` on success, `c3n` at end or error.
- `u3_book_walk_done(&iter)`: Invalidate iterator.

#### Metadata

- `u3_book_read_meta(book, ctx, key, callback)`: Read metadata field by key.
- `u3_book_save_meta(book, key, size, value)`: Write metadata field.

### Thread Safety

Implementations SHOULD use `pread`/`pwrite` syscalls for stateless I/O operations. Multiple readers MAY access the same book concurrently, but writers MUST be serialized externally.

### Durability

Implementations MUST call `fsync` (or equivalent) after:
1. Writing the header's `fir_d` field
2. Writing event batches
3. Writing metadata

## Rationale

TBD

## Backwards Compatibility

This specification introduces a new storage format that is not backwards compatible with LMDB. Migration from LMDB to Book requires:

1. Reading all events from LMDB
2. Writing them to a new Book file
3. Copying metadata to `meta.bin`

Implementations SHOULD provide migration tooling to automate this process.

## Reference Implementation

The reference implementation is available in the urbit/vere repository:
- [pkg/vere/db/book.h](https://github.com/urbit/vere/blob/ml/book/pkg/vere/db/book.h)
- [pkg/vere/db/book.c](https://github.com/urbit/vere/blob/ml/book/pkg/vere/db/book.c)

## Security Considerations

1. **Data Integrity**: CRC32 checksums detect accidental corruption but do not provide cryptographic integrity. The mug hash embedded in each event provides additional validation at the Urbit layer.

2. **File Permissions**: Book files SHOULD be created with mode 0644, restricting write access to the owner.

3. **Denial of Service**: Implementations MUST validate `let_d` values during recovery to prevent excessive memory allocation from malformed files. Values exceeding 4GB (`1ULL << 32`) SHOULD be rejected.

4. **Path Traversal**: Implementations MUST validate that paths do not escape the intended directory.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
