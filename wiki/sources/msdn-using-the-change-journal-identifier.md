---
title: "Source: Using the Change Journal Identifier (Microsoft Learn)"
tags: [windows, ntfs, source]
updated: 2026-07-19
---

# msdn-using-the-change-journal-identifier

**Raw file**: [`raw/msdn-using-the-change-journal-identifier.txt`](../../raw/msdn-using-the-change-journal-identifier.txt)
**Origin**: https://learn.microsoft.com/en-us/windows/win32/fileio/using-the-change-journal-identifier
— Win32 conceptual documentation on the change-journal identifier.

## What it is

A short conceptual page explaining the 64-bit journal identifier NTFS
stamps each change journal with, why/when it changes (NTFS version
migration across a dual-boot or removable-media move), and how it's used
together with USNs to read journal records reliably.

## What it claims

- The journal identifier is obtained via `FSCTL_QUERY_USN_JOURNAL`.
- **"To perform this and all other change journal operations, you must
  have system administrator privileges. That is, you must be a member of
  the Administrators group."** — this is an explicit, authoritative
  statement of the admin-rights requirement, previously only recorded in
  `[[usn-journal]]` as an unconfirmed claim sourced from a DevBlogs
  comment thread. This page **confirms** it.
- When an admin deletes and re-creates the journal (e.g. USN nearing its
  max value), USNs restart from zero; when NTFS merely restamps the
  journal with a new identifier (without re-creating it), USN numbering
  continues from the current value instead.
- To read a specific set of records: first get the identifier via
  `FSCTL_QUERY_USN_JOURNAL`, then read records via
  `FSCTL_READ_USN_JOURNAL` — NTFS only returns records valid for the
  identifier supplied, which acts as an integrity check against stale
  reads from a previous journal instance.
- Must scan forward from the oldest (lowest-USN) record to find records
  of interest — no random access by content.

## Concept pages informed

- [`concepts/usn-journal.md`](../concepts/usn-journal.md) — resolves the
  previously-unconfirmed admin-rights bullet to a confirmed, cited claim;
  adds the journal-identifier/USN integrity-check relationship.
