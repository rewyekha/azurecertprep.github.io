---
sidebar_position: 4.5
toc_max_heading_level: 2
title: "Challenge 22: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 22 — AZ-400 exam questions

**48 questions** built only from what Challenge 22 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-22.md`**.

| Section | Shape | Questions |
|---|---|---|
| A | Single answer | 1–16 |
| B | Multiple answer (choose two / three) | 17–23 |
| C | Repeated scenario — "Does this meet the goal?" | 24–26 |
| D | Yes/No statement grid | 27–30 |
| E | Drag and drop | 31–35 |
| F | Hot area — complete the YAML | 36–41 |
| G | Case study | 42–48 |

The **trap index**, the **blocks to memorise**, and **scoring** are at the end.

:::danger Your repeated mistake lives here

You have answered the **`dependsOn` parallelism** question wrong in more than one mock. Chaining
stages that should run side by side serialises them and destroys the parallelism the requirement
asked for. See **Q13, Q24 and Q31**. Do not skip them.

:::

---

# Section A — Single answer

---

## Q1

A workflow uses both `paths` and `paths-ignore` under the same `push` trigger and never fires.

Why?

- A. `paths-ignore` must be listed before `paths`
- B. Path filters require `fetch-depth: 0` on checkout
- C. `paths` and `paths-ignore` are mutually exclusive
- D. Path filters apply only to `pull_request` events

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-22.md`:** Break & fix Exercise 1, lines **774–801**.

```yaml
on:
  push:
    branches: [main]
    paths:
      - "backend/**"
    paths-ignore:        # line 782 - cannot use BOTH
      - "backend/docs/**"
```

**The fix (line 798)** — use `paths` with `!` negation:

```yaml
    paths:
      - "backend/**"
      - "!backend/docs/**"
```

The same pattern appears in the real backend workflow at lines **93–95**.

**Why the others fail**

- **A** — order changes nothing. The keys cannot coexist at all
- **B** — `fetch-depth: 0` matters when *your script* inspects git history (line 598), not for
  platform-level filtering
- **D** — path filters work on both `push` and `pull_request` (lines 92 and 100)

**Azure Pipelines is different (lines 164–170):** it uses `paths: include:` **and** `exclude:`
together, which is perfectly legal there. Two platforms, opposite rules — a favourite exam pairing.

</details>

---

## Q2

Which Azure Pipelines schedule setting causes a scheduled run even when no code has changed?

- A. `batch: true`
- B. `enabled: true`
- C. `trigger: none`
- D. `always: true`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-22.md`:** lines **289** and **295**.

```yaml
schedules:
  - cron: "0 2 * * 1-5"
    displayName: "Weekday nightly build"
    always: false      # line 289 - only if there are changes since the last run
  - cron: "0 4 * * 0"
    displayName: "Sunday full regression"
    always: true       # line 295 - runs regardless
```

**Why the others fail**

- **A** — `batch` applies to CI triggers: queue changes while a run is in progress and batch them
- **B** — not a schedule property
- **C** — `trigger: none` (line 281) **disables CI** so only the schedule fires. Useful, but it does
  not change the no-changes behaviour

**Why the difference matters:** a nightly build of unchanged code is wasted agent time, so
`always: false` is a cost control. A weekly full regression should run anyway, because it exercises
things a build does not — hence `always: true`.

**GitHub Actions has no `always` equivalent.** Its `schedule` triggers always run, but GitHub
**disables scheduled workflows in repositories with 60 days of no activity**.

</details>

---

## Q3

An Azure Pipelines schedule never executes. The schedule targets `develop`, and the pipeline's
`trigger` includes only `main`.

What is the cause?

- A. The schedule branch is not included in the pipeline's trigger branches
- B. The cron expression `0 2 * * *` is not valid in Azure Pipelines
- C. Schedules only fire when `always: true` is set on the entry
- D. Scheduled pipelines must set `trigger: none` to disable CI first

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-22.md`:** Break & fix Exercise 3, lines **852–887**.

```yaml
trigger:
  branches:
    include:
      - main           # only main

schedules:
  - cron: "0 2 * * *"
    branches:
      include:
        - develop      # line 863 - branch the pipeline does not track
    always: false
```

**The fix (lines 874–886):** add `develop` to the trigger branches, or point the schedule at `main`,
and consider `always: true`.

**Why the others fail**

- **B** — `0 2 * * *` is valid: 02:00 daily
- **C** — `always: true` would help *if* the branch were correct, by removing the "no changes"
  condition. But with the wrong branch there is nothing to run
- **D** — `trigger: none` is optional. It stops CI runs; it does not enable schedules

**The general lesson:** a schedule can only run on a branch the pipeline actually knows about.

</details>

---

## Q4

Which GitHub Actions trigger starts a workflow when another workflow finishes?

- A. `workflow_call`
- B. `repository_dispatch`
- C. `workflow_run`
- D. `workflow_dispatch`

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-22.md`:** lines **328–332**.

```yaml
on:
  workflow_run:
    workflows: ["Backend CI", "Frontend CI"]
    types: [completed]
    branches: [main]
```

**Critical detail (line 336):** `types: [completed]` fires whether the upstream **succeeded or
failed**. You must check the conclusion yourself:

```yaml
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
```

Omit that `if` and you deploy after a failed build. That is a classic exam trap and a real outage.

**Why the others fail**

- **A** — `workflow_call` makes a workflow **callable** by another. The caller decides when
- **B** — `repository_dispatch` fires from an external API call
- **D** — `workflow_dispatch` is the manual button (line 687)

</details>

---

## Q5

In a `workflow_run`-triggered workflow, which value identifies the commit the upstream workflow
tested?

- A. `github.sha` of the current run
- B. `github.ref` of the current run
- C. `github.event.head_commit.id`
- D. `github.event.workflow_run.head_sha`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-22.md`:** lines **341** and **343**.

```yaml
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.workflow_run.head_sha }}
```

**Why this matters more than it looks.** A `workflow_run` workflow always runs from the **default
branch**, so `github.sha` is the tip of `main` — not necessarily the commit the upstream workflow
actually tested. Without the explicit `ref`, you can deploy a *different* commit from the one that
passed CI.

**Why the others fail**

- **A** — the default-branch commit, as explained
- **B** — the ref of *this* run, again the default branch
- **C** — `head_commit` belongs to a `push` event payload, not `workflow_run`

**Other useful fields:** `workflow_run.name`, `workflow_run.head_branch`, `workflow_run.conclusion`
(lines 346–353).

</details>

---

## Q6

In Azure Pipelines, how do you trigger a pipeline when another pipeline completes?

- A. A `dependsOn` entry referencing the other pipeline's name
- B. A `resources: pipelines:` entry with a `trigger` block
- C. A `schedules` entry synchronised to the other pipeline
- D. A `workflow_run` trigger naming the other pipeline

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-22.md`:** lines **358–373**.

```yaml
resources:
  pipelines:
    - pipeline: backendCI
      source: "Backend-CI-Pipeline"
      trigger:
        branches:
          include:
            - main
        stages:
          - Test              # line 367 - fire when the Test STAGE completes
