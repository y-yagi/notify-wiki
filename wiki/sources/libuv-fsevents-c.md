---
title: "Source: libuv src/unix/fsevents.c"
tags: [macos, darwin, userspace-library, source]
updated: 2026-07-09
---

# libuv-fsevents-c

**Raw file**: [`raw/libuv-fsevents-c.txt`](../../raw/libuv-fsevents-c.txt)
**Origin**: https://github.com/libuv/libuv/blob/v1.x/src/unix/fsevents.c —
`src/unix/fsevents.c`, libuv's `v1.x` branch, fetched verbatim via
`raw.githubusercontent.com` 2026-07-09.

## What it is

The actual implementation of libuv's macOS FSEvents backend for
`uv_fs_event_t` — used to check a question the docs alone couldn't answer:
whether libuv's macOS backend uses `kFSEventStreamCreateFlagFileEvents`
(per-file granularity), following up on
[apple-fsevents-h](apple-fsevents-h.md) /
[apple-dev-fseventstreamcreateflagfileevents](apple-dev-fseventstreamcreateflagfileevents.md).

## What it claims

- **libuv does use `kFSEventStreamCreateFlagFileEvents`.** `uv__fsevents_create_stream()`
  (line 354) sets `flags = kFSEventStreamCreateFlagNoDefer | kFSEventStreamCreateFlagFileEvents`
  unconditionally when creating the stream, with a comment: "FileEvents -
  fire callback for file changes too (by default it is firing it only for
  directory changes)."
- The per-event callback (`uv__fsevents_event_cb`, from line 214) receives
  individual file/directory paths per event (via the FileEvents-enabled
  stream), strips the watched root's prefix to compute a relative path, and
  passes that non-empty relative path through to the public
  `uv_fs_event_cb` (line 189: `handle->cb(handle, event->path[0] ? event->path : NULL, event->events, 0)`).
  **This means the public callback's `filename` argument is in fact
  populated on macOS**, not merely `NULL`.
- Confirms the macOS 10.7 floor independently: line 24–27 guards the whole
  FSEvents-based implementation behind
  `TARGET_OS_IPHONE || MAC_OS_X_VERSION_MAX_ALLOWED < 1070`, with the comment
  "macOS prior to 10.7 doesn't provide the full FSEvents API so use kqueue"
  — i.e. libuv itself falls back to [[kqueue]] pre-10.7, exactly the
  threshold where `kFSEventStreamCreateFlagFileEvents` becomes available per
  [apple-dev-fseventstreamcreateflagfileevents](apple-dev-fseventstreamcreateflagfileevents.md).
- `UV_FS_EVENT_RECURSIVE` handling (line 297) filters out subdirectory
  events client-side (by checking for an additional `/` in the relative
  path) when the recursive flag isn't set — i.e. even non-recursive watches
  receive file-level events from the OS and libuv does the directory-scoping
  itself in userspace, rather than the kernel/daemon doing it.

## Concept pages informed

- [`concepts/libuv.md`](../concepts/libuv.md) — **corrects** a prior claim in
  this wiki (added 2026-07-08 from the libuv docs alone) that `filename` is
  "only non-null on Linux and Windows." That exact phrase does not appear
  verbatim in [libuv-fs-event](libuv-fs-event.md) either — likely an
  over-generalization introduced during the original ingest, not a direct
  quote. This source shows the actual macOS implementation does populate
  `filename`, contradicting the earlier claim; flagged as a correction, not
  silently overwritten.
