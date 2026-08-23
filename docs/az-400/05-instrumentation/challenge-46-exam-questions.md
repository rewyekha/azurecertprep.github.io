---
sidebar_position: 1.5
toc_max_heading_level: 2
title: "Challenge 46: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 46 — AZ-400 exam questions

**48 questions** built only from what Challenge 46 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-46.md`**.

:::danger Read this before you start

Domain 5 is only 5–10% of the exam, and it is the cheapest domain to score in — because almost every
question reduces to **detect, decide, act**.

**Detect** — a metric alert (`Http5xx > 50`) or a log alert (KQL over `exceptions`).
**Decide** — a threshold and a window. *How bad, for how long.*
**Act** — an action group: email, webhook, Azure Function.

And two facts do most of the work in this challenge.

**Annotations are pushed, not detected.** Application Insights never notices you deployed. The pipeline
must call the REST API and say so.

**Azure Monitor webhooks carry no authentication.** They cannot start an Azure DevOps pipeline
directly. Something in between — a Function or Logic App — has to authenticate.

The scenario at line 16 is what missing all of this costs: a memory leak live for **8 hours** because
nobody connected a rising error rate to the 2:15 PM deployment.

:::

---

# Section A — Multiple choice

---

## Q1

Contoso deploys five times daily and wants each deployment's impact visible on Application Insights
charts. What should they configure?

- A. Continuous profiling
- B. Deployment annotations created via the Application Insights REST API after each deployment
- C. Application Insights auto-detection of deployments
- D. Smart detection

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-46.md`:** lines **30** and **71–73**.

```bash
        az rest --method put \
          --url "https://management.azure.com$(appInsightsResourceId)/Annotations?api-version=2015-05-01" \
          --body "$ANNOTATION_PROPERTIES"
```

**Annotations are vertical markers on a time-series chart**, and they are the mechanism that answers
"did this deployment cause that?" at a glance.

**Why C is the trap, and it is the most tempting one.** Application Insights has no idea a deployment
happened — it sees telemetry from an application, not events from a pipeline. **The pipeline must push
the annotation.**

**Why the others fail** — profiling samples CPU and memory to find slow code paths, and smart detection
finds anomalies but neither marks *when* a release occurred, which is the correlation the scenario is
missing.

</details>

---

## Q2

A release must not proceed to production if Azure Monitor shows active critical alerts. Which feature
provides this?

- A. Branch policies
- B. Environment approvals
- C. Deployment gates using the "Query Azure Monitor alerts" gate type
- D. Pipeline triggers

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-46.md`:** lines **177–185**.

```text
2. Add gate: "Query Azure Monitor alerts"
   - Alert rules: alert-high-error-rate, alert-exception-spike
   - Filter: Fired
```

**A gate is an automated, repeatable check that re-evaluates on a schedule** until it passes or times
out. It differs from an approval in exactly that way.

**Why B is the near-miss worth separating.** An approval asks a **person**. A gate asks a **system** —
and the requirement here is about the state of alerts, which no human should be reading manually before
each release.

**Why A fails on target** — branch policies gate merging code, not deploying it (Challenge 41 Q4).

</details>

---

## Q3

Contoso wants automatic rollback if the error rate exceeds 5% within 10 minutes. What is the best
architecture?

- A. A developer watches the dashboard and rolls back manually
- B. An Azure Monitor alert triggers an action group webhook that invokes a rollback pipeline
- C. Smart detection with email notification
- D. A scheduled pipeline checking metrics hourly

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-46.md`:** lines **88–105** and **317–322**.

```bash
az monitor metrics alert create \
  --condition "total Http5xx > 50" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action ag-deployment-rollback
```

**Detect, decide, act — all three automated.** The alert detects, the threshold and window decide, the
action group acts.

**Why C is the closest wrong answer, and the distinction is precise.** Smart detection is genuinely
good at finding anomalies you did not think to write a rule for — and it **notifies**. It does not
invoke an action group to run anything.

**Why D fails on time.** An hourly check means up to 60 minutes of exposure. The scenario's whole
complaint is 8 hours of undetected damage.

</details>

---

## Q4

An action group's webhook targets an Azure DevOps pipeline REST API. The alert fires, the webhook is
sent, and the pipeline does not start. Why?

- A. The pipeline is disabled
- B. The Azure DevOps REST API requires authentication that the action group does not send
- C. Azure Monitor webhooks are rate-limited
- D. The pipeline can only be triggered manually

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-46.md`:** lines **409** and **425–432**.

**Azure Monitor sends a POST with an alert payload and no credentials.** The Azure DevOps API answers
with 401 and the alert records a successful delivery attempt, which is why this failure is so
confusing — the action group looks healthy.

**The fix is an intermediary** (line 425): an Azure Function or Logic App receives the unauthenticated
webhook, then authenticates outbound with a PAT or a managed identity.

**For GitHub the same pattern uses `repository_dispatch`:**

```bash
curl -X POST https://api.github.com/repos/contoso/webapp/dispatches \
  -H "Authorization: token $GITHUB_TOKEN" \
  -d '{"event_type":"deployment-health-alert"}'
```

**This is the highest-yield fact in the challenge**, because it appears as a Knowledge check, a Break
scenario, and a case study question.

</details>

---

## Q5

Deployment annotations do not appear on charts. What are the likely causes?

- A. The resource ID is wrong, the timestamp is not UTC ISO 8601, or the identity lacks write access
- B. Annotations take 24 hours to appear
- C. Annotations require the Premium tier
- D. The workspace is in a different region

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-46.md`:** lines **383** and **403**.

```text
**Fix:** Ensure the timestamp is in UTC ISO 8601 format and the service principal has
Contributor access to the Application Insights resource.
```

**Three causes, and the diagnosis for each is at lines 389–396.** Confirm the resource ID with
`az monitor app-insights component show --query id`, then GET the existing annotations to see whether
yours arrived at all.

**The timestamp format is the one that bites**, which is why every example in this challenge generates
it the same way:

```bash
$(date -u +%Y-%m-%dT%H:%M:%SZ)
```

**The `-u` is not optional.** Local time produces an annotation that lands hours away from the
deployment it marks — visible on the chart, and pointing at the wrong moment, which is worse than
missing.

</details>

---

## Q6

Why does the health check `sleep 120` before querying?

- A. To let the App Service warm up
- B. To let telemetry flow into Application Insights before querying
- C. To avoid rate limits
- D. To wait for the annotation to be indexed

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-46.md`:** lines **156–157** and **239–240**.

```bash
                # Wait for telemetry to flow
                sleep 120
