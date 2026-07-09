---
title: "Source: Apple EndpointSecurity ESMessage.h (excerpt)"
tags: [macos, darwin, kernel, source]
updated: 2026-07-09
---

# apple-endpointsecurity-esmessage-h-excerpt

**Raw file**: [`raw/apple-endpointsecurity-esmessage-h-excerpt.txt`](../../raw/apple-endpointsecurity-esmessage-h-excerpt.txt)
**Origin**: `usr/include/EndpointSecurity/ESMessage.h`, mirrored via
`phracker/MacOSX-SDKs` (`MacOSX11.3.sdk`), fetched 2026-07-09. **Not a
verbatim full-file capture** — the raw file is a curated excerpt (file-I/O
event structs + the top-level message envelope only); see the raw file's own
header comment for exactly what was kept/dropped and why. This departs from
this wiki's usual "genuine source files are captured 100% verbatim"
convention because the full header (1401 lines) is overwhelmingly about
non-file event types (process exec, memory mapping, IOKit, code signing,
etc.) out of scope for this wiki.

## What it is

The message/event payload structs Endpoint Security actually delivers to a
subscribed client — i.e. what you get back after `es_subscribe`-ing to the
event types cataloged in [apple-endpointsecurity-estypes-h](apple-endpointsecurity-estypes-h.md).

## What it claims

- `es_file_t` (the common per-file payload embedded in every file event):
  a path (`es_string_token_t`, explicitly capped around 16K and flagged
  `path_truncated` if longer — an actual documented limit, unlike this
  wiki's other mechanisms which mostly leave path-length limits unstated),
  plus a full `struct stat`. Every file event structs a pointer to one or
  more of these rather than a bare path string.
- `es_event_create_t`: reports either an already-created file
  (`ES_DESTINATION_TYPE_EXISTING_FILE`) or, for the AUTH-time variant, a
  not-yet-created one (`ES_DESTINATION_TYPE_NEW_PATH`, giving parent dir +
  filename + mode) — Endpoint Security can tell a client about a file
  creation *before* it happens, something none of this wiki's passive
  notification mechanisms (inotify, FSEvents, kqueue) can do.
- `es_event_rename_t`: source file plus a `destination_type`-tagged union of
  either the overwritten existing file or a new dir+filename pair — directly
  reports both halves of a rename in one event, unlike inotify (which
  requires stitching separate `IN_MOVED_FROM`/`IN_MOVED_TO` events by
  cookie) or FSEvents (directory-level only, no structured rename info at
  all).
- `es_event_unlink_t`/`es_event_write_t`/`es_event_close_t`/etc. are
  comparatively thin — mostly just a `target` (and `parent_dir` for unlink)
  `es_file_t` pointer. A recurring `@note` on multiple structs: "This event
  can fire multiple times for a single syscall, for example when the syscall
  has to be retried due to racing VFS operations" — an explicit duplicate-
  event caveat baked into the API contract itself, not left to the client to
  discover.
- `es_message_t` (the envelope every event arrives in): carries a
  `seq_num`/`global_seq_num` pair specifically for **detecting dropped
  events** — "the difference between the last seen sequence number ... and
  seq_num ... indicates the number of events that had to be dropped."
  Unlike FSEvents' `kFSEventStreamEventFlagKernelDropped`/`UserDropped`
  (binary "something was dropped, rescan") or inotify's `IN_Q_OVERFLOW`
  (same), Endpoint Security tells a client *how many* events it missed via
  the sequence-number gap.

## Concept pages informed

- New page: [`concepts/endpoint-security.md`](../concepts/endpoint-security.md).
