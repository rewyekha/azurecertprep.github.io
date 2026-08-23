---
sidebar_position: 1.5
toc_max_heading_level: 2
title: "Challenge 39: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 39 — AZ-400 exam questions

**48 questions** built only from what Challenge 39 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-39.md`**.

| Section | Shape | Questions |
|---|---|---|
| A | Single answer | 1–16 |
| B | Multiple answer (choose two / three) | 17–23 |
| C | Repeated scenario — "Does this meet the goal?" | 24–26 |
| D | Yes/No statement grid | 27–30 |
| E | Drag and drop | 31–35 |
| F | Hot area — complete the configuration | 36–41 |
| G | Case study | 42–48 |

The **trap index**, the **decision table**, and **scoring** are at the end.

:::danger This is the highest-value paper in the whole set

Service principal versus managed identity versus workload identity federation is the single
distinction that has cost you marks in every mock. Three facts decide almost every question:

**Stored secret?** SP yes, MI no, WIF no.
**Where can it run?** SP anywhere, MI **only on Azure-hosted compute**, WIF from a trusted OIDC issuer.
**What if it leaks?** SP usable anywhere until rotated; MI unusable off its resource; WIF valid only
for one repo, branch or environment.

:::

---

# Section A — Single answer

---

## Q1

A GitHub Actions workflow must authenticate to Azure with **no stored credential**.

Which method should you use?

- A. A service principal with a client secret
- B. Workload identity federation with OIDC
- C. A system-assigned managed identity on the runner
- D. A user-assigned managed identity

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-39.md`:** the decision table at lines **212–220**, with the implementation at
**103–122** and **140–156**.

```text
| Works from | Anywhere | Azure-hosted resources only | GitHub Actions, Azure Pipelines, external OIDC |
| Secret management | Requires storing and rotating | No secrets | No secrets |
```

**Why C and D both fail, and this is the fact to burn in:** a **GitHub-hosted runner is not an Azure
resource**. Managed identity — system-assigned or user-assigned — only works on compute that Azure
itself hosts: a VM, App Service, Container App, AKS pod, Automation account. A runner in GitHub's
fleet has no identity endpoint to call.

**Why A fails** — it stores exactly what the requirement forbids, and the scenario at line 18 is the
consequence: a shared secret in a plain-text pipeline variable, unrotated for 14 months.

**The phrase that selects WIF on the exam:** *"without storing a secret"*, *"secretless"*, *"eliminate
credential rotation"*.

</details>

---

## Q2

An **Azure VM** must read secrets from Key Vault with no stored credential.

Which method?

- A. Workload identity federation
- B. A managed identity assigned to the VM
- C. A service principal with a certificate
- D. A shared access signature

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-39.md`:** the decision table at line **219**.

```text
| Best for | Legacy systems, on-premises agents | Azure-hosted compute (VMs, App Service, AKS) | CI/CD pipelines |
```

**This is Q1 inverted, and the exam pairs them deliberately.** The VM **is** an Azure resource, so it
can carry an identity that Azure vouches for from the instance metadata endpoint. No token exchange
with an external issuer is needed.

**Why A is the trap here.** WIF is not "the modern answer to everything" — it exists to bring an
**external** workload's identity into Azure. Using it on a VM adds an OIDC issuer that is not needed
and cannot be simpler than the identity the VM already has.

**Why the others fail** — C stores a certificate (still a credential to manage and rotate), D is a
storage-specific token with no relationship to Key Vault.

</details>

---

## Q3

A workflow fails with `AADSTS70021: No matching federated identity record found.`

What is the cause?

- A. The `subject` claim does not match the token GitHub issued
- B. The service principal has no role assignment
- C. `id-token: write` is missing
- D. The client secret expired

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** Break scenario 1, lines **253–280**.

> *"The `subject` claim in the federated credential does not match the token issued by GitHub. Common
> causes include wrong repository name, wrong branch, or missing environment configuration."*

```bash
az ad app federated-credential list --id $OBJECT_ID --query "[].{name:name, subject:subject}"
```

**Read the error as a lookup failure.** Azure received a token, read its `sub` claim, and found no
federated credential registered with that exact string. Nothing about permissions is involved yet.

**The most common real cause:** the credential says `refs/heads/main` and the workflow ran on
`refs/heads/develop` (line 271). The fix is a **second credential**, not an edit — you register one
per subject.

**Why the others fail — and each has its own distinct error**

- **B** — a missing role gives `AuthorizationFailed` **after** a successful login (Break scenario 2)
- **C** — without `id-token: write` the workflow cannot request a token at all, and fails earlier with
  a token-request error
- **D** — federated credentials have no secret to expire. That is the point

</details>

---

## Q4

What is the correct federated credential **subject** for a GitHub Actions workflow running on the
`main` branch of `contoso/webapp`?

- A. `repo:contoso/webapp:ref:refs/heads/main`
- B. `repo:contoso/webapp:branch:main`
- C. `contoso/webapp:main`
- D. `repo:contoso/webapp:ref:main`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** line **108**.

```json
    "subject": "repo:contoso/webapp:ref:refs/heads/main",
```

**Memorise the shape.** `repo:<org>/<repo>:` then one of:

| Suffix | Matches |
|---|---|
| `ref:refs/heads/<branch>` | A branch (line 108) |
| `ref:refs/tags/<tag>` | A specific tag (line 242) |
| `environment:<name>` | A job declaring that environment (line 119) |
| `pull_request` | Any pull request (line 231) |

**Note `ref:refs/heads/`, not `branch:`** — it is the **full git ref**, the same value as
`github.ref`. That consistency is the memory hook: the subject contains what the workflow context
already reports.

</details>

---

## Q5

Which subject restricts authentication to jobs that declare `environment: production`?

- A. `repo:contoso/webapp:environment:production`
- B. `repo:contoso/webapp:ref:refs/heads/production`
- C. `repo:contoso/webapp:env:production`
- D. `environment:production`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** line **119**.

```json
    "subject": "repo:contoso/webapp:environment:production",
```

**Why this is the strongest form of scoping available.** The environment carries its own protection
rules — required reviewers, deployment branch policies (Challenge 24). Binding the credential to the
**environment** means a token is only issued for a job that has already passed those gates.

A branch subject says "code from this branch". An environment subject says "a deployment somebody
approved".

**Why the others fail** — B is a branch literally named `production`, C uses the wrong keyword,
D omits the repository so it identifies nothing.

</details>

---

## Q6

Which issuer URL does **Azure DevOps** use for workload identity federation?

- A. `https://vstoken.dev.azure.com/<org-id>`
- B. `https://token.actions.githubusercontent.com`
- C. `https://login.microsoftonline.com/<tenant-id>`
- D. `https://dev.azure.com/<org>`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** line **173**, contrasted with GitHub's issuer at line **107**.

