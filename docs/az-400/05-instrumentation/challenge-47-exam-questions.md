---
sidebar_position: 2.5
toc_max_heading_level: 2
title: "Challenge 47: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 47 — AZ-400 exam questions

**48 questions** built only from what Challenge 47 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-47.md`**.

:::danger Read this before you start

Telemetry questions are decided by **what you are monitoring**, and each compute platform has exactly
one right answer.

**Application code** → **Application Insights**. Requests, dependencies, exceptions, traces.
**A virtual machine** → **VM Insights**. CPU, memory, disk — and the **Map**, which shows which
*processes* talk to what.
**A Kubernetes cluster** → **Container Insights**. Node and pod metrics, container stdout/stderr.

They stack rather than compete. An app running on a VM wants both — Application Insights for the code,
VM Insights for the machine underneath it.

And two facts carry most of the marks.

**Auto-instrumentation needs no code.** Two app settings and a restart, on App Service.

**Distributed tracing needs `traceparent` propagated.** Instrumenting every service is necessary and
**not sufficient** — if a service does not forward the header, the trace breaks there, and the exam
tests that distinction directly.

The scenario at line 16 is three teams monitoring three different ways, one of them not at all.

:::

---

# Section A — Multiple choice

---

## Q1

Contoso wants Application Insights telemetry from a .NET App Service app **without modifying code**.
What should they configure?

- A. Install the Application Insights NuGet package and add SDK initialisation
- B. Set `ApplicationInsightsAgent_EXTENSION_VERSION` and the connection string as app settings
- C. Deploy an Application Insights agent as a sidecar container
- D. Configure a data collection rule with Azure Monitor Agent

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-47.md`:** lines **58–66**.

```bash
az webapp config appsettings set \
  --settings "APPLICATIONINSIGHTS_CONNECTION_STRING=$AI_CONNECTION_STRING" \
             "ApplicationInsightsAgent_EXTENSION_VERSION=~3" \
             "XDT_MicrosoftApplicationInsights_Mode=Recommended"
```

**This is codeless attach**, and it works for .NET, Java, Node.js and Python on App Service. Three
settings, then a restart (line 66) — which is required, because the agent attaches at process start.

**Why A is the correct answer to a different question.** SDK instrumentation gives more control — custom
events, custom metrics, tuned sampling (lines 78–84) — at the cost of code changes. The requirement here
explicitly excludes it.

**Why D is the VM path** (lines 101–121). Azure Monitor Agent and data collection rules instrument
**machines**, not application code.

</details>

---

## Q2

Which solution shows which **processes** on a VM communicate with which external services?

- A. Application Insights
- B. Container Insights
- C. VM Insights, Map feature
- D. Network Watcher

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-47.md`:** lines **128–131**.

```text
- Map tab: process dependencies and network connections
```

**The Map discovers running processes and their network connections** without any code change — which
is why it is the answer for the legacy order service at line 16, where nobody can instrument the
application.

**Why A would work if the app were instrumented**, and it would show *application* dependencies rather
than *process* ones. The distinction matters for the legacy VM: nobody is adding an SDK to it.

**Why D is the near-miss.** Network Watcher analyses network **flows and connectivity** at the
infrastructure level. It does not attribute traffic to the process that generated it.

</details>

---

## Q3

Application Insights is generating 50 GB daily. Which approach reduces cost while preserving visibility
into errors?

- A. Disable Application Insights
- B. Adaptive sampling with exceptions excluded from sampling
- C. A 1 GB daily cap
- D. Switch from workspace-based to classic Application Insights

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-47.md`:** lines **292–293**.

```csharp
    builder.UseAdaptiveSampling(maxTelemetryItemsPerSecond: 5,
        excludedTypes: "Event;Exception");
```

**Sampling keeps a representative fraction and scales the counts**, so aggregate metrics stay accurate
while volume falls. Excluding `Exception` means **every** error is retained — the telemetry you actually
need when debugging.

**Why C is the trap that looks like cost control.** A daily cap is a **cliff**: once reached,
*everything* stops being ingested for the rest of the day, including the exceptions from the incident
that caused the spike. Sampling degrades gracefully; a cap fails hard.

**Why D is backwards** — workspace-based is the current model (line 48) and classic is retired.

</details>

---

## Q4

An end-to-end transaction shows only the initial request, not the three downstream calls. Why?

- A. The backend services are not instrumented
- B. The services use different Application Insights resources
- C. Trace context headers are not propagated between services
- D. Sampling is filtering out dependency calls

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-47.md`:** lines **396** and **402**.

```text
**Cause:** The payment service uses a custom HTTP client that does not propagate W3C trace context headers.
```

**A is the plausible answer, and the exam's explanation is explicit that both are needed:** all services
should be instrumented **and** trace context must flow through HTTP headers. If the services were not
instrumented at all, you would see no telemetry from them anywhere — here they appear on the map (line
394) but their spans do not join the parent.

**Why B does not break correlation.** Different Application Insights resources in the same Log Analytics
workspace still correlate on `operation_Id` — the end-to-end view spans resources.

**Why D would produce gaps, not a consistent absence.** Sampling drops a fraction; this drops the same
boundary every time.

</details>

---

## Q5

Container Insights shows no data for a newly created namespace. What is the likely cause?

- A. The AKS monitoring add-on is disabled
- B. The namespace is in the ConfigMap's `exclude_namespaces` list
- C. The workspace is full
- D. The pods have no resource limits

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-47.md`:** lines **369** and **166–173**.

```toml
      [log_collection_settings.stdout]
        enabled = true
        exclude_namespaces = ["kube-system","gatekeeper-system"]
```

**Diagnose by reading the ConfigMap** (line 374), then edit it and **restart the DaemonSet** (line 387):

```bash
kubectl rollout restart daemonset omsagent -n kube-system
```

**The restart is the step people forget.** The agent reads its configuration at startup, so editing the
ConfigMap alone changes nothing until the pods recycle.

**Why A would produce no data for *any* namespace**, which is the diagnostic that separates the two.

</details>

---

## Q6

What does `--workspace $LAW_ID` do when creating Application Insights?

- A. Creates a workspace-based Application Insights resource storing data in Log Analytics
- B. Copies existing telemetry into the workspace
- C. Enables sampling
- D. Links the workspace for billing only

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** lines **44–49**.

```bash
az monitor app-insights component create \
  --app ai-contoso-webapp \
  --workspace $LAW_ID \
  --application-type web
```

**Workspace-based is the current model, and the benefit is that everything lands in one place.**
Application Insights telemetry, VM Insights data and Container Insights data all sit in the same
workspace — so a single KQL query can join across them.

