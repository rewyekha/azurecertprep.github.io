---
sidebar_position: 4.5
toc_max_heading_level: 2
title: "Challenge 49: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 49 — AZ-400 exam questions

**48 questions** built only from what Challenge 49 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-49.md`**.

:::danger Read this before you start

KQL questions on this exam almost always hinge on **one operator choice**, and there are four that carry
most of the marks.

**`summarize ... by`** — aggregate per group. The difference between "the 5 slowest **requests**" and
"the 5 slowest **endpoints**".
**`toscalar()`** — turn a sub-query into a single number so you can compare against a baseline.
**`join kind=leftanti`** — rows on the left with **no match** on the right. The "what is new since the
deployment?" operator.
**`countif()`** — count conditionally inside a `summarize`, so a rate is one pass instead of two queries.

Two more facts decide the trickier questions.

**`customDimensions` is dynamic**, so string comparisons need `tostring()` or indexer notation.
**A baseline must exclude the present**, or the spike you are detecting is inside the number you are
comparing against.

The scenario at line 16 is an SRE team browsing charts by eye. Every query in this challenge replaces a
guess with an answer.

:::

---

# Section A — Multiple choice

---

## Q1

Find the top 5 slowest API **endpoints** in the last hour by P95, excluding endpoints with fewer than 10
requests. Which query is correct?

- A. `requests | where timestamp > ago(1h) | top 5 by duration`
- B. `requests | where timestamp > ago(1h) | summarize p95 = percentile(duration, 95), count = count() by name | where count > 10 | top 5 by p95`
- C. `requests | where timestamp > ago(1h) | where duration > 1000 | summarize count() by name`
- D. `requests | where timestamp > ago(1h) | order by duration | take 5`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-49.md`:** lines **148–157**.

```kql
| summarize
    avgDuration = avg(duration),
    p95Duration = percentile(duration, 95),
    requestCount = count()
    by name
| where requestCount > 10
```

**`by name` is what makes it about endpoints.** Without it, A and D return the five slowest **individual
requests** — which could all be the same endpoint, or five unrelated one-off outliers.

**Why the `requestCount > 10` filter matters more than it looks.** A percentile over three data points
is meaningless, and a single 30-second request on a rarely-called endpoint would top the list every time.

**Why C answers a different question** — how many requests were slower than a fixed threshold, which
depends entirely on a number you picked.

</details>

---

## Q2

Which KQL construct is essential for comparing a current metric against a historical baseline?

- A. `join`
- B. `toscalar()` with a sub-query
- C. `mv-expand`
- D. `parse`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-49.md`:** lines **236–244**.

```kql
let baseline = requests
| where timestamp between (ago(7d) .. ago(1h))
| summarize baselineErrorRate = countif(success == false) * 100.0 / count();
...
| extend threshold = toscalar(baseline) * 3
| where currentErrorRate > threshold
```

**`toscalar()` collapses a one-row, one-column result into a value** you can multiply and compare
against. Without it you have a **table**, and `where currentErrorRate > baseline` is not valid.

**Why A is the near-miss.** A join *could* combine the two, and it is far more machinery than needed for
a single number — you would need a synthetic key to join on, since there is no shared column.

</details>

---

## Q3

Which query finds exception types that did **not** exist before the deployment?

- A. `exceptions | where timestamp > deployTime | where type contains "Timeout"`
- B. `exceptions | where timestamp > deployTime | distinct type | join kind=leftanti (exceptions | where timestamp < deployTime | distinct type) on type`
- C. `exceptions | summarize count() by type | where count_ == 1`
- D. `exceptions | where timestamp > deployTime | summarize count() by type | order by count_ asc`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-49.md`:** lines **121–130**.

```kql
newExceptions
| join kind=leftanti oldExceptions on type
| project NewExceptionType = type
```

**`leftanti` returns left rows with no match on the right** — precisely "post-deployment types that never
appeared before".

**Why A requires you to already know the answer.** Filtering on `contains "Timeout"` finds the exception
you were told about; the point of the query is to find the one nobody has noticed yet.

**Why C and D find *rare* exceptions, not *new* ones.** An exception that has occurred once a week for a
year is rare and not new.

</details>

---

## Q4

An Azure DevOps team wants pipeline run durations and pass rates over the last quarter for capacity
planning. Which source?

- A. The Azure DevOps REST API for pipeline runs
- B. The Azure DevOps Analytics OData endpoint
- C. Application Insights custom events
- D. Azure Monitor activity logs

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-49.md`:** lines **288–301**.

```text
https://analytics.dev.azure.com/contoso/ContosoWeb/_odata/v4.0-preview/PipelineRuns?$filter=...&$select=PipelineRunId,CompletedDate,RunDuration,RunOutcome
```

**Analytics is built for aggregate historical queries**; the REST API is built for retrieving individual
records.

**Why A is the trap for anyone who has used the REST API.** It works — you page through runs and
aggregate client-side, which is what Challenge 48 does with `gh run list` and `jq`. Over a **quarter**
that is thousands of records and many round trips, and Analytics gives you `RunDuration` as a computed
field with `$filter` and `$select` applied server-side.

**Why C and D are the wrong systems entirely** — Application Insights monitors applications, and activity
logs record Azure control-plane operations.

</details>

---

## Q5

A query filtering `customDimensions.Environment == "production"` returns nothing, though the data exists.
Why?

- A. Custom dimensions are dynamic and need explicit casting
- B. The data has not been indexed
- C. `customDimensions` is not queryable
- D. The time range is wrong

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** lines **381** and **389–400**.

```kql
// Wrong: comparing dynamic value directly
| where customDimensions.Environment == "production"

// Correct: cast to string explicitly
| where tostring(customDimensions.Environment) == "production"

// Alternative: use indexer notation
| where customDimensions["Environment"] == "production"
```

**The comparison silently fails rather than erroring**, which is why this costs so much time. A dynamic
value compared to a string returns no match, and an empty result set looks like "no data" rather than
"wrong query".

**Both fixes are correct**, and `tostring()` is the more explicit of the two.

</details>

---

## Q6

A "error rate spike" alert fires every five minutes during normal operation. What are the causes?

- A. The baseline includes the current period, and the threshold has no minimum traffic guard
- B. The alert frequency is too low
- C. The query has a syntax error
- D. The action group is misconfigured

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** lines **409** and **417–429**.

```kql
let baseline = requests
| where timestamp between (ago(7d) .. ago(2h))  // Exclude recent 2 hours
...
| where requestCount > 20  // Minimum traffic threshold
| where currentErrorRate > (toscalar(baseline) * 3)
| where currentErrorRate > 1.0  // Minimum absolute error rate
```

**Three fixes for three distinct causes.** Excluding recent data stops the spike contaminating its own
baseline. The traffic minimum stops 1 failure in 2 requests reading as 50%. The absolute floor stops
0.3% being "3× worse" than 0.1%.

**That last one is the subtle one, and it is the most common cause of alert fatigue.** On a healthy
service the baseline error rate is tiny — so a *ratio* threshold trips on noise. **Ratio and absolute
floor together** is what makes a dynamic threshold usable.

</details>

---

## Q7

What does `countif(success == false)` do inside a `summarize`?

- A. Counts only rows where the condition is true, alongside other aggregates
- B. Filters rows before aggregation
- C. Returns a boolean
- D. Counts distinct values

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** lines **78–82**.

```kql
| summarize
    totalRequests = count(),
    failedRequests = countif(success == false)
    by bin(timestamp, 5m)
