---
sidebar_position: 5.5
toc_max_heading_level: 2
title: "Challenge 50: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 50 — AZ-400 exam questions

**48 questions** built only from what Challenge 50 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-50.md`**.

:::danger Read this before you start

Performance analysis questions are almost always **"where is the time going?"**, and the answer follows
one rule.

**Read the trace from the bottom.** A service is slow because it is **waiting** for something slower.
Frontend 50 ms → Gateway 30 ms → Order Service 4500 ms → Payment Service timing out means the problem is
**Payment**, not Order. Order is just holding the line.

**Subtract the dependency from the request.** If a request takes 1500 ms and its SQL dependency takes
1200 ms, your code took 300 ms. **The database is the answer**, and CPU, network and sampling are
distractions.

And two facts about the last task, which is the one most candidates have never touched.

**Error budget = 100% − SLO.** At 99.9% over 30 days, the budget is 0.1% of requests.
**Burn rate is the ratio** of how fast you are spending it. Burn rate 1.0 means you exhaust the budget
exactly at the end of the window; 2.0 means halfway through.

The scenario at line 18 is a 2:30 PM deployment, users reporting "slow", and a decision to make between
rollback and hotfix.

:::

---

# Section A — Multiple choice

---

## Q1

After a deployment, average response time rose from 200 ms to 1500 ms. The dependency list shows SQL
calls went from 50 ms to 1200 ms. What should you investigate first?

- A. Web server CPU utilisation
- B. Network latency between App Service and SQL Database
- C. SQL query performance — new slow queries or missing indexes
- D. The Application Insights sampling configuration

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-50.md`:** lines **62–71**.

```kql
dependencies
| summarize avgDuration = avg(duration), p95Duration = percentile(duration, 95),
            callCount = count(), failRate = ...
    by target, type, name
| order by p95Duration desc
```

**Do the subtraction: 1200 of the 1500 ms is the dependency.** Your code accounts for 300 ms, and 250 ms
of that was there before. The bottleneck is not ambiguous.

**Why A is the reflex answer and the wrong one.** High CPU would slow *your* code, not the database call
— and if CPU were saturated the request time would rise **without** the dependency rising.

**Why B is the plausible infrastructure story.** Network latency between two Azure services in the same
region does not increase 24-fold at the moment of a deployment. **A change in your code did.**

**Why D would change *what you see*, not what happened** — sampling reduces volume; it does not inflate
durations.

</details>

---

## Q2

A trace shows Frontend (50 ms) → API Gateway (30 ms) → Order Service (4500 ms) → Payment Service
(timeout). Which service should the team investigate?

- A. Frontend
- B. API Gateway
- C. Order Service
- D. Payment Service

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-50.md`:** lines **559–567**, with the trace query at **148–154**.

**Order Service's 4500 ms *is* the wait for Payment.** It is the symptom, and it will disappear the
moment Payment is fixed.

**This is the single most reliable pattern in distributed-tracing questions:** the slow span with a
failing or timing-out child is not the culprit — **the child is**.

**Why C is the answer the numbers seem to point at**, and why it is wrong: 4500 ms is the largest figure
on the page, which is exactly why the exam puts it there.

</details>

---

## Q3

The SLO is 99.9% availability over 30 days. After 15 days, 80% of the error budget is consumed. What
should the team do?

- A. Nothing — the SLO has not been breached
- B. Freeze all deployments
- C. Reduce deployment frequency and increase testing rigour
- D. Lower the SLO target to 99.5%

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-50.md`:** lines **570–578** and the burn-rate query at **432–444**.

**Do the arithmetic: 80% of the budget in 50% of the window is a burn rate of 1.6.** That is over 1.0,
so the budget runs out before the window closes — but it is not a crisis.

**Why B is the over-reaction, and the challenge says so** (line 578): a full freeze is reserved for
**budget already exhausted**. Freezing at 1.6× stops reliability fixes shipping too.

**Why A misreads what a budget is for.** The point of an error budget is that it triggers action **before**
the SLO is missed. Waiting for the breach makes the whole mechanism decorative.

**Why D is the response that ends the practice.** Moving the target to match reality means the SLO no
longer constrains anything.

</details>

---

## Q4

Which Application Insights feature detects performance anomalies **without any manual configuration**?

- A. Availability tests
- B. Smart detection
- C. Log-based alerts
- D. Metric alerts with dynamic thresholds

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-50.md`:** lines **343** and **355–358**.

```text
# - Failure Anomalies: enabled (sends to subscription owners by default)
# - Slow page load time: enabled
# - Slow server response time: enabled
# - Long dependency duration: enabled
```

**Smart detection is on by default and learns the application's normal behaviour** — four detectors, no
rules to write.

**Why D is the near-miss the exam is testing.** Dynamic thresholds also learn a baseline, and they must
be **created**:

```bash
--condition "avg requests/duration > dynamic medium 3 of 5"
```

**"Without requiring manual configuration" is the discriminator**, and it selects B over D every time.

</details>

---

## Q5

The end-to-end view shows the payment service as a separate, uncorrelated trace. What is the cause?

- A. The payment service is not instrumented
- B. `traceparent` is not propagated to downstream calls
- C. Sampling removed the spans
- D. The services use different Application Insights resources

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-50.md`:** lines **477–479**.

**The payment service *appears*, which is what rules out A.** Its telemetry exists; it simply carries a
different `operation_Id`, so nothing joins it to the parent.

**The diagnosis at lines 492–501 is the query worth remembering** — count dependencies targeting payment
that have no matching request:

```kql
dependencies
| where target contains "payment"
| join kind=leftanti (
    requests | where cloud_RoleName == "payment-service" | project operation_Id
) on operation_Id
| count  // Number of unmatched traces
```

**And the fix is a one-line SDK setting** (line 514): `setDistributedTracingMode(...AI_AND_W3C)`.

</details>

---

## Q6

Why must Application Insights be initialised **before other imports** in the Node.js fix?

- A. So the SDK can patch HTTP libraries as they load
- B. To read environment variables first
- C. To avoid a circular dependency
- D. It is a style convention

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-50.md`:** lines **511–515**.

```javascript
// Ensure Application Insights is initialized BEFORE other imports
const appInsights = require('applicationinsights');
```

**The SDK works by monkey-patching `http`, `https` and common clients** so outbound calls carry
`traceparent` automatically. A library loaded first keeps an unpatched reference — and its calls leave
without the header.

**Which is why the symptom is *partial*:** some calls correlate and some do not, depending on load order.

</details>

---

## Q7

A CPU spike is visible on the VM at deployment time. Which query identifies the responsible process?

- A. `Perf | where CounterName == "% Processor Time"`
- B. `VMProcess | summarize avg(PercentProcessorTime) by ProcessName`
- C. `ContainerInventory`
- D. `requests | summarize avg(duration)`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-50.md`:** lines **528–534**.

```kql
VMProcess
| where Computer == "vm-contoso-orders"
| summarize avgCPU = avg(PercentProcessorTime) by ProcessName, bin(TimeGenerated, 5m)
| where avgCPU > 10
```

