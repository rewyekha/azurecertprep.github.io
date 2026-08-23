---
sidebar_position: 3.5
toc_max_heading_level: 2
title: "Challenge 41: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 41 — AZ-400 exam questions

**48 questions** built only from what Challenge 41 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-41.md`**.

:::danger Read this before you start

Azure DevOps security has **four separate dials**, and almost every wrong answer in this paper is the
right action on the wrong dial.

**Access level** — what a licence lets you *see*: Stakeholder, Basic, Basic + Test Plans.
**Security group** — what a *group of people* may do inside a project.
**Service connection** — what a *pipeline* may do inside Azure.
**Pipeline permissions** — *which pipelines* may use that connection at all.

A user needs all four to line up. Grant Basic but leave them out of a group and they see nothing.
Put them in the right group but leave the connection unauthorised and the pipeline fails with a
resource authorization error.

The scenario at line 19 is all four collapsed into one: everybody an administrator, one connection
for everything, and a PAT nobody owns.

:::

---

# Section A — Multiple choice

---

## Q1

Only the `Production-Deploy` pipeline may use the `Azure-Prod` service connection. What should you
configure?

- A. Leave "Grant access permission to all pipelines" enabled and add a YAML condition
- B. Disable "Grant access permission to all pipelines" and add only `Production-Deploy` to the
  connection's pipeline permissions
- C. Create a separate Azure DevOps project for production pipelines
- D. Use pipeline variables to control which stages reach the connection

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-41.md`:** lines **222–225**.

```bash
# Disable "Grant access permission to all pipelines"
az devops service-endpoint update \
  --id $ENDPOINT_ID \
  --enable-for-all false
```

**This is the built-in authorisation model**, and it is enforced by the service, not by the pipeline
author.

**Why A is the trap.** A YAML condition lives in a file **anyone with edit rights can change** — and it
is evaluated by the same pipeline it is supposed to restrain. Authorisation must be enforced somewhere
the pipeline cannot reach.

**Why the others fail**

- **C** — a whole project to solve a per-connection problem, and cross-project resource sharing brings
  its own permissions puzzle
- **D** — variables are inputs to a pipeline, not access control. A variable can be overridden at queue
  time

</details>

---

## Q2

Contoso has 200 users: 150 developers, 30 product managers who only need work items, and 20 executives
who view dashboards. Which assignment is most cost-effective?

- A. 200 Basic
- B. 150 Basic + 50 Stakeholder
- C. 200 Stakeholder
- D. 150 Basic + 30 Basic + 20 Stakeholder

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-41.md`:** lines **277–278**.

```text
| Stakeholder | Free | View backlogs, create work items, view dashboards, view wiki | Product owners, managers, executives |
| Basic | Paid (first 5 free) | Full Boards, Repos, Pipelines, Test Plans (limited) | Developers, testers, DevOps engineers |
```

**Match the capability list to the stated need, then take the cheapest that covers it.** Product
managers need work items; executives need dashboards. Stakeholder covers both, at zero cost.

**Why the others fail**

- **A** — 50 paid licences for people who never open Repos or Pipelines
- **C** — Stakeholder cannot use Repos or Pipelines. The 150 developers stop working
- **D** — the same as A, written to look different. 180 Basic licences

**Read option D carefully in the exam.** "150 Basic + 30 Basic" is a deliberate arithmetic trap; the
first two terms are both paid.

</details>

---

## Q3

An organisation policy must force all PATs to be project-scoped with a maximum lifetime of 90 days.
Where do you configure it?

- A. Project Settings > Policies
- B. Organization Settings > Policies
- C. Each user's Personal settings
- D. Microsoft Entra Conditional Access

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-41.md`:** lines **302–306**.

```text
1. Organization Settings > Policies (via web portal):
   - Restrict creation of full-scoped PATs: Enabled
   - Restrict creation of global PATs: Enabled (force project-scoped)
   - Enforce maximum PAT lifetime: 90 days
   - Restrict creation of admin-scope PATs: Enabled
```

**PAT policy is organisation-level and cannot be overridden per project.** That is the point — a
control a project administrator could relax would be no control at all, and the scenario at line 19
shows what happens when 200 people are project administrators.

**Why D is the intelligent wrong answer.** Conditional Access governs **sign-in** — device, location,
MFA. A PAT is a token already issued; Conditional Access does not set its scope or lifetime.

</details>

---

## Q4

A single pipeline deploys to development and production. Production needs release-manager approval.
What is correct?

- A. Two separate pipelines with different triggers
- B. One pipeline with stages, and an Approvals check on the **production environment**
- C. A manual intervention task in the YAML
- D. Branch policies requiring approval on `main`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-41.md`:** lines **246–252** and **266–269**.

```yaml
      - deployment: DeployToProd
        environment: 'production'
```

```text
1. Pipelines > Environments > production > Approvals and checks
2. Add "Approvals" with the Contoso-Release-Managers group
```

**The gate lives on the environment, not in the pipeline.** Any stage targeting `production` pauses —
including a pipeline written next year by someone who has never read this one.

**Why C is the near-miss that fails on reuse.** A manual intervention task gates *that pipeline*, and
only while nobody edits it out. The environment check is a **central, reusable control point**.

**Why D fails on target.** Branch policies gate **merging code**. They have nothing to say about
deploying it.

</details>

---

## Q5

After moving users out of Project Administrators into custom groups, they can no longer view work
items. Why?

- A. Their access level dropped to Stakeholder
- B. The custom groups do not inherit from the built-in Contributors group
- C. Work item permissions must be granted per user
- D. The project needs re-indexing

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-41.md`:** lines **380–394**.

```bash
az devops security group membership add \
  --group "vstfs:///Classification/TeamProject/<project-id>\\Contributors" \
  --member-id <custom-group-descriptor>
```

**A newly created custom group starts with nothing.** Project Administrators carried every permission
implicitly, so removing people from it removed permissions nobody had explicitly granted anywhere else.

**Nesting the custom group inside Contributors is the fix** — the group inherits the baseline, and you
then add or deny only what differs. Granting every permission individually to five groups is how the
matrix at lines 118–126 becomes unmaintainable.

**Why A is worth ruling out deliberately.** Access level and group membership are different dials
(see the box at the top). Moving groups does not change a licence.

</details>

---

## Q6

A new pipeline references `Azure-Prod` and fails with *"There was a resource authorization problem."*
What is the cause?

- A. The service principal lacks Contributor on the resource group
- B. The connection has "Grant access permission to all pipelines" disabled and this pipeline is not
  authorised
