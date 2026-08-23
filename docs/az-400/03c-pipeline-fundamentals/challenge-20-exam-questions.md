---
sidebar_position: 2.5
toc_max_heading_level: 2
title: "Challenge 20: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 20 — AZ-400 exam questions

**48 questions** built only from what Challenge 20 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-20.md`** where the real syntax lives. Open that file beside this one.

| Section | Shape | Questions |
|---|---|---|
| A | Single answer | 1–16 |
| B | Multiple answer (choose two / three) | 17–23 |
| C | Repeated scenario — "Does this meet the goal?" | 24–26 |
| D | Yes/No statement grid | 27–30 |
| E | Drag and drop | 31–35 |
| F | Hot area — complete the YAML | 36–41 |
| G | Case study | 42–48 |

The **trap index**, the **eight blocks to memorise**, and **scoring** are at the end.

:::warning This is the other half of the exam

Challenge 19 was GitHub Actions. This is Azure Pipelines. The exam tests **both** and constantly asks
you to translate between them. Expression syntax (`${{ }}` vs `$[ ]` vs `$( )`) is the single
most-tested thing in this challenge.

:::

---

# Section A — Single answer

---

## Q1

A pipeline sets an output variable in the `Build` stage and must read it in the `Deploy` stage.

Which expression correctly reads it?

- A. `$(stageDependencies.Build.BuildJob.outputs['setVersion.buildVersion'])`
- B. `$[ stageDependencies.Build.BuildJob.outputs['setVersion.buildVersion'] ]`
- C. `${{ stageDependencies.Build.BuildJob.outputs['setVersion.buildVersion'] }}`
- D. `$(Build.BuildJob.setVersion.buildVersion)`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-20.md`:** lines **636–647** (Break & fix Exercise 2 solution).

```yaml
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: |
              echo "##vso[task.setvariable variable=buildVersion;isOutput=true]1.2.3"
            name: setVersion              # line 638 - the step MUST be named

  - stage: Deploy
    dependsOn: Build                      # line 641 - required
    variables:
      buildVersion: $[ stageDependencies.Build.BuildJob.outputs['setVersion.buildVersion'] ]
    jobs:
      - job: DeployJob
        steps:
          - script: echo $(buildVersion)
```

**Why the others fail**

- **A** — `$( )` is **macro** syntax. It substitutes a variable's value just before a task runs. It
  cannot evaluate a `stageDependencies` expression
- **C** — `${{ }}` is **compile time**. The pipeline YAML is expanded before the run starts, so the
  Build stage has not executed and the output does not exist yet. You get an empty value
- **D** — not valid syntax. There is no such shorthand

**Term:** *runtime expression*. `$[ ]` is the only one that can read another stage's output.

</details>

---

## Q2

What is the difference between `${{ }}` and `$[ ]` in Azure Pipelines?

- A. `${{ }}` is for templates and `$[ ]` is for variables
- B. `${{ }}` is evaluated at compile time and `$[ ]` is evaluated at runtime
- C. `${{ }}` is for YAML pipelines and `$[ ]` is for classic pipelines
- D. There is no difference; they are interchangeable

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-20.md`:** `${{ }}` at lines **333–357** and **410**; `$[ ]` at line **643**.

| Syntax | Name | Evaluated | Used for |
|---|---|---|---|
| `${{ }}` | Template expression | When the YAML is **parsed**, before the run | Template parameters, `${{ if }}`, `${{ each }}` |
| `$[ ]` | Runtime expression | When the **run / stage / job begins** | `variables:` and `condition:` reading outputs |
| `$( )` | Macro | Just **before a task executes** | Task inputs and scripts |

```yaml
# Compile time - decides whether these steps EXIST at all (line 351)
  - ${{ if parameters.publishArtifact }}:
      - task: DotNetCoreCLI@2

# Runtime - reads a value that only exists once Build has run (line 643)
      buildVersion: $[ stageDependencies.Build.BuildJob.outputs['setVersion.buildVersion'] ]

# Macro - drops a value into a task input (line 98)
              version: $(dotnetVersion)
```

**Why the others fail**

- **A** — half true and therefore wrong. `${{ }}` is used far beyond templates, and `$[ ]` appears in
  `variables:` and `condition:`, not "variables" generally
- **C** — classic pipelines are not YAML at all. Neither syntax belongs to them
- **D** — the whole point is that they differ in *when* they run

**Memorise the ordering:** `${{ }}` before the run exists → `$[ ]` as the run begins → `$( )` while a
step runs.

</details>

---

## Q3

The `AzureKeyVault@2` task fetches a secret named `SqlConnectionString`. How do later steps reference
it?

- A. `$(keyVault.SqlConnectionString)`
- B. `$(SqlConnectionString)`
- C. `${{ variables.SqlConnectionString }}`
- D. `$(secrets.SqlConnectionString)`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-20.md`:** the task at lines **184–190**, consumed at line **201**.

```yaml
                - task: AzureKeyVault@2
                  inputs:
                    azureSubscription: "contoso-azure-connection"
                    KeyVaultName: "kv-contoso-staging"
                    SecretsFilter: "SqlConnectionString,AppInsightsKey"
                    RunAsPreJob: false
```

```yaml
                    AppSettings: >-
                      -ConnectionStrings__Default "$(SqlConnectionString)"
```

The task maps each Key Vault secret **directly to a pipeline variable of the same name**. No prefix,
no namespace.

**Why the others fail**

- **A** and **D** — there is no `keyVault.` or `secrets.` namespace in Azure Pipelines. That
  `secrets.` shape is GitHub Actions syntax leaking in, which is exactly why the exam offers it
- **C** — `${{ }}` is compile time. The secret is fetched during the run, so it does not exist yet

**Term:** *secrets become variables of the same name.* Also note they are **masked** in logs.

</details>

---

## Q4

Which Azure Pipelines construct is required for a job to use an environment with approvals?

- A. `job:` with a `condition` referencing the environment
- B. `deployment:` with an `environment` property
- C. `stage:` with a `dependsOn` on the environment
- D. `job:` with a `pool` scoped to the environment

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-20.md`:** lines **177–183**.

```yaml
      - deployment: DeployStagingJob
        displayName: "Deploy to staging environment"
        environment: "contoso-staging"
        strategy:
          runOnce:
            deploy:
              steps:
```

A **deployment job** is a different construct from a regular `job`. It gives you deployment history,
approvals and checks, and it requires a `strategy`.

**Why the others fail**

- **A** — a regular `job` has no `environment` property at all. A condition cannot create one
- **C** — `dependsOn` orders stages. Environments are not stages
- **D** — `pool` selects an agent. Nothing to do with approvals

**Term:** *deployment job*. Regular job = do work. Deployment job = record a deployment against an
environment.

</details>

---

## Q5

Which strategy keyword is required inside a `deployment` job in Challenge 20's pipeline?

- A. `matrix`
- B. `runOnce`
- C. `parallel`
- D. `maxParallel`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-20.md`:** lines **180–183**.

```yaml
        strategy:
          runOnce:
            deploy:
              steps:
```