```

`stages:` is worth noticing — you can trigger on a **stage** completing, not only the whole pipeline.

**Why the others fail**

- **A** — **this is your repeated mistake.** `dependsOn` works only **inside one pipeline**, between
  its own stages or jobs. It cannot reference another pipeline
- **C** — a schedule runs on a clock, not on another pipeline's completion
- **D** — `workflow_run` is GitHub Actions

**The alias also gives you artifacts (lines 382–385):**

```yaml
      - download: backendCI
        artifact: drop
```

</details>

---

## Q7

Which matrix option lets every leg finish even when one fails?

- A. `fail-fast: false`
- B. `continue-on-error: true`
- C. `max-parallel: 1`
- D. `if: always()`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-22.md`:** line **531**.

```yaml
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [18, 20, 22]
```

`fail-fast` defaults to **true**, which cancels remaining legs on the first failure. For a
cross-platform matrix you usually want `false` — otherwise a Windows failure hides whether macOS also
broke.

**Why the others fail**

- **B** — `continue-on-error: true` makes the failure **not fail the job at all**. The leg shows
  green. That hides the problem instead of reporting it
- **C** — `max-parallel: 1` runs legs one at a time. With `fail-fast` still true, the first failure
  still cancels the rest — and now the run is slower too
- **D** — `if: always()` applies to steps and jobs, not to matrix cancellation

**The distinction to keep:** `fail-fast: false` = *let them all run and report honestly*.
`continue-on-error` = *pretend it passed*.

</details>

---

## Q8

A matrix defines `os: [ubuntu-latest, windows-latest, macos-latest]` and
`node-version: [18, 20, 22]`, with one `exclude` entry.

How many jobs run?

- A. 6
- B. 8
- C. 9
- D. 12

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-22.md`:** lines **532–541**.

```yaml
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]     # 3
        node-version: [18, 20, 22]                            # 3
        exclude:
          - os: macos-latest
            node-version: 18                                  # removes 1
        include:
          - os: ubuntu-latest
            node-version: 20
            coverage: true                                    # adds a variable, not a job
```

3 × 3 = 9, minus 1 excluded = **8**.

**The subtle part is `include`.** When an `include` entry matches an **existing** combination, it
**adds variables to it** rather than creating a new job. Here `ubuntu-latest` + Node 20 already
exists, so it gains `coverage: true` — used at line 550:

```yaml
      - name: Upload coverage
        if: matrix.coverage
```

If an `include` entry matched nothing, it **would** add a job. That difference is exam material.

</details>

---

## Q9

Which Azure Pipelines matrix setting limits how many legs run at once?

- A. `fail-fast`
- B. `parallel`
- C. `batch`
- D. `maxParallel`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-22.md`:** line **572**.

```yaml
    strategy:
      matrix:
        linux_node20:
          vmImage: "ubuntu-latest"
          nodeVersion: "20.x"
        linux_node22: { ... }
        windows_node20: { ... }
      maxParallel: 3
    pool:
      vmImage: $(vmImage)       # line 574 - each leg supplies its own image
```

Note the structural difference from GitHub: Azure Pipelines matrix legs are **named** and each
defines its own variables, rather than being a cross-product of lists.

**Why the others fail**

- **A** — `fail-fast` is GitHub Actions. Azure Pipelines cancels via `cancelTimeoutInMinutes` and
  job conditions instead
- **B** — `parallel` is a **deployment** strategy for VM resources
- **C** — `batch` is a CI trigger setting

**Practical link:** with one free parallel job (Challenge 21, line 47), `maxParallel: 3` would still
queue. Matrix width is limited by purchased parallelism.

</details>

---

## Q10

A job must run only when a previous job set an output to `true`.

Which condition is correct?

- A. `if: steps.changes.outputs.frontend == 'true'`
- B. `if: needs.build.result == 'true'`
- C. `if: needs.build.outputs.changed_frontend == 'true'`
- D. `if: ${{ env.changed_frontend == 'true' }}`

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-22.md`:** lines **592–615**.

```yaml
  build:
    outputs:
      changed_frontend: ${{ steps.changes.outputs.frontend }}
    steps:
      - name: Detect changes
        id: changes
        run: echo "frontend=true" >> $GITHUB_OUTPUT

  deploy-frontend:
    needs: build
    if: needs.build.outputs.changed_frontend == 'true'
```

Note the quotes: `$GITHUB_OUTPUT` values are **strings**. `true` and `'true'` are not the same thing.

**Why the others fail**

- **A** — the `steps` context only exists **inside the job that ran the step**. From another job it
  is empty
- **B** — `needs.<job>.result` holds `success`, `failure`, `cancelled` or `skipped`. It never holds
  `true`
- **D** — `env` does not cross job boundaries

**Why this pattern exists:** path filters decide whether the *workflow* runs. This decides which
*jobs* run inside it — the monorepo problem Challenge 35 optimises further.

</details>

---

## Q11

Which expression detects that any needed job failed?

- A. `contains(needs.*.result, 'failure')`
- B. `needs.result == 'failure'` on the job
- C. `always() && failed()` on the step
- D. `failure()` on the step

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-22.md`:** lines **628–635**.

```yaml
  notify:
    needs: [deploy-frontend, deploy-backend]
    if: always()
    steps:
      - name: Send notification
        if: contains(needs.*.result, 'failure')
        run: echo "One or more deployments failed"
```

`needs.*.result` is an **array** of every needed job's result, and `contains()` searches it.

**Note the two-level condition.** The job says `if: always()` so it runs no matter what; the step
then asks whether anything failed. Without `always()`, the notify job would be skipped when an
upstream job failed — exactly when you need the notification most.

**Why the others fail**

- **B** — `needs.result` does not exist. Results are per-job
- **C** — `failed()` is **Azure Pipelines** syntax. GitHub Actions uses `failure()`
- **D** — `failure()` is true when **any previous job in the dependency chain** failed. It works in
  simple cases but cannot tell you *which*, and combined with `always()` at job level the intent gets
  muddy

</details>

---

## Q12

An Azure Pipelines stage must run even if earlier stages failed or were skipped.

Which condition achieves this?

- A. `condition: succeededOrFailed()`
- B. `condition: not(canceled())`
- C. `condition: eq(dependencies.status, 'any')`
- D. `condition: always()`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-22.md`:** lines **665–675**.

```yaml
  - stage: Notify
    dependsOn:
      - DeployStaging
      - DeployProduction
    condition: always()          # line 670 - even if skipped or failed
    jobs:
      - job: NotifyTeam
        steps:
          - script: echo "Pipeline completed"
            condition: succeededOrFailed()    # line 675
```

**The distinction between the two lines is the exam point:**

| Condition | Runs when |
|---|---|
| `always()` | Always — including when dependencies were **skipped** or the run was cancelled |
| `succeededOrFailed()` | Succeeded or failed, but **not** when skipped or cancelled |

Here `DeployProduction` only runs on a tag (line 659), so on a normal build it is **skipped**. Only
`always()` still notifies.

**Why the others fail**

- **A** — would not run when the dependency was skipped, which is the common case here
- **B** — `canceled()` exists, but `not(canceled())` still excludes skipped dependencies
- **C** — invented syntax

</details>

---

## Q13

Two services, `WebApp` and `Api`, are independent and must deploy **in parallel** to shorten the
pipeline. A developer writes `Api` with `dependsOn: WebApp`.

What is the effect?

- A. Both stages still run in parallel because they are independent
- B. The pipeline fails validation with a circular dependency error
- C. The stages run sequentially, removing the parallelism
- D. Azure DevOps detects the independence and parallelises them anyway

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-22.md`:** the fan-out pattern at lines **469–487**.

