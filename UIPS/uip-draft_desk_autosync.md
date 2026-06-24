---
title: Per-File Desk Auto-Sync
description: Event-driven, per-file synchronization between mounted desks and the host filesystem, with content-addressed reconciliation and conflict notification.
author: ~mopfel-winrux
status: Draft
type: Standards Track
category: Kernel
created: 2026-06-10
---

## Abstract

This proposal adds a per-desk auto-sync mode to %clay and the runtime's unix
driver. A mount point marked for auto-sync is watched by the runtime using
native filesystem events (inotify, FSEvents) instead of timer-driven polling.
When a file changes on the host filesystem, the runtime injects a `%into`
event containing only that file (coalescing changes that occur within a short
debounce window); when a file changes in %clay, the existing `%ergo` flow
already writes only that file to disk. A new %clay task (`%wath`) marks a
mount point for auto-sync, and new gifts (`%wath`, `%wend`) instruct the
runtime to start or stop watching.

Synchronization state is content-addressed using %clay's existing `lobe`
(a 256-bit `shax` over a file's stored blob) and the recursive directory
hash already exposed by the `%cz` care. The runtime never reconstructs
%clay's representation from disk: it tracks an *earth-local* content hash to
decide when a file has changed, and stores the *mars-issued* `lobe` opaquely
to reconcile against %clay. This split makes reconciliation independent of
whether a file's mark round-trips byte-for-byte, lets the post-restart scan
descend only divergent subtrees of the desk's merkle tree, and gives every
inbound write a compare-and-swap base so that genuine concurrent edits are
detected rather than silently resolved. Detected conflicts are surfaced to
the operator and the kernel instead of being applied silently.

The proposal also specifies that runtimes must not inject empty sync events,
eliminating the two main sources of event-log bloat in the current
`|autocommit` mechanism.

## Motivation

Synchronization between %clay and the host filesystem is already per-file in
principle: `%info` commits are lists of `[path miso]` deltas, the `%into`
task carries `(list [path (unit mime)])`, and `%ergo` gifts carry only the
files changed by a commit. In practice, several defects make continuous
syncing impractical or unsafe:

1. **Restart amplification.** The unix driver detects changes by comparing a
   per-file hash held only in memory. After a restart the watch tree is
   rebuilt with zeroed hashes, so the first commit after every boot injects a
   `%into` event containing the entire contents of the desk. %clay correctly
   filters unchanged files against its mime cache, but by then the full desk
   has already been written into the event log as that event's payload.

2. **Idle-churn amplification.** `|autocommit` is a one-second %behn timer
   loop in %hood/kiln. Every tick produces a `%wake` event, a `%dirk` round
   trip, a full rescan (read + hash of every file in the mount), and — because
   the unix driver injects the `%into` event unconditionally, even when the
   change list is null — one empty `%into` event. An idle ship with one
   auto-committed desk accretes on the order of 170,000 junk events per day.

3. **Weak change-detection hash.** The unix driver fingerprints files with a
   31-bit `mug`. A changed file collides with its predecessor with
   probability ~2⁻³¹, and on collision the change is *never* detected and
   never synced — the worst possible failure mode (a silently dropped edit).
   The same mug was additionally computed over different representations on
   the two sides of a comparison (a noun mug versus a byte mug), so files
   whose contents end in zero bytes were re-sent on every scan, forever.

4. **Silent conflict resolution.** When a file is edited on disk *and* in the
   desk between syncs, the runtime resolves the race silently — it keeps the
   disk version and supersedes the desk revision — with no signal to the
   operator, the kernel, or any agent. The policy is defensible (see
   *Reconciliation semantics*), but its silence is not: a user can lose track
   of which side of an edit won, and automated tooling has no way to react.

The result is that users who want live syncing between an editor on Earth and
a desk on Mars either suffer unbounded event-log growth or must manually
`|commit` after every save, and in either case races resolve invisibly. This
proposal makes continuous bidirectional syncing event-driven, incremental,
quiescent when idle, robust against weak-hash collisions, and explicit about
conflicts.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119 and RFC 8174.

### Content addressing: the shared currency

The reconciliation protocol is built on two distinct hashes, and keeping them
distinct is the central design decision.

- **Earth-local content hash.** A hash the runtime computes over a file's
  *bytes on disk*. It is used only to answer "did this file change on disk
  since the runtime last handled it?" — a comparison the runtime always makes
  against its own prior value. It is never compared against any value computed
  by %clay. Runtimes MUST compute it with a collision-resistant hash (e.g.
  SHA-256) over the file's byte string at its actual length, replacing the
  31-bit mug. Because it is purely runtime-local, its concrete algorithm is
  unconstrained, provided it is collision-resistant and stable.

- **Mars token (`lobe`).** %clay already content-addresses every file with a
  `lobe`: a 256-bit `shax` over the file's stored blob (`+$ lobe @uvI`). The
  runtime treats the `lobe` as an *opaque token issued by Mars*. It stores the
  `lobe` %clay reports for each synced file and echoes it back when proposing
  a write, but never computes a `lobe` itself. Two replicas therefore never
  need to agree on a byte representation — Earth compares earth-hashes to
  earth-hashes, Mars compares lobes to lobes, and the runtime only ever
  carries Mars's own tokens across the boundary.

This division is what makes the protocol indifferent to mark fidelity. A mark
that is lossy over raw bytes (notably `wain`-backed text marks, which cannot
represent a trailing newline) causes a file's on-disk form to differ from its
canonical desk form. Under a scheme that required Earth to reproduce Mars's
hash from disk, every such file would mismatch forever. Under this scheme the
mismatch never arises: the runtime fingerprints the disk bytes for change
detection and carries Mars's `lobe` for reconciliation, and the two are
independent.

%clay additionally exposes a recursive directory hash through the existing
`%cz` care (`+content-hash`): for any path, `shax` is folded over the sorted
`[sub-path lobe]` pairs of that path and all its descendants, yielding a
single `@uvI`. `.^(@uvI %cz /=desk=/)` is therefore the desk's **merkle root**,
`.^(@uvI %cz /=desk=/some/dir)` is any **subtree hash**, and a leaf's hash is
`shax(jam [~ lobe])`. The protocol reuses this directly; no new merkle
structure is introduced into %clay.

### New %clay task

One task is added to `$task:clay` in `lull.hoon`:

```hoon
[%wath pot=term on=?]                    ::  (un)mark for auto-sync
```

`pot` names an existing mount point (as created by `%mont`). `on=&` marks the
mount point for auto-sync; `on=|` unmarks it. %clay MUST crash on a `%wath`
task naming a mount point that does not exist.

A second task carries an earth-detected conflict report from the runtime into
the kernel (working mote `%dare`; the mote is not load-bearing):

```hoon
[%dare pot=term pax=path]                ::  runtime-detected sync conflict
```

`pax` is the mount-relative path on which the runtime declined to overwrite a
locally-edited file with an inbound `%ergo`. On receiving `%dare`, %clay MUST
surface a conflict notification (see *Conflict notification*) and MUST NOT
otherwise alter desk state.

### Augmented sync payloads

Two existing structures in `lull.hoon` gain content-address fields. Both are
within the `lull` kelvin surface already broken by `%wath`/`%wend`.

The `%ergo` gift carries, per file, the `lobe` %clay holds for it, so the
runtime can record the mars token alongside the bytes it writes:

```hoon
[%ergo p=term q=(list [pax=path mim=(unit mime) lob=lobe])]
```

The `%into` task gains a per-file compare-and-swap base — the `lobe` the
runtime believes Mars currently holds for the path, drawn from the last
`%ergo` it applied:

```hoon
[%into des=desk all=? fis=(list [pax=path bas=(unit lobe) mim=(unit mime)])]
```

`bas` is `~` when the runtime has no recorded token for the path (e.g. a file
created on disk, or after a degraded restart). A `~` base disables the CAS
check for that entry, which then applies as today.

The `%wath` gift carries the desk's current merkle root, letting the runtime
decide at reconciliation whether any descent is needed at all:

```hoon
[%wath pot=term rut=(unit @uvI)]         ::  start watching; rut is %cz root
```

### New %clay gifts

The watch-control gifts are emitted on the unix duct (`hez`) alongside the
existing `%dirk`, `%ergo`, `%hill`, and `%ogre`:

```hoon
[%wath pot=term rut=(unit @uvI)]         ::  start watching mount point
[%wend pot=term]                         ::  stop watching mount point
```

The conflict notification is surfaced to %hood/kiln (the mechanism mirrors
how mount notifications already reach %hood) so that it reaches the dojo and
any subscribed agent:

```hoon
[%dare pot=term pax=path won=?(%earth %mars) old=(unit aeon)]
```

`won` records which side's content was kept (`%earth` = the disk edit was
retained and the desk revision superseded; `%mars` = a desk commit landed
under a stale runtime write). `old`, when present, names the aeon at which the
superseded content remains retrievable.

