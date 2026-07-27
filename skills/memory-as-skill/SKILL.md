---
name: memory-as-skill
description: "User-controlled persistent memory system using plain Markdown files that the user can directly read, edit, and scope. Replaces reliance on opaque background memory with inspectable, editable files the user owns. Use this skill at the START of every session — read the memory files BEFORE doing anything else. Use at the END of any session with meaningful progress to update the relevant file. Trigger whenever the user references a project by name, says 'where we left off,' 'load memory,' 'what are we working on,' or starts any session that isn't a one-off question. Also trigger when a project wraps up, to archive it."
---

# Memory As A Skill

## Why this exists

The built-in memory system is opaque — the user can't see what's stored, can't edit it, can't scope it, and can't stop old irrelevant context from bleeding into new work. This skill adds a user-controlled layer on top: plain Markdown files the user owns, reads, and edits directly.

**This is your first stop for context, not your only stop.** If the memory files cover what you need, use them and don't waste tokens re-caching the thread or pulling background recall. If the files don't cover something and the task requires it, go ahead and search chats, use background memory, pull connectors — whatever's needed. The point is efficiency: load tight, scoped context first so you're not doing redundant work most sessions.

## Structure

```
memory/
├── general.md           — builder profile, cross-project rules, standing preferences
├── active/
│   ├── <project>.md     — one file per in-flight project
│   └── <subtopic>.md    — optional deep-dive files within a project scope
└── archive/
    └── <project>.md     — compacted finished projects (DO NOT LOAD unless asked)
```

Files and filenames will change constantly. New projects spin up, old ones archive out, subtopic files appear when a specific area (compiling, hardware, a subsystem) needs its own isolated context. Don't expect a fixed set — just follow the pattern.

## Session start (mandatory)

1. Read `memory/general.md`. It's small, always relevant.
2. Identify which project the user is working on. Read ONLY `memory/active/<that-project>.md`. Do not read other active files — token waste, context bleed.
3. If a relevant subtopic file exists (user mentions it or the task clearly falls within its scope), read that too.
4. **DO NOT read `archive/` at session start.** Archived projects are done. They don't apply to current work unless the user explicitly asks to reference one.

## Stop blocks

The user can place stop blocks anywhere in a memory file:

```
<!-- STOP: Do not reference anything below this line unless explicitly asked -->
```

If a stop block exists, the model reads only above it. Content below the stop block is historical context the user has chosen to freeze — it stays in the file for the user's own reference but is not active context for the model.

The user can also mark entire files or sections:

```
<!-- SCOPE: compiling only -->
<!-- SCOPE: do not carry into other projects -->
```

Respect these. They exist specifically to prevent the problem where old context from a different workstream gets dragged into unrelated work.

## During the session

Apply loaded context silently. No "according to my memory file," no "I see from our previous sessions." Just know it, like a colleague who was there.

## Session end / meaningful checkpoint

Update `active/<project>.md` with:
- Decisions made (not a blow-by-blow transcript)
- Current state / blockers
- Next concrete step

**What NOT to log:**
- Failure recaps, apologies, or "what went wrong" lists. A failure earns a memory entry ONLY if it converts to a reusable rule (e.g. "curl raw GitHub URLs directly — API hits rate limits without auth"). Bare underperformance notes are noise.
- Redundant context that's already in the file. Don't re-state things that haven't changed.

**Size discipline:** Keep each active file under ~150 lines. If it's growing, compact older entries into a terse "Background" section at the top. Only the recent/live state needs granular detail.

## Subtopic files

When a specific area within a project gets deep enough to warrant isolation (its own dependencies, its own state, would clutter the main project file), create `active/<subtopic>.md`. Examples: a complex build pipeline, a hardware subsystem, a specific integration. The main project file should note the subtopic file exists but doesn't need to duplicate its content.

## Archiving (fold-over)

When a project is done (shipped, repo closed, user says "wrap it up"):

1. Compact `active/<project>.md` into a dense summary — key decisions, final architecture, outcomes. Strip transient detail.
2. Merge anything cross-project-relevant into `memory/general.md` (reusable patterns, preferences, lessons that apply going forward).
3. Move compacted file to `memory/archive/<project>.md`.
4. Remove the `active/` version.
5. Export a copy to the user automatically — they keep their own archive independent of this skill.

## Naming convention for project skills

Project-scoped wiki skills use the `<project>-wiki` pattern:

| Skill name | Scope |
|------------|-------|
| `horizons-wiki` | Horizons Android app — UI, NPU routing, backend architecture |
| `aesop-wiki` | Aesop deploy stack — Termux daemons, bridge protocol, on-device AI |
| `termux-helper` | Cross-project utility — Termux operations, paths, packages |
| `memory-as-skill` | Cross-project utility — session memory system |

Project wikis (`*-wiki`) carry project-specific architecture, decisions, and current state. Utility skills (`termux-helper`, `defuddle`, etc.) carry reusable knowledge that applies across projects. Keep them separate — a project wiki should never duplicate utility skill content, just reference it.

## Updating this skill

If the user wants to change how memory works — structure, rules, scoping — edit this SKILL.md directly. The whole point is that everything is inspectable files, not opaque behavior.