```yaml
  - stage: UnitTest
    dependsOn: Lint          # both depend on Lint...
  - stage: IntegrationTest
    dependsOn: Lint          # ...so they run in PARALLEL with each other
```

`dependsOn` is a **command**, not a hint. Whatever you name must finish first. Writing
`Api dependsOn WebApp` tells Azure DevOps to serialise them, and it obeys.

**To run them in parallel**, give them the *same* upstream dependency — or none:

```yaml
  - stage: WebApp
    dependsOn: Build
  - stage: Api
    dependsOn: Build         # same upstream = parallel with WebApp
```

**Why the others fail**

- **A** — declaring a dependency creates one. Independence in your head is not independence in YAML
- **B** — it is perfectly valid YAML. That is what makes it dangerous: no error, just a slower
  pipeline
- **D** — nothing auto-detects this. The graph is exactly what you declare

:::danger This is your repeated mistake

You have missed this in more than one mock. **`dependsOn` and `needs` are declarations of intent.**
Before answering any ordering question, draw the graph: which stages share an upstream? Those run in
parallel. Chained ones do not.

:::

**Also remember the default:** with **no** `dependsOn`, a stage depends on the one **declared before
it**. To make a stage start immediately, write `dependsOn: []`.

</details>

---

## Q14

Which GitHub Actions `runs-on` value makes a matrix job run on the operating system for its leg?

- A. `runs-on: ${{ matrix.os }}`
- B. `runs-on: $(matrix.os)`
- C. `runs-on: matrix.os`
- D. `runs-on: [self-hosted, matrix.os]`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-22.md`:** line **529**.

```yaml
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
```

**Why the others fail**

- **B** — `$( )` is **Azure Pipelines** macro syntax. The equivalent there is `vmImage: $(vmImage)`
  at line 574
- **C** — without `${{ }}` it is the literal string `matrix.os`, which matches no runner
- **D** — mixes a self-hosted label with an unexpanded expression

</details>

---

## Q15

A `workflow_dispatch` input is declared `type: boolean`. How is it read in a step condition?

- A. `if: ${{ inputs.dry_run == 'true' }}`
- B. `if: ${{ env.dry_run }}`
- C. `if: ${{ github.event.dry_run }}`
- D. `if: ${{ inputs.dry_run }}`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-22.md`:** lines **701–722**.

```yaml
      dry_run:
        type: boolean
        default: false
```

```yaml
      - name: Deploy
        if: ${{ !inputs.dry_run }}      # line 716
      - name: Dry run
        if: ${{ inputs.dry_run }}       # line 720
```

A `type: boolean` input arrives as a **real boolean**, so use it directly and negate with `!`.

**Why the others fail**

- **A** — comparing a boolean to the string `'true'` is the same class of bug as Challenge 20's
  `eq(parameters.runTests, 'true')`. It does not match
- **B** — `env` holds workflow-defined values, not dispatch inputs
- **C** — `github.event.inputs.*` is the **legacy** form and returns strings. `inputs.*` is current
  and preserves types

**Contrast with Q10:** values from `$GITHUB_OUTPUT` are always **strings**, so `== 'true'` is right
there. Typed dispatch inputs are real booleans. Same-looking code, different rules.

</details>

---

## Q16

Which Azure Pipelines feature provides the same "choose at queue time" experience as
`workflow_dispatch` inputs?

- A. Variable groups
- B. Runtime parameters
- C. Pipeline resources
- D. Agent demands

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-22.md`:** lines **729–744**.

```yaml
trigger: none

parameters:
  - name: environment
    displayName: "Target environment"
    type: string
    default: staging
    values:
      - development
      - staging
      - production
  - name: dryRun
    type: boolean
    default: false
```

The `values` list produces a dropdown at queue time, exactly like `type: choice` + `options` in
GitHub Actions.

**One important difference (line 760):**

```yaml
                - ${{ if not(parameters.dryRun) }}:
```

Parameters are **compile-time**, so they can add or remove YAML. GitHub `inputs` are runtime values
usable only in `if:` conditions, which **skip** a step rather than remove it.

**Why the others fail**

- **A** — variable groups are fixed values, not prompts
- **C** — resources declare external repos and pipelines
- **D** — demands match agents (Challenge 21)

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** trigger types does Challenge 22 configure? (Choose three.)

- A. Path-filtered push and pull request triggers
- B. Container registry resource triggers
- C. Issue comment (`issue_comment`) triggers
- D. Scheduled (cron) triggers
- E. Branch protection rule triggers
- F. Pipeline or workflow completion triggers

<details>
<summary>Show answer</summary>

### Answer: A, D, F

**In `challenge-22.md`:** paths at lines **56–64**, schedules at lines **245–250**, completion at
lines **328–332** and **358–367**.

**Why the others fail**

- **B** — container registry triggers are a real Azure Pipelines resource type (`resources:
  containers:` with a trigger), but Challenge 22 does not cover them. Challenge 28 does
- **C** — `issue_comment` is a real GitHub event, not used here
- **E** — branch protection is a policy, not a trigger

</details>

---

## Q18

Which **two** are true about GitHub Actions path filters? (Choose two.)

- A. Path filters apply to `schedule` triggers
- B. Path filters must be combined with `fetch-depth: 0`
- C. `paths` supports `!` negation patterns
- D. A path filter can reference the workflow file itself
- E. `paths` and `paths-ignore` cannot be used together

<details>
<summary>Show answer</summary>

### Answer: C, E

Strictly, **D is also true** — line 59 does exactly that:

```yaml
    paths:
      - "frontend/**"
      - "shared/**"
      - ".github/workflows/frontend.yml"      # line 59
```

The exam would not offer three correct options, so treat C and E as the intended pair — they are the
two **rules**, while D is a good practice. Including the workflow file means editing the CI
definition re-runs it, which is how you validate a change to the pipeline itself.

**C** — line 94: `- "!backend/docs/**"`.
**E** — line 782: the Break & fix error.

**Why the others fail**

- **A** — schedules have no path context. There is no diff to filter on
- **B** — `fetch-depth: 0` is needed when **your own script** inspects history (line 598), not for
  platform filtering

</details>

---

## Q19

Which **two** are required for a `workflow_run` workflow to deploy the correct commit safely?
(Choose two.)

- A. Add `needs:` referencing the upstream workflow
- B. Check `github.event.workflow_run.conclusion == 'success'`
- C. Set `types: [requested]` on the trigger
- D. Check out `github.event.workflow_run.head_sha`
- E. Use `workflow_call` instead of `workflow_run`

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-22.md`:** lines **336** and **341**.

```yaml
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}    # B
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.workflow_run.head_sha }}            # D
```

