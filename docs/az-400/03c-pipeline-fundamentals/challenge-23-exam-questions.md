---
sidebar_position: 95
title: "Challenge 23: exam questions"
---

# Challenge 23 — AZ-400 exam questions

**48 questions** built only from what Challenge 23 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-23.md`**.

| Section | Shape | Questions |
|---|---|---|
| A | Single answer | 1–16 |
| B | Multiple answer (choose two / three) | 17–23 |
| C | Repeated scenario — "Does this meet the goal?" | 24–26 |
| D | Yes/No statement grid | 27–30 |
| E | Drag and drop | 31–35 |
| F | Hot area — complete the YAML | 36–41 |
| G | Case study | 42–48 |

The **trap index**, the **blocks to memorise**, and **scoring** are at the end.

:::tip The one distinction this challenge is built on

**Composite action = steps, runs inside the caller's job, same runner.**
**Reusable workflow = jobs, runs as its own job, own runner.**

Say it before you start. Half of Section A turns on it.

:::

---

# Section A — Single answer

---

## Q1

A reusable workflow call fails because the called workflow cannot read `AZURE_CLIENT_ID`.

What is missing from the caller?

- A. A `secrets:` block or `secrets: inherit`
- B. A `permissions:` block granting `secrets: read`
- C. An `env:` block defining the secret
- D. A `needs:` declaration on the calling job

### Answer: A

**In `challenge-23.md`:** Break & fix Exercise 1, lines **824–851**.

**Broken:**

```yaml
jobs:
  deploy:
    uses: contoso/.github/.github/workflows/deploy.yml@main
    with:
      environment: staging
    # no secrets passed
```

**Fixed — two valid forms:**

```yaml
    secrets: inherit                                    # pass everything the caller has
# or explicitly (lines 242-245):
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
```

**Secrets do not flow automatically.** A reusable workflow runs in its own job, and secrets must be
handed across the boundary deliberately.

**Why the others fail**

- **B** — there is no `secrets: read` permission. `permissions` scopes `GITHUB_TOKEN`, not secrets
- **C** — `env` holds non-secret values and does not cross the boundary either
- **D** — `needs` sets ordering between jobs, not data flow into a reusable workflow

**Which form to prefer:** `inherit` is convenient; explicit passing is least-privilege. If a question
mentions minimal access, pick explicit.

---

## Q2

Where does a reusable workflow execute relative to the job that calls it?

- A. Inside the calling job, on the same runner
- B. As its own job or jobs, on their own runners
- C. On the caller's runner but in a separate container
- D. On a GitHub-hosted runner only

### Answer: B

**In `challenge-23.md`:** the reusable workflow defines **six jobs** (lines **90–219**), and the
caller invokes it with one `uses:` at **job** level (lines **234–236**).

```yaml
jobs:
  ci-cd:
    uses: contoso/.github/.github/workflows/reusable-node-ci-cd.yml@main
```

That single line expands into `build`, `test`, `docker`, `deploy-staging` and `deploy-production` —
each on its own runner.

**Why the others fail**

- **A** — that describes a **composite action** (line 283)
- **C** — container jobs are a separate feature
- **D** — a reusable workflow's jobs can specify `runs-on: [self-hosted, ...]` like any other

**The consequence that gets tested:** because each job gets a fresh runner, files do not carry over.
That is why the reusable workflow uploads an artifact at line 107 rather than assuming the next job
can see `dist/`.

---

## Q3

Which trigger makes a workflow callable by another workflow?

- A. `on: workflow_run`
- B. `on: workflow_call`
- C. `on: workflow_dispatch`
- D. `on: repository_dispatch`

### Answer: B

**In `challenge-23.md`:** line **43**.

```yaml
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: "20"
```

**Why the others fail**

- **A** — `workflow_run` fires **after another workflow finishes**. The relationship is reversed:
  the upstream does not know you exist
- **C** — `workflow_dispatch` is the manual button
- **D** — `repository_dispatch` is an external API trigger

**Keep `workflow_call` and `workflow_run` apart** — one is *called by*, the other is *triggered
after*. That pair is a reliable exam question.

---

## Q4

A composite action's `run` step fails validation. What is missing?

- A. `id`
- B. `shell`
- C. `working-directory`
- D. `continue-on-error`

### Answer: B

**In `challenge-23.md`:** lines **291**, **295**, **304**, **315** — every `run` step declares it.

```yaml
    - name: Restore dependencies
      shell: bash
      run: dotnet restore ${{ inputs.project-path }}
```

Mandatory in composite actions, optional in workflows.

**Why the others fail**

- **A** — `id` is only needed when another step reads this one's output, as at line 302
- **C** — optional
- **D** — failure handling, not validation

---

## Q5

How does a composite action expose a value to the calling workflow?

- A. By writing to `$GITHUB_ENV`
- B. Through an `outputs` block mapping to a step output
- C. Through an `env` block at action level
- D. By uploading an artifact

### Answer: B

**In `challenge-23.md`:** lines **274–280**, produced at **322**, consumed at **343**.

```yaml
outputs:
  artifact-path:
    description: "Path to the published output"
    value: ${{ steps.publish.outputs.path }}      # line 277
```

```yaml
    - name: Publish
      id: publish
      shell: bash
      run: echo "path=$OUTPUT_PATH" >> $GITHUB_OUTPUT   # line 322
```

```yaml
      - uses: actions/upload-artifact@v4
        with:
          path: ${{ steps.build.outputs['artifact-path'] }}    # line 343
```

Note the chain: step writes to `$GITHUB_OUTPUT` → action `outputs` maps it with `value:` → caller
reads `steps.<id>.outputs.<name>`. **Same three-part shape as job outputs in Challenge 19.**

**Why the others fail**

- **A** — `$GITHUB_ENV` sets env vars for later steps. It does not populate action outputs
- **C** — `env` is input configuration, not output
- **D** — artifacts move files between jobs, not values to a caller

---

## Q6

A reusable workflow declares an output. Where does its `value` come from?

- A. A step in the calling workflow
- B. A job output inside the reusable workflow
- C. An environment variable
- D. A repository variable

### Answer: B

**In `challenge-23.md`:** lines **79–85**.

```yaml
    outputs:
      image-tag:
        description: "The published image tag"
        value: ${{ jobs.docker.outputs['image-tag'] }}     # line 82
      test-passed:
        value: ${{ jobs.test.outputs.result }}             # line 85