```

**Application Insights ingestion is not instantaneous.** Query immediately after a deployment and you
read a window in which almost nothing has arrived — so `ERROR_COUNT` is 0, the check passes, and a
genuinely broken release is declared healthy.

**A false pass is worse than a false fail here**, because it consumes the one gate standing between the
deployment and the users.

**Why A is a real concern with a different remedy** — warm-up is handled by hitting the slot before
swapping (Challenge 26), not by sleeping in a monitoring step.

</details>

---

## Q7

What does the health check do when the error count exceeds the threshold?

- A. Logs a warning and continues
- B. Logs an error and exits non-zero, failing the stage
- C. Automatically swaps slots
- D. Creates an Azure Monitor alert

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-46.md`:** lines **168–171**.

```bash
                if [ "$ERROR_COUNT" -gt 50 ]; then
                  echo "##vso[task.logissue type=error]Error rate exceeds threshold. Triggering rollback."
                  exit 1
                fi
```

**`exit 1` is what actually fails the stage.** The `logissue` line makes the reason visible in the
pipeline summary; without the exit, it is a red message in a green run.

**Note that `type=error` and `exit 1` do different jobs** — one reports, one gates. The same distinction
as Challenge 43's `type=warning`, which reports without failing.

</details>

---

## Q8

What is the difference between a metric alert and a log (scheduled query) alert?

- A. Metric alerts evaluate platform metrics at high frequency; log alerts run a KQL query on a schedule
- B. Metric alerts are free
- C. Log alerts cannot use action groups
- D. They are the same with different names

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-46.md`:** lines **96–105** and **108–117**.

```bash
az monitor metrics alert create --condition "total Http5xx > 50" --evaluation-frequency 1m
az monitor scheduled-query create --condition-query ExceptionSpike="exceptions | where timestamp > ago(5m) | summarize count()" --evaluation-frequency 5m
```

**Metrics are pre-aggregated numeric series**, so evaluation is cheap and fast — 1-minute frequency here.
**Log alerts run an arbitrary KQL query**, which is far more expressive and correspondingly heavier — 5
minutes here.

**Choose by expressiveness, not by preference.** "HTTP 5xx count" is a platform metric. "Exceptions of
this type, from this component, excluding this known noisy one" needs KQL.

</details>

---

## Q9

What do `--window-size 5m` and `--evaluation-frequency 1m` mean together?

- A. Every minute, evaluate the condition over the last 5 minutes of data
- B. Evaluate every 5 minutes over 1 minute of data
- C. Wait 5 minutes then check once per minute for an hour
- D. Alert after 5 consecutive minutes above threshold

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-46.md`:** lines **101–102**.

**Window = how much data. Frequency = how often you look.** Overlapping windows are normal and
intended — each evaluation reconsiders the most recent 5 minutes.

**The trade-off is worth understanding rather than memorising.** A short window reacts fast and is
noisy; a long window is stable and slow. For deployment rollback you want **fast**, because every
minute of delay is user-visible damage — which is why frequency here is 1 minute rather than 5.

</details>

---

## Q10

Which action group receiver can run code in response to an alert?

- A. `email`
- B. `webhook`
- C. `azurefunction`
- D. `sms`

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-46.md`:** line **267**.

```bash
  --action azurefunction TriggerRollback ".../sites/func-contoso-ops" TriggerRollback "https://func-contoso-ops.azurewebsites.net/api/TriggerRollback" useCommonAlertSchema
```

**A webhook *sends* an HTTP request; a Function *runs your code*** — which is what lets it authenticate
onward to Azure DevOps or GitHub (Q4).

**`useCommonAlertSchema` matters more than it looks.** It normalises the payload across metric, log and
activity-log alerts, so one Function handles every alert type instead of parsing three different shapes.

</details>

---

## Q11

Which KQL computes the error percentage over time?

- A. `requests | summarize totalRequests = count(), failedRequests = countif(success == false) by bin(timestamp, 5m) | extend errorPercentage = (failedRequests * 100.0) / totalRequests`
- B. `requests | where success == false | count`
- C. `exceptions | summarize count() by bin(timestamp, 5m)`
- D. `requests | summarize avg(duration)`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-46.md`:** lines **298–303**.

```kusto
| summarize
    totalRequests = count(),
    failedRequests = countif(success == false)
    by bin(timestamp, 5m)
| extend errorPercentage = (failedRequests * 100.0) / totalRequests;
```

**`countif()` counts conditionally within the same summarize**, so you get both numbers in one pass and
can divide them.

**The `* 100.0` is deliberate** — integer division would truncate the ratio to 0 for any error rate
below 100%.

**Why B is the absolute count** used by the health check at line 163, which is a different question: a
count of 50 errors means something very different at 100 requests than at 100,000.

</details>

---

## Q12

Which KQL gives response-time percentiles?

- A. `requests | summarize p50 = percentile(duration, 50), p95 = percentile(duration, 95), p99 = percentile(duration, 99) by bin(timestamp, 5m)`
- B. `requests | summarize avg(duration)`
- C. `requests | top 10 by duration`
- D. `requests | count`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-46.md`:** lines **306–312**.

**Percentiles are what you monitor; averages are what hides the problem.** If 95% of requests take
100 ms and 5% take 30 seconds, the average looks acceptable and one user in twenty is having a terrible
time.

**P95 and P99 are the standard SLO measures** for exactly that reason, and `bin(timestamp, 5m)` turns
them into a time series you can render as a chart (line 312) and correlate against an annotation.

</details>

---

## Q13

What does the annotation's `Category` field carry in this challenge?

- A. `Deployment`
- B. `Release`
- C. `Build`
- D. `Custom`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-46.md`:** lines **65** and **233**.

```json
          "Category": "Deployment",
```

**The category is what lets a chart or query filter to deployment markers only**, rather than every
annotation anyone has ever written.

**And the `Properties` field is where the correlation data lives** (line 66): build number, branch,
commit ID. When the chart shows a step change at 2:15 PM, those properties name the commit — which is
the entire answer to the scenario at line 16.

</details>

---

## Q14

Which GitHub Actions trigger lets an external system start the rollback workflow?

- A. `repository_dispatch`
- B. `workflow_dispatch`
- C. `schedule`
- D. `push`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-46.md`:** lines **320–322**.

```yaml
on:
  repository_dispatch:
    types: [deployment-health-alert]
```

**`repository_dispatch` is triggered by an API call from outside GitHub**, which is exactly the shape of
an alert-driven rollback.

**Why B is the pair it is always confused with.** `workflow_dispatch` is the **manual** trigger — the
"Run workflow" button, and an API call made by a person or a workflow. `repository_dispatch` is for
**external systems**, and it carries a `client_payload` (line 432) with the alert details.

**The `types:` filter matters** — one workflow reacts to `deployment-health-alert` while others react to
different event types from the same endpoint.

</details>

---

## Q15

How does the rollback workflow revert the application?

- A. Redeploys the previous artifact from storage
- B. Swaps the production and staging slots
- C. Reverts the git commit and redeploys
- D. Restores an App Service backup

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-46.md`:** lines **349–353**.

