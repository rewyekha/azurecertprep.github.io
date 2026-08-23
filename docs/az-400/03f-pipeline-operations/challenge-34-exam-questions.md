---
sidebar_position: 1.5
toc_max_heading_level: 2
title: "Challenge 34: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 34 — AZ-400 exam questions

**48 questions** built only from what Challenge 34 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-34.md`**.

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

:::danger Retrying without recording is the trap this challenge is built on

A retry that hides a flaky test makes the dashboard green and the problem permanent. Every correct
answer here **retries and records** — annotation, artifact, or tracking issue. See Q3, Q11, Q25 and
Q26.

:::

---

# Section A — Single answer

---

## Q1

What defines a **flaky test**?

- A. A test that fails consistently
- B. A test that passes and fails intermittently without code changes
- C. A test that takes longer than average
- D. A test with no assertions

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-34.md`:** line **31**.

> *"Flaky tests pass and fail intermittently without code changes."*

**Why the definition matters operationally:** a consistently failing test is information — something
is broken. A flaky test is **noise**, and noise is worse than a red build, because it teaches the team
that red means nothing. The scenario shows the endpoint of that (line 22): developers pushing straight
to main because "it will just fail anyway".

**How Azure DevOps detects it (line 144):** a test that **passes and fails on the same code commit**.
No code changed, so the test itself is the variable.

**Why the others fail** — A is a genuine failure, C is slow rather than unreliable, D is a bad test
that is at least deterministic.

</details>

---

## Q2

Which Jest setting retries a failed test automatically?

- A. `retryTimes: 2`
- B. `maxWorkers: 2`
- C. `bail: 2`
- D. `testTimeout: 2000`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-34.md`:** lines **92–94**.

```javascript
  // Retry failed tests up to 2 times
  // If a test passes on retry, it is marked as flaky
  retryTimes: 2,
```

**Retry alone is only half the mechanism.** It stops a flaky test failing the build; it does **not**
tell you which test was flaky. That is why the challenge pairs it with annotation and reporting (Task
5), and why Break & fix Exercise 2 exists.

**Why the others fail**

- **B** — parallelism. More workers can actually *increase* flakiness through resource contention
- **C** — `bail` stops the run after N failures. The opposite of retrying
- **D** — a per-test timeout. Raising it can mask a timing-sensitive test rather than fix it

</details>

---

## Q3

A pipeline reports a 100% test pass rate, but bugs reach production.

What is the most likely cause?

- A. Tests are not being written
- B. `|| true` on the test step and `failTaskOnFailedTests: false`
- C. The test results file is missing
- D. Tests run on the wrong branch

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-34.md`:** Break & fix Exercise 1, lines **642–652**.

```yaml
- script: npx jest --ci || true          # ERROR: swallows the failure
- task: PublishTestResults@2
  inputs:
    failTaskOnFailedTests: false          # ERROR: never fails even with failures
```

**Two independent muzzles.** `|| true` forces the shell step to exit 0 whatever Jest returns, and
`failTaskOnFailedTests: false` stops the publish task raising a failure either. Remove one and the
other still hides everything.

**The fix (lines 662–673)** removes `|| true` and sets `failTaskOnFailedTests: true` — while keeping
`condition: always()` so results are still published when tests fail. **Publish always; fail
honestly.**

**Why the others fail** — C would show *missing* results rather than a 100% pass rate; A and D would
not produce a clean green dashboard alongside production bugs.

</details>

---

## Q4

Which Azure DevOps setting must be enabled for built-in flaky test detection?

- A. `flakyDetection.isEnabled` in test management settings
- B. `failTaskOnFailedTests: true`
- C. `continueOnError: true`
- D. `retryCountOnTaskFailure`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-34.md`:** lines **149–170**.

```json
#   "flakySettings": {
#     "flakyDetection": {
#       "isEnabled": true,
#       "flakyDetectionType": "system"
#     },
#     "flakyInSummaryReport": true
#   }
```

Configured at **Project Settings → Pipelines → Test Management → Flaky test detection** (line 153).

**Two settings, two effects.** `flakyDetectionType: "system"` lets Azure DevOps identify flaky tests
automatically — it re-runs and compares on the same commit. `flakyInSummaryReport: true` decides
whether they appear in the run summary, which is what gives the team visibility.

**The alternative is `"custom"`** — you mark tests as flaky yourself, useful when you already know
which ones are unreliable.

**Why the others fail** — B controls build failure, C swallows errors, D is a task-level retry
setting.

</details>

---

## Q5

Which metric measures how long the pipeline stays broken?

- A. Success rate
- B. MTTR — mean time to recovery
- C. P95 duration
- D. Queue time

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-34.md`:** lines **357–371** and the target table at line **380**.

```javascript
            // Calculate MTTR (time from failure to next success on same branch)
            for (let i = 0; i < mainRuns.length - 1; i++) {
              if (mainRuns[i].conclusion === 'failure' && mainRuns[i+1].conclusion === 'success') {
                const recovery = (new Date(mainRuns[i+1].updated_at) - new Date(mainRuns[i].created_at)) / 1000 / 60;
                mttrValues.push(recovery);
              }
            }
```

**Target: under 30 minutes** (line 380).

**Note how it is calculated** — from the **start of the failing run** to the **end of the next
successful run**, on the same branch. That includes the time nobody noticed, which is the point: the
scenario's fourth complaint is "nobody notices when the pipeline has been broken for hours" (line 25).

**This is one of the four DORA metrics** from Challenge 04, applied to the pipeline rather than to
production.

**Why the others fail** — A measures how often it breaks, C how slow it is, D how long work waits to
start.

</details>

---

## Q6

Which four targets does the weekly health report track?