- C. The PAT expired
- D. The environment has no approvers

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-41.md`:** lines **360–362**.

**The error is about the pipeline's right to *use the connection*, not the connection's rights in
Azure.** Nothing has reached Azure yet.

**Learn to separate the two failures by where they occur.** A resource authorization problem happens
**before the job runs** — Azure DevOps refuses to hand the connection over. An RBAC failure
(`AuthorizationFailed`) happens **inside a task**, after a successful login, when Azure declines the
action.

**The fix** (lines 372–374): add the pipeline under the connection's Pipeline permissions, or click
**Permit** on the prompt if you have rights.

</details>

---

## Q7

Which service connection type removes the stored secret entirely?

- A. Azure Resource Manager with a service principal and key
- B. Workload identity federation (manual)
- C. Azure Resource Manager with a managed identity on a Microsoft-hosted agent
- D. A connection authenticated by a PAT

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-41.md`:** lines **194–201** and **211**.

```json
    "issuer": "https://vstoken.dev.azure.com/<org-guid>",
    "subject": "sc://contoso/ContosoWeb/Azure-Prod-Federated",
    "audiences": ["api://AzureADTokenExchange"]
```

**There is no secret to store, rotate, or leak.** Azure DevOps presents a token; Azure trusts it
because the issuer and subject match a federated credential.

**Why C is the Challenge 39 trap, repeated here.** A **Microsoft-hosted** agent is Microsoft's compute,
not your Azure resource — it has no managed identity endpoint. On a **self-hosted** agent inside an
Azure VM this would work, and that distinction is exactly what makes the option plausible.

**Why A is what Task 3 built and Task 4 replaces.** `az ad sp create-for-rbac` returns a key that
someone must paste into the connection — better than `Azure-All`, still a secret.

</details>

---

## Q8

What does the federated credential's `subject` identify?

- A. The Azure DevOps project
- B. The specific service connection
- C. The pipeline definition
- D. The branch being deployed

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-41.md`:** line **199**.

```json
    "subject": "sc://contoso/ContosoWeb/Azure-Prod-Federated",
```

**`sc://` means service connection**, followed by organisation, project and connection name.

**Azure DevOps binds trust to the connection rather than to a branch**, because the connection is the
object that already carries pipeline permissions and approvals. GitHub binds to `repo:...:ref:...`
instead, because a repository has no equivalent object.

**Which is why Q1 matters so much here.** The federated credential trusts the connection; the
connection's pipeline permissions decide who may use it. Leave `--enable-for-all` on and any pipeline
in the project inherits production access.

</details>

---

## Q9

Which is the correct order to remediate the `Azure-All` connection?

- A. Delete `Azure-All`, then create scoped connections
- B. Create scoped connections and resource groups, migrate pipelines, then delete `Azure-All`
- C. Reduce `Azure-All` to Reader and leave it in place
- D. Add pipeline permissions to `Azure-All` and keep using it

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-41.md`:** lines **132–180**.

**Build the replacement, migrate onto it, then remove the original.** Deleting first breaks every
pipeline in the organisation at once — and the outage will be blamed on the security work.

**Why C is a real-looking half-measure.** A Reader connection cannot deploy, so every pipeline breaks
anyway; and the underlying flaw is not the role, it is that **one identity spans all environments**.
Scoping to three resource groups is what stops a development pipeline touching production.

**Why D fails on the same point.** Pipeline permissions restrict *who calls* the connection. They do
nothing about *what it can reach* once called.

</details>

---

## Q10

Which permission does the matrix give Backend Developers on build pipelines?

- A. View and queue builds
- B. View, queue and edit build pipelines
- C. View only
- D. Full administration

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-41.md`:** lines **120–122**.

```text
| View builds | Yes | Yes | Yes | Yes |
| Queue builds | Yes | Yes | Yes | Yes |
| Edit build pipelines | No | No | Yes | No |
```

**Run it, do not rewrite it.** A developer who can edit a pipeline can add a step that uses a
production connection, so editing sits with DevOps Engineers.

**This is also what `--allow-bit 128` encodes at line 112** — the queue-build bit alone, against
`1535` for the DevOps Engineers group at line 104.

</details>

---

## Q11

Which group approves production releases in Contoso's design?

- A. Contoso-DevOps-Engineers
- B. Contoso-Release-Managers
- C. Contoso-Backend-Developers
- D. Project Administrators

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-41.md`:** lines **124** and **269**.

```text
| Approve releases | No | No | No | Yes |
```

**Look at the DevOps Engineers column, because that is where the exam sets its trap.** They can create
releases and manage service connections — and they **cannot approve**. Separating who builds the
release path from who authorises its use is the whole reason two groups exist.

**Why D fails on principle.** The remediation is to empty Project Administrators, not to route new
responsibilities into it.

</details>

---

## Q12

Which check restricts a production deployment to code from `main`?

- A. Approvals
- B. Branch control
- C. Business hours
- D. Required template

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-41.md`:** line **270**.

```text
3. Add "Branch control" to restrict to `refs/heads/main` only
```

**Branch control answers "where did this code come from?"** — and it is evaluated by the environment,
so it holds even if someone runs the pipeline manually from a feature branch.

**Why the others fail** — Approvals answers *who said yes* (line 269), Business hours answers *when*
(line 271), and Required template answers *what YAML was used*, which is a real check that this
challenge does not configure.

**All four are checks on the same environment**, and the exam expects you to pick the one matching the
stated constraint rather than the one you configured most recently.

</details>

---

## Q13

Which access level suits a dedicated QA engineer who manages test cases?

- A. Stakeholder
- B. Basic
- C. Basic + Test Plans
- D. Visual Studio subscription

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-41.md`:** lines **278–280**.

```text
| Basic | Paid (first 5 free) | Full Boards, Repos, Pipelines, Test Plans (limited) | Developers, testers, DevOps engineers |
| Basic + Test Plans | Paid add-on | Full Test Plans, test case management | Dedicated QA engineers |
```

**Basic includes Test Plans "(limited)".** The word in parentheses is the entire question — full test
case management needs the add-on.

**Why D is a distractor about *how you paid*, not *what you get*.** A Visual Studio subscription grants
**the same as Basic**, so it has the same limitation.

</details>

---

## Q14

Which command changes a user to Stakeholder?

- A. `az devops user update --user user@contoso.com --license-type stakeholder`
- B. `az devops security group membership add --group Stakeholders`
- C. `az devops user add --license-type express`
- D. `az devops security permission update --allow-bit 0`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-41.md`:** lines **286–289**.

**Access level is a licence property of the user account** — not a group, not a permission bit.

**Why B is the most tempting wrong answer**, and worth understanding rather than memorising: adding
someone to a group named "Stakeholders" changes their **permissions inside the project** while leaving
the paid licence assigned. The bill does not move.

**Why C is close but wrong** — `user add` invites a **new** user, and `express` is Basic (line 294).

