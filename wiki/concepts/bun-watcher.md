---
title: "Bun --watch / --hot"
tags: [bun, runtime, javascript, rust, userspace-library]
updated: 2026-07-23
sources: ["../sources/bun-watcher-rs.md"]
---

# Bun --watch / --hot

## Overview

Bun's `--watch` and `--hot` flags share a single native, cross-platform
filesystem watcher implemented directly against each OS's low-level
mechanism — no wrapper library in between, which contrasts with
[[deno-watch]]'s complete delegation to [[notify-rs]]. As of the current
source (`src/watcher/`, fetched 2026-07-23) this watcher is a standalone
Rust module dispatching at compile time to `INotifyWatcher`,
`KEventWatcher`, or `WindowsWatcher`; the project's own history shows it was
originally written in Zig (`src/watcher.zig`). `--watch` and `--hot` differ
only in what the shared watcher's callback does with a detected change:
`--watch` restarts the whole process (clean state every run), `--hot` swaps
the affected module(s) in an in-memory cache and re-evaluates them,
preserving state stashed on `globalThis` — useful for keeping an HTTP
server's bound socket alive across reloads.

## API / semantics

- **Not recursive-by-default the way [[notify-rs]] or [[chokidar]] are.**
  Bun does not watch a root path's whole subtree up front. The module
  resolver/bundler calls `Watcher::add_file`/`add_directory` incrementally
  as it walks the import graph, so the watch list only ever contains files
  and directories actually reachable from the entry point, plus their
  parent directories (auto-watched so sibling files created later are still
  caught). `node_modules` and anything outside the top-level project
  directory are excluded.
- The watch list (`WatchList`) is a struct-of-arrays
  (`MultiArrayList<WatchItem>`) addressed by a 16-bit `WatchItemIndex`
  rather than one heap-allocated object per watched item — a memory/
  cache-locality choice not seen in the other libraries covered in this
  wiki.
- Runs on a dedicated background OS thread (`"FileWatcher"`), independent of
  Bun's main event loop; `MAX_COUNT = 128` caps how many events are batched
  into one `on_file_update` call.
- **Linux — `INotifyWatcher`**: one `inotify_add_watch` call per watched
  file/directory (inotify has no native recursion — see [[inotify]] — so
  Bun's incremental per-module watching *is* its substitute for recursion).
  Files and directories get different watch masks (files:
  `IN_EXCL_UNLINK|IN_MOVE_SELF|IN_DELETE_SELF|IN_MOVED_TO|IN_MODIFY`;
  directories add `IN_DELETE|IN_CREATE|IN_ONLYDIR`). To counter `IN_MODIFY`'s
  documented noisiness (also called out on [[inotify]]), after an initial
  `read()` returns fewer than half the expected buffer, Bun does one short
  `ppoll()` wait (default 100µs, tunable via `BUN_INOTIFY_COALESCE_INTERVAL`)
  and a follow-up `read()` to merge closely-spaced events into a single
  batch before dispatching.
- **macOS/FreeBSD — `KEventWatcher`**: the classic kqueue model (see
  [[kqueue]]) — one open fd per watched file/directory, all registered on a
  single shared `kqueue` fd via `EVFILT_VNODE` with
  `NOTE_WRITE|NOTE_RENAME|NOTE_DELETE` (`NOTE_ATTRIB`/`NOTE_LINK` also
  decoded on return). macOS opens watched fds with `O_EVTONLY` specifically
  to avoid holding a real read/write lock on files it only intends to
  observe; FreeBSD has no equivalent flag and falls back to plain
  `O_RDONLY`. Uses the same two-phase coalescing idea as the Linux path: if
  the first `kevent()` call returns fewer than half of `CHANGELIST_COUNT`
  (128), a second `kevent()` with a 100µs timeout gathers more events before
  same-`udata` (same watch-list index) events are merged.
- **Windows — `WindowsWatcher`**: `ReadDirectoryChangesW` (see
  [[readdirectorychangesw]]) via an I/O completion port (IOCP) with
  overlapped I/O, one 64 KB buffer per watched directory, filter
  `FILE_NAME|DIR_NAME|LAST_WRITE|CREATION`. Only directories inside the
  top-level project directory are watchable; anything else is skipped with
  a warning instead of erroring.

## Limitations & gotchas

- Because watching is driven by the module graph rather than a blanket
  recursive watch, a file that exists on disk but is never `import`ed won't
  be watched — a structurally different gap from inotify's or kqueue's own
  non-recursion (those miss unregistered *subdirectories*; Bun's approach
  can additionally miss *reachable directories whose files were never
  imported*).
- The kqueue backend must keep one fd open per watched file for as long as
  it's watched (true only on macOS/FreeBSD — `REQUIRES_FILE_DESCRIPTORS`),
  so Bun carries its own eviction/compaction bookkeeping
  (`evict_list`/`flush_evictions`) to close and reclaim fds. After a
  `swap_remove` reshuffles the backing array, it must explicitly
  re-register the moved entry's kevent `udata` so future events keep
  routing to the right watch-list item — a source comment cites a real
  prior regression (referenced as Bun issue #29524) as the reason this
  rewrite step exists.
- The inotify backend's `read_ptr` continuation logic exists specifically
  because, under high load with many short-named files, a single `read()`
  can return more events than the hardcoded `MAX_COUNT` (128) batch size
  can hold in one pass — a source comment cites a parallel-file-removal
  stress test as the case that surfaced this.

## Platform notes

No single unifying library underneath: Linux and Android share the inotify
backend (grouped together because Android uses the same kernel inotify ABI
as glibc/musl Linux), macOS and FreeBSD share the kqueue backend, and
Windows is separate. `wasm32` is explicitly unsupported
(`compile_error!("Unsupported platform")`). The watcher was rewritten from
Zig (`src/watcher.zig`) into this standalone Rust module at some point
before the 2026-07-23 fetch; the exact migration commit isn't captured here.

## Related concepts

- [[inotify]], [[kqueue]], [[readdirectorychangesw]] — the native mechanisms
  Bun's three platform backends implement directly against, with no library
  in between.
- [[deno-watch]] — contrasting design: Deno delegates entirely to
  [[notify-rs]] and recursively watches whole directories; Bun hand-rolls
  all three platform backends itself and only watches what the module graph
  actually references.
- [[libuv]] — the other cross-platform C-level watcher covered in this
  wiki; unlike Bun's incremental, bundler-driven watch list, libuv's
  `uv_fs_event_t` watches whatever path the caller gives it directly, with
  no awareness of an import graph.
- [[recursive-watching]] — cross-cutting comparison; Bun doesn't fit that
  page's three categories cleanly, since its "recursion" is really
  incremental per-module watching rather than a native subtree primitive or
  a userspace tree-walk of a fixed root.

## Sources

- [bun-watcher-rs](../sources/bun-watcher-rs.md)