`runOnce` executes the deploy steps a single time. The lifecycle hooks available are `preDeploy`,
`deploy`, `routeTraffic` and `postRouteTraffic`, plus `on: failure` / `on: success`.

**Why the others fail**

- **A** — `matrix` is a strategy for **regular** jobs, not deployment jobs
- **C** — `parallel` is a real deployment strategy but only for **VM resources** in an environment,
  not for this App Service deployment
- **D** — `maxParallel` is a setting inside a matrix, not a strategy

**Also know:** `rolling` and `canary` are the other two deployment-job strategies. They appear in
Challenge 25.

</details>

---

## Q6

Where is a build artifact available to a later stage?

- A. `$(Build.ArtifactStagingDirectory)`
- B. `$(Pipeline.Workspace)`
- C. `$(System.DefaultWorkingDirectory)`
- D. `$(Agent.TempDirectory)`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-20.md`:** published from `$(Build.ArtifactStagingDirectory)` at line **126**,
consumed from `$(Pipeline.Workspace)` at line **199**.

```yaml
          - task: PublishPipelineArtifact@1        # line 123
            inputs:
              targetPath: "$(Build.ArtifactStagingDirectory)/app"
              artifact: "drop"
```

```yaml
                    packageForLinux: "$(Pipeline.Workspace)/drop/**/*.zip"    # line 199
```

A deployment job **downloads artifacts automatically** into `$(Pipeline.Workspace)/<artifactName>`.

**Why the others fail**

- **A** — the staging directory is where you **put** files before publishing, on the *building*
  agent. A later stage runs on a different agent where that folder is empty
- **C** — the source checkout directory. Deployment jobs do not check out source by default
- **D** — scratch space for the current job, used at line 154 for test results. Not shared

**Term:** *publish from the staging directory, consume from the pipeline workspace.*

</details>

---

## Q7

A stage must run only when the previous stage succeeded **and** the source branch is `main`.

Which condition is correct?

- A. `condition: eq(variables['Build.SourceBranch'], 'main')`
- B. `condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))`
- C. `condition: succeeded() && variables.Build.SourceBranch == 'main'`
- D. `condition: ${{ eq(variables['Build.SourceBranch'], 'refs/heads/main') }}`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-20.md`:** line **173**.

```yaml
  - stage: DeployStaging
    dependsOn: Test
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
```

**Why the others fail**

- **A** — two problems. It drops `succeeded()`, so the stage would run even after a failure; and
  `Build.SourceBranch` holds the **full ref** `refs/heads/main`, not `main`. Use
  `Build.SourceBranchName` if you want the short name
- **C** — `&&` and dot-property comparison are not Azure Pipelines condition syntax. Conditions use
  functions: `and()`, `or()`, `eq()`, `ne()`, `not()`
- **D** — `${{ }}` is compile time. It cannot know whether the previous stage succeeded, because
  nothing has run yet

**Two variables to keep straight:** `Build.SourceBranch` = `refs/heads/main`.
`Build.SourceBranchName` = `main`.

</details>

---

## Q8

A pipeline must publish test results even when tests fail.

Which configuration achieves this?

- A. `continueOnError: true` on the test task
- B. `condition: always()` on the publish task
- C. `dependsOn: []` on the publish task
- D. `enabled: true` on the publish task

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-20.md`:** lines **156–163**.

```yaml
          - task: PublishTestResults@2
            displayName: "Publish test results"
            condition: always()
            inputs:
              testResultsFormat: "VSTest"
              testResultsFiles: "**/*.trx"
              mergeTestResults: true