</details>

---

## Q15

The CI integration's PAT was created by a former employee with full scope and no expiry. What is the
correct remediation?

- A. Extend its expiry and document the owner
- B. Revoke it, enable the organisation PAT policies, and move the integration to a service connection
  or service principal
- C. Share it with the whole DevOps team so it is not orphaned
- D. Recreate it under a shared account with the same scope

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-41.md`:** lines **19**, **302–306**, **316–318**.

```bash
curl -X DELETE -u :$ADMIN_PAT \
  "https://vssps.dev.azure.com/contoso/_apis/tokens/pats?authorizationId=<pat-auth-id>&api-version=7.1-preview.1"
```

**Three moves, because the finding has three parts:** the token itself, the policy that allowed it, and
the design that needed a user-owned credential for machine automation.

**Fix only the token and it recurs**, which is what makes B the complete answer and A, C, D all
variations on keeping it.

**Why C is worse than doing nothing.** A credential known to many people cannot be attributed to
anyone, so the audit trail stops meaning anything.

</details>

---

## Q16

Which namespace ID governs build permissions?

- A. `33344d9c-fc72-4d6f-aba5-fa317101a7e9`
- B. `2e9eb7ed-3c0a-47d4-87c1-0ffdd275fd87`
- C. `c788c23e-1b46-4162-8f5e-d7585343b5de`
- D. There is no namespace for builds

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-41.md`:** lines **94–97**.

```text
# Build: 33344d9c-fc72-4d6f-aba5-fa317101a7e9
# Git Repositories: 2e9eb7ed-3c0a-47d4-87c1-0ffdd275fd87
# ReleaseManagement: c788c23e-1b46-4162-8f5e-d7585343b5de
```

**Nobody memorises these GUIDs**, and the exam does not ask you to. What it does ask is that you know
**permissions are namespaced by resource type** and that you can discover the IDs:

```bash
az devops security permission namespace list --query "[].{name:name, id:namespaceId}" -o table
```

That command at line 92 is the answer to any "how would you find it" variant.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** checks can be added to the `production` environment? (Choose three.)

- A. Approvals
- B. Branch control
- C. Business hours
- D. Access level
- E. Namespace bits
- F. PAT lifetime

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-41.md`:** lines **266–271**.

```text
2. Add "Approvals" with the Contoso-Release-Managers group
3. Add "Branch control" to restrict to `refs/heads/main` only
4. Add "Business hours" to prevent deployments outside working hours
```

**Three questions, three checks: who, where from, and when.**

**Why the others fail — every one is a real control on a different dial.** Access level is a user
licence (line 277), namespace bits are group permissions (line 104), and PAT lifetime is an
organisation policy (line 305). None of them is evaluated at deployment time.

</details>

---

## Q18

Which **three** are true of the redesigned service connections? (Choose three.)

- A. Each is scoped to a single resource group
- B. Each uses its own service principal
- C. The production connection can be restricted to specific pipelines
- D. All three share one service principal for simpler rotation
- E. They are created at organisation level and shared by every project
- F. They replace the need for environment approvals

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-41.md`:** lines **139–152**, **222–225**.

```bash
az ad sp create-for-rbac \
  --name "sp-ado-contoso-prod" \
  --role "Contributor" \
  --scopes "/subscriptions/<sub-id>/resourceGroups/rg-contoso-prod"
```

**A and B together are what actually contains a compromise.** Separate principals mean the development
connection's credential is worthless against production — one scope, one identity, one blast radius.

**Why the others fail**

- **D** — a shared principal recreates `Azure-All` with extra steps
- **E** — service connections are **project** resources; sharing across projects is an explicit,
  separate grant
- **F** — different dial. The connection controls *what Azure permits*; approvals control *whether a
  human said yes*

</details>

---

## Q19

Which **three** organisation policies address the orphaned PAT? (Choose three.)

- A. Restrict creation of full-scoped PATs
- B. Restrict creation of global PATs
- C. Enforce maximum PAT lifetime
- D. Require MFA for all users
- E. Disable Basic authentication for Git
- F. Restrict service connection creation

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-41.md`:** lines **303–305**.

**Each maps to one attribute of the offending token** (line 19): full scope, global reach, no
expiration. Together they make that token impossible to create again.

**Why the others are good practice but off-target.** D and E harden authentication generally; F governs
a different resource. The exam is testing whether you can map a finding to the specific policy that
prevents it, not whether you can list security measures.

</details>

---

## Q20

Which **two** distinguish a Stakeholder from a Basic user? (Choose two.)

- A. Stakeholder cannot use Repos
- B. Stakeholder cannot use Pipelines beyond viewing
- C. Stakeholder cannot create work items
- D. Stakeholder cannot view dashboards
- E. Stakeholder costs more per user

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-41.md`:** lines **277–278**.

**Stakeholder gives the *work-tracking* half of Azure DevOps and withholds the *engineering* half.**

**Why C and D are the reversed facts.** Stakeholder explicitly **can** create work items and view
dashboards — that is the entire reason it fits 50 people in Q2. If Stakeholders could not create work
items the free tier would be useless to a product manager.

</details>

---

## Q21

Which **two** must be true before a pipeline can deploy using `Azure-Prod-Federated`? (Choose two.)

- A. The federated credential's subject matches `sc://contoso/ContosoWeb/Azure-Prod-Federated`
- B. The service principal has a role assignment at the target scope
- C. The pipeline stores the client secret as a variable
- D. The agent runs on an Azure VM
- E. The user queueing the pipeline is a Project Administrator

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-41.md`:** lines **194–208**.

```bash
az role assignment create \
  --assignee $APP_CLIENT_ID \
  --role "Contributor" \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-contoso-prod"
```

**Authentication and authorisation are separate steps, and both are required.** The federated
credential decides whether Azure will *believe* the token; the role assignment decides whether the
resulting identity may *do* anything.

**The two failure modes name themselves.** Wrong subject → the login fails. Missing role assignment →
the login succeeds and the task fails with `AuthorizationFailed`.

**Why the others fail** — C defeats the purpose, D is the managed-identity requirement (Q7), and E is
the model being dismantled.

</details>

---

## Q22

Which **two** actions correctly restructure the 200 Project Administrators? (Choose two.)

- A. Create per-team, per-role groups and add users to them
- B. Nest custom groups inside Contributors so they inherit baseline permissions
- C. Deny all permissions on Project Administrators and leave everyone in it
- D. Grant each user permissions individually
- E. Move everyone to Stakeholder access

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-41.md`:** lines **42–65** and **390–394**.

**A gives the structure; B makes it work.** Without B you get Break scenario 2: correct groups,
correct intent, and users who cannot see a work item.

**Why the others fail**

