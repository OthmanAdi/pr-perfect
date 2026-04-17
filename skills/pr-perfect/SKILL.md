---
name: pr-perfect
description: >
  Create a GitHub PR with an engineered description that matches the team's established
  format — numbered problem-fix sections, technical tables, file-by-file breakdowns,
  commit lists, and verification checklists. Use this skill whenever the user asks to
  create a PR, ship code, prepare a merge request, write a PR description, or says
  anything like "ship to boss", "pr perfect", "make a PR", "prepare PR for merge", "create pull request",
  "engineer the PR", "PR description", or "ready to push". Also trigger when the user has
  finished a batch of work and says things like "let's wrap this up", "time to merge",
  "push this to main", or "get this ready for review". The skill handles everything:
  commit analysis, description generation, branch creation, and PR submission via gh CLI.
---

# Ship to Boss

Create PRs that serve as briefing documents — detailed enough for both humans and
coding agents to review, verify, and merge with confidence.

## Arguments

The skill accepts free-form `ARGUMENTS` at invocation. Only one flag needs special handling:

| Flag | Aliases | Effect |
|---|---|---|
| `--no-branch` | `--local-only`, `--no-pr`, `--draft-only` | **Skip branch creation, push, and `gh pr create` entirely.** Write the engineered description to `PR_DESCRIPTION.md` at the repo root and stop. Use this when the user commits directly to `main` and wants the description text only — e.g. to paste into a PR opened manually, attach to a changelog, or hand off to someone else to create the PR. |

Detection: treat the flag as present if any of its aliases appears as a whole token in `ARGUMENTS` (case-insensitive). When present, **Step 4 is replaced** by the Local-only variant below — everything else (Steps 1–3) runs identically.

## Why This Format Matters

The PR description is not just documentation. The reviewer (and their coding agent)
uses it to verify changes make sense, spot conflicts, and confirm nothing breaks.
Every table, every file listing, every verification bullet exists so the reviewer
can cross-reference the description against the actual diff. A vague PR description
forces the reviewer to reverse-engineer intent from raw code — that's slow and error-prone.

## Workflow

Execute these steps in order. Do not skip any step.

### Step 1: Forensic Commit Analysis

Study every commit that will be in the PR. This is the foundation — a lazy read here
produces a lazy description.

```bash
# What commits are we working with?
git fetch origin
git log origin/main..HEAD --oneline --reverse

# For EACH commit, read the full message and file list
git show <sha> --stat --format="%s%n%n%b"

# Total diff summary
git diff origin/main..HEAD --stat
```

For each commit, note:
- What problem it solves (not what it changes — WHY it changes)
- Which files it touches
- Whether it's a fix, feature, test, or refactor

### Step 2: Group Into Themes

Commits rarely map 1:1 to PR sections. Look for natural groupings:
- Multiple commits touching the same subsystem = one section
- A fix + its test = one section
- An independent feature = its own section

Aim for 2-5 numbered sections. Each section needs a clear narrative arc:
problem existed → here's what was broken → here's the fix → here's why it's safe.

### Step 3: Write the PR Description

Read `references/pr-examples.md` for the exact style from the last merged PRs. Then
write the description following this template structure:

#### Title Format

```
feat(scope): short theme 1, short theme 2, short theme 3
```

Keep under 70 characters. Use the primary scope (e.g., `ai`, `telemetry`, `test`).
Comma-separate if multiple themes. Use conventional commit prefix (`feat`, `fix`, `test`, `refactor`).

#### Body Structure

```markdown
## Summary

One paragraph (2-3 sentences) stating what this PR does at the highest level.
Lead with the impact, not the implementation. Mention how many themes/areas are covered.

### 1. Section Title — Descriptive Subtitle

Start with the PROBLEM. What was broken, missing, or wrong? Use **bold** to highlight
the failure mode or gap. Be specific — "the web-search toggle silently vanished" not
"there was a bug in web search."

**Fix:** Describe the solution. If it involves a new pattern or contract, use a table:

| Column1 | Column2 | Column3 |
|---|---|---|
| data | data | data |

### 2. Next Section Title

Same pattern: problem → fix → evidence it's safe.

(repeat for each theme)

## Changes by file

| Layer | File | Changes |
|---|---|---|
| Backend | `file.rs` | +N: brief description of what changed |
| Frontend | `Component.tsx` | +N/−M: brief description |

Every file in the diff MUST appear in this table. Group by layer (Backend, Frontend,
Agents, Config, Tests). Use the `+N` / `+N/−M` format to show line counts.

## Commits

```
sha1 type(scope): commit message 1
sha2 type(scope): commit message 2
...
```

List ALL commits in chronological order (oldest first). Use short SHA (7 chars).

## Verification

Bullet list of how to verify this works. Be specific:
- "All 11 previously failing tests now pass" (not "tests pass")
- "New config fields backward compatible (`#[serde(default)]`)" (not "backward compatible")
- "Zero changes to SSE contract" (not "nothing else changed")

