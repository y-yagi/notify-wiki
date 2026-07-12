---
title: io_uring
tags: [linux, kernel, async-io]
updated: 2026-07-12
sources: ["../sources/man7-io_uring-7.md", "../sources/lkml-io_uring-fixed-file-inotify-fanotify.md"]
---

# io_uring

## Overview

io_uring is a Linux-specific API for asynchronous I/O, built around a pair of
ring buffers — a submission queue (SQ) and completion queue (CQ) — shared
between kernel and user space via `mmap(2)`. It solves the general problem of
issuing I/O requests (reads, writes, accepts, and many other operation types)
without blocking the calling thread and without a syscall per request, by
letting the app and kernel communicate through shared memory instead of
one-off system calls per operation.

**This is not a file-change-notification API.** The current source describes
io_uring purely as a general async I/O submission/completion mechanism for
one-shot operations; it does not define a persistent "watch this path and
notify me of changes" primitive of its own. Its relevance to this wiki is as
a potential *transport* for other notification mechanisms (e.g. issuing an
async `read(2)`-equivalent against an [[inotify]] file descriptor instead of
blocking on it) rather than as a watching mechanism in its own right — but
that usage isn't described in the ingested source and would need a dedicated
source to confirm.

The one concrete mention connecting io_uring to `inotify`/`fanotify` found so
far is a single line in an August 2021 LKML side-thread: while debating how
to generalize io_uring's "fixed file table" (letting an fd-producing syscall
place its result directly into an io_uring-managed slot instead of the
process fd table), Josh Triplett listed `inotify` and `fanotify` among sixteen
candidate syscalls for this treatment. io_uring maintainer Pavel Begunkov's
reply called the whole generalization "speculations," and the thread ended
without a design proposal or patch. This is about fd-table placement, not
about io_uring reading or delivering notification events — see
[`sources/lkml-io_uring-fixed-file-inotify-fanotify.md`](../sources/lkml-io_uring-fixed-file-inotify-fanotify.md).

## API / semantics

- `io_uring_setup(2)` + `mmap(2)` map in the shared SQ/CQ ring buffers
  (single `mmap(2)` call on kernels ≥ 5.4 if `IORING_FEAT_SINGLE_MMAP` is set;
  otherwise separate calls for SQ, CQ, and the SQE array).
- To submit a request: acquire a submission queue entry (SQE) from the SQ
  tail, fill in `opcode` (e.g. `IORING_OP_READ`, `IORING_OP_READV`), `fd`,
  `addr`/`off`/`len`, and a `user_data` correlation token, then advance the
  tail and call `io_uring_enter(2)` to hand the batch to the kernel.
- The kernel dequeues SQEs from the SQ head, processes them asynchronously,
  and places one completion queue event (CQE) per SQE on the CQ tail
  (barring multishot operations or `IOSQE_CQE_SKIP_SUCCESS`, which can produce
  more or fewer CQEs than SQEs submitted).
- `struct io_uring_cqe` carries `user_data` (echoed back from the SQE),
  `res` (the underlying syscall's return value, or `-errno` on error — errno
  itself is never used since completion is async), and a `flags` bitmask
  (`IORING_CQE_F_BUFFER`, `IORING_CQE_F_MORE`, `IORING_CQE_F_SOCK_NONEMPTY`,
  `IORING_CQE_F_NOTIF`, `IORING_CQE_F_BUF_MORE`, `IORING_CQE_F_SKIP`,
  `IORING_CQE_F_32`, as of the 6.12 kernel per the source).
- Completions can arrive in a different order than submission, since requests
  are processed asynchronously; apps must use `user_data` to match a CQE back
  to its originating SQE.
- Submission Queue Polling (`IORING_SETUP_SQPOLL`) offloads SQ draining to a
  dedicated kernel thread, letting the app skip `io_uring_enter(2)` calls
  entirely at the cost of a permanently running kernel thread.
- Shared-buffer updates (ring head/tail indices) require explicit
  acquire/release memory barriers on both the kernel and user-space sides to
  stay coherent across CPUs.

## Limitations & gotchas

- SQE-referenced buffers can normally be freed once the corresponding
  `io_uring_enter(2)` call returns, *except*: (a) when `IORING_SETUP_SQPOLL`
  is enabled, where the buffer must stay valid until the request actually
  completes, since there's no synchronous `io_uring_enter(2)` return to key
  off of; and (b) for operations that read/write the buffer while inflight
  (e.g. `IORING_OP_READ`/`IORING_OP_WRITE`), where it must stay valid until
  completion regardless of SQPOLL.
- Very early kernels (5.4 and earlier) required all submitted state to remain
  stable until completion regardless of SQPOLL; apps can detect the looser
  behavior via the `IORING_FEAT_SUBMIT_STABLE` flag.
- Ordering across submitted requests is not guaranteed even within a single
  batch, unless the app explicitly links them with `IOSQE_IO_LINK` — relevant
  for anything requiring strict ordering (e.g. sequential sends/receives on
  the same stream socket).

## Related concepts

- [[inotify]] — no ingested source describes io_uring performing
  inotify-style watching itself. The closest connection found is a
  throwaway mention of `inotify`/`fanotify` as candidates for io_uring's
  fixed-file table (fd-table placement, not event delivery) in an August
  2021 LKML thread, met with skepticism and never developed further.

## Sources

- [man7-io_uring-7](../sources/man7-io_uring-7.md)
- [lkml-io_uring-fixed-file-inotify-fanotify](../sources/lkml-io_uring-fixed-file-inotify-fanotify.md)
