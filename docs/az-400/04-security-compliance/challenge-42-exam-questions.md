---
sidebar_position: 4.5
toc_max_heading_level: 2
title: "Challenge 42: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 42 — AZ-400 exam questions

**48 questions** built only from what Challenge 42 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-42.md`**.

:::danger Read this before you start

Secrets questions have a **ladder**, and the exam almost always wants the highest rung the scenario
permits.

**Rung 0** — secret in a plain-text pipeline variable. The audit finding at line 17.
**Rung 1** — secret in Key Vault, fetched by the pipeline at run time.
**Rung 2** — secret in Key Vault, referenced by the app itself with `@Microsoft.KeyVault(...)`, so the
pipeline never sees it.
**Rung 3** — **no secret exists**: managed identity to SQL, OIDC to Azure.

When an option says "store it in Key Vault" and another says "use a managed identity so there is
nothing to store", the second one wins — unless the scenario says the credential is a third party's
and cannot be replaced.

:::

---

# Section A — Multiple choice

---

## Q1

Contoso wants pipeline secrets to reflect the latest Key Vault version with no pipeline changes after a
rotation. Which approach achieves this in Azure Pipelines?

- A. Store secrets as pipeline variables and update them after each rotation
- B. Use a variable group linked to Azure Key Vault
- C. Use `AzureKeyVault@2` with specific secret version IDs
- D. Copy secrets into pipeline variables with a script task

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-42.md`:** lines **187–204**.

```yaml
variables:
  - group: KeyVault-Production-Secrets
```

**The link resolves at run time and always fetches the current version.** Rotate the secret in the
vault and the next run picks it up — no edit, no redeploy, no ticket.

**Why C is the near-miss worth understanding.** `AzureKeyVault@2` also fetches at run time and would be
correct on its own — but **pinning a version ID** freezes it. The moment the secret rotates, the
pinned version is stale, and Task 3's filter at line 162 deliberately names secrets rather than
versions.

**Why the others fail** — A is the manual process being eliminated, and D copies a value into a
variable at some point in time, which is A with extra steps.

</details>

---

## Q2

An App Service application must reach Azure SQL Database. What is the most secure approach?

- A. Store the connection string with username and password in Key Vault
- B. Use a system-assigned managed identity with Microsoft Entra authentication to SQL
- C. Use a service principal with a client certificate stored in Key Vault
- D. Use contained database users with passwords rotated monthly

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-42.md`:** lines **263–268**.

```bash
  --settings Default="Server=tcp:sql-contoso.database.windows.net,1433;Database=ContosoDb;Authentication=Active Directory Managed Identity;Encrypt=true"
```

**Look at what is missing from that connection string: there is no password.** `Authentication=Active
Directory Managed Identity` replaces it with a token the platform issues and rotates.

**Why A is the trap, and why it is such a good one.** Key Vault is the right answer to *many* questions
in this challenge, so it pattern-matches. But storing a password securely is still storing a password —
rung 1 when rung 3 is available. Compare line 54, where the original connection string carries
`Password=SecureP@ss123`.

**Why the others fail** — C swaps a password for a certificate, which still expires and still needs
rotating; D is rung 0 with a calendar reminder attached.

</details>

---

## Q3

Which Key Vault RBAC role suits a CI/CD pipeline that reads secret values but must not create or delete
them?

- A. Key Vault Administrator
- B. Key Vault Secrets Officer
- C. Key Vault Secrets User
- D. Key Vault Reader

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-42.md`:** lines **307–312**.

```text
| Key Vault Secrets User | Read secret values only |
| Key Vault Reader | Read metadata only (not secret values) |
```

**Those two rows are the whole question.** Reader sees that a secret named `SqlConnectionString`
exists, and cannot read it. Secrets User reads the value.

**Why B is over-privileged** — Officer adds create, update and delete, which a deployment pipeline
never needs and which would let a compromised pipeline overwrite production credentials.

**Note the phrase "not secret values" in the Reader row.** That parenthesis is the exam's entire basis
for the distractor.

</details>

---

## Q4

App Service settings use `@Microsoft.KeyVault(...)` references. What happens when a secret is rotated?

- A. The App Service picks up the new value immediately
- B. The App Service picks up the new value within about 24 hours via background refresh
- C. The App Service must be restarted
- D. The reference must be updated with the new version

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-42.md`:** line **261**.

```bash
  --settings "PaymentGateway__ApiKey=@Microsoft.KeyVault(VaultName=kv-contoso-secrets-001;SecretName=ApiKey-PaymentGateway)"
```

**The reference names the secret, not a version**, so it resolves to current — but resolution is
**cached and refreshed periodically**, roughly daily.

**Which is why the exam offers "immediately" and "restart required" as bookends.** Neither is right,
and the practical consequence matters: after an emergency rotation you do not wait a day, you force a
settings update or restart to pull the new value now.

</details>

---

## Q5

The `AzureKeyVault@2` task fails with *"Access denied. Caller was not found on any access policy."*
The vault has `enableRbacAuthorization` set to `true`. What is the cause?

- A. The vault firewall blocks the agent
- B. The service principal has an access policy but no RBAC role
- C. The secret has expired
- D. The service connection is not authorised for the pipeline

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-42.md`:** lines **332–334**.

**The error message is misleading on purpose, and the exam knows it.** It mentions *access policy*
because that is the legacy vocabulary — but on an RBAC-enabled vault, access policies are **not
consulted at all**. Whatever policy exists is dead configuration.

**The diagnosis is two commands** (lines 340–346): check `enableRbacAuthorization`, then list role
assignments at the vault scope. **The fix is a role assignment**, not a policy edit (lines 356–359).

**Why D is worth ruling out by its timing.** An unauthorised service connection fails *before the job
starts* with a resource authorization problem (Challenge 41 Q6). This failure happens **inside the
task**, so the connection was released and the login succeeded.

</details>

---

## Q6

An App Service setting using a Key Vault reference shows a red status in the portal. What are the two
things to check?

- A. Whether the managed identity is enabled, and whether it holds Key Vault Secrets User
- B. Whether the secret has an expiry date set
- C. Whether the vault is in the same region as the app
- D. Whether the app is running on a Premium plan

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-42.md`:** lines **368–378**.

```bash
az webapp identity show --name app-contoso-web --resource-group rg-contoso-secrets
az role assignment list --assignee $WEBAPP_IDENTITY --all \
  --query "[?contains(scope, 'kv-contoso')]"
```

**Identity, then permission — in that order**, because the second is meaningless without the first.
This is the same two-step as Challenge 39: authenticate, then authorise.

**Why C and D are plausible-sounding infrastructure noise.** Key Vault references work cross-region and
on any App Service tier. The exam includes options like these to see whether you reason from the
mechanism or from vague intuition about Azure.

</details>

---

## Q7

Why does the vault use `--enable-rbac-authorization true`?

- A. It is required for Key Vault references to work
- B. It gives centralised, granular, scopeable permissions and is Microsoft's recommendation for new
  deployments
- C. It is the only mode that supports soft delete
- D. Access policies do not work with managed identities

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-42.md`:** lines **39** and **294–301**.

