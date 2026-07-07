---
title: fanotify
tags: [linux, kernel, event-driven, access-control]
updated: 2026-07-08
sources: ["../sources/man7-fanotify-7.md"]
---

# fanotify

## Overview

fanotify is a Linux kernel API for filesystem event notification *and*
interception, introduced in Linux 2.6.36 and enabled in 2.6.37. Its motivating
use cases are virus scanning and hierarchical storage management — workloads
that need to see or gate an access before it completes, and/or watch an
entire mount rather than enumerating individual files/directories. Unlike
[[inotify]], it can make access-permission decisions (allow/deny a syscall in
flight) and can identify watched objects by file handle instead of requiring
an open fd per object.

Its create/delete/move event coverage was added late: the original API had no
equivalent of inotify's `IN_CREATE`/`IN_DELETE`/`IN_MOVED_*`, and applications
needing those before Linux 5.1 had to use inotify instead.

## API / semantics

- `fanotify_init(2)` creates a **notification group** — a kernel object
  holding the list of watched files/directories/filesystems/mounts — and
  returns an fd for it.
- `fanotify_mark(2)` adds/removes/modifies entries ("marks") in that group,
  each with a *mark mask* (events to report) and an *ignore mask* (events to
  suppress for that specific object, even though a broader mark would
  otherwise report them — e.g. to stop watching `FAN_MODIFY` on a file until
  it's next closed, for a file-cache use case).
- Marks identify files/directories by inode number and mounts by mount ID.
  Renames/moves within the same mount preserve the mark; deletion or a move
  to another mount removes it.
- **Two event categories**:
  - *Notification events* — informative; the only required action is closing
    any fd the event carries.
  - *Permission events* (`FAN_ACCESS_PERM`, `FAN_OPEN_PERM`, `FAN_OPEN_EXEC_PERM`)
    — block the triggering operation until the listener `write(2)`s a
    `struct fanotify_response { s32 fd; u32 response; }` with `FAN_ALLOW` or
    `FAN_DENY` (optionally OR'd with `FAN_AUDIT` to log the decision, or,
    since Linux 6.13 with `FAN_CLASS_PRE_CONTENT`, `FAN_DENY_ERRNO(e)` to
    return a specific errno other than `EPERM`: `EIO`, `EBUSY`, `ETXTBSY`,
    `EAGAIN`, `ENOSPC`, `EDQUOT`). Two permission events for the same object
    are never merged.
- `read(2)` on the group's fd yields one or more `struct fanotify_event_metadata`
  records (`event_len`, `vers`, `metadata_len`, `mask`, `fd`, `pid`), optionally
  followed by stacked `fanotify_event_info_*` records (`_fid`, `_pidfd`, `_mnt`,
  `_error`, `_range`) depending on which `FAN_REPORT_*` flags were passed to
  `fanotify_init(2)`. `FAN_EVENT_OK`/`FAN_EVENT_NEXT` macros iterate the buffer.
- `fd` in the metadata is an already-open fd for the object (caller must close
  it) — *unless* the group identifies objects by file handle
  (`FAN_REPORT_FID`/`_DFID`/`_DFID_NAME`), in which case `fd` is always
  `FAN_NOFD` and the object is instead identified via the `fanotify_event_info_fid`
  handle, usable with `open_by_handle_at(2)`.
- `pid` is the PID (or, with `FAN_REPORT_TID`, the TID) of the process that
  caused the event — lets a listener recognize and ignore its own accesses.
- Event bits mirror and extend inotify's: `FAN_ACCESS`, `FAN_OPEN`,
  `FAN_OPEN_EXEC`, `FAN_ATTRIB`, `FAN_CREATE`, `FAN_DELETE`, `FAN_DELETE_SELF`,
  `FAN_RENAME`, `FAN_MOVED_FROM`, `FAN_MOVED_TO`, `FAN_MOVE_SELF`, `FAN_MODIFY`,
  `FAN_CLOSE_WRITE`, `FAN_CLOSE_NOWRITE`, plus fanotify-specific
  `FAN_MNT_ATTACH`/`FAN_MNT_DETACH` (mount namespace attach/detach) and
  `FAN_FS_ERROR` (filesystem error detected — one such event is buffered per
  filesystem at a time; further errors just bump `error_count`).
  Convenience macros: `FAN_CLOSE`, `FAN_MOVE`. `FAN_ONDIR` marks that the event
  subject was a directory (only reported when the group uses file handles).
- `/proc/pid/fdinfo/fd` exposes a process's fanotify marks. Since Linux 5.13
  (backported to 5.10.220), `/proc/sys/fs/fanotify/max_queued_events`,
  `max_user_group`, and `max_user_marks` are tunable (previously hardcoded at
  16384 events, 128 groups/user, and 8192 marks/group respectively).
- Closing all fds for a group frees it; outstanding permission events are
  auto-resolved to allowed.

## Limitations & gotchas

- No coverage of network filesystem activity (user-space-triggered local
  syscalls only) and no `mmap(2)`/`msync(2)`/`munmap(2)` coverage — same two
  gaps as inotify.
- A directory's own watch does **not** fire on children being added/removed/
  changed — only on the directory itself being opened/read/closed. Per-directory
  marks are **not recursive**, and the create-then-mark-subdirectory pattern
  is racy in exactly the way inotify's is. Unlike inotify, though, fanotify
  offers a race-free alternative: mark the *mount* or the *filesystem* instead
  of individual directories, covering the whole tree atomically.
- Event queue can overflow (`FAN_Q_OVERFLOW`; raise the cap with
  `FAN_UNLIMITED_QUEUE` at `fanotify_init(2)` time).
- Pre-3.19: `fallocate(2)` generated no fanotify events (fixed in 3.19, now
  generates `FAN_MODIFY` — same fix/timeline as inotify's identical bug).
- As of Linux 3.17 (page notes these as still-open as of writing):
  - Bind-mounted objects: a mount-level mark sees only events triggered
    through *that* mount, missing activity via other mounts of the same object.
  - No UID/permission check before handing a listener an fd for the accessed
    file — a real security risk for `CAP_SYS_ADMIN` processes run by
    unprivileged users.
  - If `read(2)` copies multiple events and then hits an error, the return
    value is just the bytes successfully copied — not `-1` — so the caller
    has no way to detect that an error occurred mid-read.
- Permission handling requires `CONFIG_FANOTIFY_ACCESS_PERMISSIONS` in
  addition to the base `CONFIG_FANOTIFY` option.

## Platform notes

- Introduced Linux 2.6.36, enabled 2.6.37.
- `fdinfo` support added in Linux 3.8.
- Create/delete/move events (`FAN_CREATE`, `FAN_DELETE`, `FAN_MOVED_FROM`,
  `FAN_MOVED_TO`, `FAN_RENAME`, etc.) added in Linux 5.1 — absent before that.
- `fallocate(2)` → `FAN_MODIFY` since Linux 3.19.
- `/proc/sys/fs/fanotify/*` tunables since Linux 5.13 (and backported to 5.10.220).
- `FAN_INFO` extra response records (e.g. `FAN_RESPONSE_INFO_AUDIT_RULE`)
  since Linux 6.3.
- `FAN_DENY_ERRNO(e)` custom-errno permission denial since Linux 6.13, gated
  on `FAN_CLASS_PRE_CONTENT`.
- Linux-only (`STANDARDS: Linux`).

## Related concepts

- `[[inotify]]` — the older, lighter-weight sibling API. fanotify adds
  mount/filesystem-wide watching and access-permission interception that
  inotify cannot do; inotify had create/delete/move event coverage first
  (fanotify only gained it in Linux 5.1).
- `[[dnotify]]` — inotify's own predecessor; not directly related to fanotify
  beyond being an earlier point in the same lineage.

## Sources

- [man7-fanotify-7](../sources/man7-fanotify-7.md)
