---
title: ReadDirectoryChangesW
tags: [windows, win32, event-driven]
updated: 2026-07-08
sources: ["../sources/msdn-readdirectorychangesw.md"]
---

# ReadDirectoryChangesW

## Overview

`ReadDirectoryChangesW` (`winbase.h`, `Kernel32.dll`, since Windows XP /
Server 2003) is the Win32 API for retrieving **actual change details** —
which file, what kind of change — for a watched directory or subtree. It's
the detail-reporting counterpart to [[findfirstchangenotification]], which
only signals that *some* filtered change occurred with no further
information. Microsoft's own docs for the older API explicitly redirect here
whenever an application needs to know what happened.

## API / semantics

- Operates on a directory `HANDLE` opened via `CreateFile` with
  `FILE_FLAG_BACKUP_SEMANTICS` and `FILE_LIST_DIRECTORY` (or a broader access
  right like `GENERIC_READ` that includes it).
- Fills a caller-supplied, **DWORD-aligned** buffer with a linked list of
  `FILE_NOTIFY_INFORMATION` records (structure not detailed in this source —
  documented on its own separate reference page).
- `bWatchSubtree` and `dwNotifyFilter` mirror `FindFirstChangeNotification`'s
  parameters, but the filter set is a **superset**: adds
  `FILE_NOTIFY_CHANGE_LAST_ACCESS` and `FILE_NOTIFY_CHANGE_CREATION` on top
  of the shared `FILE_NAME`/`DIR_NAME`/`ATTRIBUTES`/`SIZE`/`LAST_WRITE`/`SECURITY`
  bits. Size and last-write detection is still tied to when the OS actually
  flushes to disk, not to the write call itself.
- **Synchronous or asynchronous (overlapped) operation**, selected by
  whether the directory was opened with `FILE_FLAG_OVERLAPPED` and whether
  an `OVERLAPPED` struct is supplied. Three ways to receive async
  completion:
  - `GetOverlappedResult`, keyed off a unique event in the `OVERLAPPED`
    struct's `hEvent` member.
  - `GetQueuedCompletionStatus`, via an I/O completion port the directory
    handle is associated with (`CreateIoCompletionPort`).
  - A completion routine callback, invoked while the calling thread is in
    an alertable wait state (mutually exclusive with completion ports —
    `hEvent` is then free for the caller's own use).
- The system allocates one buffer per directory handle on first call; its
  size is fixed for the handle's lifetime. Changes accumulate in that buffer
  between calls and are drained on the next `ReadDirectoryChangesW` call.
- Explicitly out of scope: whole-volume change tracking has a separate
  mechanism, the [[usn-journal]] ("change journals"), that this API doesn't cover.

## Limitations & gotchas

- **Buffer overflow silently discards all pending changes**: if the
  system-allocated buffer overflows between calls, the function still
  returns success (nonzero) but `lpBytesReturned` comes back 0 — the actual
  changes are gone, not truncated-but-partial. The only correct response is
  to detect the zero-byte case and fall back to a **full directory/subtree
  enumeration**, since there is no way to know what was lost. Conceptually
  the same "must full-rescan on overflow" pattern as inotify's
  `IN_Q_OVERFLOW` ([[inotify]]) and FSEvents' `KernelDropped`/`UserDropped`
  flags ([[fsevents]]), but with no advance warning flag at all — you only
  learn about it from the zero-length result.
  - `ERROR_NOTIFY_ENUM_DIR` is a related, distinct failure: the system
    itself couldn't record all changes to the directory (as opposed to the
    buffer overflowing on the client side); same enumeration fallback applies.
- **64 KB buffer cap over the network**: fails with `ERROR_INVALID_PARAMETER`
  if the buffer exceeds 64 KB while monitoring a directory over the network,
  due to underlying file-sharing-protocol packet size limits — an
  application watching remote directories must use a smaller buffer (and is
  therefore more overflow-prone) than one watching local disk.
- **`ERROR_NOACCESS` on misaligned buffers** — the buffer must be
  DWORD-aligned or the call fails outright.
- **Not universally supported** — if the network redirector or target file
  system doesn't implement this operation, it fails with
  `ERROR_INVALID_FUNCTION`; applications need a fallback path (e.g. polling)
  for such targets.
- Directory-or-subtree granularity only, same as `FindFirstChangeNotification`
  — no true per-file watch mode.

## Platform notes

- Minimum supported client: Windows XP (desktop and UWP apps).
- Minimum supported server: Windows Server 2003 (desktop and UWP apps).
- Supported over SMB 3.0 (including Transparent Failover and Scale-out File
  Shares), Cluster Shared Volume File System (CsvFS), and ReFS as of
  Windows 8 / Server 2012 — same support table as `FindFirstChangeNotification`.
- Respects transaction isolation rules if the directory handle is bound to a
  transaction.

## Related concepts

- `[[findfirstchangenotification]]` — the coarser, presence-only sibling API;
  this is what Microsoft's own docs direct you to instead when you need
  actual change details.
- `[[inotify]]` — closest Linux analogue in spirit: both deliver structured
  event records (filename + change type) over a buffer/fd the app reads,
  and both require careful overflow handling. inotify signals overflow
  explicitly (`IN_Q_OVERFLOW`) rather than only via a zero-length read.
- `[[fsevents]]` — macOS's analogue; coarser (directory-level, no per-file
  detail in the callback) but adds cross-reboot persistence that neither
  Windows API has.
- `[[usn-journal]]` — the volume-wide, persistent alternative this API's own
  docs point to for whole-volume tracking; unlike this live per-directory
  handle, the journal survives app restarts and covers the whole volume.
- [[recursive-watching]] — cross-cutting comparison of tree-watching support across all mechanisms/libraries in this wiki.

## Sources

- [msdn-readdirectorychangesw](../sources/msdn-readdirectorychangesw.md)
