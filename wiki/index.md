# notify-wiki index

A knowledge base on OS/kernel file-change notification systems.
See `/CLAUDE.md` for how this wiki is organized and maintained.

## Concepts

- [dnotify](concepts/dnotify.md)
- [fanotify](concepts/fanotify.md)
- [findfirstchangenotification](concepts/findfirstchangenotification.md)
- [fsevents](concepts/fsevents.md)
- [inotify](concepts/inotify.md)
- [kqueue](concepts/kqueue.md)
- [readdirectorychangesw](concepts/readdirectorychangesw.md)
- [usn-journal](concepts/usn-journal.md)

Expected coverage: `inotify`, `fanotify`, `dnotify`, `kqueue`/`kevent`,
`FSEvents`, `ReadDirectoryChangesW`, and userspace libraries built on top
(`libuv`, `watchman`, `chokidar`, `notify-rs`, `fsnotify`).

## Comparisons

_No pages yet._

## Sources

- [apple-fsevents-progguide](sources/apple-fsevents-progguide.md) — File System Events Programming Guide (Apple, archived)
- [kernel-dnotify](sources/kernel-dnotify.md) — dnotify.txt (kernel.org)
- [msdn-findfirstchangenotificationa](sources/msdn-findfirstchangenotificationa.md) — FindFirstChangeNotificationA function (Microsoft Learn)
- [man7-fanotify-7](sources/man7-fanotify-7.md) — fanotify(7) man page (man7.org)
- [man7-inotify-7](sources/man7-inotify-7.md) — inotify(7) man page (man7.org)
- [msdn-fsutil-usn](sources/msdn-fsutil-usn.md) — fsutil usn command reference (Microsoft Learn)
- [msdn-readdirectorychangesw](sources/msdn-readdirectorychangesw.md) — ReadDirectoryChangesW function (Microsoft Learn)
- [openbsd-kqueue-2](sources/openbsd-kqueue-2.md) — kqueue(2) man page (man.openbsd.org)

## Log

See [log.md](log.md) for the change history.
