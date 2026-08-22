---
sidebar_position: 91
title: "Challenge 25: exam questions"
---

# Challenge 25 — AZ-400 exam questions

**48 questions** built only from what Challenge 25 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-25.md`**.

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

:::danger Blue-green versus canary is a confirmed weak spot for you

They both give zero downtime, so keyword-matching fails here. The separator is always in the
question: **instant rollback and low complexity → blue-green. Validate with real user traffic →
canary.** See Q1, Q24–26 and Q42.

:::

---

# Section A — Single answer

---

## Q1

Contoso must deploy their payment API with zero downtime and instant rollback, using the **least**
operational complexity.

Which strategy should they choose?

- A. Canary deployment with Azure Traffic Manager
- B. Blue-green deployment with App Service deployment slots
- C. Rolling deployment across multiple VM instances
- D. Ring-based deployment with progressive exposure

### Answer: B

**In `challenge-25.md`:** the decision table at lines **510–516**, and lines **77–88** for the swap.

| Strategy | Downtime | Rollback speed | Cost | Complexity |
|---|---|---|---|---|
| **Blue-green** | Zero | **Instant (swap back)** | 2x | **Low** |
| Canary | Zero | Fast (route traffic away) | 1.1x | Medium |
| Rolling | Near-zero | Moderate | 1x | Medium |
| Ring-based | Zero | Fast (stop promotion) | 1.2–1.5x | High |

The swap is **atomic and built into App Service** — and the rollback is the identical command
(lines 84–88), because slots exchange content.

**Why the others fail**

- **A** — zero downtime, but rollback is "fast", not instant: DNS TTL means clients keep hitting the
  old answer for a while. It also adds traffic-splitting complexity
- **C** — **near-zero** downtime, not zero, and it leaves you managing a partially deployed state
- **D** — highest complexity of the five. Rings are for progressive confidence, not for simplicity

**Read the qualifier.** All four give roughly zero downtime; only "least operational complexity" and
"instant rollback" separate them.

---

## Q2

After a slot swap, the production app connects to the **staging** database.

What is the cause?

- A. The connection string was not marked as a slot setting
- B. The staging slot was swapped in the wrong direction
- C. Managed identity was not configured on the production slot
- D. The connection string was stored in Key Vault

### Answer: A

**In `challenge-25.md`:** Break & fix Exercise 1, lines **530–565**.

```bash
# Find which settings are sticky
az webapp config appsettings list \
  --query "[?slotSetting==\`true\`].{name:name, slotSetting:slotSetting}"
```

**The mental model:** during a swap, **content moves and slot settings stay.** A normal app setting
travels with the code into production. A **slot setting** is pinned to the slot it lives on.

**The fix (lines 560–564):**

```bash
az webapp config appsettings set \
  --slot staging \
  --slot-settings "ConnectionStrings__DefaultConnection=Server=staging-sql..."
```

Note `--slot-settings`, not `--settings`. One flag is the entire difference.

**Why the others fail**

- **B** — a swap exchanges the two slots. There is no wrong direction
- **C** — managed identity changes *how* you authenticate, not *which* server you point at
- **D** — Key Vault stores the value; it does not decide whether the reference is sticky

---

## Q3

A canary endpoint weighted at 10 receives roughly 50% of requests.

What is the most likely cause?

- A. Traffic Manager does not support weighted routing for App Service
- B. The canary endpoint has a higher priority than production
- C. Client DNS caching, made worse by low request volume
- D. The production endpoint health probe is failing

### Answer: C

**In `challenge-25.md`:** Break & fix Exercise 2, lines **569–604**.

**Traffic Manager weighting is applied at the DNS level, not per request.** A client resolves the
name once, caches the answer for the whole TTL, and sends *every* request there. With a 300-second
TTL and few clients, the observed split can be nowhere near the configured weights.

**The fix (lines 599–603):**

```bash
az network traffic-manager profile update --ttl 30
```

Lower TTL means faster convergence. Line 234 sets `--ttl 30` for exactly this reason.

**Why the others fail**

- **A** — weighted routing works fine with App Service; lines 240–257 configure it
- **B** — priority is a **different routing method**. A weighted profile does not use priority
- **D** — a failing production probe would send **all** traffic to canary, not half

**The exam point:** if a question mentions per-request accuracy or immediate switching, Traffic
Manager is the wrong tool. Use Front Door or slot traffic percentages instead.

---

## Q4

Which App Service tier is the minimum for deployment slots?

- A. Free (F1)
- B. Shared (D1)
- C. Standard (S1)
- D. Premium v3 (P1V3)

### Answer: C

**In `challenge-25.md`:** line **45** — *"Standard tier or higher required for slots"*. The challenge
uses P1V3 (line 49) for production-grade performance, not because slots demand it.

**Why the others fail**

- **A** and **B** — no slot support at all. **This is the practical trap you already hit**: your
  `contoso-api` bicep defaults to B1 rather than F1 for this reason
- **D** — works, but it is not the *minimum*. Read the qualifier

**The consequence to remember:** blue-green via slots is **impossible on the free tier**. If a
question constrains you to F1 or B1, slot swap is not an available answer.

---

## Q5

Which setting makes App Service wait for a warm-up path to return a specific status before completing
a swap?

- A. `WEBSITE_SWAP_WARMUP_PING_PATH`
- B. `WEBSITE_WARMUP_TIMEOUT`
- C. `WEBSITE_HEALTHCHECK_PATH`
- D. `WEBSITE_SLOT_READY_PATH`

### Answer: A

**In `challenge-25.md`:** Break & fix Exercise 3, lines **620–627**.

```bash
az webapp config appsettings set \
  --slot staging \
  --settings WEBSITE_SWAP_WARMUP_PING_PATH="/health" \
             WEBSITE_SWAP_WARMUP_PING_STATUSES="200"