**N files changed, +X, −Y. Safe to squash merge.**
```

### Step 4: Create Branch and PR (default)

Use this variant unless the user passed `--no-branch` (see Arguments section).

```bash
# Create feature branch from current HEAD
git checkout -b feature/<descriptive-name> HEAD

# Push to remote
git push -u origin feature/<descriptive-name>

# Write body to temp file first — avoids all shell quoting issues
# (single quotes, backticks, dollar signs in body content break heredoc-in-substitution)
cat > /tmp/pr_body.md <<'PREOF'
<body>
PREOF

# Create PR using body file — gh reads it directly, no shell parsing of content
gh pr create --base main --title "<title>" --body-file /tmp/pr_body.md
rm -f /tmp/pr_body.md
```

Branch naming: `feature/<primary-theme>` using kebab-case. Keep it short but descriptive.

After PR creation, switch back to main:
```bash
git checkout main
```

Report the PR URL to the user.

### Step 4 (Local-only variant): `--no-branch`

Runs when `ARGUMENTS` contains `--no-branch`, `--local-only`, `--no-pr`, or `--draft-only`.
**Do NOT create a branch. Do NOT push. Do NOT call `gh pr create`.** The user is
working on `main` directly and only wants the engineered description on disk.

```bash
# Write the body to PR_DESCRIPTION.md at the repo root. Overwrites any prior
# draft. The title goes on the first line as an H1 so the file stands on its
# own if pasted into GitHub's web UI.
cat > PR_DESCRIPTION.md <<'PREOF'
# <title>

<body>
PREOF
```

Then tell the user exactly three things:
1. The full path of `PR_DESCRIPTION.md`.
2. That no branch was created, no push happened, and no PR was opened.
3. How to turn it into a PR later if they want — typically:
   `gh pr create --base main --head <their-branch> --title "<title>" --body-file PR_DESCRIPTION.md`

Do NOT commit `PR_DESCRIPTION.md`. It is a scratch artifact for the user — add it to
`.gitignore` only if the user asks. Leave `git status` dirty so they see it.

## Style Rules

These rules are non-negotiable — they're what makes this team's PRs consistent.

1. **German-enterprise tone**: Technical, precise, no fluff. Tables over paragraphs when
   data is structured. Numbers over adjectives ("11 failures" not "several issues").

2. **Problem-first sections**: Every section starts with what was wrong. The reader
   should understand the pain before seeing the fix.

3. **Bold failure modes**: Use `**bold**` for the specific thing that broke or was missing.
   "The toggle **silently vanished**" — the bold text is what the reviewer's eye catches.

4. **Pipe tables for structured data**: Methods, tools, safety systems, file changes —
   anything with 2+ columns gets a markdown table.

5. **File-by-file accountability**: The "Changes by file" table is mandatory. Every file
   in the diff must be listed. This is what the reviewer's agent uses to cross-check.

6. **Concrete verification**: Each bullet in Verification must be falsifiable. The reviewer
   should be able to check each one by running a command or reading the diff.

7. **Footer**: Always end with the line count summary and "Safe to squash merge." (or
   explain why not, if applicable).

8. **No Co-Authored-By**: Never add co-author lines to commits or PR descriptions.

## Edge Cases

- **Single commit PR**: Still use the full template. One numbered section is fine.
- **Only test changes**: Lead with what was broken in the test suite, not "added tests."
- **Mixed fix + feature**: Feature sections before fix sections (features are the headline).
- **Large PRs (20+ files)**: Group the file table by subsystem, add a "Total" row.
