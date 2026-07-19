---
title: "Source: FSCTL_READ_USN_JOURNAL (Win32 apps SDK reference)"
tags: [windows, ntfs, source]
updated: 2026-07-19
---

# msdn-fsctl-read-usn-journal

**Raw file**: [`raw/msdn-fsctl-read-usn-journal.txt`](../../raw/msdn-fsctl-read-usn-journal.txt)
**Origin**: https://learn.microsoft.com/en-us/windows/win32/api/winioctl/ni-winioctl-fsctl_read_usn_journal
— Win32 SDK reference for the `FSCTL_READ_USN_JOURNAL` control code
(`winioctl.h`).

## What it is

The read/retrieve counterpart to `FSCTL_QUERY_USN_JOURNAL`: retrieves the
actual change-journal records between two USN values, via
`DeviceIoControl`.

## What it claims

- Two control codes return USN records: `FSCTL_READ_USN_JOURNAL` (select
  by USN / criteria) vs. `FSCTL_ENUM_USN_DATA` (unfiltered enumeration
  between two USNs) — use the former for selective reads, the latter for
  a full listing.
- Same volume-handle pattern as `FSCTL_QUERY_USN_JOURNAL`: `CreateFile`
  on `\\.\X:`; volume must be NTFS.
- Same Windows 8 / Server 2012 support table as the query control code:
  SMB 3.0 / TFO / Scale-out File Shares not supported; CsvFS supported
  with the same pause/resume false-positive caveat.
- Requirements: Windows XP+ / Server 2003+, desktop apps only, header
  `winioctl.h` — identical to `FSCTL_QUERY_USN_JOURNAL`'s requirements.
- Points to [[msdn-walking-a-buffer-of-change-journal-records]] for a
  full worked example.

## Concept pages informed

- [`concepts/usn-journal.md`](../concepts/usn-journal.md) — adds the
  `FSCTL_READ_USN_JOURNAL` vs `FSCTL_ENUM_USN_DATA` distinction and its
  Win32 invocation/support details.
