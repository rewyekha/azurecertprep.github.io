---
sidebar_position: 1.5
toc_max_heading_level: 2
title: "Challenge 19: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 19 — AZ-400 exam questions

**48 questions** built only from what Challenge 19 covers, in the seven shapes the live exam uses.

Every question is followed immediately by its answer, **why each wrong option is wrong**, and the
**line numbers in `challenge-19.md`** where the real syntax lives. Open that file beside this one.

| Section | Shape | Questions |
|---|---|---|
| A | Single answer | 1–16 |
| B | Multiple answer (choose two / three) | 17–23 |
| C | Repeated scenario — "Does this meet the goal?" | 24–26 |
| D | Yes/No statement grid | 27–30 |
| E | Drag and drop | 31–35 |
| F | Hot area — complete the YAML | 36–41 |
| G | Case study | 42–48 |

The **trap index**, the **eight blocks to memorise**, and **scoring** are at the very end.

:::tip How to use this

First pass: read question → read answer → read why the others fail. Do not test yourself yet.
Second pass, a day later: cover everything below the options and answer from memory.

:::

---

# Section A — Single answer

---

## Q1

A workflow has a `build` job that produces a version string, and a `docker` job that must consume it.
The `docker` job reads `${{ needs.build.outputs.version }}` but always receives an empty string. The
step that generates the value writes to `$GITHUB_OUTPUT` correctly.

What is the most likely cause?

- A. The `docker` job is missing a `needs: build` declaration
- B. The `build` job does not declare the value under a job-level `outputs` key
- C. The value must be written to `$GITHUB_ENV` instead of `$GITHUB_OUTPUT`
- D. Job outputs cannot be passed between jobs that run on different runners

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-19.md`:** lines **78–80** (`build` job `outputs`), line **98** (`id: version`),
lines **102–103** (`$GITHUB_OUTPUT`), line **167** (`docker` needs).

A step output and a job output are **two different things**. The step stored its value, but the job
never published it, so nothing crosses the job boundary.

**Broken:**

```yaml
jobs:
  build:
    steps:
      - name: Generate version info
        id: version
        run: echo "version=1.2.3" >> $GITHUB_OUTPUT
```

**Fixed — line 78 in the challenge:**

```yaml
jobs:
  build:
    outputs:
      version: ${{ steps.version.outputs.version }}
      sha_short: ${{ steps.version.outputs.sha_short }}
```

**Why the others fail**

- **A** — the question says the value is *read* as `needs.build.outputs.version`. If `needs` were
  missing, the `needs` context would not exist at all and the workflow would fail to parse, not
  return empty
- **C** — `$GITHUB_ENV` sets an environment variable for **later steps in the same job**. It never
  travels to another job
- **D** — false. Outputs travel through GitHub's service, not the runner filesystem. Different
  runners are irrelevant

**Term:** *step output* → *job output* → *needs context*.

</details>

---

## Q2

You need a workflow that can be started manually, and the person starting it must select a target
environment from a fixed list of `staging` or `production`.

Which trigger configuration should you use?

- A. `on: repository_dispatch` with a `client_payload` object
- B. `on: workflow_dispatch` with an input of `type: choice` and an `options` list
- C. `on: workflow_call` with a required input of `type: string`
- D. `on: schedule` combined with a job-level `if` condition

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-19.md`:** lines **53–62**.

```yaml
  workflow_dispatch:
    inputs:
      environment:
        description: "Target environment"
        required: true
        default: "staging"
        type: choice
        options:
          - staging
          - production
```

`type: choice` + `options` renders a **dropdown** in the Actions UI. Read it as
`${{ inputs.environment }}`.

**Why the others fail**

- **A** — `repository_dispatch` is triggered by an external **API call**, not by a person clicking a
  button. There is no UI form
- **C** — `workflow_call` makes this workflow callable *by another workflow*. It adds no button. Also
  `type: string` gives a free-text box, not a fixed list
- **D** — `schedule` runs on a cron. Nobody chooses anything

**Gotcha:** the "Run workflow" button only appears once the workflow file exists on the **default
branch**.

</details>

---

## Q3

A workflow job pushes a container image to `ghcr.io` and fails with `denied: permission_denied`. The
job authenticates using `secrets.GITHUB_TOKEN`.

What should you add to resolve the failure?

- A. A personal access token stored as a repository secret
- B. A `packages: write` entry under the `permissions` key
- C. A `contents: write` entry under the `permissions` key
- D. A classic PAT with the `write:packages` scope in the org

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-19.md`:** lines **170–172** (the `docker` job), and Break & fix Exercise 2 at lines
**430–448**.

```yaml
  docker:
    permissions:
      contents: read
      packages: write
```

The token was never the problem. `GITHUB_TOKEN` already exists — it just had no permission to write
packages.

**Why the others fail**

- **A** and **D** — both add a credential you do not need. A PAT is long-lived, must be rotated, and
  is a bigger blast radius than a token that expires with the run. On the exam, adding a PAT when the
  built-in token would work is almost always wrong
- **C** — `contents` controls the **repository files** (code, releases). Packages are a separate
  permission scope

**Term:** *least-privilege workflow permissions*. Built-in token failing → add a permission, not a
secret.

</details>

---

## Q4

You are writing a composite action. One of its steps uses `run:` to execute a shell command. The
action fails to load with a validation error.

What is missing from the step?

- A. An `id` property
- B. A `shell` property
- C. A `working-directory` property
- D. A `continue-on-error` property

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-19.md`:** lines **309** and **322** — every `run` step in the composite action has
`shell: bash`.

```yaml
    - name: Install dependencies
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: npm ci
```

In a normal workflow `shell:` is optional and defaults per operating system. **Inside a composite
action it is mandatory on every `run` step.**

**Why the others fail**

- **A** — `id` is only needed if another step reads this step's output. It is never required
- **C** — `working-directory` is optional; the challenge uses it, but omitting it just defaults to
  the workspace root
- **D** — `continue-on-error` changes failure behaviour. Nothing to do with loading

**Term:** *composite action step requirements*. This is the single most common authoring error.

</details>

---

## Q5

Your team wants to share six identical setup steps across ten workflows **inside the same job**,
without adding an extra runner.

What should you create?

- A. A reusable workflow invoked with `workflow_call`
- B. A composite action stored in `.github/actions`
- C. A starter workflow in the organization `.github` repository
- D. A job template referenced with the `extends` keyword

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-19.md`:** Task 4 begins at line **276**; the action file starts at line **281**; it is
used at lines **330–333**.

```yaml
      - name: Setup project
        uses: ./.github/actions/setup-node-project
        with:
          node-version: ${{ env.NODE_VERSION }}