```bash
          az webapp deployment slot swap \
            --name app-contoso-web \
            --resource-group rg-contoso-prod \
            --slot staging \
            --target-slot production
```

**A swap exchanges rather than copies**, so after the previous deployment's swap, staging holds the
**last known good version** — and swapping back restores it in seconds.

**That is the blue-green property from Challenge 25**, and it is why the rollback is fast enough to be
automated: no build, no artifact fetch, no deployment.

**Why C would take minutes and requires a working pipeline** — the wrong tool when users are seeing
errors right now.

</details>

---

## Q16

Why does the rollback workflow verify health after swapping?

- A. To confirm the rollback actually restored a healthy state
- B. To warm up the application
- C. To create an annotation
- D. To satisfy the environment approval

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-46.md`:** lines **357–366**.

```bash
          sleep 60
          HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://app-contoso-web.azurewebsites.net/health)
```

**Rollback is a deployment, and deployments can fail.** Assuming the swap fixed things is how an
incident goes from one broken version to two.

**Compare Challenge 38's sharper version of this check**, which compares the **reported version** rather
than just the status code — because during a swap the old instances can still answer 200 for a while.
Here the `sleep 60` is doing that job less precisely.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** properties does the deployment annotation carry for correlation? (Choose three.)

- A. Build number
- B. Branch name
- C. Commit ID
- D. Error count
- E. P95 latency
- F. Rollback status

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-46.md`:** line **66**.

```json
"Properties": "{\"BuildNumber\":\"$(Build.BuildNumber)\",\"Branch\":\"$(Build.SourceBranchName)\",\"CommitId\":\"$(Build.SourceVersion)\",\"ReleaseName\":\"$(Build.BuildNumber)\"}"
```

**These three answer "what changed?"** — which is the only question worth asking once a chart shows a
step change at a specific minute.

**Why D, E and F are the *metrics*, not the marker.** The annotation says a deployment happened and
what was in it; the telemetry says what happened next. Mixing them into one object would mean the
annotation could not be written until after the outcome was known.

</details>

---

## Q18

Which **three** appear in the GitHub Actions deployment workflow? (Choose three.)

- A. OIDC login with `id-token: write`
- B. Creating a deployment annotation via `az rest`
- C. A post-deployment health check that fails on error spikes
- D. A slot swap
- E. Creating a metric alert
- F. Publishing a SARIF file

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-46.md`:** lines **196–212**, **222–235**, **237–252**.

**Deploy, mark, verify** — the three steps that turn a deployment into an observable event.

**Why D is in the *rollback* workflow** (line 349), not this one. Keeping them separate is the design:
the deploy workflow detects and fails; the rollback workflow is triggered separately and acts.

**Why E is infrastructure configured once** (line 96), not something a deployment does every run.

</details>

---

## Q19

Which **two** are needed for an Azure Monitor alert to trigger an Azure DevOps pipeline? (Choose two.)

- A. An action group with a webhook or Azure Function receiver
- B. An intermediary that authenticates to Azure DevOps
- C. The pipeline set to `trigger: none`
- D. A service connection on the alert rule
- E. The alert rule granted Build Contributor

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-46.md`:** lines **88–93** and **425–432**.

**The action group delivers; the intermediary authenticates.** Neither is optional, and B is the half
people leave out.

**Why D and E are conceptually appealing and do not exist.** Alert rules have no identity in Azure
DevOps and no service connection — they are Azure resources that send HTTP requests. There is nothing to
grant a permission *to*.

</details>

---

## Q20

Which **two** are true of Azure DevOps deployment gates? (Choose two.)

- A. They re-evaluate on an interval until they pass or time out
- B. A minimum duration can be required before the gate passes
- C. They require a human approver
- D. They are configured in the YAML pipeline file
- E. They block merges to `main`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-46.md`:** lines **182–185**.

```text
   - Time between evaluations: 5 minutes
   - Timeout after: 30 minutes
   - Minimum duration: 10 minutes
```

**Minimum duration is the subtle one, and it is a real protection.** A gate that passes on its first
evaluation, seconds after deployment, has proved nothing — telemetry has not arrived yet (Q6). Requiring
10 minutes of *sustained* health is what makes the gate meaningful.

**Why C is the distinction from approvals** (Q2) and **why D is wrong on placement** — gates are
configured on the stage's pre-deployment conditions, not in YAML.

</details>

---

## Q21

Which **two** receiver types does `ag-deployment-events` use to reach chat tools? (Choose two.)

- A. A Teams webhook
- B. A Slack webhook
- C. An SMS receiver
- D. An email receiver
- E. A voice receiver

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-46.md`:** lines **265–266**.

```bash
  --action webhook teams-webhook "https://contoso.webhook.office.com/webhookb2/..." \
  --action webhook slack-webhook "https://hooks.slack.com/services/T00/B00/xxx"
```

**Both chat integrations are ordinary webhooks** — there is no dedicated "Teams receiver" in an action
group; the incoming-webhook URL is the whole integration.

**And one action group can hold several receivers of the same type**, which is what lets a single alert
reach two people by email and two channels by webhook simultaneously (lines 263–267).

</details>

---

## Q22

Which **two** distinguish an alert **action** from a deployment **gate**? (Choose two.)

- A. An alert action responds after a problem is detected in production
- B. A gate prevents progression before the next stage runs
- C. A gate sends email notifications
- D. An alert action is configured on the stage
- E. They are two names for the same mechanism

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-46.md`:** lines **96–105** and **177–185**.

**Before and after — that is the whole distinction.** A gate is preventive: nothing proceeds until the
condition is satisfied. An alert action is reactive: something has already gone wrong and this is the
response.

**A serious design has both**, and this challenge builds both: the gate at lines 177–185 stops a
release entering production while alerts are firing, and the alert at lines 96–105 catches what gets
through.

</details>

---

## Q23

Which **two** would improve the health check at lines 160–171? (Choose two.)

- A. Compare the error **rate** against total requests rather than an absolute count
- B. Compare the reported application version to confirm the new build is serving
- C. Reduce the sleep to 5 seconds so the check finishes sooner
- D. Remove `exit 1` so the pipeline does not fail
- E. Query a 24-hour window instead of 5 minutes

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-46.md`:** lines **163–168**, with the rate query at **298–303**.

**A is the flaw in the shipped check.** `ERROR_COUNT -gt 50` means something completely different at
100 requests than at 100,000 — and worse, it scales with traffic, so the same healthy release passes at
2 AM and fails at peak.