| extend errorRate = (failedRequests * 100.0) / totalRequests
```

**Both numbers come from one pass**, which is what makes the rate computable in a single query.

**Why B is the important distinction.** `where success == false` **before** the summarize would remove
the successful rows entirely — so `count()` would equal `countif()` and the rate would always be 100%.

</details>

---

## Q8

Why is `100.0` used rather than `100` in the error rate calculation?

- A. To force floating-point division rather than integer truncation
- B. For readability
- C. It is required by `extend`
- D. To round to one decimal place

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** line **82**.

```kql
| extend errorRate = (failedRequests * 100.0) / totalRequests
```

**Integer division truncates.** With `100`, an error rate of 0.4% computes as `0` — so every rate below
1% reports as zero, and your alerting silently ignores the entire range where a healthy service lives.

**Rounding is a separate operation** (line 116): `round(..., 1)`.

</details>

---

## Q9

What does `bin(timestamp, 5m)` do?

- A. Groups rows into 5-minute buckets so results form a time series
- B. Filters to the last 5 minutes
- C. Limits results to 5 rows
- D. Rounds durations to 5-minute precision

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** lines **81** and **198**.

**`bin` is what turns a table of individual events into a chart.** Without it, `summarize` produces one
row for the whole period and `render timechart` has nothing to plot.

**Bucket size is a trade-off worth stating.** Smaller bins show spikes and are noisier; larger bins are
smooth and hide short incidents. The challenge uses 5m for error rate (line 198), 15m for percentiles
(line 144) and 1m for request rate (line 182) — each matched to how fast the thing being measured moves.

</details>

---

## Q10

What does `join kind=fullouter` provide in the before/after comparison?

- A. Every exception type from either window, including types present in only one
- B. Only types present in both windows
- C. Only new types
- D. Only disappeared types

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** lines **113–117**.

```kql
beforeDeployment
| join kind=fullouter afterDeployment on type
| extend type = coalesce(type, type1)
| extend changePercent = round(((afterCount - beforeCount) * 100.0) / max_of(beforeCount, 1), 1)
```

**`fullouter` is required because the interesting cases are the asymmetric ones** — a type that appeared
only after (a regression) or only before (something you fixed). An `inner` join would drop both.

**And that is why `coalesce(type, type1)` is needed** (line 115): for rows present on only one side, the
other side's `type` column is null.

**`max_of(beforeCount, 1)` prevents division by zero** for a brand-new type — the reason a new exception
shows a large positive change rather than an error.

</details>

---

## Q11

What does `serialize` enable at line 225?

- A. Row order is fixed so `prev()` can reference earlier rows
- B. Results are exported to JSON
- C. The query runs on a single node
- D. Results are cached

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** lines **224–227**.

```kql
| order by timestamp asc
| serialize
| extend movingAvg = (prev(avgDuration, 1, avgDuration) + prev(avgDuration, 2, avgDuration) + avgDuration) / 3.0
| extend isAnomaly = avgDuration > (movingAvg * 2)
```

**KQL results are unordered by default**, and window functions like `prev()` need a defined sequence.
`serialize` provides it.

**Note the third argument to `prev()`** — `avgDuration` as the default. For the first two rows there is
no previous value, so it falls back to the current one, which keeps the moving average defined instead of
null.

</details>

---

## Q12

What does `mv-expand timestamp = range(beforeWindow, afterWindow, 5m)` produce?

- A. One row per 5-minute step across the window around each deployment
- B. A single row containing an array of timestamps
- C. A filter on the time range
- D. A chart

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** lines **208–215**.

```kql
| extend beforeWindow = deployTime - 30m
| extend afterWindow = deployTime + 30m
| mv-expand timestamp = range(beforeWindow, afterWindow, 5m) to typeof(datetime)
```

**`range()` builds the array; `mv-expand` turns each element into its own row.** The result is a
timestamp column the requests summary can be joined against.

**Which is what makes `relativeMinutes` possible** (line 216): every deployment's window is normalised to
minutes from its own deployment, so several deployments can be compared on one chart regardless of when
they happened.

</details>

---

## Q13

What does `datetime_diff('minute', timestamp, deployTime)` return?

- A. The number of minutes between the two, negative before the deployment
- B. A duration in milliseconds
- C. A formatted string
- D. The absolute difference

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** line **216**.

```kql
| extend relativeMinutes = datetime_diff('minute', timestamp, deployTime)
```

**Order matters and the sign is the point.** `-30` to `0` is the half hour before; `0` to `+30` is after.

**That normalisation is the whole trick of the query.** Plotting `relativeMinutes` on the x-axis puts
every deployment's before-and-after on the same scale, so a pattern across releases becomes visible.

</details>

---

## Q14

Which table holds outbound calls to databases and external APIs?

- A. `dependencies`
- B. `requests`
- C. `traces`
- D. `customEvents`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** lines **170–177**.

```kql
dependencies
| summarize
    avgDuration = avg(duration),
    failRate = round(countif(success == false) * 100.0 / count(), 1),
    callCount = count()
    by target, name
```

**`requests` is inbound — calls *to* your service. `dependencies` is outbound — calls *from* it.** That
one distinction answers most "which table" questions.

**And `by target, name` groups by *what* was called and *which operation***, which is how you find the
one slow stored procedure inside an otherwise healthy database.

</details>

---

## Q15

What does `let` provide?

- A. A named variable — scalar or tabular — reusable within the query
- B. A persistent saved function
- C. A column alias
- D. A join key

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** lines **74–77** and **105–112**.

```kql
let timeRange = ago(24h);
let errorThreshold = 5.0;
```

```kql
let beforeDeployment = exceptions
    | where timestamp between ((deploymentTime - windowSize) .. deploymentTime)
    | summarize beforeCount = count() by type;
