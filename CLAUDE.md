# notify-wiki

A personal knowledge wiki about OS/kernel file-change notification systems —
inotify, kqueue/kevent, fanotify, FSEvents, ReadDirectoryChangesW, dnotify,
io_uring-based watching, and the userspace libraries built on top of them
(libuv, watchman, chokidar, notify-rs, fsnotify, etc.).

You (the agent) own the `wiki/` layer: creating pages, updating them as new
sources arrive, maintaining cross-references, and keeping the whole thing
internally consistent. The human owns `raw/`: dropping in papers, man pages,
RFC-like design docs, blog posts, source code excerpts, and mailing list
threads, plus deciding what to investigate next.

## Directory layout

```
raw/                    Immutable primary sources, verbatim. Never edited by the agent.
  <slug>.md / .pdf / .txt / .html
wiki/
  index.md              Entry point: map of the wiki, grouped by topic
  log.md                Reverse-chronological changelog of what changed and why
  concepts/             One page per mechanism, syscall/API, or concept
  comparisons/          Cross-cutting pages comparing mechanisms/platforms
  sources/              One summary page per raw/ source, linking out to
                         the concept pages it informed
```

## Page conventions

Every page in `wiki/` starts with frontmatter:

```yaml
---
title: inotify
tags: [linux, kernel, event-driven]
updated: 2026-07-08
sources: ["../sources/lwn-inotify-2005.md"]
---
```

### concepts/*.md

Structure, in order (omit a section if there's nothing to say yet, don't pad):

1. **Overview** — one paragraph: what it is, what platform/kernel owns it, what problem it solves
2. **API / semantics** — how you use it: key syscalls/functions, event types, watch model (per-path vs per-fd vs recursive)
3. **Limitations & gotchas** — known pitfalls: watch limits, coalescing behavior, rename tracking, symlink handling, buffer overflow behavior, race conditions
4. **Platform notes** — kernel/OS version history, deprecations, replacements
5. **Related concepts** — links to other `concepts/` or `comparisons/` pages, e.g. `[[kqueue]]`
6. **Sources** — links to `sources/*.md` entries this page draws on

### comparisons/*.md

Framed around a specific axis (e.g. "recursive watching support", "coalescing behavior",
"Linux inotify vs fanotify"), with a table or side-by-side breakdown, not a repeat of
each concept page's content — link back to `concepts/` for detail instead of duplicating it.

### sources/*.md

One per file in `raw/`. Short: what the source is, what it claims, which
`concepts/` pages it fed into. This is the audit trail from wiki claims back
to primary material.

## Cross-references

Use `[[concept-name]]` wiki-links (matching the source page's filename stem,
no directory prefix) for links within `wiki/`. Keep links bidirectional in spirit —
if A links to B as "related", check whether B should link back.

## Workflows

**Ingest**: human drops a file in `raw/`. Agent reads it, writes/updates a
`sources/*.md` summary, then creates or updates the relevant `concepts/*.md`
pages, updates `wiki/index.md` if a new page was added, and appends an entry
to `wiki/log.md`.

**Query**: human asks a question. Agent answers from `wiki/` content when it's
already there; otherwise says so plainly rather than guessing, and suggests
what source would resolve it.

**Health check** (run periodically or on request): scan for contradictions
between pages, stale claims superseded by newer sources, orphan pages with no
inbound links, and concepts mentioned but not yet written up. Report findings,
don't silently fix unless asked.

## log.md format

Reverse-chronological, one entry per ingest/edit session:

```markdown
## 2026-07-08
- Added `concepts/inotify.md`, `concepts/kqueue.md` (scaffold, no sources yet)
```