Two independent hazards: `types: [completed]` fires on failure too (B), and the workflow runs from
the default branch so `github.sha` is the wrong commit (D).

**Why the others fail**

- **A** — `needs` works between jobs **inside one workflow**. It cannot reference another workflow —
  the same boundary as `dependsOn` in Q6 and Q13
- **C** — `types: [requested]` fires when the upstream **starts**, which is worse
- **E** — `workflow_call` inverts control: the caller invokes you. That is a different design

</details>

---

## Q20

Which **two** correctly describe `needs` in GitHub Actions? (Choose two.)

- A. A job may declare an array of dependencies
- B. `needs` can reference a job in another workflow
- C. `needs` guarantees the jobs run on the same runner
- D. A dependency cycle is silently ignored
- E. Jobs sharing the same dependency run in parallel

<details>
<summary>Show answer</summary>

### Answer: A, E

**In `challenge-22.md`:** lines **416–426** and **452–454**.

```yaml
  unit-tests:
    needs: lint             # both depend on lint...
  integration-tests:
    needs: lint             # ...so they run in PARALLEL          (E)

  build-image:
    needs: [unit-tests, integration-tests]                        (A)
```

**Why the others fail**

- **B** — same boundary as Q19. Cross-workflow ordering needs `workflow_run`
- **C** — every job gets its own runner. That is why each repeats `checkout` (lines 413, 420)
- **D** — a cycle is a **hard error**. Lines 808–819 show `test needs build` and `build needs test`;
  the workflow will not start at all

</details>

---

## Q21

Which **two** describe the fan-out and fan-in pattern in Challenge 22's dependency graph? (Choose
two.)

- A. `deploy-production` starts as soon as the first verification job finishes
- B. Fan-out requires a matrix strategy across the jobs
- C. `smoke-tests` and `performance-tests` both depend on `deploy-staging` and run in parallel
- D. `deploy-production` depends on both verification jobs and waits for the slowest
- E. Fan-in requires `if: always()` on the downstream job

<details>
<summary>Show answer</summary>

### Answer: C, D

**In `challenge-22.md`:** lines **438–457**.

```yaml
  # Fan-out
  smoke-tests:
    needs: deploy-staging
  performance-tests:
    needs: deploy-staging

  # Fan-in
  deploy-production:
    needs: [smoke-tests, performance-tests]
```

**Why the others fail**

- **A** — an array in `needs` means **all**, not any. It waits for the slowest and for every one to
  succeed
- **B** — a matrix runs *the same job* many times. Fan-out runs *different* jobs. Both are parallel,
  but they solve different problems
- **E** — `if: always()` would make production deploy even if verification **failed**. That is the
  opposite of a gate

</details>

---

## Q22

Which **two** Azure Pipelines conditions run a stage when the upstream stage was **skipped**?
(Choose two.)

- A. `succeededOrFailed()` on the stage
- B. `always()` on the stage
- C. `not(canceled())` on the stage
- D. `succeeded()` on the stage
- E. `eq(dependencies.DeployStaging.result, 'Skipped')`

<details>
<summary>Show answer</summary>

### Answer: B, E

**In `challenge-22.md`:** lines **659** and **670**.

```yaml
  - stage: DeployProduction
    condition: and(succeeded(), startsWith(variables['Build.SourceBranch'], 'refs/tags/v'))
    # -> SKIPPED on a normal branch build

  - stage: Notify
    dependsOn: [DeployStaging, DeployProduction]
    condition: always()          # still runs
```

**E** works too — you can test a dependency's result explicitly, and `'Skipped'` is a valid value
alongside `Succeeded`, `SucceededWithIssues`, `Failed` and `Canceled`.

**Why the others fail**

- **A** — succeeded **or failed**. Skipped is neither, so it does not run
- **C** — excludes cancellation but still requires the dependency to have actually run
- **D** — requires success

**The three-way distinction, worth memorising:**

| Outcome | `succeeded()` | `succeededOrFailed()` | `always()` |
|---|---|---|---|
| Succeeded | run | run | run |
| Failed | skip | run | run |
| **Skipped** | skip | **skip** | **run** |

</details>

---

## Q23

Which **two** matrix behaviours are correct? (Choose two.)

- A. `include` always creates an additional job in the matrix
- B. An `include` entry matching an existing combination adds variables to it
- C. `exclude` requires `fail-fast: false` to take effect
- D. An `exclude` entry removes a combination from the matrix
- E. A matrix can only vary a single dimension

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-22.md`:** lines **535–541**.

```yaml
        exclude:
          - os: macos-latest
            node-version: 18          # D - removes that combination
        include:
          - os: ubuntu-latest
            node-version: 20
            coverage: true            # B - this combination already exists, so it gains a variable
```

Then line 550 uses the added variable:

```yaml
        if: matrix.coverage
```

**Why the others fail**

- **A** — only when the entry matches **no** existing combination. Matching entries merge
- **C** — unrelated settings
- **E** — line 533–534 vary two dimensions, producing a cross-product

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso's pipeline must deploy `WebApp` and `Api` **at the same time** after the build
stage, then run a notification stage once both finish, whether they succeeded or not.

---

## Q24

**Proposed solution:** Give both `WebApp` and `Api` `dependsOn: Build`, then give `Notify`
`dependsOn: [WebApp, Api]` with `condition: always()`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-22.md`:** the same shape as lines **469–487** and **665–670**.

```yaml
  - stage: WebApp
    dependsOn: Build
  - stage: Api
    dependsOn: Build              # same upstream -> parallel with WebApp
  - stage: Notify
    dependsOn: [WebApp, Api]
    condition: always()           # runs even if one failed or was skipped
```

Both requirements are met: **shared upstream = parallel**, and `always()` covers failure and skip.

</details>

---

## Q25

**Proposed solution:** Give `WebApp` `dependsOn: Build`, give `Api` `dependsOn: WebApp`, then give
`Notify` `dependsOn: Api` with `condition: always()`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**This is the exact mistake you have made in multiple mocks.**

```yaml
  - stage: WebApp
    dependsOn: Build
  - stage: Api
    dependsOn: WebApp        # SERIALISED - Api now waits for WebApp
```

The notification half is fine. The parallelism requirement is destroyed.

Nothing errors. The pipeline is valid and it works — it is just **slower**, and it violates a stated
requirement. That is why this trap catches people: there is no red flag to notice, only a
requirement you have to actively check against.

**The habit:** when a question says "at the same time", "in parallel", or "reduce total duration",
look at what each stage depends on. **Same upstream = parallel. Chained = serial.**

</details>

---

## Q26

**Proposed solution:** Give both `WebApp` and `Api` `dependsOn: Build`, then give `Notify`
`dependsOn: [WebApp, Api]` with `condition: succeededOrFailed()`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

The parallelism half is now correct. The notification half is not.

`succeededOrFailed()` does **not** run when a dependency is **skipped** (see the table in Q22). If
`Api` is skipped — a branch condition, a path filter, a manual gate — `Notify` is skipped too, and
nobody is told.

The requirement says "once both finish, whether they succeeded or not", and a skipped stage has
finished. Only `always()` covers every outcome.

