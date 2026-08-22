---
sidebar_position: 93
title: "Challenge 33: exam questions"
---

# Challenge 33 — AZ-400 exam questions

**48 questions** built only from what Challenge 33 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-33.md`**.

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

:::danger Two identities, and the exam confuses them deliberately

The **developer** holds `Deployment Environments User` — permission to *request* an environment.
The **project environment type's managed identity** is what actually *deploys* into the target
subscription. A developer with the right role still fails if that identity has no permissions. See
Q4, Q19 and Q26.

:::

---

# Section A — Single answer

---

## Q1

Which role lets a developer create environments in a Deployment Environments project?

- A. Contributor on the subscription
- B. Deployment Environments User
- C. DevCenter Project Admin
- D. Owner on the Dev Center

### Answer: B

**In `challenge-33.md`:** lines **59–62**.

```bash
az role assignment create \
  --assignee-object-id "{developer-group-object-id}" \
  --role "Deployment Environments User" \
  --scope ".../projects/proj-contoso-platform"
```

**Note the scope — the *project*, not the subscription.** That is the whole point of the model: a
developer needs no standing permission on the target subscription at all. They ask the platform for an
environment, and the platform's identity provisions it.

**Why the others fail**

- **A** — subscription Contributor is exactly the shadow-IT problem the scenario describes (line 26).
  It lets developers create anything, anywhere, ungoverned
- **C** — `DevCenter Project Admin` is real and grants project **administration**: managing
  environment types and settings. More than a developer needs
- **D** — Owner on the Dev Center is administrative, and still not the role that grants environment
  creation

**The role trio to know:** `Deployment Environments User` (create and manage your own),
`DevCenter Project Admin` (manage the project), `Deployment Environments Reader` (view only).

---

## Q2

What does a **project environment type** map an environment type to?

- A. A target Azure subscription with a deploying identity and roles
- B. A resource group naming convention
- C. A catalog repository
- D. A Dev Box definition

### Answer: A

**In `challenge-33.md`:** lines **88–95**.

```bash
az devcenter admin project-environment-type create \
  --name Dev \
  --project-name proj-contoso-platform \
  --deployment-target-id "/subscriptions/{dev-sub-id}" \
  --identity-type SystemAssigned \
  --roles "{\"8e3af657-a8ff-443c-a75c-2fe8c4bcb635\":{}}" \
  --status Enabled
```

**Three things are bound together here**, and this is the central object of the challenge:

| Setting | Meaning |
|---|---|
| `--deployment-target-id` | **Which subscription** environments of this type land in |
| `--identity-type SystemAssigned` | **Who deploys** — the identity that acts |
| `--roles` | **What the developer gets** on the created environment |

Dev, Test and Staging each point at a **different subscription** (lines 92, 102, 112), so environment
type is how cost and blast radius are separated.

**Why the others fail** — B, C and D are all different concepts. Catalogs attach to the **Dev Center**
(line 289), not to an environment type.

---

## Q3

Where are environment **types** defined, and where are they mapped to subscriptions?

- A. Types at the Dev Center; mapping at the project
- B. Types at the project; mapping at the Dev Center
- C. Both at the Dev Center
- D. Both at the project

### Answer: A

**In `challenge-33.md`:** lines **71–84** (Dev Center) and **88–115** (project).

```bash
# Dev Center level - just the NAME
az devcenter admin environment-type create --name Dev --dev-center-name dc-contoso

# Project level - the name plus WHERE it deploys and WHO deploys
az devcenter admin project-environment-type create --name Dev \
  --project-name proj-contoso-platform \
  --deployment-target-id "/subscriptions/{dev-sub-id}"
```

**Why the two levels exist:** the Dev Center declares a **vocabulary** — every project agrees on what
"Dev" and "Staging" mean. Each project then decides **its own** target subscription for that name. Two
projects can both have "Dev" pointing at entirely different subscriptions.

**Why the others fail** — all three invert or collapse the split.

---

## Q4

A developer holding `Deployment Environments User` gets `AuthorizationFailed` on
`Microsoft.Resources/deployments/write` in the target subscription.

What is the cause?

- A. The developer needs Contributor on the target subscription
- B. The project environment type's managed identity has no role on the target subscription
- C. The catalog has not synced
- D. The environment type is disabled

### Answer: B

**In `challenge-33.md`:** Break & fix Exercise 2, lines **670–697**.

```bash
PRINCIPAL_ID=$(az devcenter admin project-environment-type show \
  --name Dev --project-name proj-contoso-platform \
  --query "identity.principalId" -o tsv)

az role assignment create \
  --assignee-object-id "$PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Contributor" \
  --scope "/subscriptions/{dev-sub-id}"
```

**Two identities, two jobs.** The developer's role authorises the **request**. The project environment
type's managed identity performs the **deployment**. The error names a subscription-scoped action, and
the developer never touches that subscription.

**Why the others fail**

- **A** — **the trap, and it defeats the entire model.** Giving developers subscription Contributor
  reintroduces shadow IT (line 26). The whole design exists so they need no standing access
- **C** — a sync failure gives "environment definition not found" (line 616), a different message
- **D** — a disabled type gives a different error and would block everyone, not just deployment

---

## Q5

Which file declares the parameters a developer is prompted for?

- A. `main.bicep`
- B. `environment.yaml`
- C. `azuredeploy.parameters.json`
- D. `catalog.json`

### Answer: B

**In `challenge-33.md`:** lines **139–170**.

```yaml
name: WebApp
version: 1.0.0
summary: Single web application with database
templatePath: main.bicep
parameters:
  - id: appServicePlanSku
    name: App Service Plan SKU
    type: string
    default: B1
    allowed:
      - F1
      - B1
      - S1
      - P1v3
