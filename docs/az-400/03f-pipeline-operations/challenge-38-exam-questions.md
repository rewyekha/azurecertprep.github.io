---
sidebar_position: 95
title: "Challenge 38: exam questions"
---

# Challenge 38 — AZ-400 exam questions

**48 questions** built only from what Challenge 38 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-38.md`**.

| Section | Shape | Questions |
|---|---|---|
| A | Single answer | 1–16 |
| B | Multiple answer (choose two / three) | 17–23 |
| C | Repeated scenario — "Does this meet the goal?" | 24–26 |
| D | Yes/No statement grid | 27–30 |
| E | Drag and drop | 31–35 |
| F | Hot area — complete the configuration | 36–41 |
| G | Case study | 42–48 |

The **trap index**, the **complete pipeline**, and **scoring** are at the end.

:::danger This is the Domain 3 capstone — half the exam in one pipeline

Every challenge from 13 to 37 appears here: package feeds, coverage gates, sharding, OIDC, container
build, Bicep what-if, blue-green swap, App Insights annotation, retention. Questions deliberately
cross those boundaries, and the **job dependency graph** is where most marks are won or lost.

:::

---

# Section A — Single answer

---

## Q1

Which configuration authenticates npm to GitHub Packages for the `@contoso` scope?

- A. `.npmrc` with `@contoso:registry` plus `setup-node` with `registry-url` and `scope`
- B. `npm login` in a run step
- C. A PAT in `package.json`
- D. `npm config set always-auth false`

### Answer: A

**In `challenge-38.md`:** lines **130–149**.

```text
@contoso:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}
always-auth=true
```

```yaml
        uses: actions/setup-node@v4
        with:
          registry-url: "https://npm.pkg.github.com"
          scope: "@contoso"
      - run: npm ci
        env:
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Three pieces that must agree:** the `.npmrc` scopes `@contoso` to GitHub Packages and reads
`NODE_AUTH_TOKEN`; `setup-node` writes the registry configuration; and the env var supplies the token
at install time.

**`NODE_AUTH_TOKEN` is `secrets.GITHUB_TOKEN`** — no PAT needed, because the package lives in the same
organisation. That is why the workflow declares `packages: write` at line 223.

**Why the others fail** — B is interactive, C would commit a credential, D disables the auth this
requires.

---

## Q2

Which job dependency prevents an image being built from code that failed its quality gates?

- A. `needs: [coverage-gate, test-integration, security-scan]`
- B. `needs: lint`
- C. `if: always()`
- D. `needs: test-unit`

### Answer: A

**In `challenge-38.md`:** line **372**.

```yaml
  build-image:
    needs: [coverage-gate, test-integration, security-scan]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

**All three gates, not one.** An array in `needs` waits for **every** listed job to succeed —
coverage above 80%, integration tests green, and no high or critical vulnerabilities.

**Why D is the near-miss:** `needs: test-unit` would wait for the shards but not for the **merged**
coverage check. A repository can have every shard green and still be at 78.5% overall, which is
exactly Break & fix Exercise 1.

**Why the others fail** — B waits only for linting, C would build regardless of any failure.

---

## Q3

The coverage gate fails at 78.5% against an 80% threshold. What is the correct response?

- A. Add tests for the uncovered error paths in `push.service.ts`
- B. Lower the threshold to 75%
- C. Exclude `push.service.ts` from coverage
- D. Set `continue-on-error: true` on the gate

### Answer: A

**In `challenge-38.md`:** Break & fix Exercise 1, lines **822–868**.

```text
#   "total": { "lines": { "pct": 78.5 } },
#   "src/services/push.service.ts": { "lines": { "pct": 45.0 } }  <-- Problem
# The push.service.ts has error handling paths that are never tested
```

**The diagnosis is in the per-file breakdown**, and it names the real problem: **error handling paths
are untested**. The added tests (lines 845–866) cover an invalid device token and a transient network
retry — precisely the paths that matter in production and are least exercised in development.

**Why the others fail — and this is the pattern the exam tests**

- **B** — lowering a threshold to pass is how a quality gate becomes decoration. The next shortfall
  lowers it again
- **C** — excludes the **least** covered file, so the number rises while the risk stays
- **D** — the gate reports and never blocks

**Line 822 states the exam's framing outright:** *"One developer suggests lowering the threshold.
Instead, find and fix the root cause."*

---

## Q4

Production deploys report success but users see the old version, and the swap appears to reverse
immediately.

What is the cause?

- A. The health check runs immediately after the swap, before the new code is serving
- B. The swap command targets the wrong slot
- C. The container image is wrong
- D. The environment lacks approval

### Answer: A

**In `challenge-38.md`:** Break & fix Exercise 2, lines **877–889**.

```yaml
    # ERROR: Checking immediately after swap without warmup time
    # The new code needs 15-30 seconds to start serving traffic
    if [ "$STATUS" != "200" ]; then
      az webapp deployment slot swap ...  # Swaps back!
```

**The rollback logic is correct and its trigger is wrong.** The check fires before the new instances
are accepting traffic, reads a non-200, and correctly performs the rollback it was told to perform.

**The fix (lines 899–930)** adds three things:

```bash
    sleep 15                                    # let the swap settle
    for i in $(seq 1 6); do                     # retry, not a single shot
      VERSION=$(curl -s .../health | jq -r '.version')
      if [ "$VERSION" = "${{ needs['build-image'].outputs.image_version }}" ]; then
```

**The version comparison is the subtle part.** A 200 only proves *something* is answering — possibly
the old version still draining. Comparing against the built version proves the **new** code is live.

---

## Q5

Which permission does the workflow need to post a coverage comment on a pull request?

- A. `pull-requests: write`
- B. `contents: write`
- C. `checks: write`
- D. `issues: write`

### Answer: A

**In `challenge-38.md`:** lines **221–225**, used at line **344**.

```yaml
permissions:
  id-token: write         # OIDC to Azure
  contents: read          # checkout
  packages: write         # publish/consume GitHub Packages
  pull-requests: write    # coverage comment
  checks: write           # check runs