```text
| Scope | Vault level only | Subscription, RG, vault, or individual secret |
| Recommended for | Legacy configurations | New deployments (Microsoft recommended) |
```

**The scope row is the substantive difference.** An access policy applies to the **whole vault** —
there is no way to say "this identity may read only `ApiKey-PaymentGateway`". RBAC can scope to an
individual secret.

**Why the others fail** — A and D are false; both work in either mode. C is false: soft delete is a
vault feature independent of the authorisation model, which is why cleanup at lines 449–450 needs both
`delete` **and** `purge`.

</details>

---

## Q8

Three API keys leaked into pipeline logs through accidental `echo` statements. Which mechanism prevents
recurrence in GitHub Actions?

- A. `echo "::add-mask::$SECRET"`
- B. `continue-on-error: true`
- C. Setting the job to `runs-on: self-hosted`
- D. Deleting the run logs after each deployment

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-42.md`:** line **133**.

```bash
          SECRET=$(az keyvault secret show ... --query value -o tsv)
          echo "::add-mask::$SECRET"
          echo "sql-connection=$SECRET" >> $GITHUB_OUTPUT
```

**Registering the value tells the runner to replace it with `***` everywhere it appears** for the rest
of the job — including in output nobody anticipated.

**Note the ordering, because it is the whole trick.** Mask **before** the value can be printed. A mask
registered after the leak does not retroactively clean the log.

**Why D is the answer that sounds responsible and is not.** By the time you delete the log the secret
has been readable by 150 contributors (line 17), and a deleted log is also a deleted audit trail. The
correct response to a leaked secret is to **rotate it**, not to hide the evidence.

</details>

---

## Q9

What does `RunAsPreJob: true` do on the `AzureKeyVault@2` task?

- A. Runs the task before any other job in the pipeline
- B. Fetches the secrets before the job's steps begin, so they are available to every step
- C. Caches secrets between pipeline runs
- D. Runs the task only on pull requests

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-42.md`:** lines **158–163**.

```yaml
  - task: AzureKeyVault@2
    inputs:
      SecretsFilter: 'SqlConnectionString,ApiKey-PaymentGateway,StorageAccountKey'
      RunAsPreJob: true
```

**Pre-job, not pre-pipeline.** The secrets become variables before the first step executes, which
matters when an earlier step — a checkout hook, a script — needs one.

**Why C is worth rejecting on principle.** Caching secrets between runs would leave them on the agent,
which is precisely the class of exposure this challenge is closing.

</details>

---

## Q10

What does `SecretsFilter: '*'` do, and why does Task 3 not use it?

- A. Nothing; it is invalid
- B. It downloads every secret in the vault, which is broader than the pipeline needs
- C. It enables wildcard matching on secret names
- D. It filters out expired secrets

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-42.md`:** line **162**.

```yaml
      SecretsFilter: 'SqlConnectionString,ApiKey-PaymentGateway,StorageAccountKey'
```

**Naming three secrets is least privilege applied at the *fetch*, not just at the role.** Even with
Secrets User on the whole vault, the pipeline pulls only what it uses — so a future secret added by
another team never lands on this agent.

**This is the same instinct as `--scopes` on a service principal**: the role says what you *may* read,
the filter says what you *do* read, and defence in depth means narrowing both.

</details>

---

## Q11

Which role does the security team need to manage all vault contents?

- A. Key Vault Administrator
- B. Key Vault Secrets Officer
- C. Key Vault Crypto Officer
- D. Key Vault Reader

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-42.md`:** lines **307–311** and **322–325**.

```text
| Key Vault Administrator | Full management of all vault contents |
```

**"All contents" is the operative phrase** — secrets, keys and certificates. The Officer roles are each
scoped to one object type: Secrets Officer for secrets (line 308), Certificates Officer for
certificates (line 310), Crypto Officer for keys (line 311).

**A useful reading of the whole table:** *Officer* means manage, *User* means use, *Reader* means see
that it exists. Once you have that, the six rows collapse into three ideas.

</details>

---

## Q12

Which event type triggers the rotation function?

- A. `Microsoft.KeyVault.SecretNewVersionCreated`
- B. `Microsoft.KeyVault.SecretNearExpiry`
- C. `Microsoft.KeyVault.SecretExpired`
- D. `Microsoft.KeyVault.VaultAccessPolicyChanged`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-42.md`:** line **282**.

```bash
  --included-event-types "Microsoft.KeyVault.SecretNearExpiry"
```

**Near expiry, not expired**, because rotation must happen **before** anything breaks. An event that
fires on expiry arrives after the outage has begun.

**And the event only fires if the secret has an expiry date**, which is why line 67–70 sets one:

```bash
az keyvault secret set-attributes \
  --name "ApiKey-PaymentGateway" \
  --expires "2025-12-31T23:59:59Z"
```

**No expiry, no event, no rotation.** That chain is a favourite exam question.

</details>

---

## Q13

Why enable diagnostic settings with the `AuditEvent` category on the vault?

- A. To alert when a secret is near expiry
- B. To record who accessed which secret and when
- C. To replicate secrets to a second region
- D. To enforce RBAC

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-42.md`:** lines **285–289**.

```bash
  --logs '[{"category":"AuditEvent","enabled":true}]'
```

**Access control decides who *may* read a secret; audit logging records who *did*.** The scenario at
line 17 could not answer either question — 34 secrets readable by 150 people, with no record of use.

**Why A is a different mechanism** — near-expiry is the Event Grid subscription in Q12. The exam pairs
these two deliberately, because both are "monitoring the vault" and they answer different questions.

</details>

---

## Q14

An App Service must read a payment API key at run time. Which pattern keeps the key out of the
pipeline entirely?

- A. Fetch it in the pipeline and set it as an app setting
- B. Use a Key Vault reference in the app setting
- C. Store it as a GitHub Actions secret
- D. Bake it into the container image

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-42.md`:** line **261**.

**The pipeline sets a *pointer*, not a *value*.** The app setting contains
`@Microsoft.KeyVault(VaultName=...;SecretName=...)`, and the App Service resolves it with **its own
managed identity** at run time.

**That is rung 2 on the ladder**, and the difference from A is worth stating plainly: in A the secret
passes through the agent, the logs and the deployment history. In B it never leaves the vault except
to the app that uses it.

**Why D is the worst option available.** A secret in an image is in every registry copy, every layer
cache and every machine that ever pulled it, and rotating it means rebuilding and redeploying.

</details>

---

## Q15

What must be true before `Authentication=Active Directory Managed Identity` works against Azure SQL?

- A. The App Service has a managed identity and that identity is known to SQL
- B. The SQL password is stored in Key Vault
- C. The App Service and SQL are in the same resource group
- D. SQL authentication is disabled on the server

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-42.md`:** lines **233–248**.

