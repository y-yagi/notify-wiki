# notify-wiki index

A knowledge base on OS/kernel file-change notification systems.
See `/CLAUDE.md` for how this wiki is organized and maintained.

## Concepts

- [dnotify](concepts/dnotify.md)
- [fanotify](concepts/fanotify.md)
- [inotify](concepts/inotify.md)

Expected coverage: `inotify`, `fanotify`, `dnotify`, `kqueue`/`kevent`,
`FSEvents`, `ReadDirectoryChangesW`, and userspace libraries built on top
(`libuv`, `watchman`, `chokidar`, `notify-rs`, `fsnotify`).

## Comparisons

_No pages yet._

## Sources

- [kernel-dnotify](sources/kernel-dnotify.md) — dnotify.txt (kernel.org)
- [man7-fanotify-7](sources/man7-fanotify-7.md) — fanotify(7) man page (man7.org)
- [man7-inotify-7](sources/man7-inotify-7.md) — inotify(7) man page (man7.org)

## Log

See [log.md](log.md) for the change history.
