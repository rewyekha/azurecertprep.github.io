---
sidebar_position: 92
title: "Challenge 26: exam questions"
---

# Challenge 26 — AZ-400 exam questions

**48 questions** built only from what Challenge 26 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-26.md`**.

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

:::tip What separates this from Challenge 25

Challenge 25 was *choosing* a strategy. This one is the **mechanics of not dropping requests**:
warm-up, health probes, batch sizes and drain-then-return. Almost every question here turns on
*when* an instance is allowed to receive traffic.

:::

---

# Section A — Single answer

---

## Q1

Contoso runs 4 VMSS instances and wants no more than 25% unavailable during an update.

Which setting enforces this?

- A. `maxBatchInstancePercent=25`
- B. `maxUnhealthyInstancePercent=25`
- C. `pauseTimeBetweenBatches="PT30S"`
- D. `automaticRepairsPolicy.enabled=true`

### Answer: A

**In `challenge-26.md`:** lines **180–187**.

```bash
az vmss update \
  --set upgradePolicy.mode=Rolling \
  --set upgradePolicy.rollingUpgradePolicy.maxBatchInstancePercent=25 \
  --set upgradePolicy.rollingUpgradePolicy.maxUnhealthyInstancePercent=25 \
  --set upgradePolicy.rollingUpgradePolicy.maxUnhealthyUpgradedInstancePercent=25 \
  --set upgradePolicy.rollingUpgradePolicy.pauseTimeBetweenBatches="PT30S"
```

**`maxBatchInstancePercent` is the batch size** — how many instances are taken out at once. With 4
instances, 25% = one instance per batch.

**Why the others fail — all three are real settings with different jobs**

- **B** — `maxUnhealthyInstancePercent` is an **abort threshold**: if more than this share of the
  *whole* scale set is unhealthy, the upgrade stops. It does not control batch size
- **C** — the pause **between** batches, giving instances time to settle before the next batch starts
- **D** — automatic repairs replace unhealthy instances during normal operation, not during an upgrade

**The distinction that gets tested:** batch size controls *how many at a time*; the unhealthy
percentages control *when to give up*.

---

## Q2

A VMSS rolling update processes 25% of instances and then halts.

What is the most likely cause?

- A. `pauseTimeBetweenBatches` is too long
- B. The first batch is failing health checks and hit the unhealthy threshold
- C. The scale set has reached its instance limit
- D. The custom script extension timed out

### Answer: B

**In `challenge-26.md`:** Break & fix Exercise 2, lines **666–698**.

> *"The first batch of updated instances is failing health checks. The `maxUnhealthyUpgradedInstancePercent`
> threshold is met, blocking further updates."*

**This is the rolling upgrade working correctly.** It updated one batch, saw the new version was
unhealthy, and stopped rather than breaking the remaining 75%.

**The fix (lines 693–697)** is two parts, and the order matters:

```bash
# 1. Fix the application defect
# 2. Then restart the upgrade
az vmss rolling-upgrade start --name $VMSS_NAME --resource-group $RESOURCE_GROUP
```

Restarting without fixing the application just fails at the same point.

**Why the others fail**

- **A** — a long pause makes it *slow*, not stopped
- **C** — instance limits block scaling out, not upgrading in place
- **D** — an extension timeout would fail individual instances, and the diagnostic at lines 677–682
  shows provisioning states, not extension errors

---

## Q3

Code is deployed to the staging slot but auto-swap never happens.

What is the most likely cause?

- A. The App Service Plan is on the Free or Basic tier
- B. The staging slot has no traffic routing configured
- C. `WEBSITE_WARMUP_PATH` is not set
- D. The production slot is stopped

### Answer: A

**In `challenge-26.md`:** Break & fix Exercise 1, lines **628–662**.

> *"The App Service plan is on the Free or Basic tier, which does not support auto-swap."*

```bash
az appservice plan update --sku S1      # Standard or higher
```

Same tier constraint as slots themselves — F1 and B1 support neither.

**Why the others fail**

- **B** — traffic routing (line 62) splits live traffic for testing. It is unrelated to swapping
- **C** — **the interesting distractor.** Without a warm-up path the swap still happens, it is just
  cold (Break & fix Exercise 3). Missing warm-up causes a *slow* swap, not *no* swap
- **D** — a stopped target would fail the swap with an error, not silently skip it

**The diagnostic to remember (lines 634–645):** check `autoSwapSlotName`, then check the activity log
for `slotsswap` operations. If the config is set and no operation was ever attempted, look at the
tier.

---

## Q4

How does auto-swap decide when to perform the swap?

- A. Immediately after the deployment completes
- B. After Azure warms up the slot by requesting its root path
- C. After a fixed 60-second delay
- D. After the health check path returns 200 three times

### Answer: B

**In `challenge-26.md`:** lines **95–100**.

```text
1. Code is deployed to the staging slot
2. Azure automatically warms up the staging slot by sending requests to its root path
3. Once warm-up is complete, Azure performs the swap automatically
4. If warm-up fails, the swap does not occur
```

**Step 4 is the safety property.** A slot that will not warm up never reaches production.

**Why the others fail**

- **A** — deployment completing is not the same as the application being ready
- **C** — no fixed delay. It waits for the warm-up result
- **D** — describes VMSS health probe semantics (`numberOfProbes: 3` at line 340), a different
  mechanism

**Refinement worth knowing:** by default warm-up pings the **root path**. Setting
`WEBSITE_SWAP_WARMUP_PING_PATH` (line 377) redirects that to a real readiness endpoint, which is far
more meaningful than whether `/` returns something.

---

## Q5

Which command routes 10% of live production traffic to the staging slot?

- A. `az webapp traffic-routing set --distribution staging=10`
- B. `az network traffic-manager endpoint update --weight 10`
- C. `az webapp deployment slot swap --percentage 10`
- D. `az webapp config set --traffic staging=10`

### Answer: A

**In `challenge-26.md`:** lines **62–65**.

```bash
az webapp traffic-routing set \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --distribution staging=10
```

**This is App Service's built-in canary**, and it beats Traffic Manager on two counts: it routes
**per request** at the App Service front end rather than in DNS, so the split is accurate and changes
apply immediately.

**Why the others fail**

- **B** — real, but that is Traffic Manager: DNS-level, probabilistic, TTL-delayed (Challenge 25 Q3)
- **C** — a swap has no percentage. It is atomic
- **D** — not a valid command

**The exam-relevant comparison:**

| Need | Use |
|---|---|
| Accurate % split within one App Service | `az webapp traffic-routing` |
| Split across regions or services | Front Door (per request) or Traffic Manager (DNS) |
| Instant full cutover | Slot swap |

---

## Q6

In an Azure Pipelines rolling deployment, which lifecycle hook re-adds an instance to the load
balancer?

- A. `preDeploy`
- B. `deploy`
- C. `routeTraffic`
- D. `postRouteTraffic`

### Answer: C

**In `challenge-26.md`:** lines **273–282**.

```yaml
        strategy:
          rolling:
            maxParallel: 25%
            preDeploy:        # drain the instance from the load balancer
            deploy:           # install the new version
            routeTraffic:     # add it back to the load balancer
            postRouteTraffic: # verify it is healthy under real traffic
            on:
              failure:        # roll this instance back
