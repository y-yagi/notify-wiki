---
title: "Source: notify-rs README"
tags: [notify-rs, userspace-library, source]
updated: 2026-07-08
---

# notify-rs-readme

**Raw file**: [`raw/notify-rs-readme.md`](../../raw/notify-rs-readme.md)
**Origin**: https://github.com/notify-rs/notify `README.md`, `main` branch.

## What it is

The project README for `notify`, the standard cross-platform filesystem
notification crate in the Rust ecosystem — used by rust-analyzer, zed,
watchexec, cargo-watch, mdBook, deno, and others per the README's own list.

## What it claims

- Backend mapping per platform: Linux/Android → inotify; macOS → FSEvents
  *or* kqueue (selectable via a feature flag — the only library in this
  batch that treats the FSEvents-vs-kqueue choice on macOS as
  user-configurable rather than fixed); Windows → ReadDirectoryChangesW;
  iOS/FreeBSD/NetBSD/OpenBSD/DragonflyBSD → kqueue; all platforms also
  support a polling fallback.
- Explicitly cites Go's fsnotify and Node's Chokidar as direct inspirations,
  created out of "frustration at the non-existence of C/Rust cross-platform
  notify libraries" (originally by Félix Saparelli).
- Ships separate debouncer crates (`notify-debouncer-mini`,
  `notify-debouncer-full`) as opt-in layers rather than baking debounce
  logic into the core `notify` crate itself — a different architectural
  choice from Chokidar, which bakes `awaitWriteFinish`/`atomic` handling
  directly into the main watcher.

## Concept pages informed

- [`concepts/notify-rs.md`](../concepts/notify-rs.md) (new)
