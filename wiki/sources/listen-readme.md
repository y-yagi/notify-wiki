---
title: "Source: Listen README"
tags: [listen, userspace-library, ruby, source]
updated: 2026-07-12
---

# listen-readme

**Raw file**: [`raw/listen-readme.md`](../../raw/listen-readme.md)
**Origin**: https://github.com/guard/listen `README.md`, `master` branch.

## What it is

The project README for `listen`, a Ruby gem (best known as the file-watching
backend of `guard`) that watches directories for modifications, additions,
and removals and reports them to a callback.

## What it claims

- Picks one of four OS-specific adapters automatically per platform — Darwin,
  Linux, \*BSD, Windows — falling back to a polling adapter (slower, but works
  everywhere including network filesystems like NFS/Samba/VM shared folders)
  when no native adapter is available or `force_polling` is set. The Darwin
  and Linux adapters ship as gem dependencies; Windows (`wdm`) and \*BSD
  (`rb-kqueue`) adapters require adding a separate gem. Acknowledgments credit
  `rb-inotify` (Linux), `rb-fsevent` (Darwin), `rb-kqueue` (\*BSD), and `wdm`
  (Windows) as the underlying wrapper gems.
- Callback reports three arrays every time — `modified`, `added`, `removed`,
  each of absolute paths — regardless of which adapter is active, and
  `:latency` (delay between change-detection polls) and `:wait_for_delay`
  (delay before invoking the callback once changes exist) are tunable
  independently of which backend is in use.
- Symlink handling is explicitly limited: symlinks are always followed, but a
  symlinked directory that points *within* an already-watched directory isn't
  supported (issue #273/#279 cited, unresolved as of this README).
- Listeners don't cross forked processes — each forked process must call
  `Listen.to` itself to get notifications.
- Devotes a large troubleshooting section to Linux's inotify watch-count
  limit specifically (not the other adapters): documents
  `fs.inotify.max_user_watches`, its default-vs-recommended values, per-watch
  memory cost (~1,080 bytes/watch, cited to a Stack Overflow answer), and
  points at `max_queued_events`/`max_user_instances` as related knobs when
  `max_user_watches` alone doesn't fix it.
- TCP-based cross-process notification existed prior to v3.0.0 and was
  removed; documented as available only by pinning to `listen ~> 2.10`.
- Notes several real-world non-native-watching failure modes: network/VM
  shared filesystems requiring polling, and "a slow encryption stack, e.g.
  btrfs + ecryptfs" as an observed source of unresponsiveness.

## Concept pages informed

- [`concepts/listen.md`](../concepts/listen.md) (new)
