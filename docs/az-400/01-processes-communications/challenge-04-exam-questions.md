---
sidebar_position: 4.5
toc_max_heading_level: 2
title: "Challenge 04: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 04 — AZ-400 exam questions

**48 questions** built only from what Challenge 04 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-04.md`**.

:::danger Read this before you start

The four DORA metrics split into **two pairs**, and knowing which pair a question is about answers most
of it.

**Throughput** — **deployment frequency** (how often you ship) and **lead time for changes** (how long
from commit to production).
**Stability** — **change failure rate** (what proportion of deploys break something) and **mean time to
recovery** (how fast you restore service).

**They are measured together because they trade off against each other.** A team that ships once a
quarter has wonderful stability and no throughput; a team that ships hourly with no tests has the
reverse. Elite performance is being good at **both**, which is why the exam always gives you four
numbers and asks for a level.

Two more facts do the rest of the work.

**Lead time is a *duration*, deployment frequency is a *rate*.** Confusing them is the single most
common error here.

**An average is not the metric you want.** Line 557 says it in the challenge's own words: use the
**median** when a few stale outliers distort the mean.

The scenario at line 21 is three questions the CTO asked and nobody could answer: how fast do we ship,
how often do we break production, and how quickly do we recover.

:::

---

# Section A — Multiple choice

---

## Q1

Which DORA metric measures the time between committing code and that code running in production?

- A. Deployment frequency, the rate at which changes reach production
- B. Change failure rate, the share of deployments needing remediation
- C. Mean time to recovery, measured from the start of an incident
- D. Lead time for changes, from commit to running in production

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-04.md`:** line **52**.

```text
Time from code commit to running in production.
```

**Lead time is a duration; deployment frequency is a rate.** That single distinction resolves most
confusions in this domain.

**And notice what the duration includes** (line 579): code review, CI/CD execution, and any manual
approval gate. A team with a fast pipeline and a two-day approval queue has a two-day lead time — which
is why the metric drives process changes and not just pipeline tuning.

**Why C is the adjacent stability metric.** MTTR also measures a duration, but it starts at an
**incident**, not at a commit.

</details>

---

## Q2

A team deploys three times per week with an average lead time of four days. What DORA level is that?

- A. High for deployment frequency and High for lead time
- B. Medium for deployment frequency and High for lead time
- C. High for deployment frequency and Medium for lead time
- D. Elite for deployment frequency and Elite for lead time

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-04.md`:** lines **46** and **57**.

```text
| High | Between once per day and once per week |
```

```text
| High | Between one day and one week |
```

**Read each number against its own table.** Three per week sits between daily and weekly — High. Four
days sits between one day and one week — High.

**The trap is that "three times a week" *sounds* impressive**, and Elite requires **multiple deploys per
day** (line 45). The gap between High and Elite is large and the exam knows the words feel closer than
the numbers are.

**Work these questions arithmetically, never by impression.** Convert to the table's units first, then
read off the row.

</details>

---

## Q3

What is the primary purpose of the Azure DevOps Analytics OData endpoint?

- A. To store build artifacts produced by pipeline runs
- B. To synchronise data between Azure DevOps and GitHub
- C. To provide a queryable reporting layer with aggregation
- D. To replace Azure Boards WIQL queries for work lists

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-04.md`:** lines **249** and **263**.

```bash
&\$apply=groupby((Area/AreaPath),aggregate(LeadTimeDays with average as AvgLeadTime, LeadTimeDays with max as MaxLeadTime, \$count as Count))
```

**Look at what that query does in one round trip: group by area path, then average, max and count.**
Doing the same against the operational REST API means paging every work item and aggregating
client-side.

**"Read-optimised" is the phrase to remember** (line 601). Analytics is a reporting store shaped for
aggregation; the operational API is shaped for individual records.

**Why D overstates it.** Boards queries (WIQL) are still how you build a *work list*. Analytics is how
you build a *trend*.

</details>

---

## Q4

Which practice most directly reduces change failure rate?

- A. Deploying more frequently so each change is smaller
- B. Reducing the number of developers touching the codebase
- C. Adding more manual approval gates before production
- D. Automated testing, progressive rollouts and feature flags

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-04.md`:** line **612**.

**Three mechanisms, three stages of the failure.** Automated testing stops the defect reaching
production; progressive rollout limits how many users meet it; feature flags let you switch it off
without redeploying.

**Why A is the answer that feels wrong and is defensible — but not the *most direct*.** Smaller, more
frequent deployments genuinely reduce failure rate, because each change is smaller and easier to
diagnose. It is an indirect effect; D names the mechanisms.

**Why C usually makes things worse.** More approvals lengthen lead time, which encourages batching, and
bigger batches fail more often. **A control that damages throughput often damages stability too** —
which is the whole reason DORA measures the four together.

</details>

---

## Q5

Analytics OData queries return 401 Unauthorized. What are the two causes to check?

- A. The organisation is on the free tier of Azure DevOps
- B. Analytics extension missing, or the PAT lacks its scope
- C. The OData query syntax contains an invalid property
- D. The project is private rather than publicly visible

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-04.md`:** lines **502–512** and **519**.

```text
**Fix:** The Azure DevOps Analytics extension must be installed on the organization. The PAT must have
Analytics (read) scope.
```

**Two different layers, and both return the same 401.** The extension is an organisation capability; the
scope is a property of your credential.

**The verification at lines 522–525 is the pattern worth copying** — request the `$metadata` document
and check for a 200 before debugging your actual query:

```bash
curl -s -o /dev/null -w "%{http_code}" -u ":$AZURE_DEVOPS_PAT" \
  "https://analytics.dev.azure.com/contoso-org/_odata/v4.0-preview/\$metadata"
```

**Why C would give a 400, not a 401.** Distinguish authentication failures from query failures by the
status code before you start rewriting the query.

</details>

---

## Q6

Deployment frequency reports zero despite active deployments. What is the most likely cause?

- A. The environment name in the query mismatches on case
- B. Deployments older than thirty days are purged by GitHub
- C. The deployments API response is paginated beyond one page
- D. The workflow lacks the permission to read deployments

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-04.md`:** lines **536–545**.

```bash
gh api repos/{owner}/{repo}/deployments --jq '.[].environment' | sort -u

