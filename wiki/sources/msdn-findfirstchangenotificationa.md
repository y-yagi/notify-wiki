---
title: "Source: FindFirstChangeNotificationA function (Microsoft Learn)"
tags: [windows, win32, source]
updated: 2026-07-08
---

# msdn-findfirstchangenotificationa

**Raw file**: [`raw/msdn-findfirstchangenotificationa.txt`](../../raw/msdn-findfirstchangenotificationa.txt)
**Origin**: https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-findfirstchangenotificationa — Win32 API reference page for `FindFirstChangeNotificationA` (fileapi.h). Last updated 2024-11-20 per the page footer.

## What it is

The official Win32 API reference for `FindFirstChangeNotificationA`
(and, implicitly, its Unicode sibling `FindFirstChangeNotificationW` via the
`FindFirstChangeNotification` encoding-neutral alias) — the oldest of
Windows' directory change-notification mechanisms. Reference-style: syntax,
parameters, return value, remarks, platform support table, requirements.

## What it claims

- Creates a **change notification handle**: waiting on it (via `WaitForSingleObject`
  or similar) succeeds once a change matching the registered filter occurs in
  the watched directory or subtree. It does **not** report changes to the
  watched directory itself, only its contents.
- **Does not report what changed** — only that *something* matching the
  filter changed. To get the actual change details (path, type of change),
  the page explicitly directs you to `ReadDirectoryChangesW` instead.
- Filter conditions (`dwNotifyFilter`, OR-able bitmask): `FILE_NOTIFY_CHANGE_FILE_NAME`,
  `FILE_NOTIFY_CHANGE_DIR_NAME`, `FILE_NOTIFY_CHANGE_ATTRIBUTES`,
  `FILE_NOTIFY_CHANGE_SIZE`, `FILE_NOTIFY_CHANGE_LAST_WRITE`,
  `FILE_NOTIFY_CHANGE_SECURITY`. Size/last-write detection depends on when
  the OS actually flushes to disk, not on the write call itself.
- Watch model: single directory (`bWatchSubtree = FALSE`) or a full subtree
  rooted at that directory (`bWatchSubtree = TRUE`) — no per-file granularity.
- After a wait is satisfied, the caller must call `FindNextChangeNotification`
  to re-arm before waiting again; `FindCloseChangeNotification` releases the
  handle.
- Symbolic-link behavior: watching a path that is itself a symlink watches
  the **target**; but if a watched directory *contains* symlinks, changes to
  the linked-to targets are invisible — only changes to the symlink entries
  themselves are reported.
- May not report notifications at all for remote/network file systems.
- Requirements: Windows XP+ (desktop apps only) / Windows Server 2003+;
  `Kernel32.dll`/`Kernel32.lib`; `fileapi.h` via `Windows.h`.

## Concept pages informed

- [`concepts/findfirstchangenotification.md`](../concepts/findfirstchangenotification.md) (new)