```

**Both forms matter.** A scalar `let` puts a threshold at the top where it is easy to change; a tabular
`let` names an intermediate result so a complex query reads as steps rather than nesting.

**Note the semicolons** — each `let` statement is terminated, and the final query has none.

</details>

---

## Q16

Which OData query parameter selects specific fields?

- A. `$select`
- B. `$filter`
- C. `$orderby`
- D. `$top`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** line **293**.

```text
$filter=CompletedDate gt 2024-11-01T00:00:00Z and PipelineName eq 'Production-Deploy'
&$select=PipelineRunId,CompletedDate,RunDuration,RunOutcome
&$orderby=CompletedDate desc
&$top=50
```

**Four parameters, four jobs: filter rows, choose columns, sort, limit.** They map onto KQL's `where`,
`project`, `order by` and `take`.

**And OData uses word operators** — `gt`, `eq`, `lt` — rather than symbols. That is the syntactic
difference the exam checks.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** KQL operators appear in the fundamentals task? (Choose three.)

- A. `where` — filter rows
- B. `summarize` — aggregate
- C. `extend` — add calculated columns
- D. `merge`
- E. `groupby`
- F. `select`

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-49.md`:** lines **32–54**.

**D, E and F are SQL vocabulary that KQL does not use**, and the exam offers them for exactly that
reason. The KQL equivalents are `join`, `summarize ... by`, and `project`.

**If you think in SQL, translate once and remember the mapping** — it is the single fastest way to stop
losing marks on syntax questions.

</details>

---

## Q18

Which **three** are needed to make a dynamic-threshold alert stable? (Choose three.)

- A. A baseline window that excludes recent data
- B. A minimum request count
- C. A minimum absolute error rate
- D. A shorter evaluation frequency
- E. A larger action group
- F. Disabling the alert at night

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-49.md`:** lines **419–429**.

**Each defeats a different false positive.** A stops the spike inflating its own baseline. B stops tiny
traffic volumes producing wild percentages. C stops a healthy service's near-zero baseline making any
blip look like a 3× regression.

**Why D makes it worse** — evaluating more often on an unstable condition produces more false alerts, not
fewer.

**Why F is the response teams eventually resort to**, and it is the admission that the alert was never
tuned.

</details>

---

## Q19

Which **two** correctly compare a dynamic custom dimension to a string? (Choose two.)

- A. `where tostring(customDimensions.Environment) == "production"`
- B. `where customDimensions["Environment"] == "production"`
- C. `where customDimensions.Environment == "production"`
- D. `where customDimensions == "production"`
- E. `where toint(customDimensions.Environment) == "production"`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-49.md`:** lines **394–400**.

**Cast explicitly, or use indexer notation.** Both resolve the dynamic type.

**Why C is the failing form from Break scenario 1**, and why it is dangerous: it returns an **empty
result** rather than an error, so the query looks correct and the conclusion is "no data".

**Why E is nonsense worth naming** — casting to an integer and comparing against a string.

</details>

---

## Q20

Which **two** are true of `join kind=leftanti`? (Choose two.)

- A. It returns left rows with no match on the right
- B. It is how you find values that are new in one set
- C. It returns rows matching on both sides
- D. It returns all rows from both sides
- E. It requires a `summarize` first

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-49.md`:** lines **128–130** and **261–265**.

```kql
| join kind=leftanti (
    exceptions
    | where timestamp between (ago(7d) .. ago(15m))
    | distinct type
) on type
```

**"What is here that was not there before" is the leftanti question**, and it appears twice in this
challenge — once for post-deployment analysis and once as a live alert.

**Why C and D are `inner` and `fullouter`** respectively (line 114 uses `fullouter`).

**Why E is false, and the `distinct` in these queries is not a summarize** — it deduplicates, which is
what makes the join a set comparison rather than a row comparison.

</details>

---

## Q21

Which **two** does the response-time degradation query use to detect an anomaly? (Choose two.)

- A. `serialize` to fix row order
- B. `prev()` to build a moving average
- C. `percentile()` over the whole window
- D. `toscalar()` for a baseline
- E. `mv-expand` for a time range

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-49.md`:** lines **225–227**.

```kql
| serialize
| extend movingAvg = (prev(avgDuration, 1, avgDuration) + prev(avgDuration, 2, avgDuration) + avgDuration) / 3.0
| extend isAnomaly = avgDuration > (movingAvg * 2)
```

**A moving average adapts to the recent trend** — so it catches a sudden doubling without alerting during
a gradual, expected rise in load.

**Which is a different technique from the baseline approach at line 236.** The baseline compares against
*seven days*; the moving average compares against *the last fifteen minutes*. Different sensitivities,
different purposes: one finds a regression, the other finds a spike.

</details>

---

## Q22

Which **two** OData entity sets does the challenge query? (Choose two.)

- A. `PipelineRuns`
- B. `WorkItems`
- C. `Commits`
- D. `Deployments`
- E. `Builds`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-49.md`:** lines **293** and **297**.

```text
_odata/v4.0-preview/PipelineRuns?$filter=... &$select=PipelineRunId,CompletedDate,RunDuration,RunOutcome
_odata/v4.0-preview/WorkItems?$filter=... &$select=WorkItemId,Title,CycleTimeDays,LeadTimeDays
```

**`TestRuns` is the third** (line 301). Three entity sets covering pipelines, work and tests.

**And note `CycleTimeDays` and `LeadTimeDays` are computed fields** Analytics provides — you do not
subtract dates yourself. That is a substantial part of why Analytics beats the REST API for this
(Q4).

</details>

---

## Q23

Which **two** make a workbook query reusable across environments? (Choose two.)

- A. A `TimeRange` parameter referenced in the query
- B. An `Environment` parameter populated by a query over `cloud_RoleName`
- C. Hard-coding `ago(24h)`
- D. A separate workbook per environment
- E. A fixed `cloud_RoleName` literal

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-49.md`:** lines **331–349**.

```json
          {
            "name": "Environment",
            "type": 2,
            "query": "requests | distinct cloud_RoleName"
          }
```

**The `Environment` parameter is populated *from the data itself***, so a new service appears in the
dropdown without anyone editing the workbook.

**And `cloud_RoleName` is the standard field for "which service"** — set by `OTEL_SERVICE_NAME` or the
SDK's role name (Challenge 47 Q41).

**Why C and E are the hard-coded forms** that force D — one workbook per environment, all drifting apart.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso's SRE team must answer, after every deployment, whether it caused an error spike or
a response-time regression — with reusable queries, alerts that do not cry wolf, and a shared workbook.

---

## Q24

