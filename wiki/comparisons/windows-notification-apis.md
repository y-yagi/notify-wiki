---
title: Windows notification APIs — history, feature differences, and current recommendation
tags: [windows, win32, ntfs, comparison]
updated: 2026-07-19
sources: ["../sources/msdn-findfirstchangenotificationa.md", "../sources/msdn-readdirectorychangesw.md", "../sources/msdn-readdirectorychangesexw.md", "../sources/oldnewthing-readdirectorychangesw-deletion-details.md", "../sources/msdn-fsutil-usn.md", "../sources/msdn-fsctl-query-usn-journal.md", "../sources/msdn-fsctl-query-usn-journal-win32.md", "../sources/msdn-using-the-change-journal-identifier.md", "../sources/msdn-walking-a-buffer-of-change-journal-records.md", "../sources/msdn-fsctl-read-usn-journal.md"]
---

# Windows notification APIs — history, feature differences, and current recommendation

Windows has accumulated four distinct file-change-notification mechanisms
over roughly 25 years, none of which fully replaced the ones before it —
each occupies a different point on the detail/scope/cost trade-off. This
page traces that history, compares them axis by axis, and gives a
recommendation for what to reach for today. See each concept page
([[findfirstchangenotification]], [[readdirectorychangesw]],
[[readdirectorychangesexw]], [[usn-journal]]) for full detail; this page
only summarizes and cross-links.

## Timeline

| API | Minimum OS | What it added |
|---|---|---|
| [[findfirstchangenotification]] | Windows XP / Server 2003 | First Win32 notification API: a waitable handle, signaled on matching change, **no change detail** |
| [[readdirectorychangesw]] | Windows XP / Server 2003 (same release) | Structured `FILE_NOTIFY_INFORMATION` records — filename + operation type — plus sync/async (overlapped, IOCP) delivery |
| [[usn-journal]] (`FSCTL_QUERY_USN_JOURNAL`/`FSCTL_READ_USN_JOURNAL`) | Windows XP / Server 2003 (same era, different design point) | Whole-**volume**, persistent, replayable change log — not a live per-directory watch at all |
| [[readdirectorychangesexw]] | Windows 10 v1709 / Server 2019 | Adds `FILE_NOTIFY_EXTENDED_INFORMATION` output option: attributes + size attached to *both* add and remove records |

Two observations from the wiki's sources:

- `FindFirstChangeNotification` and `ReadDirectoryChangesW` shipped in the
  **same** minimum-OS generation (Windows XP / Server 2003) — they were
  never sequential upgrades of each other so much as a coarse/fine pair
  offered together from the start. `ReadDirectoryChangesW` was the detailed
  option even at launch; `FindFirstChangeNotification` was the lightweight
  one for callers who only need a "something changed, go rescan" signal.
- `ReadDirectoryChangesExW` is the only API in this set with a genuinely
  recent minimum version (2017/2019) — an actual sequential addition, not a
  same-generation sibling. It targets one specific, narrow gap in
  `ReadDirectoryChangesW` (see below), not a general replacement.
- The USN journal is architecturally unrelated to the other three — it was
  never meant as a live-watch replacement, and Microsoft's own docs
  position it as the tool for whole-volume tracking specifically, pointing
  to it explicitly from `ReadDirectoryChangesW`'s reference page.

## Comparison table