```

**The four hooks are a complete drain-update-return-verify cycle**, and the order is the entire point
of rolling deployment: never update an instance that is still serving requests.

**Why the others fail**

- **A** — `preDeploy` **drains** the instance (lines 242–252)
- **B** — `deploy` installs the new version while the instance is out of rotation
- **D** — `postRouteTraffic` verifies health **after** traffic returns (lines 283–298)

---

## Q7

Which VMSS extension reports instance health to the platform?

- A. `CustomScript`
- B. `ApplicationHealthLinux`
- C. `DependencyAgentLinux`
- D. `AzureMonitorLinuxAgent`

### Answer: B

**In `challenge-26.md`:** lines **329–342**.

```bash
az vmss extension set \
  --name ApplicationHealthLinux \
  --publisher Microsoft.ManagedServices \
  --settings '{
    "protocol": "http",
    "port": 8080,
    "requestPath": "/health",
    "intervalInSeconds": 5,
    "numberOfProbes": 3,
    "gracePeriod": 600
  }'
```

Without it, the platform only knows whether the **VM** is running — not whether your **application**
inside it works. A rolling upgrade would happily march through every instance replacing a working app
with a broken one.

**Why the others fail**

- **A** — `CustomScript` runs commands; it is what deploys the app at lines 266–272
- **C** and **D** — monitoring and dependency mapping agents. They observe; they do not gate upgrades

**Note `gracePeriod: 600`** — ten minutes for a new instance to become healthy before it counts as
unhealthy. Set it below your real startup time and the platform will kill instances that were merely
still booting.

---

## Q8

After a slot swap, the first requests take 10–15 seconds.

What is the fix?

- A. Increase the App Service Plan instance count
- B. Configure `WEBSITE_SWAP_WARMUP_PING_PATH` and `WEBSITE_SWAP_WARMUP_PING_STATUSES`
- C. Enable auto-swap
- D. Add a Traffic Manager health probe

### Answer: B

**In `challenge-26.md`:** Break & fix Exercise 3, lines **702–732**.

```bash
az webapp config appsettings set --slot staging \
  --settings \
    "WEBSITE_SWAP_WARMUP_PING_PATH=/health/ready" \
    "WEBSITE_SWAP_WARMUP_PING_STATUSES=200"
```

> *"No warm-up path is configured. Azure performs the swap without ensuring the application is fully
> initialized."*

The cost is paid either way — the question is whether **your pipeline** pays it before the swap or
**your users** pay it after.

**Why the others fail**

- **A** — more instances means more cold instances. It multiplies the problem
- **C** — auto-swap does warm up the **root path**, which may return quickly while caches and
  connections are still cold. Setting an explicit readiness path is what actually helps
- **D** — a probe removes an unhealthy endpoint from rotation. It does not warm anything

---

## Q9

Which setting makes an app setting **swap** with the code rather than staying with the slot?

- A. `--slot-settings`
- B. `--settings`
- C. `--sticky-settings`
- D. `--slot-config-names`

### Answer: B

**In `challenge-26.md`:** lines **138–144**.

```bash
# Non-sticky - these WILL swap with the app code
az webapp config appsettings set \
  --settings \
    "API_VERSION=v2.3.1" \
    "FEATURE_NEW_CHECKOUT=true"
```

Compare with lines 120–126, which use `--slot-settings` for `ENVIRONMENT`, `CACHE_CONNECTION` and the
App Insights key.

**The test:** *does this value describe the code, or the place?* `API_VERSION` describes the build,
so it travels. `ENVIRONMENT=production` describes the slot, so it stays.

**Why the others fail**

- **A** — the opposite: makes it sticky
- **C** and **D** — not CLI flags. "Sticky" is the informal name for a slot setting

---

## Q10

Which Azure Pipelines task performs a slot swap?

- A. `AzureWebApp@1` with `slotName`
- B. `AzureAppServiceManage@0` with `action: 'Swap Slots'`
- C. `AzureCLI@2` only
- D. `AzureRmWebAppDeployment@4` with `swapSlot: true`

### Answer: B

**In `challenge-26.md`:** lines **575–582**.

```yaml
                - task: AzureAppServiceManage@0
                  displayName: 'Swap staging to production'
                  inputs:
                    azureSubscription: $(azureSubscription)
                    action: 'Swap Slots'
                    webAppName: $(appName)
                    resourceGroupName: $(resourceGroup)
                    sourceSlot: 'staging'
```

Note only `sourceSlot` is given — the target defaults to production.

**Why the others fail**

- **A** — `AzureWebApp@1` **deploys** to a slot (line 500). It does not swap
- **C** — an `az webapp deployment slot swap` script works, but the question asks for the task, and
  the dedicated task handles service-connection auth for you
- **D** — that task deploys and can target a slot, but has no swap action

**`AzureAppServiceManage@0` also does** Start, Stop, Restart, Delete Slot and Install Extensions. The
action parameter is what makes it a swap.

---

## Q11

Which Azure Pipelines condition triggers a rollback stage only when the previous stage failed?

- A. `condition: always()`
- B. `condition: failed()`
- C. `condition: succeededOrFailed()`
- D. `condition: eq(variables['Build.Reason'], 'Failed')`

### Answer: B

**In `challenge-26.md`:** lines **603–606**.

```yaml
  - stage: Rollback
    displayName: 'Rollback on failure'
    dependsOn: SwapToProduction
    condition: failed()
```

**Why the others fail**

- **A** — `always()` would roll back after **successful** swaps too, undoing every good release
- **C** — runs on success **and** failure. Same problem as A
- **D** — `Build.Reason` describes what triggered the run (`IndividualCI`, `Manual`, `Schedule`), not
  its outcome

**Platform note:** Azure Pipelines uses `failed()`; GitHub Actions uses `failure()`. Same idea,
different spelling, and the exam swaps them.

---

## Q12

What does `##vso[task.logissue type=error]` do in a pipeline script?