```

**`environment.yaml` is the contract with the developer**; `main.bicep` is the implementation. The
`allowed` list is what turns a free-text field into a dropdown in the Developer Portal — and it is a
**guardrail**: a developer cannot request a P1v3 in Dev if it is not on the list.

**Note `templatePath: main.bicep`** at line 144 — that is the link between the two files.

**Why the others fail** — C is ARM parameter-file syntax, D does not exist in this model, and A holds
the resources.

---

## Q6

Which parameter is automatically supplied by Azure Deployment Environments?

- A. `environmentName`
- B. `appServicePlanSku`
- C. `sqlDatabaseSku`
- D. `enableApplicationInsights`

### Answer: A

**In `challenge-33.md`:** lines **174–175**.

```bicep
@description('Name for the environment (auto-populated by ADE)')
param environmentName string
```

**It is not in `environment.yaml`'s parameter list** (lines 145–169) precisely because the developer
does not supply it — the platform passes the environment name they chose.

**Why this matters in the template:** it is what makes resource names unique per environment. Forty
developers each creating a "WebApp" need forty differently named App Services, and `environmentName`
is the discriminator.

**Why the others fail** — all three appear in the `parameters` list and are chosen by the developer.

---

## Q7

Where does the Dev Center store the credential used to read a GitHub catalog?

- A. In the catalog configuration as plain text
- B. In Azure Key Vault, referenced by `secret-identifier`
- C. In a GitHub Actions secret
- D. In the project's environment type

### Answer: B

**In `challenge-33.md`:** lines **289–296**.

```bash
az devcenter admin catalog create \
  --name contoso-environments \
  --git-hub path="/environments" \
    branch="main" \
    uri="https://github.com/contoso/environment-catalog.git" \
    secret-identifier="https://kv-contoso-devcenter.vault.azure.net/secrets/github-pat"
```

**The Dev Center reads the secret using its own managed identity** — which is why it was created with
`--identity-type SystemAssigned` at line 47. Nothing is stored in the Dev Center itself.

**The follow-on step people miss:** that managed identity needs **Key Vault access** to the secret, or
the catalog sync fails on the very first attempt.

**Why the others fail** — A is what the design avoids, C is the wrong platform, D is unrelated.

---

## Q8

A developer sees `The environment definition 'WebApp' was not found in catalog 'contoso-environments'.`

What should you check first?

- A. The developer's role assignment
- B. The catalog's sync state and sync error details
- C. The target subscription quota
- D. The Bicep template syntax

### Answer: B

**In `challenge-33.md`:** Break & fix Exercise 1, lines **611–665**.

```bash
az devcenter admin catalog show --query "{Status:syncState, LastSync:lastSyncTime}"
# Returns: {"Status": "Failed", "LastSync": "2024-01-10T08:00:00Z"}

az devcenter admin catalog get-sync-error-details ...
# Returns: "Path '/environments' not found in repository"
```

**The root cause was the `path` setting** — definitions sat at the repository root, not under
`/environments`. The fix (line 656) recreates the catalog with `path="/"` and re-syncs.

**Read the error precisely.** "Definition not found **in catalog**" points at the catalog, not at the
developer or the template. A permissions problem gives `AuthorizationFailed` (line 675); a template
problem gives a deployment error after provisioning starts.

**Why the others fail** — A, C and D produce different, distinguishable errors.

---

## Q9

Which command does a **developer** use to create an environment?

- A. `az devcenter admin environment-type create`
- B. `az devcenter dev environment create`
- C. `az deployment sub create`
- D. `az devcenter admin project create`

### Answer: B

**In `challenge-33.md`:** lines **333–340**.

```bash
az devcenter dev environment create \
  --name "feature-auth-redesign" \
  --project-name proj-contoso-platform \
  --environment-type Dev \
  --catalog-name contoso-environments \
  --environment-definition-name WebApp \
  --parameters '{"appServicePlanSku": "B1", ...}'
```

**The CLI splits into `admin` and `dev` deliberately.** `admin` commands configure the platform;
`dev` commands are what a developer runs day to day. Spotting which half a command belongs to answers
several exam questions on its own.

**Why the others fail**

- **A** and **D** — `admin` commands, run by the platform team
- **C** — deploying Bicep directly, bypassing the whole governance model

**Three ways to create an environment** (lines 329, 364): the CLI, the **Developer Portal** at
`devportal.microsoft.com`, and the API — which is what the pipeline in Task 7 uses.

---

## Q10

How does the PR pipeline name each ephemeral environment?

- A. `pr-$(System.PullRequest.PullRequestNumber)`
- B. `$(Build.BuildId)`
- C. `$(Build.SourceBranchName)`
- D. A random GUID

### Answer: A

**In `challenge-33.md`:** lines **471–472**.

```yaml
variables:
  - name: environmentName
    value: "pr-$(System.PullRequest.PullRequestNumber)"
```

**The PR number is stable across pushes**, which is what makes the pipeline idempotent: a second push
to the same PR finds the existing environment rather than creating a duplicate (lines 493–511).

**Why the others fail**

- **B** — `Build.BuildId` changes on **every run**, so each push would create a new environment and the
  old ones would accumulate
- **C** — branch names contain characters like `/` that are invalid in resource names, and two PRs
  from similar branches could collide
- **D** — a random name cannot be found again for cleanup

---

## Q11

Why does the pipeline check whether the environment exists before creating it?

- A. To make the stage idempotent across repeated pushes to the same PR
- B. To save Azure quota
- C. Because creation always fails on the second attempt
- D. To validate the catalog

### Answer: A

**In `challenge-33.md`:** lines **492–511**.

```bash
                EXISTS=$(az devcenter dev environment show \
                  --name "$(environmentName)" ... \
                  --query "name" -o tsv 2>/dev/null || true)

                if [ -z "$EXISTS" ]; then
                  echo "Creating environment $(environmentName)..."
                  ...
                else
                  echo "Environment $(environmentName) already exists."
                fi
```

**A pull request runs its pipeline on every push.** Without the check, the second commit would attempt
to create an environment that already exists — and the stage would fail on a PR that is perfectly
healthy.

**Note `2>/dev/null || true`** — the `show` command errors when the environment does not exist, and
`|| true` stops that expected error failing the step under `set -e`.

**Why the others fail** — B is a side effect, C overstates it, D is unrelated.

---

## Q12

How does the pipeline wait for provisioning to complete?

- A. A fixed `sleep 300`
- B. A loop polling `provisioningState` until `Succeeded`
- C. `az devcenter dev environment wait`
- D. It does not wait

### Answer: B

**In `challenge-33.md`:** lines **521–533**.

```bash
                for i in $(seq 1 30); do
                  STATUS=$(az devcenter dev environment show ... \
                    --query "provisioningState" -o tsv)
                  if [ "$STATUS" == "Succeeded" ]; then
                    break
                  fi
                  echo "Waiting for environment... (attempt $i, status: $STATUS)"
                  sleep 30
                done
