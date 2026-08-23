---
sidebar_position: 4.5
toc_max_heading_level: 2
title: "Challenge 37: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 37 — AZ-400 exam questions

**48 questions** built only from what Challenge 37 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-37.md`**.

| Section | Shape | Questions |
|---|---|---|
| A | Single answer | 1–16 |
| B | Multiple answer (choose two / three) | 17–23 |
| C | Repeated scenario — "Does this meet the goal?" | 24–26 |
| D | Yes/No statement grid | 27–30 |
| E | Drag and drop | 31–35 |
| F | Hot area — complete the configuration | 36–41 |
| G | Case study | 42–48 |

The **trap index**, the **mapping table**, and **scoring** are at the end.

:::danger The mapping table at lines 148–161 is the whole challenge

Twelve classic concepts, twelve YAML equivalents. Memorise it and most of Section A answers itself.
The two that catch people: **gates become environment *checks*, not YAML**, and **artifacts need an
explicit `resources: pipelines:` declaration** that classic did implicitly.

:::

---

# Section A — Single answer

---

## Q1

What is the YAML equivalent of a classic release definition's **pre-deployment gates**?

- A. Environment checks such as Invoke REST API and Azure Monitor alerts
- B. A `condition` on the stage
- C. A `dependsOn` between stages
- D. A `gates:` block in the YAML

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** the mapping table at line **156**, with configuration at lines **168–173**.

```text
| Pre-deployment gates | Environment checks (Invoke REST API, Azure Monitor) |
```

**The important structural difference:** in classic, gates are configured **per stage inside the
release definition**. In YAML they live **on the environment**, and they apply to *any* pipeline that
deploys to it (line 744).

That is an improvement in governance — someone editing the YAML cannot remove the gate — and it is
the thing that surprises people mid-migration.

**Why the others fail**

- **B** — a condition is automatic logic evaluated by the pipeline. A gate queries an external system
- **C** — ordering, not gating
- **D** — **no `gates:` key exists in YAML.** That is precisely the trap: people look for the classic
  concept as a YAML keyword and it is not there

</details>

---

## Q2

A migrated CD pipeline fails with no artifact found when using `download: current`.

What is missing?

- A. A `resources: pipelines:` declaration for the CI pipeline
- B. A `PublishPipelineArtifact` task in the CD pipeline
- C. A service connection
- D. `fetchDepth: 0` on the checkout

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** Break & fix Exercise 1, lines **655–700**.

```yaml
# BROKEN
                - download: current  # ERROR: No artifact in current pipeline
                  artifact: api-build
```

```yaml
# FIXED
resources:
  pipelines:
    - pipeline: ci-build              # alias for reference
      source: "Contoso API CI (YAML)" # exact pipeline name
      trigger:
        branches:
          include: [main]
...
                - download: ci-build  # use the alias
```

**`download: current` means "an artifact this run produced".** The CD pipeline builds nothing, so
there is nothing to download.

**Why classic did not need this:** a classic release definition had an **artifact source** linked in
the UI, so the association was implicit. YAML makes it explicit, which is the recurring theme of the
whole migration — configuration that lived in a form now lives in the file.

**Note the download path** (line 700): `$(Pipeline.Workspace)/ci-build/api-build` — workspace, then
**alias**, then artifact name.

</details>

---

## Q3

After migration, production deployments happen with no approval.

Where must the approval be configured?

- A. In the Azure DevOps UI on the environment
- B. In the YAML pipeline as an `approvals:` block
- C. In a branch policy
- D. In the variable group

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** Break & fix Exercise 2, lines **709–744**.

```text
1. Navigate to: Pipelines > Environments > production
2. Click three dots (...) > Approvals and checks
3. Add check: "Approvals"
```

> *"Environment checks must be configured in the Azure DevOps UI (they cannot be set via YAML)"*

**This is deliberate, not an omission.** If approvals lived in YAML, anyone with write access to the
pipeline could delete the gate in the same commit that ships the change. Keeping them on the
environment means removing a gate requires **environment permissions**, a different and narrower
right.

**The migration risk is that the environment exists but is empty.** The YAML references
`environment: production`, the deployment succeeds, and it looks correct — while the classic
definition's approvals were left behind.

**Why the others fail** — B does not exist, C gates merges, D holds values.

</details>

---

## Q4

What is the YAML equivalent of a classic **task group**?

- A. A YAML template referenced with `template:`
- B. A variable group
- C. A composite action
- D. A deployment group

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** the mapping table at line **158**, with the worked conversion at lines
**346–414**.

```yaml
# Classic task group: "Build and Test Node.js App"
# Converted to: templates/build-test-node.yml
parameters:
  - name: nodeVersion
    type: string
    default: "20.x"
steps:
  - task: NodeTool@0
```

**Templates are strictly more capable than task groups**, which is the argument for migrating rather
than merely tolerating it:

| | Task group | YAML template |
|---|---|---|
| Reuse scope | **Steps only** | Steps, jobs, **or stages** |
| Version control | Inside Azure DevOps | **Git** — review, history, pinning |
| Conditional logic | Limited | Full `${{ if }}` / `${{ each }}` |

Line 389 shows the last row in use: `${{ if eq(parameters.publishArtifact, true) }}`.

**Why the others fail** — B holds values, C is GitHub Actions, D is a set of target machines.

</details>

---

## Q5

What replaces a classic **deployment group** in YAML?

- A. An environment with Virtual Machine resources
- B. An agent pool
- C. A self-hosted runner group
- D. A variable group

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** the mapping table at line **157**, with the implementation at lines
**502–505**.

```yaml
        environment:
          name: contoso-prod-vms
          resourceType: VirtualMachine
          tags: "web"  # Target only VMs tagged as "web"
```

**`tags:` is the selector**, and it is what replaces classic deployment group tags: register 20 VMs,
tag some `web` and some `worker`, and each stage targets only the ones it needs.

**Registration is the same idea as a self-hosted agent** but with `--environment` (line 476):

```bash
./config.sh --environment --environmentname "contoso-prod-vms" \
  --agent $HOSTNAME --url https://dev.azure.com/contoso --auth pat --token <PAT>