```

That `uses:` sits under `steps:` — it injects six steps into the job you are already in. Same runner,
same workspace.

**Why the others fail**

- **A** — a reusable workflow is called at **job** level and always runs as its own job on its own
  runner. That directly violates "no extra runner"
- **C** — a starter workflow is a **template** shown in the "New workflow" UI. It is copied once when
  someone creates a workflow. Edit it later and existing workflows do not change
- **D** — `extends` is Azure Pipelines syntax. GitHub Actions has no such keyword

**Term:** *composite action = steps, same runner. Reusable workflow = jobs, own runner.*

</details>

---

## Q6

A `docker` job must run only when a commit is pushed to `main`, and never on a pull request.

Which job-level condition meets the requirement?

- A. `if: github.ref_name == 'main'`
- B. `if: github.event_name == 'push' && github.ref == 'refs/heads/main'`
- C. `if: github.base_ref == 'main' && github.event_name == 'push'`
- D. `if: startsWith(github.ref, 'refs/pull/') == false`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-19.md`:** line **169**.

```yaml
  docker:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

You need **both** halves: the event excludes pull requests, the ref pins the branch.

**Why the others fail**

- **A** — checks only the branch name. It does not exclude other event types, and `ref_name` behaves
  awkwardly on PR refs
- **C** — `github.base_ref` is **only populated on pull requests**. On a push it is empty, so this
  condition is always false and the job never runs
- **D** — excludes PRs but allows a push to *any* branch. A push to `feature/x` would still build and
  push an image

**Values to know:** push to main → `github.ref` = `refs/heads/main`. Pull request → `github.ref` =
`refs/pull/42/merge`.

</details>

---

## Q7

A repository variable named `APP_NAME` was created with `gh variable set APP_NAME`. A step references
it as `${{ env.APP_NAME }}` and receives an empty value.

How should the step reference the value?

- A. `${{ vars.APP_NAME }}`
- B. `${{ secrets.APP_NAME }}`
- C. `${{ github.APP_NAME }}`
- D. `${{ inputs.APP_NAME }}`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-19.md`:** created at line **349**; the broken form is ERROR 4 at line **390**; the
fix is at line **426**.

```bash
gh variable set APP_NAME --body "contoso-api"      # line 349
```

```yaml
          app-name: ${{ env.APP_NAME }}      # line 390 - empty
          app-name: ${{ vars.APP_NAME }}     # line 426 - correct
```

Compare with lines **69–72**, where `env` values *are* defined in the workflow:

```yaml
env:
  NODE_VERSION: "20.x"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
```

`NODE_VERSION` is an `env` value. `APP_NAME` is a repository **variable**. Different contexts.

**Why the others fail**

- **B** — `secrets` reads secrets. `APP_NAME` was created as a variable, so it is not there
- **C** — the `github` context holds event metadata (`github.repository`, `github.ref`,
  `github.actor`). You cannot add your own keys to it
- **D** — `inputs` holds `workflow_dispatch` or `workflow_call` inputs, not repository settings

**Term:** *`vars` for repository/environment variables. `env` only for values the workflow defined.*

</details>

---

## Q8

A workflow defines `strategy: matrix: test-type: [unit, integration]` on its `test` job.

How many jobs does this produce, and how do they execute?

- A. One job that loops through both values sequentially
- B. Two jobs that run in parallel, one per matrix value
- C. Two jobs that run sequentially in the declared order
- D. One job per runner label available in the pool

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-19.md`:** lines **120–122**, used at line **147**.

```yaml
    strategy:
      matrix:
        test-type: [unit, integration]
```

```yaml
      - name: Run ${{ matrix.test-type }} tests
        run: npm run test:${{ matrix.test-type }}
```

One job definition, two values → **two jobs, in parallel, on two separate runners**. In your run
graph this appeared as "Matrix: Run tests".

**Why the others fail**

- **A** and **C** — a matrix never runs sequentially by default. If you *want* that, you set
  `max-parallel: 1`
- **D** — runner labels are unrelated. The count comes from the matrix values

**Defaults worth memorising:** parallel by default, `fail-fast: true` by default, `max-parallel` caps
concurrency, and each leg gets its **own** workspace.

</details>

---

## Q9

The `test` job declares a `services:` block containing a Redis container with health-check options.

What is the purpose of the `--health-cmd` and `--health-retries` options?

- A. They restart the container automatically if the application crashes
- B. They delay the job's steps until the service reports it is ready
- C. They publish container health metrics to the workflow run summary
- D. They limit how long the service container is allowed to run

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-19.md`:** lines **123–133**.

```yaml
    services:
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
```

The runner starts the container, then runs `redis-cli ping` every 10 seconds, up to 5 times. Only
when it succeeds does step 1 of your job begin.

**Why the others fail**

- **A** — Docker health checks **report** status; they do not restart anything. Restart policies are
  a different setting
- **C** — nothing is published to the run summary. The health result is internal to the runner
- **D** — that would be a timeout on the job, not a health check

**Why it matters:** without this, your first integration test hits a database that is still booting.
That is the classic flaky-test cause Challenge 34 revisits.

</details>

---

## Q10

You must authenticate a workflow to Azure **without storing any long-lived credential** in the
repository.

Which permission must the job declare?

- A. `id-token: write`
- B. `contents: write`
- C. `actions: write`
- D. `deployments: write`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-19.md`:** line **229** shows the *credential-storing* version you must move away
from.

**What the challenge writes — stores a password:**

```yaml
      - name: Log in to Azure
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
```

**The secretless version:**

```yaml
  deploy-staging:
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

`id-token: write` lets the job request a short-lived OIDC token from GitHub. Azure trusts it through
a federated credential. Nothing secret is stored.

**Why the others fail**

- **B** — `contents` controls repository files
- **C** — `actions` controls workflow runs and artifacts via the API
- **D** — `deployments` lets you create deployment records. Related to environments, unrelated to
  authenticating to Azure

**Memorise this block.** Security auth is your weakest domain, and forgetting `id-token: write` is
the number-one OIDC failure.

</details>

---

## Q11

A deployment job declares `environment: name: production`. A required reviewer is configured on that
environment.

When does the approval gate take effect?

- A. When the workflow file is committed to the default branch
- B. When the job reaches the front of the queue, before its first step runs
- C. After the job's first step completes and before the deploy step
- D. Only when the workflow is triggered by `workflow_dispatch`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-19.md`:** lines **258–260**.

```yaml
  deploy-production:
    needs: deploy-staging
    environment:
      name: production
      url: https://contoso-api.azurewebsites.net
```

The job is **held in the queue**. No runner is assigned, no checkout happens, nothing executes until
someone approves.

**Why the others fail**

- **A** — committing a file triggers nothing. The gate is on the environment, evaluated at run time
- **C** — nothing runs first. If a step ran before approval, an unapproved deploy could already have
  side effects
- **D** — the gate applies to every trigger. It is a property of the environment, not the event

**Term:** *environment protection rule*. It gates the **job**, not a step.

</details>

---

## Q12

Two secrets named `DB_CONNECTION_STRING` exist: one scoped to the `staging` environment and one
scoped to the `production` environment. A job declares `environment: name: staging`.

Which value does `${{ secrets.DB_CONNECTION_STRING }}` resolve to in that job?

- A. The staging value
- B. The production value
- C. An empty string, because environment secrets need a prefix
- D. Whichever secret was created most recently

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-19.md`:** lines **345–346** create them; lines **224–225** declare the environment.