# Common issue: environment is "Production" (capitalized) but query uses "production"
```

```text
**Fix:** GitHub deployment environments are case-sensitive.
```

**A silent, total failure from one capital letter** — the filter matches nothing, the count is zero, and
the dashboard says the team has not deployed.

**The diagnostic at line 536 is the right first move: list the distinct environment names actually
present.** Never assume what the deployments call themselves.

**Why C is a real concern handled elsewhere.** The queries use `--paginate` (lines 92, 99, 138) for
exactly that reason — but pagination would under-count, not zero-count.

</details>

---

## Q7

Lead time averages are inflated by a few PRs that sat open for weeks. What is the recommended fix?

- A. Exclude every PR that stayed open longer than one week
- B. Stop measuring lead time until the backlog is cleared
- C. Increase the sample size so outliers matter less overall
- D. Use the median, or filter to PRs created inside the window

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-04.md`:** lines **557–562**.

```bash
  --jq '[.[] | ((.mergedAt | fromdateiso8601) - (.createdAt | fromdateiso8601)) / 3600] | sort | .[length/2 | floor]'
```

**Sort, then take the middle element — that is a median in one line.** A single 900-hour PR moves a mean
enormously and a median barely at all.

**Why A is the tempting version that biases the result.** Deleting the slow PRs makes the number look
better without the process improving; the median keeps them in the dataset and stops them dominating.

**Why C makes it worse, not better.** More samples from the same skewed distribution gives you a more
precise wrong number.

</details>

---

## Q8

What Elite deployment frequency does DORA define?

- A. Once per day, on a predictable daily schedule
- B. On demand, meaning multiple deploys per day
- C. Once per week, at the end of each sprint
- D. Once per month, in a scheduled release window

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-04.md`:** line **45**.

```text
| Elite | On demand (multiple deploys per day) |
```

**"On demand" is the operative phrase.** Elite is not a schedule — it is the absence of one. Deployment
happens when the change is ready, not when the window opens.

**Which is why A is High rather than Elite** (line 46): once per day is still a cadence.

</details>

---

## Q9

What Elite lead time does DORA define?

- A. Less than one day from commit to production
- B. Less than one week from commit to production
- C. Less than one hour from commit to production
- D. Less than one month from commit to production

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-04.md`:** line **56**.

**Under an hour from commit to production leaves no room for a manual step.** That is the point of the
threshold — it can only be achieved by an automated path with no human gate in the middle.

**Note the gap in the table between Elite and High** (lines 56–57): Elite is under an hour, High starts
at one day. Anything between an hour and a day sits in a band the table does not name — the exam sticks
to the listed values, so read them literally.

</details>

---

## Q10

Which two thresholds are both "less than one hour" at Elite level?

- A. Mean time to recovery and change failure rate
- B. Deployment frequency and change failure rate
- C. Lead time for changes and deployment frequency
- D. Lead time for changes and time to restore service

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-04.md`:** lines **56** and **67**.

```text
| Elite | Less than one hour |     <- lead time
| Elite | Less than one hour |     <- recovery time
```

**The symmetry is not a coincidence.** Both are limited by the same capability: how fast can a change
get from a developer's machine into production. Recovery usually *is* a deployment — a rollback or a
forward fix — so a team that can ship in an hour can recover in an hour.

**Which is the strategic point of the whole framework.** Investing in deployment speed improves a
throughput metric **and** a stability metric at once.

</details>

---

## Q11

What change failure rate range does DORA classify as Elite?

- A. 0–15% of deployments requiring remediation
- B. 16–30% of deployments requiring remediation
- C. 31–45% of deployments requiring remediation
- D. 46–60% of deployments requiring remediation

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-04.md`:** line **78**.

```text
| Elite | 0-15% |
```

**Elite is not zero, and that is deliberate.** A team with a 0% failure rate is usually not shipping
enough to learn anything — the target is a small, recoverable failure rate, not perfection.

**And note the definition at line 74**: deployments that result in **degraded service requiring
remediation**. A deployment that fails in the pipeline and never reaches production is not a change
failure; it is the pipeline working.

</details>

---

## Q12

How does the challenge approximate deployment frequency when deployment records are unavailable?

- A. Count commits landing on the default branch
- B. Count merged pull requests to `main` as a proxy
- C. Count workflow runs that completed successfully
- D. Count published releases tagged in the repository

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-04.md`:** lines **105–108**.

```bash
# Alternative: count merged PRs to main as a deployment proxy
gh pr list --state merged --base main --limit 100 \
  --json mergedAt \
  --jq 'group_by(.mergedAt[:10]) | map({date: .[0].mergedAt[:10], count: length})'
```

**It is a *proxy*, and the word matters.** It is accurate only when every merge to `main` deploys —
which is true under GitHub Flow (Challenge 01) and false the moment a release branch exists.

**Why A is a worse proxy.** A PR with fourteen commits is one deployment, so counting commits inflates
the figure by however much people commit.

**Prefer the real deployments API** (line 91). The proxy is what you use while deployment tracking is
being set up.

</details>

---

## Q13

In the lead time query, what does dividing by 3600 accomplish?

- A. It converts a value in seconds into hours
- B. It normalises the duration to a percentage
- C. It converts milliseconds into whole seconds
- D. It converts a value in hours into whole days

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-04.md`:** line **121**.

```bash
    lead_time_hours: ((.mergedAt | fromdateiso8601) - (.createdAt | fromdateiso8601)) / 3600
```

**`fromdateiso8601` yields epoch *seconds*, so the difference is seconds** — and 3600 seconds is an
hour.

**Contrast with the JavaScript version at line 322**, which divides by `1000 * 60 * 60` because
JavaScript `Date` subtraction yields **milliseconds**. Same metric, two languages, two constants — and
the exam will show you one and ask about the other.

</details>

---

## Q14

What does the workflow at Task 5 use to detect possible instability?

- A. A failed build reported on the default branch
- B. An open issue carrying the `incident` label
- C. A rollback branch created from a release tag
- D. More than three production deploys in 24 hours

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-04.md`:** lines **359–363**.

```javascript
            if (recentDeployments.length > 3) {
              core.warning(
                `${recentDeployments.length} deployments in last 24h - possible instability`
              );
            }
```

**It is a heuristic, and it is worth being honest about that.** Many deployments in a day can mean an
Elite team shipping on demand, or a team repeatedly patching a broken release. The signal is
**ambiguous by itself**.

**Which is exactly why it emits `core.warning` and not a failure.** A metric that cannot distinguish
good from bad on its own should prompt a look, not block a pipeline.

</details>

---

## Q15

Which trigger does the deployment-tracking workflow use?

- A. `push` to `main`, running after each merge completes
- B. `deployment_status`, plus `workflow_run` on completion
- C. `schedule`, running nightly against the deployments API
- D. `release`, when a release is published from a tag

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-04.md`:** lines **289–292**.

```yaml
on:
  deployment_status:
  workflow_run:
    workflows: ["Deploy to Production"]
    types: [completed]
```

