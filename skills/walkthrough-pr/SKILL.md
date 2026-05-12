---
name: walkthrough-pr
description: Walk the user through a PR as a single top-to-bottom narrative ordered by dependency/causal flow, with a heavy scrutiny pass for bugs, missing tests, scope creep, and security. Operates on a temp git worktree so it works while the main working tree is dirty. Use when the user asks to review a PR, walk through a PR, review a PR, or review one branch against another.
disable-model-invocation: true
---

# walkthrough-pr

Produce one continuous, readable PR review the user can read top-to-bottom. Same shape as `walkthrough`, plus a much heavier scrutiny pass. The user can have uncommitted changes in their main working tree — this skill never touches it.

## Resolve scope

Arguments: `<pr-branch> [dest-branch]`. `pr-branch` is required.

Resolve `pr-branch`:

1. Try `git rev-parse --verify <pr-branch>`.
2. If that fails, run `git fetch` once, then retry as `<pr-branch>` and `origin/<pr-branch>`. Use whichever resolves.
3. If still unresolved, stop and ask the user.

Resolve `dest-branch`:

- If passed explicitly, use it (resolve same way as `pr-branch`).
- Otherwise, run `git symbolic-ref refs/remotes/origin/HEAD` (yields `refs/remotes/origin/<name>`).
- If that fails (origin/HEAD not set), stop and ask the user. **Do not** guess `main` / `master` / `develop` — better to ask than review against the wrong base.

Capture the PR head SHA via `git rev-parse <pr-branch>` for use in output and verification.

## Set up the worktree

The review reads files from a detached worktree at `<repo-root>/.pr-review/<branch-slug>/`. The main working tree is never touched.

`<branch-slug>` is the PR branch name with characters illegal in folder names (notably `/`, plus `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|`) replaced with `-`. Examples: `feature/login-fix` → `feature-login-fix`; `bugfix/JIRA-123` → `bugfix-JIRA-123`. Strip any leading `origin/` first if the user passed a remote-tracking ref.

1. **First-run hygiene**: ensure `.pr-review/` is in `.git/info/exclude`. Read the file; if the line `.pr-review/` is missing, append it. (`.git/info/exclude` is per-repo and not committed, so this doesn't churn the team's `.gitignore`.)
2. **Clear the same-branch worktree if present**: if `<repo-root>/.pr-review/<branch-slug>/` already exists, run `git worktree remove --force .pr-review/<branch-slug>` (check with `git worktree list` first; ignore errors if absent). Other branches' worktrees are left untouched.
3. **Create the worktree**: `git worktree add --detach .pr-review/<branch-slug> <pr-branch-ref>`. Detached HEAD avoids "already checked out" errors when the PR branch happens to be checked out elsewhere.

The worktree **stays alive after the skill finishes** so clickable links in the review remain navigable. Re-running `/walkthrough-pr` for the same branch overwrites it; different branches accumulate side-by-side under `.pr-review/`. The user can list them with `git worktree list` and prune stale ones with `git worktree remove --force .pr-review/<branch-slug>`.

## Gather

All git commands run from the main repo (refs work without entering the worktree):

1. `git diff --stat <dest-branch>...<pr-branch>` — file list and line counts.
2. `git diff <dest-branch>...<pr-branch>` — full diff. **Three-dot** range — compares PR head against the merge-base with dest, i.e. only what this PR introduces. Don't use two-dot.
3. `git log <dest-branch>..<pr-branch> --format='%s%n%n%b'` — PR commit messages (two-dot here for the linear commit list). Author intent is a strong signal even when the code is opaque.

When you need surrounding context that isn't in the diff (e.g. to see how a touched function is called elsewhere), `Read` files via absolute paths under `<repo-root>/.pr-review/<branch-slug>/...`. Don't `Read` from the main tree — that's the user's working state, not the PR's.

## Decide reading order

Same heuristic as walkthrough — bottom-up by dependency:

1. Schema / migrations / types / data models
2. Pure utilities and shared helpers
3. Core business logic (services, domain modules)
4. Glue / handlers / controllers / API routes
5. UI / views / templates / styles
6. Tests
7. Config, fixtures, docs, snapshots, lockfiles

Deviate when one file is clearly the entry point of the change — lead with it and treat the rest as supporting context. State the chosen order and reason up front.

## Linking to lines

Whenever you call out specific code, link to it with a line-anchored markdown link pointing **into the worktree**, not the main repo. Clicking opens the actual PR-state file.

Format:

- Single line: `[validateInput](.pr-review/<branch-slug>/src/foo.ts#L42)` or `[src/foo.ts:42](.pr-review/<branch-slug>/src/foo.ts#L42)`
- Range: `[the new error branch](.pr-review/<branch-slug>/src/foo.ts#L42-L58)`