**Proposed solution:** Write a before/after exception comparison using two `let` windows joined
`fullouter` with a change percentage. Add a `leftanti` query for exception types new since the
deployment. Chart P50/P95/P99 binned at 15 minutes and error rate binned at 5 minutes. Create a
scheduled-query alert comparing the last 5 minutes against a 7-day baseline that excludes the last 2
hours, with a minimum request count and an absolute error-rate floor. Publish a workbook with `TimeRange`
and `Environment` parameters, and grant the SRE team Monitoring Reader.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-49.md`:** lines **105–130**, **137–145**, **193–199**, **419–429**, **331–357**,
**369–372**.

| Question | Query |
|---|---|
| Did exceptions increase? | `fullouter` before/after with change percent |
| Is anything **new**? | `leftanti` on `distinct type` |
| Did latency regress? | Percentiles binned over time |
| Alert me automatically | Scheduled query, 3× baseline, guarded |
| Can the team self-serve? | Parameterised workbook + Monitoring Reader |

**The two exception queries answer different questions and both are needed.** A type that doubled from
200 to 400 is a regression; a type that appeared from nothing is a **new failure mode**, and it will not
stand out in a percentage table.

</details>

---

## Q25

**Proposed solution:** Browse the Application Insights failures blade after each deployment. Where a
query is needed, filter with `where customDimensions.Environment == "production"`. Create an alert that
fires whenever the error rate exceeds three times the 7-day average, computed over the full 7 days
including the present. Save queries in a shared text file.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures.**

**Browsing the blade is the scenario, not the solution** — line 16 describes exactly this.

**`customDimensions.Environment == "production"` returns nothing** (Q5). The filter silently matches no
rows, so every query built on it reports a healthy production.

**A baseline including the present dilutes the spike it is meant to detect**, and with no minimum traffic
or absolute floor it also fires constantly at low volume — both halves of Break scenario 2.

**And a text file is not a shared artifact.** Nobody knows which version is current, nothing is
parameterised, and there is no access model — which the workbook plus Monitoring Reader provides (lines
369–372).

</details>

---

## Q26

**Proposed solution:** Write the before/after exception comparison and the `leftanti` new-exception query.
Chart percentiles and error rate over time. Create a scheduled-query alert comparing the last 5 minutes
against a 7-day baseline excluding the last 2 hours. Publish a parameterised workbook and grant
Monitoring Reader. Remove the minimum request count and absolute error-rate floor from the alert, so no
genuine spike is ever missed.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare with Q24, which is otherwise identical.

**The stated reasoning inverts how alerting fails.** Removing the guards does not catch more real spikes;
it catches more **noise**, and noise is what causes real spikes to be missed.

**Concretely, at 3 AM with 4 requests in a 5-minute window, one failure is a 25% error rate.** Against a
baseline of 0.2%, that is 125× the threshold. The alert fires, someone is paged, and nothing is wrong.

**Do that nightly for two weeks and the alert is muted** — at which point the *next* genuine spike goes
unnoticed for exactly as long as it would have without any alert at all.

**Line 427 and line 429 are the two guards**, and they exist because Break scenario 2 already happened:

```kql
| where requestCount > 20  // Minimum traffic threshold
| where currentErrorRate > 1.0  // Minimum absolute error rate
```

**An alert's job is not to fire on every anomaly. It is to be believed when it fires.**

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — operators

| # | Statement | Answer |
|---|---|---|
| 1 | `summarize ... by name` aggregates per endpoint |  |
| 2 | `top 5 by duration` returns the 5 slowest endpoints |  |
| 3 | `countif()` counts conditionally inside a summarize |  |
| 4 | `project` selects columns |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `summarize ... by name` aggregates per endpoint | **Yes** |
| 2 | `top 5 by duration` returns the 5 slowest endpoints | **No** |
| 3 | `countif()` counts conditionally inside a summarize | **Yes** |
| 4 | `project` selects columns | **Yes** |

**In `challenge-49.md`:** lines **154**, **148–157**, **80**, **52**.

Row 2 is Q1's distinction. **Without `by`, you get the slowest individual *requests*** — possibly five
outliers on the same endpoint.

</details>

---

## Q28 — baselines and alerts

| # | Statement | Answer |
|---|---|---|
| 1 | `toscalar()` converts a one-value table into a scalar |  |
| 2 | A baseline should include the current window |  |
| 3 | A minimum request count reduces false positives |  |
| 4 | A ratio threshold alone is sufficient on a healthy service |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `toscalar()` converts a one-value table into a scalar | **Yes** |
| 2 | A baseline should include the current window | **No** |
| 3 | A minimum request count reduces false positives | **Yes** |
| 4 | A ratio threshold alone is sufficient on a healthy service | **No** |

**In `challenge-49.md`:** lines **243**, **420**, **427**, **429**.

Rows 2 and 4 are the two halves of Break scenario 2, and row 4 is the one people underestimate: **when
the baseline is near zero, every ratio is enormous.**

</details>

---

## Q29 — joins

| # | Statement | Answer |
|---|---|---|
| 1 | `leftanti` returns left rows with no right match |  |
| 2 | `fullouter` keeps rows present on only one side |  |
| 3 | `inner` would surface exception types that only appeared after deployment |  |
| 4 | `coalesce(type, type1)` is needed after a fullouter join |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `leftanti` returns left rows with no right match | **Yes** |
| 2 | `fullouter` keeps rows present on only one side | **Yes** |
| 3 | `inner` would surface exception types that only appeared after deployment | **No** |
| 4 | `coalesce(type, type1)` is needed after a fullouter join | **Yes** |

**In `challenge-49.md`:** lines **129**, **114**, **113–115**.

Row 3 is why the before/after query uses `fullouter`. **An inner join drops exactly the rows you care
most about** — the new ones and the disappeared ones.

</details>

---

## Q30 — types and time

| # | Statement | Answer |
|---|---|---|
| 1 | `customDimensions` values need casting for string comparison |  |
| 2 | A failed dynamic comparison raises an error |  |
| 3 | `bin(timestamp, 5m)` creates 5-minute buckets |  |
| 4 | `serialize` is required before `prev()` |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `customDimensions` values need casting for string comparison | **Yes** |
| 2 | A failed dynamic comparison raises an error | **No** |
| 3 | `bin(timestamp, 5m)` creates 5-minute buckets | **Yes** |
| 4 | `serialize` is required before `prev()` | **Yes** |

**In `challenge-49.md`:** lines **381**, **392**, **198**, **225–226**.

Row 2 is what makes row 1 expensive. **An empty result set looks like an absence of data**, and the query
is only questioned after someone checks the portal and sees the events are there.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each question to the KQL construct that answers it.

| Question | Construct |
|---|---|
| Which endpoints are slowest at P95? |  |
| What error types are new since the deployment? |  |
| Did each error type increase or decrease? |  |
| Is the current rate 3× the historical average? |  |
| What is the error rate per 5 minutes? |  |
| Is this reading double the recent trend? |  |

**Options:** `countif()` + `bin(timestamp, 5m)` · `join kind=fullouter` · `join kind=leftanti` · `serialize` + `prev()` · `summarize percentile(duration, 95) by name` · `toscalar()` on a baseline sub-query

<details>
<summary>Show answer</summary>

| Question | Construct |
|---|---|
| Which endpoints are slowest at P95? | **`summarize percentile(duration, 95) by name`** |
| What error types are new since the deployment? | **`join kind=leftanti`** |
| Did each error type increase or decrease? | **`join kind=fullouter`** |
| Is the current rate 3× the historical average? | **`toscalar()` on a baseline sub-query** |
| What is the error rate per 5 minutes? | **`countif()` + `bin(timestamp, 5m)`** |
| Is this reading double the recent trend? | **`serialize` + `prev()`** |

**In `challenge-49.md`:** lines **150–156**, **129**, **114**, **243**, **80–82**, **225–227**.

**Six questions, six operators.** Being able to name the operator before writing the query is most of
what this challenge is testing.

</details>

---

## Q32

Match each table to what it contains.

| Table | Contains |
|---|---|
| `requests` |  |
| `dependencies` |  |
| `exceptions` |  |
| `customEvents` |  |
| `traces` |  |

**Options:** Application log messages · Events your code tracked, including deployment annotations · Inbound calls to your service · Outbound calls to databases and APIs · Thrown exceptions with type and stack

<details>
<summary>Show answer</summary>

| Table | Contains |
|---|---|
| `requests` | **Inbound calls to your service** |
| `dependencies` | **Outbound calls to databases and APIs** |
| `exceptions` | **Thrown exceptions with type and stack** |
| `customEvents` | **Events your code tracked, including deployment annotations** |
| `traces` | **Application log messages** |

**In `challenge-49.md`:** lines **33**, **170**, **90**, **190–192**, **32–84**.

**In and out is the distinction that matters most.** A slow `requests` row with a slow `dependencies`
row on the same `operation_Id` means the delay was downstream, not in your code.

</details>

---

## Q33

Arrange the steps to answer "did the 3 PM deployment cause the error spike?"

**Items:** Compare exception counts before and after with `fullouter` · Chart error rate binned at 5
minutes across the window · Identify the deployment time from `customEvents` · Run a `leftanti` query for
new exception types · Turn the finding into a scheduled-query alert

<details>
<summary>Show answer</summary>

### Answer

1. Identify the deployment time from `customEvents` — lines **190–192**
2. Chart error rate binned at 5 minutes across the window — lines **193–199**
3. Compare exception counts before and after with `fullouter` — lines **105–118**
4. Run a `leftanti` query for new exception types — lines **121–130**
5. Turn the finding into a scheduled-query alert — lines **273–283**

**Step 1 first, because everything else is relative to it.** Without the annotation there is no
`deploymentTime` to build windows around — which is Challenge 46's task, and the reason it exists.

**Steps 3 and 4 answer different questions and both are needed.** *More of the same* is a regression;
*something entirely new* is a new failure mode, and it never stands out in a percentage table because
its "before" count is zero.

**Step 5 is what stops the next investigation being manual** — the scenario at line 16 restated.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Query returns nothing though data exists |  |
| Error rate always reports 0 |  |
| Error rate always reports 100% |  |
| Alert fires every 5 minutes |  |
| `prev()` returns unexpected values |  |
| New exception types never appear |  |

**Options:** Baseline includes the present; no traffic or absolute guard · Dynamic `customDimensions` compared without casting · `inner` join instead of `leftanti` · Integer division — `100` instead of `100.0` · No `serialize`, so row order is undefined · `where success == false` before the summarize

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| Query returns nothing though data exists | **Dynamic `customDimensions` compared without casting** |
| Error rate always reports 0 | **Integer division — `100` instead of `100.0`** |
| Error rate always reports 100% | **`where success == false` before the summarize** |
| Alert fires every 5 minutes | **Baseline includes the present; no traffic or absolute guard** |
| `prev()` returns unexpected values | **No `serialize`, so row order is undefined** |
| New exception types never appear | **`inner` join instead of `leftanti`** |

**In `challenge-49.md`:** lines **392**, **82**, **80**, **420–429**, **225**, **129**.

**Every one of these produces a *plausible* result rather than an error**, which is why KQL mistakes cost
so much time. A query that returns zero errors looks like good news.

</details>

---

## Q35

Match each source to what it is for.

| Source | For |
|---|---|
| Application Insights KQL |  |
| Azure DevOps Analytics OData |  |
| Azure DevOps REST API |  |
| Azure Workbook |  |
| Scheduled query alert |  |

**Options:** A KQL query evaluated on a schedule, with actions · A shared, parameterised dashboard over KQL · Application telemetry — requests, exceptions, dependencies · Historical pipeline, work item and test analytics · Individual record retrieval and automation

<details>
<summary>Show answer</summary>

| Source | For |
|---|---|
| Application Insights KQL | **Application telemetry — requests, exceptions, dependencies** |
| Azure DevOps Analytics OData | **Historical pipeline, work item and test analytics** |
| Azure DevOps REST API | **Individual record retrieval and automation** |
| Azure Workbook | **A shared, parameterised dashboard over KQL** |
| Scheduled query alert | **A KQL query evaluated on a schedule, with actions** |

**In `challenge-49.md`:** lines **31–84**, **288–301**, **288**, **321–362**, **273–283**.

**Analytics versus REST is the pairing the exam tests** (Q4): aggregate history versus individual
records.

</details>

---

# Section F — Hot area

---

## Q36

```kql
requests
| where timestamp > ago(4h)
| summarize
    p95Duration = [BLANK 1](duration, 95),
    requestCount = [BLANK 2]()
    [BLANK 3] name
