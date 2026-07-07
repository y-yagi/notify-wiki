---
title: "Source: Documentation/filesystems/dnotify.txt (kernel.org)"
tags: [linux, kernel, source]
updated: 2026-07-08
---

# kernel-dnotify

**Raw file**: [`raw/kernel-dnotify.txt`](../../raw/kernel-dnotify.txt)
**Origin**: https://www.kernel.org/doc/Documentation/filesystems/dnotify.txt — author Stephen Rothwell.

## What it is

The short (~70 line) kernel documentation file for dnotify, Linux's original
directory-change-notification mechanism, predating inotify.

## What it claims

- Registration is via `fcntl(2)` (`F_NOTIFY`) on an open directory fd;
  notification delivery is via signals (default `SIGIO`), not a readable fd.
- Event set: `DN_ACCESS`, `DN_MODIFY`, `DN_CREATE`, `DN_DELETE`, `DN_RENAME`,
  `DN_ATTRIB` — all directory-granularity (an event on a file just says "something
  changed in this directory," no filename).
- Registration is one-shot by default; `DN_MULTISHOT` OR'd into the mask makes
  it persistent until explicitly cleared.
- With `F_SETSIG` and a real-time signal, the kernel queues notifications and
  passes a `siginfo` struct whose `si_fd` names the directory fd — otherwise
  you only learn "some event happened," not which fd/directory.
- Explicitly ignores hard links: a watch on directory `a` for file `x` (also
  linked as `b/x`) is not notified of changes made via the `b/x` path.
  Unlinked files still notify the last directory they were linked in.
- Intended to also cover local-server-mediated remote access (local NFS server,
  user-mode servers), not just direct local syscalls.
- Gated by `CONFIG_DNOTIFY`; disabled builds return `-EINVAL` from `F_NOTIFY`.
- States plainly: **superseded by inotify since Linux 2.6.13.**

## Concept pages informed

- [`concepts/dnotify.md`](../concepts/dnotify.md) (new)
- [`concepts/inotify.md`](../concepts/inotify.md) (added dnotify as a related/predecessor concept)