```

**Five permissions, five capabilities**, and you can read the workflow's behaviour off the block. That
is the least-privilege pattern applied consistently.

**Why C is the closest wrong answer:** `checks: write` creates **check runs** — the pass/fail entries
beside a commit. Posting a comment on the PR conversation is `pull-requests: write`. Both appear here
because the pipeline does both.

---

## Q6

Which authentication method does the pipeline use for Azure?

- A. OIDC with `client-id`, `tenant-id` and `subscription-id`, backed by `id-token: write`
- B. `AZURE_CREDENTIALS` service principal JSON
- C. A managed identity on the runner
- D. An ACR admin username and password

### Answer: A

**In `challenge-38.md`:** lines **221** and **390–397**.

```yaml
      - name: Log in to Azure Container Registry
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - run: az acr login --name contosonotifacr
```

**Note the two-step ACR login:** `azure/login` establishes the Azure identity, then `az acr login`
exchanges it for a registry token. No registry credential is stored.

**And `AZURE_CLIENT_ID` is environment-scoped** (lines 785–786) — staging and production use
**different** service principals, so a staging deployment cannot touch production resources.

**Why the others fail** — B stores a secret, C has no managed identity on a hosted runner, D is the
shared admin account.

---

## Q7

Which two Trivy settings make the security scan a blocking gate?

- A. `scan-type: "fs"` and `exit-code: "1"`
- B. `severity: "HIGH,CRITICAL"` alone
- C. `format: "sarif"`
- D. `ignore-unfixed: true`

### Answer: A

**In `challenge-38.md`:** lines **256–262**.

```yaml
      - name: Run Trivy vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: "fs"
          scan-ref: "."
          severity: "HIGH,CRITICAL"
          exit-code: "1"
```

**`scan-type: "fs"` scans the filesystem** — source and dependency manifests — rather than a built
image. That is deliberate here: the scan runs **before** the image is built (line 372), so a vulnerable
dependency never reaches the registry.

**`exit-code: "1"` is what converts a report into a gate**, the same distinction as Challenge 28.

**And `npm audit --audit-level=high`** (line 254) runs alongside it — two scanners with different
databases, because neither catches everything.

---

## Q8

Which job condition builds the image only on merges to `main`?

- A. `if: github.event_name == 'push' && github.ref == 'refs/heads/main'`
- B. `if: github.ref_name == 'main'`
- C. `if: always()`
- D. `if: github.event_name == 'pull_request'`

### Answer: A

**In `challenge-38.md`:** line **373**.

**Both halves are required** — the event excludes pull requests, the ref pins the branch. This is
Challenge 19 Q6 recurring, and `github.ref` holds the **full** ref.

**The consequence for pull requests:** lint, security scan, unit tests, integration tests and the
coverage gate all run, and nothing is published or deployed. A PR gets full validation with zero
side effects.

**Why the others fail** — B does not exclude other events, C builds on failures, D is inverted.

---

## Q9

Which input allows a manual run that builds and tests without deploying?

- A. `workflow_dispatch` with a `skip_deploy` boolean input
- B. `workflow_call` with a string input
- C. `repository_dispatch`
- D. A `schedule` trigger

### Answer: A

**In `challenge-38.md`:** lines **214–218**, consumed at lines **464** and **536**.

```yaml
  workflow_dispatch:
    inputs:
      skip_deploy:
        description: "Skip deployment (build and test only)"
        type: boolean
        default: false
```

```yaml
  deploy-staging:
    if: ${{ !inputs.skip_deploy }}
```

**A `type: boolean` input arrives as a real boolean**, so it negates with `!` — no string comparison
(Challenge 22 Q15).

**Why it is useful:** validating a pipeline change, or building an image for inspection, without
touching an environment.

---

## Q10

How does the coverage gate combine results from the sharded unit tests?

- A. Download with `pattern: coverage-unit-*` and `merge-multiple: true`, then merge and check
- B. Read only shard 1's coverage
- C. Re-run all tests unsharded
- D. Average the shard percentages

### Answer: A

**In `challenge-38.md`:** lines **322–340**.

```yaml
      - name: Download coverage shards
        uses: actions/download-artifact@v4
        with:
          pattern: coverage-unit-*
          merge-multiple: true
          path: coverage-parts/
```

**Each shard sees only its half of the tests**, so each produces partial coverage. Merging is what
makes the 80% figure meaningful — this is Challenge 35's Break & fix Exercise 2 applied correctly from
the start.

**Why D is wrong and worth stating:** averaging percentages is not the same as merging coverage. Two
shards at 80% each can merge to well below 80% if they cover overlapping lines, or above it if they
cover disjoint ones. Only merging the line-level data gives the true figure.

---

## Q11

Which infrastructure checks run before deployment?

- A. `az bicep build` lint, `az deployment ... validate`, and a what-if analysis
- B. `terraform plan` only
- C. A checkov policy scan only
- D. None — Bicep deploys directly

### Answer: A

**In `challenge-38.md`:** lines **438–450**.

```yaml
      - name: Lint Bicep templates
        run: az bicep build --file infrastructure/main.bicep --stdout > /dev/null
      - name: Validate deployment
      - name: What-if analysis
```

**Three depths, cheapest first** — the pattern from Challenge 31 Q32: lint needs no Azure call,
validate asks whether Azure would accept it, what-if reports what would change.

**Note `validate-infra` runs `needs: build-image`** (line 427) — the image exists before the
infrastructure that will run it is validated, so a failure at either point stops the deployment.

---

## Q12

Which mechanism records a deployment marker in Application Insights?

- A. A POST creating an annotation with deployment version, actor and commit SHA
- B. A custom metric
- C. A log message
- D. An availability test

### Answer: A

**In `challenge-38.md`:** lines **517–527**.

```json
              "AnnotationName": "Deployment",
              "Category": "Deployment",
              "EventTime": "...",
              "Properties": "{\"DeploymentVersion\":\"...\",\"TriggeredBy\":\"...\",\"CommitSha\":\"...\"}"
```

**An annotation is a vertical marker on every metrics chart**, which is what makes "did this deploy
cause the latency spike?" answerable at a glance rather than by cross-referencing timestamps.

**The three properties are the traceability chain** — version, who triggered it, and which commit.
That is Challenge 03's source-to-production trace, closed at the observability end.

**Why the others fail** — B and C record data without marking the timeline; D checks uptime.

---

## Q13

Which retention setting is applied to the coverage artifacts?

- A. `retention-days: 7`
- B. `retention-days: 90`
- C. `retention-days: 365`
- D. No retention setting

### Answer: A

**In `challenge-38.md`:** lines **288–293**, matching the requirement at line **36**.

```yaml
        with:
          name: coverage-unit-${{ matrix.shard }}
          path: coverage/coverage-final.json
          retention-days: 7
