# Change log

## 2026-07-08
- Ingested userspace-library sources: `raw/libuv-fs-event.txt`,
  `raw/watchman-cookies.md`, `raw/watchman-readme.md`,
  `raw/chokidar-readme.md`, `raw/notify-rs-readme.md`,
  `raw/fsnotify-go-readme.md`. Added matching `wiki/sources/*.md` pages and 5
  new concept pages: `concepts/libuv.md`, `concepts/watchman.md`,
  `concepts/chokidar.md`, `concepts/notify-rs.md`, `concepts/fsnotify-go.md`
  (named `fsnotify-go` rather than `fsnotify` to avoid collision with the
  Linux kernel backend page). Cross-linked each to the OS-level concept
  page(s) it wraps. Notable finds: Watchman's cookie-synchronization design
  doc documents a real production case where macOS FSEvents delivered
  events *after* `FSEventStreamFlushSync` returned — added as a new
  Limitations bullet + source on `concepts/fsevents.md`. Go's `fsnotify` is
  the only library here with a working illumos/Solaris FEN backend, which
  has no dedicated concept page yet (linked as `[[fen]]` from
  `watchman.md`/`fsnotify-go.md` as a forward reference, not yet broken —
  flagging as the next known gap). Updated `wiki/index.md`.
- Ingested `raw/lwn-fsnotify-unified-backend.txt` (Eric Paris's original fsnotify
  patch mail, Feb 2009, archived on LWN) and `raw/lwn-fanotify-api.txt` ("The
  fanotify API" by Jonathan Corbet, LWN, July 2009). Added
  `wiki/sources/lwn-fsnotify-unified-backend.md` and
  `wiki/sources/lwn-fanotify-api.md`, and a new `wiki/concepts/fsnotify.md`
  (the shared in-kernel backend for dnotify/inotify/fanotify: group/dispatch
  model, `FS_*`/`IN_*` bit parity, `*_CHILD` bits, why fanotify's global-
  listener mode is what actually forced the common backend into existence).
  Cross-linked from and into `concepts/inotify.md`, `concepts/dnotify.md`,
  `concepts/fanotify.md`. Updated `wiki/index.md`. (Note: the LWN article
  describes fanotify's originally-proposed socket/`getsockopt()`-based API,
  which differs from the syscall-based API that actually shipped and is
  documented in `concepts/fanotify.md` — flagged in both pages rather than
  treated as a contradiction, since both are accurate for their own point in
  time.)
- Ingested `raw/msdn-fsutil-usn.txt` (`fsutil usn` command reference, Microsoft Learn). Added `wiki/sources/msdn-fsutil-usn.md` and `wiki/concepts/usn-journal.md` (NTFS USN change journal: persistent whole-volume append-only log, `fsutil usn` subcommands, journal-disable cost/blast-radius, range tracking). Cross-linked from `concepts/readdirectorychangesw.md`. Updated `wiki/index.md`.
- Ingested `raw/msdn-readdirectorychangesw.txt` (Win32 `ReadDirectoryChangesW` reference, Microsoft Learn). Added `wiki/sources/msdn-readdirectorychangesw.md` and `wiki/concepts/readdirectorychangesw.md` (FILE_NOTIFY_INFORMATION-based reporting, sync/async/IOCP delivery, buffer-overflow-discards-changes gotcha, network buffer cap). Resolved the `ReadDirectoryChangesW` placeholder in `concepts/findfirstchangenotification.md`; cross-linked from `concepts/inotify.md` and `concepts/fsevents.md`. Updated `wiki/index.md`.
- Ingested `raw/msdn-findfirstchangenotificationa.txt` (Win32 `FindFirstChangeNotificationA` reference, Microsoft Learn). Added `wiki/sources/msdn-findfirstchangenotificationa.md` and `wiki/concepts/findfirstchangenotification.md` (handle-based wait model, filter flags, no change-detail reporting, symlink caveats, platform/network-FS caveats). Cross-linked from `concepts/inotify.md` and `concepts/fsevents.md`. Updated `wiki/index.md`. (Note: originally-requested ja-jp URL required sign-in/authorization when fetched; used the en-us URL instead per user correction.)
- Ingested `raw/apple-fsevents-progguide.txt` (Apple's archived File System Events Programming Guide, 5 chapters, 2012). Added `wiki/sources/apple-fsevents-progguide.md` and `wiki/concepts/fsevents.md` (stream lifecycle, coalescing/latency, event flags, persistence across reboots, security model, limitations). Cross-linked from `concepts/kqueue.md`. Updated `wiki/index.md`. (Note: live `developer.apple.com/documentation/coreservices/file_system_events` is a JS-rendered SPA and couldn't be fetched as static text; used this archived guide instead — see judgment note below.)
- Ingested `raw/openbsd-kqueue.2.txt` (kqueue(2) man page, OpenBSD). Added `wiki/sources/openbsd-kqueue-2.md` and `wiki/concepts/kqueue.md` (kqueue/kevent API, filter model, EVFILT_VNODE for file events, other filters noted for context, limitations, platform history). Cross-linked from `concepts/inotify.md`. Updated `wiki/index.md`.
- Ingested `raw/man7-fanotify.7.txt` (fanotify(7) man page, man-pages 6.18). Added `wiki/sources/man7-fanotify-7.md` and `wiki/concepts/fanotify.md` (notification groups, marks, permission events, FID-based reporting, mount/filesystem-wide watching, limitations, version history). Cross-linked from `concepts/inotify.md`. Updated `wiki/index.md`.
- Ingested `raw/kernel-dnotify.txt` (Documentation/filesystems/dnotify.txt). Added `wiki/sources/kernel-dnotify.md` and `wiki/concepts/dnotify.md` (fcntl/signal-based API, event set, hard-link caveat, superseded-by-inotify note). Cross-linked from `concepts/inotify.md`. Updated `wiki/index.md`.
- Ingested `raw/man7-inotify.7.txt` (inotify(7) man page, man-pages 6.18). Added `wiki/sources/man7-inotify-7.md` and `wiki/concepts/inotify.md` (syscalls, event struct, watch model, flags, limitations, version history). Updated `wiki/index.md`.
- Scaffolded the wiki: `raw/`, `wiki/{concepts,comparisons,sources}/`, `wiki/index.md`, `wiki/log.md`, and `/CLAUDE.md` (schema/conventions). No sources ingested yet.
