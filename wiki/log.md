# Change log

## 2026-07-09
- Ingested `raw/apple-dev-fseventstreamcreateflagfileevents.txt` (Apple
  Developer Docs live page content for `kFSEventStreamCreateFlagFileEvents`
  — availability badges + Discussion text, pasted by the user since the page
  is a JS-rendered SPA). Added
  `wiki/sources/apple-dev-fseventstreamcreateflagfileevents.md`. Resolved the
  "unconfirmed OS version" caveat left on `apple-fsevents-h.md` and
  `concepts/fsevents.md`'s Platform notes: confirmed `FileEvents` is
  available since macOS 10.7+ and Mac Catalyst 13.1+. Discussion text matched
  the header's doc comment verbatim, cross-confirming the earlier header
  mirror's accuracy rather than adding new descriptive content.
- Ingested `raw/apple-fsevents-h.txt` (Apple's `CarbonCore/FSEvents.h` header,
  MacOSX10.9 SDK, mirrored verbatim via the `phracker/MacOSX-SDKs` GitHub
  repo — used in place of the JS-rendered live Apple docs page). Added
  `wiki/sources/apple-fsevents-h.md`. Updated `concepts/fsevents.md`: replaced
  the blanket "Not for single-file monitoring" limitation with a corrected
  one — `kFSEventStreamCreateFlagFileEvents` (opt-in) does provide per-file
  granularity (`ItemCreated`/`ItemRemoved`/`ItemRenamed`/`ItemModified`/etc.
  plus `ItemIsFile`/`ItemIsDir`/`ItemIsSymlink`), at the cost of significantly
  more events; noted this doesn't change FSEvents' underlying daemon+DB
  architecture, and that whether Apple still recommends kqueue for
  single-file watching post-FileEvents isn't confirmed by any source here.
  Also updated Platform notes with the flag's documented behavior, replacing
  the earlier "not covered here" gap. Flagged that the header itself doesn't
  carry an explicit OS-version-gating macro for this flag — the "OS X 10.7"
  attribution rests on outside knowledge, not this raw source.
- Ingested `raw/wikipedia-robert-love.txt` (Wikipedia's "Robert Love" article,
  fetched/summarized 2026-07-09 — not verbatim wikitext, flagged as
  lower-confidence than the wiki's other `raw/` sources). Added
  `wiki/sources/wikipedia-robert-love.md` as background/credibility context
  for the Quora design-rationale source below; cross-linked from
  `wiki/sources/quora-love-inotify-recursive.md`. Not linked into any
  `concepts/` page — purely biographical, out of this wiki's technical scope.
- Ingested `raw/quora-love-inotify-recursive.txt` (Robert Love's Quora answer
  explaining why inotify doesn't support recursive watching — a
  design-rationale source pasted verbatim by the user after the live page
  returned HTTP 403 to automated fetches). Added
  `wiki/sources/quora-love-inotify-recursive.md`. Updated the existing "Not
  recursive" bullet in `concepts/inotify.md` with the design rationale
  (kernel-side recursive traversal isn't a first-class Unix filesystem
  operation and would be too long-running under kernel locks; intended to be
  implemented in userspace instead, with the resulting add-watch/scan race
  addressed by the "Love-Trowbridge" algorithm) and added the source to its
  Sources list. Added a matching note to `comparisons/recursive-watching.md`'s
  Takeaways, explicitly scoped to inotify only — no source here documents
  dnotify's or kqueue's own design discussions, so the same rationale isn't
  assumed to generalize to them.

## 2026-07-08
- Ran a `/lint` health check across `wiki/`. Findings: no contradictions, no
  stale `updated` dates, no orphan pages (every concept page has ≥1 inbound
  link), no un-ingested raw sources, no frontmatter gaps. One intentional
  broken link (`[[fen]]` in `watchman.md`, already logged as a known gap)
  and three unwritten-but-mentioned concepts: FEN (Solaris/illumos), Apple's
  Endpoint Security (mentioned in `fsevents.md`/`watchman.md` as FSEvents'
  suggested replacement, no page), and io_uring-based watching (mentioned
  only in `CLAUDE.md`'s scope, never in `wiki/` body text).
- Added `wiki/comparisons/recursive-watching.md`, the first page in
  `comparisons/`: a table comparing native recursive-watch support across
  all 7 OS mechanisms and 5 userspace libraries in the wiki, distinguishing
  three models (native recursive watch / no recursion primitive / not
  path-scoped at all). Highlights fanotify's mount/filesystem-wide marks as
  the only race-free tree-wide primitive among the OS mechanisms. Linked
  back from all 12 referenced concept pages' Related concepts sections, and
  registered in `wiki/index.md`.
- While building the comparison, caught and fixed an unsourced claim in
  `concepts/fsnotify-go.md`: it asserted notify-rs "defaults to recursive
  watching," but the ingested notify-rs README never states a recursive-
  watch default either way. Softened to note this is unconfirmed rather than
  assert it.
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