```

Note it is the **`jobs`** context, not `steps` — because a reusable workflow is made of jobs. Compare
with a composite action (Q5), which uses `steps` because it is made of steps. That difference follows
directly from what each one is.

**The full chain here:** step (line 146) → job output (line 145) → workflow output (line 82) →
caller.

---

## Q7

Which Azure Pipelines template type can contribute **stages**?

- A. A template whose root key is `steps:`
- B. A template whose root key is `jobs:`
- C. A template whose root key is `stages:`
- D. Any template, depending on where it is referenced

### Answer: C

**In `challenge-23.md`:** step template at **353**, job template at **427**, stage template at
**469**, and all three used at **574**, **602**, **612**.

```yaml
        steps:
          - template: templates/steps/dotnet-build.yml@templates    # line 574
      - template: templates/jobs/docker-build-push.yml@templates    # line 602
  - template: templates/stages/deploy-container-app.yml@templates   # line 612
```

**Look at the indentation.** The stage template is inserted at the top level of `stages:`; the job
template under a stage's `jobs:`; the step template under a job's `steps:`.

**Why D fails:** the root key **fixes** what the template can contribute. A `steps:` template cannot
be inserted where stages are expected — you get a validation error.

**The four types:** steps, jobs, stages, variables.

---

## Q8

A template parameter is declared `type: object` and the caller passes `"staging,production"`.

What happens?

- A. The string is split on commas automatically
- B. The pipeline fails with an unexpected-value error
- C. The parameter falls back to its default
- D. The string is treated as a single-item list

### Answer: B

**In `challenge-23.md`:** Break & fix Exercise 2, lines **859–884**.

```yaml
parameters:
  - name: environments
    type: object
    default: []

# WRONG
    environments: "staging,production"

# RIGHT
    environments:
      - staging
      - production
```

Template parameters are **type-checked at compile time**, before any agent is allocated.

**Why the others fail**

- **A** — no automatic splitting. YAML does not guess
- **C** — a default only applies when the parameter is **omitted**, not when it is wrong
- **D** — nothing coerces a string into a list

**Why `type: object` exists** — it is what makes `each` iteration possible (line 447):

```yaml
          tags:
            ${{ each tag in parameters.tags }}:
              ${{ tag }}
```

---

## Q9

A pipeline fails to resolve `steps/build.yml@templates`. The resource declares `ref: main`.

What is wrong?

- A. The alias must match the repository name
- B. `ref` must be a full ref path such as `refs/heads/main`
- C. `type` must be `github`
- D. The template path must be absolute

### Answer: B

**In `challenge-23.md`:** Break & fix Exercise 3, lines **890–915**.

```yaml
      ref: main              # line 896 - WRONG
      ref: refs/heads/main   # line 914 - correct
```

Also seen at line **558** (`refs/heads/main`) and line **790** (`refs/tags/v2.1.0`).

**Why the others fail**

- **A** — the alias is arbitrary. Line 555 uses `templates` for a repo actually named
  `ContosoPlatform/pipeline-templates`
- **C** — `type: git` means Azure Repos and is correct here. `github` would need an `endpoint`
- **D** — template paths are relative to the referenced repository's root

**Ref forms to know:** `refs/heads/<branch>`, `refs/tags/<tag>`. Line 790 pins a **tag**, which is
the production-safe choice.

---

## Q10

Which reference style makes a shared template safe against upstream changes?

- A. `@main`
- B. `@v2.1.0` or `refs/tags/v2.1.0`
- C. `@HEAD`
- D. `@latest`

### Answer: B

**In `challenge-23.md`:** line **777** with a comment saying exactly this, and line **790**.

```yaml
    uses: contoso/.github/.github/workflows/reusable-node-ci-cd.yml@v2
    # Pin to a tag/release for stability
```

```yaml
      ref: refs/tags/v2.1.0  # Pin to specific version
```

**Why the others fail**

- **A** — `@main` is a **moving target**. A change to the shared template instantly alters every
  consuming pipeline. That is fine while iterating and dangerous in production
- **C** — `HEAD` is not a valid pin
- **D** — there is no `latest` concept; that is container registry vocabulary

**The trade-off to state:** pinning gives stability but means security fixes need a deliberate bump.
Pin production, float development.

---

## Q11

What does `${{ each tag in parameters.tags }}` do?

- A. Runs the step once per tag at runtime
- B. Expands the YAML at compile time, one entry per list item
- C. Creates a matrix of parallel jobs
- D. Concatenates the tags into a string

### Answer: B

**In `challenge-23.md`:** lines **446–448** and **456–458**.

```yaml
          tags:
            ${{ each tag in parameters.tags }}:
              ${{ tag }}
```

With `tags: [123, main, latest]` (lines 607–610), the parser produces three literal entries **before
the pipeline starts**. It is YAML generation, not a loop.

**Why the others fail**

- **A** — nothing runs at runtime. By the time an agent exists, the expansion has already happened
- **C** — a matrix creates parallel jobs. `each` creates YAML in place
- **D** — no concatenation

**The pairing:** `${{ if }}` includes or excludes YAML; `${{ each }}` repeats YAML. Both compile-time,
both from Challenge 20's expression rules.

---

## Q12

In classic Azure DevOps, what is the equivalent of a YAML step template?

- A. A task group
- B. A variable group
- C. A deployment group
- D. A service connection

### Answer: A

**In `challenge-23.md`:** the comparison table, lines **697–705**.

| Feature | Task groups (classic) | YAML templates |
|---|---|---|
| Interface | Visual designer | Code |
| Sharing | Within project or organization | Across repos via resources |
| Reuse scope | **Step-level only** | **Step, job, or stage** |

**Why the others fail**

- **B** — variable groups hold values, and they exist in YAML too (line 677)
- **C** — deployment groups are sets of target machines in classic release pipelines. They map to
  **environments** in YAML
- **D** — a service connection stores credentials

**The row that generates questions:** task groups are **step-level only**. If a question needs shared
jobs or stages, task groups cannot do it — which is a reason to migrate, and Challenge 37 does
exactly that.

---

## Q13

Which construct shares six setup **steps** inside an existing job without adding a runner?

- A. A reusable workflow
- B. A composite action
- C. A starter workflow
- D. A job template

### Answer: B

**In `challenge-23.md`:** the composite action at lines **253–323**, used at lines **333–338**.

```yaml
      - name: Build and test
        id: build
        uses: contoso/.github/actions/dotnet-build-test@main
        with:
          project-path: src/Contoso.Api/Contoso.Api.csproj
