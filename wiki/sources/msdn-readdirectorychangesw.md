---
title: "Source: ReadDirectoryChangesW function (Microsoft Learn)"
tags: [windows, win32, source]
updated: 2026-07-08
---

# msdn-readdirectorychangesw

**Raw file**: [`raw/msdn-readdirectorychangesw.txt`](../../raw/msdn-readdirectorychangesw.txt)
**Origin**: https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-readdirectorychangesw — Win32 API reference page for `ReadDirectoryChangesW` (winbase.h). Last updated 2022-09-15 per the page footer.

## What it is

The official Win32 API reference for `ReadDirectoryChangesW` — the
detail-reporting counterpart to `FindFirstChangeNotification`
([[findfirstchangenotification]]), and the API Microsoft's own docs point to
whenever an application needs to know *what* changed, not just *that*
something did.

## What it claims

- Fills a caller-supplied, DWORD-aligned buffer with `FILE_NOTIFY_INFORMATION`
  records describing each change (structure itself documented on a separate
  reference page, not reproduced here).
- Requires a directory handle opened via `CreateFile` with
  `FILE_FLAG_BACKUP_SEMANTICS` (and `FILE_LIST_DIRECTORY`/`GENERIC_READ`
  access); to track whole-volume changes instead, the page points to a
  separate "change journals" mechanism.
- Supports both **synchronous** and **asynchronous (overlapped)** operation,
  selected by whether the directory handle was opened with
  `FILE_FLAG_OVERLAPPED` and whether an `OVERLAPPED` struct is passed.
  Three ways to receive async completion: `GetOverlappedResult`,
  `GetQueuedCompletionStatus` (I/O completion ports), or a completion
  routine callback (requires an alertable wait state).
- `dwNotifyFilter` bitmask is a superset of `FindFirstChangeNotification`'s:
  adds `FILE_NOTIFY_CHANGE_LAST_ACCESS` and `FILE_NOTIFY_CHANGE_CREATION`
  on top of the shared FILE_NAME/DIR_NAME/ATTRIBUTES/SIZE/LAST_WRITE/SECURITY set.
- **Buffer overflow behavior**: if the allocated buffer overflows between
  calls, the function still returns success (`TRUE`), but the entire buffer
  contents are discarded and `lpBytesReturned` is 0 — the caller must detect
  this specifically (zero bytes returned) and fall back to a full directory
  enumeration, since the actual changes are unrecoverable.
- `ERROR_NOTIFY_ENUM_DIR` is a distinct failure meaning the system itself
  couldn't record all changes — same fallback (full enumeration) applies.
- Network-specific caveat: fails with `ERROR_INVALID_PARAMETER` if the
  buffer is larger than 64 KB while monitoring a directory over the network,
  due to underlying file-sharing-protocol packet size limits.
- `ERROR_NOACCESS` if the buffer isn't DWORD-aligned.
- If the underlying network redirector or file system doesn't support this
  operation at all, fails with `ERROR_INVALID_FUNCTION`.
- Same platform support table as `FindFirstChangeNotification` (SMB 3.0 incl.
  Transparent Failover and Scale-out File Shares, CsvFS, ReFS, as of
  Windows 8 / Server 2012).
- Requirements: Windows XP+ (desktop and UWP apps) / Windows Server 2003+;
  `Kernel32.dll`/`Kernel32.lib`; `winbase.h` via `Windows.h`.

## Concept pages informed

- [`concepts/readdirectorychangesw.md`](../concepts/readdirectorychangesw.md) (new)
- [`concepts/findfirstchangenotification.md`](../concepts/findfirstchangenotification.md) (resolved the "not yet written up" placeholder for this API)
