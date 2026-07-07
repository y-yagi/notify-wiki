---
title: "Source: File System Events Programming Guide (Apple, archived)"
tags: [macos, darwin, source]
updated: 2026-07-08
---

# apple-fsevents-progguide

**Raw file**: [`raw/apple-fsevents-progguide.txt`](../../raw/apple-fsevents-progguide.txt)
**Origin**: https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/FSEvents_ProgGuide/ — Apple's archived "File System Events Programming Guide" (5 chapters: Introduction, Technology Overview, Using the File System Events API, File System Event Security, Kernel Queues: An Alternative to File System Events). Copyright 2012 Apple Inc., last updated 2012-12-13.

Note: the live `developer.apple.com/documentation/coreservices/file_system_events`
reference page is a JavaScript-rendered SPA that couldn't be fetched as static
text; this archived Programming Guide was used instead as it's static HTML
and substantially more detailed (full conceptual guide, not just an API index).

## What it is

Apple's official conceptual guide to FSEvents, the macOS/Darwin API for
directory-hierarchy-level file change notification, introduced in OS X 10.5.
Written for application developers (backup software, sync tools, project
consistency checking), not kernel engineers.

## What it claims

- Architecture: kernel code emits raw events to user space via a special
  device (shared with Spotlight), a user-space daemon (`fseventsd`) filters
  and dispatches them, and a persistent on-disk database
  (`.fseventsd/` at each volume root) retains history across reboots.
- **Directory-level granularity only** — an event says "something in this
  directory changed," never which file or what changed. Multiple file
  changes collapse into one per-directory notification; multiple
  notifications on one directory within a short window are further coalesced,
  with an application-configurable minimum latency between deliveries.
- Explicitly **not** intended for fine-grained/immediate notification (virus
  scanners, access interception) — recommends a VFS-level kernel extension
  for that — and not for watching a single file, where kqueue is recommended
  instead. FSEvents is positioned for passively monitoring large trees
  (its own justification: it underlies Apple's Time Machine backup).
  This 1:1 matches the trade-off kqueue.2 documents from the other direction.
- API: `FSEventStreamCreate`/`FSEventStreamCreateRelativeToDevice` (per-host
  vs. per-disk event-ID streams) → `FSEventStreamScheduleWithRunLoop` →
  `FSEventStreamStart` → callback delivery → `FSEventStreamStop` →
  `FSEventStreamInvalidate`/`Unschedule` → `FSEventStreamRelease`.
  Callback receives parallel arrays of paths, event IDs, and flags.
  `kFSEventStreamEventFlagMustScanSubDirs`, `KernelDropped`/`UserDropped`,
  `RootChanged`, and `EventIdsWrapped` flags all require different recovery
  actions (full/recursive rescan, in most cases).
- Persistence: events are retained across reboots and queryable from a past
  event ID or timestamp — lets an app catch up on changes made while it
  wasn't running, which kqueue cannot do.
- Security: events require the requesting process to have filesystem
  permission to reach the changed directory; logging can be disabled per-volume
  via a `.fseventsd/no_log` sentinel file; deleted-file directory names
  persist in the event log until purged.
- Explicitly frames kqueue/`EVFILT_VNODE` as the correct alternative for
  single-file, fine-grained monitoring, and gives a full working example.

## Concept pages informed

- [`concepts/fsevents.md`](../concepts/fsevents.md) (new)
- [`concepts/kqueue.md`](../concepts/kqueue.md) (added FSEvents as a related concept, replacing the earlier placeholder; corroborates existing `EVFILT_VNODE`/`NOTE_*` documentation)
