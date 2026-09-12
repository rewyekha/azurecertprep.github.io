---
sidebar_position: 1.5
toc_max_heading_level: 2
title: "Challenge 16: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 16 — AZ-400 exam questions

**48 questions** built only from what Challenge 16 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-16.md`**.

:::danger Read this before you start

**The testing pyramid is a statement about cost and speed, not about importance.**

**Unit tests** — fast, isolated, no external dependencies. Many of them.
**Integration tests** — an API against a **real** database. Fewer, slower, and they need infrastructure.
**Load tests** — behaviour under concurrency. Very few, slowest, and they answer a different question
entirely.

**Each layer catches what the one below cannot.** A unit test cannot find a broken SQL query; an
integration test cannot find a connection pool that exhausts at 50 concurrent users.

**Two mechanical facts decide most of this paper.**

**A test that runs but whose results are not *published* is invisible**, and a test whose failure does not
fail the job is decoration. `PublishTestResults@2` and k6's **thresholds** are what turn a run into a
gate.

**And almost every CI-only failure in this challenge is a race.** The database is not ready, the server
has not bound its port, the runner is slower than a laptop. **`sleep` is the wrong fix; a retry loop that
confirms readiness is the right one.**

The scenario at line 23: three deployments a day, **no automated tests**, and **four regressions in a
month** breaking production checkout.

:::

---

# Section A — Multiple choice

---

## Q1

What does `fail-fast: false` accomplish in a matrix strategy?

- A. It stops the workflow if a combination is invalid
- B. It runs the matrix combinations sequentially
- C. It suppresses reporting for failed combinations
- D. It lets every combination finish when one fails

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-16.md`:** lines **55–58**.

```yaml
    strategy:
      matrix:
        node-version: [18, 20, 22]
      fail-fast: false
```

**By default GitHub cancels every in-progress matrix job the moment one fails.** For a compatibility
matrix that is exactly wrong — you want to know **which** versions pass.

**Read line 571's reasoning**: testing across three Node versions is pointless if a failure on 18 hides
whether 20 and 22 are fine. **One run should give you the full compatibility picture.**

**Why B is a different setting.** Parallelism is controlled by `max-parallel`, not by `fail-fast`.

</details>

---

## Q2

Which configuration ensures the PostgreSQL service container is ready before steps run?

- A. `options:` with `--health-cmd` and `--health-retries`
- B. `ports: ['5432:5432']` published on the service
- C. `needs: postgres` declared on the job
- D. `services.postgres.ready: true` on the job

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-16.md`:** lines **146–150**.

```yaml
        options: >-
          --health-cmd="pg_isready -U contoso_test"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
```

**`options:` passes Docker health-check arguments**, and GitHub Actions waits for the health check to
pass before starting the job's steps.

**Why B is necessary and not sufficient.** Publishing the port makes the database **reachable**; it says
nothing about whether it is **accepting connections yet**.

**And note the honest caveat at line 499**: even with health checks, the challenge still adds an explicit
wait loop — because the container being healthy is not the same as *your* test database being ready.

</details>

---

## Q3

Which Azure Pipelines task publishes JUnit results to the Tests tab?

- A. `PublishPipelineArtifact@1`
- B. `PublishCodeCoverageResults@2`
- C. `PublishTestResults@2`
- D. `DownloadBuildArtifacts@1`

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-16.md`:** lines **390–395**.

```yaml
          - task: PublishTestResults@2
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: '**/junit.xml'
              mergeTestResults: true
```

**Three publish tasks, three different payloads**, and the exam offers all of them: test **results**,
code **coverage** (line 398), and generic **artifacts** (line 467).

**`mergeTestResults: true` is what makes the matrix readable** — three Node versions produce three JUnit
files, and merging them gives one test run rather than three competing ones.

**And `condition: always()` at line 396 is essential.** Without it the publish is skipped when tests fail
— which is exactly the run whose results you need.

</details>

---

## Q4

What is the primary purpose of k6 thresholds in CI?

- A. To set the number of virtual users per stage
- B. To define pass/fail criteria for k6's exit code
- C. To configure how long each stage runs
- D. To specify which endpoints the script tests

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-16.md`:** lines **204–208** and **604**.

```javascript
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.01'],
    checks: ['rate>0.99'],
  },
```

**A breached threshold makes k6 exit non-zero, which fails the step.** That is the difference between a
performance **report** and a performance **gate**.

**Why A and C are the `stages` block above it** (lines 199–203) — that is the *load profile*; thresholds
are the *acceptance criteria*.

**And the three thresholds cover three failure modes**: latency (P95 and P99), error rate, and functional
correctness under load via `checks`.

</details>

---

## Q5

Integration tests fail in CI with `ECONNREFUSED 127.0.0.1:5432` but pass locally. What is the cause?

- A. The service port is not published to the host
- B. The database password in the secret is wrong
- C. The container is up but not accepting connections yet
- D. PostgreSQL is not installed on the hosted runner image

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-16.md`:** lines **495–499**.

```text
The PostgreSQL service container takes time to initialize. Even though the workflow defines health
checks, the application migration step begins before the database accepts connections.
```

**"Passes locally" is the diagnostic phrase.** A developer's database has been running for days; the CI
one was created seconds ago.

**And the fix at lines 515–521 is a readiness loop, not a longer sleep:**

```bash
          for i in $(seq 1 30); do
            pg_isready -h localhost -p 5432 -U contoso_test && break
            sleep 2
          done
```

**A loop that *checks* adapts to a slow runner; a fixed `sleep` is a guess that fails under load.**

</details>

---

## Q6

Unit tests fail on Node.js 22 with `TypeError: fetch is not defined`. What is the cause and fix?

- A. Node 22 removed the global `fetch` API entirely
- B. The matrix is misconfigured for Node 22
- C. `npm ci` failed to install the `undici` polyfill
- D. A polyfill collides with built-in `fetch`; guard it

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-16.md`:** lines **501–503** and **527–534**.

```javascript
// jest.setup.js
if (typeof globalThis.fetch === 'undefined') {
  const { fetch, Headers, Request, Response } = require('undici');
  globalThis.fetch = fetch;
  ...
}
```

**The guard makes the setup version-agnostic**, which is the whole point of testing across a matrix: the
code must work on all three, not be tuned to one.

**Why A is the opposite of the truth.** Newer Node versions **added** `fetch` as a global — the polyfill
was written for older ones and now collides.

**And this is the value the matrix delivers.** A single-version pipeline on Node 20 would never have found
it, and the failure would appear when someone upgraded production.

</details>

---

## Q7

Load tests report a 100% failure rate although the application starts. What is the cause?

- A. The thresholds are set far too strictly
- B. k6 starts before the server binds the port
- C. The database has not been seeded with data
- D. k6 is not installed on the hosted runner

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-16.md`:** lines **505–507**.