```bash
gh secret set DB_CONNECTION_STRING --env staging    --body "Server=staging-db..."   # line 345
gh secret set DB_CONNECTION_STRING --env production --body "Server=prod-db..."      # line 346
```

```yaml
  deploy-staging:
    environment:
      name: staging          # this line decides which secret you get
```

**Why the others fail**

- **B** — nothing about production applies. The job did not declare that environment
- **C** — there is no prefix syntax. The same expression resolves differently per environment, which
  is the whole point
- **D** — creation order is irrelevant. Scope decides

**The important corollary:** the `build` job has **no** `environment:` (line 75), so it cannot read
either value — even though it is in the same workflow file.

</details>

---

## Q13

The `docker` job uses `cache-from: type=gha` and `cache-to: type=gha,mode=max`.

What does this configuration do?

- A. Stores the built image in GitHub Packages for later jobs
- B. Stores Docker layer cache in the GitHub Actions cache backend
- C. Caches npm dependencies used inside the Dockerfile
- D. Reuses the previous job's runner filesystem between runs

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-19.md`:** lines **208–209**.

```yaml
      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

`type=gha` means "use the GitHub Actions cache service as the storage backend for Docker layer
cache". Unchanged layers are restored instead of rebuilt.

`mode=max` also caches **intermediate** layers — which matters for your multi-stage Dockerfile, where
the build-stage layers are the expensive ones.

**Why the others fail**

- **A** — pushing the image is `push: true` at line 205. Cache and image are separate things
- **C** — npm caching is `actions/setup-node` with `cache: "npm"` (line 89). That runs on the runner,
  not inside the image build
- **D** — runners are ephemeral. Nothing on their filesystem survives

**Term:** *buildx layer cache backend*. Different from `actions/cache`, which caches files.

</details>

---

## Q14

In a step that uses `actions/setup-node@v4`, you add `cache: "npm"`.

What does this setting cache?

- A. The `node_modules` directory in the workspace
- B. The global npm download cache, keyed on the lock file
- C. The built output produced by `npm run build`
- D. The Node.js runtime binary for reuse across jobs

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-19.md`:** lines **86–89**, and the manual equivalent in the composite action at lines
**312–319**.

```yaml
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"
```

It caches the **global npm download cache** (`~/.npm`), keyed on the hash of your lock file. Your
composite action does the same thing by hand, which is why line **317** reads:

```yaml
        key: ${{ runner.os }}-node-${{ inputs.node-version }}-${{ hashFiles('**/package-lock.json') }}
```

Change a dependency → the lock file hash changes → the key changes → cache miss.

**Why the others fail**

- **A** — `npm ci` **deletes and recreates** `node_modules` every time by design. Caching it would
  fight the tool. The download cache is what makes `npm ci` fast
- **C** — build output goes to an artifact (line 109), not a cache
- **D** — `setup-node` downloads the runtime separately; that is tool caching, not this setting

</details>

---

## Q15

A container image build fails because a file that exists in the repository is reported as not found
during `COPY`.

What should you check first?

- A. The `.gitignore` file
- B. The `.dockerignore` file
- C. The runner's disk space quota
- D. The `permissions` block of the job

<details>
<summary>Show answer</summary>

### Answer: B

**This is the failure you actually hit in Task 2.**

Your `.dockerignore` contained `tests`. Your `Dockerfile` contained:

```dockerfile
COPY . .
RUN npm run lint && npm test
```

Result:

```text
No tests found, exiting with code 1
11 files checked. testMatch: ... - 0 matches
```

The tests were in git and still were not in the image. **The build context is your repository minus
everything `.dockerignore` excludes.**

**Why the others fail**

- **A** — `.gitignore` controls what git tracks. The question says the file *is* in the repository,
  so git already has it
- **C** — disk exhaustion gives "no space left on device", not "not found"
- **D** — permissions govern API access from the workflow, not file visibility to the Docker daemon

**Term:** *build context*. `.gitignore` → git. `.dockerignore` → the Docker daemon.

</details>

---

## Q16

You must map a GitHub Actions workflow to its Azure Pipelines equivalent.

Which Azure Pipelines keyword corresponds to `runs-on:`?

- A. `trigger:`
- B. `pool:`
- C. `stage:`
- D. `container:`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-19.md`:** line **77** (`runs-on: ubuntu-latest`).

```yaml
# GitHub Actions
  build:
    runs-on: ubuntu-latest
```

```yaml
# Azure Pipelines
  - job: build
    pool:
      vmImage: 'ubuntu-latest'
```

**Why the others fail**

- **A** — `trigger:` is the Azure Pipelines equivalent of `on:`, not `runs-on:`
- **C** — `stage:` groups jobs. GitHub Actions has no stage concept at all
- **D** — `container:` runs steps inside a container **on** an agent. It does not choose the agent

**Term:** memorise the full mapping table (see Q32). It generates questions in every domain.

</details>

---

# Section B — Multiple answer

---

## Q17

A step writes a value to `$GITHUB_OUTPUT`, and a later job must read it.

Which **three** items are required? (Choose three.)

- A. The producing step must have an `id` property
- B. The producing job must declare a job-level `outputs` mapping
- C. The consuming job must declare `needs` on the producing job
- D. The value must also be written to `$GITHUB_ENV`
- E. The consuming job must run on the same runner label
- F. The producing job must set `continue-on-error: false`

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-19.md`:** line **98** (A), lines **78–80** (B), line **167** (C).

```yaml
  build:
    outputs:
      version: ${{ steps.version.outputs.version }}   # B - line 78
    steps:
      - name: Generate version info
        id: version                                   # A - line 98
        run: echo "version=1.2.3" >> $GITHUB_OUTPUT   # line 102

  docker:
    needs: [build, test]                              # C - line 167
    steps:
      - run: echo ${{ needs.build.outputs.version }}
```

**Why the others fail**

- **D** — `$GITHUB_ENV` sets an env var for later steps **in the same job**. It stops at the job
  boundary. Writing to both changes nothing
- **E** — outputs travel through GitHub's service, not the filesystem. Runner labels are irrelevant
- **F** — `continue-on-error` controls what happens when a step fails. It has no role in data flow

**Term:** *the three-part output chain*. Break any link and you get `""` with no error message.

</details>

---

## Q18

Which **two** statements about composite actions are correct? (Choose two.)

- A. Every `run` step must specify a `shell` value
- B. They execute inside the calling job on the same runner
- C. They are invoked with the `workflow_call` trigger
- D. They appear as a separate job in the run graph
- E. They require an `on:` key at the top of the file

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-19.md`:** lines **299–324** (the action), lines **330–333** (the call).