- A. Writes an error to the pipeline log and marks the task as having an issue
- B. Immediately cancels the pipeline
- C. Sends an email to the pipeline owner
- D. Creates a work item in Azure Boards

### Answer: A

**In `challenge-26.md`:** lines **296** and **600**.

```bash
                        echo "##vso[task.logissue type=error]Health check failed"
                        exit 1
```

It is a **logging command** — a specially formatted line the agent interprets. It surfaces the message
in the run summary as a proper error annotation rather than a buried log line.

**Note it is paired with `exit 1`.** The logging command reports; the exit code fails the task. Log an
issue without exiting non-zero and the step still passes.

**Why the others fail**

- **B** — it does not cancel anything. The exit code decides the outcome
- **C** — notifications are configured separately
- **D** — no work item is created

**Other logging commands worth recognising:** `task.setvariable`, `task.setprogress`,
`task.complete`, `artifact.upload`.

---

## Q13

Which VMSS setting automatically replaces instances that stay unhealthy?

- A. `automaticRepairsPolicy.enabled=true`
- B. `upgradePolicy.mode=Rolling`
- C. `maxUnhealthyInstancePercent`
- D. `pauseTimeBetweenBatches`

### Answer: A

**In `challenge-26.md`:** lines **345–349**.

```bash
az vmss update \
  --set automaticRepairsPolicy.enabled=true \
  --set automaticRepairsPolicy.gracePeriod="PT30M"
```

This is **ongoing operational healing**, not a deployment feature: an instance that reports unhealthy
for longer than the grace period is deleted and replaced.

`PT30M` is ISO 8601 duration — thirty minutes. Compare with `PT30S` at line 187, thirty seconds. Being
able to read that format is worth an exam mark.

**Why the others fail**

- **B** — governs how upgrades roll out
- **C** — an abort threshold during an upgrade
- **D** — pacing between batches

**It depends on the health extension.** Without `ApplicationHealthLinux` (Q7), the platform has no
health signal to repair against.

---

## Q14

Which App Service configuration tells the platform which path to probe for instance health?

- A. `WEBSITE_SWAP_WARMUP_PING_PATH`
- B. `healthCheckPath`
- C. `WEBSITE_WARMUP_PATH`
- D. `applicationInitialization`

### Answer: B

**In `challenge-26.md`:** lines **356–359**.

```bash
az webapp config set \
  --generic-configurations '{"healthCheckPath":"/health"}'
```

**Three similar-looking settings, three different jobs — this is the most confusable set in the
challenge:**

| Setting | When it applies | What it does |
|---|---|---|
| `healthCheckPath` | Continuously | Removes unhealthy **instances** from rotation |
| `WEBSITE_SWAP_WARMUP_PING_PATH` | During a **swap** | Blocks the swap until this path answers |
| `WEBSITE_WARMUP_PATH` | On app **start** | Path hit to warm the app |

**Why the others fail** — each is real, and none is the ongoing instance health probe.

---

## Q15

In the rolling strategy, what does `maxParallel: 25%` mean with 4 instances?

- A. One instance is updated at a time
- B. All four are updated with a 25% delay
- C. 25% of requests go to the new version
- D. The update completes in four batches of 25% each

### Answer: A

**In `challenge-26.md`:** line **241**.

```yaml
        strategy:
          rolling:
            maxParallel: 25%
```

25% of 4 = **1 instance per batch**, so four batches of one.

**D is deliberately close and subtly wrong.** It says the same arithmetic but describes it as the
*update completing* in four batches, which conflates batch count with batch size. `maxParallel` is
the **size** of a batch, not how many batches there are.

**Why the others fail**

- **B** — nothing is updated simultaneously with a delay; batches are sequential
- **C** — that is traffic splitting (Q5), a completely different mechanism

**`maxParallel` accepts a percentage or an absolute number.** `maxParallel: 1` would be identical
here and clearer.

---

## Q16

Which warm-up mechanism is specific to **Windows** App Service?

- A. `WEBSITE_SWAP_WARMUP_PING_PATH`
- B. The `applicationInitialization` element in `web.config`
- C. `healthCheckPath`
- D. The `ApplicationHealthLinux` extension

### Answer: B

**In `challenge-26.md`:** lines **386–397**.

```xml
    <applicationInitialization doAppInitAfterRestart="true">
      <add initializationPage="/health" hostName="" />
      <add initializationPage="/api/products" hostName="" />
      <add initializationPage="/api/categories" hostName="" />
    </applicationInitialization>
```

This is an **IIS** feature, so it only exists on Windows App Service. It can warm **several** pages,
which the single-path app setting cannot.

**Why the others fail**

- **A** and **C** — platform-agnostic app settings, working on Linux and Windows alike
- **D** — a **VMSS** extension, and its name says Linux

---

# Section B — Multiple answer

---

## Q17

Which **four** lifecycle hooks does the Azure Pipelines `rolling` strategy provide? (Choose four.)

- A. `preDeploy`
- B. `deploy`
- C. `routeTraffic`
- D. `postRouteTraffic`
- E. `preValidate`
- F. `postSwap`

### Answer: A, B, C, D

**In `challenge-26.md`:** lines **242**, **253**, **273**, **283**.

```yaml
        strategy:
          rolling:
            maxParallel: 25%
            preDeploy:         # A - drain from the load balancer
            deploy:            # B - install the new version
            routeTraffic:      # C - return to the load balancer
            postRouteTraffic:  # D - verify under real traffic
            on:
              failure:         # roll this instance back
              success:
```

`preValidate` and `postSwap` do not exist.

**The same four hooks are available to `runOnce` and `canary` strategies.** `canary` adds an
`increments` setting. That trio — `runOnce`, `rolling`, `canary` — is the complete set of deployment
job strategies, and it is testable.

---

## Q18

Which **two** settings control when a VMSS rolling upgrade aborts? (Choose two.)

- A. `maxUnhealthyInstancePercent`
- B. `maxUnhealthyUpgradedInstancePercent`
- C. `maxBatchInstancePercent`
- D. `pauseTimeBetweenBatches`
- E. `automaticRepairsPolicy.gracePeriod`

### Answer: A, B

**In `challenge-26.md`:** lines **185–186**, and the failure at line **689**.

| Setting | Measures |
|---|---|
| `maxUnhealthyInstancePercent` | Unhealthy share of the **whole** scale set |
| `maxUnhealthyUpgradedInstancePercent` | Unhealthy share of **already-upgraded** instances |

