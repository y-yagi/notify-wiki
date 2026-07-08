---
title: "Source: fsnotify: unified filesystem notification backend (LKML/LWN)"
tags: [linux, kernel, source]
updated: 2026-07-08
---

# lwn-fsnotify-unified-backend

**Raw file**: [`raw/lwn-fsnotify-unified-backend.txt`](../../raw/lwn-fsnotify-unified-backend.txt)
**Origin**: https://lwn.net/Articles/318618/ — patch mail from Eric Paris
<eparis@redhat.com> to linux-kernel@vger.kernel.org, "[PATCH -v1 01/11]
fsnotify: unified filesystem notification backend", Mon 09 Feb 2009, archived
verbatim on LWN.net.

## What it is

The original patch introducing `fsnotify`, the in-kernel backend later merged
for Linux 2.6.31. Not a narrative article — it's the raw patch (commit
message + diff) that added `fs/notify/fsnotify.c`, `fs/notify/group.c`,
`fs/notify/notification.c`, and `include/linux/fsnotify_backend.h`.

## What it claims

- fsnotify "does not provide any userspace interface but does provide the
  basis needed for other notification schemes such as dnotify" and "can be
  extended to be the backend for inotify or the upcoming fanotify."
- Introduces a `struct fsnotify_group` abstraction: a registration unit with
  an event mask and an ops table (`handle_event`, `free_group_priv`,
  `free_event_priv`), refcounted and evicted via `fsnotify_obtain_group()` /
  `fsnotify_put_group()`.
- Central dispatch is `fsnotify(inode, mask, data, data_is)`, wired into VFS
  hook points (create, mkdir, link, open, close, access, modify, xattr
  change, rename, delete-self, unmount) via `include/linux/fsnotify.h`. It
  short-circuits immediately if no group's mask matches, via a global
  `fsnotify_mask` OR'd across all groups, and walks the group list under SRCU
  otherwise.
- Defines `FS_*` event bits in `fsnotify_backend.h`, explicitly documented as
  lining up bit-for-bit with inotify's existing `IN_*` bits "so we can easily
  convert between them," plus a parallel set of `*_CHILD` bits (same values
  shifted left 32 bits) so a directory-level listener can distinguish "this
  object changed" from "a child of this object changed" — the mechanism that
  lets dnotify (directory-granularity only) and inotify (per-child) share one
  delivery path.

## Concept pages informed

- [`concepts/fsnotify.md`](../concepts/fsnotify.md) (new)
- [`concepts/inotify.md`](../concepts/inotify.md) (added fsnotify as the shared backend)
- [`concepts/dnotify.md`](../concepts/dnotify.md) (added fsnotify as the shared backend)
- [`concepts/fanotify.md`](../concepts/fanotify.md) (added fsnotify as the shared backend)