```text
The load test job starts k6 before the Express server finishes binding to port 3000. The `sleep 5` is
insufficient on CI runners under load.
```

**Same class of bug as Q5, in a different place** — a fixed delay standing in for a readiness check.

**And the fix at lines 541–551 is the pattern to copy**, with a detail worth noticing:

```bash
          done
          curl -f http://localhost:3000/health || exit 1
```

**The final `curl -f` outside the loop is what makes it fail correctly.** Without it, a loop that exhausts
all 30 attempts falls through silently and k6 runs against a dead server anyway.

</details>

---

## Q8

What does `npm start &` followed by a readiness loop accomplish?

- A. It backgrounds the server, then waits for a response
- B. It runs the server inside a service container
- C. It restarts the server whenever it exits
- D. It runs the server only after the tests finish

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-16.md`:** lines **274–277** and **541–551**.

```bash
          npm start &
          sleep 5
          curl --retry 10 --retry-delay 2 --retry-connrefused http://localhost:3000/health
```

**The `&` is required** — without it the step blocks forever, because `npm start` never exits.

**And `--retry-connrefused` is the flag that matters here.** By default `curl --retry` does not retry a
**connection refused**, which is precisely the error you get from a server that has not bound yet.

**Load tests need a running application**, which is why this job also runs migrations and seeds (lines
266–271) — a load test against an empty database measures the wrong thing.

</details>

---

## Q9

What does `coverageThreshold` in the Jest config enforce?

- A. It reports coverage figures in the test output
- B. It sets the format of the coverage report
- C. It excludes files from the coverage measure
- D. Jest exits non-zero when a metric falls below 80%

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-16.md`:** lines **109–116**.

```javascript
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
```

**Four metrics, not one**, and **branches** is the demanding one — a function with an `if/else` can be
100% line-covered by a test that only ever takes one path (Challenge 18).

**Why B and C are the neighbouring settings.** `collectCoverageFrom` (line 104) decides *what* is
measured, `coverageReporters` (line 117) decides the *output format*, and only `coverageThreshold` makes
it **fail**.

</details>

---

## Q10

Why does the config list `cobertura` among the coverage reporters?

- A. It is more accurate than the lcov format
- B. It is required by Jest for any coverage report
- C. Azure Pipelines' coverage task reads Cobertura
- D. It produces the browsable HTML report

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-16.md`:** lines **117** and **398–400**.

```javascript
  coverageReporters: ['text', 'lcov', 'cobertura'],
```

```yaml
          - task: PublishCodeCoverageResults@2
            inputs:
              summaryFileLocation: '$(System.DefaultWorkingDirectory)/coverage/cobertura-coverage.xml'
```

**Three reporters for three consumers**: `text` for the human reading the log, `lcov` for GitHub tooling
and the uploaded artifact (line 91), `cobertura` for Azure Pipelines.

**Which is the practical lesson for a cross-platform repository** — emit every format your consumers need
in one run, rather than running the tests twice.

</details>

---

## Q11

Why does the unit-test job upload coverage only when `matrix.node-version == 20`?

- A. Coverage collection only works on Node 20
- B. Three identical reports are redundant; one suffices
- C. Node 18 and 22 do not produce coverage output
- D. It reduces the overall test execution time noticeably

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-16.md`:** lines **86–91**.

```yaml
      - name: Upload coverage report
        if: matrix.node-version == 20
```

**The matrix exists to prove **compatibility**, so every leg runs the tests — but coverage of the same
source is identical across legs.** Uploading three artifacts named the same would collide anyway.

**Contrast with the test-results upload at lines 79–84**, which runs for **every** leg and names the
artifact after the version:

```yaml
          name: test-results-node-${{ matrix.node-version }}
```

**Per-leg results, one canonical coverage.** That asymmetry is deliberate and testable.

</details>

---

## Q12

What does `if: always()` on the test-results upload accomplish?

- A. Results upload even when the test step failed
- B. It retries the upload until it succeeds
- C. It uploads on a fixed schedule regardless
- D. It ignores errors from the upload action

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-16.md`:** lines **79–80**.

```yaml
      - name: Upload test results
        if: always()
```

**Without it the step is skipped on failure**, because a step's default condition is `success()` — and the
failing run is the only one whose results anyone needs.

**The same reasoning drives `condition: always()`** on Azure Pipelines' `PublishTestResults@2` (line 396).
**Both platforms default to skipping after a failure, and both must be overridden for reporting steps.**

**And `always()` also covers cancellation**, which `succeededOrFailed()` does not (Challenge 43 Q38).

</details>

---

## Q13

Why does the integration test use `--forceExit`?

- A. It skips the `afterAll` cleanup hooks after the run
- B. It runs the integration suite faster
- C. It ignores any failing integration tests
- D. Jest exits despite an open handle keeping it alive

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-16.md`:** line **176**.

```bash
npx jest --config jest.integration.config.js --ci --forceExit
```

**Integration tests hold real resources**, and a connection pool that is not closed keeps Node's event
loop alive — so the job hangs until the runner times out.

**It is a pragmatic flag with a real cost, and that is worth knowing.** `--forceExit` hides a genuine
resource leak; the cleaner fix is closing the pool in an `afterAll` hook. **In CI it prevents a hang; in
development it hides a bug.**

**And `--ci` changes snapshot behaviour** — snapshots are not written on the fly, so a missing snapshot
fails rather than silently being created.

</details>

---

## Q14

What does the k6 `stages` block define?

- A. Only the total test duration in three parts
- B. Three separate tests run one after another
- C. A ramp: 30s to 20 users, 1m to 50, 30s to zero
- D. The windows over which thresholds are evaluated

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-16.md`:** lines **199–203**.

```javascript
  stages: [
    { duration: '30s', target: 20 },
    { duration: '1m', target: 50 },
    { duration: '30s', target: 0 },
  ],
```

**Ramp up, hold, ramp down** — and each `target` is a **virtual user count** the stage moves toward, not
a constant.

**The ramp-down matters more than it looks.** Dropping from 50 to 0 instantly leaves in-flight requests
unfinished, which shows up as errors that are an artifact of the test rather than of the application.

</details>

---

## Q15

What does `checks: ['rate>0.99']` in the thresholds measure?

- A. The distribution of HTTP status codes in the responses
- B. The share of functional `check()` assertions passed
- C. The request duration at the 99th percentile
- D. The number of virtual users active at once

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-16.md`:** lines **207** and **222–225**.