- A. Success rate, average duration, P95 duration, MTTR
- B. Lines of code, test count, coverage, complexity
- C. Deployment frequency, lead time, change failure rate, MTTR
- D. CPU, memory, disk, network

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-34.md`:** lines **375–382**.

| Metric | Target |
|---|---|
| Success rate | **> 90%** |
| Average duration | **< 15 min** |
| P95 duration | **< 25 min** |
| MTTR | **< 30 min** |

**Why P95 sits alongside the average** — and this is the exam-relevant part. An average of 14 minutes
looks healthy while a P95 of 60 means one run in twenty takes an hour. Developers remember the bad
runs, so the P95 is what actually shapes their trust in CI.

**Why the others fail**

- **C** — **the close one.** Those are the four **DORA** metrics from Challenge 04, measuring
  *delivery*. These measure the *pipeline*. MTTR appears in both, at different scopes
- **B** and **D** — code quality and infrastructure metrics

</details>

---

## Q7

Which permission does the metrics workflow need to read another workflow's runs?

- A. `actions: read`
- B. `contents: read`
- C. `checks: read`
- D. `workflows: read`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-34.md`:** lines **207–209**.

```yaml
permissions:
  actions: read      # read workflow runs and jobs
  issues: write      # create the alert issue
```

**`actions` is the scope covering workflow runs, jobs, artifacts and logs** — everything the Actions
API exposes about runs.

**Why the others fail**

- **B** — repository files
- **C** — check runs and suites, a related but different API
- **D** — not a permission scope. `workflows: write` exists and governs *modifying workflow files*, not
  reading runs

**Note both permissions map to something the job does** — read runs, create an issue. Same
least-privilege pattern as every other workflow in these challenges.

</details>

---

## Q8

Which trigger runs the metrics collector after the CI pipeline finishes?

- A. `workflow_run` with `types: [completed]`
- B. `workflow_call`
- C. `schedule`
- D. `repository_dispatch`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-34.md`:** lines **202–205**.

```yaml
on:
  workflow_run:
    workflows: ["CI Pipeline"]
    types: [completed]
```

**`completed` fires on success *and* failure**, which is correct here — you want metrics from failed
runs most of all. Compare with the alert workflow at line 485, which then filters:

```yaml
    if: github.event.workflow_run.conclusion == 'failure'
```

**The pattern to carry from Challenge 22:** `types: [completed]` is the trigger; `conclusion` is the
filter. The trigger cannot express "only when it failed".

**Why the others fail** — B makes it callable, C runs on a clock (which the *weekly report* uses at
line 313), D is an external API trigger.

</details>

---

## Q9

The degradation alert fires when what condition is met?

- A. Any single failure on main
- B. Three or more consecutive failures on main
- C. Duration exceeds 45 minutes
- D. Test pass rate drops below 90%

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-34.md`:** lines **491–513**.

```javascript
            const consecutiveFailures = recentRuns
              .filter(r => r.conclusion === 'failure').length;

            if (consecutiveFailures >= 3) {
              await github.rest.issues.create({
                title: `ALERT: ${consecutiveFailures} consecutive pipeline failures on main`,
                labels: ['pipeline-health', 'urgent', 'P1'],
                assignees: ['oncall-engineer']
              });
            }
```

**Three, not one, and the threshold is the design decision.** A single failure on main is often one
developer's mistake, already being fixed. Three in a row is systemic — and alerting on every failure
trains people to ignore the alert, which is how the scenario reached "nobody notices" in the first
place.

**Note the assignee** (line 513): an alert with no owner is a notification, not an escalation.

**Why the others fail**

- **A** — noise
- **C** — a real alert, and it is the **duration** threshold in a different workflow (line 245)
- **D** — not implemented here

</details>

---

## Q10

Which condition prevents the flaky-test annotation step from failing when no flaky tests were found?

- A. `if: always() && hashFiles('flaky-tests.txt') != ''`
- B. `continue-on-error: true`
- C. `if: success()`
- D. `if: failure()`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-34.md`:** lines **454–455** and **461–462**.

```yaml
      - name: Annotate flaky tests
        if: always() && hashFiles('flaky-tests.txt') != ''
```

**Two conditions doing two jobs.** `always()` ensures the step runs even after a failing test step —
which is exactly when flaky data exists. `hashFiles(...) != ''` skips it when the file was never
created, so a clean run does not attempt to read a missing file.

**`hashFiles()` returns an empty string when nothing matches**, which is the idiomatic "does this file
exist?" test in GitHub Actions.

**Why the others fail**

- **B** — would let a genuine error pass silently
- **C** — the test step failed, so `success()` is false and the annotation never appears
- **D** — misses the case where tests passed on the first attempt but a *previous* step wrote the file

</details>

---

## Q11

Why does the retry implementation compare results from two attempts rather than simply retrying?

- A. To distinguish flaky tests from genuine failures
- B. To speed up the test run
- C. To reduce test count
- D. To satisfy code coverage requirements

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-34.md`:** lines **436–450**.

```javascript
              const flaky = r2.testResults
                .filter(t => t.status === 'passed')
                .map(t => t.name);
              if (flaky.length > 0) {
                console.log('::warning::Flaky tests detected: ' + flaky.join(', '));
              }
              // Fail only if tests still fail on retry
              const stillFailing = r2.testResults.filter(t => t.status === 'failed');
              process.exit(stillFailing.length > 0 ? 1 : 0);
```

**The comparison is the diagnosis.** Failed then passed with no code change = **flaky**. Failed then
failed = **genuine**. A blind retry cannot tell them apart, so it treats both as "eventually fine".

**And the exit code is deliberate:** the build fails only on genuine failures, while flaky ones raise
a `::warning::` and are recorded. Green build, visible debt.

**Why the others fail** — B is false, retrying is slower; C and D are unrelated.

</details>

---

## Q12

Which `gh` command calculates the failure rate over recent runs?

- A. `gh run list --json conclusion --jq '...'`
- B. `gh workflow view`
- C. `gh run watch`
- D. `gh api /rate_limit`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-34.md`:** lines **183–185**.

```bash
gh run list --workflow=ci.yml --limit 200 --json conclusion \
  --jq '[.[] | .conclusion] | {total: length, failures: ([.[] | select(. == "failure")] | length), success_rate: (([.[] | select(. == "success")] | length) / length * 100)}'
```

**`--json` selects fields; `--jq` transforms them.** That pairing is what turns the CLI into an
analytics tool without any external system.

**Note `--limit 200`** — the default is far smaller, so a rate computed without it is based on a
handful of runs and swings wildly.

**Why the others fail** — B shows workflow metadata, C follows a run live, D reports API quota.

</details>

---

## Q13

Which Azure DevOps feature shows pass rate, average duration and flaky test breakdown without custom
code?

- A. The pipeline Analytics tab
- B. The Releases view
- C. The Artifacts feed
- D. The Test Plans hub

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-34.md`:** lines **279–285**.

