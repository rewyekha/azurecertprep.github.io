---
sidebar_position: 1.5
toc_max_heading_level: 2
title: "Challenge 31: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 31 — AZ-400 exam questions

**48 questions** built only from what Challenge 31 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-31.md`**.

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

:::tip The shape of every IaC question

**Plan on PR, apply on merge.** Read-only analysis where a human can see it; write operations only
after review. Almost every question here is that principle applied to a different tool.

:::

---

# Section A — Single answer

---

## Q1

Contoso is Azure-only, wants drift detection, and some of the team knows HCL. Which technology does
the challenge recommend for **new Azure-native** projects?

- A. ARM templates
- B. Bicep
- C. Terraform
- D. Azure CLI scripts

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-31.md`:** the decision matrix at lines **55–64**, and the decision at lines **69–70**.

```text
# Decision: Use Bicep for new Azure-native projects (simpler syntax, no state to manage)
# Decision: Use Terraform where drift detection or multi-cloud is needed
```

**The two reasons are in the matrix:** Bicep is a simplified DSL over ARM (line 57), and it is
**stateless** — Azure itself is the source of truth (line 59), so there is no state file to store,
lock, back up or corrupt.

**Why the others fail**

- **A** — verbose JSON with a steep learning curve. Bicep compiles **to** ARM, so you get the same
  deployment engine with far better authoring
- **C** — **the deliberate near-miss.** Terraform is right when you need drift detection or
  multi-cloud, and the scenario mentions both drift *and* HCL familiarity. But it introduces remote
  state, which is the operational cost line 69 avoids for Azure-native work
- **D** — imperative scripts are not declarative IaC

**The exam pattern:** the recommendation is **conditional**. Azure-only and simple → Bicep.
Multi-cloud or drift detection required → Terraform.

</details>

---

## Q2

Which IaC technology has built-in drift detection?

- A. ARM templates
- B. Bicep
- C. Terraform
- D. All three

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-31.md`:** line **64**.

| Criteria | ARM | Bicep | Terraform |
|---|---|---|---|
| Drift detection | None built-in | None built-in | **`terraform plan` detects drift** |

**Why Terraform can and Bicep cannot:** Terraform holds a **state file** recording what it believes
exists. `terraform plan` compares state, configuration and reality, so a manual portal change shows up
as a difference.

Bicep is stateless — it only compares your template to Azure. That still surfaces *some* drift via
`what-if` (which is how lines 721–737 do it), but there is no record of intent to diff against.

**Why the others fail** — A and B have no built-in drift detection; D contradicts the matrix.

</details>

---

## Q3

Which command shows what a Bicep deployment would change, without changing anything?

- A. `az deployment sub validate`
- B. `az deployment sub what-if`
- C. `az bicep build`
- D. `az deployment sub create --confirm-with-what-if`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-31.md`:** lines **234–238**.

```bash
          RESULT=$(az deployment sub what-if \
            --location eastus2 \
            --template-file main.bicep \
            --parameters @environments/dev.bicepparam \
            --no-pretty-print 2>&1)
```

**Three checks, three depths** — the exam separates them:

| Command | Checks |
|---|---|
| `az bicep build` | **Syntax and linting** — no Azure call at all (line 200) |
| `az deployment sub validate` | The template is **deployable** — Azure validates it (line 211) |
| `az deployment sub what-if` | What would actually **change** (line 234) |

`--no-pretty-print` strips the colour codes so the output can be posted to a PR comment.

**Why the others fail**

- **A** — validates, but does not report changes
- **C** — compiles locally; it never contacts Azure
- **D** — real and useful interactively, but it **prompts and then deploys**. Not read-only

</details>

---

## Q4

Which permission does the infrastructure workflow need to post what-if results to a pull request?

- A. `contents: write`
- B. `pull-requests: write`
- C. `issues: write`
- D. `id-token: write`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-31.md`:** lines **183–186**.

```yaml
permissions:
  id-token: write        # OIDC login to Azure
  contents: read         # check out the repository
  pull-requests: write   # post the what-if comment
```

**Each line maps to one capability**, which is what least privilege looks like in practice.

**Why the others fail**

- **A** — the workflow only reads code
- **C** — `issues: write` is needed by the **drift-detection** workflow (line 704), which creates
  issues. Different workflow, different need
- **D** — required, and it is for **OIDC**, not for commenting

**Note the drift workflow's permissions differ** (lines 701–704): `issues: write` instead of
`pull-requests: write`. Same principle, different job.

</details>

---

## Q5

How does the workflow authenticate to Azure?

- A. A service principal secret in `AZURE_CREDENTIALS`
- B. OIDC with `client-id`, `tenant-id` and `subscription-id`
- C. A managed identity on the runner
- D. An Azure CLI login with a stored password

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-31.md`:** lines **202–207**.

```yaml
      - name: Log in to Azure
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ env.AZURE_SUBSCRIPTION_ID }}
```

Paired with `id-token: write` at line 184. **No client secret anywhere** — the workflow requests a
short-lived OIDC token that Azure trusts via a federated credential.

**Why the others fail**

- **A** — the credential-storing form. Used in earlier challenges; this one has moved past it
- **C** — a GitHub-hosted runner is not an Azure resource and has no managed identity
- **D** — a stored password

**This is the block you must know cold.** It has now appeared in Challenges 19, 21, 23, 27, 28 and
here — security auth is the domain that has cost you marks in every mock.

</details>

---

## Q6

What does `targetScope = 'subscription'` do in `main.bicep`?

- A. Restricts the template to one subscription
- B. Allows the template to create resource groups
- C. Sets the default location
- D. Enables cross-subscription deployment

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-31.md`:** lines **133** and **144–161**.

```bicep
targetScope = 'subscription'
...
resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name: resourceGroupName
  ...
}

module networking 'modules/networking/main.bicep' = {
  scope: rg
  ...
}
```

**A resource group cannot be created from inside a resource group.** Subscription scope is what lets
the template create the group and then deploy **into** it via `scope: rg` on the module.

That is also why the deployment command is `az deployment sub create` (line 279) rather than
`az deployment group create`.

**The four scopes:** `resourceGroup` (default), `subscription`, `managementGroup`, `tenant`.

**Why the others fail** — A, C and D all misdescribe it.

</details>

---

## Q7

Which Bicep decorator restricts a parameter to a fixed set of values?

- A. `@allowed`
- B. `@description`
- C. `@secure`
- D. `@minLength`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-31.md`:** lines **89–91**.