**That is exactly what the CTO asked for at line 16:** standardised observability across three compute
platforms. One workspace is what makes "standardised" mean something operationally.

</details>

---

## Q7

What does the Azure Monitor Agent use to determine what to collect and where to send it?

- A. A data collection rule associated with the VM
- B. Workspace keys configured on the agent
- C. Tags on the VM
- D. The agent's local configuration file

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** lines **98** and **101–121**.

```text
# AMA uses Data Collection Rules for workspace targeting (configured below)
```

**This is the architectural change from the legacy Log Analytics agent**, and the exam tests it. The old
agent was configured **per machine** with a workspace ID and key. AMA is configured **centrally** by a
DCR, which is then **associated** with one or many machines (lines 118–121).

**The practical consequence: one rule, many VMs.** Change the counters collected in the DCR and every
associated machine follows, with no agent reconfiguration.

</details>

---

## Q8

Which streams does the VM Insights data collection rule specify?

- A. `Microsoft-InsightsMetrics` and `Microsoft-ServiceMap`
- B. `Microsoft-Perf` and `Microsoft-Syslog`
- C. `Microsoft-ContainerLog`
- D. `Microsoft-Event`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** lines **105–108**.

```json
    "streams": ["Microsoft-InsightsMetrics", "Microsoft-ServiceMap"],
```

**Two streams for the two tabs.** `InsightsMetrics` feeds the **Performance** tab; `ServiceMap` feeds
the **Map** tab (lines 128–131).

**Which means leaving out `Microsoft-ServiceMap` gives you charts and no dependency map** — and the map
is the one capability that made VM Insights the answer in Q2.

</details>

---

## Q9

Which command enables Container Insights on an existing AKS cluster?

- A. `az aks enable-addons --addons monitoring --workspace-resource-id $LAW_ID`
- B. `az aks update --enable-azure-monitor-metrics`
- C. `az vm extension set --name AzureMonitorLinuxAgent`
- D. `kubectl apply -f container-insights.yaml`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** lines **137–141**.

**Why B is a real command that does something different**, and the exam pairs them deliberately.
`--enable-azure-monitor-metrics` (line 150) enables **managed Prometheus** — Prometheus-format metrics
scraped into Azure Monitor. Container Insights is the logs-and-metrics add-on. Many clusters run both.

**Why C is the VM path** and **why D is not how an add-on is enabled** — the add-on deploys the agent
for you.

</details>

---

## Q10

What is `client.trackEvent` used for?

- A. Recording a business event with properties and measurements
- B. Recording an exception
- C. Recording a page view
- D. Recording an outbound HTTP call

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** lines **202–213**.

```javascript
client.trackEvent({
  name: "OrderPlaced",
  properties: { customerId: ..., region: ..., paymentMethod: ... },
  measurements: { orderValue: order.total, itemCount: order.items.length }
});
```

**Note the split, because it decides how you can query it.** `properties` are **strings** — you filter
and group by them. `measurements` are **numbers** — you aggregate them.

**Putting `orderValue` in properties would make it a string**, and `summarize sum(...)` over it would
fail. That is the detail the exam tests.

**Why D is `trackDependency`** (line 225), which is the next block down.

</details>

---

## Q11

What does `client.trackDependency` record?

- A. A call to an external service, with target, duration, result code and success
- B. A custom business metric
- C. A trace message
- D. An availability test result

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** lines **225–233**.

```javascript
client.trackDependency({
  target: "payment-gateway",
  name: "ChargeCard",
  data: "POST /api/charge",
  duration: callDurationMs,
  resultCode: response.status,
  success: response.status === 200,
  dependencyTypeName: "HTTP"
});
```

**Dependencies are what build the Application Map.** Each recorded call becomes an edge between two
nodes, which is how a distributed system's topology appears without anyone drawing it.

**And most dependencies are captured automatically** by `setAutoCollectDependencies(true)` (line 195).
You call `trackDependency` manually only for a client the SDK does not patch — the same situation that
breaks trace propagation in Break scenario 2.

</details>

---

## Q12

What does an availability test with three locations verify?

- A. That the endpoint responds correctly from multiple geographic regions
- B. That the app is deployed to three regions
- C. That failover works
- D. That latency is under 120 ms

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** lines **245–249**.

```bash
  --locations Id="us-fl-mia-edge" \
  --locations Id="emea-nl-ams-azr" \
  --locations Id="apac-sg-sin-azr" \
  --kind "ping" \
  --frequency 300 \
```

**Multiple locations distinguish "the app is down" from "one region cannot reach it".** A single test
location cannot tell you which, and a network problem between one edge and your app would page you for
an application that is perfectly healthy.

**`--frequency 300` is every five minutes** per location, so three locations produce a test roughly
every 100 seconds on average.

</details>

---

## Q13

Which condition alerts on availability dropping below 99%?

- A. `avg availabilityResults/availabilityPercentage < 99`
- B. `total Http5xx > 50`
- C. `count exceptions > 100`
- D. `avg requests/duration > 1000`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** line **258**.

```bash
  --condition "avg availabilityResults/availabilityPercentage < 99" \
  --window-size 5m \
  --evaluation-frequency 1m
```

**`availabilityResults` is the table the web tests write to**, and `availabilityPercentage` is the
aggregated metric derived from it.

**Note the direction of the comparison.** Availability alerts are `<` — you alert when a number falls.
Error alerts are `>`. Getting this backwards produces an alert that fires constantly during normal
operation, which is how alerting gets muted.

</details>

---

## Q14

What is the difference between adaptive and fixed-rate sampling?

- A. Adaptive adjusts the rate dynamically to hit a target volume; fixed-rate keeps a constant
  percentage
- B. Adaptive is server-side; fixed-rate is client-side
- C. Adaptive preserves exceptions automatically
- D. Fixed-rate is only available on App Service

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** lines **286–289**.

```csharp
    builder.UseAdaptiveSampling(maxTelemetryItemsPerSecond: 5);
    // builder.UseSampling(25.0);   // fixed-rate: keep 25%
```

**Adaptive targets a rate — 5 items per second — and varies the percentage** to hit it, so a traffic
spike does not become a cost spike. **Fixed-rate keeps a constant fraction**, so volume scales with
traffic and is predictable in a different way.

**Why C is the specific misconception worth killing.** Adaptive sampling does **not** preserve
exceptions automatically. You must say so explicitly with `excludedTypes: "Event;Exception"` (line
293) — which is exactly what Q3 turns on.

</details>

---

## Q15

What does ingestion sampling do that SDK sampling does not?

- A. It is applied server-side to all data regardless of SDK configuration
- B. It is cheaper
- C. It preserves all exceptions
- D. It works only for .NET

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** line **299**.