**`Perf` gives the machine total; `VMProcess` attributes it to a process.** That attribution is VM
Insights' Map data doing its job (Challenge 47 Q2).

**Why A tells you *that* CPU rose** and leaves you guessing between your application, an agent, a backup
job and a runaway sidecar.

</details>

---

## Q8

Which counter reports available memory on a VM?

- A. `Available MBytes`
- B. `% Processor Time`
- C. `MemoryWorkingSet`
- D. `cpuUsageNanoCores`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-50.md`:** line **220**.

```kql
| where CounterName == "Available MBytes"
```

**Note that it measures what is *free*, not what is used** — so the alarming direction is **down**, and a
threshold on it is `<`, not `>`.

**Why C is the right idea in the wrong system.** `MemoryWorkingSet` is an **App Service platform metric**
queried through `az monitor metrics list` (line 271), not a `Perf` counter.

**Why D is the container counter** at line 234.

</details>

---

## Q9

What does `avg(CounterValue / 1000000.0)` compute for `cpuUsageNanoCores`?

- A. CPU usage converted from nanocores to millicores
- B. A percentage
- C. Memory in megabytes
- D. Requests per second

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-50.md`:** lines **234–236**.

```kql
| where CounterName == "cpuUsageNanoCores"
| summarize avgCPU = avg(CounterValue / 1000000.0) by bin(TimeGenerated, 1m)
```

**Kubernetes reports CPU in nanocores; dividing by a million gives millicores**, the unit used in pod
`resources.requests` and `limits`.

**Which is what makes the number actionable.** 250 millicores against a 500 m limit is half the
allocation; the raw nanocore figure means nothing without the conversion.

</details>

---

## Q10

Which query finds the dependency with the greatest **total** impact rather than the single slowest call?

- A. `order by duration desc | take 10`
- B. `summarize avgDuration = avg(duration), slowCallCount = count() by target | extend impact = avgDuration * slowCallCount | order by impact desc`
- C. `summarize max(duration) by target`
- D. `where duration > 2000 | count()`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-50.md`:** lines **134–144**.

```kql
| extend impact = avgDuration * slowCallCount
| order by impact desc
```

**Duration × frequency is where the time actually goes.** A 10-second call made twice costs 20 seconds; a
900 ms call made 4000 times costs an hour.

**Why A and C both find the *worst single call***, which is often a one-off timeout that nobody
experienced twice.

**This is the query that decides what to fix first**, and it is the one people never write — because the
slowest call is easier to find and feels more urgent.

</details>

---

## Q11

What does `union requests, dependencies, exceptions` with a single `operation_Id` produce?

- A. The complete end-to-end timeline of one transaction
- B. A count of operations
- C. A join between the tables
- D. All telemetry for the last hour

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-50.md`:** lines **120–131**.

```kql
union requests, dependencies, exceptions
| where operation_Id == slowOperationId
| project timestamp, itemType = itemType, name, duration, success, ...
| order by timestamp asc
```

**Ordering by timestamp is what makes it a *waterfall*** — the incoming request, then each outbound call
in sequence, then any exception, so you can see where the gap is.

**And `coalesce(target, "")` at line 128 exists** because `target` is only present on dependencies. Union
requires compatible columns, so absent ones are filled.

</details>

---

## Q12

What does `iff(timestamp < deployTime, "Before", "After")` accomplish?

- A. Labels each row so a single `summarize by period` compares both windows
- B. Filters to before the deployment
- C. Sorts by deployment time
- D. Creates a deployment annotation

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-50.md`:** lines **76–83**.

```kql
| where timestamp between ((deployTime - 2h) .. (deployTime + 2h))
| extend period = iff(timestamp < deployTime, "Before", "After")
| summarize avgDuration = ..., p95Duration = ..., errorRate = ..., requestCount = count()
    by period
```

**One query, two rows, directly comparable.** No join, no `let` windows — the label does the grouping.

**Compare with the endpoint-level version at lines 190–197**, which *does* need two `let` blocks and a
join, because it groups `by name` on each side and then matches them.

**Note `"1-Before"` and `"2-After"` at line 312** — the numeric prefixes force the sort order, since
"After" sorts before "Before" alphabetically.

</details>

---

## Q13

How does the endpoint degradation query decide what counts as degraded?

- A. `degradationPct > 50` after computing the percentage change in average duration
- B. Any endpoint slower than 1 second
- C. The top 10 by absolute duration
- D. Endpoints with errors

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-50.md`:** lines **196–200**.

```kql
beforePerf
| join kind=inner afterPerf on name
| extend degradationPct = round(((afterAvg - beforeAvg) / beforeAvg) * 100, 1)
| where degradationPct > 50  // Endpoints that got 50%+ slower
```

**Relative change, not absolute.** An endpoint that went from 40 ms to 80 ms has doubled and would never
appear on a "slower than 1 second" list — yet it is a regression, and often the clearest signal of what
the deployment changed.

**Why `inner` is correct here**, unlike Challenge 49's exception comparison: an endpoint must exist on
both sides for a percentage change to mean anything. An endpoint that only exists after is a **new
endpoint**, not a degraded one.

</details>

---

## Q14

What does `toscalar(lastDeployment)` provide?

- A. The most recent deployment's timestamp as a value usable in arithmetic
- B. A table of deployments
- C. A string
- D. A count

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-50.md`:** lines **169–176**.

```kql
let lastDeployment = customEvents
| where name == "Deployment" or name == "DeploymentAnnotation"
| top 1 by timestamp desc
| project deployTime = timestamp;
let deployTime = toscalar(lastDeployment);
requests
| where timestamp between ((deployTime - 1h) .. (deployTime + 1h))
```

**This is what makes the query self-updating.** Nobody edits a hard-coded datetime — the query finds the
latest deployment itself and builds its windows around it.

**Which is the difference between a query you run and a query you save.** The hard-coded version at line
189 is fine for one investigation; this one goes in a workbook.

</details>

---

## Q15

Which condition creates a **dynamic threshold** metric alert?

- A. `avg requests/duration > 3000`
- B. `avg requests/duration > dynamic medium 3 of 5`
- C. `count requests/failed > 50`
- D. `avg requests/duration > baseline`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-50.md`:** line **377**.

```bash
  --condition "avg requests/duration > dynamic medium 3 of 5"
```

**Read the three parts.** `dynamic` means the threshold is learned rather than stated. `medium` is the
sensitivity. `3 of 5` means alert when 3 of the last 5 evaluation periods breach — which is what stops a
single noisy interval paging anyone.

**Why A is the static version** at line 365, and it is not wrong — it is just a number somebody chose,
which will be wrong at 3 AM and wrong again on Black Friday.

</details>

---

## Q16

Which query answers "what percentage of requests completed under 1 second"?