```

`uses:` under `steps:` = composite action, same runner, same workspace.

**Why the others fail**

- **A** — always its own job on its own runner (Q2)
- **C** — a starter workflow is copied once when creating a new workflow. Later edits do not
  propagate
- **D** — job templates are Azure Pipelines, and they contribute jobs, not steps

---

## Q14

Where must an organization-wide reusable workflow live to be referenced as
`contoso/.github/.github/workflows/x.yml@main`?

- A. In `.github/workflows/` of a repository named `.github`
- B. In `workflows/` at the root of any repository
- C. In `.github/actions/` of the consuming repository
- D. In a repository named `workflows`

### Answer: A

**In `challenge-23.md`:** lines **754–769**.

```text
contoso/.github/
  .github/
    workflows/
      reusable-node-ci-cd.yml
  actions/
    dotnet-build-test/
      action.yml
```

Read the reference path in three parts: `contoso/.github` is the **repository**,
`.github/workflows/x.yml` is the **path inside it**, `@main` is the **ref**. The doubled `.github`
looks like a typo and is not.

**Note the asymmetry (line 762):** composite actions live in `actions/` at the **repo root**, not
under `.github/`. So the reference is `contoso/.github/actions/dotnet-build-test@main` (line 335) —
only one `.github`.

---

## Q15

A variable group is declared at stage level. Which stage can read it?

- A. Every stage in the pipeline
- B. Only the stage where it is declared
- C. Only stages that declare `dependsOn`
- D. Only deployment jobs

### Answer: B

**In `challenge-23.md`:** lines **676–691**.

```yaml
variables:
  - group: contoso-common       # line 677 - pipeline-wide

stages:
  - stage: DeployStaging
    variables:
      - group: contoso-staging  # line 684 - THIS stage only
    jobs:
      - job: Deploy
        steps:
          - script: |
              echo "Registry: $(Registry)"            # from contoso-common
              echo "Resource Group: $(ResourceGroup)" # from contoso-staging
```

Scope is pipeline → stage → job, exactly as in Challenge 20.

**Why this design is the answer to "different value per environment":** lines 651–669 create
`contoso-staging` and `contoso-production` with the **same variable names** and different values.
Same `$(ResourceGroup)` expression, different result per stage.

---

## Q16

Which input type lets a template parameter restrict the caller to a fixed set of values?

- A. `type: string` with a `values` list
- B. `type: choice` with `options`
- C. `type: enum`
- D. `type: object` with `allowed`

### Answer: A

**In `challenge-23.md`:** lines **712–718**.

```yaml
parameters:
  - name: scanType
    type: string
    default: "full"
    values:
      - quick
      - full
```

Passing anything else fails at compile time.

**Why the others fail**

- **B** — `type: choice` with `options` is **GitHub Actions** `workflow_dispatch` syntax (line 692 of
  challenge-22). The exam offers it here on purpose
- **C** — no `enum` type exists
- **D** — no `allowed` key exists

---

# Section B — Multiple answer

---

## Q17

Which **three** describe a composite action? (Choose three.)

- A. Every `run` step requires a `shell` value
- B. It runs inside the calling job on the same runner
- C. Its outputs map to step outputs with `value:`
- D. It is invoked with `uses:` at job level
- E. It can define its own `jobs`
- F. It requires an `on:` trigger

### Answer: A, B, C

**In `challenge-23.md`:** lines **291** (A), **283** (B), **274–280** (C).

```yaml
outputs:
  artifact-path:
    value: ${{ steps.publish.outputs.path }}    # C
runs:
  using: "composite"                            # B
  steps:
    - shell: bash                               # A
      run: dotnet restore ...
```

**Why the others fail**

- **D** — that is a **reusable workflow** (line 235). A composite action is used under `steps:`
- **E** — an action file has `name`, `description`, `inputs`, `outputs`, `runs`. There is no `jobs`
- **F** — `on:` belongs to workflows

---

## Q18

Which **two** are required when calling a reusable workflow that declares required inputs and
secrets? (Choose two.)

- A. A `with:` block supplying every required input
- B. A `secrets:` block or `secrets: inherit`
- C. A `needs:` declaration
- D. A `runs-on:` value on the calling job
- E. A `permissions:` block on the calling job

### Answer: A, B

**In `challenge-23.md`:** lines **234–245**.

```yaml
jobs:
  ci-cd:
    uses: contoso/.github/.github/workflows/reusable-node-ci-cd.yml@main
    with:
      image-name: order-service          # A - required (line 57)
      azure-app-name: contoso-orders     # A - required (line 61)
    secrets:                             # B - required (lines 68-74)
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
```

**Why the others fail**

- **C** — only if you need ordering against another job
- **D** — **notice what is missing from the caller.** A job that calls a reusable workflow has **no**
  `runs-on`, because it does not run steps itself. Adding one is an error
- **E** — permissions are declared inside the reusable workflow's own jobs (lines 142–144, 184–186)

---

## Q19

Which **two** correctly pin a shared template to a stable version? (Choose two.)

- A. `uses: contoso/.github/.github/workflows/ci.yml@v2`
- B. `ref: refs/tags/v2.1.0`
- C. `uses: contoso/.github/.github/workflows/ci.yml@main`
- D. `ref: main`
- E. `ref: latest`

### Answer: A, B

**In `challenge-23.md`:** lines **777** and **790**.

**Why the others fail**

- **C** — a branch moves. Every consumer changes the moment someone pushes
- **D** — additionally **invalid syntax** in Azure Pipelines (Break & fix Exercise 3, line 896). It
  needs the full ref path
- **E** — not a thing

---

## Q20

Which **two** Azure Pipelines template levels can a stage template control that a step template
cannot? (Choose two.)

- A. `dependsOn` between stages
- B. `condition` on a stage
- C. Which tasks run in a job
- D. The agent pool for a job
- E. Task inputs

### Answer: A, B

**In `challenge-23.md`:** lines **612–631**.

```yaml
  - template: templates/stages/deploy-container-app.yml@templates
    parameters:
      environment: "production"
      dependsOn: [Deploy_staging]                                          # A
      condition: "and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))"  # B