The second one is the sharper signal: it isolates instances running the **new** version, so it
detects a bad release rather than pre-existing noise. That is the threshold that stopped the upgrade
in Break & fix Exercise 2.

**Why the others fail**

- **C** — batch size (Q1)
- **D** — pacing between batches
- **E** — replacement of unhealthy instances during normal operation, not during an upgrade

---

## Q19

Which **two** are true about App Service auto-swap? (Choose two.)

- A. It requires Standard tier or higher
- B. It swaps only after warm-up completes successfully
- C. It can be disabled by setting the auto-swap slot to an empty string
- D. It bypasses environment approvals
- E. It performs a rollback automatically if production fails

### Answer: A, B

**C is also true** (lines 105–109 disable it with `--auto-swap-slot ""`), so A and B are the intended
pair — the two *behaviours*, while C is the off switch.

**In `challenge-26.md`:** line **652** (A), lines **97–100** (B).

**Why the others fail**

- **D** — worth stating precisely: auto-swap is an **App Service** feature, so it fires on deployment
  without consulting any pipeline environment. That is not "bypassing" a gate — it means there is no
  gate to bypass. **If you need approval before production, do not use auto-swap.** Use an explicit
  swap step behind an environment (lines 565–582)
- **E** — no automatic rollback. That is the pipeline's job (lines 603–621)

---

## Q20

Which **two** settings should be sticky on the staging slot? (Choose two.)

- A. `CACHE_CONNECTION` pointing at the staging Redis instance
- B. `APPINSIGHTS_INSTRUMENTATIONKEY` for the staging resource
- C. `API_VERSION=v2.3.1`
- D. `FEATURE_NEW_CHECKOUT=true`
- E. The published application build

### Answer: A, B

**In `challenge-26.md`:** lines **129–136** (sticky) versus lines **139–144** (non-sticky).

```bash
# Sticky - describes WHERE
--slot-settings "ENVIRONMENT=staging" "CACHE_CONNECTION=redis-contoso-staging..." \
                "APPINSIGHTS_INSTRUMENTATIONKEY=<staging-key>"

# Non-sticky - describes WHAT
--settings "API_VERSION=v2.3.1" "FEATURE_NEW_CHECKOUT=true"
```

**Why the App Insights key must be sticky:** if it swapped, staging traffic would report into the
production Application Insights resource. Your production error rate would spike every time someone
tested in staging, and Challenge 25's rollback alert would fire on a phantom.

**Why the others fail**

- **C** and **D** — describe the build. They should travel with it, or production would run new code
  while advertising the old version
- **E** — the build is the content. Making it sticky would break swapping entirely

---

## Q21

Which **two** are required for VMSS automatic instance repair to work? (Choose two.)

- A. `automaticRepairsPolicy.enabled=true`
- B. An application health extension or a load balancer health probe
- C. `upgradePolicy.mode=Rolling`
- D. `maxBatchInstancePercent` set below 100
- E. A minimum of four instances

### Answer: A, B

**In `challenge-26.md`:** lines **345–349** (A) and **329–342** (B).

Automatic repair needs a **health signal** to act on. Enabling the policy without a health extension
gives the platform nothing to evaluate, so nothing is ever repaired.

**Why the others fail**

- **C** — upgrade mode governs deployments. Repairs happen during normal operation
- **D** — a batch setting, unrelated
- **E** — no minimum instance count is required

**`gracePeriod: "PT30M"`** stops repairs firing during normal startup. Too short and healthy
instances get destroyed while still booting.

---

## Q22

Which **two** improve availability during a rolling deployment? (Choose two.)

- A. Draining an instance from the load balancer before updating it
- B. Verifying instance health after traffic returns
- C. Updating all instances simultaneously to shorten the window
- D. Disabling health probes during the update
- E. Setting `maxBatchInstancePercent=100`

### Answer: A, B

**In `challenge-26.md`:** lines **242–252** (A) and **283–298** (B).

**Why the others fail — all three are the original incident**

The scenario at line 16 describes exactly this: *"all instances were updated simultaneously, causing
a 3-minute outage that resulted in 4,200 failed requests."*

- **C** and **E** — the same mistake stated two ways. Zero capacity during the update
- **D** — without probes, unhealthy instances stay in rotation and the upgrade never aborts. You lose
  both protections at once

---

## Q23

Which **two** distinguish App Service traffic routing from Traffic Manager weighted routing? (Choose
two.)

- A. App Service traffic routing is applied per request
- B. Traffic Manager weighting is applied at DNS resolution
- C. App Service traffic routing requires a Traffic Manager profile
- D. Traffic Manager can split traffic within a single App Service
- E. App Service traffic routing changes require a slot swap

### Answer: A, B

**In `challenge-26.md`:** lines **62–65**; Traffic Manager behaviour is Challenge 25 line **594**.

```bash
az webapp traffic-routing set --distribution staging=10
```

**The practical consequences:**

| | App Service traffic routing | Traffic Manager weighted |
|---|---|---|
| Level | Per request, at the front end | Per DNS resolution |
| Accuracy | Exact percentage | Probabilistic |
| Change speed | Immediate | Delayed by TTL |
| Scope | Slots of one app | Endpoints across regions |

**Why the others fail**

- **C** — completely independent features
- **D** — Traffic Manager routes to endpoints, not between slots of one app
- **E** — routing changes apply immediately; a swap is a separate operation

---

# Section C — Repeated scenario

**Scenario:** Contoso's last deployment updated all 4 instances at once, causing a 3-minute outage and
4,200 failed requests. They need deployments that keep the site available throughout, with no cold
start for users.

---

## Q24

**Proposed solution:** Deploy to a staging slot, configure `WEBSITE_SWAP_WARMUP_PING_PATH` with an
expected status of 200, validate staging, then swap to production.

Does this meet the goal? **Yes**

### Answer: Yes

**In `challenge-26.md`:** lines **372–378** and **565–582**.

Both halves are covered:

- **Availability** — production keeps serving throughout. The swap is atomic and exchanges already
  warmed instances
- **No cold start** — the swap will not complete until `/health/ready` returns 200, so the
  application is initialised before any user reaches it

This is the intended answer, and it directly fixes both symptoms in the incident.

---

## Q25

**Proposed solution:** Configure a VMSS rolling upgrade with `maxBatchInstancePercent=100` and
`pauseTimeBetweenBatches="PT30S"`.

Does this meet the goal? **No**

### Answer: No

`maxBatchInstancePercent=100` means **one batch containing every instance** — precisely what caused
the original outage.

