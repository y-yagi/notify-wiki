---
title: USN Change Journal
tags: [windows, ntfs, event-driven]
updated: 2026-07-19
sources: ["../sources/msdn-fsutil-usn.md", "../sources/oldnewthing-readdirectorychangesw-deletion-details.md", "../sources/msdn-fsctl-query-usn-journal.md", "../sources/msdn-fsctl-query-usn-journal-win32.md", "../sources/msdn-using-the-change-journal-identifier.md", "../sources/msdn-walking-a-buffer-of-change-journal-records.md", "../sources/msdn-fsctl-read-usn-journal.md"]
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
  underlying programmatic interface (user-mode) is `DeviceIoControl` on a
  volume handle, with `FSCTL_*` control codes:
  - Open the volume with `CreateFile(TEXT("\\\\.\\X:"), GENERIC_READ |
    GENERIC_WRITE, FILE_SHARE_READ | FILE_SHARE_WRITE, NULL,
    OPEN_EXISTING, 0, NULL)` — `X` is the drive letter; the volume must be
    NTFS.
  - `FSCTL_QUERY_USN_JOURNAL` — queries the current journal's identifier,
    first/next USN, and capacity into a `USN_JOURNAL_DATA` struct. No
    input buffer needed.
  - `FSCTL_READ_USN_JOURNAL` — reads records selectively (e.g. by change
    reason, or "only when file is closed") starting from a given
    `StartUsn`, using the journal identifier obtained from
    `FSCTL_QUERY_USN_JOURNAL` as an integrity check against reads from a
    stale/prior journal instance.
  - `FSCTL_ENUM_USN_DATA` — returns an unfiltered listing (enumeration) of
    all records in a USN range instead of a selective read; use this when
    you don't need `FSCTL_READ_USN_JOURNAL`'s filtering.
  - Output buffer layout for both `FSCTL_READ_USN_JOURNAL` and
    `FSCTL_ENUM_USN_DATA`: a leading `USN` (the next record's starting
    point, used as `StartUsn` on the following call), followed by zero or
    more `USN_RECORD_V2`/`USN_RECORD_V3` records. Each record is
    variable-length: its first member `RecordLength` gives the record's
    total byte size (including the trailing file name); walk the buffer
    by repeatedly advancing a pointer by `RecordLength` until the
    returned byte count is exhausted. `FileName`/`FileNameLength` give the
    file name — **no guaranteed trailing null terminator**, so length
    must come from `FileNameLength`, not string scanning. Max single
    record size: `(MaxComponentLength - 1) * sizeof(WCHAR) +
    sizeof(USN_RECORD)`, where `MaxComponentLength` comes from
    `GetVolumeInformation`.
  - This same `FSCTL_QUERY_USN_JOURNAL` control code also has a
    **kernel-mode** invocation surface documented on a separate WDK DDI
    reference page — see Limitations below.
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
- `FSCTL_QUERY_USN_JOURNAL` has **two documented invocation surfaces**: a
  **kernel-mode** one (WDK DDI reference: called via `FltFsControlFile`
  from a minifilter driver, or `ZwFsControlFile`, the kernel-mode native
  API, taking a `FileObject`/`FileHandle` for the volume) and the
  ordinary **user-mode** one (Win32 SDK reference: called via
  `DeviceIoControl` on a handle from `CreateFile`). Both return a pointer
  to a `USN_JOURNAL_DATA` structure; neither reference page details that
  structure's fields further. See API/semantics above for the user-mode
  call shape and the `USN_RECORD_V2`/`V3` buffer layout.
- **Reading the journal requires admin rights — now confirmed**:
  Microsoft's "Using the Change Journal Identifier" page states
  explicitly: *"To perform this and all other change journal operations,
  you must have system administrator privileges. That is, you must be a
  member of the Administrators group."* This confirms what had earlier
  only been a lower-confidence claim from a DevBlogs comment thread.
  Neither the kernel-mode DDI page nor the user-mode Win32 reference page
  for `FSCTL_QUERY_USN_JOURNAL` states this requirement in their own body
  text — it's documented only on this separate conceptual page — but it
  is now an authoritative, confirmed claim covering "all change journal
  operations," not just the query call. The journal is therefore **not**
  a lightweight, unprivileged alternative to
  `[[readdirectorychangesw]]`/`[[readdirectorychangesexw]]`; reading it
  requires the calling process to run as (or impersonate) a member of the
  Administrators group.
