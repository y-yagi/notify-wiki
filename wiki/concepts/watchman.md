---
title: Watchman
tags: [userspace-library, cross-platform, event-driven]
updated: 2026-07-08
sources: ["../sources/watchman-cookies.md", "../sources/watchman-readme.md"]
---

# Watchman

## Overview

Watchman is a standalone file-watching *service* (not a library linked into
your process) built and maintained by Meta's source-control team. A single
`watchman` daemon watches roots on behalf of multiple client processes over a
socket protocol, and can trigger commands or answer queries about what
changed since a previous checkpoint ("clockspec"). Its main design problem —
and the one its most substantive public documentation addresses — isn't
picking a native OS mechanism, but guaranteeing query results are
synchronized with the notification stream.

## API / semantics

- Clients talk to the daemon over a socket/CLI, e.g. `watchman watch <dir>`
  to establish a watch and `watchman -- trigger <dir> <name> <glob> --
  <cmd>` to run a command on matching changes. Multiple clients/tools can
  share one daemon's watch on a root instead of each opening their own OS-
  level watch — the resource-sharing pitch relative to every process running
  its own inotify/FSEvents/kqueue watcher independently.
- **Cookies**: to answer "what changed since I last asked" correctly,
  Watchman creates a uniquely-named temporary file inside the watched root
  (or inside `.git`/`.hg`/`.svn` if present, to avoid polluting VCS scans)
  for every query, then blocks the query until that cookie's own creation
  event comes back through the OS notification stream. Because most native
  notification APIs preserve delivery order, seeing the cookie event implies
  every filesystem operation that happened before the query started has
  already been observed and processed.
- This depends on the underlying OS mechanism actually preserving order and
  reporting exactly what changed — which holds for [[inotify]] but is
  explicitly weaker on other platforms (see Limitations).

## Limitations & gotchas

- **FSEvents cannot fully support the cookie guarantee.** Watchman combines
  cookie files with `FSEventStreamFlushSync`, but under high load (their
  example: a large `git checkout`), FSEvents-reported changes have been
  observed arriving *after* both the cookie notification and the flush call
  returned — i.e. FSEvents offers no synchronous-flush guarantee despite the
  API's name. Apple's own suggested fix is to reimplement watching on
  Endpoint Security instead; Watchman's fallback is a `settle_period`/
  `settle_timeout` quiescence heuristic rather than a hard guarantee.
- On any platform where the OS only reports "something in this directory
  changed" without saying what changed (unlike Linux inotify, which reports
  the specific child), cookie detection can miss if not every intervening
  event was individually observed — documented as "currently an unresolved
  issue."
- Watchman does not follow symlinks.
- Motivation for the whole cookie mechanism was a real production failure
  mode: with 16+ parallel Mercurial test-suite runs, Watchman would fall
  behind and answer with stale data before cookies existed.

## Platform notes

Officially supported on Windows, macOS, and recent Ubuntu/Fedora Linux
(maintained by Meta's source-control team); FreeBSD and Solaris support are
community-maintained, not official — notable since a Solaris/illumos native
watch mechanism ([[fen]]) is a documented gap elsewhere in this wiki.

## Related concepts

- [[fsevents]] — the macOS backend whose lack of a synchronous-flush
  guarantee is the specific, sourced limitation documented above.
- [[inotify]] — the backend whose ordering guarantee the cookie mechanism
  depends on to work correctly.
- [[chokidar]] / [[notify-rs]] / [[fsnotify-go]] — in-process libraries
  solving a narrower problem (one process, one watch) rather than Watchman's
  shared-daemon-plus-query-synchronization model.
- [[recursive-watching]] — cross-cutting comparison of tree-watching support across all mechanisms/libraries in this wiki.

## Sources

- [watchman-cookies](../sources/watchman-cookies.md)
- [watchman-readme](../sources/watchman-readme.md)
