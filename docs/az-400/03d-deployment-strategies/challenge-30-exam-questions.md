---
sidebar_position: 6.5
toc_max_heading_level: 2
title: "Challenge 30: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 30 — AZ-400 exam questions

**48 questions** built only from what Challenge 30 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-30.md`**.

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

:::danger Production is down — but not every gate may be skipped

A hotfix pipeline trades **thoroughness** for **speed**. The exam always asks *which* gates survive.
Security scanning never goes. Integration tests and progressive rollout do. Know the table at lines
220–228 cold.

:::

---

# Section A — Single answer

---

## Q1

From where should a hotfix branch be created?

- A. From `main`
- B. From the release tag of the broken version
- C. From the previous known-good release tag
- D. From `develop`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-30.md`:** lines **36–40**.

```bash
# "The hotfix branch is created from the release tag (not from main)
#  to avoid picking up unreleased changes."
git checkout -b hotfix/payment-over-10k release/2.4.0
```

**Why this is the whole idea.** `main` contains everything merged since v2.4.0 shipped — unreleased,
untested-in-production features. Branching from `main` would ship all of them alongside your one-line
fix, in an expedited pipeline that has **skipped integration tests**.

You want production plus one change, and nothing else.

**Why the others fail**

- **A** — drags in unreleased work
- **C** — **tempting and wrong.** `release/2.3.1` is the last known-good version (line 27), but
  branching there would *undo* everything v2.4.0 delivered. That is a **rollback**, not a hotfix, and
  it is a different decision
- **D** — no `develop` branch exists in this model, and it would have the same problem as `main`

</details>

---

## Q2

Which gate must **never** be skipped in a hotfix pipeline?

- A. Integration tests
- B. Security scanning
- C. Progressive rollout through rings
- D. Full regression smoke tests

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-30.md`:** line **131** — the step is literally named *"Security scan (never skip
this)"* — and the comparison table at lines **220–228**.

| Gate | Normal | Hotfix |
|---|---|---|
| Unit tests | Full suite (~15 min) | **Critical tests only** (~2 min) |
| Integration tests | Full suite (~30 min) | **Skipped** |
| **Security scan** | Full SAST + DAST | **SAST only (CodeQL)** — reduced, never removed |
| Manual approval | 2 reviewers | Single on-call lead |
| Progressive rollout | Rings 0–3 (48 hrs) | Direct to production |
| Total | ~2 hours | ~10–15 minutes |

**Why security survives:** a functional regression is visible and reversible in minutes. A credential
leak or an injection flaw is neither — you cannot un-disclose a secret, and an urgent fix written
under pressure is exactly when one gets committed.

**Why the others fail** — all three are explicitly reduced or skipped in the table.

</details>

---

## Q3

A hotfix deploys and the swap succeeds, but customers still hit the bug.

What is the most likely cause?

- A. The swap did not complete
- B. Traffic routing is still sending a share of traffic to the staging slot
- C. The build used the wrong branch
- D. The health check passed incorrectly

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-30.md`:** Break & fix Exercise 1, lines **687–718**.

> *"Traffic routing was still configured to send 10% to the staging slot from a previous canary test.
> After the swap, the 'staging' slot now contains the OLD (broken) production code, and 10% of traffic
> goes there."*

```bash
az webapp traffic-routing clear \
  --name app-contoso-payments \
  --resource-group rg-contoso-prod
```

**Follow what the swap does.** Before: production = broken v2.4.0, staging = fixed v2.4.1. After the
swap they **exchange**, so staging now holds the broken code. A leftover 10% routing rule keeps
sending one in ten customers to it.

**This is why line 587 clears routing *before* swapping** in the circuit-breaker flow — the order
matters.

**Why the others fail**

- **A** — the question says the swap succeeded
- **C** — a wrong branch would fail for **all** customers, not some
- **D** — a health check on `/health` cannot detect a payment-amount bug. It confirms the app is up

</details>

---

## Q4

Which condition makes the frontend deployment wait for the backend?

- A. `dependsOn: DeployBackend`
- B. `condition: always()`
- C. `pool: vmImage`
- D. A longer timeout on the frontend stage

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-30.md`:** Break & fix Exercise 2, lines **723–737**.

```yaml
- stage: DeployFrontend
  dependsOn: DeployBackend          # this ensures ordering
  condition: succeeded('DeployBackend')
```

> *"The frontend deployment completes before the Payment Service API is ready, causing UI errors for
> 2 minutes."*

**Both lines do different jobs**, and the exam separates them: `dependsOn` decides **when** the stage
may start; `condition` decides **whether** it runs once it may.

**Why the others fail**

- **B** — `always()` would deploy the frontend **even if the backend failed**, which is worse than the
  original bug
- **C** — selects an agent
- **D** — a timeout does not create ordering. Adding a sleep and hoping is the classic wrong instinct

**This is the `dependsOn` lesson again** — Challenge 22 Q13, Challenge 29 Q2. Third appearance, and
it is on the exam every time.

</details>

---

## Q5

What does a deployment circuit breaker do?

- A. Restarts the application when errors are detected
- B. Halts progressive rollout when a failure threshold is exceeded
- C. Limits how many pipelines run concurrently
- D. Throttles requests to a failing dependency

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-30.md`:** lines **504–506** and **526–551**.

```bash
          ERROR_THRESHOLD=5
          ERROR_COUNT=0
          while [ $ELAPSED -lt $MONITORING_DURATION ]; do
            if [ "$STATUS" != "200" ]; then
              ERROR_COUNT=$((ERROR_COUNT + 1))
              if [ $ERROR_COUNT -ge $ERROR_THRESHOLD ]; then
                echo "CIRCUIT BREAKER TRIPPED: Error threshold exceeded"
                exit 1
              fi
            fi
```

**Note it counts errors rather than failing on the first one.** A single failed probe during a rollout
is normal — an instance restarting, a transient network blip. Five within two minutes is a pattern.

**Why the others fail**

- **A** — it halts and reverts the rollout; it does not restart anything
- **C** — that is concurrency (Challenge 24)
- **D** — **the near-miss.** That is the *application-level* circuit breaker from resilience
  engineering, protecting a caller from a failing dependency. This is a **deployment** circuit
  breaker: same name, same idea, applied to a rollout

</details>

---

## Q6

Which step runs when the circuit breaker trips?

- A. `if: success()` — promote to 50%
- B. `if: failure()` — clear traffic routing
- C. `if: always()` — notify the team
- D. `continue-on-error: true` — proceed anyway

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-30.md`:** lines **553–559**.

```yaml
      - name: Circuit breaker - halt and rollback
        if: failure()
        run: |
          az webapp traffic-routing clear \
            --name app-contoso-payments \
            --resource-group rg-contoso-prod
