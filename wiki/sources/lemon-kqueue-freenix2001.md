---
title: "Source: Lemon, \"Kqueue: A generic and scalable event notification facility\""
tags: [bsd, kernel, freebsd, source]
updated: 2026-07-12
---

# lemon-kqueue-freenix2001

**Raw file**: `raw/lemon-kqueue-freenix2001.pdf` — **currently missing**.
This file was never committed to git (no `.gitignore` rule excludes PDFs,
and `git log` shows it was never tracked), so it isn't recoverable from
version history. It evidently existed locally when this page was written
(2026-07-12) but is absent as of 2026-07-19. The summary below is kept as
the audit trail of what the paper claimed, but none of it can be
re-verified against the source until the PDF is restored to `raw/`.
**Origin**: Jonathan Lemon (FreeBSD Project), *"Kqueue: A generic and scalable
event notification facility"*, FREENIX Track, 2001 USENIX Annual Technical
Conference (Boston, MA). Note: often cited/searched as "USENIX 2000" because
kqueue itself shipped in FreeBSD 4.1 in July 2000 and was committed to the
CVS tree in April 2000 — but the paper itself was published in 2001.

## What it is

The original design paper for kqueue/kevent, written by its author. Unlike
the OpenBSD man page already ingested
([`openbsd-kqueue-2`](openbsd-kqueue-2.md)), which documents the finished
API as a reference, this is a first-person account of *why* it was designed
the way it was, with measured performance data against `poll()`/`select()`.

## What it claims

- **Problem motivating kqueue**: `poll()`/`select()` don't scale because they
  are stateless between calls — the kernel holds no memory of what the
  application is interested in, so every call re-copies and re-walks the
  entire descriptor list (three passes: build/copy in, scan for activity,
  scan again in userspace), an O(N) cost that dominates as N grows into the
  thousands even though typically only a few hundred descriptors are
  actually active.
- **Design goals** (explicitly enumerated, Section 3): scalable to thousands
  of descriptors; flexible enough to cover event sources poll/select can't
  (file creation/deletion, signals, AIO completion) without changing the
  application-facing API to add new sources later; simple enough to convert
  existing poll/select code with minimal changes; able to return richer
  per-event info than a single bit (e.g. bytes pending, listen backlog
  size); **reliable** — no silent failure, no fixed-size lists that could
  overflow, memory allocated at syscall time rather than when activity
  occurs (to avoid losing events under memory pressure); events are
  **coalesced** (multiple packets arriving become one queued event, not one
  per packet) and **level-triggered**, not edge-triggered (an event is
  reported as long as its condition holds, matching poll/select semantics,
  rather than only at the instant it's first detected); **correct** — a
  closed descriptor's pending event must not be delivered, and interest
  registered *before* an event source is active must still correctly catch
  a pre-existing condition (race-free registration); and safe for **library
  use** — usable by third-party code sharing a process with the main
  program without conflicting over global resources, unlike signals (one
  handler per process) or a single select()/poll() event loop.
- **API**: `kqueue()`/`kqueue1()` create the queue (an ordinary fd);
  `kevent(kq, changelist, nchanges, eventlist, nevents, timeout)` both
  applies registration changes and retrieves fired events in one call,
  reducing syscalls versus separate register/poll steps. A kevent is
  uniquely identified by `<kq, ident, filter>`. `EV_CLEAR` resets a filter's
  state after delivery — used, per the paper's own example, for the VNODE
  filter specifically, since without it an event would be repeatedly
  returned.
- **Original filter set** (Section 4.1) was `EVFILT_READ`, `EVFILT_WRITE`,
  `EVFILT_AIO`, `EVFILT_VNODE`, `EVFILT_PROC`, `EVFILT_SIGNAL` — no
  `EVFILT_TIMER`, `EVFILT_DEVICE`, or `EVFILT_USER` yet (those appear in the
  current OpenBSD man page but not this 2001 design paper; the paper's
  "Conclusion" section does mention a TIMER filter as then "in the process
  of being added").
- **VNODE filter's original action set** (Figure 3 in this paper):
  `NOTE_DELETE`, `NOTE_WRITE`, `NOTE_EXTEND`, `NOTE_ATTRIB`, `NOTE_LINK`,
  `NOTE_RENAME` — **no `NOTE_TRUNCATE` or `NOTE_REVOKE`** here, both of which
  the already-ingested OpenBSD man page documents as part of the modern
  `EVFILT_VNODE`. Read as filter-set growth over time, not a contradiction.
- **fork() behavior, precisely stated**: "the current implementation does
  not pass kqueue descriptors to children unless the new child will share
  its file table with the parent via `vfork(RFFDG)`" — a narrower, more
  specific claim than the OpenBSD man page's blanket "not inherited across
  `fork(2)`."
- **No discussion of recursive/tree-wide directory watching anywhere in the
  paper** — searched specifically for this (the paper covers single-vnode
  file-level filtering only, via a descriptor to one open file or
  directory) and found no design rationale, in either direction, for why
  `EVFILT_VNODE` has no subtree/recursive primitive. This mirrors the gap
  already flagged in [[kqueue]] — even the original design paper doesn't
  motivate the lack of recursion the way Robert Love's Quora answer does
  for inotify.
- **Performance results**: measured on FreeBSD 4.3-RC (Pentium-III 600MHz);
  kqueue registration costs roughly 2x a single poll() call per descriptor,
  but querying an already-registered, inactive descriptor is near-zero cost
  versus poll's constant per-descriptor cost regardless of activity —
  crossover point is "unless polled less than 4 times, kqueue has lower
  overall cost than poll." Real-world web proxy cache and thttpd benchmarks
  showed the kqueue-converted thttpd retaining ~48% idle CPU at 10,000 idle
  connections versus the unmodified server exhausting CPU around 600
  connections.
- **Related work compared** (Section 8): POSIX signal queues (fixed-size,
  drops events under load, "stale event" problem when a descriptor is
  reused between queuing and delivery); SGI's `/dev/imon` (closest prior
  analogue to `EVFILT_VNODE`, but only one process can read the device node
  at a time); Sun's `/dev/poll` (closest overall design predecessor, but
  socket-only, no FIFO/signal/filesystem events, no extensible filter
  model, no hierarchical/priority queues since the fd it returns can't
  itself be polled).
- **Adoption at time of writing**: implemented in FreeBSD (committed April
  2000), adopted by OpenBSD, "in the process of being brought into NetBSD."
  Cites `tail -f`'s FreeBSD implementation switching from 1/4-second
  polling to a VNODE filter as a concrete before/after example.

## Concept pages informed

- [`concepts/kqueue.md`](../concepts/kqueue.md) (updated: design goals,
  filter-set history, fork() nuance, confirmed absence of a recursion
  rationale)
