---
sidebar_position: 3.5
toc_max_heading_level: 2
title: "Challenge 27: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 27 — AZ-400 exam questions

**48 questions** built only from what Challenge 27 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-27.md`**.

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

:::tip The one sentence this challenge exists for

**Feature flags decouple *deploying* from *releasing*.** Code ships dark, the flag turns it on, and
turning it off is instant with no deployment at all. Every question here is a variation on that.

:::

---

# Section A — Single answer

---

## Q1

Which Azure App Configuration tier is required for feature flags?

- A. Free
- B. Standard
- C. Premium
- D. Any tier supports feature flags

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-27.md`:** line **49**.

```bash
az appconfig create \
  --name $CONFIG_STORE \
  --sku Standard      # "Standard tier for feature flags"
```

**Why the others fail**

- **A** — the Free tier exists but is limited: fewer keys, no geo-replication, and no feature
  management
- **C** — Premium is a real newer tier, but Standard is what the challenge specifies and the minimum
  that supports flags
- **D** — the direct contradiction

**The pattern across these challenges:** slots need Standard App Service, feature flags need Standard
App Configuration. **Free tiers do not do production features**, and the exam hides the tier in the
scenario's background details.

</details>

---

## Q2

Which RBAC role lets an application read feature flags from App Configuration?

- A. Contributor
- B. App Configuration Data Reader
- C. Reader
- D. App Configuration Contributor

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-27.md`:** lines **73–76**.

```bash
az role assignment create \
  --assignee $APP_PRINCIPAL_ID \
  --role "App Configuration Data Reader" \
  --scope $CONFIG_ID
```

**The distinction the exam tests:** *management plane* versus *data plane*.

| Role | Grants |
|---|---|
| Reader / Contributor | Manage the **resource** — see it, change its SKU, delete it |
| **App Configuration Data Reader** | Read the **key-values and flags inside it** |
| App Configuration Data Owner | Read and write the data |

**Why the others fail**

- **A** and **C** — control-plane roles. A Contributor can delete the store but **cannot read a single
  flag**. That surprises people, and it is the same split as Key Vault's data-plane roles
- **D** — not a real role name

**Note the assignee is a managed identity** (line 67), so no secret is stored anywhere — the same
secretless pattern as your OIDC block.

</details>

---

## Q3

A feature flag was enabled 10 minutes ago but the application still shows old behaviour.

Which **two** causes are most likely? (Pick the single best answer for this question.)

- A. `CacheExpirationInterval` is too high, or the App Configuration middleware is missing
- B. The application needs restarting after every flag change
- C. The RBAC role assignment has not propagated
- D. Feature flags require a redeployment to take effect

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-27.md`:** Break & fix Exercise 1, lines **663–696**.

```csharp
options.UseFeatureFlags(featureFlagOptions =>
{
    featureFlagOptions.CacheExpirationInterval = TimeSpan.FromSeconds(30);
    featureFlagOptions.Label = "production";
});

app.UseAzureAppConfiguration();   // MUST come before app.MapControllers()
app.MapControllers();
```

Two independent failures with one symptom: the client caches flags for the interval you set, and the
**middleware** is what triggers a refresh check on each request. Without it, the cache never expires
in practice.

**Why the others fail**

- **B** and **D** — both contradict the entire point. If a flag needed a restart or a redeploy, it
  would not be a kill switch
- **C** — an RBAC problem produces a **403 at startup**, not stale values

</details>

---

## Q4

Internal team members are not seeing a feature despite a targeting filter that includes their group at
100%.

What is the cause?

- A. The group name is misspelled in the filter
- B. `IHttpContextAccessor` is not registered, so the targeting context is empty
- C. The default rollout percentage overrides group percentages
- D. Targeting filters require the Premium tier

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-27.md`:** Break & fix Exercise 2, lines **700–715**.

> *"The `IHttpContextAccessor` is not registered in the DI container, so the `HttpContext` is null and
> the targeting context resolver returns 'anonymous' with no groups."*

```csharp
builder.Services.AddHttpContextAccessor();
builder.Services.AddSingleton<ITargetingContextAccessor, HttpContextTargetingContextAccessor>();
```

**The chain to understand:** the targeting filter asks *"who is this user, and which groups are they
in?"* That answer comes from the targeting context accessor, which reads `HttpContext`. No
`IHttpContextAccessor` means no `HttpContext`, so every request looks like an anonymous user in no
group — and anonymous falls through to `DefaultRolloutPercentage`, which is 0 (line 136).

**Why the others fail**

- **A** — plausible in real life; the documented root cause is the missing registration
- **C** — group percentages take precedence over the default. That is what makes groups useful
- **D** — Standard supports targeting filters

</details>

---

## Q5

Feature flags work in staging but not in production, and the portal shows the flag enabled.

What is the cause?

- A. The flag was enabled with a different label than the one the app reads
- B. Production lacks the Data Reader role
- C. The production app uses a connection string instead of managed identity
- D. The flag has a time window filter that has expired

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-27.md`:** Break & fix Exercise 3, lines **720–748**.

> *"The flag was enabled with label `staging` but the production app is configured to read label
> `production`."*

```bash
# Diagnose - list every label for this feature
az appconfig feature list --feature NewCheckoutFlow \
  --query "[].{label:label, state:state}"
```

**A label makes a separate flag.** `NewCheckoutFlow` with label `staging` and `NewCheckoutFlow` with
label `production` are two independent records that happen to share a name. The app only ever sees
the label it was configured for (line 198).

**Why the others fail**

- **B** — a missing role gives a **403**, and the app usually fails to start
- **C** — connection strings work; they are just less secure
- **D** — that would apply to `MaintenanceMode` (line 110), not to a plain boolean flag

**Why labels exist:** one store, many environments. It is the same pattern as environment-scoped
secrets — same name, different scope, different value.

</details>

---

## Q6

Which filter enables a feature only between two timestamps?

- A. `Microsoft.Targeting`
- B. `Microsoft.TimeWindow`
- C. `Microsoft.Percentage`
- D. `Microsoft.Schedule`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-27.md`:** lines **110–115**.

```bash
az appconfig feature filter add \
  --feature MaintenanceMode \
  --filter-name "Microsoft.TimeWindow" \
  --filter-parameters Start="2025-03-01T00:00:00Z" End="2025-03-01T06:00:00Z"
