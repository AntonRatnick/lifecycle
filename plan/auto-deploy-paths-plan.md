# Build-Level `autoDeployPaths` For Push-Triggered Auto Deploy

## Summary
Add `environment.autoDeployPaths` to `lifecycle.yaml` as an optional include-glob list that gates push-triggered redeploy when `environment.autoDeploy` is enabled.

Chosen behavior:
- `environment.autoDeploy` remains the primary on/off switch.
- `environment.autoDeployPaths` is an optional build-level gate.
- If `autoDeployPaths` is configured, redeploy on push only when at least one changed file matches.
- If no changed file matches, skip push-triggered redeploy.
- Manual/API/comment-triggered redeploy flows bypass this gate.
- Diff basis is push range only (`before..after` from push webhook).

Why this is the easier path:
- No per-service filtering or deployable scoping required.
- No `deployableIds` queue/pipeline threading required.
- Limited changes concentrated in webhook gating + schema/validation.

---

## Public Interfaces And Types To Change

1. YAML schema and model types
- Add optional environment property:
  - `environment.autoDeployPaths?: string[]`
- Update:
  - `src/server/models/yaml/types.ts` (`LifecycleYamlConfigEnvironment`)
  - `src/server/lib/yamlSchemas/schema_1_0_0/schema_1_0_0.ts` (`environment.properties`)
  - generated artifacts:
    - `src/server/lib/jsonschema/schemas/1.0.0.json`
    - `docs/schema/yaml/1.0.0.yaml`

2. No external API changes
- Existing HTTP endpoints and queue payloads remain unchanged.

---

## Implementation Plan

1. Add changed-files helper for push range
- In `src/server/lib/github/index.ts`, add helper to fetch changed file paths from compare API using `before..after`.
- Return normalized repo-root-relative paths.
- Include modified/added/removed/renamed paths.
- Return `null` when retrieval fails or results are unusable.

2. Add path-match utility for include globs
- Create `src/server/lib/autoDeployPathFilter.ts`.
- Input:
  - `changedPaths: string[]`
  - `autoDeployPaths: string[]`
- Use `picomatch` include matching (with `dot: true`).
- Return `true` if any changed path matches at least one include glob.

3. Validate `autoDeployPaths`
- In `src/server/lib/yamlConfigValidator.ts`, after schema validation:
  - if `config.environment?.autoDeployPaths` exists, validate with `validateFileExclusionPatterns`.
- Validation constraints reused for safety:
  - disallow broad patterns `*`, `**`, `**/*`
  - disallow absolute paths
  - disallow path traversal
  - enforce max count and max length

4. Wire gate into push webhook processing
- Update `src/server/services/github.ts` in `handlePushWebhook`:
  - Keep existing deploy candidate/build selection logic.
  - Keep existing failed deploy detection logic.
  - Fetch lifecycle config once for pushed repo/branch (best effort).
  - Read `environment.autoDeploy` and `environment.autoDeployPaths`.
  - Gate behavior:
    - If `autoDeploy` is false, existing behavior already skips queueing.
    - If `autoDeploy` is true and `autoDeployPaths` is missing/empty, keep existing queue behavior.
    - If `autoDeploy` is true and `autoDeployPaths` is configured:
      - retrieve changed files for the push
      - if changed files retrieval fails, fail open and keep existing queue behavior
      - if retrieval succeeds and no path matches, skip queueing for that build
      - if match exists, queue as today
  - Preserve failed-deploy precedence:
    - if build has failed deploys, bypass `autoDeployPaths` gate and queue as today

5. Regenerate schema docs
- Run `pnpm run generate:schemas` after schema update.

---

## Test Cases And Scenarios

1. Push webhook gate tests (`src/server/services/__tests__/github.test.ts`)
- `autoDeploy=true`, `autoDeployPaths` set, matching file changed => queue occurs.
- `autoDeploy=true`, `autoDeployPaths` set, no matching files => queue skipped.
- `autoDeploy=true`, no `autoDeployPaths` => existing behavior unchanged.
- changed-files retrieval failure => fail-open existing behavior.
- failed deploys present => gate bypassed and queue behavior preserved.

2. Path-filter utility tests
- single glob match
- multi-glob match
- no matches
- dotfile path handling
- empty changed path list

3. YAML validation tests
- valid `environment.autoDeployPaths` accepted.
- unsafe/broad patterns rejected.
- unknown properties still rejected by schema (`additionalProperties: false`).

4. Regression tests for bypass flows
- manual/API redeploy endpoints unaffected.
- comment edit redeploy flow unaffected.

---

## Assumptions And Defaults

- `autoDeployPaths` is an include-list gate, not an ignore-list.
- Matching uses repo-root-relative file paths from GitHub push compare.
- This feature applies only to push-triggered redeploy decisions.
- On lifecycle YAML fetch failure, keep existing push behavior.
- On changed-files retrieval failure, keep existing push behavior (fail open).
