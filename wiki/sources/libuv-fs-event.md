---
title: "Source: libuv fs_event API documentation"
tags: [libuv, userspace-library, source]
updated: 2026-07-08
---

# libuv-fs-event

**Raw file**: [`raw/libuv-fs-event.txt`](../../raw/libuv-fs-event.txt)
**Origin**: https://docs.libuv.org/en/v1.x/fs_event.html — `uv_fs_event_t`
reference page, libuv v1.x docs.

## What it is

The API reference for libuv's `uv_fs_event_t` handle, the cross-platform
file-watching primitive used by Node.js's `fs.watch()` and many other libuv
consumers.

## What it claims

- `uv_fs_event_t` "uses the best backend for the job on each platform" —
  libuv itself doesn't document which OS mechanism backs it per-platform on
  this page (that's in the libuv source, not this reference), but confirms
  it dispatches to a native mechanism rather than polling by default.
- Only two portable event flags are exposed to callers: `UV_RENAME` and
  `UV_CHANGE` — a deliberately small lowest-common-denominator vocabulary
  compared to the richer event masks of inotify/kqueue/FSEvents/
  ReadDirectoryChangesW that sit underneath it.
- `UV_FS_EVENT_RECURSIVE` is the only flag actually implemented, and only on
  macOS and Windows — `UV_FS_EVENT_WATCH_ENTRY` and `UV_FS_EVENT_STAT` are
  declared in the API but explicitly documented as not implemented on any
  backend.
- Platform quirks called out directly in the reference: on FreeBSD the
  `filename` argument can be `NULL` "due to a kernel bug"; on macOS, events
  collected by the OS immediately *before* `uv_fs_event_start()` is called
  may still be delivered to the callback; AIX's `ahafs` backend is
  per-process and not thread-safe, and misses modify events if only the
  containing directory (not the file) is watched; z/OS's backend doesn't
  notify of file creation/deletion within a watched directory at all.

## Concept pages informed

- [`concepts/libuv.md`](../concepts/libuv.md) (new)