- A. `summarize totalRequests = count(), fastRequests = countif(duration < latencyThreshold) | extend latencyCompliance = (fastRequests * 100.0) / totalRequests`
- B. `summarize avg(duration)`
- C. `summarize percentile(duration, 95)`
- D. `where duration < 1000 | count()`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-50.md`:** lines **407–417**.

```kql
let latencyTarget = 99.0;  // 99% of requests under 1 second
let latencyThreshold = 1000;  // milliseconds
...
| extend latencyCompliance = round((fastRequests * 100.0) / totalRequests, 2)
```

**A latency SLI is a *proportion*, not an average.** "99% of requests under 1 second" is a promise about
how many users had a good experience — an average makes no promise to anyone.

**Why C is closely related and not the same.** P95 under 1000 ms and "95% of requests under 1000 ms" are
two statements of the same fact; the SLI form is the one you can express as a **budget** and burn.

**Why D gives a count without a denominator**, so it grows with traffic and says nothing about
compliance.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** does the performance summary report? (Choose three.)

- A. Request count
- B. P50, P90, P95 and P99 duration
- C. Maximum duration
- D. CPU utilisation
- E. Error budget remaining
- F. Deployment version

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-50.md`:** lines **36–43**.

```kql
| summarize
    requestCount = count(),
    avgDuration = avg(duration),
    p50 = percentile(duration, 50), p90 = ..., p95 = ..., p99 = ...,
    maxDuration = max(duration)
```

**Count, distribution, worst case — and the spread between them is the diagnosis.** P50 at 200 ms with
P99 at 8000 ms is a tail problem affecting a small group badly; both rising together is a systemic
regression.

**Why max matters despite being a single point.** It bounds the worst experience anyone had, and a max
far above P99 usually means a timeout value.

</details>

---

## Q18

Which **three** infrastructure signals does the challenge query from `Perf`? (Choose three.)

- A. `% Processor Time`
- B. `Available MBytes`
- C. `Disk Reads/sec` and `Disk Writes/sec`
- D. Request duration
- E. Exception counts
- F. Error budget

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-50.md`:** lines **211**, **220**, **243**.

**Plus network at line 251** — `Bytes Sent/sec` and `Bytes Received/sec`. **Four resources: CPU, memory,
disk, network**, which is the exam objective's own list.

**Why D and E are application telemetry** in `requests` and `exceptions`. **The `Perf` table is
infrastructure**, and knowing which table holds which signal is half of these questions.

</details>

---

## Q19

Which **two** identify the bottleneck in a slow transaction? (Choose two.)

- A. `union requests, dependencies, exceptions` filtered to one `operation_Id`, ordered by timestamp
- B. The transaction search waterfall view in the portal
- C. `summarize avg(duration)` across all requests
- D. VM CPU metrics
- E. The Application Insights sampling settings

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-50.md`:** lines **120–131** and **159–163**.

```text
4. View the end-to-end transaction timeline showing all dependencies
5. Identify the dependency taking the most time (highlighted in the waterfall view)
```

**Both do the same thing — one in KQL, one in the UI.** The waterfall is faster to read; the query is
reusable and can be saved into a workbook.

**Why C aggregates away the very thing you are looking for.** An average across all requests cannot tell
you where the time went inside one of them.

</details>

---

## Q20

Which **two** are true of smart detection? (Choose two.)

- A. It is enabled by default
- B. It covers failure anomalies, slow response time and long dependency duration
- C. It must be configured with thresholds
- D. It triggers automated rollback
- E. It replaces metric alerts

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-50.md`:** lines **343** and **355–358**.

**Why D is the Challenge 46 boundary, restated.** Smart detection **notifies** — it does not invoke an
action group to run anything.

**Why E overstates it.** Smart detection finds anomalies **you did not think to write a rule for**. A
metric alert enforces a threshold **you care about specifically**, such as "average duration over 3
seconds is unacceptable regardless of the baseline". Both, not either.

</details>

---

## Q21

Which **two** describe the multi-window burn-rate alert? (Choose two.)

- A. A fast window of 1 hour with a 14.4× threshold
- B. A slow window of 6 hours with a 6× threshold
- C. A 30-day window with a 1× threshold
- D. A fixed 5% error rate threshold
- E. Alerting only after the budget is exhausted

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-50.md`:** lines **452–468**.

```kql
// Fast burn: 14.4x budget consumption in 1 hour
// Slow burn: 6x budget consumption in 6 hours
```

**Two windows catch two different failures.** The fast window catches a sharp outage — 14.4× for an hour
consumes 2% of a 30-day budget. The slow window catches a **steady leak** that never trips a short-window
alert but exhausts the budget over days.

**And 14.4 is not arbitrary:** 1 hour is 1/720 of 30 days, and 14.4/720 = 2%. The number is chosen so the
alert fires after a meaningful fraction of budget is gone, not after a single bad minute.

**Why E is the failure this pattern exists to prevent** — by exhaustion it is too late to act.

</details>

---

## Q22

Which **two** distinguish the availability SLI from the latency SLI? (Choose two.)

- A. Availability counts successful requests against total
- B. Latency counts requests under a duration threshold against total
- C. Availability uses P95
- D. Latency uses `countif(success == false)`
- E. Both measure the same thing

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-50.md`:** lines **392–401** and **409–415**.

```kql
    successfulRequests = countif(success == true and resultCode !startswith "5")
    fastRequests = countif(duration < latencyThreshold)
```

**Two questions: did it work, and was it fast enough.** A service can be 100% available and unusably
slow.

**Note the availability definition excludes 5xx explicitly** even alongside `success == true` — belt and
braces, because a handled 500 could be recorded as a completed request.

</details>

---

## Q23

Which **two** are true about error budgets? (Choose two.)

- A. The budget is `100% − SLO` of total requests
- B. A burn rate above 1.0 means the budget will be exhausted before the window ends
- C. Lowering the SLO is the correct response to a high burn rate
- D. The budget resets on every deployment
- E. Budget consumption has no bearing on release decisions

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-50.md`:** lines **400–404** and **441–444**.

```kql
    errorBudgetTotal = round(totalRequests * (1 - sloTarget / 100), 0),
    errorBudgetUsed = totalRequests - successfulRequests
...
    burnRate = round(failedReq / (totalReq * (1 - sloTarget / 100.0)), 2)
| extend isBurningFast = burnRate > 1.0
```

**Burn rate 1.0 is the pace that exactly spends the budget over the window.** Above it, you run out
early; below it, you finish with budget unspent — which is permission to take more risk.

**Why E inverts the entire purpose.** The budget exists precisely to make release decisions objective:
budget remaining means ship; budget gone means stabilise (Q3).

</details>

---

# Section C — Repeated scenario

**Scenario:** After a 2:30 PM deployment, users report slowness and complaints rise 40% within an hour.
Contoso must identify the root cause, correlate it with the deployment, and decide between rollback and
hotfix.

---

## Q24