```yaml
runs:
  using: "composite"        # line 300
  steps:
    - name: Install dependencies
      shell: bash           # line 322 - A
      run: npm ci
```

```yaml
      - name: Setup project
        uses: ./.github/actions/setup-node-project    # line 331 - B, called from steps:
```

**Why the others fail**

- **C** — `workflow_call` is the **reusable workflow** trigger. A composite action has no triggers at
  all; it is invoked by `uses:` inside `steps:`
- **D** — it never appears as its own job. Its steps show inline within the calling job
- **E** — `on:` belongs to workflows. An action file has `name`, `inputs`, `outputs`, `runs`

</details>

---

## Q19

You want a container image tagged with the commit SHA, and also tagged `latest`, but only when the
build runs on the default branch.

Which **two** `docker/metadata-action` tag entries meet the requirement? (Choose two.)

- A. `type=sha,prefix=`
- B. `type=raw,value=latest,enable={{is_default_branch}}`
- C. `type=ref,event=pr`
- D. `type=schedule,pattern=nightly`
- E. `type=raw,value=latest`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-19.md`:** lines **195–198**.

```yaml
          tags: |
            type=sha,prefix=                                       # line 196 - A
            type=semver,pattern={{version}},value=${{ needs.build.outputs.version }}
            type=raw,value=latest,enable={{is_default_branch}}      # line 198 - B
```

**Why the others fail**

- **C** — `type=ref,event=pr` tags images built from pull requests, e.g. `pr-42`. Not asked for
- **D** — `type=schedule` only applies to scheduled runs
- **E** — **this is the trap.** Same tag, no `enable` condition. It would tag `latest` on **every**
  branch, so a push to `feature/x` overwrites the `latest` that production pulls

**Term:** *conditional tagging*. `enable={{is_default_branch}}` is the guard.

</details>

---

## Q20

Which **two** actions are required for a job to push an image to GitHub Container Registry using the
built-in token? (Choose two.)

- A. Declare `packages: write` in the job `permissions`
- B. Authenticate with `docker/login-action` using `secrets.GITHUB_TOKEN`
- C. Create a fine-grained PAT with package write scope
- D. Set `id-token: write` in the job `permissions`
- E. Enable Dependabot on the repository

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-19.md`:** lines **170–172** (A), lines **183–189** (B).

```yaml
    permissions:
      contents: read
      packages: write                        # A - line 172
    steps:
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3         # B - line 184
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```

Permission without login = not authenticated. Login without permission = 403. **You need both.**

**Why the others fail**

- **C** — the question says "using the built-in token". A PAT is a different credential
- **D** — `id-token: write` is for OIDC to a cloud provider. GHCR does not use it
- **E** — Dependabot scans dependencies. Nothing to do with registry auth

</details>

---

## Q21

Which **two** contexts can a workflow use to read values configured in repository settings? (Choose
two.)

- A. `secrets`
- B. `vars`
- C. `runner`
- D. `strategy`
- E. `matrix`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-19.md`:** Task 5, lines **340–354**.

```bash
gh secret   set AZURE_CREDENTIALS --body '{"clientId":"..."}'   # line 342 -> secrets
gh variable set APP_NAME          --body "contoso-api"          # line 349 -> vars
```

```yaml
${{ secrets.AZURE_CREDENTIALS }}
${{ vars.APP_NAME }}
```

**Why the others fail**

All three are **runtime** contexts supplied by the platform — you cannot configure them in settings.

- **C** — `runner.os`, `runner.temp`, `runner.arch`. Facts about the machine
- **D** — `strategy.job-index`, `strategy.fail-fast`. Facts about the matrix run
- **E** — `matrix.test-type` (line 147). The current matrix leg's values

</details>

---

## Q22

You must run a workflow both on pushes to `main` and on pull requests targeting `main`, and also
allow manual runs.

Which **three** trigger keys are required? (Choose three.)

- A. `push`
- B. `pull_request`
- C. `workflow_dispatch`
- D. `workflow_call`
- E. `repository_dispatch`
- F. `schedule`

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-19.md`:** lines **48–53** — exactly as written.

```yaml
on:
  push:
    branches: [main]        # A - line 49
  pull_request:
    branches: [main]        # B - line 51
  workflow_dispatch:        # C - line 53
```

**Why the others fail**

- **D** — `workflow_call` makes the workflow **callable by another workflow**. That is not an event
  a person or a push produces
- **E** — `repository_dispatch` fires from an **external API call**, not from a UI button
- **F** — `schedule` is cron-based. Nothing in the requirement mentions time

</details>

---

## Q23

Which **two** are valid reasons to move lint and unit tests out of a `Dockerfile` and into separate
pipeline jobs? (Choose two.)

- A. Test results can be published and annotated in the run
- B. The image build no longer requires test files in its context
- C. The resulting image will always be smaller than a single-stage build
- D. Docker layer caching becomes unavailable when tests are present
- E. Multi-stage builds cannot run commands in the build stage

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-19.md`:** line **167** shows why it is safe — `needs: [build, test]` means lint and
tests already passed before the image is built.

**Before — what broke your run:**

```dockerfile
COPY . .
RUN npm run lint && npm test
```

**After:**

```dockerfile
COPY src ./src
COPY scripts ./scripts
RUN npm run build
```

**Why the others fail**

- **C** — image size comes from *stages and what you copy*, not from whether tests ran. Your runtime
  stage was already slim
- **D** — false. Layer caching works with or without test steps
- **E** — false. Build stages run commands all the time — that is the point of `RUN npm run build`

**Term:** *the Dockerfile produces the artifact; the pipeline decides whether it is allowed to exist.*

</details>

---

# Section C — Repeated scenario

The same scenario appears three times with a different proposed solution. Each is independent — on
the live exam you **cannot go back**.

**Scenario:** Contoso has a workflow with a `build` job and a `deploy` job. `deploy` must run only
after `build` succeeds, and must receive the image tag `build` produced. Currently `deploy` runs at
the same time as `build` and receives no tag.

---

## Q24

**Proposed solution:** Add `needs: build` to the `deploy` job, declare an `outputs` mapping on the
`build` job, and read the value with the `needs` context in `deploy`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-19.md`:** lines **78–80**, **98**, **102**, **167**.

```yaml
  build:
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
      - id: version
        run: echo "version=1.2.3" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    steps:
      - run: echo ${{ needs.build.outputs.version }}
```

`needs` does **two** jobs at once: it creates the ordering **and** unlocks the `needs` context. One
keyword solves both halves of the requirement.

</details>

---

## Q25

**Proposed solution:** Add `if: success()` to the `deploy` job and read the value with
`${{ env.IMAGE_TAG }}` in `deploy`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Two independent failures:

