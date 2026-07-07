---
title: "Source: fanotify(7) man page (man7.org)"
tags: [linux, kernel, source]
updated: 2026-07-08
---

# man7-fanotify-7

**Raw file**: [`raw/man7-fanotify.7.txt`](../../raw/man7-fanotify.7.txt)
**Origin**: https://man7.org/linux/man-pages/man7/fanotify.7.html (man-pages 6.18, page dated 2026-02-14, fetched from tarball on 2026-05-24)

## What it is

The canonical Linux manual page (section 7, overview) for fanotify, a
filesystem-wide notification-and-interception API. Written/maintained as part
of the man-pages project.

## What it claims

- Core syscalls: `fanotify_init(2)` (create a "notification group"),
  `fanotify_mark(2)` (add/remove/modify marks on files, dirs, mounts, or whole
  filesystems), `read(2)`/`write(2)` on the fanotify fd, `close(2)`.
- Two event categories: **notification events** (informative only) and
  **permission events** (`FAN_*_PERM`, block the triggering syscall until the
  listener `write(2)`s a `fanotify_response` granting/denying access).
- Originally (Linux 2.6.36/2.6.37) had no create/delete/move events; those
  were added in Linux 5.1 — before that, inotify was the only option for them.
- Beyond inotify: can watch an entire mount or filesystem in one mark
  (race-free recursive-ish coverage), can identify objects by file handle
  (`FAN_REPORT_FID` family) instead of requiring an open fd, can attach pidfd/
  mount-id/error/range info records, and — uniquely — can intercept and deny
  access via permission events.
- Documents `fanotify_event_metadata` and the various stacked
  `fanotify_event_info_*` structures, plus the `FAN_EVENT_OK`/`FAN_EVENT_NEXT`
  iteration macros.
- `/proc/sys/fs/fanotify/{max_queued_events,max_user_group,max_user_marks}`
  tunables, with the pre-5.13 hardcoded defaults noted.
- Limitations: no network-filesystem coverage, no mmap/msync/munmap coverage,
  directory-content-change events don't fire on the directory's own watch
  unless it's a mount/filesystem-wide mark, per-directory marks are not
  recursive (same racy add-mark-after-create pattern as inotify, but mount-
  or filesystem-wide marks sidestep it), queue can overflow.
- BUGS: pre-3.19 `fallocate(2)` silence (mirrors the same bug documented for
  inotify), bind-mount blind spots, a UID/permission-check gap when handing
  out fds with `CAP_SYS_ADMIN`, and a `read(2)` partial-copy error-reporting gap.
- Two full example C programs (fd-based permission-event listener; file-handle-
  based FAN_CREATE listener).

## Concept pages informed

- [`concepts/fanotify.md`](../concepts/fanotify.md) (new)
- [`concepts/inotify.md`](../concepts/inotify.md) (added fanotify as a related concept, replacing the earlier "not yet written up" placeholder)