```

**Why the others fail** — B runs pipeline jobs, C is GitHub Actions, D holds values.

</details>

---

## Q6

Which strategy deploys to VM environment resources a few at a time?

- A. `rolling` with `maxParallel`
- B. `runOnce`
- C. `matrix`
- D. `canary` with `increments`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** lines **506–539**.

```yaml
        strategy:
          rolling:
            maxParallel: 2  # Deploy to 2 VMs at a time
            preDeploy:      # take out of the load balancer
            deploy:
            postRouteTraffic:   # health check after deploy
            on:
              failure:      # roll back
```

**With VM resources, `rolling` iterates over the *machines*** — `maxParallel: 2` means two VMs updated
at once while the rest keep serving. That is the same drain-update-return-verify cycle as Challenge 26,
now applied to registered environment resources.

**Why the others fail**

- **B** — runs the steps once, not per machine
- **C** — a **regular job** strategy, not available on deployment jobs
- **D** — `canary` is real and splits by **traffic increments** rather than by machine count. Valid,
  and not what "a few at a time" describes

</details>

---

## Q7

How do variable groups transfer from classic to YAML?

- A. Referenced by name with `variables: - group:` — the group itself is unchanged
- B. They must be recreated as YAML variables
- C. They are converted automatically during export
- D. They are replaced by environment secrets

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** lines **293–318**.

```yaml
variables:
  - group: contoso-api-common
  - group: contoso-api-production
  - name: localVar
    value: "inline-value"
```

**Nothing about the group changes** — same Library entry, same Key Vault link, same secrets. Only the
reference moves into the file.

**And stage scoping works the same way** (lines 302–317): `contoso-api-dev` on the Dev stage,
`contoso-api-production` on the Production stage, so `$(DB_HOST)` resolves differently per stage. Same
expression, different value — the pattern from Challenge 20.

**Why the others fail** — B discards the secret management, C is untrue (export handles steps, not
Library resources), D is the GitHub concept.

</details>

---

## Q8

How do service connections transfer to YAML?

- A. Referenced by the same name in task inputs; no change to the connection
- B. They must be recreated for YAML pipelines
- C. They are replaced by managed identities automatically
- D. They only work with classic pipelines

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** lines **322–327**.

```yaml
- task: AzureWebApp@1
  inputs:
    azureSubscription: "contoso-prod-sc"  # Same service connection name
```

**One extra step catches people** (line 339): the YAML pipeline needs **pipeline permissions** on the
connection. A connection restricted to specific pipelines does not automatically authorise the new
one, and the failure is an authorisation error that looks like a broken connection.

The same applies to **environments** — a newly created YAML pipeline may need explicit permission to
deploy to one.

**Why the others fail** — B, C and D are all untrue.

</details>

---

## Q9

Which classic concept maps to `jobs:` with different `pool:` settings?

- A. Agent phases
- B. Release stages
- C. Task groups
- D. Deployment groups

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** the mapping table at line **160**.

```text
| Agent phases | `jobs:` with different `pool:` settings |
```

**Classic build definitions had "phases"** — agent phases, agentless (server) phases and deployment
group phases. Each ran on a different execution surface, and YAML collapses that into `jobs:` with the
appropriate `pool:` or `environment:`.

**Why the others fail** — B maps to `stages:` (line 152), C to templates (line 158), D to environments
with VM resources (line 157).

</details>

---

## Q10

Where is the export-to-YAML feature found for a classic build pipeline?

- A. Edit the pipeline, then the three-dots menu, then "Export to YAML"
- B. Project Settings, then Pipelines, then Export
- C. `az pipelines export`
- D. The Analytics tab

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** lines **51–56**.

```text
1. Navigate to Pipelines > [Select classic pipeline]
2. Click "Edit"
3. Click the three dots menu (...)
4. Select "Export to YAML" (if available in your Azure DevOps version)
```

**Note the caveat — "if available".** The feature covers classic **build** definitions and does not
exist for classic **releases**, which is why the release migration in Task 3 is done by hand against
the mapping table.

**The per-task fallback** (line 58): "View YAML" on an individual task shows its YAML equivalent, which
you assemble manually. Slower and always available.

**Why the others fail** — B is project-level settings, C is not a valid command, D shows metrics.

</details>

---

## Q11

In the phased migration, what happens during **Phase 2**?

- A. YAML becomes the official pipeline; classic triggers are disabled but the definition is retained
- B. The classic pipeline is deleted
- C. Both pipelines run in parallel against the same environments
- D. The YAML pipeline is created for the first time

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** lines **553–557**.

```text
Phase 2: YAML primary (Weeks 3-4)
- YAML pipeline becomes the official CI/CD
- Classic pipeline triggers disabled but retained
- Team uses YAML for all new deployments
- Classic available as emergency fallback
```

**"Disabled but retained" is the whole point of the phase.** If the YAML pipeline turns out to have a
gap nobody noticed in Phase 1, re-enabling the classic trigger is a one-click recovery. Deleting it
would make that a rebuild.

**Why the others fail**

- **B** — that is Phase 3, after two clean weeks (line 560)
- **C** — that is Phase 1, and note the YAML pipeline deploys to a **shadow environment** (line 551),
  not the real one
- **D** — Phase 1

</details>

---

## Q12

Why does the Phase 1 YAML pipeline deploy to a **shadow** environment?

- A. So both pipelines can run on the same triggers without competing for the real environment
- B. To reduce cost
- C. Because YAML cannot deploy to production
- D. To skip approvals during migration

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** lines **547–551**.

```text
Phase 1: Parallel run (Weeks 1-2)
- Create YAML pipeline alongside classic
- Both trigger on the same events
- Compare results (same artifacts, same tests, same deployments)
- YAML pipeline deploys to a separate "shadow" environment
```

**Two pipelines deploying to the same environment would race**, overwrite each other, and make it
impossible to attribute a failure. A shadow environment lets you compare **outputs** — same artifacts,
same tests, same deployment result — while the classic pipeline remains the one that matters.

**Why the others fail**

- **B** — a shadow environment **adds** cost for two weeks. That is the price of a safe migration
- **C** — untrue
- **D** — approvals should be *validated* during migration, not bypassed. Break & fix Exercise 2 is
  what happens when they are forgotten

</details>

---

## Q13

What should be done with the classic definition before deleting it?

- A. Export it to JSON and archive it
- B. Nothing — deletion is reversible
- C. Convert it to a task group
- D. Move it to another project

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** lines **561** and **566–571**.

```bash
az pipelines show \
  --org https://dev.azure.com/contoso \
  --project ContosoAPI \
  --id 42 \
  --output json > archived-classic-pipelines/contoso-api-ci-classic.json