```javascript
  check(cartRes, {
    'cart created': (r) => r.status === 201,
    'cart has id': (r) => r.json('id') !== undefined,
  });
```

**This is the correctness dimension of a load test**, and it is the one people omit. `http_req_failed`
counts transport failures; **`checks` counts responses that arrived and were wrong.**

**An API returning 200 with an empty body under load passes `http_req_failed` and fails `checks`** — which
is exactly the kind of degradation that breaks a checkout flow (line 23).

</details>

---

## Q16

How does the Azure Pipelines equivalent express the Node version matrix?

- A. A list of values under the `strategy.matrix` key
- B. Three separate jobs, one per version
- C. A pipeline parameter with three values
- D. Named matrix entries, each setting `nodeVersion`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-16.md`:** lines **366–373**.

```yaml
        strategy:
          matrix:
            Node18:
              nodeVersion: '18.x'
            Node20:
              nodeVersion: '20.x'
```

**Azure Pipelines matrices are a **map of named legs to variables**; GitHub's are a **list of values**.**
Same concept, materially different syntax, and the exam swaps them.

**The named key becomes the job's display name**, which is why `Node18` reads well in the run summary —
and why the variable is referenced as `$(nodeVersion)` (line 378), not as a matrix expression.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** layers make up the testing pyramid in this challenge? (Choose three.)

- A. Unit tests — fast, isolated, no external dependencies
- B. Manual exploratory tests run before each release
- C. Integration tests — endpoints against real PostgreSQL
- D. Security scans of the built container image
- E. Load tests — a performance baseline under traffic
- F. Smoke tests run in production after each deploy

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-16.md`:** lines **29–31**.

**Three layers, three costs.** Unit tests run in seconds with nothing else present; integration tests need
a database, migrations and seed data; load tests need a running application and several minutes.

**And each catches a class the layer below cannot.** A unit test cannot detect a broken migration; an
integration test cannot detect a pool that exhausts at 50 concurrent users.

**Why the ordering matters in the workflow** (lines 134, 236): `needs:` chains them so the expensive layer
never runs after a cheap failure.

</details>

---

## Q18

Which **three** does the unit-test job produce or configure? (Choose three.)

- A. A PostgreSQL service container with health check
- B. A matrix across Node 18, 20 and 22 in parallel
- C. A running application server on port 3000
- D. JUnit XML results uploaded per matrix leg
- E. k6 thresholds on latency and error rate
- F. An lcov coverage report uploaded once

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-16.md`:** lines **55–91**.

**A, C and E belong to the layers above** — the service container to integration (line 137), the server
and k6 to load (lines 273–293).

**And that separation is the pyramid expressed as infrastructure.** The unit job needs **nothing** beyond
Node, which is why it runs three times in parallel and finishes first.

</details>

---

## Q19

Which **three** does the integration test job require that the unit job does not? (Choose three.)

- A. A PostgreSQL service container with a health check
- B. A matrix strategy across Node versions
- C. Database migrations run before the tests
- D. k6 installed and available on the runner
- E. Seed data loaded after the migration step
- F. `fail-fast: false` set on the matrix strategy

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-16.md`:** lines **137–150** and **165–173**.

```yaml
      - name: Run database migrations
        run: npx knex migrate:latest
      - name: Seed test data
        run: npx knex seed:run
```

**Three preconditions, and the order is not optional.** A container that is not ready fails the migration;
a schema that does not exist fails the seed; unseeded data makes the tests assert against nothing.

**This is the cost the pyramid is describing.** Every integration test carries this setup, which is why
there are fewer of them than unit tests.

</details>

---

## Q20

Which **two** turn a test run into a **gate** rather than a report? (Choose two.)

- A. `PublishTestResults@2` to the Tests tab
- B. Uploading the results as build artifacts
- C. `coverageThreshold` in the Jest config
- D. Posting a summary comment on the PR
- E. k6 `thresholds` producing a non-zero exit

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-16.md`:** lines **109–116** and **204–208**.

**Both cause a **non-zero exit**, which fails the step.** That is the mechanism; everything else in this
challenge reports.

**Why A, B and D are all valuable and none of them blocks.** Publishing makes results visible in the Tests
tab, artifacts preserve them, and the PR comment (lines 331–344) puts a summary where reviewers are.
**Visibility is not enforcement** — and as always, even a failing job gates only when it is a required
check (Challenge 08).

</details>

---

## Q21

Which **two** correctly fix a CI-only readiness failure? (Choose two.)

- A. A retry loop polling `pg_isready` before migrations
- B. Increasing `sleep` from 5 to 30 seconds
- C. Re-running the job automatically on failure
- D. A `/health` poll before k6, ending in `curl -f`
- E. Removing the health check from the service

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-16.md`:** lines **515–521** and **541–551**.

**Both **check a condition** rather than waiting a guessed interval**, so they adapt to a runner that is
having a bad day.

**Why B fails in both directions.** It is too short when the runner is loaded and wastes 25 seconds on
every run when it is not — and the failure it produces is intermittent, which is the hardest kind to
diagnose (Challenge 34).

**Why C hides the bug and doubles the cost**, and E removes the only synchronisation that exists.

</details>

---

## Q22

Which **two** are true of the `report-results` job? (Choose two.)

- A. It runs on every push to any branch
- B. It needs all three test jobs to complete
- C. It blocks the merge when tests fail
- D. It requires `pull-requests: write`
- E. It replaces `PublishTestResults@2`

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-16.md`:** lines **312–316**.

```yaml
    needs: [unit-tests, integration-tests, load-tests]
    if: github.event_name == 'pull_request'
    permissions:
      pull-requests: write
```

**`needs:` all three so the summary is complete; `pull-requests: write` because it posts a comment.**

**And the `if:` at line 313 restricts it to pull requests**, because there is no PR to comment on for a
push to `main`.

**Why C is the recurring distinction.** A comment informs. **Nothing here refuses a merge** — that comes
from the test jobs failing and being required checks.

</details>

---

## Q23

Which **two** Azure Pipelines tasks does the equivalent pipeline use for reporting? (Choose two.)

- A. `PublishTestResults@2` with `mergeTestResults: true`
- B. `PublishPipelineArtifact@1` for the JUnit results
- C. `DownloadBuildArtifacts@1` for the coverage file
- D. `PublishCodeCoverageResults@2` on one matrix leg
- E. `PublishBuildArtifacts@1` for the coverage report

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-16.md`:** lines **390–401**.

