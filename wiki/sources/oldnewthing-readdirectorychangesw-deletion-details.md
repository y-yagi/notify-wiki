---
title: "Source: When ReadDirectoryChangesW reports a deletion, how can I learn more about the deleted thing? (The Old New Thing)"
tags: [windows, win32, source]
updated: 2026-07-19
---

# oldnewthing-readdirectorychangesw-deletion-details

**Raw file**: [`raw/oldnewthing-readdirectorychangesw-deletion-details.txt`](../../raw/oldnewthing-readdirectorychangesw-deletion-details.txt)
**Origin**: https://devblogs.microsoft.com/oldnewthing/?p=112116 — "When
ReadDirectoryChangesW reports that a deletion occurred, how can I learn
more about the deleted thing?", Raymond Chen, The Old New Thing (Microsoft
DevBlogs), published 2026-03-06.

## What it is

A blog post answering a reader question about a practical gap in
`ReadDirectoryChangesW`: on a `FILE_ACTION_REMOVED` event, the deleted
item is already gone, so its attributes (file vs. directory, size) can't
be queried — the API gives only the name.

## What it claims

- `ReadDirectoryChangesW` "provides no way to recover information about
  the item that was deleted" — the caller gets a name and nothing else.
- Documented/expected workaround: build an in-memory cache via
  `FindFirstFile`/`FindNextFile` at startup, then use
  `ReadDirectoryChangesW` for incremental updates — querying attributes on
  `FILE_ACTION_ADDED` to populate the cache, so a later
  `FILE_ACTION_REMOVED` can be resolved from the cache instead of the
  (now-missing) file.
- **Race condition in that workaround**: if an item is created and deleted
  fast enough that the cache-update code hasn't yet called
  `GetFileAttributes` when the deletion happens, "it won't be there" —
  the program can never learn what the item originally was.
- **The fix**: `[[readdirectorychangesexw]]` called with
  `ReadDirectoryNotifyExtendedInformation` returns
  `FILE_NOTIFY_EXTENDED_INFORMATION` records that include directory
  metadata (file attributes and size) alongside *both* add and remove
  events — eliminating the race entirely, since the metadata arrives with
  the removal notification itself rather than needing a separate query.
- Selected comment-thread claims (lower confidence — reader discussion,
  not the article body):
  - The `W` suffix denotes the Unicode/"Wide" variant of a Win32 API, as
    opposed to the ANSI `A` variant.
  - A separate long-standing complaint (Igor Levicki): Windows has no
    "file close" notification, forcing developers to poll to detect when
    a written file is finished, with filesystem filter drivers as the
    only (costly, complex) alternative.
  - Counter-point (Harry Johnston): the NTFS change journal ([[usn-journal]])
    does record close operations, but reading it requires admin rights
    for `FSCTL_QUERY_USN_JOURNAL`, and it isn't available on all
    filesystems (ExFAT lacks a journal entirely).

## Concept pages informed

- [`concepts/readdirectorychangesw.md`](../concepts/readdirectorychangesw.md) (added deletion-detail limitation + cache/race-condition workaround)
- [`concepts/readdirectorychangesexw.md`](../concepts/readdirectorychangesexw.md) (added concrete detail on what `FILE_NOTIFY_EXTENDED_INFORMATION` carries and why it fixes the race)
- [`concepts/usn-journal.md`](../concepts/usn-journal.md) (added note on `FSCTL_QUERY_USN_JOURNAL` admin-rights requirement and ExFAT lacking a journal — from comment-thread discussion, flagged as lower confidence)