```

Thirty attempts, thirty seconds apart — a **fifteen-minute** ceiling.

**Why polling beats a fixed sleep:** environment provisioning time varies with what the template
creates. A fixed sleep is either too short (the next step fails) or wastes minutes on every run. Same
argument as `rollout status` in Challenge 28 and the health-check loops in Challenges 25 and 26.

**Why the others fail**

- **A** — the guess this pattern replaces
- **C** — plausible, and not the command used here
- **D** — the URL query at line 535 would return nothing

---

## Q13

How does the pipeline expose the environment URL to later stages?

- A. `##vso[task.setvariable variable=envUrl;isOutput=true]$URL`
- B. Writing it to a pipeline artifact
- C. `echo "envUrl=$URL" >> $GITHUB_OUTPUT`
- D. A variable group

### Answer: A

**In `challenge-33.md`:** lines **535–540**.

```bash
                URL=$(az devcenter dev environment show ... \
                  --query "outputs.webAppUrl.value" -o tsv)
                echo "##vso[task.setvariable variable=envUrl;isOutput=true]$URL"
```

**`isOutput=true` plus a named task** (line 515, `name: getUrl`) is the Azure Pipelines output-variable
pattern from Challenge 20. Without the `name:`, the value cannot be referenced.

**Note the query path — `outputs.webAppUrl.value`.** Those are the **Bicep template's outputs**,
surfaced by the environment. That is how a developer or a pipeline learns the URL of something they
did not deploy themselves.

**Why the others fail** — B works and is clumsy, C is GitHub Actions syntax, D holds static values.

---

## Q14

Which mechanism enforces automatic cleanup of idle Dev environments?

- A. A scheduled Automation runbook deleting resource groups past their expiry tag
- B. `--max-dev-boxes-per-user`
- C. Deleting the environment type
- D. Azure Advisor recommendations

### Answer: A

**In `challenge-33.md`:** lines **430–445**.

```powershell
Connect-AzAccount -Identity

$cutoffDate = (Get-Date).AddDays(-$MaxAgeDays)

$expiredGroups = Get-AzResourceGroup |
    Where-Object {
        $_.Tags['ade-environment-type'] -eq 'Dev' -and
        [DateTime]$_.Tags['auto-delete-after'] -lt $cutoffDate
    }
```

**Two pieces working together.** The Azure Policy at lines 384–425 uses a `modify` effect to **stamp**
the expiry tag on every Dev environment resource group; the runbook then **acts** on that tag on a
schedule.

That split is deliberate: policy is good at tagging consistently, and bad at deleting things.

**Why the others fail**

- **B** — caps how many you can have **at once**; it does nothing about ones sitting idle
- **C** — would break every existing environment of that type
- **D** — Advisor recommends; it does not act

**This directly addresses the scenario** (line 24): environments idle 80% of the time.

---

## Q15

Which control limits how many environments a single developer can hold?

- A. `--max-dev-boxes-per-user` on the project
- B. Azure subscription quota
- C. The `allowed` list in `environment.yaml`
- D. The Deployment Environments User role

### Answer: A

**In `challenge-33.md`:** lines **56** and **377–380**.

```bash
az devcenter admin project update \
  --name proj-contoso-platform \
  --max-dev-boxes-per-user 3
```

**Two complementary cost controls:** this caps **concurrent** environments per person, and the
auto-delete runbook (Q14) caps **age**. Without the second, three long-lived environments per
developer across 40 developers is 120 environments running indefinitely.

**Why the others fail**

- **B** — a hard platform ceiling, not a per-developer governance control
- **C** — restricts **SKU choices**, which is a different guardrail: it stops someone provisioning a
  P1v3 for a dev environment
- **D** — grants the ability to create; it sets no limit

---

## Q16

Which two catalog sources can a Dev Center use?

- A. GitHub and Azure Repos Git
- B. GitHub and Azure Storage
- C. Azure Container Registry and GitHub
- D. Azure Repos Git only

### Answer: A

**In `challenge-33.md`:** lines **289–296** (`--git-hub`) and **317–324** (`--ado-git`).

```bash
  --git-hub path="/environments" branch="main" \
    uri="https://github.com/contoso/environment-catalog.git" \
    secret-identifier="https://kv-.../secrets/github-pat"

  --ado-git path="/environments" branch="main" \
    uri="https://dev.azure.com/contoso/Platform/_git/environment-catalog" \
    secret-identifier="https://kv-.../secrets/ado-pat"
```

**Both take the same four settings** — path, branch, URI and a Key Vault secret identifier — which
makes them easy to confuse in a question. The only difference is the flag.

**Why the others fail** — Storage and ACR are not catalog sources; D omits GitHub.

**Why a git repository at all:** the catalog is **environment definitions as code**, so it gets pull
request review, version history and branch protection. Same reasoning as Challenge 31's IaC pipeline.

---

# Section B — Multiple answer

---

## Q17

Which **three** components must exist before a developer can create an environment? (Choose three.)

- A. A Dev Center with a project
- B. A project environment type mapped to a target subscription
- C. A synced catalog containing environment definitions
- D. A Dev Box definition
- E. A Log Analytics workspace
- F. A public IP on the target subscription

### Answer: A, B, C

**In `challenge-33.md`:** lines **43–56**, **88–95**, **289–299**.

**The three are a chain**, and the exam breaks one link at a time:

| Component | Answers |
|---|---|
| Dev Center + project | **Who** can ask, and for what |
| Project environment type | **Where** it deploys and **who** deploys it |
| Synced catalog | **What** can be deployed |

Break the third and you get "environment definition not found" (Q8). Break the second and you get
`AuthorizationFailed` (Q4).

**Why the others fail** — Dev Box is a separate product sharing the same Dev Center; E and F are
unrelated.

---

## Q18

Which **two** are configured on a **project environment type**? (Choose two.)

- A. The target subscription (`--deployment-target-id`)
- B. The deploying managed identity (`--identity-type`)
- C. The catalog repository URI
- D. The environment definition's parameters
- E. The maximum environments per user

### Answer: A, B

**In `challenge-33.md`:** lines **92–93**.

**Why the others fail — and each belongs somewhere specific**

- **C** — the **Dev Center**, via `az devcenter admin catalog create` (line 289)
- **D** — `environment.yaml` in the catalog (line 145)
- **E** — the **project**, via `--max-dev-boxes-per-user` (line 380)

**Knowing which object owns which setting is most of this challenge.** Dev Center owns catalogs and
type names; project owns limits and subscription mappings; catalog owns definitions and parameters.

