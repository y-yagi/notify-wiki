---
title: "Source: watchman watchman/watcher/fsevents.cpp"
tags: [macos, darwin, userspace-library, source]
updated: 2026-07-09
---

# watchman-fsevents-cpp

**Raw file**: [`raw/watchman-fsevents-cpp.txt`](../../raw/watchman-fsevents-cpp.txt)
**Origin**: https://github.com/facebook/watchman/blob/main/watchman/watcher/fsevents.cpp —
`main` branch, fetched verbatim via `raw.githubusercontent.com` 2026-07-09.

## What it is

Watchman's macOS FSEvents backend implementation — checked to see whether
Watchman, like [[libuv]] and notify-rs, uses `kFSEventStreamCreateFlagFileEvents`
for per-file granularity, following the same pattern used for
[libuv-fsevents-c](libuv-fsevents-c.md).

## What it claims

- **Watchman does use `kFSEventStreamCreateFlagFileEvents`, but conditionally**
  (line 419-421):
  ```
  flags = kFSEventStreamCreateFlagNoDefer | kFSEventStreamCreateFlagWatchRoot;
  if (watcher->hasFileWatching_) {
    flags |= kFSEventStreamCreateFlagFileEvents;
  }
  ```
- `hasFileWatching_` is set from a config option, `fsevents_watch_files`, which
  **defaults to `true`** (line 565): `config.getBool("fsevents_watch_files", true)`.
  So out of the box, Watchman does request file-level events — this isn't an
  opt-in a user has to discover.
- The flag choice changes which watcher name gets registered/reported
  (lines 549-551): `hasFileWatching ? "fsevents" : "dirfsevents"`, with a
  capability bit `WATCHER_HAS_PER_FILE_NOTIFICATIONS` set only when enabled.
  Disabling `fsevents_watch_files` visibly downgrades Watchman to
  directory-level granularity (the pre-10.7 FSEvents behavior) rather than
  silently changing internal behavior.
- `hasFileWatching_` is also read at two other points (lines 763, 781, 793) to
  change how paths are resolved/filtered depending on whether individual file
  paths or only directory paths are being delivered by the stream.

## Concept pages informed

- [`concepts/watchman.md`](../concepts/watchman.md) — adds a new API/semantics
  point: Watchman requests per-file FSEvents granularity by default (unlike
  the wiki's prior assumption that it, like the base FSEvents architecture
  described in [apple-fsevents-progguide](apple-fsevents-progguide.md), only
  reported directory-level changes). Notably this is *configurable*, unlike
  libuv's unconditional use of the same flag.