```

The pair works together: ping this path, and accept only these statuses.

**Why this exists (line 616):** the symptom is that the first requests after a swap take 30+ seconds
and time out, because JIT compilation and cache loading had not finished. Warm-up moves that cost
*before* the swap, where no user is waiting.

**Why the others fail**

- **B** and **D** — do not exist
- **C** — plausible and wrong. `WEBSITE_HEALTHCHECK_PATH` drives App Service's **ongoing instance
  health** monitoring, which removes unhealthy instances from rotation. It is not consulted during a
  swap

---

## Q6

How do you roll back a completed slot swap?

- A. Redeploy the previous artifact to production
- B. Run the same swap command again
- C. Restore the App Service from a backup
- D. Delete the staging slot

### Answer: B

**In `challenge-25.md`:** lines **83–88**.

```bash
# Rollback: the identical command reverses it, since slots exchange content
az webapp deployment slot swap \
  --name $APP_NAME --resource-group $RESOURCE_GROUP \
  --slot staging --target-slot production
```

**Why this is the whole argument for blue-green.** A swap does not copy — it **exchanges**. After
swapping, the old production build is sitting in the staging slot, warm and intact. Swapping again
puts it straight back, in seconds.

That is also why the workflow's rollback step (lines 208–216) is the same three lines as the deploy
step.

**Why the others fail**

- **A** — a redeploy takes minutes and needs a build. Rollback should take seconds
- **C** — backups restore content and configuration, slowly, and are for disaster recovery
- **D** — deleting the staging slot destroys the very copy you need

---

## Q7

Which Traffic Manager routing method splits traffic by percentage?

- A. Priority
- B. Weighted
- C. Performance
- D. Geographic

### Answer: B

**In `challenge-25.md`:** line **232**.

```bash
az network traffic-manager profile create \
  --routing-method Weighted \
  --ttl 30 --protocol HTTPS --port 443 --path "/health"
```

**Why the others fail — know all four, they are all testable**

- **A** — **Priority**: all traffic to the highest-priority healthy endpoint. This is failover, not
  splitting
- **C** — **Performance**: routes each client to the lowest-latency region
- **D** — **Geographic**: routes by the client's location, used for data residency

There is also **Subnet** (by client IP range) and **MultiValue** (returns several endpoints).

---

## Q8

In the canary workflow, which condition ensures the traffic-adjustment job runs only when the user is
*not* promoting?

- A. `if: inputs.promote == 'false'`
- B. `if: ${{ !inputs.promote }}`
- C. `if: ${{ inputs.promote != true }}`
- D. `if: needs.promote.result == 'skipped'`

### Answer: B

**In `challenge-25.md`:** lines **287** and **317**.

```yaml
  adjust-traffic:
    if: ${{ !inputs.promote }}      # line 287
  promote-canary:
    if: ${{ inputs.promote }}       # line 317
```

The input is declared `type: boolean` (line 277), so it arrives as a **real boolean** and negates
with `!`.

**Why the others fail**

- **A** — comparing a boolean to the string `'false'` never matches. This is the same class of bug as
  Challenge 20's `eq(parameters.runTests, 'true')`
- **C** — technically evaluates, but the idiomatic and expected form is `!`
- **D** — the two jobs are independent. Neither declares `needs` on the other

**The pattern:** two mutually exclusive jobs gated on one boolean input. Exactly one runs.

---

## Q9

According to the ring definitions, what is the advancement criterion from Ring 1 to Ring 2?

- A. No P1 or P2 alerts for 2 hours
- B. Error rate below 0.1%
- C. P95 latency under 200 ms
- D. All rings validated

### Answer: B

**In `challenge-25.md`:** lines **351–356**.

| Ring | Audience | Traffic | Duration | Criteria to advance |
|---|---|---|---|---|
| Ring 0 | Internal team | 0% external | 2 hours | No P1/P2 alerts |
| **Ring 1** | Beta users | **5%** | 24 hours | **Error rate below 0.1%** |
| Ring 2 | Early adopters | 25% | 48 hours | P95 latency under 200 ms |
| Ring 3 | General availability | 100% | Permanent | All rings validated |

Learn the **shape**, not just the numbers: each ring adds exposure *and* raises the bar. Ring 0 asks
"does it work at all?", Ring 1 "is it correct at small scale?", Ring 2 "is it fast under real load?".

---

## Q10

At Ring 1 (5% traffic), the error rate rises to 2%. What should happen?

- A. Promote to Ring 2 to gather more data
- B. Roll back Ring 1 and block promotion while investigating
- C. Increase Ring 1 to 15% to see whether the error is transient
- D. Skip to Ring 3, since 2% is acceptable for beta users

### Answer: B

**In `challenge-25.md`:** the criterion at line **354** is *error rate below 0.1%*. Observed is 2% —
**twenty times** the threshold.

**The whole purpose of rings is to catch problems at low exposure.** A failure at 5% is a success of
the process, not a reason to widen the blast radius.

**Why the others fail**

- **A** and **C** — both respond to a failed gate by exposing more users. If 5% shows a 2% error rate,
  more traffic means more failures, not better information
- **D** — nobody agreed that beta users get a broken service. Rings are progressive confidence, not
  progressive tolerance

**The exam pattern:** whenever an option "gathers more data" by increasing exposure after a gate has
failed, it is wrong.

---

## Q11

Which condition makes the workflow roll back automatically when the production health check fails?

- A. `if: always()`
- B. `if: failure()`
- C. `continue-on-error: true`
- D. `if: cancelled()`

### Answer: B

**In `challenge-25.md`:** lines **208–216**.

```yaml
      - name: Rollback on failure
        if: failure()
        run: |
          az webapp deployment slot swap ... --slot staging --target-slot production
          echo "Rolled back to previous production version"
```

`failure()` runs the step only when an earlier step in the job has failed — which is precisely when a
rollback is wanted.

**Why the others fail**

- **A** — `always()` would roll back **even after a successful deployment**, undoing every good
  release
- **C** — `continue-on-error: true` marks the failing health check as a pass. The bad build stays in
  production and the run goes green
- **D** — only on cancellation

**Note the platform difference:** GitHub Actions uses `failure()`; Azure Pipelines uses `failed()`.

---

## Q12

The post-deployment monitor loops for 300 seconds checking every 30 seconds. What does it do on
failure?

- A. Sends an alert to the action group
- B. Swaps the slots back and exits non-zero
- C. Retries indefinitely until healthy
- D. Scales out the App Service

### Answer: B

**In `challenge-25.md`:** lines **487–498**.

```bash
while [ $ELAPSED -lt $MONITOR_DURATION ]; do
  HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$PROD_URL")
  if [ "$HTTP_STATUS" != "200" ]; then
    echo "Health check failed at ${ELAPSED}s. Rolling back..."
    az webapp deployment slot swap --slot staging --target-slot production
    exit 1
  fi
  sleep $CHECK_INTERVAL
  ELAPSED=$((ELAPSED + CHECK_INTERVAL))
