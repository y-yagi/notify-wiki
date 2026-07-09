---
title: FSEvents
tags: [macos, darwin, event-driven]
updated: 2026-07-09
sources: ["../sources/apple-fsevents-progguide.md", "../sources/watchman-cookies.md", "../sources/apple-fsevents-h.md", "../sources/apple-dev-fseventstreamcreateflagfileevents.md"]
---

# FSEvents

## Overview

FSEvents (the File System Events API) is Apple's mechanism for monitoring an
entire directory hierarchy for changes, introduced in OS X 10.5 and used as
the foundation of Time Machine. It solves a different problem than
[[kqueue]]'s `EVFILT_VNODE`: rather than fine-grained, single-file,
real-time notification, FSEvents is designed for **passively watching large
trees** with directory-level granularity and, distinctively, **persistence
across reboots** — an application can ask "what changed in this tree since
event ID N (possibly days ago, possibly before I was even running)?"

## API / semantics

- Architecture: kernel code emits raw events to user space through a special
  device (also used by Spotlight); a user-space daemon (`fseventsd`) filters
  the raw stream and dispatches notifications to registered clients; a
  persistent on-disk database (in `.fseventsd/` at each volume root) retains
  the event history across reboots.
- Stream lifecycle: `FSEventStreamCreate` (per-host stream, event IDs
  increasing relative to the whole host) or `FSEventStreamCreateRelativeToDevice`
  (per-disk stream, event IDs relative to just that volume) → schedule on a
  run loop with `FSEventStreamScheduleWithRunLoop` → `FSEventStreamStart` →
  events delivered via callback → `FSEventStreamStop` →
  `FSEventStreamInvalidate` (or `FSEventStreamUnscheduleFromRunLoop` for
  finer control) → `FSEventStreamRelease`.
- The callback receives three parallel arrays per batch: paths, event IDs,
  and flags. On each event, the application is expected to (re)scan the
  directory named in the path — FSEvents tells you *that* something changed
  in a directory, not *what*.
- **Latency** is application-configurable at stream-creation time: FSEvents
  coalesces repeated changes to the same directory within that window into
  one notification, but always delivers at least one notification after the
  last change.
- Special flags requiring extra handling on receipt:
  - `kFSEventStreamEventFlagMustScanSubDirs` — events in a directory and one
    of its subdirectories were coalesced; the app must recursively rescan
    (changes may not be in an immediate child).
  - `kFSEventStreamEventFlagKernelDropped` / `UserDropped` — a kernel↔daemon
    communication error occurred; the app must do a full rescan since it
    can't know what was missed (always paired with `MustScanSubDirs`, so
    checking that one flag suffices).
  - `kFSEventStreamEventFlagRootChanged` (only if the stream was created with
    `kFSEventStreamCreateFlagWatchRoot`) — the watched root itself was
    deleted/moved/renamed (or a parent was); event ID is 0; the app must
    rescan the whole directory and, if needed, recover its new path via
    `fcntl(fd, F_GETPATH, ...)`.
  - `kFSEventStreamEventFlagEventIdsWrapped` — the 64-bit event ID counter
    wrapped (practically never happens).
- **Persistence**: an application stores the last event ID it processed and
  can later request all events since that ID (via `sinceWhen` in
  `FSEventStreamCreate`/`CreateRelativeToDevice`), or since a timestamp via
  `FSEventsGetLastEventIdForDeviceBeforeTime`. This lets an app catch up on
  changes made entirely while it wasn't running — something kqueue cannot do.
  A common pattern: combine persistent events with a cached snapshot
  (filenames + mtimes) of the watched tree, built with `lstat` (not `stat`,
  so symlink changes inside the watched tree are captured on the link
  itself, not silently followed).
- Cleanup helpers: `FSEventStreamFlushSync`/`FlushAsync` (wait for/track
  pending events to drain), `FSEventsPurgeEventsForDeviceUpToEventId`
  (root-only; destroys history — should essentially never be called since an
  app can't assume it's the only consumer of the event database).
- Per-device stream caveats: paths are volume-relative, not system-root-relative;
  device IDs aren't stable across reboots, so an app must compare volume UUIDs
  (`FSEventsCopyUUIDForDevice`) to confirm it's looking at the same disk
  before trusting stored event IDs.

## Limitations & gotchas

- **Directory-level granularity only** — never tells you which file changed
  or how; you must rescan and diff against your own cached snapshot to find
  out. Not suitable for virus scanners or anything needing to intercept or
  preempt an access (Apple explicitly recommends a VFS-level kernel
  extension for that — FSEvents has no permission/interception model at all,
  unlike [[fanotify]]).
- **Directory-level granularity is only the default, not a hard limit** —
  the `kFSEventStreamCreateFlagFileEvents` flag (present since at least the
  MacOSX10.9 SDK header) switches a stream to per-file notifications:
  `kFSEventStreamEventFlagItemCreated`/`ItemRemoved`/`ItemRenamed`/
  `ItemModified`/etc., plus `ItemIsFile`/`ItemIsDir`/`ItemIsSymlink` to
  identify the kind of item that changed. It's opt-in and the header warns
  it "will generate significantly more events than without it." This doesn't
  change the underlying architecture (still daemon round-trip + persistent
  DB writes per stream) — Apple's older guidance to use [[kqueue]]'s
  `EVFILT_VNODE` for lightweight single-file watching is not contradicted by
  this flag, but whether Apple still makes that specific recommendation
  post-`FileEvents` isn't confirmed by any source in this wiki.
