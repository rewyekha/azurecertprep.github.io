---
sidebar_position: 92
title: "Challenge 35: exam questions"
---

# Challenge 35 — AZ-400 exam questions

**48 questions** built only from what Challenge 35 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-35.md`**.

| Section | Shape | Questions |
|---|---|---|
| A | Single answer | 1–16 |
| B | Multiple answer (choose two / three) | 17–23 |
| C | Repeated scenario — "Does this meet the goal?" | 24–26 |
| D | Yes/No statement grid | 27–30 |
| E | Drag and drop | 31–35 |
| F | Hot area — complete the configuration | 36–41 |
| G | Case study | 42–48 |

The **trap index**, the **blocks to memorise**, and **scoring** are at the end.

:::tip Four levers, and the exam asks which one fits

**Cache** what does not change. **Parallelise** what is independent. **Skip** what is unaffected.
**Shrink** what moves between jobs. Every optimisation question is one of those four — and picking
the wrong lever is how the distractors work.

:::

---

# Section A — Single answer

---

## Q1

A cache configuration always reports a miss. The key is
`${{ runner.os }}-npm-${{ hashFiles('**/package.json') }}`.

What is wrong?

- A. `package.json` changes on every version bump, so the key changes constantly
- B. `runner.os` is not a valid context
- C. `hashFiles` requires an absolute path
- D. The cache path is wrong

### Answer: A

**In `challenge-35.md`:** Break & fix Exercise 1, lines **574–596**.

```yaml
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package.json') }}  # ERROR: Wrong file
    # package.json changes with every version bump
    # Should use package-lock.json which only changes when deps change
```

**The rule: key on the *lock* file.** `package.json` changes when the version is bumped, a script is
edited or a field is reordered — none of which alters the installed dependency tree. `package-lock.json`
changes **only when dependencies change**, which is exactly what the cache holds.

**The fix (lines 591–596)** also adds `restore-keys`:

```yaml
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
```

`restore-keys` gives a **partial hit** — when the exact key misses, the most recent prefix match is
restored, so a single new dependency does not throw away the whole cache.

**Why the others fail** — B, C and D are all valid as written.

---

## Q2

Test sharding works, but the coverage report shows only 25%.

What is the cause?

- A. Each shard uploads to the same artifact name, so the last one wins
- B. Coverage is disabled in the shards
- C. `fail-fast: true` cancels the other shards
- D. The merge job runs before the shards

### Answer: A

**In `challenge-35.md`:** Break & fix Exercise 2, lines **605–638**.

```yaml
  with:
    name: coverage  # ERROR: Same name across all shards - last one wins
```

**Fixed:**

```yaml
    name: coverage-shard-${{ matrix.shard }}   # unique per shard
...
  - uses: actions/download-artifact@v4
    with:
      pattern: coverage-shard-*
      merge-multiple: true
```

**Sharding splits the work, so it splits the coverage too.** Each shard sees only its quarter of the
tests, so four partial reports must be **merged** (`npx nyc merge`) before the number means anything.

**Why the others fail**

- **B** — coverage is collected (line 187); the shards are just overwriting each other
- **C** — `fail-fast: false` is set at line 167, and cancellation would show missing shards, not a
  quarter of the coverage
- **D** — `needs: test-unit` (line 198) orders it correctly

---

## Q3

Which cache does `actions/setup-node` with `cache: "npm"` populate?

- A. `node_modules`
- B. The global npm download cache (`~/.npm`)
- C. The Docker layer cache
- D. The build output

### Answer: B

**In `challenge-35.md`:** lines **67–71**.

```yaml
        uses: actions/setup-node@v4
        with:
          cache: "npm"  # Automatically caches ~/.npm based on package-lock.json
```

**That is why the challenge adds a *second* cache** at lines 73–78 for `node_modules` itself. They save
different things:

| Cache | Saves |
|---|---|
| `~/.npm` (setup-node) | **Downloading** packages from the registry |
| `node_modules` (actions/cache) | **Installing** them — the unpack and link step |

With both, a cache hit lets you skip `npm ci` entirely (line 81), which is where the 5 minutes of
install time goes.

**Why the others fail** — C is buildx cache (line 116), D is an artifact.

---

## Q4

Which condition skips dependency installation when the `node_modules` cache hit?

- A. `if: steps['cache-modules'].outputs['cache-hit'] != 'true'`
- B. `if: always()`
- C. `continue-on-error: true`
- D. `if: cache-hit == true`

### Answer: A

**In `challenge-35.md`:** lines **73–82**.

```yaml
      - name: Cache node_modules
        id: cache-modules
        uses: actions/cache@v4
      - name: Install dependencies
        if: steps['cache-modules'].outputs['cache-hit'] != 'true'
        run: npm ci
```

**Three parts, all required:** the cache step has an `id`, the install step reads
`steps.<id>.outputs.cache-hit`, and the comparison is against the **string** `'true'`.

**`cache-hit` is a string, not a boolean** — the same class as Challenge 22's `$GITHUB_OUTPUT` values.
`!= true` would not behave as expected.

**Why the others fail**

- **B** — always installs, so the cache saves nothing
- **C** — failure handling
- **D** — no `steps.<id>` prefix, so it references nothing

---

## Q5

Which Docker cache configuration reuses layers across runs?

- A. `cache-from: type=local,src=... ` with `cache-to: type=local,dest=...,mode=max`
- B. `push: true`
- C. `load: true`
- D. `pull: always`

### Answer: A

**In `challenge-35.md`:** lines **109–117**.

```yaml
          cache-from: type=local,src=/home/runner/.docker-cache
          cache-to: type=local,dest=/home/runner/.docker-cache,mode=max
```

Paired with `actions/cache` on that directory (lines 101–107), because a **hosted runner is fresh every
run** — the local cache directory only survives if something persists it.

**`mode=max` caches intermediate layers**, which matters for a multi-stage build where the expensive
work happens in an early stage.

**The Azure Pipelines equivalent uses a registry instead** (lines 152–154):

```bash
--cache-from type=registry,ref=contosoregistry.azurecr.io/contoso/api:cache
--cache-to   type=registry,ref=contosoregistry.azurecr.io/contoso/api:cache,mode=max
```

**Registry versus local is a real trade-off:** registry cache is shared across every runner and
survives cache eviction; local cache is faster to read but scoped to whatever restored it.

**Why the others fail** — B pushes the image, C loads it into the local daemon, D is not a
`build-push-action` input.

---

## Q6

Which Jest flag splits a test suite across parallel runners?

- A. `--shard=1/4`
- B. `--maxWorkers=4`
- C. `--runInBand`
- D. `--bail`

### Answer: A

**In `challenge-35.md`:** lines **166–186**.

```yaml
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]
...
          npx jest --ci --shard=${{ matrix.shard }}/4
