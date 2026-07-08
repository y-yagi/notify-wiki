---
title: "Source: Apple Developer Docs — kFSEventStreamCreateFlagFileEvents availability"
tags: [macos, darwin, source]
updated: 2026-07-09
---

# apple-dev-fseventstreamcreateflagfileevents

**Raw file**: [`raw/apple-dev-fseventstreamcreateflagfileevents.txt`](../../raw/apple-dev-fseventstreamcreateflagfileevents.txt)
**Origin**: https://developer.apple.com/documentation/coreservices/1455376-fseventstreamcreateflags/kfseventstreamcreateflagfileevents
— pasted by the user 2026-07-09 (the live page is a JS-rendered SPA and
could not be captured by automated fetch, same limitation already noted for
[apple-fsevents-progguide](apple-fsevents-progguide.md) and
[apple-fsevents-h](apple-fsevents-h.md)).

## What it is

Apple's official reference page content for the
`kFSEventStreamCreateFlagFileEvents` symbol: platform-availability badges,
the enum value, and the Discussion (description) text.

## What it claims

- Available on **Mac Catalyst 13.1+** and **macOS 10.7+**.
- Confirms the macOS 10.7 attribution that [apple-fsevents-h](apple-fsevents-h.md)
  flagged as resting on outside knowledge rather than the header text itself
  — now backed by this source.
- Adds Mac Catalyst 13.1+ as a previously-undocumented availability detail in
  this wiki.
- Discussion text is verbatim-identical to the doc comment already captured
  in [apple-fsevents-h](apple-fsevents-h.md)'s raw header file ("Request
  file-level notifications... Use this flag with care as it will generate
  significantly more events than without it.") — cross-confirms the header
  mirror is accurate and current, rather than adding new descriptive content.

## Concept pages informed

- [`concepts/fsevents.md`](../concepts/fsevents.md) — resolved the
  "unconfirmed OS version" caveat on `kFSEventStreamCreateFlagFileEvents`'s
  Platform notes entry; added Mac Catalyst availability.
