---
title: kqueue vs FSEvents
tags: [macos, darwin, comparison]
updated: 2026-07-10
sources: ["../sources/openbsd-kqueue-2.md", "../sources/apple-fsevents-progguide.md", "../sources/watchman-cookies.md", "../sources/apple-fsevents-h.md", "../sources/apple-dev-fseventstreamcreateflagfileevents.md"]
---

# kqueue vs FSEvents

Darwin's two native file-change mechanisms. Apple positions them as
complementary, not competing: [[kqueue]]'s `EVFILT_VNODE` for fine-grained,
real-time, single-file watching; [[fsevents]] for passive, large-tree,
directory-level monitoring. This page compares them axis by axis; see each
concept page for full detail.

## Comparison table

| Axis | kqueue (`EVFILT_VNODE`) | FSEvents |
|---|---|---|
| Watch scope | Per-object: one open fd per watched file/directory | Whole subtree from a single root path |
| Granularity | Per-file, explicit event type (`NOTE_WRITE`/`NOTE_DELETE`/`NOTE_RENAME`/etc.) | Directory-level by default; per-file only with opt-in `kFSEventStreamCreateFlagFileEvents` (more events) |
| Delivery timing | Immediate, edge-triggered per event | Coalesced within an app-configured latency window |
| Event identity/detail | Struct fields per fd, no path resolution built in | Paths + monotonic event IDs + flags, delivered in batches |
| Recursion | None — caller opens and registers every file/dir itself | Native — whole tree from one stream |
| Cross-reboot persistence | None | Yes — replay from a stored event ID via `sinceWhen` |
| Coalescing behavior | Repeated triggers on the same ident collapse into one kevent (can't count occurrences) | Repeated changes in the same directory within the latency window collapse into one notification |
| Drop/overflow signaling | None documented — no drop-detection mechanism in this wiki's kqueue source | `kFSEventStreamEventFlagKernelDropped`/`UserDropped` (binary — must full-rescan, no count) |
| Flush guarantee | Not applicable (no batching to flush) | `FSEventStreamFlushSync` documented as synchronous, but Watchman found in production that events can still arrive *after* it returns during heavy load (e.g. `git checkout`) |
| Permission/interception model | None | None (can't gate or block an operation) — same gap as kqueue, contrast [[fanotify]] and [[endpoint-security]] |
| Resource cost | One fd per watched object — expensive to scale to large trees | One stream, one daemon round-trip, persistent per-volume DB writes |
| Availability | FreeBSD 4.1+, OpenBSD 2.9+, macOS (BSD layer) | macOS 10.5+ (10.7+ for `FileEvents` per-file mode) |

## Takeaways

- **FSEvents' most concrete deficiency relative to kqueue is guarantee
  strength, not feature coverage.** Its own documented synchronous-flush
  primitive (`FSEventStreamFlushSync`) has been observed in production
  (Watchman, under a large `git checkout`) to return before all events have
  actually arrived — a failure mode that doesn't exist for kqueue, which has
  no batching/flush step to lie about in the first place. Apple's own
  suggested fix is to move to [[endpoint-security]] instead, not to patch
  FSEvents itself.
- **FSEvents has no drop-count, only a binary drop flag** (`KernelDropped`/
  `UserDropped`), forcing a full subtree rescan with no way to know how much
  was missed. kqueue has no analogous drop-signaling at all in the source
  material here — occurrences coalesce silently, but there's no channel by
  which the kernel tells the caller "you missed N events."
- **FSEvents' directory-only granularity is a real gap, but a fixable one**:
  `kFSEventStreamCreateFlagFileEvents` (macOS 10.7+) upgrades a stream to
  per-file events comparable in kind (though still batched, not real-time)
  to what kqueue reports for a single fd — at the cost of significantly more
  event volume. This isn't a structural gap so much as an opt-in trade-off
  Apple exposes explicitly.
- **Neither mechanism can intercept or authorize an operation** — that
  capability exists only in [[endpoint-security]]'s `AUTH` event class (or,
  on Linux, [[fanotify]]'s permission events). This is not an FSEvents-
  specific shortfall; kqueue lacks it identically.
- **The gaps run in both directions.** kqueue has no tree-wide recursion
  primitive and no cross-reboot persistence — both of which are FSEvents'
  core reasons to exist. Apple's guidance (per both concept pages) is to
  pick the mechanism matching the access pattern rather than treat one as a
  strict upgrade of the other.
- One qualification carried over from `concepts/fsevents.md`: whether Apple
  still recommends kqueue for single-file watching now that
  `kFSEventStreamCreateFlagFileEvents` exists isn't confirmed by any source
  in this wiki — the complementary positioning described here predates that
  flag.

## Related concepts

- [[kqueue]]
- [[fsevents]]
- [[endpoint-security]] — the mechanism Apple points to for the guarantees
  neither kqueue nor FSEvents provide (interception, quantified drop
  detection)
- [[fanotify]] — Linux's closest analogue to the permission/interception gap
  both Darwin mechanisms share
- [[recursive-watching]] — the cross-platform version of the recursion axis
  in the table above
