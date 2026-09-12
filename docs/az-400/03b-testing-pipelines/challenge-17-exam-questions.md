---
sidebar_position: 2.5
toc_max_heading_level: 2
title: "Challenge 17: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 17 — AZ-400 exam questions

**48 questions** built only from what Challenge 17 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-17.md`**.

:::danger Read this before you start

**An approval asks a person, once. A gate asks a system, repeatedly.**

That single sentence separates most of the answers in this paper. A **required reviewer** is a human
decision recorded once — someone clicks approve and the job proceeds. An **Azure Pipelines gate** is a
machine question re-asked on a timer: it evaluates, and if the answer is no it **waits and asks again**
until it either passes or the timeout expires. Neither one fails fast, and that is why a
**misconfigured gate looks exactly like a slow one**.

**Second: a scan that reports is not a gate. A job that fails is.**

Trivy without `exit-code: '1'` produces a beautiful SARIF file and a green tick. `npm audit` piped to
`|| true` produces nothing at all. In this challenge every real gate is one of two things: **a non-zero
exit code**, or **a protection rule attached to an environment**.

**Third, and it is where the break comes from: protection rules do not live in YAML.**

The workflow only *names* the environment — `environment: name: production`. Everything that actually
blocks (reviewers, wait timers, branch policy) is configured **on the environment itself**, through the
API or the settings UI. So a workflow file can be perfect and the deployment can still hang forever, and
**nothing you can read in the repository explains why**.

The scenario at line 24: a production incident where a deployment carrying **known high-severity
vulnerabilities** shipped without review. Five mandated conditions follow at lines 26–30 — coverage,
vulnerabilities, approvals, latency, and Defender findings. Learn those five; half the paper points at
them.

:::

---

# Section A — Multiple choice

---

## Q1

The `deploy-production` job must wait for two named reviewers. Which part of the configuration makes that
happen?

- A. The `needs:` list naming the three upstream jobs
- B. The `if:` condition restricting the job to pushes on `main`
- C. The `uses: ./.github/actions/quality-gate` step
- D. `environment: name: production`, with reviewers set on it

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-17.md`:** lines **293–295**, and the environment configuration at lines **44–48**.

```yaml
    environment:
      name: production
      url: https://api.contoso.com
```

**The YAML only names the environment.** The reviewers, the wait timer and the branch policy are set by
the `gh api ... /environments/production --method PUT` call at line 44 — they are stored on the
**environment object**, not in the repository.

**This is the single most important structural fact in the challenge.** It is why Issue 3 at line 549 —
a reviewer team that no longer exists — is invisible to anyone reading `quality-gates.yml`.

**Why A is a different constraint.** `needs:` sequences jobs. It says "these must succeed first", and it
says nothing about humans. **Both must be satisfied** (line 606), which is the point of Q18.

**Why B narrows *when* the job is considered at all** — pushes to `main` only — but a job that is
considered still has to clear the environment.

**Why C runs *inside* the job**, so it can only be reached after every gate above has already let the job
start.

</details>

---

## Q2

In the environment creation call, what does `wait_timer=5` do?

- A. The job is held 5 minutes once other rules pass
- B. The job fails if nobody approves within 5 minutes
- C. The gate re-evaluates every 5 minutes
- D. Reviewers receive a reminder every 5 minutes

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-17.md`:** line **46**.

```bash
gh api repos/contoso-ltd/ecommerce-api/environments/production \
  --method PUT \
  --field wait_timer=5
```

**A wait timer is a deliberate delay, measured in minutes**, between the moment a deployment becomes
eligible and the moment it actually runs. It exists so somebody has a window to cancel — a
"are we sure" pause, not a deadline.

**Why C is the Azure Pipelines concept the exam pairs it with.** `period` inside `evaluationOptions`
(line 471) is the re-evaluation interval for a **gate**. A GitHub wait timer waits **once**; an Azure
gate asks **repeatedly**. Confusing the two is the trap in Q28.

**Why B inverts it.** Nothing times out the reviewers here. The environment protection timeout in GitHub
is a separate, much longer window; the wait timer never fails a job.

**And the fix at line 582 sets `wait_timer=0`** — because in the break scenario an excessive timer was
part of what made the job look permanently stuck (line 551).

</details>

---

## Q3

Which `--coverageReporters` value makes the `jq '.total.lines.pct'` extraction possible?

- A. `lcov` for tooling
- B. `text` for the log
- C. `json-summary` for `jq`
- D. `cobertura` for Azure

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-17.md`:** lines **82** and **86–89**.

```bash
npx jest --coverage --ci --coverageReporters=json-summary --coverageReporters=lcov
COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
echo "percentage=$COVERAGE" >> "$GITHUB_OUTPUT"
```

**`json-summary` is the reporter that writes `coverage/coverage-summary.json`** — the small machine-readable
totals file the gate reads. Two reporters are requested precisely because they serve two consumers.

**Why A is the other one, and it is not parseable this way.** `lcov` produces a per-file trace for
tooling and the uploaded artifact (lines 91–95); `jq` cannot read it.

**Why B is for humans** — a table printed in the log, gone when the run is deleted.

**Why D is the Azure Pipelines payload.** `PublishCodeCoverageResults@2` at line 399 wants
`cobertura-coverage.xml`. Right format, wrong platform for this step.

**The habit to build: match the reporter to the reader.** Ask who consumes the file — a script, a
publishing task, or a person — and the format follows.

</details>

---

## Q4

The Trivy step scans the filesystem for critical and high findings. Which single setting decides whether
it **blocks** the pipeline?

- A. `severity: 'CRITICAL,HIGH'`
- B. `format: 'sarif'`
- C. `exit-code: '1'`
- D. `scan-type: 'fs'`

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-17.md`:** lines **118–126**.

```yaml
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@0.28.0
        with:
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
```

**`exit-code: '1'` tells Trivy to exit non-zero when it finds something at the requested severity.**
Remove it and Trivy still finds everything, still writes the SARIF, still uploads it — **and the job goes
green**.

**Why A selects *what counts as a finding*, not what happens next.** Narrow the severity and you narrow
the findings; you do not change whether findings matter.

**Why B decides the output format** so `upload-sarif` can post it to the Security tab.

**Why D chooses the target** — the filesystem rather than a container image or a repository.

**This is the report-versus-gate split in one line of YAML**, and it recurs across the whole certification:
a scanner's exit code is the only part of it a pipeline can act on.

</details>

---

## Q5

Why does the dependency audit step end its `npm audit` command with `|| true`?

- A. To ignore any vulnerabilities the audit reports
- B. Because `npm audit` cannot write JSON otherwise
- C. To retry the audit automatically on failure
- D. So the script, not npm, decides whether the step fails

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-17.md`:** lines **107–116**.

```bash
npm audit --audit-level=high --json > audit-results.json || true
CRITICAL=$(cat audit-results.json | jq '.metadata.vulnerabilities.critical // 0')
HIGH=$(cat audit-results.json | jq '.metadata.vulnerabilities.high // 0')
echo "Critical: $CRITICAL, High: $HIGH"
if [ "$CRITICAL" -gt 0 ] || [ "$HIGH" -gt 0 ]; then
  echo "::error::Found $CRITICAL critical and $HIGH high vulnerabilities"
  exit 1
fi
```

**`npm audit` exits non-zero the moment it finds anything at or above the audit level.** Under `set -e`
semantics that would kill the step immediately — before the counts are extracted and before the
`::error::` annotation is written. The engineer would see a failure with no numbers in it.

**So `|| true` moves the decision**: npm reports, the script decides. The gate is the explicit `exit 1`
on line 115, not npm's exit code.

**Why A is the misreading the exam wants.** `|| true` looks like suppression, and in the wrong place it
is. Here it is followed **four lines later** by a check that fails harder and says why.

**Read it as a rule: `|| true` is safe only when something after it can still fail.** A `|| true` with no
subsequent check is how a security scan quietly stops mattering.

**Why B is false** — `--json` writes to stdout regardless of exit code — **and C describes `--retry`**,
which is not present.

</details>

---

## Q6

Which permission must the job declare so `github/codeql-action/upload-sarif` succeeds?

- A. `security-events: write`
- B. `contents: write`
- C. `actions: write`
- D. `packages: write`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-17.md`:** lines **99–101** and the knowledge check at line **628**.

```yaml
    permissions:
      security-events: write
      contents: read