```yaml
  deploy:
    if: success()                       # does NOT wait for build
    steps:
      - run: echo ${{ env.IMAGE_TAG }}  # empty - env does not cross jobs
```

1. **`if` does not create ordering.** Without `needs`, both jobs start at the same time. `success()`
   evaluates against the jobs this one depends on — and it depends on nothing, so it is trivially
   true
2. **`env` is scoped to a job.** A value set in `build` is invisible in `deploy`. Only job `outputs`
   cross the boundary

**Term:** *only `needs` creates ordering.* A condition decides whether an already-started job
proceeds; it never decides when it starts.

</details>

---

## Q26

**Proposed solution:** Move the deploy steps into the `build` job so they run after the build steps
in the same job.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Ordering does become correct and the value is shared. But look at what the challenge separates, and
why:

```yaml
  deploy-production:
    environment:
      name: production      # line 259 - the approval gate lives HERE
```

Collapse the jobs and you lose:

- the `environment:` block, so **no approval gate** and no deployment URL
- the ability for `test` (line 115) to sit between build and deploy
- permission separation — `build` would now hold deploy credentials
- the deployment record GitHub shows on the repo

**Term:** on the exam, *"merge the jobs"* is nearly always the wrong answer. Job boundaries exist so
that environments, approvals and permissions can differ.

</details>

---

# Section D — Yes/No statement grid

Each row is scored separately.

---

## Q27 — `GITHUB_TOKEN`

| # | Statement | Answer |
|---|---|---|
| 1 | It is created automatically for each workflow run |  |
| 2 | It can push to a different repository in the same organization by default |  |
| 3 | Its permissions can be narrowed with a `permissions` key |  |
| 4 | It expires when the workflow run completes |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | It is created automatically for each workflow run | **Yes** |
| 2 | It can push to a different repository in the same organization by default | **No** |
| 3 | Its permissions can be narrowed with a `permissions` key | **Yes** |
| 4 | It expires when the workflow run completes | **Yes** |

**In `challenge-19.md`:** lines **170–172** (row 3), line **188** (rows 1 and 4).

```yaml
    permissions:
      contents: read
      packages: write                        # row 3 - line 172
    steps:
      - uses: docker/login-action@v3
        with:
          password: ${{ secrets.GITHUB_TOKEN }}    # rows 1 and 4 - line 188
```

**Row 2 is the one people get wrong.** The token is scoped to **the repository it runs in**. Writing
to `Rubys-web/deploy-manifests` from `contoso-api` needs a GitHub App installation token or a
fine-grained PAT — that is exactly Challenge 40.

Row 4 is why the built-in token beats a PAT: it is revoked the moment the run ends.

</details>

---

## Q28 — environments

| # | Statement | Answer |
|---|---|---|
| 1 | Environment secrets are readable by any job in the workflow |  |
| 2 | An environment can restrict which branches may deploy to it |  |
| 3 | The `url` value appears as a link on the deployment |  |
| 4 | A wait timer delays the job before its first step runs |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Environment secrets are readable by any job in the workflow | **No** |
| 2 | An environment can restrict which branches may deploy to it | **Yes** |
| 3 | The `url` value appears as a link on the deployment | **Yes** |
| 4 | A wait timer delays the job before its first step runs | **Yes** |

**In `challenge-19.md`:** lines **224–226** and **258–260**.

```yaml
  deploy-staging:
    environment:
      name: staging                                       # rows 1, 2, 4
      url: https://contoso-api-staging.azurewebsites.net  # row 3 - line 226
```

**Row 1 is the trap.** The `build` job (line 75) has no `environment:`, so it cannot read
`DB_CONNECTION_STRING` from staging (line 345) — even though both are in the same file.

Rows 2 and 4 are configured in **repository Settings → Environments**, not in YAML. That split
matters: someone editing the workflow cannot remove them.

</details>

---

## Q29 — composite action vs reusable workflow

| # | Statement | Answer |
|---|---|---|
| 1 | A composite action can define its own `jobs` |  |
| 2 | A reusable workflow is referenced with `uses` at job level |  |
| 3 | A composite action runs on the caller's runner |  |
| 4 | A reusable workflow can receive secrets from the caller |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A composite action can define its own `jobs` | **No** |
| 2 | A reusable workflow is referenced with `uses` at job level | **Yes** |
| 3 | A composite action runs on the caller's runner | **Yes** |
| 4 | A reusable workflow can receive secrets from the caller | **Yes** |

**In `challenge-19.md`:** lines **299–301** (row 1), lines **330–331** (row 3).

**Composite — `uses` under `steps:` (line 331):**

```yaml
    steps:
      - uses: ./.github/actions/setup-node-project
        with:
          node-version: ${{ env.NODE_VERSION }}
```

**Reusable workflow — `uses` directly under the job:**

```yaml
jobs:
  build:
    uses: Rubys-web/.github/.github/workflows/reusable-node-ci-cd.yml@main
    with:
      node-version: "20.x"
    secrets: inherit          # row 4
```

Row 1: an action file has `name`, `inputs`, `outputs`, `runs` (line 299). There is no `jobs` key.

**Notice the indentation difference between the two `uses:` lines.** That alone is a
spot-the-difference exam question.

</details>

---

## Q30 — matrix strategy

| # | Statement | Answer |
|---|---|---|
| 1 | Matrix jobs run in parallel by default |  |
| 2 | `fail-fast` defaults to `true` |  |
| 3 | `max-parallel` limits how many matrix jobs run at once |  |
| 4 | Each matrix job shares one workspace on one runner |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Matrix jobs run in parallel by default | **Yes** |
| 2 | `fail-fast` defaults to `true` | **Yes** |
| 3 | `max-parallel` limits how many matrix jobs run at once | **Yes** |
| 4 | Each matrix job shares one workspace on one runner | **No** |

**In `challenge-19.md`:** lines **120–122**.

```yaml
    strategy:
      fail-fast: true        # row 2 - this is the DEFAULT, shown here for clarity
      max-parallel: 2        # row 3
      matrix:
        test-type: [unit, integration]
```

**Row 4 is the trap.** Each leg gets its own runner and its own checkout. That is exactly why lines
**134–142** repeat `actions/checkout` and `actions/setup-node` inside the `test` job — nothing is
shared between legs.

Row 2 matters in practice: if the `unit` leg fails, the `integration` leg is **cancelled**. Set
`fail-fast: false` when you want to see every failure in one run.

</details>

---

# Section E — Drag and drop

---

## Q31

Arrange these workflow keys into the order they appear in a valid GitHub Actions file, top to bottom.

**Items:** `jobs:` · `env:` · `name:` · `on:`

<details>
<summary>Show answer</summary>

### Answer: `name:` → `on:` → `env:` → `jobs:`

**In `challenge-19.md`:** line **46**, line **48**, line **69**, line **74**.