The 30-second pause is irrelevant: with a single batch there is nothing to pause between.

**The setting to change is the batch size, not the pause.** At 25% (line 184) with 4 instances, three
instances keep serving while one updates.

**The exam pattern here:** the proposal uses the right *feature* with a value that defeats it. Read
the numbers, not just the keywords.

---

## Q26

**Proposed solution:** Enable auto-swap on the staging slot so deployments promote automatically after
warm-up.

Does this meet the goal? **No**

### Answer: No

**Availability: yes.** Auto-swap performs a normal atomic swap, and it only swaps after warm-up
succeeds (lines 97–100), so cold start is handled too.

**So why No?** Look at the scenario's other constraint — Contoso reached this point *because* an
unvalidated deployment caused an outage. Auto-swap removes the validation stage entirely: there is no
opportunity for smoke tests (lines 536–563) or an approval before production.

The requirement is not merely "no downtime" — it is "do not repeat the last incident". Auto-swap
makes bad releases reach production **faster**.

**Where auto-swap does belong:** development and test environments, where speed matters more than
gates.

---

# Section D — Yes/No statement grid

---

## Q27 — deployment slots and swapping

| # | Statement | Answer |
|---|---|---|
| 1 | Auto-swap requires Standard tier or higher | **Yes** |
| 2 | Auto-swap performs the swap even if warm-up fails | **No** |
| 3 | `az webapp traffic-routing` splits traffic per request | **Yes** |
| 4 | A slot swap is atomic from the client's perspective | **Yes** |

**In `challenge-26.md`:** line **652** (row 1), line **100** (row 2), lines **62–65** (row 3).

Row 2 is the safety property that makes auto-swap acceptable at all: *"If warm-up fails, the swap does
not occur."*

Row 4 explains why blue-green gives zero downtime — there is no moment when neither slot is serving.

---

## Q28 — rolling deployments

| # | Statement | Answer |
|---|---|---|
| 1 | `maxBatchInstancePercent` controls how many instances update at once | **Yes** |
| 2 | A rolling upgrade continues even when the first batch is unhealthy | **No** |
| 3 | `pauseTimeBetweenBatches` uses ISO 8601 duration format | **Yes** |
| 4 | Rolling deployments require duplicate infrastructure | **No** |

**In `challenge-26.md`:** line **184** (row 1), line **689** (row 2), line **187** (row 3).

Row 3: `PT30S` is 30 seconds, `PT30M` is 30 minutes (line 349), `PT1H` is an hour. Reading this format
is a small, reliable exam mark.

Row 4 is the cost argument from Challenge 25's table: rolling is **1x** infrastructure because it
updates in place, versus blue-green's 2x.

---

## Q29 — health probes

| # | Statement | Answer |
|---|---|---|
| 1 | `ApplicationHealthLinux` reports application-level health to the platform | **Yes** |
| 2 | `healthCheckPath` removes unhealthy App Service instances from rotation | **Yes** |
| 3 | Automatic instance repair works without a health signal | **No** |
| 4 | `gracePeriod` prevents repairs during instance startup | **Yes** |

**In `challenge-26.md`:** lines **329–342**, **356–359**, **345–349**.

Row 3 is the dependency people miss: `automaticRepairsPolicy.enabled=true` on its own does nothing.
The platform must be able to *tell* an instance is unhealthy, which needs the extension or a load
balancer probe.

Row 4: set `gracePeriod` shorter than your real startup time and you get a repair loop — instances
killed while booting, replaced, killed again.

---

## Q30 — settings and swaps

| # | Statement | Answer |
|---|---|---|
| 1 | `--slot-settings` values stay with the slot during a swap | **Yes** |
| 2 | `--settings` values swap with the application code | **Yes** |
| 3 | Connection strings can be marked as slot settings | **Yes** |
| 4 | The App Insights key should swap with the code | **No** |

**In `challenge-26.md`:** lines **120–144** (rows 1 and 2), lines **151–165** (row 3), line **126**
(row 4).

Row 4 has a concrete consequence worth remembering: a swapping instrumentation key means **staging
telemetry lands in the production Application Insights resource**. Your production dashboards, alerts
and DORA metrics all become wrong, and the rollback alert from Challenge 25 could fire because someone
ran a load test in staging.

---

# Section E — Drag and drop

---

## Q31

Arrange the rolling deployment lifecycle hooks in execution order.

**Items:** `routeTraffic` · `preDeploy` · `postRouteTraffic` · `deploy`

### Answer: `preDeploy` → `deploy` → `routeTraffic` → `postRouteTraffic`

**In `challenge-26.md`:** lines **242**, **253**, **273**, **283**.

| Hook | What happens | Instance serving traffic? |
|---|---|---|
| `preDeploy` | Drain from the load balancer | Draining |
| `deploy` | Install the new version | **No** |
| `routeTraffic` | Return to the load balancer | Yes |
| `postRouteTraffic` | Verify health under real traffic | Yes |

**The order is the entire safety property.** An instance is never updated while serving, and it is
verified after it starts serving again — because working in isolation does not prove it works under
real load. If `postRouteTraffic` fails, `on: failure` (line 299) rolls that instance back.

---

## Q32

Match each setting to what it controls.

| Setting | Controls |
|---|---|
| `maxBatchInstancePercent` | **How many instances update at once** |
| `maxUnhealthyInstancePercent` | **Abort threshold across the whole scale set** |
| `maxUnhealthyUpgradedInstancePercent` | **Abort threshold among upgraded instances** |
| `pauseTimeBetweenBatches` | **Wait between batches** |
| `automaticRepairsPolicy.gracePeriod` | **Time before an unhealthy instance is replaced** |

**In `challenge-26.md`:** lines **184–187** and **349**.

**Two groups:** batch size and pause control **pace**; the two unhealthy percentages control **when to
stop**. Grace period is not an upgrade setting at all — it belongs to ongoing repair.

---

## Q33

Match each configuration to its purpose.

| Configuration | Purpose |
|---|---|
| `healthCheckPath` | **Remove unhealthy instances from rotation, continuously** |
| `WEBSITE_SWAP_WARMUP_PING_PATH` | **Block a swap until the app is ready** |
| `WEBSITE_WARMUP_PATH` | **Path hit when the app starts** |
| `applicationInitialization` in `web.config` | **Warm several pages on Windows App Service** |
| `ApplicationHealthLinux` extension | **Report VMSS application health to the platform** |

**In `challenge-26.md`:** lines **356–359**, **377**, **379**, **386–397**, **329–342**.

