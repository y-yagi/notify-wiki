---
title: "Source: FSCTL_QUERY_USN_JOURNAL (Windows Driver Kit DDI reference)"
tags: [windows, ntfs, source]
updated: 2026-07-19
---

# msdn-fsctl-query-usn-journal

**Raw file**: [`raw/msdn-fsctl-query-usn-journal.txt`](../../raw/msdn-fsctl-query-usn-journal.txt)
**Origin**: https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/ntifs/ni-ntifs-fsctl_query_usn_journal
— WDK DDI reference for the `FSCTL_QUERY_USN_JOURNAL` control code
(`ntifs.h`).

## What it is

The official driver-level (WDK) reference for the `FSCTL_QUERY_USN_JOURNAL`
control code — the low-level mechanism `fsutil usn queryjournal`
([[usn-journal]]) sits on top of. Short page (~160 words): major code name,
invocation parameters, return value, and a minimal requirements table.

## What it claims

- Queries current change-journal info: presence, its records, its
  capacity.
- Invoked via one of two **kernel-mode** APIs, not directly from
  user-mode application code:
  - `FltFsControlFile` — called from within a minifilter driver, taking a
    `FileObject` pointer for the volume.
  - `ZwFsControlFile` — the kernel-mode native API, taking a `FileHandle`
    for the volume.
- `InputBuffer`/`InputBufferLength` are unused for this control code.
  `OutputBuffer` receives a pointer to a `USN_JOURNAL_DATA` structure
  (fields not detailed on this page — a separate reference).
- Returns `STATUS_SUCCESS` or an NTSTATUS error code.
- Requirements: Minimum supported client Windows XP; header `ntifs.h`; no
  minimum server version or FAT/exFAT/ReFS support notes given.

## What it does *not* say (important for cross-checking a prior claim)

- **No mention of any administrator/privilege requirement anywhere on the
  page.** This directly bears on a lower-confidence claim already
  recorded in `[[usn-journal]]` (sourced from a Microsoft DevBlogs
  comment thread) that `FSCTL_QUERY_USN_JOURNAL` "needs administrator
  privileges to call." This page neither confirms nor denies that —
  it's simply silent on access control entirely.
- This silence is plausibly explained by scope mismatch rather than
  contradiction: this page documents the **kernel-mode** invocation
  surface (`FltFsControlFile` inside a driver, or `ZwFsControlFile` from
  kernel-mode code), whereas the blog comment's claim was almost
  certainly about the ordinary **user-mode** path — calling
  `DeviceIoControl` with `FSCTL_QUERY_USN_JOURNAL` from an application —
  which this DDI page doesn't describe or reference at all. Access-control
  behavior for that user-mode path (e.g. required handle access rights
  when opening the volume via `CreateFile`) is not covered here.
- Does not detail `USN_JOURNAL_DATA`'s fields.

## Concept pages informed

- [`concepts/usn-journal.md`](../concepts/usn-journal.md) (added this
  page's kernel-mode-vs-user-mode distinction; did **not** resolve the
  existing lower-confidence admin-rights claim — flagged as still
  unconfirmed, now with an explanation for why this source couldn't
  settle it)