done
```

**Why it exists at all:** the swap-time health check (lines 193–206) proves the app answered *once*.
Problems like memory leaks, connection-pool exhaustion or a slow dependency only appear minutes
later. This is the **soak** window.

**Note it exits 1 after rolling back** — the run is marked failed, so nobody mistakes an
auto-recovered deployment for a successful one.

---

## Q13

Which alert condition does Contoso use to trigger automated rollback?

- A. `avg Http5xx > 10` over a 5-minute window
- B. `avg CpuPercentage > 80` over a 15-minute window
- C. `count Requests < 100` over a 1-minute window
- D. `avg ResponseTime > 2000` over a 10-minute window

### Answer: A

**In `challenge-25.md`:** lines **455–463**.

```bash
az monitor metrics alert create \
  --name alert-deployment-error-rate \
  --condition "avg Http5xx > 10" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action ag-deployment-rollback
```

Note the two different timings: **window-size 5m** is how much data each evaluation looks at;
**evaluation-frequency 1m** is how often it evaluates. A short frequency with a longer window gives
fast detection without reacting to a single blip.

The action group (lines 448–452) posts to a rollback webhook.

**Why the others fail**

- **B** — CPU can spike for many reasons unrelated to a bad deploy
- **C** — low request count is more likely a traffic pattern than a failure
- **D** — response time matters, but 5xx errors are the clearer failure signal

---

## Q14

The canary promotion job sets the canary endpoint to weight 100 and production to weight 0.

What has effectively happened?

- A. The canary has been deleted
- B. The canary is now serving all traffic and has become production
- C. Traffic Manager has switched to priority routing
- D. Both endpoints are disabled

### Answer: B

**In `challenge-25.md`:** lines **326–340**.

```bash
az network traffic-manager endpoint update --name ep-canary     --weight 100
az network traffic-manager endpoint update --name ep-production --weight 0
```

**Weight 0 means the endpoint is never returned**, but it is still configured and still probed. That
matters: it stays available as a rollback target — flip the weights back and the old version resumes.

**Why the others fail**

- **A** — nothing is deleted; only weights changed
- **C** — the routing method is a profile-level setting and remains Weighted
- **D** — disabling is `--endpoint-status disabled` (line 247), a different operation

---

## Q15

Which strategy has the lowest infrastructure cost multiplier in the decision table?

- A. Blue-green
- B. Canary
- C. Rolling
- D. Ring-based

### Answer: C

**In `challenge-25.md`:** lines **510–516**.

| Strategy | Infrastructure cost |
|---|---|
| Blue-green | **2x** (duplicate environment) |
| Canary | 1.1x (small canary) |
| **Rolling** | **1x** (in-place) |
| Ring-based | 1.2x–1.5x |
| Feature flags | 1x |

Rolling updates instances **in place**, so no extra capacity is needed. Feature flags are also 1x —
the code ships either way, only the toggle changes.

**The trade-off to state:** rolling is cheapest and its rollback is only "moderate", because during
the roll you are running two versions at once and must roll forward or back instance by instance.

---

## Q16

A slot setting differs from a normal app setting because it:

- A. Is encrypted at rest
- B. Stays with the slot during a swap
- C. Is only readable by managed identity
- D. Applies to all slots simultaneously

### Answer: B

**In `challenge-25.md`:** lines **66–70** and **559–564**.

```bash
az webapp config appsettings set --slot staging \
  --slot-settings ENVIRONMENT=staging ASPNETCORE_ENVIRONMENT=Staging
