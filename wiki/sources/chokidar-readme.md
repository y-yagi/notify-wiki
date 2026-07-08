---
title: "Source: Chokidar README"
tags: [chokidar, userspace-library, source]
updated: 2026-07-08
---

# chokidar-readme

**Raw file**: [`raw/chokidar-readme.md`](../../raw/chokidar-readme.md)
**Origin**: https://github.com/paulmillr/chokidar `README.md`, `master`
branch.

## What it is

The project README for Chokidar, a Node.js file-watching library built on
top of Node's core `fs.watch`/`fs.watchFile`, used in roughly 30 million
repositories per the README's own count.

## What it claims

- Chokidar exists to paper over `fs.watch`/`fs.watchFile` inconsistencies:
  normalizing duplicate events, reporting macOS filenames (raw `fs.watch` on
  macOS historically didn't), translating raw `rename` events into
  `add`/`change`/`unlink`, and always supporting recursive watching (with a
  depth limit) instead of the partial/inconsistent recursive support of raw
  platform events.
- `fs.watch` (event-driven, no polling) is the default backend; `fs.watchFile`
  (polling-based, higher resource use) is used as a fallback for cases like
  watching over a network, where `fs.watch`'s underlying native
  inotify/FSEvents/ReadDirectoryChangesW mechanisms don't work.
- Ships two write-pattern accommodations that raw fs events don't handle
  well: `atomic` (editors that write via a temp file + rename/move — treated
  as a single `change` rather than `unlink`+`add` if re-added within a short
  window) and `awaitWriteFinish` (holds `add`/`change` events until a file's
  size stops changing, for large files written in chunks).
- Troubleshooting section documents running out of file descriptors
  (`EMFILE`/`ENOSPC`) as a known operational failure mode, distinguishing
  "generic fs handle exhaustion" (fixable via `graceful-fs` or raising
  `fs.inotify.max_user_watches`) from `fs.watch`-specific handle exhaustion
  (not fixable the same way; falling back to polling is the only documented
  workaround).
- v4+ dropped bundled `fsevents` and glob support; from v4 the library's own
  native-backend usage is limited to whatever Node.js's `fs.watch` itself
  provides per platform.

## Concept pages informed

- [`concepts/chokidar.md`](../concepts/chokidar.md) (new)
