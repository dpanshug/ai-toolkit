---
name: pr-description-write
description: "Draft a review-ready pull request or merge request description from the branch diff. Repo-agnostic: detects the repo's PR template and maps substance into it, or falls back to a minimal Why / What changed / Verification skeleton. Use when creating a PR or MR, writing or improving a PR body, or when the user says 'write the PR description' or 'open a PR'."
---

# PR Description Write

Draft PR/MR bodies that give a reviewer — who has seen neither the branch nor the ticket — the full picture: what was broken, what changed and why, and how to verify it.

## Core rules

- **Substance is universal, structure is local.** Every body must carry Why / What-and-how / Proof. Structure comes from the repo's own template when one exists — never impose this skill's skeleton over an existing template.
- **Never fabricate evidence.** Cite only commands that were actually run with observed results. If no evidence exists, ask the user or offer to run lint/tests first. A missing Evidence section fails review; a fabricated one is worse.
- **No placeholders.** No TBD, no empty headings, present tense, current with the branch head.

## Procedure

### 1. Gather

- Diff and commits: `git diff $(git merge-base <base> HEAD)` (+ `--stat` for shape); `git log --oneline <base>..HEAD`
- Linked tracker: branch name, commit messages, or the user
- Evidence: commands run this session with results (lint, type-check, tests, builds, cluster checks, screenshots)

### 2. Detect repo conventions

Search in this order (first hit wins). The template is authoritative — never draft until this search has run:

1. **PR template** — search the working tree; if not found locally, check the repo's default branch (`gh api repos/{owner}/{repo}/contents/<path>` / `glab api`): `.github/PULL_REQUEST_TEMPLATE.md`, `.github/PULL_REQUEST_TEMPLATE/*.md`, root and `docs/` variants (case-insensitive); GitLab: `.gitlab/merge_request_templates/*.md`; Gitea: `.gitea/`. When the template directory holds multiple templates, pick the one matching the change type — or ask the user
2. **PR rules** in `CONTRIBUTING.md` (required sections, checklist expectations, title formats)
3. **Forge** from the remote URL → `gh` (GitHub) or `glab` (GitLab)

Close/tracker syntax: GitHub issue → `Fixes #123`; GitLab issue → `Closes #123`; Jira → plain `Fixes: <URL>` line (Jira links do not auto-close).

### 3. Draft

**Template exists** — keep its headings, checkboxes, and order; map substance into them. Drop the template's instruction comments (HTML comments) from the final body:

| Substance | Common template headings |
| --- | --- |
| Why | Problem, Why, Background, Context |
| What & how | Solution, What changed, Description, Changes |
| Proof | Evidence, Testing, How Has This Been Tested, Verification |
| Scope boundaries / deviations | Justifications, Notes, Breaking changes |

Fill every required section. Checkboxes: mark honestly — leave unchecked with a one-line reason rather than deleting.

**No template** — use this skeleton:

```md
<!-- Fixes #123 — tracker line only when one exists, syntax per forge -->

## Why
<broken workflow or gap — user-level, not the code-level cause; 1–3 sentences>

## What changed
<approach and why; explicit non-goals: what was deliberately not touched>

## Verification
- <exact commands run, in code spans>
- <manual/cluster checks; screenshots/GIFs for UI changes>
```

**Title:** the user-visible change, present tense. A repo-mandated title format wins.

### 4. Style — small, technical, high-signal

- First sentence of the change section states the change itself.
- Exact commands in code spans (`make test`), not prose ("ran tests").
- Bugs get before/after behavior lines.
- Name non-goals explicitly ("No API, RBAC, or migration changes").
- Long output in `<details>` with a one-line summary above.
- Visuals only for UI/CSS diffs; logs and commands for everything else.
- Redact tokens, passwords, and cluster credentials.

### 5. Self-check before emitting

Score the draft the way a reviewer would — fix, don't ship:

- [ ] Why, What, and Proof each substantive somewhere in the body (substance matters, heading does not)
- [ ] No placeholders or stale content
- [ ] UI/CSS changes have visual evidence
- [ ] Close-keyword syntax matches the forge and tracker
- [ ] Body fits the discovered template (or the fallback skeleton)

## Output

Default: print the title and body as a markdown block for the author to review. On request, create the PR/MR (`gh pr create --title ... --body-file ...` / `glab mr create`), keeping maintainer edits allowed. If creation fails, hand back the markdown block rather than retrying blindly.
