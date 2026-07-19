---
title: "Source: ReadDirectoryChangesExW function (Microsoft Learn)"
tags: [windows, win32, source]
updated: 2026-07-19
---

# msdn-readdirectorychangesexw

**Raw file**: [`raw/msdn-readdirectorychangesexw.txt`](../../raw/msdn-readdirectorychangesexw.txt)
**Origin**: https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-readdirectorychangesexw — Win32 API reference page for `ReadDirectoryChangesExW` (winbase.h).

## What it is

The official Win32 API reference for `ReadDirectoryChangesExW`, a newer
sibling of [[readdirectorychangesw]] that adds a selectable output
structure — plain `FILE_NOTIFY_INFORMATION` or the richer
`FILE_NOTIFY_EXTENDED_INFORMATION` — via an extra
`ReadDirectoryNotifyInformationClass` parameter.

## What it claims

- Signature adds one parameter beyond `ReadDirectoryChangesW`:
  `[in] READ_DIRECTORY_NOTIFY_INFORMATION_CLASS ReadDirectoryNotifyInformationClass`.
  All other parameters (`hDirectory`, `lpBuffer`, `nBufferLength`,
  `bWatchSubtree`, `dwNotifyFilter`, `lpBytesReturned`, `lpOverlapped`,
  `lpCompletionRoutine`) are identical in name and meaning to
  `ReadDirectoryChangesW`.
- `ReadDirectoryNotifyInformationClass` selects the buffer's record format:
  `ReadDirectoryNotifyInformation` for the classic `FILE_NOTIFY_INFORMATION`
  structure (same as `ReadDirectoryChangesW`), or
  `ReadDirectoryNotifyExtendedInformation` for `FILE_NOTIFY_EXTENDED_INFORMATION`
  (not detailed in this source — documented on its own reference page).
- `dwNotifyFilter` bitmask values are identical to `ReadDirectoryChangesW`'s
  (same FILE_NAME/DIR_NAME/ATTRIBUTES/SIZE/LAST_WRITE/LAST_ACCESS/CREATION/SECURITY set).
- Synchronous/asynchronous operation model, the three async-completion
  mechanisms (`GetOverlappedResult`, `GetQueuedCompletionStatus`, completion
  routine), buffer-overflow behavior (`TRUE` return but `lpBytesReturned == 0`,
  full contents discarded), the 64 KB network buffer cap
  (`ERROR_INVALID_PARAMETER`), DWORD-alignment requirement
  (`ERROR_NOACCESS`), and `ERROR_NOTIFY_ENUM_DIR` are all reproduced
  verbatim/identically from `ReadDirectoryChangesW`'s page.
- **New constraint not present on `ReadDirectoryChangesW`**: "ReadDirectoryChangesExW
  is currently supported only for the NTFS file system." No SMB/CsvFS/ReFS
  support table is given (unlike `ReadDirectoryChangesW`'s page, which lists
  SMB 3.0/TFO/Scale-out/CsvFS/ReFS support as of Windows 8/Server 2012) —
  this source doesn't say whether that's an oversight or a genuine
  narrowing of support.
- Requirements: **Windows 10 version 1709+ (desktop apps only)** / **Windows
  Server 2019+ (desktop apps only)** — notably later and narrower (no UWP)
  than `ReadDirectoryChangesW`'s Windows XP / Server 2003 (desktop + UWP).
  `Kernel32.dll`/`Kernel32.lib`; `winbase.h` via `Windows.h`.

## Concept pages informed

- [`concepts/readdirectorychangesexw.md`](../concepts/readdirectorychangesexw.md) (new)
- [`concepts/readdirectorychangesw.md`](../concepts/readdirectorychangesw.md) (added cross-reference to the newer Ex variant)