```text
Ingestion sampling (server-side, applied to all data regardless of SDK settings)
```

**It is the backstop for telemetry you do not control** — a service someone else instrumented, an SDK
version you cannot change, a language whose sampling options differ.

**And it is strictly worse than SDK sampling when you have the choice**, because the data has already
crossed the network before being discarded. SDK sampling avoids sending it at all.

</details>

---

## Q16

What is the risk of setting a daily cap?

- A. All ingestion stops once the cap is reached, including exceptions during an incident
- B. Telemetry is delayed until the next day
- C. Sampling is disabled
- D. The workspace is deleted

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** lines **303–306**.

```bash
az monitor app-insights component billing update \
  --app ai-contoso-webapp \
  --cap 5  # 5 GB daily cap
```

**A cap is a hard stop, and the failure mode is precisely inverted from what you want.** An incident
generates more telemetry, which reaches the cap sooner, which cuts off ingestion **during the incident
you need to diagnose**.

**Caps are a cost *guarantee*, not a cost *strategy*.** Use sampling to control volume, and a cap only
as a backstop against a runaway bill — set high enough that normal operation plus a bad day stays under
it.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** app settings enable App Service auto-instrumentation? (Choose three.)

- A. `APPLICATIONINSIGHTS_CONNECTION_STRING`
- B. `ApplicationInsightsAgent_EXTENSION_VERSION=~3`
- C. `XDT_MicrosoftApplicationInsights_Mode=Recommended`
- D. `WEBSITE_RUN_FROM_PACKAGE`
- E. `APPINSIGHTS_PROFILERFEATURE_VERSION`
- F. `DOCKER_REGISTRY_SERVER_URL`

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-47.md`:** lines **61–63**.

**Where to send it, which agent version, how much to collect** — three settings answering three
questions.

**`Recommended` mode collects more than `Default`**, including dependency tracking, which is what makes
the Application Map work for this app.

**And the restart at line 66 is part of the procedure**, not an afterthought: the agent attaches when
the process starts.

</details>

---

## Q18

Which **three** does VM Insights provide? (Choose three.)

- A. CPU, memory, disk and network performance
- B. A process dependency and network connection map
- C. Connection monitoring to external services
- D. Application request and dependency telemetry
- E. Container log collection
- F. Distributed tracing

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-47.md`:** lines **128–131**.

```text
- Performance tab: CPU, memory, disk IOPS, network
- Map tab: process dependencies and network connections
- Connection monitoring between VMs and external services
```

**Why D and F are the boundary the exam tests.** Requests, dependencies and traces come from
**Application Insights**, which requires the application to be instrumented. VM Insights sees the
machine and the processes on it — it has no idea what an HTTP route is.

</details>

---

## Q19

Which **two** are configured in the Container Insights ConfigMap? (Choose two.)

- A. Which namespaces are excluded from stdout and stderr collection
- B. Whether environment variables are collected
- C. The Log Analytics workspace ID
- D. The AKS node pool size
- E. Application Insights sampling

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-47.md`:** lines **166–179**.

```toml
      [log_collection_settings.stderr]
        enabled = true
        exclude_namespaces = ["kube-system"]
      [log_collection_settings.env_var]
        enabled = false
```

**`env_var` is disabled deliberately, and it is worth understanding why.** Container environment
variables routinely hold connection strings and keys — collecting them would write secrets into the
workspace, where they are searchable by anyone with read access. This is Challenge 43's lesson in a
different place.

**Why C is set at add-on enablement** (line 141), not in the ConfigMap.

</details>

---

## Q20

Which **two** are required for end-to-end distributed tracing? (Choose two.)

- A. Every service in the path is instrumented
- B. `traceparent` is propagated on outbound calls
- C. All services share one Application Insights resource
- D. Sampling is disabled
- E. All services run on the same platform

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-47.md`:** lines **314–319** and **396**.

```text
# traceparent: 00-<trace-id>-<span-id>-<trace-flags>
```

**Both, and neither alone.** Instrumentation without propagation gives you disconnected islands of
telemetry; propagation without instrumentation gives you a header nobody records.

**Why C and E are the answers that feel like they should be true.** Correlation is on `operation_Id`,
carried in the header — so a Node.js service on AKS and a .NET service on App Service, reporting to two
different Application Insights resources, still join into one trace.

</details>

---

## Q21

Which **two** reduce Application Insights cost while keeping error visibility? (Choose two.)

- A. Adaptive sampling targeting a fixed items-per-second rate
- B. `excludedTypes: "Event;Exception"` on the sampling processor
- C. A 1 GB daily cap
- D. Disabling dependency tracking
- E. Reducing the retention period to one day

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-47.md`:** lines **286** and **292–293**.

**A controls volume; B protects what matters.** Sampling alone would discard exceptions along with
everything else, which is precisely the telemetry you keep it for.

**Why D would cut volume and break the Application Map**, losing the distributed-tracing capability the
CTO asked for.

**Why E is a different lever with a different cost.** Retention affects how far back you can query, not
how much you ingest — and one day means an incident on Friday cannot be investigated on Monday.

</details>

---

## Q22

Which **two** describe the difference between Container Insights and managed Prometheus on AKS?
(Choose two.)

- A. Container Insights collects container logs and platform metrics
- B. Managed Prometheus scrapes Prometheus-format metrics from pods
- C. They are alternatives — enabling both is unsupported
- D. Managed Prometheus collects container stdout
- E. Container Insights requires a service mesh

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-47.md`:** lines **137–141**, **150–153**, **176–179**.

```bash
az aks enable-addons --addons monitoring --workspace-resource-id $LAW_ID
az aks update --enable-azure-monitor-metrics
```

**Logs and platform metrics on one side; application-exposed Prometheus metrics on the other.**

**Why C is the trap.** They are complementary, and this challenge enables **both** — plus the ConfigMap
sets `monitor_kubernetes_pods = true` (line 179) so annotated pods are scraped. A cluster typically
wants container logs *and* the custom metrics its applications expose.

</details>

---

## Q23

Which **two** appear in the KQL that traces a request across services? (Choose two.)