```

**Content swaps, slot settings stay.** That single sentence answers most slot questions.

**What should be sticky:** connection strings, environment names, environment-specific keys, feature
flag endpoints.
**What should not:** the application build itself, and anything that must travel with the code.

**Why the others fail**

- **A** — both kinds are encrypted at rest. Stickiness is unrelated to encryption
- **C** — an access-control concept, not a swap behaviour
- **D** — the exact opposite of what "slot setting" means

---

# Section B — Multiple answer

---

## Q17

Which **three** characteristics does the decision table give blue-green deployment? (Choose three.)

- A. Zero downtime
- B. Instant rollback
- C. 2x infrastructure cost
- D. 1x infrastructure cost
- E. High complexity
- F. Best for stateless services with many instances

### Answer: A, B, C

**In `challenge-25.md`:** line **512**.

```text
| Blue-green | Zero | Instant (swap back) | 2x (duplicate env) | Low | Mission-critical, strict SLA |
```

**Why the others fail**

- **D** — 1x is rolling and feature flags
- **E** — blue-green is **low** complexity; ring-based is high
- **F** — that describes **rolling** (line 514)

**The cost is the honest trade-off.** You pay for two full environments so that rollback is one
atomic command. For Contoso, where a minute of downtime costs $10,000 (line 16), that is trivially
worth it.

---

## Q18

Which **two** conditions in Contoso's scenario justify blue-green over rolling? (Choose two.)

- A. A 99.95% SLA with $10,000-per-minute downtime cost
- B. The need to reverse a bad release in seconds
- C. A requirement to minimise infrastructure spend
- D. A need to validate with a small subset of real users
- E. Multiple identical stateless instances

### Answer: A, B

**In `challenge-25.md`:** the scenario at line **16**, and line **520**.

> *"Ideal for Contoso's payment service where any failure must be reversed in seconds."*

**Why the others fail**

- **C** — blue-green is the **most** expensive at 2x. If cost were the constraint, rolling wins
- **D** — that is the canary use case (line 521)
- **E** — that is the rolling use case (line 522)

**The method:** each strategy in the table has one sentence at lines 520–524 naming its trigger
condition. Match the scenario's phrasing to that sentence.

---

## Q19

Which **two** settings should be marked as slot settings on the staging slot? (Choose two.)

- A. The database connection string
- B. `ASPNETCORE_ENVIRONMENT`
- C. The application build artifact
- D. The .NET runtime version
- E. The application version number

### Answer: A, B

**In `challenge-25.md`:** lines **66–70** and **559–564**.

```bash
--slot-settings ENVIRONMENT=staging ASPNETCORE_ENVIRONMENT=Staging
--slot-settings "ConnectionStrings__DefaultConnection=Server=staging-sql..."
```

Both identify **where the code is running**, so they must stay behind when the code moves.

**Why the others fail**

- **C** — the artifact **is** the content. Making it sticky would defeat the entire mechanism
- **D** — a plan-level runtime configuration, not a per-slot value in this design
- **E** — the version belongs to the build and **should** travel with it into production

**The test to apply:** *"Should this value follow the code, or stay where it is?"* Follow → normal
setting. Stay → slot setting.

---

## Q20

Which **two** are true about Traffic Manager weighted routing? (Choose two.)

- A. Weighting is applied at DNS resolution, not per request
- B. Client DNS caching delays weight changes taking effect
- C. Each individual HTTP request is routed according to the weights
- D. Weight 0 disables the endpoint permanently
- E. Lower TTL makes weight changes converge faster

### Answer: A, B

**E is also true** (line 603 lowers the TTL for exactly that reason), so treat A and B as the
intended pair — they are the two *properties*, while E is the remedy.

**In `challenge-25.md`:** lines **594** and **599–603**.

> *"Traffic Manager weighted routing is probabilistic at the DNS level, not per-request."*

**Why the others fail**

- **C** — the direct contradiction of A, and the source of the 50% surprise
- **D** — weight 0 stops the endpoint being returned but leaves it configured and probed (Q14).
  Disabling is `--endpoint-status disabled`

**The consequence for the exam:** if a question needs an accurate percentage split or an instant
switch, Traffic Manager is wrong. Azure Front Door or App Service slot traffic percentages are the
per-request answers.

---

## Q21

Which **two** are required for the blue-green workflow to gate production behind an approval? (Choose
two.)

- A. `environment: production` on the swap job
- B. A required reviewer configured on the `production` environment
- C. `needs: deploy-staging` on the swap job
- D. A branch protection rule on `main`
- E. `if: failure()` on the rollback step

### Answer: A, B

**In `challenge-25.md`:** line **178**.

```yaml
  swap-to-production:
    needs: deploy-staging
    environment: production      # the gate attaches HERE