```

**The JSON is the record of what the classic pipeline actually did** — every task, input and
condition. Months later, when someone asks why a step exists, that file is the only evidence.

**Deletion is not reversible**, which is why Phase 3 waits two weeks after Phase 2 and archives first.

**Why the others fail** — B is false, C solves nothing, D moves the problem.

</details>

---

## Q14

Which CLI command lists classic build definitions in a project?

- A. `az pipelines list --query "[?type=='build']"`
- B. `az pipelines runs list`
- C. `az devops project list`
- D. `az pipelines build queue`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** lines **66–70**.

```bash
az pipelines list \
  --org https://dev.azure.com/contoso \
  --project ContosoAPI \
  --query "[?type=='build'].{Id:id, Name:name, Type:type}" \
  --output table
```

**This is the inventory step**, and it is where a 20-pipeline migration starts: list everything, then
work out which are still triggered, which are dead, and which share task groups.

`az pipelines show --id 42` (line 73) then returns the triggers, variables and process for one
definition — the raw material for the conversion.

**Why the others fail** — B lists runs, C lists projects, D queues a build.

</details>

---

## Q15

Which YAML construct replaces a classic release definition's **artifact source**?

- A. `resources: pipelines:` with an alias
- B. `steps: - checkout:`
- C. `variables: - group:`
- D. `pool: vmImage:`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** the mapping table at line **154**, with the syntax at lines **178–184**.

```yaml
resources:
  pipelines:
    - pipeline: ci-build
      source: "Contoso API - CI (YAML)"
      trigger:
        branches:
          include: [main]
```

**The block does two jobs at once**, which is easy to miss: it makes the artifacts downloadable
**and** it triggers this pipeline when the source pipeline completes. The `trigger:` sub-block is what
replaces the classic "continuous deployment trigger".

**`pipeline:` is the alias; `source:` is the real pipeline name.** Mixing them up gives a
resource-not-found error — the same distinction as Challenge 22 Q34.

**Why the others fail** — B fetches source code, C holds values, D selects an agent.

</details>

---

## Q16

Which classic feature has **no direct YAML keyword**, forcing configuration elsewhere?

- A. Pre-deployment approvals and gates
- B. Variable groups
- C. Task groups
- D. Agent phases

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** line **724**.

> *"Environment checks must be configured in the Azure DevOps UI (they cannot be set via YAML)"*

**Everything else in the mapping table has a keyword.** Variable groups become `variables: - group:`,
task groups become `template:`, agent phases become `jobs:` with a `pool:`. Approvals and gates are
the exception — they are properties of the **environment**, and the YAML only names it.

**That asymmetry is the single most testable fact in this challenge**, and the reason Break & fix
Exercise 2 exists: the YAML looks complete and the gate is simply absent.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** classic concepts map to **environment** features in YAML? (Choose three.)

- A. Pre-deployment approvals
- B. Pre-deployment gates
- C. Deployment groups
- D. Task groups
- E. Variable groups
- F. Agent phases

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-37.md`:** the mapping table at lines **155–157**.

```text
| Pre-deployment approvals | Environment approvals and checks |
| Pre-deployment gates     | Environment checks (Invoke REST API, Azure Monitor) |
| Deployment groups        | `environment:` with VM resources |
```

**The environment absorbs three separate classic constructs**, which is the biggest conceptual shift in
the migration. In classic, approvals, gates and target machines were three unrelated things attached to
a release stage. In YAML they are all properties of one object.

**Why the others fail** — D maps to templates (line 158), E to `variables: - group:` (line 159), F to
`jobs:` with a `pool:` (line 160).

</details>

---

## Q18

Which **two** advantages do YAML templates have over classic task groups? (Choose two.)

- A. They can contribute jobs and stages, not only steps
- B. They are version-controlled in Git with review and history
- C. They are edited in a visual designer
- D. They support only one parameter
- E. They cannot be shared across repositories

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-37.md`:** the stage template at lines **419–437** (A) and the file-based model
throughout Task 5 (B).

**A matters for the release migration specifically.** A classic release stage cannot be captured by a
task group — task groups are step-level. The stage template at line 432 is what makes a repeatable
deploy stage possible:

```yaml
stages:
  - stage: Deploy_${{ parameters.environment }}
    jobs:
      - deployment: Deploy
        environment: "contoso-${{ parameters.environment }}"
```

**Why the others fail** — C is the task group's model, D is false (line 351 declares four), E is false
(Challenge 23's `resources: repositories:`).

</details>

---

## Q19

Which **two** are required for a migrated CD pipeline to consume CI artifacts? (Choose two.)

- A. A `resources: pipelines:` entry with an alias and `source:`
- B. `download: <alias>` naming that alias
- C. `download: current`
- D. A `PublishPipelineArtifact` task in the CD pipeline
- E. A shared variable group

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-37.md`:** Break & fix Exercise 1, lines **681–700**.

**The alias links them.** `- pipeline: ci-build` declares it; `download: ci-build` consumes it; and the
files land at `$(Pipeline.Workspace)/ci-build/api-build`.

**Why the others fail**

- **C** — "current" means an artifact **this** run produced, and the CD pipeline builds nothing
- **D** — publishing in the CD pipeline would create a *new* artifact, not fetch the CI one
- **E** — variable groups carry values, not files

</details>

---

## Q20

Which **two** are true about the phased migration approach? (Choose two.)