**Two triggers because deployments are recorded two different ways.** Some pipelines create a
**deployment object** (which fires `deployment_status`); others just run a workflow named "Deploy to
Production" (which fires `workflow_run`).

**And the `if:` at line 298 handles both payload shapes:**

```yaml
    if: github.event.deployment_status.state == 'success' || github.event.workflow_run.conclusion == 'success'
```

**Note the two different words for success.** A deployment status has a `state`; a workflow run has a
`conclusion` (Challenge 48 Q37).

</details>

---

## Q16

What does the weekly report workflow produce?

- A. A GitHub issue with a metrics table and labels
- B. A dashboard widget refreshed on the Azure board
- C. A Slack message posted to the engineering channel
- D. A CSV artifact uploaded to the workflow run

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-04.md`:** lines **481–487**.

```javascript
            await github.rest.issues.create({
              title: `DORA Metrics Report - Week of ${today}`,
              body: body,
              labels: ['metrics', 'automated']
            });
```

**An issue is a deliberate choice: it is commentable, assignable and closable.** The report body ends
with an action checklist (lines 475–478) — review with leads, identify improvements, update OKRs — so
the artifact is a **conversation**, which is what the CTO asked for at line 21: metrics used to drive
monthly reviews.

**A message would be read and scrolled past. An issue can be worked.**

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** are DORA metrics? (Choose three.)

- A. Deployment frequency, as deploys per period
- B. Story points completed in each sprint
- C. Lead time for changes, commit to production
- D. Lines of code written per developer
- E. Change failure rate across deployments
- F. Number of pull requests left open

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-04.md`:** lines **39–81**.

**Mean time to recovery is the fourth** (line 61). Four metrics, no more — and the discipline of the
framework is that it refuses to add a fifth.

**Why B, D and F are the metrics organisations reach for instead**, and all three measure activity
rather than outcome. Story points are team-relative and not comparable across teams; lines of code
reward verbosity; open PRs is a queue depth, not a delivery measure.

</details>

---

## Q18

Which **two** DORA metrics measure **throughput**? (Choose two.)

- A. Change failure rate across production deployments
- B. Mean time to recovery after an incident starts
- C. Deployment frequency, as deploys per time period
- D. Cycle time, from work started to work finished
- E. Lead time for changes, from commit to production

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-04.md`:** lines **39–59** against **61–81**.

**Throughput = how much and how fast you ship. Stability = how often it breaks and how fast you fix
it.** Change failure rate and MTTR are the stability pair.

**Why the pairing is the framework's central claim.** Optimising throughput alone produces a team that
ships breakage quickly; optimising stability alone produces a team that ships nothing. **Elite means
both**, and that is why the exam gives you four numbers.

**Why D is a real metric from a different framework.** Cycle time is a flow metric with a widget in this
challenge (line 236) and it is not one of the four.

</details>

---

## Q19

Which **three** does the challenge use to calculate DORA metrics from GitHub data? (Choose three.)

- A. Repository traffic and clone statistics
- B. The deployments API filtered by environment
- C. Actions billing minutes consumed per workflow
- D. Merged pull request timestamps for lead time
- E. Raw commit counts on the default branch
- F. Issues labelled `incident` for recovery time

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-04.md`:** lines **91–93**, **115–121**, **155–159**.

```bash
gh issue list --label "incident" --state closed --limit 50 \
  --json number,createdAt,closedAt \
  --jq '[.[] | { recovery_hours: ((.closedAt | fromdateiso8601) - (.createdAt | fromdateiso8601)) / 3600 }]'
```

**The MTTR approach is the one worth noticing: an incident is a *labelled issue*, and recovery is
`closedAt − createdAt`.** That works only if the team actually opens an issue when production breaks
and closes it when service is restored — a **process** precondition, not a tooling one.

**Which is the honest caveat on all three.** Every DORA calculation depends on a discipline: recording
deployments, using PRs, labelling incidents.

</details>

---

## Q20

Which **three** widgets does the challenge add to the Azure DevOps dashboard? (Choose three.)

- A. Sprint Burndown, over the current iteration
- B. Deployment frequency, over the last 30 days
- C. Velocity, across the last six sprints
- D. Mean time to recovery, over the last quarter
- E. Cycle Time, for User Stories over 30 days
- F. Change failure rate, across all environments

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-04.md`:** lines **201–241**.

```json
{"name": "Sprint Burndown",  "settings": "{\"timePeriod\":\"currentIteration\",\"aggregation\":\"storyPoints\"}"}
{"name": "Velocity",         "settings": "{\"numberOfSprints\":6}"}
{"name": "Cycle Time",       "settings": "{\"timePeriod\":\"last30Days\",\"workItemType\":\"User Story\"}"}
```

**And note what that means: the built-in widgets are *flow* metrics, not DORA metrics.** Burndown,
velocity and cycle time describe work moving through a board. Deployment frequency, MTTR and change
failure rate have no stock widget here — which is why Tasks 2, 4 and 6 compute them from APIs.

**That gap is a genuinely exam-relevant fact.** "Which DORA metric can I get from a built-in Azure
Boards widget?" has an uncomfortable answer: none of them directly.

</details>

---

## Q21

Which **two** OData features make Analytics suited to reporting? (Choose two.)

- A. Write access to update work items in place
- B. `$apply` with `groupby` and `aggregate` clauses
- C. Artifact storage for pipeline build outputs
- D. `$filter` on dates and on entity properties
- E. Real-time streaming of work item changes

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-04.md`:** lines **258** and **263**.

```text
$filter=WorkItemType eq 'User Story' and StateCategory eq 'Completed' and CompletedDate gt 2025-01-01T00:00:00Z
$apply=groupby((Area/AreaPath),aggregate(LeadTimeDays with average as AvgLeadTime, ... $count as Count))
```

**`$apply` is what makes it a reporting layer rather than a list API.** Grouping and aggregation happen
server-side, so you receive a summary instead of ten thousand records.

**Why A is definitionally wrong.** Analytics is **read-optimised** (line 601) — a reporting store. You
do not write work items through it.

**Note `StateCategory eq 'Completed'` rather than a state name.** State categories are stable across
process templates; state *names* differ between Agile, Scrum and CMMI — so filtering on the category is
what makes the query portable.

</details>

---

## Q22

Which **two** are true about the deployment-frequency query? (Choose two.)

