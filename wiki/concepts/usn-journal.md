---
title: USN Change Journal
tags: [windows, ntfs, event-driven]
updated: 2026-07-08
sources: ["../sources/msdn-fsutil-usn.md"]
---

# USN Change Journal

## Overview

The USN (Update Sequence Number) change journal is an NTFS-level,
**whole-volume, persistent, append-only log** of every change made to
files, directories, and other NTFS objects — one journal per volume. It's a
fundamentally different design point from [[readdirectorychangesw]] and
[[findfirstchangenotification]]: instead of a live per-directory watch an
application actively polls or waits on, the journal is a durable record any
number of consumers can read from at their own pace, including catching up
on changes that happened while they weren't running. It underlies several
built-in Windows services (Indexing Service, File Replication Service,
Remote Installation Services, Remote Storage) and is explicitly positioned
by Microsoft as more efficient than either timestamp-checking or
`ReadDirectoryChangesW`-style notification registration for determining all
changes to a set of files.

## API / semantics

- Managed at the admin/CLI level via `fsutil usn` (this source); the
  underlying programmatic interface is `DeviceIoControl` with `FSCTL_*`
  control codes and `USN_RECORD` structures (not detailed in this source —
  a separate reference page).
- `createjournal m=<maxsize> a=<allocationdelta>` creates (or, if one
  already exists, resizes in place) the journal on a volume. `maxsize`
  bounds the target size; `allocationdelta` is headroom added/removed at
  each end as the journal grows. NTFS only enforces the cap at NTFS
  checkpoints, so the journal can transiently grow past `maxsize + allocationdelta`
  before being trimmed back.
- `readjournal [minver=] [maxver=] [startusn=]` reads journal records from
  a given USN forward (default `startusn=0`), bounded to `USN_RECORD`
  major versions 2–4 (default range).
- `enumdata <fileref> <lowUSN> <highUSN>` enumerates records filtered to a
  USN range, starting from a given file's ordinal position on the volume.
- `queryjournal` reports the current journal's metadata (capacity, record
  range, etc.); `readdata <filename>` reads the USN data recorded for one
  specific file.
- `enablerangetracking c=<chunk-size> s=<file-size-threshold>` is a
  separate, finer-grained capability: tracking specific byte-range writes
  within files above a size threshold, rather than whole-file change events.
- `deletejournal {/d | /n}` disables/deletes the journal — `/d` returns I/O
  control immediately while disabling proceeds, `/n` blocks until disabling
  completes.

## Limitations & gotchas

- **Disabling a journal is expensive and disruptive**: NTFS must walk the
  entire Master File Table and zero the last-USN attribute on every record —
  can take minutes and may continue across a reboot. While in progress, the
  journal is in neither an active nor disabled state and all journal
  operations fail. Any application relying on the journal loses its change
  history and must fall back to a full volume rescan — the same
  "can't-know-what-was-missed, must re-enumerate" recovery pattern seen in
  [[readdirectorychangesw]]'s buffer-overflow case and inotify's
  `IN_Q_OVERFLOW` ([[inotify]]), but here it's volume-wide rather than
  per-directory, and deliberately disruptive to *other* applications sharing
  the journal, not just the one that triggered it.
  - This source doesn't cover the append-limit / wraparound failure mode
    directly, but explicitly notes the journal can grow beyond its
    configured cap between NTFS checkpoints — sizing `maxsize`/`allocationdelta`
    too small risks records rolling off before a slow consumer reads them
    (the classic USN journal gotcha, though this exact page doesn't state
    the consequence explicitly — flagging as an inference, not a direct claim).
- **Shared resource, not per-application** — unlike a `ReadDirectoryChangesW`
  handle (owned by one app) or an inotify instance (owned by one fd), the
  journal is one shared per-volume resource used by multiple OS services
  simultaneously. Disabling it for one consumer's convenience actively harms
  unrelated services (indexing, replication) relying on it.
- Structure of `USN_RECORD` itself and the full `FSCTL_*` DeviceIoControl
  interface are out of scope for this source — this page only covers the
  admin-facing `fsutil usn` surface, not the programmatic read API.

## Platform notes

- NTFS-specific — not available on FAT/exFAT/ReFS (no version/support
  history given in this source beyond "Applies to" listing current Windows
  Server/Windows 10/11 releases).

## Related concepts

- `[[readdirectorychangesw]]` — the live, per-directory-handle notification
  API; Microsoft's own docs for it point to the USN journal specifically for
  whole-volume tracking instead. Complementary rather than redundant: the
  journal survives across app restarts and covers the whole volume, at the
  cost of being coarser to consume (must track your own last-read USN) and
  carrying real administrative weight (disabling it is slow and
  volume-wide).
- `[[fsevents]]` — closest cross-platform analogue: both are persistent,
  queryable-from-a-past-checkpoint change logs rather than live callback
  streams, letting a consumer catch up on changes made while it wasn't
  running. FSEvents' scope is directory-hierarchy; the USN journal's is the
  whole NTFS volume.
- `[[inotify]]` / `[[fanotify]]` — neither has a persistent, replayable log;
  both are live-only, in-memory event streams that lose everything once the
  fd is closed or the queue overflows.
- [[recursive-watching]] — cross-cutting comparison of tree-watching support across all mechanisms/libraries in this wiki.

## Sources

- [msdn-fsutil-usn](../sources/msdn-fsutil-usn.md)