```

**This is a pattern worth noticing:** `dependsOn` and `condition` are passed as **parameters** so the
same stage template can be reused for staging and production with different placement in the graph
(compare lines 612–620 with 622–631).

**Why the others fail**

- **C**, **D**, **E** — all within reach of a step or job template. They do not require stage level

---

## Q21

Which **two** statements about task groups are correct? (Choose two.)

- A. They are a classic pipeline feature
- B. They can only encapsulate steps
- C. They can encapsulate whole stages
- D. They are version-controlled in Git
- E. They work in YAML pipelines

### Answer: A, B

**In `challenge-23.md`:** lines **697–705**.

| Feature | Task groups | YAML templates |
|---|---|---|
| Interface | Visual designer | Code |
| Version control | Built-in drafts and versions | **Git-based** |
| Reuse scope | **Step-level only** | Step, job, or stage |

**Why the others fail**

- **C** — step level only, which is the main limitation
- **D** — task groups have their own versioning **inside Azure DevOps**, not in Git. Templates are
  files, so they get code review, history and pinning for free
- **E** — task groups do not work in YAML. This is why Challenge 37 migrates them to templates

---

## Q22

Which **two** are valid ways for a caller to supply secrets to a reusable workflow? (Choose two.)

- A. `secrets: inherit`
- B. An explicit `secrets:` mapping listing each secret
- C. `env:` with the secret values
- D. `with:` containing the secret values
- E. Repository variables

### Answer: A, B

**In `challenge-23.md`:** lines **846–850**.

```yaml
    secrets: inherit                                    # A
    # or
    secrets:                                            # B
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
```

**Why the others fail**

- **C** — `env` does not cross the boundary, and it would not be masked
- **D** — **the dangerous one.** `with:` inputs appear in logs and in the run's input display. Putting
  a secret there leaks it. Inputs and secrets are separate blocks precisely for this reason
- **E** — variables are for non-sensitive values

---

## Q23

Which **two** are needed to consume an Azure Pipelines template from another repository? (Choose
two.)

- A. A `resources: repositories:` entry with an alias
- B. An `@alias` suffix on the template path
- C. A `checkout` step for the template repository
- D. A service connection for Azure Repos
- E. A variable group naming the repository

### Answer: A, B

**In `challenge-23.md`:** lines **785–793**.

```yaml
resources:
  repositories:
    - repository: shared-templates      # A - the alias
      type: git
      name: ContosoOrg/pipeline-templates
      ref: refs/tags/v2.1.0

stages:
  - template: stages/standard-deploy.yml@shared-templates    # B
```

**Why the others fail**

- **C** — templates resolve at **compile time**, before any agent exists. A checkout happens at
  runtime, far too late
- **D** — a service connection is required for **GitHub** (`type: github`), but `type: git` means
  Azure Repos in the same organization, which uses the pipeline's own identity
- **E** — variable groups hold values

---

# Section C — Repeated scenario

**Scenario:** Contoso has ten service repositories that each need the same build, test, container and
deploy pipeline. Teams must be able to set the image name and app name per service. The shared
definition must not change under a team without warning.

---

## Q24

**Proposed solution:** Create a reusable workflow in `contoso/.github` with typed inputs, and have
each service call it with `uses: contoso/.github/.github/workflows/reusable-node-ci-cd.yml@v2` plus a
`secrets:` block.

Does this meet the goal? **Yes**

### Answer: Yes

**In `challenge-23.md`:** lines **42–85**, **234–245**, **777**.

All three requirements are met:

- **One definition** — the whole pipeline lives in one file (lines 90–219)
- **Per-service configuration** — required inputs `image-name` and `azure-app-name` (lines 55–62)
- **No surprise changes** — `@v2` pins a tag, so the shared workflow can evolve on `main` without
  touching consumers until they bump

That third point is what makes this **Yes** rather than "works but risky".

---

## Q25

**Proposed solution:** Create a composite action in `contoso/.github/actions` and have each service
call it from a job step.

Does this meet the goal? **No**

### Answer: No

A composite action contributes **steps**, not jobs. Each service would still have to write the whole
job graph itself:

```yaml
jobs:
  build:      ...
  test:       needs: build
  docker:     needs: test
  deploy:     needs: docker
```

The requirement is to share the **entire pipeline**, including its structure, environments and
approvals. That structure is made of jobs (lines 90–219), which is exactly what a composite action
cannot supply.

**Where a composite action would be right:** sharing the six-step .NET build-and-test sequence at
lines 284–322 — steps inside a job that each team still shapes.

**The rule:** *sharing steps → composite action. Sharing a job graph → reusable workflow.*

---

## Q26

**Proposed solution:** Create a reusable workflow and have each service call it with `@main`.

Does this meet the goal? **No**

### Answer: No

Two of three requirements are met. The third fails.

`@main` is a **moving reference**. The moment anyone pushes to the shared repository, all ten
services pick up the change on their next run — including a change that breaks them. The requirement
says the shared definition must not change under a team without warning.

Line 778 states the fix in a comment: *"Pin to a tag/release for stability."*

**A partial solution is still No.** Note this is the same shape as Challenge 22 Q26: the mechanism is
right, one stated requirement is unmet.

---

# Section D — Yes/No statement grid

---

## Q27 — reusable workflows

| # | Statement | Answer |
|---|---|---|
| 1 | Secrets are passed automatically from caller to reusable workflow | **No** |
| 2 | A reusable workflow can declare outputs sourced from its jobs | **Yes** |
| 3 | The calling job needs a `runs-on` value | **No** |
| 4 | A reusable workflow can be pinned to a tag | **Yes** |

**In `challenge-23.md`:** lines **824–851** (row 1), **79–85** (row 2), **234–236** (row 3), **777**
(row 4).

Row 3 is the structural giveaway. Look at the caller:

```yaml
jobs:
  ci-cd:
    uses: contoso/.github/.github/workflows/reusable-node-ci-cd.yml@main
    with: ...
    secrets: ...