### %clay state and behavior

%clay records the set of auto-synced mount points (e.g. `syn=(set term)`
alongside `mon`; the concrete representation is unconstrained). %clay does not
persist any hash state for auto-sync: all content addresses are derived on
demand from the commit graph (`lobe`s in `q.yaki`, roots via `+content-hash`).

- On `%wath` with `on=&`, %clay MUST add the mount point to the set and give
  `[%wath pot rut]` on the unix duct, where `rut` is the `%cz` root of the
  mount's desk at its mounted case. With `on=|` it MUST remove it and give
  `[%wend pot]`.
- On unmounting (`%ogre` flow), %clay MUST remove the mount point from the
  set. The runtime discards its watchers with the mount point, so no `%wend`
  is required.
- At runtime initialization (the `%boat` task, after giving `%hill`), %clay
  MUST re-give `[%wath pot rut]` for every auto-synced mount point, so that
  watchers are re-armed and the runtime can reconcile after a restart or
  runtime upgrade.
- After giving `%wath`, %clay MUST make the mount self-healing. The mechanism
  is the runtime's, driven by the root and by content reads (below): files
  missing on disk are restored from the desk, unchanged files are untouched,
  and files edited on disk are left for the inbound sync. %clay MAY continue to
  give a full `%ergo` mirror as a fallback, but SHOULD rely on the runtime's
  root comparison to avoid mirroring an already-synchronized desk. Since gifts
  are not persisted, neither path adds to the event log.