```

By default a step is skipped once a previous step in the job has failed. `condition: always()`
overrides that — which is exactly what you want for test results, because **failing tests are the
results you most need to see**.

**Why the others fail**

- **A** — `continueOnError: true` on the **test** task would mark the failing tests as a warning and
  let the pipeline pass. That hides the failure instead of reporting it
- **C** — `dependsOn` applies to jobs and stages, not steps
- **D** — `enabled` controls whether a task is included at all. It does not override failure skipping

**Term:** *step condition*. `always()`, `succeeded()`, `failed()`, `succeededOrFailed()`.

</details>

---

## Q9

A trigger must fire on pushes to `main` and any `release/*` branch, but not for documentation
changes.

Which configuration is correct?

- A. `trigger: [main, release/*]` with a separate `paths` block at pipeline root
- B. `trigger:` with `branches: include:` and `paths: exclude:`
- C. `pr:` with `branches: include:` and `paths: exclude:`
- D. `trigger: none` plus a scheduled trigger filtered by path

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-20.md`:** lines **54–62**.

```yaml
trigger:
  branches:
    include:
      - main
      - release/*
  paths:
    exclude:
      - docs/**
      - "*.md"
```

**Why the others fail**

- **A** — the array shorthand `trigger: [main]` is valid, but it accepts **only** branches. You
  cannot attach path filters to it, and there is no root-level `paths` key
- **C** — `pr:` controls **pull request** validation, a different event. Lines 64–71 show it used for
  exactly that
- **D** — `trigger: none` disables CI triggering entirely

**Term:** *`trigger` = CI (push). `pr` = PR validation.* In GitHub Actions both live under `on:`.

</details>

---

## Q10

Which template type allows a template to contribute **jobs** rather than steps?

- A. A template whose root key is `steps:`
- B. A template whose root key is `jobs:`
- C. A template referenced with `extends:`
- D. A template referenced with `resources:`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-20.md`:** the steps template starts at line **331** (`steps:`); the jobs template
starts at line **386** (`jobs:`).

```yaml
# build-template.yml - line 331
steps:
  - task: UseDotNet@2
```

```yaml
# deploy-template.yml - line 386
jobs:
  - deployment: Deploy_${{ parameters.environment }}
```

The root key decides where the template can be inserted. A `steps:` template goes under a job's
`steps:` (line 437). A `jobs:` template goes under a stage's `jobs:` (line 445).

**Why the others fail**

- **A** — a steps template can only contribute steps
- **C** — `extends:` makes the **whole pipeline** inherit from a template. Useful for governance
  (the "required template" check), but it is not what distinguishes jobs from steps
- **D** — `resources:` declares external repos, pipelines and containers (line 470). It does not
  insert anything

**Four template types to know:** steps, jobs, stages, variables.

</details>

---

## Q11

A template parameter must accept only `dev`, `staging` or `prod`.

Which parameter definition enforces that?

- A. `type: string` with a `values` list
- B. `type: enum` with an `options` list
- C. `type: choice` with an `options` list
- D. `type: string` with a `default` list

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-20.md`:** lines **565–571**.

```yaml
parameters:
  - name: environment
    type: string
    values:
      - dev
      - staging
      - prod
```

If a caller passes anything else, the pipeline fails **at compile time** with a validation error —
before a single agent is used.

**Why the others fail**

- **B** — there is no `enum` type in Azure Pipelines
- **C** — `type: choice` with `options` is **GitHub Actions** `workflow_dispatch` syntax (Challenge
  19, line 59). The exam offers it deliberately, because both platforms solve the same problem with
  different keywords
- **D** — `default` provides a fallback value; it does not restrict what is allowed

</details>

---

## Q12

A template parameter is declared `type: bool` and the pipeline fails validation.

What is the correct type name?

- A. `binary`
- B. `boolean`
- C. `flag`
- D. `switch`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-20.md`:** the error is at line **573**, the fix at line **593**.

```yaml
  - name: runTests
    type: bool        # line 573 - INVALID
```

```yaml
  - name: runTests
    type: boolean     # line 593 - correct
```

**Why the others fail**

None of `binary`, `flag` or `switch` exist. The valid parameter types are: `string`, `number`,
`boolean`, `object`, `step`, `stepList`, `job`, `jobList`, `deployment`, `deploymentList`, `stage`,
`stageList`.

**Related trap from the same exercise (lines 578 and 598):**

```yaml
  - ${{ if eq(parameters.runTests, 'true') }}:   # WRONG - compares to a string
  - ${{ if eq(parameters.runTests, true) }}:     # correct - compares to a boolean
```

A quoted `'true'` is a string, and a boolean never equals a string. The condition is silently always
false, so the steps simply never appear.

</details>

---

## Q13

A variable is defined at stage level in `Build`. A job in the `Deploy` stage reads it and gets
nothing.

Why?

- A. Stage-level variables are only available within that stage
- B. Variables must be defined in a variable group to be readable
- C. The `Deploy` stage is missing a `dependsOn` on `Build`
- D. Variables must use `${{ }}` syntax to cross stages

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-20.md`:** Break & fix Exercise 2, lines **604–622**.

```yaml
  - stage: Build
    variables:
      buildOutput: "$(Build.ArtifactStagingDirectory)"
    jobs:
      - job: BuildJob
        steps:
          - script: echo $(buildOutput)     # line 615 - works

  - stage: Deploy
    jobs:
      - job: DeployJob
        steps:
          - script: echo $(buildOutput)     # line 621 - empty
```

Variable **scope** is pipeline → stage → job. A stage variable does not escape its stage.

**Why the others fail**

- **B** — variable groups make values available across a pipeline, but a plain stage variable is
  perfectly valid. The problem is scope, not where it was defined
- **C** — `dependsOn` alone would not help. It creates ordering and unlocks `stageDependencies`, but
  the plain `$(buildOutput)` macro still would not resolve
- **D** — `${{ }}` is compile time and would not fix a runtime scope problem

**The fix (lines 636–647):** turn it into an **output variable** with `isOutput=true` and read it via
`stageDependencies` with `$[ ]`.

</details>

---

## Q14

A pipeline must consume a template stored in a different GitHub repository.

What must be declared first?

- A. A `resources: repositories:` entry with a service connection endpoint
- B. A `variables: group:` entry pointing to the repository
- C. A `pool:` entry naming the external repository
- D. A `checkout:` step for the external repository

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-20.md`:** lines **470–476**, used at line **502**.

```yaml
resources:
  repositories:
    - repository: templates
      type: github
      name: contoso/pipeline-templates
      ref: refs/heads/main
      endpoint: contoso-github-connection
```

```yaml
          - template: dotnet/build.yml@templates      # line 502
```

Note the `@templates` suffix — it refers back to the **alias** defined under `resources`.

**Why the others fail**

- **B** — variable groups hold values, not repositories
- **C** — `pool` selects an agent
- **D** — `checkout` would clone the repo into the workspace at **runtime**. Templates are resolved
  at **compile time**, before any agent exists, so a checkout is far too late

**Term:** *repository resource*. The `type` can be `git` (Azure Repos, line 478), `github`, or
`bitbucket`.

</details>

---

## Q15

Which task publishes code coverage in the format `dotnet test --collect:"XPlat Code Coverage"`
produces?

- A. `PublishTestResults@2` with `testResultsFormat: "VSTest"`
- B. `PublishCodeCoverageResults@2` with a cobertura summary file
- C. `PublishPipelineArtifact@1` with the coverage folder
- D. `PublishBuildArtifacts@1` with `publishLocation: "Container"`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-20.md`:** lines **165–168**.

```yaml
          - task: PublishCodeCoverageResults@2
            inputs:
              summaryFileLocation: "$(Agent.TempDirectory)/testresults/**/coverage.cobertura.xml"
```

`--collect:"XPlat Code Coverage"` (line 154) produces **cobertura** XML. That is exactly what your
local `dotnet test` run on `contoso-webapi` produced.

**Why the others fail**

- **A** — publishes **test results** (pass/fail counts) from `.trx` files. A different artifact
- **C** and **D** — publish files for download. They produce no coverage report in the UI

**Remember the pairing:** `--logger trx` → `PublishTestResults@2`.
`--collect:"XPlat Code Coverage"` → `PublishCodeCoverageResults@2`.

</details>

---

## Q16

Azure Pipelines must build from a GitHub repository. What must exist in Azure DevOps?

- A. A GitHub service connection
- B. A GitHub personal access token stored in a variable group
- C. A GitHub Actions workflow that dispatches to Azure Pipelines
- D. A repository resource of `type: git`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-20.md`:** created at lines **515–519**, referenced at line **530**.

```bash
az devops service-endpoint github create \
  --github-url "https://github.com" \
  --name "contoso-github-connection" \
  --organization "https://dev.azure.com/contoso" \
  --project "ContosoApi"
```

```yaml
resources:
  repositories:
    - repository: self
      type: github
      name: contoso/contoso-webapi
      endpoint: contoso-github-connection     # line 530
```

**Why the others fail**

- **B** — a raw PAT in a variable group is not how repository access is configured, and it bypasses
  the auditing and scoping a service connection provides
- **C** — no dispatch is needed. Azure Pipelines connects to GitHub directly
- **D** — `type: git` means **Azure Repos**. GitHub requires `type: github` plus an endpoint

**Term:** *service connection*. This is the object Challenge 41 secures in depth.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** are valid Azure Pipelines expression syntaxes, and when does each evaluate? (Choose
three.)

- A. `${{ }}` — compile time
- B. `$[ ]` — runtime
- C. `$( )` — macro, just before a task runs
- D. `${ }` — deployment time
- E. `#[ ]` — template time
- F. `@( )` — variable group time

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-20.md`:** `${{ }}` line **333**, `$[ ]` line **643**, `$( )` line **98**.

```yaml
    displayName: "Install .NET SDK ${{ parameters.dotnetVersion }}"    # compile time
      buildVersion: $[ stageDependencies.Build... ]                    # runtime
              version: $(dotnetVersion)                                # macro
```

**Why the others fail**

`${ }`, `#[ ]` and `@( )` do not exist. They are invented to see whether you actually know the three
real ones or are pattern-matching on brackets.

**The rule:** `${{ }}` before the run exists → `$[ ]` as the run begins → `$( )` while a step runs.

</details>

---

## Q18

Which **two** items are required to share a value from one stage to another? (Choose two.)