**Two proposals, two different halves wrong.** That is exactly how the repeated-scenario format
works: check **every** requirement against **every** proposal, not just the one that looks familiar.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — path filters

| # | Statement | Answer |
|---|---|---|
| 1 | GitHub Actions allows `paths` and `paths-ignore` in the same trigger |  |
| 2 | Azure Pipelines allows `paths: include:` and `exclude:` together |  |
| 3 | GitHub `paths` supports `!` negation |  |
| 4 | Path filters can trigger on the workflow file itself |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | GitHub Actions allows `paths` and `paths-ignore` in the same trigger | **No** |
| 2 | Azure Pipelines allows `paths: include:` and `exclude:` together | **Yes** |
| 3 | GitHub `paths` supports `!` negation | **Yes** |
| 4 | Path filters can trigger on the workflow file itself | **Yes** |

**In `challenge-22.md`:** line **782** (row 1), lines **164–170** (row 2), line **94** (row 3), line
**59** (row 4).

Rows 1 and 2 together are the trap: **the two platforms have opposite rules.** GitHub forbids mixing
the keys and offers `!` instead; Azure Pipelines expects `include` and `exclude` side by side.

</details>

---

## Q28 — schedules

| # | Statement | Answer |
|---|---|---|
| 1 | Azure Pipelines `always: false` skips the run when nothing changed |  |
| 2 | A schedule can target a branch the pipeline trigger does not include |  |
| 3 | GitHub scheduled workflows are disabled after long repository inactivity |  |
| 4 | `trigger: none` prevents scheduled runs |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Azure Pipelines `always: false` skips the run when nothing changed | **Yes** |
| 2 | A schedule can target a branch the pipeline trigger does not include | **No** |
| 3 | GitHub scheduled workflows are disabled after long repository inactivity | **Yes** |
| 4 | `trigger: none` prevents scheduled runs | **No** |

**In `challenge-22.md`:** line **289** (row 1), lines **852–863** (row 2), line **281** (row 4).

Row 4 is the one people invert. `trigger: none` disables **CI** triggering only. Schedules are a
separate mechanism and keep firing — which is precisely why line 281 pairs it with a `schedules:`
block.

Row 3 is real-world important: a repo left alone for 60 days has its scheduled workflows disabled,
and GitHub emails you rather than failing loudly.

</details>

---

## Q29 — job dependencies

| # | Statement | Answer |
|---|---|---|
| 1 | Jobs with the same `needs` value run in parallel |  |
| 2 | `needs: [a, b]` waits for both to succeed |  |
| 3 | A dependency cycle produces a validation error |  |
| 4 | `dependsOn` can reference a stage in another pipeline |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Jobs with the same `needs` value run in parallel | **Yes** |
| 2 | `needs: [a, b]` waits for both to succeed | **Yes** |
| 3 | A dependency cycle produces a validation error | **Yes** |
| 4 | `dependsOn` can reference a stage in another pipeline | **No** |

**In `challenge-22.md`:** lines **416–426** (rows 1 and 2), lines **808–819** (row 3), lines
**358–367** (row 4).

Row 4 is your recurring confusion, stated as a fact. **`dependsOn` is intra-pipeline only.** For
cross-pipeline ordering you need a `resources: pipelines:` trigger.

Row 1 is the parallelism rule that Q13 and Q25 attack from the other direction.

</details>

---

## Q30 — conditions

| # | Statement | Answer |
|---|---|---|
| 1 | `if: always()` runs the job even if a needed job failed |  |
| 2 | `succeededOrFailed()` runs when a dependency was skipped |  |
| 3 | `contains(needs.*.result, 'failure')` detects any failed dependency |  |
| 4 | `needs.build.result` can equal `'skipped'` |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `if: always()` runs the job even if a needed job failed | **Yes** |
| 2 | `succeededOrFailed()` runs when a dependency was skipped | **No** |
| 3 | `contains(needs.*.result, 'failure')` detects any failed dependency | **Yes** |
| 4 | `needs.build.result` can equal `'skipped'` | **Yes** |

**In `challenge-22.md`:** lines **630–634** (rows 1 and 3), lines **670–675** (row 2).

Row 4 lists the four values: `success`, `failure`, `cancelled`, `skipped`. Comparing a result to
`'true'` — as one distractor in Q10 does — can never match.

Rows 1 and 3 together are the notification pattern: `always()` at job level so it runs, then
`contains()` at step level to decide what to say.

</details>

---

# Section E — Drag and drop

---

## Q31

Given this dependency graph, arrange the jobs into **execution waves** — jobs in the same wave run in
parallel.

```yaml
  lint:
  unit-tests:        needs: lint
  integration-tests: needs: lint
  build-image:       needs: [unit-tests, integration-tests]
  deploy-staging:    needs: build-image
  smoke-tests:       needs: deploy-staging
  performance-tests: needs: deploy-staging
  deploy-production: needs: [smoke-tests, performance-tests]
```

<details>
<summary>Show answer</summary>

### Answer — five waves

| Wave | Jobs |
|---|---|
| 1 | `lint` |
| 2 | `unit-tests`, `integration-tests` |
| 3 | `build-image` |
| 4 | `smoke-tests`, `performance-tests` — after `deploy-staging` |
| 5 | `deploy-production` |

Strictly: wave 3 is `build-image`, wave 4 is `deploy-staging`, wave 5 is the two verification jobs,
wave 6 is `deploy-production`. **Six waves, two of which fan out.**

**In `challenge-22.md`:** lines **402–457**.

**The rule, in one line:** *jobs sharing the same `needs` value run in parallel; everything else is
serial.*

Total duration is the **critical path**, not the sum. If `unit-tests` takes 2 minutes and
`integration-tests` takes 8, wave 2 costs 8 minutes, not 10.

:::danger Draw this graph on the exam

Ordering questions are where you lose marks. Sketch the waves before reading the options. It takes
twenty seconds and turns a trap into arithmetic.

:::

</details>

---

## Q32

Match each GitHub Actions trigger concept to its Azure Pipelines equivalent.

| GitHub Actions | Azure Pipelines |
|---|---|
| `on: push` with `paths` |  |
| `on: pull_request` |  |
| `on: schedule` with `cron` |  |
| `on: workflow_run` |  |
| `on: workflow_dispatch` with `inputs` |  |
| `needs:` |  |
| `if:` |  |
| `!` negation in `paths` |  |

**Options:** `condition:` · `dependsOn:` · `parameters:` (runtime parameters) · `paths: exclude:` · `pr:` · `resources: pipelines:` with `trigger` · `schedules:` with `cron` · `trigger:` with `paths: include:`

<details>
<summary>Show answer</summary>

| GitHub Actions | Azure Pipelines |
|---|---|
| `on: push` with `paths` | `trigger:` with `paths: include:` |
| `on: pull_request` | `pr:` |
| `on: schedule` with `cron` | `schedules:` with `cron` |
| `on: workflow_run` | `resources: pipelines:` with `trigger` |
| `on: workflow_dispatch` with `inputs` | `parameters:` (runtime parameters) |
| `needs:` | `dependsOn:` |
| `if:` | `condition:` |
| `!` negation in `paths` | `paths: exclude:` |