```json
    "issuer": "https://vstoken.dev.azure.com/<org-id>",
    "subject": "sc://contoso-org/contoso-project/azure-production",
```

**Two platforms, two issuers, two subject formats:**

| | Issuer | Subject |
|---|---|---|
| GitHub Actions | `https://token.actions.githubusercontent.com` | `repo:org/repo:ref:refs/heads/main` |
| Azure DevOps | `https://vstoken.dev.azure.com/<org-id>` | `sc://org/project/connection-name` |

**The `sc://` prefix means service connection**, and the subject names the connection rather than a
branch — because in Azure DevOps the **service connection** is the security boundary, protected by its
own pipeline permissions and approvals.

**Why the others fail** — B is GitHub's, C is the Entra token endpoint (not an issuer for this),
D is the organisation URL.

</details>

---

## Q7

Which permission must a GitHub Actions job declare to request an OIDC token?

- A. `id-token: write`
- B. `contents: write`
- C. `actions: write`
- D. `deployments: write`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** lines **140–142**.

```yaml
permissions:
  id-token: write
  contents: read
```

**Forgetting this is the number-one OIDC failure**, and the symptom does not mention permissions —
`azure/login` fails saying it could not get an ID token, which sends people to check the federated
credential instead.

**Note it is `write` even though you are only requesting a token.** The scope name refers to the
ability to have a token **minted**, not to modify anything.

**And `permissions:` is restrictive:** the moment you declare the block, everything not listed is set
to `none`. That is why `contents: read` appears alongside it — without it, checkout fails.

</details>

---

## Q8

Which **three** values does `azure/login@v2` need for OIDC?

- A. `client-id`, `tenant-id`, `subscription-id`
- B. `client-id`, `client-secret`, `tenant-id`
- C. `creds` JSON only
- D. `username`, `password`, `tenant`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** lines **151–156**.

```yaml
      - name: Azure Login with OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

**None of these three is a secret.** They are identifiers — you could print them in a log without
harm. Storing them as secrets is convention and tidiness, not necessity.

**The absence of `client-secret` is the whole point**, and it is the fastest way to tell an OIDC login
from a legacy one at a glance.

**Why the others fail** — B and D supply a password; C is the `AZURE_CREDENTIALS` service principal
JSON, the pattern being migrated away from.

</details>

---

## Q9

An application using a managed identity gets `AuthorizationFailed` accessing a storage account.

What is missing?

- A. An RBAC role assignment for the identity at the right scope
- B. A federated credential
- C. `id-token: write`
- D. A client secret

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** Break scenario 2, lines **285–307**.

```bash
az role assignment create \
  --assignee-object-id $IDENTITY_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/<subscription-id>/resourceGroups/rg-contoso-challenge39"
```

**Creating an identity grants nothing.** Authentication and authorisation are separate steps, and this
error means the first succeeded — Azure knows who the caller is and has no record of them being
allowed to do this.

**Note the role: `Storage Blob Data Contributor`, not `Contributor`.** That is the control-plane versus
data-plane split again — `Contributor` can delete the storage account and cannot read a blob inside it.
The same distinction as Challenge 27's App Configuration Data Reader and Challenge 29's `db_ddladmin`.

**The two errors to keep apart:**

| Error | Meaning |
|---|---|
| `AADSTS70021` | Authentication — no federated credential matches |
| `AuthorizationFailed` | Authorisation — authenticated, but no role |

</details>

---

## Q10

Why does `az role assignment create` for a managed identity use `--assignee-object-id` with
`--assignee-principal-type ServicePrincipal`?

- A. To avoid a Graph lookup and the replication delay that can cause a "principal not found" error
- B. Because managed identities have no client ID
- C. Because object IDs are shorter
- D. It is required for all role assignments

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** lines **81–85**.

```bash
az role assignment create \
  --assignee-object-id $IDENTITY_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --role "Contributor" \
  --scope "..."
```

**`--assignee` alone triggers a Microsoft Graph lookup** to resolve whatever you passed. Immediately
after creating an identity, that lookup can fail because directory replication has not caught up — the
classic "PrincipalNotFound" that succeeds when you retry a minute later.

Passing the object ID and the type skips the lookup entirely, which makes scripts deterministic.

**Why the others fail** — B is false (line 75 fetches `clientId`), C is irrelevant, D is false since
`--assignee` works for interactive use.

</details>

---

## Q11

A managed identity has both a **principal ID** and a **client ID**. What is each used for?

- A. Principal ID for role assignments; client ID for the application to authenticate as
- B. Both are interchangeable
- C. Principal ID for authentication; client ID for billing
- D. Client ID for role assignments; principal ID for the resource

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** lines **70–78**.

```bash
IDENTITY_PRINCIPAL_ID=$(az identity show ... --query principalId -o tsv)
IDENTITY_CLIENT_ID=$(az identity show ... --query clientId -o tsv)
```

| ID | Also called | Used for |
|---|---|---|
| **Principal ID** | Object ID | **RBAC role assignments** (line 82) |
| **Client ID** | Application ID | The **application** specifying which identity to use |

**The client ID matters for user-assigned identities specifically.** A resource can carry several, so
code must say which one — `new DefaultAzureCredential(new() { ManagedIdentityClientId = "..." })`.
A system-assigned identity needs no such hint, because there is only one.

**Mixing them up gives a confusing failure:** using the client ID in a role assignment either errors or
silently assigns to the wrong object.

</details>

---

## Q12

Contoso needs one identity shared by several Azure resources, surviving the deletion of any one of
them.

Which should they use?

- A. A user-assigned managed identity
- B. A system-assigned managed identity
- C. A service principal with a secret
- D. Workload identity federation

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** lines **60–66**.

> *"Create a managed identity that can be shared across multiple Azure resources without managing
> secrets."*

**The lifecycle difference is the exam question:**

| | System-assigned | User-assigned |
|---|---|---|
| Lifecycle | **Tied to one resource** — deleted with it | **Independent** resource |
| Sharing | One resource only | **Many resources** |
| Role assignments | Recreated if the resource is recreated | **Survive** |

**Why that last row matters operationally:** with system-assigned, redeploying a VM creates a **new**
principal, and every role assignment must be recreated. With user-assigned, the identity and its roles
outlive the compute — which is what makes it right for infrastructure that is rebuilt regularly.

**Why the others fail** — B is per-resource, C stores a secret, D is for external workloads.

</details>

---

## Q13

Which statement about federated credential subjects is correct?

- A. Wildcards are not supported — each tag needs its own credential
- B. `refs/tags/v*` matches all v-prefixed tags
- C. One credential covers all branches automatically
- D. Subjects are case-insensitive and flexible

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** lines **236–243**.

```json
    "subject": "repo:contoso/webapp:ref:refs/tags/v1.0.0",
    "description": "GitHub Actions release tag v1.0.0 (one credential per tag needed - wildcards not supported)"
