---
title: "Source: Bun src/watcher/"
tags: [bun, source, rust]
updated: 2026-07-23
---

# bun-watcher-rs

**Raw files**:
[`raw/bun-watcher-rs.txt`](../../raw/bun-watcher-rs.txt) (`Watcher.rs`, platform-independent core),
[`raw/bun-inotifywatcher-rs.txt`](../../raw/bun-inotifywatcher-rs.txt) (`INotifyWatcher.rs`, Linux/Android),
[`raw/bun-keventwatcher-rs.txt`](../../raw/bun-keventwatcher-rs.txt) (`KEventWatcher.rs`, macOS/FreeBSD),
[`raw/bun-windowswatcher-rs.txt`](../../raw/bun-windowswatcher-rs.txt) (`WindowsWatcher.rs`, Windows).
**Origin**: https://github.com/oven-sh/bun `src/watcher/`, `main` branch, fetched 2026-07-23.

## What it is

Bun's cross-platform filesystem watcher, the shared engine behind both
`bun --watch` and `bun --hot`. Historically implemented in Zig
(`src/watcher.zig`, per the project's own migration commentary); the current
source tree has this rewritten as a standalone Rust module dispatching to
one of three platform backends at compile time via `#[cfg(target_os = ...)]`.

## What it claims

- No wrapper library: each backend calls its OS's native API directly.
  Linux/Android use `inotify_add_watch`/`read()` on an `inotify_init1` fd;
  macOS/FreeBSD register `EVFILT_VNODE` kevents (one fd per watched
  file/directory) on a shared `kqueue` fd; Windows uses
  `ReadDirectoryChangesW` through an I/O completion port with overlapped I/O.
- Watching is **not** a single recursive watch on a root path. The watch
  list is built incrementally as Bun's module resolver/bundler walks the
  import graph — files and directories are added to `Watcher` only once
  actually referenced, with `node_modules` and anything outside the
  top-level directory excluded.
- The watch list itself is a struct-of-arrays (`MultiArrayList<WatchItem>`),
  addressed by a 16-bit `WatchItemIndex` rather than one heap object per
  watched item.
- Both the Linux and macOS/FreeBSD backends implement a two-phase read to
  coalesce bursts of events: after an initial read/kevent call returns fewer
  than half of the expected batch, a second call with a short timeout
  (100µs by default; `BUN_INOTIFY_COALESCE_INTERVAL` env var on Linux) waits
  for more events before events sharing the same watch-list index are merged
  and dispatched together.
- `--watch` and `--hot` share this exact watcher; they differ only in what
  the `on_file_update` callback does — full process restart for `--watch`,
  in-place module-cache invalidation (preserving `globalThis` state) for
  `--hot`.
- kqueue-backend fds must stay open for as long as an item is watched, so
  the code carries explicit eviction/compaction bookkeeping
  (`evict_list`/`flush_evictions`), including re-registering a moved
  watchlist entry's kevent `udata` after an array `swap_remove` — a source
  comment cites a real prior regression, referenced as issue #29524.

## Concept pages informed

- [`concepts/bun-watcher.md`](../concepts/bun-watcher.md) (new)
