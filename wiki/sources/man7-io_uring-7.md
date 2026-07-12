---
title: "Source: io_uring(7) man page (man7.org)"
tags: [linux, kernel, source]
updated: 2026-07-12
---

# man7-io_uring-7

**Raw file**: [`raw/man7-io_uring.7.txt`](../../raw/man7-io_uring.7.txt)
**Origin**: https://man7.org/linux/man-pages/man7/io_uring.7.html (liburing project page, dated 2020-07-26, upstream commit as of 2026-05-18, fetched 2026-05-24 per its COLOPHON)

## What it is

The man7.org overview page (section 7) for io_uring, Linux's asynchronous I/O
API. Maintained as part of the liburing project rather than the core
man-pages project.

## What it claims

- Describes the general io_uring model: a submission queue (SQ) and completion
  queue (CQ), both ring buffers shared between kernel and user space via `mmap(2)`,
  set up with `io_uring_setup(2)`.
- Walks through the request lifecycle: fill a submission queue entry (SQE)
  with an opcode (e.g. `IORING_OP_READ`, `IORING_OP_READV`) and parameters,
  add it to the SQ tail, call `io_uring_enter(2)` to hand it to the kernel,
  then dequeue a matching completion queue event (CQE) from the CQ head. Each
  SQE gets exactly one matching CQE (unless multishot/`IOSQE_CQE_SKIP_SUCCESS`
  behavior is configured), correlated via the `user_data` field since
  completions can arrive out of order.
- Documents `struct io_uring_sqe` and `struct io_uring_cqe` layouts, and the
  memory-ordering (acquire/release barrier) requirements for updating shared
  ring head/tail indices safely across kernel/user space.
- Documents Submission Queue Polling (`IORING_SETUP_SQPOLL`): a kernel thread
  polls the SQ so the app can skip the `io_uring_enter(2)` call entirely,
  trading a dedicated kernel thread for fewer syscalls.
- Notes SQE pointer-lifetime rules: buffers referenced by pointer in an SQE
  can generally be freed once `io_uring_enter(2)` returns (unless SQPOLL is
  enabled, or the operation reads/writes the buffer while inflight, e.g.
  `IORING_OP_READ`/`IORING_OP_WRITE`).
- Frames the performance case for io_uring as avoiding both per-request syscall
  overhead (worse on hardware with Spectre/Meltdown mitigations) and buffer-copy
  overhead, via batching multiple SQEs into one `io_uring_enter(2)` call.
- Includes a full worked C example (stdin-to-stdout copy) and lists
  `io_uring_enter(2)`, `io_uring_register(2)`, `io_uring_setup(2)` as related pages.

**Notably absent**: this page does not mention file-change notification,
`inotify`, `fanotify`, or any watch/event-subscription primitive. It documents
io_uring purely as a general-purpose async I/O submission/completion
mechanism (reads, writes, accepts, etc. issued as one-shot requests) — it does
not describe a persistent "watch for changes and notify me" API of its own.

## Concept pages informed

- [`concepts/io_uring.md`](../concepts/io_uring.md) (new)