- A. It counts all deployments regardless of environment
- B. It queries the Azure DevOps Analytics endpoint
- C. It filters on `environment == "production"` exactly
- D. It requires the Analytics extension to be installed
- E. Environment names are matched case-sensitively

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-04.md`:** lines **93** and **545**.

**The filter is what makes it *production* frequency**, and the case-sensitivity is what makes the whole
thing silently return zero (Q6).

**Why B and D belong to the Azure DevOps half of the challenge** (lines 249–279). The GitHub
calculations use `gh api` and `gh pr list`; Analytics is the Azure DevOps route to the same numbers.

</details>

---

## Q23

Which **two** improve the quality of a lead time figure? (Choose two.)

- A. Removing every PR that stayed open longer than a week
- B. Reporting the median rather than the arithmetic mean
- C. Rounding each duration to the nearest whole day
- D. Averaging the figure across all repositories at once
- E. Restricting the sample to PRs created in the window

<details>
<summary>Show answer</summary>

### Answer: B, E

**In `challenge-04.md`:** lines **557–562**.

**B resists outliers; E ensures the window is the window** — a PR opened four months ago and merged
yesterday is not evidence about this month's process.

**Why A is the version that lies.** Excluding the slow tail makes the number improve while the process
does not, and the slow tail is usually where the interesting problem is.

**Why D destroys the signal.** Averaging a fast service with a slow monolith produces a number that
describes neither, and hides the team that needs help — which is precisely what the CTO wanted the data
for (line 21: distinguish high-performing teams from those struggling).

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must measure delivery performance automatically, classify it against DORA levels,
show it on dashboards, and use it to drive monthly improvement conversations.

---

## Q24

**Proposed solution:** Calculate deployment frequency from the deployments API filtered to production,
lead time from merged PR timestamps reported as a median, change failure rate from failed deployment
statuses, and MTTR from issues labelled `incident`. Add an Azure DevOps dashboard with burndown,
velocity and cycle time widgets, and use Analytics OData with `$apply` for cycle and lead time trends.
Add a workflow that records deployment events, and a scheduled workflow that opens a weekly issue with
the metric table, levels and an action checklist.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-04.md`:** lines **91–162**, **171–241**, **255–278**, **286–366**, **380–487**.

| CTO's question (line 21) | Metric | Source |
|---|---|---|
| How fast do we ship? | Deployment frequency, lead time | Deployments API, merged PRs |
| How often do we break production? | Change failure rate | Deployment statuses, reverts |
| How quickly do we recover? | MTTR | `incident` issues |
| Used in monthly reviews | Weekly issue with an action checklist | Scheduled workflow |

**The median clause is what makes the lead time figure trustworthy** (Q7), and the action checklist is
what makes the report a conversation rather than a number nobody acts on.

</details>

---

## Q25

**Proposed solution:** Ask each team lead to estimate their deployment frequency and lead time in the
monthly review. Track lines of code committed per developer as a productivity measure. Report the mean
lead time across all repositories combined. Measure change failure rate as the number of failed builds.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures, and each is a different way of measuring the wrong thing.**

**Estimates are what Contoso already has.** Line 21 says teams report anecdotal estimates and there is no
data — asking the same people again produces the same anecdotes.

**Lines of code per developer is not a delivery metric.** It rewards verbosity, punishes deletion, and
is not one of the four (Q17). Worse, once it is reported it will be optimised.

**A mean across all repositories** blends fast services with slow ones and hides the struggling team the
CTO explicitly wants to identify (Q23).

**And failed *builds* are not change failures.** Line 74 defines the metric as deployments causing
**degraded service requiring remediation** — a build that fails in CI is the pipeline **preventing** a
change failure, and counting it inverts the meaning.

</details>

---

## Q26

**Proposed solution:** Calculate deployment frequency from the deployments API filtered to production,
lead time from merged PR timestamps, change failure rate from failed deployment statuses, and MTTR from
`incident` issues. Add the Azure DevOps dashboard and Analytics queries. Add the deployment-tracking
workflow and the scheduled weekly report. Report lead time as the mean, since the median discards
information about the outliers.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**The stated reasoning is wrong about what a median does.** It does not discard anything: the outliers
remain in the dataset, they simply stop dominating the summary. Line 562 sorts the whole list and takes
the middle element — nothing is removed.

**And the practical consequence is a number that misleads in both directions.** One PR opened before a
holiday and merged after it can add hundreds of hours to a mean over a small sample. In a week with
twelve PRs, one 900-hour outlier and eleven 4-hour merges gives a mean of **79 hours** — a "Medium"
classification for a team that is actually shipping in an afternoon.

**Then it misleads the other way.** The following week, without the outlier, the mean collapses and the
team appears to have made a dramatic improvement they did not make. **A metric that swings on one data
point cannot support the monthly improvement conversation it exists for.**

**Break scenario 3 names this exactly** (line 549): *"Lead time calculation is inflated by stale PRs."*
The challenge's own answer is the median, or a window filter — and preferably both.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — the four metrics

| # | Statement | Answer |
|---|---|---|
| 1 | Lead time measures commit to production |  |
| 2 | Deployment frequency is a duration |  |
| 3 | MTTR measures time to restore service after an incident |  |
| 4 | Change failure rate counts failed builds |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Lead time measures commit to production | **Yes** |
| 2 | Deployment frequency is a duration | **No** |
| 3 | MTTR measures time to restore service after an incident | **Yes** |
| 4 | Change failure rate counts failed builds | **No** |

**In `challenge-04.md`:** lines **52**, **41**, **63**, **74**.

Row 2 is the rate-versus-duration confusion, and row 4 inverts the definition — a build that fails
**prevented** a change failure.

</details>

---

## Q28 — DORA levels

| # | Statement | Answer |
|---|---|---|
| 1 | Elite deployment frequency is multiple deploys per day |  |
| 2 | Elite lead time is under one hour |  |
| 3 | Elite change failure rate is 0% |  |
| 4 | Elite MTTR is under one hour |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Elite deployment frequency is multiple deploys per day | **Yes** |
| 2 | Elite lead time is under one hour | **Yes** |
| 3 | Elite change failure rate is 0% | **No** |
| 4 | Elite MTTR is under one hour | **Yes** |

**In `challenge-04.md`:** lines **45**, **56**, **78**, **67**.

Row 3 is the one people over-correct on. **Elite is 0–15%**, not zero — a team with no failures is
usually a team not shipping.

Rows 2 and 4 are the "less than one hour" pair (Q10), and they share a cause.

</details>

---

## Q29 — calculation

| # | Statement | Answer |
|---|---|---|
| 1 | `fromdateiso8601` returns epoch seconds |  |
| 2 | Dividing by 3600 converts seconds to hours |  |
| 3 | The median is more robust to outliers than the mean |  |
| 4 | Counting merged PRs is an exact deployment count |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `fromdateiso8601` returns epoch seconds | **Yes** |
| 2 | Dividing by 3600 converts seconds to hours | **Yes** |
| 3 | The median is more robust to outliers than the mean | **Yes** |
| 4 | Counting merged PRs is an exact deployment count | **No** |