**This is the most confusable group in the challenge.** Sort them by *when* they run: continuously
(health check), during a swap (swap warm-up), at app start (warm-up path and applicationInit), and
continuously on VMSS (health extension).

---

## Q34

Arrange the Azure Pipelines slot-swap pipeline stages in order.

**Items:** `SwapToProduction` · `Build` · `Rollback` · `ValidateStaging` · `DeployStaging`

### Answer

1. `Build` — line **468**
2. `DeployStaging` — line **490**, `environment: 'staging'`
3. `ValidateStaging` — line **536**, smoke tests
4. `SwapToProduction` — line **565**, `environment: 'production'` ← the gate
5. `Rollback` — line **603**, `condition: failed()`

```yaml
  - stage: ValidateStaging
    dependsOn: DeployStaging
  - stage: SwapToProduction
    dependsOn: ValidateStaging
  - stage: Rollback
    dependsOn: SwapToProduction
    condition: failed()
```

**Rollback is a stage, not a step**, so it can react to the swap stage failing for any reason —
including the post-swap validation loop at lines 592–601.

**Note `ValidateStaging` is a plain `job`, not a `deployment`** (line 540). It only reads, so it needs
no environment and no approval.

---

## Q35

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Auto-swap never fires | **App Service Plan on Free or Basic tier** |
| Rolling update halts after one batch | **Upgraded instances failing health checks** |
| First requests after a swap take 10–15 seconds | **No warm-up path configured** |
| Production telemetry spikes when staging is tested | **App Insights key not marked as a slot setting** |

**In `challenge-26.md`:** lines **652**, **689**, **720**, **126**.

**The diagnostic habit:** each symptom points at a different layer — platform tier, application
health, application startup, and configuration stickiness. Name the layer first.

---

# Section F — Hot area

---

## Q36

```bash
az vmss update \
  --set upgradePolicy.mode=[BLANK 1] \
  --set upgradePolicy.rollingUpgradePolicy.[BLANK 2]=25 \
  --set upgradePolicy.rollingUpgradePolicy.pauseTimeBetweenBatches="[BLANK 3]"
```

Requirement: update one instance at a time out of four, pausing 30 seconds between batches.

- **BLANK 1:** `Rolling` / `Automatic` / `Manual` / `Batch`
- **BLANK 2:** `maxBatchInstancePercent` / `maxUnhealthyInstancePercent` / `maxParallel` / `batchSize`
- **BLANK 3:** `PT30S` / `30s` / `00:00:30` / `PT30M`

### Answer: `Rolling`, `maxBatchInstancePercent`, `PT30S`

**In `challenge-26.md`:** lines **183–187**.

The other upgrade modes are real: `Automatic` updates everything at once with no health gating, and
`Manual` requires you to trigger each instance yourself. `PT30M` would be thirty **minutes**.

---

## Q37

```bash
az webapp deployment slot [BLANK 1] \
  --name $APP_NAME \
  --slot staging \
  --[BLANK 2] production
```

Requirement: deployments to staging should promote themselves automatically.

- **BLANK 1:** `auto-swap` / `swap` / `create` / `config`
- **BLANK 2:** `auto-swap-slot` / `target-slot` / `destination` / `promote-to`

### Answer: `auto-swap`, `auto-swap-slot`

**In `challenge-26.md`:** lines **81–85**.

Note the near-miss: `az webapp deployment slot swap --target-slot production` performs a **one-off
manual swap**. This command **configures** automatic behaviour. One does it now; the other arranges
for it to happen every time.

To disable, pass an empty string (line 109).

---

## Q38

```bash
az webapp config appsettings set \
  --slot staging \
  --[BLANK 1] "ENVIRONMENT=staging" "CACHE_CONNECTION=redis-contoso-staging..." \
  --[BLANK 2] "API_VERSION=v2.3.1"
```

- **BLANK 1:** `slot-settings` / `settings` / `sticky` / `slot-config`
- **BLANK 2:** `settings` / `slot-settings` / `app-settings` / `shared`

### Answer: `slot-settings`, `settings`

**In `challenge-26.md`:** lines **133** and **142**.

`ENVIRONMENT` and `CACHE_CONNECTION` describe **where**, so they stay. `API_VERSION` describes
**what**, so it travels with the build.

(In practice these are two separate commands; combined here to test the distinction.)

---

## Q39

```yaml
      - deployment: RollingDeploy
        environment: 'production'
        strategy:
          [BLANK 1]:
            maxParallel: [BLANK 2]
            preDeploy:
            deploy:
            routeTraffic:
            postRouteTraffic:
```

- **BLANK 1:** `rolling` / `runOnce` / `canary` / `matrix`
- **BLANK 2:** `25%` / `4` / `100%` / `all`

### Answer: `rolling`, `25%`

**In `challenge-26.md`:** lines **240–241**.

`runOnce` executes the steps once, not per instance. `canary` is the third strategy, adding
`increments`. `matrix` is for regular jobs, not deployment jobs.

`100%` would update all four at once — the original outage.

---

## Q40

```bash
az vmss extension set \
  --name [BLANK 1] \
  --publisher Microsoft.ManagedServices \
  --settings '{
    "requestPath": "/health",
    "intervalInSeconds": 5,
    "numberOfProbes": 3,
    "[BLANK 2]": 600
  }'
```

- **BLANK 1:** `ApplicationHealthLinux` / `CustomScript` / `AzureMonitorLinuxAgent` /
  `DependencyAgentLinux`
- **BLANK 2:** `gracePeriod` / `timeout` / `startupDelay` / `warmupSeconds`

### Answer: `ApplicationHealthLinux`, `gracePeriod`

**In `challenge-26.md`:** lines **332** and **341**.

`gracePeriod: 600` gives a new instance ten minutes to become healthy before it counts against you.
Set it below real startup time and healthy instances get killed while booting.

---

## Q41

```yaml
  - stage: Rollback
    dependsOn: SwapToProduction
    condition: [BLANK 1]
    jobs:
      - deployment: RollbackSwap
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: [BLANK 2]
                  inputs:
                    action: '[BLANK 3]'
                    sourceSlot: 'staging'
```

- **BLANK 1:** `failed()` / `always()` / `succeeded()` / `succeededOrFailed()`
- **BLANK 2:** `AzureAppServiceManage@0` / `AzureWebApp@1` / `AzureCLI@2` /
  `AzureRmWebAppDeployment@4`
- **BLANK 3:** `Swap Slots` / `Restart Azure App Service` / `Stop Azure App Service` /
  `Start Swap With Preview`