```

**`security-events: write` is the scope that lets a workflow post code-scanning alerts.** It is what
makes results from Trivy, CodeQL or Microsoft Security DevOps appear in the repository's **Security tab**
rather than only in the job log.

**Note what sits beside it: `contents: read`.** The block is written out in full because declaring
`permissions:` at all **replaces** the default token scopes for that job. Naming only
`security-events: write` would leave the job unable to check out the repository.

**Why B, C and D are the plausible neighbours.** `contents: write` pushes commits and creates releases;
`actions: write` manipulates workflow runs and caches; `packages: write` publishes to GitHub Packages.
None of them touch code scanning.

**The same block is repeated on the Defender job** at lines 485–487, which is the tell that it belongs to
the *upload*, not to the scanner.

</details>

---

## Q7

Why does the SARIF upload step carry `if: always()`?

- A. To make the upload run before the scan step
- B. To upload even when the scan failed the job
- C. Because SARIF uploads are slow to complete
- D. To retry the upload until it succeeds

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-17.md`:** lines **128–132**, repeated at lines **499–503**.

```yaml
      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif
```

**Default step behaviour is `if: success()`.** Trivy has just been configured to exit 1 on a critical
finding (Q4), so without `always()` the upload is skipped **precisely on the runs that found something**.

**The result is perverse and very hard to notice**: clean scans populate the Security tab, dirty scans
leave it empty. The tab looks reassuring because it only ever receives good news.

**Why A misreads `always()` as ordering.** Step order is textual; `if:` only decides whether a step runs.

**Why C and D are inventions** — `always()` says nothing about duration and performs no retries.

**This is the same reflex as `condition: always()` on `PublishTestResults@2`** at line 397: anything whose
job is to *record what happened* must run even when what happened was a failure.

</details>

---

## Q8

In `.github/actions/quality-gate/action.yml`, which key is mandatory on every `run` step and is not
required in a workflow file?

- A. `id:`
- B. `working-directory:`
- C. `continue-on-error:`
- D. `shell:`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-17.md`:** lines **232–238**.

```yaml
runs:
  using: 'composite'
  steps:
    - name: Evaluate quality gates
      id: evaluate
      shell: bash
      run: |
```

**Composite actions have no default shell.** A workflow job inherits one from the runner OS; a composite
action is meant to be reusable across runners, so it refuses to guess. Omit `shell:` and the action fails
to load with a validation error — before a single line of the script runs.

**Why A is present here but optional.** `id: evaluate` exists because the `outputs:` block at lines
224–230 refers to `steps.evaluate.outputs.*`. Drop the id and the outputs break; drop `shell:` and the
whole action breaks.

**Why B and C are ordinary optional keys** on any step, composite or not.

**Memorise the trio that identifies a composite action**: `runs.using: 'composite'`, `runs.steps`, and
`shell:` on every `run`.

</details>

---

## Q9

What does this block in the composite action accomplish?

```yaml
outputs:
  gate-passed:
    description: 'Whether all quality gates passed'
    value: ${{ steps.evaluate.outputs.passed }}
```

- A. It exposes a step output as the action's output
- B. It sets an environment variable for later jobs
- C. It fails the action if the value is false
- D. It writes the value to the job summary

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-17.md`:** lines **224–230**.

**An action's outputs are declared, not inherited.** The step writes `passed=...` into `$GITHUB_OUTPUT`
(line 273); `value:` then lifts that step output to the action's public surface so the caller can use
`steps.<id>.outputs.gate-passed`.

**Why C is the tempting answer, and it is a different mechanism entirely.** Nothing in the `outputs:`
block can fail anything. The failure comes from `exit 1` at line 280 — see Q10.

**Why B confuses outputs with `GITHUB_ENV`.** Outputs are per-step values passed by reference; env vars
are process variables. Outputs cross job boundaries when a job declares them (line 65); env vars do not.

**Why D describes `$GITHUB_STEP_SUMMARY`**, which this challenge does not use.

**Notice the challenge never actually consumes these outputs.** The deploy job calls the action and
relies on it failing. That is worth seeing: **the outputs are documentation; the exit code is the gate.**

</details>

---

## Q10

Coverage comes back at 62% against a threshold of 80. What stops the deployment?

- A. The `outputs.gate-passed` value being `false`
- B. `exit 1` at the end of the action's script
- C. The environment's required reviewers holding it
- D. The `needs:` list of upstream jobs failing

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-17.md`:** lines **278–281**.

```bash
if [ "$PASSED" = "false" ]; then
  echo "::error::Quality gates failed. Deployment blocked."
  exit 1
fi
```

**A non-zero exit fails the step, which fails the job, which stops every later step in it.** The deploy
step at line 315 never runs.

**Why A is a value nobody is reading.** `gate-passed` would be `false`, and the calling workflow ignores
it (Q9). If the script ended after writing outputs, **the deployment would proceed with 62% coverage**.

**Why C and D are already satisfied by this point.** Reviewers approved and dependencies succeeded — that
is how the job started. The composite action is the **last** gate, running inside the job.

**Draw the sequence once and it stays learnt**: `needs:` → `if:` → environment rules → steps in order →
the in-job gate. Every one of them can stop the deployment, and they stop it at different moments and
with different visibility.

</details>

---

## Q11

The deploy job passes `security-scan-passed: ${{ needs['security-scan'].result == 'success' }}`. Given the
job's `needs:` list and no `if: always()`, what will this expression always evaluate to when the step
runs?

- A. `false` when any upstream test fails
- B. It depends on the environment approval
- C. `null` until the security scan completes
- D. `true` — a failed dependency skips the job

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-17.md`:** lines **291** and **312**.

```yaml
    needs: [test-and-coverage, security-scan, performance-baseline]
```

**`needs:` without `if: always()` means the job is *skipped* when any dependency fails.** So by the time
line 312 is evaluated, `security-scan` has already succeeded. The input is defensive — belt and braces —
not a live decision.

**This matters because it is the exam's favourite reasoning step.** Given a `needs:` list, ask first
*whether the job runs at all*; only then reason about expressions inside it.

**Where the expression would earn its keep** is a job written `if: always()`, which runs regardless and
must then inspect `needs.*.result` for `'success'`, `'failure'`, `'cancelled'` or `'skipped'` itself.

**Why A inverts the logic** — a failing test skips the deploy job rather than passing `false` into it —
**and why C invents a state.** Contexts are resolved when the step is evaluated, not left pending.

</details>

---

## Q12

In Azure Pipelines, what happens when a pre-deployment gate evaluation returns a negative result?

- A. The gate polls at the interval until it passes or times out
- B. The deployment is cancelled immediately with an error
- C. The pipeline fails and must be restarted by hand
- D. The result is logged and the deployment proceeds

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-17.md`:** lines **468–475**, and the knowledge check at line **617**.

```json
  "timeout": 43200,
  "retryOn": "all",
  "evaluationOptions": {
    "period": 300000,
    "timeout": 86400000,
    "initialDelay": 0
  }
```

**Gates poll.** `initialDelay` is how long to wait before the first evaluation, `period` is the gap
between attempts — 300000 ms, five minutes — and `timeout` is when to give up. Only at the timeout does
the deployment fail.

**This is the defining difference from a required status check**, which is evaluated once per commit and
is either green or not.

**Why polling exists at all**: gates ask questions whose answers change on their own. "Are there active
Azure Monitor alerts?" "Is the health endpoint returning healthy?" A transient alert should delay a
release, not kill it — so the system waits for the world to settle.

**And why it is dangerous**: a gate that can **never** pass because its criteria are wrong looks
identical to a gate patiently waiting for a slow system. That is Issue 2 at line 545 — and it burns 24
hours before anyone gets an error.

</details>

---

## Q13

The Azure Pipelines health gate uses `successCriteria: "eq(root['status'], 'healthy')"`, but the API
returns `{"status": "ok"}`. What is the observable behaviour?

- A. The gate fails immediately with a clear error
- B. The gate is skipped because the criteria do not match
- C. The gate polls every five minutes and never passes
- D. The deployment proceeds with a warning logged

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-17.md`:** lines **465** and **545–547**.

**The endpoint is healthy. The request succeeds. The criteria simply never match**, so the gate returns a
negative result, waits `period`, and asks again — indefinitely, until the 24-hour timeout at line 472.