**In `challenge-04.md`:** lines **121**, **121**, **562**, **105**.

Row 4: the challenge calls it a **proxy** (line 105), and it is exact only when every merge deploys.

</details>

---

## Q30 — dashboards and Analytics

| # | Statement | Answer |
|---|---|---|
| 1 | Analytics OData supports server-side aggregation |  |
| 2 | The Analytics extension must be installed on the organisation |  |
| 3 | A PAT needs Analytics (read) scope |  |
| 4 | Burndown, velocity and cycle time are DORA metrics |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Analytics OData supports server-side aggregation | **Yes** |
| 2 | The Analytics extension must be installed on the organisation | **Yes** |
| 3 | A PAT needs Analytics (read) scope | **Yes** |
| 4 | Burndown, velocity and cycle time are DORA metrics | **No** |

**In `challenge-04.md`:** lines **263**, **519**, **519**, **201–241**.

Row 4 is the distinction that matters when someone claims a dashboard "shows DORA". **Those three are
flow metrics** — useful, and not the four (Q20).

</details>

---

# Section E — Drag and drop

---

## Q31

Match each CTO question to the metric that answers it.

| Question | Metric |
|---|---|
| How fast do we ship features? |  |
| How often do we ship? |  |
| How often do we break production? |  |
| How quickly do we recover? |  |
| Which of these are throughput? |  |
| Which of these are stability? |  |

**Options:** Change failure rate · Deployment frequency · Failure rate and MTTR · Frequency and lead time · Lead time for changes · Mean time to recovery

<details>
<summary>Show answer</summary>

| Question | Metric |
|---|---|
| How fast do we ship features? | **Lead time for changes** |
| How often do we ship? | **Deployment frequency** |
| How often do we break production? | **Change failure rate** |
| How quickly do we recover? | **Mean time to recovery** |
| Which of these are throughput? | **Frequency and lead time** |
| Which of these are stability? | **Failure rate and MTTR** |

**In `challenge-04.md`:** lines **21**, **41**, **52**, **63**, **74**.

**The CTO asked three questions and DORA answers them with four metrics** — because "how fast do we
ship" splits into a rate and a duration, and those are genuinely different things to improve.

</details>

---

## Q32

Match each metric to its Elite threshold.

| Metric | Elite |
|---|---|
| Deployment frequency |  |
| Lead time for changes |  |
| Mean time to recovery |  |
| Change failure rate |  |

**Options:** 0–15% · Less than one hour · On demand — multiple per day

<details>
<summary>Show answer</summary>

| Metric | Elite |
|---|---|
| Deployment frequency | **On demand — multiple per day** |
| Lead time for changes | **Less than one hour** |
| Mean time to recovery | **Less than one hour** |
| Change failure rate | **0–15%** |

**In `challenge-04.md`:** lines **45**, **56**, **67**, **78**.

**Two "under one hour" rows and one percentage band.** If you remember only that shape, you can
reconstruct most of the table under exam pressure.

</details>

---

## Q33

Arrange the steps to produce an automated weekly DORA report.

**Items:** Open an issue containing the table and an action checklist · Calculate deployment frequency
from production deployments in the last seven days · Classify each figure against the DORA levels ·
Calculate lead time from PRs merged in the last seven days

<details>
<summary>Show answer</summary>

### Answer

1. Calculate deployment frequency from production deployments in the last seven days — lines **399–412**
2. Calculate lead time from PRs merged in the last seven days — lines **421–448**
3. Classify each figure against the DORA levels — lines **413–414**, **449**
4. Open an issue containing the table and an action checklist — lines **461–487**

**Step 3 is the one that turns data into a conversation.** A report saying "3 deploys, 41 hours" invites
argument about whether that is good; a report saying "**High**, **High**" against a published scale does
not.

**And note the guard at lines 437–441**: when no PRs merged this week, the workflow outputs `N/A` rather
than dividing by zero. **A metrics job that crashes on a quiet week gets switched off.**

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Deployment frequency reports zero |  |
| Analytics query returns 401 |  |
| Lead time swings wildly week to week |  |
| Lead time looks great, nothing improved |  |
| The weekly report job fails on a quiet week |  |
| "DORA dashboard" shows only burndown and velocity |  |

**Options:** Built-in widgets are flow metrics, not DORA · Environment name case mismatch · Extension not installed, or PAT lacks Analytics scope · Mean over a small sample with outliers · No guard for an empty PR set · Slow PRs excluded from the sample

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| Deployment frequency reports zero | **Environment name case mismatch** |
| Analytics query returns 401 | **Extension not installed, or PAT lacks Analytics scope** |
| Lead time swings wildly week to week | **Mean over a small sample with outliers** |
| Lead time looks great, nothing improved | **Slow PRs excluded from the sample** |
| The weekly report job fails on a quiet week | **No guard for an empty PR set** |
| "DORA dashboard" shows only burndown and velocity | **Built-in widgets are flow metrics, not DORA** |

**In `challenge-04.md`:** lines **545**, **519**, **557**, **557**, **437–441**, **201–241**.

**Rows 3 and 4 are the same statistic abused in two directions** — and both produce a number the monthly
review cannot act on.

</details>

---

## Q35

Match each metric to where the challenge gets it.

| Metric | Source |
|---|---|
| Deployment frequency |  |
| Lead time for changes |  |
| Change failure rate |  |
| Mean time to recovery |  |
| Cycle time trend |  |
| Sprint progress |  |

**Options:** Analytics OData with `$apply` · Burndown and velocity widgets · Deployment statuses, or revert/hotfix PR titles · Deployments API, filtered to production · Issues labelled `incident`, created to closed · Merged PR `createdAt` to `mergedAt`

<details>
<summary>Show answer</summary>

| Metric | Source |
|---|---|
| Deployment frequency | **Deployments API, filtered to production** |
| Lead time for changes | **Merged PR `createdAt` to `mergedAt`** |
| Change failure rate | **Deployment statuses, or revert/hotfix PR titles** |
| Mean time to recovery | **Issues labelled `incident`, created to closed** |
| Cycle time trend | **Analytics OData with `$apply`** |
| Sprint progress | **Burndown and velocity widgets** |

**In `challenge-04.md`:** lines **91–93**, **115–121**, **137–148**, **155–159**, **263**, **201–224**.

**Every one of these depends on a discipline being kept.** Deployments recorded, work done through PRs,
incidents labelled. **The metric is only as honest as the practice underneath it** — which is the
caveat to state whenever an exam question asks you to "measure" something.