- A. Phase 1 runs both pipelines with YAML deploying to a shadow environment
- B. Phase 2 disables classic triggers but retains the definition as a fallback
- C. Phase 1 deletes the classic pipeline immediately
- D. Phase 3 happens the day after Phase 2
- E. The classic definition is discarded without archiving

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-37.md`:** lines **547–562**.

**Why the others fail**

- **C** — deleting before validating removes the fallback entirely
- **D** — Phase 3 waits **two weeks with no issues** (line 560). Some failure modes only appear on a
  monthly release or a quarterly job
- **E** — line 561 archives the definition JSON for reference

**The shape of the strategy is worth generalising:** run both, switch the default, keep the old one
warm, and only then remove it. That is the same instinct as slot swaps and revision weights — the
previous version stays alive until you are sure.

</details>

---

## Q21

Which **two** transfer to YAML **without modification**? (Choose two.)

- A. Variable groups
- B. Service connections
- C. Task groups
- D. Release gates
- E. Deployment groups

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-37.md`:** lines **293–297** and **322–327**.

**Both are Library or project-level resources**, not pipeline definitions — so the pipeline changes and
the resource does not.

**The one caveat for both** (line 339): **pipeline permissions**. A resource restricted to specific
pipelines will refuse the new YAML pipeline until it is authorised, and the error looks like a broken
connection rather than a permissions problem.

**Why the others fail** — C must be rewritten as templates, D must be reconfigured as environment
checks, E must be recreated as an environment with VM resources.

</details>

---

## Q22

Which **two** steps register an on-premises VM with a YAML environment? (Choose two.)

- A. Create an environment with a Virtual Machine resource in the UI
- B. Run the generated registration script with `--environment --environmentname`
- C. Add the VM to an agent pool
- D. Create a deployment group
- E. Install the Guest Configuration extension

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-37.md`:** lines **462–478**.

```bash
./config.sh --environment --environmentname "contoso-prod-vms" \
  --agent $HOSTNAME --url https://dev.azure.com/contoso \
  --auth pat --token <PAT>
```

**`--environment` is what distinguishes this from a normal agent registration.** The same agent
package, a different flag, and the machine appears as an environment **resource** rather than as a
pool agent.

**Why the others fail**

- **C** — a pool agent runs jobs; it is not a deployment target
- **D** — the classic construct being replaced
- **E** — Challenge 32's Machine Configuration, unrelated

</details>

---

## Q23

Which **two** describe how approvals differ between classic and YAML? (Choose two.)

- A. Classic configures them per stage in the release definition
- B. YAML configures them on the environment, applying to any pipeline deploying there
- C. YAML configures them in the pipeline file
- D. Classic approvals apply to every pipeline automatically
- E. YAML has no approval mechanism

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-37.md`:** line **744**.

> *"In classic pipelines, approvals and gates are configured per-stage in the release definition. In
> YAML pipelines, they are configured on the environment itself and apply to any pipeline that deploys
> to that environment."*

**The governance consequence is the exam-relevant part.** Because the gate lives on the environment,
someone editing the pipeline cannot remove it — they can only stop referencing the environment, which
is visible in a code review.

**Why the others fail** — C contradicts line 724, D is false (classic approvals belong to one release
definition), E is false.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must migrate a classic release definition that deploys to production with two
required approvers, consumes artifacts from a classic build, and targets on-premises VMs — without
losing the approval gate or disrupting active deployments.

---

## Q24

