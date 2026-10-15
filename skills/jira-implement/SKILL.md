---
name: jira-implement
description: Takes a Jira URL or issue key, understands the task, classifies the work type, and orchestrates end-to-end implementation — from studying patterns through PR creation and review handling. Delegates to project-specific reference files and existing skills where they exist.
---

# Jira-to-Implementation Orchestrator

Accepts a Jira URL or issue key. Understands the task, studies patterns, implements, verifies, creates a PR, and handles review comments.

## Arguments

`$ARGUMENTS` — A Jira URL (e.g., `https://redhat.atlassian.net/browse/RHOAIENG-72324`) or issue key (e.g., `RHOAIENG-72324`). If empty, ask the user.

## Phase 0: Fetch and Understand

1. **Extract the issue key** from `$ARGUMENTS`. Parse from URL if needed (strip `https://redhat.atlassian.net/browse/`).

2. **Fetch the issue** via `user-atlassian` MCP:
   ```
   jira_get_issue(issue_key: "<KEY>", fields: "*all")
   ```

3. **Extract and present** to the user:
   - Summary, description, acceptance criteria
   - Issue type, priority, status, assignee
   - Labels, linked issues, prototype/design URLs
   - Dependencies mentioned in description

4. **Classify work type** — present to user for confirmation:

   | Type | Signals |
   |------|---------|
   | New package scaffold | "Create packages/X", "new package", "scaffold", "skeleton", new SupportedArea |
   | Feature implementation | AC describes new functionality, data fetching, UI sections in existing package |
   | Bug fix | Issue type is Bug, "crashes", "broken", "regression", "doesn't work" |
   | UI change | "update layout", "change nav", "add empty state", "redesign", screenshots |
   | Refactor | "move", "extract", "consolidate", "rename", "migrate", "decouple" |
   | Test addition | "add test", "cypress", "unit test", "test coverage", "mock test" |

5. **Load project-specific reference** — check for a `<project-name>.md` file in this skill's directory. For odh-dashboard, load `odh-dashboard.md`. If none exists, proceed with generic workflow.

## Phase 1: Study Patterns

Read relevant rules and find reference code based on classification.

### Rules to read (project-specific)

Check the project's `AGENTS.md` or `CLAUDE.md` for a rule table. Read the rules relevant to the work type. At minimum, always read the coding conventions rule.

### Find reference patterns

- **New package**: Find the closest existing package (same type: bundled plugin or federated module). Read its `extensions.ts`, `package.json`, main page component, and route file.
- **Feature**: Find the closest existing feature in the target package or a sibling package. Read its hooks, components, context, and tests.
- **Bug fix**: If a `diagnose` skill exists in the project, delegate reproduction to it. Otherwise, reproduce manually: read error reports, find the code path, understand the expected vs actual behavior.
- **Refactor**: Read both the source and target locations. Understand all consumers of the code being moved.
- **Test**: Read the project's test conventions. Find an existing test file in the same area as a template.

### Use background agents for exploration

Launch explore subagents in parallel to study different aspects of the codebase. Don't try to read everything sequentially.

## Phase 2: Ask Clarifying Questions

Only ask what the Jira description, AC, and codebase exploration cannot answer. Use the `AskQuestion` tool.

### Per-type guidance

**New package scaffold:**
- Feature flag name and which flag group it belongs to
- SupportedArea enum name
- Nav section, URL path, page title
- Admin or RBAC gating requirements
- Page section titles or layout structure
- Bundled plugin vs federated module (if not obvious from description)

**Feature implementation:**
- Data sources / API endpoints
- Polling/refresh requirements
- Loading, error, and empty state expectations

**Bug fix:**
- Reproduction steps (if not in the Jira)
- Which environments are affected

**Refactor / UI change / Test:**
- Usually minimal questions needed — the Jira and codebase provide enough context

## Phase 3: Implement

1. **Create branch**: `<JIRA-KEY>/<short-kebab-description>`

2. **Implement** using the project's conventions and the patterns studied in Phase 1.

3. **Follow project-specific checklists** from the reference file (e.g., `odh-dashboard.md`). Common checklists:
   - Feature flag registration (all required files)
   - Package scaffold (all config + code + metadata files)
   - OWNERS / code ownership files

4. **Convention enforcement** — apply the common pitfalls and conventions from the reference file. These are mistakes learned the hard way — don't repeat them.

## Phase 4: Verify

If the project has a `dev-workflow` skill, delegate to it. Otherwise:

1. **Type-check** — run the project's type checker on affected packages
2. **Lint** — run linter on all new and modified files, fix any introduced errors
3. **Tests** — run related unit tests, ensure they pass
4. **Self code-review** — re-read every file changed and check against the AC checklist item by item

## Phase 5: Commit, Push, PR

1. **Commit** with `<JIRA-KEY>: <short description>` message format. Never amend+force-push unless the user explicitly asks.

2. **Push** to the user's fork (`origin`).

3. **Create PR** — check for:
   - A PR template in the project (`.github/pull_request_template.md`)
   - A `pull-requests` rule in the project's rules
   - Use `user-GitHubMCP` MCP if available, otherwise `gh pr create`
   - Only check `[x]` on checklist items you can substantiate

## Phase 6: Review Handling

When the user asks to check review comments:

1. **Fetch comments** via `gh api repos/{owner}/{repo}/pulls/{number}/comments`

2. **For each comment**, evaluate properly:
   - Read the actual code being referenced (not just the diff snippet)
   - Check if the finding is valid by reading the surrounding codebase context
   - Cross-reference with how other packages handle the same pattern
   - Classify as: **fix** (valid, should change) / **skip** (false positive or not applicable) / **discuss** (valid concern but tradeoffs exist)

3. **Present assessment to the user** with your reasoning before making any changes. Never blindly fix all comments.

4. **After user decides**, make fixes as new commits (not amend).
