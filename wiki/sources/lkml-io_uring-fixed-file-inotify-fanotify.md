---
title: "Source: LKML thread on io_uring fixed-file-table candidates (inotify/fanotify mention)"
tags: [linux, kernel, source]
updated: 2026-07-12
---

# lkml-io_uring-fixed-file-inotify-fanotify

**Raw file**: [`raw/lkml-io_uring-fixed-file-inotify-fanotify.txt`](../../raw/lkml-io_uring-fixed-file-inotify-fanotify.txt)
**Origin**: linux-kernel mailing list, August 2021, archived on the lkml.iu.edu hypermail
mirror (folder `linux/kernel/2108.1/`), fetched 2026-07-12. Three messages:
[07437](https://lkml.iu.edu/hypermail/linux/kernel/2108.1/07437.html) (Josh Triplett),
[07528](https://lkml.iu.edu/hypermail/linux/kernel/2108.1/07528.html) and
[07581](https://lkml.iu.edu/hypermail/linux/kernel/2108.1/07581.html) (both Pavel Begunkov).

## What it is

A three-message side-thread within the review of the patch series "open/accept
directly into io_uring fixed file table" (Aug 2021). The parent patch series
itself is about letting `open`/`accept` place their resulting fd directly into
io_uring's internal "fixed file table" (an io_uring-managed fd slot table,
bypassing the process fd table) rather than about file-change notification at
all.

## What it claims

- The core of the thread is a design debate between Josh Triplett and Pavel
  Begunkov (an io_uring maintainer) over the wire encoding for the
  fixed-file-table index (dedicated `IOSQE_OPEN_FIXED` flag vs. overloading an
  existing SQE field vs. a 16-bit vs. 32-bit index) — none of which is
  file-notification-specific.
- In arguing that the mechanism should generalize beyond `open`/`accept`, Josh
  Triplett lists candidate fd-producing syscalls that could similarly target
  the fixed-file table: "pipe, dup3, socket, socketpair, pidfds (...),
  epoll_create, **inotify, fanotify**, signalfd, timerfd, eventfd,
  memfd_create, userfaultfd, open_tree, fsopen, fsmount, memfd_secret." He
  says he'd personally want pipe/socket/pidfd/memfd_create/fsopen &c. most —
  inotify/fanotify are named but not singled out as a priority.
- Pavel Begunkov's response is skeptical of the whole generalization: "We
  could argue for many of those whether they should be in io_uring, and
  whether there are many benefits having them async and so. ... that's
  speculations."
- The thread ends (message 3/3) on unresolved bikeshedding about SQE field
  encoding; there is no further follow-up on inotify/fanotify specifically
  within this thread, and no patch implementing it.

**Bottom line for this wiki**: this is the one concrete LKML mention found
connecting io_uring to inotify/fanotify, but it is a single line in a list of
sixteen candidate syscalls, raised as a hypothetical and met with skepticism —
not a design proposal, RFC, or patch for inotify/fanotify support in io_uring.
It does not describe or confirm async reading of inotify/fanotify events via
io_uring; it's about fd-table placement, a different and narrower concern.

## Concept pages informed

- [`concepts/io_uring.md`](../concepts/io_uring.md) (updated)
