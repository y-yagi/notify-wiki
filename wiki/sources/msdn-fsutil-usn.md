---
title: "Source: fsutil usn command reference (Microsoft Learn)"
tags: [windows, ntfs, source]
updated: 2026-07-08
---

# msdn-fsutil-usn

**Raw file**: [`raw/msdn-fsutil-usn.txt`](../../raw/msdn-fsutil-usn.txt)
**Origin**: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-usn — Windows/Windows Server command reference for `fsutil usn`, the admin CLI for managing the NTFS USN change journal. Last updated 2024-11-01 per the page footer.

## What it is

The command-line reference for `fsutil usn` — not the programmatic
`DeviceIoControl`/`FSCTL_*` API itself, but the admin tool for creating,
querying, reading, and deleting a volume's **USN change journal**, the
NTFS-level, whole-volume change log that [[readdirectorychangesw]]'s own
documentation points to as the alternative for volume-wide tracking.

## What it claims

- The USN change journal is a **persistent, append-only log of every change
  NTFS makes to files/directories/other NTFS objects on a volume** — one
  journal per volume, one record per change, each record identifying the
  type of change and the object changed. New records are always appended to
  the end of the stream.
- Positioned as **more efficient than timestamp-checking or registering for
  file notifications** (i.e. more efficient than [[readdirectorychangesw]])
  for determining all modifications to a set of files — used internally by
  the Indexing Service, File Replication Service (FRS), Remote Installation
  Services (RIS), and Remote Storage.
- Journal size is bounded (`m=<maxsize>`) with headroom (`a=<allocationdelta>`)
  added/removed at each end; NTFS only trims the journal back down at NTFS
  checkpoints, so it can transiently exceed `maxsize + allocationdelta`
  before being trimmed.
- `createjournal` on a volume that already has a journal **updates its
  size parameters in place** rather than erroring — lets an admin expand
  an active journal's capacity without disabling it first.
- `deletejournal`/disabling a journal is **expensive and slow**: NTFS must
  walk the entire Master File Table (MFT) and zero the last-USN attribute
  on every record, taking minutes and potentially continuing across a
  reboot. While disabling is in progress, the journal is in neither an
  active nor disabled state and all journal operations return errors — and
  it "adversely affects other applications... using the journal" (any app
  relying on the journal for volume-wide change tracking loses that data
  and must fall back to a full rescan, same recovery pattern as
  [[readdirectorychangesw]]'s buffer-overflow case, but volume-wide instead
  of per-directory).
- `readjournal` reads records starting from a given USN (`startusn=`,
  default 0) with `minver`/`maxver` bounding which `USN_RECORD` structure
  version to return (major versions 2–4 supported; structure itself not
  detailed on this page).
- `enumdata` filters records by a `[lowUSN, highUSN]` range starting from a
  given file's ordinal position on the volume.
- `enablerangetracking` — a distinct capability for tracking specific
  byte-range writes within large files (`c=<chunk-size>`,
  `s=<file-size-threshold>`), not just whole-file change events.

## Concept pages informed

- [`concepts/usn-journal.md`](../concepts/usn-journal.md) (new)
- [`concepts/readdirectorychangesw.md`](../concepts/readdirectorychangesw.md) (added USN journal as a related, volume-wide alternative)
