---
sidebar_position: 3.5
toc_max_heading_level: 2
title: "Challenge 48: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 48 — AZ-400 exam questions

**48 questions** built only from what Challenge 48 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-48.md`**.

:::danger Read this before you start

GitHub has **three separate insights surfaces**, and the exam's favourite mistake is naming the wrong
one.

**Repository Insights** — traffic, contributors, commits, code frequency, dependency graph. *About the
code and who touches it.*
**Actions insights** — run durations, success rates, billed minutes. *About the pipelines.*
**Projects Insights** — burn-down, velocity, cycle time. *About the work items.*

A burn-down chart is not in Repository Insights. Build duration is not in Projects.

And one pattern decides every alerting question: **notify from a separate job, not a final step.**

```yaml
  notify-failure:
    needs: [build]
    if: failure()
```

A step at the end of a job **does not run when the job fails early** — which is the only case you wanted
it for. A separate job with `needs:` and `if: failure()` always runs.

The scenario at line 17 is a manager who cannot answer three basic questions: average build time, which
workflow fails most, and how often teams deploy.

:::

---

# Section A — Multiple choice

---

## Q1

The engineering manager wants deployment frequency for the last 30 days from the terminal. Which
approach provides it?

- A. The GitHub Actions billing page
- B. `gh api repos/contoso/webapp/deployments` filtered by date
- C. Downloading workflow logs and counting manually
- D. The Repository Insights traffic page

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-48.md`:** lines **302–308**.

```bash
gh api repos/contoso/webapp/deployments --jq '[
  .[] | select((.created_at | fromdateiso8601) > (now - 2592000))
] | group_by(.environment) | .[] | {
  environment: .[0].environment,
  count: length,
  frequency: (length / 30 | tostring + " per day")
}'
```

**The Deployments API is the authoritative record** — every deployment with its environment, SHA,
creator and timestamp.

**Why A counts minutes, not deployments.** Billing tells you how much compute Actions consumed; a
single workflow run may deploy to three environments or none.

**Why D is the wrong insights surface.** Traffic is page views and clones — who looked at the
repository, not what was shipped from it.

**Note `2592000` is 30 days in seconds**, which is how a date filter is expressed against
`fromdateiso8601`.

</details>

---

## Q2

A workflow must notify Teams **only when the production deployment job fails**. What is correct?

- A. A notification step at the end of the deployment job
- B. A separate job with `needs: [deploy]` and `if: failure()`
- C. GitHub webhook delivery to Teams
- D. Built-in GitHub email notifications

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-48.md`:** lines **192–195**.

```yaml
  notify-failure:
    runs-on: ubuntu-latest
    needs: [build]
    if: failure()
```

**A is the trap, and it fails in exactly the case it was written for.** Steps in a job stop at the first
failure — so if `npm test` fails at line 189, the notification step at the end of that job never runs.

**A separate job is evaluated independently.** `needs:` establishes the dependency, `if: failure()`
overrides the default "only run if everything succeeded".

**Why D lacks the targeting.** Built-in email notifies on activity you subscribe to across repositories.
It cannot format a message with the branch, commit and author and post it into a team channel (lines
214–217).

</details>

---

## Q3

Where should Contoso configure burn-down charts and cycle time?

- A. Repository Insights
- B. GitHub Projects Insights
- C. Actions workflow insights
- D. Azure DevOps Analytics

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-48.md`:** lines **141–170**.

```text
**Chart 3: Cycle time**
- Type: Line chart
- X-axis: Closed date
- Y-axis: Duration (days from created to closed)
- Filter: Status = Done
```

**Projects Insights charts work on *project items* and their status transitions**, which is what a
burn-down measures.

**Why A is the wrong surface**, and it is the one most people name. Repository Insights (lines 69–78)
covers Pulse, Contributors, Community, Traffic, Commits, Code frequency, Dependency graph, Network and
Forks. **Not one of those is a work-item chart.**

**Why D would be right in the other product** — and Contoso is not leaving GitHub (line 17).

</details>

---

## Q4

Azure DevOps must notify on build failure **and** on the first success after a failure. How?

- A. Two subscriptions — one for failure, one for success where the previous build failed
- B. One subscription for all build completions
- C. A pipeline step checking the previous build's status
- D. A service hook with state transition filters

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **316–322**.

```text
# 2. Create subscription:
#    - Category: Build
#    - Event: A build completes > with status Failed
```

**Notification subscriptions filter on event state**, so a recovery notification is just a second
subscription with a different filter.

**Why B produces noise that gets muted.** Notifying on every completion means a green build emails
everyone several times a day, and within a week the rule is filtered into a folder nobody opens.

**Why C moves platform configuration into pipeline code** — the pipeline would need to query its own
history through the REST API on every run, and the logic breaks whenever the pipeline is renamed or
cloned.

</details>

---

## Q5

Actions insights show failures for a workflow, but developers confirm the builds are green. What is the
cause?

- A. The insights page is cached
- B. A skipped job causes a downstream job to fail, so the workflow conclusion is `failure`
- C. Billing minutes were exceeded
- D. The workflow file is invalid

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-48.md`:** line **455**.

**Skipped is neutral; the *downstream* job is what fails.** A job with a path filter or condition is
skipped, and a job that `needs:` it evaluates `success()` against a skipped dependency — which is not
success.

**The fix at lines 471–478 is the pattern worth learning:**

```yaml
  test:
    if: always()
    needs: [build]
    steps:
      - run: echo "Tests running"
        if: needs.build.result == 'success'