- A. A join between `requests` and `dependencies` on `operation_Id`
- B. `project-away` to drop the duplicated join column
- C. `summarize percentile(duration, 95)`
- D. `render timechart`
- E. `where success == false`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-47.md`:** lines **351–360**.

```kusto
| join kind=inner (
    dependencies
    | project operation_Id, target, name, duration, success
) on operation_Id
| project-away operation_Id1
```

**`operation_Id` is the correlation key** — the same value on the incoming request and every dependency
call made while handling it, which is exactly what `traceparent` carries between services.

**`project-away operation_Id1` removes the duplicate** the join creates. Cosmetic, and it is in the
challenge because a join always produces it.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must standardise observability across a legacy order service on VMs, payment and
inventory services on AKS, and a web frontend on App Service — with distributed tracing end to end and
costs under control.

---

## Q24

**Proposed solution:** Create one workspace-based Application Insights resource in a shared Log Analytics
workspace. Enable App Service auto-instrumentation with the three app settings and restart. Install
Azure Monitor Agent on the VM with a data collection rule carrying `Microsoft-InsightsMetrics` and
`Microsoft-ServiceMap`. Enable the AKS monitoring add-on and managed Prometheus, with a ConfigMap that
collects stdout and stderr and leaves `env_var` disabled. Set the connection string and
`OTEL_SERVICE_NAME` on each AKS deployment. Configure adaptive sampling at 5 items per second excluding
exceptions, and add multi-region availability tests with an alert below 99%.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-47.md`:** lines **33–49**, **58–66**, **93–121**, **137–153**, **166–179**, **337–344**,
**292–293**, **240–262**.

| Platform or need | Mechanism |
|---|---|
| Web frontend | Auto-instrumentation, no code change |
| Legacy VM | AMA + DCR, Performance and Map |
| AKS services | Container Insights + managed Prometheus |
| Distributed tracing | Connection string per service, `traceparent` propagated |
| One place to query | One workspace |
| Cost | Adaptive sampling, exceptions preserved |
| External reachability | Multi-region availability tests |

**The single workspace is what makes it "standardised".** Three platforms reporting into three separate
stores would still be three teams looking at three dashboards — the state at line 16.

</details>

---

## Q25

**Proposed solution:** Have each team keep its current tooling but add a monthly report. Enable
Application Insights on the App Service only. Set a 1 GB daily cap to control costs. Investigate
distributed tracing later.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures, and the first is the scenario restated as a solution.**

**"Each team keeps its current tooling" is the problem.** Line 16 describes the VM team checking RDP
sessions and the AKS team reading `kubectl logs` — that is not monitoring, it is looking.

**App Service only leaves two of three platforms dark**, including the payment service, which is where
a failure costs the most.

**A 1 GB cap is the wrong instrument** (Q16): ingestion stops entirely once reached, so a busy incident
day silently loses its own evidence.

**And distributed tracing is not an add-on you retrofit cheaply.** It requires every service
instrumented and every hop propagating `traceparent` — which is a design decision, not a setting to
enable later.

</details>

---

## Q26

**Proposed solution:** Create a workspace-based Application Insights resource. Enable auto-instrumentation
on App Service. Install AMA on the VM with a data collection rule. Enable Container Insights on AKS with
stdout and stderr collection. Set the connection string on each AKS deployment. Configure adaptive
sampling at 5 items per second. Enable `env_var` collection in the ConfigMap so the team can see each
container's configuration alongside its logs.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical
apart from sampling exclusions.

**Enabling `env_var` collection writes container environment variables into Log Analytics.** In a
microservices deployment those variables hold the Application Insights connection string, database
connection strings, and whatever else the deployment injects — including, at line 338–342, a value
sourced from a Kubernetes **secret**.

**The result is a secret carefully stored in a secret, then copied into a queryable log store** where
anyone with workspace read access can find it, and where it is retained for the workspace's retention
period. That is Challenge 42's ladder collapsing back to rung 0 through a monitoring setting.

**Line 174 disables it deliberately**, and that default is the answer:

```toml
      [log_collection_settings.env_var]
        enabled = false
```

**The design has also lost `excludedTypes`**, so adaptive sampling now discards exceptions along with
everything else — the exact telemetry the sampling was configured to protect.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — Application Insights

| # | Statement | Answer |
|---|---|---|
| 1 | Auto-instrumentation on App Service requires no code changes |  |
| 2 | The app must be restarted for auto-instrumentation to attach |  |
| 3 | Workspace-based Application Insights stores data in Log Analytics |  |
| 4 | The SDK and auto-instrumentation offer identical control |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Auto-instrumentation on App Service requires no code changes | **Yes** |
| 2 | The app must be restarted for auto-instrumentation to attach | **Yes** |
| 3 | Workspace-based Application Insights stores data in Log Analytics | **Yes** |
| 4 | The SDK and auto-instrumentation offer identical control | **No** |

**In `challenge-47.md`:** lines **57–66**, **44–49**, **78–84**.

Row 4 is the trade: auto-instrumentation is zero effort and fixed behaviour; the SDK adds custom events,
custom metrics and fine-grained sampling.

</details>

---

## Q28 — platform coverage

| # | Statement | Answer |
|---|---|---|
| 1 | VM Insights maps process-level dependencies |  |
| 2 | Container Insights collects container stdout and stderr |  |
| 3 | Application Insights monitors VM CPU and memory |  |
| 4 | An app on a VM can use both Application Insights and VM Insights |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | VM Insights maps process-level dependencies | **Yes** |
| 2 | Container Insights collects container stdout and stderr | **Yes** |
| 3 | Application Insights monitors VM CPU and memory | **No** |
| 4 | An app on a VM can use both Application Insights and VM Insights | **Yes** |

**In `challenge-47.md`:** lines **130**, **166–173**, **128**.

Rows 3 and 4 together are the boundary and the resolution. **They stack.** Application Insights sees the
code; VM Insights sees the machine.

</details>

---

## Q29 — distributed tracing

| # | Statement | Answer |
|---|---|---|
| 1 | Correlation uses W3C trace context |  |
| 2 | Instrumenting every service is sufficient on its own |  |
| 3 | `operation_Id` links a request to its dependencies |  |
| 4 | Services must share one Application Insights resource |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Correlation uses W3C trace context | **Yes** |
| 2 | Instrumenting every service is sufficient on its own | **No** |
| 3 | `operation_Id` links a request to its dependencies | **Yes** |
| 4 | Services must share one Application Insights resource | **No** |

**In `challenge-47.md`:** lines **314–319**, **396**, **351–359**.

Row 2 is the single most-tested fact in this challenge. **A service that does not forward `traceparent`
breaks the trace at that hop**, even though it is fully instrumented and appears on the map.

</details>

---

## Q30 — cost control

| # | Statement | Answer |
|---|---|---|
| 1 | Adaptive sampling targets a telemetry rate |  |
| 2 | Adaptive sampling preserves exceptions by default |  |
| 3 | Ingestion sampling applies regardless of SDK settings |  |
| 4 | A daily cap stops all ingestion once reached |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Adaptive sampling targets a telemetry rate | **Yes** |
| 2 | Adaptive sampling preserves exceptions by default | **No** |
| 3 | Ingestion sampling applies regardless of SDK settings | **Yes** |
| 4 | A daily cap stops all ingestion once reached | **Yes** |

