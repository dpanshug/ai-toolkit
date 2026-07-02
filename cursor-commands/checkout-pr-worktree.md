---
description: Checkout a GitHub PR into a separate worktree (keeps current branch undisturbed)
---

Checkout PR #${{PR_NUMBER}} from the upstream repository into a git worktree so the current branch remains untouched.

Run these commands:
```bash
git fetch upstream pull/${{PR_NUMBER}}/head:pr-${{PR_NUMBER}} && git worktree add ../odh-dashboard-pr-${{PR_NUMBER}} pr-${{PR_NUMBER}}
```

Confirm the worktree was created successfully and print the path.

Remind the user:
- The PR is checked out at `../odh-dashboard-pr-${{PR_NUMBER}}`
- Run `npm install` in that directory before testing
- When done: `git worktree remove ../odh-dashboard-pr-${{PR_NUMBER}}`