**Proposed solution:** Create a multi-stage YAML pipeline with a `resources: pipelines:` entry for the
CI build, deployment jobs targeting an environment with VM resources, and configure two required
approvers on that environment in the UI. Run it in parallel against a shadow environment for two
weeks before switching over.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-37.md`:** lines **178–184**, **502–505**, **726–732**, **547–551**.

| Requirement | Mechanism |
|---|---|
| Consume CI artifacts | `resources: pipelines:` alias + `download:` |
| Target on-premises VMs | `environment:` with `resourceType: VirtualMachine` |
| Keep the approval gate | Approvals check configured **on the environment** |
| No disruption | Phase 1 parallel run against a shadow environment |

The approval being configured in the UI is not a gap — it is where it belongs (line 744).

</details>

---

## Q25

**Proposed solution:** Export the classic build to YAML, add `download: current` for the artifacts,
reference `environment: production`, and delete the classic pipelines the same day.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Three failures, and two of them are silent.**

**Artifacts.** `download: current` finds nothing (Break & fix Exercise 1) — a visible failure, at
least.

**Approvals.** Referencing `environment: production` does **not** create the approval. If the
environment has no checks configured, deployments proceed unreviewed — and the pipeline is green, so
nobody notices (Break & fix Exercise 2).

**No fallback.** Deleting the classic pipelines the same day removes the recovery path at exactly the
moment it is most likely to be needed. Phase 3 exists two weeks after Phase 2 for this reason.

**And export only covers builds** (line 55), so the classic **release** definitions were never
converted at all.

</details>

---

## Q26

**Proposed solution:** Create the multi-stage YAML pipeline with the correct `resources: pipelines:`
entry and VM environment, run it in parallel for two weeks, then switch over and retain the classic
definition — but assume the environment inherits the classic definition's approvals.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except one assumption, and it is the one that matters for compliance.

**Approvals do not migrate.** They were properties of the classic **release definition's stage**. The
YAML environment is a **new object** with no checks until someone adds them.

**The parallel run does not catch it either**, which is the cruel part. Phase 1 deploys to a *shadow*
environment (line 551) — which also has no approvals, and is not supposed to. The comparison shows
matching artifacts and matching deployment results, so the migration looks validated.

**The gap surfaces in Phase 2**, on the first real production deployment, when it goes straight through.

**What to add:** an explicit checklist item verifying environment checks exist before Phase 2. The
challenge lists exactly what to configure at lines 726–741.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — the mapping

| # | Statement | Answer |
|---|---|---|
| 1 | A classic release definition becomes a multi-stage YAML pipeline |  |
| 2 | Task groups become YAML templates |  |
| 3 | Deployment groups become agent pools |  |
| 4 | Variable groups are referenced unchanged |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A classic release definition becomes a multi-stage YAML pipeline | **Yes** |
| 2 | Task groups become YAML templates | **Yes** |
| 3 | Deployment groups become agent pools | **No** |
| 4 | Variable groups are referenced unchanged | **Yes** |

**In `challenge-37.md`:** lines **151**, **158**, **157**, **159**.

Row 3 is the near-miss: deployment groups become **environments with VM resources**, not pools. A pool
runs pipeline **jobs**; an environment is a **deployment target** with history, approvals and tags.

</details>

---

## Q28 — artifacts and triggers

| # | Statement | Answer |
|---|---|---|
| 1 | `download: current` works for artifacts from another pipeline |  |
| 2 | `resources: pipelines:` can also trigger the pipeline |  |
| 3 | `source:` is the pipeline's real name; `pipeline:` is the alias |  |
| 4 | Classic linked artifacts automatically; YAML requires a declaration |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `download: current` works for artifacts from another pipeline | **No** |
| 2 | `resources: pipelines:` can also trigger the pipeline | **Yes** |
| 3 | `source:` is the pipeline's real name; `pipeline:` is the alias | **Yes** |
| 4 | Classic linked artifacts automatically; YAML requires a declaration | **Yes** |

**In `challenge-37.md`:** lines **669**, **182–184**, **683–684**, **658–659**.

Row 2 is the double duty: the same block that makes artifacts available also replaces the classic
continuous-deployment trigger.

Row 4 is the theme of the whole migration — implicit UI configuration becomes explicit file
configuration.

</details>

---

## Q29 — approvals and checks

| # | Statement | Answer |
|---|---|---|
| 1 | Environment checks can be defined in YAML |  |
| 2 | Checks apply to every pipeline deploying to that environment |  |
| 3 | Referencing an environment creates its approvals automatically |  |
| 4 | Classic gates map to environment checks |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Environment checks can be defined in YAML | **No** |
| 2 | Checks apply to every pipeline deploying to that environment | **Yes** |
| 3 | Referencing an environment creates its approvals automatically | **No** |
| 4 | Classic gates map to environment checks | **Yes** |

**In `challenge-37.md`:** lines **724**, **744**, **710–712**, **156**.

Row 3 is the silent failure. Line 712's comment says it plainly: *"Environment exists but has no checks
configured."* The YAML is valid, the deployment succeeds, and the gate simply is not there.

Row 2 is the upside: configure once, and every pipeline deploying to production inherits the gate.

</details>

---

## Q30 — migration process

| # | Statement | Answer |
|---|---|---|
| 1 | Phase 1 runs both pipelines against the same production environment |  |
| 2 | Phase 2 retains the classic definition as a fallback |  |
| 3 | The classic definition should be archived as JSON before deletion |  |
| 4 | Export to YAML works for classic release definitions |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Phase 1 runs both pipelines against the same production environment | **No** |
| 2 | Phase 2 retains the classic definition as a fallback | **Yes** |
| 3 | The classic definition should be archived as JSON before deletion | **Yes** |
| 4 | Export to YAML works for classic release definitions | **No** |

**In `challenge-37.md`:** lines **551**, **555–557**, **561**, **55**.

Row 1: the YAML pipeline deploys to a **shadow** environment, so the two never contend for the same
target.

Row 4 is a practical limit worth knowing before planning the work: **8 build definitions can be
exported; the 12 release definitions must be converted by hand** against the mapping table. That is
where the effort actually is.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each classic concept to its YAML equivalent.

| Classic | YAML |
|---|---|
| Build definition |  |
| Release definition |  |
| Release stages |  |
| Environment (classic) |  |
| Artifacts source |  |
| Pre-deployment approvals |  |
| Pre-deployment gates |  |
| Deployment groups |  |
| Task groups |  |
| Variable groups |  |
| Agent phases |  |
| Parallel deployment |  |

**Options:** Environment approvals and checks · Environment checks (Invoke REST API, Azure Monitor) · `environment:` in deployment jobs · `environment:` with VM resources · `jobs:` with different `pool:` settings · Multi-stage YAML pipeline with `stages` · `resources: pipelines:` · `stages:` with `- stage:` blocks · `strategy: parallel:` or matrix · `trigger`, `pool`, `steps` in a YAML file · `variables: - group:` · YAML templates (`template:`)

<details>
<summary>Show answer</summary>

| Classic | YAML |
|---|---|
| Build definition | **`trigger`, `pool`, `steps` in a YAML file** |
| Release definition | **Multi-stage YAML pipeline with `stages`** |
| Release stages | **`stages:` with `- stage:` blocks** |
| Environment (classic) | **`environment:` in deployment jobs** |
| Artifacts source | **`resources: pipelines:`** |
| Pre-deployment approvals | **Environment approvals and checks** |
| Pre-deployment gates | **Environment checks (Invoke REST API, Azure Monitor)** |
| Deployment groups | **`environment:` with VM resources** |
| Task groups | **YAML templates (`template:`)** |
| Variable groups | **`variables: - group:`** |
| Agent phases | **`jobs:` with different `pool:` settings** |
| Parallel deployment | **`strategy: parallel:` or matrix** |

**In `challenge-37.md`:** lines **148–161**.

**This table is the challenge.** Learn it as a whole rather than as twelve facts — most exam questions
give you one side and ask for the other.

**The two that behave differently from the rest:** approvals and gates have **no YAML keyword** (they
live on the environment), and artifact sources need an **explicit declaration** that classic did
implicitly.

</details>

---

## Q32

Arrange the migration phases in order, with their key action.

**Items:** Delete classic and archive the JSON · Disable classic triggers, keep the definition ·
Run both pipelines with YAML on a shadow environment

<details>
<summary>Show answer</summary>

### Answer

| Phase | Weeks | Action |
|---|---|---|
| 1 | 1–2 | Run both; YAML deploys to a **shadow** environment; compare results |
| 2 | 3–4 | YAML becomes official; classic **triggers disabled, definition retained** |
| 3 | 5+ | Delete classic after **two clean weeks**; archive the definition JSON |

**In `challenge-37.md`:** lines **547–562**.

**The retained-but-disabled middle phase is the safety net.** Re-enabling a trigger is one click;
rebuilding a deleted definition from memory is not.

</details>

---

## Q33

Arrange the steps to make a migrated CD pipeline consume CI artifacts.

**Items:** Add `download: <alias>` in the deployment steps · Declare `resources: pipelines:` ·
Set `source:` to the CI pipeline's exact name · Give the resource an alias · Reference
`$(Pipeline.Workspace)/<alias>/<artifact>`

<details>
<summary>Show answer</summary>

### Answer

1. Declare `resources: pipelines:` — line **681**
2. Give the resource an alias (`- pipeline: ci-build`) — line **683**
3. Set `source:` to the CI pipeline's exact name — line **684**
4. Add `download: ci-build` in the deployment steps — line **698**
5. Reference `$(Pipeline.Workspace)/ci-build/api-build` — line **700**

**The alias appears three times** — in the declaration, in the download, and in the path. Getting it
consistent is most of the fix.

</details>

---

## Q34

Match each migration artefact to where it must be configured.

| Item | Configured in |
|---|---|
| Multi-stage structure |  |
| Artifact source and trigger |  |
| Approvals and gates |  |
| VM registration |  |
| Variable group contents |  |
| Service connection |  |

**Options:** Azure DevOps UI — on the environment · **Library** — unchanged · **On the VM** (`config.sh --environment`) · **Project settings** — unchanged, plus pipeline permissions · YAML file · **YAML file** (`resources: pipelines:`)

<details>
<summary>Show answer</summary>

| Item | Configured in |
|---|---|
| Multi-stage structure | **YAML file** |
| Artifact source and trigger | **YAML file** (`resources: pipelines:`) |
| Approvals and gates | **Azure DevOps UI — on the environment** |
| VM registration | **On the VM** (`config.sh --environment`) |
| Variable group contents | **Library** — unchanged |
| Service connection | **Project settings** — unchanged, plus pipeline permissions |

**In `challenge-37.md`:** lines **175–186**, **724**, **476–478**, **293–297**, **322–339**.

**Three different places, and that is the migration's real complexity.** A pull request showing a
complete-looking YAML pipeline can still be missing the approval, the VM registration or the pipeline
permission — none of which appear in the diff.

</details>

---

## Q35

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| CD pipeline finds no artifact |  |
| Production deploys with no approval |  |
| Task input references an unknown service connection |  |
| VMs never receive a deployment |  |
| Only 8 of 20 pipelines could be exported |  |

**Options:** `download: current` with no `resources: pipelines:` · Environment exists but has no checks configured · Export to YAML covers builds, not classic releases · Machines not registered with `--environment`, or wrong `tags:` · Pipeline lacks permission on the connection

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| CD pipeline finds no artifact | **`download: current` with no `resources: pipelines:`** |
| Production deploys with no approval | **Environment exists but has no checks configured** |
| Task input references an unknown service connection | **Pipeline lacks permission on the connection** |
| VMs never receive a deployment | **Machines not registered with `--environment`, or wrong `tags:`** |
| Only 8 of 20 pipelines could be exported | **Export to YAML covers builds, not classic releases** |

**In `challenge-37.md`:** lines **669**, **712**, **339**, **476–505**, **55**.

**The second row is the dangerous one** because it produces a **successful** deployment. Every other
symptom here is a visible failure.

</details>

---

# Section F — Hot area

---

## Q36

```yaml
[BLANK 1]:
  pipelines:
    - [BLANK 2]: ci-build
      [BLANK 3]: "Contoso API CI (YAML)"
      trigger:
        branches:
          include: [main]