**In `challenge-47.md`:** lines **286**, **292–293**, **299**, **303–306**.

Row 2 is the correction most people need: exceptions are preserved **only** when you name them in
`excludedTypes`.

Row 4 is why a cap is a backstop rather than a strategy — the cliff arrives soonest on the worst day.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each monitoring need to its solution.

| Need | Solution |
|---|---|
| Request and dependency telemetry from a web app |  |
| CPU, memory and disk on a legacy VM |  |
| Which processes on a VM talk to which services |  |
| Container stdout and pod metrics on AKS |  |
| Prometheus metrics exposed by a pod |  |
| Reachability from three continents |  |

**Options:** Application Insights · Availability test · Container Insights · Managed Prometheus · VM Insights, Map · VM Insights, Performance

<details>
<summary>Show answer</summary>

| Need | Solution |
|---|---|
| Request and dependency telemetry from a web app | **Application Insights** |
| CPU, memory and disk on a legacy VM | **VM Insights, Performance** |
| Which processes on a VM talk to which services | **VM Insights, Map** |
| Container stdout and pod metrics on AKS | **Container Insights** |
| Prometheus metrics exposed by a pod | **Managed Prometheus** |
| Reachability from three continents | **Availability test** |

**In `challenge-47.md`:** lines **44–49**, **129**, **130**, **137–141**, **150–153**, **240–251**.

**Choose by *what* you are watching, not *where* it runs.** Code → Application Insights. Machine → VM
Insights. Cluster → Container Insights. They stack on the same host.

</details>

---

## Q32

Match each SDK call to what it records.

| Call | Records |
|---|---|
| `trackEvent` |  |
| `trackMetric` |  |
| `trackDependency` |  |
| `setAutoCollectRequests(true)` |  |
| `setAutoCollectExceptions(true)` |  |

**Options:** A business event with string properties and numeric measurements · A single named numeric value · An outbound call — target, duration, result code, success · Incoming requests, automatically · Unhandled exceptions, automatically

<details>
<summary>Show answer</summary>

| Call | Records |
|---|---|
| `trackEvent` | **A business event with string properties and numeric measurements** |
| `trackMetric` | **A single named numeric value** |
| `trackDependency` | **An outbound call — target, duration, result code, success** |
| `setAutoCollectRequests(true)` | **Incoming requests, automatically** |
| `setAutoCollectExceptions(true)` | **Unhandled exceptions, automatically** |

**In `challenge-47.md`:** lines **202–233** and **194–196**.

**Properties are strings you filter by; measurements are numbers you aggregate.** Put a number in
properties and `summarize sum(...)` cannot use it.

</details>

---

## Q33

Arrange the steps to instrument the VM.

**Items:** Associate the rule with the VM · Install the Azure Monitor Agent extension · Create the Log
Analytics workspace · Create the data collection rule with the required streams

<details>
<summary>Show answer</summary>

### Answer

1. Create the Log Analytics workspace — lines **33–36**
2. Install the Azure Monitor Agent extension — lines **93–97**
3. Create the data collection rule with the required streams — lines **101–110**
4. Associate the rule with the VM — lines **118–121**

**Step 4 is the one people omit**, and the symptom is characteristic: the agent is installed and
healthy, the rule exists and looks correct, and **no data arrives**. The agent has nothing to do until
an association tells it which rule applies.

**And the DCR must be created before the association** because the association references its ID (line
113).

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| No telemetry after setting the connection string on App Service |  |
| No data from the VM despite a healthy agent |  |
| VM Performance works, Map is empty |  |
| Container Insights blank for one namespace |  |
| Trace stops at one service |  |
| Exceptions missing from a busy day |  |

**Options:** App not restarted · `Microsoft-ServiceMap` stream missing · Namespace in `exclude_namespaces` · No DCR association · Sampling without `excludedTypes`, or a daily cap hit · `traceparent` not propagated

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| No telemetry after setting the connection string on App Service | **App not restarted** |
| No data from the VM despite a healthy agent | **No DCR association** |
| VM Performance works, Map is empty | **`Microsoft-ServiceMap` stream missing** |
| Container Insights blank for one namespace | **Namespace in `exclude_namespaces`** |
| Trace stops at one service | **`traceparent` not propagated** |
| Exceptions missing from a busy day | **Sampling without `excludedTypes`, or a daily cap hit** |

**In `challenge-47.md`:** lines **66**, **118–121**, **106**, **170**, **396**, **292–293** with
**303–306**.

**Four of these six leave every component reporting healthy.** That is the theme: telemetry failures are
usually silent, and the only signal is data that is not there.

</details>

---

## Q35

Match each cost lever to its behaviour.

| Lever | Behaviour |
|---|---|
| Adaptive sampling |  |
| Fixed-rate sampling |  |
| `excludedTypes: "Event;Exception"` |  |
| Ingestion sampling |  |
| Daily cap |  |

**Options:** Hard stop — all ingestion ceases · Keeps a constant percentage · Never samples away errors · Server-side, regardless of SDK settings · Varies the rate to hit a target items-per-second

<details>
<summary>Show answer</summary>

| Lever | Behaviour |
|---|---|
| Adaptive sampling | **Varies the rate to hit a target items-per-second** |
| Fixed-rate sampling | **Keeps a constant percentage** |
| `excludedTypes: "Event;Exception"` | **Never samples away errors** |
| Ingestion sampling | **Server-side, regardless of SDK settings** |
| Daily cap | **Hard stop — all ingestion ceases** |

**In `challenge-47.md`:** lines **286**, **289**, **293**, **299**, **303–306**.

**Ranked by how gracefully they fail:** exclusions, then adaptive, then fixed-rate, then ingestion
sampling, then the cap — which does not degrade at all, it stops.

</details>

---

# Section F — Hot area

---

## Q36

```bash
az webapp config appsettings set \
  --name app-contoso-web \
  --settings "[BLANK 1]=$AI_CONNECTION_STRING" \
             "[BLANK 2]=~3" \
             "XDT_MicrosoftApplicationInsights_Mode=[BLANK 3]"
```

- **BLANK 1:** `APPLICATIONINSIGHTS_CONNECTION_STRING` / `APPINSIGHTS_INSTRUMENTATIONKEY` /
  `AI_CONNECTION` / `WEBSITE_APPINSIGHTS`