```

Same rule as Challenge 24: **protection rules only apply to jobs that declare the environment**, and
the reviewer is configured on the environment, not in YAML.

**Why the others fail**

- **C** — `needs` creates **ordering**, not approval. Necessary but not sufficient, and not what the
  question asks
- **D** — branch protection gates merges, not deployments
- **E** — rollback behaviour, unrelated to approval

---

## Q22

Which **two** signals does Contoso use to detect a bad production deployment? (Choose two.)

- A. A `/health` endpoint returning non-200
- B. An Azure Monitor metric alert on `Http5xx`
- C. Traffic Manager endpoint weight changes
- D. App Service Plan CPU percentage
- E. A drop in deployment frequency

### Answer: A, B

**In `challenge-25.md`:** lines **193–206** and **488–490** (A), lines **455–463** (B).

Two layers, deliberately:

| Signal | Speed | Catches |
|---|---|---|
| `/health` probe in the workflow | Seconds | The app failing to start or respond |
| `Http5xx` metric alert | 1–5 minutes | Errors under real traffic that a synthetic probe misses |

**Why the others fail**

- **C** — weights are a control you set, not a signal you observe
- **D** — a symptom of many things; too noisy to trigger a rollback
- **E** — a DORA metric measured over weeks, not a deployment signal

---

## Q23

Which **two** strategies decouple *deploying* code from *releasing* a feature? (Choose two.)

- A. Feature flags
- B. Ring-based deployment
- C. Blue-green deployment
- D. Rolling deployment
- E. Canary deployment

### Answer: A, B

**In `challenge-25.md`:** lines **516** and **524** (A), lines **351–356** (B).

> *"Feature flags: When you want to deploy code without activating features."*

Rings qualify in a weaker sense — the code is deployed everywhere, but only some audiences reach it.
Feature flags are the pure case, and Challenge 27 is entirely about them.

**Why the others fail**

- **C**, **D**, **E** — all three release the moment they deploy. Traffic reaches the new code as
  soon as it is live; there is no separate activation step

**Why the distinction matters:** with flags, rollback is a toggle with **no deployment at all** —
"Instant (toggle flag)" at line 516. That is the fastest rollback of any strategy in the table.

---

# Section C — Repeated scenario

**Scenario:** Contoso's payment API must deploy with zero downtime. If the new version misbehaves,
the team must be able to return to the previous version **within seconds**. Duplicate infrastructure
cost is acceptable.

---

## Q24

**Proposed solution:** Deploy to a staging slot, validate `/health`, then swap staging to production.
Roll back by running the swap command again.

Does this meet the goal? **Yes**

### Answer: Yes

**In `challenge-25.md`:** lines **155–216**.

Every requirement is met:

- **Zero downtime** — the swap is atomic; warmed instances are exchanged
- **Rollback in seconds** — the same command reverses it (lines 83–88), because the old build is
  still sitting in staging
- **Cost accepted** — 2x is exactly what the scenario permits

This is blue-green as designed, and it is the intended answer.

---

## Q25

**Proposed solution:** Deploy the new version to a canary App Service and use Traffic Manager
weighted routing at 10%, increasing gradually. Roll back by setting the canary weight to 0.

Does this meet the goal? **No**

### Answer: No

Zero downtime: yes. **Rollback within seconds: no.**

Setting the weight to 0 changes what Traffic Manager *answers*, but clients that already resolved the
name keep using the cached IP until their TTL expires. With `--ttl 30` (line 234) that is up to 30
seconds; at the default it is far longer. Break & fix Exercise 2 (line 594) documents this exact
behaviour.

**Canary is not wrong — it is wrong for *this* requirement.** Choose canary when the goal is
validating with real traffic. Choose blue-green when the goal is instant reversal.

**Note this is the pair you keep confusing.** Both give zero downtime; only the rollback clause
separates them.

---

## Q26

**Proposed solution:** Deploy to a staging slot on a **Basic (B1)** App Service Plan, validate
`/health`, then swap staging to production.

Does this meet the goal? **No**

### Answer: No

The strategy is right; the **tier makes it impossible**.

Line 45 states plainly: *"Standard tier or higher required for slots."* On B1 the
`az webapp deployment slot create` command fails, so there is no staging slot to deploy to and no
swap to perform.

**Why this question exists:** you already met this constraint in your own `contoso-api` bicep, where
the plan defaults to B1 and the staging slot is guarded because F1 has no slots. The exam phrases it
as *"the team uses the Basic tier to control cost"* — a cost detail buried in the background that
silently eliminates the obvious answer.

**Read the environment details.** The tier, the region and the plan are not scenery.

---

# Section D — Yes/No statement grid

---

## Q27 — deployment slots

| # | Statement | Answer |
|---|---|---|
| 1 | Slots are available on the Free tier | **No** |
| 2 | A swap exchanges content between two slots | **Yes** |
| 3 | Slot settings travel with the code during a swap | **No** |
| 4 | Running the swap command again rolls back | **Yes** |

**In `challenge-25.md`:** line **45** (row 1), lines **83–88** (rows 2 and 4), lines **547** and
**559–564** (row 3).

Row 3 is the inversion trap. **Content swaps; slot settings stay.** Getting this backwards is exactly
the Break & fix Exercise 1 failure, where production ended up pointing at the staging database.

Row 2 explains row 4: because it is an *exchange* rather than a copy, the previous build survives in
the other slot, warm and ready.

---

## Q28 — Traffic Manager

| # | Statement | Answer |
|---|---|---|
| 1 | Weighted routing distributes each HTTP request by weight | **No** |
| 2 | DNS TTL affects how quickly weight changes take effect | **Yes** |
| 3 | Weight 0 removes an endpoint from DNS responses | **Yes** |
| 4 | Traffic Manager provides instant traffic switching | **No** |

**In `challenge-25.md`:** line **594** (rows 1 and 4), lines **599–603** (row 2), lines **333–338**
(row 3).

Rows 1 and 4 are the same fact from two angles, and together they are why Traffic Manager is the
wrong answer to "instant rollback".

**What *is* instant:** App Service slot traffic percentage
(`az webapp traffic-set --distribution staging=10`) and Azure Front Door, because both route at the
**proxy** rather than in DNS. Neither is in this challenge, and both are on the exam.

---

## Q29 — ring-based deployment

| # | Statement | Answer |
|---|---|---|
| 1 | Ring 0 receives 0% external traffic | **Yes** |
| 2 | Failing a ring's criteria should block promotion | **Yes** |
| 3 | Ring-based has the lowest complexity of the five strategies | **No** |
| 4 | Each ring has a minimum soak duration | **Yes** |

**In `challenge-25.md`:** lines **351–356** (rows 1, 2, 4), line **515** (row 3).

Row 4 is easy to overlook: 2 hours, 24 hours, 48 hours. **Time in a ring is itself a gate** — some
failures (memory leaks, connection exhaustion, a nightly batch job) only surface after hours. A
metric that looks fine after ten minutes proves very little.

Row 3: ring-based is rated **High** complexity, the highest in the table.

---

## Q30 — automated rollback

| # | Statement | Answer |
|---|---|---|
| 1 | `if: failure()` runs the rollback only when a prior step failed | **Yes** |
| 2 | `if: always()` is the correct condition for a rollback step | **No** |
| 3 | Post-deployment monitoring catches failures a single health check misses | **Yes** |
| 4 | The metric alert evaluates every 5 minutes | **No** |

**In `challenge-25.md`:** line **209** (rows 1 and 2), lines **478–503** (row 3), lines **460–461**
(row 4).

Row 2 is worth pausing on: `always()` would roll back after **successful** deployments too, silently
reverting every good release.

Row 4 mixes up the two timings. `--window-size 5m` is how much data each evaluation examines;
`--evaluation-frequency 1m` is how often it runs. It evaluates **every minute** over a five-minute
window.

---

# Section E — Drag and drop

---

## Q31

Match each strategy to its rollback speed and infrastructure cost.

| Strategy | Rollback speed | Cost |
|---|---|---|
| Blue-green | **Instant (swap back)** | **2x** |
| Canary | **Fast (route traffic away)** | **1.1x** |
| Rolling | **Moderate (roll forward/back)** | **1x** |
| Ring-based | **Fast (stop promotion)** | **1.2–1.5x** |
| Feature flags | **Instant (toggle flag)** | **1x** |

**In `challenge-25.md`:** lines **510–516**.

**Two rows share "Instant" and they are not the same thing.** Blue-green swaps *infrastructure*;
feature flags toggle *behaviour* with no deployment at all. If a question forbids any deployment
during rollback, only feature flags qualify.

**Cheapest with instant rollback: feature flags at 1x.** That single row is why Challenge 27 exists.

---

## Q32

Arrange the blue-green workflow jobs in execution order.

**Items:** `swap-to-production` · `build` · `post-deployment-monitor` · `deploy-staging`

### Answer

1. `build` — line **116**
2. `deploy-staging` — line **139**, `environment: staging`
3. `swap-to-production` — line **175**, `environment: production` ← **the approval gate**
4. `post-deployment-monitor` — line **469**, `needs: swap-to-production`

```yaml
  build:
  deploy-staging:
    needs: build
    environment: staging
  swap-to-production:
    needs: deploy-staging
    environment: production
  post-deployment-monitor:
    needs: swap-to-production