```

**The three built-in filters:**

| Filter | Enables the feature |
|---|---|
| `Microsoft.TimeWindow` | Between a start and end time |
| `Microsoft.Targeting` | For named users, groups, or a percentage |
| `Microsoft.Percentage` | For a random share of evaluations |

**Why the others fail**

- **A** — audience-based, not time-based
- **C** — real, but percentage-based. Note it differs from Targeting: `Microsoft.Percentage` is
  **random per evaluation**, so the same user can flip between on and off. Targeting is **sticky per
  user**, which is what an A/B test needs
- **D** — does not exist

</details>

---

## Q7

In a targeting filter, what does `Audience.DefaultRolloutPercentage=0` mean?

- A. The feature is disabled entirely
- B. Users not in a named group or user list do not get the feature
- C. The feature rolls out to 0% and then increases automatically
- D. The filter is ignored

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-27.md`:** lines **130–142**.

```bash
    Audience.DefaultRolloutPercentage=0        # everyone else: no
    Audience.Groups.0.Name="InternalTeam"
    Audience.Groups.0.RolloutPercentage=100    # internal team: all of them
    Audience.Groups.1.Name="BetaTesters"
    Audience.Groups.1.RolloutPercentage=50     # beta testers: half
    Audience.Users.0="admin@contoso.com"       # named users: always
```

**This one filter expresses the entire ring strategy from Challenge 25:** named users are Ring 0,
InternalTeam at 100% is Ring 0/1, BetaTesters at 50% is Ring 1, and raising
`DefaultRolloutPercentage` walks you to Ring 3.

**Why the others fail**

- **A** — the flag is still enabled; the audience is just narrow. Disabling is
  `az appconfig feature disable`
- **C** — nothing increases on its own. You change the number
- **D** — the default is exactly what makes the groups meaningful

**Precedence to remember:** named **users** win, then **groups**, then the **default** percentage.

</details>

---

## Q8

Which attribute gates an entire controller action behind a feature flag?

- A. `[FeatureGate("FeatureName")]`
- B. `[Authorize("FeatureName")]`
- C. `[RequireFeature("FeatureName")]`
- D. `[FeatureFlag("FeatureName")]`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-27.md`:** line **268**.

```csharp
    [FeatureGate("NewPaymentProcessor")]
    [HttpPost("v2/payment")]
    public async Task<IActionResult> ProcessPaymentV2([FromBody] PaymentRequest request)
```

When the flag is off, the action returns **404** — the endpoint simply does not exist for that user.

**`[FeatureGate]` versus `IsEnabledAsync` — know when to use each:**

| Approach | Use when |
|---|---|
| `[FeatureGate]` | The whole endpoint should appear or disappear |
| `if (await _featureManager.IsEnabledAsync(...))` | You are choosing **between two behaviours** (line 257) |

The checkout example needs `IsEnabledAsync` because there is always a checkout — the question is
whether it is the new one or the legacy one.

**Why the others fail** — `[Authorize]` is authentication; the other two do not exist.

</details>

---

## Q9

Which middleware call enables dynamic feature-flag refresh in a .NET application?

- A. `app.UseAzureAppConfiguration()`
- B. `app.UseFeatureManagement()`
- C. `app.UseConfigurationRefresh()`
- D. `builder.Services.AddAzureAppConfiguration()`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-27.md`:** lines **208** and **218–220**.

```csharp
builder.Services.AddAzureAppConfiguration();   // registers the service
var app = builder.Build();
app.UseAzureAppConfiguration();                 // the MIDDLEWARE - triggers refresh
app.MapControllers();
```

**D is the deliberate near-miss.** Both lines are needed, and they are different things:
`AddAzureAppConfiguration()` registers services; `UseAzureAppConfiguration()` inserts the middleware
that actually checks for updates as requests flow through.

Registering the service without the middleware is Break & fix Exercise 1 — flags never refresh.

**Order matters (line 694):** the middleware must come **before** `MapControllers()`, or requests
reach your controllers before the refresh check runs.

**Why the others fail** — B and C do not exist.

</details>

---

## Q10

Which two NuGet packages does the challenge install for feature management?

- A. `Microsoft.Azure.AppConfiguration.AspNetCore` and `Microsoft.FeatureManagement.AspNetCore`
- B. `Azure.Identity` and `Microsoft.Extensions.Configuration`
- C. `Microsoft.ApplicationInsights` and `Microsoft.FeatureManagement`
- D. `Azure.Data.AppConfiguration` and `Microsoft.AspNetCore.Mvc`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-27.md`:** lines **177–178**.

```bash
dotnet add package Microsoft.Azure.AppConfiguration.AspNetCore
dotnet add package Microsoft.FeatureManagement.AspNetCore
```

Two packages doing two jobs: the first **connects** to App Configuration and refreshes; the second
**evaluates** flags and filters, and provides `[FeatureGate]`.

**Why the others fail** — `Azure.Identity` is used (line 187) but is a dependency, not the feature
package. `Azure.Data.AppConfiguration` is the low-level data SDK without ASP.NET integration.

</details>

---

## Q11

How does the application authenticate to App Configuration in `Program.cs`?

- A. A connection string in `appsettings.json`
- B. `DefaultAzureCredential` with a managed identity
- C. A stored access key
- D. A service principal secret

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-27.md`:** lines **192–195** and **226–231**.

```csharp
var appConfigEndpoint = builder.Configuration["AppConfig:Endpoint"];
options.Connect(new Uri(appConfigEndpoint), new DefaultAzureCredential())
```

```json
{ "AppConfig": { "Endpoint": "https://appconfig-contoso-prod.azconfig.io" } }
```

**Notice what is in `appsettings.json`: only an endpoint URL.** No key, no secret, nothing sensitive —
so the file can sit in source control safely.

`DefaultAzureCredential` tries a chain of credential sources; in Azure it finds the managed identity
that received the Data Reader role at line 74.

**Why the others fail**

- **A** and **C** — connection strings and access keys work, and both are long-lived secrets you must
  store and rotate. This is the same argument as OIDC versus a service principal secret
- **D** — again a stored secret

</details>

---

## Q12

What is the effect of `CacheExpirationInterval = TimeSpan.FromSeconds(30)`?

- A. Flags are re-read from App Configuration at most every 30 seconds
- B. Flags expire and become disabled after 30 seconds
- C. The application restarts every 30 seconds
- D. Flag changes take exactly 30 seconds to apply

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-27.md`:** line **199**.

```csharp
featureFlagOptions.CacheExpirationInterval = TimeSpan.FromSeconds(30);
```

It is a **polling interval**, not a countdown on the flag. The client keeps serving its cached values
until the interval elapses, then checks for changes on the next request.

**Why the others fail**

- **B** — flags do not expire. Only the cache does
- **C** — nothing restarts
- **D** — **the subtle one.** "At most 30 seconds", not "exactly". A change made just before an
  expiry is picked up almost immediately; one made just after waits nearly the full interval

**The trade-off:** shorter interval means faster kill-switch response and more requests to App
Configuration. 30 seconds is a reasonable production value; the default is longer.

</details>

---

## Q13

In the CI/CD workflow, when is the feature flag enabled?

- A. Before deploying to staging
- B. After the slot swap to production succeeds
- C. During the build stage
- D. Only manually, never in the pipeline

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-27.md`:** lines **423–438**.