**Nothing about this looks like a bug.** There is no error, no red mark, no failing test. The deployment
is "in progress", which is the same thing the UI shows during a legitimate wait.

**The fix is one character class wide** (lines 569–573):

```json
{
  "successCriteria": "eq(root['status'], 'ok')"
}
```

**And line 575 states the discipline directly:** always verify gate criteria against the **actual**
endpoint response before enabling the gate. Call the endpoint, read the JSON, then write the expression.

**Why A is what everyone expects and none of them get**, and why B and D describe systems that fail open.
A gate that fails open would not be a gate.

</details>

---

## Q14

The `deploy-production` job never starts. Everything else is green and two reviewers have approved. What
is Issue 1?

- A. The `production` environment does not exist yet
- B. The `needs:` list contains a circular reference
- C. The runner pool has no available capacity for it
- D. The job is a required check but only runs post-merge

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-17.md`:** lines **541–543**, and the knowledge check at line **639**.

**A deadlock built out of two correct settings.** The `if:` at line 292 restricts the job to
`github.event_name == 'push'` on `main`. Branch protection then demands that this same job report success
before the pull request may merge. Neither side is wrong on its own; together, nothing moves.

**The fix is to require only checks that run on pull requests** (lines 558–564):

```bash
gh api repos/contoso-ltd/ecommerce-api/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["test-and-coverage","security-scan","performance-baseline"]}' \
  --field enforce_admins=true \
  --field required_pull_request_reviews='{"required_approving_review_count":2}'
```

**Why B is the right instinct pointed at the wrong object.** The `needs:` graph is a clean chain; the
cycle is between **branch protection** and **trigger conditions**, which live in different systems and
are never displayed together.

**The rule to carry into the exam: a required status check must be a check that runs on the pull
request.** Anything gated on `push` to the protected branch can never satisfy it.

</details>

---

## Q15

Which Azure Pipelines task runs Microsoft Defender for DevOps scanning?

- A. `PublishCodeCoverageResults@2`
- B. `MicrosoftSecurityDevOps@1`
- C. `SecurityScan@2`
- D. `AzureSecurityCenter@1`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-17.md`:** lines **409–412**.

```yaml
          - task: MicrosoftSecurityDevOps@1
            displayName: 'Run Microsoft Security DevOps'
            inputs:
              categories: 'secrets,dependencies'
```

**Its GitHub Actions counterpart is `microsoft/security-devops-action@v1`** at line 494 — same product,
same `categories` input, different platform.

**Note the categories differ between the two examples in this challenge**: `secrets,dependencies` in
Azure Pipelines (line 412), `IaC,secrets,code` in the GitHub workflow (line 497). The scanner is a
harness that runs several analysers; **categories choose which ones**.

**Why C and D do not exist as task names.** Both sound right, which is the point — the exam manufactures
plausible task identifiers, and the defence is having read the real one.

**Why A is a real task doing something unrelated** — publishing a cobertura report, line 399.

</details>

---

## Q16

Contoso's third requirement is that **at least two team members approve** the change. Where is that
enforced?

- A. In both places — different approvals, different moments
- B. Only in branch protection's review count
- C. Only on the environment, via its required reviewers
- D. In the composite quality-gate action's script

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-17.md`:** line **47** (environment reviewers), line **564**
(`required_pull_request_reviews`), and lines **437** (two reviewers from `release-approvers`).

**They approve different things.**

| Mechanism | Approves | When | Blocks |
|---|---|---|---|
| Branch protection review count | The **code change** | Before merge | The merge |
| Environment required reviewers | The **deployment** | After merge, before the job runs | The deploy job |

**A pull request review says "this change is sound".** An environment approval says "release this, now,
to production" — often by a different group, with a release window and an incident context in mind.

**Which is why both exist here.** Requirement 3 is satisfied at merge time by the review count, and the
environment gate then holds the deployment for a release approver.

**Why D is impossible.** A composite action runs inside the job; the job has already started, so no
approval it asked for could gate anything.

**The exam's version of this question usually hides in the word "deployment".** Approvals on *changes*
are branch protection; approvals on *deployments* are environments.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** jobs must succeed before `deploy-production` is eligible to run? (Choose three.)

- A. `defender-scan`
- B. `test-and-coverage`
- C. `report-results`
- D. `security-scan`
- E. `BuildAndTest`
- F. `performance-baseline`

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-17.md`:** line **291**.

```yaml
    needs: [test-and-coverage, security-scan, performance-baseline]
```

**Read the `needs:` list, not the requirement list.** Contoso mandates five conditions (lines 26–30), but
only three jobs are wired as dependencies.

**Why A is the interesting omission.** `defender-scan` is added in Task 6 (line 483) and satisfies
requirement 5 — yet it is **not** in `needs:`. As written, a critical Defender finding fails its own job
and the deployment proceeds anyway. That is a genuine defect in the challenge's workflow and a
first-class exam scenario: **a gate nobody depends on is not a gate.**

**Why E is the Azure Pipelines job** (line 381), a different file entirely.

</details>

---

## Q18

`deploy-production` starts only when several independent conditions all hold. Which **three** are they?
(Choose three.)

- A. Every job in `needs:` completed successfully first
- B. The composite action returned `gate-passed: true`
- C. The `if:` expression is true — a push on `refs/heads/main`
- D. Coverage was published to the Azure DevOps Tests tab
- E. The `production` environment's rules are satisfied
- F. The SARIF upload step succeeded

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-17.md`:** lines **291–295**, and the knowledge check at line **606**.

```yaml
    needs: [test-and-coverage, security-scan, performance-baseline]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment:
      name: production
```

**Three separate systems, evaluated in that order**, and all three must agree before a single step runs.
`needs:` is the dependency graph, `if:` is the trigger filter, and the environment adds approvals, wait
timers and branch policy from outside the file.

**Why B is after the fact.** The composite action runs at line 307 — **inside** the job. It can stop a
deployment, but it cannot stop a job from starting.

**Why D belongs to the other platform**, and **why F is a reporting step** whose success or failure
concerns the Security tab, not the deployment.

**The exam phrases this as "what must be satisfied".** The answer is never one mechanism. It is the
dependency graph *and* the condition *and* the environment.

</details>

---

## Q19

Which **two** mechanisms inside `security-scan` can actually fail the job? (Choose two.)

- A. `format: 'sarif'` on the Trivy step
- B. The explicit `exit 1` after counting vulnerabilities
- C. The `upload-sarif` step with `if: always()`
- D. Trivy's `exit-code: '1'` input
- E. `permissions: security-events: write`

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-17.md`:** lines **113–116** and **124**.

**Both produce a non-zero exit, and nothing else in the job does.** Everything else selects, formats or
publishes.

**Why C looks like a candidate and is not.** It carries `if: always()` (line 130), so it runs after a
failure — but uploading findings does not create one. If Trivy exits 0 with fifty medium findings, they
land in the Security tab and the job is green.

**Why A and E are prerequisites for C**, not decisions.

**The pattern to internalise: two independent scanners, two independent exit codes.** `npm audit` covers
declared dependencies from the lockfile; Trivy scans the filesystem. They overlap and neither subsumes
the other, which is why the challenge runs both in one job.

</details>

---

## Q20

Which **three** thresholds does `load-tests/gate-check.js` declare? (Choose three.)

- A. `http_req_duration: ['p(95)<200']`
- B. `vus: 20` virtual users
- C. `http_req_failed: ['rate<0.01']`
- D. `duration: '30s'` run length
- E. `checks: ['rate>0.99']`
- F. `iterations: 1000`

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-17.md`:** lines **335–343**.

```javascript
export const options = {
  vus: 20,
  duration: '30s',
  thresholds: {
    http_req_duration: ['p(95)<200'],
    http_req_failed: ['rate<0.01'],
    checks: ['rate>0.99'],
  },
};
```

**Thresholds are acceptance criteria; `vus` and `duration` are the load profile.** B and D describe *what
traffic is generated* — twenty virtual users for thirty seconds — and neither can fail anything.

**Only the `thresholds` block controls k6's exit code**, and therefore whether the
`performance-baseline` job passes.

**And each of the three answers a different question.** `http_req_duration` at p(95) is the latency
Contoso mandated at line 29; `http_req_failed` is the proportion of requests that errored; `checks`
catches responses that **arrived successfully and were wrong** — the assertions at lines 357–360.

**Why the percentile and not an average.** An average is dragged down by fast responses and hides the
tail that users actually complain about. `p(95)<200` says: **nineteen requests in twenty finish under
200 ms.**

</details>

---

## Q21

The deployment is blocked indefinitely. Which **three** root causes does the solution identify? (Choose
three.)

- A. Insufficient runner capacity for the deploy job
- B. A circular dependency between protection and the trigger
- C. A missing `security-events: write` permission
- D. The Azure health gate's criteria not matching the response
- E. Coverage falling below the 80% threshold
- F. A vanished reviewer team ID, plus a long wait timer

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-17.md`:** lines **541–551**.