```

**`--shard` splits across *machines*; `--maxWorkers` splits across *cores on one machine*.** They
compose — four runners each using multiple workers — and they solve different problems. Sharding is
what turns 12 minutes of unit tests into roughly 3.

**Why `fail-fast: false` matters here** (line 167): with the default `true`, one failing shard cancels
the other three, so you see one failure instead of all of them and have to re-run to learn the rest.

**Why the others fail** — C forces serial execution, D stops on first failure.

---

## Q7

Which Azure Pipelines variables let a job know its shard position?

- A. `$(System.JobPositionInPhase)` and `$(System.TotalJobsInPhase)`
- B. `$(Agent.Id)` and `$(Agent.MachineName)`
- C. `$(Build.BuildId)` and `$(Build.BuildNumber)`
- D. `$(System.JobId)` and `$(System.StageName)`

### Answer: A

**In `challenge-35.md`:** lines **238–242**.

```yaml
    strategy:
      parallel: 4  # Azure DevOps auto-splits test files across 4 agents
...
          npx jest --ci \
            --shard=$(System.JobPositionInPhase)/$(System.TotalJobsInPhase)
```

**`strategy: parallel: 4` creates four identical jobs**, and these two variables are how each one
learns which slice it owns — position 1 of 4, 2 of 4, and so on.

That is a genuine platform difference: GitHub needs an explicit `matrix` listing the shards; Azure
Pipelines just takes a number and supplies the position.

**Why the others fail** — B identifies the agent, C identifies the build, D identifies the job and
stage without any position information.

---

## Q8

Which action detects which paths changed and exposes the result to later jobs?

- A. `dorny/paths-filter`
- B. `actions/checkout` with `fetch-depth: 0`
- C. `actions/cache`
- D. `tj-actions/changed-files` only

### Answer: A

**In `challenge-35.md`:** lines **258–281**.

```yaml
  detect-changes:
    outputs:
      api: ${{ steps.filter.outputs.api }}
      docs_only: ${{ steps.filter.outputs.docs_only }}
    steps:
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            api:
              - 'packages/api/**'
              - 'shared/**'