| Axis | `FindFirstChangeNotification` | `ReadDirectoryChangesW` | `ReadDirectoryChangesExW` | USN Change Journal |
|---|---|---|---|---|
| Change detail | None — presence/absence signal only | Filename + operation type (`FILE_NOTIFY_INFORMATION`) | Same, plus optional extended records with attributes+size on add **and** remove | Full structured records (`USN_RECORD_V2`/`V3`): filename, reason bitmask, USN, timestamps |
| Scope | One directory or subtree | One directory or subtree | One directory or subtree | Whole NTFS **volume** |
| Delivery model | Wait on a handle, re-arm each time | Live buffer, drained via sync or async (overlapped/IOCP) read | Same as `ReadDirectoryChangesW` | Pull-based: read forward from a `StartUsn` cursor at your own pace |
| Persistence across restarts | None | None | None | Yes — durable, append-only log; resume from last-read USN |
| Deleted-item detail | N/A (no detail of any kind) | Name only — attributes/size unrecoverable by the time the record arrives | Attributes + size attached directly to removal records — closes the gap | Full record per change, including deletions |
| Overflow/drop signaling | N/A | Silent: success + 0 bytes returned; must full-rescan | Same silent-zero-byte pattern as `ReadDirectoryChangesW` | No live overflow — but records can roll off if a slow consumer doesn't keep up with `maxsize`/`allocationdelta` sizing (inferred, not explicitly stated in the sources gathered) |
| Per-file granularity | No — directory/subtree only | No — directory/subtree only | No — directory/subtree only | No per-file *watch*, but every record does carry the specific filename that changed |
| Admin/privilege requirement | None documented | None documented | None documented | **Yes — confirmed**: "you must have system administrator privileges... a member of the Administrators group" for all change-journal operations |
| Network/SMB/CsvFS support | SMB 3.0 (+TFO, +SO) and ReFS supported (Win8/Server2012+); CsvFS qualified (pause/resume false positives) | Same support table as `FindFirstChangeNotification` | **NTFS-only** per its own docs — no SMB/CsvFS/ReFS table given at all | SMB 3.0/TFO/SO **not** supported; CsvFS supported for query, "see comment" for read; filesystem scope itself disputed between sources (NTFS-only vs. "ReFS or NTFS 3.0+" — see [[usn-journal]] Limitations) |
| App model | Desktop apps only | Desktop **and UWP** | Desktop apps only | Desktop apps only (implied by control-code requirements table) |
| Minimum OS | Windows XP / Server 2003 | Windows XP / Server 2003 | Windows 10 1709 / Server 2019 | Windows XP / Server 2003 |

## Takeaways / current recommendation

- **For "I need to know what changed, in one directory tree, right now"**:
  use [[readdirectorychangesw]]. It's the widest-supported detailed API
  (XP+, desktop and UWP, SMB/CsvFS/ReFS), and unless the specific
  deleted-item gap below matters to you, it remains the default choice —
  this hasn't changed even after `ReadDirectoryChangesExW` shipped.
- **Use [[readdirectorychangesexw]] specifically when you need attributes
  or size on deletion events** — e.g. distinguishing a deleted file from a
  deleted directory, or knowing a deleted file's size, without racing a
  `FindFirstFile`/`FindNextFile` cache against fast create+delete
  sequences. Otherwise its narrower support (NTFS-only per its docs,
  desktop-only, Windows 10 1709+/Server 2019+ minimum) makes it a
  targeted upgrade, not a blanket replacement — pick it only when that one
  gap actually bites.
- **`FindFirstChangeNotification` is legacy-only today.** It offers
  nothing `ReadDirectoryChangesW` doesn't already cover, at strictly worse
  granularity (no detail at all). The wiki found no scenario where it's
  the better choice over `ReadDirectoryChangesW` for new code; it's listed
  here mainly for historical completeness and because existing code still
  uses it.
- **Reach for [[usn-journal]] only when the requirement is genuinely
  volume-wide and/or must survive process/app restarts** — e.g. a search
  indexer or backup/replication tool that needs to catch up on everything
  that happened while it wasn't running. It is **not** a lightweight
  alternative to the live APIs: it requires administrator privileges to
  read (now confirmed, see [[usn-journal]]), disabling it is slow and
  disruptive to other consumers sharing the volume, and consuming it
  correctly means hand-rolling USN-cursor tracking and buffer-walking
  logic that `ReadDirectoryChangesW`/`ExW` handle for you.
- **Two open ambiguities carried over from the concept pages, not resolved
  here**:
  1. Whether `ReadDirectoryChangesExW`'s "NTFS-only" claim means literally
     no network support, or is just an undocumented gap relative to
     `ReadDirectoryChangesW`'s explicit SMB/CsvFS/ReFS table.
  2. Whether the USN journal supports ReFS — one source says NTFS-only,
     another says "ReFS or NTFS 3.0 or later." See [[usn-journal]]
     Limitations/Platform notes for both, flagged rather than guessed at.

## Related concepts

- [[findfirstchangenotification]]
- [[readdirectorychangesw]]
- [[readdirectorychangesexw]]
- [[usn-journal]]
- [[recursive-watching]] — cross-platform version of the tree-watching axis
- [[kqueue-vs-fsevents]] — the equivalent history/comparison exercise for
  Darwin's two native mechanisms