```bicep
@description('Environment name used for naming conventions')
@allowed(['dev', 'test', 'staging', 'prod'])
param environmentName string
```

An invalid value fails **before deployment starts**, so no partial change occurs.

**Why the others fail**

- **B** — documentation, surfaced in tooling
- **C** — `@secure()` marks a parameter as a secret so its value is never logged or stored in
  deployment history. Essential for passwords, and unrelated to value restriction
- **D** — string or array length

**The Azure Pipelines equivalent** is `type: string` with a `values` list (Challenge 20 Q11). Same
idea, different syntax, and the exam swaps them.

</details>

---

## Q8

A Terraform apply fails with a state lock error after a previous pipeline run crashed.

What is the correct response?

- A. Delete the state file and re-run
- B. Verify the lease is stale, then `terraform force-unlock <id>`
- C. Run `terraform apply -lock=false`
- D. Wait for the lock to expire automatically

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-31.md`:** Break & fix Exercise 2, lines **896–931**.

```bash
# Verify the lock is stale (previous run no longer active)
az storage blob show \
  --account-name stcontosoterraform \
  --container-name tfstate \
  --name contoso-infra.tfstate \
  --query "properties.lease.status"

# Force unlock (use only when confirmed stale)
terraform force-unlock a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

**Verify first, force second.** If another run is genuinely still applying, force-unlocking lets two
processes write the same state simultaneously — which corrupts it.

**Why the others fail**

- **A** — deleting state means Terraform forgets every resource it manages. The next apply tries to
  **recreate everything**, and real production resources get destroyed and rebuilt
- **C** — disables locking entirely. Same corruption risk, permanently
- **D** — the azurerm backend uses **blob leases**, and this lease is held by a dead process. Nothing
  releases it

**The prevention (lines 929–931):** `timeoutInMinutes: 30` on the pipeline step, so a hung run is
killed rather than holding the lock indefinitely.

</details>

---

## Q9

Which Terraform backend setting enables authentication without a stored secret?

- A. `use_oidc = true`
- B. `use_msi = true`
- C. `client_secret`
- D. `access_key`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-31.md`:** lines **348–360**.

```hcl
  backend "azurerm" {
    resource_group_name  = "rg-contoso-tfstate"
    storage_account_name = "stcontosoterraform"
    container_name       = "tfstate"
    key                  = "contoso-infra.tfstate"
    use_oidc             = true
  }
}

provider "azurerm" {
  features {}
  use_oidc = true
}
```

**It appears twice, deliberately** — once for the **backend** (reading and writing state) and once for
the **provider** (creating resources). They authenticate separately, and forgetting the backend one is
a common failure.

**Why the others fail**

- **B** — `use_msi` is real and uses a **managed identity**, which is correct on a self-hosted agent
  running in Azure but not on a GitHub-hosted runner
- **C** and **D** — both stored secrets

</details>

---

## Q10

Why does the pipeline use a separate state file per environment?

- A. To isolate blast radius so one environment's state cannot affect another
- B. To reduce storage costs
- C. Because Terraform cannot manage multiple resource groups
- D. To speed up `terraform plan`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-31.md`:** line **671**.

```yaml
    backendAzureRmKey: "contoso-$(environment).tfstate"
```

**One shared state file would mean a single `terraform apply` could destroy production while you
intended to change dev.** It also serialises every environment on one lock: a long dev apply blocks a
production hotfix.

**Why the others fail**

- **B** — state files are tiny
- **C** — Terraform manages many resource groups happily
- **D** — a marginal side effect, not the reason

**Bicep has no equivalent problem**, because it is stateless. This is the operational cost line 69
weighs when it recommends Bicep for Azure-native work.

</details>

---

## Q11

Which **two** Terraform commands validate configuration without contacting Azure? (Pick the single
best answer.)

- A. `terraform init -backend=false` and `terraform validate`
- B. `terraform plan` and `terraform apply`
- C. `terraform state list` and `terraform show`
- D. `terraform import` and `terraform refresh`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-31.md`:** lines **577–578**.

```bash
terraform init -backend=false
terraform validate
terraform fmt -check -recursive
```

**`-backend=false` is the key flag.** A normal `init` connects to the remote backend and needs
credentials. With it disabled, Terraform downloads providers and checks syntax locally — so validation
can run on a pull request with **no Azure access at all**.

`terraform fmt -check -recursive` is a third local check: it fails if any file is not canonically
formatted, without rewriting anything.

**Why the others fail**

- **B** — both contact Azure and read state
- **C** — both read state, which lives in Azure Storage
- **D** — both contact Azure, and `import` mutates state

</details>

---

## Q12

What does `checkov` scan for in the testing job?

- A. Syntax errors in Bicep
- B. Security and compliance misconfigurations in IaC
- C. Terraform state drift
- D. Unused parameters

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-31.md`:** lines **604–616**.

```yaml
      - name: Run checkov for security scanning
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: .
          framework: bicep
          output_format: sarif
          output_file_path: results.sarif

      - name: Upload SARIF results
        if: always()
        uses: github/codeql-action/upload-sarif@v3
```

Checkov applies **policy** — public network access, missing encryption, permissive NSG rules, absent
diagnostic settings. A template can be syntactically perfect and still deploy an open storage account.

**Why the others fail**

- **A** — that is `az bicep build` (line 598)
- **C** — that is `what-if` or `terraform plan`
- **D** — a **linter** rule, configured at line 562

**Note `if: always()` on the upload.** Checkov fails the step on findings, so without it the SARIF is
never uploaded precisely when there is something to see. Same pattern as Trivy in Challenge 28.

</details>

---

## Q13

Which `bicepconfig.json` rule prevents a secure parameter from having a default value?

- A. `no-hardcoded-env-urls`
- B. `secure-parameter-default`
- C. `no-unused-params`
- D. `use-recent-api-versions`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-31.md`:** lines **556–570**.

```json
      "rules": {
        "no-hardcoded-env-urls":    { "level": "error" },
        "no-unused-params":         { "level": "warning" },
        "prefer-interpolation":     { "level": "warning" },
        "secure-parameter-default": { "level": "error" },
        "simplify-interpolation":   { "level": "warning" },
        "use-recent-api-versions":  { "level": "warning", "maxAllowedAgeInDays": 730 }
      }
