# CLAUDE.md

Guidance for Claude Code working on this repository.

## Project Overview

**footage-skills** — a Claude Code skill for turning real recorded footage into finished
vertical shorts.

**Repository**: https://github.com/milojarow/footage-skills

Sibling to the operator's AI-imagery video pack: same deliverable, opposite input. This one
never generates the subject — it cuts, captions, packages and mixes a take that already exists.

## Repository Structure

```
footage-skills/
├── .claude-plugin/          marketplace.json + plugin.json
├── CLAUDE.md                this file (gitignored)
├── README.md
├── LICENSE                  MIT
├── evaluations/footage/     GREEN scenarios
└── skills/footage/
    ├── SKILL.md
    └── reference/           cut-and-transcript · captions-karaoke ·
                             overlays-and-pip · audio-levels · silent-failures
```

## The load-bearing content

**`reference/silent-failures.md` is the reason this skill exists.** Every entry came from a
run where lint passed and the render succeeded. If a future session finds another defect of
that shape — no error, wrong output — it belongs there, with the symptom, the cause and the
command that catches it.

Do not soften those entries into general advice. Their value is that they are specific enough
to recognize in the wild.

## Conventions

- Generic content only. No operator names, client names, hostnames, paths under a real home
  directory, API keys, voice IDs or transcript IDs. This repo is public; the scrub is the gate,
  not `.gitignore`.
- Numbers in the reference files are **measured**, not estimated. If a figure changes, re-measure
  and update it rather than rounding it away.
- Teach the system, not the inventory. Name techniques and thresholds; do not enumerate the
  operator's projects, voices or accounts.

## Updating

After any production run that discovers a new failure mode or overturns a documented number.
The git log is the diary.
