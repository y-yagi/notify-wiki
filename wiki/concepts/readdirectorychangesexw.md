---
title: ReadDirectoryChangesExW
tags: [windows, win32, event-driven]
updated: 2026-07-19
sources: ["../sources/msdn-readdirectorychangesexw.md", "../sources/oldnewthing-readdirectorychangesw-deletion-details.md"]
---

# ReadDirectoryChangesExW

## Overview

`ReadDirectoryChangesExW` (`winbase.h`, `Kernel32.dll`, since Windows 10
version 1709 / Server 2019) is a newer sibling of
[[readdirectorychangesw]] that adds the ability to request a richer,
extended change-record format in addition to the classic one. Everything
about how you open the directory handle, arm the wait, and receive
completion is otherwise identical to `ReadDirectoryChangesW`.

## API / semantics

- Same parameter list as `ReadDirectoryChangesW` (`hDirectory`, `lpBuffer`,
  `nBufferLength`, `bWatchSubtree`, `dwNotifyFilter`, `lpBytesReturned`,
  `lpOverlapped`, `lpCompletionRoutine`), plus one new parameter:
  `ReadDirectoryNotifyInformationClass` of type
  `READ_DIRECTORY_NOTIFY_INFORMATION_CLASS`.
- That new parameter selects the buffer's record layout:
  - `ReadDirectoryNotifyInformation` — the buffer is filled with
    `FILE_NOTIFY_INFORMATION` records, i.e. identical output to
    `ReadDirectoryChangesW`.
  - `ReadDirectoryNotifyExtendedInformation` — the buffer is filled with
    `FILE_NOTIFY_EXTENDED_INFORMATION` records instead. The MSDN reference
    itself doesn't detail this structure's fields, but a follow-up source
    (a Microsoft DevBlogs post) confirms it carries directory metadata —
    including file attributes and size — attached to *both* add and
    remove events. This is the key practical advantage over the plain
    `FILE_NOTIFY_INFORMATION` format: see Limitations & gotchas below.
- `dwNotifyFilter` bitmask is identical to `ReadDirectoryChangesW`'s: the
  same FILE_NAME/DIR_NAME/ATTRIBUTES/SIZE/LAST_WRITE/LAST_ACCESS/CREATION/SECURITY
  set, no additions.
- Synchronous or asynchronous (overlapped) operation, with the same three
  completion mechanisms as `ReadDirectoryChangesW`: `GetOverlappedResult`,
  `GetQueuedCompletionStatus` (I/O completion port), or a completion
  routine callback in an alertable wait state.
- Same per-handle system-allocated buffer model as `ReadDirectoryChangesW`:
  fixed size for the handle's lifetime, changes accumulate between calls.

## Limitations & gotchas

- **NTFS-only**: the source states "ReadDirectoryChangesExW is currently
  supported only for the NTFS file system." No SMB/CsvFS/ReFS network
  support table is given at all, unlike `ReadDirectoryChangesW`'s page —
  it's not documented here whether that means no network support or just
  an undocumented gap.
- Same buffer-overflow silent-discard behavior as `ReadDirectoryChangesW`:
  the call still returns success but `lpBytesReturned` comes back 0, and
  the only correct response is a full directory/subtree re-enumeration.
- Same 64 KB buffer cap over the network (`ERROR_INVALID_PARAMETER`) and
  DWORD-alignment requirement (`ERROR_NOACCESS`) as `ReadDirectoryChangesW`
  — though given the NTFS-only claim above, the network cap's relevance to
  this specific API isn't clear from this source.
- `ERROR_NOTIFY_ENUM_DIR` — same distinct "system couldn't record all
  changes" failure as `ReadDirectoryChangesW`, same full-enumeration
  fallback.
- Directory-or-subtree granularity only, same as `ReadDirectoryChangesW`
  and `FindFirstChangeNotification` — no true per-file watch mode.
- **Solves `ReadDirectoryChangesW`'s deleted-item information gap**: with
  `ReadDirectoryChangesW`, a `FILE_ACTION_REMOVED` record carries only a
  name — the deleted item is already gone by the time the app learns
  about it, so attributes and size can't be recovered, and the usual
  cache-based workaround (populate from `FindFirstFile`/`FindNextFile`,
  update on add events) has its own race if create+delete happen faster
  than the cache can be updated. `ReadDirectoryChangesExW` with
  `ReadDirectoryNotifyExtendedInformation` attaches file attributes and
  size directly to removal records (as well as add records), eliminating
  the race — no separate query or cache needed. See
  `[[readdirectorychangesw]]`'s own Limitations section for the problem
  this solves.
- **Desktop apps only** — no UWP app support listed, unlike
  `ReadDirectoryChangesW` (which supports both desktop and UWP).

## Platform notes

- Minimum supported client: Windows 10, version 1709 (desktop apps only).
- Minimum supported server: Windows Server 2019 (desktop apps only).
- Significantly newer minimum requirement than `ReadDirectoryChangesW`
  (Windows XP / Server 2003) — this is a recent addition to the Win32
  directory-notification family, not a drop-in replacement usable
  everywhere `ReadDirectoryChangesW` is.

## Related concepts

- `[[readdirectorychangesw]]` — the original, more widely-supported
  sibling this API extends; identical parameter set and semantics aside
  from the added `ReadDirectoryNotifyInformationClass` selector and the
  narrower platform/filesystem support.
- `[[findfirstchangenotification]]` — the coarser, presence-only ancestor
  API in the same Win32 family; neither this API nor that one report
  changes without an explicit read/buffer step, but this one (like
  `ReadDirectoryChangesW`) at least fills in structured records rather
  than a bare signal.
- [[recursive-watching]] — cross-cutting comparison of tree-watching
  support across all mechanisms/libraries in this wiki.
- [[windows-notification-apis]] — history and feature-by-feature
  comparison across all four Windows notification APIs, with a current
  recommendation.

## Sources

- [msdn-readdirectorychangesexw](../sources/msdn-readdirectorychangesexw.md)
- [oldnewthing-readdirectorychangesw-deletion-details](../sources/oldnewthing-readdirectorychangesw-deletion-details.md)
