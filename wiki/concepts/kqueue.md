---
title: kqueue
tags: [bsd, macos, kernel, event-driven]
updated: 2026-07-12
sources: ["../sources/openbsd-kqueue-2.md", "../sources/lemon-kqueue-freenix2001.md"]
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

Kqueue's own design paper (Jonathan Lemon, FreeBSD Project, FREENIX Track of
the 2001 USENIX Annual Technical Conference) frames it as a direct answer to
`poll()`/`select()` not scaling: both are stateless between calls, so the
kernel re-copies and re-scans the entire descriptor list every time even
though typically only a few hundred of several thousand descriptors are
actually active. kqueue instead keeps that interest-set state in the kernel
across calls, so an already-registered, inactive descriptor costs the caller
close to nothing to re-check.

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

### Design goals (from the original design paper)

Lemon's paper states the goals kqueue was designed against, each a direct
reaction to a specific `poll()`/`select()` shortcoming:

- **Scalable** to thousands of descriptors — the motivating problem is that
  poll/select are stateless between calls, forcing an O(N) re-copy and
  re-scan of the entire descriptor list every call even though usually only
  a few hundred are active.
- **Flexible**, covering event sources poll/select can't (file
  creation/deletion, signals, AIO completion) via the same interface, and
  extensible to future sources without changing the API — the origin of
  kqueue's filter model.
- **Simple** enough that existing poll/select code converts with minimal
  changes.
- **Reliable**: no silent failure, no fixed-size lists that can overflow,
  and memory allocated at syscall time rather than when activity occurs (to
  avoid losing events under memory pressure) — a direct contrast with POSIX
  signal queues, which silently bound queue length and can drop events.
- **Correct**: closing a descriptor must not leave a stale event behind, and
  registering interest in a condition that's already true (e.g. data
  already pending on a socket) must still generate an event — no race
  between "start watching" and "the thing already happened."
- **Level-triggered**, not edge-triggered, matching poll/select semantics
  (an event is reported as long as its condition holds, not just once at
  first detection) — and events are **coalesced** (many packets arriving
  become one queued event, not one per packet), since an unbounded number
  of discrete occurrences can't be held in finite memory without a
  drop-or-coalesce decision.
- **Library-safe**: usable by third-party code sharing a process with the
  main program, unlike signals (one handler per process) or a single
  poll/select event loop — kqueue moves interest-set state into the kernel
  specifically so a library can maintain its own private kqueue without
  conflicting with the application's.

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

`NOTE_TRUNCATE` and `NOTE_REVOKE` aren't in the original 2001 design paper's
`EVFILT_VNODE` action set (only `NOTE_DELETE`/`WRITE`/`EXTEND`/`ATTRIB`/
`LINK`/`RENAME` there) — filter-set growth over time, not a contradiction
between the two sources.

### Other filters (context, not file-notification-specific)

`EVFILT_READ`/`EVFILT_WRITE`/`EVFILT_EXCEPT` (I/O readiness on sockets, pipes,
FIFOs, vnodes, BPF devices), `EVFILT_PROC` (`NOTE_EXIT`/`NOTE_FORK`/`NOTE_EXEC`/
`NOTE_TRACK` on a PID), `EVFILT_SIGNAL` (signal delivery, coexists with
`sigaction(3)` at lower precedence, always sets `EV_CLEAR`), `EVFILT_TIMER`
(one-shot or repeating timers with second/ms/us/ns or absolute-time units),
`EVFILT_DEVICE` (`NOTE_CHANGE`, e.g. hotplug), `EVFILT_USER` (purely
userspace-triggered events via `NOTE_TRIGGER`, for cross-thread signaling
through the same queue).

None of `EVFILT_TIMER`/`EVFILT_DEVICE`/`EVFILT_USER` appear in the original
2001 design paper — its filter set was just `READ`/`WRITE`/`AIO`/`VNODE`/
`PROC`/`SIGNAL`. The paper's own conclusion mentions a timer filter as then
"in the process of being added," consistent with these being later additions
rather than a discrepancy between sources.

## Limitations & gotchas

- `EVFILT_VNODE` requires an **open file descriptor per watched file** — no
  built-in directory-recursive or path-based watching; an application must
  open and register every file/directory it cares about itself (this is the
  same fd-per-object burden inotify avoids for large trees only via its own
  per-path watch-descriptor bookkeeping — kqueue pushes it further, to raw fds).
  **No source in this wiki explains why** — even kqueue's own design paper
  (Lemon, 2001) doesn't discuss directory recursion or tree-wide watching at
  all, unlike inotify, where co-designer Robert Love has given an explicit
  rationale for the same omission (see [[inotify]] and
  [quora-love-inotify-recursive](../sources/quora-love-inotify-recursive.md)).
  This looks like a genuine documentation gap rather than an intentional
  design silence — resolving it would need something like a kqueue mailing
  list thread or a BSD kernel developer's own account, not just the paper
  and man page ingested so far.
- Repeated triggers of the same filter/ident **coalesce into a single kevent**
  — like inotify's event coalescing, this means kqueue cannot be used to
  reliably *count* occurrences, only to detect "at least one happened."
- The filter re-checks its condition when the user retrieves the event; if
  the condition has since resolved, the kevent silently vanishes rather than
  being delivered — application logic needs to tolerate events that "don't
  happen" after all.
- kqueues are not inherited over `fork(2)` and cannot be passed via
  UNIX-domain socket fd-passing. More precisely (per the original design
  paper): a kqueue descriptor *is* passed to a child that shares its parent's
  file table via `vfork(RFFDG)` — the general "not inherited" statement holds
  for ordinary `fork(2)`, not every child-creation path.

## Platform notes

- First appeared in FreeBSD 4.1 (committed to the CVS tree in April 2000);
  available in OpenBSD since 2.9; at the time of the original design paper
  (2001) it was "in the process of being brought into NetBSD" as well.
- Also available on macOS/Darwin for `EVFILT_VNODE`-based file watching,
  where Apple positions it as the fine-grained, single-file complement to
  the whole-tree, persistent [[fsevents]] mechanism.
- FreeBSD's `tail -f`, cited in the original design paper as a concrete
  before/after case, switched from polling the file every 1/4 second to a
  VNODE filter — same functionality, less overhead.

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
- `[[bun-watcher]]` — Bun's macOS/FreeBSD backend (`KEventWatcher`)
  registers `EVFILT_VNODE` kevents directly, one fd per watched item, on a
  shared `kqueue` fd — the same fd-per-file model described above, plus its
  own eviction/compaction bookkeeping to reclaim fds as watched items are
  removed.
- [[recursive-watching]] — cross-cutting comparison of tree-watching support across all mechanisms/libraries in this wiki.
- [[kqueue-vs-fsevents]] — systematic axis-by-axis comparison against FSEvents.

## Sources

- [openbsd-kqueue-2](../sources/openbsd-kqueue-2.md)
- [lemon-kqueue-freenix2001](../sources/lemon-kqueue-freenix2001.md)