- **ReFS support — resolved, prior wiki text overstated the exclusion**:
  an earlier version of this page claimed "not available on FAT/exFAT/
  ReFS," but re-checking the `fsutil usn` source directly shows it never
  actually mentions ReFS (or FAT/FAT32) at all — it only describes NTFS
  behavior, without asserting other filesystems are excluded. The
  ReFS-exclusion claim was an unsupported generalization introduced while
  writing this page, not something either source states. "Walking a
  Buffer of Change Journal Records" is explicit and authoritative here:
  *"The target volume for USN operations must be ReFS or NTFS 3.0 or
  later."* Treat ReFS as supported. See Platform notes below for the
  corrected filesystem-support summary.

## Platform notes

- **Supported filesystems**: NTFS 3.0+ and ReFS, per "Walking a Buffer of
  Change Journal Records" (*"The target volume for USN operations must be
  ReFS or NTFS 3.0 or later"*) — this is the most specific, authoritative
  statement on filesystem scope found so far. The `fsutil usn` reference
  page doesn't contradict this; it simply never mentions ReFS (or FAT/
  FAT32) either way, describing only NTFS behavior.
- **ExFAT has no change journal at all** — stated in a DevBlogs
  comment-thread discussion (lower confidence — reader claim, not from
  Microsoft's own reference docs), consistent with ExFAT's absence from
  every other source's filesystem list here.
- No version/support history given in the `fsutil usn` source beyond
  "Applies to" listing current Windows Server/Windows 10/11 releases.
- `FSCTL_QUERY_USN_JOURNAL` / `FSCTL_READ_USN_JOURNAL` (user-mode,
  `winioctl.h`) minimum supported: Windows XP / Server 2003 (desktop apps
  only). Network/clustered filesystem support (Windows 8 / Server 2012
  era): SMB 3.0, SMB 3.0 Transparent Failover, and SMB 3.0 Scale-out File
  Shares are **not** supported for either control code. Cluster Shared
  Volume File System (CsvFS) support differs: **yes** for
  `FSCTL_QUERY_USN_JOURNAL`, "see comment" for `FSCTL_READ_USN_JOURNAL`
  — both carry a caveat that an application may see false positives
  across CsvFS pause/resume.

## Related concepts

- `[[readdirectorychangesw]]` / `[[readdirectorychangesexw]]` — the live,
  per-directory-handle notification APIs; Microsoft's own docs for
  `ReadDirectoryChangesW` point to the USN journal specifically for
  whole-volume tracking instead. Complementary rather than redundant: the
  journal survives across app restarts and covers the whole volume, at the
  cost of being coarser to consume (must track your own last-read USN) and
  carrying real administrative weight (disabling it is slow and
  volume-wide, plus reading it may itself require admin rights — see
  Limitations above).
- `[[fsevents]]` — closest cross-platform analogue: both are persistent,
  queryable-from-a-past-checkpoint change logs rather than live callback
  streams, letting a consumer catch up on changes made while it wasn't
  running. FSEvents' scope is directory-hierarchy; the USN journal's is the
  whole NTFS volume.
- `[[inotify]]` / `[[fanotify]]` — neither has a persistent, replayable log;
  both are live-only, in-memory event streams that lose everything once the
  fd is closed or the queue overflows.
- [[recursive-watching]] — cross-cutting comparison of tree-watching support across all mechanisms/libraries in this wiki.
- [[windows-notification-apis]] — history and feature-by-feature
  comparison across all four Windows notification APIs, with a current
  recommendation.

## Sources

- [msdn-fsutil-usn](../sources/msdn-fsutil-usn.md)
- [oldnewthing-readdirectorychangesw-deletion-details](../sources/oldnewthing-readdirectorychangesw-deletion-details.md)
- [msdn-fsctl-query-usn-journal](../sources/msdn-fsctl-query-usn-journal.md)
- [msdn-fsctl-query-usn-journal-win32](../sources/msdn-fsctl-query-usn-journal-win32.md)
- [msdn-using-the-change-journal-identifier](../sources/msdn-using-the-change-journal-identifier.md)
- [msdn-walking-a-buffer-of-change-journal-records](../sources/msdn-walking-a-buffer-of-change-journal-records.md)
- [msdn-fsctl-read-usn-journal](../sources/msdn-fsctl-read-usn-journal.md)