```

**Why it is an `error` and not a warning:** a default on a `@secure()` parameter puts a real secret in
the template file, which lands in source control and in deployment history. That is a credential leak
committed by design.

**Note the two levels.** `error` fails the build; `warning` reports. The two `error` rules are both
security-related — that is the pattern to read off this file.

**Why the others fail** — all real rules with different purposes: hardcoded URLs, unused parameters,
and API versions older than 730 days.

</details>

---

## Q14

On a pull request, which operations should run?

- A. Validate, lint and what-if only
- B. Validate and apply to dev
- C. Apply to all environments
- D. Nothing — infrastructure changes are applied manually

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-31.md`:** lines **633–635**.

```text
- On pull request: validate, lint, plan/what-if (read-only, informational)
- On merge to main: apply the changes (write operations)
```

**This is the central principle of the challenge.** A pull request comes from anyone, including a fork
— running write operations there would let an unreviewed change modify real infrastructure.

The what-if output is posted as a PR comment (lines 243–258) so the reviewer sees **exactly** what the
merge will do before approving it.

**Why the others fail**

- **B** and **C** — write operations before review
- **D** — manual application is the problem the CTO mandated fixing (line 30)

</details>

---

## Q15

Which branch protection setting requires the what-if job to pass before merging?

- A. `required_pull_request_reviews`
- B. `required_status_checks` with the job names in `contexts`
- C. `enforce_admins`
- D. `restrictions`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-31.md`:** lines **625–628**.

```bash
gh api repos/{owner}/{repo}/branches/main/protection --method PUT \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field required_status_checks='{"strict":true,"contexts":["Validate Bicep","What-if analysis"]}' \
  --field enforce_admins=true
```

**The `contexts` values are the job `name:` values** — "Validate Bicep" (line 193) and "What-if
analysis" (line 217). A mismatch means the check silently never appears as required, which is the
Challenge 19 Break & fix failure.

**`"strict": true`** additionally requires the branch to be up to date with `main` before merging, so
what-if reflects the current state of the world.

**Why the others fail**

- **A** — requires a human review, which is complementary
- **C** — applies the rules to administrators too. Good practice, different setting
- **D** — restricts who may push

</details>

---

## Q16

Which schedule expression runs drift detection every weekday at 06:00 UTC?

- A. `0 6 * * 1-5`
- B. `6 0 * * 1-5`
- C. `0 6 * * *`
- D. `0 6 1-5 * *`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-31.md`:** line **698**.

```yaml
on:
  schedule:
    - cron: "0 6 * * 1-5"  # Every weekday at 06:00 UTC
  workflow_dispatch:
```

**Cron field order:** minute, hour, day-of-month, month, day-of-week. So `0 6` is 06:00, and `1-5` in
the fifth field is Monday to Friday.

**Why the others fail**

- **B** — fields reversed: minute 6 of hour 0, i.e. 00:06
- **C** — every day including weekends
- **D** — `1-5` in the **day-of-month** position means the 1st to the 5th of each month

**Note `workflow_dispatch` alongside it** — so an engineer can trigger a drift check manually after
suspecting a manual change, without waiting for the schedule.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** are true about Bicep compared with Terraform? (Choose three.)

- A. Bicep is stateless — Azure is the source of truth
- B. Bicep is Azure-only
- C. Bicep has no built-in drift detection
- D. Bicep requires a remote state store
- E. Bicep supports multi-cloud deployment
- F. Bicep uses HCL syntax

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-31.md`:** lines **58–64**.

| Criteria | Bicep | Terraform |
|---|---|---|
| Multi-cloud | **Azure only** | Multi-cloud |
| State | **Stateless** | Requires remote state |
| Drift detection | **None built-in** | `terraform plan` |

**A is the operational advantage and C is its cost** — and they are the same fact. No state file means
nothing to store, lock, version or corrupt; it also means no record of intent to diff reality against.

**Why the others fail** — D, E and F describe Terraform. Bicep has its own DSL, not HCL.

</details>

---

## Q18

Which **three** checks does the pipeline run **before** any deployment? (Choose three.)

- A. `az bicep build` — lint and compile
- B. `az deployment sub validate`
- C. `az deployment sub what-if`
- D. `az deployment sub create`
- E. `terraform apply`
- F. `checkov` policy scan

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**F is also a pre-deployment check** (line 604), so treat A, B and C as the intended trio — they are
the three **deployment** checks, in ascending depth, while checkov is a separate policy gate.

**In `challenge-31.md`:** lines **200**, **211**, **234**.

| Check | Contacts Azure? | Answers |
|---|---|---|
| `az bicep build` | **No** | Is it syntactically valid? |
| `az deployment sub validate` | Yes | Would Azure accept it? |
| `az deployment sub what-if` | Yes | What exactly would change? |

**Why the others fail** — D and E are the write operations these checks gate.

</details>

---

## Q19

Which **two** permissions does the drift-detection workflow need that the deployment workflow does
not? (Choose two — one is shared, identify the difference.)

- A. `issues: write`
- B. `id-token: write`
- C. `pull-requests: write`
- D. `contents: write`
- E. `actions: write`

<details>
<summary>Show answer</summary>

### Answer: A (with B shared)

**In `challenge-31.md`:** compare lines **183–186** with lines **701–704**.

```yaml
# Deployment workflow                  # Drift workflow
permissions:                           permissions:
  id-token: write                        id-token: write     <- shared
  contents: read                         contents: read      <- shared
  pull-requests: write                   issues: write       <- the difference
```

**The difference follows from what each workflow produces.** The deployment workflow comments on a
**pull request**; the drift workflow creates an **issue** (line 744), because drift is discovered on a
schedule when no PR exists.

**Why the others fail** — B and shared `contents: read` appear in both; C is the deployment workflow's;
D and E are needed by neither.

</details>

---

## Q20

Which **two** protect the Terraform state file from loss? (Choose two.)

- A. Blob soft delete with a 30-day retention
- B. Blob versioning
- C. A separate state file per environment
- D. `use_oidc = true`
- E. `terraform force-unlock`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-31.md`:** lines **330–333** and **676–678**.