**In `challenge-22.md`:** paths **56 / 164**, pr **60 / 172**, schedule **246 / 284**, completion
**328 / 358**, manual **687 / 729**, dependencies **411 / 470**, conditions **615 / 650**, negation
**94 / 168**.

**The two rows that trip people:**

- `workflow_run` ↔ `resources: pipelines:` — **not** `dependsOn`. Neither `needs` nor `dependsOn`
  crosses a pipeline boundary
- `!` negation ↔ `exclude:` — GitHub forbids mixing include and ignore keys; Azure Pipelines expects
  both keys together

</details>

---

## Q33

Arrange the Azure Pipelines conditions from **most** restrictive to **least** restrictive.

**Items:** `always()` · `succeeded()` · `succeededOrFailed()`

<details>
<summary>Show answer</summary>

### Answer: `succeeded()` → `succeededOrFailed()` → `always()`

| Condition | Runs on success | on failure | on **skipped** | on cancel |
|---|---|---|---|---|
| `succeeded()` | yes | no | no | no |
| `succeededOrFailed()` | yes | yes | **no** | no |
| `always()` | yes | yes | **yes** | yes |

**In `challenge-22.md`:** lines **670** and **675**.

**The gap that matters is "skipped".** A stage gated by a branch or tag condition (line 659) is
*skipped* on ordinary builds — and `succeededOrFailed()` will not run after it. Q26 is built on
exactly this.

</details>

---

## Q34

Arrange the steps to configure a pipeline-completion trigger in Azure Pipelines.

**Items:** Declare a `resources: pipelines:` entry · Add a `trigger` block with branch filters ·
Give the resource an alias · Download the upstream artifact by alias · Name the `source` pipeline

<details>
<summary>Show answer</summary>

### Answer

1. Declare `resources: pipelines:` — line **359**
2. Give the resource an alias (`- pipeline: backendCI`) — line **360**
3. Name the `source` pipeline — line **361**
4. Add a `trigger` block with branch filters — lines **362–367**
5. Download the artifact by alias — lines **382–383**

```yaml
resources:
  pipelines:                              # 1
    - pipeline: backendCI                 # 2 - the alias
      source: "Backend-CI-Pipeline"       # 3 - the real pipeline NAME
      trigger:                            # 4
        branches:
          include: [main]
        stages: [Test]
...
      - download: backendCI               # 5 - by alias, not by source name
        artifact: drop
```

**The alias-versus-source distinction is the exam point.** `source` is the pipeline's real name in
Azure DevOps; `pipeline:` is the local alias you use everywhere else. Mixing them up gives a
resource-not-found error.

</details>

---

## Q35

Match each requirement to the correct mechanism.

| Requirement | Mechanism |
|---|---|
| Skip the whole workflow when only docs changed |  |
| Skip one job when its folder did not change |  |
| Run nightly only when code changed |  |
| Deploy only after CI succeeds |  |
| Notify whether or not the deploy worked |  |
| Test three Node versions at once |  |

**Options:** `always: false` on the schedule · `if: always()` / `condition: always()` · Job `if` reading a previous job's output · Matrix strategy · Path filter on the trigger · `workflow_run` + conclusion check

<details>
<summary>Show answer</summary>

| Requirement | Mechanism |
|---|---|
| Skip the whole workflow when only docs changed | **Path filter on the trigger** |
| Skip one job when its folder did not change | **Job `if` reading a previous job's output** |
| Run nightly only when code changed | **`always: false` on the schedule** |
| Deploy only after CI succeeds | **`workflow_run` + conclusion check** |
| Notify whether or not the deploy worked | **`if: always()` / `condition: always()`** |
| Test three Node versions at once | **Matrix strategy** |

**In `challenge-22.md`:** lines **56**, **615**, **289**, **336**, **630**, **532**.

**Rows 1 and 2 look identical and are not.** A path filter stops the **run** from starting at all —
no agent, no cost. A job condition runs *inside* a workflow that already started, so you still pay
for the change-detection job (lines 596–611). Use the filter when the whole workflow is irrelevant;
use the condition when part of it is.

</details>

---

# Section F — Hot area

---

## Q36

```yaml
on:
  push:
    branches: [main]
    [BLANK 1]:
      - "backend/**"
      - "[BLANK 2]backend/docs/**"
```

- **BLANK 1:** `paths-ignore` / `filters` / `paths` / `include`
- **BLANK 2:** `-` / `^` / `!` / `~`

<details>
<summary>Show answer</summary>

### Answer: `paths`, `!`

**In `challenge-22.md`:** lines **92–95**, and the Break & fix at lines **782–798**.

You cannot add a `paths-ignore` key alongside `paths`, so the exclusion must be a **negation pattern
inside** `paths`.

</details>

---

## Q37

```yaml
schedules:
  - cron: "0 4 * * 0"
    displayName: "Sunday full regression"
    branches:
      include:
        - main
    [BLANK 1]: true
```

- **BLANK 1:** `enabled` / `batch` / `force` / `always`

<details>
<summary>Show answer</summary>

### Answer: `always`

**In `challenge-22.md`:** line **295**.

`always: true` runs the schedule even with no new commits. Its sibling at line 289 uses
`always: false` for the nightly build, so unchanged code does not burn agent minutes.

</details>

---

## Q38

```yaml
on:
  [BLANK 1]:
    workflows: ["Backend CI"]
    types: [completed]

jobs:
  deploy:
    if: ${{ github.event.workflow_run.[BLANK 2] == 'success' }}
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.workflow_run.[BLANK 3] }}
```

- **BLANK 1:** `workflow_call` / `workflow_run` / `workflow_dispatch` / `repository_dispatch`
- **BLANK 2:** `status` / `result` / `outcome` / `conclusion`
- **BLANK 3:** `sha` / `commit` / `head_sha` / `ref`

<details>
<summary>Show answer</summary>

### Answer: `workflow_run`, `conclusion`, `head_sha`

**In `challenge-22.md`:** lines **329**, **336**, **341**.

`status` tells you the run **finished**; `conclusion` tells you **how**. `result` and `outcome`
belong to the `needs` and `steps` contexts, not to `workflow_run`.

`head_sha` is essential: the workflow itself runs from the default branch, so `github.sha` would be
the wrong commit.

</details>

---

## Q39

```yaml
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      [BLANK 1]: false
      matrix:
        os: [ubuntu-latest, windows-latest]
        node-version: [20, 22]
        [BLANK 2]:
          - os: windows-latest
            node-version: 20
```

- **BLANK 1:** `continue-on-error` / `max-parallel` / `fail-fast` / `strict`
- **BLANK 2:** `include` / `omit` / `skip` / `exclude`

<details>
<summary>Show answer</summary>

### Answer: `fail-fast`, `exclude`

**In `challenge-22.md`:** lines **531** and **535**.

With that `exclude`, the matrix produces **3** jobs, not 4.

`continue-on-error` would mark failures as passes. `omit` and `skip` do not exist.

</details>

---

## Q40

```yaml
  notify:
    needs: [deploy-frontend, deploy-backend]
    if: [BLANK 1]
    steps:
      - name: Send notification
        if: [BLANK 2](needs.*.result, 'failure')
```