- **C** — an explicit **Deny wins over Allow** in Azure DevOps, so this would break people in
  unpredictable ways depending on their other group memberships. It also leaves the membership list
  looking wrong to an auditor
- **D** — 200 users' worth of individual grants, unauditable and unmaintainable. Groups exist for this
- **E** — destroys the developers' ability to work, and access level is the wrong dial anyway

</details>

---

## Q23

Which **two** does the weekly PAT audit pipeline do? (Choose two.)

- A. Lists PATs through the `tokens/pats` REST API
- B. Emits a warning for tokens expiring within 14 days
- C. Automatically revokes expired tokens
- D. Rotates tokens and updates variable groups
- E. Blocks pipelines that use PATs

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-41.md`:** lines **343–352**.

```powershell
        $expiringSoon = $response.patTokens | Where-Object {
          $_.validTo -lt (Get-Date).AddDays(14) -and $_.validTo -gt (Get-Date)
        }
        if ($expiringSoon) {
          Write-Host "##vso[task.logissue type=warning]PATs expiring within 14 days:"
```

**Note the two-sided filter.** `-lt` 14 days from now **and** `-gt` now — so already-expired tokens are
excluded. The audit is about tokens that are *about to* break something, not ones that already have.

**And `##vso[task.logissue type=warning]` warns without failing the run** (line 349). A monitoring job
that fails on findings gets muted within a month.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must implement least privilege across a 200-user Azure DevOps organisation.
Developers must run pipelines but not edit them; only release managers may approve production; each
environment must have its own Azure credential; production deployments must come from `main`; and no
user-owned PAT may hold full scope indefinitely.

---

## Q24

**Proposed solution:** Create per-team groups nested inside Contributors, granting developers the
queue-build bit and DevOps Engineers full build permissions. Create three service principals scoped to
one resource group each and a service connection per environment. Disable "Grant access permission to
all pipelines" on the production connection and authorise only the production pipeline. Add Approvals
with Contoso-Release-Managers and Branch control on the `production` environment. Enable the
organisation PAT policies.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-41.md`:** lines **42–65**, **112**, **139–152**, **222–225**, **266–270**, **302–306**,
**390–394**.

| Requirement | Mechanism |
|---|---|
| Run but not edit pipelines | `--allow-bit 128` for developers |
| Only release managers approve | Approvals check bound to that group |
| Per-environment credentials | Three SPs, three resource-group scopes |
| Production from `main` only | Branch control on the environment |
| No indefinite full-scope PATs | Organisation policies |
| Users can still see work items | Custom groups nested in Contributors |

**The last row is the one candidates leave out**, and it is the difference between a design that is
correct on paper and one that survives contact with 200 users on Monday morning.

</details>

---

## Q25

**Proposed solution:** Create per-team groups. Keep the single `Azure-All` connection but disable
"Grant access permission to all pipelines" and authorise each pipeline individually. Add a manual
intervention task to the production stage. Extend the existing PAT's expiry to 12 months.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Three failures, one per requirement.**

**`Azure-All` still holds Contributor on the entire production subscription.** Pipeline permissions
control *who may call* the connection; they do not shrink *what it reaches* once called. A development
pipeline that is authorised — perhaps by mistake, perhaps deliberately — can delete production.

**A manual intervention task is not an approval gate.** It lives in the YAML, so anyone who can edit
the pipeline can remove it, and it does not bind approval to Contoso-Release-Managers. Q4's lesson.

**Extending the PAT keeps a full-scope, user-owned credential belonging to someone who left.** The
expiry was never the whole problem.

</details>

---

## Q26

**Proposed solution:** Create per-team groups nested inside Contributors with the correct permission
bits. Create three scoped service principals and three service connections. Add Approvals and Branch
control on the `production` environment. Enable the organisation PAT policies. Leave "Grant access
permission to all pipelines" enabled on all three connections so teams are never blocked.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**With `--enable-for-all` left on, every pipeline in the project can reference `Azure-Prod`.** A
developer who can create a pipeline — and creating is not the same as editing an existing one — can
write a stage that uses the production connection and deploys straight into `rg-contoso-prod`.

**The environment checks do not save you.** Approvals and Branch control fire on stages that declare
`environment: 'production'`. A stage that simply runs `AzureCLI@2` against `Azure-Prod` without naming
an environment never touches them.

**That is the sharpest lesson in this challenge.** Environment checks gate **deployments**;
pipeline permissions gate **the credential**. You need both, and only one of them is visible in the
YAML.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — groups and permissions

| # | Statement | Answer |
|---|---|---|
| 1 | A new custom group starts with no permissions |  |
| 2 | Nesting a custom group in Contributors grants it baseline access |  |
| 3 | Backend Developers can edit build pipelines |  |
| 4 | Permissions are namespaced by resource type |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A new custom group starts with no permissions | **Yes** |
| 2 | Nesting a custom group in Contributors grants it baseline access | **Yes** |
| 3 | Backend Developers can edit build pipelines | **No** |
| 4 | Permissions are namespaced by resource type | **Yes** |

**In `challenge-41.md`:** lines **380–394**, **122**, **94–97**.

Row 1 and row 2 are Break scenario 2 stated forwards. **Create a group, grant nothing, wonder why
nobody can work.**

Row 3 is the matrix: queue yes, edit no.

</details>

---

## Q28 — service connections

| # | Statement | Answer |
|---|---|---|
| 1 | A service connection is a project-level resource |  |
| 2 | Disabling "grant to all pipelines" requires explicit authorisation per pipeline |  |
| 3 | A federated connection stores a client secret |  |
| 4 | Pipeline permissions limit what the connection can do in Azure |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A service connection is a project-level resource | **Yes** |
| 2 | Disabling "grant to all pipelines" requires explicit authorisation per pipeline | **Yes** |
| 3 | A federated connection stores a client secret | **No** |
| 4 | Pipeline permissions limit what the connection can do in Azure | **No** |

**In `challenge-41.md`:** lines **159**, **222–225**, **194–201**.

Row 4 is Q25's failure written as a single statement. **Pipeline permissions answer *who calls it*.
RBAC scope answers *what it can reach*.** Confusing the two is the most expensive mistake in this
challenge.

</details>

---

## Q29 — environments and approvals

| # | Statement | Answer |
|---|---|---|
| 1 | Approvals configured on an environment apply to any pipeline targeting it |  |
| 2 | Branch control restricts which branch may deploy |  |
| 3 | A stage that never names an environment is still gated by its checks |  |
| 4 | Approvals are declared in the pipeline YAML |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Approvals configured on an environment apply to any pipeline targeting it | **Yes** |
| 2 | Branch control restricts which branch may deploy | **Yes** |
| 3 | A stage that never names an environment is still gated by its checks | **No** |
| 4 | Approvals are declared in the pipeline YAML | **No** |

**In `challenge-41.md`:** lines **252**, **266–271**.

Row 3 is the gap Q26 walks through. Row 4 recurs from Challenges 24, 37 and 38: **the YAML names the
environment; the checks live on the environment**.

Rows 1 and 4 together are why this design is durable — the control is not in a file that a pipeline
author can edit.

</details>

---

## Q30 — access levels and PATs

| # | Statement | Answer |
|---|---|---|
| 1 | Stakeholder access is free |  |
| 2 | Stakeholder users can push to Repos |  |
| 3 | PAT maximum lifetime is set per project |  |
| 4 | A Visual Studio subscription grants the same capabilities as Basic |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Stakeholder access is free | **Yes** |
| 2 | Stakeholder users can push to Repos | **No** |
| 3 | PAT maximum lifetime is set per project | **No** |
| 4 | A Visual Studio subscription grants the same capabilities as Basic | **Yes** |

**In `challenge-41.md`:** lines **277–280**, **302–305**.

Row 3 is Q3: **organisation-level, and deliberately not overridable**.

Row 4 is the distinction between *entitlement* and *capability* — the subscription changes who pays,
not what the user can do.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each requirement to the correct control.

| Requirement | Control |
|---|---|
| Only release managers may approve production |  |
| Only `main` may deploy to production |  |
| Only one pipeline may use `Azure-Prod` |  |
| A development pipeline must not reach production resources |  |
| Product managers must not consume paid licences |  |
| Developers must run but not edit pipelines |  |

**Options:** Approvals check on the environment · Branch control check · Build namespace permission bits · Pipeline permissions on the connection · RBAC scope on the service principal · Stakeholder access level

<details>
<summary>Show answer</summary>

| Requirement | Control |
|---|---|
| Only release managers may approve production | **Approvals check on the environment** |
| Only `main` may deploy to production | **Branch control check** |
| Only one pipeline may use `Azure-Prod` | **Pipeline permissions on the connection** |
| A development pipeline must not reach production resources | **RBAC scope on the service principal** |
| Product managers must not consume paid licences | **Stakeholder access level** |
| Developers must run but not edit pipelines | **Build namespace permission bits** |

**In `challenge-41.md`:** lines **269**, **270**, **222–225**, **142–152**, **277**, **112**.

**Six requirements, six different dials.** If you can place these six without hesitating, you can
answer almost any Azure DevOps security question the exam asks — because every one of them is a
variation on "which dial does this?"

</details>

---

## Q32

Match each Contoso group to its defining capability.

| Group | Capability |
|---|---|
| Contoso-Backend-Developers |  |
| Contoso-DevOps-Engineers |  |
| Contoso-Release-Managers |  |
| Contoso-Stakeholders |  |
| Project Administrators |  |

**Options:** Approve production releases · Edit pipelines and manage service connections · Queue builds, no editing · Should hold almost nobody · View work items and dashboards

<details>
<summary>Show answer</summary>

| Group | Capability |
|---|---|
| Contoso-Backend-Developers | **Queue builds, no editing** |
| Contoso-DevOps-Engineers | **Edit pipelines and manage service connections** |
| Contoso-Release-Managers | **Approve production releases** |
| Contoso-Stakeholders | **View work items and dashboards** |
| Project Administrators | **Should hold almost nobody** |

**In `challenge-41.md`:** lines **118–126** and **63–65**.

**Read the two middle rows as a pair.** DevOps Engineers **build the road**; Release Managers **decide
who drives on it**. Neither can do the other's job, and that separation is what the exam is testing
when it offers "give DevOps Engineers approval rights so they can unblock deployments."

</details>

---

## Q33

Arrange the steps to replace `Azure-All` with least-privilege service connections.

**Items:** Delete `Azure-All` · Create a resource group per environment · Update pipelines to reference
the new connections · Create a service principal scoped to each resource group · Create a service
connection per environment · Disable "grant access to all pipelines" and authorise specific pipelines

<details>
<summary>Show answer</summary>

### Answer

1. Create a resource group per environment — lines **134–136**
2. Create a service principal scoped to each resource group — lines **139–152**
3. Create a service connection per environment — lines **159–179**
4. Disable "grant access to all pipelines" and authorise specific pipelines — lines **222–225**
5. Update pipelines to reference the new connections — line **259**
6. Delete `Azure-All` — line **451**

**Deleting last is the point of the question.** The old connection stays live until every pipeline has
moved; remove it first and the entire organisation stops deploying.

**Step 4 before step 5 is the other detail worth noticing.** Lock the connection down *before* pipelines
start using it, so the first pipeline authorised is the one you intended — rather than opening it wide
and trying to narrow it once teams depend on it.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| "There was a resource authorization problem" |  |
| `AuthorizationFailed` inside an `AzureCLI@2` task |  |
| Users cannot see work items after restructuring |  |
| Login fails on a federated connection |  |
| A user cannot open Repos |  |

**Options:** Custom group not nested in Contributors · Pipeline not authorised for the connection · Service principal has no role at that scope · Stakeholder access level · Subject does not match the service connection

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| "There was a resource authorization problem" | **Pipeline not authorised for the connection** |
| `AuthorizationFailed` inside an `AzureCLI@2` task | **Service principal has no role at that scope** |
| Users cannot see work items after restructuring | **Custom group not nested in Contributors** |
| Login fails on a federated connection | **Subject does not match the service connection** |
| A user cannot open Repos | **Stakeholder access level** |

**In `challenge-41.md`:** lines **362**, **205–208**, **382**, **199**, **277**.

**The first two are the pair to keep straight, and they are separated by *when* they happen.** A
resource authorization problem occurs **before the job starts** — Azure DevOps will not release the
connection. `AuthorizationFailed` occurs **inside the task**, after a successful login, when Azure
declines the action.

**The last row is a reminder that not every access problem is a permission problem.** Check the licence
before you go looking at namespace bits.

</details>

---

## Q35

Match each policy to what it prevents.

| Policy | Prevents |
|---|---|
| Restrict full-scoped PATs |  |
| Restrict global PATs |  |
| Enforce maximum PAT lifetime |  |
| Disable "grant access to all pipelines" |  |
| Scope the service principal to a resource group |  |

**Options:** A connection reaching the whole subscription · A token spanning every project · A token that can do everything · A token that never expires · Any pipeline using a production connection

<details>
<summary>Show answer</summary>

| Policy | Prevents |
|---|---|
| Restrict full-scoped PATs | **A token that can do everything** |
| Restrict global PATs | **A token spanning every project** |
| Enforce maximum PAT lifetime | **A token that never expires** |
| Disable "grant access to all pipelines" | **Any pipeline using a production connection** |
| Scope the service principal to a resource group | **A connection reaching the whole subscription** |

**In `challenge-41.md`:** lines **303–305**, **225**, **142**.

**The top three describe the exact token in line 19** — full scope, organisation-wide, no expiration.
The bottom two describe `Azure-All`. Between them they are the entire remediation, and the exam case
study is built from precisely this list.

</details>

---

# Section F — Hot area

---

## Q36

```bash
az devops service-endpoint update \
  --id $ENDPOINT_ID \
  --[BLANK 1] [BLANK 2]
```

Requirement: only named pipelines may use this service connection.

- **BLANK 1:** `enable-for-all` / `authorize-all` / `restrict-access` / `pipeline-permissions`
- **BLANK 2:** `false` / `true` / `restricted` / `none`

<details>
<summary>Show answer</summary>

### Answer: `enable-for-all`, `false`

**In `challenge-41.md`:** lines **223–225**.

**Setting it to `false` does not name the allowed pipelines** — it closes the door, and each pipeline
must then be added explicitly under Pipeline permissions (line 373).

**Which is why Break scenario 1 exists.** The very next pipeline someone writes fails with a resource
authorization error, and that failure is the control working, not a bug.

</details>

---

## Q37

```bash
az ad sp create-for-rbac \
  --name "sp-ado-contoso-prod" \
  --role "[BLANK 1]" \
  --scopes "[BLANK 2]"
```

Requirement: deploy resources to production only, with least privilege.

- **BLANK 1:** `Contributor` / `Owner` / `Reader` / `User Access Administrator`
- **BLANK 2:** `/subscriptions/<sub-id>/resourceGroups/rg-contoso-prod` / `/subscriptions/<sub-id>` /
  `/` / `/subscriptions/<sub-id>/resourceGroups`

<details>
<summary>Show answer</summary>

### Answer: `Contributor`, the **resource group** scope

**In `challenge-41.md`:** lines **149–152**.

**Contributor is the least privilege that can still deploy.** Reader cannot create anything; Owner adds
the ability to grant roles, so a compromised connection could escalate itself.

**The scope is where `Azure-All` went wrong** (line 19): Contributor on the entire production
subscription. The role was defensible; the scope was not.

</details>

---

## Q38

```json
{
  "name": "ado-contoso-prod-connection",
  "issuer": "[BLANK 1]",
  "subject": "[BLANK 2]",
  "audiences": ["[BLANK 3]"]
}
```

- **BLANK 1:** `https://vstoken.dev.azure.com/<org-guid>` /
  `https://token.actions.githubusercontent.com` / `https://dev.azure.com/contoso` /
  `https://login.microsoftonline.com`
- **BLANK 2:** `sc://contoso/ContosoWeb/Azure-Prod-Federated` / `repo:contoso/ContosoWeb:ref:main` /
  `pipeline://contoso/Production-Deploy` / `contoso/ContosoWeb`
- **BLANK 3:** `api://AzureADTokenExchange` / `https://management.azure.com` / `azure-devops` /
  `https://vstoken.dev.azure.com`

<details>
<summary>Show answer</summary>

### Answer: `https://vstoken.dev.azure.com/<org-guid>`,
`sc://contoso/ContosoWeb/Azure-Prod-Federated`, `api://AzureADTokenExchange`

**In `challenge-41.md`:** lines **197–200**.

**Issuer names the platform, subject names the exact object, audience is always
`api://AzureADTokenExchange`.**

**The second option in each list is the GitHub form** — `token.actions.githubusercontent.com` and
`repo:...` — and mixing the two platforms is the most common error on this question. Azure DevOps
trusts a **service connection**; GitHub trusts a **repository and ref**.

</details>

---

## Q39

```bash
az devops security permission update \
  --namespace-id 33344d9c-fc72-4d6f-aba5-fa317101a7e9 \
  --subject <backend-developers-group-descriptor> \
  --token "<project-id>" \
  --allow-bit [BLANK 1] \
  --deny-bit [BLANK 2]
```

Requirement: developers may queue builds but not edit pipelines.

- **BLANK 1:** `128` / `1535` / `1` / `0`
- **BLANK 2:** `0` / `128` / `1535` / `1`

<details>
<summary>Show answer</summary>

### Answer: `128`, `0`

**In `challenge-41.md`:** lines **108–113**.

```bash
  --allow-bit 128 \
  --deny-bit 0
```

**Allow only what is needed and deny nothing.** `1535` is the DevOps Engineers value at line 104 — the
full build permission set.

**Why `--deny-bit 0` matters more than it looks.** In Azure DevOps an explicit **Deny beats Allow**
across every group a user belongs to. Denying here would override permissions the same person legitimately
holds through another group, producing failures that are very hard to trace. **Grant narrowly; deny
almost never.**

</details>

---

## Q40

```bash
az devops user update \
  --user pm@contoso.com \
  --license-type [BLANK 1]

az devops user add \
  --email-id newdev@contoso.com \
  --license-type [BLANK 2]
```

Requirement: the product manager needs work items only; the new developer needs full Repos and
Pipelines.

- **BLANK 1:** `stakeholder` / `express` / `advanced` / `none`
- **BLANK 2:** `express` / `stakeholder` / `basic` / `professional`

<details>
<summary>Show answer</summary>

### Answer: `stakeholder`, `express`

**In `challenge-41.md`:** lines **287–295**.

**`express` is the CLI name for Basic.** That mismatch between the portal label and the CLI value is
exactly the kind of detail the exam uses, and the only defence is having typed the command once.

**`advanced` is Basic + Test Plans** — the QA add-on from Q13, not what a new developer needs.

</details>

---

## Q41

```yaml
  - stage: DeployProd
    dependsOn: Build
    jobs:
      - [BLANK 1]: DeployToProd
        [BLANK 2]: 'production'
        strategy:
          [BLANK 3]:
            deploy:
              steps:
                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'Azure-Prod-Federated'
```

- **BLANK 1:** `deployment` / `job` / `stage` / `template`
- **BLANK 2:** `environment` / `pool` / `resource` / `target`
- **BLANK 3:** `runOnce` / `rolling` / `canary` / `matrix`

<details>
<summary>Show answer</summary>

### Answer: `deployment`, `environment`, `runOnce`

**In `challenge-41.md`:** lines **249–256**.

**`deployment` rather than `job` is what unlocks everything else.** Only a deployment job can declare
`environment:`, and only a job bound to an environment is subject to that environment's approvals and
branch control.

**Write `job:` here and the pipeline still runs** — straight past every check, exactly as in Q26. It is
a silent failure, not a syntax error, which is what makes it worth memorising.

**`strategy: runOnce` is the simplest lifecycle**, and it is required syntax on a deployment job even
when there is only one step.

</details>

---

# Section G — Case study

## Case study: Contoso Azure DevOps remediation

### Background

Contoso Ltd's Azure DevOps organisation grew organically over three years. **All 200 users are members
of Project Administrators** because "it was easier." Every pipeline uses one service connection,
`Azure-All`, with **Contributor on the entire production subscription**. The PAT used by the CI
integration was created by a **former employee**, with **full scope and no expiration**.

### Requirements

**People**

- 150 developers need Repos, Pipelines and Boards; they must run pipelines but not edit them
- 30 product managers and 20 executives need work items and dashboards only, at no licence cost
- A platform team manages pipelines and service connections
- A release management team, and only that team, approves production

**Credentials**

- Each environment must have its own Azure credential, scoped to its own resource group
- The production connection must not carry a stored secret
- Only the production deployment pipeline may use the production connection

**Governance**

- No PAT may be created with full scope, global reach, or an unlimited lifetime
- Production deployments must originate from `main`
- The organisation must be able to report on PATs approaching expiry

---

## Q42

How should the 200 users be restructured?

- A. Five per-team, per-role groups nested inside Contributors, with Project Administrators reduced to
  a handful of platform staff
- B. Keep Project Administrators and add Deny entries for risky permissions
- C. Grant permissions individually to each of the 200 users
- D. Move everyone to Stakeholder access

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-41.md`:** lines **42–65** and **390–394**.

**Groups by team and role, nested for the baseline.** The nesting is what prevents Break scenario 2 —
without it, day one of the new model is 200 people who cannot see a work item.

**Why the others fail**

- **B** — Deny overrides Allow everywhere, so it breaks people unpredictably depending on their other
  memberships, and the membership list still reads "200 administrators" to an auditor
- **C** — unauditable and unmaintainable at 200 users
- **D** — wrong dial, and it stops 150 developers working

</details>

---

## Q43

Which credential design meets the Credentials requirements?

- A. Three service principals scoped to one resource group each, with the production connection using
  workload identity federation
- B. One service principal with Contributor on the subscription, used by three connections
- C. Three service principals with Owner on the subscription
- D. A PAT-authenticated connection per environment

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-41.md`:** lines **139–152** and **188–208**.

**Three requirements, and only A satisfies all three:** separate credentials, resource-group scope, and
no stored secret on production.

**Why the others fail**

- **B** — three connections sharing one identity is `Azure-All` with three names. A compromise anywhere
  reaches everywhere
- **C** — Owner is broader than Contributor, not narrower, and subscription scope is the finding
- **D** — a PAT authenticates to **Azure DevOps**, not to Azure. It is the wrong credential for the
  wrong boundary, and it is the object being removed

</details>

---

## Q44

How do you ensure only the production deployment pipeline uses `Azure-Prod-Federated`?

- A. Disable "Grant access permission to all pipelines" and add that pipeline to the connection's
  pipeline permissions
- B. Add an `if` condition to the stage
- C. Rely on the environment's Approvals check
- D. Store the connection name in a secret variable

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-41.md`:** lines **222–225** and **372–373**.

**Why C is the most instructive wrong answer**, and worth spending a moment on: Approvals gate a
**deployment job that names the environment**. Any stage can call `AzureCLI@2` with
`azureSubscription: 'Azure-Prod-Federated'` and never declare an environment at all — no approval, no
branch control, full production access.

**Only pipeline permissions gate the credential itself**, which is why this control and the environment
checks are complementary rather than alternatives.

**Why D is not a control.** A pipeline author can read the variable, and the connection name was never
the secret.

</details>

---

## Q45

Which **two** governance settings meet the PAT requirements? (Choose two.)

- A. Restrict creation of full-scoped and global PATs
- B. Enforce a maximum PAT lifetime of 90 days
- C. Ask each team to document its PATs in the wiki
- D. Rotate the existing PAT annually
- E. Share one audited PAT across all integrations

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-41.md`:** lines **303–305**.

**These are enforced by the platform.** Nothing depends on anyone remembering.

**Why the others fail**

- **C** — documentation is not a control; it records what happened rather than preventing it
- **D** — keeps a full-scope, user-owned token in service
- **E** — a shared credential cannot be attributed to any caller, which destroys the audit trail the
  requirement exists to protect

</details>

---

## Q46

How should the organisation report on PATs approaching expiry?

- A. A scheduled pipeline calling the `tokens/pats` REST API, warning on tokens expiring within 14 days
- B. A monthly manual review of Organization Settings
- C. Wait for pipelines to fail and investigate
- D. Set all PATs to the same expiry date so they are easy to remember

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-41.md`:** lines **325–353**.

```yaml
schedules:
  - cron: "0 8 * * 1"
    displayName: Weekly PAT audit
```

**Monday at 08:00 — before the working week, not after something breaks.**

**Why D is worth rejecting explicitly.** Synchronising every expiry converts many small, absorbable
failures into one simultaneous outage across every integration at once.

**And note the warning rather than a failure** (line 349): `##vso[task.logissue type=warning]` surfaces
the finding without failing the run, which is what keeps an audit job alive long enough to be useful.

</details>

---

## Q47

Three months after the redesign, a developer reports that a **newly created** pipeline fails
immediately with *"There was a resource authorization problem"* when referencing `Azure-Dev`. Other
pipelines using `Azure-Dev` work. Nothing in Azure changed.

What happened, and what is the correct response?

- A. The new pipeline was never authorised for the connection — add it under Pipeline permissions, or
  click Permit
- B. The service principal's role assignment was removed — recreate it
- C. The federated credential's subject no longer matches — update it
- D. Re-enable "Grant access permission to all pipelines" so this never happens again

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-41.md`:** lines **360–374**.

**"Other pipelines work" is the diagnostic sentence.** The connection is healthy and its Azure
permissions are intact; the only thing missing is this pipeline's authorisation.

**Why D is the answer that will be suggested in the room, and why to refuse it.** It resolves the
symptom by removing the control — and it re-opens the path in Q26, where any pipeline can reach any
environment. The friction here is a **one-click Permit by someone with authority**, which is the design
working exactly as intended.

**Why the others fail** — B produces `AuthorizationFailed` inside the task, after a successful login;
C is a federated-connection failure and `Azure-Dev` uses a service principal (line 159), and it would
fail at login rather than before the job starts.

</details>

---

## Q48

The security team asks Contoso to demonstrate that a production deployment could not have been
performed by a developer acting alone.

What can Contoso now show, and what does this illustrate?

- A. Four independent controls that would each have stopped it: pipeline permissions on the connection,
  the environment approval bound to Release Managers, branch control on `main`, and the production
  service principal's scope
- B. Nothing; Azure DevOps does not record deployment authorisation
- C. The pipeline run log only
- D. The PAT audit report

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-41.md`:** lines **222–225**, **269**, **270**, **149–152**.

**Walk the four controls in the order an attempt would hit them.**

The developer writes a pipeline referencing `Azure-Prod-Federated`. **Pipeline permissions** refuse it
before the job starts. Suppose it were authorised: the deployment job names `production`, so
**Approvals** pauses for a Release Manager — a group the developer is not in. Suppose an approval were
obtained: **Branch control** rejects the run unless the code is on `main`, which is itself protected.
And if all three were somehow satisfied, the production service principal is **scoped to
`rg-contoso-prod`** — so the damage is bounded to one resource group rather than the subscription.

**What it illustrates: least privilege is layered, not singular.** Each control is individually
bypassable by someone with enough rights. The design's value is that **no single person holds all four
sets of rights** — the platform team manages connections but cannot approve, release managers approve
but cannot edit pipelines, and developers can do neither.

**Compare with the starting state at line 19**, where the honest answer to the security team's question
was: *any of our 200 project administrators could have done it, using a connection with Contributor on
the whole subscription, and we could not prove which one.*

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **YAML condition as an access control** | Q1, Q25, Q44 | Anyone who edits the file removes it. Control must sit outside the pipeline |
| **Pipeline permissions confused with RBAC scope** | Q9, Q25, Q28, Q44 | *Who may call it* vs *what it may reach*. Both are needed |
| **Environment checks assumed to gate every stage** | Q26, Q29, Q41, Q44 | Only stages that declare `environment:` |
| **`job:` instead of `deployment:`** | Q41 | No environment, no checks, and no error |
| **Manual intervention instead of an Approvals check** | Q4, Q25 | Lives in YAML, not bound to a group |
| **"150 Basic + 30 Basic"** | Q2 | Read the arithmetic. That is 180 paid licences |
| **Stakeholder cannot create work items** | Q20 | It can. That is why it fits 50 people |
| **Custom groups inherit permissions automatically** | Q5, Q22, Q27, Q42 | They inherit nothing. Nest inside Contributors |
| **Deny used to restrict** | Q22, Q39 | Deny beats Allow everywhere. Grant narrowly instead |
| **PAT policy at project level** | Q3, Q30 | Organisation Settings, not overridable |
| **Managed identity on a Microsoft-hosted agent** | Q7 | Not your compute. Self-hosted on an Azure VM would work |
| **GitHub issuer/subject on an Azure DevOps credential** | Q38 | `vstoken.dev.azure.com` and `sc://` |
| **Deleting `Azure-All` first** | Q9, Q33 | Build, migrate, then delete |
| **Access level mistaken for a permission problem** | Q5, Q34 | Check the licence before the namespace bits |

---

# What to memorise

**In `challenge-41.md`:** lines **118–126**, **275–280**, **302–306**.

```text
THE FOUR DIALS
Access level        what the LICENCE lets you see      Stakeholder | Basic | Basic+Test Plans
Security group      what a GROUP may do in a project   namespace bits, nested in Contributors
Service connection  what a PIPELINE may do in Azure    service principal + RBAC scope
Pipeline permission WHICH pipelines may use it         --enable-for-all false

Wrong dial = wrong answer. Almost every distractor in this domain is a real control, misapplied.
```

```text
PERMISSION MATRIX                Backend  Frontend  DevOps  Release
View builds                        Y        Y         Y       Y
Queue builds                       Y        Y         Y       Y
Edit build pipelines               N        N         Y       N
Create releases                    N        N         Y       Y
Approve releases                   N        N         N       Y     <- ONLY Release Managers
Manage service connections         N        N         Y       N
Edit project settings              N        N         Y       N

ACCESS LEVELS
Stakeholder            FREE     work items, dashboards, wiki      PMs, executives
Basic                  paid     Boards, Repos, Pipelines, Test Plans (LIMITED)
Basic + Test Plans     add-on   full test case management         dedicated QA
Visual Studio sub      incl.    same as Basic
CLI names:  stakeholder | express (= Basic) | advanced (= Basic + Test Plans)
```

```bash
# Lock a service connection to named pipelines   (lines 222-225)
az devops service-endpoint update --id $ENDPOINT_ID --enable-for-all false
# then Project Settings > Service connections > <name> > Security > Pipeline permissions > +

# Least-privilege service principal              (lines 149-152)
az ad sp create-for-rbac --name "sp-ado-contoso-prod" \
  --role "Contributor" \
  --scopes "/subscriptions/<sub-id>/resourceGroups/rg-contoso-prod"   # RG, never subscription

# Federated (no secret) service connection       (lines 194-201)
  "issuer":    "https://vstoken.dev.azure.com/<org-guid>"
  "subject":   "sc://<org>/<project>/<connection-name>"
  "audiences": ["api://AzureADTokenExchange"]
# then: az ad sp create --id $APP_CLIENT_ID   +   az role assignment create --scope <RG>

# Permissions                                    (lines 108-113)
--allow-bit 128    queue builds only (developers)
--allow-bit 1535   full build permissions (DevOps Engineers)
--deny-bit 0       Deny beats Allow EVERYWHERE - grant narrowly, deny almost never

# Access levels                                  (lines 287-295)
az devops user update --user <upn> --license-type stakeholder
az devops user add --email-id <upn> --license-type express      # express = Basic
```

```yaml
# The gated deployment                           (lines 246-263)
  - stage: DeployProd
    jobs:
      - deployment: DeployToProd        # deployment, NOT job - or no checks apply
        environment: 'production'       # the checks live HERE, not in this file
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'Azure-Prod-Federated'

# On the environment (portal):  Approvals -> Release Managers
#                               Branch control -> refs/heads/main
#                               Business hours
```

```text
ORG POLICIES  (Organization Settings > Policies - NOT project level)
  Restrict full-scoped PATs        Enabled
  Restrict global PATs             Enabled  (forces project-scoped)
  Maximum PAT lifetime             90 days
  Restrict admin-scope PATs        Enabled

TWO ERRORS, TWO STAGES
  "There was a resource authorization problem"  -> BEFORE the job. Pipeline not authorised
  AuthorizationFailed                           -> INSIDE the task. No RBAC role at that scope
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 42 |
| 38–43 | Re-read the trap index and the four dials, then move on |
| 30–37 | Rewrite the four dials and the permission matrix from memory, then retake |
| Below 30 | Redo Tasks 3, 4 and 5 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 41.

:::danger The one question

**Which dial does this?**

Licence, group, connection scope, or pipeline permission.

Name the dial before you read the options, and three of the four answers usually disqualify themselves.

:::
