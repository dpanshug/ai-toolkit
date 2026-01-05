---
description: Checkout a GitHub PR branch locally
---

Checkout PR #${{PR_NUMBER}} from the upstream repository.

Run this command:
```bash
git fetch upstream pull/${{PR_NUMBER}}/head:pr-${{PR_NUMBER}} && git checkout pr-${{PR_NUMBER}}
```

Confirm when the checkout is successful.