```bash
az storage blob service-properties update \
  --enable-delete-retention true --delete-retention-days 30

az storage blob service-properties update \
  --enable-versioning true
```

**They cover different accidents.** Soft delete recovers a state file someone **deleted**. Versioning
recovers from a state file that was **corrupted or overwritten** — you can list versions (line 681)
and restore an earlier one.

**Why the others fail**

- **C** — limits blast radius, which is valuable and different
- **D** — authentication
- **E** — releases a stale lock; it protects nothing

**Why state deserves this care:** losing it means Terraform no longer knows what it manages, and the
next apply tries to recreate 200 production resources.

</details>

---

## Q21

Which **two** describe `plan on PR, apply on merge`? (Choose two.)

- A. Pull requests run only read-only operations
- B. What-if output is posted to the PR for the reviewer
- C. Pull requests apply to a dev environment for testing
- D. Merges to `main` require a second what-if before applying
- E. Applies run on every push to any branch

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-31.md`:** lines **633–635** and **243–258**.

```yaml
      - name: Post what-if to PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
```

```yaml
  deploy-dev:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

**The `if` conditions are what enforce the split.** The comment step only runs on a PR; the deploy job
only on a push to `main`.

**Why the others fail**

- **C** — a write operation from unreviewed code
- **D** — the what-if already ran; a second one adds nothing before the same apply
- **E** — every branch applying is the opposite of gated

</details>

---

## Q22

Which **two** Bicep decorators improve safety and documentation? (Choose two.)

- A. `@allowed` restricting a parameter to valid values
- B. `@secure` marking a parameter as a secret
- C. `@description` on every parameter
- D. `@minValue` on a string parameter
- E. `@batchSize` on a parameter

<details>
<summary>Show answer</summary>

### Answer: A, B

**C is genuinely good practice** (used at lines 86, 89, 93) and is documentation rather than safety —
so A and B are the intended pair.

**In `challenge-31.md`:** lines **90** (A) and the `secure-parameter-default` rule at line **564** (B).

**Why `@secure` matters:** it stops the value appearing in logs and in the deployment history that
Azure retains. Without it, a password passed as a parameter is readable in the portal afterwards.

**Why the others fail**

- **D** — `@minValue` applies to integers, not strings
- **E** — `@batchSize` controls parallel loop deployment; it is a real decorator applied to
  **resource loops**, not parameters

</details>

---

## Q23

Which **two** problems in Contoso's scenario does IaC directly solve? (Choose two.)

- A. Configuration drift between environments
- B. No audit trail for infrastructure changes
- C. Slow application build times
- D. Insufficient database capacity
- E. Lack of container scanning

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-31.md`:** the scenario at lines **22–28**.

> *"Configuration drift between environments (staging has different SKUs than production)"*
> *"No audit trail for who changed what and when"*

**How each is solved:** drift disappears because every environment deploys the **same template** with
different parameter files (lines 41–45). The audit trail becomes **git history** plus pull request
review — who changed what, when, and who approved it.

The scenario also lists 3 production incidents from manual misconfiguration and a 2-week provisioning
lead time; both follow from the same two causes.

**Why the others fail** — C, D and E are real concerns addressed by other challenges, not by IaC.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must deploy Azure-only infrastructure across five environments. Every change
must be peer-reviewed with visibility of what will change, no credential may be stored, and manual
portal changes must be detected.

---

## Q24

**Proposed solution:** Use Bicep with OIDC authentication. On pull requests run lint, validate and
what-if, posting results as a PR comment. On merge to `main`, apply. Run a scheduled what-if against
production and open an issue when drift is found.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-31.md`:** lines **183–207**, **216–258**, **692–750**.

| Requirement | Mechanism |
|---|---|
| Azure-only, simple | Bicep — stateless, no backend to operate (line 69) |
| Peer review with visibility | What-if posted to the PR (line 243) + branch protection (line 627) |
| No stored credential | OIDC with `id-token: write` (lines 184, 205) |
| Detect manual changes | Scheduled what-if that opens an issue (lines 721–750) |

The drift job compensates for Bicep's missing built-in drift detection — `what-if` against the
template is close enough for this purpose.

</details>

---

## Q25

**Proposed solution:** Use Terraform with a single shared state file for all five environments,
authenticate with a service principal client secret, and run `terraform apply` on every pull request
so reviewers can see the real result.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Three failures, each serious on its own.**

**Applying on a pull request** is a write operation from unreviewed code, against real infrastructure.
It inverts the principle at lines 633–635.

**A single shared state file** means one careless apply can destroy production while targeting dev,
and every environment serialises on one lock (Q10).

**A client secret** violates the no-stored-credential requirement; lines 353 and 359 use `use_oidc`
precisely to avoid it.

Terraform itself is a defensible choice here — drift detection is a stated requirement. Everything
around it is wrong.

</details>

---

## Q26

**Proposed solution:** Use Bicep with OIDC. On pull requests run lint, validate and what-if with the
results posted as a comment. On merge to `main`, apply. Rely on Azure Activity Log for drift.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

The first three requirements are met. Drift detection is not — narrowly, and worth understanding.

**Activity Log records *operations*, not *differences*.** It tells you that someone called
`Microsoft.Network/virtualNetworks/write` at 14:32. It does not tell you whether the current
configuration still matches your template, and it will not surface a change made before logging was
reviewed, or one whose effect was later partially reverted.

**What-if compares desired state to actual state**, so it answers the question that matters: *is
production still what the template says it should be?*

Activity Log is a genuinely useful **complement** — it names the person and the time, which what-if
cannot. It is not a substitute.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — technology choice

| # | Statement | Answer |
|---|---|---|
| 1 | Bicep requires a remote state store |  |
| 2 | Terraform detects drift with `terraform plan` |  |
| 3 | Bicep compiles to ARM templates |  |
| 4 | ARM templates support multi-cloud deployment |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Bicep requires a remote state store | **No** |
| 2 | Terraform detects drift with `terraform plan` | **Yes** |
| 3 | Bicep compiles to ARM templates | **Yes** |
| 4 | ARM templates support multi-cloud deployment | **No** |

**In `challenge-31.md`:** lines **58–64**.

Row 1 is Bicep's main operational advantage: Azure holds the truth, so there is no file to protect,
lock or recover.