- A. The producing step must have a `name`
- B. The variable must be set with `isOutput=true`
- C. The consuming stage must use `${{ }}` to read it
- D. Both stages must run on the same agent pool
- E. The value must be published as a pipeline artifact

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-20.md`:** lines **637–638** and **643**.

```yaml
          - script: |
              echo "##vso[task.setvariable variable=buildVersion;isOutput=true]1.2.3"
            name: setVersion      # A - without this you cannot reference it
```

```yaml
      buildVersion: $[ stageDependencies.Build.BuildJob.outputs['setVersion.buildVersion'] ]
```

Look at the reference path: `stageDependencies.<Stage>.<Job>.outputs['<stepName>.<varName>']`. The
step **name** is part of the address.

**Why the others fail**

- **C** — `${{ }}` is compile time and cannot see runtime output. It must be `$[ ]`
- **D** — agent pools are irrelevant. The value travels through Azure DevOps, not the filesystem
- **E** — artifacts move **files**. This is a variable

**Note:** `dependsOn` is also required in practice (line 641) — it was left out of the options here,
but the exam sometimes includes it as a third correct answer.

</details>

---

## Q19

Which **two** statements about `deployment` jobs are correct? (Choose two.)

- A. They require a `strategy` such as `runOnce`
- B. They automatically download pipeline artifacts
- C. They automatically check out the source repository
- D. They can use the `matrix` strategy
- E. They cannot reference variable groups

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-20.md`:** lines **177–183** (A), line **199** (B).

```yaml
      - deployment: DeployStagingJob
        environment: "contoso-staging"
        strategy:                                   # A - mandatory
          runOnce:
            deploy:
              steps:
                - task: AzureRmWebAppDeployment@4
                  inputs:
                    packageForLinux: "$(Pipeline.Workspace)/drop/**/*.zip"    # B
```

Nothing in that job downloads the artifact — it arrives automatically.

**Why the others fail**

- **C** — the reverse is true. A deployment job does **not** check out source by default. If you need
  the repo you must add `- checkout: self` explicitly. This catches people constantly
- **D** — `matrix` is for regular jobs. Deployment strategies are `runOnce`, `rolling`, `canary`
- **E** — false. Lines 174–175 attach `contoso-staging` to the stage containing a deployment job

</details>

---

## Q20

Which **two** are valid ways to restrict when a stage runs? (Choose two.)

- A. `dependsOn` on the stage
- B. `condition` on the stage
- C. `trigger` on the stage
- D. `pool` on the stage
- E. `pr` on the stage

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-20.md`:** lines **172–173**.

```yaml
  - stage: DeployStaging
    dependsOn: Test                                                          # A - ordering
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))   # B - gate
```

They do different jobs and you usually need both:

- `dependsOn` decides **when** the stage may start
- `condition` decides **whether** it runs once it may start

**Why the others fail**

- **C** and **E** — `trigger` and `pr` are **pipeline-level** keys (lines 54 and 64). They control
  what starts the whole run, not individual stages
- **D** — `pool` selects an agent

**Trap worth knowing:** the moment you add a custom `condition`, the implicit `succeeded()` is
**replaced**. That is why line 173 spells out `and(succeeded(), ...)` — omit it and the stage would
run even after a failure.

</details>

---

## Q21

Which **two** template features let a template include or exclude blocks based on a parameter?
(Choose two.)

- A. `${{ if ... }}` conditional insertion
- B. `${{ each ... }}` iteration
- C. `condition:` on the step
- D. `$[ ]` runtime expression
- E. `continueOnError`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-20.md`:** lines **351** and **410**.

```yaml
  - ${{ if parameters.publishArtifact }}:           # line 351
      - task: DotNetCoreCLI@2
```

```yaml
                ${{ if ne(parameters.slot, '') }}:  # line 410 - inserts INPUTS, not steps
                  deployToSlotOrASE: true
                  SlotName: ${{ parameters.slot }}
```

Line 410 is worth staring at: conditional insertion can add **any YAML**, including two task inputs,
not just whole steps.

**Why the others fail**

- **C** — `condition:` still **includes** the step; it just skips it at runtime, and the skipped step
  is visible in the log. Conditional insertion means the step never exists
- **D** — `$[ ]` reads values at runtime. It cannot add or remove YAML
- **E** — controls what happens after a failure

**The distinction the exam tests:** `${{ if }}` removes the step from the pipeline. `condition:`
keeps it and skips it.

</details>

---

## Q22

Which **two** resource types can be declared under `resources:`? (Choose two.)

- A. `repositories`
- B. `pipelines`
- C. `environments`
- D. `variables`
- E. `pools`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-20.md`:** lines **470–493**.

```yaml
resources:
  repositories:                     # A - external repos for templates or checkout
    - repository: templates
      type: github
  pipelines:                        # B - artifacts and triggers from another pipeline
    - pipeline: infrastructurePipeline
      source: "Contoso-Infrastructure-Deploy"
      trigger:
        branches:
          include:
            - main
  containers:                       # also valid - container jobs
    - container: build-tools
```

**Why the others fail**

- **C** — environments are referenced by a deployment job's `environment:` key (line 179), never
  declared as a resource
- **D** — variables have their own top-level `variables:` key (line 76)
- **E** — pools are selected with `pool:` (line 73)

**The four resource types:** `repositories`, `pipelines`, `containers`, `builds`. That
`pipelines` trigger at lines 485–489 is how one pipeline starts another — and it is the correct
answer whenever a question says "trigger after another pipeline completes".

</details>

---

## Q23

Which **two** GitHub Actions concepts map to Azure Pipelines templates? (Choose two.)

- A. Reusable workflow
- B. Composite action
- C. Starter workflow
- D. Repository dispatch
- E. Environment

<details>
<summary>Show answer</summary>

### Answer: A, B

Both are "reuse" mechanisms; Azure Pipelines covers the same ground with template types.

| GitHub Actions | Azure Pipelines | In `challenge-20.md` |
|---|---|---|
| Composite action (steps) | `steps:` template | line **331** |
| Reusable workflow (jobs) | `jobs:` template | line **386** |
| — | `stages:` template | — |

```yaml
          - template: templates/build-template.yml    # line 437 - steps template
            parameters:
              dotnetVersion: "8.0.x"
```

```yaml
      - template: templates/deploy-template.yml       # line 445 - jobs template
        parameters:
          environment: "contoso-staging"
```

**Why the others fail**

- **C** — a starter workflow is a one-time copy shown in the "New workflow" UI. Azure Pipelines has
  no equivalent, and it is not reuse
- **D** — `repository_dispatch` is an external API trigger. Its Azure Pipelines counterpart is a
  `pipelines` resource trigger, not a template
- **E** — environments exist on **both** platforms with the same name and purpose

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso's pipeline has a `Build` stage that computes a version number and a `Deploy`
stage that must use it. `Deploy` currently receives an empty value.

---

## Q24

**Proposed solution:** Set the variable with `##vso[task.setvariable variable=buildVersion;isOutput=true]`,
give the step a `name`, add `dependsOn: Build` to the Deploy stage, and read it with
`$[ stageDependencies.Build.BuildJob.outputs['setVersion.buildVersion'] ]`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-20.md`:** lines **636–647** — this is the documented fix, complete.

```yaml
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: |
              echo "##vso[task.setvariable variable=buildVersion;isOutput=true]1.2.3"
            name: setVersion

  - stage: Deploy
    dependsOn: Build
    variables:
      buildVersion: $[ stageDependencies.Build.BuildJob.outputs['setVersion.buildVersion'] ]