| where requestCount > 10
| order by p95Duration desc
| take 10
```

- **BLANK 1:** `percentile` / `avg` / `max` / `percentile_array`
- **BLANK 2:** `count` / `dcount` / `sum` / `countif`
- **BLANK 3:** `by` / `on` / `group by` / `over`

<details>
<summary>Show answer</summary>

### Answer: `percentile`, `count`, `by`

**In `challenge-49.md`:** lines **148–157**.

**`by` is KQL's grouping keyword** — `group by` is SQL and does not parse here.

**And `dcount` is the substantive distractor:** it counts *distinct* values, so it would return how many
unique somethings there were rather than how many requests. The filter needs volume, not variety.

</details>

---

## Q37

```kql
let baseline = requests
| where timestamp between (ago(7d) .. [BLANK 1])
| summarize baselineErrorRate = countif(success == false) * 100.0 / count();
requests
| where timestamp > ago(5m)
| summarize currentErrorRate = ..., requestCount = count()
| where requestCount > [BLANK 2]
| where currentErrorRate > ([BLANK 3](baseline) * 3)
| where currentErrorRate > 1.0
```

- **BLANK 1:** `ago(2h)` / `now()` / `ago(5m)` / `ago(7d)`
- **BLANK 2:** `20` / `0` / `1` / `1000`
- **BLANK 3:** `toscalar` / `tostring` / `toreal` / `materialize`

<details>
<summary>Show answer</summary>

### Answer: `ago(2h)`, `20`, `toscalar`

**In `challenge-49.md`:** lines **419–429**.

**`ago(2h)` excludes recent data from the baseline** so the incident does not raise the number it is
being compared against.

**`toscalar` is required for the comparison.** `toreal` converts a value's type; it does not collapse a
table into one.

**And note all three guards are present in the fixed version** — the exclusion, the traffic minimum, and
the absolute floor at line 429.

</details>

---

## Q38

```kql
let deploymentTime = datetime(2024-11-15T14:15:00Z);
exceptions
| where timestamp > deploymentTime
| [BLANK 1] type
| join [BLANK 2] (
    exceptions
    | where timestamp between (ago(7d) .. deploymentTime)
    | distinct type
) on type
```

- **BLANK 1:** `distinct` / `summarize` / `project` / `take`
- **BLANK 2:** `kind=leftanti` / `kind=inner` / `kind=fullouter` / `kind=rightouter`

<details>
<summary>Show answer</summary>

### Answer: `distinct`, `kind=leftanti`

**In `challenge-49.md`:** lines **122–129**.

**`distinct` makes each side a set of types** rather than a list of occurrences, which is what turns the
join into a set difference.

**`kind=inner` would return types present in *both* windows** — the exact opposite of what "new" means,
and the most common wrong answer because inner is the default people reach for.

</details>

---

## Q39

```kql
requests
| where timestamp > ago(24h)
| summarize
    totalReq = [BLANK 1](),
    failedReq = [BLANK 2](success == false)
    by [BLANK 3](timestamp, 5m)
