---
title: "Deno --watch"
tags: [deno, runtime, javascript, userspace-library]
updated: 2026-07-23
sources: ["../sources/deno-file-watcher-rs.md"]
---

# Deno --watch

## Overview

Deno's `--watch` flag (usable with `deno run`, `deno test`, `deno bundle`,
etc., and underlying the lower-level `Deno.watchFs` API) restarts the
running command whenever a watched file changes. Unlike [[bun-watcher]],
Deno implements no OS-level watching of its own: `cli/util/file_watcher.rs`
is a thin wrapper around the [[notify-rs]] crate, adding debouncing, restart
policy, and signal handling on top.

## API / semantics

- Uses notify's `RecommendedWatcher` with `RecursiveMode::Recursive` — the
  platform backend actually doing the work (inotify/kqueue/FSEvents/
  ReadDirectoryChangesW, chosen per OS by notify-rs itself, see
  [[notify-rs]]) is entirely outside this file's concern.
- Watches **base directories**, not the already-resolved file list —
  `watch_paths_for_file_patterns` explicitly prefers directories so files
  created after the watcher starts are still picked up. Exclude patterns
  (`PathOrPatternSet`) filter both initial registration and live events.
- notify's raw events are debounced by a Deno-side `DebouncedReceiver`:
  changed paths accumulate into a `HashSet<PathBuf>` and flush after
  `DEBOUNCE_INTERVAL` (200ms) of inactivity, via a `select!` loop racing a
  timer against new events.
- Two restart modes (`WatcherRestartMode`): `Automatic` restarts immediately
  on any debounced change; `Manual` instead hands changed paths to the
  caller over a channel and lets it decide when to restart.
- Shutdown favors synthetic signals over relying on raw OS process-kill
  semantics: Ctrl+C is left to the OS's default handler
  (`deno_signals::ctrl_c_allow_default`) so it isn't swallowed while
  synchronous JS is running, but if the running script installed its own
  SIGINT/SIGTERM handlers, Deno raises the signal synthetically
  (`deno_signals::raise`, works uniformly including on Windows) and waits up
  to `GRACEFUL_SHUTDOWN_TIMEOUT` (500ms) before force-terminating.

## Limitations & gotchas

- Per the original `--watch` PR (#5955, 2020), notify's `RecommendedWatcher`
  could historically delay change detection by up to ~10 seconds under rapid
  repeated edits — the file change itself was detected promptly, but
  recompiling lagged. This report predates the debounce/restart-mode design
  described above; not confirmed here whether the underlying delay was ever
  fully resolved or just made less visible by the newer debounce layer.
- Because Deno delegates entirely to notify-rs, every platform caveat
  documented on [[inotify]], [[kqueue]], [[fsevents]], and
  [[readdirectorychangesw]] (coalescing, no native recursion on Linux/kqueue,
  etc.) applies transitively to `--watch`, with no Deno-specific mitigation
  beyond the 200ms debounce window.

## Platform notes

No Deno-specific platform branching in `file_watcher.rs` — entirely
inherited from whichever backend notify-rs selects per OS (see
[[notify-rs]]'s backend table).

## Related concepts

- [[notify-rs]] — the crate Deno's watcher is built directly on top of; Deno
  adds debouncing, restart-mode selection, and synthetic-signal shutdown, but
  no watching logic of its own.
- [[bun-watcher]] — contrasting design: Bun implements its own per-platform
  watcher directly against inotify/kqueue/ReadDirectoryChangesW instead of
  delegating to a library, and builds its watch list incrementally from the
  module graph rather than recursively watching a root directory.
- [[recursive-watching]] — cross-cutting comparison; Deno's recursive watch
  is inherited wholesale from notify-rs's `RecursiveMode::Recursive`, not
  implemented by Deno itself.

## Sources

- [deno-file-watcher-rs](../sources/deno-file-watcher-rs.md)
