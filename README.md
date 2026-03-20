# ship-to-boss

A Claude Code skill that writes PR descriptions so you don't have to pretend you enjoy it.

## The Problem

You just spent 4 hours writing code. Now you need to write a PR description. Your brain says "fixed stuff" but your boss needs a 100-line document with tables, file breakdowns, and verification checklists because his coding agent reviews PRs before he does.

So you ask Claude to write it. Claude asks you to re-explain the format. You paste examples. Claude produces something close but wrong. You manually fix 15 things. Next week, same dance.

This skill kills the dance.

## What It Does

One command. `/ship-to-boss`. That's it.

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
### 1. Section — Problem → Fix (with tables)
### 2. Section — Problem → Fix
## Changes by file (every file, no exceptions)
## Commits (full list, chronological)
## Verification (falsifiable bullets)
**N files changed, +X, -Y. Safe to squash merge.**
```

Sections lead with what was broken. Bold the failure mode. Tables for structured data. File-by-file accountability so the reviewer's agent can verify. Concrete verification so someone can check each bullet against the code.

## Installation

Drop the skill into your project:

```
your-project/
  .claude/
    skills/
      ship-to-boss/
        SKILL.md
        references/
          pr-examples.md    <-- your team's actual merged PRs
```

The `references/pr-examples.md` file ships with 3 example PRs. Replace them with your own team's merged PRs to calibrate the style. The skill reads this file every time it runs.

### Updating the Style Reference

When your team's PR format evolves, update the examples:

```bash
# Pull latest 3 merged PRs into the reference file
for n in $(gh pr list --state merged --limit 3 --json number --jq '.[].number'); do
  gh pr view $n --json title,body --jq '"## PR #'$n'\n\n**Title:** \(.title)\n\n\(.body)\n\n---"'
done > .claude/skills/ship-to-boss/references/pr-examples.md
```

## Usage

```
/ship-to-boss
```

Or just say things like:
- "create a PR"
- "ship this to main"
- "prepare the merge"
- "let's wrap this up"
- "ready to push"

The skill triggers on all of these.

## Who This Is For

Engineers who:
- Work on teams where PRs get reviewed by humans AND their coding agents
- Have a boss who merges monolithically and needs the PR description as a map
- Would rather write code than write about writing code

## License

MIT. Go ship things.