```

No `runs-on`, no `steps`. The job **is** the call. Adding `runs-on` is an error, and spotting its
absence is a fast way to tell a reusable-workflow call from a composite-action step.

---

## Q28 — composite actions

| # | Statement | Answer |
|---|---|---|
| 1 | `shell` is required on every `run` step | **Yes** |
| 2 | A composite action can read the `secrets` context directly | **No** |
| 3 | It runs in the caller's workspace | **Yes** |
| 4 | Its outputs use `value:` referencing a step output | **Yes** |

**In `challenge-23.md`:** lines **291** (row 1), **283** (row 3), **277** (row 4).

**Row 2 is the one to learn.** A composite action has **no access to the `secrets` context**. If it
needs a secret, the caller must pass it as an **input**:

```yaml
      - uses: ./.github/actions/deploy
        with:
          token: ${{ secrets.DEPLOY_TOKEN }}     # caller resolves it, action receives an input
```

Reusable workflows are the opposite — they have a dedicated `secrets:` block (lines 68–78). Two
reuse mechanisms, two different secret models.

---

## Q29 — Azure Pipelines templates

| # | Statement | Answer |
|---|---|---|
| 1 | A `steps:` template can be inserted where stages are expected | **No** |
| 2 | Parameters are type-checked at compile time | **Yes** |
| 3 | `${{ each }}` expands YAML before the run starts | **Yes** |
| 4 | Templates can reference other templates | **Yes** |

**In `challenge-23.md`:** lines **574 / 602 / 612** (row 1), **859–870** (row 2), **447** (row 3),
line **703** (row 4).

Row 2 is genuinely useful: a wrong parameter type fails **before an agent is allocated**. Compare
with a runtime failure, which costs you a queued job, a checkout and several minutes.

Row 4 — nesting is supported on both platforms. GitHub allows up to **four** levels of nested
reusable workflows.

---

## Q30 — sharing and versioning

| # | Statement | Answer |
|---|---|---|
| 1 | `@main` gives consumers a stable, unchanging definition | **No** |
| 2 | `ref: main` is valid in an Azure Pipelines repository resource | **No** |
| 3 | Org-wide reusable workflows live in a repository named `.github` | **Yes** |
| 4 | Composite actions in that repo live under `.github/workflows/` | **No** |

**In `challenge-23.md`:** lines **777–778** (row 1), **896** (row 2), **754–769** (rows 3 and 4).

Row 4 is the path asymmetry from Q14:

```text
contoso/.github/
  .github/workflows/reusable-node-ci-cd.yml   -> contoso/.github/.github/workflows/...@main
  actions/dotnet-build-test/action.yml        -> contoso/.github/actions/dotnet-build-test@main
```

Workflows need the inner `.github/workflows/`. Actions sit at the repository root. Getting this wrong
gives a file-not-found that looks like a permissions problem.

---

# Section E — Drag and drop

---

## Q31

Match each reuse mechanism to what it contributes.

| Mechanism | Contributes |
|---|---|
| Composite action | **Steps**, inside the caller's job |
| Reusable workflow | **Jobs**, each on its own runner |
| Azure Pipelines steps template | **Steps**, inside a job |
| Azure Pipelines jobs template | **Jobs**, inside a stage |
| Azure Pipelines stages template | **Stages**, inside the pipeline |
| Task group (classic) | **Steps** only |

**In `challenge-23.md`:** lines **283**, **90**, **353**, **427**, **469**, **705**.

**The cross-platform pairing:**

| GitHub Actions | Azure Pipelines |
|---|---|
| Composite action | steps template |
| Reusable workflow | jobs or stages template |
| — | task group (classic only) |

GitHub has **two** mechanisms; Azure Pipelines has **one mechanism at three levels**. That asymmetry
is why exam questions phrase the same need differently per platform.

---

## Q32

Arrange the output chain of a composite action, from producer to consumer.

**Items:** Caller reads `steps.<id>.outputs.<name>` · Step writes to `$GITHUB_OUTPUT` ·
Action `outputs` maps it with `value:` · Step declares an `id`

### Answer

1. Step declares an `id` — line **314**
2. Step writes to `$GITHUB_OUTPUT` — line **322**
3. Action `outputs` maps it with `value:` — line **277**
4. Caller reads `steps.<id>.outputs.<name>` — line **343**

```yaml
    - name: Publish
      id: publish                                       # 1
      shell: bash
      run: echo "path=$OUTPUT_PATH" >> $GITHUB_OUTPUT   # 2

outputs:
  artifact-path:
    value: ${{ steps.publish.outputs.path }}            # 3
```

```yaml
          path: ${{ steps.build.outputs['artifact-path'] }}    # 4
```

**Same four-link shape as job outputs in Challenge 19.** Miss any link and you get an empty string
with no error — the silent failure that keeps appearing across these challenges.

---

## Q33

Arrange the equivalent output chain for a **reusable workflow**.

**Items:** Workflow `outputs` maps it with `value: jobs.<job>.outputs.<name>` ·
Step writes to `$GITHUB_OUTPUT` · Job `outputs` maps the step output · Step declares an `id`

### Answer

1. Step declares an `id` — line **156** (`id: meta`)
2. Step writes to `$GITHUB_OUTPUT` — implicit in the action's own output
3. **Job** `outputs` maps the step output — line **146**
4. **Workflow** `outputs` maps the job output — line **82**

```yaml
  docker:
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}       # 3 - job level
...
    outputs:
      image-tag:
        value: ${{ jobs.docker.outputs['image-tag'] }}   # 4 - workflow level
```

**One more link than a composite action**, because a reusable workflow has a job layer that a
composite action does not. Compare Q32 (four links, `steps`) with this (five links, `steps` → `jobs`).

---

## Q34

Arrange these template files by the level at which they are inserted, from innermost to outermost.

**Items:** `stages/deploy-container-app.yml` · `steps/dotnet-build.yml` · `jobs/docker-build-push.yml`

### Answer: steps → jobs → stages

```yaml
stages:
  - stage: Build
    jobs:
      - job: BuildApp
        steps:
          - template: templates/steps/dotnet-build.yml@templates      # line 574

  - stage: Docker
    jobs:
      - template: templates/jobs/docker-build-push.yml@templates      # line 602

  - template: templates/stages/deploy-container-app.yml@templates     # line 612