**B is Challenge 38's lesson applied here.** A low error count proves nothing if the old version is
still serving.

**Why C and E fail in opposite directions.** Five seconds is before telemetry arrives (Q6); 24 hours
dilutes a spike into invisibility and delays the answer far past the point of usefulness.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso deploys five times daily with no correlation between deployments and regressions.
A memory leak ran undetected for 8 hours. They need deployment impact visible immediately, releases
blocked while alerts are firing, and automated rollback when health degrades after deployment.

---

## Q24

**Proposed solution:** Create a deployment annotation via the Application Insights REST API in every
deployment, with build number, branch and commit in its properties. Add a validation stage that waits
for telemetry, queries the error rate, and fails on breach. Configure a metric alert on `Http5xx` with a
5-minute window evaluated every minute, wired to an action group whose Azure Function authenticates to
GitHub and raises a `repository_dispatch`. The rollback workflow swaps slots and verifies health. Add a
"Query Azure Monitor alerts" gate on the production stage.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-46.md`:** lines **60–73**, **142–171**, **96–105**, **267**, **320–322**, **349–366**,
**177–185**.

| Requirement | Mechanism |
|---|---|
| Correlate deployment with metrics | Annotation carrying commit and build |
| Catch a bad release before users do | Validation stage querying Application Insights |
| Catch what the validation misses | Metric alert, 5m window, 1m frequency |
| Alert can actually start a pipeline | Function intermediary that authenticates |
| Fast rollback | Slot swap, then health verification |
| Do not release into a live incident | Azure Monitor alerts gate |

**The Function is the load-bearing piece.** Every other element is standard; without it the alert fires
into a 401 and nothing happens (Q4).

</details>

---

## Q25

**Proposed solution:** Enable Application Insights smart detection and subscribe the ops team to email
alerts. Add a scheduled pipeline that checks error rates every hour. When someone notices a problem,
they run the rollback pipeline manually.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures, and together they reproduce the incident exactly.**

**Smart detection notifies; it does not act** (Q3). Useful for anomalies nobody wrote a rule for, and it
starts no rollback.

**Hourly checks mean up to an hour of exposure per cycle**, and the memory leak in the scenario built
gradually — an hourly snapshot is precisely the cadence that misses a slow ramp.

**No annotations means no correlation**, which was the actual root cause. The error rate was visible;
nothing tied it to the 2:15 PM deployment.

**And "someone notices" is the control that already failed** — for 8 hours.

</details>

---

## Q26

**Proposed solution:** Create deployment annotations. Add a validation stage that waits for telemetry and
queries the error rate. Configure a metric alert with a 5-minute window evaluated every minute. Wire the
action group's webhook **directly** to the Azure DevOps pipeline runs REST API so rollback triggers
without extra components. Add an Azure Monitor alerts gate on production.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the webhook target — and compare with Q24, which is otherwise identical.

**Azure Monitor action groups send webhooks without authentication headers.** The Azure DevOps pipeline
runs API requires a PAT or OAuth token, so the request returns 401 and the rollback never starts.

**What makes this dangerous is that nothing looks broken.** The alert fires, the action group records
the webhook as sent, and the dashboard shows an alert with an action attached. The failure is only
visible in the absence of a pipeline run — which nobody checks until the next incident.

**Line 92 in the challenge shows exactly this URL** being configured, and Break scenario 2 at line 407
is the challenge telling you it does not work:

```bash
  --action webhook rollback-webhook "https://dev.azure.com/contoso/ContosoWeb/_apis/pipelines/15/runs?api-version=7.1-preview.1"