**Proposed solution:** Summarise request percentiles for the surrounding four hours. Rank dependencies by
P95 and by duration × call count. Pull one slow `operation_Id` and union requests, dependencies,
exceptions and traces ordered by timestamp. Compare before and after using `iff` on the deployment time,
and list endpoints whose average duration rose more than 50%. Check `Perf` for CPU, memory, disk and
network, and `VMProcess` to attribute any CPU spike to a process. Decide using the error budget's burn
rate.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-50.md`:** lines **34–51**, **62–71**, **134–144**, **148–154**, **74–83**, **190–201**,
**208–253**, **528–534**, **432–444**.

| Question | Query |
|---|---|
| How bad, and for whom? | Percentile distribution |
| Where is the time going? | Dependencies by P95 **and** by impact |
| Inside one slow request? | Union on `operation_Id`, ordered |
| Did the deployment do it? | Before/after with `iff` |
| Which endpoints specifically? | Degradation percentage, `inner` join |
| Is it the infrastructure? | `Perf` four counters, `VMProcess` for attribution |
| Roll back or hotfix? | Burn rate against the error budget |

**The last row is what makes it a decision rather than an investigation.** Everything above identifies
the cause; the budget says how much urgency the business can afford.

</details>

---

## Q25

**Proposed solution:** Look at the average response time on the overview blade. Restart the App Service to
clear the slowness. If it recurs, check the web server's CPU. Decide on rollback based on whether
complaints continue.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures.**

**An average hides the distribution.** With P50 at 200 ms and P99 at 8000 ms, the average can look
acceptable while 1% of users are timing out — which is exactly the population that files complaints.

**Restarting is a guess that also destroys the evidence.** In-flight state, connection pools and the
current process are gone, and if the fault is a slow query it returns within minutes.

**CPU is the wrong first stop** when the dependency data is available (Q1). It is the reflex answer, and
the trace will usually have already ruled it out.

**And "whether complaints continue" is not a rollback criterion.** Complaint volume lags the fault by
minutes to hours and depends on the time of day. The error budget is the criterion (Q3, Q23).

</details>

---

## Q26

**Proposed solution:** Summarise percentiles. Rank dependencies by P95 and by impact. Trace one slow
operation end to end. Compare before and after with `iff`. Check `Perf` and `VMProcess`. Then, because
average duration is now well above the historical baseline, roll back immediately and investigate
afterwards — the fastest path back to health.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

The investigation is right, and the decision skips the one question that determines whether rollback is
even the correct action.

**Rollback restores the previous *code*. It does not restore a previous *database*.** If the deployment
ran a schema migration — a new column, a dropped index, an altered stored procedure — the old code now
meets a database it does not expect. That is Challenge 27's expand-contract problem, and it turns one
incident into two.

**And the trace evidence may already exclude rollback.** If the bottleneck is a downstream **payment
provider** timing out (Q2), rolling back your own code changes nothing at all.

**"Investigate afterwards" also discards the evidence.** After the swap, the failing version is no longer
serving — so the live telemetry that identifies the slow query stops arriving, and the post-mortem is
run against a healthy system.

**Roll back when the fault is in your code and reversible; hotfix when the fault is downstream or the
deployment is not cleanly reversible.** The trace and the migration history answer that, and the burn
rate answers how fast you must act — which is the sequence Q24 follows and this one skips.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — reading a trace

| # | Statement | Answer |
|---|---|---|
| 1 | A slow parent span waiting on a failing child means the child is the cause |  |
| 2 | The service with the largest duration is always the culprit |  |
| 3 | Subtracting dependency time from request time isolates your own code |  |
| 4 | `union` on `operation_Id` ordered by timestamp gives the transaction timeline |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A slow parent span waiting on a failing child means the child is the cause | **Yes** |
| 2 | The service with the largest duration is always the culprit | **No** |
| 3 | Subtracting dependency time from request time isolates your own code | **Yes** |
| 4 | `union` on `operation_Id` ordered by timestamp gives the transaction timeline | **Yes** |

**In `challenge-50.md`:** lines **559–567**, **62–71**, **120–131**.

Rows 1 and 2 are the same fact stated twice, because it is the one the exam tests most. **Read the trace
from the bottom.**

</details>

---

## Q28 — infrastructure

| # | Statement | Answer |
|---|---|---|
| 1 | `Perf` holds VM and container counters |  |
| 2 | `VMProcess` attributes CPU to a named process |  |
| 3 | `Available MBytes` rising is a warning sign |  |
| 4 | `cpuUsageNanoCores` divided by 1,000,000 gives millicores |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `Perf` holds VM and container counters | **Yes** |
| 2 | `VMProcess` attributes CPU to a named process | **Yes** |
| 3 | `Available MBytes` rising is a warning sign | **No** |
| 4 | `cpuUsageNanoCores` divided by 1,000,000 gives millicores | **Yes** |

**In `challenge-50.md`:** lines **208–253**, **528–531**, **220**, **236**.

Row 3 is the direction trap. **Available memory going *up* is fine; going *down* is the problem** — the
counter measures what is free.

</details>

---

## Q29 — deployment correlation

| # | Statement | Answer |
|---|---|---|
| 1 | `iff` labels rows Before and After for a single grouped comparison |  |
| 2 | `toscalar` on the latest deployment makes the query self-updating |  |
| 3 | The endpoint degradation query uses a `fullouter` join |  |
| 4 | `"1-Before"` and `"2-After"` control sort order |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `iff` labels rows Before and After for a single grouped comparison | **Yes** |
| 2 | `toscalar` on the latest deployment makes the query self-updating | **Yes** |
| 3 | The endpoint degradation query uses a `fullouter` join | **No** |
| 4 | `"1-Before"` and `"2-After"` control sort order | **Yes** |

**In `challenge-50.md`:** lines **77**, **174**, **197**, **312**.

Row 3 is the deliberate contrast with Challenge 49. **A percentage change needs both sides**, so `inner`
is correct here — an endpoint present only after is new, not degraded.

</details>

---

## Q30 — SLO and alerting

| # | Statement | Answer |
|---|---|---|
| 1 | Error budget is `100% − SLO` of total requests |  |
| 2 | Burn rate above 1.0 exhausts the budget before the window ends |  |
| 3 | Smart detection triggers automated remediation |  |
| 4 | `dynamic medium 3 of 5` requires 3 of 5 periods to breach |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Error budget is `100% − SLO` of total requests | **Yes** |
| 2 | Burn rate above 1.0 exhausts the budget before the window ends | **Yes** |
| 3 | Smart detection triggers automated remediation | **No** |
| 4 | `dynamic medium 3 of 5` requires 3 of 5 periods to breach | **Yes** |

**In `challenge-50.md`:** lines **400**, **443–444**, **343**, **377**.

Row 3 recurs from Challenge 46: **detection notifies, action groups act.**

Row 4 is what makes a dynamic alert usable — a single anomalous interval does not page anyone.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each symptom to what it points at.

| Symptom | Points at |
|---|---|
| Request 1500 ms, SQL dependency 1200 ms |  |
| Request 1500 ms, dependencies 50 ms, CPU at 95% |  |
| Parent span 4500 ms, child timing out |  |
| P50 unchanged, P99 tripled |  |
| `Available MBytes` falling steadily |  |
| One endpoint 50% slower, others unchanged |  |

**Options:** A change in that endpoint's code path · A memory leak · A tail problem — a subset of requests · The child service · The database query · Your own code or an undersized instance

<details>
<summary>Show answer</summary>

| Symptom | Points at |
|---|---|
| Request 1500 ms, SQL dependency 1200 ms | **The database query** |
| Request 1500 ms, dependencies 50 ms, CPU at 95% | **Your own code or an undersized instance** |
| Parent span 4500 ms, child timing out | **The child service** |
| P50 unchanged, P99 tripled | **A tail problem — a subset of requests** |
| `Available MBytes` falling steadily | **A memory leak** |
| One endpoint 50% slower, others unchanged | **A change in that endpoint's code path** |

**In `challenge-50.md`:** lines **548–556**, **208–213**, **559–567**, **36–43**, **220**, **196–200**.

**Every row is the same move: subtract what you can attribute, and look at what is left.**

</details>

---

## Q32

Match each table or counter to what it measures.

| Source | Measures |
|---|---|
| `requests` |  |
| `dependencies` |  |
| `Perf` — `% Processor Time` |  |
| `VMProcess` — `PercentProcessorTime` |  |
| `Perf` — `cpuUsageNanoCores` |  |
| `customEvents` — `Deployment` |  |

**Options:** Container CPU · CPU per process · Inbound call duration and success · Outbound call duration, by target · VM CPU total · When the release happened

<details>
<summary>Show answer</summary>

| Source | Measures |
|---|---|
| `requests` | **Inbound call duration and success** |
| `dependencies` | **Outbound call duration, by target** |
| `Perf` — `% Processor Time` | **VM CPU total** |
| `VMProcess` — `PercentProcessorTime` | **CPU per process** |
| `Perf` — `cpuUsageNanoCores` | **Container CPU** |
| `customEvents` — `Deployment` | **When the release happened** |

**In `challenge-50.md`:** lines **34**, **62**, **211**, **531**, **234**, **169–170**.

**The two CPU rows are the pair the exam separates.** The machine total tells you *something* is
consuming CPU; the per-process figure tells you *what* — which is Break scenario 2 exactly.

</details>

---

## Q33

Arrange the investigation for "the app is slow since the 2:30 PM deployment".

**Items:** Compare before and after around the deployment time · Check infrastructure counters and
attribute any CPU spike to a process · Rank dependencies by P95 and by impact · Summarise request
percentiles for the surrounding hours · Trace one slow operation end to end · Decide rollback or hotfix
using the burn rate

<details>
<summary>Show answer</summary>

### Answer

1. Summarise request percentiles for the surrounding hours — lines **34–51**
2. Compare before and after around the deployment time — lines **74–83**
3. Rank dependencies by P95 and by impact — lines **62–71**, **134–144**
4. Trace one slow operation end to end — lines **148–154**
5. Check infrastructure counters and attribute any CPU spike to a process — lines **208–253**, **528–534**
6. Decide rollback or hotfix using the burn rate — lines **432–444**

**Steps 1 and 2 come first because they establish *whether there is a problem* and *whether the
deployment owns it*.** Investigating a dependency before confirming the timing wastes the hour you have.

**Step 5 is deliberately late.** Infrastructure is the answer far less often than instinct suggests, and
the dependency ranking in step 3 usually rules it out — if the time is in a SQL call, CPU is not the
story.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Payment service appears as an uncorrelated trace |  |
| Some calls correlate, some do not |  |
| CPU spike visible, culprit unknown |  |
| Slowest-call list dominated by one-off timeouts |  |
| Average looks fine, users complain |  |
| Endpoint regression missed |  |

**Options:** Absolute threshold instead of percentage change · Looking at `Perf` totals instead of `VMProcess` · Ranking by duration instead of duration × count · Reading the average instead of P95/P99 · SDK initialised after other imports · `traceparent` not propagated

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| Payment service appears as an uncorrelated trace | **`traceparent` not propagated** |
| Some calls correlate, some do not | **SDK initialised after other imports** |
| CPU spike visible, culprit unknown | **Looking at `Perf` totals instead of `VMProcess`** |
| Slowest-call list dominated by one-off timeouts | **Ranking by duration instead of duration × count** |
| Average looks fine, users complain | **Reading the average instead of P95/P99** |
| Endpoint regression missed | **Absolute threshold instead of percentage change** |

**In `challenge-50.md`:** lines **479**, **511**, **528–531**, **142**, **36–43**, **198–199**.

**Rows 4, 5 and 6 are all the same error in different clothes:** choosing a metric that is easy to compute
over the one that reflects what users experience.

</details>

---

## Q35

Match each SLO concept to its definition.

| Concept | Definition |
|---|---|
| SLI |  |
| SLO |  |
| Error budget |  |
| Burn rate |  |
| Fast-burn alert |  |
| Slow-burn alert |  |

**Options:** `100% − SLO` of total requests · 14.4× over 1 hour · 6× over 6 hours · How fast the budget is being spent, relative to 1.0 · The measurement — availability, latency compliance · The target for that measurement, e.g. 99.9%

<details>
<summary>Show answer</summary>

| Concept | Definition |
|---|---|
| SLI | **The measurement — availability, latency compliance** |
| SLO | **The target for that measurement, e.g. 99.9%** |
| Error budget | **`100% − SLO` of total requests** |
| Burn rate | **How fast the budget is being spent, relative to 1.0** |
| Fast-burn alert | **14.4× over 1 hour** |
| Slow-burn alert | **6× over 6 hours** |

**In `challenge-50.md`:** lines **390–404**, **432–444**, **452–468**.

**Burn rate 1.0 is the reference point worth anchoring on.** It is the pace that spends the budget
exactly over the window — so 2.0 exhausts it halfway through, and 0.5 leaves half unspent.

</details>

---

# Section F — Hot area

---

## Q36

```kql
requests
| where timestamp > ago(4h)
| summarize
    p50 = [BLANK 1](duration, 50),
    p95 = [BLANK 1](duration, 95),
    p99 = [BLANK 1](duration, 99),
    maxDuration = [BLANK 2](duration)
