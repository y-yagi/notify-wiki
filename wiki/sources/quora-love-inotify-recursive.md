---
title: "Source: Robert Love on why inotify isn't recursive (Quora)"
tags: [linux, kernel, source, design-rationale]
updated: 2026-07-09
---

# quora-love-inotify-recursive

**Raw file**: [`raw/quora-love-inotify-recursive.txt`](../../raw/quora-love-inotify-recursive.txt)
**Origin**: Quora answer by Robert Love (co-designer of inotify, with John
McCutchan) to "Inotify monitoring of directories is not recursive. Is there
any specific reason for this design in Linux kernel?"
https://www.quora.com/Inotify-monitoring-of-directories-is-not-recursive-Is-there-any-specific-reason-for-this-design-in-Linux-kernel
(page returns HTTP 403 to automated fetches; text pasted verbatim by the user
from the live page on 2026-07-09).

## What it is

A first-person design-rationale account from one of inotify's two original
authors, answering the "why" question directly rather than just documenting
the resulting behavior (which the man page and other sources already cover).
Background on the author: [wikipedia-robert-love](wikipedia-robert-love.md).

## What it claims

- Love and McCutchan originally *wanted* recursive notification in inotify;
  Love's own motivating use case was desktop indexing (Beagle) watching a
  whole home directory.
- They dropped it because recursive directory traversal isn't a first-class
  kernel operation on Unix-style filesystems — it's iterative listing +
  descend, same as userspace `find(1)` — and doing that inside the kernel,
  especially while holding locks, would be too long-running an operation.
  Since the kernel has no inherent advantage over userspace at this but does
  bear extra cost (locking, latency), it made sense to leave recursion out.
- The intent was for userspace libraries to implement the recursive walk
  themselves; it would still be slow (directory traversal is inherently
  slow, and most filesystems have no cheap way to enumerate all directories)
  but no slower than if the kernel did it.
- Acknowledges the known downside of the userspace-recursion approach: a race
  where a subdirectory created during the tree walk ends up unwatched. States
  this is addressed by an algorithm they called "Love-Trowbridge", discussed
  on the GNOME dashboard-hackers mailing list, October 2004
  (https://mail.gnome.org/archives/dashboard-hackers/2004-October/thread.html#00022).

## Concept pages informed

- [`concepts/inotify.md`](../concepts/inotify.md) — added design-rationale
  detail to the existing "Not recursive" limitation bullet.
- [`comparisons/recursive-watching.md`](../comparisons/recursive-watching.md) —
  added a note on *why* path-based kernel mechanisms tend to avoid a native
  recursive primitive, per this source (specific to inotify's design
  discussion, not confirmed to generalize to dnotify/kqueue's own design
  history).