The `%ergo` production path MUST attach each file's current `lobe` (from the
committing `yaki`'s `q` map) to the gift. The `%into` handling path filters
unchanged files against the mime cache exactly as today, then MUST apply the
CAS check below before committing.

#### Compare-and-swap on `%into`

For each entry `[pax bas mim]` in an incoming `%into` whose `bas` is non-null,
%clay MUST compare `bas` against the `lobe` it currently holds for `pax` (the
value in the head commit's namespace map, or `~` if `pax` is absent):

- **Match** — the runtime based its edit on the revision %clay still holds.
  Commit normally (a clean fast-forward).
- **Mismatch** — a desk commit landed on `pax` between the runtime's last
  read and this write. %clay MUST still apply the write (the policy is
  unchanged: the live edit on disk expresses intent), and MUST additionally
  emit a `%dare` notification with `won=%mars` and `old` set to the aeon at
  which the superseded content is retained.

The CAS check changes no commit outcome; it only converts a previously
invisible concurrent modification into an explicit notification.

### Runtime (unix driver) behavior

On receiving a `%wath` effect for a mount point, the runtime MUST begin
watching that mount point's directory tree using native filesystem-event
facilities where available (e.g. inotify on Linux, FSEvents on macOS). On
receiving `%wend`, it MUST stop. A runtime without filesystem-event support
MAY approximate watching by polling, but the observable event-injection
behavior below still applies.

#### Per-file state

For each synced file the runtime maintains, and SHOULD persist across
restarts (see *Persistence*): the **earth-local content hash** of the bytes
last known to be synchronized, and the **mars token** (`lobe`) %clay last
reported for the file. The earth-local hash drives change detection; the
token is echoed as the CAS base on outbound `%into` and used to reconstruct
subtree hashes during reconciliation.

#### Change detection and injection

While a mount point is watched:

- On notification of filesystem changes, the runtime MUST rescan only the
  affected subtree(s), and MUST inject a single `%into` event (with `all=|`)
  containing only files whose earth-local content hash differs from the value
  last known to be synchronized. Each entry carries the file's recorded mars
  token as its CAS base (`~` if none is recorded).
- The runtime SHOULD debounce until quiescence: each observed change SHOULD
  extend the coalescing window (RECOMMENDED order of 100ms of silence),
  bounded by a maximum delay from the first change (RECOMMENDED order of 1s)
  so that a continuously-writing process cannot starve synchronization. This
  batches multi-step editor save sequences and multi-file saves into one
  `%into` event.
- The runtime SHOULD NOT sync a deletion the first time a file is observed
  missing. A missing file SHOULD be rechecked after a grace period
  (RECOMMENDED order of 300ms) and its deletion synced only if it is still
  absent. Editors commonly delete or rename a file away moments before
  rewriting it; without a grace period, such saves propagate a transient
  deletion into the desk's revision history, with visible side effects.
- The runtime SHOULD periodically rescan watched mount points as a backstop
  (RECOMMENDED order of 30s), since filesystem notification mechanisms can
  drop events (e.g. inotify queue overflow or exhausted watch descriptors). A
  backstop rescan of an unchanged tree produces no event, per the requirement
  below.
- The runtime MUST NOT inject a `%into` event whose change list is null.
  (This requirement applies to the `%dirk`-triggered scan path as well, and
  is a behavioral fix independent of auto-sync.)
- A watched mount point remains subject to ordinary `%dirk` commits; `|commit`
  continues to work unchanged.
- Symbolic links are followed, exactly as in ordinary mounts and commits.
  Note that a change to a link's target outside any watched directory
  generates no filesystem event, and so syncs on the next commit or scan
  rather than immediately.

#### Applying `%ergo`

When applying an `%ergo`, for each file the runtime MUST record the
accompanying `lobe` as that file's mars token, and MUST update the file's
earth-local content hash to the hash of the bytes it writes (or, when it
declines to write, to a value that keeps the disk edit visible as a change —
see below). The write decision compares the bytes on disk against the bytes
last known to be synchronized:

- Disk matches the last-synchronized content → overwrite (or skip if the
  `%ergo` payload is byte-identical to disk). Record the new token and hash.
- Disk is absent → write the file. Record the token and hash.
- Disk differs from the last-synchronized content → the file was edited on
  disk. The runtime MUST NOT overwrite it; the edit syncs inward instead, and
  the runtime MUST record the mars token from the `%ergo` and set the
  earth-local hash so the disk edit is still seen as a pending change. Whether
  this is a *conflict* depends on whether a sync baseline existed:
  - If the runtime had **no** earth-local hash recorded for the file — a
    first-sight file (first mount, or after lost/absent persistence) — this is
    **not** a conflict: with no baseline, "changed on both sides" is not
    meaningful. The runtime keeps the disk content and MUST NOT emit `%dare`.
    During the initial mount this is the common case for every file whose
    on-disk form differs from its mark's canonical form; emitting `%dare` here
    would flood spurious conflicts.
  - If the runtime **had** a recorded baseline and the disk diverged from it,
    this is a genuine **concurrent edit** (changed on disk AND in the desk):
    the runtime MUST emit a `%dare` task to %clay naming the path, and SHOULD
    log it locally (e.g. via `u3l_log`) so terminal-only operators see it.

#### Reconciliation (restart and `%wath`)

The runtime does **not** recompute the desk's content hashes. It treats both
the per-file `lobe` and the `%cz` directory hash as **opaque tokens issued by
%clay**: it caches the values %clay reports — the `lobe` carried on `%ergo`,
the root `rut` carried on `%wath`, and subtree `%cz` hashes peeked during a
descent — and compares cached against current, never deriving them itself.
(The runtime cannot reproduce a `%cz` hash: `+content-hash` folds it in
`~(tap by …)` — mug-ordered treap — order, which is not reproducible from a
flat token set in the runtime.)

At reconciliation the runtime receives the desk's current root `rut` in the
`%wath` gift and compares it against the last root it cached for the mount:

- **Equal** — Mars did not change while the runtime was not watching. The
  runtime performs only an earth-side rescan: files whose disk bytes differ
  from their recorded earth-local hash are injected as `%into` (offline
  edits). A file the runtime has a record of but which is **missing on disk**
  MUST be restored from the desk (read its content via `%cx` and write it),
  never synced as a deletion — at reconciliation the desk is ground truth for
  existence (see *Reconciliation semantics*).
- **Differ** — descend. The runtime peeks the desk's subtree `%cz` hashes and
  compares them against its cached subtree hashes, recursing only into
  divergent subtrees. For each divergent or missing leaf it reads the file's
  content (`%cx`) and current token, and applies it under the same write rules
  as `%ergo` — a file edited on disk while the runtime was down is preserved
  and synced inward; a file changed only on Mars, or missing on disk, is
  written to disk.

A runtime with **no** cached tokens for a mount (first sync, lost or corrupt
persistence) MUST treat the entire tree as divergent and reconcile every file
by the same rules — a full reconciliation, correct but O(desk).

The runtime reads desk content and hashes through the namespace (`u3_pier_peek`
/ `u3_pier_peek_last` with the `%cx` and `%cz` cares), an established runtime
capability. Because reconciliation is content-read-driven, %clay applies marks
authoritatively and the runtime never interprets desk contents.

Restoring files that are missing on disk is a **correctness** requirement of
this design, not an optimization: a mount whose disk copy is damaged (a wiped
directory, a restored backup) MUST converge by restoring the desk's files, not
by propagating their absence into the desk. A runtime MUST NOT let a
reconciliation-time missing file sync inward as a deletion.

As an interim, a runtime MAY instead rely on %clay giving a full `%ergo`
mirror at `%wath`/`%boat` and skip the pull entirely: the mirror's writes
arrive within the deletion grace window, so they restore missing files before
any deletion is synced. This is correct but pays a whole-desk checkout on
every reconciliation. The root-gated pull above replaces it, and any
implementation of the pull MUST preserve the missing-file restoration the
mirror provided.

#### Persistence

Across restarts, the runtime SHOULD persist per synced mount a sidecar mapping
each file to `{earth-local hash, mars token}` (and MAY record the last-seen
root). The sidecar makes the post-boot reconciliation inject only files that
actually changed while the ship was down, and lets the merkle comparison
descend only divergent subtrees rather than full-mirror the desk.

The sidecar is advisory. A missing, stale, or corrupt sidecar MUST degrade to
the no-tokens behavior (full reconciliation against the desk root), never to
an incorrect commit. Runtimes MUST validate sidecar structure on load and
discard it wholesale on any parse failure. Runtimes that cannot persist fall
back to full reconciliation after every boot — degraded but correct, since
%clay's mime-cache filter and the CAS check still yield accurate commits.

### Conflict notification

A conflict is any reconciliation in which both replicas changed a file
independently. Detection happens at two sites, and both MUST surface a
notification rather than resolve silently:

- **Earth-side (runtime).** In the `%ergo`/reconciliation write path, a file
  the runtime *had previously synced* (a baseline exists) whose disk bytes now
  differ from that baseline while Mars also changed it: the runtime keeps the
  disk edit and emits `%dare` (`won=%earth`). A first-sight file with no
  baseline is not a conflict (see *Applying `%ergo`*) and emits nothing.
- **Mars-side (%clay).** In `%into` handling, a CAS base that does not match
  %clay's current `lobe`: %clay applies the write and emits `%dare`
  (`won=%mars`, `old` naming the superseded aeon).

%hood/kiln surfaces `%dare` to the dojo, e.g.:

```
clay: %base sync conflict /lib/foo/hoon: disk version committed at 43,
      desk version retained at 42  (recover: +cat /~bus/base/42/lib/foo/hoon)
```

The notification is informational; it changes no commit outcome. Its purpose
is to make the existing, deliberate resolution policy *visible*.

### User interface

%hood/kiln gains a poke (e.g. `%kiln-autosync`) passing `[%wath pot on]` to
%clay, exposed via generators:

```
|autosync %desk           ::  mark a mounted desk for auto-sync
|cancel-autosync %desk    ::  unmark it
```

Conflict notifications print to the dojo as shown above; no generator is
required to receive them.

### Latency

Auto-sync makes a file change observable in %clay one commit later, so its
latency floor is the cost of a %clay commit; the watcher machinery adds only
the debounce window plus single-digit milliseconds of runtime work. Profiling
one-file commits on a development ship showed the commit cost is dominated by
the agent and mark rebuilds that `+park` performs via `+goad` before giving
%gall its `%load`: about 4.4s of a 4.9s commit in the measured configuration
(`+build-agents` ~2.5s, `+build-marks` ~1.9s), independent of what the commit
touched. This cost is a known issue with `+goad`; urbit/urbit#7353 scopes the
rebuild to the desk that changed rather than all live desks. A commit to a
non-live desk skips the rebuild entirely and was measured at roughly 10ms of
vane time (~180ms end to end, most of which is the debounce window).

Reconciliation latency benefits directly from the merkle root: an
already-synchronized desk reconciles in one root comparison and zero content
reads, where the previous design checked out and `%ergo`'d the whole desk.

## Rationale

**Why content-address with `lobe`/`%cz` rather than a new hash?** %clay
already computes a 256-bit `lobe` for every file and a recursive `%cz` tree
hash over those lobes, and already exposes both through the namespace. Reusing
them means %clay grows no new hashing machinery, no new scry care, and no new
persisted state for auto-sync; the kernel change reduces to attaching a token
to `%ergo`, a base to `%into`, a root to `%wath`, and a CAS comparison plus a
notification in the `%into` path.

**Why separate the earth-local hash from the mars token?** Because the two
replicas store files in different representations and need not converge on a
common one. Change detection is inherently a one-sided question ("did *my*
disk change?") and must be answered in the disk representation. Reconciliation
is a cross-replica question, and is answered by carrying Mars's own token
across the boundary rather than asking Earth to recompute it. Collapsing the
two into a single hash is exactly what forced earlier designs to demand
byte-faithful marks; keeping them separate dissolves that requirement. The
weak-hash collision risk (defect 3) is likewise contained: it can only affect
the earth-local change trigger, which is upgraded to a collision-resistant
hash, and it can never cause a cross-replica disagreement because no
cross-replica hash is computed on disk.

**Why compare-and-swap rather than vector clocks?** There are exactly two
replicas, and %clay already totally orders every event through the log. A
two-entry vector clock degenerates to "what revision was this edit based on,"
which is precisely a compare-and-swap on the base token (the idiom of git's
`--force-with-lease` and HTTP's `If-Match`). %clay's mount is a working tree,
not a peer; treating it as git treats the index — a base snapshot plus a
three-way comparison — is both simpler and stronger than peer-to-peer merge
machinery.

**Why notify rather than block or auto-merge?** The resolution policy (disk
wins for live edits; the desk is ground truth for existence) is deliberate and
data-preserving (see below). Blocking would stall the editor; auto-merging
text would fabricate content. Notification keeps the safe policy and removes
only its silence, which is the actual defect. The asymmetry matters and is
reflected in the message: a superseded *Mars* revision is never destroyed
(%clay retains every aeon), whereas a clobbered *Earth* edit has no history —
which is exactly why the policy keeps the disk version, and why making that
choice visible is worthwhile.

**Why filesystem events rather than faster polling?** Polling couples cost to
desk size and poll frequency rather than to change volume; `|autocommit`
demonstrates the failure mode. Filesystem events make idle cost zero in both
CPU and event-log terms, and libuv (already the runtime's event loop) wraps
the platform facilities portably.

**Why is persistence a SHOULD?** It is a runtime-local optimization invisible
to Arvo: with or without it, %clay computes the same commits. The merkle root
delivered in `%wath` lets even a persistence-less runtime skip a full mirror
only when it has tokens to reconstruct its side; without tokens it degrades to
full reconciliation. Mandating a particular on-disk format would overconstrain
other runtimes (e.g. Ares) without an interoperability benefit.

**Relation to `|autocommit`.** `|autocommit` is left intact but becomes
redundant for auto-synced desks. It MAY be deprecated in a later proposal once
auto-sync has shipped.

## Backwards Compatibility

The new task, gifts, and payload fields extend `lull.hoon` and therefore
require a kelvin decrement to ship. They are bundled into the same kelvin
event as `%wath`/`%wend`.

Mismatched pairs degrade gracefully:

- An updated Arvo on an older runtime: the unix driver ignores effect tags and
  payload fields it does not recognize, so `%wath`/`%wend` are dropped, the
  augmented `%ergo`/`%wath` fields are ignored, and the desk simply is not
  auto-synced. `|commit` and `|autocommit` behave as today.
- Older Arvo on an updated runtime: stock %clay never gives `%wath`, so no
  watchers are created and behavior is unchanged. The empty-`%into`
  suppression and the collision-resistant change hash apply regardless and are
  strictly beneficial.

The persisted sidecar is advisory: a missing or stale sidecar MUST degrade to
full reconciliation, never to an incorrect commit. A runtime that does not
implement the CAS base simply sends `bas=~`, disabling conflict detection
while leaving sync correct.

## Open Questions

**Should `wain`-backed text marks move to a byte-faithful representation?**

This proposal removes the *correctness* stake in this question: because the
runtime never reconstructs Mars's representation from disk, a lossy mark no
longer causes perpetual mismatch, re-sends, or false conflicts. What remains
is purely cosmetic. A `%txt` file saved with a trailing newline converges in
one round trip to the mark's canonical (newline-stripped) form; editors with
the file open observe one external modification per non-canonical save. The
CAS check does *not* fire on this rewrite, because it compares Mars tokens to
Mars tokens, not disk bytes to canonical bytes.

`%hoon` does not exhibit even the cosmetic rewrite: its noun form is a cord
(`++noun @t`, with `+mime` passing octets through), so `.hoon` files
round-trip byte-for-byte, and its line-based diffing is recovered by
delegating `+grad` to `%txt`. The same repair would remove the cosmetic
rewrite for `%txt` and other `wain`-backed marks: store a cord, convert to
`wain` only for diffing and display. The trade-off is migration (every
`!<(wain ...)` consumer, existing `%txt` files, and `%txt-diff` history), and
whether it is worth byte-faithful text files is left open here. Either
resolution is compatible with this proposal.

**Mote and notification routing.** The conflict task/gift is specified here
with the working mote `%dare`; the final mote and the exact %clay→%hood
subscription mechanism are implementation details to be settled against the
existing mount-notification plumbing, and do not affect the protocol.

## Reference Implementation

- Arvo: [urbit/urbit#7362](https://github.com/urbit/urbit/pull/7362)
- runtime: [urbit/vere#1031](https://github.com/urbit/vere/pull/1031)

Implemented and tested end-to-end: per-file bidirectional sync, the
collision-resistant earth-local hash, the `lobe`/CAS/`%dare` conflict path
(both directions, surfaced to the dojo via %hood), and empty-`%into`
suppression. The **root-gated reconciliation pull** (the opaque-`%cz` descent
in *Reconciliation*) is **not yet implemented**: reconciliation currently uses
%clay's full `%ergo` mirror at `%wath`/`%boat`, which is correct (and provides
the missing-file restoration described above) but pays a whole-desk checkout.
The pull is the remaining work and MUST preserve that restoration.

The implementation touches:

- `pkg/arvo/sys/lull.hoon` — `%wath`/`%dare` tasks; `%wath`/`%wend`/`%dare`
  gifts; `lobe` on `%ergo`, CAS base on `%into`, root on `%wath`.
- `pkg/arvo/sys/vane/clay.hoon` — auto-sync set in state (+ state version
  bump), `%wath` handling, re-arming with root at `%boat`, cleanup at unmount,
  `lobe` attachment in `+ergo`, CAS comparison and `%dare` emission in `+into`.
- `pkg/arvo/lib/hood/kiln.hoon`, `pkg/arvo/gen/hood/autosync.hoon`,
  `pkg/arvo/gen/hood/cancel-autosync.hoon` — user interface and conflict
  surfacing.
- `pkg/vere/io/unix.c` — `uv_fs_event_t` watchers per directory, debounce
  timer, dirty-subtree rescan, collision-resistant earth-local hashing, the
  `{hash, token}` sidecar persisted in the pier, CAS base population on
  `%into`, conflict detection (baseline-gated) and `%dare` injection, storage
  of the `%wath` root for reconciliation, and suppression of empty `%into`
  events. The root-gated `%cz`/`%cx` reconciliation pull is stubbed (the root
  is stored; the descent is pending).

## Security Considerations

The proposal adds no new namespace exposure: auto-sync moves data only between
a mount point the operator already created with `|mount` and the desk it
mirrors, in the same direction and with the same content as the existing
`%dirk`/`%into`/`%ergo` flow.

- **Event-injection volume.** Filesystem watchers turn host filesystem
  activity into Arvo events. A runaway process writing to a mounted directory
  could inject events at high frequency. The debounce window bounds the event
  rate, and the per-file hash comparison bounds payload size to actual
  changes; runtimes MAY additionally rate-limit injection. This is no worse
  than current `|autocommit` behavior, which injects events at a fixed rate
  regardless of activity.
- **Sidecar integrity.** The persisted `{hash, token}` sidecar, if tampered
  with or corrupted, can cause changed files to be skipped (a stale earth-hash
  collision) or unchanged files to be re-reconciled (token loss). The latter
  is the safe degraded mode specified above; the former requires write access
  to the pier, which already implies full control of the ship. The merkle root
  delivered in `%wath` bounds the former: a tampered sidecar whose
  reconstructed root disagrees with the desk root forces a descent that
  re-derives the true state from the desk. Runtimes MUST validate sidecar
  structure on load and discard it wholesale on any parse failure.
- **CAS base honesty.** The CAS base is supplied by the runtime, which already
  has full control of the ship; a dishonest base can suppress or fabricate a
  conflict *notification* but cannot change a commit outcome, since the base
  is advisory and the write applies regardless. Conflict notifications are
  therefore an integrity aid for cooperating runtimes, not a security boundary.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