- Coalescing across a directory and its subdirectories can force a full
  recursive rescan (`MustScanSubDirs`) — the notification doesn't tell you
  how deep the actual change was.
- Event IDs are **not necessarily consecutive** even for a client watching
  everything from the root, because of the permission model below — a client
  can't infer "no events happened" from a gap in IDs. Only root-privileged
  applications are guaranteed to see every event.
- **Security/permission gating**: an application only receives events for
  paths it could already reach via normal filesystem permissions — prevents
  using FSEvents as a privacy side-channel to enumerate directory names in
  another user's inaccessible directories.
- Deleted files still generate events and leave their name lingering in the
  on-disk event history (readable in `.fseventsd/`, root-only, undocumented/
  unstable format) until explicitly purged — the only purge granularity is
  "everything before event ID N," not individual record removal.
- Volumes can opt out of logging entirely by placing an empty `no_log` file
  in `.fseventsd/` — useful for backup targets or otherwise sensitive volumes.
- There's an inherent scan-vs-notification race: because notification
  latency is nondeterministic, an app must start monitoring *before*
  building its initial snapshot, and treat any directory that changes during
  the initial scan as needing a full rescan rather than trusting a
  timestamp comparison.
- **`FSEventStreamFlushSync` is not actually a hard guarantee under load.**
  [[watchman]] (a real production consumer) reports that during a large
  `git checkout`, FSEvents-reported changes have arrived *after* both a
  cookie-file notification and a `FSEventStreamFlushSync` call had already
  returned — i.e. the API's own documented flush primitive can lie about
  having flushed everything. Apple's suggested replacement is to build a
  watcher on [[endpoint-security]] instead, whose AUTH-time events and
  quantified drop-detection are a direct answer to this gap (at the cost of
  needing a security entitlement + Full Disk Access); there is no documented
  fix within the FSEvents API itself.

## Platform notes

- Introduced in OS X v10.5.
- Underlying mechanism (kernel device → `fseventsd` daemon → persistent
  per-volume database) unchanged in spirit since introduction, per this 2012
  edition of the guide; the live current Apple reference page
  (`developer.apple.com/documentation/coreservices/file_system_events`) is a
  JS-rendered SPA and wasn't captured verbatim for this wiki.
- `kFSEventStreamCreateFlagFileEvents` (per-file granularity, opt-in) was
  added after the 2012 guide's coverage. Confirmed available since **macOS
  10.7** and, later, **Mac Catalyst 13.1+**, per Apple's live reference page.
  See [apple-fsevents-h](../sources/apple-fsevents-h.md) and
  [apple-dev-fseventstreamcreateflagfileevents](../sources/apple-dev-fseventstreamcreateflagfileevents.md).

## Related concepts

- `[[kqueue]]` — the other Darwin file-notification option, positioned by
  Apple as complementary rather than competing: `EVFILT_VNODE` for
  fine-grained single-file watching, FSEvents for passive whole-tree
  monitoring with cross-reboot persistence.
- `[[fanotify]]` / `[[inotify]]` — the Linux analogues. Closest to FSEvents
  conceptually is fanotify's mount/filesystem-wide marking (both let you
  watch "everything under here" without enumerating each file), but neither
  Linux API has FSEvents' cross-reboot persistent event database.
- `[[findfirstchangenotification]]` — Windows' nearest equivalent in spirit
  (directory-level, not per-file), though it's coarser still: FSEvents at
  least reports which directory changed, while this API only signals that
  a filtered change occurred somewhere in the watched tree.
- `[[readdirectorychangesw]]` — Windows' more detailed API; unlike FSEvents
  it has no cross-reboot persistence, but its `FILE_NOTIFY_INFORMATION`
  records give more per-change detail than FSEvents' directory-path-only
  callback.
- `[[watchman]]` — a real-world consumer whose query-synchronization design
  documents FSEvents' synchronous-flush limitation from production
  experience, not just the API reference.
- [[endpoint-security]] — Apple's suggested replacement for FSEvents in
  security-sensitive contexts, per Watchman's own notes; solves a different
  problem primarily (security monitoring/authorization) but covers file
  events too, without FSEvents' flush-guarantee gap.
- [[recursive-watching]] — cross-cutting comparison of tree-watching support across all mechanisms/libraries in this wiki.
- [[kqueue-vs-fsevents]] — systematic axis-by-axis comparison against kqueue.

## Sources

- [apple-fsevents-progguide](../sources/apple-fsevents-progguide.md)
- [watchman-cookies](../sources/watchman-cookies.md)
- [apple-fsevents-h](../sources/apple-fsevents-h.md)
- [apple-dev-fseventstreamcreateflagfileevents](../sources/apple-dev-fseventstreamcreateflagfileevents.md)