Row 3 is not in the table but follows from line 57 ("simplified DSL"): Bicep is a **transpiler** over
ARM, which is why `az bicep build` produces ARM JSON and why deployment behaviour is identical.

</details>

---

## Q28 — pipeline design

| # | Statement | Answer |
|---|---|---|
| 1 | Pull requests should run write operations |  |
| 2 | What-if output should be visible to the reviewer |  |
| 3 | `az bicep build` contacts Azure |  |
| 4 | Branch protection can require the what-if job to pass |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Pull requests should run write operations | **No** |
| 2 | What-if output should be visible to the reviewer | **Yes** |
| 3 | `az bicep build` contacts Azure | **No** |
| 4 | Branch protection can require the what-if job to pass | **Yes** |

**In `challenge-31.md`:** lines **633–635**, **243–258**, **200**, **627**.

Row 3 has a practical consequence: linting can run in a job with **no Azure credentials at all**,
which is why it is the first and cheapest gate.

Row 4 depends on the `contexts` values matching the job `name:` values exactly.

</details>

---

## Q29 — state management

| # | Statement | Answer |
|---|---|---|
| 1 | The azurerm backend locks state using blob leases |  |
| 2 | `terraform force-unlock` should be run whenever a lock error appears |  |
| 3 | Each environment should have its own state file |  |
| 4 | Deleting the state file is a safe way to recover from a lock |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | The azurerm backend locks state using blob leases | **Yes** |
| 2 | `terraform force-unlock` should be run whenever a lock error appears | **No** |
| 3 | Each environment should have its own state file | **Yes** |
| 4 | Deleting the state file is a safe way to recover from a lock | **No** |

**In `challenge-31.md`:** lines **643**, **919–927**, **671**.

Row 2 is the discipline: **verify the lease is stale first** (line 920). Forcing while another apply
is running lets two processes write state at once.

Row 4 is the destructive answer that looks like a fix. Without state, Terraform believes nothing
exists — and the next apply recreates 200 production resources from scratch.

</details>

---

## Q30 — testing and drift

| # | Statement | Answer |
|---|---|---|
| 1 | `checkov` finds security misconfigurations in IaC |  |
| 2 | `terraform fmt -check` rewrites files to canonical format |  |
| 3 | A scheduled what-if can detect drift for Bicep |  |
| 4 | Drift detection should open an issue rather than auto-remediate |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `checkov` finds security misconfigurations in IaC | **Yes** |
| 2 | `terraform fmt -check` rewrites files to canonical format | **No** |
| 3 | A scheduled what-if can detect drift for Bicep | **Yes** |
| 4 | Drift detection should open an issue rather than auto-remediate | **Yes** |

**In `challenge-31.md`:** lines **604**, **579**, **721–750**.

Row 2: `-check` **reports** and exits non-zero without changing anything, which is what you want in
CI. Plain `terraform fmt` rewrites.

Row 4 is a judgement worth stating. The drift workflow creates an issue (line 744) rather than
re-applying, because drift is sometimes an **emergency fix** someone made deliberately at 3am.
Auto-remediating would revert it without anyone knowing. A human decides whether to fold the change
into the template or revert it.

</details>

---

# Section E — Drag and drop

---

## Q31

Arrange the infrastructure pipeline jobs in execution order.

**Items:** `deploy-prod` · `validate` · `deploy-dev` · `what-if`

<details>
<summary>Show answer</summary>

### Answer

1. `validate` — line **192**: lint, then `az deployment sub validate`
2. `what-if` — line **216**: `needs: validate`, posts to the PR
3. `deploy-dev` — line **260**: `needs: what-if`, only on push to `main`
4. `deploy-prod` — line **285**: `needs: deploy-dev`, environment `infrastructure-prod`

**The gate between 2 and 3 is a condition, not a job:**

```yaml
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

On a pull request the pipeline stops after what-if. Only a merge reaches the deploy jobs.

**And note dev before prod** — the same template is proved in a real environment before production
sees it.

</details>

---

## Q32

Match each command to the depth of checking it performs.

| Command | Checks |
|---|---|
| `az bicep build` |  |
| `terraform validate` |  |
| `az deployment sub validate` |  |
| `az deployment sub what-if` |  |
| `terraform plan` |  |
| `az deployment sub create` |  |

**Options:** Applies the change · Azure would accept this template · **Syntax and configuration — no Azure call** (with `-backend=false`) · Syntax and lint — no Azure call · What would actually change · What would change, including drift from state

<details>
<summary>Show answer</summary>

| Command | Checks |
|---|---|
| `az bicep build` | **Syntax and lint — no Azure call** |
| `terraform validate` | **Syntax and configuration — no Azure call** (with `-backend=false`) |
| `az deployment sub validate` | **Azure would accept this template** |
| `az deployment sub what-if` | **What would actually change** |
| `terraform plan` | **What would change, including drift from state** |
| `az deployment sub create` | **Applies the change** |

**In `challenge-31.md`:** lines **200**, **577–578**, **211**, **234**, **279**.

**Order your gates cheapest first.** Local checks need no credentials and fail in seconds; Azure calls
cost time and access. A syntax error should never consume an OIDC token.

</details>

---

## Q33

Match each requirement to its technology.

| Requirement | Choose |
|---|---|
| Azure-only, no state to operate |  |
| Multi-cloud deployment |  |
| Built-in drift detection |  |
| Existing JSON templates to maintain |  |
| Team already knows HCL |  |
| Simplest authoring for Azure resources |  |

**Options:** ARM · Bicep · Terraform

<details>
<summary>Show answer</summary>

| Requirement | Choose |
|---|---|
| Azure-only, no state to operate | **Bicep** |
| Multi-cloud deployment | **Terraform** |
| Built-in drift detection | **Terraform** |
| Existing JSON templates to maintain | **ARM** |
| Team already knows HCL | **Terraform** |
| Simplest authoring for Azure resources | **Bicep** |

**In `challenge-31.md`:** lines **55–70**.

**The recommendation is conditional, not absolute** (lines 69–70). An exam question gives you one
deciding constraint — multi-cloud, drift, existing investment, team skill — and that constraint picks
the tool.

</details>

---

## Q34

Arrange the drift-detection flow in order.

**Items:** Create a GitHub issue · Run what-if against production · Check whether the output contains
`noChange` · Log in to Azure with OIDC · Trigger on schedule

<details>
<summary>Show answer</summary>

### Answer

1. Trigger on schedule — line **697**, `cron: "0 6 * * 1-5"`
2. Log in to Azure with OIDC — line **714**
3. Run what-if against production — line **721**
4. Check for `noChange` — line **730**
5. Create a GitHub issue — line **739**, guarded by `if: steps.drift.outputs.drift_detected == 'true'`

```bash
          if echo "$RESULT" | grep -q "noChange"; then
            echo "drift_detected=false" >> $GITHUB_OUTPUT
          else
            echo "drift_detected=true" >> $GITHUB_OUTPUT