**Three unrelated misconfigurations producing one symptom.** That is the lesson: "Waiting" is not a
diagnosis. It is the state every gate uses while it has not yet said yes, so it covers a deadlock, a
never-satisfiable criterion and a stale reference equally well.

**Why E contradicts the evidence.** Line 527 records coverage at 85%. Two reviewers approved (line 529),
performance passed (line 528), the security scan reported success (line 526) — the observations at lines
525–531 exist to eliminate exactly these guesses.

**Why C would produce a loud, specific error** — a failed upload step — not silence.

**And Issue 3 has a detail worth quoting**: GitHub **silently skips** the review requirement when the
referenced team is gone (line 551). A protection rule pointing at nothing does not fail closed and does
not warn. It simply stops protecting.

</details>

---

## Q22

After Fix 1, which **three** contexts remain as required status checks on `main`? (Choose three.)

- A. `test-and-coverage`
- B. `deploy-production`
- C. `security-scan`
- D. `quality-gate`
- E. `performance-baseline`
- F. `DeployProd`

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-17.md`:** line **562**.

```bash
  --field required_status_checks='{"strict":true,"contexts":["test-and-coverage","security-scan","performance-baseline"]}'
```

**All three run on `pull_request` events** (lines 56–60), so they can report a status against the PR's
head commit. That is the qualification — not importance, not severity.

**Why B is precisely what Fix 1 removes**, and re-adding it recreates the deadlock (Q14, Q26).

**Note `"strict":true` alongside them.** That is the "require branches to be up to date before merging"
setting: the PR must be rebased or merged onto the latest `main` before the checks count. It costs a
re-run on every base update and it is what stops two independently-green PRs from combining into a broken
`main`.

**Why D is the composite action's directory name**, not a check, and **why F is the Azure Pipelines
deployment job**.

</details>

---

## Q23

Besides approvals, which **two** check types does the challenge configure on the Azure DevOps `production`
environment? (Choose two.)

- A. A branch control check on the source
- B. A business hours deployment window check
- C. An Azure Monitor alerts gate
- D. An exclusive lock on the environment
- E. An Invoke REST API health check

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-17.md`:** lines **437–439**.

```bash
# Add approvals: Require 2 reviewers from the release-approvers group
# Add gate: Azure Monitor alerts (no active alerts on production)
# Add gate: REST API health check
```

**Both ask about the state of the *running system*, not the build.** "Is production currently alerting?"
and "is the API healthy?" are questions no CI job can answer, because their answers change minute by
minute — which is exactly why they are polled rather than evaluated once.

**Why A, B and D are real Azure DevOps checks not used here.** Branch control restricts which branches may
deploy — the equivalent of GitHub's `deployment_branch_policy` at line 48. Business hours confines
deployments to a window. Exclusive lock prevents concurrent runs against the same environment. Knowing
they exist is useful; the question asks what **this** challenge configures.

**And line 435 tells you where they live**: Pipelines → Environments → production → **Approvals and
checks**. Not in the YAML.

</details>

---

# Section C — Repeated scenario

**Scenario:** After a deployment carrying known high-severity vulnerabilities reached production without
review, Contoso mandates that no code reaches production unless tests pass with over 80% coverage, zero
critical or high dependency vulnerabilities exist, at least two team members approve, p95 latency stays
below 200 ms, and Defender for DevOps reports no critical findings.

---

## Q24