```yaml
name: Contoso API CI/CD      # line 46

on:                          # line 48
  push:
    branches: [main]

env:                         # line 69
  NODE_VERSION: "20.x"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:                        # line 74
  build:
```

YAML itself does not enforce order — but this is the convention the exam uses, and `jobs:` always
comes last because it is the body of the file.

</details>

---

## Q32

Match each GitHub Actions keyword to its Azure Pipelines equivalent.

| GitHub Actions | Azure Pipelines |
|---|---|
| `on:` |  |
| `runs-on:` |  |
| `needs:` |  |
| `run:` |  |
| `uses:` |  |

**Options:** `dependsOn:` · `pool:` · `script:` · `task:` · `trigger:`

<details>
<summary>Show answer</summary>

| GitHub Actions | Azure Pipelines |
|---|---|
| `on:` | `trigger:` |
| `runs-on:` | `pool:` |
| `needs:` | `dependsOn:` |
| `run:` | `script:` |
| `uses:` | `task:` |

### The same pipeline on both platforms

```yaml
# GitHub Actions - lines 48, 77, 117, 92, 82 of challenge-19.md
on:
  push:
    branches: [main]
jobs:
  test:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v4
      - run: npm test
```

```yaml
# Azure Pipelines
trigger:
  branches:
    include: [main]
jobs:
  - job: test
    pool:
      vmImage: 'ubuntu-latest'
    dependsOn: build
    steps:
      - task: NodeTool@0
      - script: npm test
```

Two more worth adding to the table, because they appear constantly:

| GitHub Actions | Azure Pipelines |
|---|---|
| `${{ secrets.NAME }}` | `$(NAME)` from a variable group |
| reusable workflow | `template:` |

**Memorise this.** It generates questions in every domain, not just this one.

</details>

---

## Q33

Arrange the jobs of the Challenge 19 workflow into their execution order.

**Items:** `deploy-production` · `docker` · `build` · `deploy-staging` · `test`

<details>
<summary>Show answer</summary>

### Answer: `build` → `test` → `docker` → `deploy-staging` → `deploy-production`

**In `challenge-19.md`:** the order is never stated. You read it off the `needs` keys.

```yaml
  build:                     # line 75  - no needs -> starts first
  test:
    needs: build             # line 117
  docker:
    needs: [build, test]     # line 167 - waits for BOTH
  deploy-staging:
    needs: docker            # line 222
  deploy-production:
    needs: deploy-staging    # line 256
```

Note line 167: `docker` lists **both** `build` and `test`. It needs `build` for its outputs (the
version) and `test` for the guarantee that tests passed.

**Term:** *`needs` is the dependency graph.* The exam draws these as ordering questions constantly.

</details>

---

## Q34

Arrange these steps into the correct order for a job that pushes an image to GHCR.

**Items:** Extract metadata · Build and push · Check out the repository · Log in to the registry ·
Set up Buildx

<details>
<summary>Show answer</summary>

### Answer: Check out → Set up Buildx → Log in → Extract metadata → Build and push

**In `challenge-19.md`:** lines **176–213**.

```yaml
      - name: Checkout repository            # 1 - line 177, need the Dockerfile
        uses: actions/checkout@v4

      - name: Set up Docker Buildx           # 2 - line 180, enables cache-from/cache-to
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR                 # 3 - line 184, must precede any push
        uses: docker/login-action@v3

      - name: Extract metadata for Docker    # 4 - line 192, produces steps.meta.outputs.tags
        id: meta
        uses: docker/metadata-action@v5

      - name: Build and push image           # 5 - line 202, consumes those tags
        id: push
        uses: docker/build-push-action@v5
        with:
          tags: ${{ steps.meta.outputs.tags }}      # line 206
```

**Two hard rules:** login before push, and metadata before whatever reads `steps.meta.outputs`.

Buildx must come before the build step because `cache-from: type=gha` (line 208) requires the buildx
driver — the default Docker builder cannot use that cache backend.

</details>

---

## Q35

Match each value to the place it should be stored.

| Value | Storage | Command in `challenge-19.md` |
|---|---|---|
| Azure service principal JSON |  |  |
| Application name used everywhere |  |  |
| Staging database connection string |  |  |
| App Service plan name for production |  |  |

<details>
<summary>Show answer</summary>

| Value | Storage | Command in `challenge-19.md` |
|---|---|---|
| Azure service principal JSON | repository **secret** | line **342** |
| Application name used everywhere | repository **variable** | line **349** |
| Staging database connection string | environment **secret** | line **345** |
| App Service plan name for production | environment **variable** | line **354** |

```bash
gh secret   set AZURE_CREDENTIALS      --body '{"clientId":"..."}'            # 342
gh secret   set DB_CONNECTION_STRING   --env staging --body "Server=..."     # 345
gh variable set APP_NAME               --body "contoso-api"                  # 349
gh variable set APP_SERVICE_PLAN       --env production --body "contoso-..." # 354
```

### The decision rule — two questions, four answers

1. **Is it sensitive?** → secret. Otherwise → variable
2. **Does it differ per environment?** → environment scope. Otherwise → repository scope

|  | Repository scope | Environment scope |
|---|---|---|
| **Sensitive** | `AZURE_CREDENTIALS` | `DB_CONNECTION_STRING` |
| **Not sensitive** | `APP_NAME` | `APP_SERVICE_PLAN` |

That table is the whole model. Learn it as a grid, not as four facts.

</details>

---

# Section F — Hot area

---

## Q36

```yaml
on:
  [BLANK 1]:
    inputs:
      environment:
        required: true
        type: [BLANK 2]
        options:
          - staging
          - production
```

- **BLANK 1:** `workflow_call` / `workflow_dispatch` / `repository_dispatch` / `schedule`
- **BLANK 2:** `string` / `choice` / `environment` / `boolean`

<details>
<summary>Show answer</summary>

### Answer: `workflow_dispatch`, `choice`

**In `challenge-19.md`:** lines **53** and **59**.

`workflow_call` would make it callable by another workflow. `type: string` gives a free-text box, not
a dropdown. Read the value as `${{ inputs.environment }}`.

Note line **63** uses the other input type you should know:

```yaml
      skip_tests:
        type: boolean
        default: false
```

which is then consumed at line **119** as `if: ${{ !inputs.skip_tests }}`.

</details>

---

## Q37

```yaml
docker:
  runs-on: ubuntu-latest
  [BLANK 1]:
    contents: read
    [BLANK 2]: write
```

- **BLANK 1:** `permissions` / `defaults` / `concurrency` / `strategy`
- **BLANK 2:** `packages` / `contents` / `id-token` / `deployments`

<details>
<summary>Show answer</summary>

### Answer: `permissions`, `packages`

**In `challenge-19.md`:** lines **170–172**.

`defaults` sets shell and working directory. `concurrency` cancels overlapping runs. `strategy` holds
the matrix. Only `permissions` scopes the token.

</details>

---

## Q38