```

**Step 5 only fires when drift exists** — otherwise the workflow runs silently every weekday and
creates nothing. A daily "no drift" issue would train everyone to ignore the label.

</details>

---

## Q35

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Deployment fails: no parameters supplied |  |
| Production deployed by accident from a default |  |
| `terraform apply` blocked by a lock |  |
| Drift job runs but never reports anything |  |
| Required check never satisfied on the PR |  |

**Options:** A crashed run left a stale blob lease · `contexts` name does not match the job `name:` · `noChange` check inverted, or drift genuinely absent · `param environmentName string = 'production'` · `--parameters` flag missing from the command

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| Deployment fails: no parameters supplied | **`--parameters` flag missing from the command** |
| Production deployed by accident from a default | **`param environmentName string = 'production'`** |
| `terraform apply` blocked by a lock | **A crashed run left a stale blob lease** |
| Drift job runs but never reports anything | **`noChange` check inverted, or drift genuinely absent** |
| Required check never satisfied on the PR | **`contexts` name does not match the job `name:`** |

**In `challenge-31.md`:** lines **844**, **811**, **910**, **730**, **627**.

**Break & fix Exercise 1's second error is the dangerous one.** A parameter defaulting to
`'production'` (line 811) means anyone who forgets a parameter file deploys to production. The fix
(line 854) removes the default entirely, forcing an explicit choice.

</details>

---

# Section F — Hot area

---

## Q36

```yaml
permissions:
  [BLANK 1]: write
  contents: read
  [BLANK 2]: write

jobs:
  validate:
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
```

Requirement: authenticate without a stored secret, and post what-if output to the pull request.

- **BLANK 1:** `id-token` / `contents` / `packages` / `deployments`
- **BLANK 2:** `pull-requests` / `issues` / `actions` / `checks`

<details>
<summary>Show answer</summary>

### Answer: `id-token`, `pull-requests`

**In `challenge-31.md`:** lines **184** and **186**.

`issues: write` is the drift workflow's permission (line 704) — it creates issues, not PR comments.

</details>

---

## Q37

```bicep
[BLANK 1] = 'subscription'

@description('Environment to deploy')
@[BLANK 2](['dev', 'test', 'staging', 'prod'])
param environmentName string

resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = { ... }

module networking 'modules/networking/main.bicep' = {
  [BLANK 3]: rg
  ...
}
```

- **BLANK 1:** `targetScope` / `scope` / `deploymentScope` / `level`
- **BLANK 2:** `allowed` / `values` / `restrict` / `oneOf`
- **BLANK 3:** `scope` / `targetScope` / `resourceGroup` / `parent`

<details>
<summary>Show answer</summary>

### Answer: `targetScope`, `allowed`, `scope`

**In `challenge-31.md`:** lines **133**, **90**, **155**.

Note the two different keywords: **`targetScope`** at file level declares where the deployment runs;
**`scope`** on a module says where that module deploys into. Creating a resource group needs the
first; deploying into it needs the second.

`values` is Azure Pipelines parameter syntax (Challenge 20) — offered here as a cross-platform
distractor.

</details>

---

## Q38

```hcl
terraform {
  backend "azurerm" {
    storage_account_name = "stcontosoterraform"
    container_name       = "tfstate"
    key                  = "[BLANK 1]"
    [BLANK 2]            = true
  }
}

provider "azurerm" {
  features {}
  [BLANK 2] = true
}
```

Requirement: isolate state per environment and authenticate without a secret.

- **BLANK 1:** `contoso-$(environment).tfstate` / `terraform.tfstate` / `shared.tfstate` /
  `contoso.tfstate`
- **BLANK 2:** `use_oidc` / `use_msi` / `use_cli` / `use_azuread`

<details>
<summary>Show answer</summary>

### Answer: `contoso-$(environment).tfstate`, `use_oidc`

**In `challenge-31.md`:** lines **671** and **353**/**359**.

A single shared key means one state for all five environments — the blast-radius problem from Q10.

`use_oidc` appears **twice**: backend and provider authenticate separately.

</details>

---

## Q39

```bash
terraform init -[BLANK 1]=false
terraform validate
terraform fmt -[BLANK 2] -recursive
```

Requirement: validate on a pull request runner with no Azure credentials, and fail if formatting is
wrong without modifying files.

- **BLANK 1:** `backend` / `upgrade` / `lock` / `input`
- **BLANK 2:** `check` / `write` / `diff` / `list`

<details>
<summary>Show answer</summary>

### Answer: `backend`, `check`

**In `challenge-31.md`:** lines **577** and **579**.

`-backend=false` skips connecting to remote state, so no credentials are needed. `-check` reports and
exits non-zero; plain `fmt` rewrites files, which is not what CI should do.

</details>

---

## Q40

```yaml
      - name: Run checkov for security scanning
        uses: bridgecrewio/checkov-action@v12
        with:
          framework: [BLANK 1]
          output_format: [BLANK 2]

      - name: Upload SARIF results
        if: [BLANK 3]
        uses: github/codeql-action/upload-sarif@v3
```

- **BLANK 1:** `bicep` / `terraform` / `kubernetes` / `dockerfile`
- **BLANK 2:** `sarif` / `json` / `cli` / `junitxml`
- **BLANK 3:** `always()` / `success()` / `failure()` / `github.event_name == 'push'`

<details>
<summary>Show answer</summary>

### Answer: `bicep`, `sarif`, `always()`

**In `challenge-31.md`:** lines **608–613**.

`always()` matters: checkov fails the step on findings, so without it the SARIF upload is skipped
exactly when there are results to review. Same pattern as Trivy in Challenge 28 and
`PublishTestResults` in Challenge 20.

</details>

---

## Q41

```yaml
  deploy-dev:
    needs: what-if
    if: github.event_name == '[BLANK 1]' && github.ref == '[BLANK 2]'
    environment: [BLANK 3]