```

- **BLANK 1:** `percentile` / `percentrank` / `quantile` / `avg`
- **BLANK 2:** `max` / `top` / `arg_max` / `last`

<details>
<summary>Show answer</summary>

### Answer: `percentile`, `max`

**In `challenge-50.md`:** lines **39–43**.

**`percentile()` is the KQL name** — `quantile` is other query languages, and `percentrank` computes the
inverse.

**And `arg_max` is the interesting distractor:** it returns the **row** containing the maximum, not the
value. Useful when you want to know *which* request was slowest; wrong when you want the number.

</details>

---

## Q37

```kql
dependencies
| where duration > 2000
| summarize avgDuration = avg(duration), slowCallCount = count() by target, name, type
| extend impact = [BLANK 1]
| order by impact [BLANK 2]
```

Requirement: find the dependency costing the most total time.

- **BLANK 1:** `avgDuration * slowCallCount` / `avgDuration` / `slowCallCount` /
  `avgDuration / slowCallCount`
- **BLANK 2:** `desc` / `asc`

<details>
<summary>Show answer</summary>

### Answer: `avgDuration * slowCallCount`, `desc`

**In `challenge-50.md`:** lines **142–143**.

**Duration alone finds the worst call; count alone finds the busiest dependency.** Neither is total cost,
and the product is (Q10).

**This is the query that decides what to fix first**, and it frequently disagrees with intuition — a
900 ms call made 4000 times costs far more than a 30-second call made twice.

</details>

---

## Q38

```kql
let deployTime = datetime(2024-11-15T14:30:00Z);
requests
| where timestamp [BLANK 1] ((deployTime - 2h) .. (deployTime + 2h))
| extend period = [BLANK 2](timestamp < deployTime, "Before", "After")
| summarize avgDuration = ..., errorRate = ... by [BLANK 3]
```

- **BLANK 1:** `between` / `in` / `has` / `contains`
- **BLANK 2:** `iff` / `case` / `if` / `switch`
- **BLANK 3:** `period` / `timestamp` / `name` / `bin(timestamp, 5m)`

<details>
<summary>Show answer</summary>

### Answer: `between`, `iff`, `period`

**In `challenge-50.md`:** lines **76–83**.

**`iff` is KQL's two-branch conditional** — `if` alone is not a KQL function, and `case` is the
multi-branch form.

**And `by period` is what produces exactly two comparable rows.** Grouping by `bin(timestamp, 5m)`
instead would give a time series — useful for a chart (line 184), and not a before/after summary.

</details>

---

## Q39

```kql
[BLANK 1]
| where TimeGenerated > ago(2h)
| where Computer == "vm-contoso-orders"
| summarize avgCPU = avg([BLANK 2]) by [BLANK 3], bin(TimeGenerated, 5m)
| where avgCPU > 10
```

Requirement: identify which process is consuming CPU.

- **BLANK 1:** `VMProcess` / `Perf` / `ContainerInventory` / `Heartbeat`
- **BLANK 2:** `PercentProcessorTime` / `CounterValue` / `CpuPercentage` / `cpuUsageNanoCores`
- **BLANK 3:** `ProcessName` / `Computer` / `InstanceName` / `CounterName`

<details>
<summary>Show answer</summary>

### Answer: `VMProcess`, `PercentProcessorTime`, `ProcessName`

**In `challenge-50.md`:** lines **528–532**.

**`Perf` with `CounterValue` gives the machine total** and cannot be grouped by process — the column does
not exist there.

**`VMProcess` is populated by VM Insights' Map data collection** (Challenge 47 Q8, the
`Microsoft-ServiceMap` stream). **Omit that stream and this query returns nothing** — which links the two
challenges directly.

</details>

---

## Q40

```bash
az monitor metrics alert create \
  --name "alert-dynamic-response-time" \
  --condition "avg requests/duration > [BLANK 1] [BLANK 2] [BLANK 3]" \
  --window-size 5m --evaluation-frequency 5m
