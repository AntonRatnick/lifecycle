# Per-Service File-Change Ignore Globs For Push-Triggered Deploys

## Summary
Add `lifecycle.yaml` support for per-service ignore globs so push-triggered deploy/redeploy only runs for services that have at least one changed file not ignored by that service.

Decisions locked for this implementation:
- Config is per service (`services[].fileChangeIgnoreGlobs`).
- Diff basis is push range only (`before..after` from GitHub push webhook).
- Mixed changes deploy only affected deployables.
- Manual/API/comment-triggered redeploy actions bypass this filter.
- If changed files cannot be retrieved reliably, fail open (keep current behavior).
- If a build has failed deploys, bypass filtering for that build and preserve current failed-recovery behavior.
- Globs match repo-root-relative paths, for example `src/api/foo.ts`.

---

## Public Interfaces And Types To Change

1. YAML schema and model types
- Add optional property: `services[].fileChangeIgnoreGlobs?: string[]`.
- Update:
  - `src/server/models/yaml/YamlService.ts` (`Service` interface)
  - `src/server/models/yaml/types.ts` (types used in GitHub/YAML helper flow)
  - `src/server/lib/yamlSchemas/schema_1_0_0/schema_1_0_0.ts` (services item properties)
  - generated artifacts:
    - `src/server/lib/jsonschema/schemas/1.0.0.json`
    - `docs/schema/yaml/1.0.0.yaml`

2. Internal queue payload and processing signatures
- Extend queue payloads and method signatures with:
  - `deployableIds?: number[]`
- Thread through:
  - `GithubService` enqueue calls (push flow only)
  - `BuildService.processResolveAndDeployBuildQueue`
  - `BuildService.processBuildQueue`
  - `BuildService.resolveAndDeployBuild(...)`
  - deploy/build/manifest and env-resolve helper calls

No external HTTP API contract changes.

---

## Implementation Plan

1. Add changed-files helper for push range
- In `src/server/lib/github/index.ts`, add a helper to fetch changed file paths for a push using compare API (`before..after`).
- Return repo-root-relative paths as plain strings.
- Include modified/added/removed/renamed file paths.
- If retrieval fails or returns an unusable result, return `null` and let caller fail open.

2. Add service-file-change matching utility
- Create `src/server/lib/fileChangeFilter.ts`.
- Input:
  - `changedPaths: string[]`
  - per-service ignore glob map
  - service/deployable name
- Match with `picomatch` (`dot: true`).
- Service/deployable is affected when at least one changed path is not ignored.
- Service with empty/missing globs is treated as affected.

3. Enforce glob safety in YAML validation
- Keep JSON schema as structural validation only.
- In `src/server/lib/yamlConfigValidator.ts` (`validate_1_0_0`), after schema validation:
  - iterate services
  - call `validateFileExclusionPatterns` for `fileChangeIgnoreGlobs` when present
  - throw `ValidationError` on invalid patterns
- Safety semantics reused:
  - reject `*`, `**`, `**/*`
  - reject absolute paths
  - reject path traversal
  - enforce max count and length limits from existing validator

4. Wire filtering into push webhook flow
- In `src/server/services/github.ts` `handlePushWebhook`:
  - keep current build-candidate selection logic.
  - keep current failed deploy detection logic.
  - fetch changed files once per push event.
  - fetch lifecycle config once for pushed repo+branch (best effort), build map `service.name -> fileChangeIgnoreGlobs`.
  - for each candidate build:
    - if build has failed deploys: enqueue current behavior without filtering.
    - else if changed files unavailable: enqueue current behavior (fail open).
    - else compute affected deployables from build deploy list:
      - full-yaml mode primary key: `deploy.deployable.name`
      - fallback key: `deploy.service.name`
    - if none affected: skip queue for that build.
    - if affected subset exists: enqueue `resolve-deploy` with `buildId` and `deployableIds`.

5. Support deployable-scoped execution in build pipeline
- Extend build/deploy flows to accept optional `deployableIds` alongside existing `githubRepositoryId`:
  - `Deploy.findOrCreateDeploys(...)`
  - `BuildEnvironmentVariables.resolve(...)`
  - `buildImages(...)`
  - `deployCLIServices(...)`
  - `generateAndApplyManifests(...)`
  - deploy image detail updates
- Filter precedence:
  - if `deployableIds` provided, process only those deployables/deploy records.
  - else if `githubRepositoryId` provided, use repo-scoped behavior.
  - else keep full build-wide behavior.

6. Schema/doc generation
- Run `pnpm run generate:schemas` after schema/type updates.
- Ensure generated docs include `services.fileChangeIgnoreGlobs`.

---

## Test Cases And Scenarios

1. Push webhook filtering (`src/server/services/__tests__/github.test.ts`)
- ignored-only changes for service A => A excluded from queue scope.
- mixed changes => queue only affected `deployableIds`.
- no globs configured => current behavior preserved.
- changed-file retrieval failure => fail-open current behavior.
- failed deploys present => filtering bypassed for that build.

2. File-change matcher unit tests (`src/server/lib/...` new test file)
- empty globs
- all paths ignored
- mixed ignored/non-ignored
- unknown service name
- dotfile path matching

3. Build/deploy pipeline scoping tests
- when `deployableIds` is supplied, only target deployables are:
  - env-resolved
  - built
  - CLI-deployed
  - manifest-applied and image-state updated
- non-target deployables remain untouched.

4. YAML validation tests
- valid `fileChangeIgnoreGlobs` accepted.
- invalid broad/unsafe patterns rejected via validator hook.
- unknown properties still rejected (`additionalProperties: false`).

5. Regression checks for bypass paths
- manual/API service redeploy endpoints unchanged.
- comment edit redeploy flow unchanged.

---

## Assumptions And Defaults

- Change filtering is based only on push webhook commit range (`before..after`), not PR base/head and not deployed SHA.
- Matching is path-only, not change-status-aware.
- Feature applies only to push-triggered deploy/redeploy.
- Manual/API/comment-triggered redeploy remains exempt.
- On YAML fetch/read failure in push flow, skip filtering and preserve existing behavior.
- On changed-file retrieval failure, skip filtering and preserve existing behavior.
