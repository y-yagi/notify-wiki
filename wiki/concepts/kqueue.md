---
title: kqueue
tags: [bsd, macos, kernel, event-driven]
updated: 2026-07-08
sources: ["../sources/openbsd-kqueue-2.md"]
---

# kqueue

## Overview

kqueue (with its companion call `kevent`) is a general-purpose kernel event
notification mechanism from the BSD family, first appearing in FreeBSD 4.1
and available in OpenBSD since 2.9; it's also the basis of file-change
watching on macOS's BSD layer. Unlike [[inotify]] and [[fanotify]], which are
Linux file-notification-specific APIs, kqueue is a *general* event queue —
file changes are just one of several event sources ("filters") it supports,
alongside socket readiness, process lifecycle events, signals, timers, and
user-defined events. For file-change monitoring specifically, the relevant
filter is `EVFILT_VNODE`.

## API / semantics

- `kqueue(void)` / `kqueue1(flags)` create a kernel event queue and return a
  file descriptor. `kqueue1` additionally accepts `O_CLOEXEC`. The queue is
  not inherited across `fork(2)` and cannot be passed over UNIX-domain sockets.
- `kevent(kq, changelist, nchanges, eventlist, nevents, timeout)` does two
  things in one call: applies all pending changes from `changelist` (register/
  modify/remove events), then — unless `nevents` is 0 — waits (per `timeout`)
  and returns fired events into `eventlist`. A `NULL` timeout blocks
  indefinitely; a zero-valued (non-NULL) `timespec` polls without blocking.
  The same array may be reused for both `changelist` and `eventlist`.
- A kevent is identified by the `(ident, filter)` pair, unique per kqueue;
  re-registering the same pair modifies the existing entry rather than adding
  a duplicate.
- `struct kevent { ident; filter; flags; fflags; data; udata; }` — `ident` is
  usually a file descriptor (interpretation depends on filter), `filter`
  selects the kernel filter, `flags` are queue-management actions, `fflags`/
  `data` are filter-specific in/out fields, `udata` is opaque and passed
  through unchanged. `EV_SET()` is a convenience macro for filling the struct.
- Key `flags`: `EV_ADD` (register; auto-enables unless `EV_DISABLE` is also
  set), `EV_ENABLE`/`EV_DISABLE` (toggle delivery without removing the
  filter), `EV_DELETE` (deregister — also automatic on last `close(2)` of a
  descriptor-based event), `EV_ONESHOT` (auto-delete after first delivery),
  `EV_DISPATCH` (auto-disable, not delete, after delivery), `EV_CLEAR` (reset
  edge-triggered state after each read — some filters set this internally),
  `EV_RECEIPT` (return `EV_ERROR`-tagged confirmation for changelist entries
  without waiting for/draining real events — useful for bulk registration),
  `EV_EOF`/`EV_ERROR` (filter-specific/error signaling on return).
- **Filter re-execution model**: the filter runs at registration (to catch a
  preexisting condition), on every new event, *and* again when the user tries
  to retrieve the kevent — if the condition no longer holds by then, the
  kevent is silently dropped rather than returned. Multiple triggers of the
  same filter **coalesce into one queued kevent** rather than queuing separately.

### EVFILT_VNODE (file-level events)

Takes a file descriptor as `ident` and a bitmask of events to watch in
`fflags`:

- `NOTE_DELETE` — `unlink(2)` was called on the watched file.
- `NOTE_WRITE` — a write occurred.
- `NOTE_EXTEND` — the file was extended.
- `NOTE_TRUNCATE` — the file was truncated.
- `NOTE_ATTRIB` — attributes changed.
- `NOTE_LINK` — link count changed.
- `NOTE_RENAME` — the file was renamed.
- `NOTE_REVOKE` — access revoked via `revoke(2)`, or the underlying filesystem
  was unmounted.

`fflags` on return holds which of these actually fired. Note this is
**fd-per-watched-file**, not path-based and not directory-recursive — you
open each file you want to watch and register its fd.

### Other filters (context, not file-notification-specific)

`EVFILT_READ`/`EVFILT_WRITE`/`EVFILT_EXCEPT` (I/O readiness on sockets, pipes,
FIFOs, vnodes, BPF devices), `EVFILT_PROC` (`NOTE_EXIT`/`NOTE_FORK`/`NOTE_EXEC`/
`NOTE_TRACK` on a PID), `EVFILT_SIGNAL` (signal delivery, coexists with
`sigaction(3)` at lower precedence, always sets `EV_CLEAR`), `EVFILT_TIMER`
(one-shot or repeating timers with second/ms/us/ns or absolute-time units),
`EVFILT_DEVICE` (`NOTE_CHANGE`, e.g. hotplug), `EVFILT_USER` (purely
userspace-triggered events via `NOTE_TRIGGER`, for cross-thread signaling
through the same queue).

## Limitations & gotchas

- `EVFILT_VNODE` requires an **open file descriptor per watched file** — no
  built-in directory-recursive or path-based watching; an application must
  open and register every file/directory it cares about itself (this is the
  same fd-per-object burden inotify avoids for large trees only via its own
  per-path watch-descriptor bookkeeping — kqueue pushes it further, to raw fds).
- Repeated triggers of the same filter/ident **coalesce into a single kevent**
  — like inotify's event coalescing, this means kqueue cannot be used to
  reliably *count* occurrences, only to detect "at least one happened."
- The filter re-checks its condition when the user retrieves the event; if
  the condition has since resolved, the kevent silently vanishes rather than
  being delivered — application logic needs to tolerate events that "don't
  happen" after all.
- kqueues are not inherited over `fork(2)` and cannot be passed via
  UNIX-domain socket fd-passing.

## Platform notes

- First appeared in FreeBSD 4.1; available in OpenBSD since 2.9.
- Also available on macOS/Darwin for `EVFILT_VNODE`-based file watching,
  where Apple positions it as the fine-grained, single-file complement to
  the whole-tree, persistent [[fsevents]] mechanism.

## Related concepts

- `[[inotify]]` — Linux's dedicated file-notification API; unlike kqueue it's
  path/watch-descriptor based (not raw-fd-per-file) and delivers events via a
  single readable fd rather than a general kernel event queue shared with
  sockets/timers/signals/etc.
- `[[fanotify]]` — Linux's other file-notification API; like kqueue's
  `EVFILT_VNODE` it can identify objects without necessarily holding a
  conventional open fd (fanotify via file handles, kqueue always via fd).
- `[[fsevents]]` — Darwin's whole-tree, persistent, directory-granularity
  sibling; Apple's own guidance is kqueue for single-file fine-grained
  watching and FSEvents for passive large-tree monitoring.

## Sources

- [openbsd-kqueue-2](../sources/openbsd-kqueue-2.md)
