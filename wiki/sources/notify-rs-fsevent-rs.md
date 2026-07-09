---
title: "Source: notify-rs notify/src/fsevent.rs"
tags: [macos, darwin, userspace-library, source]
updated: 2026-07-09
---

# notify-rs-fsevent-rs

**Raw file**: [`raw/notify-rs-fsevent-rs.txt`](../../raw/notify-rs-fsevent-rs.txt)
**Origin**: https://github.com/notify-rs/notify/blob/main/notify/src/fsevent.rs —
`main` branch, fetched verbatim via `raw.githubusercontent.com` 2026-07-09.
(Note: the crate is now a Cargo workspace; the core `notify` crate's FSEvents
backend lives at `notify/src/fsevent.rs`, not `src/fsevent.rs` as in older
tags.)

## What it is

notify-rs's macOS FSEvents backend implementation (one of two macOS backends
it supports — the other being [[kqueue]], selected via a Cargo feature flag)
— checked to see whether it, like [[libuv]] and Watchman, uses
`kFSEventStreamCreateFlagFileEvents` for per-file granularity.

## What it claims

- **notify-rs sets `kFSEventStreamCreateFlagFileEvents` unconditionally** when
  constructing its `FsEventWatcher` (lines 348-350):
  ```rust
  flags: fs::kFSEventStreamCreateFlagFileEvents
      | fs::kFSEventStreamCreateFlagNoDefer
      | fs::kFSEventStreamCreateFlagWatchRoot,
  ```
  Unlike Watchman's config-gated approach, there is no option to disable this
  — any caller using notify-rs's FSEvents backend gets file-level granularity
  by default, with no opt-out.
- This only applies when the FSEvents backend is actually selected at compile
  time; notify-rs also supports a kqueue backend on macOS via a Cargo feature
  flag (documented in [notify-rs-readme](notify-rs-readme.md) and
  `concepts/notify-rs.md`), which doesn't touch FSEvents at all.

## Concept pages informed

- [`concepts/notify-rs.md`](../concepts/notify-rs.md) — adds a new,
  source-confirmed point: when the FSEvents backend is used, notify-rs
  requests per-file granularity unconditionally, same conclusion as
  [libuv-fsevents-c](libuv-fsevents-c.md) but with no config knob (contrast
  with [watchman-fsevents-cpp](watchman-fsevents-cpp.md), where it's
  default-on but toggleable).