```

All four parts are present: `isOutput=true`, the step `name`, the `dependsOn`, and `$[ ]`.

</details>

---

## Q25

**Proposed solution:** Define `buildVersion` as a stage-level variable in `Build` and reference it as
`$(buildVersion)` in `Deploy`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

This is exactly the broken code at lines **604–622**.

```yaml
  - stage: Build
    variables:
      buildOutput: "..."          # scoped to this stage only
  - stage: Deploy
    jobs:
      - job: DeployJob
        steps:
          - script: echo $(buildOutput)     # empty
```

Variable scope is pipeline → stage → job. A stage variable is invisible outside its stage, and
`$( )` cannot reach across the boundary.

**Term:** *variable scope*. Only **output variables** cross stages.

</details>

---

## Q26

**Proposed solution:** Write the version to a file, publish it as a pipeline artifact from `Build`,
and read the file in `Deploy`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

It is clumsy, but it genuinely works. Deployment jobs download artifacts automatically into
`$(Pipeline.Workspace)` (line **199**), so the file is there.

**Why the exam still prefers Q24's approach**

- an output variable is a first-class pipeline value; a file must be parsed by a script
- you cannot use a file's contents in a stage-level `variables:` block or a `condition:`, because
  those are evaluated before your script runs
- it adds an artifact upload and download for a single string

**The lesson:** on "does this meet the goal" questions, judge whether it **works**, not whether it is
elegant. Two different proposals can both be Yes.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — expressions

| # | Statement | Answer |
|---|---|---|
| 1 | `${{ }}` can read an output variable from a previous stage |  |
| 2 | `$[ ]` is evaluated when the stage or job begins |  |
| 3 | `$( )` can be used inside a task input |  |
| 4 | `${{ }}` can add or remove steps from the pipeline |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `${{ }}` can read an output variable from a previous stage | **No** |
| 2 | `$[ ]` is evaluated when the stage or job begins | **Yes** |
| 3 | `$( )` can be used inside a task input | **Yes** |
| 4 | `${{ }}` can add or remove steps from the pipeline | **Yes** |

**In `challenge-20.md`:** line **351** (row 4), line **643** (rows 1 and 2), line **98** (row 3).

Row 1 is the trap and it is the single most-tested fact in this challenge. `${{ }}` runs when the
YAML is parsed — **before any stage has executed** — so there is no output to read. You get an empty
string with no error.

Row 4 is the flip side and the reason `${{ }}` exists:

```yaml
  - ${{ if parameters.publishArtifact }}:      # the steps below do not EXIST if false
      - task: DotNetCoreCLI@2
```

</details>

---

## Q28 — deployment jobs and environments

| # | Statement | Answer |
|---|---|---|
| 1 | A `deployment` job requires a `strategy` |  |
| 2 | A `deployment` job checks out the source repository by default |  |
| 3 | The `environment` property enables approvals and checks |  |
| 4 | The `environment` property provisions Azure resources |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A `deployment` job requires a `strategy` | **Yes** |
| 2 | A `deployment` job checks out the source repository by default | **No** |
| 3 | The `environment` property enables approvals and checks | **Yes** |
| 4 | The `environment` property provisions Azure resources | **No** |

**In `challenge-20.md`:** lines **177–183**.

Row 2 catches almost everyone. A regular job checks out source automatically; a **deployment job does
not**. If your deploy steps need a file from the repo, add `- checkout: self` yourself.

Row 4 matters because the name suggests otherwise. An Azure DevOps *environment* is a **logical
record** for tracking deployments and attaching checks. It creates nothing in Azure. Your `infra`
artifact (line 133) and Bicep are what provision resources.

</details>

---

## Q29 — variables and variable groups

| # | Statement | Answer |
|---|---|---|
| 1 | A stage-level variable is visible in later stages |  |
| 2 | A variable group can be scoped to a single stage |  |
| 3 | Key Vault secrets become variables of the same name |  |
| 4 | Secrets fetched from Key Vault are masked in logs |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A stage-level variable is visible in later stages | **No** |
| 2 | A variable group can be scoped to a single stage | **Yes** |
| 3 | Key Vault secrets become variables of the same name | **Yes** |
| 4 | Secrets fetched from Key Vault are masked in logs | **Yes** |

**In `challenge-20.md`:** lines **604–622** (row 1), lines **174–175** (row 2), lines **184–201**
(rows 3 and 4).

Row 2 is worth noticing in the real pipeline:

```yaml
  - stage: DeployStaging
    variables:
      - group: contoso-staging        # line 175 - stage-scoped
  - stage: DeployProduction
    variables:
      - group: contoso-production     # line 225 - different group, same variable names