---

## Q19

Which **two** identities are involved when a developer creates an environment? (Choose two.)

- A. The developer, holding `Deployment Environments User` on the project
- B. The project environment type's managed identity, deploying into the subscription
- C. The developer's personal subscription Contributor role
- D. The Dev Center's managed identity, deploying resources
- E. A service principal stored in the catalog

### Answer: A, B

**In `challenge-33.md`:** lines **61** and **93**, with the failure at lines **674–697**.

**D is the near-miss worth being precise about.** The Dev Center **does** have a managed identity
(line 47), and it is used to read the **catalog secret** from Key Vault (Q7). It is not what deploys
resources — that is the project environment type's identity.

So there are three identities in total, each with one job:

| Identity | Job |
|---|---|
| Developer | Request an environment |
| Dev Center | Read the catalog credential from Key Vault |
| Project environment type | Deploy into the target subscription |

**Why the others fail** — C is the shadow-IT anti-pattern; E stores a credential where it does not
belong.

---

## Q20

Which **two** does `environment.yaml` define? (Choose two.)

- A. The parameters a developer is prompted for
- B. The path to the IaC template
- C. The target subscription
- D. The deploying identity
- E. The maximum environment lifetime

### Answer: A, B

**In `challenge-33.md`:** lines **144–169**.

```yaml
templatePath: main.bicep        # B
parameters:                     # A
  - id: appServicePlanSku
    type: string
    default: B1
    allowed: [F1, B1, S1, P1v3]
```

**Why the others fail** — C and D belong to the project environment type; E is enforced by the
auto-delete runbook (line 430), not declared in the definition.

**The `allowed` list is a governance control, not just UI polish.** It is what stops a developer
provisioning a P1v3 plan for a throwaway environment, and it is enforced regardless of whether they
use the portal, the CLI or the API.

---

## Q21

Which **two** cost controls does the challenge implement? (Choose two.)

- A. `--max-dev-boxes-per-user 3`
- B. A scheduled runbook deleting environments older than 7 days
- C. Azure Reservations
- D. Deleting the Dev Center nightly
- E. Restricting developers to the F1 SKU only

### Answer: A, B

**In `challenge-33.md`:** lines **380** and **430–445**.

**They control different dimensions.** The limit caps **how many** exist concurrently; the runbook caps
**how long** each survives. Either alone is insufficient — three environments per developer with no
expiry is 120 permanent environments across 40 developers.

**E is close but overstated:** the `allowed` list (lines 151–155) *includes* F1 and does not restrict
to it. Restricting the SKU list is a real third control; the two implemented are A and B.

**Why the others fail** — C is a billing commitment unrelated to environment sprawl; D destroys the
platform.

---

## Q22

Which **two** are true about the PR-environment pipeline? (Choose two.)

- A. It names environments by pull request number for idempotency
- B. It polls `provisioningState` until `Succeeded` before reading outputs
- C. It creates a new environment on every push to the PR
- D. It uses `$(Build.BuildId)` as the environment name
- E. It deploys the Bicep template directly with `az deployment sub create`

### Answer: A, B

**In `challenge-33.md`:** lines **472**, **493–511**, **521–533**.

**Why the others fail**

- **C** — the opposite. The existence check (line 499) is what prevents it
- **D** — would change on every run, defeating the idempotency
- **E** — bypasses Deployment Environments entirely, losing the governance, the tagging and the
  cleanup story

**What this pattern buys you:** every pull request gets a real, isolated environment with the same
definition production uses — which is exactly the "inconsistency between developer environments and
production" problem from line 25.

---

## Q23

Which **two** problems in Contoso's scenario does Deployment Environments solve? (Choose two.)

- A. A 3–5 day IT ticket wait for a dev environment
- B. Shadow IT from developers provisioning outside governance
- C. Slow application build times
- D. Container image vulnerabilities
- E. Database migration ordering

### Answer: A, B

**In `challenge-33.md`:** the scenario at lines **20–27**.

**How each is solved:** self-service turns a multi-day ticket into a CLI command or a portal form.
Shadow IT disappears because the *sanctioned* path is now faster than going around it — developers
route around governance when governance is slow, not because they want to.

The scenario also lists shared environments causing conflicts, and idle cost — solved by per-developer
environments and the auto-delete runbook respectively.

**Why the others fail** — C, D and E belong to other challenges.

---

# Section C — Repeated scenario

**Scenario:** Contoso's 40 developers must create their own dev environments on demand, with no
standing access to the target subscriptions, a capped SKU list, and automatic cleanup after 7 days.

---

## Q24

**Proposed solution:** Create a Dev Center and project, define a Dev environment type mapped to the
dev subscription with a system-assigned identity granted Contributor there, publish a catalog with an
`allowed` SKU list, grant developers `Deployment Environments User` on the project, and schedule a
runbook that deletes environments older than 7 days.

Does this meet the goal? **Yes**

### Answer: Yes

**In `challenge-33.md`:** lines **43–62**, **88–95**, **151–155**, **430–445**.

| Requirement | Mechanism |
|---|---|
| Self-service | `az devcenter dev environment create` or the Developer Portal |
| No standing subscription access | Developers hold a **project-scoped** role only |
| Capped SKUs | `allowed` list in `environment.yaml` |
| Automatic cleanup | Scheduled runbook on the expiry tag |

The Contributor grant on the **project environment type's identity** is the step that makes
deployments actually work (Break & fix Exercise 2).

---

## Q25

**Proposed solution:** Grant all 40 developers Contributor on the dev subscription and share Bicep
templates in a repository so they can deploy their own environments with `az deployment sub create`.

Does this meet the goal? **No**

### Answer: No

**This is the shadow IT the scenario is trying to eliminate** (line 26), formalised.

**No standing access: violated outright.** Forty subscription Contributors can create anything,
anywhere in that subscription — not just what the templates define.

**Capped SKUs: unenforceable.** The template's `@allowed` decorator only constrains people who use the
template. A Contributor can deploy whatever they like directly.

**Cleanup: nothing tracks what was created**, by whom, or when. There is no `ade-environment-type` tag
to key a runbook on.

**What it does solve** is the 3–5 day wait — which is exactly why teams drift into this pattern. Speed
without guardrails.

---

## Q26

