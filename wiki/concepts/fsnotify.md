---
title: fsnotify
tags: [linux, kernel, event-driven, internals]
updated: 2026-07-08
sources: ["../sources/lwn-fsnotify-unified-backend.md", "../sources/lwn-fanotify-api.md"]
---

# fsnotify

## Overview

fsnotify is the in-kernel filesystem-event backend shared by [[dnotify]],
[[inotify]], and [[fanotify]] on Linux. It is **not** a userspace-facing API
itself — there's no `fsnotify(2)` syscall — but a common dispatch and
registration layer that those three APIs sit on top of. It was merged for
Linux 2.6.31, from a patch series posted by Eric Paris in February 2009.
Simplifying the dnotify/inotify implementations was a side effect; the actual
motivation was giving fanotify (then still called TALPA, aimed at malware
scanners and HSM software) a backend capable of watching an entire mount or
the whole system, not just individually-marked files.

## API / semantics

- **Groups** (`struct fsnotify_group`) are the registration unit. Each of
  dnotify, inotify, and fanotify obtains a group via
  `fsnotify_obtain_group(group_num, mask, ops)`, supplying an event mask and
  an `fsnotify_ops` table (`handle_event`, `free_group_priv`,
  `free_event_priv`) that lets each API plug in its own delivery mechanism
  (signals for dnotify, a readable fd's event queue for inotify/fanotify).
- **Central dispatch**: `fsnotify(inode, mask, data, data_is)`, called from
  VFS operations (create, mkdir, link, open, close, access, modify, xattr
  change, rename, delete, unmount). It's a fast no-op unless the global
  `fsnotify_mask` (the OR of every registered group's mask) matches, then
  walks the group list under SRCU building at most one shared event.
- **Event bits** (`FS_ACCESS`, `FS_MODIFY`, `FS_CREATE`, `FS_DELETE`, etc., in
  `include/linux/fsnotify_backend.h`) are defined to line up bit-for-bit with
  inotify's `IN_*` bits, so inotify needs no translation table. A parallel
  set of `*_CHILD` bits (same values shifted `<< 32`) lets a directory-level
  listener distinguish "the directory itself changed" from "a child of the
  directory changed" — the mechanism that lets dnotify (directory-only
  granularity) and inotify (per-child granularity) share one delivery path.
- fanotify's "global listener" mode (an fsnotify group interested in every
  file on the system, rather than requiring a mark per inode) required
  extending the group infrastructure beyond what dnotify/inotify needed —
  this, per the LWN sources, is the actual reason fsnotify became a common
  backend rather than each API keeping its own separate plumbing.

## Limitations & gotchas

- Easy to confuse with the unrelated Go library of the same name
  (`github.com/fsnotify/fsnotify`), which wraps inotify/kqueue/
  `ReadDirectoryChangesW` in userspace and has no connection to this kernel
  subsystem beyond the shared name.
- The originally posted fanotify userspace API (`PF_FANOTIFY` socket,
  `getsockopt(SOL_FANOTIFY, ...)` for reads) described in the July 2009 LWN
  article never shipped — the merged fanotify uses `fanotify_init(2)`/
  `fanotify_mark(2)` syscalls instead. The fsnotify backend itself was
  largely unaffected by that uAPI change; only the layer built on top of it
  differs from what's described in that source.

## Platform notes

- Merged for Linux 2.6.31 (patch series posted Feb 2009).
- dnotify and inotify were re-plumbed onto fsnotify as part of that merge;
  fanotify was designed against it from the start but merged later, in
  Linux 2.6.36/2.6.37, with a syscall-based uAPI rather than the
  socket-based one originally proposed.

## Related concepts

- [[dnotify]] — one of the three APIs sharing this backend; directory-only
  granularity, predates fsnotify but was re-implemented on top of it.
- [[inotify]] — ditto; per-file/per-directory granularity.
- [[fanotify]] — the API fsnotify's group model was actually built to
  support (mount/filesystem-wide watching, permission events).

## Sources

- [lwn-fsnotify-unified-backend](../sources/lwn-fsnotify-unified-backend.md)
- [lwn-fanotify-api](../sources/lwn-fanotify-api.md)