```

**Coverage shards are intermediate data** — consumed by the gate job minutes later and worthless after
that. The 90-day default (Challenge 36 Q2) would keep them for three months at no benefit.

**Note the artifact name includes the shard** — `coverage-unit-${{ matrix.shard }}` — avoiding the
overwrite bug from Challenge 35.

---

## Q14

Which environment configuration requires manual approval for production only?

- A. Reviewers configured on the `production` environment; none on `staging`
- B. `wait_timer` on both environments
- C. A branch policy on `main`
- D. `if: github.actor == 'admin'`

### Answer: A

**In `challenge-38.md`:** lines **775–782**.

```bash
gh api repos/{owner}/{repo}/environments/staging --method PUT \
  --field wait_timer=0 \
  --field deployment_branch_policy='{"protected_branches":true,...}'

gh api repos/{owner}/{repo}/environments/production --method PUT \
  --field wait_timer=0 \
  --field reviewers='[{"type":"User","id":12345}]' \
  --field deployment_branch_policy='{"protected_branches":true,...}'
```

**The only difference is `reviewers`** — staging deploys automatically, production waits for a human.
That is exactly the requirement at line 31.

**Both environments restrict deployment branches** to protected branches, so neither can be deployed
from an arbitrary branch.

**Why the others fail** — B delays both without asking anyone, C gates merging, D is not an approval.

---

## Q15

Which secrets are environment-scoped, and which are shared?

- A. `AZURE_CLIENT_ID` per environment; `AZURE_TENANT_ID` and `AZURE_SUBSCRIPTION_ID` at repository
  level
- B. All three per environment
- C. All three at repository level
- D. All three in the workflow file

### Answer: A

**In `challenge-38.md`:** lines **785–790**.

```bash
gh secret set AZURE_CLIENT_ID --env staging    --body "{staging-sp-client-id}"
gh secret set AZURE_CLIENT_ID --env production --body "{prod-sp-client-id}"

gh secret set AZURE_TENANT_ID --body "{tenant-id}"
gh secret set AZURE_SUBSCRIPTION_ID --body "{subscription-id}"
```

**The client ID is the identity, so it differs per environment** — and that is the isolation. A
staging deployment authenticates as the staging principal, which has no rights in production.

**Tenant and subscription are the same for both**, so scoping them per environment would duplicate a
value with no benefit.

**The decision rule from Challenge 24 Q35:** *differs per environment* → environment scope. *Same
everywhere* → repository scope.

---

## Q16

Which condition ensures pipeline metrics are recorded even when deployment fails?

- A. `if: always()` on the metrics job
- B. `if: success()`
- C. `needs: deploy-production` alone
- D. `continue-on-error: true`

### Answer: A

**In `challenge-38.md`:** lines **803–815**.

```yaml
  pipeline-metrics:
    needs: [deploy-production]
    if: always()
    steps:
      - name: Record deployment metrics
        # ... console.log(`Deployment status: ${{ needs['deploy-production'].result }}`)
```

**Failed deployments are the ones worth measuring.** Without `always()`, the metrics job is skipped
precisely when MTTR and failure-rate data matter — the point Challenge 34 makes about visibility.

**Note it reads `needs['deploy-production'].result`** (line 815) rather than assuming success, so the
recorded status is accurate.

**Why the others fail** — B skips on failure, C alone still skips when the dependency fails,
D relates to the job's own errors.

---

# Section B — Multiple answer

---

## Q17

Which **three** jobs must succeed before the container image is built? (Choose three.)

- A. `coverage-gate`
- B. `test-integration`
- C. `security-scan`
- D. `validate-infra`
- E. `deploy-staging`
- F. `lint`

### Answer: A, B, C

**In `challenge-38.md`:** line **372**.

```yaml
    needs: [coverage-gate, test-integration, security-scan]
```

**`lint` is a transitive dependency, not a direct one.** `test-unit` needs `lint` (line 270), and
`coverage-gate` needs the shards — so linting must pass, but it is not in the array.

**Why D and E fail — the order is the opposite of intuition.** `validate-infra` needs `build-image`
(line 427), and `deploy-staging` needs both. The image is built first, then the infrastructure that
will host it is validated, then deployment happens.

---

## Q18

Which **three** quality gates run on a pull request? (Choose three.)

- A. Lint and type checking
- B. Unit tests with the coverage gate
- C. Security scanning
- D. Container image build
- E. Staging deployment
- F. Blue-green production swap

### Answer: A, B, C

**In `challenge-38.md`:** the PR trigger at line **212** and the build condition at line **373**.

```yaml
  build-image:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

**Everything before the image build runs on a PR; nothing after it does.** That is the same
build-on-PR, publish-on-merge principle as Challenge 28 Q2, expressed as a job condition rather than a
`push:` input.

**And the coverage comment** (lines 344–345) is posted **only** on pull requests — the reviewer sees
the number in the conversation where the decision is made.

---

## Q19

Which **two** fix the broken blue-green verification? (Choose two.)

- A. Wait for the swap to settle before the first health check
- B. Retry the health check and compare the reported version against the built version
- C. Remove the rollback step
- D. Increase the swap timeout
- E. Check health before swapping instead

### Answer: A, B

**In `challenge-38.md`:** Break & fix Exercise 2, lines **899–920**.

```bash
    sleep 15
    for i in $(seq 1 6); do
      STATUS=$(curl ...)
      if [ "$STATUS" = "200" ]; then
        VERSION=$(curl -s .../health | jq -r '.version')
        if [ "$VERSION" = "${{ needs['build-image'].outputs.image_version }}" ]; then
```

**B is the part people omit.** A 200 proves *something* answered — possibly the old version still
draining connections. Comparing the version proves the swap actually took effect.

**Why the others fail**

- **C** — removes the safety net rather than fixing its trigger
- **D** — the swap itself succeeded; the verification is what is mistimed
- **E** — the warm-up check **already** runs before the swap (lines 568–584). This is the *post*-swap
  verification, and both are needed