```

**`if: always()` lets the job run, then `needs.build.result` is checked explicitly** — instead of the
implicit `success()` that treats a skip as a failure.

</details>

---

## Q6

The Slack notification job runs successfully but no message appears. What should you check first?

- A. The webhook URL by POSTing a test message directly
- B. The workflow's `permissions` block
- C. The Slack workspace's billing status
- D. Whether the job used `if: failure()`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **487–496**.

```bash
curl -X POST "$SLACK_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{"text": "test message"}'
# If response is "invalid_token" or "channel_not_found", the webhook is broken
```

**The job "runs successfully" because the action posted and got a response.** An expired webhook returns
an error body with a 200-family status in some cases, so the step does not fail.

**That is why testing the webhook directly is the first move** — it isolates GitHub from Slack in one
command.

**The fix is a new webhook and an updated secret** (line 506):

```bash
gh secret set SLACK_WEBHOOK_URL --body "https://hooks.slack.com/services/NEW/WEBHOOK/URL"
```

</details>

---

## Q7

Which API returns page views and unique visitors for a repository?

- A. `repos/{owner}/{repo}/traffic/views`
- B. `repos/{owner}/{repo}/stats/contributors`
- C. `repos/{owner}/{repo}/actions/workflows`
- D. `orgs/{org}/settings/billing/actions`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **35–39**.

```bash
gh api repos/contoso/webapp/traffic/views --jq '{
  totalViews: .count,
  uniqueVisitors: .uniques,
  ...
}'
```

**Note the `count` versus `uniques` distinction**, which runs through every traffic endpoint. Views
counts page loads; uniques counts distinct visitors — and the ratio tells you whether ten people looked
once or one person refreshed ten times.

**Traffic data requires push access** (line 34), which is worth knowing: a read-only collaborator cannot
retrieve it.

</details>

---

## Q8

How do you calculate a workflow's success rate over its last 100 runs?

- A. `gh run list --limit 100 --json conclusion` and count `success` against total
- B. `gh api .../actions/workflows` and read the `state` field
- C. Read the billing API
- D. Count green ticks in the UI

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **100–107**.

```bash
gh run list --workflow deploy.yml --limit 100 --json conclusion \
  --jq '{
    total: length,
    success: [.[] | select(.conclusion == "success")] | length,
    failure: [.[] | select(.conclusion == "failure")] | length,
    cancelled: [.[] | select(.conclusion == "cancelled")] | length,
    successRate: (([.[] | select(.conclusion == "success")] | length) * 100 / length | tostring + "%")
  }'
```

**Why B is the near-miss worth understanding.** `state` on a workflow is `active` or
`disabled_manually` — whether the workflow is *enabled*, not how its runs went.

**And note `cancelled` is counted separately.** A cancelled run is neither a success nor a failure —
folding it into failures would understate the true success rate.

</details>

---

## Q9

How is a run's duration calculated in these examples?

- A. `updatedAt` minus `startedAt`, both parsed with `fromdateiso8601`
- B. The `duration` field returned by the API
- C. `createdAt` minus `startedAt`
- D. The billing API's minutes field

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **91–97**.

```bash
    duration: ((.updatedAt | fromdateiso8601) - (.startedAt | fromdateiso8601) | tostring + "s")
```

**There is no `duration` field**, so you compute it — which is why `fromdateiso8601` appears in almost
every query in this challenge.

**And `startedAt` is not `createdAt`.** A run created while all runners are busy sits queued; `createdAt`
would include that wait, and `startedAt` measures the execution itself. **Which one you want depends on
the question** — "how long does CI take?" is `startedAt`; "how long until I get feedback?" is
`createdAt`.

</details>

---

## Q10

Why does the average-duration query filter to `conclusion == "success"`?

- A. Failed runs terminate early and would skew the average downwards
- B. Failed runs have no timestamps
- C. The API rejects mixed conclusions
- D. Cancelled runs are excluded automatically

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **110–111**.

```bash
gh run list --workflow deploy.yml --limit 50 --status completed --json startedAt,updatedAt,conclusion \
  --jq '[.[] | select(.conclusion == "success") | ...] | (add / length | floor | ...)'
```

**A run that fails at minute two is not a two-minute build** — it is a build that did not happen.
Averaging it in makes the pipeline look faster than it is, and the average improves whenever the failure
rate rises.

**Which is the general lesson for pipeline metrics:** measure duration over **successful** runs, and
measure failures as a **separate** metric. Mixing them produces a number that moves for the wrong
reasons.

</details>

---

## Q11

Which API returns Actions minutes consumed?

- A. `orgs/{org}/settings/billing/actions`
- B. `repos/{owner}/{repo}/actions/runs`
- C. `repos/{owner}/{repo}/traffic/clones`
- D. `orgs/{org}/actions/usage`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **114–118**.

```bash
gh api orgs/contoso/settings/billing/actions --jq '{
  totalMinutesUsed: .total_minutes_used,
  includedMinutes: .included_minutes,
  paidMinutesUsed: .total_paid_minutes_used
}'
```

**Three numbers, and the third is the one that matters commercially.** `included_minutes` is the plan's
allowance; `total_paid_minutes_used` is what is being billed beyond it.

**This is the metric behind Challenge 35's optimisation work** — caching and sharding reduce
`total_paid_minutes_used`, and this endpoint is how you prove it.

</details>

---

## Q12

Which trigger fires the deployment tracker workflow?

- A. `deployment_status`
- B. `push`
- C. `workflow_run`
- D. `release`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **266–272**.

```yaml
on:
  deployment_status:

jobs:
  track:
    if: github.event.deployment_status.state == 'success'
```

**`deployment_status` fires whenever a deployment's state changes** — `pending`, `in_progress`,
`success`, `failure` — so the `if:` at line 272 narrows it to successful ones.

**Why this rather than `push`:** a push does not always deploy, and a deployment does not always follow a
push. The event models the thing being measured.

</details>

---

## Q13

What does `if: failure()` mean on a job?

- A. Run this job only if a job it depends on failed
- B. Run this job only if the previous step failed
- C. Always run this job
- D. Mark this job as failed

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **194–195**.

**Job-level conditions are evaluated against the jobs in `needs:`.** Without `needs: [build]`, `failure()`
would have no dependency to inspect — which is why both lines appear together.

**And the default when you write no `if:` is `success()`** — run only if every needed job succeeded. That
default is exactly what makes a notification job silent in the case you built it for.

</details>

---

## Q14

What is the Azure Pipelines equivalent of `if: failure()` for the retention audit's warning?

- A. `##vso[task.logissue type=warning]`
- B. `exit 1`
- C. `condition: failed()`
- D. `continueOnError: true`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** line **373**.

```powershell
        if ($builds.count -gt 0) {
          Write-Host "##vso[task.logissue type=warning]$($builds.count) builds have artifacts expiring within 5 days"
        }
```

**The question is about surfacing a finding without failing the run**, and `logissue type=warning` is
that mechanism — the same one used in Challenge 41's PAT audit.

**Why B would fail the scheduled audit**, which is wrong: nothing is broken, artifacts are simply nearing
expiry. **A monitoring job that fails on findings gets disabled.**

</details>

---

## Q15

What does `$(System.AccessToken)` provide in the retention audit?

- A. The pipeline's own OAuth token for authenticating to the Azure DevOps REST API
- B. A PAT stored in a variable group
- C. The service connection's credentials
- D. The user's login token

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **367–369**.