```

- **BLANK 1:** `push` / `pull_request` / `schedule` / `workflow_dispatch`
- **BLANK 2:** `refs/heads/main` / `main` / `refs/pull/main` / `heads/main`
- **BLANK 3:** `infrastructure-dev` / `production` / `dev` / *(omit)*

<details>
<summary>Show answer</summary>

### Answer: `push`, `refs/heads/main`, `infrastructure-dev`

**In `challenge-31.md`:** lines **263** and **265**.

`github.ref` holds the **full ref**, so `refs/heads/main`, not `main` — the same value trap as
`Build.SourceBranch` in Azure Pipelines (Challenge 20 Q7).

Omitting the environment would remove the approval gate and the deployment record entirely
(Challenge 24 Q1).

</details>

---

# Section G — Case study

## Case study: Contoso infrastructure modernisation

### Background

Contoso manages **200+ Azure resources across 5 environments** (dev, test, staging, production-east,
production-west), all provisioned manually through the portal by four operations engineers.

**Consequences so far:** configuration drift between environments, no audit trail, **3 production
incidents** last quarter from manual misconfiguration, and a **2-week lead time** to provision a new
environment.

The CTO has mandated IaC with automated testing, peer review and CI/CD deployment.

### Requirements

**Technology and structure**

- Azure-only workloads; the team wants the simplest authoring experience
- One template set must serve all five environments, differing only by parameters
- Modules must be reusable across networking, compute, database and monitoring

**Pipeline**

- Every change must be peer-reviewed with visibility of what will change
- No write operation may run from an unreviewed pull request
- Security misconfigurations must be caught before deployment

**Operations**

- No credential may be stored in the repository
- Manual portal changes to production must be detected
- Production deployment must be approved by a human

---

## Q42

Which technology should Contoso choose, and why?

- A. Bicep — Azure-only, stateless, simplest authoring
- B. Terraform — better drift detection
- C. ARM templates — no new tooling needed
- D. Azure CLI scripts in a pipeline

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-31.md`:** lines **57–70**.

The stated constraints are **Azure-only** and **simplest authoring**, which is precisely what line 69
recommends Bicep for. Being stateless removes an entire operational burden — no storage account to
protect, no lease to unstick, no per-environment state key.

**Why the others fail**

- **B** — a fair choice if drift detection were the *primary* driver, and it brings remote state that
  the requirements do not justify. The scheduled what-if in Q46 covers drift adequately
- **C** — verbose JSON with a steep curve (line 57), and Bicep gives the same deployment engine
- **D** — imperative scripts are not declarative and produce no what-if

</details>

---

## Q43

How should one template set serve five environments?

- A. One `main.bicep` with a parameter file per environment
- B. Five copies of `main.bicep`, one per environment
- C. One template with `if` conditions for each environment
- D. Five separate repositories

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-31.md`:** lines **40–46**.

```text
  environments/
    dev.bicepparam
    test.bicepparam
    staging.bicepparam
    prod-east.bicepparam
    prod-west.bicepparam
  main.bicep
```

**This is what eliminates drift.** Every environment deploys the **same** template; only the
parameters differ. Staging cannot silently acquire a different SKU because there is no separate
staging template to change.

**Why the others fail**

- **B** — five copies drift apart. That is the current problem with extra steps
- **C** — conditionals per environment make the template progressively unreadable, and untested
  branches accumulate
- **D** — five repositories multiply the drift problem across repos

</details>

---

## Q44

Which **two** meet the peer-review requirements? (Choose two.)

- A. What-if output posted as a PR comment
- B. Branch protection requiring the validate and what-if checks
- C. Applying to dev on every pull request
- D. A nightly report of changes made
- E. `enforce_admins: false` so leads can merge quickly

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-31.md`:** lines **243–258** and **625–628**.

**A gives the reviewer the information; B makes the gate enforceable.** Without B, a reviewer could
approve before the checks finish, or merge despite a failure.

**Why the others fail**

- **C** — a write operation from unreviewed code
- **D** — after the fact. Review must happen before the change
- **E** — the challenge sets `enforce_admins=true` (line 628). Exempting admins means the people who
  deploy most often bypass the gate

</details>

---

## Q45

Which configuration catches security misconfigurations before deployment?

- A. `checkov` with `framework: bicep`, uploading SARIF
- B. `az bicep build`
- C. `az deployment sub what-if`
- D. Azure Policy applied after deployment

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-31.md`:** lines **604–616**.

**Why the others fail**

- **B** — syntax and lint rules. A template can lint clean and still open a storage account to the
  internet
- **C** — reports what will change, not whether the change is safe
- **D** — **the useful near-miss.** Azure Policy is a genuine control and it evaluates **after**
  deployment, so a non-compliant resource is either created and then flagged, or denied at deploy
  time with a failure the pipeline must interpret. Catching it in the PR is earlier and cheaper.
  In practice you want both

</details>

---

## Q46

Which configuration detects manual portal changes to production?

- A. A scheduled workflow running what-if against the production parameter file
- B. Azure Activity Log alerts
- C. `terraform plan` — Contoso uses Bicep
- D. A resource lock on the production resource group

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-31.md`:** lines **692–750**.

```bash
          RESULT=$(az deployment sub what-if \
            --template-file main.bicep \
            --parameters @environments/prod-east.bicepparam \
            --no-pretty-print 2>&1)
          if echo "$RESULT" | grep -q "noChange"; then
```

**Why the others fail**

- **B** — records **operations**, not **differences**. It tells you a write happened; it cannot tell
  you whether the current state still matches your template (Q26)
- **C** — Contoso chose Bicep, so there is no `terraform plan`
- **D** — **worth understanding.** A resource lock *prevents* changes rather than detecting them, and
  a `CanNotDelete` or `ReadOnly` lock also blocks **your pipeline**. Locks and IaC conflict unless the
  pipeline removes and reapplies them

</details>

---

## Q47

