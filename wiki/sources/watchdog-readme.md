---
title: "Source: watchdog README"
tags: [watchdog, userspace-library, source]
updated: 2026-07-20
---

# watchdog-readme

**Raw file**: [`raw/watchdog-readme.md`](../../raw/watchdog-readme.md)
**Origin**: https://github.com/gorakhargosh/watchdog `README.rst`, `master` branch.

## What it is

The project README for `watchdog`, a Python library and shell-utility
(`watchmedo`) for monitoring filesystem events. Requires Python 3.9+.

## What it claims

- Backend mapping per platform: Linux 2.6+ → inotify; macOS → FSEvents *or*
  kqueue; FreeBSD/BSD → kqueue; Windows → ReadDirectoryChangesW (via I/O
  completion ports, or worker threads); an OS-independent polling fallback
  (`PollingObserver`) exists but is explicitly called "slow and not
  recommended."
- On kqueue, warns that the caller must raise the OS file-descriptor limit
  (`ulimit -n`) above the number of files being watched, since kqueue
  consumes one fd per watched file/directory — described as an "inherent
  problem" that makes kqueue "not a very scalable way to monitor a deeply
  nested directory."
- Explicit interoperability caveat: editors like Vim don't modify files
  in-place — they write a backup and swap it in — so watchdog's on-modified
  events never fire for files edited that way unless Vim is reconfigured.
- Explicit interoperability caveat: watching a CIFS network share requires
  manually opting into `PollingObserver`, i.e. watchdog does not
  auto-detect network filesystems and fall back itself.
- `observer.schedule(handler, path, recursive=True)` is a boolean flag at
  schedule time — same shape as [[chokidar]]'s always-on recursion and
  [[readdirectorychangesw]]'s `bWatchSubtree`, unlike [[fsnotify-go]]
  (no recursion primitive) or [[libuv]] (recursive only on macOS/Windows).
- Ships a `watchmedo` CLI/tricks system (YAML-configured event handlers,
  `shell-command` subcommand) as an optional extra (`watchdog[watchmedo]`),
  parallel in spirit to how other libraries in this wiki separate optional
  layers from the core watcher (e.g. [[notify-rs]]'s separate debouncer
  crates).
- Has support for being built/run under free-threaded CPython, but states a
  full thread-safety audit hasn't been completed, specifically calling out
  the macOS FSEvents interface as the affected area.

## Concept pages informed

- [`concepts/watchdog.md`](../concepts/watchdog.md) (new)
