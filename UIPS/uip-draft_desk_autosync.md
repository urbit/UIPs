---
title: Per-File Desk Auto-Sync
description: Event-driven, per-file synchronization between mounted desks and the host filesystem.
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
runtime to start or stop watching. The proposal also specifies that runtimes
should persist per-file content hashes across restarts and must not inject
empty sync events, eliminating the two main sources of event-log bloat in the
current `|autocommit` mechanism.

## Motivation

Synchronization between %clay and the host filesystem is already per-file in
principle: `%info` commits are lists of `[path miso]` deltas, the `%into`
task carries `(list [path (unit mime)])`, and `%ergo` gifts carry only the
files changed by a commit. In practice, three defects make continuous syncing
impractically expensive:

1. **Restart amplification.** The unix driver detects changes by comparing a
   per-file hash (`gum_w`, a mug of the file's bytes) held only in memory.
   After a restart the watch tree is rebuilt with zeroed hashes, so the first
   commit after every boot injects a `%into` event containing the entire
   contents of the desk. %clay correctly filters unchanged files against its
   mime cache, but by then the full desk has already been written into the
   event log as that event's payload.

2. **Idle-churn amplification.** `|autocommit` is a one-second %behn timer
   loop in %hood/kiln. Every tick produces a `%wake` event, a `%dirk` round
   trip, a full rescan (read + hash of every file in the mount), and — because
   the unix driver injects the `%into` event unconditionally, even when the
   change list is null — one empty `%into` event. An idle ship with one
   auto-committed desk accretes on the order of 170,000 junk events per day.

3. **Hash-comparison bug.** When the unix driver declines to overwrite a
   locally-modified file, it caches the mug of the *noun* rather than the mug
   of the file's *bytes*. For file contents ending in zero bytes these
   differ, so such files are re-sent in every scan, forever.

The result is that users who want live syncing between an editor on Earth and
a desk on Mars either suffer unbounded event-log growth or must manually
`|commit` after every save. This proposal makes continuous bidirectional
syncing event-driven, incremental, and quiescent when idle.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119 and RFC 8174.

### New %clay task

One task is added to `$task:clay` in `lull.hoon`:

```hoon
[%wath pot=term on=?]                    ::  (un)mark for auto-sync
```

`pot` names an existing mount point (as created by `%mont`). `on=&` marks the
mount point for auto-sync; `on=|` unmarks it. %clay MUST crash on a `%wath`
task naming a mount point that does not exist.

### New %clay gifts

Two gifts are added to `$gift:clay`, emitted on the unix duct (`hez`)
alongside the existing `%dirk`, `%ergo`, `%hill`, and `%ogre`:

```hoon
[%wath pot=term]                         ::  start watching mount point
[%wend pot=term]                         ::  stop watching mount point
```

### %clay state and behavior

%clay records the set of auto-synced mount points (e.g. `syn=(set term)`
alongside `mon` in its state; the concrete representation is unconstrained).

- On `%wath` with `on=&`, %clay MUST add the mount point to the set and give
  `[%wath pot]` on the unix duct. With `on=|` it MUST remove it and give
  `[%wend pot]`.
- On unmounting (`%ogre` flow), %clay MUST remove the mount point from the
  set. The runtime discards its watchers with the mount point, so no `%wend`
  is required.
- At runtime initialization (the `%boat` task, after giving `%hill`), %clay
  MUST re-give `[%wath pot]` for every auto-synced mount point, so that
  watchers are re-armed after a restart or runtime upgrade.
- After giving `%wath` (both on the `%wath` task and at `%boat`), %clay
  MUST give a full `%ergo` for the mount point — a mirror of the desk's
  current state. Combined with the runtime's write rules below, this
  makes the mount self-healing: files missing on disk are restored from
  the desk, unchanged files are untouched, and files edited on disk are
  left for the inbound sync. Since gifts are not persisted, the mirror
  adds nothing to the event log.

The commit path is unchanged: `%into` handling already filters unchanged
files against the mime cache and commits per-file subsets, and `%ergo` gifts
already carry only the files changed by each commit.

### Runtime (unix driver) behavior

On receiving a `%wath` effect for a mount point, the runtime MUST begin
watching that mount point's directory tree using native filesystem-event
facilities where available (e.g. inotify on Linux, FSEvents on macOS). On
receiving `%wend`, it MUST stop. A runtime without filesystem-event support
MAY approximate watching by polling, but the observable event-injection
behavior below still applies.

While a mount point is watched:

- On notification of filesystem changes, the runtime MUST rescan only the
  affected subtree(s), and MUST inject a single `%into` event (with `all=|`)
  containing only files whose content hash differs from the last content
  known to be synchronized with %clay.
- The runtime SHOULD debounce until quiescence: each observed change
  SHOULD extend the coalescing window (RECOMMENDED order of 100ms of
  silence), bounded by a maximum delay from the first change (RECOMMENDED
  order of 1s) so that a continuously-writing process cannot starve
  synchronization. This batches multi-step editor save sequences and
  multi-file saves into one `%into` event.
- The runtime SHOULD NOT sync a deletion the first time a file is
  observed missing. A missing file SHOULD be rechecked after a grace
  period (RECOMMENDED order of 300ms) and its deletion synced only if it
  is still absent. Editors commonly delete or rename a file away moments
  before rewriting it; without a grace period, such saves propagate a
  transient deletion into the desk's revision history, with visible
  side effects (e.g. application reloads).
- The runtime SHOULD periodically rescan watched mount points as a
  backstop (RECOMMENDED order of 30s), since filesystem notification
  mechanisms can drop events (e.g. inotify queue overflow or exhausted
  watch descriptors). A backstop rescan of an unchanged tree produces no
  event, per the requirement below.
- The runtime MUST NOT inject a `%into` event whose change list is null.
  (This requirement applies to the `%dirk`-triggered scan path as well, and
  is a behavioral fix independent of auto-sync.)
- A watched mount point remains subject to ordinary `%dirk` commits; `|commit`
  continues to work unchanged.
- Symbolic links are followed, exactly as in ordinary mounts and commits.
  Note that a change to a link's target outside any watched directory
  generates no filesystem event, and so syncs on the next commit or scan
  rather than immediately.

Across restarts, the runtime SHOULD persist enough per-file state (e.g. a
`path → mug` cache per mount point, updated when `%ergo` is applied and when
an injected `%into` event is committed) that the post-boot reconciliation
scan injects only files that actually changed while the ship was down. A
runtime that cannot do so falls back to current behavior (one full-desk
`%into` after boot), which is degraded but correct: %clay's mime-cache filter
still yields an accurate commit.

When comparing file contents, the runtime MUST compare hashes computed over
the file's byte string at its actual length on both sides of the comparison.
(The current implementation caches a noun mug on one side and a byte mug on
the other, which disagree for contents with trailing zero bytes.)

When applying `%ergo`, the runtime MUST NOT overwrite a file whose on-disk
content differs from the content last known to be synchronized: such a file
was edited on disk, and the edit syncs inward instead. Files absent from
disk are written; files matching the last-synchronized content are
overwritten (or skipped when identical to the `%ergo` payload).

### Reconciliation semantics

Together, the rules above give auto-sync a definite conflict-resolution
policy: **the desk is the ground truth for existence; the disk is believed
for live edits.**

- A change observed by the watcher — including a deletion that survives
  the grace period — expresses intent, and syncs into the desk.
- At reconciliation (marking a mount for auto-sync, or rebooting), a file
  present in the desk but missing on disk is restored to disk, never
  deleted from the desk. A deletion made while the runtime was not
  watching is indistinguishable from a damaged mirror (a wiped directory,
  a restored backup), and the failure mode is chosen to preserve data:
  an unwanted restoration costs one deletion on a live ship, while an
  unwanted desk deletion destroys content.
- A file edited while the runtime was not watching carries its own
  content — positive evidence of intent — and syncs into the desk; the
  mirror declines to overwrite it.

There is no two-sided conflict case: the desk cannot change while the
ship is down.

One consequence of desk-as-ground-truth deserves a note: file contents
on disk converge to the *canonical form* of the file's mark. Marks that
are lossy over raw bytes — notably `wain`-backed text marks, which
cannot represent a trailing newline — cause a sync-back rewrite when an
editor saves a non-canonical form (most editors append a final newline
by default). The sequence converges in one round trip, but editors with
the file open will observe it change on disk once per save. The durable
fix is mark-side (a text representation that round-trips); runtimes
MUST NOT suppress the rewrite, as that reintroduces silent divergence
between disk and desk.

### User interface

%hood/kiln gains a poke (e.g. `%kiln-autosync`) passing `[%wath pot on]` to
%clay, exposed via generators:

```
|autosync %desk           ::  mark a mounted desk for auto-sync
|cancel-autosync %desk    ::  unmark it
```

### Latency

Auto-sync makes a file change observable in %clay one commit later, so
its latency floor is the cost of a %clay commit; the watcher machinery
adds only the debounce window plus single-digit milliseconds of runtime
work. Profiling one-file commits on a development ship showed the
commit cost is dominated by the agent and mark rebuilds that `+park`
performs via `+goad` before giving %gall its `%load`: about 4.4s of a
4.9s commit in the measured configuration (`+build-agents` ~2.5s,
`+build-marks` ~1.9s), independent of what the commit touched. This
cost is a known issue with `+goad`; urbit/urbit#7353 scopes the rebuild
to the desk that changed rather than all live desks, which removes the
cross-desk amplification. A commit to a live desk still rebuilds that
desk's agents and marks; a commit to a non-live desk skips the rebuild
entirely and was measured at roughly 10ms of vane time (~180ms end to
end, most of which is the debounce window).

Reducing the rebuild cost further — narrowing it to the commits and
files that can actually affect build inputs, or caching builds across
commits — is complementary to this proposal and out of scope.

## Rationale

**Why a %clay task rather than a runtime-only switch?** The set of
auto-synced desks is durable ship state: it must survive restarts and runtime
upgrades, and the natural control surface for desk operations is the dojo via
%hood. Storing the flag in %clay and re-arming watchers via gifts at `%boat`
follows the existing pattern used for `%hill`/mount re-initialization. A
runtime-only design (config file or CLI flag) was considered and rejected
because it splits desk state between Mars and Earth and provides no dojo
affordance.

**Why filesystem events rather than faster polling?** Polling couples cost to
desk size and poll frequency rather than to change volume. The existing
`|autocommit` demonstrates the failure mode: cost is paid every second even
when nothing changes. Filesystem events make the idle cost zero in both CPU
and event-log terms, and libuv (already the runtime's event loop) wraps the
platform facilities portably.

**Why per-file events rather than batching commits?** The point of the
proposal is that the event log should grow in proportion to actual change
volume. One small `%into` per save (with debounce coalescing naturally
related changes) achieves this; any coarser batching reintroduces either
latency or amplification.

**Why quiescence debouncing and a deletion grace period?** Editors do
not save files atomically from the watcher's perspective: common
patterns are write-temp-then-rename, delete-then-write, and
truncate-then-write. Rename-based saves are inherently safe (the new
content lands atomically under the final name), but the other two
expose windows in which the file is missing or partial. A fixed-delay
debounce can fire inside such a window, committing a transient
deletion or truncated content. Extending the window until the
filesystem quiesces, and confirming deletions after a grace period,
makes the synced sequence match user intent rather than syscall
interleavings. Both heuristics are runtime-local: a transient state
that does slip through still converges, since the next change event
re-syncs the file.

**Why is the mug cache persistence a SHOULD?** It is a runtime-local
optimization invisible to Arvo: with or without it, %clay computes the same
commits. Mandating a particular on-disk format would overconstrain other
runtimes (e.g. Ares) without an interoperability benefit.

**Relation to `|autocommit`.** `|autocommit` is left intact but becomes
redundant for auto-synced desks. It MAY be deprecated in a later proposal
once auto-sync has shipped.

## Backwards Compatibility

The new task and gifts extend `lull.hoon` and therefore require a kelvin
decrement to ship.

Mismatched pairs degrade gracefully:

- An updated Arvo on an older runtime: the unix driver ignores effect tags it
  does not recognize, so `%wath`/`%wend` gifts are dropped and the desk
  simply is not auto-synced. `|commit` and `|autocommit` behave as today.
- Older Arvo on an updated runtime: stock %clay never gives `%wath`, so no
  watchers are created and behavior is unchanged. The empty-`%into`
  suppression and hash-comparison fixes apply regardless and are strictly
  beneficial.

The persisted hash cache is advisory: a missing or stale cache file MUST
degrade to the zeroed-hash behavior (full reconciliation scan), never to an
incorrect commit.

## Open Questions

**Should `wain`-backed text marks move to a byte-faithful representation?**

Auto-sync makes mark canonicalization visible: file contents on disk
converge to the canonical form of the file's mark, and `wain` (a list of
lines) cannot represent a trailing newline. Most editors append a final
newline on save, so every save of a `%txt` file is followed one commit
later by a sync-back rewrite stripping it, which editors with the file
open report as an external modification. Under the previous polling sync
this same mismatch existed but manifested differently (permanent silent
divergence, with the file re-sent on every scan).

Notably, `%hoon` does not have this problem: its noun form is a cord
(`++noun @t`, with `+mime` passing octets through), so `.hoon` files
round-trip byte-for-byte, and its line-based diffing is recovered by
delegating `+grad` to `%txt`. This suggests the repair for `%txt` and
other `wain`-backed marks: store a cord, convert to `wain` only for
diffing and display.

The trade-off is migration: every consumer that expects `!<(wain ...)`
from a `%txt` cage breaks, existing desks hold `%txt` files under the
old type, and `%txt-diff` history references line-based diffs. Whether
that migration is worth byte-faithful text files — or whether the
canonicalization behavior should simply be documented and tolerated —
is left open here; either resolution is compatible with this proposal,
since the sync layer converges under any deterministic canonical form.

## Reference Implementation

Implemented and tested end-to-end:

- Arvo: [urbit/urbit#7362](https://github.com/urbit/urbit/pull/7362)
- runtime: [urbit/vere#1031](https://github.com/urbit/vere/pull/1031)

The implementation touches:

- `pkg/arvo/sys/lull.hoon` — `%wath` task; `%wath`/`%wend` gifts.
- `pkg/arvo/sys/vane/clay.hoon` — auto-sync set in state (+ state version
  bump), `%wath` handling, re-arming at `%boat`, cleanup at unmount.
- `pkg/arvo/lib/hood/kiln.hoon`, `pkg/arvo/gen/hood/autosync.hoon`,
  `pkg/arvo/gen/hood/cancel-autosync.hoon` — user interface.
- `pkg/vere/io/unix.c` — `uv_fs_event_t` watchers per directory, debounce
  timer, dirty-subtree rescan, per-mount `path → mug` sidecar persisted in
  the pier, suppression of empty `%into` events, and the byte-mug
  comparison fix in `_unix_write_file_soft()`.

## Security Considerations

The proposal adds no new namespace exposure: auto-sync moves data only
between a mount point the operator already created with `|mount` and the desk
it mirrors, in the same direction and with the same content as the existing
`%dirk`/`%into`/`%ergo` flow.

Two considerations are worth noting:

- **Event-injection volume.** Filesystem watchers turn host filesystem
  activity into Arvo events. A runaway process writing to a mounted directory
  could inject events at high frequency. The debounce window bounds the event
  rate, and the per-file hash comparison bounds payload size to actual
  changes; runtimes MAY additionally rate-limit injection. This is no worse
  than current `|autocommit` behavior, which injects events at a fixed rate
  regardless of activity.
- **Hash-cache integrity.** The persisted mug cache, if tampered with or
  corrupted, can cause changed files to be skipped (stale hash collision) or
  unchanged files to be re-sent (cache loss). The latter is the safe
  degraded mode specified above. The former requires write access to the
  pier, which already implies full control of the ship; runtimes SHOULD
  nevertheless validate cache-file structure on load and discard it wholesale
  on any parse failure.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