Eighteen months on, Contoso has 40 Bicep modules. A change to the shared networking module breaks
three environments at once because every template references it by relative path.

What should they do, and what does this illustrate?

- A. Publish modules to a Bicep registry and reference them by version
- B. Copy the module into each environment folder
- C. Stop using modules and inline the resources
- D. Add more tests to the networking module

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-31.md`:** line **60** — *"Modularity: Modules with registry"*.

**The problem is the relative path**, which is an unversioned reference:

```bicep
module networking 'modules/networking/main.bicep' = {     // always the current commit
```

A registry lets consumers pin a version:

```bicep
module networking 'br:contosoregistry.azurecr.io/bicep/modules/networking:v1.2.0' = {
```

**Now each environment adopts a new module version deliberately**, exactly as pinning a reusable
workflow to `@v2` does in Challenge 23. The blast radius of a module change becomes one pull request
per consumer instead of everything at once.

**Why the others fail**

- **B** — 40 copies that drift apart
- **C** — abandons reuse to avoid versioning it
- **D** — **the plausible one.** More tests catch more bugs and change nothing about the fact that
  every consumer takes the change simultaneously. Testing reduces the chance of a bad version;
  versioning reduces the *blast radius* when one slips through

</details>

---

## Q48

A junior engineer's pull request fails the `secure-parameter-default` lint rule. They ask to downgrade
it from `error` to `warning` so the PR can merge.

What should you do?

- A. Keep it as `error` and remove the default from the secure parameter
- B. Downgrade to `warning` and open a follow-up issue
- C. Add the parameter to an exclusion list
- D. Disable `bicepconfig.json` for that module

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-31.md`:** line **564**.

```json
        "secure-parameter-default": { "level": "error" },
```

**What the rule is actually preventing:** a default value on a `@secure()` parameter means a real
credential is written into the template file. That file goes into git — permanently, in history, even
if removed later — and into Azure's deployment history where it is readable in the portal.

**The fix is to remove the default**, so the value must be supplied at deployment time from Key Vault
or a secret.

**Why the others fail**

- **B** — a warning that never blocks is a warning nobody acts on. And the secret is committed in the
  meantime
- **C** — an exclusion for the one case the rule exists to catch
- **D** — disables every rule to avoid one

**The pattern across these challenges:** when a guard blocks you — `BlockOnPossibleDataLoss`, Trivy's
`exit-code`, a rolling upgrade halting, this lint rule — ask whether it is **right** before loosening
it. It nearly always is.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Terraform chosen for Azure-only work** | Q1, Q42 | Bicep unless drift detection or multi-cloud is required |
| **Applying on a pull request** | Q14, Q21, Q25, Q44 | Plan on PR, apply on merge |
| **Shared Terraform state across environments** | Q10, Q25 | One state file per environment |
| **Deleting state to fix a lock** | Q8, Q29 | Verify the lease, then `force-unlock` |
| **`validate` confused with `what-if`** | Q3, Q32 | Validate = acceptable. What-if = what changes |
| **Activity Log offered as drift detection** | Q26, Q46 | It records operations, not differences |
| **Default value on an environment parameter** | Q35, Q48 | No default means an explicit choice |
| **Secure parameter with a default** | Q48 | The secret ends up in git and deployment history |
| **`github.ref` compared to `main`** | Q41 | It is `refs/heads/main` |
| **Missing `if: always()` on SARIF upload** | Q12, Q40 | The scan fails the step, so the upload is skipped |
| **Unversioned module references** | Q47 | Publish to a registry and pin a version |
| **Lint mistaken for policy scanning** | Q12, Q45 | `bicep build` checks syntax; checkov checks security |

---

# The blocks to memorise

Line numbers are in `challenge-31.md`.

```text
# 1. The decision matrix  (lines 55-70)
Criteria           ARM              Bicep                  Terraform
Multi-cloud        Azure only       Azure only             Multi-cloud
State              Stateless        Stateless              Remote state required
Preview            what-if          what-if                terraform plan
Drift detection    none built-in    none built-in          terraform plan
Decision: Bicep for Azure-native. Terraform when drift detection or multi-cloud is needed.

# 2. Checking depth, cheapest first
az bicep build              syntax + lint, NO Azure call
terraform validate          syntax, NO Azure call (with -backend=false)
az deployment sub validate  Azure would accept it
az deployment sub what-if   what would change
az deployment sub create    applies it
```

```yaml
# 3. Least-privilege permissions  (lines 183-186)
permissions:
  id-token: write        # OIDC
  contents: read
  pull-requests: write   # post what-if  (drift workflow uses issues: write)

# 4. OIDC login  (lines 202-207) - memorise this
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ env.AZURE_SUBSCRIPTION_ID }}

# 5. Plan on PR, apply on merge  (lines 244, 263)
      - name: Post what-if to PR
        if: github.event_name == 'pull_request'
  deploy-dev:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

```bicep
// 6. Subscription-scope template  (lines 133-161)
targetScope = 'subscription'
@allowed(['dev', 'test', 'staging', 'prod'])
param environmentName string        // NO default - force an explicit choice
resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = { ... }
module networking 'modules/networking/main.bicep' = {
  scope: rg
}
```

```hcl
# 7. Terraform backend  (lines 348-360, 671)
  backend "azurerm" {
    storage_account_name = "stcontosoterraform"
    container_name       = "tfstate"
    key                  = "contoso-<environment>.tfstate"   # one per environment
    use_oidc             = true
  }
provider "azurerm" { features {}  use_oidc = true }
```

```bash
# 8. State protection and lock recovery  (lines 330, 676, 920-931)
--enable-delete-retention true --delete-retention-days 30   # soft delete
--enable-versioning true                                    # version history
az storage blob show --query "properties.lease.status"      # verify stale FIRST
terraform force-unlock <lock-id>
# prevention: timeoutInMinutes: 30 on the pipeline step
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 31 is exam-ready. Move to Challenge 32 |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 2 and 5, then retake this |
| Below 30 | Redo the challenge, writing the decision matrix from memory first |

Record your result in `AZ-400-Learning-Log.md` under Challenge 31.

:::tip The one thing

**Plan on PR, apply on merge — and never store a credential to do either.**

Read-only analysis where a reviewer can see it, write operations only after approval, OIDC for both.
That sentence answers most of this challenge.

:::