```

That is how one pipeline uses `$(SqlConnectionString)` for two different databases — same expression,
different group per stage.

</details>

---

## Q30 — triggers

| # | Statement | Answer |
|---|---|---|
| 1 | `trigger:` controls continuous integration on push |  |
| 2 | `pr:` controls pull request validation |  |
| 3 | `trigger: none` disables CI triggering |  |
| 4 | Path filters can be applied to the array form `trigger: [main]` |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `trigger:` controls continuous integration on push | **Yes** |
| 2 | `pr:` controls pull request validation | **Yes** |
| 3 | `trigger: none` disables CI triggering | **Yes** |
| 4 | Path filters can be applied to the array form `trigger: [main]` | **No** |

**In `challenge-20.md`:** lines **54–71**.

Row 4 is the subtle one. The shorthand accepts branches only:

```yaml
trigger: [main, release/*]           # branches only - no paths possible
```

```yaml
trigger:                             # full form - branches AND paths (lines 54-62)
  branches:
    include: [main, "release/*"]
  paths:
    exclude: ["docs/**", "*.md"]
```

**Also know:** `pr: none` disables PR validation, and `drafts: false` (line 554) stops draft PRs from
triggering builds.

</details>

---

# Section E — Drag and drop

---

## Q31

Arrange the Azure Pipelines hierarchy from outermost to innermost.

**Items:** `steps` · `stages` · `jobs` · `tasks`

<details>
<summary>Show answer</summary>

### Answer: `stages` → `jobs` → `steps` → `tasks`

**In `challenge-20.md`:** lines **87**, **90**, **93**, **94**.

```yaml
stages:                     # line 87
  - stage: Build
    jobs:                   # line 90
      - job: BuildJob
        steps:              # line 93
          - task: UseDotNet@2      # line 94
```

**GitHub Actions has no stage layer.** Its hierarchy is `jobs` → `steps`. That missing level is why
Azure Pipelines can gate an entire stage with a `condition` while GitHub Actions must condition each
job.

</details>

---

## Q32

Match each GitHub Actions keyword to its Azure Pipelines equivalent.

| GitHub Actions | Azure Pipelines |
|---|---|
| `on: push` |  |
| `on: pull_request` |  |
| `runs-on:` |  |
| `needs:` |  |
| `run:` |  |
| `uses:` |  |
| `${{ secrets.NAME }}` |  |
| reusable workflow |  |
| `environment:` |  |

**Options:** `dependsOn:` · `environment:` · `$(NAME)` · `pool: vmImage` · `pr:` · `script:` · `task:` · `template:` · `trigger:`

<details>
<summary>Show answer</summary>

| GitHub Actions | Azure Pipelines |
|---|---|
| `on: push` | `trigger:` |
| `on: pull_request` | `pr:` |
| `runs-on:` | `pool: vmImage` |
| `needs:` | `dependsOn:` |
| `run:` | `script:` |
| `uses:` | `task:` |
| `${{ secrets.NAME }}` | `$(NAME)` |
| reusable workflow | `template:` |
| `environment:` | `environment:` |

### The same idea on both platforms

```yaml
# GitHub Actions (challenge-19.md lines 48, 77, 117)
on:
  push:
    branches: [main]
jobs:
  test:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - run: dotnet test
```

```yaml
# Azure Pipelines (challenge-20.md lines 54, 73, 139)
trigger:
  branches:
    include: [main]
pool:
  vmImage: "ubuntu-latest"
stages:
  - stage: Test
    dependsOn: Build
    jobs:
      - job: UnitTests
        steps:
          - script: dotnet test
```

Note the last row: `environment:` is the **same word on both platforms** for the same idea. That is
rare, and the exam uses it as a comfortable anchor before asking something harder.

</details>

---

## Q33

Arrange the stages of Challenge 20's pipeline in execution order.

**Items:** `DeployProduction` · `Test` · `Build` · `DeployStaging`

<details>
<summary>Show answer</summary>

### Answer: `Build` → `Test` → `DeployStaging` → `DeployProduction`

**In `challenge-20.md`:** read it off the `dependsOn` keys.

```yaml
  - stage: Build              # line 88  - no dependsOn, runs first
  - stage: Test
    dependsOn: Build          # line 139
  - stage: DeployStaging
    dependsOn: Test           # line 172
  - stage: DeployProduction
    dependsOn: DeployStaging  # line 222
```

**Default behaviour worth knowing:** with **no** `dependsOn`, a stage depends on the one declared
before it. Writing `dependsOn: []` makes a stage run **immediately, in parallel** with the first
stage — which is how you fan out.

</details>

---

## Q34

Arrange the steps of the production deployment in the order Challenge 20 defines them.

**Items:** Swap canary to production · Fetch secrets from Key Vault · Validate canary slot ·
Deploy to the canary slot

<details>
<summary>Show answer</summary>

### Answer: Fetch secrets → Deploy to canary slot → Validate canary → Swap to production

**In `challenge-20.md`:** lines **234–278**.

```yaml
                - task: AzureKeyVault@2          # 1 - line 234
                - task: AzureRmWebAppDeployment@4  # 2 - line 242, SlotName: canary (line 255)
                - script: |                       # 3 - line 257, health check the slot
                - task: AzureCLI@2                # 4 - line 267, slot swap
```

This is the **blue-green pattern** in full: deploy to an idle slot, prove it healthy, then swap. If
step 3 exits non-zero (line 262), the swap never happens and production is untouched.

Compare with staging (lines 184–218), which has no slot and no swap — staging **is** the test
environment.

</details>

---

## Q35

Match each item to where it belongs.

| Item | Where |
|---|---|
| Files needed by a later stage |  |
| A string produced by one stage, needed by another |  |
| A secret shared by all stages |  |
| A value that differs per stage |  |

**Options:** output variable · pipeline **artifact** · stage-scoped variable group · variable group linked to Key Vault

<details>
<summary>Show answer</summary>

| Item | Where |
|---|---|
| Files needed by a later stage | pipeline **artifact** |
| A string produced by one stage, needed by another | **output variable** |
| A secret shared by all stages | **variable group linked to Key Vault** |
| A value that differs per stage | **stage-scoped variable group** |

### Where each appears in `challenge-20.md`

```yaml
          - task: PublishPipelineArtifact@1     # artifact - line 123
              artifact: "drop"

              echo "##vso[task.setvariable variable=buildVersion;isOutput=true]1.2.3"   # line 637

variables:
  - group: contoso-common                        # pipeline-wide - line 77

  - stage: DeployStaging
    variables:
      - group: contoso-staging                   # stage-scoped - line 175
```

**The decision rule:** *files* → artifact. *A single value* → output variable. *A secret* → Key Vault
via a variable group. *Differs per environment* → scope the group to the stage.

</details>

---

# Section F — Hot area

---

## Q36

```yaml
  - stage: DeployStaging
    [BLANK 1]: Test
    [BLANK 2]: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
```

- **BLANK 1:** `dependsOn` / `needs` / `after` / `requires`
- **BLANK 2:** `condition` / `if` / `when` / `filter`

<details>
<summary>Show answer</summary>

### Answer: `dependsOn`, `condition`

**In `challenge-20.md`:** lines **172–173**.

`needs` and `if` are **GitHub Actions** keywords — offered here because the exam constantly tests
whether you can keep the two platforms apart. `after`, `requires`, `when` and `filter` do not exist
on either platform.

</details>

---

## Q37

```yaml
      - [BLANK 1]: DeployStagingJob
        [BLANK 2]: "contoso-staging"
        strategy:
          [BLANK 3]:
            deploy:
              steps:
```

- **BLANK 1:** `job` / `deployment` / `stage` / `task`
- **BLANK 2:** `environment` / `pool` / `container` / `resource`
- **BLANK 3:** `runOnce` / `matrix` / `parallel` / `single`

<details>
<summary>Show answer</summary>

### Answer: `deployment`, `environment`, `runOnce`

**In `challenge-20.md`:** lines **177–181**.

A plain `job` has no `environment` property, so BLANK 1 and BLANK 2 must go together. `matrix` is for
regular jobs; `parallel` applies only to VM resources; `single` does not exist.

</details>

---

## Q38

```yaml
          - task: [BLANK 1]
            inputs:
              KeyVaultName: "kv-contoso-staging"
              SecretsFilter: "SqlConnectionString,AppInsightsKey"

          - script: echo [BLANK 2]
```

- **BLANK 1:** `AzureKeyVault@2` / `AzureCLI@2` / `AzureRmWebAppDeployment@4` / `UseDotNet@2`
- **BLANK 2:** `$(SqlConnectionString)` / `$(keyVault.SqlConnectionString)` /
  `${{ secrets.SqlConnectionString }}` / `$(secrets.SqlConnectionString)`

<details>
<summary>Show answer</summary>

### Answer: `AzureKeyVault@2`, `$(SqlConnectionString)`

**In `challenge-20.md`:** lines **184–189** and **201**.

The secret becomes a pipeline variable of the **same name**, with no prefix. The `secrets.` forms are
GitHub Actions syntax, and `${{ }}` is compile time so it could not hold a runtime-fetched value
anyway.

</details>

---

## Q39

```yaml
parameters:
  - name: runTests
    type: [BLANK 1]
    default: true

steps:
  - ${{ if eq(parameters.runTests, [BLANK 2]) }}:
      - script: echo "Running tests"
```

- **BLANK 1:** `bool` / `boolean` / `binary` / `flag`
- **BLANK 2:** `true` / `'true'` / `"true"` / `$(true)`

<details>
<summary>Show answer</summary>

### Answer: `boolean`, `true`

**In `challenge-20.md`:** the errors at lines **573** and **578**; the fixes at **593** and **598**.

Both blanks are the same trap in two forms. `bool` is not a valid type name, and `'true'` is a
**string** — a boolean never equals a string, so the condition is silently always false and the steps
just never appear. No error, no warning.

</details>

---

## Q40

```yaml
[BLANK 1]:
  repositories:
    - repository: templates
      type: github
      name: contoso/pipeline-templates
      [BLANK 2]: contoso-github-connection

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - template: dotnet/build.yml[BLANK 3]
```

- **BLANK 1:** `resources` / `variables` / `extends` / `imports`
- **BLANK 2:** `endpoint` / `connection` / `serviceConnection` / `auth`
- **BLANK 3:** `@templates` / `#templates` / `:templates` / `/templates`

<details>
<summary>Show answer</summary>

### Answer: `resources`, `endpoint`, `@templates`

**In `challenge-20.md`:** lines **470–476** and **502**.

The `@alias` suffix is the part people forget. Without it, Azure Pipelines looks for
`dotnet/build.yml` in the **current** repository and fails with a file-not-found error at compile
time.

</details>

---

## Q41

```yaml
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: |
              echo "##vso[task.setvariable variable=buildVersion;[BLANK 1]]1.2.3"
            [BLANK 2]: setVersion

  - stage: Deploy
    dependsOn: Build
    variables:
      buildVersion: [BLANK 3] stageDependencies.Build.BuildJob.outputs['setVersion.buildVersion'] ]
```

- **BLANK 1:** `isOutput=true` / `isSecret=true` / `global=true` / `scope=pipeline`
- **BLANK 2:** `name` / `id` / `displayName` / `label`
- **BLANK 3:** `$[` / `${{` / `$(` / `@[`

<details>
<summary>Show answer</summary>

### Answer: `isOutput=true`, `name`, `$[`

**In `challenge-20.md`:** lines **637–643**.

Three separate traps in one block:

- `isSecret=true` masks a value in logs; it does not make it an output
- `displayName` is the label shown in the UI — it is **not** the reference key. Only `name` is
- `${{` is compile time and would evaluate before Build ran

**This block is the single most valuable thing to memorise from Challenge 20.**

</details>

---

# Section G — Case study

## Case study: Contoso Ltd enterprise team

### Background

Contoso's enterprise team uses Azure DevOps. They maintain a .NET 8 Web API deployed to Azure App
Service across staging and production. Source code lives in a **GitHub** repository.

### Requirements

**Build**

- The pipeline must build only when `src/**` or `pipelines/**` change
- Documentation changes must never trigger a build
- Test results and code coverage must appear in the run summary even when tests fail

**Deployment**

- Staging deploys only from `main`
- Production deploys to a `canary` slot, is health-checked, then swapped
- Each environment uses its own database connection string, stored in Azure Key Vault

**Reuse and governance**

- Three teams must share one build definition
- Production deployment must be approved before it starts

---

## Q42

You must meet the requirement that documentation changes never trigger a build. What should you
configure?

- A. `trigger:` with `paths: exclude:`
- B. `pr:` with `paths: exclude:`
- C. A `condition` on the Build stage checking changed files
- D. `trigger: none` with a scheduled build

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-20.md`:** lines **59–62**.

```yaml
trigger:
  branches:
    include: [main, "release/*"]
  paths:
    exclude:
      - docs/**
      - "*.md"
```

**Why the others fail**

- **B** — `pr:` filters pull request validation, a different event. Line 64 shows it doing that job
- **C** — a condition evaluates **after** the run has started. An agent is already allocated and
  billed. Path filters stop the run from starting at all
- **D** — disables CI entirely, which breaks the other build requirements

</details>

---

## Q43

You must meet the requirement for test results and coverage. Which **two** configurations are needed?
(Choose two.)

- A. `PublishTestResults@2` with `condition: always()`
- B. `PublishCodeCoverageResults@2` pointing at the cobertura file
- C. `PublishPipelineArtifact@1` with the test folder
- D. `continueOnError: true` on the test task
- E. A `Test` stage with `dependsOn: []`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-20.md`:** lines **156–168**.

```yaml
          - task: PublishTestResults@2
            condition: always()                    # A - publishes even after failures
            inputs:
              testResultsFormat: "VSTest"
              testResultsFiles: "**/*.trx"

          - task: PublishCodeCoverageResults@2     # B
            inputs:
              summaryFileLocation: "...coverage.cobertura.xml"
```

**Why the others fail**

- **C** — makes files downloadable but produces no results or coverage tab
- **D** — would let failing tests **pass the pipeline**. That silently defeats the quality gate,
  which is the opposite of what the requirement wants
- **E** — `dependsOn: []` would run Test in parallel with Build, so there would be nothing to test

</details>

---

## Q44

You must meet the requirement that each environment uses its own connection string from Key Vault.
What should you configure?

- A. One variable group per environment, each scoped to its stage
- B. One pipeline-level variable group holding both connection strings
- C. A `${{ if }}` expression selecting the connection string per stage
- D. Two separate pipelines, one per environment

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-20.md`:** lines **174–175** and **224–225**, with the Key Vault tasks at **188** and
**238**.

```yaml
  - stage: DeployStaging
    variables:
      - group: contoso-staging        # line 175
  - stage: DeployProduction
    variables:
      - group: contoso-production     # line 225
```

Same expression `$(SqlConnectionString)` in both stages, different value — because the **scope**
differs. That is the same model as GitHub environment secrets in Challenge 19.

**Why the others fail**

- **B** — both values in one scope means two different variable names and per-stage logic everywhere
- **C** — compile-time selection of a **secret** would require the secret to be known at parse time,
  which defeats fetching it from Key Vault at runtime
- **D** — duplicating a whole pipeline to change one value

</details>

---

## Q45

You must meet the requirement that three teams share one build definition. What should you create?

- A. A `steps:` template with parameters, referenced by each team's pipeline
- B. A task group in the classic editor
- C. A variable group containing the build commands
- D. A separate pipeline that each team triggers manually

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-20.md`:** the template at lines **317–365**, used at lines **437–440**.

```yaml
parameters:
  - name: dotnetVersion
    type: string
    default: "8.0.x"
  - name: projectPath
    type: string
```

```yaml
          - template: templates/build-template.yml
            parameters:
              dotnetVersion: "8.0.x"
              projectPath: "src/Contoso.Api/Contoso.Api.csproj"
```

Parameters are what make it shareable — each team passes its own project path.

**Why the others fail**

- **B** — task groups are a **classic** pipeline feature. They do not work in YAML, and Challenge 37
  is specifically about migrating them **to** templates
- **C** — variable groups hold values, not steps
- **D** — manual triggering is not reuse

</details>

---

## Q46

You must meet the requirement that production deployment is approved before it starts. What should
you configure?

- A. An approval check on the `contoso-production` environment
- B. A branch policy requiring reviewers on `main`
- C. A `condition` on the DeployProduction stage
- D. A manual `trigger` for the production stage

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-20.md`:** line **229**.

```yaml
      - deployment: DeployProductionJob
        environment: "contoso-production"      # approvals are configured on this environment
```

**Why the others fail**

- **B** — branch policies gate **merging code**, not deploying it. Same boundary as Challenge 19 Q43
- **C** — a condition is automatic logic. It cannot ask a human
- **D** — there is no per-stage `trigger` key. `trigger` is pipeline-level (line 54)

**Where approvals live:** Pipelines → Environments → select → Approvals and checks. Not in YAML —
which is the point, because someone editing the YAML cannot remove them.

</details>

---

## Q47

The pipeline builds from GitHub. A pull request must trigger validation, but draft pull requests must
not.

Which configuration meets the requirement?

- A. `pr:` with `drafts: false`
- B. `trigger:` with `drafts: false`
- C. A branch policy in Azure Repos
- D. `pr: none` with a status check

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-20.md`:** lines **547–554**.

```yaml
pr:
  branches:
    include:
      - main
  paths:
    include:
      - src/**
  drafts: false
```

**Why the others fail**

- **B** — `trigger:` handles pushes. Drafts are a pull request concept, so the key has no meaning
  there
- **C** — the code is in **GitHub**, not Azure Repos. Azure Repos branch policies do not apply
- **D** — `pr: none` disables PR validation completely, so nothing would run on any PR

</details>

---

## Q48

A deployment job fails because a Bicep file from the repository is missing at deploy time.

What is the cause?

- A. Deployment jobs do not check out the source repository by default
- B. The artifact was published to the wrong directory
- C. The service connection lacks permission to read the repository
- D. `$(Pipeline.Workspace)` is only available in regular jobs

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-20.md`:** deployment jobs start at lines **177** and **227**. Notice there is **no**
`checkout` step in either.

```yaml
      - deployment: DeployStagingJob
        environment: "contoso-staging"
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureKeyVault@2       # no checkout anywhere above this
```

A **regular** job checks out source automatically. A **deployment** job does not. Either add
`- checkout: self`, or publish the files as an artifact — which is exactly why line **130** publishes
the `infra` folder:

```yaml
          - task: PublishPipelineArtifact@1
            inputs:
              targetPath: "infra"
              artifact: "infra"
```

**Why the others fail**

- **B** — the artifact question is a red herring; the file was never published *or* checked out
- **C** — service connections authenticate to **Azure**, not to the repository
- **D** — `$(Pipeline.Workspace)` works in both. Line 199 uses it inside a deployment job

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **`${{ }}` cannot read runtime output** | Q1, Q2, Q27 | Compile time runs before any stage. Use `$[ ]` |
| **Custom `condition` replaces `succeeded()`** | Q7, Q20 | Always write `and(succeeded(), ...)` |
| **`Build.SourceBranch` is the full ref** | Q7 | `refs/heads/main`, not `main`. Short name is `Build.SourceBranchName` |
| **Deployment jobs do not check out source** | Q19, Q28, Q48 | Add `- checkout: self` or publish an artifact |
| **`type: bool` is invalid** | Q12, Q39 | It is `boolean`. And compare to `true`, never `'true'` |
| **GitHub keywords offered as distractors** | Q11, Q36, Q38 | `needs`, `if`, `type: choice`, `secrets.` are all GitHub Actions |
| **Stage variables do not cross stages** | Q13, Q25, Q29 | Only output variables cross. Scope is pipeline → stage → job |
| **`displayName` is not the reference key** | Q41 | Only `name:` makes a step referenceable |
| **`@alias` missing on external templates** | Q40 | `template: path.yml@repoAlias` |
| **Environments do not provision anything** | Q28 | They are a tracking and approval record |
| **`${{ if }}` vs `condition:`** | Q21 | Conditional insertion removes the step. `condition` skips it |
| **Path filters vs a stage condition** | Q42 | A filter stops the run starting. A condition burns an agent first |

---

# The eight blocks to memorise

Line numbers are in `challenge-20.md`.

```yaml
# 1. Cross-stage output variable  (lines 637-643) - THE most important block
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: echo "##vso[task.setvariable variable=buildVersion;isOutput=true]1.2.3"
            name: setVersion
  - stage: Deploy
    dependsOn: Build
    variables:
      buildVersion: $[ stageDependencies.Build.BuildJob.outputs['setVersion.buildVersion'] ]

# 2. Deployment job  (lines 177-183)
      - deployment: DeployStagingJob
        environment: "contoso-staging"
        strategy:
          runOnce:
            deploy:
              steps:

# 3. Stage gate  (lines 172-173)
    dependsOn: Test
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))

# 4. Key Vault to variable  (lines 184-201)
                - task: AzureKeyVault@2
                  inputs:
                    KeyVaultName: "kv-contoso-staging"
                    SecretsFilter: "SqlConnectionString,AppInsightsKey"
                # then use it as:  $(SqlConnectionString)

# 5. Trigger with path filters  (lines 54-62)
trigger:
  branches:
    include: [main, "release/*"]
  paths:
    exclude: ["docs/**", "*.md"]

# 6. Template with typed parameters  (lines 318-329, 437-440)
parameters:
  - name: projectPath
    type: string
  - name: publishArtifact
    type: boolean
    default: true
steps:
  - ${{ if parameters.publishArtifact }}:
      - task: DotNetCoreCLI@2

# 7. External template repository  (lines 470-476, 502)
resources:
  repositories:
    - repository: templates
      type: github
      name: contoso/pipeline-templates
      endpoint: contoso-github-connection
# used as:  - template: dotnet/build.yml@templates

# 8. Test results and coverage  (lines 156-168)
          - task: PublishTestResults@2
            condition: always()
          - task: PublishCodeCoverageResults@2
            inputs:
              summaryFileLocation: "**/coverage.cobertura.xml"
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 20 is exam-ready. Move to Challenge 21 |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 1 and 3, then retake this |
| Below 30 | Redo the whole challenge, typing every line. Do not move on yet |

Record your result in `AZ-400-Learning-Log.md` under Challenge 20, and note **which trap** caught you.

:::tip The one thing

If you take a single block from this challenge into the exam, make it the **cross-stage output
variable**. `isOutput=true` + a step `name` + `dependsOn` + `$[ stageDependencies... ]`. It generates
more exam questions than anything else here.

:::