---

## Q20

Which **two** are configured on the composite action rather than in each job? (Choose two.)

- A. Node.js setup with the GitHub Packages registry and scope
- B. `node_modules` caching keyed on `package-lock.json`
- C. The coverage threshold
- D. Environment approvals
- E. The Docker build

### Answer: A, B

**In `challenge-38.md`:** lines **171–194**.

```yaml
runs:
  using: "composite"
  steps:
    - uses: actions/setup-node@v4
      with:
        registry-url: "https://npm.pkg.github.com"
        scope: "@contoso"
    - id: cache
      uses: actions/cache@v4
      with:
        key: ${{ runner.os }}-modules-${{ hashFiles('package-lock.json') }}
    - if: steps.cache.outputs['cache-hit'] != 'true'
      shell: bash
      run: npm ci
```

**Five jobs use `- uses: ./.github/actions/setup-project`** — lint, security-scan, test-unit,
test-integration and coverage-gate. Without the composite action, that registry configuration and
cache logic would be duplicated five times and drift.

**Note `shell: bash`** on the `run` step (line 192) — mandatory in a composite action (Challenge 23
Q4).

**Why the others fail** — C is in the coverage-gate script, D is on the environment in the UI, E is in
`build-image`.

---

## Q21

Which **two** security scanners run before the image is built? (Choose two.)

- A. `npm audit --audit-level=high`
- B. Trivy with `scan-type: "fs"`
- C. CodeQL
- D. Trivy scanning the built image
- E. Defender for Containers

### Answer: A, B

**In `challenge-38.md`:** lines **253–262**.

**Two scanners, two databases.** `npm audit` uses the npm advisory database and understands the
dependency tree; Trivy uses its own vulnerability database and also inspects files. Neither is a
superset of the other.

**Both scan the filesystem, before the build** (line 372 gates on `security-scan`), so a vulnerable
dependency never becomes an image.

**Why the others fail**

- **C** — CodeQL finds **code** defects like injection flaws, a different class. Real and not used here
- **D** — image scanning is Challenge 28's pattern; this pipeline scans the source instead
- **E** — registry-side scanning, complementary and not part of this workflow

---

## Q22

Which **two** describe the production deployment sequence? (Choose two.)

- A. Deploy to the staging slot, warm it, swap, then verify production
- B. Roll back by swapping again if verification fails
- C. Deploy directly to the production slot
- D. Route 10% of traffic before swapping
- E. Delete the staging slot after swapping

### Answer: A, B

**In `challenge-38.md`:** lines **559–608**.

```yaml
      - name: Deploy to staging slot (blue-green)
      - name: Warm up staging slot
      - name: Swap slots (blue-green deployment)
      - name: Verify production health
        # ... on failure: az webapp deployment slot swap ... (rollback)
```

**Warm before swap, verify after** — the Challenge 26 pattern. The warm-up loop (lines 577–584)
ensures the slot is serving before it becomes production, and the post-swap verification catches
anything the warm-up missed.

**Why the others fail**

- **C** — deploying straight to production destroys the rollback path (Challenge 30 Q26)
- **D** — traffic routing is a canary, a different strategy
- **E** — the staging slot **holds the previous version** after the swap. Deleting it removes the way
  back

---

## Q23

Which **two** optimisations from Challenge 35 appear in this pipeline? (Choose two.)

- A. Test sharding with a matrix strategy
- B. `node_modules` caching in the composite action
- C. Self-hosted runners
- D. Turborepo incremental builds
- E. `--maxWorkers=4`

### Answer: A, B

**In `challenge-38.md`:** lines **271–274** and **181–187**.

```yaml
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2]
```

**Plus Docker layer caching** at lines 416–417 (`cache-from: type=gha`), which is a third — so A and B
are the intended pair for **test** optimisation specifically.

**Why the others fail** — C, D and E are all real Challenge 35 techniques not used here. Two shards
rather than four is proportionate to this service's suite size; sharding has overhead, and four shards
on a small suite can be slower than two.

---

# Section C — Repeated scenario

**Scenario:** Contoso must ship the Notification Service with 80% minimum coverage, no high-severity
vulnerabilities in the image, automatic staging deployment, and production behind manual approval with
a verified blue-green swap.

---

## Q24

**Proposed solution:** Gate `build-image` on `coverage-gate`, `test-integration` and `security-scan`.
Deploy to staging automatically with an environment that has no reviewers. Deploy to production through
an environment with a required reviewer, deploying to the staging slot, warming it, swapping, then
verifying both status and version with retries before rolling back on failure.

Does this meet the goal? **Yes**

### Answer: Yes

**In `challenge-38.md`:** lines **372**, **465–467**, **537–539**, **775–782**, **899–930**.

| Requirement | Mechanism |
|---|---|
| 80% coverage | `coverage-gate` merges shards, exits 1 below threshold |
| No high-severity vulnerabilities | `security-scan` with `exit-code: "1"`, before the build |
| Staging automatic | Environment with **no reviewers** |
| Production approved | Environment with `reviewers` |
| Verified blue-green | Warm, swap, retry-verify **version**, roll back on failure |

The version comparison is what makes the verification trustworthy rather than merely a 200 check.

---

## Q25

**Proposed solution:** Gate `build-image` on `test-unit` only. Deploy to staging and production
automatically. Verify production with a single health check immediately after the swap.

Does this meet the goal? **No**

### Answer: No

**Three failures, and each is a different lesson from Domain 3.**

**`needs: test-unit` skips the merged coverage check.** Every shard can pass while the combined figure
sits at 78.5% — Break & fix Exercise 1 exactly. It also skips integration tests and the security scan
entirely.

**Production deploying automatically** violates the manual approval requirement. An environment
without reviewers is not a gate.

**The single immediate health check** is Break & fix Exercise 2: it reads a non-200 from a slot that
has not started serving and rolls back a perfectly good release.

---

## Q26

**Proposed solution:** Gate `build-image` on all three quality jobs, deploy to staging automatically,
require a reviewer on production, warm the slot, swap, and verify with a retry loop checking for HTTP
200.

Does this meet the goal? **No**

### Answer: No