```

**The fix is an intermediary** (line 425) — Azure Function or Logic App — that holds the credential.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — annotations

| # | Statement | Answer |
|---|---|---|
| 1 | Application Insights detects deployments automatically |  |
| 2 | The annotation timestamp must be UTC ISO 8601 |  |
| 3 | The identity creating annotations needs write access to the resource |  |
| 4 | Annotation properties can carry the commit ID |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Application Insights detects deployments automatically | **No** |
| 2 | The annotation timestamp must be UTC ISO 8601 | **Yes** |
| 3 | The identity creating annotations needs write access to the resource | **Yes** |
| 4 | Annotation properties can carry the commit ID | **Yes** |

**In `challenge-46.md`:** lines **30**, **403**, **403**, **66**.

Row 1 is the misconception the whole task exists to correct. **The pipeline pushes the annotation.**

Rows 2 and 3 are Break scenario 1's two causes, and both produce the same symptom: nothing on the chart.

</details>

---

## Q28 — alerts

| # | Statement | Answer |
|---|---|---|
| 1 | Metric alerts can evaluate as often as every minute |  |
| 2 | Log alerts run a KQL query on a schedule |  |
| 3 | An action group can hold email, webhook and Function receivers together |  |
| 4 | Action group webhooks include authentication headers |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Metric alerts can evaluate as often as every minute | **Yes** |
| 2 | Log alerts run a KQL query on a schedule | **Yes** |
| 3 | An action group can hold email, webhook and Function receivers together | **Yes** |
| 4 | Action group webhooks include authentication headers | **No** |

**In `challenge-46.md`:** lines **102**, **113–114**, **263–267**, **409**.

Row 4 is the fact this challenge tests three separate times. **No credentials, therefore no direct
pipeline trigger.**

</details>

---

## Q29 — gates and validation

| # | Statement | Answer |
|---|---|---|
| 1 | A gate re-evaluates until it passes or times out |  |
| 2 | A minimum duration can be required before a gate passes |  |
| 3 | Querying telemetry immediately after deployment gives reliable results |  |
| 4 | `exit 1` in a health check fails the stage |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A gate re-evaluates until it passes or times out | **Yes** |
| 2 | A minimum duration can be required before a gate passes | **Yes** |
| 3 | Querying telemetry immediately after deployment gives reliable results | **No** |
| 4 | `exit 1` in a health check fails the stage | **Yes** |

**In `challenge-46.md`:** lines **182–185**, **156–157**, **170**.

Row 3 is why both workflows sleep 120 seconds, and row 2 is the same protection expressed as a gate
setting rather than a script.

</details>

---

## Q30 — rollback

| # | Statement | Answer |
|---|---|---|
| 1 | `repository_dispatch` lets an external system start a workflow |  |
| 2 | A slot swap restores the previous version quickly |  |
| 3 | Rollback needs no verification because it restores known-good code |  |
| 4 | An Azure Function can authenticate onward to GitHub or Azure DevOps |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `repository_dispatch` lets an external system start a workflow | **Yes** |
| 2 | A slot swap restores the previous version quickly | **Yes** |
| 3 | Rollback needs no verification because it restores known-good code | **No** |
| 4 | An Azure Function can authenticate onward to GitHub or Azure DevOps | **Yes** |

**In `challenge-46.md`:** lines **320–322**, **349–353**, **357–366**, **425–432**.

Row 3 is the assumption that turns one incident into two. **A rollback is a deployment** — verify it
(Q16).

</details>

---

# Section E — Drag and drop

---

## Q31

Match each requirement to its mechanism.

| Requirement | Mechanism |
|---|---|
| See when a deployment happened on a metrics chart |  |
| Block a release while critical alerts are firing |  |
| Detect a 5xx spike within a minute |  |
| Detect an exception pattern KQL can express |  |
| Start a rollback pipeline from an alert |  |
| Fail the deployment stage on a bad health check |  |

**Options:** Action group → Azure Function → dispatch · Deployment annotation · `exit 1` in the validation step · Log (scheduled query) alert · Metric alert, 1-minute frequency · Query Azure Monitor alerts gate

<details>
<summary>Show answer</summary>

| Requirement | Mechanism |
|---|---|
| See when a deployment happened on a metrics chart | **Deployment annotation** |
| Block a release while critical alerts are firing | **Query Azure Monitor alerts gate** |
| Detect a 5xx spike within a minute | **Metric alert, 1-minute frequency** |
| Detect an exception pattern KQL can express | **Log (scheduled query) alert** |
| Start a rollback pipeline from an alert | **Action group → Azure Function → dispatch** |
| Fail the deployment stage on a bad health check | **`exit 1` in the validation step** |

**In `challenge-46.md`:** lines **71–73**, **178–181**, **96–105**, **108–117**, **267** with **425–432**,
**170**.

**Rows 3 and 4 are chosen by expressiveness.** A pre-aggregated platform metric → metric alert. Anything
needing a query → log alert.

</details>

---

## Q32

Match each timing value to its meaning.

| Value | Meaning |
|---|---|
| `--window-size 5m` |  |
| `--evaluation-frequency 1m` |  |
| `Time between evaluations: 5 minutes` |  |
| `Timeout after: 30 minutes` |  |
| `Minimum duration: 10 minutes` |  |
| `sleep 120` |  |

**Options:** How long health must hold before the gate passes · How much data each evaluation examines · How often the condition is evaluated · How often the gate re-checks · Waiting for telemetry to arrive before querying · When the gate gives up and fails

<details>
<summary>Show answer</summary>

| Value | Meaning |
|---|---|
| `--window-size 5m` | **How much data each evaluation examines** |
| `--evaluation-frequency 1m` | **How often the condition is evaluated** |
| `Time between evaluations: 5 minutes` | **How often the gate re-checks** |
| `Timeout after: 30 minutes` | **When the gate gives up and fails** |
| `Minimum duration: 10 minutes` | **How long health must hold before the gate passes** |
| `sleep 120` | **Waiting for telemetry to arrive before querying** |

**In `challenge-46.md`:** lines **101**, **102**, **183**, **184**, **185**, **157**.

**Minimum duration and `sleep 120` solve the same problem in two places** — one in the gate's
configuration, one in the script. Both exist because **a check that runs too early always passes**.

</details>

---

## Q33

Arrange the steps of a monitored deployment with automated rollback.

**Items:** Alert fires and calls the action group · Deploy the application · Create the deployment
annotation · Wait for telemetry, then query error rate · Function authenticates and raises
`repository_dispatch` · Rollback workflow swaps slots and verifies health

<details>
<summary>Show answer</summary>

### Answer

1. Deploy the application — line **214**
2. Create the deployment annotation — line **222**
3. Wait for telemetry, then query error rate — lines **239–246**
4. Alert fires and calls the action group — lines **96–105**
5. Function authenticates and raises `repository_dispatch` — lines **267**, **430–432**
6. Rollback workflow swaps slots and verifies health — lines **349–366**

**Steps 1 and 2 are adjacent for a reason.** The annotation must be written **immediately after** the
deployment, because its whole value is marking the correct minute on the chart.

**And steps 3 and 4 are two independent safety nets, not a sequence.** The in-pipeline check catches a
fast, obvious failure within minutes. The alert catches the slow one — the memory leak at line 16 that
took hours to become visible and would have passed any two-minute check.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Annotations missing from charts |  |
| Annotation appears hours from the deployment |  |
| Alert fires, pipeline never runs |  |
| Health check always passes |  |
| Gate passes immediately after deployment |  |

**Options:** No minimum duration configured · Queried before telemetry arrived, or an absolute threshold · Timestamp generated in local time · Webhook to an API needing authentication · Wrong resource ID, non-UTC timestamp, or no write access

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| Annotations missing from charts | **Wrong resource ID, non-UTC timestamp, or no write access** |
| Annotation appears hours from the deployment | **Timestamp generated in local time** |
| Alert fires, pipeline never runs | **Webhook to an API needing authentication** |
| Health check always passes | **Queried before telemetry arrived, or an absolute threshold** |
| Gate passes immediately after deployment | **No minimum duration configured** |

**In `challenge-46.md`:** lines **383**, **64**, **409**, **157** with **168**, **185**.

**Three of these five fail silently and look like success**, which is the theme of this challenge:
missing annotations, an unfired rollback, and a health check that passes on empty data all leave a green
pipeline behind them.

</details>

---

## Q35

Match each receiver to what it does.

| Receiver | Does |
|---|---|
| `email` |  |
| `webhook` |  |
| `azurefunction` |  |
| Teams/Slack incoming webhook URL |  |
| `useCommonAlertSchema` |  |

**Options:** Normalises the payload across alert types · Notifies a person · Posts into a channel — still a `webhook` receiver · Runs code that can authenticate onward · Sends an unauthenticated HTTP POST

<details>
<summary>Show answer</summary>

| Receiver | Does |
|---|---|
| `email` | **Notifies a person** |
| `webhook` | **Sends an unauthenticated HTTP POST** |
| `azurefunction` | **Runs code that can authenticate onward** |
| Teams/Slack incoming webhook URL | **Posts into a channel — still a `webhook` receiver** |
| `useCommonAlertSchema` | **Normalises the payload across alert types** |

**In `challenge-46.md`:** lines **263–267**.

**The middle two rows are the pair that decides Q4, Q26 and Q44.** A webhook reaches anything that
accepts anonymous POSTs — a chat channel does. **An API that requires a token does not**, and that is
where the Function goes.

</details>

---

# Section F — Hot area

---

## Q36

```bash
az rest --method [BLANK 1] \
  --url "https://management.azure.com$(appInsightsResourceId)/[BLANK 2]?api-version=2015-05-01" \
  --body "$ANNOTATION_PROPERTIES"
