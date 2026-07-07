---
title: "Source: inotify(7) man page (man7.org)"
tags: [linux, kernel, source]
updated: 2026-07-08
---

# man7-inotify-7

**Raw file**: [`raw/man7-inotify.7.txt`](../../raw/man7-inotify.7.txt)
**Origin**: https://man7.org/linux/man-pages/man7/inotify.7.html (man-pages 6.18, page dated 2026-02-14, fetched from tarball on 2026-05-24)

## What it is

The canonical Linux manual page for the inotify API (section 7, overview page,
not an individual syscall page). Written/maintained by Michael Kerrisk as part
of the man-pages project.

## What it claims

- Describes the four core syscalls (`inotify_init(2)`, `inotify_init1(2)`,
  `inotify_add_watch(2)`, `inotify_rm_watch(2)`) and reading events via `read(2)`.
- Defines `struct inotify_event` layout (`wd`, `mask`, `cookie`, `len`, `name[]`)
  and buffer-sizing/`EINVAL` behavior.
- Enumerates all `IN_*` event bits and watch-creation flags, and which events
  fire for the watched directory itself vs. objects inside it.
- Documents `/proc/sys/fs/inotify/{max_queued_events,max_user_instances,max_user_watches}`
  tunables.
- Lists limitations: no PID/UID attribution, no network filesystem coverage,
  no recursive watching, watch-descriptor/pathname caching burden on the app,
  event coalescing, queue overflow, and the inherent raciness of matching
  `IN_MOVED_FROM`/`IN_MOVED_TO` pairs via `cookie`.
- Lists kernel-version-specific bugs (pre-3.19 `fallocate(2)` silence,
  pre-2.6.16 broken `IN_ONESHOT`, pre-2.6.25 coalescing logic bug, and the
  theoretical watch-descriptor-recycling-vs-pending-events bug still present
  as of the page's writing).
- Includes a full C example program using `inotify_init1(IN_NONBLOCK)` + `poll(2)`.

## Concept pages informed

- [`concepts/inotify.md`](../concepts/inotify.md) (new)
