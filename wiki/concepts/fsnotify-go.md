---
title: fsnotify (Go)
tags: [userspace-library, go, cross-platform, event-driven]
updated: 2026-07-08
sources: ["../sources/fsnotify-go-readme.md"]
---

# fsnotify (Go)

## Overview

`fsnotify` is the standard cross-platform filesystem-notification library
for Go. Despite sharing a name with the Linux kernel's [[fsnotify]] backend,
the two are unrelated beyond the name — this is a userspace Go package, one
of several libraries (alongside [[chokidar]] and [[notify-rs]], both of
which cite it as a design inspiration) that wrap native OS mechanisms behind
one API.

## API / semantics

- `fsnotify.NewWatcher()` returns a `Watcher` with `Events` and `Errors`
  channels; `watcher.Add(path)` registers a path.
- Backend-per-platform, more conservative than notify-rs or Chokidar about
  claiming support: inotify (Linux), kqueue (BSD/macOS), ReadDirectoryChangesW
  (Windows), and FEN (illumos) are all marked "Supported." fanotify (Linux
  5.9+), native FSEvents (macOS), USN Journal support (Windows), and a
  generic polling fallback are all explicitly listed as **not yet
  implemented**, each tracked against an open upstream issue rather than
  silently unsupported.
- Does not watch subdirectories automatically — recursive watching is "on
  the roadmap," unlike [[chokidar]], which defaults to recursive watching
  (with a depth limit). notify-rs's README doesn't state its recursive-watch
  default either way — not confirmed here.

## Limitations & gotchas

- Inherits every native backend's blind spots directly and says so plainly:
  no NFS/SMB/`/proc`/`/sys` coverage, "fsnotify requires support from
  underlying OS to work."
- Linux: removing an open file emits `CHMOD` immediately but `REMOVE` only
  once every file descriptor on it is closed — explicitly attributed to
  inotify's own behavior ("not much can be changed about this"), not a
  library bug.
- macOS: without a native FSEvents backend, the library rides on kqueue,
  which (per the kqueue backend's own per-file-descriptor model) means
  Spotlight indexing generates a stream of spurious `Chmod` events; the
  documented workaround is excluding the folder from Spotlight indexing
  rather than a code fix.
- BSD/macOS (kqueue backend): one file descriptor is held per watched file,
  not per directory — watching a directory of N files costs N+1 file
  descriptors, hitting `kern.maxfiles`/`kern.maxfilesperproc` limits faster
  than inotify's watch-based (not fd-based) accounting on Linux.
- Watching individual files (rather than their parent directory) is
  discouraged: editors that write atomically (temp file + rename over the
  original) orphan the watch on the now-nonexistent original inode.

## Platform notes

Notable as the only library in this batch with a working **illumos/Solaris**
backend (FEN — File Events Notification, `port_create`/`port_get`-based),
though the README notes Solaris itself is "currently untested" alongside
Android. This is the closest existing pointer in this wiki toward the FEN
mechanism, which has no dedicated concept page yet.

## Related concepts

- [[inotify]], [[kqueue]], [[readdirectorychangesw]] — supported native
  backends.
- [[fanotify]] — explicitly requested but not yet implemented (tracked
  upstream, not a design rejection).
- [[fsevents]] — macOS's native mechanism, which this library still doesn't
  use directly (falls back to kqueue on macOS instead).
- [[fsnotify]] — the unrelated Linux kernel backend of the same name; see
  that page's Limitations section for the naming collision.
- [[chokidar]] / [[notify-rs]] — both cite this library as a direct
  inspiration.
- [[watchdog]] — Python's equivalent in spirit; unlike fsnotify-go, ships
  recursive watching (`recursive=True`) as a shipped feature rather than a
  roadmap item.
- [[recursive-watching]] — cross-cutting comparison of tree-watching support across all mechanisms/libraries in this wiki.

## Sources

- [fsnotify-go-readme](../sources/fsnotify-go-readme.md)