```

- **BLANK 1:** `put` / `get` / `patch` / `delete`
- **BLANK 2:** `Annotations` / `Events` / `Metrics` / `Deployments`

<details>
<summary>Show answer</summary>

### Answer: `put`, `Annotations`

**In `challenge-46.md`:** lines **71–72**.

**`put` because the annotation carries its own `Id`** (line 62, `$(Build.BuildId)`), so re-running the
step for the same build updates rather than duplicates.

**Why `Deployments` is the intuitive wrong answer** — there is no such endpoint. The Application
Insights resource exposes `Annotations`, and "deployment" is the **Category** you put inside one
(line 65).

</details>

---

## Q37

```bash
"EventTime": "$([BLANK 1] [BLANK 2] +%Y-%m-%dT%H:%M:%SZ)"
```

- **BLANK 1:** `date` / `time` / `now` / `timestamp`
- **BLANK 2:** `-u` / `-l` / `-R` / *(nothing)*

<details>
<summary>Show answer</summary>

### Answer: `date`, `-u`

**In `challenge-46.md`:** line **64**.

```bash
"EventTime": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

**`-u` forces UTC**, and the trailing `Z` in the format string asserts UTC to whatever reads it. Omit
`-u` and you produce a local-time value **labelled as UTC** — an annotation that lands hours away from
the deployment it marks.

**That combination is the exam's target**, because the string still *looks* correct.

</details>

---

## Q38

```bash
az monitor metrics alert create \
  --name "alert-high-error-rate" \
  --condition "total [BLANK 1] > 50" \
  --window-size [BLANK 2] \
  --evaluation-frequency [BLANK 3] \
  --action ag-deployment-rollback \
  --severity 1
```

Requirement: detect a 5xx spike quickly after a deployment.

- **BLANK 1:** `Http5xx` / `Http4xx` / `Requests` / `CpuTime`
- **BLANK 2:** `5m` / `1h` / `24h` / `1m`
- **BLANK 3:** `1m` / `5m` / `15m` / `1h`

<details>
<summary>Show answer</summary>

### Answer: `Http5xx`, `5m`, `1m`

**In `challenge-46.md`:** lines **100–102**.

**5xx is server-side failure; 4xx is client error** — a spike in 404s usually means a bad link, not a
bad deployment.

**And the window is larger than the frequency on purpose.** Looking every minute at the last five
minutes smooths single-minute noise while still reacting within about a minute of a genuine spike.

</details>

---

## Q39

```yaml
on:
  [BLANK 1]:
    types: [deployment-health-alert]
```

Requirement: an Azure Function must be able to start this workflow when an alert fires.

- **BLANK 1:** `repository_dispatch` / `workflow_dispatch` / `workflow_run` / `schedule`

<details>
<summary>Show answer</summary>

### Answer: `repository_dispatch`

**In `challenge-46.md`:** lines **320–322**.

**`repository_dispatch` is the external-system trigger**, invoked with a POST to
`/repos/{owner}/{repo}/dispatches` (line 430) carrying an `event_type` and a `client_payload`.

**`workflow_dispatch` is the manual trigger** — the button in the UI. It can be called by API too,
which is what makes it a credible distractor, but `repository_dispatch` is the one designed for this and
the one that carries the alert details.

</details>

---

## Q40

```bash
ERROR_COUNT=$(az monitor app-insights query \
  --app ai-contoso-webapp \
  --resource-group rg-contoso-prod \
  --analytics-query "requests | where timestamp > [BLANK 1] | where success == [BLANK 2] | count" \
  --query "[BLANK 3]" -o tsv)
```

- **BLANK 1:** `ago(5m)` / `now()` / `datetime(2026-01-01)` / `startofday(now())`
- **BLANK 2:** `false` / `true` / `0` / `null`
- **BLANK 3:** `tables[0].rows[0][0]` / `value` / `results[0]` / `count`

<details>
<summary>Show answer</summary>

### Answer: `ago(5m)`, `false`, `tables[0].rows[0][0]`

**In `challenge-46.md`:** lines **163–164**.

**`success == false` is the failed-request filter** in the `requests` table — not a status-code
comparison, which is what makes it easy to mis-write.

**And `tables[0].rows[0][0]` is how you extract a single scalar** from the KQL result shape: first
table, first row, first column. The exam shows this exact JMESPath because writing it is the difference
between having run the lab and having read about it.

</details>

---

## Q41

```bash
az monitor action-group create \
  --name ag-deployment-events \
  --short-name DeployEvt \
  --action [BLANK 1] TriggerRollback ".../sites/func-contoso-ops" TriggerRollback \
      "https://func-contoso-ops.azurewebsites.net/api/TriggerRollback" [BLANK 2]
```

- **BLANK 1:** `azurefunction` / `webhook` / `logicapp` / `automationrunbook`
- **BLANK 2:** `useCommonAlertSchema` / `useSimpleSchema` / `raw` / `v2`

<details>
<summary>Show answer</summary>

### Answer: `azurefunction`, `useCommonAlertSchema`

**In `challenge-46.md`:** line **267**.

**`azurefunction` runs code**, which is what lets it hold a credential and call an authenticated API.

**`useCommonAlertSchema` normalises the payload** so one Function handles metric alerts, log alerts and
activity-log alerts identically. Without it, each alert type arrives in its own shape and the Function
needs three parsers — and breaks whenever a new alert type is added.

</details>

---

# Section G — Case study

## Case study: Contoso deployment observability

### Background

Contoso Ltd deploys its flagship web application **five times daily**. The operations team has **no
correlation between deployments and performance regressions**. Last week a deployment introduced a
**memory leak that went undetected for 8 hours**, because nobody connected the rising error rate to the
**2:15 PM deployment**.

### Requirements

**Correlation**

- Every deployment must be visible on Application Insights charts at the minute it occurred
- The marker must identify the build, branch and commit
- Response-time percentiles and error rate must be viewable against those markers

**Detection**

- A 5xx spike must be detected within about a minute
- Exception patterns that require a query must also be detectable
- Detection must work for slow degradations, not only immediate failures

**Response**

- A bad deployment must fail its own pipeline before reaching users where possible
- An alert must be able to trigger a rollback with no human involved
- Rollback must be fast and must be verified
- No release may proceed into production while critical alerts are firing

---

## Q42

How should deployments be made visible on Application Insights charts?

- A. Create a deployment annotation via the Annotations REST API in the deployment job, with build,
  branch and commit in its properties
