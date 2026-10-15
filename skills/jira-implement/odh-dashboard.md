# ODH Dashboard — Project Reference

Project-specific knowledge for the Jira implementation skill when working on `opendatahub-io/odh-dashboard`.

## Project basics

- **Repo**: `opendatahub-io/odh-dashboard`
- **Monorepo**: npm workspaces + Turbo
- **Key docs**: `AGENTS.md`, `CLAUDE.md`, `BOOKMARKS.md` at repo root
- **Rules**: `.claude/rules/` — read relevant rules before starting work (see rule table in AGENTS.md)
- **Existing skills**: `.claude/skills/` — check for skills that overlap with the task
- **Branch naming**: `RHOAIENG-XXXXX/<short-kebab-description>`
- **Commit format**: `RHOAIENG-XXXXX: <short description>`
- **Fork remote**: `origin` (user's fork), `upstream` (opendatahub-io)

## Feature flag registration — 7 files

When adding a new feature flag, ALL of these must be updated. Missing any will cause CI failures or runtime issues.

### 1. Type definition — `packages/k8s-core/src/k8sTypes.ts`

Add as optional boolean at the end of `DashboardCommonConfig`:
```typescript
gpuaas?: boolean;
```

### 2. SupportedArea enum — `packages/plugin-core/src/areas/types.ts`

Add enum entry with a comment section header:
```typescript
/* GPUaaS */
GPUAAS_INFRASTRUCTURE = 'gpuaas-infrastructure',
```

### 3. Flag default + state map — `frontend/src/concepts/areas/const.ts`

**a)** Add to the correct flag group (see decision matrix below):
```typescript
gpuaas: false,
```

**b)** Add to `SupportedAreasStateMap` at the end:
```typescript
[SupportedArea.GPUAAS_INFRASTRUCTURE]: {
  featureFlags: ['gpuaas'],
  requiredComponents: [DataScienceStackComponent.KUEUE],
},
```

### 4. CRD schema — `manifests/common/crd/odhdashboardconfigs.opendatahub.io.crd.yaml`

Add before the `x-kubernetes-validations` section under `dashboardConfig.properties`:
```yaml
                    gpuaas:
                      type: boolean
                      description: 'When true, enables the GPUaaS Infrastructure page.'
```

### 5. Backend type — `backend/src/types.ts`

Add to the `dashboardConfig` type:
```typescript
gpuaas: boolean;
```

### 6. Backend default — `backend/src/utils/constants.ts`

Add to the `dashboardConfig` defaults object:
```typescript
gpuaas: false,
```

### 7. Test mock — `frontend/src/__mocks__/mockDashboardConfig.ts`

Three locations in this file:
- **Type**: add `gpuaas?: boolean;` to the config type
- **Function parameter**: add `gpuaas = false,` to the function defaults
- **Return object**: add `gpuaas,` to the `dashboardConfig` return

## Flag group decision matrix

| Condition | Group | Example |
|-----------|-------|---------|
| `disableX` pattern, established feature | `coreDashboardFlags`, `projectManagementFlags`, `modelServingFlags`, or `advancedAIMLFlags` | `disableModelServing` |
| Positive flag, feature going GA | `advancedAIMLFlags` | `trainingJobs`, `gpuaas` |
| Positive flag, tech preview | `techPreviewFlags` | `observabilityDashboard`, `modelAsService` |
| Temporary dev-only flag, expected to be removed | `devTemporaryFeatureFlags` | `disableKueue`, `nimWizard` |

## Package scaffold checklist

### Bundled plugin (no webpack, no MF, compiled into host)

Reference packages: `model-training`, `feature-store`, `observability`

- [ ] `packages/<name>/package.json` — name, exports (`./extensions`), deps, scripts (include `test-unit`)
- [ ] `packages/<name>/tsconfig.json` — extends `@odh-dashboard/tsconfig`
- [ ] `packages/<name>/.eslintrc.js` — `recommendedReactTypescript(__dirname)`
- [ ] `packages/<name>/jest.config.ts` — re-export from `@odh-dashboard/jest-config`
- [ ] `packages/<name>/extensions.ts` — `app.area` + `app.navigation/href` + `app.route`
- [ ] `packages/<name>/src/` — routes, page components, constants
- [ ] `packages/<name>/OWNERS` — approvers, reviewers, area label
- [ ] Feature flag in all 7 locations (see above)
- [ ] `npm install` to wire workspace
- [ ] Verify `package-lock.json` (see lock file section below)