```

**Note `shared/**` appears under both `api` and `web`** (lines 273, 276) — a change to the shared
package must rebuild both consumers. That is the same monorepo rule as Challenge 22 Q47, and missing
it means a shared change silently skips the tests that would catch it.

**How this differs from a trigger path filter:** a trigger filter decides whether the **workflow**
runs; this decides which **jobs** run inside it. You still pay for the detection job, and you skip the
expensive ones.

**Why the others fail** — B enables history for a script to diff manually, C caches, D is a real
alternative action but not the one used.

---

## Q9

Why does the deploy job use `always()` in its condition?

- A. Because upstream test jobs may be **skipped** rather than successful
- B. To deploy even when tests fail
- C. To run on schedule
- D. To ignore the docs-only check

### Answer: A

**In `challenge-35.md`:** lines **309–316**.

```yaml
  deploy:
    needs: [test-api, test-web, detect-changes]
    if: |
      always() &&
      needs['detect-changes'].outputs.docs_only != 'true' &&
      (needs['test-api'].result == 'success' || needs['test-api'].result == 'skipped') &&
      (needs['test-web'].result == 'success' || needs['test-web'].result == 'skipped')
```

**A change touching only the API skips `test-web`** — and a skipped dependency would normally skip the
deploy too. `always()` lets the job evaluate its own condition, which then explicitly accepts
`'skipped'` as an acceptable result.

**This is the `succeededOrFailed()` versus `always()` distinction from Challenge 22 Q22**, in GitHub
form: only `always()` runs after a **skipped** dependency.

**Why B is wrong and worth being precise about:** the condition does **not** deploy on failure. It
accepts only `success` or `skipped`; a `failure` result fails both clauses and the deploy is blocked.

---

## Q10

What was inefficient about the "before" artifact configuration?

- A. It uploaded the entire workspace including `node_modules`
- B. It used too short a retention period
- C. It compressed the artifact
- D. It uploaded to the wrong path

### Answer: A

**In `challenge-35.md`:** lines **326–330** versus **345–350**.

```yaml
# BEFORE
      path: .  # Uploads everything including node_modules (500MB+)

# AFTER
      path: packages/api/dist/
      retention-days: 1          # short retention for intermediate artifacts
      compression-level: 6       # balance speed vs size
```

**500 MB versus about 5 MB** (line 361), and you pay for it **twice** — once uploading and once
downloading in the consuming job. For a pipeline running 20 times a day that is the difference between
minutes and seconds per run.

**`retention-days: 1` is the second saving.** Intermediate artifacts have no value after the run, and
storage is billed. Compare with Challenge 36's retention strategy.

**Why the others fail** — B, C and D describe the **fixed** version's settings.

---

## Q11

At $0.008 per Linux minute, what is Contoso's current monthly spend?

- A. ~$72
- B. ~$120
- C. ~$216
- D. ~$400

### Answer: C

**In `challenge-35.md`:** lines **402–412**.

```text
# Current spend: 20 runs/day * 45 min * $0.008 = $7.20/day = ~$216/month
# With optimization (target 15 min):
# 20 runs/day * 15 min * $0.008 = $2.40/day = ~$72/month
```

**The arithmetic is worth being able to do in your head:** runs per day × minutes × rate × working
days. 20 × 45 × $0.008 = $7.20/day; × 30 ≈ $216, or × 22 working days ≈ $158.

**The optimisation target ($100) is met by duration alone** — cutting 45 minutes to 15 takes the bill
from ~$216 to ~$72 with no change in runner strategy.

**Why the others fail** — A is the **post**-optimisation figure, B is the four-parallel-job Azure
Pipelines cost (line 508), D is not derived from the numbers.

---

## Q12

Where is the self-hosted break-even, and what does it imply after optimisation?

- A. ~17,500 minutes/month — after optimisation Contoso should stay hosted
- B. ~5,000 minutes/month — self-hosting is always cheaper
- C. ~50,000 minutes/month — self-hosting is never worth it
- D. There is no break-even

### Answer: A

**In `challenge-35.md`:** lines **407–412**.

```text
# Azure VM (Standard_D4s_v3): ~$140/month
# Break-even: ~$140/$0.008 = 17,500 minutes/month
# Current usage: 20 * 45 * 22 = 19,800 min/month (worth self-hosting)
# After optimization: 20 * 15 * 22 = 6,600 min/month (stay hosted)
```

**Optimisation changes the answer to the runner question.** Before, at 19,800 minutes, self-hosting
was justified. After, at 6,600, hosted is cheaper *and* has no maintenance.

**This is the exam-relevant insight:** fix the pipeline before buying infrastructure to run a slow
pipeline faster. And Challenge 21's ~$500/month maintenance figure is not even in this calculation —
including it pushes the break-even much higher.

---

## Q13

Which jobs should stay on hosted runners in a mixed strategy?

- A. Short jobs such as lint
- B. Long integration test jobs
- C. All jobs
- D. Only deployment jobs

### Answer: A

**In `challenge-35.md`:** lines **418–431**.

```yaml
  lint:
    runs-on: ubuntu-latest              # Short job, use hosted
  integration-tests:
    runs-on: [self-hosted, linux, x64]  # Long job, use self-hosted
```

**Short jobs are the worst fit for self-hosted runners**, which Challenge 21 Q48 showed from the other
direction: an ephemeral self-hosted runner spends more time provisioning than a 40-second lint takes
to run. Hosted runners bill per minute, so a 3-minute lint costs $0.024 — trivial.

**Long jobs are where self-hosted pays**, both in per-minute cost avoided and in warm caches that
persist between runs.

**Why the others fail** — B is the inversion, C ignores the trade-off, D has no basis.

---

## Q14

What is Azure Pipelines' free parallel job allowance, and the cost beyond it?

- A. 1 Microsoft-hosted parallel job with 1,800 minutes; $40/month per extra
- B. 5 free parallel jobs; $15/month per extra
- C. Unlimited free; pay per minute
- D. 1 free job; $120/month per extra

### Answer: A

**In `challenge-35.md`:** lines **502–508**.

```text
# Free tier: 1 Microsoft-hosted parallel job (1800 min/month)
# Additional: $40/month per parallel job (unlimited minutes)
# With 4 parallel jobs: $120/month but 15 min total duration
```

**Note "unlimited minutes"** — Azure Pipelines charges for **concurrency**, not consumption. That is
the opposite of GitHub Actions' per-minute model, and it changes the optimisation strategy: on Azure
Pipelines, running longer costs nothing extra; running *wider* does.

**$120 for three extra jobs**, since the first is free.

**Why the others fail** — B mixes in the $15 self-hosted licence from Challenge 21, C describes GitHub,
D miscounts the free job.

---

## Q15

Which Turborepo flag builds only packages affected by the last commit?

- A. `--filter='...[HEAD~1]'`
- B. `--force`
- C. `--parallel`
- D. `--no-cache`

### Answer: A

**In `challenge-35.md`:** lines **533–537**.

```yaml
      - run: npx turbo run build --filter='...[HEAD~1]'
        env:
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
          TURBO_TEAM: contoso
```

**The `...` prefix means "and everything that depends on them"** — so a change to `shared` builds
`shared`, `api` and `web`. Dropping it would build only the changed package and miss its consumers.

**`fetch-depth: 2` is required** (line 523): comparing against `HEAD~1` needs the previous commit, and
the default shallow clone has only one.

**`TURBO_TOKEN` enables the remote cache**, so a package already built on another machine is restored
rather than rebuilt.

**Why the others fail** — B forces a rebuild, C is about concurrency, D disables the optimisation.

---

## Q16

What is the `nx` equivalent for building only affected projects?

- A. `npx nx affected --target=build --base=HEAD~1 --head=HEAD`
- B. `npx nx run-many --target=build --all`
- C. `npx nx build`
- D. `npx nx reset`

### Answer: A

**In `challenge-35.md`:** lines **549–564**.

```bash
          AFFECTED=$(npx nx show projects --affected --base=HEAD~1 --head=HEAD)
          echo "projects=$AFFECTED" >> $GITHUB_OUTPUT
          if [ -z "$AFFECTED" ]; then
            echo "skip=true" >> $GITHUB_OUTPUT
          fi
```

**Note the empty-set guard.** When nothing is affected — a docs-only change — `nx affected` would
otherwise run with no projects, and the `skip=true` output lets the later steps be skipped cleanly
(lines 559, 563).

**Why the others fail**

- **B** — `--all` is the opposite: build everything regardless
- **C** — builds the default project
- **D** — clears the local cache

---

# Section B — Multiple answer

---

## Q17

Which **three** caches does the optimised pipeline use? (Choose three.)

- A. The npm download cache (`~/.npm`) via `setup-node`
- B. `node_modules` via `actions/cache`
- C. Docker build layers via buildx
- D. Test results
- E. Deployment artifacts
- F. Source code

### Answer: A, B, C

**In `challenge-35.md`:** lines **71**, **73–78**, **101–117**.

**Three caches, three different costs avoided:**

| Cache | Avoids |
|---|---|
| `~/.npm` | Downloading packages from the registry |
| `node_modules` | Unpacking and linking them |
| Docker layers | Rebuilding unchanged image layers |

**Why the others fail** — test results and artifacts are **outputs** (upload/download, not cache), and
source comes from `checkout`.

**The distinction the exam draws:** a **cache** is a performance optimisation that may miss and must
be reproducible without it. An **artifact** is a deliverable that must exist. Never cache something a
later job cannot rebuild.

---

## Q18

Which **two** are required to make sharded coverage reports meaningful? (Choose two.)

- A. Unique artifact names per shard
- B. A merge job downloading with `pattern:` and `merge-multiple: true`
- C. `fail-fast: true`
- D. A single shared artifact name
- E. `--runInBand`

### Answer: A, B

**In `challenge-35.md`:** lines **621–632** and **207–218**.

```yaml
    name: coverage-shard-${{ matrix.shard }}
...
  - uses: actions/download-artifact@v4
    with:
      pattern: coverage-shard-*
      merge-multiple: true
```

**Then the merge itself** (line 216): `npx nyc merge` combines the JSON fragments before reporting.

**Why the others fail**

- **C** — cancels the remaining shards on the first failure, so you get fewer results, not better ones
- **D** — Break & fix Exercise 2's bug: the last upload wins and you see 25%
- **E** — forces serial execution, undoing the sharding

---

## Q19

Which **two** correctly describe path-based conditional execution? (Choose two.)

- A. `dorny/paths-filter` outputs feed later jobs' `if` conditions
- B. `shared/**` must appear in the filters of every dependent package
- C. It replaces trigger-level path filters
- D. It costs nothing when nothing matches
- E. It requires `fetch-depth: 0`

### Answer: A, B

**In `challenge-35.md`:** lines **260–285** and **272–276**.

**Why the others fail**

- **C** — **complementary, not a replacement.** A trigger filter stops the workflow starting at all
  (free); a job condition runs inside a workflow that already started. Use the filter for "this
  workflow is irrelevant", the condition for "part of it is"
- **D** — you still pay for the `detect-changes` job — a runner, a checkout and the filter step —
  even when everything is skipped
- **E** — `paths-filter` handles the comparison itself; the deep history requirement belongs to the
  Turborepo and nx approaches (line 523)

---

## Q20

Which **two** reduce artifact transfer time between jobs? (Choose two.)

- A. Uploading only `packages/api/dist/` instead of the whole workspace
- B. `retention-days: 1` on intermediate artifacts
- C. `compression-level: 0`
- D. Uploading `node_modules` to every job
- E. Setting `path: .`

### Answer: A, B

**In `challenge-35.md`:** lines **345–350**.

```yaml
          name: api-dist
          path: packages/api/dist/
          retention-days: 1
          compression-level: 6
```

**A is the transfer saving** — 5 MB instead of 500 MB, paid twice per run.

**B is a storage saving** rather than a transfer one, and it is the honest answer here because
intermediate artifacts are worthless after the run. Retention costs money every day they persist.

**Why C is the interesting wrong answer:** `compression-level: 0` means **no compression**, so upload
is faster but the payload is larger. Level 6 (line 350) is the stated balance. Zero is only right when
the content is already compressed.

**Why D and E fail** — both are the "before" anti-pattern.

---

## Q21

Which **two** are true about Azure Pipelines parallel jobs? (Choose two.)

- A. The free tier includes 1 Microsoft-hosted parallel job with 1,800 minutes
- B. Extra parallel jobs cost $40/month each with unlimited minutes
- C. Azure Pipelines charges per minute like GitHub Actions
- D. `dependsOn: []` forces a job to run last
- E. `strategy: parallel: 4` requires a matrix definition

### Answer: A, B

**In `challenge-35.md`:** lines **502–506**.

**The billing models are genuinely different**, and the exam tests it: GitHub Actions charges for
**minutes consumed**; Azure Pipelines charges for **concurrency**. So on Azure Pipelines a longer
pipeline costs nothing extra, but a wider one does.

**Why the others fail**

- **C** — that is GitHub's model
- **D** — `dependsOn: []` makes a job start **immediately, in parallel** with the first stage (line
  482). The exact opposite
- **E** — `parallel: 4` needs no matrix; the platform supplies the position (Q7)

---

## Q22

Which **two** conditions must the deploy job tolerate for conditional execution to work? (Choose two.)

- A. An upstream job result of `success`
- B. An upstream job result of `skipped`
- C. An upstream job result of `failure`
- D. `docs_only == 'true'`
- E. A cancelled run

### Answer: A, B

**In `challenge-35.md`:** lines **311–315**.

```yaml
      (needs['test-api'].result == 'success' || needs['test-api'].result == 'skipped') &&
      (needs['test-web'].result == 'success' || needs['test-web'].result == 'skipped')
```

**Skipped must be acceptable** because a change touching only the API legitimately skips `test-web` —
and a skipped dependency is not a failed one.

**Why the others fail**

- **C** — a genuine failure must block. The condition excludes it
- **D** — line 313 explicitly requires `docs_only != 'true'`, so a docs-only change does **not** deploy
- **E** — cancellation should not deploy

---

## Q23

Which **two** monorepo tools build only affected packages? (Choose two.)

- A. Turborepo with `--filter='...[HEAD~1]'`
- B. `nx affected --base=HEAD~1 --head=HEAD`
- C. `npm ci --workspaces`
- D. `jest --shard`
- E. `docker build --no-cache`

### Answer: A, B

**In `challenge-35.md`:** lines **534** and **560**.

**Both need git history** — `fetch-depth: 2` at line 523 — because "affected" is computed by diffing
against a previous commit.

**Both also support a remote cache** (`TURBO_TOKEN` at line 536), which extends the idea: a package
someone else already built is **restored rather than rebuilt**, even on a fresh runner.

**Why the others fail**

- **C** — installs across all workspaces; the opposite of selective
- **D** — splits **tests** across machines, a different lever
- **E** — explicitly disables caching

---

# Section C — Repeated scenario

**Scenario:** Contoso's pipeline runs 45 minutes at ~$216/month across 20 runs a day. The target is
under 15 minutes and under $100/month, without reducing test coverage or reliability.

---

## Q24

**Proposed solution:** Cache `~/.npm` and `node_modules` keyed on `package-lock.json`, shard unit
tests across 4 parallel runners and merge coverage, add Docker layer caching, skip unaffected package
jobs with a paths filter, and upload only `dist/` between jobs.

Does this meet the goal? **Yes**

### Answer: Yes

**In `challenge-35.md`:** lines **67–117**, **166–218**, **258–316**, **345–350**.

**All four levers, applied where each fits:**

| Job | Before | Lever |
|---|---|---|
| Install (5 min) | | **Cache** — skipped entirely on a hit |
| Unit tests (12 min) | | **Parallelise** — 4 shards, ~3 min |
| Docker build (8 min) | | **Cache** — layers reused |
| Package tests | | **Skip** — unaffected packages |
| Artifact transfer | 500 MB | **Shrink** — 5 MB |

Coverage is preserved because the shards are **merged** (line 216) rather than discarded, which is the
"without reducing coverage" half of the requirement.

**And the cost follows the duration:** 20 × 15 × $0.008 ≈ $72/month (line 405).

---

## Q25

**Proposed solution:** Move all jobs to a self-hosted Azure VM to eliminate per-minute charges.

Does this meet the goal? **No**

### Answer: No

**Cost: arguably yes. Duration: no. And the cost claim is weaker than it looks.**

**The pipeline is still 45 minutes.** Self-hosting changes who pays for the minutes, not how many there
are. The stated target is *under 15 minutes*, and this proposal does nothing about it.

**The economics are also marginal.** At ~$140/month for the VM (line 408) against ~$216 hosted, the
saving is about $76 — before Challenge 21's ~$500/month maintenance figure, which reverses it
entirely.

**And it removes the incentive to fix anything.** With unlimited minutes, a slow pipeline costs
nothing visible, so it stays slow — while developers keep waiting 45 minutes.

**The correct order:** optimise first, then re-evaluate the runner question. After optimisation the
break-even calculation (line 412) says stay hosted.

---

## Q26

**Proposed solution:** Cache `~/.npm` keyed on `package.json`, shard unit tests across 4 runners, and
upload each shard's coverage as an artifact named `coverage`.

Does this meet the goal? **No**

### Answer: No

Both Break & fix defects, in one proposal.

**The cache key is wrong** (Exercise 1, line 578). `package.json` changes on every version bump, so
the key changes constantly and the cache **always misses**. The 5-minute install stays.

**The coverage artifacts collide** (Exercise 2, line 610). Four shards uploading to the name
`coverage` means the last one wins, and the report shows 25% — which breaks the "without reducing
coverage" requirement, or at least the ability to prove it.

**The sharding itself is correct**, which is what makes this realistic: the expensive structural work
is right and two small details silently undo it. The duration improves somewhat; the cache saving and
the coverage do not.

---

# Section D — Yes/No statement grid

---

## Q27 — caching

| # | Statement | Answer |
|---|---|---|
| 1 | Cache keys should hash the lock file, not `package.json` | **Yes** |
| 2 | `restore-keys` allows a partial cache hit | **Yes** |
| 3 | `setup-node` with `cache: "npm"` caches `node_modules` | **No** |
| 4 | A cache miss should fail the build | **No** |

**In `challenge-35.md`:** lines **594**, **595–596**, **71**, **81**.

Row 3 is the distinction from Q3: `setup-node` caches `~/.npm`, the **download** cache. Caching
`node_modules` needs a separate `actions/cache` step.

Row 4 is the defining property of a cache. A miss must simply cost time — line 81 falls back to
`npm ci`. Anything that *must* exist is an **artifact**, not a cache.

---

## Q28 — parallelism

| # | Statement | Answer |
|---|---|---|
| 1 | `--shard=1/4` splits tests across machines | **Yes** |
| 2 | `--maxWorkers` splits across cores on one machine | **Yes** |
| 3 | Sharded coverage reports must be merged | **Yes** |
| 4 | `fail-fast: true` is correct for a test matrix | **No** |

**In `challenge-35.md`:** lines **186**, **216**, **167**.

Rows 1 and 2 compose rather than compete: four runners each using several workers.

Row 4: with `fail-fast: true`, one failing shard cancels the rest, so you learn about one failure and
have to re-run to find the others. `fail-fast: false` (line 167) reports everything in one pass.

---

## Q29 — conditional execution

| # | Statement | Answer |
|---|---|---|
| 1 | A shared package must appear in every dependent's filter | **Yes** |
| 2 | A skipped job counts as a failure for downstream jobs | **No** |
| 3 | `always()` is needed when a dependency may be skipped | **Yes** |
| 4 | Trigger path filters and job conditions are interchangeable | **No** |

**In `challenge-35.md`:** lines **272–276**, **311–315**.

Row 2 is precise: `skipped` is its own result, distinct from `success` and `failure`. But **by default
a skipped dependency skips the downstream job**, which is why row 3's `always()` plus an explicit
`== 'skipped'` check is required.

Row 4: a trigger filter costs **nothing** when it does not match; a job condition costs a runner and a
checkout for the detection job.

---

## Q30 — cost

| # | Statement | Answer |
|---|---|---|
| 1 | GitHub Actions charges per minute consumed | **Yes** |
| 2 | Azure Pipelines charges per parallel job, not per minute | **Yes** |
| 3 | Self-hosting is cheaper at any usage level | **No** |
| 4 | Reducing duration reduces GitHub Actions cost proportionally | **Yes** |

**In `challenge-35.md`:** lines **398**, **502–506**, **410–412**.

**Rows 1 and 2 are the difference that changes strategy.** On GitHub, shortening the pipeline saves
money directly. On Azure Pipelines, shortening it saves *time* while cost is driven by how many
parallel jobs you buy — so sharding into four shards costs $120/month there and nothing extra on
GitHub.

Row 3: break-even is ~17,500 minutes/month (line 410), and Contoso lands **below** it after
optimisation.

---

# Section E — Drag and drop

---

## Q31

Match each optimisation lever to the job it fixes in Contoso's pipeline.

| Job (before) | Lever |
|---|---|
| Install deps — 5 min | **Cache** `~/.npm` and `node_modules` |
| Unit tests — 12 min | **Parallelise** into 4 shards |
| Integration tests — 15 min | **Skip** when the package is unaffected |
| Docker build — 8 min | **Cache** build layers |
| Artifact transfer | **Shrink** to `dist/` only |

**In `challenge-35.md`:** the breakdown at line **23**, with the fixes at **67–117**, **166–188**,
**283–298**, **101–117**, **345–350**.

**Match the lever to the reason the job is slow.** Install is slow because it repeats identical work →
cache. Unit tests are slow because there is a lot of it → parallelise. Integration tests are slow and
often irrelevant → skip. Choosing the wrong lever — parallelising an install, or caching a test run —
is how the distractors in this section work.

---

## Q32

Match each cache to what it stores.

| Cache | Stores |
|---|---|
| `setup-node` with `cache: "npm"` | **`~/.npm` — downloaded packages** |
| `actions/cache` on `node_modules` | **The installed dependency tree** |
| buildx `type=local` | **Docker layers on the runner's disk** |
| buildx `type=registry` | **Docker layers in a container registry** |
| Turborepo remote cache | **Built package outputs, shared across machines** |

**In `challenge-35.md`:** lines **71**, **77**, **116**, **153**, **536**.

**Local versus registry cache is a real decision.** Local is faster to read but scoped to whatever
restored it; registry is shared by every runner and survives cache eviction, at the cost of a network
round-trip.

---

## Q33

Arrange the optimised pipeline's jobs into execution waves.

```yaml
  install:                              # no needs
  detect-changes:                       # no needs
  lint:            needs: install
  test-unit:       needs: install       # matrix: shard [1,2,3,4]
  docker-build:    needs: [test-unit, lint]
  merge-coverage:  needs: test-unit
  deploy:          needs: [test-api, test-web, detect-changes]
```

### Answer — four waves

| Wave | Jobs |
|---|---|
| 1 | `install`, `detect-changes` |
| 2 | `lint`, `test-unit` (×4 shards) |
| 3 | `docker-build`, `merge-coverage` |
| 4 | `deploy` |

**In `challenge-35.md`:** lines **62**, **94**, **165**, **198**, **258**, **310**.

**`install` and `detect-changes` share wave 1** because neither declares `needs` — same upstream (none)
means parallel, the rule from Challenge 22.

**Wave 3 has two jobs running side by side:** `docker-build` needs both `test-unit` and `lint`;
`merge-coverage` needs only `test-unit`. They are independent of each other.

**Total duration is the critical path**, not the sum — which is why four 3-minute shards cost 3
minutes, not 12.

---

## Q34

Match each cost model to its platform.

| Model | Platform |
|---|---|
| $0.008 per Linux minute consumed | **GitHub Actions** |
| 1 free parallel job, $40/month per extra, unlimited minutes | **Azure Pipelines** |
| ~$140/month VM, unlimited minutes, you maintain it | **Self-hosted** |
| ~17,500 min/month break-even against a self-hosted VM | **GitHub Actions vs self-hosted** |

**In `challenge-35.md`:** lines **398**, **502–506**, **408**, **410**.

**The strategic consequence:** on GitHub Actions, **shorter** saves money. On Azure Pipelines,
**narrower** saves money. Sharding into four costs $120/month on Azure Pipelines and nothing extra on
GitHub — so the same optimisation has opposite cost implications depending on the platform.

---

## Q35

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Cache always misses | **Key hashes `package.json` instead of the lock file** |
| Coverage report shows 25% | **All shards upload to the same artifact name** |
| Deploy is skipped when only the API changed | **Missing `always()` and a `skipped` check** |
| Turborepo rebuilds everything | **`fetch-depth` too shallow to diff `HEAD~1`** |
| Artifact upload takes minutes | **`path: .` including `node_modules`** |

**In `challenge-35.md`:** lines **578**, **610**, **311–315**, **523**, **330**.

**The fourth row is the quiet one.** With a shallow clone there is no `HEAD~1` to compare against, so
"affected" resolves to everything — the optimisation silently reverts to a full build and nothing
errors.

---

# Section F — Hot area

---

## Q36

```yaml
      - uses: actions/cache@v4
        id: cache-modules
        with:
          path: node_modules
          key: ${{ runner.os }}-modules-${{ hashFiles('[BLANK 1]') }}
          restore-keys: |
            ${{ runner.os }}-modules-

      - name: Install dependencies
        if: steps['cache-modules'].outputs['cache-hit'] != '[BLANK 2]'
        run: npm ci
```

- **BLANK 1:** `package-lock.json` / `package.json` / `node_modules/**` / `*.js`
- **BLANK 2:** `true` / `false` / `1` / `yes`

### Answer: `package-lock.json`, `true`

**In `challenge-35.md`:** lines **78** and **81**.

The lock file changes only when dependencies change. And `cache-hit` is a **string**, so the comparison
is against `'true'` — the same string-versus-boolean trap as Challenge 22 Q10.

---

## Q37

```yaml
    strategy:
      [BLANK 1]: false
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - run: npx jest --ci --[BLANK 2]=${{ matrix.shard }}/4
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-[BLANK 3]
```

- **BLANK 1:** `fail-fast` / `continue-on-error` / `max-parallel` / `strict`
- **BLANK 2:** `shard` / `maxWorkers` / `split` / `partition`
- **BLANK 3:** `${{ matrix.shard }}` / `latest` / `all` / *(omit)*

### Answer: `fail-fast`, `shard`, `${{ matrix.shard }}`

**In `challenge-35.md`:** lines **167**, **186**, **193**.

`fail-fast: false` reports every shard's failures in one run. BLANK 3 is Break & fix Exercise 2 — a
fixed name means the last shard overwrites the rest and you see 25% coverage.

---

## Q38

```yaml
      - uses: docker/build-push-action@v5
        with:
          cache-from: type=[BLANK 1],src=/home/runner/.docker-cache
          cache-to: type=[BLANK 1],dest=/home/runner/.docker-cache,mode=[BLANK 2]
```

- **BLANK 1:** `local` / `gha` / `registry` / `inline`
- **BLANK 2:** `max` / `min` / `full` / `all`

### Answer: `local`, `max`

**In `challenge-35.md`:** lines **116–117**.

`src`/`dest` paths indicate `type=local` — `gha` and `registry` use different parameters (`ref` for
registry, line 153). `mode=max` caches **intermediate** layers, which is what makes a multi-stage build
benefit.

---

## Q39

```yaml
  deploy:
    needs: [test-api, test-web, detect-changes]
    if: |
      [BLANK 1] &&
      needs['detect-changes'].outputs.docs_only != 'true' &&
      (needs['test-api'].result == 'success' || needs['test-api'].result == '[BLANK 2]')
```

- **BLANK 1:** `always()` / `success()` / `failure()` / `cancelled()`
- **BLANK 2:** `skipped` / `failure` / `cancelled` / `neutral`

### Answer: `always()`, `skipped`

**In `challenge-35.md`:** lines **312** and **314**.

Without `always()`, a skipped dependency skips the deploy. Accepting `'skipped'` is what allows a
change touching only one package to still deploy — while `'failure'` remains excluded.

---

## Q40

```yaml
      - uses: actions/upload-artifact@v4
        with:
          name: api-dist
          path: [BLANK 1]
          retention-days: [BLANK 2]
          compression-level: [BLANK 3]
```

Requirement: pass only the deployable output to the next job, and do not pay to store it.

- **BLANK 1:** `packages/api/dist/` / `.` / `node_modules/` / `packages/api/`
- **BLANK 2:** `1` / `90` / `30` / `0`
- **BLANK 3:** `6` / `0` / `9` / `12`

### Answer: `packages/api/dist/`, `1`, `6`

**In `challenge-35.md`:** lines **348–350**.

`path: .` is the 500 MB anti-pattern. Level 0 means **no compression** — faster to write, more to
transfer; level 9 is slowest. Six is the stated balance, and 12 is not a valid value.

---

## Q41

```yaml
      - uses: actions/checkout@v4
        with:
          fetch-depth: [BLANK 1]
      - run: npx turbo run build --filter='[BLANK 2]'
        env:
          [BLANK 3]: ${{ secrets.TURBO_TOKEN }}
```

- **BLANK 1:** `2` / `0` / `1` / `10`
- **BLANK 2:** `...[HEAD~1]` / `[HEAD]` / `*` / `all`
- **BLANK 3:** `TURBO_TOKEN` / `NPM_TOKEN` / `GITHUB_TOKEN` / `TURBO_CACHE`

### Answer: `2`, `...[HEAD~1]`, `TURBO_TOKEN`

**In `challenge-35.md`:** lines **523**, **534**, **536**.

Depth **2** is the minimum for a `HEAD~1` comparison; `0` (full history) works and is slower. The `...`
prefix includes **dependents**, so a change to `shared` also builds `api` and `web`.

---

# Section G — Case study

## Case study: Contoso pipeline optimisation

### Background

`contoso-platform` is a Node.js monorepo with three packages: `api`, `web` and `shared`.

**Current performance:** 45 minutes average, **20 runs per day**, ~$200/month.
**Job breakdown:** install 5 min, lint 3 min, unit tests 12 min, integration tests 15 min, Docker
build 8 min, deploy 2 min.

**Target:** under **15 minutes** and under **$100/month**, with no loss of test coverage or
reliability.

### Requirements

**Speed**

- Repeated work must not be redone on every run
- Long test suites must run concurrently
- Work unrelated to the change must be skipped

**Cost**

- Monthly spend must fall below $100
- The runner strategy must be justified by measured usage

**Quality**

- Coverage must remain complete and reportable
- Genuine failures must still fail the build

---

## Q42

Which **two** eliminate the 5-minute install on most runs? (Choose two.)

- A. `setup-node` with `cache: "npm"`
- B. `actions/cache` on `node_modules` keyed on `package-lock.json`
- C. `npm install` instead of `npm ci`
- D. Committing `node_modules` to the repository
- E. `--prefer-offline` alone

### Answer: A, B

**In `challenge-35.md`:** lines **67–82**.

**They stack.** The npm cache avoids the download; the `node_modules` cache avoids the install
entirely, because the `if:` at line 81 skips `npm ci` on a hit.

**Why the others fail**

- **C** — `npm install` may resolve differently from the lock file, so builds stop being reproducible.
  `npm ci` is the CI-correct command
- **D** — hundreds of megabytes in git, platform-specific binaries, and merge conflicts on every
  dependency change
- **E** — helps only when there is already a populated cache, which is what A provides

---

## Q43

Which configuration cuts the 12-minute unit test job to roughly 3 minutes?

- A. Four matrix shards running `jest --shard=N/4`, with merged coverage
- B. `--maxWorkers=4` on a single runner
- C. Running only a quarter of the tests
- D. `--bail` to stop on first failure

### Answer: A

**In `challenge-35.md`:** lines **166–218**.

**Why the others fail**

- **B** — **the closest wrong answer.** More workers on one runner does help, and it is bounded by that
  machine's cores and memory. Four separate runners is four machines' worth of capacity, and hosted
  runners are small
- **C** — reduces coverage, which the requirements forbid
- **D** — faster only when something fails, and it hides the remaining failures

**Coverage is preserved by the merge job** (line 216), which is the part that makes A satisfy the
quality requirement rather than just the speed one.

---

## Q44

Which configuration skips the 15-minute integration tests when they are irrelevant?

- A. `dorny/paths-filter` outputs gating the job with `if:`
- B. `continue-on-error: true` on the integration job
- C. Running integration tests only nightly
- D. Deleting the integration tests

### Answer: A

**In `challenge-35.md`:** lines **258–298**.

**Why the others fail**

- **B** — still runs for 15 minutes; it just ignores the result, which also breaks the reliability
  requirement
- **C** — **a real strategy and a different trade-off.** Moving integration tests to a nightly run
  makes CI fast and delays failure discovery by up to a day. Acceptable for some teams; it is not
  "skip when irrelevant"
- **D** — loses the coverage

**And remember `shared/**` must be in the filter** (lines 273, 276), or a change to the shared package
skips the tests most likely to catch its impact.

---

## Q45

After optimisation the pipeline runs 15 minutes. What is the monthly cost, and what runner strategy
does that imply?

- A. ~$72/month — stay on hosted runners
- B. ~$216/month — move to self-hosted
- C. ~$140/month — self-host
- D. ~$120/month — buy parallel jobs

### Answer: A

**In `challenge-35.md`:** lines **404–412**.

```text
# 20 runs/day * 15 min * $0.008 = $2.40/day = ~$72/month
# After optimization: 20 * 15 * 22 = 6,600 min/month (stay hosted)
```

**6,600 minutes is well under the ~17,500-minute break-even** (line 410), so the VM would cost more
than the minutes it replaces — before any maintenance time.

**The sequence matters more than the number.** Optimising first changed the correct answer to the
runner question. Self-hosting a 45-minute pipeline would have locked in the waste.

---

## Q46

Which **two** preserve coverage and reliability while sharding? (Choose two.)

- A. Unique artifact names per shard, merged in a later job
- B. `fail-fast: false` so every shard reports
- C. `continue-on-error: true` on the test jobs
- D. Reporting coverage from shard 1 only
- E. `--bail` in each shard

### Answer: A, B

**In `challenge-35.md`:** lines **621–638** and **167**.

**A protects coverage; B protects reliability reporting.** Without A you see 25% (Break & fix Exercise
2). Without B, one failing shard cancels the others and you learn about one failure per run.

**Why the others fail**

- **C** — the build passes with failing tests. Challenge 34's dishonesty problem
- **D** — a quarter of the truth presented as the whole
- **E** — stops early, so the remaining tests in that shard never run

---

## Q47

Six months on, the pipeline runs 14 minutes on average — but the `detect-changes` job now takes 90
seconds and runs on every push, including docs-only changes.

What should Contoso change?

- A. Add a trigger-level `paths-ignore` for documentation so the workflow does not start at all
- B. Remove the paths filter entirely
- C. Increase the runner size
- D. Cache the `detect-changes` job

### Answer: A

**In `challenge-35.md`:** the job-level filtering at lines **258–281**; the trigger form is Challenge 22
lines 92–95.

**Trigger filters are free; job conditions are not.** `detect-changes` allocates a runner, checks out
the repository and runs the filter — 90 seconds and a billed minute — before concluding there is
nothing to do.

```yaml
on:
  push:
    branches: [main]
    paths:
      - '!**/*.md'
      - '!docs/**'