**Proposed solution:** Create a `production` environment with required reviewers from the release-approvers
team, a short wait timer and a protected-branches deployment policy. Add PR-triggered jobs that run tests
with coverage and fail below 80%, run `npm audit` and Trivy with `exit-code: '1'`, run Defender for DevOps
and fail on critical findings, and run k6 with a `p(95)<200` threshold against a started application. Make
the deploy job depend on all of those jobs, restrict it to pushes on `main`, and target the `production`
environment. Require the PR-time checks in branch protection with a two-review minimum.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-17.md`:** lines **44–48**, **107–126**, **293–295**, **335–343**, **505–515**, **558–564**.

| Requirement | Enforced by | Line |
|---|---|---|
| Coverage above 80% | Jest coverage, threshold check in the gate action | 82, 246 |
| Zero critical or high vulnerabilities | `npm audit` `exit 1` plus Trivy `exit-code: '1'` | 115, 124 |
| Two approvals | Branch protection review count, plus environment reviewers | 564, 47 |
| p95 under 200 ms | k6 `http_req_duration` threshold | 339 |
| No critical Defender findings | `defender-scan` job failing on error-level results | 505–514 |

**Every requirement maps to something that can say no**, and the two approval moments are distinguished
(Q16). The deploy job clears the dependency graph, the trigger filter and the environment before it runs
a step.

**The clause doing the quiet work is the last one.** Requiring only PR-time checks is what keeps the
whole design from deadlocking — and it is the sentence Q26 changes.

</details>

---

## Q25

**Proposed solution:** Run all scans inside the deploy job after the application is deployed, so the
pipeline is fast. Append `|| true` to the audit, Trivy and coverage commands so a noisy scanner cannot
block a release. Email the SARIF report to the release manager nightly. Configure the `production`
environment with a 12-hour wait timer instead of reviewers.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures, and each one is a whole category.**

**Scanning after deployment inverts the mandate.** The incident at line 24 was a vulnerable build
reaching production. A scan that runs post-deployment can only tell you that it happened again.

**`|| true` on every command removes every gate at once.** Compare it with the legitimate use at line 109,
where `|| true` is followed four lines later by an explicit `exit 1` (Q5). Here nothing follows. The jobs
go green regardless — **the most expensive possible way to have no security scanning.**

**A nightly email is not a gate.** Nobody is blocked by an unread inbox, and by the time it is read the
release has shipped. Compare requirement 5's actual implementation at lines 505–514, which counts
error-level results and exits non-zero.

**And a 12-hour wait timer is a delay, not an approval.** It asks nobody anything; it just makes every
release slow. Requirement 3 needs a **human decision** — reviewers on the environment (line 47) — not
elapsed time.

</details>

---

## Q26

**Proposed solution:** Create a `production` environment with required reviewers, a short wait timer and a
protected-branches policy. Add PR-triggered jobs for coverage, dependency scanning with Trivy, Defender
for DevOps and a k6 latency threshold, each failing on breach. Make the deploy job depend on all of them,
restrict it to pushes on `main`, and target the `production` environment. To guarantee nothing merges
until production has actually accepted the change, add `deploy-production` to the required status checks
on `main` alongside the others.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**`deploy-production` runs only on `push` to `main`** (line 292). Requiring it before merge asks the pull
request to wait for something that cannot happen until the pull request merges. **Nothing merges again.**

**And the failure mode is silence.** No red X, no error message — the check simply sits as "Expected", and
the merge button stays disabled. Teams lose days to this because every individual setting reads as
reasonable.

**The stated intention is not even achievable.** "Nothing merges until production accepted the change"
inverts the order of events: you deploy what is on `main`, so the change must merge first. The
requirement being reached for — **do not ship broken code** — is satisfied by the PR-time checks that are
already listed.

**Fix 1 at line 556 states the rule outright:** only gate checks that run on pull requests should be
required. Deployment verification belongs **after** the merge, as the environment gate it already is.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — environments and protection rules

| # | Statement | Answer |
|---|---|---|
| 1 | Required reviewers are configured on the environment, not in the workflow YAML |  |
| 2 | `environment: name: production` alone enforces approvals |  |
| 3 | `wait_timer` delays a deployment by a fixed number of minutes |  |
| 4 | `deployment_branch_policy` with `protected_branches` limits which branches may deploy |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Required reviewers are configured on the environment, not in the workflow YAML | **Yes** |
| 2 | `environment: name: production` alone enforces approvals | **No** |
| 3 | `wait_timer` delays a deployment by a fixed number of minutes | **Yes** |
| 4 | `deployment_branch_policy` with `protected_branches` limits which branches may deploy | **Yes** |

**In `challenge-17.md`:** lines **44–48**, **293–295**, **46**, **48**.

Row 2 is the whole break scenario in one line — the YAML names the environment; the environment carries
the rules.

Row 4 is GitHub's equivalent of an Azure DevOps branch control check (Q23).

</details>

---

## Q28 — approvals versus gates

| # | Statement | Answer |
|---|---|---|
| 1 | An Azure Pipelines gate is re-evaluated on a timer until it passes or times out |  |
| 2 | An approval is a one-time human decision |  |
| 3 | `period` in `evaluationOptions` is the re-evaluation interval |  |
| 4 | A failed gate evaluation immediately fails the deployment |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | An Azure Pipelines gate is re-evaluated on a timer until it passes or times out | **Yes** |
| 2 | An approval is a one-time human decision | **Yes** |
| 3 | `period` in `evaluationOptions` is the re-evaluation interval | **Yes** |
| 4 | A failed gate evaluation immediately fails the deployment | **No** |

**In `challenge-17.md`:** lines **617**, **437**, **471**, **545–547**.

Row 4 is the consequence that makes Issue 2 so expensive: the deployment stays "in progress" for the full
timeout at line 472 — 86400000 ms, twenty-four hours.

Rows 1 and 2 are the sentence to carry into the exam: **a gate asks a system, repeatedly; an approval asks
a person, once.**

</details>

---

## Q29 — security scanning and SARIF

| # | Statement | Answer |
|---|---|---|
| 1 | `security-events: write` is required to upload SARIF |  |
| 2 | Uploading SARIF blocks the deployment when findings exist |  |
| 3 | Trivy without `exit-code: '1'` reports findings and exits zero |  |
| 4 | `if: always()` on the upload step preserves results from failed scans |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `security-events: write` is required to upload SARIF | **Yes** |
| 2 | Uploading SARIF blocks the deployment when findings exist | **No** |
| 3 | Trivy without `exit-code: '1'` reports findings and exits zero | **Yes** |
| 4 | `if: always()` on the upload step preserves results from failed scans | **Yes** |

**In `challenge-17.md`:** lines **99–101**, **128–132**, **124**, **130**.

Row 2 is the recurring split: SARIF populates the Security tab, which is **visibility**. Only an exit code
is **enforcement**.

Row 4 matters more than it looks — without it, the Security tab receives results only from runs that found
nothing.

</details>

---

## Q30 — the blocked deployment

| # | Statement | Answer |
|---|---|---|
| 1 | A post-merge job can safely be a required status check on the same branch |  |
| 2 | GitHub silently skips a review requirement when the referenced team no longer exists |  |
| 3 | The Azure gate's success criteria should be verified against the real endpoint response |  |
| 4 | "Waiting" in the UI is enough to identify which gate is blocking |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A post-merge job can safely be a required status check on the same branch | **No** |
| 2 | GitHub silently skips a review requirement when the referenced team no longer exists | **Yes** |
| 3 | The Azure gate's success criteria should be verified against the real endpoint response | **Yes** |
| 4 | "Waiting" in the UI is enough to identify which gate is blocking | **No** |

**In `challenge-17.md`:** lines **541–543**, **551**, **575**, **523–531**.

Row 2 is the failure-open behaviour: a protection rule pointing at a deleted team stops protecting without
saying so.

Row 4 is why the observations at lines 525–531 are listed one by one — you eliminate causes, because the
symptom names none of them.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each of Contoso's five mandated conditions to the mechanism that enforces it.

| Requirement | Enforced by |
|---|---|
| Tests pass with over 80% coverage |  |
| Zero critical or high vulnerabilities |  |
| At least two approvals |  |
| p95 latency below 200 ms |  |
| No critical Defender findings |  |

**Options:** A k6 `http_req_duration` threshold failing the load job · Branch protection review count, and required reviewers on the environment · `defender-scan` counting error-level SARIF results and exiting non-zero · Jest coverage plus the gate action's threshold comparison and `exit 1` · `npm audit` counts with an explicit `exit 1`, and Trivy's `exit-code: '1'`

<details>
<summary>Show answer</summary>

| Requirement | Enforced by |
|---|---|
| Tests pass with over 80% coverage | **Jest coverage plus the gate action's threshold comparison and `exit 1`** |
| Zero critical or high vulnerabilities | **`npm audit` counts with an explicit `exit 1`, and Trivy's `exit-code: '1'`** |
| At least two approvals | **Branch protection review count, and required reviewers on the environment** |
| p95 latency below 200 ms | **A k6 `http_req_duration` threshold failing the load job** |
| No critical Defender findings | **`defender-scan` counting error-level SARIF results and exiting non-zero** |

**In `challenge-17.md`:** lines **82** and **243–255**, **107–126**, **47** and **564**, **339**,
**505–514**.

**Five requirements, five different enforcement surfaces** — a script comparison, two scanner exit codes,
two approval systems, a load-test threshold, and a jq query over SARIF. **None of them is a report.**

</details>

---

## Q32

Match each setting to its effect.

| Setting | Effect |
|---|---|
| `wait_timer=5` |  |
| `deployment_branch_policy` protected branches |  |
| `exit-code: '1'` |  |
| `if: always()` |  |
| `shell: bash` |  |
| `period: 300000` |  |

**Options:** A fixed five-minute delay before the deployment may run · Only protected branches may deploy to this environment · Required on every `run` step in a composite action · The gate is re-evaluated every five minutes · The upload runs even after the scan failed the job · Trivy exits non-zero when it finds a matching severity

<details>
<summary>Show answer</summary>

| Setting | Effect |
|---|---|
| `wait_timer=5` | **A fixed five-minute delay before the deployment may run** |
| `deployment_branch_policy` protected branches | **Only protected branches may deploy to this environment** |
| `exit-code: '1'` | **Trivy exits non-zero when it finds a matching severity** |
| `if: always()` | **The upload runs even after the scan failed the job** |
| `shell: bash` | **Required on every `run` step in a composite action** |
| `period: 300000` | **The gate is re-evaluated every five minutes** |

**In `challenge-17.md`:** lines **46**, **48**, **124**, **130**, **237**, **471**.

**Two of these six can fail something; the rest change when, whether or how often.** Sorting them into
those two piles is most of Section A.

</details>

---

## Q33

A commit is pushed to `main`. Arrange what must happen before the `az webapp deploy` command executes.

**Items:** The environment's required reviewers approve · `test-and-coverage`, `security-scan` and
`performance-baseline` all succeed · The composite quality-gate action exits zero · The wait timer
elapses · The `if:` expression evaluates true

<details>
<summary>Show answer</summary>

### Answer

1. The `if:` expression evaluates true — line **292**
2. `test-and-coverage`, `security-scan` and `performance-baseline` all succeed — line **291**
3. The environment's required reviewers approve — lines **47**, **293–295**
4. The wait timer elapses — line **46**
5. The composite quality-gate action exits zero — lines **307–313**, **278–281**

**Steps 1 and 2 decide whether the job is considered at all**, steps 3 and 4 are the environment holding
it at the door, and step 5 happens **inside** the job, as its second-to-last step.

**The ordering is the exam's real question.** Everything before step 5 blocks the job from starting and
is configured outside the file; step 5 is the only gate visible in the workflow itself.

**And it explains the debugging asymmetry.** A failure at step 5 gives you a log line and an
`::error::` annotation. A block at step 3 gives you the word "Waiting".

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| The PR shows a required check as "Expected" and never merges |  |
| A healthy API, a gate that never passes, no error for 24 hours |  |
| No reviewer is ever notified, yet the job stays pending |  |
| Fifty high-severity findings and a green pipeline |  |
| The Security tab is empty on exactly the runs that found problems |  |
| The action fails to load before running a line of script |  |

**Options:** A composite `run` step with no `shell:` · A post-merge job listed as a required status check · A renamed team, so the reviewer reference resolves to nothing · A scanner with no failing exit code · `if: always()` missing on the SARIF upload · Success criteria not matching the actual response body

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| The PR shows a required check as "Expected" and never merges | **A post-merge job listed as a required status check** |
| A healthy API, a gate that never passes, no error for 24 hours | **Success criteria not matching the actual response body** |
| No reviewer is ever notified, yet the job stays pending | **A renamed team, so the reviewer reference resolves to nothing** |
| Fifty high-severity findings and a green pipeline | **A scanner with no failing exit code** |
| The Security tab is empty on exactly the runs that found problems | **`if: always()` missing on the SARIF upload** |
| The action fails to load before running a line of script | **A composite `run` step with no `shell:`** |

**In `challenge-17.md`:** lines **541–543**, **545–547**, **549–551**, **124**, **130**, **237**.

**The first three all present as waiting; the last three all present as success.** Between them they cover
every way this challenge can fail without producing a red mark.

</details>

---

## Q35

Match each GitHub Actions concept to its Azure Pipelines counterpart.

| GitHub Actions | Azure Pipelines |
|---|---|
| `environment: name: production` |  |
| Environment required reviewers |  |
| Required status checks on a branch |  |
| `microsoft/security-devops-action@v1` |  |
| No direct equivalent — checks run once per commit |  |
| `needs: [Build, SecurityScan]` |  |

**Options:** Approvals under Approvals and checks · Branch policies with required pipeline validation · `dependsOn: [Build, SecurityScan]` · `environment: 'production'` on a `deployment` job · Gates polled on `period` until `timeout` · `MicrosoftSecurityDevOps@1`

<details>
<summary>Show answer</summary>

| GitHub Actions | Azure Pipelines |
|---|---|
| `environment: name: production` | **`environment: 'production'` on a `deployment` job** |
| Environment required reviewers | **Approvals under Approvals and checks** |
| Required status checks on a branch | **Branch policies with required pipeline validation** |
| `microsoft/security-devops-action@v1` | **`MicrosoftSecurityDevOps@1`** |
| No direct equivalent — checks run once per commit | **Gates polled on `period` until `timeout`** |
| `needs: [Build, SecurityScan]` | **`dependsOn: [Build, SecurityScan]`** |

**In `challenge-17.md`:** lines **293–295** and **418–419**, **47** and **437**, **562**, **494** and
**409**, **468–474**, **291** and **416**.

**The fifth row is the one worth memorising.** GitHub has approvals, wait timers and branch policies, but
it has **no polling gate**. If a scenario needs "delay the release while an alert is active, and proceed
automatically when it clears", that is an Azure Pipelines gate — and in GitHub you would have to build it
as a job that waits.

</details>

---

# Section F — Hot area

---

## Q36

```bash
gh api repos/contoso-ltd/ecommerce-api/environments/production \
  --method PUT \
  --field [BLANK 1]=5 \
  --field [BLANK 2]='[{"type":"Team","id":12345}]' \
  --field [BLANK 3]='{"protected_branches":true,"custom_branch_policies":false}'