### Answer: `failed()`, `AzureAppServiceManage@0`, `Swap Slots`

**In `challenge-26.md`:** lines **606**, **614**, **618**.

`always()` would roll back successful releases. `AzureWebApp@1` deploys rather than swaps.

**`Start Swap With Preview` is worth knowing** — it is a real multi-phase swap that applies production
settings to the staging slot so you can validate before completing. It is not what a rollback needs.

---

# Section G — Case study

## Case study: Contoso customer web application

### Background

Contoso's customer-facing web app runs on Azure App Service, **Premium V3 with 4 instances**, serving
**2 million page views per day**. Backend services run on a VMSS.

The last deployment updated all instances at once, causing a **3-minute outage** and **4,200 failed
requests**.

### Requirements

**Availability**

- No deployment may reduce capacity below 75%
- Users must never hit a cold instance
- The site must stay available throughout every release

**Validation**

- Staging must be smoke-tested before production
- A small share of real traffic should reach staging before a full swap
- Production must be validated after the swap, with automatic rollback on failure

**Configuration**

- Staging must use the staging Redis and the staging Application Insights resource
- The deployed version number must move to production with the code

---

## Q42

Which VMSS configuration meets the 75% capacity requirement?

- A. `maxBatchInstancePercent=25`
- B. `maxBatchInstancePercent=100`
- C. `maxUnhealthyInstancePercent=75`
- D. `upgradePolicy.mode=Automatic`

### Answer: A

**In `challenge-26.md`:** line **184**.

25% of 4 instances = 1 instance out at a time, leaving 3 of 4 serving — exactly 75%.

**Why the others fail**

- **B** — all four at once. The original outage
- **C** — an abort threshold, not a capacity guarantee. It says when to stop, not how many to take out
- **D** — `Automatic` mode updates all instances with no batching and no health gating. It is the
  fastest way to repeat the incident

---

## Q43

Which **two** configurations prevent users hitting a cold instance? (Choose two.)

- A. `WEBSITE_SWAP_WARMUP_PING_PATH` with an expected status of 200
- B. A `/health/ready` endpoint that returns 503 until data is loaded
- C. Increasing the instance count to 8
- D. `healthCheckPath` set to `/health`
- E. Enabling auto-swap

### Answer: A, B

**In `challenge-26.md`:** lines **377–378** (A) and **415–437** (B).

The two work as a pair, and neither is sufficient alone:

```csharp
    [HttpGet("ready")]
    public async Task<IActionResult> Ready()
    {
        var cacheStatus = await _cache.GetStringAsync("warmup-check");   // warm the cache connection
        var productCount = await _products.GetCountAsync();               // pre-load hot data
        if (productCount == 0)
            return StatusCode(503, "Data not yet loaded");                // honest "not ready"
        return Ok(new { status = "ready", products = productCount });
    }
```

**The endpoint has to tell the truth.** A readiness path that returns 200 immediately makes the
warm-up setting useless — the swap proceeds against a cold app. This endpoint actually touches the
cache and the repository before answering.

**Why the others fail**

- **C** — more instances means more cold instances
- **D** — ongoing instance health, not swap gating
- **E** — auto-swap warms only the **root path** by default, which may answer while caches are cold.
  It also removes the validation stage (Q26)

---

## Q44

Which configuration sends a small share of real traffic to staging before the swap?

- A. `az webapp traffic-routing set --distribution staging=10`
- B. `az network traffic-manager endpoint update --weight 10`
- C. `az webapp deployment slot swap --percentage 10`
- D. `maxParallel: 10%`

### Answer: A

**In `challenge-26.md`:** lines **62–65**.

App Service traffic routing is **per request** at the front end, so 10% really means 10% and the
change applies immediately.

**Why the others fail**

- **B** — Traffic Manager works at DNS level: probabilistic, TTL-delayed, and it routes between
  endpoints rather than between slots of one app
- **C** — a swap has no percentage
- **D** — a rolling deployment batch size, not traffic splitting

**Useful detail:** `x-ms-routing-name=staging` in the URL pins a session to the staging slot, which is
how you test deliberately rather than waiting to be randomly routed.

---

## Q45

Which **two** meet the post-swap validation requirement? (Choose two.)

- A. A retry loop checking `/health` after the swap
- B. A `Rollback` stage with `condition: failed()`
- C. `continue-on-error: true` on the validation step
- D. Auto-swap
- E. A Traffic Manager health probe

### Answer: A, B

**In `challenge-26.md`:** lines **591–601** (A) and **603–621** (B).

```yaml
                      for i in {1..5}; do
                        STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$PROD_URL")
                        if [ "$STATUS" == "200" ]; then exit 0; fi
                        sleep 10
                      done
                      echo "##vso[task.logissue type=error]Production validation failed"
                      exit 1
```

The retry loop **detects**; the rollback stage **reacts**. You need both.

**Why the others fail**

- **C** — turns the failure into a pass, so the rollback stage never triggers
- **D** — swaps without validating
- **E** — removes an unhealthy endpoint from Traffic Manager rotation. That is failover, not rollback,
  and the bad build stays deployed

---

## Q46

Which **two** configurations meet the staging Redis and Application Insights requirement? (Choose
two.)

- A. `CACHE_CONNECTION` as a slot setting on staging
- B. `APPINSIGHTS_INSTRUMENTATIONKEY` as a slot setting on staging
- C. Both values as normal app settings
- D. Both values stored in Key Vault without slot settings
- E. Identical values in both slots

### Answer: A, B

**In `challenge-26.md`:** lines **129–136**.

**What happens without stickiness — spell it out:** after the first swap, the production app points at
the staging Redis and reports telemetry into the staging App Insights resource. Production sessions
break, and your production dashboards go quiet while staging's fill with real user data.

**Why the others fail**

- **C** — normal settings swap with the code. That is the bug
- **D** — Key Vault stores the value; it does not decide which slot reads which secret. The reference
  still swaps
- **E** — identical values would point staging at production Redis, which is worse

---

## Q47

The team enables auto-swap to speed up releases. Two weeks later a bad build reaches production
without smoke tests.

What should they change?

- A. Disable auto-swap and use an explicit swap stage behind a production environment
- B. Add more warm-up paths to auto-swap
- C. Increase the warm-up timeout
- D. Enable automatic instance repair

### Answer: A

**In `challenge-26.md`:** disable with lines **105–109**; the explicit stage is lines **565–582**.

```bash
az webapp deployment slot auto-swap --slot staging --auto-swap-slot ""
```