```text
1. Pipelines > Select pipeline > Analytics tab
2. Key metrics available:
   - Pass rate (last 14/30/90 days)
   - Average duration with trend
   - Test pass rate with flaky test breakdown
   - Duration breakdown by task/stage
```

**"Duration breakdown by task/stage" is the one that pays for itself** — it tells you *where* the 45
minutes goes, which is the input Challenge 35 needs to optimise.

**Why this is worth an exam mark:** GitHub Actions has no equivalent built-in, which is why the
challenge writes a metrics collector (line 199) for GitHub and simply points at the Analytics tab for
Azure DevOps.

**Why the others fail** — B is classic releases, C is packages, D is manual test case management.

</details>

---

## Q14

Which endpoint supports custom pipeline analytics queries in Azure DevOps?

- A. The Analytics OData endpoint
- B. The Build REST API
- C. Application Insights
- D. Azure Resource Graph

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-34.md`:** lines **290–300**.

```text
# URL: https://analytics.dev.azure.com/contoso/ContosoAPI/_odata/v4.0-preview/PipelineRuns
#   ?$apply=filter(Pipeline/PipelineName eq 'contoso-api-ci' and CompletedDate ge 2024-01-01T00:00:00Z)
#   /groupby((CompletedDateSK, RunOutcome), aggregate($count as RunCount))
```

**The OData vocabulary is testable:** `$apply` with `filter(...)`, `/groupby((dimensions), aggregate(...))`.
This is the same Analytics service Challenge 04 used for DORA metrics — one endpoint, different
entity sets.

**Why the others fail**

- **B** — returns raw run data; you aggregate it yourself
- **C** — application telemetry, not pipeline data
- **D** — queries Azure resources, and Azure DevOps pipelines are not ARM resources

</details>

---

## Q15

What does the P95 duration tell you that the average does not?

- A. How long the slowest 5% of runs take
- B. The median duration
- C. The total pipeline cost
- D. The failure rate

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-34.md`:** lines **355** and **379**.

```javascript
            const p95Duration = durations.sort((a, b) => a - b)[Math.floor(durations.length * 0.95)]?.toFixed(1) || 'N/A';
```

**A single slow run can hide inside an average and dominate the P95.** With targets of 15 minutes
average and 25 minutes P95 (lines 378–379), a pipeline averaging 14 minutes but with a P95 of 55 is
failing — and it is failing in the way developers actually feel.

**Note the `?.` and `|| 'N/A'`** — with very few runs the index may not exist, and the report degrades
gracefully rather than crashing.

**Why the others fail** — B is P50, C and D are different measures entirely.

</details>

---

## Q16

The health report runs on `cron: "0 9 * * 1"`. When is that?

- A. Every Monday at 09:00 UTC
- B. Every day at 09:00 UTC
- C. The first of every month at 09:00
- D. Every hour on Mondays

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-34.md`:** line **313**.

```yaml
on:
  schedule:
    - cron: "0 9 * * 1"  # Every Monday at 9 AM UTC
  workflow_dispatch:
```

**Fields:** minute, hour, day-of-month, month, day-of-week. `1` in the fifth field is Monday.

**Why Monday morning:** the report covers the previous week and lands before planning, so pipeline
health becomes an input to what the team commits to rather than a complaint raised later.

**Note `workflow_dispatch` alongside it** — the engineering manager can pull the report on demand
without waiting for Monday.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** metrics does the weekly health report calculate? (Choose three.)

- A. Success rate
- B. P95 duration
- C. MTTR
- D. Code coverage
- E. Deployment frequency
- F. Cyclomatic complexity

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-34.md`:** lines **348**, **355**, **369**.

```javascript
            const successRate = ((succeeded.length / completed.length) * 100).toFixed(1);
            const p95Duration = durations.sort((a, b) => a - b)[Math.floor(durations.length * 0.95)]...
            const avgMTTR = mttrValues.length > 0 ? ... : 'N/A';
```

**Why the others fail**

- **D** — a code quality metric from Challenge 18
- **E** — a **DORA** metric measuring delivery to production, not pipeline health
- **F** — static analysis

**The distinction the exam draws:** pipeline health measures **the pipeline**; DORA measures **the
delivery process**. MTTR appears in both with different scopes — pipeline recovery here, service
recovery in DORA.

</details>

---

## Q18

Which **two** changes fix the misleading 100% pass rate? (Choose two.)

- A. Remove `|| true` from the test step
- B. Set `failTaskOnFailedTests: true`
- C. Remove `condition: always()` from the publish task
- D. Set `continueOnError: true`
- E. Delete the failing tests

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-34.md`:** Break & fix Exercise 1, lines **662–673**.

**Why C is deliberately wrong and worth understanding.** `condition: always()` on the **publish** task
is correct and must stay — it is what surfaces the results when tests fail, which is precisely when
you need to see them. Removing it would fix the false green and blind you to the detail.

**Publish always; fail honestly.** The two settings are independent.

**Why the others fail**

- **D** — another muzzle, this time at task level
- **E** — deleting tests to make the build green is the same failure in a more expensive form

</details>

---

## Q19

Which **two** are required to make retries useful rather than harmful? (Choose two.)

- A. Compare first-attempt and retry results to identify flaky tests
- B. Annotate or record which tests were flaky
- C. Retry the entire suite until it passes
- D. Set `continue-on-error: true` on the test job
- E. Increase `retryTimes` until failures disappear

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-34.md`:** lines **436–450** and **454–466**.

```yaml
      - name: Annotate flaky tests
        if: always() && hashFiles('flaky-tests.txt') != ''
        run: |
          while IFS= read -r test; do
            echo "::warning file=$test::This test is flaky - passed on retry without code changes"
          done < flaky-tests.txt
```

**Why the others fail — and each is a way of hiding the problem**

- **C** — Break & fix Exercise 2's broken loop. It eventually goes green and records nothing
- **D** — the build passes regardless of the outcome
- **E** — raising the retry count buries flakiness deeper. A test needing five retries is worse than
  one needing two, and the dashboard would look identical