```powershell
        $headers = @{
          Authorization = "Basic $([Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$(System.AccessToken)")))"
        }
```

**It is the build service identity's token**, issued for the run and expiring with it — the Azure
Pipelines equivalent of `GITHUB_TOKEN`.

**Which makes it the correct choice here**: no stored PAT, nothing to rotate, and its permissions are
governed by the project's build service account rather than a person's.

</details>

---

## Q16

Which weekly-digest metric measures pull requests merged in the last seven days?

- A. `gh pr list --state merged --json mergedAt` filtered on `now - 604800`
- B. `gh pr list --state open --json number | length`
- C. `gh run list --workflow ci.yml`
- D. `gh api .../traffic/views`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **406–407**.

```bash
          PRS_MERGED=$(gh pr list --state merged --limit 100 --json mergedAt \
            --jq '[.[] | select((.mergedAt | fromdateiso8601) > (now - 604800))] | length')
```

**`604800` is seven days in seconds**, the weekly counterpart to the 30-day `2592000` at line 303.

**Note the difference in kind between the two PR metrics.** `PRS_MERGED` is a **flow** measure — work
completed in a period. `PRS_OPEN` (line 408) is a **stock** measure — a snapshot with no date filter,
because "open" is a current state.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** are available in Repository Insights? (Choose three.)

- A. Traffic — views, clones and referrers
- B. Contributors — commit frequency per person
- C. Code frequency — additions and deletions per week
- D. Sprint burn-down
- E. Workflow success rate
- F. Cycle time

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-48.md`:** lines **69–78**.

```text
- Pulse: recent activity summary
- Contributors: commit frequency per contributor
- Community: community health files
- Traffic: page views, clones, referrers
- Commits: commit frequency over time
- Code frequency: additions and deletions per week
- Dependency graph / Network / Forks
```

**Read the list and notice what is absent.** No work items, no workflow runs. Repository Insights is
about **the code and who touches it**.

**D and F are Projects Insights** (lines 147–170); **E is computed from the Actions API** (lines
100–107).

</details>

---

## Q18

Which **three** charts does the challenge configure in Projects Insights? (Choose three.)

- A. Burn-down — items not Done over time
- B. Items by assignee, grouped by priority
- C. Cycle time — days from created to closed
- D. Workflow duration trend
- E. Repository traffic
- F. Billed Actions minutes

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-48.md`:** lines **147–170**.

**Each chart answers a different management question:** how much work remains, who is loaded with what,
and how long work takes to get through.

**Note that all three filter on `Status`** — Done, In Progress, or not Done. **Projects charts are built
on item fields**, so the quality of the charts depends entirely on whether people keep status current.

</details>

---

## Q19

Which **two** ensure a notification fires when a job fails? (Choose two.)

- A. `needs: [build]` on the notification job
- B. `if: failure()` on the notification job
- C. A notification step at the end of the build job
- D. `continue-on-error: true` on the build job
- E. `if: always()` with no `needs:`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-48.md`:** lines **192–195**.

**`needs:` gives the condition something to evaluate; `if: failure()` overrides the default `success()`.**
Both are required.

**Why D would break the alerting entirely** — `continue-on-error` makes the job report success, so
`failure()` never fires and the pipeline reports green over a broken build.

**Why E fires on every run**, including successful ones, which is how a channel gets muted.

</details>

---

## Q20

Which **two** does the Slack failure notification include for triage? (Choose two.)

- A. Repository, branch, commit and actor
- B. A button linking directly to the run
- C. The full build log
- D. The test coverage percentage
- E. The previous successful commit

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-48.md`:** lines **214–227**.

```json
                    {"type": "mrkdwn", "text": "*Branch:*\n${{ github.ref_name }}"},
                    ...
                      "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
```

**Enough context to decide whether it is yours, and one click to the detail.** That is the design of a
good alert — someone glancing at their phone can tell in two seconds whether to act.

**Why C would be actively harmful.** Pasting a full log into a channel is unreadable, and it can leak
values a masked log would have hidden in the UI but that a copy might not (Challenge 43).

</details>

---

## Q21

Which **two** are true of `deployment_status`? (Choose two.)

- A. It fires on every state change of a deployment
- B. The payload carries the environment and SHA
- C. It fires only on successful deployments
- D. It replaces the need for the Deployments API
- E. It requires `workflow_dispatch`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-48.md`:** lines **266–278**.

```yaml
    if: github.event.deployment_status.state == 'success'
```

```bash
          echo "Deployment to ${{ github.event.deployment.environment }} succeeded"
          echo "SHA: ${{ github.event.deployment.sha }}"
```

**The `if:` at line 272 exists precisely because C is false** — without it the job would also run for
`pending`, `in_progress` and `failure`.

**Why D confuses a stream with a store.** The event tells you about **one** deployment as it happens; the
Deployments API (line 292) returns **history**, which is what deployment frequency needs.

</details>

---

## Q22

Which **two** distinguish Azure DevOps notification subscriptions from a pipeline task that sends email?
(Choose two.)

- A. Subscriptions are configured centrally and apply across pipelines
- B. Subscriptions can filter on event state, such as build failed
- C. Subscriptions run inside the pipeline
- D. Subscriptions require a PAT in a variable group
- E. Subscriptions can only email the build requester

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-48.md`:** lines **316–348**.

```json
        "clauses": [{
          "fieldName": "Definition name",
          "operator": "=",
          "value": "Production-Deploy"
        }]
```

**Central configuration is the practical difference.** A notification task must be added to every
pipeline and maintained in each — and it disappears the moment somebody clones a pipeline and edits it.

**And the filter is the mechanism for the recovery notification in Q4** — one subscription per state you
care about.

</details>

---

## Q23

Which **two** metrics does the weekly digest report as **flow** rather than **stock**? (Choose two.)

- A. PRs merged in the last seven days
- B. Issues closed in the last seven days
- C. PRs currently open
- D. Issues currently open
- E. Deploy pipeline success rate

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-48.md`:** lines **406–413**.

```bash
          PRS_MERGED=$(... select((.mergedAt | fromdateiso8601) > (now - 604800)) ...)
          PRS_OPEN=$(gh pr list --state open --json number --jq 'length')
