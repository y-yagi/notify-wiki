---
title: "Source: Apple EndpointSecurity ESTypes.h"
tags: [macos, darwin, kernel, source]
updated: 2026-07-09
---

# apple-endpointsecurity-estypes-h

**Raw file**: [`raw/apple-endpointsecurity-estypes-h.txt`](../../raw/apple-endpointsecurity-estypes-h.txt)
**Origin**: `usr/include/EndpointSecurity/ESTypes.h`, mirrored via
`phracker/MacOSX-SDKs` (`MacOSX11.3.sdk`), fetched verbatim 2026-07-09.

## What it is

The `es_event_type_t` enum and other core Endpoint Security types — the full
catalog of events a client can subscribe to, and the AUTH-vs-NOTIFY event
model. Used alongside [apple-endpointsecurity-esclient-h](apple-endpointsecurity-esclient-h.md)
and [apple-endpointsecurity-esmessage-h-excerpt](apple-endpointsecurity-esmessage-h-excerpt.md)
to assess Endpoint Security as a file-notification mechanism.

## What it claims

- Every event type is either `ES_ACTION_TYPE_AUTH` (must be answered before
  the kernel proceeds, subject to a deadline) or `ES_ACTION_TYPE_NOTIFY`
  (fire-and-forget, after the fact) — encoded directly in the enum member
  name (`ES_EVENT_TYPE_AUTH_*` vs `ES_EVENT_TYPE_NOTIFY_*`), not a separate
  flag.
- File-relevant `NOTIFY` events available since macOS 10.15: `NOTIFY_OPEN`,
  `NOTIFY_CLOSE`, `NOTIFY_CREATE`, `NOTIFY_LINK`, `NOTIFY_UNLINK`,
  `NOTIFY_RENAME` (present in the enum, per
  [apple-endpointsecurity-esmessage-h-excerpt](apple-endpointsecurity-esmessage-h-excerpt.md)
  for the corresponding structs), plus `NOTIFY_MOUNT`/`UNMOUNT`,
  `NOTIFY_EXCHANGEDATA`, extended-attribute and mode/flags/owner change
  events. There is no single "anything changed here" event the way
  inotify's per-watch mask or FSEvents' coalesced directory notification
  work — a client subscribes to the specific syscalls it cares about.
  Corresponding `AUTH_*` variants exist for a subset (e.g. `AUTH_OPEN`,
  `AUTH_RENAME`, `AUTH_UNLINK`) letting a client block/allow the operation
  before it happens — capability no other mechanism in this wiki has except
  fanotify's permission events.
- The enum is explicitly append-only and versioned: a comment on
  `es_event_type_t` (not fully captured in this excerpt's event list, but
  present in the full header) notes new event types are added over time and
  gated behind OS version macros — mirroring the `kFSEventStreamCreateFlagFileEvents`
  version-gating pattern already seen in [[fsevents]].

## Concept pages informed

- New page: [`concepts/endpoint-security.md`](../concepts/endpoint-security.md).