```

Requirement: a learned baseline that does not fire on a single anomalous interval.

- **BLANK 1:** `dynamic` / `static` / `baseline` / `auto`
- **BLANK 2:** `medium` / `strict` / `loose` / `default`
- **BLANK 3:** `3 of 5` / `1 of 1` / `5 of 5` / `2 of 2`

<details>
<summary>Show answer</summary>

### Answer: `dynamic`, `medium`, `3 of 5`

**In `challenge-50.md`:** line **377**.

```bash
  --condition "avg requests/duration > dynamic medium 3 of 5"
```

**`3 of 5` is the requirement's second half** — three breaching periods out of the last five. `1 of 1`
fires on any single interval, which is the noise this setting exists to suppress.

**And sensitivity is `low`, `medium` or `high`** — higher sensitivity means a tighter band and more
alerts.

</details>

---

## Q41

```kql
let sloTarget = 99.9;
requests
| where timestamp > ago(30d)
| summarize totalReq = count(), failedReq = countif(success == false) by bin(timestamp, 1d)
| extend
    dailyErrorBudget = totalReq * (1 - sloTarget / [BLANK 1]),
    burnRate = round(failedReq / (totalReq * (1 - sloTarget / [BLANK 1])), 2)
| extend isBurningFast = burnRate > [BLANK 2]
```

- **BLANK 1:** `100.0` / `100` / `1000` / `10.0`
- **BLANK 2:** `1.0` / `0.1` / `99.9` / `14.4`

<details>
<summary>Show answer</summary>

### Answer: `100.0`, `1.0`

**In `challenge-50.md`:** lines **441–444**.

**`100.0` keeps the arithmetic in floating point** — with `100`, `sloTarget / 100` is `0` in integer
arithmetic and the budget becomes the whole request count, so nothing ever burns fast. Challenge 49's
silent killer, in a new place.

**And `1.0` is the reference pace** (Q35): above it the budget runs out before the window closes.
**`14.4` is the fast-burn threshold** at line 467, which is a different alert over a 1-hour window.

</details>

---

# Section G — Case study

## Case study: Contoso post-deployment slowdown

### Background

After the latest deployment at **2:30 PM**, users report the Contoso web application is **"slow"**. The
support team sees a **40% increase in complaint tickets within an hour**. Contoso must identify the root
cause, correlate it with the deployment, and **decide whether to roll back or hotfix**.

### Requirements

**Diagnosis**

- Quantify the degradation in a way that reflects user experience, not an average
- Determine whether the time is being spent in application code or in a dependency
- Identify which specific endpoints degraded, including ones that were always fast
- Attribute any infrastructure spike to a named process
- Confirm the degradation began at the deployment and not before

**Tracing**

- A slow transaction must be viewable end to end, including exceptions
- Every service in the path must correlate into one trace

**Decision**

- The urgency of the response must be driven by the error budget, not by ticket volume
- The choice between rollback and hotfix must be justified by the evidence

---

## Q42

How should the degradation be quantified?

- A. Percentile distribution — P50, P90, P95, P99 and max — over the surrounding hours
- B. Average response time
- C. Total request count
- D. The number of support tickets

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-50.md`:** lines **36–51**.

**"Reflects user experience, not an average" is the requirement**, and percentiles are what express it.

**Why B fails in a specific and common way.** With P50 at 200 ms and P99 at 8000 ms, the average sits near
the median and looks acceptable — while one request in a hundred takes eight seconds. **Those are the
users filing the 40% more tickets.**

**Why D is a lagging proxy.** Tickets depend on time of day, user patience and whether anyone knows how to
report a problem. The telemetry is direct.

</details>

---

## Q43

How do you determine whether the time is in code or in a dependency?

- A. Rank dependencies by P95 **and** by duration × call count, then subtract dependency time from
  request duration