</details>

---

# Section F — Hot area

---

## Q36

```bash
gh pr list --state merged --base main --limit 50 \
  --json createdAt,mergedAt \
  --jq '[.[] | ((.mergedAt | [BLANK 1]) - (.createdAt | [BLANK 1])) / [BLANK 2]] |
    (add / length)'
```

Requirement: average lead time in hours.

- **BLANK 1:** `todate` / `tonumber` / `fromdateiso8601` / `strptime`
- **BLANK 2:** `1000` / `3600` / `86400` / `60`

<details>
<summary>Show answer</summary>

### Answer: `fromdateiso8601`, `3600`

**In `challenge-04.md`:** lines **127–128**.

**`fromdateiso8601` converts an ISO timestamp to epoch seconds**, so the subtraction yields seconds and
3600 gives hours.

**`86400` is the distractor and it is a real constant** — seconds per day, which you would use if the
requirement asked for days. Read the unit in the question.

**And in JavaScript the same calculation divides by `1000 * 60 * 60`** (line 322), because `Date`
subtraction gives milliseconds (Q13).

</details>

---

## Q37

```bash
gh api repos/OWNER/REPO/deployments \
  --[BLANK 1] \
  --jq '[.[] | select(.[BLANK 2] == "production")] | length'
```

- **BLANK 1:** `limit 30` / `slurp` / `raw-field` / `paginate`
- **BLANK 2:** `ref` / `environment` / `task` / `description`

<details>
<summary>Show answer</summary>

### Answer: `paginate`, `environment`

**In `challenge-04.md`:** lines **91–93**.

**Without `--paginate` you get the first page only**, so a busy repository silently under-counts its
deployment frequency — a wrong number that looks plausible, which is worse than an error.

**And `environment` is case-sensitive** (Q6). List the distinct values first (line 536) rather than
assuming.

</details>

---

## Q38

```bash
$filter=WorkItemType eq 'User Story' and [BLANK 1] eq 'Completed'
  and CompletedDate [BLANK 2] 2025-01-01T00:00:00Z
&$apply=groupby((Area/AreaPath),aggregate(LeadTimeDays with [BLANK 3] as AvgLeadTime))
```

- **BLANK 1:** `State` / `Status` / `StateCategory` / `Reason`
- **BLANK 2:** `>` / `gt` / `after` / `since`
- **BLANK 3:** `avg` / `mean` / `sum` / `average`

<details>
<summary>Show answer</summary>

### Answer: `StateCategory`, `gt`, `average`

**In `challenge-04.md`:** lines **258** and **263**.

**`StateCategory` rather than `State` is the portability decision.** Categories — Proposed, InProgress,
Resolved, Completed — are consistent across Agile, Scrum and CMMI. State **names** are not: Agile has
"Closed" where Scrum has "Done", so a query filtering on `State` breaks the moment a project uses a
different template.

**And OData uses word operators** — `gt`, `ge`, `lt`, `le`, `eq`, `ne` — because `>` and `&` are
reserved in a URL.

</details>

---

## Q39

```yaml
on:
  [BLANK 1]:
  workflow_run:
    workflows: ["Deploy to Production"]
    types: [completed]

jobs:
  record-deployment:
    if: github.event.deployment_status.[BLANK 2] == 'success'
        || github.event.workflow_run.[BLANK 3] == 'success'
```

- **BLANK 1:** `deployment` / `deployment_status` / `release` / `push`
- **BLANK 2:** `conclusion` / `status` / `state` / `result`
- **BLANK 3:** `state` / `conclusion` / `status` / `outcome`

<details>
<summary>Show answer</summary>

### Answer: `deployment_status`, `state`, `conclusion`

**In `challenge-04.md`:** lines **290–298**.

**Two payloads, two words for the same idea, and the exam tests exactly this.** A deployment status has
a **`state`**; a workflow run has a **`conclusion`**. Swap them and the condition is never true, so the
job silently never runs and deployment frequency reports zero.

**That is the same class of silent failure as the environment-name mismatch** — a metric that reports
nothing rather than erroring.

</details>

---

## Q40

```javascript
const sevenDaysAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString();
...
core.setOutput('frequency', thisWeek.length >= [BLANK 1] ? 'Elite' :
  thisWeek.length >= [BLANK 2] ? 'High' : 'Medium');
```

Requirement: classify weekly deployment count against DORA levels.

- **BLANK 1:** `1` / `30` / `7` / `100`
- **BLANK 2:** `7` / `1` / `0` / `5`

<details>
<summary>Show answer</summary>

### Answer: `7`, `1`

**In `challenge-04.md`:** lines **413–414**.

```javascript
            core.setOutput('frequency', thisWeek.length >= 7 ? 'Elite' :
              thisWeek.length >= 1 ? 'High' : 'Medium');
```

**Seven deployments in seven days is one per day, which the workflow treats as the Elite floor** — a
simplification of "multiple deploys per day" (line 45) that is good enough for a weekly report.

**And `>= 1` for High matches the table's "between once per day and once per week"** (line 46): at least
one deployment in the week.

**Note the ordering of a ternary chain matters.** Elite must be tested first; reverse the two conditions
and everything with one or more deployments reports High.

</details>

---

## Q41

```javascript
const avg = leadTimes.reduce((a, b) => a + b, 0) / leadTimes.length;
core.setOutput('level', avg < [BLANK 1] ? 'Elite' : avg < [BLANK 2] ? 'High' : 'Medium');
```

Requirement: classify average lead time in **hours** against DORA levels.

- **BLANK 1:** `24` / `60` / `0.5` / `1`
- **BLANK 2:** `24` / `720` / `168` / `48`

<details>
<summary>Show answer</summary>

### Answer: `1`, `168`

**In `challenge-04.md`:** line **449**.

**`168` is one week in hours** — 7 × 24 — which is the High/Medium boundary at line 57. `1` is the Elite
threshold from line 56.

**Getting the unit right is the whole question.** The values are in hours because line 444 divides by
`1000 * 60 * 60`; if the array held days, the constants would be 1/24 and 7.

</details>

---

# Section G — Case study

## Case study: Contoso delivery performance

### Background

Contoso Ltd's CTO asked three questions at an all-hands: **"How fast do we ship features? How often do
we break production? How quickly do we recover when things go wrong?"** The room went silent. The
organisation has **never measured** its software delivery performance. Teams report **anecdotal
estimates**, and there is **no data to distinguish high-performing teams from those struggling**. The
CTO wants DORA metrics tracked **automatically**, shown on dashboards, and used in **monthly engineering
reviews**.

