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