- **BLANK 2:** `ApplicationInsightsAgent_EXTENSION_VERSION` / `AI_AGENT_VERSION` /
  `APPINSIGHTS_EXTENSION` / `WEBSITE_EXTENSION_VERSION`
- **BLANK 3:** `Recommended` / `Default` / `Full` / `Minimal`

<details>
<summary>Show answer</summary>

### Answer: `APPLICATIONINSIGHTS_CONNECTION_STRING`,
`ApplicationInsightsAgent_EXTENSION_VERSION`, `Recommended`

**In `challenge-47.md`:** lines **61–63**.

**`APPINSIGHTS_INSTRUMENTATIONKEY` is the deprecated form**, and it is the distractor that catches
anyone working from older material. Connection strings replaced instrumentation keys because they carry
the regional ingestion endpoint as well as the identifier.

**And the restart at line 66 completes the procedure** — the agent attaches at process start.

</details>

---

## Q37

```bash
az monitor app-insights component create \
  --app ai-contoso-webapp \
  --[BLANK 1] $LAW_ID \
  --application-type [BLANK 2]
```

- **BLANK 1:** `workspace` / `log-analytics` / `destination` / `sink`
- **BLANK 2:** `web` / `other` / `java` / `general`

<details>
<summary>Show answer</summary>

### Answer: `workspace`, `web`

**In `challenge-47.md`:** lines **44–49**.

**`--workspace` is what makes it workspace-based** rather than classic — the current model, and the
reason a single KQL query can join application telemetry against VM and container data.

</details>

---

## Q38

```json
{
  "streams": ["Microsoft-InsightsMetrics", "[BLANK 1]"],
  "destinations": ["law-contoso-observability"]
}
```

Requirement: VM Insights must show both the Performance tab and the Map tab.

- **BLANK 1:** `Microsoft-ServiceMap` / `Microsoft-Perf` / `Microsoft-Syslog` /
  `Microsoft-ContainerLog`

<details>
<summary>Show answer</summary>

### Answer: `Microsoft-ServiceMap`

**In `challenge-47.md`:** line **106**.

**Two streams, two tabs.** Omit `Microsoft-ServiceMap` and you get charts with an empty Map — which is
the one capability that made VM Insights the right answer for the legacy service (Q2).

</details>

---

## Q39

```toml
  [log_collection_settings.stdout]
    enabled = [BLANK 1]
    [BLANK 2] = ["kube-system","gatekeeper-system"]
  [log_collection_settings.env_var]
    enabled = [BLANK 3]
```

Requirement: collect application logs, skip platform namespaces, and do not write secrets to the
workspace.

- **BLANK 1:** `true` / `false`
- **BLANK 2:** `exclude_namespaces` / `include_namespaces` / `namespaces` / `filter`
- **BLANK 3:** `false` / `true`

<details>
<summary>Show answer</summary>

### Answer: `true`, `exclude_namespaces`, `false`

**In `challenge-47.md`:** lines **167–174**.

**`env_var = false` is the security-relevant one.** Container environment variables carry connection
strings and keys — including, in this challenge, a value sourced from a Kubernetes secret (lines
338–342). Collecting them copies secrets into a queryable log store.

**And `exclude_namespaces` is a denylist**, which is why a *new* namespace is collected by default and
Break scenario 1 happens only when someone has added it to the list.

</details>

---

## Q40

```csharp
    builder.UseAdaptiveSampling(
        maxTelemetryItemsPerSecond: [BLANK 1],
        [BLANK 2]: "Event;Exception");
```

Requirement: cut volume while keeping every error.

- **BLANK 1:** `5` / `500` / `0` / `100`
- **BLANK 2:** `excludedTypes` / `includedTypes` / `preserveTypes` / `filterTypes`

<details>
<summary>Show answer</summary>

### Answer: `5`, `excludedTypes`

**In `challenge-47.md`:** lines **292–293**.

**"Excluded" means excluded *from sampling*, therefore kept in full.** The word reads like the opposite
of what it does, which is exactly why the exam uses it.

**`includedTypes` would be the reverse** — sample only these — and would discard the exceptions this
configuration exists to protect.

</details>

---

## Q41

```yaml
          env:
            - name: [BLANK 1]
              valueFrom:
                secretKeyRef:
                  name: app-insights-secret
                  key: connection-string
            - name: [BLANK 2]
              value: "payment-service"
```

- **BLANK 1:** `APPLICATIONINSIGHTS_CONNECTION_STRING` / `AI_KEY` / `OTEL_EXPORTER_ENDPOINT` /
  `APPINSIGHTS_INSTRUMENTATIONKEY`
- **BLANK 2:** `OTEL_SERVICE_NAME` / `SERVICE_NAME` / `APP_NAME` / `OTEL_RESOURCE`

<details>
<summary>Show answer</summary>

### Answer: `APPLICATIONINSIGHTS_CONNECTION_STRING`, `OTEL_SERVICE_NAME`

**In `challenge-47.md`:** lines **338–344**.

**`OTEL_SERVICE_NAME` is what labels this service's node on the Application Map.** Without it, services
appear under a generic or duplicated name and the map becomes unreadable in exactly the way a
microservices architecture cannot afford.

**And note the connection string comes from `secretKeyRef`** rather than a literal — which is the
correct handling, and the reason enabling `env_var` collection in Q26 was so damaging.

</details>

---

# Section G — Case study

## Case study: Contoso standardised observability

### Background

Contoso Ltd runs a microservices architecture across three compute platforms: a **legacy order service
on Azure VMs**, **payment and inventory services on AKS**, and a **web frontend on App Service**. The VM
team checks RDP sessions, the AKS team reads `kubectl logs`, and **the App Service team has no
monitoring at all**. The CTO wants standardised observability with **distributed tracing end to end**.

### Requirements

**Coverage**

- Every platform must report telemetry into a single queryable store
- The web frontend must be instrumented **without code changes**
- The legacy VM must expose process-level dependencies, since its application cannot be modified
- AKS must provide container logs and the Prometheus metrics its services expose

**Tracing**

- A request entering the frontend must be traceable through payment and inventory
- Each service must be identifiable on the Application Map

**Operations**

- Telemetry cost must be controlled without losing errors
- External reachability must be verified from multiple regions
- No secrets may be written into the telemetry store

---

## Q42

How should the web frontend be instrumented?

- A. Auto-instrumentation via `ApplicationInsightsAgent_EXTENSION_VERSION` and the connection string,
  followed by a restart
- B. Add the Application Insights SDK and initialisation code
- C. Deploy the Azure Monitor Agent to the App Service
- D. Enable Container Insights

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** lines **57–66**.

**"Without code changes" is the requirement**, and codeless attach is the only option that satisfies it.