- B. Enable smart detection
- C. Tag the App Service with the build number
- D. Write the build number to the application log

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-46.md`:** lines **60–73** and **222–235**.

**Only annotations render as markers on the metric charts** — which is the specific thing the operations
team was missing.

**Why D is the workaround teams reach for, and why it falls short.** A log line is searchable but it is
not on the chart. To use it you must already suspect a deployment and go looking, which is exactly the
step that did not happen for 8 hours.

**Why C tags a resource, not a moment in time.** A tag tells you the current build; it carries no
history and appears on no chart.

</details>

---

## Q43

Which **two** detect problems, and how do they differ? (Choose two.)

- A. A metric alert on `Http5xx` with a 5-minute window evaluated every minute
- B. A log alert running KQL over the `exceptions` table
- C. Smart detection
- D. A daily summary email
- E. Manual dashboard review

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-46.md`:** lines **96–105** and **108–117**.

**Metric alerts are fast and cheap over pre-aggregated numeric series; log alerts are expressive and
slower.** That is the whole trade, and it is why Contoso needs both: `Http5xx` is a platform metric, and
"exceptions of this shape in this component" is not.

**Why C is a genuinely useful capability that fails the requirement.** Smart detection finds anomalies
nobody wrote a rule for — and it notifies rather than acting (Q3), so it cannot drive rollback.

</details>

---

## Q44

How should an alert trigger the rollback?

- A. Action group → Azure Function → authenticated `repository_dispatch` → rollback workflow
- B. Action group webhook pointing directly at the Azure DevOps pipeline runs API
- C. Action group email to the on-call engineer, who triggers rollback
- D. A scheduled workflow polling for active alerts

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-46.md`:** lines **267**, **320–322**, **425–432**.

```bash
curl -X POST https://api.github.com/repos/contoso/webapp/dispatches \
  -H "Authorization: token $GITHUB_TOKEN" \
  -d '{"event_type":"deployment-health-alert","client_payload":{"alert":"high-error-rate"}}'
```

**The Function exists to hold the credential.** That is its entire job in this architecture.

**Why B is the design almost everyone builds first**, and Break scenario 2 exists because of it. The
webhook is delivered, the API returns 401, and the action group reports success — a silent failure that
is only discovered during the next incident.

**Why C reintroduces the human latency** the requirement exists to remove, and **why D adds polling
delay** on top of the alert's own detection time.

</details>

---

## Q45

Which **two** protect the deployment before it reaches users? (Choose two.)

- A. A validation stage that waits for telemetry and fails on a breached threshold
- B. A "Query Azure Monitor alerts" gate on the production stage
- C. A metric alert
- D. A deployment annotation
- E. A rollback workflow

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-46.md`:** lines **142–171** and **177–185**.

**Both are *preventive*: they stop progression.** The validation stage fails the release; the gate
refuses to start it.

**Why C and E are the *reactive* half.** They are essential and they operate **after** the deployment is
live — the layered design in Q24 needs both halves.

**Why D is neither.** An annotation makes a problem **diagnosable**; it prevents and blocks nothing.

</details>

---

## Q46

How should the rollback restore service, and what must follow?

- A. Swap production with staging, then verify health and notify the team
- B. Rebuild from the previous commit and redeploy
- C. Restore an App Service backup
- D. Scale out to absorb the errors

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-46.md`:** lines **349–374**.

**A swap is near-instantaneous because staging already holds the previous version** — no build, no
artifact fetch, no deployment (Q15).

**Verification is not optional** (line 357), and neither is notification (line 368). An automated
rollback that nobody is told about means the team finds a mysteriously reverted production at some later
point, with the failed deployment uninvestigated.

**Why B is minutes rather than seconds**, and it needs a functioning pipeline at the worst possible
moment. **Why D treats the symptom** — a memory leak is not fixed by adding instances, it is spread
across more of them.

</details>

---

## Q47

Three months after implementation, the operations team notices annotations have stopped appearing for
one of the four deployment pipelines, though that pipeline reports success. The team recently moved that
pipeline to a new service connection scoped to a resource group containing only the App Service.

What happened, and what is the fix?

- A. The new service connection's identity has no write access to the Application Insights resource, so
  the annotation call fails — grant it access, or widen the scope to include the Application Insights
  resource group
- B. The annotation API version is deprecated
- C. The timestamp format changed
- D. Application Insights stopped accepting annotations from that region

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-46.md`:** lines **383** and **403**.

**"Pipeline reports success" is the diagnostic detail, and it points straight at the cause.** The
annotation step runs `az rest`, and depending on how the step is written a failed call can leave the
task green — so the only visible symptom is a chart with no markers.

**Scoping a service connection to the App Service's resource group is good practice** (Challenge 41) and
it silently removed a permission the pipeline depended on. **Application Insights is frequently in a
different resource group from the app it monitors**, and the annotation call is an ARM write against
*that* resource.

**The fix is the narrower of the two options where possible:** grant that identity a role on the
Application Insights resource specifically, rather than widening the connection's scope back out.

**And the durable lesson:** add `--verbose` or an explicit exit-code check to the annotation step, so a
403 fails loudly instead of leaving a green pipeline and an empty chart.

</details>

---

## Q48

Six months later, the same class of incident recurs — a slow memory leak — but this time it is caught in
**11 minutes** instead of 8 hours.

Walk through what caught it, and explain what the design actually changed.

- A. The metric alert detected the sustained 5xx rise, the action group's Function raised a dispatch,
  the rollback workflow swapped slots and verified health — and the annotation on the chart named the
  commit, so the fix took minutes rather than a day