```

**Filter as early as possible.** This is Challenge 22 Q48 restated: use a trigger filter when the whole
workflow is irrelevant, and a job condition when only part of it is. Contoso needs **both** — the
trigger for docs-only pushes, the job filter for which package changed.

**Why the others fail** — B loses the per-package skipping, C makes a mostly-idle job faster, D cannot
cache a decision that depends on the current diff.

---

## Q48

A year later the pipeline is 12 minutes and costs $65/month. The team wants it under 5 minutes and
proposes buying four Azure Pipelines parallel jobs at $120/month.

What is wrong with this reasoning?

- A. It would raise cost above the $100 target, and duration is bounded by the critical path, not
  parallelism
- B. Parallel jobs are free
- C. Azure Pipelines cannot run jobs in parallel
- D. The pipeline is already fast enough

### Answer: A

**In `challenge-35.md`:** lines **502–508**, with the wave structure from Q33.

**Two problems, and the second is the more interesting one.**

**Cost.** $120/month for parallel jobs exceeds the $100 target on its own, before any other spend.
Note this is also a **platform confusion**: they are on GitHub Actions, where sharding costs nothing
extra — parallel-job licences are the Azure Pipelines model.

**Physics.** Total duration is the **critical path**. If install takes 2 minutes, the longest shard 3,
Docker build 4 and deploy 2 — sequential dependencies totalling 11 — then adding runners cannot go
below that. More parallelism only helps where work is genuinely **independent and currently
serialised**.

**What would actually help:** shorten the critical path. A smaller Docker context, a faster base image,
more aggressive layer caching, or removing a dependency edge so two jobs can share a wave.

**Why D is tempting and wrong to state that way:** 12 minutes may well be fine, and that is a judgement
for the team. The exam answer is the technical one — the proposal fails its own cost target and would
not achieve the duration goal either.

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Cache keyed on `package.json`** | Q1, Q26, Q36 | Key on the **lock** file; add `restore-keys` |
| **Shards sharing an artifact name** | Q2, Q18, Q26, Q37 | Unique name per shard, then merge |
| **`setup-node` cache assumed to cover `node_modules`** | Q3, Q27 | It caches `~/.npm` only |
| **`cache-hit` compared as a boolean** | Q4, Q36 | It is the string `'true'` |
| **`fail-fast: true` on a test matrix** | Q28, Q46 | One failure cancels the rest |
| **Missing `always()` with skipped dependencies** | Q9, Q29, Q39 | Accept `'skipped'`, exclude `'failure'` |
| **`path: .` on an artifact** | Q10, Q20, Q40 | Upload only the deployable output |
| **Self-hosting before optimising** | Q25, Q45 | Optimisation changes the break-even |
| **GitHub and Azure cost models mixed** | Q14, Q21, Q30, Q48 | Per minute vs per parallel job |
| **Job condition where a trigger filter belongs** | Q19, Q47 | Trigger filters are free; jobs cost a runner |
| **Shallow clone with `affected` tooling** | Q15, Q35 | `fetch-depth: 2` minimum |
| **Parallelism assumed to beat the critical path** | Q48 | Duration is bounded by the longest chain |

---

# The blocks to memorise

Line numbers are in `challenge-35.md`.

```yaml
# 1. Two caches, one skip  (lines 67-82)
      - uses: actions/setup-node@v4
        with:
          cache: "npm"                                    # caches ~/.npm
      - uses: actions/cache@v4
        id: cache-modules
        with:
          path: node_modules
          key: ${{ runner.os }}-modules-${{ hashFiles('package-lock.json') }}
      - if: steps['cache-modules'].outputs['cache-hit'] != 'true'
        run: npm ci

