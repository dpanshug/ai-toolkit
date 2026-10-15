---
description: Checkout a GitHub PR branch locally
---

Checkout PR #$1 from the upstream repository.

If no PR number was provided, ask which PR to checkout before doing anything.

Run this command:
```bash
git fetch upstream pull/$1/head:pr-$1 && git checkout pr-$1
```

Confirm when the checkout is successful.