```yaml
          - task: PublishCodeCoverageResults@2
            inputs:
              summaryFileLocation: '.../coverage/cobertura-coverage.xml'
            condition: eq(variables['nodeVersion'], '20.x')
```

**The condition mirrors the GitHub Actions `if:` at line 87** — one canonical coverage report from the
matrix, not three (Q11).

**Why B is the wrong task for the payload.** An artifact preserves a file; **`PublishTestResults@2`
parses** it into the Tests tab with pass and fail counts, history and flaky-test detection.

**And `PublishPipelineArtifact@1` is used correctly** at line 467 — for the k6 JSON, which nothing parses.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must ensure nothing reaches production without automated tests at every layer,
across three daily deployments, after four regressions in a month broke checkout.

---

## Q24

**Proposed solution:** Add a unit-test job with a Node 18/20/22 matrix and `fail-fast: false`, publishing
JUnit results per leg and one coverage report, with an 80% Jest `coverageThreshold`. Add an integration
job needing the unit job, with a PostgreSQL service container carrying a `pg_isready` health check, an
explicit wait loop, migrations and seed data. Add a load job needing integration, starting the server and
polling `/health` until it responds before running k6 with latency, error-rate and check thresholds.
Publish all results and post a PR summary.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-16.md`:** lines **52–91**, **109–116**, **133–188**, **234–303**, **310–344**.

| Layer | What it catches | Gate |
|---|---|---|
| Unit, three Node versions | Logic errors and version incompatibilities | Coverage threshold, test failures |
| Integration with a real database | Broken queries, migrations, contracts | Test failures |
| Load with thresholds | Latency and error-rate regressions | k6 exit code |

**The wait loops are the clause that makes it *reliable***, not merely correct. A pipeline that fails
intermittently on readiness gets re-run reflexively, and re-running is how a real failure gets ignored.

</details>

---

## Q25

**Proposed solution:** Add one job that runs unit and integration tests together on Node 20 against a
shared staging database. Upload results as artifacts. Run load tests manually before each release. Use
`sleep 30` to wait for the database.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures.**

**A shared staging database makes tests non-deterministic.** Two concurrent runs mutate the same rows, so
a failure means "someone else's test was running", and the team learns to re-run rather than
investigate. The service container exists so each run gets a **fresh, isolated** database.

**One Node version misses the compatibility class entirely** — the `fetch` collision on Node 22 (line
483) is invisible until production upgrades.

**Manual load tests before a release cannot keep up with three deployments a day** (line 23), and they are
skipped first when a release is urgent.

**And `sleep 30` is a guess** (Q21): too short under load, wasted otherwise, and producing intermittent
failures either way.

</details>

---

## Q26

**Proposed solution:** Add the matrix unit job with `fail-fast: false`, coverage thresholds and per-leg
JUnit publishing. Add the integration job with a service container, health check, wait loop, migrations
and seeds. Add the load job with a readiness poll and k6. Publish everything and post a PR summary.
Because load tests add several minutes to every pull request, remove the k6 `thresholds` block so the
job reports timings without ever failing the run.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**Without thresholds, k6 always exits zero** (Q4). The job runs, produces a JSON artifact nobody opens,
and consumes the same several minutes it was supposed to justify — **the cost with none of the benefit.**

**And it removes the only automated performance signal in the pipeline.** The CTO's mandate at line 25 is
"passing automated tests at every layer"; a layer that cannot fail is not a test.

**The stated reasoning also misdiagnoses the cost.** The complaint is **duration**, and the fix for
duration is to move the load test off the PR path — run it on merge to `main`, or nightly, or as a
deployment gate (Challenge 17) — **keeping the thresholds** so it still fails when performance regresses.

**Deleting the acceptance criteria to make a slow test cheaper leaves a slow test that proves nothing.**

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — matrix strategy

| # | Statement | Answer |
|---|---|---|
| 1 | `fail-fast: false` lets every combination finish |  |
| 2 | The default cancels in-progress legs when one fails |  |
| 3 | `fail-fast: false` makes the matrix run sequentially |  |
| 4 | Azure Pipelines matrices are named entries setting variables |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `fail-fast: false` lets every combination finish | **Yes** |
| 2 | The default cancels in-progress legs when one fails | **Yes** |
| 3 | `fail-fast: false` makes the matrix run sequentially | **No** |
| 4 | Azure Pipelines matrices are named entries setting variables | **Yes** |

**In `challenge-16.md`:** lines **58**, **571**, **58**, **366–373**.

Row 3 confuses `fail-fast` with `max-parallel`; row 4 is the syntax difference the exam swaps (Q16).

</details>

---

## Q28 — service containers

| # | Statement | Answer |
|---|---|---|
| 1 | `options:` passes Docker health-check arguments |  |
| 2 | A health check guarantees the test database is ready |  |
| 3 | Each run gets a fresh, isolated database |  |
| 4 | Migrations and seeding must run before the tests |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `options:` passes Docker health-check arguments | **Yes** |
| 2 | A health check guarantees the test database is ready | **No** |
| 3 | Each run gets a fresh, isolated database | **Yes** |
| 4 | Migrations and seeding must run before the tests | **Yes** |

**In `challenge-16.md`:** lines **146**, **499**, **137–143**, **165–173**.

Row 2 is the challenge's own caveat, and it is why the explicit wait loop exists despite the health check.

Row 3 is why a service container beats a shared staging database (Q25).

</details>

---

## Q29 — reporting and gating

| # | Statement | Answer |
|---|---|---|
| 1 | `if: always()` uploads results even when tests failed |  |
| 2 | `PublishTestResults@2` parses JUnit into the Tests tab |  |
| 3 | Uploading an artifact makes a test a gate |  |
| 4 | k6 thresholds fail the step on breach |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `if: always()` uploads results even when tests failed | **Yes** |
| 2 | `PublishTestResults@2` parses JUnit into the Tests tab | **Yes** |
| 3 | Uploading an artifact makes a test a gate | **No** |
| 4 | k6 thresholds fail the step on breach | **Yes** |

**In `challenge-16.md`:** lines **80**, **390–393**, **79–84**, **604**.

Row 3 is the recurring split: **visibility is not enforcement** (Q20).

Row 1 applies on both platforms — `always()` in Actions, `condition: always()` in Azure Pipelines.

</details>

---

## Q30 — CI-only failures

| # | Statement | Answer |
|---|---|---|
| 1 | A fixed `sleep` is an adequate readiness mechanism |  |
| 2 | `curl --retry-connrefused` retries a refused connection |  |
| 3 | A polyfill can conflict with a newer runtime's built-in |  |
| 4 | `npm start &` is required so the step does not block |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A fixed `sleep` is an adequate readiness mechanism | **No** |
| 2 | `curl --retry-connrefused` retries a refused connection | **Yes** |
| 3 | A polyfill can conflict with a newer runtime's built-in | **Yes** |
| 4 | `npm start &` is required so the step does not block | **Yes** |

**In `challenge-16.md`:** lines **507**, **277**, **503**, **275**.

Row 1 is the cause of two of the three break symptoms.

Row 2 is the flag that makes `curl --retry` useful against a server that has not bound yet — without it,
the first refusal is fatal.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each test layer to what only it can catch.

| Layer | Catches |
|---|---|
| Unit tests |  |
| Unit tests across a version matrix |  |
| Integration tests |  |
| Load tests |  |
| `checks` inside a load test |  |
| Coverage thresholds |  |

**Options:** Broken queries, migrations and API contracts · Code nobody exercised at all · Latency and error-rate regressions under concurrency · Logic errors, in isolation, in seconds · Responses that arrive and are wrong · Runtime incompatibilities, such as a polyfill collision

<details>
<summary>Show answer</summary>

| Layer | Catches |
|---|---|
| Unit tests | **Logic errors, in isolation, in seconds** |
| Unit tests across a version matrix | **Runtime incompatibilities, such as a polyfill collision** |
| Integration tests | **Broken queries, migrations and API contracts** |
| Load tests | **Latency and error-rate regressions under concurrency** |
| `checks` inside a load test | **Responses that arrive and are wrong** |
| Coverage thresholds | **Code nobody exercised at all** |

**In `challenge-16.md`:** lines **29–31**, **483**, **165–178**, **204–206**, **207**, **109–116**.

**Read it as a ladder of cost.** Each rung needs more infrastructure and more time, and each finds a class
the rung below is structurally blind to.

</details>

---

## Q32

Match each configuration to its effect.

| Configuration | Effect |
|---|---|
| `fail-fast: false` |  |
| `options: --health-cmd=...` |  |
| `coverageThreshold` |  |
| k6 `thresholds` |  |
| `if: always()` |  |
| `--forceExit` |  |

**Options:** Every matrix leg finishes · Jest exits despite an open handle · Non-zero exit below 80% · Non-zero exit on a breached SLO · Reporting steps run after a failure · The job waits for the container's health check

<details>
<summary>Show answer</summary>

| Configuration | Effect |
|---|---|
| `fail-fast: false` | **Every matrix leg finishes** |
| `options: --health-cmd=...` | **The job waits for the container's health check** |
| `coverageThreshold` | **Non-zero exit below 80%** |
| k6 `thresholds` | **Non-zero exit on a breached SLO** |
| `if: always()` | **Reporting steps run after a failure** |
| `--forceExit` | **Jest exits despite an open handle** |

**In `challenge-16.md`:** lines **58**, **146**, **109**, **204**, **80**, **176**.

**Two of these six cause a failure; the rest change what runs or what is collected.** Knowing which is
which is most of Section B.

</details>

---

## Q33

Arrange the integration test job's steps.

**Items:** Run the integration tests · Wait for PostgreSQL to accept connections · Seed test data · Run
database migrations · Install dependencies

<details>
<summary>Show answer</summary>

### Answer

1. Install dependencies — lines **162–163**
2. Wait for PostgreSQL to accept connections — lines **515–521**
3. Run database migrations — lines **165–168**
4. Seed test data — lines **170–173**
5. Run the integration tests — lines **175–180**

**Step 2 is the one absent from the original workflow and added by the fix**, and its position is the
whole point: **before** the first command that touches the database.

**And steps 3 and 4 cannot swap.** Seeding inserts rows into tables the migration creates, so a reversed
order fails with a missing relation — which reads like a broken seed script rather than an ordering bug.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| `ECONNREFUSED 127.0.0.1:5432` in CI only |  |
| `TypeError: fetch is not defined` on Node 22 |  |
| 100% load-test failure with the app running |  |
| The job hangs after integration tests pass |  |
| No results in the Tests tab after a failure |  |
| A slow load test that never fails |  |

**Options:** An open handle; no `--forceExit` or `afterAll` cleanup · An unguarded polyfill colliding with the built-in · `condition: always()` missing on the publish task · k6 started before the server bound its port · Migrations started before the container was ready · No `thresholds` block

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| `ECONNREFUSED 127.0.0.1:5432` in CI only | **Migrations started before the container was ready** |
| `TypeError: fetch is not defined` on Node 22 | **An unguarded polyfill colliding with the built-in** |
| 100% load-test failure with the app running | **k6 started before the server bound its port** |
| The job hangs after integration tests pass | **An open handle; no `--forceExit` or `afterAll` cleanup** |
| No results in the Tests tab after a failure | **`condition: always()` missing on the publish task** |
| A slow load test that never fails | **No `thresholds` block** |

**In `challenge-16.md`:** lines **497**, **503**, **507**, **176**, **396**, **204**.

**The first three all pass on a developer machine**, which is the signature of a race: the local
environment is warm and the runner is cold.

</details>

---

## Q35

Match each artifact to its consumer.

| Artifact | Consumer |
|---|---|
| `junit-results.xml` |  |
| `coverage/lcov.info` |  |
| `coverage/cobertura-coverage.xml` |  |
| `load-test-results.json` |  |
| The PR summary comment |  |
| k6's exit code |  |

**Options:** A human reviewer · A pipeline artifact — nothing parses it · GitHub tooling and the uploaded artifact · `PublishCodeCoverageResults@2` · `PublishTestResults@2` and the Tests tab · The pipeline — this is the gate

<details>
<summary>Show answer</summary>

| Artifact | Consumer |
|---|---|
| `junit-results.xml` | **`PublishTestResults@2` and the Tests tab** |
| `coverage/lcov.info` | **GitHub tooling and the uploaded artifact** |
| `coverage/cobertura-coverage.xml` | **`PublishCodeCoverageResults@2`** |
| `load-test-results.json` | **A pipeline artifact — nothing parses it** |
| The PR summary comment | **A human reviewer** |
| k6's exit code | **The pipeline — this is the gate** |

**In `challenge-16.md`:** lines **77**, **91**, **400**, **302**, **331–344**, **604**.

**Five of the six inform; one decides.** The k6 exit code is the only row that can stop a deployment, and
it exists only because thresholds are configured.

</details>

---

# Section F — Hot area

---

## Q36

```yaml
    strategy:
      matrix:
        node-version: [18, 20, 22]
      [BLANK 1]: false

    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          [BLANK 2]: 'npm'