**Proposed solution:** Create the Dev Center, project, environment type and catalog. Grant developers
`Deployment Environments User` on the project. Publish the catalog with an `allowed` SKU list and
schedule the cleanup runbook — but leave the project environment type's identity without a role on
the dev subscription.

Does this meet the goal? **No**

### Answer: No

Every requirement is designed correctly and **nothing can be deployed**.

The developer's request is authorised — they hold the right role on the project — so the environment
creation is accepted. It then fails during deployment with:

```text
ERROR: AuthorizationFailed - The client does not have authorization to perform action
'Microsoft.Resources/deployments/write' over scope '/subscriptions/{dev-sub-id}/...'
```

**The failure is confusing precisely because the developer's permissions are right.** The error names
a subscription the developer has no relationship with, which sends people to check the developer's
role — the one thing that is already correct.

**The rule to carry:** in Deployment Environments, *requesting* and *deploying* are two different
authorisations held by two different principals.

---

# Section D — Yes/No statement grid

---

## Q27 — structure

| # | Statement | Answer |
|---|---|---|
| 1 | Environment types are defined at the Dev Center | **Yes** |
| 2 | Project environment types map a type to a subscription | **Yes** |
| 3 | Catalogs attach to the project | **No** |
| 4 | A project can limit environments per user | **Yes** |

**In `challenge-33.md`:** lines **71–84**, **92**, **289–292**, **380**.

Row 3 is the one to get right: `az devcenter admin catalog create` takes `--dev-center-name`, not
`--project-name`. Catalogs are **shared across every project** in the Dev Center, which is what makes
one blessed set of environment definitions reachable organisation-wide.

---

## Q28 — permissions

| # | Statement | Answer |
|---|---|---|
| 1 | `Deployment Environments User` is assigned at project scope | **Yes** |
| 2 | Developers need Contributor on the target subscription | **No** |
| 3 | The project environment type's identity deploys the resources | **Yes** |
| 4 | The Dev Center's identity reads the catalog secret from Key Vault | **Yes** |

**In `challenge-33.md`:** lines **62**, **93**, **47** and **296**.

**Rows 2 and 3 together are the model.** Developers get a scoped role on the *project*; the platform's
identity holds the powerful role on the *subscription*. That inversion is what removes standing
access without removing self-service.

Row 4 is the third identity, and it explains why the Dev Center needs one at all.

---

## Q29 — catalogs and definitions

| # | Statement | Answer |
|---|---|---|
| 1 | A catalog can be a GitHub or Azure Repos Git repository | **Yes** |
| 2 | The catalog credential is stored in Key Vault | **Yes** |
| 3 | `environment.yaml` declares the developer-facing parameters | **Yes** |
| 4 | A failed sync produces an `AuthorizationFailed` error to the developer | **No** |

