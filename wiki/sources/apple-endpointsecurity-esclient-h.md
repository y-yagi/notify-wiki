---
title: "Source: Apple EndpointSecurity ESClient.h"
tags: [macos, darwin, kernel, source]
updated: 2026-07-09
---

# apple-endpointsecurity-esclient-h

**Raw file**: [`raw/apple-endpointsecurity-esclient-h.txt`](../../raw/apple-endpointsecurity-esclient-h.txt)
**Origin**: `usr/include/EndpointSecurity/ESClient.h`, mirrored via
`phracker/MacOSX-SDKs` (`MacOSX11.3.sdk`), fetched verbatim 2026-07-09.
Apple's live docs page (`developer.apple.com/documentation/endpointsecurity`)
is a JS-rendered SPA, same limitation as the FSEvents docs pages earlier in
this wiki — the header was used instead, per the established pattern (see
[apple-fsevents-h](apple-fsevents-h.md)).

## What it is

The client lifecycle API for Apple's EndpointSecurity framework (macOS
10.15+) — the mechanism Apple recommends as a replacement for FSEvents in
security-sensitive contexts (see [[watchman]]'s and [[fsevents]]'s own notes
citing this). Used to check what Endpoint Security actually is and whether
it's a realistic alternative for general file-change notification, not just
security tooling.

## What it claims

- Lifecycle: `es_new_client(client, handler_block)` connects and registers a
  handler block invoked serially, in delivery order, for every subscribed
  message; `es_delete_client(client)` tears it down. Requires the
  `com.apple.developer.endpoint-security.client` entitlement **and** end-user
  approval via Full Disk Access (TCC) — neither of which any other mechanism
  in this wiki requires just to watch files.
- Subscription: `es_subscribe(client, events, count)` / `es_unsubscribe(...)`
  / `es_unsubscribe_all(client)` — additive; new subscriptions don't clear
  old ones.
- Two response paths for AUTH-type events (which this wiki's file-watching
  scope mostly doesn't use — see [apple-endpointsecurity-estypes-h](apple-endpointsecurity-estypes-h.md)
  for the AUTH/NOTIFY split): `es_respond_auth_result` (allow/deny) and
  `es_respond_flags_result` (masked flags, `ES_EVENT_TYPE_AUTH_OPEN` only).
  A client that fails to respond to an AUTH event before its `deadline` is
  killed by the OS — a real-time obligation FSEvents/kqueue/inotify don't
  impose.
- Muting: `es_mute_process`/`es_unmute_process` (by audit token) and
  `es_mute_path_prefix`/`es_mute_path_literal`/`es_unmute_all_paths` (by
  path) — built-in noise reduction not available as a kernel-level primitive
  in any of this wiki's other mechanisms (Watchman has to build its own
  cookie/settle-period heuristics on top of FSEvents instead, see
  [[watchman]]).

## Concept pages informed

- New page: [`concepts/endpoint-security.md`](../concepts/endpoint-security.md).