```

**`traffic-routing clear` is the revert.** The canary code lives in the staging slot; clearing the
routing rule sends 100% of traffic back to production. No deployment, no swap — the rollback is
removing a rule.

**Why the others fail**

- **A** — `success()` promotes to 50% (line 562), the opposite path
- **C** — notification is useful and changes nothing
- **D** — would let a tripped breaker continue the rollout, defeating the entire mechanism

</details>

---

## Q7

In the progressive rollout, what happens immediately before the final swap to 100%?

- A. Traffic routing is cleared
- B. The staging slot is deleted
- C. A new revision is created
- D. The canary weight is set to 100

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-30.md`:** lines **584–594**.

```yaml
      - name: Promote to 100% (swap)
        if: success()
        run: |
          az webapp traffic-routing clear ...
          az webapp deployment slot swap --slot staging --target-slot production
```

**Clear, then swap — and the order is not cosmetic.** If you swap while a 50% routing rule is still
in place, the slots exchange and that rule now sends 50% of traffic to the **old** code sitting in
staging. That is Break & fix Exercise 1, arriving from a different direction.

**Why the others fail**

- **B** — the staging slot is what makes rollback possible. Deleting it destroys the way back
- **C** — revisions are Container Apps (line 601); this is App Service
- **D** — setting the routing weight to 100 would route everything at staging **without swapping**,
  leaving production still holding the old code and the deployment in a half-finished state

</details>

---

## Q8

Why does the emergency rollback script swap the **staging** slot back into production?

- A. Staging always contains the previous production version after a swap
- B. Staging is a copy of the last backup
- C. Staging is redeployed automatically
- D. Production cannot be modified directly

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-30.md`:** lines **403–408**.

```bash
# The staging slot contains the previous production version after a swap
az webapp deployment slot swap \
  --slot staging --target-slot production
```

**A swap is an exchange, not a copy.** The version that was in production is now sitting in staging,
still warm. Swapping again puts it straight back.

**The corollary the exam tests:** you get **one** free rollback. Swap a third time and you are back to
the broken version. If two bad releases go out in succession, the previous-but-one build is gone and
you must redeploy it.

**Why the others fail** — B, C and D all misdescribe how slots work.

**Note `set -euo pipefail` at line 396:** exit on error, on undefined variable, and on any failure in
a pipe. In an emergency script, silently continuing after a failed command is the worst outcome.

</details>

---

## Q9

Which `dotnet test` filter runs only critical tests in the hotfix pipeline?

- A. `--filter "Category=Critical|Category=Payment"`
- B. `--filter "FullyQualifiedName~Critical"`
- C. `--no-build`
- D. `--configuration Release`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-30.md`:** lines **124–129**.

```bash
          dotnet test tests/PaymentService.UnitTests/PaymentService.UnitTests.csproj \
            --configuration Release \
            --filter "Category=Critical|Category=Payment" \
            --no-build
```

**The `|` is OR**, so tests carrying either trait run. That takes ~2 minutes instead of ~15 (line
222).

**What this requires beforehand:** the tests must already be **categorised**. `[Trait("Category",
"Critical")]` has to exist in the codebase before the incident. A hotfix path only works if it was
built in advance — during the outage is too late to start tagging tests.

**Why the others fail**

- **B** — filters by test name, which is brittle and depends on naming discipline
- **C** — skips rebuilding, a speed optimisation with no selection effect
- **D** — the build configuration

</details>

---

## Q10

Which trigger starts the hotfix workflow?

- A. `push` to `hotfix/**` branches and `release/*.*.*` tags
- B. `pull_request` to `main`
- C. `workflow_dispatch` only
- D. `schedule`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-30.md`:** lines **96–101**.

```yaml
on:
  push:
    branches:
      - 'hotfix/**'
    tags:
      - 'release/*.*.*'