Everything is right except one detail — and it is the one that makes verification meaningful.

**Checking only for HTTP 200 does not prove the swap took effect.** During and just after a slot swap,
the old instances continue serving in-flight and newly arriving requests for a short period. A 200 in
that window comes from the **previous** version.

The result is a release that reports verified while users are on old code — precisely the symptom in
Break & fix Exercise 2's title: *"Production deployment reports success but users see the old
version."*

**The fix is the version comparison** (line 912):

```bash
        VERSION=$(curl -s .../health | jq -r '.version')
        if [ "$VERSION" = "${{ needs['build-image'].outputs.image_version }}" ]; then
```

**Which is why `build-image` declares `image_version` as a job output** (line 374) — so the verifier
knows what it is looking for.

---

# Section D — Yes/No statement grid

---

## Q27 — pipeline structure

| # | Statement | Answer |
|---|---|---|
| 1 | `build-image` waits for coverage, integration tests and the security scan | **Yes** |
| 2 | The image is built on pull requests | **No** |
| 3 | `validate-infra` runs before `build-image` | **No** |
| 4 | `deploy-staging` waits for both `build-image` and `validate-infra` | **Yes** |

**In `challenge-38.md`:** lines **372**, **373**, **427**, **463**.

Row 3 inverts the real order: `validate-infra` declares `needs: build-image`. The image exists first,
then the infrastructure that will host it is validated.

Row 2 is the PR behaviour — full validation, no publishing.

---

## Q28 — quality gates

| # | Statement | Answer |
|---|---|---|
| 1 | Sharded coverage must be merged before checking the threshold | **Yes** |
| 2 | Lowering the threshold is the correct fix for a shortfall | **No** |
| 3 | `exit-code: "1"` makes Trivy block the pipeline | **Yes** |
| 4 | The security scan runs after the image is built | **No** |

**In `challenge-38.md`:** lines **322–340**, **822**, **262**, **372**.

Row 4 matters: scanning **before** the build means a vulnerable dependency never becomes an image that
someone could deploy later by accident.

Row 2 is the exam's framing at line 822 — find the root cause, which here is untested error-handling
paths in `push.service.ts`.

---

## Q29 — deployment

| # | Statement | Answer |
|---|---|---|
| 1 | Staging deploys without approval; production requires a reviewer | **Yes** |
| 2 | The production deploy writes to the staging slot before swapping | **Yes** |
| 3 | An HTTP 200 immediately after a swap proves the new version is live | **No** |
| 4 | The rollback is another slot swap | **Yes** |

**In `challenge-38.md`:** lines **775–782**, **559**, **883–884**, **924–928**.

Row 3 is the capstone's sharpest lesson. **Verify the version, not just the status code** — during a
swap the old instances are still answering.

Row 4: the rollback is the identical command, because a swap exchanges rather than copies
(Challenge 25 Q6).

---

## Q30 — configuration placement

| # | Statement | Answer |
|---|---|---|
| 1 | `AZURE_CLIENT_ID` differs per environment | **Yes** |
| 2 | `AZURE_TENANT_ID` is a repository secret | **Yes** |
| 3 | Approvals are declared in the workflow YAML | **No** |
| 4 | Coverage artifacts use `retention-days: 7` | **Yes** |

**In `challenge-38.md`:** lines **785–790**, **775–782**, **293**.

Row 1 is the isolation: different service principals mean a staging deployment has no rights in
production.

Row 3 recurs from Challenge 24 and Challenge 37 — the YAML **names** the environment; the protection
rules live on it, configured through the API or UI (lines 775–782).

---

# Section E — Drag and drop

---

## Q31

Arrange the pipeline jobs into execution waves.

```yaml
  lint:              (no needs)
  security-scan:     (no needs)
  test-unit:         needs: lint        # matrix shard [1, 2]
  test-integration:  needs: lint
  coverage-gate:     needs: test-unit
  build-image:       needs: [coverage-gate, test-integration, security-scan]
  validate-infra:    needs: build-image
  deploy-staging:    needs: [build-image, validate-infra]
  deploy-production: needs: [deploy-staging, build-image]
  pipeline-metrics:  needs: deploy-production, if: always()
```

### Answer — seven waves

| Wave | Jobs |
|---|---|
| 1 | `lint`, `security-scan` |
| 2 | `test-unit` (×2 shards), `test-integration` |
| 3 | `coverage-gate` |
| 4 | `build-image` |
| 5 | `validate-infra` |
| 6 | `deploy-staging` |
| 7 | `deploy-production`, then `pipeline-metrics` |

**In `challenge-38.md`:** lines **237**, **246**, **267–270**, **295–298**, **372**, **427**, **463**,
**535**, **806**.

**`security-scan` is in wave 1**, which people miss — it declares no `needs`, so it starts immediately
alongside `lint` rather than waiting for it. The gate is at line 372, where `build-image` waits for it.

**Total duration is the critical path**, so the two shards cost one shard's time, and `security-scan`
is free unless it is slower than the whole lint-test-coverage chain.

---

## Q32

Match each Domain 3 challenge to where it appears in this capstone.

| Challenge | Appears as |
|---|---|
| 13–15 packages | **`.npmrc` scope + `NODE_AUTH_TOKEN` for `@contoso`** |
| 16–18 testing | **Sharded unit tests, integration tests, 80% coverage gate** |
| 19–24 pipelines | **Composite action, matrix, environments, approvals** |
| 25–30 deployments | **Blue-green slot swap with warm-up and rollback** |
| 31–33 IaC | **Bicep lint, validate and what-if before deploying** |
| 34–37 operations | **Caching, sharding, 7-day retention, metrics job** |

**In `challenge-38.md`:** the skills map at lines **18–23**, with implementations at **130–149**,
**267–340**, **156–194**, **559–608**, **438–450**, **181–187**.

**This is why the capstone is worth taking seriously as revision.** One pipeline exercises 25
challenges, and the exam's Domain 3 questions are drawn from the same surface.

---

## Q33

Arrange the production deployment steps in order.

**Items:** Swap slots · Verify version and status with retries · Warm up the staging slot · Deploy the
container to the staging slot · Roll back if verification fails

### Answer