| extend errorPercent = round((failedReq * [BLANK 4]) / totalReq, 2)
```

- **BLANK 1:** `count` / `sum` / `dcount` / `countif`
- **BLANK 2:** `countif` / `count` / `sumif` / `iff`
- **BLANK 3:** `bin` / `floor` / `round` / `range`
- **BLANK 4:** `100.0` / `100` / `1.0` / `0.01`

<details>
<summary>Show answer</summary>

### Answer: `count`, `countif`, `bin`, `100.0`

**In `challenge-49.md`:** lines **195–199**.

**`100.0` rather than `100` is the one that silently breaks the query** (Q8). With an integer, every
error rate below 1% computes as `0` — so the chart shows a flat zero line through the entire range where
a healthy service actually operates.

**And `bin` is what produces the time series.** Without it there is a single row and nothing to plot.

</details>

---

## Q40

```kql
requests
| where [BLANK 1](customDimensions.Environment) == "production"
```

- **BLANK 1:** `tostring` / `toint` / `todynamic` / `parse_json`

<details>
<summary>Show answer</summary>

### Answer: `tostring`

**In `challenge-49.md`:** line **396**.

```kql
| where tostring(customDimensions.Environment) == "production"
```

**Or the indexer form** at line 400, which is equivalent:

```kql
| where customDimensions["Environment"] == "production"
```

**`parse_json` is the distractor for people who know the field is JSON.** It converts a *string* into
dynamic — the opposite direction. `customDimensions` is already dynamic; the problem is getting a
**string** back out of it.

</details>

---

## Q41

```text
https://analytics.dev.azure.com/contoso/ContosoWeb/_odata/v4.0-preview/[BLANK 1]
  ?$filter=CompletedDate [BLANK 2] 2024-11-01T00:00:00Z
  &$select=PipelineRunId,CompletedDate,RunDuration,RunOutcome
  &$orderby=CompletedDate desc
  &$top=50
```

- **BLANK 1:** `PipelineRuns` / `Builds` / `Releases` / `Runs`
- **BLANK 2:** `gt` / `>` / `after` / `since`

<details>
<summary>Show answer</summary>

### Answer: `PipelineRuns`, `gt`

**In `challenge-49.md`:** line **293**.

**OData uses word operators** — `gt`, `ge`, `lt`, `le`, `eq`, `ne` — because `>` and `&` are reserved in a
URL. That is the syntactic tell the exam checks.

**And the entity set is `PipelineRuns`**, alongside `WorkItems` (line 297) and `TestRuns` (line 301).

</details>

---

# Section G — Case study

## Case study: Contoso deployment forensics

### Background

Contoso Ltd's SRE team must answer two questions after every deployment: **"Did the last deployment cause
the error spike?"** and **"Is the response-time regression correlated with the 3 PM release?"** Today they
**browse Application Insights looking for anomalies** without structured queries.

### Requirements

**Investigation**

- Exception counts must be comparable across a window before and after a deployment, including types that
  appear on only one side
- Exception types that are entirely new since a deployment must be identifiable
- Response-time percentiles and error rate must be chartable over time
- Slow endpoints must be ranked by P95, excluding low-traffic noise

**Alerting**

- An alert must compare the current error rate against a historical baseline
- The alert must not fire during normal operation, including overnight low traffic

**Sharing**

- Queries must be reusable by the whole team, parameterised by time range and environment
- The SRE team must have read access without being able to change the resource

---

## Q42

How should the before/after exception comparison be written?

- A. Two `let` windows joined `fullouter` on `type`, with `coalesce` and a change percentage guarded by
  `max_of(beforeCount, 1)`
- B. Two queries run separately and compared by eye
- C. An `inner` join between the two windows
- D. `summarize count() by type` over the whole day

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** lines **107–118**.

```kql
beforeDeployment
| join kind=fullouter afterDeployment on type
| extend type = coalesce(type, type1)
| extend changePercent = round(((afterCount - beforeCount) * 100.0) / max_of(beforeCount, 1), 1)
```

**"Including types that appear on only one side" is the requirement**, and it selects `fullouter`
specifically.

**Why C fails on exactly that clause** — an inner join drops the new types and the resolved ones, which
are the two most interesting rows in the result.

**Why D loses the before/after structure entirely**, and **why B is the scenario at line 16**.

</details>

---

## Q43

How should new exception types be identified?

- A. `distinct type` after the deployment, `leftanti` joined against `distinct type` from the preceding
  seven days
- B. Filtering on the exception name the team already suspects
- C. Exceptions with a count of 1
- D. Sorting post-deployment exceptions ascending by count

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** lines **121–130**.

**Seven days of history is the comparison set**, which matters: a type that last occurred five days ago
is not new, even though it is absent from the half hour before the deployment.

**Why B requires knowing the answer first** (Q3), and **why C and D find rare rather than new** — a
long-standing exception that fires once a week satisfies both and predates the release by months.

</details>

---

## Q44

How should the alert avoid firing overnight?

- A. A baseline excluding the last 2 hours, plus `requestCount > 20` and an absolute error-rate floor
- B. Disabling the alert between midnight and 6 AM
- C. Raising the ratio from 3× to 10×
- D. Increasing the evaluation window to 1 hour

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** lines **419–429**.

**The traffic minimum is the part that specifically addresses overnight.** Low volume is what makes a
percentage unstable — one failure in four requests is 25%, and no ratio threshold survives that.

**Why B creates a blind window** in which a real incident goes unalerted, and it is the workaround teams
adopt after the alert has already lost their trust.

**Why C reduces sensitivity everywhere** to fix a problem that only occurs at low volume — so a genuine
daytime 4× spike now goes unreported.

**Why D helps and is not sufficient.** A longer window accumulates more requests, and at 3 AM an hour may
still be a handful of them.

</details>

---

## Q45

Which **two** satisfy the sharing requirements? (Choose two.)

- A. A workbook with `TimeRange` and `Environment` parameters
- B. Granting the SRE team Monitoring Reader on the Application Insights resource
- C. Emailing query text to the team
- D. Granting Contributor so the team can edit queries
- E. A screenshot in the runbook

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-49.md`:** lines **331–357** and **369–372**.