```yaml
build:
  runs-on: ubuntu-latest
  [BLANK 1]:
    version: ${{ steps.version.outputs.version }}
  steps:
    - name: Generate version
      [BLANK 2]: version
      run: echo "version=1.2.3" >> [BLANK 3]
```

- **BLANK 1:** `outputs` / `env` / `with` / `vars`
- **BLANK 2:** `name` / `id` / `key` / `ref`
- **BLANK 3:** `$GITHUB_ENV` / `$GITHUB_OUTPUT` / `$GITHUB_STATE` / `$GITHUB_PATH`

<details>
<summary>Show answer</summary>

### Answer: `outputs`, `id`, `$GITHUB_OUTPUT`

**In `challenge-19.md`:** lines **78**, **98**, **102**.

All three blanks are the same chain as Q1 and Q17. **If you missed any blank here, redo Task 1.**

The distractors, so you know what they actually do:

- `$GITHUB_ENV` — sets an env var for later steps in **this job only**
- `$GITHUB_PATH` — prepends a directory to `PATH` for later steps
- `$GITHUB_STATE` — passes state between an action's main and post phases
- `name:` — the label shown in the UI. It is **not** the reference key; only `id:` is

</details>

---

## Q39

```yaml
name: "Setup Node.js project"
runs:
  [BLANK 1]: "composite"
  steps:
    - name: Install dependencies
      [BLANK 2]: bash
      run: npm ci
```

- **BLANK 1:** `using` / `type` / `mode` / `kind`
- **BLANK 2:** `shell` / `run-with` / `interpreter` / `env`

<details>
<summary>Show answer</summary>

### Answer: `using`, `shell`

**In `challenge-19.md`:** lines **300** and **322**.

`using` also accepts `node20` and `docker` for the other two action types. `composite` is the one
that runs a list of steps.

</details>

---

## Q40

```yaml
deploy-staging:
  needs: docker
  runs-on: ubuntu-latest
  [BLANK 1]:
    name: staging
    [BLANK 2]: https://contoso-api-staging.azurewebsites.net
```

- **BLANK 1:** `environment` / `concurrency` / `defaults` / `container`
- **BLANK 2:** `url` / `link` / `endpoint` / `host`

<details>
<summary>Show answer</summary>

### Answer: `environment`, `url`

**In `challenge-19.md`:** lines **224–226**.

`container:` would run the job's steps inside a container image — a completely different feature that
looks similar because it also takes a nested block.

</details>

---

## Q41

```yaml
permissions:
  [BLANK 1]: write
steps:
  - uses: azure/login@v2
    with:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ [BLANK 2].AZURE_SUBSCRIPTION_ID }}
```

- **BLANK 1:** `id-token` / `contents` / `packages` / `actions`
- **BLANK 2:** `secrets` / `env` / `inputs` / `needs`

<details>
<summary>Show answer</summary>

### Answer: `id-token`, `secrets`

**In `challenge-19.md`:** line **229** shows the credential-based version this replaces.

**Learn this block by heart.** It reappears in Challenges 39, 42 and the capstone, and security auth
is the domain that has cost you points in every mock.

`id-token: write` is what lets the job request the OIDC token. Without it, `azure/login` fails with a
token-request error, and the message does not say "add a permission".

</details>

---

# Section G — Case study

## Case study: Contoso Ltd

### Background

Contoso Ltd is migrating CI/CD from Jenkins to GitHub Actions. The primary application is a
containerized Node.js REST API deployed to Azure App Service.

### Current environment

- A single repository, `contoso-api`, in the `Rubys-web` organization
- Deployments are manual and take about 40 minutes
- Credentials are stored as long-lived secrets in Jenkins

### Requirements

**Build**

- Unit and integration tests must run in parallel with each other
- The container image must be tagged with the commit SHA on every push to `main`
- The image must be pushed to GitHub Container Registry

**Deployment**

- Every production deployment must be approved by a member of the platform team
- Production must never be deployed from a branch other than `main`
- Staging must be verified automatically before production is deployed

**Security**

- No long-lived Azure credential may be stored in the repository
- Workflow tokens must have the minimum permissions needed
- Six setup steps repeated across workflows must be defined in one place

---

## Q42

You must meet the requirement for unit and integration tests. What should you configure on the `test`
job?

- A. Two separate jobs, each with its own `needs` declaration
- B. A `strategy` block with a `matrix` containing both test types
- C. A single job with two sequential `run` steps
- D. A `services` block with two container definitions

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-19.md`:** lines **120–122**, consumed at line **147**.

```yaml
    strategy:
      matrix:
        test-type: [unit, integration]
```

**Why the others fail**

- **A** — this *does* run them in parallel, but duplicates the entire job definition twice. When a
  matrix fits, the exam wants the matrix
- **C** — sequential steps in one job is the opposite of the requirement
- **D** — `services` starts **support containers** (line 123, Redis). It does not run tests

</details>

---

## Q43

You must meet the approval requirement for production. What should you configure?

- A. A required reviewer on the `production` environment
- B. A branch protection rule requiring one approving review
- C. A `workflow_dispatch` trigger with a confirmation input
- D. A `concurrency` group scoped to the production job

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-19.md`:** lines **258–259**.

```yaml
  deploy-production:
    environment:
      name: production      # required reviewer configured on this environment
```

**Why the others fail**

- **B is the trap.** Branch protection requires review before **merging code**. It says nothing about
  deploying. Merge once, then deploy that same commit fifty times with zero approvals
- **C** — an input is just a form field. Anyone triggering the run fills it in themselves. That is
  not an approval by another person
- **D** — `concurrency` prevents overlapping runs. It gates timing, not permission

**The boundary to memorise:** environments gate **deployments**; branch protection gates **merges**.
The exam swaps them on purpose.

</details>

---

## Q44

You must meet the requirement that production is never deployed from another branch. Which **two**
options achieve this? (Choose two.)

- A. A deployment branch rule on the `production` environment
- B. A job-level `if` condition checking `github.ref`
- C. A `concurrency` group named after the branch
- D. A `paths-ignore` filter on the push trigger
- E. A required status check on the `main` branch

<details>
<summary>Show answer</summary>

### Answer: A, B

**A** — Settings → Environments → production → deployment branches. Enforced by GitHub even if
someone edits the workflow file.

**B** — in the workflow, the same pattern as line **169**:

```yaml
  deploy-production:
    if: github.ref == 'refs/heads/main'
```

Both are accepted, and using both is defence in depth: the environment rule survives a YAML edit.

**Why the others fail**

- **C** — `concurrency` controls how many runs execute at once, not which branch may deploy
- **D** — `paths-ignore` decides whether the workflow **triggers**, based on which files changed
- **E** — a status check governs merging into `main`, not deploying from it

</details>

---

## Q45

You must meet the security requirement for Azure credentials. What should you implement?