```

**Flow measures throughput over a period; stock measures a level at a moment.** The date filter is what
distinguishes them, and only A and B have one.

**Why both belong in the digest.** Flow alone hides a growing backlog — a team can merge 20 PRs a week
while the open count climbs. Stock alone hides whether anything is moving.

**Why E is a ratio**, not a count, and it describes reliability rather than volume.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso's engineering manager needs visibility into build time, workflow reliability and
deployment frequency without leaving GitHub, plus proactive alerts when critical workflows fail.

---

## Q24

**Proposed solution:** Query the Actions API for run durations and success rates over the last 50 to 100
runs, filtering duration to successful runs. Query the Deployments API grouped by environment for a
30-day frequency. Add a `notify-failure` job with `needs: [build]` and `if: failure()` posting to Slack
and Teams with repository, branch, commit, actor and a link to the run. Build Projects Insights charts
for burn-down, assignee load and cycle time. Add a scheduled weekly digest combining workflow success
rates with PR and issue flow and stock counts.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-48.md`:** lines **100–111**, **302–308**, **192–234**, **147–170**, **380–446**.

| Question the manager could not answer | Mechanism |
|---|---|
| Average CI build time | Duration over **successful** runs |
| Which workflows fail most | Success rate per workflow |
| How often teams deploy | Deployments API, grouped by environment |
| Broken builds discovered hours later | `if: failure()` job → Slack and Teams |
| Sprint progress | Projects Insights charts |
| Weekly trend | Scheduled digest |

**Note that every metric here is pulled, not pushed.** Nothing depends on developers remembering to
report anything — which is why the numbers stay honest.

</details>

---

## Q25

**Proposed solution:** Enable email notifications on the repository so everyone is told about all
activity. Read the Repository Insights traffic page for deployment frequency. Add a notification step at
the end of the build job. Ask team leads to report cycle time in the weekly meeting.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures, one per requirement.**

**"All activity" email is the definition of alert fatigue.** Within a week everyone has a filter rule,
and the one failure that mattered goes to the same folder as 400 routine notifications.

**Traffic is page views and clones** (line 49), not deployments. It is the wrong insights surface for the
question — Q1's error.

**A notification step at the end of the build job does not run when the build fails** (Q2). It is the
single most common mistake in this challenge, and it produces alerting that works perfectly in testing
and never fires in production.

**And self-reported cycle time is not a metric.** Projects Insights computes it from actual status
transitions (lines 161–165); a number recalled in a meeting measures memory.

</details>

---

## Q26

**Proposed solution:** Query the Actions API for durations and success rates. Query the Deployments API
for frequency. Add a `notify-failure` job with `needs: [build]` and `if: failure()` posting to Slack.
Build Projects Insights charts. Add a scheduled weekly digest. Calculate average build duration across
**all** completed runs, so the number reflects real-world experience including failures.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare with Q24, which is otherwise identical.

**Averaging failed runs into build duration produces a number that moves for the wrong reason.** A build
that fails at minute two is not a two-minute build; it is a build that did not complete. Include enough
of them and the average falls.

**Which creates a genuinely perverse signal: the pipeline appears to get faster as it gets less
reliable.** A week where the failure rate doubles shows an *improving* build-time trend, and the manager
concludes the optimisation work is paying off.

**Line 111 filters deliberately**, and the reasoning generalises to every pipeline metric:

```bash
  --jq '[.[] | select(.conclusion == "success") | ...] | (add / length | floor | ...)'
```

**Duration over successful runs. Failures as a separate metric.** Two numbers that each mean one thing,
rather than one number that means neither.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — insights surfaces

| # | Statement | Answer |
|---|---|---|
| 1 | Repository Insights includes traffic and contributors |  |
| 2 | Repository Insights includes burn-down charts |  |
| 3 | Projects Insights charts are built from project item fields |  |
| 4 | Workflow success rate is computed from the Actions API |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Repository Insights includes traffic and contributors | **Yes** |
| 2 | Repository Insights includes burn-down charts | **No** |
| 3 | Projects Insights charts are built from project item fields | **Yes** |
| 4 | Workflow success rate is computed from the Actions API | **Yes** |

**In `challenge-48.md`:** lines **69–78**, **147–170**, **100–107**.

Row 2 is the most-tested confusion in this challenge. **Three surfaces, three subjects: the code, the
pipelines, the work.**

</details>

---

## Q28 — failure notification

| # | Statement | Answer |
|---|---|---|
| 1 | A step at the end of a job runs when the job fails early |  |
| 2 | `if: failure()` requires `needs:` to have something to evaluate |  |
| 3 | The default job condition is `success()` |  |
| 4 | `continue-on-error: true` would keep failure notifications working |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A step at the end of a job runs when the job fails early | **No** |
| 2 | `if: failure()` requires `needs:` to have something to evaluate | **Yes** |
| 3 | The default job condition is `success()` | **Yes** |
| 4 | `continue-on-error: true` would keep failure notifications working | **No** |

**In `challenge-48.md`:** lines **192–195**.

Row 1 is why the separate job exists. Row 4 is the change that silently disables the whole mechanism —
the job reports success, so `failure()` never fires.

</details>

---

## Q29 — metrics quality

| # | Statement | Answer |
|---|---|---|
| 1 | Duration should be averaged over successful runs only |  |
| 2 | Cancelled runs should be counted as failures |  |
| 3 | `startedAt` excludes time spent queued |  |
| 4 | PRs merged this week and PRs open measure the same thing |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Duration should be averaged over successful runs only | **Yes** |
| 2 | Cancelled runs should be counted as failures | **No** |
| 3 | `startedAt` excludes time spent queued | **Yes** |
| 4 | PRs merged this week and PRs open measure the same thing | **No** |

**In `challenge-48.md`:** lines **110–111**, **105**, **96**, **406–408**.

Row 3 is worth holding onto: `createdAt` to `updatedAt` measures **feedback latency**; `startedAt` to
`updatedAt` measures **execution time**. Different questions, different fixes.

Row 4 is flow versus stock (Q23).

</details>

---

## Q30 — alert delivery

| # | Statement | Answer |
|---|---|---|
| 1 | An expired Slack webhook can leave the job reporting success |  |
| 2 | Testing the webhook with `curl` isolates GitHub from Slack |  |
| 3 | Azure DevOps subscriptions can filter by pipeline definition name |  |
| 4 | Azure DevOps notifications require a pipeline task |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | An expired Slack webhook can leave the job reporting success | **Yes** |
| 2 | Testing the webhook with `curl` isolates GitHub from Slack | **Yes** |
| 3 | Azure DevOps subscriptions can filter by pipeline definition name | **Yes** |
| 4 | Azure DevOps notifications require a pipeline task | **No** |