```bash
az role assignment create \
  --assignee <sre-team-group-id> \
  --role "Monitoring Reader" \
  --scope ".../components/ai-contoso-webapp"
```

**Monitoring Reader is read without modify**, which is precisely the requirement — and it is the
least-privilege answer, the same instinct as Challenge 42's Key Vault Secrets User.

**Why D over-grants.** Contributor could delete the Application Insights resource, and the requirement
says read.

**Why C and E are not artifacts.** Neither can be parameterised, neither has a version, and both are
stale the moment a schema changes.

</details>

---

## Q46

How should slow endpoints be ranked?

- A. `summarize percentile(duration, 95) by name`, filtered to endpoints with more than 10 requests
- B. `top 10 by duration`
- C. `summarize avg(duration) by name`
- D. `where duration > 2000 | count()`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** lines **148–157**.

**Two requirements, two clauses:** `by name` makes it about endpoints, and the count filter excludes
low-traffic noise.

**Why C is the version most people write, and why it hides the problem.** An endpoint where 99% of calls
take 50 ms and 1% take 20 seconds has a healthy average and a terrible P95 — and it is the 1% who are
complaining.

**Why B returns individual requests** (Q1), and **why D depends on a threshold you invented.**

</details>

---

## Q47

Six weeks after the queries go live, the SRE team reports that the error-rate workbook chart has shown a
flat zero line for a fortnight, while the failures blade clearly shows failing requests. The query was
recently edited to "tidy up the arithmetic".

What happened, and what is the fix?

- A. The `100.0` was changed to `100`, so integer division truncates every rate below 1% to zero —
  restore the floating-point literal
- B. The `bin()` size was changed
- C. `customDimensions` casting was removed
- D. The workbook parameters are misconfigured

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** lines **82** and **199**.

```kql
| extend errorPercent = round((failedReq * 100.0) / totalReq, 2)
```

**"Flat zero, not empty" is the diagnostic detail.** An empty chart would mean no rows matched — a filter
or parameter problem. A chart plotting **zero** means rows are being returned and the computed value is
`0`.

**And a healthy service lives entirely in the range this bug erases.** At 0.4% error rate,
`failedReq * 100 / totalReq` computes as `0` in integer arithmetic. Only a rate above 1% would show
anything at all — so the chart works during a catastrophe and reads zero the rest of the time.

**`round(..., 2)` does not save it**, because the truncation happens before the rounding.

**The durable lesson:** in KQL, **make the literal a float wherever you are dividing to get a
percentage.** The failure is silent, plausible, and points the wrong way — it says everything is fine.

</details>

---

## Q48

Three months later, the SRE team answers "did the 3 PM release cause this?" in under five minutes,
consistently.

Walk through how, and explain what actually changed.

- A. The deployment annotation gives `deploymentTime`; the `fullouter` comparison shows which exception
  types moved and by how much; the `leftanti` query shows what is entirely new; the percentile chart shows
  whether latency shifted at that minute — and the alert now finds most of it before anyone asks
- B. The team browses the failures blade faster now
- C. The workbook shows a single number
- D. The alert emails a full stack trace

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-49.md`:** lines **190–192**, **107–118**, **121–130**, **137–145**, **273–283**.

**Follow the four moves, because each answers a different question.**

`customEvents` gives the **exact minute** to build windows around — without it every window is a guess.
The `fullouter` comparison answers **"is there more of what we already had?"**. The `leftanti` query
answers **"is there something we have never seen?"** — a question the percentage table structurally
cannot answer, because a new type has no "before" to compare against. And the percentile chart answers
**"did latency shift, and did it shift *there*?"**.

**What actually changed is not the data.** Every one of these facts was in Application Insights the whole
time — the exceptions, the durations, the deployment events. The SRE team at line 16 was looking at the
same telemetry.

**What changed is that the questions became *queries*.** "Did the deployment cause it?" is not a
searchable thing; **"which exception types have a post-deployment count and no pre-deployment count"
is** — and it returns the same answer every time, for every person, in seconds.

**That is the sentence to give any exam question about interrogating logs.** Browsing scales with the
observer's patience and memory; a query scales with nothing. **The value of KQL here is not that it finds
more — it is that it turns a judgement into a repeatable, alertable fact**, which is exactly why step 5
of Q33 is turning the finding into a scheduled query.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **`top N by duration` for slowest endpoints** | Q1, Q27, Q46 | Without `by name` you get individual requests |
| **Average instead of P95** | Q46 | An average hides the worst 1% |
| **No minimum request count on a percentile** | Q1, Q46 | A percentile over 3 points means nothing |
| **`join` where `toscalar()` is needed** | Q2 | A baseline is one number, not a table |
| **`inner` instead of `leftanti`** | Q3, Q20, Q29, Q38 | Inner returns what is in **both** |
| **`inner` instead of `fullouter`** | Q10, Q29, Q42 | Inner drops the new and the resolved types |
| **Dynamic compared without casting** | Q5, Q19, Q30, Q40 | `tostring()` or the indexer. Fails **silently** |
| **`100` instead of `100.0`** | Q8, Q34, Q39, Q47 | Integer division truncates every rate below 1% to zero |
| **`where success == false` before the summarize** | Q7, Q34 | Removes the denominator. Rate is always 100% |
| **Baseline including the present** | Q6, Q18, Q28, Q37 | The spike inflates its own comparison |
| **Ratio threshold with no absolute floor** | Q6, Q18, Q26, Q28 | Near-zero baseline makes every blip a 3× event |
| **No minimum traffic on an alert** | Q18, Q26, Q44 | 1 failure in 4 requests is 25% |
| **`prev()` without `serialize`** | Q11, Q21, Q30 | Row order is undefined |
| **REST API for quarter-long analytics** | Q4, Q35 | Analytics OData aggregates server-side |
| **SQL keywords — `group by`, `select`, `merge`** | Q17, Q36 | `by`, `project`, `join` |
| **`>` in an OData filter** | Q41 | `gt`, `ge`, `lt`, `le`, `eq`, `ne` |
| **Contributor granted for read access** | Q45 | Monitoring Reader |

---

# What to memorise

**In `challenge-49.md`:** lines **31–84**, **105–130**, **236–255**, **419–429**.

```kql
// THE FOUR OPERATORS THAT CARRY THIS CHALLENGE
summarize <agg> by <column>     // per-GROUP. without 'by' you get individual rows
toscalar(<subquery>)            // table -> single value, for baseline comparisons
join kind=leftanti              // left rows with NO match = "what is NEW"
countif(<condition>)            // conditional count INSIDE summarize = rate in one pass

