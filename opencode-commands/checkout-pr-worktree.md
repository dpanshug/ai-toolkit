---
description: Checkout a GitHub PR into a separate worktree (keeps current branch undisturbed)
---

Checkout PR #$1 from the upstream repository into a git worktree so the current branch remains untouched.

If no PR number was provided, ask which PR to checkout before doing anything.

Run these commands:
```bash
git fetch upstream pull/$1/head:pr-$1 && git worktree add ../odh-dashboard-pr-$1 pr-$1
```

Confirm the worktree was created successfully and print the path.

Remind the user:
- The PR is checked out at `../odh-dashboard-pr-$1`
- Run `npm install` in that directory before testing
- When done: `git worktree remove ../odh-dashboard-pr-$1`
