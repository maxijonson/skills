# Skills

[![skills.sh](https://skills.sh/b/maxijonson/skills)](https://skills.sh/maxijonson/skills)

This repository contains personal AI agent skills I've created. Most of the skills in this repo were created with the help of [Matt Pocock's write-a-skill](https://github.com/mattpocock/skills/blob/main/skills/productivity/write-a-skill/SKILL.md) skill.

## Installation

To install one or more skills, use the [skills CLI](https://github.com/vercel-labs/skills) and follow the instructions.

```bash
npx skills@latest add maxijonson/skills
```

## Skills

### [walkthrough](skills/walkthrough/SKILL.md)

**Description**

Analyzes and walks you through the current git diff or a specific commit/range. Then, writes a single, top-to-bottom explanation of the changes, ordered by dependency/causal flow rather than alphabetically. The walkthrough makes use of line-anchored markdown links to help you easily jump to the relevant code in your editor as you read.

**When to use**

This skill was mostly designed for current working tree diffs where a lot of code was written by AI and feels too large to review file-by-file. It makes it easier to understand how the changes fit together as a whole, rather than getting lost in the details of each file.

### [walkthrough-pr](skills/walkthrough-pr/SKILL.md)

**Description**

This skill builds on the learnings from the `walkthrough` skill, but is designed for pull request reviews (or just branch diffs in general). It will:

1. Create a worktree for the PR branch (under `.pr-review/<branch-slug>`) so that you don't need to switch contexts to review the code.
2. Exclude the worktree from your git through `.git/info/exclude` so it doesn't touch your `.gitignore`.
3. Run through a similar walkthrough process as the `walkthrough` skill, with links anchored to the worktree files.
4. Create a `WALKTHROUGH.md` file in the root of the worktree, so you can review it in your editor or share it with others.

**When to use**

This skill is designed for pull request reviews, especially when the PR is large and complex. It allows you to review the code in a more structured way, without needing to switch branches or context. The generated `WALKTHROUGH.md` can also be a helpful artifact to share with others or refer back to later.
