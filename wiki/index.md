# notify-wiki index

A knowledge base on OS/kernel file-change notification systems.
See `/CLAUDE.md` for how this wiki is organized and maintained.

## Concepts

- [chokidar](concepts/chokidar.md)
- [dnotify](concepts/dnotify.md)
- [fanotify](concepts/fanotify.md)
- [findfirstchangenotification](concepts/findfirstchangenotification.md)
- [fsevents](concepts/fsevents.md)
- [fsnotify](concepts/fsnotify.md)
- [fsnotify-go](concepts/fsnotify-go.md)
- [inotify](concepts/inotify.md)
- [kqueue](concepts/kqueue.md)
- [libuv](concepts/libuv.md)
- [notify-rs](concepts/notify-rs.md)
- [readdirectorychangesw](concepts/readdirectorychangesw.md)
- [usn-journal](concepts/usn-journal.md)
- [watchman](concepts/watchman.md)

Expected coverage: `inotify`, `fanotify`, `dnotify`, `kqueue`/`kevent`,
`FSEvents`, `ReadDirectoryChangesW` — all present. Userspace libraries built
on top of these (`libuv`, `watchman`, `chokidar`, `notify-rs`, Go's
`fsnotify`) are now covered too. Remaining known gap: Solaris/illumos FEN
(`port_create`/`port_get`), referenced from `fsnotify-go.md` and
`watchman.md` but with no dedicated concept page yet.

## Comparisons

- [recursive-watching](comparisons/recursive-watching.md) — which mechanisms/libraries support watching a directory tree natively vs. requiring the caller to walk and register every subdirectory itself

## Sources

- [apple-fsevents-progguide](sources/apple-fsevents-progguide.md) — File System Events Programming Guide (Apple, archived)
- [kernel-dnotify](sources/kernel-dnotify.md) — dnotify.txt (kernel.org)
- [msdn-findfirstchangenotificationa](sources/msdn-findfirstchangenotificationa.md) — FindFirstChangeNotificationA function (Microsoft Learn)
- [man7-fanotify-7](sources/man7-fanotify-7.md) — fanotify(7) man page (man7.org)
- [man7-inotify-7](sources/man7-inotify-7.md) — inotify(7) man page (man7.org)
- [msdn-fsutil-usn](sources/msdn-fsutil-usn.md) — fsutil usn command reference (Microsoft Learn)
- [msdn-readdirectorychangesw](sources/msdn-readdirectorychangesw.md) — ReadDirectoryChangesW function (Microsoft Learn)
- [openbsd-kqueue-2](sources/openbsd-kqueue-2.md) — kqueue(2) man page (man.openbsd.org)
- [lwn-fsnotify-unified-backend](sources/lwn-fsnotify-unified-backend.md) — fsnotify unified backend patch mail (LWN.net archive)
- [lwn-fanotify-api](sources/lwn-fanotify-api.md) — "The fanotify API" (LWN.net)
- [libuv-fs-event](sources/libuv-fs-event.md) — libuv `uv_fs_event_t` API docs
- [watchman-cookies](sources/watchman-cookies.md) — Watchman "Query Synchronization" design doc
- [watchman-readme](sources/watchman-readme.md) — Watchman README.markdown
- [chokidar-readme](sources/chokidar-readme.md) — Chokidar README
- [notify-rs-readme](sources/notify-rs-readme.md) — notify-rs README
- [fsnotify-go-readme](sources/fsnotify-go-readme.md) — fsnotify (Go) README

## Log

See [log.md](log.md) for the change history.
