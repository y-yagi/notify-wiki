---
title: dnotify
tags: [linux, kernel, event-driven, deprecated]
updated: 2026-07-08
sources: ["../sources/kernel-dnotify.md"]
---

# dnotify

## Overview

dnotify ("directory notification") was Linux's original kernel mechanism for
notifying applications when a directory, or a file within it, changed.
Authored by Stephen Rothwell, it predates [[inotify]] and was superseded by
it in Linux 2.6.13. It solves the same basic problem as inotify — avoiding
polling for filesystem changes — but at coarser granularity and via a very
different delivery mechanism: signals instead of a readable file descriptor.

## API / semantics

- Registration is via `fcntl(2)` with `F_NOTIFY`, on an already-open file
  descriptor for the directory to watch (no separate instance/watch-descriptor
  model like inotify's).
- Delivery is via signal, not `read(2)`. By default this is `SIGIO`, which
  carries no information beyond "something happened." Using `F_SETSIG` with a
  real-time signal (`SIGRTMIN + <n>`, with `+1` recommended since `SIGRTMIN`
  itself is often blocked) gets a `siginfo` struct whose `si_fd` field
  identifies which watched directory fd the event occurred on, and allows the
  kernel to queue multiple pending notifications instead of coalescing them
  into signal delivery.
- Event mask bits, all directory-granularity (they report that *something* of
  that type happened in the directory, not which file): `DN_ACCESS`,
  `DN_MODIFY`, `DN_CREATE`, `DN_DELETE`, `DN_RENAME`, `DN_ATTRIB`.
- Registration is one-shot by default — the app must re-register with `fcntl(2)`
  after each notification. OR'ing `DN_MULTISHOT` into the mask makes the
  registration persistent until the app explicitly clears it (by registering
  for an empty event set).
- Gated at build time by `CONFIG_DNOTIFY`; if disabled, `fcntl(fd, F_NOTIFY, ...)`
  fails with `EINVAL`.

## Limitations & gotchas

- **No filename in the event** — only directory-level granularity, unlike
  inotify's per-child `name` field.
- **Hard links are explicitly unhandled**: watching directory `a` for changes
  to file `x` will not see changes made via a different hard-linked path
  `b/x`. Unlinked files still generate one final notification in the last
  directory they were linked in.
- **Signal-based delivery is lossy by default** — without `F_SETSIG` + a
  real-time signal, multiple pending events collapse into a single
  undifferentiated `SIGIO`.
- Requires holding an open fd per watched directory (no separate lightweight
  watch-descriptor abstraction).

## Platform notes

- Superseded by [[inotify]] as of Linux 2.6.13; the kernel doc for dnotify
  itself says to use inotify instead going forward.

## Related concepts

- [[inotify]] — direct successor; replaced dnotify in Linux 2.6.13 with a
  readable-fd model, per-file watches, and filename-carrying events.

## Sources

- [kernel-dnotify](../sources/kernel-dnotify.md)