**The principle:** *a retry must leave a trace.* Green build, visible debt.

</details>

---

## Q20

Which **three** problems appear in the broken retry loop? (Choose three.)

```yaml
- name: Run tests
  run: |
    for i in 1 2 3; do
      npx jest --ci && break || echo "Attempt $i failed, retrying..."
    done
```

- A. No tracking of which tests needed retries
- B. No annotation or reporting of flaky tests
- C. The exit code may be wrong
- D. It retries too few times
- E. It runs tests in parallel

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-34.md`:** lines **683–691** — the comments name all three.

**C is the subtle one.** The step's exit code comes from the **last command in the loop**, which is the
`echo`. So even when all three attempts fail, the step can exit 0 and the build goes green.

**And it retries the whole suite**, not just the failures — slower, and it makes it impossible to tell
which test was unreliable.

**Why the others fail** — D is wrong, three attempts is reasonable; E is not what the loop does.

</details>

---

## Q21

Which **two** permissions does the alert workflow need? (Choose two.)

- A. `actions: read`
- B. `issues: write`
- C. `contents: write`
- D. `pull-requests: write`
- E. `packages: write`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-34.md`:** lines **207–209** and **506–513**.

`actions: read` to list workflow runs; `issues: write` to create the alert issue.

**Why the others fail** — the workflow neither writes code, comments on pull requests, nor publishes
packages. Each permission it holds maps to something it actually does.

**Compare with Challenge 31's IaC workflow**, which used `pull-requests: write` for what-if comments
and `issues: write` for drift. The permission follows the **output**, and reading it off the block is
a fast way to answer these questions.

</details>

---

## Q22

Which **two** behaviours in Contoso's scenario indicate lost trust in CI? (Choose two.)

- A. Developers push directly to main to skip CI
- B. Nobody notices when the pipeline has been broken for hours
- C. Build queue times spike during peak hours
- D. The pipeline takes 45 minutes
- E. Tests are written in Jest

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-34.md`:** lines **22** and **25**.

> *"Developers push directly to main to skip CI ('it will just fail anyway')"*
> *"Nobody notices when the pipeline has been broken for hours"*

**Both are consequences, not causes**, which is why they are the trust signals. C and D are the
*causes* — slow and queued — and a 30% failure rate is what turns them into "why bother".

**The order of repair the challenge implies:** fix flakiness first (so red means something), then
add alerting (so breakage is noticed), then optimise duration — which is Challenge 35.

**Why the others fail** — C and D are symptoms of pipeline performance, E is a tooling choice.

</details>

---

## Q23

Which **two** distinguish system flaky detection from custom flaky detection in Azure DevOps? (Choose
two.)

- A. System detection identifies flaky tests automatically by re-running on the same commit
- B. Custom detection relies on tests being marked manually
- C. System detection requires `retryTimes` in Jest
- D. Custom detection only works with MSTest
- E. System detection disables failing builds

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-34.md`:** lines **163–167**.

```json
#     "flakyDetection": {
#       "isEnabled": true,
#       "flakyDetectionType": "system"
#     },
```

**The trade-off:** system detection needs no maintenance and only recognises flakiness **after** it has
happened at least twice. Custom marking is immediate and only as accurate as the team's discipline.

**Why the others fail**

- **C** — `retryTimes` is a **Jest** setting used in the GitHub path. Azure DevOps system detection is
  platform-side
- **D** — framework-agnostic; it works from published test results
- **E** — it **classifies** results. Whether a flaky test fails the build is controlled separately by
  `failTaskOnFailedTests`

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso's CI pipeline fails 30% of the time, largely from flaky tests. The team must
stop flaky tests failing builds, know exactly which tests are flaky so they can be fixed, and still
fail the build on genuine failures.

---

## Q24

