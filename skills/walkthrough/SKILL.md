---
name: walkthrough
description: Walk the user through a git diff or commit as a single top-to-bottom narrative, ordered by dependency/causal flow rather than alphabetically. Use when user asks to walk through, review, explain, or be guided through a diff, commit, branch, or PR — especially AI-generated changes that feel too large to review file-by-file.
disable-model-invocation: true
---

# Walkthrough

Produce one continuous, readable document that walks the user through a change in a logical order — not the alphabetical order their editor shows. The user reads it top to bottom and builds understanding as they go.

## Resolve scope

From the argument:

- **No argument** → `git diff HEAD` (working tree vs HEAD — staged + unstaged combined; matches the VS Code Git tab).
- **A commit ref** (`HEAD`, `HEAD~2`, `abc123`, `main`, etc.) → `git show <ref>`.
- **A range** (`a..b` or `a...b`) → `git diff <range>`.
- **`branch`** → current branch vs its merge-base with `main` (fall back to `master` if `main` doesn't exist): `git diff $(git merge-base HEAD main)...HEAD`.

If the argument is ambiguous (could be a branch name or a file path, etc.), ask before running.

## Gather

1. `git diff --stat <scope>` — file list and line counts.
2. `git diff <scope>` — full content. For commits use `git show <ref>` instead.
3. For a commit or range, also pull the message(s): `git log --format='%s%n%n%b' <range-or-ref>`. The author's intent is a strong signal even when the code is opaque.

## Decide reading order

Default heuristic — bottom-up by dependency:

1. Schema / migrations / types / data models
2. Pure utilities and shared helpers
3. Core business logic (services, domain modules)
4. Glue / handlers / controllers / API routes
5. UI / views / templates / styles
6. Tests
7. Config, fixtures, docs, snapshots, lockfiles

Deviate when one file is clearly the entry point of the change (e.g. a single new feature module everything else supports) — lead with it and treat the rest as supporting context. State the chosen order and the reason up front so the user can follow your logic.

## Write the walkthrough

Output everything inline in chat as one continuous document — no pagination, no "say next to continue".

## Linking to lines

Whenever you call out specific code — a function, a branch, a risky line — link to it with a line-anchored markdown link, not just a bare file path. The reader should be able to click and land on the exact line.

Format:

- Single line: `[validateInput](src/foo.ts#L42)` or `[src/foo.ts:42](src/foo.ts#L42)`
- Range: `[the new error branch](src/foo.ts#L42-L58)` or `[src/foo.ts:42-58](src/foo.ts#L42-L58)`

Pick line numbers from the **post-change** side of the diff — the `+A` in each `@@ -X,Y +A,B @@` hunk header, then count down. Those numbers match what the user sees in their editor at HEAD (or at the commit ref, for `git show`).

When you're walking through a commit ref rather than the working tree, line numbers refer to the file at that commit. Mention this once up front if relevant.

For deletions, there's no "after" line to link to — describe the removal in prose and link the surrounding context instead. Don't fabricate anchors.

Use prose link text ("the new error branch", "`validateInput`") rather than raw `path:42` whenever it reads better. Save raw `path:line` form for when you genuinely just want to point at a location.

Structure:

### 1. Overview (2–3 sentences)

What this change does and why, inferred from the code, the commit message, and any new tests. Mention the rough size (e.g. "12 files, ~300 lines added across X, Y, Z").

### 2. Reading order

A short numbered list naming the files in the order you'll cover them, with a brief reason for the grouping.

### 3. File-by-file

One section per file (or per group, for trivial files — see below). Each section:

- Heading: clickable path, e.g. `[src/foo.ts](src/foo.ts)`, plus a one-line role ("new domain entity for X", "wires the new entity into the API layer", etc.). Keep the heading link file-level — line-anchored links go inline in the body where you reference specific code.
- **What changed** — a summary, not a re-paste of the diff.
- **Why** — inferred intent; explicitly flag when you're uncertain.
- **Scrutinize** — anything non-obvious, risky, or worth a second look. Omit this bullet when there's nothing to flag.

### 4. Things to scrutinize

A final section flagging anything suspicious across the whole diff: dead code, hallucinated APIs, missing error handling, missing test coverage, scope creep, accidental reverts, leftover TODOs, debug logging, secrets, formatting-only churn mixed with logic changes.

If nothing is suspicious, say so in one sentence and stop.

## Length and tone

- One paragraph per file for typical changes. Expand to several for files with non-trivial logic. Collapse trivial files (lockfiles, snapshots, mass renames, generated code) into a single grouped paragraph at the end of the file-by-file section.
- The user already has the diff — don't paste large hunks back at them. Quote specific lines only when calling something out, and prefer a line-anchored link over a quoted block when the link alone is enough.
- Plain prose with markdown structure. No checklists unless the user asks for review-style output.
