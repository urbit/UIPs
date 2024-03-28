---
title: Pierport Protocol
description: Portability protocol for piers and related data
author: ~littul-pocdev
status: Draft
type: Process
created: 2024-03-27
---

## Abstract

This proposal specifies a protocol for pier portability across hosting providers and user custody. The protocol aims to be minimal with extensibility in mind. In addition, the protocol aims to assume as little as possible about the current, or future layout of underlying pier structure.

## Motivation

The core of Urbit's philosophy has the idea that user's ship is portable, and thereby, truly theirs. A pier is intended to contain everything the user needs to either host themselves, or to be able to delegate a hosting provider to host for them. However, at the present moment, the way the pier is intended to be packaged and transported is not standardized, which can cause difficulties in exercising control of their data. This proposal aims to minimize friction and provider lock-in by defining a standard set of procedures hosting environments should follow in order to provide the best user data portability possible.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

This specification defines a REST API that allows pier holders to transfer ownership of the pier and related data to a different entity that SHOULD then exercise best practices in securing the pier.

### Import REST API Endpoint

`pierport/v1/import/~<patp>`, `POST` - uploads a pier. Upon success, returns either HTTP ACCEPTED code with request ID, or HTTP OK, indicating full import completion.

#### Headers

- `Content-Type`
  - `application/zip`
    - Specifies that the incoming stream is a ZIP file.
  - `application/zstd`
    - Specifies that the incoming stream is a zstandard compressed TAR archive. This is the RECOMMENDED method for archiving the pier for migration.
  - `application/json`
    - Specifies that the incoming stream contains a JSON object with a URL to the pier object.
- `Authorization`
  - An opaque token that authorizes a given `<patp>` to be imported to the target entity.
- `Pierport-Size`
  - An OPTIONAL header that hints at uncompressed size of the pier. Receiver MAY choose to ignore it as sender may not be trusted.

##### `Content-Type: application/json`

The sender must provide the JSON object in the following format:

```
{
  "url": "https://example.org",
  "authorization": "<token>" | null,
  "format": "zip" | "zstd",
}
```

The receiver would then issue a `GET` request with appropriate `Authorization` header (if provided) and download the pier that way.

### Archive structure

The pier MUST either have its data laid out at the root of the archive, or under a subdirectory titled the same as the ship's `@p`. For example, in the second case, the layout of `taster-dozzod-dozzod` ship may look as follows:

- `taster-dozzod-dozzod`
  - `.urb`
  - `.run`
  - `.bin`
  - `.prt`

The protocol implementer MAY choose to filter out any unexpected data in the pier. However, the protocol specifies explicit behavior of the following root entries:

- `.urb` SHALL contain the internal data of the pier. Implementer MUST not modify it in unexpected ways that break the layout expected by the reference implementation urbit runtime (currently Vere). The sender MUST provide the data in the format expected by the reference implementation Urbit runtime.

- `.run` is OPTIONAL, but RECOMMENDED, and SHALL contain the urbit runtime used to run the ship. Sender SHALL attach an official release build of the runtime, and if it is not the case, the receiver MAY choose to reject the pier. If `.run` is not provided in the archive, the receiver MAY assume the latest Urbit runtime (which may not be compatible with the pier and fail), or MAY use `.urb` directory awareness to parse the expected urbit runtime by the pier. If the runtime is provided, receiver SHOULD verify that it is an officially released runtime.

- `.bin` is OPTIONAL. It MAY contain the ship's `pace` and previously used urbit runtimes. The receiver MAY choose to ignore the runtimes, and MAY reject the pier if the `pace` is not set to `once`. The receiver MAY choose to change the pier's `pace` in accordance with their internal practices. 

- `.prt` is OPTIONAL. It MAY contain additional pierport extensions, as well as an `info.json` file, that contains additional metadata about the pier.

#### `.prt/info.json`

This file, under `.prt` subdirectory, MAY contain additional metadata about the pier. The receiver MUST preserve all fields throughout the period of pier possession, unless it is explicitly aware of them.

The json structure MAY contain the following fields, but SHALL NOT be limited to:

- `keys` - SHALL be a dictionary containing various keys associated with the Urbit identity. Currently, only `masterticket` is defined.
  - `masterticket`- SHALL be the ownership key for the identity of the ship in the `@p` form. Pier holder MAY remove the field if the pier owner chooses to export it into their possession, without the pier. This protocol does not specify export procedures, but all pier holders SHALL ensure the key is not lost, until the owner confirmed self-custody of the key.
- `ext` is a dictionary that MAY contain provider-specific extensions. These are not standardized, however, each extension MUST be named in the following format: `x-<provider>-<extension>`, ex.: `x-redhorizon-example`.