```

**Two entry points, deliberately.** Pushing the branch starts building immediately; pushing the tag
marks the release. During an incident you want the pipeline moving the moment code is pushed, not
after a PR review cycle.

**Why the others fail**

- **B** — a PR workflow means waiting for review before anything builds. Production is down
- **C** — manual only would need someone to remember to click it
- **D** — a schedule is unrelated to an incident

**Note `hotfix/**` uses a double asterisk** so it matches `hotfix/payment-over-10k` and any nested
path.

</details>

---

## Q11

Which environment does the hotfix deployment job target?

- A. `production`
- B. `production-hotfix`
- C. `staging`
- D. No environment

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-30.md`:** line **151**.

```yaml
  deploy-hotfix:
    needs: build-and-scan
    environment: production-hotfix
```

**A separate environment exists so it can have different protection rules.** The table at line 225
says normal releases need **two reviewers**; a hotfix needs **one on-call lead**. Both are approvals —
different environments, different rules.

That is also why the answer is not D: even in an emergency there is a human gate and a recorded
deployment. It is *lighter*, not *absent*.

**Why the others fail**

- **A** — would inherit the two-reviewer rule and stall the hotfix
- **C** — the hotfix targets production
- **D** — no environment means no approval and no deployment record

</details>

---

## Q12

What does the cherry-pick workflow do when the cherry-pick conflicts?

- A. Fails the workflow
- B. Creates a branch and opens a pull request for manual resolution
- C. Force-pushes the change to `main`
- D. Retries with `--strategy=ours`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-30.md`:** lines **665–679**.

```bash
          else
            git cherry-pick --abort
            git checkout -b auto/cherry-pick-$HOTFIX_SHA
            git cherry-pick $HOTFIX_SHA || true
            git add -A
            git commit -m "fix: cherry-pick hotfix (conflicts resolved manually)"
            git push origin auto/cherry-pick-$HOTFIX_SHA
            gh pr create --title "Cherry-pick hotfix to main (conflicts)" --base main ...
```

**Automate the easy path, escalate the hard one.** A clean cherry-pick is pushed straight to `main`; a
conflicted one becomes a reviewable PR. The automation never guesses at a merge.

**Why the others fail**

- **A** — failing silently loses the fix. `main` would ship without it in the next release, and the
  bug returns
- **C** — force-pushing to `main` in an automated workflow is destructive
- **D** — `--strategy=ours` **discards the incoming change**, so the fix would not be applied at all

</details>

---

## Q13

Why does the cherry-pick workflow use `fetch-depth: 0`?

- A. To fetch the full history so cherry-pick can resolve commits
- B. To speed up the checkout
- C. To fetch only the latest commit
- D. To avoid fetching tags

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-30.md`:** lines **642–645**.

```yaml
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.PAT_TOKEN }}
```

`actions/checkout` defaults to a **shallow** clone of depth 1. Cherry-picking needs the commit graph
of both branches, plus `git describe --tags` (line 659) needs tag history. Neither works on a shallow
clone.

**Why the others fail**

- **B** — `0` means **full** history, which is slower
- **C** — that is the default, depth 1
- **D** — tags are controlled by `fetch-tags`

**Also note `token: ${{ secrets.PAT_TOKEN }}`.** This is Q16's subject: pushing to `main` needs a
credential the default `GITHUB_TOKEN` may not have.

</details>

---

## Q14

Which Container Apps commands implement a revision-based circuit breaker?

- A. `az containerapp update --revision-suffix` then `ingress traffic set` with weights
- B. `az containerapp restart`
- C. `az containerapp revision deactivate`
- D. `az containerapp env update`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-30.md`:** lines **599–618**.

```bash
az containerapp update --image ...:v2.4.1 --revision-suffix hotfix-v241

az containerapp ingress traffic set \
  --revision-weight "ca-contoso-payments--hotfix-v241=10" \
  --revision-weight "ca-contoso-payments--stable=90"

# If errors detected, immediately route all traffic back
az containerapp ingress traffic set \
  --revision-weight "ca-contoso-payments--stable=100"
```

**`--revision-suffix` gives the revision a readable name**, which matters when you must type the
rollback command under pressure — `--hotfix-v241` is legible where a generated hash is not.

**Why the others fail**

- **B** — restarts the same code
- **C** — deactivating stops a revision entirely. Weight 0 keeps it available for inspection
- **D** — environment-level settings

**Same pattern as App Service, different nouns:** slots and traffic routing there, revisions and
weights here. Both keep the old version alive so reverting is a routing change.

</details>

---

## Q15

In the dependency graph, which build jobs depend on `BuildCore`?

- A. Only `BuildPaymentService`
- B. `BuildPaymentService` and `BuildNotificationService`
- C. All build jobs including `BuildFrontend`
- D. None — they build in parallel

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-30.md`:** lines **259–316**.

```yaml
      - job: BuildCore                    # the shared NuGet package
      - job: BuildPaymentService
        dependsOn: BuildCore              # line 274
      - job: BuildNotificationService
        dependsOn: BuildCore              # line 292
      - job: BuildFrontend                # no dependsOn - runs in parallel
```

**Both consumers wait for the shared library and then run in parallel with each other** — same
upstream, so they are siblings. `BuildFrontend` depends on nothing and starts immediately.

**This is the parallelism graph from Challenge 22 Q13.** Same upstream = parallel. Chained = serial.
Adding `dependsOn: BuildPaymentService` to the notification service would serialise two jobs that have
no reason to wait.

**Why they must wait at all (line 281):**

```bash
dotnet nuget add source $(Pipeline.Workspace)/core-package --name local
```

They consume the freshly built package from the pipeline artifact, so it has to exist first.

</details>

---

## Q16

Why does the cherry-pick workflow use `secrets.PAT_TOKEN` rather than `secrets.GITHUB_TOKEN`?

- A. `GITHUB_TOKEN` cannot push to a protected `main` branch or trigger downstream workflows
- B. `GITHUB_TOKEN` expires too quickly
- C. `PAT_TOKEN` is faster
- D. `GITHUB_TOKEN` cannot read the repository

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-30.md`:** line **645**.

**Two real limits, and both apply here:**

1. **Pushes made with `GITHUB_TOKEN` do not trigger further workflows.** That is deliberate, to stop
   infinite loops — but it means the push to `main` would run no CI at all
2. Branch protection on `main` typically requires a pull request. A PAT or GitHub App belonging to a
   principal with an appropriate bypass can push where the workflow token cannot

**Why the others fail**

- **B** — `GITHUB_TOKEN` lasts for the run, which is ample
- **C** — no performance difference
- **D** — it reads fine; the workflow checks out with it by default

**The better answer in production:** a **GitHub App** installation token rather than a personal PAT.
An App has its own identity, scoped permissions and no dependency on an individual's account — which
is exactly Challenge 40's subject, and why your `deploy-manifests` repo exists.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** gates does the hotfix pipeline reduce or skip? (Choose three.)

- A. Integration tests — skipped entirely
- B. Progressive ring rollout — replaced by direct-to-production
- C. Manual approval — reduced from two reviewers to one
- D. Security scanning — removed
- E. Unit tests — removed
- F. Health checks — removed

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-30.md`:** the comparison table, lines **220–228**.

**Why the others fail — each is *reduced*, not removed**

- **D** — full SAST + DAST becomes **SAST only (CodeQL)**. The step is named *"never skip this"*
  (line 131)
- **E** — the full suite becomes **critical tests only** (line 128)
- **F** — full regression becomes a **health check** (line 171)

**The pattern to read off the table:** gates that verify *this specific change* survive in reduced
form. Gates that verify *the whole system* are deferred. Nothing is removed outright.

</details>

---

## Q18

Which **two** are true about the hotfix branching model? (Choose two.)

- A. The branch is created from the broken release tag
- B. The fix must be cherry-picked back to `main` afterwards
- C. The branch is created from `main`
- D. The hotfix branch becomes the new `main`
- E. The fix is merged to `main` before deploying

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-30.md`:** lines **36–40** (A) and **61–83** (B).

**Why B is not optional:** the hotfix branch was cut from a *tag*, so `main` never receives the fix.
Forget the cherry-pick and the next regular release **reintroduces the bug** — the most common
hotfix failure in real teams, and it always shows up weeks later when nobody connects the two events.

**Why the others fail**

- **C** — brings in unreleased work
- **D** — the hotfix branch is deleted after the cherry-pick (line 81)
- **E** — merging to `main` first means shipping everything else on `main` too, and it costs time
  production does not have

</details>

---

## Q19

Which **two** happen when the circuit breaker trips? (Choose two.)

- A. The step exits non-zero
- B. Traffic routing is cleared, returning all traffic to production
- C. The application is restarted
- D. The staging slot is deleted
- E. The rollout promotes to 50% anyway

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-30.md`:** lines **541–544** and **553–559**.

```bash
              if [ $ERROR_COUNT -ge $ERROR_THRESHOLD ]; then
                echo "CIRCUIT BREAKER TRIPPED: Error threshold exceeded"
                echo "tripped=true" >> $GITHUB_OUTPUT
                exit 1
              fi
```

**The `exit 1` is what makes the mechanism work.** It is the signal that turns the later
`if: failure()` step on and the `if: success()` promotion steps off. Without it the breaker detects
and does nothing.

**Why the others fail**

- **C** — the canary code is not restarted; traffic simply stops reaching it
- **D** — the slot is preserved for diagnosis
- **E** — promotion is guarded by `if: success()` (line 563)

</details>

---

## Q20

Which **two** are correct about the progressive rollout stages? (Choose two.)

- A. 10% canary, then 50%, then 100% via slot swap
- B. Traffic routing is cleared before the final swap
- C. Traffic weight is set to 100% instead of swapping
- D. Each stage runs regardless of the previous result
- E. The 50% stage skips monitoring

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-30.md`:** lines **519–594**.

```bash
--distribution staging=10      # canary
--distribution staging=50      # promote
az webapp traffic-routing clear && az webapp deployment slot swap   # 100%
```

**Why B matters so much:** swapping with a routing rule still active means the rule now points at the
**old** code in staging (Break & fix Exercise 1). Clear first, then swap.

**Why the others fail**

- **C** — weight 100 leaves production still holding the old code and the deployment unfinished
- **D** — every promotion step is `if: success()` (lines 563, 571, 585)
- **E** — the 50% stage monitors too (lines 570–582)

</details>

---

## Q21

Which **two** ensure services deploy in dependency order? (Choose two.)

- A. `dependsOn` between stages
- B. `condition: succeeded('DeployBackend')`
- C. A `sleep` before the frontend deployment
- D. `condition: always()` on the frontend stage
- E. Deploying everything in one stage

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-30.md`:** lines **733–737**.

```yaml
- stage: DeployFrontend
  dependsOn: DeployBackend
  condition: succeeded('DeployBackend')
```

**They are complementary.** `dependsOn` waits; `condition` checks the outcome. Note that once you
write a custom `condition`, the implicit `succeeded()` is **replaced** — which is why it is stated
explicitly here (Challenge 20 Q20).

**Why the others fail**

- **C** — a sleep is a guess. It fails when the backend is slower than expected and wastes time when
  it is faster
- **D** — deploys the frontend even after the backend failed
- **E** — one stage means no ordering, no separate approvals and no clean failure boundary. This is
  the "merge the jobs" distractor from Challenge 22 Q26

</details>

---

## Q22

Which **two** are true about the shared-library dependency graph? (Choose two.)

- A. `BuildPaymentService` and `BuildNotificationService` run in parallel with each other
- B. Both consume the Core package from a pipeline artifact via a local NuGet source
- C. `BuildFrontend` depends on `BuildCore`
- D. `BuildNotificationService` depends on `BuildPaymentService`
- E. `BuildCore` runs last

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-30.md`:** lines **272–302**.

```yaml
      - job: BuildPaymentService
        dependsOn: BuildCore
        steps:
          - task: DownloadPipelineArtifact@2
          - script: |
              dotnet nuget add source $(Pipeline.Workspace)/core-package --name local
```

**Both depend on `BuildCore`, so they are siblings and run in parallel.** Adding a dependency between
them would serialise two jobs with nothing to wait for.

**Why B is the mechanism that makes ordering necessary:** they build against the **freshly packed**
Core package from this run, not the published feed version. The artifact must exist first.

**Why the others fail**

- **C** — the frontend has no `dependsOn` (line 308) and starts immediately
- **D** — would serialise them for no reason
- **E** — `BuildCore` runs first; everything else waits on it

</details>

---

## Q23

Which **two** cause a cherry-pick to `main` to conflict? (Choose two.)

- A. `main` has diverged significantly since the release tag
- B. Refactoring moved the affected code to a different file
- C. The hotfix branch was deleted
- D. `fetch-depth: 0` was not set
- E. The tag was pushed before the branch

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-30.md`:** Break & fix Exercise 3, lines **742–765**.

> *"The file structure on `main` changed (refactoring moved the affected code), so the cherry-pick
> cannot apply cleanly."*

**The fix is a manual port, not a cleverer cherry-pick** (lines 753–765): branch from `main`, apply
the **logical** fix in its new location, open a PR for review.

That distinction is worth holding: a cherry-pick moves a **patch**; when the surrounding code has
moved, what you actually need to move is the **change in behaviour**.

**Why the others fail**

- **C** — the commit SHA still exists in the repository
- **D** — a shallow clone makes cherry-pick **fail to find commits**, a different error from a
  conflict
- **E** — push order does not affect merging

</details>

---

# Section C — Repeated scenario

**Scenario:** Production v2.4.0 is dropping payment transactions over $10,000. The normal pipeline
takes 2 hours. Contoso needs a verified fix in production in under 15 minutes, without shipping
unreleased work and without skipping security checks.

---

## Q24

**Proposed solution:** Branch from `release/2.4.0`, apply the minimal fix, run critical tests and
CodeQL, deploy to the staging slot, health-check for 30 seconds, swap to production, then cherry-pick
to `main`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-30.md`:** lines **40–58**, **108–216**, **63–83**.

Every constraint is satisfied:

| Constraint | How |
|---|---|
| Under 15 minutes | Critical tests only, no integration suite, direct to production (line 228) |
| No unreleased work | Branch from the **tag**, not `main` (line 36) |
| Security kept | CodeQL still runs (line 131) |
| Fix reaches future releases | Cherry-pick to `main` (line 68) |

The cherry-pick is the part that is easy to omit and the reason the bug would otherwise return.

</details>

---

## Q25

**Proposed solution:** Branch from `main`, apply the fix, run the full test suite and security scan,
deploy through rings 0 to 3, then merge to `main`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Two failures, and both are disqualifying.**

**Time.** The full suite plus ring rollout is the ~2 hour path (line 228). Rings alone take 48 hours
in the normal model (line 226). Customers are losing transactions.

**Unreleased work.** Branching from `main` ships every unmerged-to-production change alongside the
fix — and does so through a pipeline that, in this proposal, has not tested that combination in
production.

The safety is real, and the requirement was **15 minutes**. This is the normal pipeline wearing a
hotfix label.

</details>

---

## Q26

**Proposed solution:** Branch from `release/2.4.0`, apply the fix, skip all tests and the security
scan to save time, deploy straight to the production slot, and verify manually afterwards.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Fast enough, and wrong for two reasons.

**Security is non-negotiable** (line 131). An urgent fix written under incident pressure is exactly
when a hardcoded credential or an unsanitised input gets committed. A functional regression is
reversible in seconds; a leaked secret is not reversible at all.

**Deploying straight to production removes the rollback path.** The whole reason the pipeline deploys
to **staging** and then swaps (lines 164–191) is that the swap leaves the previous version in the
staging slot, ready to swap back. Deploy directly to production and the previous build is simply
overwritten — you have no way back at the moment you are most likely to need one.

**The exam pattern:** "skip everything to go faster" is never right. The correct hotfix path is
**reduced** gates, not **absent** ones.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — hotfix branching

| # | Statement | Answer |
|---|---|---|
| 1 | A hotfix branch is created from the release tag |  |
| 2 | Branching from `main` avoids unreleased changes |  |
| 3 | The fix must be cherry-picked to `main` afterwards |  |
| 4 | The hotfix branch is deleted after the cherry-pick |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A hotfix branch is created from the release tag | **Yes** |
| 2 | Branching from `main` avoids unreleased changes | **No** |
| 3 | The fix must be cherry-picked to `main` afterwards | **Yes** |
| 4 | The hotfix branch is deleted after the cherry-pick | **Yes** |

**In `challenge-30.md`:** lines **36–40**, **68**, **81–82**.

Row 2 is the statement inverted — branching from `main` **includes** unreleased changes, which is
precisely why the model avoids it.

Row 3 is the step teams forget. The hotfix ships from a tag-based branch, so `main` never sees it, and
the next release quietly reintroduces the bug.

</details>

---

## Q28 — expedited pipeline

| # | Statement | Answer |
|---|---|---|
| 1 | Integration tests are skipped in the hotfix pipeline |  |
| 2 | Security scanning is skipped in the hotfix pipeline |  |
| 3 | The hotfix uses a separate environment with a lighter approval |  |
| 4 | The hotfix deploys to staging and swaps, rather than straight to production |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Integration tests are skipped in the hotfix pipeline | **Yes** |
| 2 | Security scanning is skipped in the hotfix pipeline | **No** |
| 3 | The hotfix uses a separate environment with a lighter approval | **Yes** |
| 4 | The hotfix deploys to staging and swaps, rather than straight to production | **Yes** |

**In `challenge-30.md`:** lines **220–228**, **151**, **164–191**.

Row 4 is easy to misread, because the step at line 164 is *named* "Deploy directly to production
slot" while its `slot-name` input is `staging`. The **swap** is what promotes it — and that is what
preserves the rollback path.

Row 3: `production-hotfix` exists so a single on-call lead can approve, where `production` requires
two reviewers.

</details>

---

## Q29 — circuit breaker and rollback

| # | Statement | Answer |
|---|---|---|
| 1 | The circuit breaker trips on the first failed health check |  |
| 2 | Tripping clears traffic routing, restoring 100% to production |  |
| 3 | Traffic routing must be cleared before the final swap |  |
| 4 | Swapping twice returns you to the broken version |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | The circuit breaker trips on the first failed health check | **No** |
| 2 | Tripping clears traffic routing, restoring 100% to production | **Yes** |
| 3 | Traffic routing must be cleared before the final swap | **Yes** |
| 4 | Swapping twice returns you to the broken version | **Yes** |

**In `challenge-30.md`:** lines **531–545**, **557**, **587–594**.

Row 1: `ERROR_THRESHOLD=5`. Counting rather than reacting to the first error is what stops a transient
blip aborting a good rollout.

Row 4 is the limit worth remembering: a swap **exchanges**, so you get exactly **one** free rollback.
A third swap puts the broken build back into production.

</details>

---

## Q30 — dependency ordering

| # | Statement | Answer |
|---|---|---|
| 1 | Jobs sharing `dependsOn: BuildCore` run in parallel |  |
| 2 | A `sleep` is an acceptable substitute for `dependsOn` |  |
| 3 | `condition: succeeded('DeployBackend')` replaces the implicit `succeeded()` |  |
| 4 | Frontend must deploy before the backend API |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Jobs sharing `dependsOn: BuildCore` run in parallel | **Yes** |
| 2 | A `sleep` is an acceptable substitute for `dependsOn` | **No** |
| 3 | `condition: succeeded('DeployBackend')` replaces the implicit `succeeded()` | **Yes** |
| 4 | Frontend must deploy before the backend API | **No** |

**In `challenge-30.md`:** lines **274–292**, **733–737**, **726**.

Row 1 is the parallelism rule for the third time across this domain. **Same upstream = parallel.**

Row 4 is the incident from Break & fix Exercise 2: the frontend went out first and produced UI errors
for two minutes because the API it calls did not exist yet. **Dependencies deploy before their
consumers** — the same principle as schema before code in Challenge 29.

</details>

---

# Section E — Drag and drop

---

## Q31

Arrange the hotfix lifecycle in order.

**Items:** Cherry-pick the fix to `main` · Create a branch from the release tag · Swap staging to
production · Tag the hotfix release · Apply the minimal fix · Delete the hotfix branch

<details>
<summary>Show answer</summary>

### Answer

1. Create a branch from the release tag — line **40**
2. Apply the minimal fix — line **44**
3. Tag the hotfix release — line **54**
4. Swap staging to production — line **185**
5. Cherry-pick the fix to `main` — line **68**
6. Delete the hotfix branch — line **81**

```bash
git checkout -b hotfix/payment-over-10k release/2.4.0   # 1
git commit -m "fix: ..."                                 # 2
git tag -a release/2.4.1 -m "Hotfix: ..."                # 3
# pipeline deploys and swaps                             # 4
git checkout main && git cherry-pick <sha>               # 5
git branch -d hotfix/payment-over-10k                    # 6
```

**Steps 5 and 6 happen *after* production is verified.** Cherry-picking first would delay the fix
reaching customers, and deleting the branch before the cherry-pick would lose the reference.

</details>

---

## Q32

Arrange the progressive rollout with its circuit-breaker checkpoints.

**Items:** Clear routing and swap to 100% · Monitor the canary for 120 seconds · Route 10% to staging
· Monitor at 50% · Promote to 50%

<details>
<summary>Show answer</summary>

### Answer

1. Route 10% to staging — line **519**
2. Monitor the canary for 120 seconds — line **526**, `ERROR_THRESHOLD=5`
3. Promote to 50% — line **562**, `if: success()`
4. Monitor at 50% — line **570**
5. Clear routing and swap to 100% — line **584**

**Every promotion is gated by `if: success()`**, and a trip at any checkpoint runs the `if: failure()`
step that clears routing. Exposure only ever increases after the previous level proved healthy.

**This is Challenge 25's ring model implemented with slot traffic routing** — and unlike Traffic
Manager it splits **per request**, so 10% really means 10%.

</details>

---

## Q33

Match each recovery mechanism to its situation.

| Mechanism | Use when |
|---|---|
| Slot swap back |  |
| `traffic-routing clear` |  |
| Container Apps revision weight to stable |  |
| Forward-fix migration |  |
| Manual port + PR |  |

**Options:** A **canary** is bad and no swap has happened yet · A **cherry-pick conflicts** because the code moved · The bad release is a **Container Apps revision** · The **database schema** is wrong · The **new release** is bad and the previous build is in staging

<details>
<summary>Show answer</summary>

| Mechanism | Use when |
|---|---|
| Slot swap back | The **new release** is bad and the previous build is in staging |
| `traffic-routing clear` | A **canary** is bad and no swap has happened yet |
| Container Apps revision weight to stable | The bad release is a **Container Apps revision** |
| Forward-fix migration | The **database schema** is wrong |
| Manual port + PR | A **cherry-pick conflicts** because the code moved |

**In `challenge-30.md`:** lines **404**, **557**, **615**, and Challenge 29 line **376**, line
**751**.

**Every one of these keeps the previous version alive** — the recurring principle across the whole
domain. Except the last two, which acknowledge the cases where you cannot simply route backwards:
schema and source history both only move forward.

</details>

---

## Q34

Arrange the build jobs into execution waves.

```yaml
      - job: BuildCore
      - job: BuildPaymentService       dependsOn: BuildCore
      - job: BuildNotificationService  dependsOn: BuildCore
      - job: BuildFrontend             (no dependsOn)
```

<details>
<summary>Show answer</summary>

### Answer — two waves

| Wave | Jobs |
|---|---|
| 1 | `BuildCore`, **`BuildFrontend`** |
| 2 | `BuildPaymentService`, `BuildNotificationService` |

**In `challenge-30.md`:** lines **259–316**.

**`BuildFrontend` is in wave 1**, which is the part people miss. It declares no `dependsOn`, so it
starts immediately alongside `BuildCore` — it does not consume the shared package.

Total duration is the **critical path**: `BuildCore` plus the slower of the two consumers. If the
frontend takes longer than that whole chain, it becomes the critical path instead.

</details>

---

## Q35

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Hotfix swapped but some customers still see the bug |  |
| UI errors for 2 minutes after deployment |  |
| Cherry-pick fails with conflicts |  |
| Cherry-pick cannot find the commit |  |
| The next release reintroduces the fixed bug |  |

**Options:** Frontend stage missing `dependsOn` on the backend · Leftover traffic routing sending traffic to staging · `main` diverged; the code was refactored elsewhere · Shallow clone — `fetch-depth: 0` not set · The fix was never cherry-picked to `main`

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| Hotfix swapped but some customers still see the bug | **Leftover traffic routing sending traffic to staging** |
| UI errors for 2 minutes after deployment | **Frontend stage missing `dependsOn` on the backend** |
| Cherry-pick fails with conflicts | **`main` diverged; the code was refactored elsewhere** |
| Cherry-pick cannot find the commit | **Shallow clone — `fetch-depth: 0` not set** |
| The next release reintroduces the fixed bug | **The fix was never cherry-picked to `main`** |

**In `challenge-30.md`:** lines **711**, **730**, **749**, **644**, **63–68**.

**The last row is the one that bites weeks later**, long after the incident is closed and nobody
connects the regression to the hotfix that was never merged back.

</details>

---

# Section F — Hot area

---

## Q36

```bash
git checkout -b hotfix/payment-over-10k [BLANK 1]
# ... apply the fix ...
git tag -a [BLANK 2] -m "Hotfix: payment amount overflow fix"
```

- **BLANK 1:** `release/2.4.0` / `main` / `release/2.3.1` / `develop`
- **BLANK 2:** `release/2.4.1` / `release/2.5.0` / `hotfix/2.4.0` / `v2.4.0`

<details>
<summary>Show answer</summary>

### Answer: `release/2.4.0`, `release/2.4.1`

**In `challenge-30.md`:** lines **40** and **54**.

Branch from the **broken** release, not the last-good one — branching from 2.3.1 would undo everything
2.4.0 delivered.

The tag increments the **patch** number: 2.4.0 → 2.4.1. A hotfix is by definition the smallest
possible change, so semver says patch.

</details>

---

## Q37

```yaml
on:
  push:
    branches:
      - '[BLANK 1]'
    tags:
      - 'release/*.*.*'
```

- **BLANK 1:** `hotfix/**` / `hotfix` / `main` / `release/**`

<details>
<summary>Show answer</summary>

### Answer: `hotfix/**`

**In `challenge-30.md`:** line **99**.

`**` matches any depth, so `hotfix/payment-over-10k` and `hotfix/team/urgent-fix` both trigger. A bare
`hotfix` would match only a branch named exactly that.

</details>

---

## Q38

```yaml
      - name: Run critical tests only (skip integration/e2e)
        run: |
          dotnet test ... --filter "[BLANK 1]"

      - name: Security scan (never skip this)
        uses: [BLANK 2]
```

- **BLANK 1:** `Category=Critical|Category=Payment` / `Category=All` / `Priority=1` /
  `FullyQualifiedName~Test`
- **BLANK 2:** `github/codeql-action/analyze@v3` / `actions/setup-dotnet@v4` /
  `aquasecurity/trivy-action@master` / `azure/login@v2`

<details>
<summary>Show answer</summary>

### Answer: `Category=Critical|Category=Payment`, `github/codeql-action/analyze@v3`

**In `challenge-30.md`:** lines **128** and **132**.

`|` is OR — tests with either trait run. **This only works if the tests were categorised in advance**;
an incident is far too late to start adding traits.

Trivy is real and scans **container images** (Challenge 28). This is a .NET source scan, so CodeQL.

</details>

---

## Q39

```yaml
      - name: Circuit breaker - halt and rollback
        if: [BLANK 1]
        run: |
          az webapp traffic-routing [BLANK 2] \
            --name app-contoso-payments \
            --resource-group rg-contoso-prod
```

- **BLANK 1:** `failure()` / `always()` / `success()` / `cancelled()`
- **BLANK 2:** `clear` / `set --distribution staging=0` / `show` / `delete`

<details>
<summary>Show answer</summary>

### Answer: `failure()`, `clear`

**In `challenge-30.md`:** lines **554–559**.

`always()` would clear routing after a **successful** canary too, aborting a good rollout.

`set --distribution staging=0` would technically stop traffic reaching staging, but `clear` removes
the rule entirely — which is what prevents the leftover-rule failure in Break & fix Exercise 1.

</details>

---

## Q40

```yaml
      - name: Promote to 100% (swap)
        if: success()
        run: |
          az webapp traffic-routing [BLANK 1]
          az webapp deployment slot swap --slot [BLANK 2] --target-slot [BLANK 3]
```

- **BLANK 1:** `clear` / `set --distribution staging=100` / `show` / *(omit this line)*
- **BLANK 2:** `staging` / `production` / `canary` / `hotfix`
- **BLANK 3:** `production` / `staging` / `canary` / `default`

<details>
<summary>Show answer</summary>

### Answer: `clear`, `staging`, `production`

**In `challenge-30.md`:** lines **587–594**.

**Omitting BLANK 1 is the trap.** After the swap, a surviving routing rule would send traffic to the
**old** code now sitting in staging — Break & fix Exercise 1 exactly.

</details>

---

## Q41

```yaml
      - uses: actions/checkout@v4
        with:
          fetch-depth: [BLANK 1]
          token: ${{ secrets.[BLANK 2] }}
```

Requirement: cherry-pick commits between branches and push the result to `main`.

- **BLANK 1:** `0` / `1` / `2` / `10`
- **BLANK 2:** `PAT_TOKEN` / `GITHUB_TOKEN` / `AZURE_CREDENTIALS` / `NPM_TOKEN`

<details>
<summary>Show answer</summary>

### Answer: `0`, `PAT_TOKEN`

**In `challenge-30.md`:** lines **644–645**.

`fetch-depth: 0` fetches full history — cherry-pick and `git describe --tags` both need it, and the
default is a shallow depth-1 clone.

`PAT_TOKEN` is needed because pushes made with `GITHUB_TOKEN` **do not trigger further workflows**,
and protected-branch pushes typically need a principal with a bypass. In production a **GitHub App**
installation token is better than a personal PAT — Challenge 40's subject.

</details>

---

# Section G — Case study

## Case study: Contoso incident response

### Background

Production **v2.4.0** silently drops payment transactions over $10,000. The normal release pipeline
takes **2 hours**: full integration tests, security scans, two-reviewer approval and a 48-hour ring
rollout. Customer impact grows by the minute.

- Payment Service: App Service `app-contoso-payments`
- Shared library: NuGet package `Contoso.Payments.Core`
- Frontend: Static Web Apps `swa-contoso-store`
- Broken version `v2.4.0` (tag `release/2.4.0`); last good `v2.3.1` (tag `release/2.3.1`)

### Requirements

**Incident response**

- A verified fix in production in under 15 minutes
- No unreleased work may ship with the fix
- Security scanning must still run
- The fix must not be lost from future releases

**Resilience**

- A canary rollout must halt automatically if errors exceed a threshold
- The frontend must never deploy before the API it calls
- A bad release must be reversible in seconds

---

## Q42

Where should the hotfix branch be created from, and why?

- A. `release/2.4.0` — production plus the fix, with no unreleased work
- B. `main` — so the fix is already merged
- C. `release/2.3.1` — the last known-good version
- D. A new branch from the default branch protection ruleset

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-30.md`:** lines **36–40**.

**Why C is the interesting wrong answer.** `release/2.3.1` *is* the last known-good version, so it
looks safe. But branching there removes every feature 2.4.0 shipped. That is a **rollback decision** —
sometimes right, and a different decision from a hotfix, with different stakeholders.

**Why the others fail**

- **B** — ships unreleased work through a pipeline that has skipped integration tests
- **D** — a branch protection ruleset is a policy, not a source

</details>

---

## Q43

Which **two** keep the hotfix under 15 minutes while preserving essential safety? (Choose two.)

- A. Run only tests tagged `Critical` or `Payment`
- B. Keep CodeQL and drop DAST
- C. Skip the security scan
- D. Deploy directly to the production slot with no staging step
- E. Skip the health check

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-30.md`:** lines **124–135** and the table at **220–228**.

**Both are *reductions*, and that is the pattern.** Critical tests take ~2 minutes instead of ~15.
SAST still runs; only the slower dynamic scan is deferred.

**Why the others fail**

- **C** — the step is named "never skip this"
- **D** — **the subtle one.** Deploying straight to production overwrites the previous build, so
  there is nothing left in staging to swap back to. You would save perhaps thirty seconds and lose
  your rollback path at the moment you most need it
- **E** — the health check is already cut to 30 seconds (line 171). Removing it means swapping a
  build nobody has verified

</details>

---

## Q44

Which configuration halts the canary automatically when errors exceed the threshold?

- A. A monitoring step that counts errors and exits 1 at the threshold, with an `if: failure()`
  rollback step
- B. `continue-on-error: true` on the canary step
- C. An Azure Monitor alert emailing the on-call engineer
- D. A manual approval gate after the canary

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-30.md`:** lines **526–559**.

```bash
              if [ $ERROR_COUNT -ge $ERROR_THRESHOLD ]; then
                echo "CIRCUIT BREAKER TRIPPED"
                exit 1
              fi
```

```yaml
      - name: Circuit breaker - halt and rollback
        if: failure()
```

**Detect, then react.** The `exit 1` is the link between them — remove it and the breaker observes
without acting.

**Why the others fail**

- **B** — swallows the failure, so the rollout promotes to 50% with a broken canary
- **C** — a human in the loop is not *automatic*, and email latency is minutes
- **D** — a manual gate stops the rollout waiting for a person even when everything is healthy

</details>

---

## Q45

Which **two** ensure the frontend never deploys before the API? (Choose two.)

- A. `dependsOn: DeployBackend` on the frontend stage
- B. `condition: succeeded('DeployBackend')`
- C. A 120-second sleep before the frontend deploys
- D. Deploying both in the same stage
- E. `condition: always()` on the frontend stage

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-30.md`:** Break & fix Exercise 2, lines **733–737**.

**Why the others fail**

- **C** — a guess. Too short and it still races; too long and every deployment pays for it
- **D** — one stage removes ordering, separate approvals and any clean failure boundary
- **E** — deploys the frontend even when the backend failed, which is the outage made worse

**The wider principle across this domain:** dependencies deploy before consumers. Shared library
before services (line 274), schema before code (Challenge 29), API before frontend.

</details>

---

## Q46

Which mechanism reverses a bad release in seconds?

- A. Swap the staging slot back into production
- B. Re-run the pipeline on the previous tag
- C. Restore the App Service from backup
- D. Redeploy `release/2.3.1` from source

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-30.md`:** lines **392–421**.

```bash
# The staging slot contains the previous production version after a swap
az webapp deployment slot swap --slot staging --target-slot production
```

**Why the others fail**

- **B** and **D** — both need a full build and deploy, minutes at best
- **C** — backups restore content slowly and are for disaster recovery

**The limit to state:** you get **one** free rollback. Swap again and the broken build returns. After
using it, redeploy a known-good build into staging so a second rollback remains available.

</details>

---

## Q47

The hotfix is deployed and verified. The incident is closed. Three weeks later, the same payment bug
appears in v2.5.0.

What went wrong?

- A. The fix was never cherry-picked to `main`
- B. The hotfix tag was deleted
- C. The staging slot was not cleared
- D. The circuit breaker suppressed the error

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-30.md`:** lines **61–68**.

```bash
git checkout main
git cherry-pick <hotfix-commit-sha>
```

**The mechanics make this failure almost inevitable without automation.** The hotfix branch was cut
from `release/2.4.0`, a tag. It never touched `main`. When v2.5.0 is built from `main`, the buggy code
is still there — and by then nobody connects the regression to an incident three weeks old.

**That is why the cherry-pick is automated** (lines 630–680), triggered by the release tag, opening a
PR when it cannot apply cleanly.

**Why the others fail**

- **B** — a deleted tag does not change `main`
- **C** — a slot issue affects the current deployment, not a future release
- **D** — the breaker halts rollouts; it does not hide code

</details>

---

## Q48

During the next incident, the hotfix pipeline is triggered but the deployment job never starts. The
build and scan job completed successfully.

What is the most likely cause, and what should Contoso change?

- A. The `production-hotfix` environment is waiting for an approver who is unavailable
- B. The circuit breaker tripped
- C. `fetch-depth` was not set
- D. The security scan failed silently

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-30.md`:** line **151**.

```yaml
  deploy-hotfix:
    needs: build-and-scan
    environment: production-hotfix
```

The environment gate holds the job **before its first step** (Challenge 24 Q11). If the configured
approver is asleep, on a flight, or has left the company, the hotfix stops there — with the build
green and nothing obviously wrong.

**What to change:** make the approver a **team**, not a person. Line 225 says "single approver
(on-call lead)" — that must be an on-call **rotation group** so anyone currently on call can approve.
A named individual is a single point of failure in the one process designed for emergencies.

**Why the others fail**

- **B** — the breaker runs during progressive rollout, after deployment starts
- **C** — affects the cherry-pick workflow, not this one
- **D** — a failed scan fails `build-and-scan`, which the question says succeeded

**The wider lesson:** an emergency process must be tested when there is no emergency. An approval
nobody can grant, an on-call rota nobody updated, a test category nobody applied — all of them look
fine until the day they matter.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Branching from `main` for a hotfix** | Q1, Q18, Q25, Q42 | Branch from the release **tag**, not `main` |
| **Branching from the last-good tag** | Q1, Q42 | That is a rollback, not a hotfix |
| **Skipping the security scan** | Q2, Q17, Q26, Q43 | Reduce it (SAST only). Never remove it |
| **Deploying straight to production** | Q26, Q43 | Deploy to staging and swap, or you lose the rollback path |
| **Leftover traffic routing before a swap** | Q3, Q7, Q20, Q40 | `traffic-routing clear` **before** swapping |
| **Forgetting the cherry-pick to `main`** | Q18, Q35, Q47 | The bug returns in the next release |
| **`sleep` instead of `dependsOn`** | Q21, Q45 | A guess is not ordering |
| **`always()` on a rollback or breaker step** | Q6, Q39 | Use `failure()`, or you abort good rollouts |
| **`continue-on-error` on a canary check** | Q44 | Swallows the failure and promotes anyway |
| **Circuit breaker on the first error** | Q5, Q29 | Count to a threshold; transients happen |
| **Expecting more than one free rollback** | Q8, Q29, Q46 | A swap exchanges. The third swap restores the bug |
| **`GITHUB_TOKEN` for pushes to `main`** | Q16, Q41 | It does not trigger workflows; use an App or PAT |
| **A named individual as hotfix approver** | Q48 | Use an on-call team, or the emergency path deadlocks |

---

# The blocks to memorise

Line numbers are in `challenge-30.md`.

```bash
# 1. Hotfix branching  (lines 40-58) - from the TAG, not main
git checkout -b hotfix/payment-over-10k release/2.4.0
git commit -m "fix: ..."
git tag -a release/2.4.1 -m "Hotfix: ..."
git push origin hotfix/payment-over-10k && git push origin release/2.4.1

# 2. Cherry-pick back - the step teams forget  (lines 63-83)
git checkout main && git pull origin main
git cherry-pick <hotfix-commit-sha>
git push origin main
git push origin --delete hotfix/payment-over-10k

# 3. Emergency rollback  (lines 393-408)
set -euo pipefail
az webapp deployment slot swap --slot staging --target-slot production
# staging holds the previous production version - ONE free rollback

# 4. Container Apps circuit breaker  (lines 600-618)
az containerapp update --image ...:v2.4.1 --revision-suffix hotfix-v241
az containerapp ingress traffic set --revision-weight "...--hotfix-v241=10" \
                                    --revision-weight "...--stable=90"
az containerapp ingress traffic set --revision-weight "...--stable=100"   # revert
```

```text
# 5. Normal vs hotfix gates  (lines 220-228)
Gate                 Normal                 Hotfix
Unit tests           full (~15m)            critical only (~2m)
Integration tests    full (~30m)            SKIPPED
Security scan        SAST + DAST            SAST only  <- never removed
Manual approval      2 reviewers            1 on-call lead
Progressive rollout  rings 0-3 (48h)        direct to production
Smoke tests          full regression        health check only
Total                ~2 hours               ~10-15 minutes
```

```yaml
# 6. Expedited pipeline essentials  (lines 96-191)
on:
  push:
    branches: ['hotfix/**']
    tags: ['release/*.*.*']
...
      - run: dotnet test ... --filter "Category=Critical|Category=Payment"
      - uses: github/codeql-action/analyze@v3      # never skip
  deploy-hotfix:
    needs: build-and-scan
    environment: production-hotfix                  # lighter approval

# 7. Circuit breaker  (lines 526-559)
          ERROR_THRESHOLD=5
          ERROR_COUNT=0
          # ... count errors, exit 1 at the threshold
      - name: Circuit breaker - halt and rollback
        if: failure()
        run: az webapp traffic-routing clear ...

# 8. Progressive rollout, clear BEFORE swap  (lines 519-594)
          --distribution staging=10     # canary   -> monitor
          --distribution staging=50     # promote  -> monitor
          traffic-routing clear && slot swap        # 100%

# 9. Dependency ordering  (lines 274-292, 733-737)
      - job: BuildPaymentService
        dependsOn: BuildCore            # both consumers share one upstream
      - job: BuildNotificationService
        dependsOn: BuildCore            # -> they run in PARALLEL
- stage: DeployFrontend
  dependsOn: DeployBackend
  condition: succeeded('DeployBackend')
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 30 is exam-ready. Section 03d is complete |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 2 and 5, then retake this |
| Below 30 | Redo the challenge, writing the normal-vs-hotfix table from memory first |

Record your result in `AZ-400-Learning-Log.md` under Challenge 30.

:::danger The one thing

**A hotfix reduces gates. It never removes them.**

Fewer tests, one approver, no rings — but the security scan runs, the deploy still goes through
staging so a swap-back exists, and the fix is cherry-picked to `main` so it does not come back. Any
option that removes safety entirely is the wrong answer, however fast it looks.

:::
