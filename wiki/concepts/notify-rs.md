---
title: notify-rs
tags: [userspace-library, rust, cross-platform, event-driven]
updated: 2026-07-09
sources: ["../sources/notify-rs-readme.md", "../sources/notify-rs-fsevent-rs.md"]
---

# notify-rs

## Overview

`notify` is the de facto standard cross-platform filesystem-notification
crate for Rust, used by rust-analyzer, zed, watchexec, cargo-watch, mdBook,
and others. It was created explicitly out of the absence of a mature
C/Rust cross-platform notify library, citing Go's [[fsnotify-go]] and
Node's [[chokidar]] as direct inspirations.

## API / semantics

- Backend-per-platform: Linux/Android → [[inotify]]; macOS → **either**
  [[fsevents]] or [[kqueue]], selectable via a Cargo feature flag; Windows →
  [[readdirectorychangesw]]; iOS/FreeBSD/NetBSD/OpenBSD/DragonflyBSD →
  [[kqueue]]; a polling fallback is available on all platforms. notify-rs is
  the only library in this batch that exposes the FSEvents-vs-kqueue choice
  on macOS to the caller rather than hard-coding one.
- Debouncing is deliberately kept out of the core `notify` crate: it ships as
  separate opt-in crates, `notify-debouncer-mini` and
  `notify-debouncer-full`, rather than being built into the base watcher the
  way Chokidar's `awaitWriteFinish`/`atomic` options are.
- When the [[fsevents]] backend is selected, notify-rs sets
  `kFSEventStreamCreateFlagFileEvents` unconditionally — per-file granularity
  with no opt-out, source-confirmed in
  [notify-rs-fsevent-rs](../sources/notify-rs-fsevent-rs.md). Contrast with
  [[watchman]], which defaults the same flag on but leaves it configurable.
  Doesn't apply when the kqueue backend is chosen instead.

## Limitations & gotchas

- Because macOS backend choice is a compile-time feature flag, an
  application's exposure to FSEvents-specific caveats (event coalescing,
  historical-event replay from a stored event ID, no synchronous-flush
  guarantee — see [[watchman]]'s cookie-mechanism notes) versus kqueue's own
  caveats (per-file descriptor cost, no native recursive watching) is a
  build-time decision, not something discoverable purely from the crate's
  runtime API.

## Platform notes

See backend mapping above; this is the most explicit multi-backend-per-OS
setup among the libraries covered in this wiki (every other library here
picks one native mechanism per OS with no user-facing choice).

## Related concepts

- [[inotify]], [[fsevents]], [[kqueue]], [[readdirectorychangesw]] — the
  native mechanisms notify-rs dispatches to per platform. The FSEvents
  backend contributes unconditional per-file granularity, discussed above.
- [[chokidar]] — cited inspiration; Node.js equivalent, bakes debounce logic
  into the core watcher rather than splitting it into separate crates.
- [[fsnotify-go]] — cited inspiration; Go equivalent, notably still lacking
  a fanotify or polling backend that notify-rs already has.
- [[recursive-watching]] — cross-cutting comparison of tree-watching support across all mechanisms/libraries in this wiki.

## Sources

- [notify-rs-readme](../sources/notify-rs-readme.md)
- [notify-rs-fsevent-rs](../sources/notify-rs-fsevent-rs.md)