### Federated module (own webpack, MF remote)

Use the existing `module-onboarding` skill for the scaffold, then apply the flag checklist on top.

## OWNERS file template

```yaml
approvers:
  - <github-username>
  - <github-username>

reviewers:
  - <github-username>
  - <github-username>

labels:
  - area/<package-name>
```

The `area/<package-name>` label must be created in the GitHub repo. Use:
```bash
gh label create "area/<name>" --repo opendatahub-io/odh-dashboard --color "0075ca" --description "<description>"
```

## Common conventions and pitfalls

### React imports
```typescript
// Correct
import * as React from 'react';

// Wrong — inconsistent with codebase convention
import React from 'react';
```

### Wildcard route redirects
```tsx
// Correct — replace prevents extra history entries
<Route path="*" element={<Navigate to="." replace />} />

// Wrong — pushes to history, breaks back button
<Route path="*" element={<Navigate to="." />} />
```

### ApplicationsPage in packages
```typescript
// Packages must suppress the lint warning
// eslint-disable-next-line @odh-dashboard/no-restricted-imports -- standard page shell wrapper
import ApplicationsPage from '@odh-dashboard/internal/pages/ApplicationsPage';
```

### Nav item ordering
Do not add `group` property to nav items unless there is a documented ordering contract for that section. Items without `group` are ordered by default.

### Extension area patterns
`reliantAreas` + `featureFlags` referencing the same flag is the established pattern (used by `model-training`, `feature-store`). The duplication is intentional defense-in-depth.

### Admin gating
Use the `ADMIN_USER` string constant in `flags.required` on both nav and route extensions:
```typescript
const ADMIN_USER = 'ADMIN_USER';
// ...
flags: { required: [PLUGIN_FLAG, ADMIN_USER] }
```
This is set from `useUser().isAdmin` in `PluginStoreAreaFlagsProvider`.

### DSC null gating (false positive)
CodeRabbit may flag that `requiredComponents` is bypassed when `dscStatus` is null. This is a false positive — `AreaContextProvider` renders a spinner until DSC status loads, then an error page if it's null. The `isAreaAvailable` function never runs with null `dscStatus` for nav/route gating.

### data-testid
Add `data-testid` on all interactive and testable elements. Use descriptive kebab-case names with a component prefix:
```tsx
data-testid="infrastructure-cluster-section"
```

## package-lock.json — macOS vs CI

npm on macOS adds platform-specific markers (`"dev": true`, `"peer": true` on fsevents-related packages) that CI on Linux strips. This causes the "Check for uncommitted changes" CI step to fail.

**Workaround for new workspace packages:**
1. `git checkout main -- package-lock.json` to start from clean base
2. Manually add only the workspace link entries:
   - `node_modules/@odh-dashboard/<name>` link entry
   - `packages/<name>` metadata entry
3. Do NOT run `npm install` after — it will re-add platform markers

This is a known limitation of cross-platform npm lock files. The manual entries are deterministic (workspace links mirror package.json exactly).

## PR creation

- Use `.github/pull_request_template.md` as the body structure
- Read `.claude/rules/pull-requests.md` for agent-specific guidance
- Create via `user-GitHubMCP` MCP (`create_pull_request`) with `head: "dpanshug:<branch>"`, `base: "main"`, targeting `opendatahub-io/odh-dashboard`
- Honest checklist: only check `[x]` items you can substantiate. Leave "manually tested" and "tested on cluster" unchecked if you only ran automated checks.

## Review comment evaluation

When evaluating CodeRabbit or preflight review comments:

1. **Check the actual codebase**, not just the PR diff — many comments reference behavior in files the PR didn't change
2. **Cross-reference with other packages** — if model-training does the same thing without issue, the comment is likely a false positive
3. **Present each comment to the user** with: what it says, whether it's valid, what other packages do, and your recommendation (fix/skip/discuss)
4. **Never remove dependencies** based on "unused" comments without checking if they'll be needed in follow-on stories
5. **New commits for fixes**, never amend+force-push unless user asks