```yaml
      - name: Swap to production
        run: az webapp deployment slot swap ... --slot staging --target-slot production

      - name: Enable feature flag for internal team
        run: |
          az appconfig feature enable --feature NewCheckoutFlow --label production --yes
```

**The order is the whole idea.** Deploy the code with the flag off — nobody sees it. Verify the
deployment is healthy. *Then* turn the feature on. Deployment risk and release risk are separated in
time, so if the flag causes a problem you disable it without touching the deployment.

**Why the others fail**

- **A** and **C** — enabling before the code is live would expose a feature whose code is not there
- **D** — the pipeline does it, though a manual workflow also exists (Task 6, line 476)

</details>

---

## Q14

The Azure Pipelines toggle task disables the flag when the health check fails.

What pattern is this?

- A. Blue-green deployment
- B. An automated kill switch driven by deployment health
- C. A canary release
- D. A rolling update

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-27.md`:** lines **457–471**.

```bash
      if [ "$HEALTH_STATUS" == "200" ]; then
        az appconfig feature enable  --feature $FEATURE_NAME --label $LABEL --yes
      else
        az appconfig feature disable --feature $FEATURE_NAME --label $LABEL --yes
        echo "Deployment unhealthy - feature flag '$FEATURE_NAME' disabled"
      fi
```

**The key property: disabling is instant and requires no deployment.** Compare the recovery paths —
a slot swap back takes seconds and moves infrastructure; a flag toggle takes one API call and moves
nothing.

**Why the others fail** — A, C and D are all deployment strategies. Nothing is being deployed here;
only configuration changes.

</details>

---

## Q15

An A/B test uses `Audience.DefaultRolloutPercentage=50`. How are users assigned to variants?

- A. Randomly on every request
- B. Consistently per user, based on a hash of their identifier
- C. Alternately, one user to each variant in turn
- D. By geographic region

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-27.md`:** lines **597–603**.

```bash
az appconfig feature filter add \
  --feature CheckoutExperiment \
  --filter-name "Microsoft.Targeting" \
  --filter-parameters Audience.DefaultRolloutPercentage=50
```

**Stickiness is what makes an A/B test valid.** The targeting filter hashes the user identifier, so a
given user always lands in the same variant. If assignment changed per request, a user could start
checkout in one flow and finish in the other — and your conversion data would be meaningless.

**Why the others fail**

- **A** — that is `Microsoft.Percentage`, which is random per evaluation and therefore **wrong for
  A/B testing**. This distinction is the exam-relevant part
- **C** — no round-robin exists, and it would not be sticky
- **D** — geography is not part of the targeting filter

</details>

---

## Q16

Which KQL function counts only the rows matching a condition, for A/B analysis?

- A. `count()`
- B. `countif()`
- C. `sum()`
- D. `dcount()`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-27.md`:** lines **648–656**.

```kusto
customEvents
| where name == "CheckoutCompleted"
| extend variant = tostring(customDimensions["variant"])
| summarize
    totalAttempts = count(),
    successCount = countif(tostring(customDimensions["success"]) == "true")
    by variant
| extend conversionRate = round(100.0 * successCount / totalAttempts, 2)
```

`count()` gives the denominator, `countif()` gives the numerator, and `by variant` splits both by
group — a conversion rate per variant in one query.

**Why the others fail**

- **A** — counts everything, so it cannot express "successful ones"
- **C** — sums a numeric column
- **D** — distinct count, useful for unique users but not for conversions

**Two details worth carrying into Challenge 49:** `customDimensions` is dynamic so values need
`tostring()`, and `100.0` forces floating-point division — `100` alone would truncate to an integer.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** are built-in Azure App Configuration feature filters? (Choose three.)

- A. `Microsoft.TimeWindow`
- B. `Microsoft.Targeting`
- C. `Microsoft.Percentage`
- D. `Microsoft.Geography`
- E. `Microsoft.DeviceType`
- F. `Microsoft.Schedule`

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-27.md`:** lines **114** and **134**.

| Filter | Enables when |
|---|---|
| `Microsoft.TimeWindow` | Now is between Start and End |
| `Microsoft.Targeting` | The user matches a user, group, or percentage — **sticky per user** |
| `Microsoft.Percentage` | A random share of evaluations — **not sticky** |

**Why the others fail** — `Microsoft.Geography`, `Microsoft.DeviceType` and `Microsoft.Schedule` do
not exist as built-ins. You can write **custom** filters implementing `IFeatureFilter` for exactly
these kinds of rules, which is the more useful thing to know.

</details>

---

## Q18

Which **two** are required for a targeting filter to identify users correctly? (Choose two.)

- A. `builder.Services.AddHttpContextAccessor()`
- B. A registered `ITargetingContextAccessor` implementation
- C. `AddFeatureFilter<PercentageFilter>()`
- D. A connection string instead of managed identity
- E. `CacheExpirationInterval` set below 60 seconds

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-27.md`:** lines **211–212**, and the failure at lines **700–715**.

```csharp
builder.Services.AddFeatureManagement()
    .AddFeatureFilter<TargetingFilter>();

builder.Services.AddHttpContextAccessor();                                     // A
builder.Services.AddSingleton<ITargetingContextAccessor,
                              HttpContextTargetingContextAccessor>();          // B
```

**Three links in a chain:** the filter asks the accessor, the accessor reads `HttpContext`, and
`IHttpContextAccessor` is what supplies it. Break any link and every user evaluates as
**anonymous with no groups**, which falls through to `DefaultRolloutPercentage`.

**Why the others fail**

- **C** — `PercentageFilter` is a different filter. Targeting needs `TargetingFilter` (line 205)
- **D** — authentication method, unrelated to user identity inside the app
- **E** — affects refresh speed, not identity

</details>

---

## Q19

Which **two** describe the correct order of operations in the deploy-then-enable pipeline? (Choose
two.)

- A. Deploy the code with the flag disabled
- B. Enable the flag only after the deployment is verified healthy
- C. Enable the flag before deploying so users see it immediately
- D. Deploy and enable in the same step to reduce pipeline duration
- E. Disable the flag before every deployment as a matter of routine

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-27.md`:** lines **94–99** (A) and **414–438** (B).

```bash
# Flag created, then immediately disabled - "safe deployment"
az appconfig feature disable --feature NewCheckoutFlow --label production --yes
```

```yaml
      - name: Validate deployment
      - name: Swap to production
      - name: Enable feature flag for internal team
```

**Why the others fail**

- **C** — enables a feature whose code is not deployed yet. Users hit missing endpoints
- **D** — couples them again, which is what flags exist to prevent. If the deploy is fine but the
  feature is bad, you now have no way to separate them
