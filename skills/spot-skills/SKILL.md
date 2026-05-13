---
name: spot-skills
description: Sweep the current conversation for skill, hook, or memory opportunities and file them as ready-to-act candidates. Use when the user runs /spot-skills to identify whether anything in the just-completed work should be captured as a skill, settings.json hook, or persistent memory entry.
disable-model-invocation: true
---

# spot-skills

Manual sweep of the current conversation for opportunities the user might want to capture as a skill, hook, or memory entry. Detection only — never authors skills, never modifies hooks, never lists past candidates. The user acts on results in fresh sessions.

## What this skill is not

- Not a skill author. When a candidate is a skill, the output is a copy-paste prompt the user runs in a fresh session via `/grill-me /write-a-skill`.
- Not a hook editor. Hook candidates produce a copy-paste prompt for `/update-config`.
- Not a cross-session scanner. Analyzes only the current conversation plus any CLAUDE.md files in scope.
- Not a candidate browser. To review what's been filed, the user opens `~/.claude/skill-candidates/` directly.

## Workflow

1. **Read in-scope CLAUDE.md files.** Same locations the `claude-md-improver` skill uses: `./CLAUDE.md`, `./.claude.local.md`, `~/.claude/CLAUDE.md`, monorepo `./packages/*/CLAUDE.md`, and any nested `CLAUDE.md` files surfaced in this session.
2. **Scan the current conversation** for procedural patterns (S2) and explicit "always X" instructions (S5).
3. **Classify** each detection as `skill`, `hook`, or `memory` per the rubric below.
4. **Dedup against existing skills.** Glob `~/.claude/skills/*/SKILL.md` and `~/.claude/plugins/marketplaces/*/plugins/*/skills/*/SKILL.md`. Read each frontmatter `description:` and check for domain overlap. If overlap exists, still file the candidate but record the existing skill in the `Existing coverage` field with a one-line note on whether the new candidate is sufficiently distinct.
5. **Dedup against existing candidate files.** Glob `~/.claude/skill-candidates/*.md`. If a file exists for the same topic (handle or evidence overlap), skip — don't re-file.
6. **File the candidate.**
   - `skill` and `hook` → write `~/.claude/skill-candidates/<handle>.md` using the structure below. Create the directory if missing.
   - `memory` → save directly using the auto-memory system documented in the system prompt (frontmatter + a pointer line in `MEMORY.md`). Do not write to `skill-candidates/`.
7. **Emit the inline summary** using the template below.

## Detection criteria

Apply these thresholds strictly. False positives erode trust faster than false negatives — if nothing meets the bar, say so.

**S1 — CLAUDE.md migration candidate.** A section qualifies only if all three hold:
- Roughly 30 lines or more.
- One coherent topic (e.g., a deployment procedure, a domain glossary), not a grab bag.
- *Conditionally* relevant — only useful for some task types. Universal rules ("always write tests", project-wide style) belong in CLAUDE.md and don't count.

**S2 — Multi-step procedure executed in this session.** Qualifies only if:
- The procedure has 3 or more distinct steps.
- It's plausibly recurring — the kind of thing the user would do again. One-off bug investigations don't count.

**S5 — Explicit "always do X" instruction.** Flag every instance where the user said "always", "from now on", "remember to", "every time", "before/after X", or similar durable instruction. High signal, low false-positive rate.

## Classification rubric

For each detection, pick the closest match:

- **hook** — "Always do X automatically when Y happens" (tool event, file pattern). Examples: format on save, block edits to `.env`, run tests after Edit.
- **memory** — "Always prefer X over Y" / personal preferences / role context / codebase conventions. Things Claude should know about *the user* or *the project*, not actions to perform.
- **skill** — A repeatable multi-step procedure with clear inputs/outputs, OR a CLAUDE.md migration candidate (S1 always classifies as skill).

When in doubt between hook and skill: a hook fires automatically on a system event; a skill is invoked when the user or Claude decides to do the task.

## Candidate file structure

Write to `~/.claude/skill-candidates/<handle>.md`. The handle is kebab-case, 3–5 words, derived from the topic. Each file contains:

````markdown
# <handle>

- **Type:** skill | hook
- **Detected:** YYYY-MM-DD
- **Trigger signal:** S1 (CLAUDE.md migration) | S2 (multi-step procedure) | S5 (explicit "always X")
- **Evidence:** <quote, file:line, or 1–3 sentence pattern description>
- **Why it could be a skill/hook:** <1–2 sentence rationale>
- **Existing coverage:** none | <skill name> at <path> — note on whether new candidate is sufficiently distinct
- **Status:** open

## Next action

Open a fresh session and paste:

```
<see "Next action prompts" below>
```
````

## Next action prompts

The "Next action" line is the prompt the user pastes verbatim into a new session. Write it with full context from the conversation — better-formed now than later.

**For skill candidates** — use third-person, end with "Use when…" per the `write-a-skill` description format. Make it pushy (skills under-trigger by default):

```
/grill-me /write-a-skill Build a skill that <crisp behaviour>. Trigger when <user phrases / contexts / file types>. Inputs: <…>. Outputs: <…>. Make sure to use this skill whenever <pushy trigger conditions>.
```

**For hook candidates:**

```
/update-config Add a <PreToolUse|PostToolUse|Stop|UserPromptSubmit> hook that <behaviour>. Matcher: <tool name or pattern>. Command: <shell command>.
```

## Inline summary template

```
Found N candidate(s) from this conversation:

1. [skill] <handle>
   File: ~/.claude/skill-candidates/<handle>.md
   Why: <one-line rationale>
   Existing coverage: <none | name>

2. [hook] <handle>
   File: ~/.claude/skill-candidates/<handle>.md
   Why: <one-line rationale>

3. [memory] <handle>
   Saved to: <path written via the auto-memory system>
   Why: <one-line rationale>

To act on a skill/hook candidate, open its file and copy the "Next action" line into a new conversation.
```

If nothing meets the thresholds, output exactly:

```
No candidates from this conversation.
```

Do not invent candidates to pad the result. An empty result is a valid result.
