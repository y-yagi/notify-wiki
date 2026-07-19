---
title: "Source: FSCTL_QUERY_USN_JOURNAL (Win32 apps SDK reference)"
tags: [windows, ntfs, source]
updated: 2026-07-19
---

# msdn-fsctl-query-usn-journal-win32

**Raw file**: [`raw/msdn-fsctl-query-usn-journal-win32.txt`](../../raw/msdn-fsctl-query-usn-journal-win32.txt)
**Origin**: https://learn.microsoft.com/en-us/windows/win32/api/winioctl/ni-winioctl-fsctl_query_usn_journal
— Win32 SDK reference for the `FSCTL_QUERY_USN_JOURNAL` control code
(`winioctl.h`).

## What it is

The user-mode counterpart to the WDK DDI page already ingested as
[[msdn-fsctl-query-usn-journal|msdn-fsctl-query-usn-journal]]. Same control
code, different API surface: this page documents calling it via
`DeviceIoControl` from an ordinary application, after opening the volume
with `CreateFile("\\.\X:", ...)`.

## What it claims

- Queries current change-journal info (presence, records, capacity) via
  `DeviceIoControl` with `FSCTL_QUERY_USN_JOURNAL`.
- Volume handle obtained via `CreateFile` on `\\.\X:`; volume must be
  formatted NTFS.
- Windows 8 / Server 2012 support table: SMB 3.0, TFO, and Scale-out File
  Shares all **not** supported; Cluster Shared Volume File System (CsvFS)
  **is** supported, but may produce false positives across CsvFS
  pause/resume.
- Requirements: Windows XP+ / Server 2003+, desktop apps only, header
  `winioctl.h`.
- **Still no privilege/administrator statement on this page either** —
  like the DDI page, it's silent on access control. That statement turned
  out to live on a separate conceptual page (see
  [[msdn-using-the-change-journal-identifier]]).

## Concept pages informed

- [`concepts/usn-journal.md`](../concepts/usn-journal.md) — adds the
  user-mode `DeviceIoControl`/`CreateFile` invocation pattern and the
  SMB/CsvFS support table for `FSCTL_QUERY_USN_JOURNAL`.