```

- **BLANK 1:** `resources` / `variables` / `extends` / `imports`
- **BLANK 2:** `pipeline` / `name` / `alias` / `id`
- **BLANK 3:** `source` / `pipeline` / `definition` / `path`

<details>
<summary>Show answer</summary>

### Answer: `resources`, `pipeline`, `source`

**In `challenge-37.md`:** lines **681–684**.

`pipeline:` is the **alias** used everywhere else in the file; `source:` is the pipeline's **real
name** in Azure DevOps. Swapping them gives a resource-not-found error at compile time.

</details>

---

## Q37

```yaml
      - deployment: DeployToVMs
        environment:
          name: contoso-prod-vms
          [BLANK 1]: VirtualMachine
          [BLANK 2]: "web"
        strategy:
          [BLANK 3]:
            maxParallel: 2
```

- **BLANK 1:** `resourceType` / `type` / `kind` / `target`
- **BLANK 2:** `tags` / `labels` / `filter` / `select`
- **BLANK 3:** `rolling` / `runOnce` / `canary` / `matrix`

<details>
<summary>Show answer</summary>

### Answer: `resourceType`, `tags`, `rolling`

**In `challenge-37.md`:** lines **504–507**.

`tags: "web"` is what replaces classic deployment group tags — register 20 VMs and target only the
subset a given stage needs. `rolling` with `maxParallel: 2` updates two machines at a time.

</details>

---

## Q38

```yaml
parameters:
  - name: publishArtifact
    type: [BLANK 1]
    default: true

steps:
  - ${{ [BLANK 2] eq(parameters.publishArtifact, [BLANK 3]) }}:
    - task: PublishPipelineArtifact@1
```

- **BLANK 1:** `boolean` / `bool` / `binary` / `flag`
- **BLANK 2:** `if` / `when` / `condition` / `case`
- **BLANK 3:** `true` / `'true'` / `"true"` / `1`

<details>
<summary>Show answer</summary>

### Answer: `boolean`, `if`, `true`

**In `challenge-37.md`:** lines **362** and **389**.

All three are Challenge 20's parameter traps recurring: `bool` is invalid, `condition:` is a **runtime**
step key rather than compile-time insertion, and comparing a boolean to the string `'true'` silently
never matches.

</details>

---

## Q39

```yaml
stages:
  - stage: Dev
    [BLANK 1]:
      - group: contoso-api-dev
    jobs:
      - job: Deploy
        steps:
          - script: echo "DB_HOST=[BLANK 2]"