**Proposed solution:** Run tests once. On failure, retry only the failed tests, compare the two result
sets, emit a `::warning::` naming any test that passed on retry, upload a flaky-test artifact, and exit
non-zero only if tests still fail.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-34.md`:** lines **413–466**.

All three requirements are met by different parts of the same block:

| Requirement | Mechanism |
|---|---|
| Flaky tests do not fail the build | Exit code driven by the **retry** result (line 450) |
| Know which tests are flaky | `::warning::` annotation + uploaded artifact (lines 444, 461) |
| Genuine failures still fail | `stillFailing.length > 0 ? 1 : 0` |

**Note it retries only the failed tests** (line 433), not the whole suite — faster, and it keeps the
comparison precise.

</details>

---

## Q25

**Proposed solution:** Wrap the test command in a loop that retries up to three times and breaks on
success.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

This is Break & fix Exercise 2 verbatim (lines 683–691), and it fails **two** of the three
requirements.

**Nothing is recorded.** No annotation, no artifact, no issue. The team still cannot answer "which
tests should we fix?", so the flakiness is permanent.

**The exit code is unreliable.** The loop's last command is the `echo`, so a run where all three
attempts failed can still exit 0 — meaning **genuine failures stop failing the build too**.

**It satisfies exactly one requirement** — flaky tests no longer fail the build — and does so by
making *all* failures stop failing the build. That is the worst outcome available.

</details>

---

## Q26

**Proposed solution:** Set `retryTimes: 2` in `jest.config.js` and publish JUnit results with
`failTaskOnFailedTests: true`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Two of three requirements are met, and the interesting one is missing.

**Flaky tests no longer fail the build** — Jest retries internally and reports a pass.
**Genuine failures still fail** — `failTaskOnFailedTests: true` is correct.

**But nobody learns which tests were flaky.** Jest's internal retry is silent in the JUnit output: a
test that failed twice and passed on the third attempt is published simply as *passed*. The dashboard
is green, the flakiness is invisible, and the 30% problem quietly persists.

**What would fix it:** either Azure DevOps **system flaky detection** (line 165), which classifies from
the platform side, or the compare-and-annotate approach from Q24.

**The recurring lesson:** `retryTimes` alone converts a visible problem into an invisible one.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — flaky tests

| # | Statement | Answer |
|---|---|---|
| 1 | A flaky test passes and fails on the same commit |  |
| 2 | Retrying without recording resolves flakiness |  |
| 3 | Azure DevOps can detect flaky tests automatically |  |
| 4 | `retryTimes` in Jest reports which tests were retried in JUnit output |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A flaky test passes and fails on the same commit | **Yes** |
| 2 | Retrying without recording resolves flakiness | **No** |
| 3 | Azure DevOps can detect flaky tests automatically | **Yes** |
| 4 | `retryTimes` in Jest reports which tests were retried in JUnit output | **No** |

**In `challenge-34.md`:** lines **31**, **144**, **165**, **94**.

Row 2 is the thesis of the challenge. Retrying changes the **symptom**; the test is just as unreliable
tomorrow, and now nobody can see it.

Row 4 is Q26's point: Jest's internal retry produces a clean "passed" in the results file. Detection
has to come from comparing runs or from platform-side classification.

</details>

---

## Q28 — build honesty

| # | Statement | Answer |
|---|---|---|
| 1 | `\|\| true` on a test step hides failures |  |
| 2 | `failTaskOnFailedTests: false` lets a build pass with failing tests |  |
| 3 | `condition: always()` on the publish task should be removed |  |
| 4 | `continueOnError: true` on a test task makes failures visible |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `\|\| true` on a test step hides failures | **Yes** |
| 2 | `failTaskOnFailedTests: false` lets a build pass with failing tests | **Yes** |
| 3 | `condition: always()` on the publish task should be removed | **No** |
| 4 | `continueOnError: true` on a test task makes failures visible | **No** |

**In `challenge-34.md`:** lines **644**, **652**, **668**.

**Row 3 is the trap inside the fix.** Two settings look similar and do opposite things:

| Setting | Effect |
|---|---|
| `condition: always()` on **publish** | Results are visible even when tests failed — **keep it** |
| `failTaskOnFailedTests: false` | The build passes despite failures — **remove it** |

Row 4: `continueOnError` turns a failure into a **warning**. The run shows partially succeeded, which
most people read as green.

</details>

---

## Q29 — metrics

| # | Statement | Answer |
|---|---|---|
| 1 | MTTR measures time from failure to the next success |  |
| 2 | P95 duration is the same as the average |  |
| 3 | Success rate above 90% is the stated target |  |
| 4 | Queue time is part of total run duration |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | MTTR measures time from failure to the next success | **Yes** |
| 2 | P95 duration is the same as the average | **No** |
| 3 | Success rate above 90% is the stated target | **Yes** |
| 4 | Queue time is part of total run duration | **Yes** |

**In `challenge-34.md`:** lines **357–366**, **355**, **377**, and the scenario at **24**.

Row 4 matters for the scenario: queue times spike between 9 and 11 AM (line 24), so a run's measured
duration includes time it spent **waiting for an agent**. Optimising the build itself will not move
that number — buying parallelism or self-hosted capacity will, which is Challenges 21 and 35.

Row 2: a healthy average with an unhealthy P95 is the most common real pattern, and the P95 is what
people remember.

</details>

---

## Q30 — alerting

| # | Statement | Answer |
|---|---|---|
| 1 | Alerting on every failure builds trust in the alert |  |
| 2 | Three consecutive failures indicates a systemic problem |  |
| 3 | An alert should have an assignee |  |
| 4 | `workflow_run` with `types: [completed]` fires on failures too |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Alerting on every failure builds trust in the alert | **No** |
| 2 | Three consecutive failures indicates a systemic problem | **Yes** |
| 3 | An alert should have an assignee | **Yes** |
| 4 | `workflow_run` with `types: [completed]` fires on failures too | **Yes** |

**In `challenge-34.md`:** lines **505–513**, **479–485**.

Row 1 is the alert-fatigue principle, and the scenario proves it in reverse: a pipeline failing 30% of
the time would generate an alert every third run. People filter those to a folder within a week, and
then genuinely nobody notices when it breaks.

Row 3: line 513 assigns `oncall-engineer`. An unassigned P1 belongs to everyone, which means nobody.

</details>

---

# Section E — Drag and drop

---

## Q31

Arrange the flaky-test detection flow in order.

**Items:** Emit a `::warning::` annotation · Run the full test suite · Compare the two result sets ·
Retry only the failed tests · Exit non-zero only if tests still fail

<details>
<summary>Show answer</summary>

### Answer

1. Run the full test suite — line **413**, `continue-on-error: true`
2. Retry only the failed tests — line **433**
3. Compare the two result sets — line **437**
4. Emit a `::warning::` annotation — line **444**
5. Exit non-zero only if tests still fail — line **450**

**Step 1 needs `continue-on-error: true`** or the job stops before the retry can happen. Step 2 retries
**only the failures**, which keeps step 3's comparison meaningful — a whole-suite retry cannot tell you
which specific test recovered.

</details>

---

## Q32

Match each metric to what it reveals.

| Metric | Reveals |
|---|---|
| Success rate |  |
| Average duration |  |
| P95 duration |  |
| MTTR |  |
| Queue time |  |
| Flaky test count |  |

**Options:** How long it stays broken · How long work waits for an agent · How much of the failure rate is noise · How often the pipeline breaks · The bad runs developers remember · Typical feedback time

<details>
<summary>Show answer</summary>

| Metric | Reveals |
|---|---|
| Success rate | **How often the pipeline breaks** |
| Average duration | **Typical feedback time** |
| P95 duration | **The bad runs developers remember** |
| MTTR | **How long it stays broken** |
| Queue time | **How long work waits for an agent** |
| Flaky test count | **How much of the failure rate is noise** |

**In `challenge-34.md`:** lines **375–382**, plus the scenario at **24**.

**Each points at a different fix.** Low success rate with high flaky count → fix the tests. Healthy
average with a bad P95 → find the outlier job. High MTTR → alerting and ownership. High queue time →
parallelism or self-hosted capacity, which is Challenge 21.

</details>

---

## Q33

Arrange these pipeline-health interventions in the order that restores trust fastest.

**Items:** Optimise pipeline duration · Detect and annotate flaky tests · Alert on consecutive
failures · Publish a weekly health report

<details>
<summary>Show answer</summary>

### Answer

1. **Detect and annotate flaky tests** — until red means something, nothing else matters
2. **Alert on consecutive failures** — so breakage is noticed (line 25's complaint)
3. **Optimise duration** — Challenge 35's subject
4. **Publish a weekly health report** — sustains the improvement and shows the trend

**Why flakiness comes first.** With a 30% failure rate driven by flaky tests, faster feedback just
delivers wrong answers sooner, and alerting on an unreliable pipeline generates noise people learn to
ignore. Reliability precedes speed.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| 100% pass rate but bugs in production |  |
| Tests pass on retry with no code change |  |
| Build goes green even when all retries fail |  |
| Nobody notices main is broken for hours |  |
| Developers push directly to main |  |

**Options:** `\|\| true` and `failTaskOnFailedTests: false` · Exit code taken from the loop's last `echo` · Flaky tests · Lost trust from a 30% failure rate · No alerting on consecutive failures

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| 100% pass rate but bugs in production | **`\|\| true` and `failTaskOnFailedTests: false`** |
| Tests pass on retry with no code change | **Flaky tests** |
| Build goes green even when all retries fail | **Exit code taken from the loop's last `echo`** |
| Nobody notices main is broken for hours | **No alerting on consecutive failures** |
| Developers push directly to main | **Lost trust from a 30% failure rate** |

**In `challenge-34.md`:** lines **644–652**, **31**, **691**, **25**, **22**.

**The middle row is the one to be able to explain.** In a shell loop, the step's exit status is that of
the **last command executed**. When every attempt fails, the last command is the `echo` in the `||`
branch, which exits 0.

</details>

---

## Q35

Match each tool to its platform.

| Capability | Platform |
|---|---|
| Built-in flaky test detection in project settings |  |
| `retryTimes` in `jest.config.js` |  |
| `gh run list --json ... --jq` analytics |  |
| Analytics OData `PipelineRuns` endpoint |  |
| `::warning::` workflow annotations |  |
| `PublishTestResults@2` with `failTaskOnFailedTests` |  |

**Options:** Azure DevOps · Azure Pipelines · GitHub · GitHub Actions · Test framework — either platform

<details>
<summary>Show answer</summary>

| Capability | Platform |
|---|---|
| Built-in flaky test detection in project settings | **Azure DevOps** |
| `retryTimes` in `jest.config.js` | **Test framework — either platform** |
| `gh run list --json ... --jq` analytics | **GitHub** |
| Analytics OData `PipelineRuns` endpoint | **Azure DevOps** |
| `::warning::` workflow annotations | **GitHub Actions** |
| `PublishTestResults@2` with `failTaskOnFailedTests` | **Azure Pipelines** |

**In `challenge-34.md`:** lines **165**, **94**, **179**, **292**, **444**, **145**.

**The asymmetry is worth stating:** Azure DevOps has **built-in** flaky detection and analytics; GitHub
Actions requires you to build both from the API. That is why the challenge writes a metrics collector
for GitHub and simply points at the Analytics tab for Azure DevOps.

</details>

---

# Section F — Hot area

---

## Q36

```javascript
module.exports = {
  testEnvironment: 'node',
  [BLANK 1]: 2,
  reporters: [
    'default',
    ['[BLANK 2]', { outputDirectory: './test-results', outputName: 'junit.xml' }]
  ]
};
```

- **BLANK 1:** `retryTimes` / `maxWorkers` / `bail` / `testTimeout`
- **BLANK 2:** `jest-junit` / `jest-html-reporter` / `default` / `jest-flaky`

<details>
<summary>Show answer</summary>

### Answer: `retryTimes`, `jest-junit`

**In `challenge-34.md`:** lines **94** and **97**.

`jest-junit` produces the JUnit XML that both `PublishTestResults@2` (line 140) and
`dorny/test-reporter` (line 81) consume — one format, both platforms.

</details>

---

## Q37

```yaml
  - task: PublishTestResults@2
    condition: [BLANK 1]
    inputs:
      testResultsFormat: "JUnit"
      [BLANK 2]: true
