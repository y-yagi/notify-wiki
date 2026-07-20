---
title: Listen
tags: [userspace-library, ruby, cross-platform, event-driven]
updated: 2026-07-12
sources: ["../sources/listen-readme.md"]
---

# Listen

## Overview

`listen` is a Ruby gem — best known as the file-watching engine inside
`guard` — that watches one or more directories and reports modifications,
additions, and removals to a callback. Like [[libuv]] and [[chokidar]], it
exists to hide which native OS mechanism is doing the work; unlike those two
it does so by delegating to a set of separate per-platform Ruby gems rather
than implementing the native calls itself.

## API / semantics

- `Listen.to(*dirs, **options) { |modified, added, removed| ... }` returns a
  listener; `#start`/`#stop`/`#pause` control it. The callback always
  receives three arrays of absolute paths, in the same `modified, added,
  removed` order, regardless of which adapter produced the underlying event —
  the same "collapse everything to one small vocabulary" move [[libuv]] makes
  with `UV_RENAME`/`UV_CHANGE`, but three buckets instead of two.
- Adapter selection is automatic per platform and delegated to separate
  wrapper gems, not implemented in `listen` itself: `rb-inotify` on Linux,
  `rb-fsevent` on Darwin (both ship as direct dependencies), `rb-kqueue` on
  \*BSD and `wdm` on Windows (both require adding the gem manually). A polling
  adapter is the universal fallback, used automatically when no native
  adapter applies (e.g. network filesystems) or forced via `force_polling`.
- `:latency` and `:wait_for_delay` are tunable independently of the active
  adapter — `:latency` gates how often changes are checked for, `:wait_for_delay`
  gates how long to wait before invoking the callback once changes exist.
- `:ignore`/`:ignore!`/`:only` filter by regex against relative paths before
  the callback fires.

## Limitations & gotchas

- Symlinked directories pointing *within* an already-watched directory are
  not supported (symlinks themselves are always followed, but that specific
  nesting case isn't) — open issue as of this README, not a version-specific
  limitation.
- No cross-process notification: each forked process must call `Listen.to`
  itself, since listeners don't propagate across `fork`.
- Linux inotify's per-user watch limit (`fs.inotify.max_user_watches`) gets
  disproportionate troubleshooting attention in the README relative to other
  adapters' limits — suggesting it's the failure mode users hit most often in
  practice, not that the other adapters (kqueue, FSEvents, ReadDirectoryChangesW)
  are limit-free.
- TCP-based cross-process notification was removed in v3.0.0; only available
  by pinning `listen ~> 2.10`.
- Some filesystems (VM/Vagrant shared folders, NFS, Samba, sshfs) don't work
  with any native adapter and require polling — a real-world instance of the
  same "not every mount actually supports the native mechanism" gap seen with
  [[inotify]] and network filesystems generally.
- Not confirmed in this source whether recursive subtree watching is native
  per-adapter or built by `listen` walking the tree itself in userspace — the
  README doesn't state a recursive-watch model either way (see
  [[recursive-watching]]).

## Related concepts

- [[inotify]] — Linux backend (via the separate `rb-inotify` gem); the
  specific adapter the README's troubleshooting section is written around.
- [[fsevents]] — Darwin backend (via `rb-fsevent`).
- [[kqueue]] — \*BSD backend (via `rb-kqueue`, an optional dependency).
- [[readdirectorychangesw]] — implied Windows backend, wrapped by `wdm`
  (README doesn't confirm which native Win32 API `wdm` itself calls).
- [[libuv]] / [[chokidar]] / [[watchdog]] — comparable "one callback
  vocabulary over several native backends" libraries; `listen` is the same
  pattern applied to Ruby, built on separate per-platform gems rather than
  one shared C library (watchdog) or shared C library (libuv/Chokidar).
- [[recursive-watching]] — cross-cutting comparison of tree-watching support;
  `listen` is listed there as not documented in this source.

## Sources

- [listen-readme](../sources/listen-readme.md)
