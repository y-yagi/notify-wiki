---
title: libuv (uv_fs_event_t)
tags: [userspace-library, cross-platform, event-driven]
updated: 2026-07-08
sources: ["../sources/libuv-fs-event.md"]
---

# libuv (uv_fs_event_t)

## Overview

libuv is the C event-loop library underlying Node.js (also used standalone by
other projects). Its `uv_fs_event_t` handle is a cross-platform file-watching
primitive: give it a path, get a callback on changes, without the caller
having to know which native OS mechanism is doing the work underneath.

## API / semantics

- `uv_fs_event_init()` / `uv_fs_event_start(handle, cb, path, flags)` /
  `uv_fs_event_stop()` — standard libuv handle lifecycle.
- The callback receives `(handle, filename, events, status)`. `filename` is
  only populated on Linux and Windows ("Only non-null on Linux and Windows"
  per the reference) — reflecting that inotify and ReadDirectoryChangesW both
  report a changed filename within a watched directory, while other backends
  (kqueue-based ones) don't give libuv one to pass through.
- `events` is one of only two portable values: `UV_RENAME` or `UV_CHANGE` —
  every native backend's richer event vocabulary (inotify's `IN_*` bits,
  kqueue's `NOTE_*` filters, FSEvents' flags, ReadDirectoryChangesW's
  `FILE_ACTION_*`) is collapsed down to this pair before reaching the caller.
- `UV_FS_EVENT_RECURSIVE` is the only flag actually implemented, and only on
  macOS and Windows — both platforms whose native mechanism ([[fsevents]] and
  [[readdirectorychangesw]] respectively) can watch a subtree without the
  caller re-registering per directory. On Linux, where [[inotify]] has no
  native recursive mode, libuv doesn't attempt to emulate one by walking the
  directory tree itself; recursive watching is simply unavailable.

## Limitations & gotchas

- The two-flag `UV_RENAME`/`UV_CHANGE` vocabulary means anything built
  directly on libuv (e.g. Node's `fs.watch`) inherits a lower-resolution
  event model than talking to the native OS API directly — callers that need
  to distinguish create vs. delete vs. attribute-change vs. move-from/
  move-to must either re-derive it by `stat()`-ing before and after, or go
  around libuv to the native API.
- Documented per-platform quirks are all native-mechanism leakage, not libuv
  bugs: FreeBSD kqueue can hand back a `NULL` filename ("due to a kernel
  bug"); on macOS, FSEvents events queued immediately *before*
  `uv_fs_event_start()` is called can still arrive at the callback (FSEvents'
  own coalescing/latency window bleeding through); AIX's `ahafs` backend is
  per-process and not thread-safe, and misses modify events when only a
  containing directory (not the file itself) is watched; z/OS's backend
  never reports file creation/deletion within a watched directory at all.

## Platform notes

libuv picks "the best backend for the job on each platform" but the
reference page doesn't enumerate the mapping itself — based on the
behavior documented (recursive support on macOS/Windows only, filename
population on Linux/Windows only), it lines up with [[inotify]] on Linux,
some native mechanism on macOS (see [[fsevents]]), [[kqueue]] on
BSDs, and [[readdirectorychangesw]] on Windows, plus AIX's `ahafs` and a
z/OS-specific backend not covered elsewhere in this wiki.

## Related concepts

- [[inotify]] — Linux backend; contributes the filename-in-callback behavior.
- [[fsevents]] — macOS backend; contributes the recursive-watch support and
  the pre-start-event-delivery quirk.
- [[kqueue]] — BSD backend; contributes the FreeBSD null-filename bug.
- [[readdirectorychangesw]] — Windows backend; contributes filename-in-
  callback and recursive-watch support.
- [[chokidar]] — a comparable "smooth over native event differences"
  library, but for Node's `fs.watch`/`fs.watchFile` rather than libuv
  directly (Node's `fs.watch` is itself built on libuv's `uv_fs_event_t`).
- [[recursive-watching]] — cross-cutting comparison of tree-watching support across all mechanisms/libraries in this wiki.

## Sources

- [libuv-fs-event](../sources/libuv-fs-event.md)