**In `challenge-48.md`:** lines **485–496**, **334–339**, **316–322**.

Row 1 is the silent-failure theme of Domain 5 in miniature. **An alerting path that is never tested is
not an alerting path** — it is an assumption.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each question to the surface that answers it.

| Question | Surface |
|---|---|
| Who commits most often? |  |
| How many people cloned the repo? |  |
| Which workflow fails most? |  |
| How much are we spending on Actions? |  |
| How much sprint work remains? |  |
| How long does an item take to finish? |  |
| How often do we deploy to production? |  |

**Options:** Actions API — success rate per workflow · Billing API — `settings/billing/actions` · Deployments API, grouped by environment · Projects Insights — burn-down · Projects Insights — cycle time · Repository Insights — Contributors · Repository Insights — Traffic

<details>
<summary>Show answer</summary>

| Question | Surface |
|---|---|
| Who commits most often? | **Repository Insights — Contributors** |
| How many people cloned the repo? | **Repository Insights — Traffic** |
| Which workflow fails most? | **Actions API — success rate per workflow** |
| How much are we spending on Actions? | **Billing API — `settings/billing/actions`** |
| How much sprint work remains? | **Projects Insights — burn-down** |
| How long does an item take to finish? | **Projects Insights — cycle time** |
| How often do we deploy to production? | **Deployments API, grouped by environment** |

**In `challenge-48.md`:** lines **55–59**, **42–46**, **100–107**, **114–118**, **147–152**, **161–165**,
**302–308**.

**Code, pipelines, work, money, shipments** — five subjects, and each has exactly one home.

</details>

---

## Q32

Match each `jq` fragment to what it does.

| Fragment | Does |
|---|---|
| `select(.conclusion == "success") \| length` |  |
| `(.updatedAt \| fromdateiso8601) - (.startedAt \| fromdateiso8601)` |  |
| `select((.created_at \| fromdateiso8601) > (now - 2592000))` |  |
| `group_by(.environment)` |  |
| `add / length \| floor` |  |

**Options:** Averages and truncates · Computes duration in seconds · Counts successful runs · Filters to the last 30 days · Splits deployments per environment

<details>
<summary>Show answer</summary>

| Fragment | Does |
|---|---|
| `select(.conclusion == "success") \| length` | **Counts successful runs** |
| `(.updatedAt \| fromdateiso8601) - (.startedAt \| fromdateiso8601)` | **Computes duration in seconds** |
| `select((.created_at \| fromdateiso8601) > (now - 2592000))` | **Filters to the last 30 days** |
| `group_by(.environment)` | **Splits deployments per environment** |
| `add / length \| floor` | **Averages and truncates** |

**In `challenge-48.md`:** lines **103**, **96**, **303**, **304**, **111**.

**`fromdateiso8601` appears in almost every query** because the API returns ISO timestamps and arithmetic
needs epoch seconds. **604800 = one week, 2592000 = 30 days** — the two constants worth recognising on
sight.

</details>

---

## Q33

Arrange the steps to add failure alerting to an existing CI workflow.

**Items:** Add `if: failure()` to the notification job · Create the Slack incoming webhook · Add a
`notify-failure` job with `needs: [build]` · Store the webhook URL as a repository secret · Test by
forcing a failure

<details>
<summary>Show answer</summary>

### Answer

1. Create the Slack incoming webhook — line **503**
2. Store the webhook URL as a repository secret — line **506**
3. Add a `notify-failure` job with `needs: [build]` — line **194**
4. Add `if: failure()` to the notification job — line **195**
5. Test by forcing a failure — Break scenario 2's diagnosis at line **493**

**Step 5 is not optional, and it is the step teams skip.** An alerting path nobody has fired is an
assumption — and its failure mode is silence, which is indistinguishable from "nothing has gone wrong"
(Q30).

**Steps 3 and 4 are listed separately on purpose.** `needs:` without `if:` notifies on every run;
`if: failure()` without `needs:` has no dependency to evaluate.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Notification never fires on failure |  |
| Notification fires on every run |  |
| Job succeeds, no Slack message |  |
| Insights show failures for green builds |  |
| Build duration trending down as reliability drops |  |
| Traffic API returns 403 |  |

**Options:** Caller lacks push access · Expired or deleted webhook · Failed runs averaged into duration · `needs:` present, `if: failure()` missing · Skipped job breaks a downstream `success()` · Step at the end of the job, not a separate job

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| Notification never fires on failure | **Step at the end of the job, not a separate job** |
| Notification fires on every run | **`needs:` present, `if: failure()` missing** |
| Job succeeds, no Slack message | **Expired or deleted webhook** |
| Insights show failures for green builds | **Skipped job breaks a downstream `success()`** |
| Build duration trending down as reliability drops | **Failed runs averaged into duration** |
| Traffic API returns 403 | **Caller lacks push access** |

**In `challenge-48.md`:** lines **192–195**, **195**, **487**, **455**, **111**, **34**.

**The fifth row is the subtle one**, because the metric looks like it is improving. **A number that moves
in the good direction for a bad reason is worse than no number.**

</details>

---

## Q35

Match each alerting mechanism to its platform and scope.

| Mechanism | Platform and scope |
|---|---|
| `if: failure()` job posting to a webhook |  |
| Notification subscription filtered on Definition name |  |
| `##vso[task.logissue type=warning]` |  |
| Scheduled digest workflow |  |
| `deployment_status` triggered workflow |  |

**Options:** Azure DevOps — central, per project · Azure Pipelines — surfaces a finding without failing · GitHub — reacts to deployment events · GitHub Actions — per workflow · GitHub Actions — periodic summary, not an alert

<details>
<summary>Show answer</summary>

| Mechanism | Platform and scope |
|---|---|
| `if: failure()` job posting to a webhook | **GitHub Actions — per workflow** |
| Notification subscription filtered on Definition name | **Azure DevOps — central, per project** |
| `##vso[task.logissue type=warning]` | **Azure Pipelines — surfaces a finding without failing** |
| Scheduled digest workflow | **GitHub Actions — periodic summary, not an alert** |
| `deployment_status` triggered workflow | **GitHub — reacts to deployment events** |

**In `challenge-48.md`:** lines **192–195**, **316–322**, **373**, **380–384**, **266–272**.

**The distinction between the first and the fourth row matters operationally.** An alert interrupts and
must be actionable; a digest informs and must be skimmable. Sending the digest as an alert trains people
to ignore alerts.