- **BLANK 1:** `success()` / `always()` / `failure()` / `cancelled()`
- **BLANK 2:** `includes` / `has` / `any` / `contains`

<details>
<summary>Show answer</summary>

### Answer: `always()`, `contains`

**In `challenge-22.md`:** lines **630** and **634**.

Two levels: the **job** must run regardless (`always()`), then the **step** decides what to say.
Without `always()`, the job is skipped when an upstream job fails — exactly when the notification
matters most.

`needs.*.result` is an array, and `contains()` is the function that searches one.

</details>

---

## Q41

```yaml
  - stage: Api
    [BLANK 1]: Build
  - stage: WebApp
    [BLANK 2]: Build
  - stage: Notify
    dependsOn: [Api, WebApp]
    [BLANK 3]: always()
```

Requirement: `Api` and `WebApp` must deploy **in parallel**, and `Notify` must run whatever happens.

- **BLANK 1:** `needs` / `after` / `dependsOn` / `condition`
- **BLANK 2:** `dependsOn: Api` / `needs` / `dependsOn` / `condition`
- **BLANK 3:** `if` / `when` / `condition` / `trigger`

<details>
<summary>Show answer</summary>

### Answer: `dependsOn`, `dependsOn` (on **Build**, not on `Api`), `condition`

**Both stages must name the same upstream.** Pointing `WebApp` at `Api` would serialise them — the
mistake in Q13 and Q25, and the one you have made in more than one mock.

```yaml
  - stage: Api
    dependsOn: Build       # same upstream...
  - stage: WebApp
    dependsOn: Build       # ...therefore parallel
```

`needs` and `if` are GitHub Actions keywords.

</details>

---

# Section G — Case study

## Case study: Contoso monorepo

### Background

Contoso keeps `frontend/`, `backend/`, `shared/` and `infra/` in one repository. Every push currently
builds everything, so a README change triggers a 20-minute pipeline.

### Requirements

**Efficiency**

- A change to `frontend/` must not build the backend
- Documentation-only changes must not trigger any build
- A change to `shared/` must build both frontend and backend

**Testing**

- Tests must run on Node 20 and 22, on Linux and Windows
- One platform failing must not hide failures on the others

**Orchestration**

- Unit and integration tests must run in parallel after linting
- The image must build only after both test suites pass
- Smoke and performance tests must run in parallel after staging deploys
- Production must deploy only after both verifications pass

**Operations**

- A nightly build must run on weekdays only when code changed
- A full regression must run every Sunday regardless of changes
- The team must be notified whether the deployment succeeded or failed

---

## Q42

Which configuration meets the efficiency requirements?

- A. One workflow with `paths-ignore` for documentation and a job per component
- B. Separate workflows per component, each with `paths` filters including `shared/**`
- C. One workflow that checks changed files in a script and exits early if none match
- D. Separate repositories for frontend and backend, each with its own workflow

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-22.md`:** lines **49–121**.

```yaml
# frontend.yml
    paths:
      - "frontend/**"
      - "shared/**"          # shared triggers BOTH workflows

# backend.yml
    paths:
      - "backend/**"
      - "!backend/**/*.md"   # docs excluded
      - "shared/**"
