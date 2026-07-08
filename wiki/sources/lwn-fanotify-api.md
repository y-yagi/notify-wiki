---
title: "Source: The fanotify API (LWN.net)"
tags: [linux, kernel, source]
updated: 2026-07-08
---

# lwn-fanotify-api

**Raw file**: [`raw/lwn-fanotify-api.txt`](../../raw/lwn-fanotify-api.txt)
**Origin**: https://lwn.net/Articles/339399/ — "The fanotify API" by Jonathan
Corbet, LWN.net, July 1, 2009 (plus reader comment thread, archived).

## What it is

An LWN staff article covering the *first* posted fanotify API, written right
after fsnotify (see [`lwn-fsnotify-unified-backend`](lwn-fsnotify-unified-backend.md))
was merged for 2.6.31. Explains why fsnotify was built — not to simplify
dnotify/inotify, but as the support structure fanotify needed.

## What it claims

- fanotify's real motivation was Linux malware scanners (fanotify was
  previously named "TALPA"); the article frames dnotify/inotify
  simplification as a side effect of fsnotify, not its purpose.
- Describes the **originally proposed** userspace API, which never shipped as
  such: an application opens a `PF_FANOTIFY` socket, binds it to a
  `struct fanotify_addr` (family/priority/group_num/mask/timeout), then reads
  events via `getsockopt(SOL_FANOTIFY, FANOTIFY_GET_EVENT)` and responds to
  permission events via `setsockopt(FANOTIFY_ACCESS_RESPONSE)`. The merged
  fanotify (documented in `concepts/fanotify.md`) instead uses dedicated
  syscalls (`fanotify_init(2)`/`fanotify_mark(2)`) — a materially different
  uAPI from this proposal.
- Event mask in the original proposal: `FAN_ACCESS`, `FAN_MODIFY`,
  `FAN_CLOSE`, `FAN_OPEN`, `FAN_ACCESS_PERM`, `FAN_OPEN_PERM`,
  `FAN_EVENT_ON_CHILD` (immediate children only — same non-recursive
  limitation the final API has), `FAN_GLOBAL_LISTENER` (system-wide
  notification, later becoming fanotify's mount/filesystem-wide marking).
- Comment thread (Eric Paris himself replying) corrects some article details
  and records early design debate: objections to `getsockopt()` as an event
  interface, a proposal to instead extend inotify with an `IN_RECURSIVE`
  flag rather than add a whole new API, and confirmation that both the
  "directed" (per-inode marks) and "global listener" modes shared the same
  underlying fsnotify group infrastructure.

## Concept pages informed

- [`concepts/fsnotify.md`](../concepts/fsnotify.md) (new — early API design history)
- [`concepts/fanotify.md`](../concepts/fanotify.md) (noted the pre-merge socket-based
  API proposal differs from the shipped syscall-based one)
