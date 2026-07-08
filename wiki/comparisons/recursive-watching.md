---
title: "Comparison: recursive watching support"
tags: [comparison, recursive-watching]
updated: 2026-07-08
---

# Recursive watching support

"Can I watch a directory tree without registering every subdirectory myself?"
splits the mechanisms and libraries in this wiki into three different
answers, not just yes/no:

1. **Native recursive watch** — one call at setup time covers the whole
   subtree; the kernel/OS does the walking.
2. **No recursion primitive** — the caller must discover and register every
   subdirectory itself, usually racing against concurrent subdirectory
   creation.
3. **Not path-scoped at all** — there is no "watch this subtree" concept;
   the source is a global/volume-wide/mount-wide stream that the caller
   filters by path afterward.

## OS-level mechanisms

| Mechanism | Support | Model |
|---|---|---|
| [[dnotify]] | No | Per-directory fd, directory-granularity events only — no subtree concept even in principle. |
| [[inotify]] | No | Per-path watches; a tree requires walking it and adding one watch per subdirectory, racing the add-watch-vs-mkdir window. |
| [[fanotify]] | No *(path marks)* / Yes *(mount or filesystem marks)* | Per-directory marks are explicitly non-recursive — same race as inotify. But marking the *mount* or *filesystem* instead covers the whole tree atomically and race-free; no other Linux mechanism here offers that. |
| [[kqueue]] | No | `EVFILT_VNODE` is fd-per-watched-file; the caller opens and registers every file/directory itself, with no tree-level primitive at all. |
| [[fsevents]] | Yes | Watching a path natively covers its whole subtree; trade-off is coalescing (`kFSEventStreamEventFlagMustScanSubDirs`) can force the app into a full recursive rescan because the notification doesn't say how deep the actual change was. |
| [[readdirectorychangesw]] | Yes | A single `bWatchSubtree` boolean at watch-setup time. |
| [[usn-journal]] | N/A | Not path-scoped — one append-only log per *volume*; there's no "watch this subtree" call, the caller filters the shared journal by path after the fact. |

## Userspace libraries

| Library | Support | Model |
|---|---|---|
| [[libuv]] | Partial | `UV_FS_EVENT_RECURSIVE` flag exists in the API but is only implemented on macOS and Windows — riding on [[fsevents]]'s and [[readdirectorychangesw]]'s native support respectively. Unavailable on Linux and BSD backends. |
| [[watchman]] | Not documented in our source | Watches whole "roots," but the ingested `cookies.md` source doesn't describe the subtree-discovery mechanism itself — not confirmed here. |
| [[chokidar]] | Yes | README states recursive watching is always supported (with a `depth` option to limit it), regardless of backend. |
| [[notify-rs]] | Not documented in our source | The notify-rs README doesn't state a recursive-watch default either way — worth checking `docs.rs` directly before relying on this. |
| [[fsnotify-go]] | No | README states recursive watching is "on the roadmap," not shipped; each subdirectory needs its own explicit `Add()` call. |

## Takeaways

- **fanotify's mount/filesystem marks are the only race-free tree-wide
  primitive among the OS mechanisms** — every path-based watch API here
  (dnotify, inotify, kqueue) requires the caller to walk the tree and accept
  a window where a subdirectory can be created and populated before its
  watch is registered.
- Libraries that claim recursive support either (a) inherit it for free from
  a native OS primitive ([[libuv]] on macOS/Windows), or (b) must be doing
  the walk-and-register work themselves in userspace when the backend has no
  native primitive (implied for [[chokidar]] on Linux, where the underlying
  `fs.watch`/inotify has none — not explicitly documented in its README, but
  the only way the claim is consistent with [[inotify]]'s own limitations).
- Two entries above are marked "not documented in our source" rather than a
  guessed yes/no — resolving them would mean ingesting docs.rs's notify-rs
  API reference and Watchman's watch-establishment code/docs, not just the
  README and cookies.md we have today.
