---
title: watchdog
tags: [userspace-library, python, cross-platform, event-driven]
updated: 2026-07-20
sources: ["../sources/watchdog-readme.md"]
---

# watchdog

## Overview

`watchdog` is a Python library (3.9+) and companion CLI (`watchmedo`) for
monitoring filesystem events, dispatching to a native backend per platform
and falling back to polling where none is available.

## API / semantics

- `Observer().schedule(handler, path, recursive=True)` — recursion is an
  explicit boolean at schedule time, the same shape as [[chokidar]]'s
  always-on recursive watching and [[readdirectorychangesw]]'s
  `bWatchSubtree`. `FileSystemEventHandler` subclasses receive events (e.g.
  `on_any_event`); the observer also works as a context manager.
- Backend-per-platform: Linux 2.6+ → [[inotify]]; macOS → [[fsevents]] *or*
  [[kqueue]]; FreeBSD/BSD → [[kqueue]]; Windows →
  [[readdirectorychangesw]] (I/O completion ports, or worker threads); an
  OS-independent `PollingObserver` fallback exists but the README calls it
  "slow and not recommended."
- `watchmedo`, an optional CLI extra (`watchdog[watchmedo]`), reads
  `tricks.yaml` files to run YAML-configured event handlers ("tricks") and
  supports a `shell-command` subcommand for running arbitrary commands on
  events.

## Limitations & gotchas

- On the [[kqueue]] backend, the caller must manually raise the process's
  open-file-descriptor limit (`ulimit -n`) above the number of files being
  watched, since kqueue holds one fd per watched file/directory. The README
  calls this an "inherent problem" that makes kqueue "not a very scalable
  way to monitor a deeply nested directory" — the same fd-per-file cost
  documented generically on the [[kqueue]] page, here stated as a concrete
  operational requirement for library users.
- Editors that write-and-swap instead of modifying in place (Vim, by the
  README's own example) never trigger on-modified events, because the file
  watchdog is watching gets replaced rather than written to — an
  application-visible consequence of watching by path/inode rather than by
  content.
- CIFS network shares are not auto-detected; watching one requires manually
  opting into `PollingObserver` instead of the platform-native observer.
- Free-threaded CPython is supported to build/run under, but the README
  states a full thread-safety audit hasn't been completed, specifically
  flagging the macOS FSEvents interface as unaudited.

## Related concepts

- [[inotify]], [[fsevents]], [[kqueue]], [[readdirectorychangesw]] — the
  native mechanisms watchdog dispatches to per platform.
- [[chokidar]], [[notify-rs]], [[fsnotify-go]], [[listen]] — other
  userspace libraries covered in this wiki with the same per-platform
  native-backend-plus-polling-fallback shape; watchdog's Vim/CIFS caveats
  are the most explicit interoperability documentation of any of them.
- [[recursive-watching]] — cross-cutting comparison of tree-watching
  support across all mechanisms/libraries in this wiki; watchdog's
  `recursive=True` flag belongs in the "native recursive watch" row
  alongside chokidar and readdirectorychangesw.

## Sources

- [watchdog-readme](../sources/watchdog-readme.md)