```

**The design point:** every cheap automatic check happens **before** the human gate, so a reviewer is
only interrupted once staging is proven healthy. And monitoring continues **after** the swap, because
the swap succeeding is not the same as the release being good.

---

## Q33

Arrange the ring promotion sequence with its traffic percentages.

**Items:** 25% early adopters · 100% GA · 5% beta · 0% internal

### Answer

| Order | Ring | Traffic | Soak | Gate |
|---|---|---|---|---|
| 1 | Ring 0 — internal | **0% external** | 2 hours | No P1/P2 alerts |
| 2 | Ring 1 — beta | **5%** | 24 hours | Error rate < 0.1% |
| 3 | Ring 2 — early adopters | **25%** | 48 hours | P95 latency < 200 ms |
| 4 | Ring 3 — GA | **100%** | Permanent | All rings validated |

**In `challenge-25.md`:** lines **351–356**, implemented at lines **370–433**.

Notice the script's Ring 3 (lines 412–430) does **two** things: swaps the slot to production *and*
sets weights to 100/0. Promotion to GA both moves the code and retires the canary path.

---

## Q34

Arrange the steps of a safe slot-swap deployment, in order.

**Items:** Swap staging to production · Validate the staging health endpoint · Deploy the artifact to
the staging slot · Monitor production after the swap · Wait for the slot to warm up

### Answer

1. Deploy the artifact to the staging slot — line **155**
2. Wait for the slot to warm up — line **162**
3. Validate the staging health endpoint — line **165**
4. Swap staging to production — line **185**
5. Monitor production after the swap — line **478**

**Every step earns its place:**

- Skip **2** and the health check hits a cold app and fails
- Skip **3** and you swap a broken build into production
- Skip **5** and you miss the failures that only appear minutes later

Better than `sleep 30` at line 163: configure `WEBSITE_SWAP_WARMUP_PING_PATH` (line 626) so App
Service warms the slot *during* the swap rather than guessing at a fixed delay.

---

## Q35

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Production connects to the staging database after a swap | **Connection string not marked as a slot setting** |
| A 10%-weighted canary receives ~50% of requests | **DNS caching plus DNS-level probabilistic routing** |
| First requests after a swap take 30+ seconds | **App initialisation not completed before the swap** |
| Slot creation fails on the plan | **App Service Plan below Standard tier** |

**In `challenge-25.md`:** lines **547**, **594**, **616**, **45**.

**The diagnostic habit:** each symptom points at a **different layer** — configuration, DNS,
application startup, and platform tier. Name the layer first, then the fix.

---

# Section F — Hot area

---

## Q36

```bash
az appservice plan create \
  --name asp-contoso-payments \
  --sku [BLANK 1] \
  --is-linux

az webapp deployment slot [BLANK 2] \
  --name $APP_NAME \
  --slot staging
```

Requirement: the **minimum** tier that supports deployment slots.

- **BLANK 1:** `S1` / `F1` / `B1` / `P1V3`
- **BLANK 2:** `create` / `add` / `new` / `provision`

### Answer: `S1`, `create`

**In `challenge-25.md`:** lines **45** and **60**.

`F1` and `B1` have **no slot support**. `P1V3` works but is not the minimum — and the question asks
for the minimum.

---

## Q37

```bash
az webapp config appsettings set \
  --name $APP_NAME \
  --slot staging \
  --[BLANK 1] "ConnectionStrings__DefaultConnection=Server=staging-sql..."
```

Requirement: the staging connection string must **not** move to production during a swap.

- **BLANK 1:** `slot-settings` / `settings` / `sticky-settings` / `slot-config`

### Answer: `slot-settings`

**In `challenge-25.md`:** lines **560–564**.

`--settings` creates a normal app setting that **swaps with the code** — the exact bug in Break & fix
Exercise 1. One flag is the whole difference between a working blue-green deploy and production
writing to the staging database.

---

## Q38

```bash
az network traffic-manager profile create \
  --name tm-contoso-payments \
  --routing-method [BLANK 1] \
  --ttl [BLANK 2] \
  --path "/health"
```

Requirement: split traffic by percentage, and let weight changes take effect quickly.

- **BLANK 1:** `Weighted` / `Priority` / `Performance` / `Geographic`
- **BLANK 2:** `30` / `300` / `3600` / `86400`

### Answer: `Weighted`, `30`

**In `challenge-25.md`:** lines **232** and **234**.

The TTL is the fix from Break & fix Exercise 2 (line 603). A 300-second TTL is what produced the 50%
surprise; 30 seconds converges ten times faster.

---

## Q39

```yaml
      - name: Rollback on failure
        if: [BLANK 1]
        run: |
          az webapp deployment slot swap \
            --slot staging --target-slot [BLANK 2]
```

- **BLANK 1:** `failure()` / `always()` / `success()` / `cancelled()`
- **BLANK 2:** `production` / `staging` / `canary` / `internal`

### Answer: `failure()`, `production`

**In `challenge-25.md`:** lines **209** and **215**.

`always()` would undo every successful deployment. The swap arguments are **identical** to the
forward deployment because a swap is an exchange (Q6).

---

## Q40

```bash
az monitor metrics alert create \
  --name alert-deployment-error-rate \
  --condition "avg [BLANK 1] > 10" \
  --window-size [BLANK 2] \
  --evaluation-frequency [BLANK 3] \
  --action ag-deployment-rollback
```

- **BLANK 1:** `Http5xx` / `CpuPercentage` / `Requests` / `ResponseTime`
- **BLANK 2:** `5m` / `1m` / `15m` / `1h`
- **BLANK 3:** `1m` / `5m` / `15m` / `1h`

### Answer: `Http5xx`, `5m`, `1m`

**In `challenge-25.md`:** lines **459–461**.

Keep the two timings straight: **window-size** is how much data each evaluation examines;
**evaluation-frequency** is how often it runs. Short frequency, longer window — fast detection
without firing on a single blip.

---

## Q41

```bash
az webapp config appsettings set \
  --slot staging \
  --settings [BLANK 1]="/health" \
             [BLANK 2]="200"