```

- **BLANK 1:** `continue-on-error` / `max-parallel` / `fail-fast` / `strict`
- **BLANK 2:** `registry-url` / `cache` / `scope` / `always-auth`

<details>
<summary>Show answer</summary>

### Answer: `fail-fast`, `cache`

**In `challenge-16.md`:** lines **58** and **68**.

**`cache: 'npm'` is a real saving across three legs** — it caches the npm download directory keyed on the
lockfile, so `npm ci` fetches nothing on a repeat run.

**And `continue-on-error` is the distractor that does something different and worse.** It would mark a
failing leg as **successful**, so the job goes green with broken tests — whereas `fail-fast: false` lets
the leg fail honestly while the others finish.

</details>

---

## Q37

```yaml
    services:
      postgres:
        image: postgres:16
        ports:
          - [BLANK 1]
        options: >-
          --health-cmd="[BLANK 2] -U contoso_test"
          --health-interval=10s
          --health-retries=5
```

- **BLANK 1:** `5432` / `localhost:5432` / `0:5432` / `5432:5432`
- **BLANK 2:** `psql` / `pg_isready` / `pg_ctl` / `healthcheck`

<details>
<summary>Show answer</summary>

### Answer: `5432:5432`, `pg_isready`

**In `challenge-16.md`:** lines **144–147**.

**`pg_isready` is PostgreSQL's purpose-built readiness probe** — it returns an exit code without opening a
session, which is what a health check needs.

**And `psql` would be the wrong tool** even though it would work: it needs a database to connect to and
authenticates, so a transient auth failure would report the container as unhealthy.

</details>

---

## Q38

```javascript
  coverageThreshold: {
    global: {
      [BLANK 1]: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
  coverageReporters: ['text', 'lcov', '[BLANK 2]'],
```

Requirement: enforce the hardest coverage metric, and emit a format Azure Pipelines can publish.

- **BLANK 1:** `files` / `paths` / `branches` / `modules`
- **BLANK 2:** `html` / `json` / `clover` / `cobertura`

<details>
<summary>Show answer</summary>

### Answer: `branches`, `cobertura`

**In `challenge-16.md`:** lines **111** and **117**.

**Branch coverage is the demanding metric** (Q9): 100% line coverage is achievable by a test that never
takes the `else` path.

**And `cobertura` is specifically what `PublishCodeCoverageResults@2` consumes** (line 400) — `lcov` is
the format the GitHub side uses, which is why both are emitted.

</details>

---

## Q39

```javascript
export const options = {
  stages: [
    { duration: '30s', target: 20 },
    { duration: '1m', target: 50 },
    { duration: '30s', target: [BLANK 1] },
  ],
  thresholds: {
    http_req_duration: ['[BLANK 2]<500', 'p(99)<1000'],
    http_req_failed: ['[BLANK 3]<0.01'],
  },
};
```

- **BLANK 1:** `50` / `0` / `100` / `20`
- **BLANK 2:** `avg` / `max` / `p(95)` / `med`
- **BLANK 3:** `count` / `sum` / `pct` / `rate`

<details>
<summary>Show answer</summary>

### Answer: `0`, `p(95)`, `rate`

**In `challenge-16.md`:** lines **202** and **205–206**.

**`p(95)` rather than `avg` is the same principle as Challenge 50**: an average hides the slow tail, and
the tail is what users complain about.

**`rate<0.01` is a proportion** — under 1% of requests failing — not a count, so the threshold holds
regardless of how many requests the run makes.

**And `target: 0` is the ramp-down** (Q14), which prevents in-flight requests being counted as failures.

</details>

---

## Q40

```bash
          npm start &
          for i in $(seq 1 30); do
            if curl -s http://localhost:3000/health > /dev/null 2>&1; then break; fi
            sleep 2
          done
          curl [BLANK 1] http://localhost:3000/health || [BLANK 2]
```

- **BLANK 1:** `-s` / `-f` / `-v` / `-L`
- **BLANK 2:** `true` / `continue` / `exit 1` / `echo "warning"`

<details>
<summary>Show answer</summary>

### Answer: `-f`, `exit 1`

**In `challenge-16.md`:** line **551**.

**`curl -f` makes an HTTP error status a non-zero exit** — without it, `curl` happily returns 0 after
printing a 500 page, so the check passes against a broken server.

**And the final `|| exit 1` is what converts an exhausted loop into a failure** (Q7). Without it, thirty
failed attempts fall through and k6 runs against nothing, producing the 100% failure rate that looks like
an application bug.

</details>

---

## Q41

```yaml
          - task: [BLANK 1]
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: '**/junit.xml'
              [BLANK 2]: true
            condition: [BLANK 3]
```

Requirement: one merged test run in the Tests tab, published even when tests fail.

- **BLANK 1:** `PublishPipelineArtifact@1` / `PublishCodeCoverageResults@2` /
  `PublishTestResults@2` / `PublishBuildArtifacts@1`
- **BLANK 2:** `failTaskOnFailedTests` / `publishRunAttachments` / `mergeTestResults` / `mergeResults`
- **BLANK 3:** `succeeded()` / `failed()` / `succeededOrFailed()` / `always()`

<details>
<summary>Show answer</summary>

### Answer: `PublishTestResults@2`, `mergeTestResults`, `always()`

**In `challenge-16.md`:** lines **390–396**.

**`mergeTestResults: true` combines the three matrix legs into one run** rather than three competing ones
(Q3).

**And `always()` over `succeededOrFailed()`** — the latter misses a **cancelled** run, which is precisely
when you most want to see how far the tests got (Challenge 43 Q38).

</details>

---

# Section G — Case study

## Case study: Contoso e-commerce testing pyramid

### Background

Contoso Ltd's e-commerce platform **deploys three times daily** with **no automated testing** in the
CI/CD pipeline. Over the past month **four deployments introduced regressions** that broke production
checkout. The CTO's mandate: **"Nothing reaches production without passing automated tests at every
layer."** The application is a **Node.js Express API with a PostgreSQL backend**.

### Requirements

**Coverage of layers**

- Fast isolated tests must run on every push and pull request
- API behaviour must be validated against a **real** database, not a mock
- Checkout performance must be verified against a baseline

**Reliability**

- Tests must not fail intermittently because of CI timing
- A compatibility problem on a newer Node version must be visible before production upgrades
- Test results must be visible even when the run fails

**Enforcement**

- A drop in coverage must fail the build
- A latency or error-rate regression must fail the build
- Results must reach reviewers where they are working

---

## Q42

How should the three layers be sequenced, and why?

- A. All three layers in parallel for the fastest feedback
- B. Unit → integration → load, chained with `needs:`
- C. Load → integration → unit, most expensive first
- D. One job running every layer on a single runner

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-16.md`:** lines **134**, **236**, **312**.

```yaml
  integration-tests:
    needs: unit-tests
...
  load-tests:
    needs: integration-tests
```

**Fail fast and cheaply.** A broken unit test means the integration job's database setup and the load
job's several minutes are certainly wasted.

**Why A is defensible on wall-clock and wrong on cost.** Running all three in parallel gives faster
feedback when everything passes and burns the full cost on every failure — and on three deployments a day
that is the common case during a bad week.

**Why D loses the isolation.** One job means one runner, one Node version, and no ability to give the
integration layer a service container the unit layer does not need.

</details>

---

## Q43

How is "a real database, not a mock" satisfied without shared-state flakiness?

- A. A per-job service container, migrated and seeded
- B. A shared staging database all runs connect to
- C. An in-memory SQLite substitute for faster runs
- D. Mocks of the data repository layer

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-16.md`:** lines **137–150** and **165–173**.

**A service container is created and destroyed with the job**, so every run starts from an identical,
private state — which is what makes the tests deterministic.

**Why B is the answer teams reach for and why it causes the worst kind of failure** (Q25). Two concurrent
runs share rows, so failures become intermittent and the team learns to re-run rather than read.

**Why C fails the requirement literally.** SQLite is a different engine with different SQL, so a query
that works there can still fail against PostgreSQL — which is exactly the class this layer exists to
catch.

</details>

---

## Q44

Which **two** stop intermittent CI timing failures? (Choose two.)

- A. A `pg_isready` retry loop before migrations
- B. A longer fixed `sleep` before each step
- C. Automatic job re-runs on any failure
- D. A `/health` poll before k6, ending in `curl -f`
- E. Removing the load test from the pipeline

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-16.md`:** lines **515–521** and **541–551**.

**Both replace a guess with a **check**, and both fail loudly when the condition never becomes true.**

**Why C is the most damaging of the wrong answers.** Automatic re-runs make intermittent failures
invisible, so a genuine race and a genuine bug become indistinguishable — and the pipeline's signal is
destroyed (Challenge 34).

</details>

---

## Q45

How is a Node 22 compatibility problem made visible before production upgrades?

- A. Testing on Node 20 only, the production version
- B. Upgrading production first and watching for errors
- C. A matrix across 18, 20 and 22 with `fail-fast: false`
- D. A separate nightly job running on Node 22 alone

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-16.md`:** lines **55–58** and **527–534**.

**The matrix finds it; `fail-fast: false` makes the result readable.** Without the flag, a Node 18 failure
cancels the 22 leg and hides the very thing you were testing for.

**And the guarded polyfill is what lets one codebase satisfy all three** (Q6) — the fix is
`typeof globalThis.fetch === 'undefined'`, not pinning to a version.

**Why D delays the signal by up to a day** and detaches it from the pull request that caused it.

</details>

---

## Q46

Which **two** enforce the requirements rather than reporting on them? (Choose two.)

- A. `PublishTestResults@2` to the Tests tab
- B. Jest `coverageThreshold` at 80% on all four metrics
- C. The PR summary comment from `report-results`
- D. Uploading artifacts with `if: always()`
- E. k6 `thresholds` on latency, error rate and checks

<details>
<summary>Show answer</summary>

### Answer: B, E

**In `challenge-16.md`:** lines **109–116** and **204–208**.

**Both exit non-zero, which fails the step** (Q20). The other three make results **visible**, which is the
"results must reach reviewers" requirement and a different one.

**And the third requirement — "results visible even when the run fails" — is D's job**, which is why
`if: always()` is not optional on reporting steps.

**Three requirements, two mechanisms, no overlap.** The exam expects you to keep them apart.

</details>

---

## Q47

Six months in, the platform team notices that the load-test job has been passing on every run for eleven
weeks, including through a release that measurably slowed checkout in production. The job runs, k6
executes, and the JSON artifact is uploaded each time. A workflow refactor around that date consolidated
several `options` blocks.

What is the most likely cause?

- A. k6 stopped being installed on the runner
- B. The application server was not starting
- C. The test database was left empty after seeding
- D. The `thresholds` block was dropped in the refactor

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-16.md`:** lines **204–208** and **604**.

```javascript
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.01'],
    checks: ['rate>0.99'],
  },
```

**"Passing on every run" over eleven weeks is the tell.** A load test that has *never* failed across a
release that demonstrably regressed performance is not measuring anything.

**Without thresholds, k6 reports metrics and exits zero regardless** (Q4). The step goes green, the
artifact uploads, and nothing in the run indicates that P95 doubled.

**Why A, B and C would all fail loudly.** Missing k6 fails the install step; a dead server produces the
100% failure rate from the break scenario; an empty database fails the checks — **all three are visible,
and none matches "passes every time".**

**The durable lesson: for any automated test, ask when it last failed.** A check that has never failed is
either protecting nothing or measuring nothing, and the two are indistinguishable from the outside.

</details>

---

## Q48

A year on, checkout regressions are caught before merge, the pipeline has not produced an intermittent
failure in months, and a reviewer sees the full test picture in the pull request.

Which explanation best accounts for the change?

- A. Each layer gates what the one below cannot catch
- B. The team started writing many more tests
- C. Deployments were reduced to once per week
- D. A manual QA stage was added before each release

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-16.md`:** lines **29–31**, **52–91**, **133–188**, **204–208**, **515–551**, **310–344**.

**Take the mandate at line 25 — *automated tests at every layer* — and ask what each layer buys.**

*Unit tests* are the only layer cheap enough to run on every push to every branch, three times a day. They
catch the ordinary mistake before anyone waits on infrastructure.

*Integration tests* are the only layer that can catch what actually broke checkout four times in a month
(line 23) — a query, a migration, a contract between the API and the database. **A mock would have passed
every one of those.**

*Load tests with thresholds* catch what functional tests structurally cannot: behaviour that is correct at
one request and wrong at fifty.

**And the readiness loops are not a detail.** A pipeline that fails intermittently gets re-run out of
habit, and once re-running is a habit, **a real failure is indistinguishable from a flaky one** — which
would have undone everything above.

**What actually changed is not diligence.** The team was not refusing to test; there was **no layer at
which a regression could be caught automatically**, so every one of them reached production.

**The graded idea: the pyramid is a statement about *where* a class of bug is cheapest to find.** Put each
check at the lowest layer that can see the failure, make that layer fail the build, and publish what it
found — and the four regressions become four failed pull requests.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **`continue-on-error` instead of `fail-fast: false`** | Q36 | It marks a failing leg green |
| **Health check assumed to guarantee readiness** | Q2, Q5, Q28 | Add an explicit `pg_isready` loop |
| **`sleep` used as a readiness mechanism** | Q7, Q21, Q30, Q44 | Too short under load; a guess either way |
| **A retry loop with no failing check after it** | Q40 | An exhausted loop falls through silently |
| **`curl` without `-f`** | Q40 | A 500 response still exits zero |
| **Load test without `thresholds`** | Q4, Q26, Q34, Q47 | Always exits zero. Cost with no signal |
| **Artifacts or comments treated as gates** | Q20, Q22, Q29, Q46 | Visibility is not enforcement |
| **Publish step skipped on failure** | Q12, Q29, Q41 | `if: always()` / `condition: always()` |
| **`succeededOrFailed()` for reporting** | Q41 | Misses cancellation |
| **Shared staging database for integration tests** | Q25, Q43 | Concurrent runs make failures intermittent |
| **Single Node version** | Q6, Q25, Q45 | The polyfill collision is invisible until upgrade |
| **Coverage measured on lines only** | Q9, Q38 | Branches is the demanding metric |
| **`avg` instead of `p(95)`** | Q39 | An average hides the tail users feel |
| **`--forceExit` treated as a fix** | Q13 | It hides an unclosed resource |

---

# What to memorise

**In `challenge-16.md`:** lines **29–31**, **52–92**, **133–188**, **194–229**, **347–471**.

```text
THE PYRAMID - cost and speed, not importance
  UNIT         fast, isolated, no dependencies         many    -> logic, and (via a MATRIX) runtimes
  INTEGRATION  real database, migrations, seeds        fewer   -> queries, migrations, contracts
  LOAD         running app + concurrency               fewest  -> latency and error rate under load
  chain them with needs: so an expensive layer never runs after a cheap failure

REPORT vs GATE - the split the exam tests everywhere
  GATE     Jest coverageThreshold (non-zero exit)  |  k6 thresholds (non-zero exit)
  REPORT   PublishTestResults@2 | upload-artifact | PR comment
  ...and a failing job only blocks a merge when it is a REQUIRED STATUS CHECK
```

```yaml
# Unit job - matrix                                 (lines 52-91)
strategy:
  matrix: {node-version: [18, 20, 22]}
  fail-fast: false            # default CANCELS the other legs. you need the full picture
                              # NOT continue-on-error - that marks a failing leg GREEN
- uses: actions/setup-node@v4
  with: {node-version: "${{ matrix.node-version }}", cache: 'npm'}
- run: npx jest --coverage --ci --reporters=default --reporters=jest-junit
- uses: actions/upload-artifact@v4
  if: always()                # or the FAILING run's results are never uploaded
  with: {name: "test-results-node-${{ matrix.node-version }}"}    # per-leg name
- if: matrix.node-version == 20                                    # ONE canonical coverage

# jest.config.js
coverageThreshold: {global: {branches: 80, functions: 80, lines: 80, statements: 80}}
#   BRANCHES is the hard one - 100% lines can miss every else path
coverageReporters: ['text', 'lcov', 'cobertura']
#   text=humans   lcov=GitHub   cobertura=PublishCodeCoverageResults@2
```

```yaml
# Integration job - a REAL database, per run        (lines 133-188)
services:
  postgres:
    image: postgres:16
    env: {POSTGRES_USER: ..., POSTGRES_PASSWORD: ..., POSTGRES_DB: ...}
    ports: ["5432:5432"]
    options: >-
      --health-cmd="pg_isready -U contoso_test"     # readiness PROBE, not psql
      --health-interval=10s --health-timeout=5s --health-retries=5

# the health check is NOT enough - add an explicit wait   (lines 515-521)
for i in $(seq 1 30); do pg_isready -h localhost -p 5432 -U contoso_test && break; sleep 2; done

# then, in order:  migrate  ->  seed  ->  test
npx knex migrate:latest ; npx knex seed:run ; npx jest --config jest.integration.config.js --ci --forceExit
#   --forceExit: an open pool keeps node alive. it HIDES the leak - close it in afterAll instead
```

```javascript
// k6 - the load profile and the ACCEPTANCE CRITERIA   (lines 198-228)
export const options = {
  stages: [ {duration:'30s', target:20}, {duration:'1m', target:50}, {duration:'30s', target:0} ],
  //  ramp up -> hold -> RAMP DOWN (target 0, or in-flight requests count as failures)
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],   // percentiles, never avg
    http_req_failed:   ['rate<0.01'],                 // a PROPORTION
    checks:            ['rate>0.99'],                 // responses that arrived and were WRONG
  },
};
// NO thresholds -> k6 ALWAYS exits 0 -> the job is cost with no signal
```

```yaml
# Start the app before load testing                 (lines 273-277, 541-551)
npm start &                                   # & or the step blocks forever
for i in $(seq 1 30); do curl -s .../health >/dev/null 2>&1 && break; sleep 2; done
curl -f http://localhost:3000/health || exit 1
#   -f  : an HTTP error becomes a non-zero exit
#   || exit 1 : an EXHAUSTED loop must fail, or k6 runs against nothing
#   curl --retry-connrefused : plain --retry does NOT retry a refused connection