```

<details>
<summary>Show answer</summary>

### Answer

| Blank | Value |
|---|---|
| 1 | `wait_timer` |
| 2 | `reviewers` |
| 3 | `deployment_branch_policy` |

**In `challenge-17.md`:** lines **44–48**.

**The three protection rules a GitHub environment can carry** — a delay, a set of approvers, and a branch
restriction. Learn them as a trio; the exam asks which one solves a given requirement.

**Blank 2 is where Issue 3 lives.** That team ID is a number. When the team is renamed or deleted, the
number stops resolving and the rule stops applying (line 551) — **and nothing warns you**. The fix at line
589 verifies the team by listing its members before trusting it.

</details>

---

## Q37

```yaml
      - name: Run tests with coverage
        run: npx jest --coverage --ci --coverageReporters=[BLANK 1] --coverageReporters=lcov

      - name: Extract coverage percentage
        id: coverage
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '[BLANK 2]')
          echo "percentage=$COVERAGE" >> "[BLANK 3]"
```

<details>
<summary>Show answer</summary>

### Answer

| Blank | Value |
|---|---|
| 1 | `json-summary` |
| 2 | `.total.lines.pct` |
| 3 | `$GITHUB_OUTPUT` |

**In `challenge-17.md`:** lines **82** and **86–89**.

**Blank 1 is the dependency nobody notices until it is missing.** Drop `json-summary` and
`coverage-summary.json` is never written, so `jq` reads nothing, `COVERAGE` is empty, and the comparison
downstream misbehaves rather than failing cleanly.

**Blank 3 is the modern mechanism.** Appending `name=value` to the file named by `$GITHUB_OUTPUT` is what
replaced the deprecated `::set-output::` command, and it is what makes line 66's job-level
`outputs: coverage-percentage` resolvable.

**Blank 2 is `lines`, deliberately.** Contoso's requirement is line coverage — but see the trap index:
**line coverage is the forgiving metric.** Branch coverage is where untested `else` paths surface.

</details>

---

## Q38

```yaml
runs:
  using: '[BLANK 1]'
  steps:
    - name: Evaluate quality gates
      id: evaluate
      [BLANK 2]: bash
      run: |
        ...
        echo "summary[BLANK 3]" >> "$GITHUB_OUTPUT"
        echo -e "$SUMMARY" >> "$GITHUB_OUTPUT"
        echo "EOF" >> "$GITHUB_OUTPUT"
```

<details>
<summary>Show answer</summary>

### Answer

| Blank | Value |
|---|---|
| 1 | `composite` |
| 2 | `shell` |
| 3 | `<<EOF` |

**In `challenge-17.md`:** lines **232–237** and **274–276**.

**Blank 3 is the multi-line output idiom.** A step output is a single `name=value` line, so a value
containing newlines needs a delimiter block: open with `name<<EOF`, write the content, close with a line
containing only `EOF`.

**Get it wrong and the failure is confusing rather than fatal** — the first line becomes the value and the
rest is parsed as further assignments.

**Blank 1 distinguishes the three action types** — `composite` for steps in YAML, `node20` for JavaScript
actions, `docker` for container actions. Only `composite` requires blank 2 (Q8).

</details>

---

## Q39

```yaml
  deploy-production:
    runs-on: ubuntu-latest
    [BLANK 1]: [test-and-coverage, security-scan, performance-baseline]
    [BLANK 2]: github.ref == 'refs/heads/main' && github.event_name == 'push'
    [BLANK 3]:
      name: production
      url: https://api.contoso.com
```

<details>
<summary>Show answer</summary>

### Answer

| Blank | Value |
|---|---|
| 1 | `needs` |
| 2 | `if` |
| 3 | `environment` |

**In `challenge-17.md`:** lines **289–295**.

**Three keys, three independent gates** (Q18) — and each fails in a visibly different way. A `needs:`
failure shows the job **skipped**; a false `if:` shows it **skipped** with no upstream failure; an
unsatisfied environment rule shows it **waiting**.

**The `url:` is not a gate.** It records the deployment target so GitHub can show a link on the run and
on the environment's deployment history. Cosmetic, useful, and never blocking.

</details>

---

## Q40

```javascript
export const options = {
  vus: 20,
  duration: '30s',
  thresholds: {
    http_req_duration: ['[BLANK 1]'],
    http_req_failed: ['[BLANK 2]'],
    checks: ['[BLANK 3]'],
  },
};
```

<details>
<summary>Show answer</summary>

### Answer

| Blank | Value |
|---|---|
| 1 | `p(95)<200` |
| 2 | `rate<0.01` |
| 3 | `rate>0.99` |

**In `challenge-17.md`:** lines **335–343**, matching the mandate at line **29**.

**Three different kinds of wrong.** Too slow, errored, and answered-but-incorrect. A load test that only
measures latency will happily report 40 ms while every response is a 500.

**Note what the workflow passes and the script ignores.** Line 191 sets `K6_THRESHOLD_P95: 200` as an
environment variable, but the script reads only `__ENV.BASE_URL` (line 346) — the threshold is hardcoded
at line 339. **Changing the environment variable would change nothing**, which is exactly the kind of
silently-ineffective configuration the exam likes to place in a scenario.

</details>

---

## Q41

```json
{
  "settings": {
    "inputs": {
      "urlSuffix": "/api/health",
      "[BLANK 1]": "eq(root['status'], 'healthy')"
    }
  },
  "evaluationOptions": {
    "[BLANK 2]": 300000,
    "timeout": [BLANK 3],
    "initialDelay": 0
  }
}
```

<details>
<summary>Show answer</summary>

### Answer

| Blank | Value |
|---|---|
| 1 | `successCriteria` |
| 2 | `period` |
| 3 | `86400000` |

**In `challenge-17.md`:** lines **465** and **469–474**.

**Read the three numbers as a sentence:** start immediately, ask every five minutes, give up after
twenty-four hours.

**Blank 1 holds the bug.** The expression asks for `healthy`; the API returns `ok` (line 547). The gate is
working perfectly and blocking forever — and the fix at line 572 changes one word.

**The discipline at line 575 is the exam-ready version:** call the endpoint, read the response, then write
the criteria. Never the other way round.

</details>

---

# Section G — Case study

**Contoso Ltd — ecommerce-api.** A deployment carrying known high-severity vulnerabilities reached
production without review. The release manager has mandated five conditions before any production
deployment: over 80% coverage, zero critical or high dependency vulnerabilities, at least two approvals,
p95 latency below 200 ms, and no critical Defender for DevOps findings. The team runs GitHub Actions, with
some projects still on Azure DevOps. Gates must be implemented on both.

---

## Q42

The team wants the two-approval requirement to apply to **deployments**, not just to code review. What do
they configure?

- A. `required_pull_request_reviews` with a count of 2
- B. Required reviewers on the `production` environment
- C. A `needs:` entry for a manual-approval job
- D. `enforce_admins=true` on the protected branch

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-17.md`:** lines **44–48**, **437**.