</details>

---

# Section F — Hot area

---

## Q36

```yaml
  notify-failure:
    runs-on: ubuntu-latest
    [BLANK 1]: [build]
    [BLANK 2]: [BLANK 3]
```

Requirement: notify only when the `build` job fails.

- **BLANK 1:** `needs` / `depends-on` / `after` / `requires`
- **BLANK 2:** `if` / `when` / `condition` / `only`
- **BLANK 3:** `failure()` / `always()` / `success()` / `cancelled()`

<details>
<summary>Show answer</summary>

### Answer: `needs`, `if`, `failure()`

**In `challenge-48.md`:** lines **192–195**.

**`condition:` is Azure Pipelines; `if:` is GitHub Actions.** Mixing the two platforms' keywords is the
most reliable way the exam separates people who have written both.

**And `always()` would notify on every run** — including the green ones, which is how a channel gets
muted within a week.

</details>

---

## Q37

```bash
gh run list --workflow deploy.yml --limit 100 --json [BLANK 1] \
  --jq '{
    total: length,
    success: [.[] | select(.[BLANK 1] == "[BLANK 2]")] | length,
    successRate: (([.[] | select(.[BLANK 1] == "[BLANK 2]")] | length) * 100 / length | tostring + "%")
  }'
```

- **BLANK 1:** `conclusion` / `status` / `state` / `result`
- **BLANK 2:** `success` / `completed` / `passed` / `green`

<details>
<summary>Show answer</summary>

### Answer: `conclusion`, `success`

**In `challenge-48.md`:** lines **100–106**.

**`status` and `conclusion` are different fields, and this is the distinction to memorise.** `status` is
the lifecycle — `queued`, `in_progress`, `completed`. `conclusion` is the outcome — `success`,
`failure`, `cancelled`, `skipped`.

**A run can be `completed` and have failed.** Filtering on `status == "completed"` counts every finished
run regardless of result, which is why line 110 uses `--status completed` to *select the population* and
then filters on `conclusion` to *measure the outcome*.

</details>

---

## Q38

```bash
gh api repos/contoso/webapp/[BLANK 1] --jq '[
  .[] | select((.created_at | [BLANK 2]) > (now - [BLANK 3]))
] | group_by(.environment) | .[] | {environment: .[0].environment, count: length}'
```

Requirement: deployment frequency per environment over the last 30 days.

- **BLANK 1:** `deployments` / `actions/runs` / `releases` / `traffic/views`
- **BLANK 2:** `fromdateiso8601` / `todate` / `tonumber` / `strptime`
- **BLANK 3:** `2592000` / `604800` / `86400` / `31536000`

<details>
<summary>Show answer</summary>

### Answer: `deployments`, `fromdateiso8601`, `2592000`

**In `challenge-48.md`:** lines **302–308**.

**`2592000` is 30 × 86400.** The three constants worth recognising: `86400` a day, `604800` a week,
`2592000` thirty days.

**Why `releases` is the interesting distractor.** A release is a tagged artifact; a deployment is an
artifact **reaching an environment**. Teams that release weekly may deploy daily, and the question asks
about deployment.

</details>

---

## Q39

```yaml
on:
  [BLANK 1]:

jobs:
  track:
    if: github.event.[BLANK 1].[BLANK 2] == '[BLANK 3]'
```

- **BLANK 1:** `deployment_status` / `deployment` / `release` / `push`
- **BLANK 2:** `state` / `status` / `conclusion` / `result`
- **BLANK 3:** `success` / `completed` / `active` / `deployed`

<details>
<summary>Show answer</summary>

### Answer: `deployment_status`, `state`, `success`

**In `challenge-48.md`:** lines **266–272**.

**Three different words for "it went well" across three contexts**, and the exam relies on the confusion:
Actions runs use `conclusion == "success"`, deployment statuses use `state == "success"`, and Azure
Pipelines uses `Succeeded`.

**The `if:` is required** because the event also fires for `pending`, `in_progress` and `failure`.

</details>

---

## Q40

```powershell
        if ($builds.count -gt 0) {
          Write-Host "##vso[task.[BLANK 1] type=[BLANK 2]]$($builds.count) builds have artifacts expiring within 5 days"
        }
```

Requirement: surface the finding in the run summary without failing the scheduled audit.

- **BLANK 1:** `logissue` / `setvariable` / `complete` / `uploadfile`
- **BLANK 2:** `warning` / `error` / `info` / `debug`

<details>
<summary>Show answer</summary>

### Answer: `logissue`, `warning`

**In `challenge-48.md`:** line **373**.

**`type=error` would mark the run failed**, and a weekly audit that fails whenever it finds something is
an audit that gets disabled in month two.

**`task.complete` is the distractor for people who half-remember the logging commands** — it sets the
task's *result*, which is the opposite of what a reporting job wants.

</details>

---

## Q41

```bash
          PRS_MERGED=$(gh pr list --state [BLANK 1] --limit 100 --json [BLANK 2] \
            --jq '[.[] | select((.[BLANK 2] | fromdateiso8601) > (now - 604800))] | length')
```

- **BLANK 1:** `merged` / `closed` / `open` / `all`
- **BLANK 2:** `mergedAt` / `closedAt` / `updatedAt` / `createdAt`

<details>
<summary>Show answer</summary>

### Answer: `merged`, `mergedAt`

**In `challenge-48.md`:** lines **406–407**.

**`closed` includes PRs closed *without* merging** — abandoned work, duplicates, superseded proposals.
Counting those as delivery inflates the number with things that shipped nothing.

**And `mergedAt` is the matching timestamp.** Using `updatedAt` would count a PR merged three months ago
that someone commented on yesterday.

</details>

---

# Section G — Case study

## Case study: Contoso engineering visibility

### Background

Contoso Ltd's engineering manager wants visibility into **team velocity, workflow efficiency and
deployment patterns without leaving GitHub**. Today nobody knows the **average CI build time**, **which
workflows fail most often**, or **how frequently teams deploy**. The manager also wants **proactive
alerts** when critical workflows fail, so the team does not discover broken builds hours later.

### Requirements

**Metrics**

- Average build duration must reflect how long a successful build actually takes
- Workflow reliability must be measurable per workflow
- Deployment frequency must be reported per environment over a rolling 30 days
- Sprint progress and cycle time must be visible to the manager

**Alerting**

