<p align="center">
  <img src="https://raw.githubusercontent.com/OthmanAdi/pr-perfect/main/media/logo.png" alt="pr-perfect" width="200">
</p>

<h1 align="center">pr-perfect</h1>

<p align="center">
  A Claude Code skill that writes PR descriptions so you don't have to pretend you enjoy it.
</p>

<p align="center">
  <code>npx skills add OthmanAdi/pr-perfect</code>
</p>

---

## Install

```bash
npx skills add OthmanAdi/pr-perfect
```

That's it. Now you have `/pr-perfect` in every project.

## The Problem

You just spent 4 hours writing code. Now you need to write a PR description. Your brain says "fixed stuff" but your boss needs a 100-line document with tables, file breakdowns, and verification checklists because his coding agent reviews PRs before he does.

So you ask Claude to write it. Claude asks you to re-explain the format. You paste examples. Claude produces something close but wrong. You manually fix 15 things. Next week, same dance.

This skill kills the dance.

## What It Does

One command. That's it.

Behind the scenes:
- Opens every commit in the diff and reads the actual changes (not just the message)
- Groups commits into coherent themes with problem-to-fix narrative arcs
- Writes the description in your team's exact format (learned from your last 3 merged PRs)
- Creates a feature branch, pushes it, opens the PR via `gh`
- Hands you a URL

The output reads like a briefing document — because that's what it is. Your boss's agent cross-references the description against the diff. Vague descriptions mean manual review. Detailed descriptions mean fast merges.

## The Format

Every PR follows this structure:

```
## Summary
### 1. Section — Problem -> Fix (with tables)
### 2. Section — Problem -> Fix
## Changes by file (every file, no exceptions)
## Commits (full list, chronological)
## Verification (falsifiable bullets)
**N files changed, +X, -Y. Safe to squash merge.**
```

Sections lead with what was broken. Bold the failure mode. Tables for structured data. File-by-file accountability so the reviewer's agent can verify. Concrete verification so someone can check each bullet against the code.

## Usage

```
/pr-perfect
```

Or just say things like:
- "create a PR"
- "ship this to main"
- "prepare the merge"
- "let's wrap this up"
- "ready to push"

The skill triggers on all of these.

## Customizing the Style

The skill ships with example PRs in `references/pr-examples.md`. Replace them with your own team's merged PRs to calibrate the output:

```bash
for n in $(gh pr list --state merged --limit 3 --json number --jq '.[].number'); do
  gh pr view $n --json title,body --jq '"## PR #'$n'\n\n**Title:** \(.title)\n\n\(.body)\n\n---"'
done > .claude/skills/pr-perfect/references/pr-examples.md
```

Now the skill writes PRs that look like yours, not like some generic template.

## Who This Is For

Engineers who:
- Work on teams where PRs get reviewed by humans AND their coding agents
- Have a boss who merges chaotically and needs the PR description as a map
- Would rather write code than write about writing code

## License

MIT. Go ship things.