- **E** — **the interesting wrong answer.** Routinely disabling before each deploy would turn a
  working feature off for every unrelated release. Flags carry state deliberately; only *new* flags
  start disabled

</details>

---

## Q20

Which **two** make an A/B test statistically valid with feature flags? (Choose two.)

- A. Sticky per-user assignment via the targeting filter
- B. Tracking a variant dimension on every telemetry event
- C. Using `Microsoft.Percentage` for random assignment
- D. Reassigning users to variants hourly
- E. Enabling the feature for all users

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-27.md`:** lines **597–603** (A) and **622–638** (B).

```csharp
    var variant = isNewCheckout ? "new_checkout" : "legacy_checkout";
    _telemetryClient.TrackEvent("CheckoutStarted", new Dictionary<string, string>
    {
        ["variant"] = variant,
        ["userId"] = User.Identity?.Name ?? "anonymous"
    });
    _telemetryClient.TrackMetric("CheckoutDurationMs", stopwatch.ElapsedMilliseconds,
        new Dictionary<string, string> { ["variant"] = variant });
```

Without the `variant` dimension the KQL at line 650 has nothing to group by, and the experiment
produces no comparison at all.

**Why the others fail**

- **C** — random per evaluation, so a user can switch variants mid-session. Their behaviour is then
  attributable to neither
- **D** — the same problem, deliberately
- **E** — with everyone on one variant there is no control group

</details>

---

## Q21

Which **two** advantages do feature flags have over blue-green deployment for rollback? (Choose two.)

- A. Rollback requires no deployment at all
- B. Rollback can be scoped to specific users or groups
- C. Rollback is faster than a slot swap in every case
- D. Rollback requires no infrastructure duplication
- E. Rollback automatically reverts database changes

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-27.md`:** lines **156–167** (A) and **130–142** (B).

```bash
az appconfig feature disable --feature NewCheckoutFlow --label production --yes
```

Challenge 25's table (line 516) rates feature flags **1x cost, instant rollback** — the only row with
both.