### Requirements

**Measurement**

- All four DORA metrics must be produced from system data, not from estimates
- Figures must be robust to a small number of extreme outliers
- Per-team figures must remain distinguishable, not blended into one organisation-wide average

**Presentation**

- Azure DevOps teams need dashboard widgets for sprint-level flow
- Trends over time must be queryable with aggregation, not by paging every record

**Use**

- A recurring report must classify each metric against DORA levels
- The report must prompt an action, not just publish a number

---

## Q42

How should lead time be measured so that it is robust to outliers?

- A. Mean across every merged pull request ever recorded
- B. The longest pull request within the measurement window
- C. Median of `createdAt` to `mergedAt` for PRs in the window
- D. The fastest pull request within the measurement window

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-04.md`:** lines **115–121** and **557–562**.

**Both halves of the requirement are addressed: the median resists the outlier, and the window keeps the
sample relevant.**

**Why A is what the shipped workflow actually does** (line 447) — and Break scenario 3 at line 549 is the
challenge telling you it has a known weakness. That is worth noticing: the lab's own code has the flaw
the lab then teaches you to fix.

**Why B and D are not summaries.** A single extreme value describes one PR, not the process.

</details>

---

## Q43

Which figures classify a team deploying three times weekly with a four-day lead time?

- A. Elite for deployment frequency and Elite for lead time
- B. High for deployment frequency and High for lead time
- C. Medium for deployment frequency and High for lead time
- D. High for deployment frequency and Elite for lead time

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-04.md`:** lines **46** and **57**.

**Convert first, then read the table.** Three per week is between daily and weekly → High. Four days is
between one day and one week → High.

**Why A is the impression rather than the arithmetic** (Q2). Elite needs multiple deploys **per day** and
a lead time under **one hour** — the gap is an order of magnitude, and the labels obscure that.

</details>

---

## Q44

How should per-team distinguishability be preserved?

- A. Group by area path with `$apply`, per-team aggregates
- B. Report only the organisation-wide total each month
- C. Report the best-performing team's figures as the target
- D. Average the figures across all repositories together

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-04.md`:** line **263**.

```text
$apply=groupby((Area/AreaPath),aggregate(LeadTimeDays with average as AvgLeadTime, LeadTimeDays with max as MaxLeadTime, $count as Count))
```

**`groupby((Area/AreaPath))` is the requirement expressed as a query.** One call returns an average, a
maximum and a count **per team**.

**Why B and D defeat the CTO's stated purpose.** Line 21 asks for data that distinguishes high performers
from teams struggling; an organisation-wide average is precisely the number that cannot.

**And note `MaxLeadTime` alongside the average.** Reporting a maximum next to a mean is a cheap way to
see whether the mean is being distorted, without switching to a median.

</details>

---

## Q45

Which **two** make the weekly report drive improvement rather than just publish numbers? (Choose two.)

- A. Posting the raw numbers to an engineering channel
- B. Emailing the figures to the CTO each Monday
- C. Classifying each metric against its DORA level
- D. Storing the figures in a reporting database
- E. An action checklist included in the issue body

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-04.md`:** lines **413–414**, **449**, **475–478**.

```text
### Actions
- [ ] Review metrics with engineering leads
- [ ] Identify improvement opportunities
- [ ] Update team OKRs if needed
```

**C gives the number meaning; E gives it an owner and a next step.** Without the level, "41 hours" is
arguable; without the checklist, nothing happens after the report is read.

**Why A and B deliver without prompting.** A message is read and scrolled past — which is why the report
is an **issue** (Q16): assignable, commentable, closable.

</details>

---

## Q46

Which metric would a built-in Azure DevOps dashboard widget **not** give Contoso directly?

- A. Sprint burndown for the current iteration
- B. Velocity across the last several sprints
- C. Cycle time for user stories over 30 days
- D. Change failure rate across deployments

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-04.md`:** lines **201–241**.

**The three widgets configured are burndown, velocity and cycle time — all flow metrics.** None of the
four DORA metrics has a stock widget here, which is why Tasks 2, 4, 5 and 6 compute them from APIs and
workflows.

**This is a genuinely useful thing to know going into the exam**, because "add a dashboard widget" is
offered as the answer to DORA questions and it is usually wrong. **Flow metrics come from widgets; DORA
metrics come from queries.**

</details>

---

## Q47

Ten months in, the CTO notes that lead time has "improved" from 62 hours to 9 hours over one quarter,
while deployment frequency and change failure rate are unchanged and no process was altered. The
platform team recently changed the report to exclude pull requests open longer than seven days, calling
them stale.

What is the most likely explanation?

- A. The team genuinely got faster over the whole quarter
- B. Excluding the slow tail moved the number, not the work
- C. The Analytics extension was reconfigured recently
- D. Deployment environments were renamed in the workflow

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-04.md`:** lines **549** and **557**.

**"No process was altered" and "the other three metrics are unchanged" are the two details that give it
away.** A real sevenfold improvement in lead time would move deployment frequency too — the two
throughput metrics are coupled, because shipping faster means shipping more often.

**What actually changed is the denominator.** Removing every PR over seven days deletes exactly the cases
that make lead time long, so the figure falls without a single delivery improving. **The slow tail is
not noise — it is the problem the metric exists to reveal.**

**This is Q23's option A shipped to production**, and it is the most common way a metrics programme
quietly stops being useful: the number becomes the goal, and the easiest way to move a number is to
change what it counts.

**The fix restores the full sample and switches to the median** (line 562), which is the challenge's own
answer to outlier distortion — the outliers stay in the dataset and stop dominating the summary.

**And the general defence: when a metric improves sharply, ask what changed in the measurement before
you celebrate the delivery.**

</details>

---

## Q48

A year on, the monthly review opens with four numbers, a level against each, and an argument about which
one to work on next.

Which explanation best accounts for the change?

