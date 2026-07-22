---
title: Chokidar
tags: [userspace-library, nodejs, event-driven]
updated: 2026-07-08
sources: ["../sources/chokidar-readme.md"]
---

# Chokidar

## Overview

Chokidar is a Node.js file-watching library (originally extracted from the
Brunch build tool in 2012) built on top of Node's core `fs.watch`/
`fs.watchFile`. It exists specifically to normalize the inconsistencies of
those two primitives — duplicate events, missing macOS filenames, raw
`rename`-only event naming — into a consistent `add`/`change`/`unlink`
vocabulary, and to always support recursive watching.

Used directly by [Vite](https://github.com/vitejs/vite)'s dev server for
watching the project root and feeding HMR (`packages/vite/src/node/server/index.ts`,
`packages/vite/src/node/watch.ts` as of the `main` branch); Vite's production
`--build --watch` mode uses Rollup/Rolldown's own watcher instead.

## API / semantics

- `chokidar.watch(paths, options)` returns an `FSWatcher` emitting `add`,
  `addDir`, `change`, `unlink`, `unlinkDir`, `ready`, `raw`, and `error`
  events (plus an `all` convenience event).
- Backend selection: `fs.watch` (event-driven, no polling — Node's own
  wrapper over the platform's native mechanism, e.g. libuv's
  `uv_fs_event_t`) is the default; `fs.watchFile` (interval-based polling)
  is used when explicitly requested via `usePolling`, needed for watching
  over a network, or in other cases where native events don't work.
- Two write-pattern accommodations baked directly into the watcher, not left
  to the caller: `atomic` (editors that write via a temp file then
  rename/move it over the target — collapses the resulting delete+create
  into a single `change` event if the re-add happens within a short window)
  and `awaitWriteFinish` (holds `add`/`change` events until a file's size
  stops changing, for large files written in chunks).

## Limitations & gotchas

- Two distinct file-descriptor exhaustion failure modes documented
  separately: generic `fs` handle exhaustion (fixable with `graceful-fs` or
  by raising `fs.inotify.max_user_watches`) versus `fs.watch`-specific
  handle exhaustion, which has no fix other than falling back to
  `usePolling: true` (resource-intensive).
- `awaitWriteFinish`'s accuracy trades directly against responsiveness — the
  README calls out that the right stability threshold is "heavily dependent
  on the OS and hardware," and setting it high (for accuracy) makes the
  watcher noticeably less responsive.
- As of v4, bundled `fsevents` and glob support were both removed; Chokidar's
  own native-backend involvement is now entirely whatever Node's `fs.watch`
  provides per platform, rather than anything Chokidar controls directly.

## Platform notes

Chokidar itself doesn't talk to OS notification APIs directly — it rides on
Node's `fs.watch`, which is built on libuv's `uv_fs_event_t` (see [[libuv]]
for the platform backend mapping and its own two-flag event vocabulary that
Chokidar layers its `add`/`change`/`unlink` naming on top of).

## Related concepts

- [[libuv]] — the layer immediately underneath Node's `fs.watch`, which
  Chokidar wraps; libuv's platform-mechanism selection and event-flag
  collapsing (`UV_RENAME`/`UV_CHANGE`) are what Chokidar is normalizing
  further.
- [[notify-rs]] — explicitly cites Chokidar as a design inspiration (per the
  notify-rs README); comparable "recursive watching + debounce" goals in a
  different language.
- [[fsnotify-go]] — also cited by notify-rs as a sibling inspiration;
  narrower in scope (no built-in debounce/atomic-write handling, unlike
  Chokidar).
- [[listen]] — Ruby's equivalent in spirit (one callback over several native
  backends, plus a polling fallback), but wraps separate per-platform gems
  rather than one shared library like libuv underneath Chokidar.
- [[watchdog]] — Python's equivalent in spirit (native backend per platform
  plus a polling fallback); watchdog additionally documents concrete
  interoperability gotchas (Vim's write-and-swap pattern, CIFS shares) not
  called out in Chokidar's own README.
- [[recursive-watching]] — cross-cutting comparison of tree-watching support across all mechanisms/libraries in this wiki.

## Sources

- [chokidar-readme](../sources/chokidar-readme.md)