- B. Check web server CPU
- C. Look at the slowest single request
- D. Review the deployment diff

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-50.md`:** lines **62–71** and **134–144**.

**Both rankings are needed and they answer different questions.** P95 finds the dependency that is slow
*per call*; impact finds the one that costs the most *in total* — and they are often different targets
(Q10).

**The subtraction is the actual determination.** Request 1500 ms minus dependency 1200 ms leaves 300 ms of
your code — so the dependency owns it (Q1).

**Why D is a good next step and a poor first one.** The diff tells you what changed; it does not tell you
which change mattered, and a large deployment has many.

</details>

---

## Q44

How should the bottleneck service be identified in a distributed system?

- A. Union requests, dependencies, exceptions and traces on one `operation_Id`, ordered by timestamp, and
  read the trace from the bottom
- B. Investigate the service with the largest span duration
- C. Check each service's CPU in turn
- D. Restart services until it improves

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-50.md`:** lines **148–154** and **559–567**.

**"Read from the bottom" is the rule** (Q2): a 4500 ms parent waiting on a timing-out child is a symptom
of the child.

**Why B is the trap and it is very well disguised**, because the largest number on the screen genuinely is
the Order Service. Its 4500 ms is entirely wait time.

**Why C would take an hour** and would find nothing — a service waiting on a timeout is idle, not busy.

</details>

---

## Q45

Which **two** confirm the degradation began at the deployment? (Choose two.)

- A. `iff(timestamp < deployTime, "Before", "After")` with a grouped summary
- B. The endpoint degradation query with `degradationPct > 50`
- C. The support ticket timeline
- D. Total request count for the day
- E. The deployment's build number

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-50.md`:** lines **74–83** and **190–201**.

**A establishes *that* it changed at that minute; B establishes *what* changed.** Together they are the
correlation.

**Why B catches regressions an absolute threshold misses** (Q13): an endpoint going from 40 ms to 80 ms is
a 100% degradation and would never appear on a "slower than 1 second" list — and it may be the clearest
fingerprint of the code change.

**Why C is the lagging proxy again**, and **why E identifies the release without measuring anything.**

</details>

---

## Q46

How should the decision urgency be set?

- A. Burn rate against the error budget — above 1.0 means the budget will not survive the window
- B. The number of complaint tickets
- C. Whether the average exceeds 1 second
- D. Manager judgement

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-50.md`:** lines **432–444** and **452–468**.

**The requirement says "driven by the error budget, not by ticket volume"**, which rules out B directly.

**And the multi-window pattern gives two answers at once.** A 14.4× burn over an hour is an emergency — 2%
of a monthly budget in sixty minutes. A 6× burn over six hours is serious and not an emergency. **The same
outage, graded by how fast it is consuming what you can afford to lose.**

**Why C is a threshold somebody picked** with no relationship to what was promised.

</details>

---

## Q47

The investigation shows the slowdown is entirely in a SQL dependency that went from 50 ms to 1200 ms. The
deployment included a schema migration that added a column and changed a stored procedure. The burn rate
over the last hour is 9×.

Should Contoso roll back or hotfix, and why?

- A. Hotfix — the fault is in the database layer, and a code rollback would leave the old code against a
  migrated schema. Add the missing index or revert the stored procedure
- B. Roll back immediately — it is always the fastest route to health
- C. Do nothing — the burn rate is below the 14.4× fast-burn threshold
- D. Scale up the App Service plan

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-50.md`:** lines **62–71**, **548–556**, **452–468**.

**The evidence names the layer, and the layer determines the remedy.** 1200 of 1500 ms is SQL, and a
changed stored procedure with a new column is the classic shape of a missing index or a plan regression.

**Why B is dangerous here specifically, not in general.** Rolling back the code does **not** roll back the
migration. The previous version now runs against a schema it was never tested on — Challenge 27's
expand-contract problem, and a second incident on top of the first.

**Why C misreads the thresholds.** 9× is below fast-burn and far above the 6× slow-burn threshold, and
above 1.0 by a factor of nine. **It is not an emergency page; it is an act-today.**

**Why D is the response that costs money and fixes nothing.** More CPU does not make a table scan faster,
and the App Service is not the constrained resource — the dependency data already said so.

</details>

---

## Q48

At the post-incident review, Contoso is asked how a 40%-complaint incident was diagnosed and resolved in
under an hour, when the previous one took a day.

Walk through it, and explain what actually changed.

- A. Percentiles quantified it, the dependency ranking located it, the trace confirmed the layer,
  before/after tied it to 2:30 PM, the endpoint comparison named the code path, and the burn rate set the
  urgency — each step narrowing the search rather than sampling it
- B. The team had more people looking
- C. Smart detection alerted and someone restarted the app
- D. The average response time chart showed the problem

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-50.md`:** lines **36–51**, **62–71**, **134–154**, **74–83**, **190–201**, **432–444**.

**Follow the narrowing, because that is the whole answer.**

Percentiles said **how bad and for whom** — P99 tripled while P50 barely moved, so a subset of users was
badly affected. The dependency ranking said **where** — 1200 of 1500 ms in one SQL target. The trace
confirmed **the layer**, ruling out the application code and the services either side of it. Before/after
tied it to **2:30 PM**. The endpoint comparison named **which code path**. And the burn rate said **how
fast to act**.

**Six queries, each eliminating most of the remaining search space.** The previous incident took a day
because it was a sequence of guesses — restart, check CPU, check the network — each of which takes
minutes to try and proves nothing when it fails.

**What actually changed is not tooling.** Every one of these queries runs against telemetry Application
Insights was already collecting during the previous incident. **What changed is having a fixed order to
ask the questions in**, so each answer constrains the next — and having the last question be *how urgent
is this* rather than *how loud are the complaints*.

**That is the sentence to give any exam case study about performance analysis.** The skill being tested is
not knowing which blade to open. It is **knowing what to eliminate first** — and the evidence, not the
instinct, decides whether you roll back or hotfix.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Largest span blamed for the slowdown** | Q2, Q27, Q44 | Read from the bottom. It is waiting |
| **CPU checked before the dependency data** | Q1, Q25, Q33, Q43 | Subtract dependency time from request time |
| **Average instead of percentiles** | Q17, Q25, Q42, Q48 | P50 vs P99 is the whole diagnosis |
| **Ranking dependencies by duration alone** | Q10, Q34, Q37, Q43 | duration × count is total cost |
| **Absolute threshold for endpoint regression** | Q13, Q34, Q45 | 40 ms → 80 ms is a 100% regression |
| **`Perf` totals when attribution is needed** | Q7, Q32, Q39 | `VMProcess` and `ProcessName` |
| **`Available MBytes` read as "used"** | Q8, Q28 | It measures free. Falling is the problem |
| **Restarting before capturing evidence** | Q25 | The fault returns and the state is gone |
| **Rollback assumed always safest** | Q26, Q47 | It does not roll back a migration |
| **Ticket volume as the urgency signal** | Q25, Q46 | Burn rate against the error budget |
| **SLO lowered in response to a high burn rate** | Q3, Q23 | Then it constrains nothing |
| **Waiting for the SLO breach** | Q3, Q21, Q23 | The budget exists to act before it |
| **Smart detection expected to remediate** | Q20, Q30 | It notifies. Action groups act |
| **Smart detection confused with dynamic thresholds** | Q4 | Dynamic alerts must be created |
| **`sloTarget / 100` in integer arithmetic** | Q41 | `100.0`, or the budget becomes everything |
| **`fullouter` for endpoint degradation** | Q13, Q29 | A percentage change needs both sides |