**Auto-swap fires on deployment, inside App Service.** The pipeline is not consulted, so there is no
place to insert smoke tests or an approval. Warm-up proves the app *starts*; it proves nothing about
whether it is *correct*.

**Why the others fail**

- **B** and **C** — both make warm-up more thorough. A build can start perfectly and still be wrong
- **D** — repairs unhealthy instances. A bad build that runs happily is not unhealthy

**The trade-off, stated plainly:** auto-swap trades control for speed. Right for dev and test, wrong
for production once you have a validation stage worth running.

---

## Q48

After adopting rolling deployments, an upgrade halts after the first batch. Instances in that batch
show `provisioningState` as succeeded, but the application returns 500 on `/health`.

What is happening, and what should Contoso do?

- A. The health extension is misconfigured; remove it and retry
- B. The rolling upgrade correctly stopped on an unhealthy batch; fix the application, then restart
  the upgrade
- C. `pauseTimeBetweenBatches` is too short; increase it
- D. `gracePeriod` expired; increase it to 30 minutes

### Answer: B

**In `challenge-26.md`:** Break & fix Exercise 2, lines **666–698**.

**Read the two signals separately.** `provisioningState: succeeded` means Azure finished *deploying*
the instance. `/health` returning 500 means the **application** is broken. The platform sees both, and
the unhealthy-upgraded threshold stopped the rollout at 25% instead of breaking all four instances.

**This is a success, not a failure.** Three of four instances still serve the old, working version.

```bash
# 1. Fix the application defect
# 2. Then:
az vmss rolling-upgrade start --name $VMSS_NAME --resource-group $RESOURCE_GROUP
```

**Why the others fail**

- **A** — removing the health extension removes the protection. The upgrade would then complete and
  break the entire scale set
- **C** — pacing is irrelevant. The upgrade stopped on health, not on timing
- **D** — a grace period concerns instances still starting. These instances started and are actively
  returning errors

**The habit:** when a safety mechanism stops something, ask whether it was **right** before you loosen
it. Most "fix the pipeline" options on the exam are really "disable the safety net".

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **`maxBatchInstancePercent` vs unhealthy thresholds** | Q1, Q18, Q32 | Batch size = how many at once. Unhealthy % = when to abort |
| **`maxBatchInstancePercent=100`** | Q25, Q42 | One batch of everything. The original outage |
| **Auto-swap on Free or Basic tier** | Q3, Q27 | Standard or higher, same as slots |
| **Auto-swap removes validation** | Q19, Q26, Q47 | It fires in App Service. The pipeline never sees it |
| **Three warm-up-ish settings confused** | Q5, Q8, Q14, Q33 | healthCheckPath = ongoing. SWAP_WARMUP = during a swap. WARMUP_PATH = on start |
| **Readiness endpoint that always returns 200** | Q43 | It must actually check dependencies, or warm-up is theatre |
| **App Insights key not sticky** | Q20, Q30, Q46 | Staging telemetry pollutes production dashboards and alerts |
| **`always()` on a rollback stage** | Q11, Q41 | Use `failed()`, or you undo good releases |
| **Traffic Manager offered for slot traffic** | Q5, Q23, Q44 | `az webapp traffic-routing` is per request; Traffic Manager is DNS |
| **Disabling the safety net as the fix** | Q48 | Ask whether the mechanism was right before loosening it |
| **`AzureWebApp@1` offered for swapping** | Q10 | That deploys. `AzureAppServiceManage@0` swaps |
| **`continue-on-error` on validation** | Q45 | Turns a real failure into a green run |

---

# The blocks to memorise

Line numbers are in `challenge-26.md`.

```bash
# 1. VMSS rolling upgrade policy  (lines 180-187)
az vmss update \
  --set upgradePolicy.mode=Rolling \
  --set upgradePolicy.rollingUpgradePolicy.maxBatchInstancePercent=25 \
  --set upgradePolicy.rollingUpgradePolicy.maxUnhealthyInstancePercent=25 \
  --set upgradePolicy.rollingUpgradePolicy.maxUnhealthyUpgradedInstancePercent=25 \
  --set upgradePolicy.rollingUpgradePolicy.pauseTimeBetweenBatches="PT30S"

# 2. Slot traffic routing - per request, immediate  (lines 62-65)
az webapp traffic-routing set --distribution staging=10

# 3. Auto-swap on and off  (lines 81-85, 105-109)
az webapp deployment slot auto-swap --slot staging --auto-swap-slot production
az webapp deployment slot auto-swap --slot staging --auto-swap-slot ""

# 4. Sticky vs travelling settings  (lines 120-144)
--slot-settings "ENVIRONMENT=production" "CACHE_CONNECTION=..." "APPINSIGHTS_..."   # stays
--settings      "API_VERSION=v2.3.1"                                                # travels

# 5. Swap warm-up  (lines 377-378)
"WEBSITE_SWAP_WARMUP_PING_PATH=/health/ready"
"WEBSITE_SWAP_WARMUP_PING_STATUSES=200"

# 6. VMSS health and repair  (lines 329-349)
az vmss extension set --name ApplicationHealthLinux \
  --settings '{"requestPath":"/health","intervalInSeconds":5,"numberOfProbes":3,"gracePeriod":600}'
az vmss update --set automaticRepairsPolicy.enabled=true \
               --set automaticRepairsPolicy.gracePeriod="PT30M"
```

```yaml
# 7. Rolling strategy with all four hooks  (lines 237-316)
      - deployment: RollingDeploy
        environment: 'production'
        strategy:
          rolling:
            maxParallel: 25%
            preDeploy:         # drain
            deploy:            # update
            routeTraffic:      # return
            postRouteTraffic:  # verify
            on:
              failure:         # roll back this instance

# 8. Swap and rollback  (lines 575-621)
                - task: AzureAppServiceManage@0
                  inputs:
                    action: 'Swap Slots'
                    sourceSlot: 'staging'
  - stage: Rollback
    dependsOn: SwapToProduction
    condition: failed()
```

**ISO 8601 durations:** `PT30S` = 30 seconds, `PT30M` = 30 minutes, `PT1H` = 1 hour.

**Three deployment job strategies:** `runOnce`, `rolling`, `canary`.

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 26 is exam-ready. Move to Challenge 27 |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 4 and 6, then retake this |
| Below 30 | Redo the challenge, and write the four rolling hooks from memory before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 26.

:::tip The one thing

**An instance must never be updated while it is serving traffic, and never serve traffic before it is
warm.**

Drain → update → return → verify. Every setting in this challenge exists to enforce one half of that
sentence.

:::