- A failed critical workflow must notify a team channel immediately
- The notification must carry enough context to triage without opening the run
- Azure DevOps pipelines must notify on failure and on recovery
- Routine noise must not train the team to ignore alerts

**Operations**

- Metrics must be collected without asking developers to report anything
- The alerting path itself must be verifiable

---

## Q42

How should average build duration be measured?

- A. `updatedAt` minus `startedAt`, averaged over runs where `conclusion == "success"`
- B. Averaged over all completed runs
- C. Read from the `duration` field on each run
- D. Derived from billed Actions minutes

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **110–111**.

**"How long a successful build actually takes" is the requirement**, and it rules out B directly.

**Why B is the answer that produces a number moving the wrong way** (Q26): more failures, lower average,
apparent improvement.

**Why C does not exist** — there is no `duration` field; it is computed (Q9). **Why D measures spend**:
billed minutes aggregate across concurrent jobs and matrix legs, so they answer a cost question rather
than a latency one.

</details>

---

## Q43

How should deployment frequency be reported?

- A. The Deployments API filtered to 30 days and grouped by environment
- B. Counting workflow runs on the deploy workflow
- C. Counting releases
- D. Reading the Actions billing page

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **302–308**.

**Only the Deployments API records the environment**, which the requirement asks for explicitly.

**Why B is close and wrong in a way worth understanding.** A deploy workflow run may deploy to staging
only, may deploy to three environments, or may fail before deploying anything. **Counting runs counts
attempts; counting deployments counts arrivals.**

**Why C conflates release with deployment** (Q38).

</details>

---

## Q44

How should the critical workflow notify on failure?

- A. A `notify-failure` job with `needs: [build]` and `if: failure()`, posting repository, branch,
  commit, actor and a run link
- B. A notification step appended to the build job
- C. Repository email notifications for all activity
- D. A scheduled job that checks for failures each morning

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **192–234**.

**The separate job is the mechanism; the payload is the requirement about context** (lines 214–227).

**Why B fails in the exact case it exists for** (Q2) — the job stops at the failing step.

**Why C creates the noise the last requirement forbids**, and **why D reintroduces the "hours later"
problem** the scenario opens with.

</details>

---

## Q45

Which **two** does Azure DevOps need for failure and recovery notifications? (Choose two.)

- A. A subscription for "a build completes with status Failed"
- B. A second subscription for success where the previous build failed
- C. A pipeline task querying build history
- D. A single subscription for all completions
- E. A service hook to Slack

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-48.md`:** lines **316–322** and **334–339**.

**Two states, two subscriptions**, each filtered by definition name so only the production pipeline is
in scope.

**Why D violates the noise requirement** and **why C moves platform configuration into pipeline code**
(Q4), where it is duplicated in every pipeline and lost on the next clone.

</details>

---

## Q46

How should sprint progress and cycle time be made visible?

- A. Projects Insights charts — burn-down filtered to Status != Done, and cycle time from created to
  closed
- B. Repository Insights Pulse
- C. A weekly manual report from team leads
- D. Counting open issues in the digest

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **147–165**.

**Projects Insights computes both from actual item status transitions**, which satisfies the "without
asking developers to report anything" requirement.

**Why B is the wrong surface** (Q3, Q17). **Why C is the requirement's explicit exclusion.**

**Why D is genuinely useful and insufficient.** Open issue count is in the digest (line 413) and it is a
**stock** measure — it tells you the level, not the rate or the duration (Q23).

</details>

---

## Q47

Five months in, the manager notices the Slack channel has had **no** failure notifications for six
weeks, while the Actions insights page shows the CI workflow failing roughly twice a week. The
`notify-failure` job appears in every run's job list and shows a green tick.

What happened, and what should be checked?

- A. The Slack webhook is dead — the job posts, receives an error response the action does not treat as
  fatal, and reports success. Test the webhook directly and rotate the secret
- B. `if: failure()` was removed
- C. The workflow no longer has a `build` job
- D. Actions insights is reporting incorrectly

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **483–496** and **503–506**.

```bash
curl -X POST "$SLACK_WEBHOOK_URL" -H "Content-Type: application/json" -d '{"text": "test message"}'
# If response is "invalid_token" or "channel_not_found", the webhook is broken
```

**"Green tick, no message" is Break scenario 2 exactly**, and the detail that confirms it is that the job
**ran** — so the condition is intact and the dependency resolved.

**Why B and C would show a different symptom.** With `if: failure()` removed the job would run on *every*
run, including green ones; without a `build` job the workflow would fail to parse. Neither produces a job
that runs on failures and posts nothing.

**The durable lesson, and it is the sharpest one in this challenge: an alerting path degrades silently.**
Six weeks of no alerts read identically to six weeks of no failures. **Only a periodic test distinguishes
them** — which is why the digest at line 380 is valuable beyond its content: a Monday message that stops
arriving is itself a signal.

</details>

---

## Q48

At the quarterly review the manager can now answer all three original questions in under a minute.

Explain what each number is, and what the design actually changed.

- A. Build time from `startedAt` to `updatedAt` over successful runs; reliability as success rate per
  workflow from `conclusion`; deployment frequency from the Deployments API grouped by environment — and
  every one is pulled from the platform's own record rather than reported by anyone
- B. All three come from the Repository Insights tab
- C. All three come from the weekly digest, which the team fills in
- D. All three come from the Actions billing page

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-48.md`:** lines **110–111**, **100–107**, **302–308**.

**Take the three in turn.**

**Build time** is `startedAt` to `updatedAt` over runs where `conclusion == "success"` — execution time,
not queue time, and not contaminated by runs that died early (Q42).

**Reliability** is the success rate per workflow, with `cancelled` counted separately so a deliberate
stop is not recorded as a fault (Q8).

**Deployment frequency** is the Deployments API grouped by environment over 30 days — arrivals in an
environment, not attempts at a pipeline (Q43).

**What the design actually changed is not that the data appeared.** GitHub had every one of these numbers
the whole time — the runs, the conclusions, the deployments were all recorded before the manager asked.
**What was missing was a query.**

**And the second, less obvious change is who produces the numbers.** Every metric here is **pulled from
the platform's own record**. Nothing depends on a developer filling in a field, a lead remembering a
figure, or anyone being honest about a bad week. That is what makes the numbers usable in a review —
and it is the difference between a metric and a report.