# 2. Shard and merge  (lines 166-218)
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]
      # npx jest --shard=${{ matrix.shard }}/4
      # upload as coverage-${{ matrix.shard }}, then:
      #   download pattern: coverage-*  merge-multiple: true
      #   npx nyc merge

# 3. Azure Pipelines equivalent  (lines 228-242)
    strategy:
      parallel: 4
      # --shard=$(System.JobPositionInPhase)/$(System.TotalJobsInPhase)

# 4. Docker layer cache  (lines 116-117 local, 153-154 registry)
          cache-from: type=local,src=/home/runner/.docker-cache
          cache-to: type=local,dest=/home/runner/.docker-cache,mode=max

# 5. Conditional jobs  (lines 258-316)
  detect-changes:  # dorny/paths-filter, shared/** in EVERY dependent filter
  test-api:
    if: needs['detect-changes'].outputs.api == 'true'
  deploy:
    if: always() && needs['detect-changes'].outputs.docs_only != 'true' &&
        (needs['test-api'].result == 'success' || needs['test-api'].result == 'skipped')

# 6. Artifact discipline  (lines 345-350)
          path: packages/api/dist/     # not '.'
          retention-days: 1
          compression-level: 6
```

```text
# 7. Cost models
GitHub Actions    $0.008/min Linux, $0.016 Windows, $0.08 macOS  -> shorter = cheaper
Azure Pipelines   1 free parallel job (1800 min), $40/mo each     -> narrower = cheaper
Self-hosted VM    ~$140/month, unlimited minutes                  -> you maintain it
Break-even        ~$140 / $0.008 = ~17,500 min/month

# 8. Contoso's numbers  (lines 402-412)
Before   20 runs x 45 min x 22 days = 19,800 min/month = ~$216   (self-hosting justified)
After    20 runs x 15 min x 22 days =  6,600 min/month = ~$72    (stay hosted)
```

**The four levers:** cache what does not change, parallelise what is independent, skip what is
unaffected, shrink what moves.

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 35 is exam-ready. Move to Challenge 36 |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 1 and 2, then retake this |
| Below 30 | Redo the challenge, working the cost arithmetic by hand first |

Record your result in `AZ-400-Learning-Log.md` under Challenge 35.

:::tip The one thing

**Optimise before you buy capacity.**

At 45 minutes, self-hosting looked justified. At 15 minutes it is not. The same is true of parallel
jobs, bigger runners and every other purchase — fix the work first, then measure again.

:::
