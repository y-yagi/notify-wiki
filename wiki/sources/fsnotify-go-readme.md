---
title: "Source: fsnotify (Go) README"
tags: [fsnotify-go, userspace-library, source]
updated: 2026-07-08
---

# fsnotify-go-readme

**Raw file**: [`raw/fsnotify-go-readme.md`](../../raw/fsnotify-go-readme.md)
**Origin**: https://github.com/fsnotify/fsnotify `README.md`, `main` branch.

## What it is

The project README for `fsnotify`, the standard Go cross-platform
filesystem-notification library. (Same name as, but otherwise unrelated to,
the Linux kernel's [[fsnotify]] backend — see that page's Limitations
section.)

## What it claims

- Platform/backend table: inotify (Linux, supported), kqueue (BSD/macOS,
  supported), ReadDirectoryChangesW (Windows, supported), FEN (illumos,
  supported — notable as the only library in this batch with a working
  Solaris/illumos backend). fanotify, native FSEvents, USN Journals, and a
  generic polling fallback are all listed as explicitly **not yet
  implemented**, each tracked by an open upstream issue.
- Does not watch subdirectories automatically — recursive watching is
  "on the roadmap" (issue #18), not shipped, unlike Chokidar and notify-rs
  which both default to recursive watching.
- Documents the same NFS/SMB/`/proc`/`/sys` non-coverage already known from
  the inotify man page, framed here as "fsnotify requires support from
  underlying OS to work" rather than an inotify-specific limitation — i.e.
  every backend inherits its host mechanism's blind spots.
- macOS-specific gotcha: without a native FSEvents backend, Spotlight
  indexing generates a stream of spurious Chmod events; documented
  workaround is excluding the watched folder from Spotlight indexing, not a
  library-side fix.
- Linux-specific gotcha, directly reflecting inotify semantics: removing an
  open file emits `CHMOD` immediately and `REMOVE` only once all file
  descriptors on it are closed — described as "the event that inotify
  sends, so not much can be changed about this."
- kqueue-specific gotcha: one file descriptor is held per watched file (not
  per directory), so watching a directory of N files consumes N+1 fds,
  hitting `kern.maxfiles`/`kern.maxfilesperproc` limits faster than
  inotify's watch-based (not fd-based) accounting would.

## Concept pages informed

- [`concepts/fsnotify-go.md`](../concepts/fsnotify-go.md) (new)
- [`concepts/fsnotify.md`](../concepts/fsnotify.md) (name-collision note already
  present; no change needed)