**Why B is the better answer to a different requirement.** The SDK gives custom events, custom metrics
and tuned sampling — everything at lines 78–84 and 202–233. If the requirement had said "track
`OrderPlaced` with the customer's region", B would be correct.

**Why C is the VM mechanism** and **why D is the AKS mechanism** — both real, both on the wrong
platform.

</details>

---

## Q43

How should the legacy VM be monitored?

- A. VM Insights via Azure Monitor Agent and a DCR carrying `Microsoft-InsightsMetrics` and
  `Microsoft-ServiceMap`, associated with the VM
- B. Application Insights SDK added to the order service
- C. Network Watcher connection monitor
- D. `kubectl logs`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** lines **93–121** and **128–131**.

**"Its application cannot be modified" eliminates B**, which is the requirement doing its job — the
legacy service is legacy precisely because nobody is changing it.

**`Microsoft-ServiceMap` is the stream that satisfies "process-level dependencies"** (Q38). Drop it and
the requirement is unmet while everything still looks configured.

**And the association at lines 118–121 is what activates it** — the step whose absence produces a
healthy agent and no data (Q33).

</details>

---

## Q44

How should the AKS services be covered?

- A. Container Insights add-on with a workspace, plus managed Prometheus, with a ConfigMap collecting
  stdout and stderr and `env_var` disabled
- B. Container Insights only
- C. Managed Prometheus only
- D. Application Insights only

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** lines **137–153** and **166–179**.

**The requirement names two things — container logs and Prometheus metrics — so it needs both add-ons.**
They are complementary, not alternatives (Q22).

**And `env_var: false` satisfies the "no secrets in telemetry" requirement**, which is a separate
requirement that this one setting happens to answer (Q26).

**Why D is worth thinking through rather than dismissing.** The AKS *services* should also report to
Application Insights — that is how their spans join the trace (Q45). D fails because it leaves the
**cluster** unmonitored: no node metrics, no pod restarts, no container logs.

</details>

---

## Q45

Which **two** make a request traceable from the frontend through payment and inventory? (Choose two.)

- A. Each service configured with the Application Insights connection string
- B. `traceparent` propagated on every outbound call
- C. All three services in one resource group
- D. Sampling disabled
- E. All three services on the same platform

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-47.md`:** lines **337–344** and **314–319**.

**Instrumentation plus propagation.** Q20's pair, applied to the case study.

**The failure mode is specific and worth recognising:** if the payment service uses a custom HTTP client
that drops the header (line 396), the map still shows all three services — and the end-to-end view stops
at the frontend. **Everything looks instrumented, and the trace is broken.**

</details>

---

## Q46

How should cost be controlled without losing errors?

- A. Adaptive sampling at a target rate with `excludedTypes: "Event;Exception"`
- B. A 1 GB daily cap
- C. Fixed-rate sampling at 25%
- D. Disabling dependency tracking

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** lines **292–293**.

**The exclusion is the half that meets the requirement.** Sampling alone reduces cost and discards
exceptions along with everything else.

**Why B fails in the worst possible way** (Q16): the cap is reached soonest on a high-traffic incident
day, so ingestion stops during the incident you need to diagnose.

**Why C would work partially** — 25% of exceptions is still a loss, and the requirement says errors must
be preserved. **Why D breaks the Application Map** and with it the tracing requirement.

</details>

---

## Q47

Four months in, the platform team notices the Application Map no longer shows the inventory service as a
distinct node — its calls appear attributed to the payment service. Both services still report telemetry
and both appear healthy. The team recently standardised their Kubernetes deployment manifests onto a
shared Helm chart.

What happened, and what is the fix?

- A. Both deployments now set the same `OTEL_SERVICE_NAME`, so their telemetry collapses onto one node —
  parameterise the value per service
- B. The inventory service lost its connection string
- C. `traceparent` propagation broke
- D. Sampling removed the inventory telemetry

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** lines **343–344**.

```yaml
            - name: OTEL_SERVICE_NAME
              value: "payment-service"
```

**"Both still report telemetry" is the diagnostic sentence.** Data is arriving; it is arriving under the
wrong identity.

**A shared chart with a hard-coded service name is exactly how this happens.** The value at line 344 is a
literal, and templating a manifest without parameterising it copies one service's identity onto every
service that uses the chart.

**Why B would show the inventory service disappearing entirely** rather than merging, and **why C would
break the trace between services** while leaving both nodes visible — which is the opposite symptom.

**The durable lesson:** `OTEL_SERVICE_NAME` is an **identity**, not a label. Anything that templates
deployments must treat it the way it treats the image name — per service, never inherited.

</details>

---

## Q48

A year after the rollout, a customer reports that checkout occasionally takes 30 seconds. The team finds
it in minutes.

Walk through how, and explain what changed compared with the starting state.

- A. Percentile latency identified the affected window, the end-to-end transaction view followed one slow
  `operation_Id` from the frontend through payment to a slow external dependency, and VM Insights
  confirmed the order service was not involved
- B. The team checked RDP sessions on the VM
- C. `kubectl logs` on the payment pod showed the delay
- D. The daily cost report showed a telemetry spike

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-47.md`:** lines **351–360**, **225–233**, **128–131**.

```kusto
requests
| where name == "POST /api/orders"
| join kind=inner (dependencies | project operation_Id, target, name, duration, success) on operation_Id
```

**Follow the three moves.** Percentiles find *when* — an average would have hidden a problem affecting
one checkout in fifty. The join on `operation_Id` finds *where* — following a single slow transaction
across service boundaries, which only works because `traceparent` propagated at every hop. And
`trackDependency` data names *what* — the external call, its duration and its result code.

**Why B and C describe the starting state at line 16**, and why they could never have answered this
question. An RDP session shows a machine; `kubectl logs` shows one container. **Neither can follow a
single request across three platforms**, and the delay was in a dependency of a service neither team
owned.

**What actually changed, stated plainly:** the three teams were not short of data before — the VM had
metrics, the pods had logs, and the frontend had an application that worked. **What they lacked was a
shared identifier.** `operation_Id`, carried in `traceparent` from the first hop to the last, is what
turns three separate telemetry streams into one story.