**D is arguably true too** (1x versus blue-green's 2x), so A and B are the intended pair: they are
about *rollback*, while D is about cost.

**Why the others fail**

- **C** — too absolute. A slot swap is also seconds. The real difference is that a flag toggle changes
  **nothing about what is deployed**
- **E** — **the important one.** Neither flags nor slot swaps revert database changes. That is exactly
  why Challenge 29 insists on additive, backward-compatible migrations

</details>

---

## Q22

Which **two** are true about feature flag labels? (Choose two.)

- A. The same flag name with different labels is two independent records
- B. An application reads flags for one configured label
- C. Labels are optional and default to `production`
- D. Labels control which users see a feature
- E. Labels must match the Azure region

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-27.md`:** lines **720–747** (A) and **198** (B).

```csharp
featureFlagOptions.Label = "production";
```

**Why the others fail**

- **C** — labels are optional, and the default is **no label** (an empty label), not `production`. An
  app configured for `production` cannot see an unlabelled flag — a real and confusing failure
- **D** — that is the targeting **filter**. Labels separate **environments**, filters separate
  **audiences**. Do not mix them
- **E** — nothing to do with regions

</details>

---

## Q23

Which **two** requirements from Contoso's scenario do feature flags satisfy that a deployment strategy
alone cannot? (Choose two.)

- A. Deploy code to production with features hidden
- B. Instantly disable a feature without redeploying
- C. Achieve zero downtime during deployment
- D. Update instances gradually
- E. Warm the application before receiving traffic

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-27.md`:** the scenario at lines **16–21**.

> *1. Deploy code to production with features hidden behind flags*
> *4. Instantly disable a feature without redeploying if issues are detected*

**Why the others fail** — C, D and E are all **deployment** concerns solved in Challenges 25 and 26 by
slots, batches and warm-up. They are orthogonal: you still need a deployment strategy *and* flags.

**The division of labour:**

| Concern | Solved by |
|---|---|
| Getting code live safely | Slots, rolling, canary |
| Deciding who sees a feature | Feature flags |
| Turning a feature off | Feature flags |
| Turning a *release* back | Slot swap |

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must ship the new checkout flow to production without any customer seeing it,
enable it for the internal team first, then for 50% of beta testers, and be able to switch it off
instantly if problems appear.

---

## Q24

**Proposed solution:** Deploy the code with `NewCheckoutFlow` disabled, then add a
`Microsoft.Targeting` filter with `DefaultRolloutPercentage=0`, `InternalTeam` at 100% and
`BetaTesters` at 50%, and enable the flag.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-27.md`:** lines **94–99** and **130–142**.

Every requirement maps to a line of the filter:

| Requirement | Configuration |
|---|---|
| No customer sees it | `DefaultRolloutPercentage=0` |
| Internal team first | `Groups.0.Name="InternalTeam"`, `RolloutPercentage=100` |
| 50% of beta testers | `Groups.1.Name="BetaTesters"`, `RolloutPercentage=50` |
| Switch off instantly | `az appconfig feature disable` |

The code is live for everyone; only the **evaluation** differs per user.

</details>

---

## Q25

**Proposed solution:** Deploy the new checkout to a staging slot and use
`az webapp traffic-routing set --distribution staging=10` to send 10% of traffic to it.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

It fails two requirements outright.

**"Internal team first" is impossible.** Traffic routing splits by request at random. You cannot say
"the internal team, and only the internal team". There is no notion of *who* — only *how many*.

**"Without any customer seeing it" is violated immediately.** 10% of traffic is 10% of customers.

Rollback would work — set the distribution back to 0. But the audience requirements are the point of
the scenario, and this mechanism has no concept of audience.

**The distinction to carry:** traffic routing selects **requests**; targeting filters select
**users**. When a requirement names a group of people, you need flags.

</details>

---

## Q26

**Proposed solution:** Deploy the code with `NewCheckoutFlow` disabled, add a `Microsoft.Percentage`
filter set to 50, and enable the flag once the internal team has approved.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Two failures, and the second is subtle enough to be worth the question.

**No audience control.** `Microsoft.Percentage` cannot express "internal team first" or "beta testers
only". It only knows a number.

**Assignment is not sticky.** `Microsoft.Percentage` evaluates randomly **per evaluation**, so the
same user can see the new checkout on one page and the legacy one on the next. For a checkout flow
that is not a cosmetic problem — a user could add items in one flow and pay in the other.

`Microsoft.Targeting` with `DefaultRolloutPercentage=50` would look almost identical in configuration
and behave correctly, because it hashes the user identifier.

**The rule:** *anything user-facing and multi-step needs sticky assignment.* That means Targeting, not
Percentage.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — App Configuration setup

| # | Statement | Answer |
|---|---|---|
| 1 | Feature flags require the Standard tier |  |
| 2 | The Contributor role allows reading feature flag values |  |
| 3 | An application can authenticate with a managed identity |  |
| 4 | `App Configuration Data Reader` is a data-plane role |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Feature flags require the Standard tier | **Yes** |
| 2 | The Contributor role allows reading feature flag values | **No** |
| 3 | An application can authenticate with a managed identity | **Yes** |
| 4 | `App Configuration Data Reader` is a data-plane role | **Yes** |

**In `challenge-27.md`:** line **49**, lines **73–76**, line **195**.

Row 2 is the control-plane versus data-plane split, and it genuinely surprises people: a Contributor
can **delete the whole store** but cannot read a single key inside it. Azure separates managing a
resource from accessing its data — the same model as Key Vault and Storage.

</details>

---

## Q28 — flag evaluation

| # | Statement | Answer |
|---|---|---|
| 1 | A flag change requires an application restart |  |
| 2 | `CacheExpirationInterval` sets how often flags are re-read |  |
| 3 | `app.UseAzureAppConfiguration()` is required for dynamic refresh |  |
| 4 | Named users in a targeting filter override the default percentage |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A flag change requires an application restart | **No** |
| 2 | `CacheExpirationInterval` sets how often flags are re-read | **Yes** |
| 3 | `app.UseAzureAppConfiguration()` is required for dynamic refresh | **Yes** |
| 4 | Named users in a targeting filter override the default percentage | **Yes** |

**In `challenge-27.md`:** lines **199**, **218**, **141–142**.

Row 1 is the defining property. If a flag needed a restart, it would be a configuration setting, not a
kill switch — and the "instantly disable without redeploying" requirement would be unmet.

Row 3 is Break & fix Exercise 1: register the service *and* add the middleware, in that order, before
`MapControllers()`.

</details>

---

## Q29 — targeting and A/B testing

| # | Statement | Answer |
|---|---|---|
| 1 | `Microsoft.Targeting` assigns users consistently |  |
| 2 | `Microsoft.Percentage` assigns users consistently |  |
| 3 | An A/B test needs a variant dimension on telemetry |  |
| 4 | Group percentages take precedence over the default percentage |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `Microsoft.Targeting` assigns users consistently | **Yes** |
| 2 | `Microsoft.Percentage` assigns users consistently | **No** |
| 3 | An A/B test needs a variant dimension on telemetry | **Yes** |
| 4 | Group percentages take precedence over the default percentage | **Yes** |

**In `challenge-27.md`:** lines **597–603**, **622–638**, **136–140**.

Rows 1 and 2 together are the most exam-relevant pair in this challenge. **Sticky (Targeting) for
anything a user experiences; random (Percentage) only for stateless sampling.**

Row 3: without `["variant"] = variant` on the telemetry, the KQL at line 650 has nothing to group by
and the experiment yields no answer.

</details>

---

## Q30 — flags versus deployment strategies

| # | Statement | Answer |
|---|---|---|
| 1 | Disabling a flag requires a deployment |  |
| 2 | Feature flags remove the need for a deployment strategy |  |
| 3 | Feature flags can roll back a database schema change |  |
| 4 | Feature flags let code ship before the feature is released |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Disabling a flag requires a deployment | **No** |
| 2 | Feature flags remove the need for a deployment strategy | **No** |
| 3 | Feature flags can roll back a database schema change | **No** |
| 4 | Feature flags let code ship before the feature is released | **Yes** |

**In `challenge-27.md`:** lines **156–167**, the scenario at **16–21**.

Row 2 is the over-correction to watch for. Flags decide **who sees a feature**; they do nothing about
getting the binary onto the servers safely. You still need slots, rolling batches and warm-up.

Row 3 matters and leads directly into Challenge 29: a flag toggles **code paths**, not **data**. If
the new code dropped a column, turning the flag off will not bring it back — which is why migrations
must be additive and backward-compatible.

</details>

---

# Section E — Drag and drop

---

## Q31

Arrange the steps to ship a feature safely with a flag, in order.

**Items:** Enable the flag for the internal team · Deploy the code to production · Create the flag,
disabled · Raise the default rollout percentage · Verify the deployment is healthy

<details>
<summary>Show answer</summary>

### Answer

1. Create the flag, disabled — lines **87–99**
2. Deploy the code to production — lines **407–429**
3. Verify the deployment is healthy — lines **414–421**
4. Enable the flag for the internal team — lines **431–438**
5. Raise the default rollout percentage — line **136**

**The order encodes the whole idea.** Step 1 before step 2 means the code arrives dark. Step 3 before
step 4 separates *deployment* failure from *feature* failure — if step 3 fails you have a deployment
problem; if something breaks after step 4 you have a feature problem, and one API call reverses it.

</details>

---

## Q32

Match each feature filter to its behaviour.

| Filter | Behaviour |
|---|---|
| `Microsoft.TimeWindow` |  |
| `Microsoft.Targeting` |  |
| `Microsoft.Percentage` |  |
| Custom `IFeatureFilter` |  |

**Options:** Any rule you implement · Enabled between a start and end time · Enabled for a random share of evaluations — **not sticky** · Enabled per user, group, or percentage — **sticky**

<details>
<summary>Show answer</summary>

| Filter | Behaviour |
|---|---|
| `Microsoft.TimeWindow` | Enabled between a start and end time |
| `Microsoft.Targeting` | Enabled per user, group, or percentage — **sticky** |
| `Microsoft.Percentage` | Enabled for a random share of evaluations — **not sticky** |
| Custom `IFeatureFilter` | Any rule you implement |

**In `challenge-27.md`:** lines **114** and **134**.

**Choosing between Targeting and Percentage:**

| Need | Filter |
|---|---|
| A/B test with valid data | Targeting |
| Consistent user experience | Targeting |
| Named groups or users | Targeting |
| Random sampling of stateless work | Percentage |

For anything a person interacts with, the answer is Targeting.

</details>

---

## Q33

Match each Contoso requirement to its configuration.

| Requirement | Configuration |
|---|---|
| Nobody sees the feature by default |  |
| Internal team sees it |  |
| Half the beta testers see it |  |
| Two named people always see it |  |
| Maintenance banner during a window |  |
| Instant kill switch |  |

**Options:** `Audience.DefaultRolloutPercentage=0` · `Audience.Users.0`, `Audience.Users.1` · `az appconfig feature disable` · `Groups.0.Name="InternalTeam"`, `RolloutPercentage=100` · `Groups.1.RolloutPercentage=50` · `Microsoft.TimeWindow` filter

<details>
<summary>Show answer</summary>

| Requirement | Configuration |
|---|---|
| Nobody sees the feature by default | `Audience.DefaultRolloutPercentage=0` |
| Internal team sees it | `Groups.0.Name="InternalTeam"`, `RolloutPercentage=100` |
| Half the beta testers see it | `Groups.1.RolloutPercentage=50` |
| Two named people always see it | `Audience.Users.0`, `Audience.Users.1` |
| Maintenance banner during a window | `Microsoft.TimeWindow` filter |
| Instant kill switch | `az appconfig feature disable` |

**In `challenge-27.md`:** lines **130–142**, **110–115**, **163–167**.

**One targeting filter expresses a whole ring strategy.** Compare with Challenge 25, where rings
needed slots, a Traffic Manager profile and a bash script. Here the entire progression is six
parameters, and moving between rings is an `az` command with no deployment.

</details>

---

## Q34

Arrange the .NET configuration steps in `Program.cs`, in order.

**Items:** `app.UseAzureAppConfiguration()` · `builder.Configuration.AddAzureAppConfiguration(...)` ·
`app.MapControllers()` · `builder.Services.AddFeatureManagement()` · `var app = builder.Build()`

<details>
<summary>Show answer</summary>

### Answer

1. `builder.Configuration.AddAzureAppConfiguration(...)` — line **193**
2. `builder.Services.AddFeatureManagement()` — line **204**
3. `var app = builder.Build()` — line **215**
4. `app.UseAzureAppConfiguration()` — line **218**
5. `app.MapControllers()` — line **220**

**Two halves separated by `Build()`.** Everything on `builder` configures services; everything on
`app` builds the request pipeline. You cannot add services after `Build()`, and you cannot add
middleware before it.

**Step 4 before step 5 is the requirement from Break & fix Exercise 1** (line 694): the refresh
middleware must run before requests reach controllers, or flags are evaluated from a stale cache.

</details>

---

## Q35

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Flag change not visible after 10 minutes |  |
| Internal team not seeing a 100% group flag |  |
| Works in staging, not production |  |
| Application gets 403 reading flags |  |
| A user sees both variants in one session |  |

**Options:** Cache interval too high, or refresh middleware missing · `IHttpContextAccessor` not registered · Label mismatch · `Microsoft.Percentage` used instead of Targeting · Missing `App Configuration Data Reader` role

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| Flag change not visible after 10 minutes | **Cache interval too high, or refresh middleware missing** |
| Internal team not seeing a 100% group flag | **`IHttpContextAccessor` not registered** |
| Works in staging, not production | **Label mismatch** |
| Application gets 403 reading flags | **Missing `App Configuration Data Reader` role** |
| A user sees both variants in one session | **`Microsoft.Percentage` used instead of Targeting** |

**In `challenge-27.md`:** lines **681**, **708**, **737**, **73–76**, **597–603**.

**Read the symptom shape:** *stale* means caching. *Everyone anonymous* means identity. *Environment
differs* means labels. *403* means RBAC. *Inconsistent per request* means the wrong filter.

</details>

---

# Section F — Hot area

---

## Q36

```bash
az appconfig create \
  --name appconfig-contoso-prod \
  --sku [BLANK 1]

az role assignment create \
  --assignee $APP_PRINCIPAL_ID \
  --role "[BLANK 2]" \
  --scope $CONFIG_ID
```

- **BLANK 1:** `Standard` / `Free` / `Basic` / `Premium`
- **BLANK 2:** `App Configuration Data Reader` / `Reader` / `Contributor` /
  `App Configuration Contributor`

<details>
<summary>Show answer</summary>

### Answer: `Standard`, `App Configuration Data Reader`

**In `challenge-27.md`:** lines **49** and **75**.

Free has no feature management. `Reader` and `Contributor` are **control-plane** roles — they can see
and manage the resource but cannot read a single flag inside it.

</details>

---

## Q37

```bash
az appconfig feature filter add \
  --feature MaintenanceMode \
  --filter-name "[BLANK 1]" \
  --filter-parameters Start="2025-03-01T00:00:00Z" End="2025-03-01T06:00:00Z"
```

- **BLANK 1:** `Microsoft.TimeWindow` / `Microsoft.Targeting` / `Microsoft.Percentage` /
  `Microsoft.Schedule`

<details>
<summary>Show answer</summary>

### Answer: `Microsoft.TimeWindow`

**In `challenge-27.md`:** lines **110–115**.

The timestamps are UTC (`Z`). Note this is a **maintenance banner**, not a maintenance mode that
blocks traffic — the flag controls what the application displays.

</details>

---

## Q38

```bash
az appconfig feature filter add \
  --feature NewPaymentProcessor \
  --filter-name "Microsoft.Targeting" \
  --filter-parameters \
    Audience.[BLANK 1]=0 \
    Audience.Groups.0.Name="InternalTeam" \
    Audience.Groups.0.[BLANK 2]=100
```

- **BLANK 1:** `DefaultRolloutPercentage` / `BaseRolloutPercentage` / `FallbackPercentage` /
  `GlobalPercentage`
- **BLANK 2:** `RolloutPercentage` / `Percentage` / `Weight` / `Share`

<details>
<summary>Show answer</summary>

### Answer: `DefaultRolloutPercentage`, `RolloutPercentage`

**In `challenge-27.md`:** lines **136** and **138**.

The naming is worth memorising exactly: **`DefaultRolloutPercentage`** at audience level, and
**`RolloutPercentage`** inside each group. Groups and users are indexed from `0`.

</details>

---

## Q39

```csharp
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(new Uri(appConfigEndpoint), new [BLANK 1]())
           .UseFeatureFlags(o =>
           {
               o.Label = "production";
               o.[BLANK 2] = TimeSpan.FromSeconds(30);
           });
});
```

- **BLANK 1:** `DefaultAzureCredential` / `ClientSecretCredential` / `StorageSharedKeyCredential` /
  `AzureKeyCredential`
- **BLANK 2:** `CacheExpirationInterval` / `RefreshInterval` / `PollingInterval` / `TimeToLive`

<details>
<summary>Show answer</summary>

### Answer: `DefaultAzureCredential`, `CacheExpirationInterval`

**In `challenge-27.md`:** lines **195** and **199**.

`DefaultAzureCredential` picks up the managed identity in Azure and your developer login locally —
the same credential class as your OIDC block, and the reason nothing secret sits in
`appsettings.json`.

</details>

---

## Q40

```csharp
var app = builder.Build();

