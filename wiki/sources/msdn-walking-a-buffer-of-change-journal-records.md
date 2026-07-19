---
title: "Source: Walking a Buffer of Change Journal Records (Microsoft Learn)"
tags: [windows, ntfs, source]
updated: 2026-07-19
---

# msdn-walking-a-buffer-of-change-journal-records

**Raw file**: [`raw/msdn-walking-a-buffer-of-change-journal-records.txt`](../../raw/msdn-walking-a-buffer-of-change-journal-records.txt)
**Origin**: https://learn.microsoft.com/en-us/windows/win32/fileio/walking-a-buffer-of-change-journal-records
— Win32 conceptual documentation with a full C code example.

## What it is

A how-to page showing the output-buffer layout shared by
`FSCTL_READ_USN_JOURNAL` and `FSCTL_ENUM_USN_DATA`, and a complete C
example (`CreateFile` → `FSCTL_QUERY_USN_JOURNAL` → loop of
`FSCTL_READ_USN_JOURNAL`) that walks the returned `USN_RECORD` buffer.

## What it claims

- Both `FSCTL_READ_USN_JOURNAL` and `FSCTL_ENUM_USN_DATA` return a
  leading USN followed by zero or more `USN_RECORD_V2`/`USN_RECORD_V3`
  records; use `FSCTL_ENUM_USN_DATA` for an unfiltered listing between
  two USNs, `FSCTL_READ_USN_JOURNAL` to select more specifically (e.g.
  by change reason, or only-on-close).
- **"The target volume for USN operations must be ReFS or NTFS 3.0 or
  later."** — this names ReFS as a supported filesystem, which is in
  tension with `[[usn-journal]]`'s existing Platform-notes claim
  ("NTFS-specific — not available on FAT/exFAT/ReFS"), sourced from
  `fsutil usn` docs. **Flagging as a discrepancy, not silently
  resolving it** — the two sources may be describing different points
  in time or different feature sets (e.g. ReFS support could be a later
  addition), but this needs the user's attention.
- `USN_RECORD_V2`/`V3` are variable-length: `RecordLength` (first member)
  gives the total struct size including the trailing file name;
  `FileName`/`FileNameLength` give the name, with **no guaranteed
  trailing null terminator** — length must come from `FileNameLength`,
  not string scanning.
- Buffer-walk pattern: leading `USN` (8 bytes) to skip, then repeatedly
  advance a `PUSN_RECORD` pointer by `RecordLength` bytes until the
  returned byte count is exhausted; the final USN in the buffer becomes
  `StartUsn` for the next call (pagination/continuation cursor).
- Max single-record size formula: `(MaxComponentLength - 1) * sizeof(WCHAR)
  + sizeof(USN_RECORD)`, where `MaxComponentLength` comes from
  `GetVolumeInformation`.
- Full example calls `FSCTL_QUERY_USN_JOURNAL` once to get the journal ID
  and `FirstUsn`, then loops `FSCTL_READ_USN_JOURNAL` reading into a
  4096-byte buffer.

## Concept pages informed

- [`concepts/usn-journal.md`](../concepts/usn-journal.md) — adds
  `USN_RECORD_V2`/`V3` structural detail, the buffer-walk/continuation
  pattern, and surfaces the ReFS-support discrepancy against the
  existing Platform-notes claim.
