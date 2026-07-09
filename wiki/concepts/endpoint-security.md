---
title: Endpoint Security
tags: [macos, darwin, kernel, event-driven, security]
updated: 2026-07-09
sources: ["../sources/apple-endpointsecurity-esclient-h.md", "../sources/apple-endpointsecurity-estypes-h.md", "../sources/apple-endpointsecurity-esmessage-h-excerpt.md"]
---

# Endpoint Security

## Overview

Endpoint Security (`EndpointSecurity.framework`) is Apple's kernel-backed
security-event framework, introduced in macOS 10.15 (Catalina) as the
replacement for third-party kernel extensions doing similar work. It's
positioned in this wiki because both [[fsevents]]'s and [[watchman]]'s own
documentation point to it as Apple's suggested fix for FSEvents' lack of a
hard synchronous-flush guarantee — but it solves a different core problem
than any other mechanism here: it's built for **security monitoring and
in-line authorization** (allow/deny a syscall before it completes), with
file-change notification as one of several event categories it happens to
cover, not its primary design goal.

## API / semantics

- Client lifecycle: `es_new_client(client, handler_block)` registers a
  handler invoked serially, in delivery order, for every subscribed event;
  `es_subscribe`/`es_unsubscribe` (additive) pick which `es_event_type_t`
  values to receive; `es_delete_client` tears down. Requires the
  `com.apple.developer.endpoint-security.client` entitlement (Apple-approved,
  application-only) **and** the user granting Full Disk Access via TCC — a
  much higher barrier to entry than any other mechanism in this wiki, all of
  which are usable by any unprivileged process watching its own accessible
  files.
- Every event type is either `ES_ACTION_TYPE_AUTH` or `ES_ACTION_TYPE_NOTIFY`
  (encoded in the enum name itself, e.g. `ES_EVENT_TYPE_AUTH_UNLINK` vs.
  `ES_EVENT_TYPE_NOTIFY_UNLINK`). AUTH events block the originating syscall
  until the client calls `es_respond_auth_result` (allow/deny) or
  `es_respond_flags_result`, and there's a hard per-message `deadline` — a
  client that doesn't respond in time is killed by the OS. NOTIFY events are
  fire-and-forget, after the syscall already completed.
- File-relevant NOTIFY events (available since 10.15): `NOTIFY_CREATE`,
  `NOTIFY_OPEN`, `NOTIFY_CLOSE`, `NOTIFY_LINK`, `NOTIFY_UNLINK`,
  `NOTIFY_RENAME`, `NOTIFY_WRITE`, `NOTIFY_EXCHANGEDATA`, plus
  extended-attribute and mode/flags/owner change events, and
  `NOTIFY_MOUNT`/`NOTIFY_UNMOUNT`. Unlike inotify's bitmask-per-watch or
  FSEvents' single coalesced "something changed here" callback, a client
  subscribes per-syscall — there's no single "any file event" catch-all.
- Every file event carries an `es_file_t`: path (capped ~16K, with an
  explicit `path_truncated` flag if longer) plus a full `struct stat` — the
  event payload includes file metadata directly, not just a name the client
  must `stat()` itself.
- `es_event_create_t` and `es_event_rename_t` report structured before/after
  information a passive watcher can't: create events can fire at AUTH time
  *before* the file exists (dir + filename + mode, not yet a real path), and
  rename delivers both the source and a tagged union of the destination
  (existing file overwritten, or new dir+filename) in one event — no
  separate FROM/TO events to correlate, unlike inotify's `IN_MOVED_FROM`/
  `IN_MOVED_TO` cookie-pairing.
- Built-in noise reduction: `es_mute_process` (by audit token) and
  `es_mute_path_prefix`/`es_mute_path_literal` suppress events at the
  kernel/daemon level, before delivery — a capability [[watchman]] has to
  approximate in userspace with its own cookie/settle-period heuristics on
  top of FSEvents.
- Drop detection is quantified, not binary: `es_message_t` carries
  `seq_num`/`global_seq_num`, and the gap between consecutive values tells a
  client exactly how many events were dropped — contrast with FSEvents'
  `kFSEventStreamEventFlagKernelDropped`/`UserDropped` or inotify's
  `IN_Q_OVERFLOW`, both of which only signal "some drop happened, rescan."

## Limitations & gotchas

- **Not a general-purpose file-watching API** — the entitlement + TCC
  approval requirement means it's realistically only available to
  Apple-approved security products (EDR agents, antivirus), not arbitrary
  applications or CLI tools the way inotify/FSEvents/kqueue are.
- Every file event struct's doc comments carry the same caveat: "This event
  can fire multiple times for a single syscall, for example when the
  syscall has to be retried due to racing VFS operations" — duplicate
  events are an explicit, documented part of the contract, not an
  occasional bug.
- AUTH events impose a real-time obligation (respond before `deadline` or be
  killed) that doesn't exist anywhere else in this wiki — a client
  subscribing to AUTH events takes on a liveness requirement that a passive
  NOTIFY-only or FSEvents/inotify-style watcher never has.
- No cross-reboot persistence or historical replay the way FSEvents has
  (`sinceWhen`/stored event IDs) — Endpoint Security only delivers events
  for the period a client is actively connected.

## Platform notes

- Introduced macOS 10.15 (Catalina); all APIs and event types captured here
  are marked `API_AVAILABLE(macos(10.15))` in the header, with later event
  types presumably gated behind higher version macros not captured in this
  wiki's excerpted portion of the header.
- `iOS`/`tvOS`/`watchOS` explicitly excluded (`API_UNAVAILABLE`) — macOS-only,
  same as [[fsevents]].

## Related concepts

- [[fsevents]] — the mechanism Apple's own guidance (and [[watchman]]'s
  production notes) suggest replacing with Endpoint Security specifically to
  fix FSEvents' lack of a hard flush guarantee; Endpoint Security's
  synchronous AUTH-time events and quantified drop-detection are the direct
  answer to that gap, at the cost of needing a security entitlement.
- [[watchman]] — cites Endpoint Security as Apple's suggested fix; no source
  in this wiki documents Watchman actually having adopted it.
- [[fanotify]] — the closest Linux analogue: also has a permission/AUTH-style
  event model (`FAN_OPEN_PERM` etc.) gating a syscall on a userspace
  decision, though fanotify doesn't require an Apple-style entitlement to
  use.
- [[kqueue]] — Apple's other, much lower-barrier-to-entry Darwin mechanism;
  contrast Endpoint Security's entitlement/TCC/deadline requirements against
  kqueue's plain `EVFILT_VNODE`, usable by any process on files it can open.

## Sources

- [apple-endpointsecurity-esclient-h](../sources/apple-endpointsecurity-esclient-h.md)
- [apple-endpointsecurity-estypes-h](../sources/apple-endpointsecurity-estypes-h.md)
- [apple-endpointsecurity-esmessage-h-excerpt](../sources/apple-endpointsecurity-esmessage-h-excerpt.md)