---

# What to memorise

**In `challenge-50.md`:** lines **34–83**, **134–154**, **208–253**, **390–444**.

```text
THE RULE: READ THE TRACE FROM THE BOTTOM
  Frontend 50ms -> Gateway 30ms -> Order 4500ms -> Payment TIMEOUT
  Order is not slow. Order is WAITING. The answer is Payment.

THE SUBTRACTION
  request 1500ms - dependency 1200ms = 300ms of your code  ->  the DATABASE is the answer
  request 1500ms - dependency   50ms = 1450ms of your code ->  your code, or an undersized instance

ORDER OF INVESTIGATION  (each step narrows the next)
  1 percentiles            how bad, and for whom        P50 vs P99
  2 before / after         did the deployment own it    iff(timestamp < deployTime, ...)
  3 dependencies           where is the time            by P95 AND by impact
  4 end-to-end trace       inside one slow request      union on operation_Id
  5 infrastructure         only if 1-4 point there      Perf, then VMProcess
  6 burn rate              how urgently to act          vs the error budget
```

```kql
// PERCENTILE SUMMARY                                        (lines 36-43)
| summarize requestCount = count(), avgDuration = avg(duration),
    p50 = percentile(duration,50), p90 = ..., p95 = ..., p99 = ..., maxDuration = max(duration)

// DEPENDENCIES - two rankings, two questions                (lines 62-71, 134-144)
dependencies | summarize avgDuration = avg(duration), p95Duration = percentile(duration,95),
    callCount = count(), failRate = round(countif(success == false)*100.0/count(),1)
    by target, type, name | order by p95Duration desc            // slow PER CALL
| extend impact = avgDuration * slowCallCount | order by impact desc   // slow IN TOTAL  <- fix this first

// BEFORE / AFTER - one query, two rows                      (lines 74-83)
| where timestamp between ((deployTime - 2h) .. (deployTime + 2h))
| extend period = iff(timestamp < deployTime, "Before", "After")     // "1-Before"/"2-After" to force sort
| summarize avgDuration = ..., p95Duration = ..., errorRate = ..., requestCount = count() by period

// WHICH ENDPOINTS DEGRADED - inner join, PERCENTAGE change  (lines 190-201)
before | join kind=inner after on name
| extend degradationPct = round(((afterAvg - beforeAvg) / beforeAvg) * 100, 1)
| where degradationPct > 50        // 40ms -> 80ms is a regression an absolute threshold misses

// END-TO-END TRACE                                          (lines 148-154)
union (requests | where operation_Id == opId | extend itemType = "request"),
      (dependencies | ... "dependency"), (exceptions | ... "exception"), (traces | ... "trace")
| project timestamp, itemType, name, duration, success, target, resultCode | order by timestamp asc

// SELF-UPDATING DEPLOYMENT TIME                             (lines 169-174)
let deployTime = toscalar(customEvents | where name == "Deployment" | top 1 by timestamp desc
                          | project deployTime = timestamp);
```

```kql
// INFRASTRUCTURE                                            (lines 208-253, 528-534)
Perf | where CounterName == "% Processor Time" and InstanceName == "_Total"   // VM CPU total
Perf | where CounterName == "Available MBytes"        // memory FREE - falling is the problem
Perf | where CounterName == "Disk Reads/sec" or CounterName == "Disk Writes/sec"
Perf | where CounterName == "Bytes Sent/sec" or CounterName == "Bytes Received/sec"
Perf | where ObjectName == "K8SContainer" and CounterName == "cpuUsageNanoCores"
     | summarize avg(CounterValue / 1000000.0)        // nanocores -> MILLICORES

VMProcess | summarize avgCPU = avg(PercentProcessorTime) by ProcessName, bin(TimeGenerated, 5m)
//  Perf = the machine.  VMProcess = WHICH PROCESS.  needs the Microsoft-ServiceMap stream (Ch.47)

// App Service platform metrics via CLI                      (lines 260-280)
az monitor metrics list --metric "CpuPercentage" | "MemoryWorkingSet" | "Http5xx,Http4xx,Http2xx"
```

```text
ALERTING                                                     (lines 343-382)
Smart detection   ON BY DEFAULT, no rules to write, ML-learned
                  Failure Anomalies | Slow page load | Slow server response | Long dependency duration
                  it NOTIFIES. it does not remediate.
Static alert      --condition "avg requests/duration > 3000"
Dynamic alert     --condition "avg requests/duration > dynamic medium 3 of 5"
                  dynamic = learned | low|medium|high = sensitivity | 3 of 5 = periods that must breach
                  MUST BE CREATED - that is what separates it from smart detection

SLI / SLO / ERROR BUDGET                                     (lines 390-444, 452-470)
SLI            the measurement:  availability = successful/total | latency = under-threshold/total
SLO            the target:       99.9%
ERROR BUDGET   100% - SLO  ->  totalRequests * (1 - sloTarget / 100.0)     <- 100.0, not 100
BURN RATE      failedReq / dailyErrorBudget
               1.0  = spends the budget exactly at the end of the window
               >1.0 = exhausted early    <1.0 = budget left over = permission to take risk
MULTI-WINDOW   fast burn 14.4x over 1h   (= 2% of a 30-day budget in one hour)
               slow burn  6.0x over 6h   (catches a steady leak short windows never trip)

DECISION       80% budget at 50% of window = burn 1.6 -> reduce frequency, raise rigour. NOT a freeze.
               freeze is for budget EXHAUSTED. lowering the SLO is never the answer.
ROLLBACK vs HOTFIX
               rollback  fault is in YOUR code and cleanly reversible
               hotfix    fault is downstream, OR the deploy included a MIGRATION rollback would not undo
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Domain 5 complete. You have finished the course |
| 38–43 | Re-read the trap index and the order of investigation, then move on |
| 30–37 | Write the six-step order and the burn-rate definitions from memory, then retake |
| Below 30 | Redo Tasks 1, 2 and 7 hands-on against real telemetry before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 50.

:::danger The three rules

**Read the trace from the bottom.** The slow span with a failing child is the symptom; the child is the
cause.

**Subtract the dependency.** Request minus dependency is your code. Whatever is left is where to look.

**Let the budget set the urgency.** Burn rate above 1.0 means act; ticket volume means nothing.

And before rolling back, ask what the deployment did to the database.

:::