1. Deploy the container to the staging slot — line **559**
2. Warm up the staging slot — line **568**, retry loop until 200
3. Swap slots — line **589**
4. Verify version and status with retries — line **597**, fixed at lines 899–920
5. Roll back if verification fails — lines **924–928**

**Steps 2 and 4 are both retry loops and they check different things.** The warm-up asks *"is the new
code ready?"* before it becomes production. The verification asks *"did the swap take effect?"* after.

**Skipping either produces a documented failure:** no warm-up gives users a cold start (Challenge 26),
no version check gives a false success (Break & fix Exercise 2).

---

## Q34

Match each configuration to where it lives.

| Configuration | Lives in |
|---|---|
| Job graph and quality gates | **The workflow YAML** |
| Coverage threshold check | **The workflow YAML** (coverage-gate script) |
| Required reviewers on production | **The environment** — API or UI |
| Deployment branch policy | **The environment** |
| `AZURE_CLIENT_ID` per environment | **Environment secrets** |
| `AZURE_TENANT_ID` | **Repository secrets** |
| Registry scope for `@contoso` | **`.npmrc` + `setup-node`** |

**In `challenge-38.md`:** lines **233–340**, **775–782**, **785–790**, **130–143**.

**Three locations, and a complete-looking YAML can still be missing the gate.** That is the same
lesson as Challenge 37 Q34 — the diff does not show the environment configuration.

---

## Q35

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Coverage gate fails at 78.5% | **Untested error paths in `push.service.ts`** |
| Production reports success but serves old code | **Health check ran before the swap settled; no version check** |
| `npm ci` fails to resolve `@contoso/notification-sdk` | **`NODE_AUTH_TOKEN` not set, or scope missing from `setup-node`** |
| Image built from code with failing integration tests | **`build-image` gated on `test-unit` instead of all three jobs** |
| Production deploys without approval | **No reviewers configured on the environment** |

**In `challenge-38.md`:** lines **829**, **883**, **139–149**, **372**, **781**.

**Two of these produce green pipelines** — the old-version swap and the missing approval. Those are the
ones worth being able to spot from a description rather than from a failure.

---

# Section F — Hot area

---

## Q36

```yaml
permissions:
  [BLANK 1]: write      # OIDC to Azure
  contents: read
  [BLANK 2]: write      # publish and consume GitHub Packages
  pull-requests: write
```

- **BLANK 1:** `id-token` / `deployments` / `actions` / `checks`
- **BLANK 2:** `packages` / `contents` / `repository-projects` / `security-events`

### Answer: `id-token`, `packages`

**In `challenge-38.md`:** lines **221** and **223**.

`id-token: write` is required for OIDC — omit it and `azure/login` fails with a token-request error
that does not mention permissions. `packages: write` covers the `@contoso` scope on GitHub Packages.

---

## Q37

```yaml
  build-image:
    needs: [[BLANK 1], test-integration, security-scan]
    if: github.event_name == '[BLANK 2]' && github.ref == '[BLANK 3]'
```

- **BLANK 1:** `coverage-gate` / `test-unit` / `lint` / `validate-infra`
- **BLANK 2:** `push` / `pull_request` / `workflow_dispatch` / `schedule`
- **BLANK 3:** `refs/heads/main` / `main` / `heads/main` / `refs/main`

### Answer: `coverage-gate`, `push`, `refs/heads/main`

**In `challenge-38.md`:** lines **372–373**.

`test-unit` is the near-miss: the shards can each pass while the **merged** figure is below threshold.
`github.ref` holds the full ref.

---

## Q38

```yaml
      - name: Download coverage shards
        uses: actions/download-artifact@v4
        with:
          [BLANK 1]: coverage-unit-*
          [BLANK 2]: true
          path: coverage-parts/
```

- **BLANK 1:** `pattern` / `name` / `filter` / `match`
- **BLANK 2:** `merge-multiple` / `combine` / `flatten` / `merge`

### Answer: `pattern`, `merge-multiple`

**In `challenge-38.md`:** lines **325–326**.

`name:` downloads exactly one artifact. `pattern:` plus `merge-multiple: true` collects every shard
into one directory so the merge step can combine them — Challenge 35's fix applied from the start.

---

## Q39

```yaml
      - name: Deploy to staging slot (blue-green)
      - name: Warm up staging slot
      - name: Swap slots
      - name: Verify production health
        run: |
          sleep [BLANK 1]
          for i in $(seq 1 6); do
            VERSION=$(curl -s .../health | jq -r '[BLANK 2]')
            if [ "$VERSION" = "${{ needs['build-image'].outputs.[BLANK 3] }}" ]; then
```

- **BLANK 1:** `15` / `0` / `120` / `1`
- **BLANK 2:** `.version` / `.status` / `.uptime` / `.commit`
- **BLANK 3:** `image_version` / `sha` / `tag` / `build_id`

### Answer: `15`, `.version`, `image_version`

**In `challenge-38.md`:** lines **903**, **911**, **912**.

`sleep 0` is the bug — checking immediately reads the draining old version. The version comparison is
what distinguishes "something answered" from "the new code is live", and `image_version` is the job
output declared at line 374 for exactly this purpose.

---

## Q40

```bash
gh api repos/{owner}/{repo}/environments/production --method PUT \
  --field wait_timer=0 \
  --field [BLANK 1]='[{"type":"User","id":12345}]' \
  --field deployment_branch_policy='{"[BLANK 2]":true,"custom_branch_policies":false}'

gh secret set AZURE_CLIENT_ID --[BLANK 3] production --body "{prod-sp-client-id}"
```

- **BLANK 1:** `reviewers` / `approvers` / `gates` / `checks`
- **BLANK 2:** `protected_branches` / `all_branches` / `main_only` / `require_review`
- **BLANK 3:** `env` / `repo` / `org` / `scope`

### Answer: `reviewers`, `protected_branches`, `env`

**In `challenge-38.md`:** lines **779–786**.

`reviewers` is what makes production manual — staging omits it entirely. `--env` scopes the client ID
so staging and production authenticate as different principals.

---

## Q41

```yaml
  test-unit:
    needs: lint
    strategy:
      [BLANK 1]: false
      matrix:
        shard: [1, 2]
    steps:
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-unit-[BLANK 2]
          retention-days: [BLANK 3]
```

