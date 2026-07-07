---
title: "Source: kqueue(2) man page (man.openbsd.org)"
tags: [bsd, kernel, source]
updated: 2026-07-08
---

# openbsd-kqueue-2

**Raw file**: [`raw/openbsd-kqueue.2.txt`](../../raw/openbsd-kqueue.2.txt)
**Origin**: https://man.openbsd.org/kqueue.2 — OpenBSD manual page for `kqueue`, `kqueue1`, `kevent`, `EV_SET`. Authored by Jonathan Lemon (originally for FreeBSD).

## What it is

The canonical OpenBSD manual page for the kqueue/kevent kernel event
notification mechanism — a general event-queue facility, not a
file-change-specific API. File-change monitoring is one of several event
sources (`EVFILT_VNODE`) it exposes, alongside sockets, processes, signals,
timers, and user-triggered events.

## What it claims

- Core API: `kqueue(2)`/`kqueue1(2)` create a kernel event queue (fd);
  `kevent(2)` both registers/modifies events (via a `changelist`) and retrieves
  fired events (into an `eventlist`) in one call; `EV_SET()` macro builds a
  `struct kevent`.
- A kevent is uniquely identified by the `(ident, filter)` pair per kqueue;
  re-adding one modifies it in place rather than duplicating it.
- `struct kevent` fields: `ident`, `filter`, `flags` (`EV_ADD`, `EV_ENABLE`,
  `EV_DISABLE`, `EV_DISPATCH`, `EV_DELETE`, `EV_RECEIPT`, `EV_ONESHOT`,
  `EV_CLEAR`, `EV_EOF`, `EV_ERROR`), `fflags`/`data` (filter-specific),
  `udata` (opaque, passed through unchanged).
- Filters aggregate repeated triggers into one queued kevent rather than
  queuing duplicates (contrast with per-event queuing).
- kqueues are not inherited across `fork(2)` and cannot be passed over
  UNIX-domain sockets.
- Documents all predefined filters, most relevantly for this wiki
  `EVFILT_VNODE` (file-level events: `NOTE_DELETE`, `NOTE_WRITE`,
  `NOTE_EXTEND`, `NOTE_TRUNCATE`, `NOTE_ATTRIB`, `NOTE_LINK`, `NOTE_RENAME`,
  `NOTE_REVOKE`) — plus `EVFILT_READ`/`WRITE`/`EXCEPT`, `EVFILT_PROC`,
  `EVFILT_SIGNAL`, `EVFILT_TIMER`, `EVFILT_DEVICE`, `EVFILT_USER`.
- History: `kqueue()`/`kevent()` first appeared in FreeBSD 4.1, available in
  OpenBSD since 2.9.

## Concept pages informed

- [`concepts/kqueue.md`](../concepts/kqueue.md) (new)
- [`concepts/inotify.md`](../concepts/inotify.md) (added kqueue as a related concept, replacing the earlier placeholder)