```

Listing `shared/**` in both files is what satisfies the third requirement.

**Why the others fail**

- **A** — one workflow cannot build only the changed component. And `paths-ignore` alone does not
  separate frontend from backend
- **C** — the workflow still starts, allocates an agent and checks out code before deciding to stop.
  You pay for it
- **D** — splitting the repository is a far larger change than the requirement asks for, and it
  breaks `shared/`. Challenge 12 covers that trade-off properly

</details>

---

## Q43

Which matrix configuration meets the testing requirements?

- A. `matrix: os: [ubuntu-latest, windows-latest], node-version: [20, 22]` with `fail-fast: true`
- B. Four separate jobs, one per combination of OS and Node version, with no matrix
- C. `matrix: os: [ubuntu-latest, windows-latest], node-version: [20, 22]` with `fail-fast: false`
- D. One job on `ubuntu-latest` looping over both Node versions in a shell script

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-22.md`:** lines **530–534**.

2 × 2 = **4 parallel jobs**, and `fail-fast: false` means a Windows failure does not cancel the Linux
legs. The requirement "one platform failing must not hide failures on the others" is a direct
instruction to disable fail-fast.

**Why the others fail**

- **A** — the default. The first failure cancels the rest, hiding the others
- **B** — works, but duplicates the job definition four times. When a matrix fits, the exam wants the
  matrix
- **D** — sequential, and one failure stops the loop

</details>

---

## Q44

Which **two** configurations meet the orchestration requirements? (Choose two.)

- A. `integration-tests` declares `needs: unit-tests`
- B. `build-image` declares `needs: unit-tests`
- C. `unit-tests` and `integration-tests` both declare `needs: lint`
- D. All test jobs use `if: always()` so none is skipped
- E. `build-image` declares `needs: [unit-tests, integration-tests]`

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-22.md`:** lines **409–426**.

**C** gives parallelism — same upstream. **E** gives the fan-in — an array waits for all.

**Why the others fail**

- **A** — **the trap, and your repeated one.** Chaining serialises the two test suites and violates
  "in parallel"
- **B** — the image would build after unit tests alone, before integration tests finish. It could
  ship code that fails integration
- **D** — `always()` would build the image even when tests **failed**, destroying the gate

</details>

---

## Q45

Which configuration meets both nightly-build requirements?

- A. One `schedules` entry with `always: true` running every day including Sunday
- B. Two GitHub `cron` entries with no extra settings, one for weekdays and another for Sunday
- C. A single cron running daily with a stage condition on the day of week
- D. Two `schedules` entries, one `always: false` for weekdays and one `always: true` for Sunday

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-22.md`:** lines **283–295**.

```yaml
schedules:
  - cron: "0 2 * * 1-5"
    displayName: "Weekday nightly build"
    always: false          # only when code changed
  - cron: "0 4 * * 0"
    displayName: "Sunday full regression"
    always: true           # regardless
```

**Why the others fail**

- **A** — one setting cannot be both. Weekday builds would run on unchanged code
- **B** — **GitHub Actions has no `always` equivalent**; its schedules always fire. The requirement
  "only when code changed" is an Azure Pipelines feature, so this is the wrong platform for it
- **C** — one schedule cannot have two different `always` behaviours, and a stage condition runs
  after the agent is already allocated

**Bonus (line 311):** `Build.CronSchedule.DisplayName` lets a stage detect *which* schedule started
the run — which is how the `FullRegression` stage runs only on Sunday.

</details>

---

## Q46

Which configuration meets the notification requirement?

- A. A `Notify` stage with `dependsOn` on both deploy stages and `condition: succeededOrFailed()`
- B. A `Notify` stage with `dependsOn` on both deploy stages and `condition: always()`
- C. A notification step at the end of the production deploy job with `condition: always()`
- D. A separate scheduled pipeline that polls the deployment status every 15 minutes

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-22.md`:** lines **665–674**.

**Why the others fail**

- **A** — does not run when a dependency was **skipped**. Production is gated on a tag (line 659), so
  it is skipped on ordinary builds and nobody is notified
- **C** — the step only runs if its job runs. Production is gated on a tag (line 659), so on an
  ordinary build the job never starts and the notification never sends
- **D** — a poll on a timer is not "notified when it finishes"

</details>

---

## Q47

After implementing path filters, a change to `shared/utils/logger.js` builds the frontend but not the
backend.

What is the cause?

- A. `shared/**` is missing from the backend workflow's `paths`
- B. The backend workflow uses `paths-ignore` alongside `paths`
- C. `shared/` needs its own workflow with a `paths` filter
- D. Path filters do not match nested directories under `shared/`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-22.md`:** compare line **58** with line **96** — both workflows list `shared/**`
precisely so a shared change builds both.

```yaml
# frontend.yml
    paths:
      - "frontend/**"
      - "shared/**"      # line 58
# backend.yml
    paths:
      - "backend/**"
      - "shared/**"      # line 96 - if this is missing, backend never builds
```

**Why the others fail**

- **B** — that would cause the Break & fix failure from Q1: the trigger never fires **at all**, not
  selectively
- **C** — a workflow for `shared/` would test the shared code but still not build its consumers
- **D** — `**` matches nested directories by design

**The general lesson for monorepos:** a shared library must appear in the path filter of **every**
component that depends on it. Miss one and that component silently stops being tested against
changes it depends on — the worst kind of failure, because everything stays green.

</details>

---

## Q48

The team wants to reduce cost further. Currently every push to `main` runs the frontend workflow,
which spends 90 seconds on change detection before deciding to skip its deploy jobs.

What should they change?

- A. Add `fail-fast: false` to the matrix so legs are not cancelled early
- B. Increase `max-parallel` so change detection finishes sooner
- C. Move the decision from a job condition into the trigger's path filter
- D. Add `if: always()` to the deploy jobs so they no longer wait on detection

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-22.md`:** compare the trigger filter at lines **56–59** with the in-workflow change
detection at lines **596–611**.

```yaml
# Costs nothing when it does not match - no agent is ever allocated
on:
  push:
    paths:
      - "frontend/**"
      - "shared/**"
```

```yaml
# Costs an agent, a checkout and 90 seconds before deciding to do nothing
      - name: Detect changes
        id: changes
        run: git diff --name-only HEAD~1 | grep -q '^frontend/'
```

**The rule: filter as early as possible.** Trigger filters are free; in-workflow detection is not.

**Why the others fail**

- **A** — affects matrix cancellation, not whether the workflow starts
- **B** — more parallelism costs more, not less
- **D** — `always()` makes *more* run

**When in-workflow detection is still right:** when one workflow must handle several components and
decide **per job** which to deploy — the pattern at lines 613–625. The two techniques are
complementary: filter the workflow, then condition the jobs.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **`dependsOn` chained instead of shared** | Q13, Q25, Q41, Q44 | Same upstream = parallel. Chained = serial. Draw the graph |
| **`dependsOn` / `needs` across pipelines** | Q6, Q19, Q29 | Intra-pipeline only. Use `resources: pipelines:` or `workflow_run` |
| **`paths` + `paths-ignore` together** | Q1, Q27 | GitHub forbids it — use `!`. Azure Pipelines expects include + exclude |
| **`succeededOrFailed()` vs `always()`** | Q12, Q22, Q26, Q33 | Only `always()` covers **skipped** |
| **`workflow_run` fires on failure too** | Q4, Q19, Q38 | Check `conclusion == 'success'` |
| **`workflow_run` checks out the wrong commit** | Q5, Q19 | Use `head_sha`, not `github.sha` |
| **`fail-fast: false` vs `continue-on-error`** | Q7 | One reports honestly, the other hides the failure |
| **`include` assumed to add a job** | Q8, Q23 | Matching an existing combination adds variables instead |
| **String vs boolean comparison** | Q10, Q15 | `$GITHUB_OUTPUT` gives strings; typed inputs give booleans |
| **Schedule branch not in trigger branches** | Q3, Q28 | The schedule can only run on a branch the pipeline tracks |
| **`trigger: none` assumed to stop schedules** | Q28 | It only disables CI |
| **Job condition used where a path filter belongs** | Q35, Q48 | Filter first — it is free. Conditions cost an agent |
| **Shared folder missing from one path filter** | Q47 | Every consumer must list the shared path |

---

# The blocks to memorise

Line numbers are in `challenge-22.md`.

```yaml
# 1. Path filter with negation  (lines 92-95) - GitHub
on:
  push:
    branches: [main]
    paths:
      - "backend/**"
      - "!backend/docs/**"       # NEVER add paths-ignore alongside

# 2. Path filter  (lines 164-170) - Azure Pipelines, opposite rule
trigger:
  branches:
    include: [main]
  paths:
    include: [frontend/*, shared/*]
    exclude: [frontend/docs/*]   # include AND exclude are fine here

# 3. Schedule with change detection  (lines 283-295)
schedules:
  - cron: "0 2 * * 1-5"
    always: false                # skip when nothing changed
  - cron: "0 4 * * 0"
    always: true                 # run regardless

# 4. Workflow completion trigger  (lines 328-341)
on:
  workflow_run:
    workflows: ["Backend CI"]
    types: [completed]
jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.workflow_run.head_sha }}

# 5. Pipeline completion trigger  (lines 358-367) - Azure Pipelines
resources:
  pipelines:
    - pipeline: backendCI
      source: "Backend-CI-Pipeline"
      trigger:
        branches:
          include: [main]
        stages: [Test]

# 6. Parallel then fan-in  (lines 409-426) - THE one you keep missing
  unit-tests:
    needs: lint                  # same upstream...
  integration-tests:
    needs: lint                  # ...so these two run in PARALLEL
  build-image:
    needs: [unit-tests, integration-tests]   # waits for BOTH

# 7. Matrix with exclude and include  (lines 530-541)
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [18, 20, 22]
        exclude:
          - os: macos-latest
            node-version: 18     # 9 - 1 = 8 jobs
        include:
          - os: ubuntu-latest
            node-version: 20
            coverage: true       # adds a VARIABLE to an existing job

# 8. Notify whatever happens  (lines 628-635)
  notify:
    needs: [deploy-frontend, deploy-backend]
    if: always()
    steps:
      - if: contains(needs.*.result, 'failure')
        run: echo "One or more deployments failed"
```

**The condition table — learn it as a grid:**

| Outcome | `succeeded()` | `succeededOrFailed()` | `always()` |
|---|---|---|---|
| Succeeded | run | run | run |
| Failed | skip | run | run |
| **Skipped** | skip | **skip** | **run** |
| Cancelled | skip | skip | run |

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 22 is exam-ready. Move to Challenge 23 |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 5 and 7, then retake this |
| Below 30 | Redo the whole challenge, drawing the dependency graph by hand |

Record your result in `AZ-400-Learning-Log.md` under Challenge 22.

:::danger If you missed Q13, Q25, Q41 or Q44

That is the `dependsOn` parallelism trap, and it has now caught you across several mocks. Before you
move on, write out the six-wave graph from Q31 on paper from memory. **Same upstream = parallel.
Chained = serial.** Say it out loud.

:::