**In `challenge-33.md`:** lines **289**/**317**, **296**, **145**, **616**.

Row 4 separates the two Break & fix exercises by their **error message**:

| Message | Cause |
|---|---|
| `environment definition ... was not found in catalog` | Catalog sync failed or wrong `path` |
| `AuthorizationFailed ... deployments/write` | Environment type identity lacks a role |

Reading the message tells you which of the two to investigate, which is worth an exam mark on its own.

---

## Q30 — governance and lifecycle

| # | Statement | Answer |
|---|---|---|
| 1 | `--max-dev-boxes-per-user` limits concurrent environments | **Yes** |
| 2 | The `allowed` parameter list constrains SKU choices | **Yes** |
| 3 | Deployment Environments deletes idle environments automatically | **No** |
| 4 | Azure Policy can tag environment resource groups with an expiry | **Yes** |

**In `challenge-33.md`:** lines **380**, **151–155**, **384–425**, **430–445**.

**Row 3 is the assumption that costs money.** There is no built-in idle-expiry. The challenge builds
it from two parts — a policy that **tags** and a runbook that **deletes** — and without that
combination, environments live until someone remembers them.

Given the scenario says environments sit idle 80% of the time (line 24), that gap is the entire cost
problem.

---

# Section E — Drag and drop

---

## Q31

Arrange the platform setup steps in order.

**Items:** Create project environment types mapped to subscriptions · Create the Dev Center · Attach
the catalog · Grant developers `Deployment Environments User` · Create the project · Create
environment types

### Answer

1. Create the Dev Center — line **43**
2. Create the project — line **50**
3. Create environment types (Dev Center level) — line **71**
4. Create project environment types mapped to subscriptions — line **88**
5. Attach the catalog — line **289**
6. Grant developers `Deployment Environments User` — line **59**

**Steps 3 and 4 must be in that order** — the project mapping references a type name that has to exist
at the Dev Center first.

**Step 6 last is deliberate.** Granting access before the platform works means developers hit
failures on their first attempt, which is how self-service tools acquire a reputation nobody can shake.

---

## Q32

Match each object to what it owns.

| Object | Owns |
|---|---|
| Dev Center | **Catalogs and environment type names** |
| Project | **Per-user limits and developer role assignments** |
| Project environment type | **Target subscription and deploying identity** |
| Catalog | **Environment definitions** |
| `environment.yaml` | **Developer-facing parameters and template path** |
| `main.bicep` | **The resources actually deployed** |

**In `challenge-33.md`:** lines **43**, **50**, **88**, **289**, **139**, **173**.

**Most wrong answers in this challenge are ownership mistakes** — putting the catalog on the project,
the subscription on the Dev Center, or the parameters in the Bicep file. Learn the table, not the
individual commands.

---

## Q33

Match each identity to its role in one environment creation.

| Identity | Role |
|---|---|
| Developer | **Requests the environment** (`Deployment Environments User` on the project) |
| Dev Center managed identity | **Reads the catalog credential from Key Vault** |
| Project environment type identity | **Deploys resources into the target subscription** |

**In `challenge-33.md`:** lines **61**, **47** with **296**, **93** with **694**.

**Three principals, three scopes, and none of them is the developer touching the subscription.** That
separation is the entire security argument for the product — and Q4, Q19 and Q26 all attack it from
different angles.

---

## Q34

Arrange the PR-environment pipeline steps in order.

**Items:** Publish the URL as an output variable · Check whether the environment already exists ·
Poll `provisioningState` until `Succeeded` · Create the environment if absent · Read
`outputs.webAppUrl.value`

### Answer

1. Check whether the environment already exists — line **493**
2. Create the environment if absent — line **501**
3. Poll `provisioningState` until `Succeeded` — line **522**
4. Read `outputs.webAppUrl.value` — line **535**
5. Publish the URL as an output variable — line **540**

**Each step exists because of a specific failure:**

- Skip 1 → the second push to the PR fails on "already exists"
- Skip 3 → step 4 reads an empty output from a half-provisioned environment
- Skip 5 → later stages have no way to reach the environment

---

## Q35

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| `environment definition 'WebApp' was not found in catalog` | **Catalog sync failed — wrong `path`** |
| `AuthorizationFailed` on `deployments/write` | **Project environment type identity has no subscription role** |
| The catalog never syncs at all | **Dev Center identity cannot read the Key Vault secret** |
| PR pipeline fails on the second push | **No existence check before create** |
| Dev environments accumulate indefinitely | **No expiry tag policy or cleanup runbook** |

**In `challenge-33.md`:** lines **616–636**, **675–697**, **296**, **499**, **430–445**.

**The middle row is the one with no worked example in the challenge** — but it follows directly from
Q7. The Dev Center reads the PAT from Key Vault using its own managed identity, so if that identity
has no Key Vault access, the sync never even starts.

---

# Section F — Hot area

---

## Q36

```bash
az devcenter admin devcenter create \
  --name dc-contoso \
  --identity-type [BLANK 1]

az role assignment create \
  --assignee-object-id "{developer-group-object-id}" \
  --role "[BLANK 2]" \
  --scope ".../projects/proj-contoso-platform"
```

- **BLANK 1:** `SystemAssigned` / `UserAssigned` / `None` / `SystemAndUserAssigned`
- **BLANK 2:** `Deployment Environments User` / `Contributor` / `DevCenter Project Admin` /
  `Owner`

### Answer: `SystemAssigned`, `Deployment Environments User`

**In `challenge-33.md`:** lines **47** and **61**.

The Dev Center's identity is what reads the catalog secret from Key Vault. The developer role is
scoped to the **project**, deliberately never to a subscription.

---

## Q37

```bash
az devcenter admin project-environment-type create \
  --name Dev \
  --project-name proj-contoso-platform \
  --[BLANK 1] "/subscriptions/{dev-sub-id}" \
  --identity-type SystemAssigned \
  --[BLANK 2] "{\"8e3af657-a8ff-443c-a75c-2fe8c4bcb635\":{}}" \
  --status [BLANK 3]
```

- **BLANK 1:** `deployment-target-id` / `subscription-id` / `scope` / `target-resource-id`
- **BLANK 2:** `roles` / `permissions` / `role-assignments` / `access`
- **BLANK 3:** `Enabled` / `Active` / `On` / `Available`

### Answer: `deployment-target-id`, `roles`, `Enabled`

**In `challenge-33.md`:** lines **92–95**.

That GUID is the built-in **Contributor** role definition ID — the role granted to the developer **on
the environment they create**, so they can work with the resources inside it.

`--status Enabled` matters: a disabled type stays configured but accepts no new environments, which is
how you retire a type without breaking existing ones.

---

## Q38

```yaml
name: WebApp
version: 1.0.0
[BLANK 1]: main.bicep
parameters:
  - id: appServicePlanSku
    type: string
    default: B1
    [BLANK 2]:
      - F1
      - B1
      - S1
      - P1v3
```

- **BLANK 1:** `templatePath` / `template` / `bicepFile` / `path`
- **BLANK 2:** `allowed` / `values` / `options` / `choices`

### Answer: `templatePath`, `allowed`

**In `challenge-33.md`:** lines **144** and **151**.

`values` is Azure Pipelines parameter syntax (Challenge 20); `options` is GitHub Actions
`workflow_dispatch` (Challenge 19). Three platforms, three keywords for the same idea — and the exam
mixes them deliberately.

---

## Q39

```bash
az devcenter admin catalog create \
  --name contoso-environments \
  --dev-center-name dc-contoso \
  --[BLANK 1] path="/environments" \
    branch="main" \
    uri="https://github.com/contoso/environment-catalog.git" \
    [BLANK 2]="https://kv-contoso-devcenter.vault.azure.net/secrets/github-pat"
```

- **BLANK 1:** `git-hub` / `ado-git` / `git` / `repository`
- **BLANK 2:** `secret-identifier` / `token` / `pat` / `credential`

### Answer: `git-hub`, `secret-identifier`

**In `challenge-33.md`:** lines **293** and **296**.

`--ado-git` is the Azure Repos equivalent (line 321), taking the same four settings. `secret-identifier`
is a **Key Vault URI**, not the secret itself — the Dev Center resolves it with its managed identity.

---

## Q40

```bash
az devcenter [BLANK 1] environment create \
  --name "feature-auth-redesign" \
  --project-name proj-contoso-platform \
  --environment-type Dev \
  --catalog-name contoso-environments \
  --[BLANK 2] WebApp \
  --parameters '{"appServicePlanSku": "B1"}'
```

- **BLANK 1:** `dev` / `admin` / `env` / `user`
- **BLANK 2:** `environment-definition-name` / `template-name` / `definition` / `blueprint-name`

### Answer: `dev`, `environment-definition-name`

**In `challenge-33.md`:** lines **333** and **339**.

`admin` commands configure the platform; `dev` commands are what developers run. Spotting which half a
command belongs to is worth marks by itself.

---

## Q41

```yaml
variables:
  - name: environmentName
    value: "[BLANK 1]"
...
                URL=$(az devcenter dev environment show ... \
                  --query "[BLANK 2]" -o tsv)
                echo "##vso[task.setvariable variable=envUrl;[BLANK 3]]$URL"
```

- **BLANK 1:** `pr-$(System.PullRequest.PullRequestNumber)` / `$(Build.BuildId)` /
  `$(Build.SourceBranchName)` / `env-$(Build.BuildNumber)`
- **BLANK 2:** `outputs.webAppUrl.value` / `properties.url` / `resources[0].url` /
  `provisioningState`
- **BLANK 3:** `isOutput=true` / `isSecret=true` / `global=true` / `scope=stage`

### Answer: `pr-$(System.PullRequest.PullRequestNumber)`, `outputs.webAppUrl.value`, `isOutput=true`

**In `challenge-33.md`:** lines **472**, **539**, **540**.

The PR number is stable across pushes; `Build.BuildId` changes every run and would create a new
environment each time.

`outputs.webAppUrl.value` reads a **Bicep template output** surfaced by the environment.
`isOutput=true` plus a named task is the cross-job pattern from Challenge 20.

---

# Section G — Case study

## Case study: Contoso self-service platform

### Background

Contoso has **40 developers** on a microservices platform. Requesting a dev/test environment means an
IT ticket and a **3–5 business day** wait.

**Consequences:** developers share environments and break each other's configuration; long-lived
environments sit **idle 80% of the time**; developer environments differ from production; and
developers provision their own resources outside governance.

### Requirements

**Self-service**

- Developers must provision environments on demand, without a ticket
- Environments must match production's shape
- Each pull request should get its own isolated environment

**Governance**

- Developers must have no standing access to target subscriptions
- Dev, Test and Staging must deploy to **different** subscriptions
- Developers must not be able to choose oversized SKUs

**Cost**

- No more than 3 concurrent environments per developer
- Dev environments must be removed automatically after 7 days

---

## Q42

Which structure meets the "different subscriptions per environment type" requirement?

- A. Three project environment types, each with its own `--deployment-target-id`
- B. Three Dev Centers, one per subscription
- C. Three catalogs, one per environment type
- D. A single environment type with a subscription parameter

### Answer: A

**In `challenge-33.md`:** lines **88–115**.

```bash
--name Dev      --deployment-target-id "/subscriptions/{dev-sub-id}"
--name Test     --deployment-target-id "/subscriptions/{test-sub-id}"
--name Staging  --deployment-target-id "/subscriptions/{staging-sub-id}"
```

**Why the others fail**

- **B** — three Dev Centers means three catalogs, three sets of role assignments and three places to
  keep in step
- **C** — catalogs hold **definitions**; they have no concept of a target subscription
- **D** — a parameter is chosen by the **developer**, which would let anyone deploy into staging. The
  subscription must be fixed by the platform, not requested

---

## Q43

Which **two** meet the "no standing access" requirement? (Choose two.)

- A. Developers hold `Deployment Environments User` on the project
- B. The project environment type's identity holds Contributor on the target subscription
- C. Developers hold Contributor on the dev subscription
- D. Developers hold Reader on the target subscription
- E. A shared service principal credential distributed to developers

### Answer: A, B

**In `challenge-33.md`:** lines **61** and **694**.

**Together they are the inversion that makes the model work.** The powerful role sits on a **platform
identity** that only ever acts through approved environment definitions; the developer's role is
project-scoped and grants nothing directly in the subscription.

**Why the others fail**

- **C** — standing access, and the shadow-IT problem restated
- **D** — Reader is less dangerous and still standing access, and it does not enable anything they need
- **E** — a shared credential is unattributable and unrotatable

---

## Q44

Which control stops developers choosing oversized SKUs?

- A. The `allowed` list in `environment.yaml`
- B. Azure Policy on the target subscription
- C. `--max-dev-boxes-per-user`
- D. The `@allowed` decorator in `main.bicep`

### Answer: A

**In `challenge-33.md`:** lines **151–155**.

**Why the others fail — and the distinctions matter**

- **B** — a real and useful **backstop**, and it rejects the deployment *after* the developer has
  waited for it. The `allowed` list stops the choice being offered at all
- **C** — limits count, not size
- **D** — **the closest wrong answer.** `main.bicep` does have `@allowed` at line 178, and it would
  reject a bad value — but only at deployment time, and it does not populate the portal's dropdown.
  `environment.yaml` is the developer-facing contract; the Bicep decorator is the implementation's
  own guard. In practice you want both, and the question asks which control shapes the **choice**

---

## Q45

Which **two** meet the cost requirements? (Choose two.)

- A. `--max-dev-boxes-per-user 3` on the project
- B. A scheduled runbook deleting Dev environments older than 7 days
- C. An Azure Policy `deny` effect on expensive SKUs
- D. Azure Reservations for the dev subscription
- E. Deleting the Dev environment type weekly

### Answer: A, B

**In `challenge-33.md`:** lines **380** and **430–445**.

**Two dimensions: how many, and how long.** The requirement states both explicitly, and each needs its
own mechanism.

**Why the others fail**

- **C** — governs size, not count or lifetime, and the `allowed` list already covers it
- **D** — a billing commitment. Reserving capacity for idle environments makes the waste cheaper, not
  smaller
- **E** — would break every existing environment of that type

---

## Q46

How should each pull request get an isolated environment?

- A. A PR pipeline creating an ADE environment named after the PR number, with an existence check
- B. A shared `pr-testing` environment that every PR deploys into
- C. Developers manually creating an environment per PR
- D. Deploying the Bicep template directly from the PR pipeline

### Answer: A

**In `challenge-33.md`:** lines **463–511**.

**Why the others fail**

- **B** — shared environments causing conflicts is the original problem (line 23)
- **C** — manual steps get skipped, and nothing cleans up afterwards
- **D** — bypasses Deployment Environments, so you lose the governance, the tags the cleanup runbook
  keys on, and the guarantee that the PR environment matches the blessed definition

**And the consistency benefit:** the PR environment uses the **same definition** as everything else, so
it genuinely matches production's shape — the third requirement in the self-service group.

---

## Q47

Six months on, a developer reports that environments created from the `Microservice` definition fail,
while `WebApp` works. The catalog shows `syncState: Succeeded`.

What is the most likely cause?

- A. The `Microservice` definition's Bicep template has an error, surfacing at deployment time
- B. The catalog has not synced
- C. The developer lacks permissions
- D. The environment type is disabled

### Answer: A

**In `challenge-33.md`:** the two documented failure modes are catalog sync (line 616) and
authorisation (line 675). This is neither.

**Reason it through by elimination.** A sync failure would affect **every** definition and the state
says `Succeeded`. A permissions problem sits on the **project environment type**, so it would fail
`WebApp` too. A disabled type would likewise affect everything.

One definition failing while another succeeds points at **that definition's template** — a bad API
version, a missing parameter, a resource the target subscription has no quota for.

**Where to look next:** the environment's deployment error in the target subscription, since the
Bicep deployment runs there under the environment type's identity.

**The reasoning pattern:** *what is different between the thing that works and the thing that does
not?* Shared components are exonerated by the working case.

---

## Q48

Contoso's dev subscription cost has not fallen despite the 7-day cleanup runbook. Investigation shows
many resource groups older than 30 days, all tagged `ade-environment-type: Dev`, but with no
`auto-delete-after` tag.

What went wrong?

- A. The Azure Policy `modify` effect only tags resources on create or update, so pre-existing
  environments were never tagged
- B. The runbook's managed identity lacks permissions
- C. The tag name is misspelled in the runbook
- D. The 7-day threshold is too long

### Answer: A

**In `challenge-33.md`:** the policy at lines **384–425** and the runbook filter at lines **441–445**.

```powershell
$expiredGroups = Get-AzResourceGroup |
    Where-Object {
        $_.Tags['ade-environment-type'] -eq 'Dev' -and
        [DateTime]$_.Tags['auto-delete-after'] -lt $cutoffDate
    }
```

**The filter requires *both* tags.** Environments created **before** the policy was assigned have the
type tag but never received the expiry tag, so they fail the second condition and are silently skipped
forever.

**This is Challenge 32's remediation lesson in a different costume.** A `modify` effect acts on
**create and update** events; existing resources are flagged and left alone. The fix is a **remediation
task** to tag them retrospectively:

```bash
az policy remediation create \
  --policy-assignment "auto-expire-dev-environments" \
  --resource-discovery-mode ReEvaluateCompliance
```

**Why the others fail**

- **B** — a permissions failure would produce errors in the runbook output, and it would affect *all*
  deletions, not just untagged ones
- **C** — a misspelling would match nothing, so **no** environment would ever be deleted. Some are
  being deleted
- **D** — 30-day-old resource groups are well past a 7-day threshold. The threshold is not what is
  excluding them

**The general lesson, and it recurs:** a policy that tags going forward does not fix what already
exists. Whenever you introduce a tag-driven process, ask what happens to the resources that predate
it.

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Developer given subscription Contributor** | Q4, Q25, Q43 | Project-scoped role only. The platform identity deploys |
| **Wrong identity blamed for `AuthorizationFailed`** | Q4, Q19, Q26 | Requesting and deploying are two principals |
| **Catalog attached to the project** | Q18, Q27 | Catalogs belong to the Dev Center |
| **Subscription set on the Dev Center** | Q2, Q3, Q42 | The project environment type maps it |
| **Parameters expected in `main.bicep`** | Q5, Q20, Q44 | `environment.yaml` is the developer contract |
| **`values` or `options` instead of `allowed`** | Q38 | Three platforms, three keywords |
| **`Build.BuildId` as the environment name** | Q10, Q22, Q41 | The PR number is stable across pushes |
| **No existence check in the PR pipeline** | Q11, Q22, Q34 | The second push fails without it |
| **Fixed sleep instead of polling** | Q12 | Poll `provisioningState` until `Succeeded` |
| **Assuming built-in idle expiry** | Q14, Q30, Q45 | Policy tags, runbook deletes. Neither is automatic alone |
| **`modify` policy assumed to tag existing resources** | Q48 | It acts on create and update. Run a remediation task |
| **Catalog sync vs permissions confused** | Q8, Q29 | Read the error: "not found in catalog" vs `AuthorizationFailed` |

---

# The blocks to memorise

Line numbers are in `challenge-33.md`.

```bash
# 1. Platform setup, in order  (lines 43-62)
az devcenter admin devcenter create --name dc-contoso --identity-type SystemAssigned
az devcenter admin project create --name proj-contoso-platform --dev-center-name dc-contoso \
  --max-dev-boxes-per-user 3
az devcenter admin environment-type create --name Dev --dev-center-name dc-contoso
az devcenter admin project-environment-type create --name Dev \
  --project-name proj-contoso-platform \
  --deployment-target-id "/subscriptions/{dev-sub-id}" \
  --identity-type SystemAssigned --status Enabled
az role assignment create --role "Deployment Environments User" \
  --scope ".../projects/proj-contoso-platform"          # PROJECT scope, never subscription

# 2. The role the platform identity needs  (lines 687-697)
PRINCIPAL_ID=$(az devcenter admin project-environment-type show --query "identity.principalId" -o tsv)
az role assignment create --assignee-object-id "$PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal --role "Contributor" \
  --scope "/subscriptions/{dev-sub-id}"

# 3. Catalog from git  (lines 289-299)
az devcenter admin catalog create --dev-center-name dc-contoso \
  --git-hub path="/environments" branch="main" \
    uri="https://github.com/contoso/environment-catalog.git" \
    secret-identifier="https://kv-....vault.azure.net/secrets/github-pat"
az devcenter admin catalog sync ...
# Azure Repos alternative: --ado-git with the same four settings

# 4. Developer commands  (lines 333-361)
az devcenter dev environment create --environment-type Dev \
  --catalog-name contoso-environments --environment-definition-name WebApp \
  --parameters '{"appServicePlanSku": "B1"}'
az devcenter dev environment list / show --query "outputs" / delete --yes
```

```yaml
# 5. environment.yaml - the developer contract  (lines 139-170)
name: WebApp
version: 1.0.0
templatePath: main.bicep
parameters:
  - id: appServicePlanSku
    type: string
    default: B1
    allowed: [F1, B1, S1, P1v3]      # guardrail, not just UI

# 6. PR environment pipeline  (lines 471-540)
variables:
  - name: environmentName
    value: "pr-$(System.PullRequest.PullRequestNumber)"     # stable across pushes
# 1. check exists  2. create if absent  3. poll provisioningState
# 4. read outputs.webAppUrl.value  5. setvariable ... isOutput=true
```

```text
# 7. Ownership map - most wrong answers are ownership mistakes
Dev Center               catalogs, environment type NAMES, its own identity
Project                  per-user limits, developer role assignments
Project environment type target subscription, deploying identity, roles on the environment
Catalog                  environment definitions
environment.yaml         developer parameters, templatePath
main.bicep               the resources

# 8. Three identities
Developer                 requests             (Deployment Environments User, project scope)
Dev Center identity       reads catalog secret (Key Vault)
Project env type identity deploys              (Contributor, target subscription)
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 33 is exam-ready. Section 03e is complete |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 2 and 4, then retake this |
| Below 30 | Redo the challenge, writing the ownership map from memory first |

Record your result in `AZ-400-Learning-Log.md` under Challenge 33.

:::tip The one thing

**The developer requests; the platform deploys.**

Every design decision here follows from that split — the project-scoped role, the environment type's
managed identity, the `allowed` parameter list, and the catalog held as code. If a proposed answer
gives the developer subscription access, it has undone the product.

:::