# AZURE PIPELINES equivalents                       (lines 361-471)
strategy: {matrix: {Node18: {nodeVersion: '18.x'}, Node20: {...}}}   # NAMED entries -> $(nodeVersion)
- task: PublishTestResults@2
  inputs: {testResultsFormat: 'JUnit', testResultsFiles: '**/junit.xml', mergeTestResults: true}
  condition: always()
- task: PublishCodeCoverageResults@2
  inputs: {summaryFileLocation: '.../cobertura-coverage.xml'}
  condition: eq(variables['nodeVersion'], '20.x')
- task: PublishPipelineArtifact@1     # for files nothing parses, e.g. k6 JSON
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 17 |
| 38–43 | Re-read the trap index and the report-vs-gate table, then move on |
| 30–37 | Write the three layers and the two gating mechanisms from memory, then retake |
| Below 30 | Redo Tasks 1, 3 and 4 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 16.

:::danger The three rules

**Put each check at the lowest layer that can see the failure.** Unit for logic, integration for queries
and contracts, load for behaviour under concurrency.

**A test that cannot fail the build is a report.** Coverage thresholds and k6 thresholds are the only two
gates here — and a load test with no `thresholds` block always passes.

**Never wait with `sleep`.** Poll for readiness, and make the loop fail when it runs out — otherwise the
pipeline becomes intermittent, and an intermittent pipeline trains the team to re-run instead of read.

:::