```

Requirement: the swap must not complete until the app responds correctly.

- **BLANK 1:** `WEBSITE_SWAP_WARMUP_PING_PATH` / `WEBSITE_HEALTHCHECK_PATH` /
  `WEBSITE_WARMUP_PATH` / `WEBSITE_READY_PATH`
- **BLANK 2:** `WEBSITE_SWAP_WARMUP_PING_STATUSES` / `WEBSITE_WARMUP_STATUS` /
  `WEBSITE_HEALTHCHECK_STATUS` / `WEBSITE_PING_RESULT`

### Answer: `WEBSITE_SWAP_WARMUP_PING_PATH`, `WEBSITE_SWAP_WARMUP_PING_STATUSES`

**In `challenge-25.md`:** lines **626–627**.

`WEBSITE_HEALTHCHECK_PATH` is real but different — it drives **ongoing** instance health monitoring,
removing unhealthy instances from rotation. It plays no part in a swap.

**Why this beats `sleep 30`:** a fixed sleep is a guess. Warm-up settings make App Service wait for
the actual signal, however long it takes.

---

# Section G — Case study

## Case study: Contoso payments platform

### Background

Contoso's payment API handles 50,000 transactions per minute at peak, with a contractual **99.95%**
SLA. Deployment downtime costs approximately **$10,000 per minute**. The CTO has mandated
"always available" and eliminated the Thursday maintenance window.

### Environment

- Azure App Service, **Premium v3 (P1v3)**, East US 2
- Azure Traffic Manager for DNS-based routing
- Application Insights for monitoring
- Resource group `rg-contoso-payments-prod`

### Requirements

**Core deployment**

- Zero downtime on every release
- A bad release must be reversed **within seconds**
- Production configuration must never leak into staging or vice versa

**Progressive rollout**

- A new fraud-detection algorithm must be validated on a small share of real transactions before full
  rollout
- Internal staff must see it before any customer does

**Safety**

- The deployment must roll back automatically if 5xx errors spike
- Problems appearing minutes after a release must also trigger rollback

---

## Q42

Which strategy meets the **core deployment** requirements?

- A. Blue-green with deployment slots
- B. Canary with Traffic Manager
- C. Rolling deployment
- D. Recreate with a maintenance window

### Answer: A

**In `challenge-25.md`:** lines **512** and **520**.

The deciding clause is **"reversed within seconds"**. Only blue-green offers instant rollback via an
atomic swap, and P1v3 supports slots comfortably.

**Why the others fail**

- **B** — "fast", not instant. DNS caching delays it (Break & fix Exercise 2)
- **C** — **near-zero** downtime and moderate rollback. At $10,000 per minute, near-zero is not zero
- **D** — the maintenance window the CTO has already abolished

---

## Q43

Which strategy meets the **fraud-detection validation** requirement?

- A. Blue-green with deployment slots
- B. Canary deployment with weighted routing
- C. Recreate deployment
- D. A larger staging environment

### Answer: B

**In `challenge-25.md`:** line **521**.

> *"Canary: when you want to validate with real traffic before full rollout."*

A fraud-detection algorithm can only be judged against **real transactions**. Synthetic staging
traffic will not reveal how it behaves against genuine fraud patterns.

**Why the others fail**

- **A** — blue-green sends **all** traffic at once the instant you swap. There is no partial exposure
- **C** — downtime, and still all-or-nothing
- **D** — a bigger staging environment is still not real traffic

**Q42 and Q43 together are the whole point.** One platform, one release, **two different strategies**
for two different requirements. That is why "which is better, blue-green or canary?" has no answer
without the requirement.

---

## Q44

Which addition satisfies "internal staff must see it before any customer does"?

- A. Add Ring 0 with an internal slot at 0% external traffic
- B. Increase the canary weight to 50%
- C. Deploy to staging and email the internal team a link
- D. Use a feature flag enabled for everyone

### Answer: A

**In `challenge-25.md`:** line **353** and the script at lines **373–381**.

```bash
az webapp deployment slot swap --slot internal --target-slot staging
echo "Internal team can validate at: https://${APP_NAME}-internal.azurewebsites.net"
```

Ring 0 receives **0% external traffic** and soaks for **2 hours**, gated on no P1/P2 alerts.

**Why the others fail**

- **B** — exposes half of all customers, the opposite of "before any customer"
- **C** — closer than it looks, and still wrong: staging is not part of the promotion path, so
  validating there proves nothing about the artifact that will be promoted. Ring 0 is a real ring
  with a gate and a soak time
- **D** — enabled for everyone means customers see it immediately

---

## Q45

Which **two** configurations prevent configuration leaking between slots? (Choose two.)

- A. Mark the connection string as a slot setting
- B. Mark `ASPNETCORE_ENVIRONMENT` as a slot setting
- C. Store both connection strings in one Key Vault
- D. Use identical values in both slots
- E. Disable the staging slot between deployments

### Answer: A, B

**In `challenge-25.md`:** lines **66–70** and **559–564**.

Both values describe **where the code is running**, so both must stay with the slot when the content
moves.

**Why the others fail**

- **C** — Key Vault stores values safely, but the app still needs to know *which* secret to read.
  Without stickiness, the production app reads the staging reference
- **D** — identical values would mean staging writes to the production database. That is worse than
  the original bug
- **E** — a disabled slot cannot be deployed to or warmed up

---

## Q46

Which **two** configurations meet the automatic-rollback requirements? (Choose two.)

- A. An Azure Monitor metric alert on `Http5xx` wired to a rollback action group
- B. A post-deployment monitoring job that swaps back on failure
- C. `continue-on-error: true` on the health check step
- D. Traffic Manager health probes on `/health`
- E. `if: always()` on the rollback step

### Answer: A, B

**In `challenge-25.md`:** lines **448–463** (A) and **478–503** (B).

Two layers covering two windows: the alert watches production continuously; the monitoring job covers
the **five minutes immediately after the swap**, when regressions are most likely.

**Why the others fail**

- **C** — turns a failing health check into a pass. The bad build stays live and the run goes green
- **D** — probes remove an unhealthy **endpoint** from Traffic Manager rotation. That is failover, not
  rollback, and it does nothing about a bad slot swap
- **E** — would roll back successful deployments too

---

## Q47

Six months later, Contoso wants to enable the new fraud algorithm for 5% of users **without
deploying**, and disable it instantly if fraud reports rise.

Which strategy meets this?

- A. Canary deployment
- B. Feature flags
- C. Blue-green deployment
- D. Ring-based deployment

### Answer: B

**In `challenge-25.md`:** lines **516** and **524**.

```text
| Feature flags | Zero | Instant (toggle flag) | 1x | Medium | Decoupling deploy from release |
```

The phrase **"without deploying"** eliminates every other option — all of them require a deployment
to change what users get. A flag is a configuration change with no build, no artifact and no swap.

**Why the others fail**

- **A** — canary routes traffic to a **differently deployed** version. There must be something new to
  deploy
- **C** — requires a deployment and switches everyone at once
- **D** — requires a deployment per ring

**This is Challenge 27's entire subject.** Note the cost too: 1x, because one codebase serves both
behaviours.

---

## Q48

After adopting blue-green, a release passes its health check, swaps successfully, and is fine for six
minutes. At minute eight, 5xx errors spike as a connection pool exhausts under peak load.

What should Contoso change?

- A. Extend post-deployment monitoring beyond 300 seconds and keep the metric alert
- B. Increase the warm-up sleep from 30 seconds to 120 seconds
- C. Switch from blue-green to rolling deployment
- D. Remove the health check, since it did not catch the problem

### Answer: A

**In `challenge-25.md`:** the monitor window at line **483** (`MONITOR_DURATION=300`) and the alert at
lines **455–463**.

The monitoring job covered **five** minutes; the failure arrived at **eight**. The metric alert with
its rollback action group (line 462) is what covers the longer tail — and this is precisely why two
layers exist.

**Why the others fail**

- **B** — warm-up addresses **cold start**, a first-request problem. This failure appeared under
  sustained load, minutes in
- **C** — rolling would have exposed the same defect more slowly and rolled back more slowly. The
  strategy is not the problem
- **D** — the health check did its job: it confirmed the app started. No single check catches
  everything, and removing a working signal because it is not omniscient is backwards

**The general lesson:** deployment safety is **layered** — pre-swap validation, post-swap soak, and
continuous alerting. Each covers a window the others cannot, and a question describing a failure in
one window is asking which layer was missing.

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Blue-green vs canary** | Q1, Q25, Q42, Q43 | Instant rollback + low complexity → blue-green. Validate with real traffic → canary |
| **Slot settings inverted** | Q2, Q16, Q19, Q27 | Content swaps, slot settings stay |
| **Traffic Manager assumed per-request** | Q3, Q20, Q28 | DNS-level and probabilistic. TTL delays every change |
| **Slots on Free or Basic tier** | Q4, Q26, Q36 | Standard or higher. F1 and B1 have no slots |
| **`always()` on a rollback step** | Q11, Q30, Q39, Q46 | Use `failure()`, or you undo good releases |
| **`continue-on-error` on a health check** | Q46 | Turns a real failure into a green run |
| **Widening exposure after a failed gate** | Q10 | A failed ring means roll back, never promote |
| **`WEBSITE_HEALTHCHECK_PATH` for warm-up** | Q5, Q41 | That is ongoing instance health. Swap warm-up is `WEBSITE_SWAP_WARMUP_PING_PATH` |
| **window-size vs evaluation-frequency** | Q30, Q40 | Window = data examined. Frequency = how often |
| **Weight 0 read as "disabled"** | Q14, Q20 | Weight 0 stops DNS answers; the endpoint stays configured and probed |
| **One strategy for every requirement** | Q42, Q43 | One release can need blue-green *and* canary |
| **Single-layer rollback safety** | Q48 | Pre-swap check, post-swap soak, continuous alert |

---

# The blocks to memorise

Line numbers are in `challenge-25.md`.

```text
# 1. The decision table  (lines 510-516) - THE most exam-relevant thing here
Strategy      Downtime   Rollback              Cost      Complexity
Blue-green    Zero       Instant (swap back)   2x        Low
Canary        Zero       Fast (route away)     1.1x      Medium
Rolling       Near-zero  Moderate              1x        Medium
Ring-based    Zero       Fast (stop promo)     1.2-1.5x  High
Feature flags Zero       Instant (toggle)      1x        Medium

