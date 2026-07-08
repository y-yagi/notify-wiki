---
title: "Source: FSEvents.h (Apple CarbonCore header, via MacOSX-SDKs mirror)"
tags: [macos, darwin, source]
updated: 2026-07-09
---

# apple-fsevents-h

**Raw file**: [`raw/apple-fsevents-h.txt`](../../raw/apple-fsevents-h.txt)
**Origin**: `CarbonCore/FSEvents.h`, Apple Inc. (© 2006–2011), from the
MacOSX10.9 SDK, mirrored verbatim at
https://github.com/phracker/MacOSX-SDKs
(`MacOSX10.9.sdk/.../CarbonCore.framework/Versions/A/Headers/FSEvents.h`),
fetched 2026-07-09. Used in place of the live
`developer.apple.com/documentation/coreservices/file_system_events` page,
which is a JS-rendered SPA not capturable as static text (same issue noted
for [apple-fsevents-progguide](apple-fsevents-progguide.md)).

## What it is

The actual C header defining the `FSEventStream` API: all
`FSEventStreamCreateFlags` and `FSEventStreamEventFlags` enum values with
their original doc comments. Supersedes the 2012 Programming Guide on
anything added to the API after that guide was written.

## What it claims

- `kFSEventStreamCreateFlagFileEvents` (value `0x00000010`): "Request
  file-level notifications. Your stream will receive events about individual
  files in the hierarchy you're watching instead of only receiving directory
  level notifications. Use this flag with care as it will generate
  significantly more events than without it." — i.e. per-file granularity is
  opt-in, not the default, and explicitly traded off against event volume.
- A block of `FSEventStreamEventFlags` values is documented as "only set if
  you specified the FileEvents... flag when creating the stream":
  `kFSEventStreamEventFlagItemCreated`, `ItemRemoved`, `ItemInodeMetaMod`,
  `ItemRenamed`, `ItemModified`, `ItemFinderInfoMod`, `ItemChangeOwner`,
  `ItemXattrMod`, `ItemIsFile`, `ItemIsDir`, `ItemIsSymlink`. Together these
  give both a specific per-item change type and the item's kind (regular
  file / directory / symlink) — not available at all without the flag.
- Does not itself state a minimum OS version for `FileEvents` in this
  particular header copy (no `AVAILABLE_MAC_OS_X_VERSION_10_7` macro present
  in this SDK's header text). Confirmed separately as macOS 10.7+ (and Mac
  Catalyst 13.1+) by
  [apple-dev-fseventstreamcreateflagfileevents](apple-dev-fseventstreamcreateflagfileevents.md).
- Other flags in the same header, used for cross-reference elsewhere in this
  wiki: `kFSEventStreamCreateFlagNoDefer`, `kFSEventStreamCreateFlagWatchRoot`,
  `kFSEventStreamCreateFlagIgnoreSelf`, `kFSEventStreamCreateFlagMarkSelf`,
  and the general/root-change/mount/unmount event flags already covered by
  [apple-fsevents-progguide](apple-fsevents-progguide.md).

## Concept pages informed

- [`concepts/fsevents.md`](../concepts/fsevents.md) — corrected the blanket
  "Not for single-file monitoring" claim: file-level notification is
  possible via `kFSEventStreamCreateFlagFileEvents`, opt-in, at the cost of
  higher event volume. Updated Platform notes with the flag's actual
  documented behavior instead of leaving it as an unconfirmed gap.