Pick line numbers from the **post-change** side of the diff — the `+A` in each `@@ -X,Y +A,B @@` hunk header, then count down. Those numbers match the file at the PR head, which is what the worktree contains.

For deletions, there's no "after" line to link — describe the removal in prose and link surrounding context. Don't fabricate anchors.

Use prose link text ("the new error branch", "`validateInput`") rather than raw `path:42` whenever it reads better.

## Write the review

Compose the review as one continuous markdown document, then save and display it:

1. **Save** the full content to `<repo-root>/.pr-review/<branch-slug>/WALKTHROUGH.md` using the `Write` tool. This is the shareable artifact — the user can copy/paste the file into a Bitbucket comment, Slack, etc.
2. **Display** the exact same content inline in chat — no pagination, no "say next to continue". The chat output and the file content must be identical (single source of truth).

State up front in the document: "Links open files in `.pr-review/<branch-slug>/`, which holds the PR branch state. The worktree is overwritten on the next `/walkthrough-pr` run for this same branch." Substitute the real branch slug in the actual output — readers should see a concrete path.

### 1. Overview (2–3 sentences)

What this PR does and why, inferred from the code, commit messages, and any new tests. Mention the PR branch, the dest branch, the head SHA, and rough size (e.g. "9 files, ~240 lines added across X, Y, Z, against `main` (PR head `abc1234`)").

### 2. Reading order

Short numbered list naming the files in the order you'll cover them, with a brief reason for the grouping.

### 3. File-by-file

One section per file (or per group, for trivial files). Each section:

- Heading: clickable worktree path, e.g. `[.pr-review/<branch-slug>/src/foo.ts](.pr-review/<branch-slug>/src/foo.ts)`, plus a one-line role ("new domain entity for X", "wires the new entity into the API layer", etc.). Keep the heading link file-level — line-anchored links go inline in the body.
- **What changed** — a summary, not a re-paste of the diff.
- **Why** — inferred intent; flag uncertainty.
- **Scrutinize** — anything non-obvious, risky, or worth a second look. Omit when there's nothing to flag.

### 4. Things to scrutinize (heavy pass)

A final section with cross-file concerns. Walk through each category below explicitly — even briefly stating "nothing here" — so nothing gets quietly skipped:

- **Logic bugs & off-by-ones** — boundary conditions, inverted comparisons, wrong loop variables.
- **Tests** — missing coverage for new branches; tests that assert nothing meaningful (e.g. only checking that something didn't throw); deleted tests; snapshot churn that hides logic changes.
- **Error handling** — swallowed exceptions, generic catches, missing fallbacks at boundaries (network, FS, deserialization), error states that silently succeed.
- **Edge cases** — empty inputs, nulls/undefined, zero-length collections, concurrency / race conditions, retry behavior, idempotency.
- **Security** — secrets in code or config, injection (SQL/shell/HTML), auth bypass, missing authz checks, unsafe deserialization, SSRF, prompt injection in LLM contexts.
- **Scope creep** — drive-by changes unrelated to the PR's stated intent; refactors mixed with feature work; formatting churn obscuring real changes.
- **Accidental reverts** — code that re-introduces something previously removed; stale code paths revived.
- **Dependency changes** — new packages, version bumps (especially major), unexpected lockfile churn, transitive risk.
- **API / contract breaks** — renamed/removed exports, changed function signatures, response shape changes — and whether call sites elsewhere in the repo were updated.
- **Performance** — N+1 queries, unbounded loops, sync I/O on hot paths, unneeded re-renders, large allocations in tight loops.
- **Hallucinated APIs** — calls to functions/methods that don't exist on the imported module (especially common in AI-generated PRs). Verify by `Read`ing the imported module from the worktree when in doubt.
- **Leftovers** — TODOs, debug logging, `console.log`, commented-out code, dead branches.

If a category genuinely has nothing, say so in one phrase ("Tests: comprehensive, asserts the new branch."). If everything is clean, end with one sentence saying so.

## Length and tone

- One paragraph per file for typical changes. Expand to several for non-trivial logic. Collapse trivial files (lockfiles, snapshots, mass renames, generated code) into a single grouped paragraph.
- The user already has the diff — don't paste large hunks back. Quote specific lines only when calling something out, and prefer a line-anchored link over a quoted block when the link alone is enough.
- Plain prose with markdown structure.

## Footer

End the review with a one-line cleanup hint:

> Worktree at `.pr-review/<branch-slug>/` (PR head `<sha>`). Saved review: `.pr-review/<branch-slug>/WALKTHROUGH.md` — share this file as-is. To remove the worktree now: `git worktree remove --force .pr-review/<branch-slug>`. Other branches' worktrees under `.pr-review/` are untouched — list them with `git worktree list`.

Substitute the real branch slug and SHA in the actual output.