```

**The subject is matched as an exact string**, which is a real operational constraint: a
tag-triggered release workflow would need a new federated credential for every release.

**The practical answer is to scope by environment instead** (line 119). One
`repo:contoso/webapp:environment:production` credential covers every release that deploys to
production, whatever tag triggered it — and it inherits the environment's approval gate.

**Note:** Entra has since added limited wildcard support in preview for some subject patterns. The
challenge — and the exam — treat subjects as exact matches, which is the safe answer.

</details>

---

## Q14

Which subject allows pull request workflows to authenticate?

- A. `repo:contoso/webapp:pull_request`
- B. `repo:contoso/webapp:ref:refs/pull/*`
- C. `repo:contoso/webapp:pr`
- D. `repo:contoso/webapp:environment:pr`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** lines **229–231**.

```json
    "name": "github-pull-request",
    "subject": "repo:contoso/webapp:pull_request",
```

**Think hard before creating this one.** A pull request can come from **anyone** who can open one,
including a fork. A `pull_request` credential means code you have not reviewed can obtain an Azure
token with whatever roles that app registration holds.

**If you need it**, give the app registration a **separate, minimal** role — read-only, or scoped to a
throwaway resource group — rather than the same Contributor role your deployments use.

**Why the others fail** — B assumes wildcards, C and D use invalid keywords.

</details>

---

## Q15

Which audience value do Azure federated credentials use?

- A. `api://AzureADTokenExchange`
- B. `https://management.azure.com`
- C. `https://token.actions.githubusercontent.com`
- D. `azure-cli`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** lines **110**, **121**, **176**, **233**, **244** — **every** federated
credential in the challenge.

```json
    "audiences": ["api://AzureADTokenExchange"]
```

**It is constant.** The issuer changes per platform and the subject changes per scope; the audience is
always this value for Azure workload identity federation.

**What it means:** the audience claim tells the issuer *who the token is for*. GitHub mints a token
addressed to Azure's token-exchange endpoint, so a token intended for Azure cannot be replayed against
some other service that trusts the same issuer.

**Why the others fail** — B is the ARM resource identifier, C is the **issuer**, D is a client ID.

</details>

---

## Q16

A service principal secret was created with `--years 1`. What happens after a year?

- A. Authentication fails until the secret is rotated
- B. It renews automatically
- C. The service principal is deleted
- D. Role assignments are removed

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** line **42**, with the decision table's rotation row at line **215**.

```bash
az ad sp create-for-rbac --name "sp-contoso-pipeline-dev" --years 1
```

```text
| Rotation needed | Yes (expiry 1-2 years) | No | No (token exchange) |
```

**The scenario shows what actually happens** (line 18): *"secrets are rotated manually (last rotation
was 14 months ago)"*. Either the secret had a longer expiry, or pipelines are already failing and
nobody has connected the two.

**Expiry is a reliability problem as much as a security one.** A credential that expires at an
unpredictable moment breaks deployments during an incident, and the failure — an authentication error
— rarely says "your secret expired" clearly.

**Neither MI nor WIF has this failure mode**, which is the strongest practical argument for migrating.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** are true about workload identity federation? (Choose three.)

- A. No secret is stored or rotated
- B. Tokens are valid only for a specific repo, branch or environment
- C. It works with GitHub Actions and Azure Pipelines
- D. It works on any Azure VM without configuration
- E. It requires a client secret with a 1-year expiry
- F. It replaces RBAC role assignments

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-39.md`:** the decision table at lines **214–220**.

```text
| Secret management   | ... | ... | No secrets to manage |
| Risk if compromised | ... | ... | Token valid only for specific repo/branch/environment |
| Works from          | ... | ... | GitHub Actions, Azure Pipelines, external OIDC providers |
```

**Why the others fail**

- **D** — that describes **managed identity**. WIF needs a trusted external issuer, and a VM has no
  OIDC issuer of its own
- **E** — the opposite; the absence of a secret is the feature
- **F** — **the important one.** WIF is **authentication only**. The app registration still needs an
  RBAC role assignment (line 125), and forgetting it produces `AuthorizationFailed` after a
  perfectly successful login

</details>

---

## Q18

Which **two** describe managed identity's limitations? (Choose two.)

- A. It only works from Azure-hosted resources
- B. A system-assigned identity dies with its resource
- C. It requires secret rotation
- D. It cannot be granted RBAC roles
- E. It cannot be used with Key Vault

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-39.md`:** line **216** and lines **60–66**.

**A is the boundary that decides Q1 and Q2**, and the exam tests it in both directions: a GitHub-hosted
runner cannot use MI; an Azure VM should.

**B is why user-assigned exists.** Redeploy the VM and the system-assigned principal is gone, taking
every role assignment with it. A user-assigned identity is its own resource and survives.

**Why the others fail** — C is what MI **eliminates**, D and E are exactly what it is for.

</details>

---

## Q19

Which **three** components does workload identity federation for GitHub Actions require? (Choose
three.)

- A. An app registration with a service principal
- B. A federated credential with issuer, subject and audience
- C. An RBAC role assignment for the service principal
- D. A client secret
- E. A managed identity on the runner
- F. A stored `AZURE_CREDENTIALS` JSON

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-39.md`:** lines **94–100**, **103–111**, **125–128**.

```bash
az ad app create --display-name "sp-contoso-github-oidc"
az ad sp create --id $APP_ID                    # A - the SP for the app
az ad app federated-credential create ...       # B - trust
az role assignment create --assignee $APP_ID \  # C - permission
  --role "Contributor" --scope "..."
```

**Three objects, three jobs — and the exam breaks one at a time:**

| Missing | Symptom |
|---|---|
| App registration or SP | Nothing to authenticate as |
| Federated credential | `AADSTS70021` |
| Role assignment | `AuthorizationFailed` |

**Note `az ad app create` and `az ad sp create` are separate steps.** The app registration is the
identity definition; the service principal is its instance in your tenant, and it is the SP that
receives role assignments.

**Why D, E and F fail** — all three are the patterns WIF replaces.

</details>

---

## Q20

Which **two** subject formats are valid for GitHub Actions federated credentials? (Choose two.)

- A. `repo:contoso/webapp:ref:refs/heads/main`
- B. `repo:contoso/webapp:environment:production`
- C. `repo:contoso/webapp:branch:main`
- D. `github:contoso/webapp:main`
- E. `repo:contoso/webapp:*`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-39.md`:** lines **108** and **119**.

**Why the others fail**

- **C** — `branch:` is not a keyword. It is `ref:refs/heads/<branch>` — the **full git ref**
- **D** — the prefix is `repo:`, not `github:`
- **E** — no wildcards (line 243)

**The memory hook:** the subject mirrors what the workflow already knows about itself. `github.ref`
gives `refs/heads/main`; the subject embeds that exact string.

</details>

---

## Q21

Which **two** problems in Contoso's scenario does workload identity federation solve? (Choose two.)

- A. A shared service principal secret stored as a plain-text pipeline variable
- B. Secrets rotated manually, last rotated 14 months ago
- C. Three teams with Contributor on the entire production subscription
- D. Forty microservices deployed across two platforms
- E. Pipelines running on self-hosted agents

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-39.md`:** the scenario at line **18**, with the decision table's secret and rotation
rows at **214–215**.

**Why C is the important near-miss.** Over-broad scope is an **RBAC** problem, not an authentication
one. Migrating those teams to WIF without changing the scope gives you three secretless identities
that are still Contributor over all of production.

**Both must be fixed**, and the challenge does both: WIF removes the secrets (lines 103–122) while
`--scopes` on a resource group (line 41) fixes the scope. Line 18 names them as two separate audit
findings for exactly this reason.

**Why D and E fail** — D is scale, E is where agents run.

</details>

---

## Q22

Which **two** distinguish system-assigned from user-assigned managed identity? (Choose two.)

- A. System-assigned is tied to a single resource's lifecycle
- B. User-assigned can be shared by multiple resources
- C. System-assigned can be shared by multiple resources
- D. User-assigned requires a client secret
- E. System-assigned survives resource deletion

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-39.md`:** lines **60–66**.

**The practical consequence to be able to state:** with system-assigned, tearing down and recreating a
VM produces a **new principal**, so every role assignment must be recreated. With user-assigned, the
identity is a separate resource — roles are assigned once and survive every redeployment.

That makes user-assigned the right default for IaC-managed infrastructure, and system-assigned the
simpler choice for a long-lived resource with one purpose.

**Why the others fail** — C and E invert A, D is false for both kinds.

</details>

---

## Q23

Which **two** errors distinguish an authentication failure from an authorisation failure? (Choose
two.)

- A. `AADSTS70021: No matching federated identity record found` — authentication
- B. `AuthorizationFailed` — authorisation
- C. `AADSTS70021` — authorisation
- D. `AuthorizationFailed` — expired secret
- E. Both mean the role assignment is missing

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-39.md`:** lines **255** and **287**.

**Reading the error correctly saves half the diagnosis:**

| Error | Stage | Fix |
|---|---|---|
| `AADSTS70021` | Azure could not find a credential matching the token's subject | Add or correct the **federated credential** |
| `AuthorizationFailed` | Identity confirmed, action not permitted | Add the **RBAC role assignment** |

**`AuthorizationFailed` is the more encouraging of the two** — it means the entire secretless
authentication chain worked. Only the permission is missing.

**Why the others fail** — C inverts the meaning, D describes a different symptom, E is true for only
one of them.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must let a GitHub Actions workflow deploy to a production resource group with no
stored credential, restricted so that only deployments to the `production` environment can obtain a
token, and with permissions limited to that resource group.

---

## Q24

**Proposed solution:** Create an app registration and service principal, add a federated credential
with issuer `https://token.actions.githubusercontent.com` and subject
`repo:contoso/webapp:environment:production`, assign Contributor scoped to the resource group, and run
the workflow with `permissions: id-token: write` and a job declaring `environment: production`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-39.md`:** lines **94–128** and **140–156**.

| Requirement | Mechanism |
|---|---|
| No stored credential | Federated credential — no secret exists |
| Only the production environment | Subject `environment:production` |
| Scoped permissions | Role assignment at **resource group** scope |
| Workflow can request a token | `id-token: write` + the job declaring the environment |

**The last row is not decoration.** The token's subject is derived from the job's context, so a job
that does **not** declare `environment: production` produces a different subject and gets
`AADSTS70021` — the restriction is enforced by Azure, not by convention.

</details>

---

## Q25

**Proposed solution:** Create a service principal with `az ad sp create-for-rbac --years 2`, store the
secret as a repository secret, assign Contributor at the subscription scope, and use
`azure/login@v2` with `creds`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**All three requirements fail, and this is the scenario's starting state** (line 18).

**A stored credential.** A repository secret is better than a plain-text pipeline variable and it is
still a secret that must be rotated — in two years, by someone who remembers.

**No environment restriction.** A repository secret is available to **any** workflow in the repository
that references it. A pull request workflow, a scheduled job, anything.

**Over-broad scope.** Subscription-level Contributor is the exact audit finding at line 18 — three
teams already did this.

**It works**, which is why it persists. Every requirement is about what happens when it goes wrong.

</details>

---

## Q26

**Proposed solution:** Create an app registration and service principal, add a federated credential
with the correct issuer and subject `repo:contoso/webapp:environment:production`, run the workflow with
`permissions: id-token: write` and `environment: production` — but do not assign any RBAC role.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Authentication succeeds and nothing can be deployed.**

`azure/login` completes — the subject matches, Azure issues a token, the step goes green. The next step
fails with `AuthorizationFailed`.

**The confusion this causes in practice:** the login step passing makes people assume identity is
configured correctly, so they look at the deployment task, the resource, the region — anything except
the missing role.

```bash
az role assignment create \
  --assignee $APP_ID \
  --role "Contributor" \
  --scope "/subscriptions/<subscription-id>/resourceGroups/rg-contoso-challenge39"
```

**Federation is authentication. RBAC is authorisation.** Two configurations, two failure modes, two
different error messages — and this is the pairing the exam tests most often in Domain 4.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — the three methods

| # | Statement | Answer |
|---|---|---|
| 1 | Managed identity works from a GitHub-hosted runner |  |
| 2 | Workload identity federation stores no secret |  |
| 3 | A service principal secret requires rotation |  |
| 4 | Managed identity requires rotation |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Managed identity works from a GitHub-hosted runner | **No** |
| 2 | Workload identity federation stores no secret | **Yes** |
| 3 | A service principal secret requires rotation | **Yes** |
| 4 | Managed identity requires rotation | **No** |

**In `challenge-39.md`:** the decision table at lines **214–216**.

**Row 1 is the single most testable fact in this challenge.** A GitHub-hosted runner is not an Azure
resource; there is no instance metadata endpoint to ask. On a **self-hosted** runner *in an Azure VM*,
managed identity does work — which is the nuance the exam sometimes uses to make the wrong answer look
right.

Rows 3 and 4 together are the operational argument: one method has an expiry date that will break a
deployment at an unpredictable moment, and two do not.

</details>

---

## Q28 — federated credentials

| # | Statement | Answer |
|---|---|---|
| 1 | The subject must match the token's claim exactly |  |
| 2 | Wildcards can match multiple tags |  |
| 3 | The audience is always `api://AzureADTokenExchange` |  |
| 4 | One credential covers every branch in the repository |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | The subject must match the token's claim exactly | **Yes** |
| 2 | Wildcards can match multiple tags | **No** |
| 3 | The audience is always `api://AzureADTokenExchange` | **Yes** |
| 4 | One credential covers every branch in the repository | **No** |

**In `challenge-39.md`:** lines **257**, **243**, **110**, **271–280**.

Row 4 explains the fix in Break scenario 1: a workflow on `develop` needs its **own** credential
(line 276), not an edit to the `main` one. You register one per subject, and an app registration can
hold several.

Row 3 is the constant amid two variables — issuer changes per platform, subject changes per scope,
audience never changes.

</details>

---

## Q29 — managed identity

| # | Statement | Answer |
|---|---|---|
| 1 | Creating an identity grants it permissions |  |
| 2 | Principal ID is used for role assignments |  |
| 3 | Client ID tells an application which identity to use |  |
| 4 | A user-assigned identity survives deletion of a resource using it |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Creating an identity grants it permissions | **No** |
| 2 | Principal ID is used for role assignments | **Yes** |
| 3 | Client ID tells an application which identity to use | **Yes** |
| 4 | A user-assigned identity survives deletion of a resource using it | **Yes** |

**In `challenge-39.md`:** lines **285–307**, **70–85**, **60–66**.

Row 1 is Break scenario 2, and it is the same shape as Challenge 33's project environment type and
Challenge 32's policy assignment: **an identity exists; permissions are a separate grant.** Three
services, one lesson.

Row 3 matters when a resource carries several user-assigned identities — the code must name one.

</details>

---

## Q30 — errors and diagnosis

| # | Statement | Answer |
|---|---|---|
| 1 | `AADSTS70021` means the federated credential subject did not match |  |
| 2 | `AuthorizationFailed` means authentication succeeded |  |
| 3 | A missing `id-token: write` produces `AADSTS70021` |  |
| 4 | `az ad app federated-credential list` shows configured subjects |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `AADSTS70021` means the federated credential subject did not match | **Yes** |
| 2 | `AuthorizationFailed` means authentication succeeded | **Yes** |
| 3 | A missing `id-token: write` produces `AADSTS70021` | **No** |
| 4 | `az ad app federated-credential list` shows configured subjects | **Yes** |

**In `challenge-39.md`:** lines **255**, **287**, **140–141**, **263**.

**Row 3 is a genuinely useful distinction.** Without `id-token: write` the workflow never obtains a
token, so `azure/login` fails at the **request** stage with a token-request error — before Azure is
ever asked to match a subject. Three failures, three different messages, three different fixes.

Row 4 is the first diagnostic command to reach for: list the subjects and compare them against what
the workflow actually is.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each requirement to the correct authentication method.

| Requirement | Method |
|---|---|
| GitHub Actions deploying to Azure, no stored secret |  |
| An Azure VM reading from Key Vault |  |
| An on-premises build agent outside Azure |  |
| One identity shared by several Azure resources |  |
| Azure Pipelines service connection, no secret |  |
| A legacy script that cannot use OIDC |  |

**Options:** Managed identity · Service principal + secret · User-assigned managed identity · Workload identity federation

<details>
<summary>Show answer</summary>

| Requirement | Method |
|---|---|
| GitHub Actions deploying to Azure, no stored secret | **Workload identity federation** |
| An Azure VM reading from Key Vault | **Managed identity** |
| An on-premises build agent outside Azure | **Service principal + secret** |
| One identity shared by several Azure resources | **User-assigned managed identity** |
| Azure Pipelines service connection, no secret | **Workload identity federation** |
| A legacy script that cannot use OIDC | **Service principal + secret** |

**In `challenge-39.md`:** the decision table at lines **212–220**.

**The deciding question is always the same: where does the workload run?**

- Inside Azure → **managed identity**
- On a platform with a trusted OIDC issuer (GitHub, Azure DevOps) → **WIF**
- Anywhere else → **service principal**, accepting the secret and its rotation

</details>

---

## Q32

Match each federated credential subject to what it authorises.

| Subject | Authorises |
|---|---|
| `repo:contoso/webapp:ref:refs/heads/main` |  |
| `repo:contoso/webapp:ref:refs/tags/v1.0.0` |  |
| `repo:contoso/webapp:environment:production` |  |
| `repo:contoso/webapp:pull_request` |  |
| `sc://contoso-org/contoso-project/azure-production` |  |

**Options:** An **Azure DevOps service connection** · **Any pull request** — including forks · Jobs declaring the **production environment** · That **one specific tag** · Workflows on the **main branch**

<details>
<summary>Show answer</summary>

| Subject | Authorises |
|---|---|
| `repo:contoso/webapp:ref:refs/heads/main` | Workflows on the **main branch** |
| `repo:contoso/webapp:ref:refs/tags/v1.0.0` | That **one specific tag** |
| `repo:contoso/webapp:environment:production` | Jobs declaring the **production environment** |
| `repo:contoso/webapp:pull_request` | **Any pull request** — including forks |
| `sc://contoso-org/contoso-project/azure-production` | An **Azure DevOps service connection** |

**In `challenge-39.md`:** lines **108**, **242**, **119**, **231**, **174**.

**Ordered by blast radius, narrowest first:** the tag, then the environment (gated by approvals), then
the branch, then `pull_request` — which anyone who can open a PR can trigger.

**The service connection subject is the Azure DevOps equivalent of the environment one:** it names the
protected object rather than the code.

</details>

---

## Q33

Arrange the steps to configure workload identity federation for GitHub Actions.

**Items:** Assign an RBAC role to the service principal · Create the app registration · Add the
federated credential · Create the service principal · Add `id-token: write` to the workflow

<details>
<summary>Show answer</summary>

### Answer

1. Create the app registration — line **94**
2. Create the service principal — line **100**
3. Add the federated credential — line **103**
4. Assign an RBAC role to the service principal — line **125**
5. Add `id-token: write` to the workflow — line **141**

```bash
az ad app create --display-name "sp-contoso-github-oidc"
az ad sp create --id $APP_ID
az ad app federated-credential create --id $OBJECT_ID --parameters '{ ... }'
az role assignment create --assignee $APP_ID --role "Contributor" --scope "..."
```

**Steps 3 and 4 are the pair people conflate.** Step 3 is **who may authenticate**; step 4 is **what
they may do**. Skip 3 and you get `AADSTS70021`; skip 4 and you get `AuthorizationFailed`.

**Note the two different IDs:** the federated credential uses `$OBJECT_ID` (the app registration's
object), the role assignment uses `$APP_ID` (the application ID).

</details>

---

## Q34

Match each error to its cause and fix.

| Error | Cause | Fix |
|---|---|---|
| `AADSTS70021` |  |  |
| `AuthorizationFailed` |  |  |
| Token request failed |  |  |
| Authentication fails after ~1 year |  |  |

<details>
<summary>Show answer</summary>

| Error | Cause | Fix |
|---|---|---|
| `AADSTS70021` | Subject does not match the token | **Add or correct the federated credential** |
| `AuthorizationFailed` | No RBAC role at the scope | **Create the role assignment** |
| Token request failed | `id-token: write` missing | **Add the permission** |
| Authentication fails after ~1 year | Client secret expired | **Rotate — or migrate to WIF** |

**In `challenge-39.md`:** lines **255**, **287**, **141**, **215**.

**Diagnose by stage.** Could it get a token? Did Azure recognise it? Was it allowed to act? Each
question has one failure mode and one fix, and the error message tells you which.

</details>

---

## Q35

Match each risk to the method that mitigates it.

| Risk | Mitigation |
|---|---|
| A leaked credential used from anywhere |  |
| A credential expiring and breaking deployments |  |
| An identity disappearing when a VM is rebuilt |  |
| Over-broad access after a compromise |  |
| A shared secret visible to all contributors |  |

**Options:** MI or WIF — no expiry · RBAC scoped to a resource group · User-assigned managed identity · WIF — no secret exists · WIF — token bound to repo, branch or environment

<details>
<summary>Show answer</summary>

| Risk | Mitigation |
|---|---|
| A leaked credential used from anywhere | **WIF — token bound to repo, branch or environment** |
| A credential expiring and breaking deployments | **MI or WIF — no expiry** |
| An identity disappearing when a VM is rebuilt | **User-assigned managed identity** |
| Over-broad access after a compromise | **RBAC scoped to a resource group** |
| A shared secret visible to all contributors | **WIF — no secret exists** |

**In `challenge-39.md`:** lines **220**, **215**, **60–66**, **41**, **18**.

**The last two rows are the audit findings from the scenario**, and they need different fixes: WIF
removes the secret, and scoping fixes the permissions. Doing only one leaves half the finding open —
the trap in Q21.

</details>

---

# Section F — Hot area

---

## Q36

```bash
az ad app federated-credential create \
  --id $OBJECT_ID \
  --parameters '{
    "name": "github-main-branch",
    "issuer": "[BLANK 1]",
    "subject": "[BLANK 2]",
    "audiences": ["[BLANK 3]"]
  }'
```

Requirement: allow GitHub Actions workflows on the `main` branch of `contoso/webapp`.

- **BLANK 1:** `https://token.actions.githubusercontent.com` /
  `https://vstoken.dev.azure.com/<org-id>` / `https://github.com/contoso` /
  `https://login.microsoftonline.com`
- **BLANK 2:** `repo:contoso/webapp:ref:refs/heads/main` / `repo:contoso/webapp:branch:main` /
  `contoso/webapp:main` / `repo:contoso/webapp:*`
- **BLANK 3:** `api://AzureADTokenExchange` / `https://management.azure.com` / `azure-cli` /
  `https://token.actions.githubusercontent.com`

<details>
<summary>Show answer</summary>

### Answer: `https://token.actions.githubusercontent.com`, `repo:contoso/webapp:ref:refs/heads/main`,
`api://AzureADTokenExchange`

**In `challenge-39.md`:** lines **107–110**.

**Three fields, three rules:** issuer identifies the **platform**, subject identifies the **exact
workload**, audience is **always** `api://AzureADTokenExchange`.

</details>

---

## Q37

```yaml
permissions:
  [BLANK 1]: write
  contents: read

jobs:
  deploy:
    environment: [BLANK 2]
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          [BLANK 3]: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

- **BLANK 1:** `id-token` / `contents` / `packages` / `deployments`
- **BLANK 2:** `production` / `staging` / `default` / *(omit)*
- **BLANK 3:** `subscription-id` / `client-secret` / `creds` / `resource-group`

<details>
<summary>Show answer</summary>

### Answer: `id-token`, `production`, `subscription-id`

**In `challenge-39.md`:** lines **141**, **147**, **156**.

**BLANK 2 is not cosmetic.** If the federated credential's subject is
`repo:contoso/webapp:environment:production`, omitting the environment produces a different subject
and the login fails with `AADSTS70021`. The credential and the workflow must agree.

**The absence of `client-secret` is the tell** that this is OIDC.

</details>

---

## Q38

```bash
az role assignment create \
  --[BLANK 1] $IDENTITY_PRINCIPAL_ID \
  --assignee-principal-type [BLANK 2] \
  --role "[BLANK 3]" \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-contoso-challenge39"
```

Requirement: let a managed identity read and write blobs, with least privilege.

- **BLANK 1:** `assignee-object-id` / `assignee` / `assignee-client-id` / `principal`
- **BLANK 2:** `ServicePrincipal` / `User` / `Group` / `ManagedIdentity`
- **BLANK 3:** `Storage Blob Data Contributor` / `Contributor` / `Owner` / `Reader`

<details>
<summary>Show answer</summary>

### Answer: `assignee-object-id`, `ServicePrincipal`, `Storage Blob Data Contributor`

**In `challenge-39.md`:** lines **303–307**.

`--assignee-object-id` with the type avoids the Graph lookup and its replication delay.
**`ServicePrincipal` is correct for a managed identity** — there is no `ManagedIdentity` principal
type; managed identities are service principals in the directory.

And `Contributor` is control-plane: it can delete the storage account and **cannot read a blob**.

</details>

---

## Q39

```bash
# Azure DevOps federated credential
az ad app federated-credential create --id $OBJECT_ID --parameters '{
    "issuer": "[BLANK 1]",
    "subject": "[BLANK 2]",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

- **BLANK 1:** `https://vstoken.dev.azure.com/<org-id>` /
  `https://token.actions.githubusercontent.com` / `https://dev.azure.com/contoso` /
  `https://app.vssps.visualstudio.com`
- **BLANK 2:** `sc://contoso-org/contoso-project/azure-production` /
  `repo:contoso/webapp:ref:refs/heads/main` / `pipeline:contoso-project` /
  `contoso-org/azure-production`

<details>
<summary>Show answer</summary>

### Answer: `https://vstoken.dev.azure.com/<org-id>`, `sc://contoso-org/contoso-project/azure-production`

**In `challenge-39.md`:** lines **173–174**.

`sc://` = **service connection**, then organisation, project and connection name. Azure DevOps binds
the credential to the **connection** rather than to a branch, because the connection is the object that
carries pipeline permissions and approvals.

</details>

---

## Q40

```bash
az identity create --name id-contoso-pipeline --resource-group rg-contoso-challenge39

IDENTITY_[BLANK 1]=$(az identity show ... --query [BLANK 2] -o tsv)   # for role assignments
IDENTITY_[BLANK 3]=$(az identity show ... --query clientId -o tsv)    # for the application
```

- **BLANK 1:** `PRINCIPAL_ID` / `CLIENT_ID` / `TENANT_ID` / `OBJECT_NAME`
- **BLANK 2:** `principalId` / `clientId` / `id` / `tenantId`
- **BLANK 3:** `CLIENT_ID` / `PRINCIPAL_ID` / `APP_ID` / `SECRET`

<details>
<summary>Show answer</summary>

### Answer: `PRINCIPAL_ID`, `principalId`, `CLIENT_ID`

**In `challenge-39.md`:** lines **70–78**.

**Principal ID (object ID) → RBAC. Client ID (application ID) → the application says which identity to
use.** Swapping them either errors or silently assigns a role to the wrong object.

</details>

---

## Q41

```bash
# Legacy approach being migrated away from
az ad sp create-for-rbac \
  --name "sp-contoso-pipeline-dev" \
  --role "[BLANK 1]" \
  --scopes "[BLANK 2]" \
  --years [BLANK 3]
```

Requirement: least privilege, and the shortest practical credential lifetime.

- **BLANK 1:** `Contributor` / `Owner` / `Reader` / `User Access Administrator`
- **BLANK 2:** `/subscriptions/<id>/resourceGroups/rg-contoso-challenge39` / `/subscriptions/<id>` /
  `/` / `/providers/Microsoft.Management/managementGroups/contoso`
- **BLANK 3:** `1` / `2` / `5` / `99`

<details>
<summary>Show answer</summary>

### Answer: `Contributor`, the **resource group** scope, `1`

**In `challenge-39.md`:** lines **40–42**.

**The scope is the point.** Subscription-level Contributor is the audit finding at line 18 — three
teams did exactly that. Scoping to a resource group limits what a leaked secret can reach.

**And a shorter expiry is a feature**, not an inconvenience: it forces rotation to be a practised
routine rather than a 14-month-old memory.

</details>

---

# Section G — Case study

## Case study: Contoso identity modernisation

### Background

Contoso runs **40 microservices** across Azure Pipelines and GitHub Actions. A security audit found:

- **12 pipelines** authenticate with a **shared service principal** whose secret sits in a
  **plain-text pipeline variable** visible to all contributors
- **Three teams** independently created service principals with **Contributor on the entire production
  subscription**
- Secrets are rotated **manually** — last rotation **14 months** ago

### Requirements

**Eliminate stored secrets**

- GitHub Actions workflows must authenticate to Azure without a stored credential
- Azure Pipelines service connections must do the same
- Azure-hosted applications must use an identity with no secret

**Least privilege**

- No identity may hold Contributor over an entire subscription
- Production deployment tokens must be obtainable only from approved deployments

**Operations**

- No credential may require manual rotation
- One identity must be shareable across several Azure resources and survive their replacement

---

## Q42

Which method should the GitHub Actions workflows use?

- A. Workload identity federation with a federated credential per environment
- B. The existing shared service principal, moved to a repository secret
- C. A user-assigned managed identity on the runners
- D. A certificate-based service principal

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** lines **103–128** and the decision table at **216**.

**Why the others fail**

- **B** — a repository secret is better storage for the same problem. Still shared, still rotated
  manually, still available to any workflow in the repository
- **C** — **the trap, and the one that has cost you marks.** GitHub-hosted runners are not Azure
  resources and have no identity endpoint. Managed identity cannot work there
- **D** — a certificate is still a credential with an expiry and a rotation burden

</details>

---

## Q43

Which method should the Azure Pipelines service connections use?

- A. Workload identity federation with issuer `https://vstoken.dev.azure.com/<org-id>`
- B. A service principal with a secret in the connection
- C. A managed identity on the Microsoft-hosted agent
- D. A PAT stored in a variable group

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** lines **168–186**.

```text
2. Select "Azure Resource Manager"
3. Select "Workload Identity federation (manual)"
```

**The subject names the connection**, `sc://org/project/connection`, so the credential is bound to that
one service connection — which in turn carries its own pipeline permissions and approvals.

**Why C fails for the same reason as Q42:** a **Microsoft-hosted** agent is Microsoft's compute, not
your Azure resource. On a **self-hosted** agent running in an Azure VM, managed identity would work —
that nuance is exactly what the exam uses to make the wrong answer plausible.

</details>

---

## Q44

Which method should Azure-hosted applications use for Key Vault access?

- A. Managed identity with a data-plane role
- B. Workload identity federation
- C. The shared service principal
- D. A connection string with a Key Vault access key

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** the decision table at line **219**, with the role assignment pattern at
**303–307**.

**Why the others fail**

- **B** — WIF exists to bring **external** workloads into Azure. An Azure-hosted app already has an
  identity Azure vouches for
- **C** — the shared secret being eliminated
- **D** — Key Vault has no "access key"; that is storage-account vocabulary

**And note the role must be data-plane** — Key Vault Secrets User, not Contributor. Contributor can
delete the vault and cannot read a secret from it (Q9).

</details>

---

## Q45

Which **two** meet the least-privilege requirements? (Choose two.)

- A. Role assignments scoped to individual resource groups
- B. A federated credential with subject `environment:production`
- C. Contributor at subscription scope for each team
- D. Owner scoped to the resource group
- E. A single shared identity for all 40 services

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-39.md`:** line **41** and line **119**.

**A limits what the identity can reach; B limits when a token can be obtained at all.** Together they
answer both halves of the requirement.

**Why the others fail**

- **C** — the audit finding
- **D** — **Owner is worse than Contributor**, not better: it adds the ability to grant roles, so a
  compromised identity can escalate its own privileges
- **E** — one identity for 40 services means one compromise reaches all of them, and the audit trail
  cannot attribute actions to a service

</details>

---

## Q46

Which method meets "one identity shared across several resources, surviving their replacement"?

- A. A user-assigned managed identity
- B. A system-assigned managed identity
- C. Workload identity federation
- D. A service principal with a secret

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** lines **60–66**.

**Both halves of the requirement select user-assigned.** System-assigned is tied to exactly one
resource and is deleted with it, taking its role assignments along.

**Why it matters in an IaC world:** infrastructure is rebuilt routinely (Challenge 31). If every
redeploy created a new principal, every role assignment would need recreating — and the deployment
that recreates them needs permission to assign roles, which is a much larger grant than the workload
itself needs.

</details>

---

## Q47

Six months after migration, a workflow that has been deploying to production successfully starts
failing with `AADSTS70021`. Nothing about the Azure configuration changed. The team recently renamed
the repository from `contoso/webapp` to `contoso/web-platform`.

What happened, and what is the fix?

- A. The subject embeds the repository name, so it no longer matches — add a credential with the new
  name
- B. The client secret expired
- C. The role assignment was removed
- D. `id-token: write` was removed

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** lines **108** and **257**.

```json
    "subject": "repo:contoso/webapp:ref:refs/heads/main",
```

**The subject is an exact string containing the repository path.** Renaming the repository changes
every token GitHub issues, and Azure finds no matching credential.

**The same class of breakage applies to:** renaming the organisation, renaming a branch (`master` →
`main`), and renaming an environment. Each changes the subject.

**The fix is a new credential** with the new subject — and because credentials are additive, you can
add it **before** the rename and remove the old one afterwards, avoiding any window of failure.

**Why the others fail**

- **B** — there is no secret. That is the point of WIF
- **C** — a missing role gives `AuthorizationFailed`, and the login itself would succeed
- **D** — a missing permission fails at the token-request stage with a different error

</details>

---

## Q48

After migration, an auditor asks Contoso to prove that a production deployment last Tuesday was
performed by an approved pipeline and not by a person with a stored credential.

What can Contoso produce, and what does this illustrate?

- A. Entra sign-in logs showing the federated claims — repository, branch and environment — for that
  token exchange
- B. Nothing; sign-ins are not logged for federated identities
- C. The pipeline log only
- D. The client secret's usage history

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-39.md`:** the decision table at line **218**.

```text
| Audit trail | Sign-in logs | Sign-in logs | Sign-in logs with federated claims |
```

**All three methods produce sign-in logs, and only WIF records *why* the token was issued.** The
federated claims name the repository, the ref and the environment — so the log shows the deployment
came from `repo:contoso/webapp:environment:production`, a subject that only exists for a job that
passed the environment's approval.

**With a shared service principal secret, the same log entry proves almost nothing.** It shows the
service principal signed in. It cannot tell you whether that was a pipeline or a developer who copied
the secret out of a plain-text variable — which is exactly the situation at line 18.

**What it illustrates:** the security benefit of WIF is not only that no secret exists. It is that
**every authentication is attributable to a specific, approved workload**. That is the argument to make
when someone asks why migrating was worth the effort.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Managed identity on a GitHub-hosted runner** | Q1, Q27, Q42, Q43 | Not an Azure resource. No identity endpoint |
| **WIF proposed for an Azure VM** | Q2, Q44 | The VM already has an identity Azure vouches for |
| **Federation assumed to grant permissions** | Q17, Q26, Q29 | Authentication only. RBAC is separate |
| **`AADSTS70021` vs `AuthorizationFailed`** | Q3, Q9, Q23, Q30, Q34 | Subject mismatch vs missing role |
| **`branch:` instead of `ref:refs/heads/`** | Q4, Q20, Q36 | The subject holds the **full git ref** |
| **Wildcards in subjects** | Q13, Q20, Q28 | Exact match. One credential per subject |
| **Environment omitted from the job** | Q37 | The subject is derived from the job's context |
| **`id-token: write` forgotten** | Q7, Q30, Q37 | Fails at the token request, with a different error |
| **Control-plane role for data access** | Q9, Q38, Q44 | Contributor cannot read a blob or a secret |
| **Owner offered as "more secure"** | Q45 | Owner adds privilege escalation |
| **Principal ID and client ID swapped** | Q11, Q40 | Principal → RBAC. Client → the application |
| **Secret moved rather than removed** | Q25, Q42 | A repository secret is still a rotated secret |

---

# The decision table

**In `challenge-39.md`:** lines **212–220**. Learn this and most of Domain 4's identity questions
follow.

```text
Criteria             SP + secret              Managed identity          Workload identity federation
-------------------  -----------------------  ------------------------  ----------------------------
Secret management    store and rotate         none                      none
Rotation needed      YES (1-2 year expiry)    no                        no (token exchange)
Works from           anywhere                 AZURE-HOSTED ONLY         GitHub, Azure DevOps, OIDC
Scope control        custom RBAC              custom RBAC               custom RBAC
Audit trail          sign-in logs             sign-in logs              sign-in logs + FEDERATED CLAIMS
Best for             legacy, on-prem agents   VMs, App Service, AKS     CI/CD pipelines
Risk if compromised  usable ANYWHERE          only on its resource      only for that repo/branch/env
```

```bash
# Federated credential - GitHub  (lines 103-122)
  "issuer":    "https://token.actions.githubusercontent.com"
  "subject":   "repo:<org>/<repo>:ref:refs/heads/main"        # branch
             | "repo:<org>/<repo>:environment:production"     # environment  <- strongest
             | "repo:<org>/<repo>:ref:refs/tags/v1.0.0"       # one per tag, no wildcards
             | "repo:<org>/<repo>:pull_request"               # careful: forks
  "audiences": ["api://AzureADTokenExchange"]                 # ALWAYS

# Federated credential - Azure DevOps  (lines 173-174)
  "issuer":  "https://vstoken.dev.azure.com/<org-id>"
  "subject": "sc://<org>/<project>/<connection-name>"

# The three objects WIF needs  (lines 94-128)
az ad app create                     # 1. app registration
az ad sp create --id $APP_ID         # 2. service principal
az ad app federated-credential create --id $OBJECT_ID   # 3. trust  -> or AADSTS70021
az role assignment create --assignee $APP_ID ...        # 4. permission -> or AuthorizationFailed

# Managed identity  (lines 64-85)
az identity create ...
principalId  -> role assignments      clientId -> the application picks this identity
az role assignment create --assignee-object-id <principalId> \
  --assignee-principal-type ServicePrincipal --role "<data-plane role>" --scope "<narrow>"
```

```yaml
# The workflow half  (lines 140-156) - memorise this block
permissions:
  id-token: write
  contents: read
jobs:
  deploy:
    environment: production          # must match the credential's subject
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Your weakest domain is no longer weak. Move to Challenge 40 |
| 38–43 | Solid. Re-read the trap index and the decision table, then move on |
| 30–37 | Rewrite the decision table from memory, then retake this paper |
| Below 30 | **Stop and redo this challenge.** It is worth more marks than any other in Domain 4 |

Record your result in `AZ-400-Learning-Log.md` under Challenge 39.

:::danger The three sentences

**Stored secret?** SP yes. MI no. WIF no.

**Where can it run?** SP anywhere. MI **only on Azure-hosted compute**. WIF from a trusted OIDC issuer.

**If it leaks?** SP works anywhere until rotated. MI is useless off its resource. WIF is valid only for
one repo, branch or environment.

Say them out loud. They answer more exam questions than any other three sentences in this course.

:::
