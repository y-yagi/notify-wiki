---
title: "Source: Deno file_watcher.rs"
tags: [deno, source, rust]
updated: 2026-07-23
---

# deno-file-watcher-rs

**Raw file**: [`raw/deno-file-watcher-rs.txt`](../../raw/deno-file-watcher-rs.txt)
**Origin**: https://github.com/denoland/deno `cli/util/file_watcher.rs`, `main` branch, fetched 2026-07-23.

## What it is

The Rust module that implements Deno's `--watch` flag (used by `deno run
--watch`, `deno test --watch`, `deno bundle --watch`, etc.). It sits on top
of the `notify` crate rather than talking to any OS API directly.

## What it claims

- Built entirely on `notify::RecommendedWatcher` + `RecursiveMode::Recursive`
  — no OS-specific branching in this file at all; platform behavior is
  entirely notify-rs's responsibility.
- `watch_paths_for_file_patterns` deliberately watches **base directories**
  rather than the already-resolved file list, so files created after the
  watcher starts are still detected. Exclude patterns (`PathOrPatternSet`)
  filter both initial registration and live events.
- A custom debounce layer, `DebouncedReceiver`, sits between notify's raw
  events and the rest of Deno: paths accumulate into a `HashSet<PathBuf>`
  and flush after `DEBOUNCE_INTERVAL = Duration::from_millis(200)` of
  inactivity, via a `tokio::select!` loop racing a sleep timer against new
  events.
- Two restart modes, `enum WatcherRestartMode { Automatic, Manual }` — stored
  behind a `Mutex` on `WatcherCommunicator`. `Automatic` restarts immediately
  on any debounced change; `Manual` instead pushes changed paths down a
  channel and lets the caller decide when to restart.
- Shutdown uses synthetic signals for cross-platform consistency: Ctrl+C
  itself is left to the OS's default handler
  (`deno_signals::ctrl_c_allow_default`, to avoid blocking while synchronous
  JS code is running), but if the running JS has installed its own
  SIGINT/SIGTERM handlers, Deno raises the signal synthetically via
  `deno_signals::raise` (works uniformly, including on Windows) and waits up
  to `GRACEFUL_SHUTDOWN_TIMEOUT = Duration::from_millis(500)` before forcing
  termination.

## Concept pages informed

- [`concepts/deno-watch.md`](../concepts/deno-watch.md) (new)
- [`concepts/notify-rs.md`](../concepts/notify-rs.md) (updated: added Deno as
  a concrete consumer, with its debounce/restart-mode layer noted as
  Deno-side, not part of the crate itself)