```

**Read the indentation, not the filename.** The stage template sits at the outermost level with a
single `-`; the job template is nested one level; the step template two.

Directory layout at lines **802–816** mirrors this: `stages/`, `jobs/`, `steps/`, `variables/`.

---

## Q35

Match each requirement to the correct mechanism.

| Requirement | Mechanism |
|---|---|
| Share a whole build-test-deploy pipeline across ten repos | **Reusable workflow** |
| Share six setup steps inside an existing job | **Composite action** |
| Share a deploy **stage** across Azure Pipelines | **Stage template** |
| Share values that differ per environment | **Environment-scoped variable group** |
| Prevent a shared definition changing without warning | **Pin to a tag** |
| Restrict a parameter to `quick` or `full` | **`type: string` with `values`** |
| Repeat a task input once per list item | **`${{ each }}`** |

**In `challenge-23.md`:** lines **90**, **283**, **469**, **684**, **777**, **716**, **447**.

**Row 4 in detail (lines 651–669):** `contoso-staging` and `contoso-production` define the **same
variable names** with different values. The template writes `$(ResourceGroup)` once and each stage
resolves it differently — the same pattern as GitHub environment secrets.

---

# Section F — Hot area

---

## Q36

```yaml
on:
  [BLANK 1]:
    inputs:
      image-name:
        required: true
        type: string
    [BLANK 2]:
      AZURE_CLIENT_ID:
        required: true
    outputs:
      image-tag:
        value: ${{ [BLANK 3].docker.outputs['image-tag'] }}
```

- **BLANK 1:** `workflow_call` / `workflow_run` / `workflow_dispatch` / `repository_dispatch`
- **BLANK 2:** `secrets` / `env` / `with` / `vars`
- **BLANK 3:** `jobs` / `steps` / `needs` / `inputs`

### Answer: `workflow_call`, `secrets`, `jobs`

**In `challenge-23.md`:** lines **43**, **68**, **82**.

BLANK 3 is the one to get right: a reusable workflow's outputs come from the **`jobs`** context,
because a workflow is made of jobs. A composite action would use `steps`.

---

## Q37

```yaml
jobs:
  ci-cd:
    [BLANK 1]: contoso/.github/.github/workflows/reusable-node-ci-cd.yml@v2
    [BLANK 2]:
      image-name: order-service
    [BLANK 3]: inherit
```

- **BLANK 1:** `uses` / `runs-on` / `template` / `extends`
- **BLANK 2:** `with` / `inputs` / `parameters` / `env`
- **BLANK 3:** `secrets` / `env` / `permissions` / `vars`

### Answer: `uses`, `with`, `secrets`

**In `challenge-23.md`:** lines **236–246** and **846**.

Note there is **no `runs-on`** in this job — the giveaway from Q27. `template` and `parameters` are
Azure Pipelines words; `extends` too.

---

## Q38

```yaml
name: ".NET build and test"
[BLANK 1]:
  using: "composite"
  steps:
    - name: Restore
      [BLANK 2]: bash
      run: dotnet restore

[BLANK 3]:
  artifact-path:
    value: ${{ steps.publish.outputs.path }}
```

- **BLANK 1:** `runs` / `jobs` / `on` / `steps`
- **BLANK 2:** `shell` / `run-with` / `interpreter` / `env`
- **BLANK 3:** `outputs` / `returns` / `exports` / `results`

### Answer: `runs`, `shell`, `outputs`

**In `challenge-23.md`:** lines **282**, **291**, **274**.

An action file has exactly these top-level keys: `name`, `description`, `inputs`, `outputs`, `runs`.
No `on`, no `jobs`.

---

## Q39

```yaml
resources:
  repositories:
    - repository: shared-templates
      type: [BLANK 1]
      name: ContosoOrg/pipeline-templates
      ref: [BLANK 2]

stages:
  - template: stages/standard-deploy.yml[BLANK 3]
```

- **BLANK 1:** `git` / `github` / `azure` / `repo`
- **BLANK 2:** `refs/tags/v2.1.0` / `v2.1.0` / `main` / `HEAD`
- **BLANK 3:** `@shared-templates` / `@ContosoOrg` / `#shared-templates` / `/shared-templates`

### Answer: `git`, `refs/tags/v2.1.0`, `@shared-templates`

**In `challenge-23.md`:** lines **788–793**.

`type: git` = Azure Repos in the same organization. `type: github` would additionally need an
`endpoint` (a service connection).

BLANK 2 combines two lessons: the ref must be a **full path** (Break & fix Exercise 3, line 896) and
a **tag** is the stable choice (line 790).

BLANK 3 uses the **alias**, not the repository name.

---

## Q40

```yaml
parameters:
  - name: tags
    type: [BLANK 1]
    default:
      - "$(Build.BuildId)"
      - "latest"

steps:
  - task: Docker@2
    inputs:
      tags:
        ${{ [BLANK 2] tag in parameters.tags }}:
          ${{ tag }}
```

- **BLANK 1:** `object` / `string` / `array` / `list`
- **BLANK 2:** `each` / `for` / `foreach` / `loop`

### Answer: `object`, `each`

**In `challenge-23.md`:** lines **421–425** and **447**.

`type: object` covers both lists and maps — there is no separate `array` or `list` type. `each` is
the only iteration keyword, and it expands YAML at **compile time**.

Passing a string here is Break & fix Exercise 2 (line 869).

---

## Q41

```yaml
parameters:
  - name: scanType
    type: string
    default: "full"
    [BLANK 1]:
      - quick
      - full

steps:
  - ${{ [BLANK 2] eq(parameters.scanType, 'full') }}:
      - task: ComponentGovernanceComponentDetection@0
```

- **BLANK 1:** `values` / `options` / `allowed` / `enum`
- **BLANK 2:** `if` / `when` / `condition` / `case`