app.[BLANK 1]();
app.MapControllers();
```

Requirement: feature flag changes must take effect without restarting the app.

- **BLANK 1:** `UseAzureAppConfiguration` / `UseFeatureManagement` / `UseConfigurationRefresh` /
  `AddAzureAppConfiguration`

<details>
<summary>Show answer</summary>

### Answer: `UseAzureAppConfiguration`

**In `challenge-27.md`:** lines **218–220**, and Break & fix Exercise 1 at line **694**.

`AddAzureAppConfiguration` is the near-miss: it registers the **service** on `builder.Services` (line
208) and is also required — but it is not middleware and cannot be called on `app`.

**Order matters.** This line must come before `MapControllers()`.

</details>

---

## Q41

```csharp
    [[BLANK 1]("NewPaymentProcessor")]
    [HttpPost("v2/payment")]
    public async Task<IActionResult> ProcessPaymentV2(...)

    // elsewhere, choosing between two behaviours:
    if (await _featureManager.[BLANK 2]("NewCheckoutFlow"))
```

- **BLANK 1:** `FeatureGate` / `RequireFeature` / `FeatureFlag` / `Authorize`
- **BLANK 2:** `IsEnabledAsync` / `GetFlagAsync` / `EvaluateAsync` / `CheckFeatureAsync`

<details>
<summary>Show answer</summary>

### Answer: `FeatureGate`, `IsEnabledAsync`

**In `challenge-27.md`:** lines **268** and **257**.

Both appear in the same controller because they solve different problems: `[FeatureGate]` makes the
endpoint **return 404** when off, while `IsEnabledAsync` picks between the new and legacy checkout —
there is always a checkout.

</details>

---

# Section G — Case study

## Case study: Contoso feature release platform

### Background

Contoso ships a "big bang" release every two weeks, bundling many features. When something breaks,
nobody can tell which feature caused it, and the only remedy is rolling back the whole release.

### Requirements

**Release control**

- Code must reach production with new features invisible to customers
- Features must be enabled for internal staff before anyone else
- Beta testers must be enabled gradually by percentage
- A feature must be switchable off instantly, with no deployment

**Experimentation**

- A 50/50 A/B test comparing the new checkout against the existing one
- Results must be comparable per variant in Application Insights

**Operations**

- No secret may be stored in application configuration
- Flag changes must reach running instances within about a minute
- Staging and production must be able to hold different values for the same flag

---

## Q42

Which configuration meets "invisible to customers, internal staff first, beta testers gradually"?

- A. A `Microsoft.Targeting` filter with a 0 default and per-group percentages
- B. A `Microsoft.Percentage` filter set to 5
- C. Separate deployments per audience
- D. App Service traffic routing at 5%

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-27.md`:** lines **130–142**.

