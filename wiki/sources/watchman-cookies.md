---
title: "Source: Watchman — Query Synchronization (cookies.md)"
tags: [watchman, userspace-library, source]
updated: 2026-07-08
---

# watchman-cookies

**Raw file**: [`raw/watchman-cookies.md`](../../raw/watchman-cookies.md)
**Origin**: https://github.com/facebook/watchman `website/docs/cookies.md`
("Query Synchronization", category: Internals), fetched from the `main`
branch.

## What it is

A design doc explaining Watchman's *cookie* mechanism: how it guarantees a
query sees all filesystem operations that happened before the query started,
despite the underlying OS notification API only guaranteeing eventual,
not synchronous, delivery.

## What it claims

- A cookie is a uniquely-named temporary file created (under the watched
  root's lock) inside the watched root — or inside a VCS directory
  (`.git`/`.hg`/`.svn`) if present, so cookies don't pollute working-copy
  scans. The calling thread blocks until the root's notify thread observes
  the cookie's own creation event coming back through the OS notification
  stream, which — because most notification APIs preserve ordering — implies
  every earlier event has already been processed.
- This directly depends on OS-level ordering guarantees: "File monitoring
  systems like inotify typically provide an ordering guarantee: notifications
  arrive in the order they happen." Cookies are the userspace synchronization
  primitive layered on top of that guarantee.
- On platforms where the OS only reports "something in this directory
  changed" without saying what (i.e. not Linux inotify), cookie detection is
  "currently an unresolved issue" if not every intervening event was
  individually observed.
- **macOS/FSEvents is called out as a documented weak point**: FSEvents
  provides no synchronous-flush guarantee. Watchman combines cookie files
  with `FSEventStreamFlushSync`, but under high load (e.g. a large `git
  checkout`) FSEvents-reported changes have been observed to arrive *after*
  the cookie notification and the flush call both returned. Apple's own
  suggested fix is to reimplement the watcher on Endpoint Security instead of
  FSEvents; Watchman falls back to a `settle_period`/`settle_timeout`
  quiescence heuristic instead.
- Motivation for cookies traces to production pain: with 16+ parallel
  Mercurial test-suite runs, watchman would fall behind and return stale
  results before cookies were added; cookies eliminated that failure mode.

## Concept pages informed

- [`concepts/watchman.md`](../concepts/watchman.md) (new)
- [`concepts/fsevents.md`](../concepts/fsevents.md) (added a Limitations note
  on FSEvents' lack of a synchronous-flush guarantee, sourced from a real
  consumer's production experience rather than the Apple guide alone)