### Answer: `values`, `if`

**In `challenge-23.md`:** lines **716** and **733**.

`options` is GitHub Actions `workflow_dispatch` vocabulary. `condition:` exists in Azure Pipelines
but is a **runtime** key on a step — it skips the step. `${{ if }}` is **compile-time** and removes
the step entirely. That distinction was Challenge 20 Q21 and it recurs here.

---

# Section G — Case study

## Case study: Contoso platform team

### Background

Contoso runs ten Node.js microservices, each in its own repository, plus three .NET services. Every
team has copied and modified the same pipeline, so fixes must be applied thirteen times.

### Requirements

**Standardisation**

- One definition of the build, test, container and deploy pipeline for Node services
- Teams set only the image name, app name and Node version
- The .NET services share a build-and-test step sequence but keep their own job structure

**Governance**

- A change to the shared pipeline must not reach teams until they opt in
- The security scan must support a `quick` and a `full` mode, and reject any other value
- Every deployment must go through staging before production

**Azure DevOps**

- The enterprise team uses Azure Pipelines and needs the same deploy stage reused across services
- Staging and production must use different resource groups without duplicating the template

---

## Q42

Which mechanism meets the standardisation requirement for the ten Node services?

- A. A reusable workflow in `contoso/.github`
- B. A composite action in `contoso/.github`
- C. A starter workflow
- D. A shared branch merged into each repository

### Answer: A

**In `challenge-23.md`:** lines **42–85** and **234–245**.

The shared thing is a **job graph** — build, test, docker, deploy-staging, deploy-production — and
only reusable workflows can contribute jobs.

**Why the others fail**

- **B** — steps only. Each team would still write the graph (Q25)
- **C** — copied once at creation. Later fixes do not propagate, which is the exact problem being
  solved
- **D** — merging a branch into thirteen repos is copying with extra steps

---

## Q43

Which mechanism meets the .NET requirement?

- A. A reusable workflow
- B. A composite action
- C. A stage template
- D. A task group

### Answer: B

**In `challenge-23.md`:** lines **253–323**, used at lines **333–338**.

Read the requirement precisely: *"share a build-and-test step sequence but keep their own job
structure."* Shared steps, own jobs → composite action.

**Why the others fail**

- **A** — would impose a job structure, which the requirement explicitly rules out
- **C** and **D** — Azure Pipelines and classic constructs. These are GitHub repositories

**Q42 and Q43 together are the whole challenge.** Same organisation, same shared repository,
different mechanism — because one shares a graph and the other shares a sequence.

---

## Q44

Which configuration meets the governance requirement about opting in to changes?

- A. Reference the shared workflow with `@v2`
- B. Reference the shared workflow with `@main`
- C. Require a pull request review on the shared repository
- D. Copy the workflow into each repository

### Answer: A

**In `challenge-23.md`:** lines **777–778**.

**Why the others fail**

- **B** — every push to the shared repo reaches all thirteen services immediately (Q26)
- **C** — review improves quality but changes nothing about propagation. A reviewed change on `main`
  still lands everywhere at once
- **D** — back to thirteen copies

**How teams opt in:** bump the ref to `@v3` when they are ready. That is a normal pull request in
their own repository, reviewable and revertible.

---

## Q45

Which parameter definition meets the security-scan requirement?

- A. `type: string` with `values: [quick, full]`
- B. `type: choice` with `options: [quick, full]`
- C. `type: string` with `default: "full"` only
- D. `type: object` with a list of allowed values

### Answer: A

**In `challenge-23.md`:** lines **712–718**.

```yaml
parameters:
  - name: scanType
    type: string
    default: "full"
    values:
      - quick
      - full
```

Anything else fails at **compile time**, before an agent is allocated.

**Why the others fail**

- **B** — GitHub Actions syntax
- **C** — a default applies when the parameter is omitted. It does not reject a wrong value
- **D** — `object` accepts a list but validates nothing

---

## Q46

The Azure DevOps enterprise team needs the same deploy stage for staging and production, with
different resource groups. What should they build?

- A. Two stage templates, one per environment
- B. One stage template with parameters, referenced twice
- C. One stage template plus two variable groups only
- D. A step template called from both stages

### Answer: B

**In `challenge-23.md`:** lines **612–631** — the same template referenced twice.

```yaml
  - template: templates/stages/deploy-container-app.yml@templates
    parameters:
      environment: "staging"
      resourceGroup: "contoso-staging-rg"
      dependsOn: [Docker]

  - template: templates/stages/deploy-container-app.yml@templates
    parameters:
      environment: "production"
      resourceGroup: "contoso-prod-rg"
      dependsOn: [Deploy_staging]
      condition: "and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))"
```

**Why the others fail**

- **A** — two templates is duplication, the problem being solved
- **C** — variable groups supply values but cannot contribute a stage
- **D** — a step template cannot create a stage or attach an environment

**Notice `dependsOn` and `condition` passed as parameters.** That is what lets one template occupy
two different positions in the graph — and it is why production waits for staging, satisfying the
third governance requirement.

---

## Q47

A service's pipeline fails with "template file not found" for
`templates/steps/dotnet-build.yml@templates`, though the file exists in the template repository.

Which **two** causes are most likely? (Choose two.)

- A. The `resources: repositories:` entry uses `ref: main` instead of `refs/heads/main`
- B. The alias in the `@` suffix does not match the declared `repository` value
- C. The template repository was not checked out at runtime
- D. The pipeline lacks a service connection for Azure Repos
- E. The template uses `type: object` parameters

### Answer: A, B

**In `challenge-23.md`:** Break & fix Exercise 3, lines **890–915** (A); lines **555** and **574**
(B).

```yaml
    - repository: templates          # the alias...
      ref: refs/heads/main
...
          - template: templates/steps/dotnet-build.yml@templates    # ...must match here
```

**Why the others fail**

- **C** — templates resolve at **compile time**, before an agent exists. A checkout cannot help, and
  no checkout is needed
- **D** — `type: git` (line 556) means Azure Repos in the same organization, authenticated by the
  pipeline's own identity. A service connection is only needed for `type: github`
- **E** — a type mismatch produces "unexpected value" (Exercise 2), not "file not found"