```bash
az webapp identity assign --name app-contoso-web --resource-group rg-contoso-secrets
az sql server ad-admin create --object-id $WEBAPP_IDENTITY
```

**Two halves again: the identity must exist, and SQL must recognise it.**

**And read the comment above that command, because it is the exam-relevant caveat** (lines 242–243):
making the app's identity the **server AD admin** is server-level access, and the challenge itself flags
that a **contained database user** is the least-privilege alternative. The lab takes the shortcut; the
exam expects you to know it is one.

</details>

---

## Q16

Which command grants the App Service identity permission to read secrets?

- A. `az keyvault set-policy --object-id $WEBAPP_IDENTITY --secret-permissions get`
- B. `az role assignment create --assignee-object-id $WEBAPP_IDENTITY --assignee-principal-type ServicePrincipal --role "Key Vault Secrets User"`
- C. `az webapp config appsettings set --settings KEYVAULT_ACCESS=true`
- D. `az keyvault update --enable-rbac-authorization false`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-42.md`:** lines **251–255**.

```bash
az role assignment create \
  --assignee-object-id $WEBAPP_IDENTITY \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope ".../vaults/kv-contoso-secrets-001"
```

**Why A does nothing on this vault.** `set-policy` writes an access policy, and the vault was created
with `--enable-rbac-authorization true` (line 39) — the policy is ignored. This is Break scenario 1
approached from the other direction.

**Why D is the disastrous "fix" the exam likes offering.** Turning RBAC off to make the access policy
work downgrades the vault to the legacy model, discards every existing role assignment's effect, and
removes per-secret scoping.

**And note `--assignee-principal-type ServicePrincipal`** — a managed identity **is** a service
principal in the directory. There is no `ManagedIdentity` principal type.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** are advantages of Azure RBAC over access policies for Key Vault? (Choose three.)

- A. Permissions can be scoped to an individual secret
- B. Management is centralised with the rest of Azure RBAC
- C. Conditional access is supported through Microsoft Entra ID
- D. It is faster at run time
- E. It removes the need for a managed identity
- F. It is the only mode supporting soft delete

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-42.md`:** lines **296–301**.

```text
| Scope | Vault level only | Subscription, RG, vault, or individual secret |
| Management | Per-vault configuration | Centralized with Azure RBAC |
| Conditional access | No | Yes (via Microsoft Entra ID) |
```

**Row by row, those are the three real differences.** Everything else in the table is a nuance of the
same three.

**Why the others fail** — D is invented, E confuses authorisation with identity (you still need an
identity to assign a role *to*), and F is false: soft delete is independent of the authorisation model.

</details>

---

## Q18

Which **three** eliminate a stored secret rather than merely protecting one? (Choose three.)

- A. Managed identity authentication to Azure SQL
- B. OIDC login from GitHub Actions to Azure
- C. A federated service connection in Azure Pipelines
- D. Storing the SQL password in Key Vault
- E. Storing the SQL password as a GitHub secret
- F. Encrypting the password before storing it as a pipeline variable

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-42.md`:** lines **263–268**, **87–103**, **160**.

**Ask one question of each option: after this change, does a credential still exist somewhere?**

A, B and C all answer no — the platform issues a short-lived token on demand. D, E and F all answer
yes; they differ only in **where** the credential sits and how well it is guarded.

**D is not wrong as an action** — Key Vault is a large improvement over line 17's plain-text variables.
It is simply not what the question asked. **Read the verb: "eliminate", not "protect".**

</details>

---

## Q19

Which **two** does a variable group linked to Key Vault provide over the `AzureKeyVault@2` task?
(Choose two.)

- A. It is reusable across multiple pipelines from one definition
- B. It is managed in the Library UI rather than in each YAML file
- C. It fetches secrets faster
- D. It removes the need for a service connection
- E. It works without any role assignment

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-42.md`:** lines **187–204**.

```yaml
variables:
  - group: KeyVault-Production-Secrets
```

**One definition, referenced by many pipelines** — change the linked secret list once and every
consumer follows. The task at line 158 must be repeated, and kept consistent, in each pipeline that
needs it.

**Why D and E are the same misunderstanding.** The variable group still authenticates through the
**Azure subscription** named at line 190 — `Azure-Prod-Federated` — and that identity still needs Key
Vault Secrets User. The abstraction changes *where the configuration lives*, not *what permissions are
required*.

</details>

---

## Q20

Which **two** prevent secrets from appearing in logs? (Choose two.)

- A. Secrets from a linked variable group are masked automatically
- B. `echo "::add-mask::$SECRET"` in GitHub Actions
- C. Printing only the length of the value
- D. Setting `continue-on-error: true`
- E. Using `-o tsv` on the CLI query

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-42.md`:** lines **210–212** and **133**.

```yaml
  - script: |
      echo "Secrets from variable group are automatically masked in logs"
      echo "$(SqlConnectionString)"