**Deployment approvals belong to the environment.** They are requested when the job becomes eligible — after
the merge — and they hold the job in a pending state until an authorised reviewer responds.

**Why A is the other approval, and the challenge configures it too** (line 564). It gates the **merge**,
which is a decision about the change, not about the release (Q16).

**Why C does not exist in GitHub Actions.** There is no manual-approval job type; approval is an
environment feature. In Azure Pipelines the equivalent is likewise a check on the environment, not a task
in the YAML.

**Why D is orthogonal.** `enforce_admins=true` stops administrators bypassing branch protection — a good
setting, and unrelated to counting approvers.

</details>

---

## Q43

The release manager asks: "Can the pipeline pause a release automatically while production is alerting, and
resume on its own once the alert clears?" Which platform feature does that?

- A. A GitHub environment wait timer
- B. A required status check on the branch
- C. A branch control check on the environment
- D. An Azure Pipelines Azure Monitor alerts gate

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-17.md`:** line **438**, with the polling model at lines **468–474**.

**Only a gate re-asks the question.** The Azure Monitor gate evaluates, finds an active alert, waits
`period`, and evaluates again — so a transient alert delays the release and then lets it through with no
human involvement.

**Why A waits a fixed interval and then proceeds regardless.** A wait timer does not know what production
is doing.

**Why B is evaluated once per commit.** A status check is a fact about a build, not about the running
system.

**Why C restricts which branch may deploy** — the Azure DevOps equivalent of line 48.

**This is the clearest illustration of the paper's opening sentence.** The question "is production
healthy right now" has an answer that changes on its own, so it must be asked **repeatedly**. No
one-shot mechanism can express it.

</details>

---

## Q44

A developer adds Defender for DevOps exactly as shown in Task 6 and reports that critical findings still do
not stop a production deployment. Why?

- A. `defender-scan` is missing from `needs:` on deploy
- B. `security-events: write` is missing from the job
- C. The SARIF upload failed on the runner
- D. Defender findings are advisory only by design

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-17.md`:** lines **291** and **483–514**.

**The job fails correctly and nothing depends on it.** Lines 505–514 count error-level results and exit 1,
so `defender-scan` goes red — but `deploy-production` needs only `test-and-coverage`, `security-scan` and
`performance-baseline` (line 291). The deployment proceeds beside a failing job.

**This is requirement 5 not being met by the challenge's own workflow**, and it is the most
exam-realistic scenario in the file: **every part works, the wiring is missing.**

**The fix is one list entry**, plus adding `defender-scan` to the required status checks at line 562 so it
also blocks the merge.

**Why B and C would produce loud errors** in the upload step rather than a silently permitted deployment,
and **why D is false** — the job's own `exit 1` proves the findings are treated as blocking.

</details>

---

## Q45

Coverage is reported at 85% and the gate passes, yet a null-check branch that was never executed by any
test ships and causes an incident. What does this demonstrate?

- A. The line-coverage threshold should be raised to 95%
- B. Line coverage measures execution, not verification
- C. `json-summary` reported the wrong figure
- D. Coverage should be measured after deployment

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-17.md`:** lines **82**, **87**, **244–251**.

**`.total.lines.pct` is line coverage** (line 87). A line inside an `if` counts as covered the moment the
`if` is entered by any path; the `else` that was never taken does not reduce the number.

**And coverage of any kind measures execution, not verification.** A test with no assertions at all
executes the code and reports full coverage. Coverage tells you what was **not** tested; it cannot tell
you that what was tested was tested well.

**Why A treats a measurement problem as a number problem.** Raising a line-coverage threshold to 95%
produces tests written to touch lines, which is the pathology, not the cure.

**The right change is to gate on branches too** — the metric that forces both sides of every condition to
be executed, and the reason Challenge 18 exists.

</details>

---

## Q46

The Azure DevOps team reports that a release has been "in progress" for nine hours with no error. The
health endpoint returns HTTP 200. Where do you look first?

- A. Runner capacity available for the deployment stage
- B. The `dependsOn` list on the deployment stage
- C. The gate's `successCriteria` against the real response
- D. The service connection's credentials and scope

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-17.md`:** lines **465**, **545–547**, **575**.

**HTTP 200 and a blocked gate together point at exactly one thing: the criteria.** The request is
succeeding; the expression evaluating its body is not matching. `eq(root['status'], 'healthy')` against
`{"status": "ok"}` is negative every five minutes, forever, silently.

**Nine hours is diagnostic in itself.** It is longer than any real dependency takes and shorter than the
24-hour timeout at line 472 — so the release is still inside its polling window and has not yet produced
the one error message it will ever produce.

**Why A is the reflex answer to "slow" and the wrong one for "waiting".** No agent is running; the
deployment has not been dispatched.

**Why D would fail the request**, giving a non-200 and a visible gate error, and **why B would have
prevented the stage from starting at all** rather than leaving it in progress.

</details>

---

## Q47

Contoso wants a single reusable check that several repositories can call to evaluate coverage, security
and performance results together. Which approach matches Task 2?

- A. A reusable workflow called with `uses:` at job level
- B. A composite action with typed inputs, ending in `exit 1`
- C. A branch protection rule on each repository
- D. A required status check on each repository

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-17.md`:** lines **203–282**.

**A composite action packages steps for reuse inside a job**, takes declared inputs (lines 209–222),
publishes declared outputs (lines 224–230), and — critically — **can fail the calling job** with a
non-zero exit.

**Why A is a real and often better mechanism** for whole-job reuse, and it is not what Task 2 builds. A
reusable workflow is called at job level and brings its own runner; a composite action runs as steps
inside a job you already have, which is what a gate evaluated mid-job needs.

**Why C and D are settings, not code.** They decide **whether** a result blocks something. They cannot
compute one.

**The design point worth keeping:** the action takes results as inputs rather than re-running the tools.
That keeps the expensive work in parallel jobs and the decision in one cheap, testable place.

</details>

---

## Q48

Write the one-sentence rule that would have prevented all three issues in the break scenario.

- A. Always run security scans before any deployment
- B. Increase timeouts so gates have time to pass
- C. Every gate must be verified against what it gates
- D. Reduce the number of gates in the pipeline

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-17.md`:** lines **541–551**, **575**, **586–590**.

**Three issues, one shared shape: a gate that was never checked against the world it was gating.**

| Issue | The unverified assumption | Line |
|---|---|---|
| Required check deadlock | That `deploy-production` reports on pull requests | 541–543 |
| Health gate criteria | That the API returns `"status": "healthy"` | 545–547 |
| Missing reviewer team | That team ID 12345 still exists | 549–551 |

**Every one of them was configured correctly according to its own syntax.** No linter, no schema and no
dry run would have flagged any of them, because each is a statement about something outside the file.

**Which is why Fix 3 ends with a verification command** rather than a configuration change (line 589):

```bash
gh api orgs/contoso-ltd/teams/release-approvers/members --jq '.[].login'
```