- A. The teams worked harder once they knew they were measured
- B. The dashboard made everyone across engineering more aware
- C. The numbers now come from systems, not people's estimates
- D. The CTO began asking sharper questions at each review

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-04.md`:** lines **39–81**, **263**, **413–414**, **475–487**, **21**.

**Take the CTO's three questions and follow what each now has behind it.**

*How fast do we ship?* — a **rate** and a **duration**, from deployment records and merged PRs, so
"fast" has two independent meanings and both are visible.

*How often do we break production?* — change failure rate, defined as deployments causing degraded
service (line 74) rather than any red build, so the number means what people think it means.

*How quickly do we recover?* — MTTR from labelled incidents, which is also the metric most improved by
the same investment as lead time (Q10).

**And the DORA levels are what make the conversation possible.** "41 hours" invites debate; "High, and
Elite is under an hour" identifies a gap and its size.

**What actually changed is not effort.** The scenario at line 21 does not describe a lazy organisation —
it describes one with **no instrument**. Teams estimated because estimating was the only option, and
estimates cannot distinguish a struggling team from a modest one.

**The graded insight is that measurement had to become a by-product of the work.** Deployments recorded
by the deployment, lead time derived from the PRs people already open, incidents counted from the issues
they already file. **A metric that requires someone to report it inherits the reliability of their
memory** — which is exactly what the room's silence was.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Lead time vs deployment frequency** | Q1, Q27, Q31 | One is a duration, one is a rate |
| **"Three a week" mistaken for Elite** | Q2, Q28, Q43 | Elite is multiple **per day** and under **one hour** |
| **Elite change failure rate assumed 0%** | Q11, Q28 | 0–15% |
| **Failed builds counted as change failures** | Q25, Q27 | Degraded **service** requiring remediation |
| **Mean lead time over a small sample** | Q7, Q23, Q26, Q42 | Median, plus a window filter |
| **Excluding slow PRs to "clean" the data** | Q23, Q47 | Moves the number, not the process |
| **Blending teams into one average** | Q23, Q25, Q44 | `groupby((Area/AreaPath))` |
| **Environment name case mismatch** | Q6, Q22, Q34, Q37 | Case-sensitive. Silent zero |
| **`state` vs `conclusion`** | Q39 | Deployment status has `state`; workflow run has `conclusion` |
| **Missing `--paginate`** | Q37 | Silent under-count on a busy repository |
| **`State` instead of `StateCategory`** | Q38 | Categories are portable across process templates |
| **Widgets assumed to show DORA** | Q20, Q30, Q46 | Burndown, velocity and cycle time are flow metrics |
| **Analytics 401 debugged as a query error** | Q5 | 401 = extension or PAT scope. 400 = query |
| **No guard for an empty sample** | Q33, Q34 | A metrics job that crashes gets switched off |
| **Vanity metrics — lines of code, story points across teams** | Q17, Q25 | Activity is not delivery |

---

# What to memorise

**In `challenge-04.md`:** lines **39–81**, **255–278**, **399–449**.

```text
THE FOUR - two pairs, measured together because they TRADE OFF
  THROUGHPUT   deployment frequency   how OFTEN            (a RATE)
               lead time for changes  commit -> production (a DURATION)
  STABILITY    change failure rate    % of deploys causing DEGRADED SERVICE needing remediation
               MTTR                   incident -> service restored

                      Elite                    High                 Medium
  deploy freq    on demand, MULTIPLE/DAY   daily..weekly        weekly..monthly
  lead time      < 1 HOUR                  1 day..1 week        1 week..1 month
  MTTR           < 1 HOUR                  < 1 day              1 day..1 week
  change fail    0-15%                     16-30%               31-45%

  lead time and MTTR share the "< 1 hour" bar - both are limited by how fast you can SHIP.
  Elite change failure rate is NOT 0%.
  a build that fails in CI is NOT a change failure - it is the pipeline working.
```

```bash
# GitHub calculations                               (lines 91-162)
deployment frequency  gh api repos/O/R/deployments --paginate \
                        --jq '[.[] | select(.environment == "production")] | length'
                      #  --paginate or you silently under-count
                      #  environment is CASE-SENSITIVE -> wrong case = ZERO
                      #  proxy when no deployment records: count merged PRs to main

lead time             ((.mergedAt | fromdateiso8601) - (.createdAt | fromdateiso8601)) / 3600
                      #  fromdateiso8601 -> epoch SECONDS   /3600 = hours   /86400 = days
                      #  JavaScript Date subtraction -> MILLISECONDS  -> /(1000*60*60)
                      #  report the MEDIAN:  ... | sort | .[length/2 | floor]

change failure rate   deployment statuses state == "failure"/"error"
                      or  PR titles matching  test("revert|hotfix|rollback"; "i")

MTTR                  gh issue list --label "incident" --state closed
                      ((.closedAt | fromdateiso8601) - (.createdAt | fromdateiso8601)) / 3600
```

```text
AZURE DEVOPS ANALYTICS - the reporting layer        (lines 249-278)
  https://analytics.dev.azure.com/ORG/PROJECT/_odata/v4.0-preview
  $filter   WorkItemType eq 'User Story' and StateCategory eq 'Completed'
            and CompletedDate gt 2025-01-01T00:00:00Z
            ^ StateCategory (portable) NOT State (differs per process template)
            ^ word operators: gt ge lt le eq ne  -- never > <
  $apply    groupby((Area/AreaPath),aggregate(LeadTimeDays with average as AvgLeadTime,
                                              LeadTimeDays with max as MaxLeadTime,
                                              $count as Count))
            ^ SERVER-SIDE aggregation. this is why Analytics beats paging the REST API
  entity sets: WorkItems | PipelineRuns
  401 ->  extension not installed (org level)  OR  PAT missing Analytics (read)
          verify:  curl -o /dev/null -w "%{http_code}" .../_odata/v4.0-preview/$metadata   -> 200

DASHBOARD WIDGETS are FLOW metrics, not DORA        (lines 201-241)
  Sprint Burndown (currentIteration, storyPoints) | Velocity (numberOfSprints) | Cycle Time (last30Days)
  none of the four DORA metrics has a stock widget here
```

```yaml
# Deployment tracking                               (lines 289-298)
on:
  deployment_status:                       # payload has  .state
  workflow_run:
    workflows: ["Deploy to Production"]    # payload has  .conclusion
    types: [completed]
if: github.event.deployment_status.state == 'success'
    || github.event.workflow_run.conclusion == 'success'

# Weekly classification                             (lines 413-414, 449)
deploys >= 7 ? 'Elite' : deploys >= 1 ? 'High' : 'Medium'      # per week
avgHours < 1 ? 'Elite' : avgHours < 168 ? 'High' : 'Medium'    # 168 = one week in hours
# guard the empty week or the job divides by zero and gets switched off
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 05 |
| 38–43 | Re-read the trap index and the level table, then move on |
| 30–37 | Rewrite the four metrics and their Elite thresholds from memory, then retake |
| Below 30 | Redo Tasks 1, 2 and 6 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 04.

:::danger The two pairs

**Throughput** — deployment frequency and lead time. **Stability** — change failure rate and MTTR.
Elite means good at both, which is why every question hands you four numbers.

**And check the statistic before you believe the trend.** A mean over a small sample swings on one
outlier, and excluding the slow tail moves the number without moving the process.

The room went silent because there was no instrument — not because nobody was trying.

:::