```

Requirement: always publish results, and fail the build when tests fail.

- **BLANK 1:** `always()` / `succeeded()` / `failed()` / `succeededOrFailed()`
- **BLANK 2:** `failTaskOnFailedTests` / `continueOnError` / `publishRunAttachments` /
  `mergeTestResults`

<details>
<summary>Show answer</summary>

### Answer: `always()`, `failTaskOnFailedTests`

**In `challenge-34.md`:** lines **668–672**.

**The two settings pull in opposite directions and both are correct.** `always()` guarantees results
are **visible** after a failure; `failTaskOnFailedTests: true` guarantees the build is **honest**
about it.

</details>

---

## Q38

```yaml
on:
  [BLANK 1]:
    workflows: ["CI Pipeline"]
    types: [completed]

jobs:
  check-health:
    if: github.event.workflow_run.[BLANK 2] == 'failure'
```

- **BLANK 1:** `workflow_run` / `workflow_call` / `workflow_dispatch` / `schedule`
- **BLANK 2:** `conclusion` / `status` / `result` / `outcome`

<details>
<summary>Show answer</summary>

### Answer: `workflow_run`, `conclusion`

**In `challenge-34.md`:** lines **478** and **485**.

`types: [completed]` fires on success **and** failure — `status` would tell you it finished,
`conclusion` tells you **how**. Same pair as Challenge 22 Q38.

</details>

---

## Q39

```yaml
      - name: Annotate flaky tests
        if: [BLANK 1] && [BLANK 2]('flaky-tests.txt') != ''
        run: |
          echo "::[BLANK 3]::This test is flaky"
```

- **BLANK 1:** `always()` / `success()` / `failure()` / `cancelled()`
- **BLANK 2:** `hashFiles` / `fileExists` / `exists` / `contains`
- **BLANK 3:** `warning` / `error` / `notice` / `debug`

<details>
<summary>Show answer</summary>

### Answer: `always()`, `hashFiles`, `warning`

**In `challenge-34.md`:** lines **455** and **458**.

`hashFiles()` returns an empty string when nothing matches — the idiomatic file-existence test.
`::warning::` is right because flakiness should be **visible without failing the build**; `::error::`
would turn recorded debt into a hard failure.

</details>

---

## Q40

```javascript
            const consecutiveFailures = recentRuns
              .filter(r => r.conclusion === 'failure').length;

            if (consecutiveFailures >= [BLANK 1]) {
              await github.rest.issues.create({
                labels: ['pipeline-health', 'urgent', 'P1'],
                [BLANK 2]: ['oncall-engineer']
              });
            }