**Reading errors precisely pays off:** *file not found* = resolution or ref. *Unexpected value* =
parameter type. *Permission denied* = endpoint or scope.

---

## Q48

Six months later, the shared reusable workflow adds a mandatory `security-scan` job. Three teams are
still on `@v2` and unaffected. One team bumps to `@v3` and their pipeline fails because the new job
requires a secret they have not configured.

What is the correct fix, and what does this illustrate?

- A. Add the secret to that repository and pass it in the caller's `secrets:` block
- B. Switch that team back to `@main`
- C. Make the secret optional in the reusable workflow
- D. Use `secrets: inherit` in every consuming repository

### Answer: A

**In `challenge-23.md`:** lines **68–78** show the secrets contract, lines **242–245** show a caller
satisfying it.

```yaml
    secrets:
      AZURE_CLIENT_ID:
        required: true      # the reusable workflow's contract
```

The team must create the secret and pass it. **A reusable workflow's `inputs` and `secrets` blocks
are a contract**, and bumping the ref means accepting the new version of that contract.

**Why the others fail**

- **B** — `@main` is less stable, not more. It would also pull in every other unreleased change
- **C** — weakens the security control for everyone to avoid one team's configuration task. If a scan
  needs credentials, making them optional means it silently does not scan
- **D** — `secrets: inherit` only passes secrets the caller **has**. If the repository does not have
  it, inherit passes nothing. It is not a substitute for creating the secret

**What it illustrates:** version pinning worked exactly as designed. Three teams were insulated from
a breaking change, and the fourth adopted it deliberately with a visible, fixable failure. That is
the argument for pinning, stated as an outcome rather than a rule.

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Composite action used where jobs are needed** | Q13, Q25, Q42, Q43 | Steps → composite. Job graph → reusable workflow |
| **Secrets assumed to flow automatically** | Q1, Q18, Q22, Q27 | Pass them explicitly or use `secrets: inherit` |
| **Composite action reading `secrets` context** | Q28 | It cannot. Pass the secret as an input |
| **`@main` treated as stable** | Q10, Q19, Q26, Q44 | Pin to a tag for anything consumers depend on |
| **`ref: main` instead of `refs/heads/main`** | Q9, Q39, Q47 | Full ref path, always |
| **Alias vs repository name** | Q23, Q39, Q47 | `@alias` refers to the `repository:` value |
| **`type: choice` offered in Azure Pipelines** | Q16, Q45 | That is GitHub. Azure Pipelines uses `type: string` + `values` |
| **Object parameter given a string** | Q8, Q40 | Pass a real YAML list |
| **`runs-on` added to a reusable-workflow call** | Q18, Q27 | The calling job runs no steps of its own |
| **Wrong output context** | Q6, Q32, Q33, Q36 | Composite → `steps`. Reusable workflow → `jobs` |
| **Task group assumed to work in YAML** | Q12, Q21 | Classic only, and step-level only |
| **Checkout expected to fetch templates** | Q23, Q47 | Templates resolve at compile time |
| **`.github` path asymmetry** | Q14, Q30 | Workflows: `.github/workflows/`. Actions: repo root `actions/` |

---

# The blocks to memorise

Line numbers are in `challenge-23.md`.

```yaml
# 1. Reusable workflow contract  (lines 42-85)
on:
  workflow_call:
    inputs:
      image-name:
        required: true
        type: string
    secrets:
      AZURE_CLIENT_ID:
        required: true
    outputs:
      image-tag:
        value: ${{ jobs.docker.outputs['image-tag'] }}    # jobs context

# 2. Calling it  (lines 234-245) - note: NO runs-on
jobs:
  ci-cd:
    uses: contoso/.github/.github/workflows/reusable-node-ci-cd.yml@v2
    with:
      image-name: order-service
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
    # or:  secrets: inherit

# 3. Composite action  (lines 274-323) - steps context
outputs:
  artifact-path:
    value: ${{ steps.publish.outputs.path }}
runs:
  using: "composite"
  steps:
    - id: publish
      shell: bash
      run: echo "path=$OUTPUT_PATH" >> $GITHUB_OUTPUT

# 4. Calling it  (lines 333-338) - under steps:
      - id: build
        uses: contoso/.github/actions/dotnet-build-test@main
        with:
          project-path: src/Contoso.Api/Contoso.Api.csproj

# 5. External template repository  (lines 785-793)
resources:
  repositories:
    - repository: shared-templates
      type: git
      name: ContosoOrg/pipeline-templates
      ref: refs/tags/v2.1.0
stages:
  - template: stages/standard-deploy.yml@shared-templates

# 6. Restricted parameter  (lines 712-721)
parameters:
  - name: scanType
    type: string
    default: "full"
    values: [quick, full]
  - name: failOnHighSeverity
    type: boolean
    default: true

# 7. Iterating a list parameter  (lines 421-425, 447)
  - name: tags
    type: object
    default: ["$(Build.BuildId)", "latest"]
...
          tags:
            ${{ each tag in parameters.tags }}:
              ${{ tag }}

# 8. One stage template, two placements  (lines 612-631)
  - template: templates/stages/deploy-container-app.yml@templates
    parameters:
      environment: "staging"
      dependsOn: [Docker]
  - template: templates/stages/deploy-container-app.yml@templates
    parameters:
      environment: "production"
      dependsOn: [Deploy_staging]
      condition: "and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))"
```

**The decision table:**

| You want to share | GitHub Actions | Azure Pipelines |
|---|---|---|
| Steps inside a job | Composite action | `steps:` template |
| A job | Reusable workflow | `jobs:` template |
| A stage | — (no stage concept) | `stages:` template |
| Values | Repository or environment variables | Variable group |

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 23 is exam-ready. Move to Challenge 24 |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 1 and 2 side by side, then retake |
| Below 30 | Redo the challenge, writing both a composite action and a reusable workflow by hand |

Record your result in `AZ-400-Learning-Log.md` under Challenge 23.

:::tip The one thing

**Composite = steps, same runner, `steps` context, secrets passed as inputs.**
**Reusable workflow = jobs, own runners, `jobs` context, dedicated `secrets:` block.**

Every one of those five differences follows from the first. Learn the first and derive the rest.

:::