**That is the sentence to give any exam question about observability in a microservices architecture.**
Collecting more telemetry from more places produces three dashboards. **Correlation is what produces an
answer** — and it is a design decision made at instrumentation time, not a query you can write
afterwards.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **SDK proposed when "no code changes" is required** | Q1, Q42 | Three app settings and a restart |
| **App not restarted after enabling auto-instrumentation** | Q27, Q34 | The agent attaches at process start |
| **Application Insights offered for VM or cluster metrics** | Q2, Q18, Q28 | Code → App Insights. Machine → VM Insights. Cluster → Container Insights |
| **DCR created but never associated** | Q33, Q34, Q43 | Healthy agent, correct rule, no data |
| **`Microsoft-ServiceMap` omitted** | Q8, Q38, Q43 | Performance works, Map is empty |
| **Instrumentation assumed sufficient for tracing** | Q4, Q20, Q29, Q45 | `traceparent` must propagate at every hop |
| **"Same resource / same platform" for correlation** | Q20, Q29 | Correlation is on `operation_Id`, not topology |
| **Adaptive sampling assumed to keep exceptions** | Q14, Q21, Q30, Q46 | Only with `excludedTypes` |
| **`includedTypes` instead of `excludedTypes`** | Q40 | Excluded *from sampling* = kept in full |
| **Daily cap used as a cost strategy** | Q3, Q16, Q25, Q46 | Hard stop, soonest on the worst day |
| **`env_var` collection enabled** | Q19, Q26, Q39, Q44 | Copies secrets into a queryable store |
| **Container Insights and managed Prometheus seen as alternatives** | Q22, Q44 | Complementary. Enable both |
| **`APPINSIGHTS_INSTRUMENTATIONKEY`** | Q36 | Deprecated. Use the connection string |
| **Number placed in `properties` instead of `measurements`** | Q10, Q32 | Strings filter, numbers aggregate |
| **`OTEL_SERVICE_NAME` hard-coded in a shared chart** | Q47 | It is an identity. Parameterise it |
| **Averages instead of percentiles** | Q48 | An average hides the slow tail |

---

# What to memorise

**In `challenge-47.md`:** lines **57–66**, **93–131**, **137–179**, **286–306**, **314–319**.

```text
CHOOSE BY WHAT YOU ARE WATCHING - they STACK, they do not compete
  application code   Application Insights   requests, dependencies, exceptions, traces
  a virtual machine  VM Insights            Performance (CPU/mem/disk/net) + MAP (process deps)
  a Kubernetes cluster Container Insights   node & pod metrics, container stdout/stderr
  pod-exposed metrics  Managed Prometheus   az aks update --enable-azure-monitor-metrics
  reachability         Availability test    multi-region ping, alert on availabilityPercentage < 99

An app on a VM wants App Insights AND VM Insights. One workspace underneath all of it.
```

```bash
# App Service - CODELESS attach                     (lines 58-66)
az webapp config appsettings set --settings \
  "APPLICATIONINSIGHTS_CONNECTION_STRING=<conn>"    \  # NOT APPINSIGHTS_INSTRUMENTATIONKEY (deprecated)
  "ApplicationInsightsAgent_EXTENSION_VERSION=~3"   \
  "XDT_MicrosoftApplicationInsights_Mode=Recommended"
az webapp restart ...                                  # REQUIRED - the agent attaches at process start

# Workspace-based App Insights                      (lines 44-49)
az monitor app-insights component create --app <n> --workspace $LAW_ID --application-type web

# VM - Azure Monitor Agent + DCR                    (lines 93-121)
az vm extension set --name AzureMonitorLinuxAgent --publisher Microsoft.Azure.Monitor
az monitor data-collection rule create ... streams: ["Microsoft-InsightsMetrics",   # Performance tab
                                                     "Microsoft-ServiceMap"]        # MAP tab
az monitor data-collection rule association create --resource <vm-id> --rule-id $DCR_ID
#   AMA is configured CENTRALLY by DCRs (not per-machine workspace keys, like the legacy agent)
#   NO ASSOCIATION = healthy agent, correct rule, ZERO DATA

# AKS                                               (lines 137-153)
az aks enable-addons --addons monitoring --workspace-resource-id $LAW_ID   # Container Insights
az aks update --enable-azure-monitor-metrics                               # managed Prometheus
```

```toml
# container-azm-ms-agentconfig, namespace kube-system     (lines 166-179)
[log_collection_settings.stdout]
  enabled = true
  exclude_namespaces = ["kube-system","gatekeeper-system"]   # DENYLIST - new namespaces collected by default
[log_collection_settings.env_var]
  enabled = false          # LEAVE FALSE - env vars carry connection strings and secrets
[prometheus_data_collection_settings.cluster]
  interval = "1m"
  monitor_kubernetes_pods = true
# after editing:  kubectl rollout restart daemonset omsagent -n kube-system
```

```text
DISTRIBUTED TRACING - BOTH are required            (lines 314-319, 396)
  1. every service instrumented (connection string set)
  2. traceparent PROPAGATED on every outbound call:  00-<trace-id>-<span-id>-<trace-flags>
  A service that drops the header breaks the trace AT THAT HOP - and still appears on the map.
  Correlation key = operation_Id.  Different resources / platforms still correlate.
  OTEL_SERVICE_NAME = the node's IDENTITY on the map. Parameterise it per service.

  requests | where name == "POST /api/orders" | project operation_Id, duration, resultCode
  | join kind=inner (dependencies | project operation_Id, target, name, duration, success) on operation_Id
  | project-away operation_Id1
```

```text
SDK CALLS                                           (lines 194-233)
  trackEvent       properties = STRINGS (filter/group)   measurements = NUMBERS (aggregate)
  trackMetric      one named numeric value
  trackDependency  target, name, data, duration, resultCode, success, dependencyTypeName -> APPLICATION MAP
  setAutoCollectRequests / Dependencies / Exceptions -> most of it for free

COST                                                (lines 286-306)
  adaptive sampling   UseAdaptiveSampling(maxTelemetryItemsPerSecond: 5)   varies % to hit a RATE
  fixed-rate          UseSampling(25.0)                                    constant %
  KEEP ERRORS         excludedTypes: "Event;Exception"   <- excluded FROM SAMPLING = kept in full
                      adaptive sampling does NOT preserve exceptions by default
  ingestion sampling  server-side, applies regardless of SDK settings
  daily cap           az monitor app-insights component billing update --cap 5
                      HARD STOP. reached soonest on the worst day. backstop, not a strategy
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 48 |
| 38–43 | Re-read the trap index and the platform table, then move on |
| 30–37 | Rewrite the platform table and the two tracing requirements from memory, then retake |
| Below 30 | Redo Tasks 1, 2 and 3 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 47.

:::danger The two facts

**Auto-instrumentation needs no code** — the connection string, the agent version, `Recommended`, and a
restart.

**Instrumenting every service is not enough for tracing.** `traceparent` must propagate at every hop, or
the trace stops there while everything still looks healthy.

Three teams with three dashboards were not short of data. They were short of a shared `operation_Id`.

:::