```bash
    Audience.DefaultRolloutPercentage=0        # customers: invisible
    Audience.Groups.0.Name="InternalTeam"
    Audience.Groups.0.RolloutPercentage=100    # internal: all
    Audience.Groups.1.Name="BetaTesters"
    Audience.Groups.1.RolloutPercentage=50     # beta: half
```

**Why the others fail**

- **B** — a number cannot name a group, and Percentage is not sticky
- **C** — separate deployments per audience is the maintenance burden flags exist to remove
- **D** — routes **requests**, not **users**. It cannot express "internal staff"

</details>

---

## Q43

Which configuration meets the instant kill-switch requirement?

- A. `az appconfig feature disable` on the flag
- B. A slot swap back to the previous version
- C. Redeploying the previous build
- D. Scaling the App Service to zero

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-27.md`:** lines **163–167**.

```bash
az appconfig feature disable --feature NewCheckoutFlow --label production --yes
```

The requirement says **"with no deployment"**. This is one API call; running instances pick it up on
their next cache refresh.

**Why the others fail**

- **B** — fast, but it reverts the **whole release**, including unrelated fixes. That is exactly the
  problem stated in the background
- **C** — minutes, and a build
- **D** — that is an outage, not a rollback

</details>

---

## Q44

Which **two** configurations make the A/B test produce usable data? (Choose two.)

- A. `Microsoft.Targeting` with `DefaultRolloutPercentage=50`
- B. Tracking a `variant` dimension on checkout events and metrics
- C. `Microsoft.Percentage` set to 50
- D. Enabling the feature for all users after one day
- E. Recording only successful checkouts

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-27.md`:** lines **597–603** and **622–638**.

**Why the others fail**

- **C** — random per evaluation, so a user can switch variants mid-checkout
- **D** — ending the control group ends the experiment
- **E** — **the subtle one.** The KQL at line 652 needs `count()` for total attempts *and*
  `countif()` for successes. Recording only successes leaves you with a numerator and no
  denominator — you cannot compute a conversion rate at all

</details>

---

## Q45

Which configuration meets "no secret in application configuration"?

- A. `DefaultAzureCredential` with a managed identity granted Data Reader
- B. The App Configuration connection string in `appsettings.json`
- C. An App Configuration access key in an environment variable
- D. A service principal secret in Key Vault

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-27.md`:** lines **195**, **226–231**, **73–76**.

```json
{ "AppConfig": { "Endpoint": "https://appconfig-contoso-prod.azconfig.io" } }
```

An endpoint URL is not a secret, so this file is safe in source control.

**Why the others fail**

- **B** and **C** — both are long-lived credentials that must be stored and rotated
- **D** — **the near-miss worth understanding.** Key Vault protects the secret well, but a secret
  still exists, still expires, and still has to be rotated. The requirement says *no secret* — so the
  answer is an identity, not a better hiding place

**This is the same reasoning as OIDC versus a stored service principal secret**, and it is the pattern
that has cost you marks in every mock.

</details>

---

## Q46

Which setting ensures flag changes reach instances within about a minute?

- A. `CacheExpirationInterval = TimeSpan.FromSeconds(30)`
- B. `CacheExpirationInterval = TimeSpan.FromHours(1)`
- C. Restarting the App Service after every change
- D. `WEBSITE_SWAP_WARMUP_PING_PATH`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-27.md`:** line **199**.

Thirty seconds means a change is visible within thirty seconds at worst, comfortably inside "about a
minute".

**Why the others fail**

- **B** — up to an hour, and the default is longer than you would want for a kill switch
- **C** — defeats the purpose, and a restart drops in-flight requests
- **D** — a slot warm-up setting from Challenge 26. Unrelated

**The trade-off to state:** shorter interval means faster kill-switch response and more requests to
App Configuration. Thirty seconds is a sensible production balance.

</details>

---

## Q47

Which configuration lets staging and production hold different values for `NewCheckoutFlow`?