# 2. The rings  (lines 351-356)
Ring 0  internal      0% external   2h   no P1/P2 alerts
Ring 1  beta          5%           24h   error rate < 0.1%
Ring 2  early adopter 25%          48h   P95 latency < 200ms
Ring 3  GA            100%         perm  all rings validated
```

```bash
# 3. Slot swap and its rollback  (lines 77-88) - the SAME command
az webapp deployment slot swap --name $APP --resource-group $RG \
  --slot staging --target-slot production

# 4. Sticky configuration  (lines 66-70, 560-564)
az webapp config appsettings set --slot staging \
  --slot-settings ASPNETCORE_ENVIRONMENT=Staging          # note: --slot-settings

# 5. Swap warm-up  (lines 626-627)
--settings WEBSITE_SWAP_WARMUP_PING_PATH="/health" \
           WEBSITE_SWAP_WARMUP_PING_STATUSES="200"

# 6. Weighted Traffic Manager  (lines 229-257)
az network traffic-manager profile create --routing-method Weighted --ttl 30 --path "/health"
az network traffic-manager endpoint create --name ep-production --weight 90
az network traffic-manager endpoint create --name ep-canary     --weight 10

# 7. Rollback alert  (lines 455-463)
az monitor metrics alert create --condition "avg Http5xx > 10" \
  --window-size 5m --evaluation-frequency 1m --action ag-deployment-rollback
```

```yaml
# 8. Automatic rollback in the workflow  (lines 208-216)
      - name: Rollback on failure
        if: failure()
        run: az webapp deployment slot swap --slot staging --target-slot production
```

**Minimum tier for slots: Standard (S1).** Free and Basic cannot do blue-green.

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 25 is exam-ready. Move to Challenge 26 |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Re-read Tasks 1 and 6, then retake this |
| Below 30 | Redo the challenge, and write the decision table from memory before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 25.

:::danger If you missed Q1, Q25, Q42 or Q43

That is the blue-green versus canary confusion, and it is a confirmed weak spot for you. Write the
decision table from memory on paper before moving on. **The strategies are not ranked — the
requirement picks one.**

:::