- **BLANK 1:** `fail-fast` / `continue-on-error` / `max-parallel` / `strict`
- **BLANK 2:** `${{ matrix.shard }}` / `latest` / `all` / *(omit)*
- **BLANK 3:** `7` / `90` / `365` / `1`

### Answer: `fail-fast`, `${{ matrix.shard }}`, `7`

**In `challenge-38.md`:** lines **272**, **291**, **293**.

`fail-fast: false` reports both shards' failures in one run. The shard suffix prevents the overwrite
bug from Challenge 35. Seven days matches the stated requirement at line 36 and avoids the 90-day
default.

---

# Section G — Case study

## Case study: Contoso Notification Service

### Background

Contoso is launching the **Notification Service** — a Node.js 20 Express API handling email, SMS and
push notifications. You are building the CI/CD pipeline from scratch.

### Requirements

**Source and packages**

- GitHub repository, TypeScript
- Depends on the private `@contoso/notification-sdk` npm package

**Quality**

- Lint, type check, unit tests at **80% coverage minimum**, integration tests, security scan
- Pull requests must be fully validated without publishing anything

**Delivery**

- Container image pushed to **Azure Container Registry**
- Infrastructure deployed via **Bicep**, validated in the pipeline
- **Staging automatic**, **production manual approval** with a **blue-green slot swap**

**Observability and operations**

- Smoke tests after deployment, Application Insights deployment annotation
- Caching, parallel jobs, **7-day** artifact retention
- No credential stored for Azure

---

## Q42

Which **two** satisfy the private package requirement? (Choose two.)

- A. `.npmrc` scoping `@contoso` to `npm.pkg.github.com` with `NODE_AUTH_TOKEN`
- B. `setup-node` with `registry-url` and `scope`, plus `packages: write`
- C. Committing the SDK source into the repository
- D. A personal access token stored as a repository secret
- E. Publishing the SDK to the public npm registry

### Answer: A, B

**In `challenge-38.md`:** lines **130–133** and **139–149**.

**Why the others fail**

- **C** — vendoring the dependency abandons versioning and updates
- **D** — **works and is unnecessary.** `secrets.GITHUB_TOKEN` (line 149) already authenticates to
  GitHub Packages in the same organisation, so a PAT adds a long-lived credential for no benefit.
  Cross-organisation access would be different (Challenge 40)
- **E** — publishing a private SDK publicly

---

## Q43

Which configuration enforces the 80% coverage minimum correctly?

- A. Merge the sharded coverage, compute the total, and exit 1 below 80
- B. Fail each shard below 80%
- C. Check the first shard's coverage
- D. Report coverage without failing

### Answer: A

**In `challenge-38.md`:** lines **322–340**.

```bash
            echo "::error::Coverage ${COVERAGE}% is below the 80% threshold"
            exit 1
```

**Why B is the interesting wrong answer.** Per-shard thresholds sound stricter and are not equivalent:
a shard containing mostly well-covered files can pass at 90% while another sits at 60%, and the merged
total — the number that matters — is never computed. Coverage is a property of the codebase, not of an
arbitrary test split.

**Why the others fail** — C is a quarter of the truth, D is a report rather than a gate.

---

## Q44

Which **two** meet the "no stored credential for Azure" requirement? (Choose two.)

- A. `permissions: id-token: write` on the workflow
- B. `azure/login@v2` with `client-id`, `tenant-id` and `subscription-id`
- C. `AZURE_CREDENTIALS` service principal JSON
- D. ACR admin username and password
- E. A managed identity on the GitHub-hosted runner

### Answer: A, B

**In `challenge-38.md`:** lines **221** and **390–396**.

**Why the others fail**

- **C** and **D** — both store a long-lived secret
- **E** — a GitHub-hosted runner is not an Azure resource and has no managed identity. Challenge 21
  Q21 and Challenge 25 Q45 make the same point

**And note `az acr login` afterwards** (line 397) — the Azure identity is exchanged for a registry
token rather than a registry credential being stored.

---

## Q45

Which **two** satisfy the deployment requirements? (Choose two.)

- A. A `staging` environment with no reviewers
- B. A `production` environment with a required reviewer
- C. Both environments with reviewers
- D. Neither environment configured
- E. A `workflow_dispatch` confirmation input for production

### Answer: A, B

**In `challenge-38.md`:** lines **775–782**.

**Why the others fail**

- **C** — staging must be automatic; a reviewer on staging slows every release for no benefit
- **D** — no environment means no gate, no deployment record and no environment secrets
  (Challenge 24 Q1)
- **E** — an input is filled in by whoever triggers the run. That is not another person approving

---

## Q46

Which sequence satisfies "blue-green slot swap" with verification?

- A. Deploy to slot → warm up → swap → verify status **and version** with retries → roll back on
  failure
- B. Deploy to production → verify → roll back
- C. Deploy to slot → swap → single health check
- D. Deploy to slot → swap → no verification

### Answer: A

**In `challenge-38.md`:** lines **559–608**, corrected at **899–930**.

**Why the others fail**

- **B** — deploying straight to production leaves nothing in the slot to swap back to
- **C** — Break & fix Exercise 2. The single immediate check reads the draining old version and rolls
  back a good release
- **D** — a bad release stays live

---

## Q47

The pipeline is live. A release passes every gate and deploys successfully, but Application Insights
shows a latency increase starting at the deployment time — and nobody can tell which of three
deployments that day caused it.

What is missing, and what should Contoso add?

- A. Deployment annotations carrying the version, actor and commit SHA
- B. More smoke tests
- C. A longer warm-up
- D. Higher coverage

### Answer: A

**In `challenge-38.md`:** lines **517–527**.

```json
              "AnnotationName": "Deployment",
              "Properties": "{\"DeploymentVersion\":\"...\",\"TriggeredBy\":\"...\",\"CommitSha\":\"...\"}"
```

**The annotation is what puts a marker on the metrics chart**, so a latency step change lines up
visually with a specific deployment rather than with a timestamp someone has to correlate by hand.

**The three properties turn "which deployment" into "which commit".** Version identifies the build,
commit SHA identifies the change, actor identifies who to ask.

**Note the challenge annotates staging** (line 517). Extending it to production is the gap this
question describes — and it closes the Challenge 03 traceability chain at the observability end.