#### `.prt/s3`

This directory is OPTIONAL, and MAY contain user's S3 state. Receivers MAY ignore it, but the behavior MUST be consistent with exposed capabilities.

#### `.prt/x-<provider>-<extension>`

Provider specific extensions. Receiver MAY choose to ignore them, but they MAY choose to indicate support for in exposed capabilities.

#### `.prt/<other>`

Currently unspecified extensions. Receiver MAY ignore them, but behavior MUST be consistent with extensions exposed through capabilities endpoint.

### Session status REST API Endpoint

`pierport/v1/import/~<patp>/<id>`, `GET` - queries status of the import.

#### Return data

The endpoint MUST return a json with only one of the following keys:

```
{
  "done": {},
  "failed": {
    "reason": "<reason>" | null,
  },
  "importing": {
    "status": "<status>" | null,
  }
}
```

A complete (failed or done) session MUST be retained for at least a period of 60 seconds. This is to allow the sender to poll the status efficiently.

### Capabilities REST API Endpoint

`pierport/v1/capabilities`, `GET` - queries the OPTIONAL capabilities the receiver supports. Returns JSON.

#### Return data

The protocol implementer MUST return a JSON object with an `extensions` key that contains an array of strings. This MAY be extended with additional fields.

```
{
  "info": ["keys/masterticket", "info/ext/provider-example"],
  "extensions": ["s3", "x-provider-example"],
}
```

- `info` specifies what keys in `info.json` are understood by the receiver. This MAY be used as a hint for the capabilities receiver exposes, ex.: masterticket export.
- `extensions` specify what subdirectories under `.prt` the import destination is aware of. This enables exporters to handle cases where the destination does not support the same feature set as the source.

## Rationale

We chose zstd tar archive as RECOMMENDED format, because supporting such archives on Unix systems is trivial, and data compression performance is superior to DEFLATE, gzip, and LZMA. Recommending this as the industry standard will provide faster imports with smaller resource usage. However, zip is still supported, because it currently is common practice for user data to be exported in this format, and it would be detrimental from user experience standpoint to not be able to import existing pier archives. ZSTD zip was considered, however, it currently provides the worst of both worlds - archiving with zstandard is not yet widely supported on major platforms. In addition, most zip archives cannot be decompressed from a stream, while tar can always be, meaning, less disk storage is needed on import destinations.

JSON payload is supported as a way to aid import sources to provide migration from cloud environments. For instance, source hosting provider may migrate a pier to destination hosting provider, by providing a signed S3 URL to the destination provider. This avoids a unnecessary copy.

We choose to support both archive file layouts, because they are the common ways piers are currently archived.

We include the S3 extension in the standard, because the current Urbit application ecosystem heavily relies on S3 to share data across the network. Of course, an S3 bucket is not truly portable without its associated domain, however, the data on the S3 bucket still belongs to the user; therefore, it is important to expose a method to full data ownership.

We include the masterticket in `info.json`, because inexperienced users may not be concerned with management of their cryptographic keys. It would be detrimental to the user experience to force self-custody of identity upon migration between hosting providers. Inclusion of masterticket is meant to provide flexibility, without sacrificing ability to switch to self-custody, should the user wish to do that.

We include the `~<patp>` in the REST API to enable for early exit in case of duplicate imports, and simplify protocol implementation in internal, trusted infrastructure. The pier does contain the underlying urbit ID number inside the event log, and a secure pierport implementation would verify it, however, the protocol attempts to assume as little as possible about the `.urb` subdirectory, and requiring verification would force it to assume current format of event log storage (LMDB over several epochs).

Mounted desks are merely mirrors of underlying clay state, therefore, protocol implementors MAY choose to not unpack the desk directories. This may lead to inconsistent view (mounted desk without corresponding filesystem entry), but the Urbit runtime should be able to handle it.

Sessions only expose polling for simplicity sake. In the future, the protocol may be extended with long-polling, however, it was not added, because it is unclear if it is necessary.

## Reference Implementation

_Reference implementation to be linked later._

## Security Considerations

The primary security consideration is authenticity and freshness of the pier. Uploading an out-of-date pier may lead to necessity of breaching, and associating a rogue pier (a fakezod masqueraded as real ship) may lead to impersonation on earth side, as well as spam. Ideally, there should be a cryptographic verification step (ex.: a signed message on Mars-side, or a signed message using the Urbit identity) in order to prevent such security issues. However, the current protocol design does not prevent such provisions from being implemented in future iterations, therefore, we did not feel the necessity to solve it immediately.

Needs further discussion.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