- A. Labels — one flag record per environment
- B. Two App Configuration stores
- C. A targeting filter with an `Environment` group
- D. Two feature names, `NewCheckoutFlow_Staging` and `NewCheckoutFlow_Prod`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-27.md`:** lines **90**, **198**, **720–747**.

```bash
az appconfig feature enable --feature NewCheckoutFlow --label production --yes
```

```csharp
featureFlagOptions.Label = "production";
```

**Why the others fail**

- **B** — **works**, and is heavier: two resources, two role assignments, two sets of RBAC to keep in
  step. Labels are the built-in answer. (Separate stores do become right at strict isolation
  boundaries, so this is a judgement call the exam usually resolves toward labels)
- **C** — targeting selects **users**, not environments
- **D** — different names mean the application must know which name to ask for, so you are back to
  environment-specific configuration with none of the tooling

**And the failure mode to remember:** enabling a flag under the wrong label is Break & fix Exercise 3
— it looks enabled in the portal and does nothing in production.

</details>

---

## Q48

After adopting flags, Contoso accumulates 40 flags, several enabled at 100% for over a year. A
developer removes an old flag from App Configuration, and the application starts throwing errors.

What happened, and what should Contoso do?

- A. The code still evaluates the removed flag; missing flags evaluate as false, changing behaviour
- B. Removing a flag requires restarting the application
- C. The Data Reader role was revoked with the flag
- D. The cache expiration interval was too short

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-27.md`:** the evaluation call at line **257**.

```csharp
if (await _featureManager.IsEnabledAsync("NewCheckoutFlow"))
```

When the flag no longer exists, `IsEnabledAsync` returns **false**. The application silently reverts
to the legacy path — which, a year later, may no longer work with the current database schema or
downstream services.

**The correct order for retiring a flag:**

1. Confirm the flag has been at 100% long enough to be trusted
2. **Remove the conditional from the code**, keeping only the new path
3. Deploy that change
4. **Then** delete the flag from App Configuration

**Why the others fail**

- **B** — flags are read dynamically. No restart involved
- **C** — role assignments are on the store, not on individual flags
- **D** — a short interval makes changes apply *sooner*; it did not cause this

**The wider point — flag debt is real.** Forty flags means forty conditionals, and the number of code
paths grows combinatorially. Every flag needs a planned removal date, and Challenge 34's pipeline
health work is where you would track it.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Targeting vs Percentage** | Q6, Q15, Q17, Q26, Q29, Q44 | Targeting is sticky per user. Percentage is random per evaluation |
| **Control-plane role offered for data access** | Q2, Q27, Q36 | Contributor cannot read flags. Use `App Configuration Data Reader` |
| **Label mismatch** | Q5, Q22, Q47 | A label makes a separate record. Empty label is not `production` |
| **Missing refresh middleware** | Q3, Q9, Q28, Q40 | `AddAzureAppConfiguration()` **and** `UseAzureAppConfiguration()`, before `MapControllers()` |
| **`IHttpContextAccessor` not registered** | Q4, Q18, Q35 | Every user becomes anonymous with no groups |
| **Flags assumed to replace deployment strategy** | Q23, Q30 | Flags decide who sees a feature. Slots decide how code arrives |
| **Flags assumed to roll back data** | Q21, Q30 | Toggling code paths never reverts a schema change |
| **Traffic routing offered for audience control** | Q25, Q42 | Routing selects requests. Targeting selects users |
| **Key Vault offered as "no secret"** | Q45 | A hidden secret is still a secret. Use an identity |
| **`AddAzureAppConfiguration` vs `UseAzureAppConfiguration`** | Q9, Q40 | Service registration vs middleware. Both required |
| **Deleting a flag before removing the code** | Q48 | A missing flag evaluates as false |
| **Enabling the flag before deploying** | Q19 | Deploy dark, verify, then enable |

---

# The blocks to memorise

Line numbers are in `challenge-27.md`.

```bash
# 1. Store and access  (lines 45-76)
az appconfig create --sku Standard                  # Standard required for flags
az role assignment create --assignee $APP_PRINCIPAL_ID \
  --role "App Configuration Data Reader" --scope $CONFIG_ID

# 2. Flag lifecycle  (lines 87-99, 156-167)
az appconfig feature set     --feature NewCheckoutFlow --label production --yes
az appconfig feature disable --feature NewCheckoutFlow --label production --yes   # deploy dark
az appconfig feature enable  --feature NewCheckoutFlow --label production --yes   # release
az appconfig feature disable --feature NewCheckoutFlow --label production --yes   # kill switch

# 3. Targeting filter - the whole ring strategy  (lines 130-142)
az appconfig feature filter add --feature NewPaymentProcessor \
  --filter-name "Microsoft.Targeting" \
  --filter-parameters \
    Audience.DefaultRolloutPercentage=0 \
    Audience.Groups.0.Name="InternalTeam"  Audience.Groups.0.RolloutPercentage=100 \
    Audience.Groups.1.Name="BetaTesters"   Audience.Groups.1.RolloutPercentage=50 \
    Audience.Users.0="admin@contoso.com"

# 4. Time window filter  (lines 110-115)
  --filter-name "Microsoft.TimeWindow" \
  --filter-parameters Start="2025-03-01T00:00:00Z" End="2025-03-01T06:00:00Z"
```

```csharp
// 5. Program.cs wiring  (lines 193-220)
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(new Uri(appConfigEndpoint), new DefaultAzureCredential())
           .UseFeatureFlags(o =>
           {
               o.Label = "production";
               o.CacheExpirationInterval = TimeSpan.FromSeconds(30);
           });
});
builder.Services.AddFeatureManagement().AddFeatureFilter<TargetingFilter>();
builder.Services.AddAzureAppConfiguration();
builder.Services.AddHttpContextAccessor();
builder.Services.AddSingleton<ITargetingContextAccessor, HttpContextTargetingContextAccessor>();

var app = builder.Build();
app.UseAzureAppConfiguration();   // BEFORE MapControllers
app.MapControllers();

// 6. Two ways to use a flag  (lines 257, 268)
if (await _featureManager.IsEnabledAsync("NewCheckoutFlow")) { ... }   // choose behaviour
[FeatureGate("NewPaymentProcessor")]                                   // 404 when off

// 7. A/B telemetry  (lines 622-638)
_telemetryClient.TrackEvent("CheckoutStarted",
    new Dictionary<string, string> { ["variant"] = variant, ["userId"] = ... });
```

```kusto
// 8. A/B analysis  (lines 648-656)
customEvents
| where name == "CheckoutCompleted"
| extend variant = tostring(customDimensions["variant"])
| summarize totalAttempts = count(),
            successCount = countif(tostring(customDimensions["success"]) == "true")
    by variant
| extend conversionRate = round(100.0 * successCount / totalAttempts, 2)
```

**Precedence in a targeting filter:** named **users** → **groups** → **default** percentage.

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 27 is exam-ready. Move to Challenge 28 |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 2 and 4, then retake this |
| Below 30 | Redo the challenge, and write the targeting filter parameters from memory |

Record your result in `AZ-400-Learning-Log.md` under Challenge 27.

:::tip The one thing

**Deploy dark, verify, then release — and the kill switch is a config change, not a deployment.**

Every advantage feature flags have over blue-green and canary follows from that one property.

:::