```

- **BLANK 1:** `3` / `1` / `5` / `10`
- **BLANK 2:** `assignees` / `reviewers` / `owners` / `watchers`

<details>
<summary>Show answer</summary>

### Answer: `3`, `assignees`

**In `challenge-34.md`:** lines **505** and **513**.

One failure is usually a developer already fixing it; three in a row is systemic. And an unassigned P1
belongs to nobody.

</details>

---

## Q41

```bash
gh run list --workflow=ci.yml --limit [BLANK 1] --[BLANK 2] conclusion \
  --[BLANK 3] '[.[] | .conclusion] | {total: length, failures: (...)}'
```

- **BLANK 1:** `200` / `10` / `1` / `5`
- **BLANK 2:** `json` / `format` / `output` / `fields`
- **BLANK 3:** `jq` / `filter` / `query` / `template`

<details>
<summary>Show answer</summary>

### Answer: `200`, `json`, `jq`

**In `challenge-34.md`:** lines **184–185**.

A rate computed over the default handful of runs swings wildly; 200 gives a stable figure. `--json`
selects fields and `--jq` transforms them — the pairing that makes the CLI an analytics tool.

</details>

---

# Section G — Case study

## Case study: Contoso CI recovery

### Background

Contoso's `contoso-api-ci` pipeline averages **45 minutes** and fails **~30%** of runs. The failure
rate has climbed for three months and teams have lost confidence.

**Observed behaviours:** developers push directly to main to skip CI; the same tests pass on retry
with no code change; queue times spike between 9 and 11 AM; nobody notices when the pipeline has been
broken for hours.

### Requirements

**Reliability**

- Flaky tests must not fail the build
- The team must know exactly which tests are flaky, ranked by frequency
- Genuine failures must still fail the build

**Visibility**

- Weekly reporting on success rate, duration and MTTR
- An alert when the pipeline is systemically broken, with an owner

**Honesty**

- Test results must be published even when tests fail
- The build must never report success while tests are failing

---

## Q42

Which **two** meet the flaky-test requirements? (Choose two.)

- A. Retry only failed tests and compare result sets to classify them
- B. Annotate flaky tests and upload a flaky-test artifact
- C. Set `retryTimes: 5` in Jest
- D. Add `continue-on-error: true` to the test job
- E. Quarantine failing tests by deleting them

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-34.md`:** lines **436–450** and **454–466**.

**Why the others fail**

- **C** — retries silently. The JUnit output shows a pass, so the second requirement — knowing which
  tests are flaky — is unmet (Q26)
- **D** — the build passes on genuine failures too, breaking the third requirement
- **E** — coverage falls silently and the underlying defect ships

</details>

---

## Q43

Which configuration ensures genuine failures still fail the build?

- A. Exit non-zero when tests fail on retry
- B. `continue-on-error: true` on the test step
- C. `|| true` after the test command
- D. `failTaskOnFailedTests: false`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-34.md`:** lines **448–450**.

```javascript
              // Fail only if tests still fail on retry
              const stillFailing = r2.testResults.filter(t => t.status === 'failed');
              process.exit(stillFailing.length > 0 ? 1 : 0);
```

**Why the others fail** — all three are muzzles. B and C silence the step; D silences the publish task.
Any of them produces the 100%-pass-rate-with-production-bugs failure from Break & fix Exercise 1.

**Note the first attempt *does* use `continue-on-error: true`** (line 415) — deliberately, so the
retry can run. The honesty is enforced at the **end**, by the comparison's exit code.

</details>

---

## Q44

Which **two** meet the visibility requirements? (Choose two.)

- A. A scheduled weekly report of success rate, P95 duration and MTTR
- B. An alert on three consecutive failures, assigned to the on-call engineer
- C. An email on every failed run
- D. A dashboard nobody is required to check
- E. Enabling verbose logging

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-34.md`:** lines **311–392** and **505–513**.

**Why the others fail**

- **C** — at a 30% failure rate that is an alert every third run. People filter it within a week, and
  then genuinely nobody notices (line 25)
- **D** — a dashboard is **pull**; the requirement is to be told. The report and the alert are **push**
- **E** — more log volume, no more signal

**The pairing is deliberate:** the alert catches **acute** breakage, the weekly report catches
**gradual** decline — which is what has been happening for three months.

</details>

---

## Q45

Which **two** metrics best diagnose the 9–11 AM queue spike? (Choose two.)

- A. Queue time as a component of total duration
- B. Concurrent runs during peak hours
- C. Code coverage percentage
- D. Test pass rate
- E. Flaky test count

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-34.md`:** the scenario at line **24**, with duration measurement at lines **351–354**.

**Why this matters for the fix.** Total duration measured as `updated_at - created_at` (line 352)
**includes queue time**. So a pipeline that "takes 45 minutes" may be building for 20 and waiting for
25 — and optimising the build would move nothing.

The remedy is capacity, not speed: more parallel jobs (Challenge 21's $40/month per hosted job) or
self-hosted agents. Challenge 35 makes that trade-off explicitly.

**Why the others fail** — C, D and E measure test quality, not scheduling.

</details>

---

## Q46

Which **two** ensure test result honesty? (Choose two.)

- A. Remove `|| true` from the test command
- B. `failTaskOnFailedTests: true` on the publish task
- C. Remove `condition: always()` from the publish task
- D. `continueOnError: true` on the test task
- E. `failTaskOnMissingResultsFile: false`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-34.md`:** lines **663–673**.

**Why the others fail**

- **C** — **keep it.** `always()` on the publish task is what makes failing results **visible**.
  Removing it hides the detail exactly when it matters
- **D** — another muzzle
- **E** — **the subtle one.** With `false`, a run where the test step crashed before writing any
  results file passes silently. Line 673 sets it to `true` precisely so "no results" is treated as a
  failure rather than as nothing to report

</details>

---

## Q47

Three months after the fixes, success rate is 94% and flaky tests are annotated. But the same five
tests appear as flaky every week and nobody fixes them.

What is missing, and what should Contoso do?