**Why B treats a permanent failure as a slow one** — the gate at line 545 would not pass in a year — and
**why D throws away the mandate** rather than fixing the plumbing.

</details>

---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Protection rules assumed to live in YAML** | Q1, Q27, Q42 | The workflow names the environment; the rules sit on it |
| **`wait_timer` confused with a gate `period`** | Q2, Q28, Q32, Q43 | A timer waits once; a gate asks repeatedly |
| **Scanner with no failing exit code** | Q4, Q19, Q29, Q34 | `exit-code: '1'`, or an explicit `exit 1` after the count |
| **A scan whose exit code is swallowed and never re-raised** | Q5, Q25 | Swallowing is safe only when a later line can still fail |
| **SARIF upload treated as a gate** | Q19, Q29 | The Security tab is visibility, not enforcement |
| **`if: always()` missing on the upload** | Q7, Q29, Q34 | Findings are lost from exactly the runs that had them |
| **Composite `run` step with no `shell:`** | Q8, Q32, Q34, Q38 | The action fails to load, not to run |
| **Action outputs believed to gate** | Q9, Q10 | Only `exit 1` blocks; outputs are documentation |
| **`needs.*.result` reasoned about before asking whether the job runs** | Q11 | Without `if: always()`, a failed dependency skips the job |
| **Post-merge job as a required status check** | Q14, Q22, Q26, Q30 | Required checks must run on `pull_request` |
| **A failing gate job nothing depends on** | Q17, Q44 | Add it to `needs:` and to the required contexts |
| **Gate criteria never checked against the real response** | Q13, Q41, Q46, Q48 | Call the endpoint first, then write the expression |
| **A stale team or reference assumed still valid** | Q21, Q36, Q48 | It fails open and silently — verify membership |
| **"Waiting" treated as a diagnosis** | Q21, Q30, Q46 | It is the state of every unmet gate |
| **Line coverage trusted as proof of testing** | Q37, Q45 | It measures execution, not verification — use branches |
| **An unread environment variable assumed to configure a threshold** | Q40 | k6 reads `__ENV` only where the script says so |

---

# What to memorise

**In `challenge-17.md`:** lines **24–32**, **44–48**, **107–132**, **232–282**, **289–313**, **428–476**,
**541–590**.

```text
APPROVAL vs GATE - the sentence that decides half this paper
  APPROVAL   a person, ONCE      -> environment reviewers, PR review count
  WAIT TIMER a clock, ONCE       -> a fixed delay, asks nobody anything
  GATE       a system, REPEATEDLY-> Azure Pipelines only: initialDelay -> period -> timeout
  STATUS CHECK  a build, once per commit -> must run on pull_request to be requireable

REPORT vs GATE
  GATE     a non-zero exit code  |  a protection rule on the environment
  REPORT   SARIF upload | artifacts | PublishTestResults | the deployment url:
  ...and a failing job blocks a merge only when it is a REQUIRED STATUS CHECK
  ...and it blocks a DEPLOY only when something has it in needs:

WHERE THINGS LIVE - the reason the break is invisible
  quality-gates.yml   ->  names the environment
  the ENVIRONMENT     ->  reviewers, wait timer, deployment branch policy
  BRANCH PROTECTION   ->  required contexts, review count, enforce_admins
  none of these three can see the others
```

```bash
# GitHub environment - the three protection rules       (lines 44-48)
gh api repos/OWNER/REPO/environments/production --method PUT \
  --field wait_timer=5 \
  --field reviewers='[{"type":"Team","id":12345}]' \
  --field deployment_branch_policy='{"protected_branches":true,"custom_branch_policies":false}'
#   a renamed team -> the ID stops resolving -> the rule is SILENTLY SKIPPED   (line 551)
#   verify it:  gh api orgs/OWNER/teams/release-approvers/members --jq '.[].login'

# Branch protection - only PR-TIME checks               (lines 558-564)
--field required_status_checks='{"strict":true,"contexts":["test-and-coverage","security-scan","performance-baseline"]}'
--field required_pull_request_reviews='{"required_approving_review_count":2}'
#   deploy-production here = DEADLOCK. it runs on push to main, after the merge   (541-543)
```

```yaml
# The security job - two scanners, two exit codes       (lines 97-132)
permissions: {security-events: write, contents: read}   # write REPLACES the defaults
- run: |
    npm audit --audit-level=high --json > audit-results.json || true   # || true so the SCRIPT decides
    CRITICAL=$(jq '.metadata.vulnerabilities.critical // 0' audit-results.json)
    if [ "$CRITICAL" -gt 0 ]; then echo "::error::..."; exit 1; fi      # <- THIS is the gate
- uses: aquasecurity/trivy-action@0.28.0
  with: {scan-type: 'fs', severity: 'CRITICAL,HIGH', exit-code: '1', format: 'sarif'}
#        exit-code: '1'  <- without it: findings, SARIF, and a GREEN tick
- uses: github/codeql-action/upload-sarif@v3
  if: always()          # or the Security tab only ever gets CLEAN runs

# Coverage extraction                                    (lines 82-89)
npx jest --coverage --ci --coverageReporters=json-summary --coverageReporters=lcov
#   json-summary -> coverage/coverage-summary.json  (the file jq reads)
COVERAGE=$(jq '.total.lines.pct' coverage/coverage-summary.json)
echo "percentage=$COVERAGE" >> "$GITHUB_OUTPUT"
#   .lines is FORGIVING. branches is the metric that finds untested else paths
```

```yaml
# Composite action - reusable, and it can FAIL the caller  (lines 203-282)
runs:
  using: 'composite'
  steps:
    - id: evaluate
      shell: bash            # MANDATORY on every composite run step
      run: |
        echo "passed=$PASSED" >> "$GITHUB_OUTPUT"
        echo "summary<<EOF" >> "$GITHUB_OUTPUT"   # multi-line output needs a delimiter
        echo -e "$SUMMARY"  >> "$GITHUB_OUTPUT"
        echo "EOF"          >> "$GITHUB_OUTPUT"
        if [ "$PASSED" = "false" ]; then exit 1; fi   # the OUTPUT does not gate. the EXIT does

# The deploy job - three independent gates                (lines 289-295)
needs: [test-and-coverage, security-scan, performance-baseline]   # skipped if any fails
if: github.ref == 'refs/heads/main' && github.event_name == 'push'
environment: {name: production, url: https://api.contoso.com}     # url is COSMETIC
#   order: if -> needs -> environment rules -> steps -> the in-job gate action
```

```json
// Azure Pipelines gate - the only POLLING mechanism      (lines 444-476)
{ "settings": { "inputs": {
      "urlSuffix": "/api/health",
      "successCriteria": "eq(root['status'], 'ok')"   // verify against the REAL response first
  } },
  "evaluationOptions": { "initialDelay": 0, "period": 300000, "timeout": 86400000 } }
//   period 300000 ms = every 5 min   timeout 86400000 ms = 24 h
//   a WRONG criterion looks exactly like a SLOW system, for a full day
```

```yaml
# Azure Pipelines equivalents                            (lines 364-426)
- stage: DeployProduction
  dependsOn: [Build, SecurityScan]          # = needs:
  jobs:
    - deployment: DeployProd
      environment: 'production'             # approvals + gates configured ON the environment
- task: MicrosoftSecurityDevOps@1           # = microsoft/security-devops-action@v1
  inputs: {categories: 'secrets,dependencies'}
- task: PublishTestResults@2
  condition: always()                       # = if: always()
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 18 |
| 38–43 | Re-read the trap index and the approval-versus-gate table, then move on |
| 30–37 | Write the three protection rules and the three break issues from memory, then retake |
| Below 30 | Redo Tasks 1, 3 and 5 hands-on, then work the break scenario before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 17.

:::danger The three rules

**An approval asks a person once; a gate asks a system repeatedly.** Wait timers and status checks are
one-shot. Only Azure Pipelines gates poll — `initialDelay`, then `period`, until `timeout` — and that is
the only way to say "wait until production is healthy, then continue".

**Nothing gates unless something refuses to proceed.** A scanner needs a non-zero exit code, a failing job
needs to be in `needs:` and in the required contexts, and an action's outputs never block anything —
`exit 1` does.

**Verify every gate against the world it gates.** The check must run on the event you want to block, its
criteria must match the response the endpoint actually returns, and the team it names must still exist.
All three break-scenario issues were valid configuration pointing at something that was not true.

:::