- B. The validation stage caught it during deployment
- C. Smart detection emailed the team
- D. The gate blocked the release

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-46.md`:** lines **96–105**, **267**, **349–366**, **66**.

**Why B is specifically wrong, and this is the point of the question.** The validation stage sleeps 120
seconds and queries a 5-minute window (lines 239–246). **A memory leak does not manifest in two
minutes** — it needs traffic and time. The in-pipeline check would have passed, exactly as it did in the
original incident.

**What caught it was the layer that keeps watching after the pipeline goes green.** The alert evaluates
every minute over a rolling 5-minute window, indefinitely — so a degradation that takes ten minutes to
become visible is detected roughly ten minutes in, not eight hours in.

**And the annotation is what made the *fix* fast rather than the detection.** Detection told Contoso
*something* was wrong; the marker on the chart, carrying `BuildNumber`, `Branch` and `CommitId` (line
66), told them **which change** — turning a day of bisecting into a glance.

**What the design actually changed, stated plainly:** the original system could see the error rate the
whole time. **Nothing was missing from the data; what was missing was the link between the data and the
change, and anything that acted on it without a human.** That is the sentence to give any exam case
study about monitoring in a DevOps context — instrumentation is not about collecting more, it is about
**correlating** and **responding**.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Application Insights auto-detects deployments** | Q1, Q27, Q42 | It does not. The pipeline pushes the annotation |
| **Action group webhook straight to a pipeline API** | Q4, Q19, Q26, Q44 | No auth headers. Needs a Function or Logic App |
| **Smart detection expected to trigger actions** | Q3, Q25, Q43 | It notifies. It does not invoke |
| **Querying telemetry immediately after deploy** | Q6, Q23, Q29 | Ingestion lag. A check that runs too early always passes |
| **Absolute error count as a threshold** | Q23 | Scales with traffic. Use a rate |
| **Local time in `EventTime`** | Q5, Q34, Q37 | `date -u`. A wrong marker is worse than none |
| **`workflow_dispatch` for external triggers** | Q14, Q39 | `repository_dispatch` carries `client_payload` |
| **Window and frequency confused** | Q9, Q32, Q38 | Window = how much data. Frequency = how often |
| **Gate mistaken for approval** | Q2, Q20 | A gate asks a system, on a schedule |
| **No minimum duration on a gate** | Q20, Q34 | It passes before telemetry exists |
| **Rollback assumed successful** | Q16, Q30, Q46 | A rollback is a deployment. Verify it |
| **In-pipeline check expected to catch slow degradation** | Q48 | Two minutes cannot see a memory leak |
| **Annotation step failing silently** | Q47 | A 403 can leave the task green and the chart empty |
| **Averages instead of percentiles** | Q12 | P95/P99. An average hides the worst 5% |

---

# What to memorise

**In `challenge-46.md`:** lines **60–73**, **96–117**, **177–185**, **255–267**.

```text
DETECT -> DECIDE -> ACT
  detect   metric alert (platform metric, cheap, 1m frequency)
           log alert    (KQL, expressive, 5m frequency)
  decide   --window-size  = HOW MUCH DATA        --evaluation-frequency = HOW OFTEN YOU LOOK
           window > frequency  -> overlapping windows, smooths noise, still reacts in ~1 min
  act      action group: email | webhook | azurefunction

TWO FACTS THAT ANSWER HALF THIS CHALLENGE
  1. Application Insights NEVER detects a deployment. The pipeline PUSHES an annotation.
  2. Action group webhooks carry NO AUTHENTICATION -> cannot start an Azure DevOps/GitHub pipeline.
     Use an Azure Function (or Logic App) that authenticates onward.

PREVENT vs REACT
  prevent  validation stage (exit 1)  |  "Query Azure Monitor alerts" GATE on the stage
  react    metric/log alert -> action group -> Function -> dispatch -> rollback
  A slow degradation is caught by the REACTIVE layer. A 2-minute in-pipeline check cannot see it.
```

```bash
# Deployment annotation                             (lines 60-73)
az rest --method put \
  --url "https://management.azure.com<appInsightsResourceId>/Annotations?api-version=2015-05-01" \
  --body '{
    "Id": "<build id>",
    "AnnotationName": "Release <build number>",
    "EventTime": "'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'",     # -u = UTC. NOT optional
    "Category": "Deployment",
    "Properties": "{\"BuildNumber\":..,\"Branch\":..,\"CommitId\":..}"   # WHAT CHANGED
  }'
# fails silently -> no markers. causes: wrong resource ID | non-UTC time | no WRITE access
# App Insights is often in a DIFFERENT resource group from the app -> scope the connection accordingly
```

```bash
# Alerts                                            (lines 96-117)
az monitor metrics alert create --condition "total Http5xx > 50" \
  --window-size 5m --evaluation-frequency 1m --action ag-deployment-rollback --severity 1
#   Http5xx = server failure.  Http4xx = client error (bad links, not bad deploys)

az monitor scheduled-query create \
  --condition "count 'ExceptionSpike' > 100" \
  --condition-query ExceptionSpike="exceptions | where timestamp > ago(5m) | summarize count()" \
  --evaluation-frequency 5m --window-size 5m

# Action group                                      (lines 259-267)
--action email <name> <address>
--action webhook <name> <url>            # anonymous POST - fine for Teams/Slack, NOT for pipeline APIs
--action azurefunction <name> <function-resource-id> <function-name> <url> useCommonAlertSchema
#   useCommonAlertSchema -> one payload shape for metric/log/activity alerts -> one parser
```

```yaml
# In-pipeline validation                            (lines 155-172)
  - script: |
      sleep 120                                   # telemetry ingestion lag - or the check always passes
      ERROR_COUNT=$(az monitor app-insights query --app ai-contoso-webapp -g rg-contoso-prod \
        --analytics-query "requests | where timestamp > ago(5m) | where success == false | count" \
        --query "tables[0].rows[0][0]" -o tsv)
      if [ "$ERROR_COUNT" -gt 50 ]; then
        echo "##vso[task.logissue type=error]Error rate exceeds threshold."
        exit 1                                    # exit 1 FAILS the stage. logissue only reports
      fi
# better: compare a RATE, and compare the reported VERSION (Challenge 38)

# Rollback, triggered externally                    (lines 320-366)
on:
  repository_dispatch:                            # external systems. workflow_dispatch = manual button
    types: [deployment-health-alert]
steps:
  - az webapp deployment slot swap --slot staging --target-slot production   # seconds, no rebuild
  - curl .../health                                # VERIFY. a rollback is a deployment
  - notify the team                                # or production silently reverts and nobody investigates
```

```text
GATES  (Release pipeline > Stage > Pre-deployment conditions > Gates)   lines 177-185
  gate type: "Query Azure Monitor alerts"   alert rules: <names>   filter: Fired
  Time between evaluations  5 min      how often it re-checks
  Timeout after            30 min      when it gives up and fails
  Minimum duration         10 min      health must HOLD - stops it passing before telemetry exists
  gate = asks a SYSTEM, repeatedly.  approval = asks a PERSON, once.

KQL WORTH KNOWING                                   (lines 298-312)
error rate    requests | summarize total=count(), failed=countif(success==false) by bin(timestamp,5m)
              | extend errorPercentage = (failed * 100.0) / total     # 100.0 or integer division truncates
percentiles   requests | summarize p50=percentile(duration,50), p95=percentile(duration,95),
                                   p99=percentile(duration,99) by bin(timestamp,5m) | render timechart
              monitor PERCENTILES, not averages - an average hides the worst 5%
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 47 |
| 38–43 | Re-read the trap index and the two facts, then move on |
| 30–37 | Rewrite detect-decide-act and the window/frequency rule from memory, then retake |
| Below 30 | Redo Tasks 1, 2 and 7 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 46.

:::danger The two facts

**Application Insights never detects a deployment.** The pipeline pushes the annotation, in UTC, with
the commit in its properties.

**Action group webhooks carry no credentials.** They cannot start a pipeline. Put an Azure Function in
between.

Contoso's 8-hour incident was never a data problem — the error rate was on the chart the whole time.

:::