**The sentence to carry into the exam:** engineering metrics questions are almost always answered by
**the API that already holds the fact**, not by a process that asks someone for it.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Notification step at the end of a job** | Q2, Q19, Q25, Q28, Q44 | The job stops at the failing step. Separate job |
| **`needs:` without `if: failure()`** | Q19, Q34, Q36 | Notifies on every run. Channel gets muted |
| **`continue-on-error` breaking alerting** | Q19, Q28 | The job reports success, so `failure()` never fires |
| **Burn-down sought in Repository Insights** | Q3, Q17, Q27, Q46 | Three surfaces: code, pipelines, work |
| **Traffic used for deployment frequency** | Q1, Q25 | Traffic is views and clones |
| **Counting workflow runs as deployments** | Q43 | Runs are attempts. Deployments record the environment |
| **Releases counted as deployments** | Q38, Q43 | A release is a tag; a deployment reaches an environment |
| **Duration averaged over all runs** | Q10, Q26, Q29, Q42 | Failures die early. Success-only, failures separate |
| **`status` used instead of `conclusion`** | Q37 | `completed` is a lifecycle state, not an outcome |
| **Cancelled counted as failure** | Q8, Q29 | Neither success nor fault |
| **`createdAt` instead of `startedAt`** | Q9, Q29 | One includes queue time. Different question |
| **`closed` instead of `merged` for PRs** | Q41 | Closed includes abandoned work |
| **`type=error` on a reporting job** | Q14, Q40 | Fails the audit. Use `warning` |
| **Assuming a silent channel means no failures** | Q30, Q47 | An untested alert path is an assumption |
| **`condition:` in a GitHub workflow** | Q36 | `if:` in Actions, `condition:` in Azure Pipelines |

---

# What to memorise

**In `challenge-48.md`:** lines **69–78**, **100–118**, **147–170**, **192–195**, **302–308**.

```text
THREE INSIGHTS SURFACES - name the wrong one and the answer is wrong
  Repository Insights   Pulse | Contributors | Community | Traffic | Commits
                        Code frequency | Dependency graph | Network | Forks      THE CODE
  Actions (via API)     run duration | success rate | billed minutes             THE PIPELINES
  Projects Insights     burn-down | items by assignee | cycle time | by label    THE WORK

THE ALERTING PATTERN - a step at the end of a job does NOT run when the job fails
  notify-failure:
    needs: [build]        # gives the condition something to evaluate
    if: failure()         # default is success()
  payload: repository, branch, commit, actor + a LINK to the run
```

```bash
# Success rate                                      (lines 100-107)
gh run list --workflow deploy.yml --limit 100 --json conclusion --jq '{
  total: length,
  success: [.[] | select(.conclusion == "success")] | length,
  failure: [.[] | select(.conclusion == "failure")] | length,
  cancelled: [.[] | select(.conclusion == "cancelled")] | length }'
#   status     = lifecycle: queued | in_progress | COMPLETED
#   conclusion = outcome:   success | failure | cancelled | skipped
#   a run can be COMPLETED and have FAILED

# Duration - SUCCESSFUL RUNS ONLY                   (lines 110-111)
gh run list --workflow deploy.yml --limit 50 --status completed --json startedAt,updatedAt,conclusion \
  --jq '[.[] | select(.conclusion == "success")
        | ((.updatedAt|fromdateiso8601) - (.startedAt|fromdateiso8601))] | (add/length|floor)'
#   there is NO duration field - compute it
#   startedAt -> execution time    createdAt -> includes QUEUE time (feedback latency)
#   averaging failures in makes the pipeline look FASTER as it gets LESS RELIABLE

# Deployment frequency                              (lines 302-308)
gh api repos/<o>/<r>/deployments --jq '[.[] | select((.created_at|fromdateiso8601) > (now - 2592000))]
  | group_by(.environment) | .[] | {environment: .[0].environment, count: length}'
#   86400 = day   604800 = week   2592000 = 30 days
#   runs = attempts | releases = tags | DEPLOYMENTS = arrivals in an environment

# Billing                                           (lines 114-118)
gh api orgs/<org>/settings/billing/actions --jq '{total_minutes_used, included_minutes, total_paid_minutes_used}'

# Traffic (needs PUSH access)                       (lines 35-52)
gh api repos/<o>/<r>/traffic/views     .count (page loads) vs .uniques (distinct visitors)
gh api repos/<o>/<r>/traffic/clones  | .../popular/referrers | .../popular/paths
gh api repos/<o>/<r>/stats/contributors | .../stats/commit_activity
```

```yaml
# Deployment events                                 (lines 266-278)
on:
  deployment_status:                # fires on pending | in_progress | success | failure
jobs:
  track:
    if: github.event.deployment_status.state == 'success'
#   "it went well" has three names:  Actions conclusion=success | deployment state=success
#                                    Azure Pipelines Succeeded

# Skipped-job trap                                  (lines 455, 471-478)
  test:
    if: always()                    # let the job run
    needs: [build]
    steps:
      - if: needs.build.result == 'success'    # check EXPLICITLY - default success() treats skip as fail

# Azure Pipelines reporting job                     (line 373)
Write-Host "##vso[task.logissue type=warning]..."   # surface WITHOUT failing. type=error fails the run
$(System.AccessToken)                               # the run's own OAuth token - no PAT to rotate
```

```text
AZURE DEVOPS NOTIFICATIONS  (Project Settings > Notifications)   lines 316-348
  Category: Build | Event: A build completes with status Failed
  Filter: "Definition name" = "Production-Deploy"     <- scope it, or everything is noise
  RECOVERY = a SECOND subscription: succeeds AND last build was failed
  central + per project. a notification TASK must be added to every pipeline and is lost on clone

FLOW vs STOCK in the weekly digest                  (lines 406-413)
  flow  (has a date filter)   PRs merged this week | issues closed this week
  stock (no date filter)      PRs open | issues open
  flow alone hides a growing backlog. stock alone hides whether anything moves
  --state merged + .mergedAt   (NOT --state closed - that includes abandoned PRs)
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 49 |
| 38–43 | Re-read the trap index and the three surfaces, then move on |
| 30–37 | Rewrite the three surfaces and the alerting pattern from memory, then retake |
| Below 30 | Redo Tasks 2, 4 and 5 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 48.

:::danger The two rules

**Name the surface first.** Code → Repository Insights. Pipelines → Actions API. Work → Projects
Insights. Three quarters of this challenge is deciding which one.

**Notify from a separate job.** `needs:` plus `if: failure()`. A step at the end of the job does not run
in the only case you wrote it for.

And a silent alert channel is not good news until you have tested it.

:::