**Why the others fail** — B, C and D are all real improvements that would not help identify *which*
deployment caused the change.

---

## Q48

Six months on, the pipeline is stable but takes 28 minutes. Analysis shows: lint 2 min, security scan
6 min, unit shards 4 min each, integration 7 min, coverage gate 1 min, image build 5 min, infra
validate 2 min, staging deploy 3 min.

Where is the critical path, and what should Contoso change first?

- A. `lint` → `test-integration` → `build-image` → `validate-infra` → `deploy-staging` ≈ 19 min;
  target the 7-minute integration tests
- B. Add more unit test shards
- C. Move to self-hosted runners
- D. Remove the security scan

### Answer: A

**In `challenge-38.md`:** the dependency graph at lines **237–463**.

**Work the graph.** `security-scan` (6 min) has no `needs`, so it runs in parallel from the start and
is not on the path. `test-integration` needs `lint` and takes 7 minutes — longer than
`test-unit` + `coverage-gate` (4 + 1 = 5). So:

```text
lint 2 -> test-integration 7 -> build-image 5 -> validate-infra 2 -> deploy-staging 3  = 19 min
```

**Adding shards would not help**, which is why B is the tempting wrong answer: unit tests are already
off the critical path. Making them faster changes nothing.

**What would help, in order:** split or parallelise the integration tests, and look at whether
`validate-infra` genuinely needs `build-image` — if not, it could move to wave 1 and save two more
minutes.

**Why the others fail**

- **C** — Challenge 35 Q25's lesson: optimise before buying capacity, and the queue is not the problem
  here
- **D** — removing a security gate to save time it does not even cost, since it runs in parallel

**The general lesson:** measure the **critical path**, not the sum. Optimising a job that runs in
parallel with a longer one saves nothing at all.

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Gating on `test-unit` instead of `coverage-gate`** | Q2, Q17, Q25, Q37 | Shards can pass while the merged total fails |
| **Lowering a threshold to pass** | Q3, Q28, Q43 | Find the untested paths. A moved gate is no gate |
| **Verifying a swap with status only** | Q4, Q19, Q26, Q39 | Compare the **version**. 200 can be the old code |
| **No warm-up before the swap** | Q22, Q33 | Warm, swap, then verify — two separate loops |
| **Approvals expected in YAML** | Q14, Q30, Q45 | They live on the environment |
| **Reviewers on staging** | Q45 | Staging is automatic; only production gates |
| **`AZURE_CREDENTIALS` instead of OIDC** | Q6, Q44 | `id-token: write` + client/tenant/subscription |
| **Shared artifact name across shards** | Q41 | Suffix with `${{ matrix.shard }}` |
| **90-day default retention left in place** | Q13, Q41 | Intermediate artifacts get 7 days |
| **Scanning after the image build** | Q7, Q21, Q28 | Scan the filesystem first, before publishing |
| **`security-scan` assumed to depend on `lint`** | Q31 | It declares no `needs` — wave 1 |
| **Optimising a job off the critical path** | Q48 | Measure the longest chain, not the sum |

---

# The complete pipeline

Line numbers are in `challenge-38.md`.

```text
# Job graph  (lines 237-815)
wave 1  lint                 security-scan          (no needs)
wave 2  test-unit [shard 1,2]   test-integration    (needs: lint)
wave 3  coverage-gate                               (needs: test-unit)
wave 4  build-image        (needs: coverage-gate, test-integration, security-scan
                            + if: push && refs/heads/main)
wave 5  validate-infra     (needs: build-image)
wave 6  deploy-staging     (needs: build-image, validate-infra | environment: staging)
wave 7  deploy-production  (needs: deploy-staging, build-image | environment: production)
        pipeline-metrics   (needs: deploy-production | if: always())
```

```yaml
# 1. Permissions  (lines 221-225)
permissions:
  id-token: write        # OIDC
  contents: read
  packages: write        # @contoso scope
  pull-requests: write   # coverage comment
  checks: write

# 2. Private package auth  (lines 130-149)
# .npmrc:  @contoso:registry=https://npm.pkg.github.com
#          //npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}
      - uses: actions/setup-node@v4
        with:
          registry-url: "https://npm.pkg.github.com"
          scope: "@contoso"
      - run: npm ci
        env:
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

# 3. Shard, then merge  (lines 271-340)
      matrix:
        shard: [1, 2]
          name: coverage-unit-${{ matrix.shard }}   # unique
          retention-days: 7
...
          pattern: coverage-unit-*
          merge-multiple: true
          # exit 1 if merged coverage < 80

# 4. Scan BEFORE building  (lines 253-262)
      - run: npm audit --audit-level=high
      - uses: aquasecurity/trivy-action@master
        with:
          scan-type: "fs"
          severity: "HIGH,CRITICAL"
          exit-code: "1"

# 5. IaC checks, cheapest first  (lines 438-450)
      az bicep build --stdout > /dev/null      # lint
      az deployment ... validate               # acceptable?
      az deployment ... what-if                # what changes?

# 6. Blue-green with a REAL verification  (lines 559-608, fixed 899-930)
      deploy to staging slot -> warm up (retry to 200)
      -> swap -> sleep 15 -> retry: status 200 AND version == image_version
      -> on failure: swap back
```

```bash
# 7. Environments and secrets  (lines 775-790)
# staging:    wait_timer=0, protected_branches - NO reviewers
# production: wait_timer=0, protected_branches + reviewers=[...]
gh secret set AZURE_CLIENT_ID --env staging|production   # differs per environment
gh secret set AZURE_TENANT_ID                            # shared, repository level
```

**The capstone's own lesson:** every gate is only as strong as what depends on it, and every
verification is only as good as what it actually checks.

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Domain 3 is exam-ready. Section 03f is complete |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 3 and 5, and both Break & fix exercises |
| Below 30 | Redraw the job graph from memory, then revisit Challenges 25, 28 and 35 |

Record your result in `AZ-400-Learning-Log.md` under Challenge 38.

:::danger The two things

**Gate on the merged result, not the parts.** `coverage-gate`, not `test-unit`.

**Verify the version, not the status code.** A 200 after a swap can come from the code you just
replaced.

Both appear in Break & fix, both produce green pipelines, and both are the kind of question that
separates a pass from a high score.

:::