// JOIN KINDS
inner      only rows matching on both sides
leftanti   left rows with no match          <- new exception types
fullouter  everything from both sides       <- before/after, keeps one-sided rows
           ...then coalesce(type, type1) because one side is null

// TABLES
requests      inbound calls TO your service
dependencies  outbound calls FROM it (databases, APIs) - by target, name
exceptions    type, outerMessage, innermostMessage, details[0].rawStack
customEvents  what your code tracked - including DEPLOYMENT ANNOTATIONS
traces        application log messages
```

```kql
// ERROR RATE - the shape to write from memory                (lines 78-82, 195-199)
requests
| where timestamp > ago(24h)
| summarize totalReq = count(),
            failedReq = countif(success == false)     // NOT 'where success == false' first
            by bin(timestamp, 5m)                     // bin = the time series
| extend errorPercent = round((failedReq * 100.0) / totalReq, 2)
//                                      ^^^^^ 100.0 or integer division truncates <1% to ZERO

// SLOWEST ENDPOINTS                                          (lines 148-157)
requests | where timestamp > ago(4h)
| summarize p95Duration = percentile(duration, 95), requestCount = count() by name
| where requestCount > 10                             // exclude low-traffic noise
| order by p95Duration desc | take 10

// PERCENTILES OVER TIME                                      (lines 137-145)
| summarize p50 = percentile(duration,50), p95 = percentile(duration,95),
            p99 = percentile(duration,99) by bin(timestamp, 15m)
| render timechart

// WHAT IS NEW SINCE THE DEPLOYMENT                           (lines 121-130)
exceptions | where timestamp > deploymentTime | distinct type
| join kind=leftanti (exceptions | where timestamp between (ago(7d) .. deploymentTime)
                                 | distinct type) on type

// BEFORE / AFTER                                             (lines 105-118)
let deploymentTime = datetime(2024-11-15T14:15:00Z);
let windowSize = 30m;
... | where timestamp between ((deploymentTime - windowSize) .. deploymentTime) | summarize beforeCount = count() by type;
beforeDeployment | join kind=fullouter afterDeployment on type
| extend type = coalesce(type, type1)
| extend changePercent = round(((afterCount - beforeCount) * 100.0) / max_of(beforeCount, 1), 1)
//                                                             ^^^^^^ avoids divide-by-zero for NEW types
```

```kql
// DYNAMIC FIELDS - fails SILENTLY, returns empty              (lines 389-400)
| where customDimensions.Environment == "production"              // WRONG - no error, no rows
| where tostring(customDimensions.Environment) == "production"    // right
| where customDimensions["Environment"] == "production"           // also right

// MOVING AVERAGE / ANOMALY                                    (lines 224-227)
| order by timestamp asc
| serialize                                        // REQUIRED before prev() - order is otherwise undefined
| extend movingAvg = (prev(avgDuration,1,avgDuration) + prev(avgDuration,2,avgDuration) + avgDuration)/3.0
| extend isAnomaly = avgDuration > (movingAvg * 2)
//   moving average  -> catches a SPIKE vs the last few minutes
//   7-day baseline  -> catches a REGRESSION vs normal

// DYNAMIC-THRESHOLD ALERT - all three guards                  (lines 419-429)
let baseline = requests | where timestamp between (ago(7d) .. ago(2h))   // 1. EXCLUDE the present
  | summarize baselineErrorRate = countif(success == false) * 100.0 / count();
requests | where timestamp > ago(5m)
| summarize currentErrorRate = ..., requestCount = count()
| where requestCount > 20                                  // 2. minimum TRAFFIC
| where currentErrorRate > (toscalar(baseline) * 3)
| where currentErrorRate > 1.0                             // 3. minimum ABSOLUTE rate
// remove any one of the three and it fires nightly, then gets muted
```

```text
AZURE DEVOPS ANALYTICS (OData)                                (lines 288-301)
  https://analytics.dev.azure.com/<org>/<project>/_odata/v4.0-preview/<EntitySet>
  entity sets:  PipelineRuns | WorkItems | TestRuns
  $filter  rows   (word operators: gt ge lt le eq ne - NOT > <)
  $select  columns        $orderby  sort        $top  limit
  computed fields you do not calculate: RunDuration, CycleTimeDays, LeadTimeDays
  Analytics = AGGREGATE HISTORY.  REST API = individual records.

WORKBOOKS                                                     (lines 331-357, 369-372)
  parameter type 4 = time range     -> referenced as TimeRange:start in the query
  parameter type 2 = dropdown from a query, e.g. requests | distinct cloud_RoleName
  share with:  az role assignment create --role "Monitoring Reader"   (read, cannot modify)
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 50 |
| 38–43 | Re-read the trap index and the four operators, then move on |
| 30–37 | Write the error-rate query and the alert's three guards from memory, then retake |
| Below 30 | Redo Tasks 2, 3 and 5 hands-on against real telemetry before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 49.

:::danger The four operators

**`summarize ... by`** — per group, not per row. **`toscalar()`** — a baseline is one number.
**`leftanti`** — what is new. **`countif()`** — a rate in one pass.

And two silent killers: **`customDimensions` needs `tostring()`**, and **`100` instead of `100.0` turns
every error rate below 1% into zero.**

Neither raises an error. Both tell you everything is fine.

:::