- A. Ownership and prioritisation — track flaky tests as issues with owners and a quarantine policy
- B. More retries
- C. A faster pipeline
- D. Deleting the five tests

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-34.md`:** the tracking-issue pattern at lines **725–738**.

```javascript
      // Create or update tracking issue for flaky tests
      const issues = await github.rest.issues.listForRepo({ labels: 'flaky-test', state: 'open' });
      // Add comment with today's flaky occurrence
```

**Annotations are ephemeral.** A `::warning::` lives in one run's log; nobody reviews last Tuesday's
run. Converting flaky occurrences into a **tracking issue with a count** turns invisible noise into
ranked, ownable work — which is the "ranked by frequency" half of the requirement.

**A quarantine policy is the other half:** a test flaky more than N times in a week is moved out of the
blocking suite and its issue is assigned. That caps the damage while the fix is scheduled.

**Why the others fail**

- **B** — retries are already working; the tests are still unreliable
- **C** — speed does not fix correctness
- **D** — **the tempting one.** Deleting removes the noise *and* the coverage, so the untested
  behaviour ships silently. Quarantine keeps the test visible and non-blocking; deletion loses it

</details>

---

## Q48

Six months on, success rate sits at 96% and average duration at 12 minutes. But the P95 is 48 minutes
and developers still complain CI is slow.

What is happening, and what should Contoso investigate?

- A. A subset of runs is far slower — investigate the duration breakdown by job and queue time
- B. The average is calculated incorrectly
- C. The success rate target is too low
- D. Developers are exaggerating

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-34.md`:** lines **378–379** and the Analytics capability at line **285**.

**The numbers are consistent, not contradictory.** An average of 12 with a P95 of 48 means most runs
are quick and **one in twenty takes four times as long**. Developers do not experience the average —
they experience the run they are waiting on, and they remember the bad ones.

**Where to look, in order:**

1. **Duration breakdown by task or stage** (line 285) — is one job the outlier?
2. **Queue time** — the 9–11 AM spike (line 24) would produce exactly this distribution
3. **The slowest jobs across runs** — line 192's `gh run view --json jobs` sorted by duration

**Why the others fail**

- **B** — both figures come from the same array (lines 354–355)
- **C** — success rate is not the complaint
- **D** — the P95 says they are right

**The general lesson, and it is worth carrying into Challenge 35:** **averages hide tails, and users
live in the tail.** A target that only checks the mean will report health while the experience is
poor.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Retrying without recording** | Q11, Q19, Q25, Q26, Q42 | A retry must leave a trace — annotation, artifact or issue |
| **`retryTimes` assumed to report flakiness** | Q26, Q27 | Jest publishes a clean "passed". Detection needs comparison or platform-side |
| **`\|\| true` on a test step** | Q3, Q18, Q43, Q46 | Let the step fail. Publish always, fail honestly |
| **`condition: always()` removed from publish** | Q18, Q46 | Keep it. It is what makes failures visible |
| **`continue-on-error` as a fix** | Q19, Q43 | It turns a failure into a warning nobody reads |
| **Exit code from a retry loop** | Q20, Q34 | The last command's status wins — usually the `echo` |
| **Alerting on every failure** | Q30, Q44 | Alert on a pattern, with an owner |
| **Average duration read alone** | Q15, Q48 | The P95 is what developers experience |
| **Queue time forgotten** | Q29, Q45 | Total duration includes waiting for an agent |
| **`status` used instead of `conclusion`** | Q38 | `status` = finished. `conclusion` = how |
| **Deleting flaky tests** | Q42, Q47 | Quarantine keeps the coverage; deletion loses it |
| **DORA metrics confused with pipeline metrics** | Q6, Q17 | Same word MTTR, different scope |

---

# The blocks to memorise

Line numbers are in `challenge-34.md`.

```text
# 1. Pipeline health targets  (lines 375-382)
Success rate       > 90%
Average duration   < 15 min
P95 duration       < 25 min
MTTR               < 30 min

# 2. MTTR definition  (lines 357-366)
Time from the START of a failing run to the END of the next successful run, same branch.
```

```yaml
# 3. Honest test reporting  (lines 663-673) - Azure Pipelines
- script: npx jest --ci --reporters=default --reporters=jest-junit   # no '|| true'
- task: PublishTestResults@2
  condition: always()                  # publish even on failure
  inputs:
    failTaskOnFailedTests: true        # but fail the build
    failTaskOnMissingResultsFile: true

# 4. Flaky detection by comparison  (lines 413-450)
      - name: Run tests (attempt 1)
        continue-on-error: true
      - name: Identify and retry failed tests
        # retry ONLY the failures, compare the two result sets
        # passed on retry  -> ::warning:: + record
        # still failing    -> exit 1

# 5. Annotate only when data exists  (lines 454-458)
        if: always() && hashFiles('flaky-tests.txt') != ''
        run: echo "::warning file=$test::This test is flaky"

# 6. Degradation alert  (lines 478-513)
on:
  workflow_run:
    workflows: ["CI Pipeline"]
    types: [completed]
jobs:
  check-health:
    if: github.event.workflow_run.conclusion == 'failure'
    # alert at >= 3 consecutive failures, with assignees
```

```json
// 7. Azure DevOps flaky detection  (lines 163-169)
"flakySettings": {
  "flakyDetection": { "isEnabled": true, "flakyDetectionType": "system" },
  "flakyInSummaryReport": true
}
```

```bash
# 8. Failure rate from the CLI  (lines 184-185)
gh run list --workflow=ci.yml --limit 200 --json conclusion \
  --jq '[.[] | .conclusion] | {total: length, failures: (...), success_rate: (...)}'
```

**Azure DevOps has built-in flaky detection and an Analytics tab. GitHub Actions requires you to build
both from the API.**

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 34 is exam-ready. Move to Challenge 35 |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 1 and 5, then retake this |
| Below 30 | Redo the challenge, writing the four health targets from memory first |

Record your result in `AZ-400-Learning-Log.md` under Challenge 34.

:::danger The one thing

**A retry that leaves no trace converts a visible problem into a permanent one.**

Retry so flaky tests do not block the build — then annotate, record and rank them so somebody fixes
them. Green build, visible debt. Every wrong answer in this challenge skips the second half.

:::
