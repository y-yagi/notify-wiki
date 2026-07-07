---
title: FindFirstChangeNotification
tags: [windows, win32, event-driven]
updated: 2026-07-08
sources: ["../sources/msdn-findfirstchangenotificationa.md"]
---

# FindFirstChangeNotification

## Overview

`FindFirstChangeNotification` is the oldest of the Win32 directory
change-notification APIs (`fileapi.h`, `Kernel32.dll`, available since
Windows XP / Server 2003). It gives an application a waitable handle that
becomes signaled when a change matching a filter occurs in a watched
directory or subtree — but it is a **presence/absence signal only**: it
tells you *that* something changed, never *what*. For change details, Win32
provides a separate, more detailed API: [[readdirectorychangesw]].

## API / semantics

- `FindFirstChangeNotification(lpPathName, bWatchSubtree, dwNotifyFilter)`
  creates the notification handle and starts watching immediately.
  `lpPathName` must be an absolute path (empty/relative is rejected); the
  `\\?\` prefix extends it past `MAX_PATH`.
- `bWatchSubtree`: `FALSE` watches only the named directory's immediate
  contents; `TRUE` watches the whole subtree rooted there. No finer
  granularity (e.g. per-file) is available in this API.
- `dwNotifyFilter` is an OR-able bitmask selecting which kinds of change
  satisfy the wait: `FILE_NOTIFY_CHANGE_FILE_NAME` (create/delete/rename of
  files), `FILE_NOTIFY_CHANGE_DIR_NAME` (create/delete of directories),
  `FILE_NOTIFY_CHANGE_ATTRIBUTES`, `FILE_NOTIFY_CHANGE_SIZE`,
  `FILE_NOTIFY_CHANGE_LAST_WRITE`, `FILE_NOTIFY_CHANGE_SECURITY`. Size and
  last-write-time changes are only detected once the OS actually flushes to
  disk — not synchronously with the write call, and later still under heavy
  caching.
- Usage pattern: pass the returned `HANDLE` to a wait function
  (`WaitForSingleObject`/`WaitForMultipleObjects`). Once satisfied, call
  `FindNextChangeNotification` to re-arm the same handle before waiting
  again. `FindCloseChangeNotification` releases it.
- Symbolic-link handling: if `lpPathName` itself is a symlink, the watch
  follows through to the target. But if a *watched directory* contains
  symlinks, changes made to the link targets are invisible — only changes to
  the symlink directory entries themselves (e.g. creating/deleting the link)
  are reported.

## Limitations & gotchas

- **No change details at all** — no path, no operation type, nothing beyond
  "the filter you asked about was satisfied." An application must rescan and
  diff to find out what actually happened, similar in spirit to [[fsevents]]'s
  directory-level granularity, but even coarser (FSEvents at least gives you
  the changed directory's path; this API doesn't even give you that — you
  already know the path, since you supplied it).
- **Not per-file** — same tree-vs-single-directory tradeoff as `bWatchSubtree`,
  no way to watch an individual file's changes distinct from the directory's.
- **May silently not work over network/remote file systems** — the page
  states notifications "may not be returned" in that case, with no further
  detail on how to detect the failure mode.
- **Symlinked targets inside a watched directory are invisible** — a common
  trap for anything symlink-farm-like (e.g. package manager staging dirs).
- Superseded in practice by `ReadDirectoryChangesW` for anything needing to
  know *what* changed — this API is now mostly useful as a lightweight
  "wake me up" signal when the caller is going to rescan anyway.

## Platform notes

- Minimum supported client: Windows XP (desktop apps only).
- Minimum supported server: Windows Server 2003 (desktop apps only).
- Supported over SMB 3.0 (including Transparent Failover and Scale-out File
  Shares) and ReFS as of Windows 8 / Server 2012; Cluster Shared Volume File
  System (CsvFS) support is qualified — applications may see false positives
  during CsvFS pause/resume.
- `FindFirstChangeNotification` is an encoding-neutral alias resolving to
  `FindFirstChangeNotificationA` or `FindFirstChangeNotificationW` based on
  the `UNICODE` preprocessor constant; mixing the alias with encoding-specific
  code can cause build or runtime mismatches.

## Related concepts

- `[[readdirectorychangesw]]` — the more detailed sibling API Microsoft's own
  docs point to for actual change information (path, operation type),
  including synchronous and asynchronous (overlapped/IOCP) delivery modes.
- `[[fsevents]]` — macOS's directory-level notification API; shares the
  "tells you something changed, not what" model, though FSEvents at least
  reports which directory.
- `[[inotify]]` — Linux's nearest equivalent in spirit (watch-based,
  per-directory or per-file), but inotify's `read()` event stream reports
  the specific change type and, where applicable, the affected filename —
  a level of detail this API lacks entirely.

## Sources

- [msdn-findfirstchangenotificationa](../sources/msdn-findfirstchangenotificationa.md)