```

**A is the one candidates disbelieve.** That script deliberately echoes the secret — and the log shows
`***`, because Azure Pipelines masks any value it knows is a secret.

**Why C is a good habit that is not the mechanism.** Line 168 prints
`${#SQLCONNECTIONSTRING}` to prove a secret loaded without revealing it — sound practice, and it
depends on the author choosing to do it. Masking protects you when nobody thought about it.

**Masking is not perfect, and the exam does not claim it is.** Split a secret across lines, base64 it,
or transform it and the masker no longer recognises the string. That is why elimination beats
protection.

</details>

---

## Q21

Which **two** are required for the Event Grid rotation flow to fire? (Choose two.)

- A. The secret has an expiry date
- B. An event subscription for `Microsoft.KeyVault.SecretNearExpiry`
- C. Diagnostic settings sending `AuditEvent` to Log Analytics
- D. The vault uses access policies
- E. The Function App holds Key Vault Administrator

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-42.md`:** lines **67–70** and **277–282**.

**No expiry date means no near-expiry event.** A secret with `expires: null` never approaches
anything, so the subscription sits silent forever — the single most common reason a correctly built
rotation pipeline never runs.

**Why C is the adjacent-but-different control** — audit logging records access. It is not part of the
event path.

**Why E is over-scoped even though the function does need permission.** Rotating a secret needs
**Secrets Officer** (create/update, line 308), not Administrator over keys and certificates too.

</details>

---

## Q22

Which **two** describe Key Vault references in App Service? (Choose two.)

- A. They resolve using the app's managed identity
- B. They refresh periodically rather than instantly
- C. They require the secret version to be pinned
- D. They only work on Premium tiers
- E. They are resolved by the deployment pipeline

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-42.md`:** lines **251–261** and **368**.

**A is why Break scenario 2 exists** — no identity or no role, and the reference shows red. The
reference is not magic; it is the app authenticating to the vault on its own behalf.

**Why E is exactly backwards, and this is the point of the pattern.** The pipeline writes the
*reference string*. The **app** resolves it. That is what keeps the secret out of the agent, the logs
and the deployment history (Q14).

</details>

---

## Q23

Which **two** correctly diagnose a 403 from Key Vault on an RBAC-enabled vault? (Choose two.)

- A. `az keyvault show --query "properties.enableRbacAuthorization"`
- B. `az role assignment list --scope "<vault-resource-id>"`
- C. `az keyvault show --query "properties.accessPolicies"`
- D. `az keyvault secret list`
- E. `az webapp log tail`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-42.md`:** lines **339–346**.

```bash
az keyvault show --name kv-contoso-secrets-001 \
  --query "properties.enableRbacAuthorization"

az role assignment list \
  --scope ".../vaults/kv-contoso-secrets-001" \
  --query "[].{principal:principalName, role:roleDefinitionName}" -o table
```

**Establish the mode first, then look in the right place.** Checking role assignments on a
policy-based vault, or policies on an RBAC vault, produces a confidently wrong conclusion.

**Why C is the trap the error message sets for you.** The message says "not found on any access
policy", so listing access policies feels like the obvious next step — and on this vault those policies
are inert.

**Why D fails as a diagnostic** — it needs the very permission under investigation, so it tells you
only that something is wrong, which you already knew.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must remove 34 plain-text secrets readable by 150 contributors, stop secrets
reaching pipeline logs, ensure rotations propagate without pipeline edits, and eliminate stored
credentials wherever the platform allows.

---

## Q24

**Proposed solution:** Create an RBAC-enabled Key Vault and move all 34 secrets into it. Grant each
pipeline identity Key Vault Secrets User scoped to the vault. Link a variable group to the vault for
Azure Pipelines and use OIDC login plus the CLI for GitHub Actions. Convert the App Service to a
managed identity for SQL, and use Key Vault references for the remaining third-party keys. Set expiry
dates and subscribe a rotation function to `SecretNearExpiry`. Enable `AuditEvent` diagnostics.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-42.md`:** lines **35–40**, **187–192**, **87–103**, **263–268**, **261**, **67–70**,
**277–289**, **316–319**.

| Requirement | Mechanism |
|---|---|
| No plain-text secrets | Key Vault with RBAC, Secrets User per identity |
| Nothing in logs | Automatic masking + `::add-mask::` |
| Rotation without pipeline edits | Variable group link resolves at run time |
| Eliminate credentials where possible | Managed identity to SQL, OIDC to Azure |
| Third-party keys still needed | Key Vault references, resolved by the app |
| Rotation is driven, not remembered | Expiry dates + `SecretNearExpiry` |
| Access is provable | `AuditEvent` diagnostics |

**Note how it handles the keys that *cannot* be eliminated.** A payment gateway key belongs to a third
party — there is no managed identity to swap in. The design puts it on the highest rung available:
stored in the vault, referenced by the app, never touched by the pipeline.

</details>

---

## Q25

**Proposed solution:** Create a Key Vault using access policies. Grant every pipeline service principal
Key Vault Administrator. Fetch secrets in each pipeline with a script task and set them as pipeline
variables. Keep the SQL password in the connection string but store that string in Key Vault.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures, and each one is a rung the design refused to climb.**

**Access policies are vault-level only** (line 297). No per-secret scoping, no conditional access, and
Microsoft recommends RBAC for new deployments (line 301).

**Key Vault Administrator for a pipeline is the opposite of least privilege.** A compromised pipeline
could overwrite or delete every secret, key and certificate in the vault. It needs **Secrets User**.

**Copying secrets into pipeline variables recreates the finding.** The values now live in the pipeline
again — which is where they started, at line 17.

**Keeping the SQL password stores a credential that did not need to exist.** Managed identity
authentication removes it entirely (line 268).

</details>

---

## Q26

**Proposed solution:** Create an RBAC-enabled Key Vault, grant each pipeline identity Key Vault Secrets
User, link a variable group for Azure Pipelines, use OIDC for GitHub Actions, convert SQL access to
managed identity, use Key Vault references for third-party keys, and enable `AuditEvent` diagnostics.
Do not set expiry dates on secrets, since expiry causes outages.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**Without an expiry date, `SecretNearExpiry` never fires.** The rotation function is deployed, wired
up, and permanently idle. Rotation reverts to whoever remembers, which is how the scenario produced
three-year-old keys in the first place.

**The stated reasoning is the real error, and it inverts cause and effect.** Expiry does not cause
outages — **unmanaged** expiry does. The expiry date is what *creates the warning* that prevents the
outage:

```bash
az keyvault secret set-attributes \
  --name "ApiKey-PaymentGateway" \
  --expires "2025-12-31T23:59:59Z"
```

**A secret with no expiry is not safe. It is unmonitored** — and the exam builds Yes/No triplets on
exactly this kind of confident, plausible, wrong justification.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — Key Vault authorisation

| # | Statement | Answer |
|---|---|---|
| 1 | On an RBAC-enabled vault, access policies are ignored |  |
| 2 | RBAC can scope permissions to an individual secret |  |
| 3 | Key Vault Reader can read secret values |  |
| 4 | Key Vault Secrets User can delete secrets |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | On an RBAC-enabled vault, access policies are ignored | **Yes** |
| 2 | RBAC can scope permissions to an individual secret | **Yes** |
| 3 | Key Vault Reader can read secret values | **No** |
| 4 | Key Vault Secrets User can delete secrets | **No** |

**In `challenge-42.md`:** lines **334**, **297**, **312**, **309**.

Row 1 is Break scenario 1's cause, and row 3 is the distinction the exam tests most often: **metadata
versus value**.

Row 4: reading is not managing. Deleting requires Secrets Officer.

</details>

---

## Q28 — secrets in pipelines

| # | Statement | Answer |
|---|---|---|
| 1 | Linked variable group secrets are masked in logs automatically |  |
| 2 | `::add-mask::` must run before the value could be printed |  |
| 3 | A variable group fetches the latest version at run time |  |
| 4 | `SecretsFilter` must list every secret in the vault |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Linked variable group secrets are masked in logs automatically | **Yes** |
| 2 | `::add-mask::` must run before the value could be printed | **Yes** |
| 3 | A variable group fetches the latest version at run time | **Yes** |
| 4 | `SecretsFilter` must list every secret in the vault | **No** |

**In `challenge-42.md`:** lines **211**, **133**, **189**, **162**.

Row 2 is the ordering rule — masking is not retroactive.

Row 4: naming only what you use is least privilege applied at the fetch (Q10).

</details>

---

## Q29 — the secretless pattern

| # | Statement | Answer |
|---|---|---|
| 1 | Managed identity authentication to SQL removes the password |  |
| 2 | A Key Vault reference is resolved by the App Service, not the pipeline |  |
| 3 | Key Vault references need the secret version pinned |  |
| 4 | A managed identity is a service principal in the directory |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Managed identity authentication to SQL removes the password | **Yes** |
| 2 | A Key Vault reference is resolved by the App Service, not the pipeline | **Yes** |
| 3 | Key Vault references need the secret version pinned | **No** |
| 4 | A managed identity is a service principal in the directory | **Yes** |

**In `challenge-42.md`:** lines **268**, **261**, **253**.

Row 2 is the whole value of the pattern (Q14, Q22).

Row 4 is why `--assignee-principal-type ServicePrincipal` is correct for a managed identity, and why
there is no `ManagedIdentity` principal type to choose.

</details>

---

## Q30 — rotation and audit

| # | Statement | Answer |
|---|---|---|
| 1 | `SecretNearExpiry` fires only if the secret has an expiry date |  |
| 2 | `AuditEvent` diagnostics record who read which secret |  |
| 3 | Event Grid subscriptions replace the need for RBAC |  |
| 4 | App Service picks up a rotated secret instantly |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `SecretNearExpiry` fires only if the secret has an expiry date | **Yes** |
| 2 | `AuditEvent` diagnostics record who read which secret | **Yes** |
| 3 | Event Grid subscriptions replace the need for RBAC | **No** |
| 4 | App Service picks up a rotated secret instantly | **No** |

**In `challenge-42.md`:** lines **67–70**, **282**, **289**, and Q4.

Row 1 is Q26's failure, stated as a fact.

Rows 2 and 3 separate **detective** from **preventive** control: logging tells you what happened; RBAC
decides what is allowed. Neither substitutes for the other.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each identity to the Key Vault role it should hold.

| Identity | Role |
|---|---|
| CI/CD pipeline reading secrets at deploy time |  |
| Security team managing all vault contents |  |
| Rotation function creating new secret versions |  |
| Auditor listing secret names and expiry dates |  |
| Service managing TLS certificates |  |

**Options:** Key Vault Administrator · Key Vault Certificates Officer · Key Vault Reader · Key Vault Secrets Officer · Key Vault Secrets User

<details>
<summary>Show answer</summary>

| Identity | Role |
|---|---|
| CI/CD pipeline reading secrets at deploy time | **Key Vault Secrets User** |
| Security team managing all vault contents | **Key Vault Administrator** |
| Rotation function creating new secret versions | **Key Vault Secrets Officer** |
| Auditor listing secret names and expiry dates | **Key Vault Reader** |
| Service managing TLS certificates | **Key Vault Certificates Officer** |

**In `challenge-42.md`:** lines **307–312**, **316–325**.

**Officer manages, User uses, Reader only sees that it exists.** Three words cover the whole table.

**The rotation function row is the one people get wrong** by reaching for Administrator. It writes
secrets, so Secrets Officer — and nothing about keys or certificates.

</details>

---

## Q32

Match each requirement to the correct mechanism.

| Requirement | Mechanism |
|---|---|
| App reads a third-party API key at run time |  |
| App authenticates to Azure SQL with no password |  |
| Pipeline authenticates to Azure with no secret |  |
| Rotation propagates without editing pipelines |  |
| Rotation is triggered rather than remembered |  |
| Prove who read a secret last Tuesday |  |

**Options:** `AuditEvent` diagnostic logs · Expiry date + `SecretNearExpiry` event · Key Vault reference in the app setting · Managed identity + Entra authentication · OIDC / workload identity federation · Variable group linked to Key Vault

<details>
<summary>Show answer</summary>

| Requirement | Mechanism |
|---|---|
| App reads a third-party API key at run time | **Key Vault reference in the app setting** |
| App authenticates to Azure SQL with no password | **Managed identity + Entra authentication** |
| Pipeline authenticates to Azure with no secret | **OIDC / workload identity federation** |
| Rotation propagates without editing pipelines | **Variable group linked to Key Vault** |
| Rotation is triggered rather than remembered | **Expiry date + `SecretNearExpiry` event** |
| Prove who read a secret last Tuesday | **`AuditEvent` diagnostic logs** |

**In `challenge-42.md`:** lines **261**, **268**, **87–103**, **187–204**, **67–70** with **282**,
**285–289**.

**Rows 1–3 are the ladder.** Rung 2 when the credential belongs to someone else, rung 3 when it does
not.

</details>

---

## Q33

Arrange the steps to move a pipeline from plain-text secrets to Key Vault.

**Items:** Delete the plain-text pipeline variables · Create the vault with RBAC authorisation · Link a
variable group to the vault · Store the secrets in the vault · Grant the pipeline identity Key Vault
Secrets User · Update the pipeline to consume the variable group

<details>
<summary>Show answer</summary>

### Answer

1. Create the vault with RBAC authorisation — lines **35–40**
2. Store the secrets in the vault — lines **51–64**
3. Grant the pipeline identity Key Vault Secrets User — lines **316–319**
4. Link a variable group to the vault — lines **187–192**
5. Update the pipeline to consume the variable group — lines **203–204**
6. Delete the plain-text pipeline variables — line **17**

**Deleting last, as always.** Remove the variables before the vault path is proven and every pipeline
fails at once.

**Step 3 before step 4 is the detail worth catching.** Linking a variable group **reads the vault
immediately** to list the secrets you can select. Without the role assignment first, the Library UI
returns an empty list and the link cannot be created.

**And rotate every secret afterwards.** They were readable by 150 contributors (line 17) — moving a
compromised value into a vault stores a compromised value securely.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| "Caller was not found on any access policy" on an RBAC vault |  |
| Red status on a Key Vault reference in App Service |  |
| Rotation function never runs |  |
| Pipeline reads a stale secret after rotation |  |
| Secret visible in the run log |  |

**Options:** A pinned secret version · Identity missing or lacking Secrets User · No RBAC role assignment · The secret has no expiry date · Value never registered for masking

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| "Caller was not found on any access policy" on an RBAC vault | **No RBAC role assignment** |
| Red status on a Key Vault reference in App Service | **Identity missing or lacking Secrets User** |
| Rotation function never runs | **The secret has no expiry date** |
| Pipeline reads a stale secret after rotation | **A pinned secret version** |
| Secret visible in the run log | **Value never registered for masking** |

**In `challenge-42.md`:** lines **334**, **368**, **67–70** with **282**, Q1, **133**.

**The first row's message is a false lead by design** — read the vault's authorisation mode before
believing the error's vocabulary.

</details>

---

## Q35

Match each control to what it provides.

| Control | Provides |
|---|---|
| RBAC role assignment |  |
| `AuditEvent` diagnostics |  |
| Secret expiry date |  |
| `::add-mask::` |  |
| `SecretsFilter` |  |

**Options:** A deadline that generates a warning · Only the secrets this pipeline needs · Redaction in logs · Who did read a secret · Who may read a secret

<details>
<summary>Show answer</summary>

| Control | Provides |
|---|---|
| RBAC role assignment | **Who may read a secret** |
| `AuditEvent` diagnostics | **Who did read a secret** |
| Secret expiry date | **A deadline that generates a warning** |
| `::add-mask::` | **Redaction in logs** |
| `SecretsFilter` | **Only the secrets this pipeline needs** |

**In `challenge-42.md`:** lines **316–319**, **285–289**, **67–70**, **133**, **162**.

**Rows 1 and 2 are preventive and detective**, and an exam answer that offers logging as a substitute
for permissions — or the reverse — is wrong on that basis alone.

</details>

---

# Section F — Hot area

---

## Q36

```bash
az keyvault create \
  --name kv-contoso-secrets-001 \
  --resource-group rg-contoso-secrets \
  --[BLANK 1] [BLANK 2] \
  --sku [BLANK 3]
```

Requirement: Microsoft's recommended authorisation model for a new vault, software-protected keys.

- **BLANK 1:** `enable-rbac-authorization` / `enable-access-policies` / `enable-purge-protection` /
  `enable-soft-delete`
- **BLANK 2:** `true` / `false`
- **BLANK 3:** `standard` / `premium` / `basic` / `free`

<details>
<summary>Show answer</summary>

### Answer: `enable-rbac-authorization`, `true`, `standard`

**In `challenge-42.md`:** lines **35–40**.

**`standard` is software-protected; `premium` adds HSM-backed keys.** Nothing in this challenge needs
an HSM, and premium costs more — so standard is the right answer *because of the requirement*, not by
default.

**RBAC true is line 301's recommendation**, and it is what makes Break scenario 1 possible when someone
later configures an access policy out of habit.

</details>

---

## Q37

```bash
az keyvault secret set-attributes \
  --vault-name kv-contoso-secrets-001 \
  --name "ApiKey-PaymentGateway" \
  --[BLANK 1] "2025-12-31T23:59:59Z"
```

- **BLANK 1:** `expires` / `not-before` / `disabled` / `tags`

<details>
<summary>Show answer</summary>

### Answer: `expires`

**In `challenge-42.md`:** lines **67–70**.

**This one line is what makes the entire rotation flow possible.** Without it, the Event Grid
subscription at line 282 never fires, and Q26's design is quietly broken.

**`not-before` is the real distractor**, and worth knowing: it sets the time a secret *becomes* valid —
useful when staging a replacement ahead of a cutover, and no help at all for rotation warnings.

</details>

---

## Q38

```yaml
  - task: AzureKeyVault@2
    inputs:
      azureSubscription: 'Azure-Prod-Federated'
      KeyVaultName: 'kv-contoso-secrets-001'
      [BLANK 1]: 'SqlConnectionString,ApiKey-PaymentGateway,StorageAccountKey'
      [BLANK 2]: true
```

- **BLANK 1:** `SecretsFilter` / `SecretNames` / `Secrets` / `IncludeSecrets`
- **BLANK 2:** `RunAsPreJob` / `ContinueOnError` / `CacheSecrets` / `MaskValues`

<details>
<summary>Show answer</summary>

### Answer: `SecretsFilter`, `RunAsPreJob`

**In `challenge-42.md`:** lines **162–163**.

**`RunAsPreJob: true` makes the secrets available to every step in the job**, including ones that run
before your first script.

**`MaskValues` is the trap for people who half-remember Q20**, because masking sounds like something
you would configure. You do not — Azure Pipelines masks known secret values automatically (line 211).

</details>

---

## Q39

```bash
az role assignment create \
  --[BLANK 1] $WEBAPP_IDENTITY \
  --assignee-principal-type [BLANK 2] \
  --role "[BLANK 3]" \
  --scope ".../vaults/kv-contoso-secrets-001"
```

Requirement: the App Service reads secret values at run time, and nothing more.

- **BLANK 1:** `assignee-object-id` / `assignee` / `assignee-client-id` / `principal`
- **BLANK 2:** `ServicePrincipal` / `ManagedIdentity` / `User` / `Group`
- **BLANK 3:** `Key Vault Secrets User` / `Key Vault Secrets Officer` / `Key Vault Administrator` /
  `Key Vault Reader`

<details>
<summary>Show answer</summary>

### Answer: `assignee-object-id`, `ServicePrincipal`, `Key Vault Secrets User`

**In `challenge-42.md`:** lines **251–255**.

**`ManagedIdentity` is not a principal type.** Managed identities appear in the directory as service
principals, which is why every role assignment in this challenge uses `ServicePrincipal`.

**`--assignee-object-id` with the explicit type skips the Graph lookup** and its replication delay —
the reason a role assignment made seconds after `az identity create` can otherwise fail.

</details>

---

## Q40

```yaml
      - name: Get secret via Azure CLI
        id: get-secret
        run: |
          SECRET=$(az keyvault secret show --vault-name kv-contoso-secrets-001 \
            --name SqlConnectionString --query value -o tsv)
          echo "[BLANK 1]$SECRET"
          echo "sql-connection=$SECRET" >> [BLANK 2]
```

- **BLANK 1:** `::add-mask::` / `::warning::` / `::set-output name=secret::` / `::group::`
- **BLANK 2:** `$GITHUB_OUTPUT` / `$GITHUB_ENV` / `$GITHUB_STATE` / `$GITHUB_PATH`

<details>
<summary>Show answer</summary>

### Answer: `::add-mask::`, `$GITHUB_OUTPUT`

**In `challenge-42.md`:** lines **133–134**.

```bash
          echo "::add-mask::$SECRET"
          echo "sql-connection=$SECRET" >> $GITHUB_OUTPUT
```

**Mask first, then publish.** Reverse those two lines and the value can appear in a log before the
runner knows to redact it.

**`$GITHUB_ENV` is the substantive distractor.** It would set an environment variable for later steps
in the **same job**; `$GITHUB_OUTPUT` publishes a step output that other steps reference as
`steps['get-secret'].outputs['sql-connection']` (line 142) and that a job can expose to downstream jobs.

</details>

---

## Q41

```bash
az webapp config appsettings set \
  --name app-contoso-web \
  --resource-group rg-contoso-secrets \
  --settings "PaymentGateway__ApiKey=[BLANK 1](VaultName=kv-contoso-secrets-001;SecretName=[BLANK 2])"
```

- **BLANK 1:** `@Microsoft.KeyVault` / `$KeyVault` / `${{ keyvault }}` / `@AzureKeyVault`
- **BLANK 2:** `ApiKey-PaymentGateway` / `PaymentGateway__ApiKey` / `kv-contoso-secrets-001` /
  `latest`

<details>
<summary>Show answer</summary>

### Answer: `@Microsoft.KeyVault`, `ApiKey-PaymentGateway`

**In `challenge-42.md`:** line **261**.

**Two different names, and the exam swaps them.** `PaymentGateway__ApiKey` is what the **application**
reads — the .NET configuration key, where `__` becomes a nesting separator. `ApiKey-PaymentGateway` is
what the **vault** calls it (line 58). The reference maps one to the other.

**And no version is specified**, which is what lets the reference follow rotations (Q4).

</details>

---

# Section G — Case study

## Case study: Contoso secrets remediation

### Background

Contoso Ltd stores database connection strings and API keys as **plain-text pipeline variables** in
Azure Pipelines and GitHub Actions. A security audit found **34 secrets readable by all 150
contributors across 12 repositories**, and **three API keys already leaked into pipeline logs** through
accidental `echo` statements.

### Requirements

**Storage**

- All secrets must live in a central vault with per-identity, per-secret authorisation
- Pipelines must read only the secrets they use
- Where the platform allows it, no credential may be stored at all

**Runtime**

- The App Service must reach Azure SQL without a password
- A third-party payment key must be available to the app without passing through any pipeline
- A rotated secret must take effect without editing or redeploying any pipeline

**Operations**

- Rotation must be triggered by the platform, not by a calendar
- The organisation must be able to prove who read which secret and when
- Secrets must never be readable in logs

---

## Q42

Which vault configuration meets the Storage requirements?

- A. RBAC-enabled vault, Key Vault Secrets User per pipeline identity, `SecretsFilter` naming only the
  secrets used
- B. Access-policy vault with get and list granted to a shared service principal
- C. RBAC-enabled vault with Key Vault Administrator for every pipeline
- D. One vault per repository with access policies

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-42.md`:** lines **35–40**, **162**, **316–319**.

**Three requirements, three mechanisms:** RBAC gives per-identity and per-secret authorisation, Secrets
User is read-only, and the filter narrows what is actually fetched.

**Why the others fail**

- **B** — vault-level scope only (line 297), and a shared principal makes access unattributable
- **C** — a pipeline that can delete every secret in the vault
- **D** — twelve vaults multiply the management surface without gaining per-secret scoping, because
  access policies still cannot provide it

</details>

---

## Q43

How should the App Service reach Azure SQL?

- A. System-assigned managed identity with `Authentication=Active Directory Managed Identity`
- B. The connection string with credentials, stored in Key Vault and read at start-up
- C. A service principal with a certificate in Key Vault
- D. A contained database user whose password is rotated monthly by a function

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-42.md`:** lines **233–248** and **263–268**.

**"Without a password" is answered only by having no password.** Everything else stores one somewhere
and rotates it on some schedule.

**Why B is the most seductive wrong answer in the whole paper.** It is genuinely good practice, and it
is what half the industry does. It is rung 1 when rung 3 is available — and the requirement was written
to exclude it.

**One caveat the challenge itself raises** (lines 242–243): making the app identity the SQL **server AD
admin** is server-level access. The least-privilege form is a **contained database user mapped to the
managed identity** — same authentication, far smaller grant. Expect an exam option to test whether you
noticed.

</details>

---

## Q44

How should the third-party payment key reach the application?

- A. A Key Vault reference in the app setting, resolved by the app's managed identity
- B. Fetched by the pipeline and written as a plain app setting
- C. Stored as a GitHub Actions secret and injected at deploy time
- D. Committed to an encrypted file in the repository

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-42.md`:** lines **251–261**.

**The requirement says "without passing through any pipeline", and only A satisfies it literally.** The
pipeline writes a pointer; the app resolves the value.

**Why this key cannot be eliminated, and why that is the point.** It belongs to the payment gateway.
There is no managed identity to substitute — so the correct answer is the highest rung available for a
credential you do not control.

**Why B and C both fail the same way** — the value passes through the agent, and from there into
deployment history and anything that logs its inputs. That is how three keys ended up in logs already.

</details>

---

## Q45

Which **two** ensure a rotated secret takes effect without pipeline edits? (Choose two.)

- A. A variable group linked to Key Vault
- B. A Key Vault reference without a pinned version
- C. `AzureKeyVault@2` with the secret version ID specified
- D. Copying secrets into pipeline variables after each rotation
- E. Restarting the App Service on a schedule

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-42.md`:** lines **187–204** and **261**.

**Both resolve by name, so both follow the latest version.** A covers the pipeline path; B covers the
runtime path.

**Why C is the exact inverse** — pinning a version is what *prevents* the update, and it is the trap in
Q1.

**Why E is a workaround for a problem that does not exist.** References refresh on their own (Q4); a
scheduled restart adds downtime and hides the mechanism from whoever maintains it next.

</details>

---

## Q46

How should rotation be triggered?

- A. Expiry dates on secrets plus an Event Grid subscription to `Microsoft.KeyVault.SecretNearExpiry`
  invoking a rotation function
- B. A quarterly calendar reminder for the platform team
- C. A pipeline that rotates every secret weekly regardless of expiry
- D. Rotate only after a secret has expired and something breaks

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-42.md`:** lines **67–70** and **277–282**.

**The platform detects the deadline and calls the function.** Nobody has to remember, and nothing
depends on staffing.

**Why C is a real design that fails on cost and risk.** Rotating everything weekly means weekly
opportunities to break a consumer that cached a value, for secrets that may not need it — and it makes
the audit log so noisy that a genuine unauthorised change hides in it.

**Why D describes the starting state**, where a key expires and the outage is the notification.

</details>

---

## Q47

Six months after migration, a nightly pipeline that reads `StorageAccountKey` starts failing with 403.
Other pipelines reading the same vault still work, and nothing in the vault changed. The platform team
recently rebuilt the App Service and its agent pool infrastructure from Bicep.

What happened, and what is the fix?

- A. The identity was recreated, so its old role assignment no longer applies — reassign Key Vault
  Secrets User to the new principal
- B. The secret expired — rotate it
- C. The vault switched to access policies — reconfigure it
- D. `SecretsFilter` no longer matches — update it

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-42.md`:** lines **233–240**, **251–255**, **344–346**.

**"Other pipelines still work" tells you the vault is healthy.** The failure is specific to one
identity — and a rebuild is the classic way an identity changes underneath a role assignment.

**A system-assigned managed identity is destroyed and recreated with the resource** (line 233). New
principal ID, new object — and the role assignment still points at the old one, where it sits as an
orphaned entry that looks correct in a listing until you compare the GUIDs.

**This is the argument for a user-assigned managed identity** in infrastructure that gets rebuilt, from
Challenge 39: it survives the resource's replacement, and its role assignments survive with it.

**Why the others fail** — an expired secret gives a different error and would break every consumer; C
would break the pipelines that still work; D would produce a missing-variable failure, not a 403.

</details>

---

## Q48

The auditor asks Contoso to show who read the payment gateway key in the last 30 days, and to
demonstrate that a contributor could not read it today.

What can Contoso produce, and what does this illustrate?

- A. `AuditEvent` diagnostic logs in Log Analytics showing each read with its caller, plus RBAC
  assignments showing only the App Service identity holds Secrets User on that vault
- B. Nothing; Key Vault does not log reads
- C. The pipeline run logs only
- D. The list of secrets and their expiry dates

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-42.md`:** lines **285–289** and **316–325**.

```bash
  --logs '[{"category":"AuditEvent","enabled":true}]'
```

**Two questions, two controls, and the auditor is deliberately asking both.** "Who did read it" is
**detective** — the audit log. "Who could read it" is **preventive** — the role assignments. A design
with only one of them cannot answer.

**Before the migration, neither question had an answer.** 34 secrets in plain-text pipeline variables
readable by 150 contributors, with no record of access — so the honest response was *any of 150
people, and we cannot tell you whether they did.*

**What it illustrates: centralising secrets is not primarily about encryption.** The values were
already stored on Microsoft's infrastructure before the migration. What the vault adds is a
**chokepoint** — one place where access is granted, one place where it is recorded, and one place to
revoke it.

**And the strongest version of the answer goes further.** For SQL there is nothing to audit at all,
because there is no secret to read — the identity is the credential. **The best access log is the one
that has nothing to record.**

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **"Store it in Key Vault" when nothing needs storing** | Q2, Q18, Q25, Q43 | Managed identity and OIDC remove the credential |
| **Access policy on an RBAC vault** | Q5, Q16, Q23, Q27 | Policies are ignored. Check `enableRbacAuthorization` first |
| **Turning RBAC off to make a policy work** | Q16 | Downgrades the vault and loses per-secret scoping |
| **Key Vault Reader can read values** | Q3, Q27, Q31 | Metadata only. Secrets **User** reads values |
| **Administrator or Officer for a read-only pipeline** | Q3, Q25, Q42 | Secrets User. Officer can overwrite production |
| **Pinning a secret version** | Q1, Q45 | Freezes it. Rotation silently stops propagating |
| **No expiry date** | Q12, Q21, Q26, Q30, Q46 | No expiry, no `SecretNearExpiry`, no rotation |
| **Masking assumed automatic in GitHub Actions** | Q8, Q20, Q40 | Azure Pipelines masks known secrets; Actions needs `::add-mask::` |
| **Masking registered after printing** | Q28, Q40 | Not retroactive. Mask first |
| **Pipeline resolving the Key Vault reference** | Q14, Q22, Q29, Q44 | The **app** resolves it. That is the point |
| **`ManagedIdentity` as a principal type** | Q39 | It is a `ServicePrincipal` |
| **Deleting logs instead of rotating a leaked secret** | Q8 | Rotate. A deleted log is a deleted audit trail |
| **System-assigned identity surviving a rebuild** | Q47 | It does not. Use user-assigned for rebuilt infrastructure |
| **Logging offered in place of permissions** | Q30, Q35, Q48 | Detective ≠ preventive. You need both |

---

# What to memorise

**In `challenge-42.md`:** lines **294–312**, **67–70**, **261**, **268**.

```text
THE LADDER  (the exam wants the highest rung the scenario permits)
 0  secret in a plain-text pipeline variable        <- the audit finding, line 17
 1  secret in Key Vault, fetched by the pipeline
 2  secret in Key Vault, referenced by the APP      @Microsoft.KeyVault(...)
 3  NO SECRET EXISTS                                managed identity / OIDC

Third-party credential you cannot replace -> rung 2 is the ceiling.
Anything Azure can vouch for              -> rung 3.
```

```text
KEY VAULT RBAC ROLES          Officer = manage | User = use | Reader = see it exists
  Key Vault Administrator      everything in the vault              security team
  Key Vault Secrets Officer    create/read/update/delete secrets     rotation function
  Key Vault Secrets User       READ SECRET VALUES only               pipelines, apps
  Key Vault Reader             metadata only, NOT values             auditors
  Key Vault Certificates Officer / Crypto Officer   certificates / keys

RBAC vs ACCESS POLICIES
  scope        vault only            vs   subscription / RG / vault / INDIVIDUAL SECRET
  management   per-vault             vs   centralised Azure RBAC
  cond. access no                    vs   yes (Entra ID)
  recommended  legacy                vs   NEW DEPLOYMENTS
  On an RBAC vault, access policies are IGNORED - the 403 message still says "access policy"
```

```bash
# Vault                                            (lines 35-40)
az keyvault create --name kv-... --enable-rbac-authorization true --sku standard

# Expiry - without this there is no rotation event (lines 67-70)
az keyvault secret set-attributes --name "ApiKey-PaymentGateway" --expires "2025-12-31T23:59:59Z"

# Role for an app or pipeline identity             (lines 251-255)
az role assignment create --assignee-object-id $IDENTITY \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" --scope ".../vaults/kv-..."

# Rotation trigger + audit                         (lines 277-289)
--included-event-types "Microsoft.KeyVault.SecretNearExpiry"
--logs '[{"category":"AuditEvent","enabled":true}]'
```

```yaml
# Azure Pipelines - task                           (lines 158-163)
  - task: AzureKeyVault@2
    inputs:
      azureSubscription: 'Azure-Prod-Federated'
      KeyVaultName: 'kv-contoso-secrets-001'
      SecretsFilter: 'SqlConnectionString,ApiKey-PaymentGateway'   # name them, never '*'
      RunAsPreJob: true                                            # before the job's steps

# Azure Pipelines - variable group (reusable, resolves latest at run time)   (lines 203-204)
variables:
  - group: KeyVault-Production-Secrets
# secrets from a linked group are MASKED AUTOMATICALLY

# GitHub Actions - masking is NOT automatic        (lines 133-134)
echo "::add-mask::$SECRET"                 # mask FIRST
echo "sql-connection=$SECRET" >> $GITHUB_OUTPUT
```

```text
RUNTIME SECRETLESS                                 (lines 261, 268)
app setting        PaymentGateway__ApiKey=@Microsoft.KeyVault(VaultName=kv-...;SecretName=ApiKey-PaymentGateway)
                   ^ app config key                                              ^ VAULT secret name
                   no version pinned -> follows rotation, refreshes ~24h, resolved BY THE APP

connection string  Server=tcp:...;Database=ContosoDb;Authentication=Active Directory Managed Identity;Encrypt=true
                   ^ no password at all
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 43 |
| 38–43 | Re-read the trap index and the ladder, then move on |
| 30–37 | Rewrite the ladder and the role table from memory, then retake |
| Below 30 | Redo Tasks 1, 3 and 5 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 42.

:::danger The one question

**Does a credential still exist after this change?**

If yes, you are protecting a secret. If no, you have eliminated one.

The exam almost always wants the answer that eliminates — unless the credential belongs to a third
party, in which case put it in the vault and let the **app**, not the pipeline, resolve it.

:::