- A. Workload identity federation with OIDC and `id-token: write`
- B. A service principal secret stored as `AZURE_CREDENTIALS`
- C. A system-assigned managed identity on the runner
- D. A fine-grained PAT rotated every thirty days

<details>
<summary>Show answer</summary>

### Answer: A

The constraint is in the requirements: **"No long-lived Azure credential may be stored."**

**Violates it — this is what `challenge-19.md` line 229 does:**

```yaml
      - uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
```

**Meets it:**

```yaml
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

**Why the others fail**

- **B** — stores exactly what the requirement forbids. This is the option the *challenge itself*
  uses, which is why it feels right. Read the constraint
- **C** — **worth understanding properly.** A GitHub-hosted runner is not an Azure resource, so it has
  no managed identity. Managed identity is for things **running in Azure**: a VM, App Service,
  Container App, AKS pod. This distinction is a guaranteed exam question
- **D** — a PAT is a long-lived credential. Rotating it does not make it short-lived, and it
  authenticates to GitHub, not Azure

</details>

---

## Q46

You must meet the requirement for the six repeated setup steps. What should you create?

- A. A composite action in `.github/actions`
- B. A starter workflow in the `.github` repository
- C. A YAML anchor at the top of each workflow
- D. A `defaults` block applied to every job

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-19.md`:** Task 4, lines **276–334**.

```yaml
      - uses: ./.github/actions/setup-node-project
        with:
          node-version: ${{ env.NODE_VERSION }}
```

**Why the others fail**

- **B is the trap.** A starter workflow is a template in the "New workflow" UI. It is **copied once**
  when someone creates a workflow. Change the starter later and existing workflows do not update. A
  composite action is referenced live, so one edit updates every caller
- **C** — GitHub Actions does **not** support YAML anchors. The parser rejects them
- **D** — `defaults` sets `shell` and `working-directory`. It cannot contain steps

</details>

---

## Q47

You must verify staging automatically before production is deployed. Which configuration meets the
requirement?

- A. A smoke-test step in `deploy-staging`, with `deploy-production` declaring `needs: deploy-staging`
- B. A smoke-test step in `deploy-production` running before the swap step
- C. A scheduled workflow that checks staging every fifteen minutes
- D. A branch protection rule requiring a passing status check on `main`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-19.md`:** lines **240–252** (the smoke test), line **256** (`needs`).

```yaml
      - name: Run smoke tests against staging      # line 240
        run: |
          for i in {1..10}; do
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" .../health)
            if [ "$STATUS" = "200" ]; then exit 0; fi
            sleep 10
          done
          exit 1

  deploy-production:
    needs: deploy-staging                          # line 256 - blocked if the test exits 1
```

**The retry loop matters.** App Service needs time to warm up after a deploy, so a single immediate
`curl` would fail on a perfectly healthy app. Ten attempts, ten seconds apart.

**Why the others fail**

- **B** — by the time `deploy-production` starts, the production deployment has already begun and the
  environment approval has already been consumed. Too late
- **C** — a schedule is not tied to a deployment. It could pass fifteen minutes before a bad deploy
- **D** — status checks gate merges, not deployments (same boundary as Q43)

</details>

---

## Q48

After implementing the workflow, the `docker` job fails with `denied: permission_denied` when pushing
to GHCR. The security team refuses to allow any personal access token.

What should you do?

- A. Add `packages: write` to the job's `permissions` block
- B. Create a classic PAT with `write:packages` and store it as a secret
- C. Change the registry to Azure Container Registry
- D. Grant the repository `admin` role to the workflow actor

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-19.md`:** lines **170–172**, and Break & fix Exercise 2 at lines **430–448**.

```yaml
    permissions:
      contents: read
      packages: write        # the entire fix
```

**Why the others fail**

- **B** — the last sentence explicitly forbids it. **This is your recorded failure pattern**: the
  technically-workable option that ignores the stated constraint
- **C** — swapping registries to avoid fixing a one-line permission. The exam includes options like
  this to test whether you solve the problem or route around it
- **D** — repository roles govern people, not the workflow token's scope. Admin on the repo would not
  change what `GITHUB_TOKEN` is allowed to do

**Name the constraint, then eliminate.** "No PAT" removes B before you evaluate anything else.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Silent empty value** | Q1, Q17, Q38 | Nothing errors when the chain breaks. Check all three: step `id`, job `outputs`, consumer `needs` |
| **Constraint in the last line** | Q45, Q48 | "No long-lived credential", "no PAT". Read the last sentence first, then eliminate |
| **Permission, not credential** | Q3, Q20, Q48 | Built-in token failing? Add a permission, do not add a secret |
| **Environment vs branch protection** | Q43, Q47 | Environments gate deployments. Branch protection gates merges |
| **`if` does not create ordering** | Q25 | Only `needs` creates ordering |
| **Composite vs reusable** | Q5, Q18, Q29, Q46 | Composite = steps, same runner. Reusable workflow = jobs, own runner |
| **Merge-the-jobs distractor** | Q26 | It "works" but kills environments, approvals and parallelism |
| **`.dockerignore` is not `.gitignore`** | Q15, Q23 | Build context = repo minus `.dockerignore` |
| **Managed identity on a GitHub runner** | Q45 | The runner is not an Azure resource. No managed identity exists |
| **Starter workflow vs composite action** | Q46 | Starter = copied once. Composite = referenced live |
| **`latest` with no condition** | Q19 | `enable={{is_default_branch}}` or a feature branch overwrites production's tag |
| **Matrix legs share nothing** | Q30 | Own runner, own workspace. That is why checkout repeats |

---

# The eight blocks to memorise

If you remember nothing else from Challenge 19, remember these. Line numbers are in
`challenge-19.md`.

```yaml
# 1. The output chain  (lines 78, 98, 102, 167)
  build:
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
      - id: version
        run: echo "version=1.2.3" >> $GITHUB_OUTPUT
  docker:
    needs: build
    steps:
      - run: echo ${{ needs.build.outputs.version }}

# 2. GHCR push permission  (lines 170-172)
    permissions:
      contents: read
      packages: write

# 3. Secretless Azure login  (replaces line 229)
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

# 4. Manual run with a dropdown  (lines 53-62)
on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]

# 5. Environment gate  (lines 258-260)
    environment:
      name: production
      url: https://contoso-api.azurewebsites.net

# 6. Parallel test legs  (lines 120-122)
    strategy:
      matrix:
        test-type: [unit, integration]

# 7. Composite action step  (lines 300, 322)
runs:
  using: "composite"
  steps:
    - shell: bash
      run: npm ci

# 8. Main-branch-only job  (line 169)
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 19 is exam-ready. Move to Challenge 20 |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 1 and 4 of the challenge, then retake this |
| Below 30 | Redo the whole challenge, typing every line. Do not move to Challenge 20 yet |

Record your result in `AZ-400-Learning-Log.md` under Challenge 19, and note **which trap** caught you
— not just which question number.