```

- **BLANK 1:** `variables` / `parameters` / `env` / `settings`
- **BLANK 2:** `$(DB_HOST)` / `${{ DB_HOST }}` / `$[DB_HOST]` / `%DB_HOST%`

<details>
<summary>Show answer</summary>

### Answer: `variables`, `$(DB_HOST)`

**In `challenge-37.md`:** lines **304–309**.

Stage-scoped variable groups let the **same expression** resolve differently per stage — `contoso-api-dev`
here, `contoso-api-production` at line 313. `$( )` is macro syntax, correct inside a task input or
script.

</details>

---

## Q40

```bash
./config.sh --[BLANK 1] --[BLANK 2] "contoso-prod-vms" \
  --agent $HOSTNAME --url https://dev.azure.com/contoso \
  --auth pat --token <PAT>
```

- **BLANK 1:** `environment` / `deploymentgroup` / `pool` / `agent`
- **BLANK 2:** `environmentname` / `groupname` / `poolname` / `targetname`

<details>
<summary>Show answer</summary>

### Answer: `environment`, `environmentname`

**In `challenge-37.md`:** lines **476–478**.

`--deploymentgroup` is the **classic** flag being replaced; `--pool` registers a normal build agent
rather than a deployment target.

</details>

---

## Q41

```bash
# Archive before deletion
az pipelines [BLANK 1] --id 42 --output json > archived/contoso-api-ci-classic.json

# Phase 3
az pipelines [BLANK 2] --id 42 --yes
```

- **BLANK 1:** `show` / `list` / `export` / `get`
- **BLANK 2:** `delete` / `disable` / `archive` / `remove`

<details>
<summary>Show answer</summary>

### Answer: `show`, `delete`

**In `challenge-37.md`:** lines **567–571** and **589–593**.

`az pipelines export` does not exist — the archive is simply `show --output json` redirected to a file.
Deletion is irreversible, which is why the archive comes first and Phase 3 waits two weeks.

</details>

---

# Section G — Case study

## Case study: Contoso classic-to-YAML migration

### Background

Contoso has **20 classic pipelines** built over four years:

- **8 classic build definitions** (CI)
- **12 classic release definitions** (CD) with multiple stages and gates
- Shared **task groups** used across pipelines
- **Variable groups** with environment-specific secrets
- **Deployment groups** for on-premises VM deployments

### Requirements

**Fidelity**

- Production approvals must survive the migration
- On-premises VM deployments must continue, targeting the same machine subsets
- CD pipelines must consume artifacts from their CI pipeline

**Reuse**

- Shared task groups must become reusable across pipelines
- Environment-specific secrets must not be duplicated

**Safety**

- No disruption to active deployments during migration
- A fallback must exist if the YAML pipeline misbehaves
- The classic definitions must be recoverable for reference

---

## Q42

Which **two** are needed for the 12 classic release definitions? (Choose two.)

- A. Multi-stage YAML pipelines with `stages:`
- B. Manual conversion — export to YAML does not cover releases
- C. Export to YAML from the release editor
- D. One YAML file per release stage
- E. Conversion to task groups first

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-37.md`:** the mapping at line **151** and the export caveat at line **55**.

**B is the planning reality.** The 8 build definitions can be exported; the 12 releases must be
rewritten by hand against the mapping table — which is where the effort actually sits, and it is worth
knowing before estimating the work.

**Why the others fail**

- **C** — export exists for classic **builds**
- **D** — stages belong in one pipeline file; splitting them loses the ordering and the shared
  resources
- **E** — task groups are step-level and cannot express a release stage

</details>

---

## Q43

Which configuration preserves production approvals?

- A. Configure an Approvals check on the `production` environment in the UI
- B. Add an `approvals:` block to the YAML
- C. Add a branch policy on `main`
- D. Set `condition: succeeded()` on the production stage

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** lines **726–732**.

**Why the others fail**

- **B** — no such YAML key exists (line 724)
- **C** — branch policies gate **merging**, not deploying. The same boundary as Challenge 24 Q43
- **D** — automatic logic. A condition cannot ask a human

**And the migration hazard:** approvals do **not** carry across. The environment is a new object with
no checks until someone adds them, and the pipeline works perfectly without them.

</details>

---

## Q44

Which configuration keeps the on-premises VM deployments working with the same machine subsets?

- A. An environment with `resourceType: VirtualMachine` and `tags:` matching the old deployment group
  tags
- B. A self-hosted agent pool
- C. A `pool: name:` targeting the VMs
- D. `strategy: matrix` over VM names

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** lines **502–505**.

```yaml
        environment:
          name: contoso-prod-vms
          resourceType: VirtualMachine
          tags: "web"
```

**`tags:` is the direct replacement for deployment group tags**, so the same subsetting logic survives.

**Why the others fail**

- **B** and **C** — a pool runs **jobs**; an environment is a **deployment target** with history,
  approvals and per-machine tracking
- **D** — a matrix is a regular-job strategy and is not available on deployment jobs

</details>

---

## Q45

Which **two** meet the reuse requirements? (Choose two.)

- A. Convert task groups to YAML templates with typed parameters
- B. Reference existing variable groups with `variables: - group:`
- C. Copy the shared steps into each pipeline
- D. Recreate the variable groups as inline YAML variables
- E. Publish task groups to a package feed

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-37.md`:** lines **346–414** and **293–297**.

**B specifically satisfies "must not be duplicated"** — the groups already exist in the Library with
their Key Vault links and secrets. Only the reference moves.

**Why the others fail**

- **C** — duplication is what templates exist to remove
- **D** — inlining secrets into YAML puts them in source control. The exact failure Challenge 31's
  `secure-parameter-default` lint rule guards against
- **E** — task groups are not packages

</details>

---

## Q46

Which **two** meet the safety requirements? (Choose two.)

- A. Phase 1 parallel run with YAML deploying to a shadow environment
- B. Phase 2 disabling classic triggers while retaining the definition
- C. Deleting classic pipelines as each YAML pipeline is created
- D. Running both pipelines against production simultaneously
- E. Migrating all 20 pipelines in one weekend

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-37.md`:** lines **547–557**.

**Why the others fail**

- **C** — removes the fallback the requirements ask for
- **D** — two pipelines contending for the same environment race and overwrite each other, and a
  failure cannot be attributed to either
- **E** — a big-bang migration of 20 pipelines removes any chance of learning from the first few

</details>

---

## Q47

Three months in, 15 pipelines are migrated. A team reports that their YAML pipeline deploys
successfully but a downstream integration test environment never receives the new build, though the
classic pipeline used to update it.

What is the most likely cause?

- A. The classic release had an additional stage or artifact consumer that was not migrated
- B. The environment is missing approvals
- C. The variable group is not linked
- D. The service connection expired

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** the mapping table at lines **151–154**, and the artifact-source row
specifically.

**Reason it through.** The deployment "succeeds", so authentication, permissions and the artifact
download all work. What is missing is a **consumer** — a stage that used to run after production, or a
separate pipeline that consumed the classic release's artifact.

**Classic release definitions frequently had more stages than anyone remembered**, and a downstream
pipeline may have been linked to the classic build as an **artifact source** — an implicit link that
does not exist until someone writes a `resources: pipelines:` block pointing at the *new* pipeline.

**Where to look:** the archived classic definition JSON (line 567). That is precisely why Phase 3
archives it — it is the record of every stage and every consumer.

**Why the others fail** — B blocks with a pending approval, C and D produce visible failures.

</details>

---

## Q48

Migration is complete and the classic definitions are deleted. Six months later an auditor asks why a
specific production deployment step existed and who approved a release from last year.

What should Contoso be able to produce, and what does this illustrate?

- A. The archived classic definition JSON for the step, and Azure DevOps deployment history for the
  approval
- B. Nothing — the classic pipelines are gone
- C. The YAML file only
- D. A screenshot from before migration

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-37.md`:** the archive at lines **561** and **566–571**.

**Two different records answer two different questions.**

**"Why did this step exist?"** — the archived JSON. The YAML shows what the pipeline does *now*; the
JSON shows what the classic definition did, including steps that were deliberately dropped during
migration and steps whose purpose nobody could explain.

**"Who approved it?"** — the **environment's deployment history**, which records approvals against the
environment rather than against the pipeline. That history survives the pipeline definition, which is
another consequence of approvals living on the environment (line 744).

**What it illustrates:** migration is not only about making the new thing work. The old definition is
**evidence**, and an audit trail spans both. Line 561's "archive classic definition JSON for reference"
looks like tidiness and is actually compliance.

**Why the others fail**

- **B** — true only if the archive step was skipped, which is the failure the challenge warns against
- **C** — the YAML answers what happens now, not what happened then
- **D** — a screenshot is not an auditable record

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Looking for a `gates:` YAML keyword** | Q1, Q16, Q29 | Gates become **environment checks**, configured in the UI |
| **Approvals assumed to migrate** | Q3, Q26, Q43 | The environment is a new object with no checks |
| **`download: current` for another pipeline's artifact** | Q2, Q19, Q25 | Declare `resources: pipelines:` and download by alias |
| **Alias confused with `source:`** | Q15, Q36 | `pipeline:` is the alias; `source:` is the real name |
| **Deployment groups mapped to agent pools** | Q5, Q27, Q44 | Environment with `resourceType: VirtualMachine` |
| **Export assumed to cover releases** | Q10, Q30, Q42 | Builds only. Releases are converted by hand |
| **Deleting classic before validating** | Q20, Q25, Q46 | Phase 2 retains it disabled; Phase 3 waits two weeks |
| **Skipping the JSON archive** | Q13, Q48 | It is the audit record of what the pipeline used to do |
| **Both pipelines against production** | Q12, Q46 | Phase 1 uses a shadow environment |
| **Pipeline permissions forgotten** | Q8, Q21, Q35 | Connections and environments must authorise the new pipeline |
| **Secrets inlined during migration** | Q45 | Reference the existing variable group |
| **`type: bool` and `'true'`** | Q38 | `boolean`, compared to `true` |

---

# The mapping table to memorise

**In `challenge-37.md`:** lines **148–161**. This is the highest-value thing in the challenge.

```text
Classic concept              YAML equivalent
---------------------------  ----------------------------------------------
Build definition             trigger, pool, steps in a YAML file
Release definition           multi-stage YAML pipeline with stages
Release stages               stages: with - stage: blocks
Environment (classic)        environment: in deployment jobs
Artifacts source             resources: pipelines:   <- explicit now
Pre-deployment approvals     environment approvals and checks  <- UI, not YAML
Pre-deployment gates         environment checks (Invoke REST API, Azure Monitor)
Deployment groups            environment: with VM resources
Task groups                  YAML templates (template:)
Variable groups              variables: - group:     <- unchanged
Agent phases                 jobs: with different pool: settings
Parallel deployment          strategy: parallel: or matrix
```

```yaml
# Artifact consumption  (lines 681-700)
resources:
  pipelines:
    - pipeline: ci-build                    # alias
      source: "Contoso API CI (YAML)"       # real name
      trigger:
        branches:
          include: [main]
...
                - download: ci-build        # by alias
                - script: ls $(Pipeline.Workspace)/ci-build/api-build

# VM environment  (lines 502-508)
        environment:
          name: contoso-prod-vms
          resourceType: VirtualMachine
          tags: "web"
        strategy:
          rolling:
            maxParallel: 2
```

```text
# Phased migration  (lines 547-562)
Phase 1  wk 1-2  both run; YAML -> SHADOW environment; compare results
Phase 2  wk 3-4  YAML official; classic triggers disabled, definition RETAINED
Phase 3  wk 5+   delete classic after 2 clean weeks; ARCHIVE the definition JSON

# Configured outside the YAML file
Approvals and checks   -> Azure DevOps UI, on the environment
VM registration        -> config.sh --environment --environmentname
Pipeline permissions   -> service connections and environments
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 37 is exam-ready. Move to Challenge 38 |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Re-read Task 2's mapping table and Break & fix, then retake |
| Below 30 | Redo the challenge, writing the twelve-row mapping table from memory first |

Record your result in `AZ-400-Learning-Log.md` under Challenge 37.

:::danger The one thing

**Approvals and gates do not live in the YAML — and they do not migrate.**

Everything else in the mapping table has a keyword. Approvals and checks belong to the **environment**,
are configured in the UI, and a newly created environment has none. The pipeline will deploy happily
without them.

:::
