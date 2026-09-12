---
sidebar_position: 6.5
toc_max_heading_level: 2
title: "Challenge 24: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 24 — AZ-400 exam questions

**48 questions** built only from what Challenge 24 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-24.md`**.

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

:::tip The boundary this whole challenge sits on

**Environments gate deployments. Branch protection gates merges.**

They are different objects protecting different events, and the exam swaps them deliberately. If a
question is about *who may deploy*, the answer is on the environment.

:::

---

# Section A — Single answer

---

## Q1

A production deployment completes with no approval, even though the `production` environment has a
required reviewer.

What is the most likely cause?

- A. The reviewer approved the run automatically
- B. The workflow was run via `workflow_dispatch`
- C. The job does not declare an `environment`
- D. Branch protection was not enabled on `main`

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-24.md`:** Break & fix Exercise 1, lines **570–596**.

```yaml
jobs:
  deploy-production:
    runs-on: ubuntu-latest
    # no environment: -> no protection rules are evaluated
    steps:
      - run: az webapp deploy --name contoso-api-prod ...
```

**Fixed:**

```yaml
    environment:
      name: production
      url: https://contoso-api.azurewebsites.net
```

**Protection rules live on the environment, and they only apply to jobs that reference it.** A job
without `environment:` is an ordinary job with no gate — and nothing warns you.

**Why the others fail**

- **A** — auto-approval does not exist. `prevent_self_review` (line 61) tightens it further
- **B** — the trigger is irrelevant. The gate is a property of the environment
- **D** — branch protection governs merging, not deploying. **This is the boundary the exam tests
  most**

</details>

---

## Q2

A production environment has a 15-minute wait timer, but deployments proceed immediately. The job
declares `environment: Production`.

What is wrong?

- A. Wait timers only apply to `workflow_dispatch` runs
- B. Wait timers require at least one required reviewer
- C. Wait timers are ignored on the repository's default branch
- D. Environment names are case-sensitive and must match exactly

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-24.md`:** Break & fix Exercise 3, lines **648–670**.

```yaml
    environment: Production      # line 652 - capital P
    environment: production      # line 667 - matches the created environment
```

Referencing a name that does not exist **creates a new, unprotected environment on the fly**. That is
why there is no error: you got exactly what you asked for, just not what you meant.

**Why the others fail**

- **A**, **B**, **C** — all invented restrictions. A wait timer applies to every run of a job
  referencing that environment, with or without reviewers

**How to spot it in real life:** the repository's Environments page will show two entries, one of
them empty. That extra environment is the tell.

</details>

---

## Q3

Which setting stops the person who triggered a workflow from approving their own deployment?

- A. `blockedApprovers`
- B. `prevent_self_review`
- C. `minRequiredApprovers`
- D. `wait_timer`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-24.md`:** line **61**.

```json
{
  "wait_timer": 15,
  "prevent_self_review": true,
  "reviewers": [
    {"type": "User", "id": 12345},
    {"type": "Team", "id": 67890}
  ]
}
```

This is a **separation of duties** control: the person who pushed the change cannot also be the one
who approves shipping it.

**Why the others fail**

- **A** — `blockedApprovers` (line 331) is the **Azure DevOps** equivalent concept, and it is a list
  of specific people rather than a rule about the requester
- **C** — `minRequiredApprovers` (line 328) sets how many, not who
- **D** — a wait timer delays; it does not restrict

</details>

---

## Q4

Which GitHub environment setting restricts deployments to specific branches or tags?

- A. Branch protection rules on the repository's `main`
- B. A `CODEOWNERS` file covering the deploy workflow
- C. `concurrency` groups keyed on the branch name
- D. `deployment_branch_policy` with branch policies

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-24.md`:** lines **41–44** and **49–54**, plus the tag policy at lines **183–185**.

```json
  "deployment_branch_policy": {
    "protected_branches": false,
    "custom_branch_policies": true
  }
```

```bash
gh api --method POST .../environments/production/deployment-branch-policies \
  --field name="main"

gh api --method POST .../environments/production/deployment-branch-policies \
  --field name="v*" \
  --field type="tag"
```

Note the two mutually exclusive modes: `protected_branches: true` means "any protected branch", while
`custom_branch_policies: true` lets you list exact patterns.

**Why the others fail**

- **A** — the merge boundary again
- **B** — CODEOWNERS assigns reviewers to code paths
- **C** — concurrency controls overlapping runs

</details>

---

## Q5

Which Azure Pipelines check prevents two runs deploying to the same environment simultaneously?

- A. Exclusive lock
- B. Business hours
- C. Approval check
- D. Invoke REST API

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-24.md`:** lines **422–431**.

```yaml
stages:
  - stage: DeployProduction
    lockBehavior: sequential      # options: sequential, runLatest
    jobs:
      - deployment: Production
        environment: "contoso-production"
```

**The two `lockBehavior` values matter:**

| Value | Behaviour |
|---|---|
| `sequential` | Queued runs wait their turn. Every run eventually deploys |
| `runLatest` | Only the newest queued run proceeds; older ones are cancelled |

**Why the others fail**

- **B** — time-window gate
- **C** — human gate, and two approved runs could still overlap
- **D** — calls an external system for a verdict

**Not `dependsOn`.** That orders stages **within one pipeline run**. Exclusive lock coordinates
**across separate runs**. This has caught you before.

</details>

---

## Q6

Which GitHub Actions feature is equivalent to an Azure Pipelines exclusive lock?

- A. `needs` between the deploy jobs
- B. `environment` on the deploy job
- C. `concurrency` with a group
- D. `permissions` on the deploy job

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-24.md`:** lines **452–454** and **460–462**.

```yaml
concurrency:
  group: production-deploy
  cancel-in-progress: false      # queue instead of cancel
```

The mapping is direct:

| Azure Pipelines | GitHub Actions |
|---|---|
| `lockBehavior: sequential` | `cancel-in-progress: false` |
| `lockBehavior: runLatest` | `cancel-in-progress: true` |

**Why the others fail**

- **A** — orders jobs inside one run
- **B** — provides gates and secrets, not mutual exclusion
- **D** — token scope

</details>

---

## Q7

For a production deployment, which `cancel-in-progress` value is correct?

- A. `true`, so the newest deployment always wins
- B. It makes no practical difference for production
- C. It must be omitted entirely for production
- D. `false`, so an in-progress deployment finishes

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-24.md`:** line **454** for production, contrasted with line **462** for staging.

```yaml
concurrency:
  group: production-deploy
  cancel-in-progress: false     # production - let it finish

    concurrency:
      group: staging-deploy
      cancel-in-progress: true  # staging - newest wins
```

**Why `false` for production:** cancelling mid-deployment can leave the environment half-updated —
some instances on the new version, some on the old, or a database migration applied without its
application code. Queueing is safe; interrupting is not.

**Why `true` is right for staging:** staging exists to test the latest code. An older deploy in
flight is already obsolete, and a broken staging is cheap.

**The rule:** *cancel where interruption is cheap, queue where it is dangerous.*

</details>

---

## Q8

A concurrency group named `production` is used by two independent services, and their deployments
cancel each other.

What is the fix?

- A. Scope the group name to each service
- B. Set `cancel-in-progress: false` on both
- C. Use different environments per service
- D. Add `needs` between the two workflows

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-24.md`:** Break & fix Exercise 2, lines **604–640**.

```yaml
# Broken - both services share one group
concurrency:
  group: production
  cancel-in-progress: true
```

```yaml
# Fixed
concurrency:
  group: production-service-a
# or dynamically (line 638):
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
```

**A concurrency group is a lock name.** Anything sharing the name shares the lock, across workflows
and repositories. The dynamic form is the safest default because two different workflows can never
collide accidentally.

**Why the others fail**

- **B** — they would queue instead of cancelling, which is still wrong. Independent services should
  not wait for each other at all
- **C** — different environments give different gates but do **not** separate concurrency groups
- **D** — `needs` cannot cross workflows, and serialising them is the opposite of the goal

</details>

---

## Q9

What does a custom deployment protection rule use to communicate its verdict to GitHub?

- A. A workflow output written to `$GITHUB_OUTPUT`
- B. A commit status check on the deployed SHA
- C. A callback URL supplied in the webhook payload
- D. A `repository_dispatch` event sent to the repo

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-24.md`:** lines **265** and **277–287**.

```javascript
const { action, environment, deployment_callback_url } = req.body;
...
fetch(deployment_callback_url, {
  method: 'POST',
  body: JSON.stringify({
    environment_name: environment,
    state: approved ? 'approved' : 'rejected',
    comment: comment
  })
});
```

The flow is: deployment requested → GitHub calls your App's webhook → your service decides → it POSTs
`approved` or `rejected` back to the callback URL. The deployment waits until it hears back.

**Why the others fail**

- **A** — the job has not started. There is no output
- **B** — commit status checks gate **merges**, not deployments
- **D** — that is an inbound trigger, not a verdict channel

**Two details worth keeping:** the handler verifies the webhook signature first (lines 238–244), and
it only acts when `action === 'requested'` (line 267).

</details>

---

## Q10

How is a custom deployment protection rule registered on an environment?

- A. By adding a `protection` key to the workflow's `environment` block
- B. By enabling the GitHub App with its integration ID on the environment
- C. By adding the GitHub App to `CODEOWNERS` for the workflow file
- D. By creating a deployment branch policy naming the GitHub App

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-24.md`:** lines **300–302**.

```bash
gh api --method POST \
  repos/contoso/contoso-api/environments/production/deployment_protection_rules \
  --field integration_id=<github-app-id>
```

Custom protection rules are always backed by a **GitHub App** — which is why Challenge 40 spends time
on Apps. An App has its own identity and installation permissions, so the check runs as the App, not
as a user.

**Why the others fail**

- **A** — no such workflow key. Protection lives on the environment, never in YAML
- **C** — CODEOWNERS is about code review
- **D** — a branch policy restricts branches, not logic

</details>

---

## Q11

Which Azure Pipelines approval setting defines how long an approval can remain pending?

- A. `timeout`
- B. `wait_timer`
- C. `executionOrder`
- D. `minRequiredApprovers`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-24.md`:** line **332**.

```json
    "settings": {
      "approvers": [{"id": "<user-id>", "displayName": "VP Engineering"}],
      "minRequiredApprovers": 1,
      "executionOrder": "anyOrder",
      "instructions": "Review the staging deployment results before approving.",
      "blockedApprovers": [],
      "timeout": 43200
    }
```

43200 minutes = **30 days**, the default.

**Why the others fail**

- **B** — `wait_timer` is the **GitHub** setting (line 60), and it is a **delay before** the
  deployment, not a deadline for a human
- **C** — `executionOrder` decides whether approvers act in sequence or in any order
- **D** — how many approvals are needed

**Do not confuse wait timer with timeout.** One postpones the deployment; the other expires the
approval request.

</details>

---

## Q12

A workflow job declares `environment: production`. Which secrets can it read?

- A. Repository secrets plus secrets scoped to `production`
- B. Only secrets scoped to the `production` environment
- C. Secrets from every environment in the repository
- D. Only repository secrets, never environment secrets

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-24.md`:** lines **486–492** and **507–520**.

```bash
gh secret set DATABASE_URL --env staging    --body "postgresql://...staging-db..."
gh secret set DATABASE_URL --env production --body "postgresql://...prod-db..."
```

```yaml
  deploy:
    environment: production      # line 511 - scopes what is visible
    steps:
      - run: echo "Log level: ${{ vars.LOG_LEVEL }}"
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}    # resolves to the PRODUCTION value
```

Repository secrets remain available; environment secrets are **added** and take precedence on name
collision.

**Why the others fail**

- **B** — too restrictive; shared repository secrets still work
- **C** — the isolation would be pointless. A staging job can never read production secrets
- **D** — the environment layer would be pointless

</details>

---

## Q13

Which check blocks a deployment while the target application has an active incident?

- A. Azure Monitor alerts
- B. Business hours
- C. Exclusive lock
- D. Approval check

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-24.md`:** lines **366–377**.

```text
# The check queries Azure Monitor for active alerts on the target resource
# If there are active Sev0 or Sev1 alerts, the deployment is blocked
```

**Why the others fail**

- **B** — time-based. An incident at 11am on a Tuesday passes a business-hours check happily
- **C** — prevents overlapping deployments, not deployment during an incident
- **D** — a human *could* catch it, but the requirement is automatic

**The pattern the exam repeats:** match the gate to the **kind** of condition. Time → business hours.
System health → Azure Monitor alerts. Human judgement → approval. Policy on an artifact → evaluate
artifact. Overlap → exclusive lock.

</details>

---

## Q14

A workflow must deploy to production only from version tags such as `v1.2.3`.

Which **trigger** achieves this?

- A. `on: push: branches: [main]`
- B. `on: release: types: [published]`
- C. `on: workflow_dispatch` with a version input
- D. `on: push: tags: ["v[0-9]+.[0-9]+.[0-9]+"]`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-24.md`:** lines **199–202**.

```yaml
on:
  push:
    tags:
      - "v[0-9]+.[0-9]+.[0-9]+"
```

The tag name is then available as `GITHUB_REF_NAME` (line 214):

```yaml
        run: echo "version=${GITHUB_REF_NAME#v}" >> $GITHUB_OUTPUT
```

That `#v` strips the leading `v`, turning `v1.2.3` into `1.2.3`.

**Why the others fail**

- **A** — fires on every push to main, tagged or not
- **B** — plausible and different: it fires when a **GitHub Release** is published, which may or may
  not be how this team tags. The question says tags
- **C** — manual, not automatic

**Pair it with the environment (lines 183–185):** a `type="tag"` deployment branch policy of `v*`
enforces the same rule at the environment level, so it holds even if the workflow is edited. Trigger
plus policy is defence in depth.

</details>

---

## Q15

In Azure DevOps, where are approvals and checks configured?

- A. In the pipeline YAML under the `deployment` job
- B. In the branch policies of the repository
- C. On the environment, in project settings
- D. In the service connection's security settings

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-24.md`:** lines **312–313** and **343**.

```text
# Project Settings > Environments > New environment
# Project Settings > Environments > production > Checks > Add check > Business Hours
```

The pipeline only **references** the environment (line 358):

```yaml
      - deployment: Production
        environment: "contoso-production"  # Has business hours check configured
```

**Why this split is deliberate — and testable:** if checks lived in YAML, anyone able to edit the
pipeline could remove them. Keeping them in project settings means changing a gate requires
environment permissions, not just repository write access.

**Why the others fail**

- **A** — the YAML names the environment and nothing more
- **B** — merge-time policy
- **D** — a service connection holds credentials. It does have its own approvals and checks, which is
  a related but separate control

</details>

---

## Q16

Which GitHub environment setting delays a deployment for a fixed period before it starts?

- A. `timeout`
- B. `wait_timer`
- C. `cancel-in-progress`
- D. `lockBehavior`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-24.md`:** line **60**.

```json
{
  "wait_timer": 15,
  "prevent_self_review": true
}
```

Fifteen minutes, counted **before the job starts** — no runner is assigned during the wait.

**Why it exists:** it gives a team a window to cancel a deployment they realise is wrong, without
requiring anyone to be online to approve it. Useful for automated releases outside working hours.

**Why the others fail**

- **A** — the Azure DevOps approval expiry (line 332)
- **C** — concurrency behaviour
- **D** — Azure Pipelines lock behaviour

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** protection rules can a GitHub environment enforce? (Choose three.)

- A. Required reviewers
- B. Required status checks
- C. Wait timer
- D. Linear history on merges
- E. Deployment branch and tag policy
- F. Signed commits on pushes

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-24.md`:** lines **57–75**.

```json
{
  "wait_timer": 15,                          // C
  "prevent_self_review": true,
  "reviewers": [                             // A
    {"type": "User", "id": 12345},
    {"type": "Team", "id": 67890}
  ],
  "deployment_branch_policy": {              // E
    "custom_branch_policies": true
  }
}
```

**Why the others fail — all three are branch protection rules, not environment rules**

- **B** — required status checks gate **merging a pull request**
- **D** — linear history is a merge-strategy rule
- **F** — signed commits are a push-time rule

**That grouping is the exam's favourite trap in this challenge.** Environment rules answer *may this
deploy?* Branch rules answer *may this merge?*

</details>

---

## Q18

Which **two** are valid reviewer types on a GitHub environment? (Choose two.)

- A. `Organization`
- B. `User`
- C. `App`
- D. `Team`
- E. `Role`

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-24.md`:** lines **62–65**.

```json
  "reviewers": [
    {"type": "User", "id": 12345},
    {"type": "Team", "id": 67890}
  ]
```

A team reviewer means **any member** of that team can approve, which is what you want for on-call
rotations — naming individuals creates a bottleneck when someone is on holiday.

**Why the others fail**

- **A** — too broad to be a meaningful gate
- **C** — an App **can** gate a deployment, but as a **custom protection rule** (line 302), not as a
  reviewer. Different mechanism
- **E** — roles are a repository permission concept

</details>

---

## Q19

Which **two** Azure Pipelines approval settings control *who* and *how many*? (Choose two.)

- A. `executionOrder`
- B. `timeout` in seconds
- C. `approvers`
- D. `instructions`
- E. `minRequiredApprovers`

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-24.md`:** lines **325–328**.

```json
      "approvers": [{"id": "<user-id>", "displayName": "VP Engineering"}],
      "minRequiredApprovers": 1,
      "executionOrder": "anyOrder",
      "instructions": "Review the staging deployment results before approving.",
      "blockedApprovers": [],
      "timeout": 43200
```

**Why the others fail**

- **A** — `executionOrder` decides sequence (`anyOrder` or in turn), not who or how many
- **B** — how long the request stays open
- **D** — text shown to the approver. Worth writing well in real life, but it controls nothing

</details>

---

## Q20

Which **two** describe `lockBehavior` in Azure Pipelines? (Choose two.)

- A. `sequential` queues runs so each one deploys in turn
- B. `sequential` cancels the in-progress run when a new one queues
- C. `runLatest` runs all queued runs in parallel
- D. `runLatest` cancels older queued runs and deploys only the newest
- E. `lockBehavior` replaces the need for `dependsOn`

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-24.md`:** line **431**.

```yaml
  - stage: DeployProduction
    lockBehavior: sequential   # or runLatest
```

**Why the others fail**

- **B** — sequential never interrupts. It queues
- **C** — the whole point of the lock is that runs do **not** overlap
- **E** — different concerns. `dependsOn` orders stages **within one run**; the lock coordinates
  **between runs**. Confusing the two is your recurring error

</details>

---

## Q21

Which **two** are true about GitHub concurrency groups? (Choose two.)

- A. Groups are scoped automatically to the workflow
- B. A group can be defined at workflow level or job level
- C. `cancel-in-progress: true` queues the newer run
- D. Concurrency replaces environment protection rules
- E. Any run sharing the group name shares the lock

<details>
<summary>Show answer</summary>

### Answer: B, E

**In `challenge-24.md`:** lines **452–454** (workflow level), **460–462** (job level), **604–614**
(the shared-name failure).

```yaml
concurrency:                    # workflow level
  group: production-deploy
jobs:
  deploy-staging:
    concurrency:                # job level - different group
      group: staging-deploy
```

**Why the others fail**

- **A** — **the exact bug in Break & fix Exercise 2.** Nothing is scoped automatically. Two unrelated
  workflows both using `group: production` fight over one lock. That is why line 638 recommends
  `${{ github.workflow }}-${{ github.ref }}`
- **C** — inverted. `true` cancels the in-progress run; `false` queues
- **D** — concurrency controls overlap; environments control permission

</details>

---

## Q22

Which **two** Azure Pipelines checks are appropriate for a production environment that must not
deploy during an incident or outside working hours? (Choose two.)

- A. Exclusive lock
- B. Approval check
- C. Business hours
- D. Azure Monitor alerts
- E. Invoke REST API

<details>
<summary>Show answer</summary>

### Answer: C, D

**In `challenge-24.md`:** lines **341–348** and **366–377**.

```text
Business Hours:
  Time zone: Eastern Time (US and Canada)
  Days: Monday through Thursday
  Start time: 09:00   End time: 17:00

Azure Monitor alerts:
  If there are active Sev0 or Sev1 alerts, the deployment is blocked
```

Note the days: **Monday through Thursday**. No Friday deployments — a deliberate practice so nobody
is debugging a production release over a weekend.

**Why the others fail**

- **A** — prevents overlap, not bad timing
- **B** — a human could enforce both, but the requirement is automatic
- **E** — could call an external system that checks either, but the built-in checks exist for exactly
  these two conditions. Reach for a custom call only when nothing built in fits

</details>

---

## Q23

Which **two** must be true for environment secrets to isolate staging from production? (Choose two.)

- A. The secrets have different names per environment
- B. Each secret is created with `--env <environment>`
- C. The workflow uses `secrets: inherit`
- D. Each job declares the matching `environment`
- E. The environments have required reviewers

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-24.md`:** lines **486–489** (B) and **511** (D).

```bash
gh secret set DATABASE_URL --env staging    --body "...staging-db..."
gh secret set DATABASE_URL --env production --body "...prod-db..."
```

```yaml
  deploy:
    environment: production      # this line decides which value you get
```

**Why the others fail**

- **A** — **the opposite is the point.** The same name with different values is what lets one
  workflow line serve both environments
- **C** — `secrets: inherit` is for reusable workflows (Challenge 23), unrelated
- **E** — reviewers gate deployment; they do not scope secrets

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must ensure production deployments are approved by a platform team member, can
only come from `main`, and never overlap with another production deployment.

---

## Q24

**Proposed solution:** Configure the `production` environment with a Team reviewer and a deployment
branch policy for `main`, add a `concurrency` group with `cancel-in-progress: false`, and have the
deploy job declare `environment: production`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

All three requirements are met.

```yaml
  deploy-production:
    environment:
      name: production           # reviewer + branch policy apply here
      url: https://contoso-api.azurewebsites.net
    concurrency:
      group: production-deploy
      cancel-in-progress: false  # queue, never interrupt
```

- **Approval** — Team reviewer on the environment (lines 62–65)
- **Branch restriction** — deployment branch policy for `main` (lines 74–75)
- **No overlap** — concurrency group, queueing rather than cancelling (line 454)

The last clause of the proposal matters most: without `environment: production` on the job, none of
the first two apply (Q1).

</details>

---

## Q25

**Proposed solution:** Configure branch protection on `main` requiring one approving review, and add
a `concurrency` group with `cancel-in-progress: true`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Both** halves fail, for different reasons.

**Approval:** branch protection requires review before **merging code**. Once merged, that commit can
be deployed any number of times with no further approval. The requirement is about deploying, not
merging.

**Overlap:** `cancel-in-progress: true` **cancels** the in-progress deployment. Production could be
left half-updated — some instances new, some old. The requirement says deployments must not overlap,
and interrupting one mid-flight is not the same as preventing overlap.

**And the branch restriction is missing entirely.** Nothing here stops a deploy from another branch.

</details>

---

## Q26

**Proposed solution:** Configure the `production` environment with a Team reviewer and a `main`
deployment branch policy, add `concurrency` with `cancel-in-progress: false`, but declare the job as
`environment: Production`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is configured correctly, and **one capital letter** defeats it.

```yaml
    environment: Production      # the environment you created is "production"
```

GitHub creates a **new, unprotected environment** named `Production` on first use (Break & fix
Exercise 3, lines 648–670). No reviewer, no branch policy — because those were set on the lowercase
one.

The concurrency group still works, since it is defined in the workflow rather than on the
environment. So overlap is prevented while approval and branch restriction silently are not.

**Why this is worth a question of its own:** it produces no error, no warning, and a green run. The
only visible symptom is an extra entry on the Environments page. **Config that fails silently is more
dangerous than config that fails loudly**, and the exam likes testing whether you notice.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — GitHub environments

| # | Statement | Answer |
|---|---|---|
| 1 | Protection rules apply to jobs that do not declare the environment |  |
| 2 | Environment names are case-sensitive |  |
| 3 | A wait timer holds the job before its first step |  |
| 4 | Environment secrets override repository secrets of the same name |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Protection rules apply to jobs that do not declare the environment | **No** |
| 2 | Environment names are case-sensitive | **Yes** |
| 3 | A wait timer holds the job before its first step | **Yes** |
| 4 | Environment secrets override repository secrets of the same name | **Yes** |

**In `challenge-24.md`:** lines **570–584** (row 1), **652–667** (row 2), **60** (row 3), **486–492**
(row 4).

Rows 1 and 2 are the same failure wearing different clothes: in both, the job is not attached to the
protected environment, so no gate applies. One omits the key; the other misspells the value.

Row 4 is what makes the pattern usable — one `${{ secrets.DATABASE_URL }}` in the workflow, resolved
per environment.

</details>

---

## Q28 — approvals

| # | Statement | Answer |
|---|---|---|
| 1 | `prevent_self_review` stops the triggering user approving their own deployment |  |
| 2 | An Azure DevOps approval can expire |  |
| 3 | A GitHub Team can be a required reviewer |  |
| 4 | Approvals are configured in the pipeline YAML |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `prevent_self_review` stops the triggering user approving their own deployment | **Yes** |
| 2 | An Azure DevOps approval can expire | **Yes** |
| 3 | A GitHub Team can be a required reviewer | **Yes** |
| 4 | Approvals are configured in the pipeline YAML | **No** |

**In `challenge-24.md`:** line **61** (row 1), line **332** (row 2), line **64** (row 3), lines
**312–313** (row 4).

Row 4 is the governance point. Approvals live on the **environment**, in project or repository
settings. If they lived in YAML, anyone with write access to the pipeline could delete the gate in
the same commit that ships the change.

Row 2: `timeout: 43200` minutes = 30 days.

</details>

---

## Q29 — concurrency and locks

| # | Statement | Answer |
|---|---|---|
| 1 | `cancel-in-progress: false` queues the newer run |  |
| 2 | Two workflows can share one concurrency group |  |
| 3 | `lockBehavior: runLatest` cancels older queued runs |  |
| 4 | `dependsOn` prevents concurrent runs of the same stage |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `cancel-in-progress: false` queues the newer run | **Yes** |
| 2 | Two workflows can share one concurrency group | **Yes** |
| 3 | `lockBehavior: runLatest` cancels older queued runs | **Yes** |
| 4 | `dependsOn` prevents concurrent runs of the same stage | **No** |

**In `challenge-24.md`:** line **454** (row 1), lines **604–614** (row 2), line **431** (row 3).

Row 2 is stated as a **capability**, and it is the cause of Break & fix Exercise 2. Sharing a group is
sometimes what you want — two workflows that both touch production genuinely should not overlap. It
is only a bug when the services are independent.

**Row 4 is your recurring mistake, stated plainly.** `dependsOn` orders stages **inside one run**. It
has no effect across runs. Use exclusive lock or `lockBehavior`.

</details>

---

## Q30 — checks

| # | Statement | Answer |
|---|---|---|
| 1 | A business hours check blocks deployment outside a time window |  |
| 2 | An Azure Monitor alerts check blocks deployment during an active incident |  |
| 3 | A custom GitHub protection rule is backed by a GitHub App |  |
| 4 | Branch protection can gate a deployment to an environment |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A business hours check blocks deployment outside a time window | **Yes** |
| 2 | An Azure Monitor alerts check blocks deployment during an active incident | **Yes** |
| 3 | A custom GitHub protection rule is backed by a GitHub App | **Yes** |
| 4 | Branch protection can gate a deployment to an environment | **No** |

**In `challenge-24.md`:** lines **343–348**, **376**, **300–302**.

Row 4 is the single most repeated boundary in this challenge, and it appears again in Challenge 19
and Challenge 20. **Branch protection gates merges. Environments gate deployments.** The same commit
that was merged once can be deployed fifty times; only the environment sees those fifty events.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each Azure Pipelines check to the condition it enforces.

| Check | Enforces |
|---|---|
| Approval |  |
| Business hours |  |
| Azure Monitor alerts |  |
| Exclusive lock |  |
| Invoke REST API |  |
| Branch control |  |
| Evaluate artifact |  |
| Required template |  |

**Options:** A human must authorise · An external system's verdict · Deployment only inside a time window · No active incident on the target · One run at a time to the environment · Only an approved source branch · The container image satisfies a policy · The pipeline extends an approved template

<details>
<summary>Show answer</summary>

| Check | Enforces |
|---|---|
| Approval | A **human** must authorise |
| Business hours | Deployment only inside a **time window** |
| Azure Monitor alerts | No **active incident** on the target |
| Exclusive lock | **One run at a time** to the environment |
| Invoke REST API | An **external system's** verdict |
| Branch control | Only an approved **source branch** |
| Evaluate artifact | The **container image** satisfies a policy |
| Required template | The pipeline **extends an approved template** |

**In `challenge-24.md`:** approval **316–338**, business hours **341–348**, Azure Monitor **366–377**,
exclusive lock **422–431**.

The last two are not in Challenge 24 and **are on the exam** — they were in your gap-filler note.
Evaluate artifact uses Rego policies via Open Policy Agent; required template forces a pipeline to
`extends:` a blessed template.

**Match the gate to the kind of condition:** time, health, human, policy, overlap, source.

</details>

---

## Q32

Match each GitHub Actions concept to its Azure Pipelines equivalent.

| GitHub Actions | Azure Pipelines |
|---|---|
| Environment required reviewers |  |
| `wait_timer` |  |
| Deployment branch policy |  |
| `concurrency` + `cancel-in-progress: false` |  |
| `concurrency` + `cancel-in-progress: true` |  |
| Custom protection rule (GitHub App) |  |
| Environment secrets |  |

**Options:** Approval check · Branch control check · Invoke Azure Function / Invoke REST API check · `lockBehavior: runLatest` · `lockBehavior: sequential` · (no direct equivalent — closest is a delayed approval) · Variable group scoped to a stage

<details>
<summary>Show answer</summary>

| GitHub Actions | Azure Pipelines |
|---|---|
| Environment required reviewers | Approval check |
| `wait_timer` | (no direct equivalent — closest is a delayed approval) |
| Deployment branch policy | Branch control check |
| `concurrency` + `cancel-in-progress: false` | `lockBehavior: sequential` |
| `concurrency` + `cancel-in-progress: true` | `lockBehavior: runLatest` |
| Custom protection rule (GitHub App) | Invoke Azure Function / Invoke REST API check |
| Environment secrets | Variable group scoped to a stage |

**In `challenge-24.md`:** lines **62–65 / 325–328**, **74–75**, **454 / 431**, **300–302 / 372**,
**486–492 / 527–561**.

**The row to notice is `wait_timer`.** GitHub has a fixed pre-deployment delay; Azure Pipelines has
no equivalent gate. Occasionally the exam asks which platform offers a feature, not just how to
configure it.

</details>

---

## Q33

Arrange the stages of a custom deployment protection rule, in order.

**Items:** Handler POSTs `approved` or `rejected` to the callback URL · GitHub sends a webhook to the
App · Deployment is requested · Handler verifies the webhook signature · Deployment proceeds or is
blocked

<details>
<summary>Show answer</summary>

### Answer

1. Deployment is requested — the job reaches its `environment`
2. GitHub sends a webhook to the App — line **260**
3. Handler verifies the webhook signature — lines **238–244**, **261**
4. Handler POSTs the verdict to the callback URL — lines **277–288**
5. Deployment proceeds or is blocked

```javascript
app.post('/webhook/deployment-protection', (req, res) => {
  if (!verifySignature(req)) return res.status(401).send('Invalid signature');   // 3
  const { action, environment, deployment_callback_url } = req.body;
  if (action !== 'requested') return res.status(200).send('OK');
  fetch(deployment_callback_url, { ... state: approved ? 'approved' : 'rejected' });  // 4
});
```

**Signature verification comes first for a reason.** The endpoint is public; without it, anyone could
POST a fake "approved" webhook. Note also `crypto.timingSafeEqual` at line 243 — a plain `===`
comparison leaks information through timing.

</details>

---

## Q34

Arrange the deployment pipeline of Challenge 24 in execution order, and mark where the gate applies.

**Items:** `smoke-tests` · `deploy-production` · `build` · `deploy-staging`

<details>
<summary>Show answer</summary>

### Answer

1. `build` — line **88**
2. `deploy-staging` — line **104**, `environment: staging`
3. `smoke-tests` — line **126**, no environment
4. `deploy-production` — line **146**, `environment: production` ← **the gate**

```yaml
  deploy-staging:
    needs: build
    environment:
      name: staging                # branch policy only, no reviewer
  smoke-tests:
    needs: deploy-staging          # no environment - runs freely
  deploy-production:
    needs: smoke-tests
    environment:
      name: production             # reviewer + wait timer + branch policy
```

**Two design points worth stating:**

- `smoke-tests` has **no** environment because it only reads. Attaching one would demand approval to
  run a test
- The gate sits at the **last possible moment**. Everything cheap and automatic happens first, so a
  human is only interrupted once there is something real to approve

</details>

---

## Q35

Match each requirement to the correct mechanism.

| Requirement | Mechanism |
|---|---|
| A person must authorise the production release |  |
| Only `main` may deploy to production |  |
| Only `v*` tags may deploy to production |  |
| Two production deploys must never overlap |  |
| Give the team 15 minutes to cancel |  |
| Block deploys during an incident |  |
| The requester must not approve their own deploy |  |
| Staging and production use different databases |  |

**Options:** Azure Monitor alerts check · Concurrency group / exclusive lock · Deployment branch policy · Deployment branch policy, `type: tag` · Environment required reviewer · Environment secrets · `prevent_self_review` · Wait timer

<details>
<summary>Show answer</summary>

| Requirement | Mechanism |
|---|---|
| A person must authorise the production release | **Environment required reviewer** |
| Only `main` may deploy to production | **Deployment branch policy** |
| Only `v*` tags may deploy to production | **Deployment branch policy, `type: tag`** |
| Two production deploys must never overlap | **Concurrency group / exclusive lock** |
| Give the team 15 minutes to cancel | **Wait timer** |
| Block deploys during an incident | **Azure Monitor alerts check** |
| The requester must not approve their own deploy | **`prevent_self_review`** |
| Staging and production use different databases | **Environment secrets** |

**In `challenge-24.md`:** lines **62**, **74**, **183–185**, **452**, **60**, **376**, **61**,
**486–489**.

**Every one of these is configured on the environment, not in YAML** — except concurrency, which is
the one exception and lives in the workflow.

</details>

---

# Section F — Hot area

---

## Q36

```json
{
  "[BLANK 1]": 15,
  "[BLANK 2]": true,
  "reviewers": [
    {"type": "[BLANK 3]", "id": 67890}
  ]
}
```

Requirement: delay 15 minutes, stop the requester self-approving, and let any member of a group
approve.

- **BLANK 1:** `timeout` / `delay` / `wait_timer` / `hold`
- **BLANK 2:** `block_author` / `prevent_self_review` / `require_other` / `separate_duties`
- **BLANK 3:** `Group` / `Role` / `Organization` / `Team`

<details>
<summary>Show answer</summary>

### Answer: `wait_timer`, `prevent_self_review`, `Team`

**In `challenge-24.md`:** lines **60–64**.

`timeout` is the **Azure DevOps** approval expiry (line 332) — a different platform and a different
meaning. Only `User` and `Team` are valid reviewer types.

</details>

---

## Q37

```bash
gh api --method POST \
  repos/contoso/contoso-api/environments/production/[BLANK 1] \
  --field name="v*" \
  --field type="[BLANK 2]"
```

- **BLANK 1:** `branch-protection` / `rulesets` / `deployment-branch-policies` / `checks`
- **BLANK 2:** `branch` / `ref` / `tag` / `release`

<details>
<summary>Show answer</summary>

### Answer: `deployment-branch-policies`, `tag`

**In `challenge-24.md`:** lines **183–185**.

`branch-protection` and `rulesets` govern **merges**. Only `deployment-branch-policies` restricts
which refs may deploy to an environment.

The `type` field takes `branch` (line 54) or `tag` (line 185).

</details>

---

## Q38

```yaml
jobs:
  deploy-production:
    runs-on: ubuntu-latest
    [BLANK 1]:
      name: [BLANK 2]
      url: https://contoso-api.azurewebsites.net
```

The environment was created as lowercase `production`.

- **BLANK 1:** `environments` / `deployment` / `environment` / `target`
- **BLANK 2:** `Production` / `PRODUCTION` / `production` / `prod`

<details>
<summary>Show answer</summary>

### Answer: `environment`, `production`

**In `challenge-24.md`:** lines **149–151**, and Break & fix Exercise 3 at lines **652–667**.

Both blanks are failure modes you have now seen twice. Omitting BLANK 1 means **no gate applies**
(Exercise 1). Getting BLANK 2's case wrong **creates a new unprotected environment** (Exercise 3).
Neither produces an error.

</details>

---

## Q39

```yaml
concurrency:
  group: production-deploy
  [BLANK 1]: false

jobs:
  deploy-staging:
    concurrency:
      group: staging-deploy
      [BLANK 1]: [BLANK 2]
```

Requirement: production deployments queue; staging deployments cancel older runs.

- **BLANK 1:** `cancel` / `queue` / `lock` / `cancel-in-progress`
- **BLANK 2:** `false` / `true`

<details>
<summary>Show answer</summary>

### Answer: `cancel-in-progress`, `true`

**In `challenge-24.md`:** lines **454** and **462**.

The value flips per environment: `false` for production because interrupting a live deployment can
leave it half-updated; `true` for staging because the newest code is the only code that matters and a
broken staging is cheap.

</details>

---

## Q40

```yaml
stages:
  - stage: DeployProduction
    [BLANK 1]: sequential
    jobs:
      - [BLANK 2]: Production
        [BLANK 3]: "contoso-production"
```

- **BLANK 1:** `concurrency` / `exclusive` / `lockBehavior` / `serialize`
- **BLANK 2:** `job` / `deployment` / `stage` / `task`
- **BLANK 3:** `pool` / `resource` / `target` / `environment`

<details>
<summary>Show answer</summary>

### Answer: `lockBehavior`, `deployment`, `environment`

**In `challenge-24.md`:** lines **429–434**.

`concurrency` is the GitHub keyword. A plain `job` has no `environment` property, so BLANK 2 and
BLANK 3 must go together — the same pairing as Challenge 20 Q37.

</details>

---

## Q41

```javascript
const { action, environment, [BLANK 1] } = req.body;
if (action !== '[BLANK 2]') return res.status(200).send('OK');

fetch([BLANK 1], {
  method: 'POST',
  body: JSON.stringify({
    environment_name: environment,
    state: approved ? 'approved' : '[BLANK 3]'
  })
});
```

- **BLANK 1:** `webhook_url` / `deployment_callback_url` / `response_url` / `callback`
- **BLANK 2:** `created` / `pending` / `queued` / `requested`
- **BLANK 3:** `denied` / `rejected` / `failed` / `blocked`

<details>
<summary>Show answer</summary>

### Answer: `deployment_callback_url`, `requested`, `rejected`

**In `challenge-24.md`:** lines **265**, **267**, **285**.

The state values are exactly `approved` and `rejected`. The `action` guard matters: the App receives
several event types and must only respond to `requested`, or it will answer events that are not
asking a question.

</details>

---

# Section G — Case study

## Case study: Contoso release governance

### Background

Contoso deploys `contoso-api` to staging and production on Azure App Service. A recent incident was
caused by a deployment that went out during an active outage, and another by two deployments
overlapping.

### Requirements

**Production gates**

- A platform team member must approve every production deployment
- The person who triggered the run must not be able to approve it
- The team needs 15 minutes to cancel an automated release before it starts
- Only the `main` branch and `v*` tags may deploy to production

**Safety**

- Two production deployments must never run at the same time, and an in-flight deployment must never
  be interrupted
- Deployments must not proceed while a Sev0 or Sev1 alert is active
- The Azure DevOps team additionally wants no deployments on Fridays or weekends

**Configuration**

- Staging and production must use different database connection strings
- Log level must be `debug` in staging and `warning` in production

---

## Q42

Which **two** settings meet the approval requirements? (Choose two.)

- A. A branch protection rule requiring one approving review
- B. `minRequiredApprovers: 1` in a GitHub environment
- C. A Team reviewer on the `production` environment
- D. A CODEOWNERS entry for the deploy workflow
- E. `prevent_self_review: true` on the environment

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-24.md`:** lines **61–65**.

A **Team** reviewer rather than a named individual means any platform team member can approve, so
holidays do not block releases.

**Why the others fail**

- **A** — gates merging, not deploying
- **B** — `minRequiredApprovers` is **Azure DevOps** syntax (line 328). GitHub environments do not use
  that field
- **D** — CODEOWNERS requires review of a **file change**. Once the workflow is merged, it deploys
  without further review

</details>

---

## Q43

Which configuration meets the 15-minute cancellation window?

- A. `timeout: 15` on the approval check in Azure DevOps
- B. `wait_timer: 15` on the `production` environment
- C. `sleep 900` as the first step of the deploy job
- D. A scheduled trigger that fires 15 minutes later

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-24.md`:** line **60**.

The job is held **before it starts** — no runner assigned, nothing billed, and the run is visibly
waiting so anyone can cancel it.

**Why the others fail**

- **A** — `timeout` is how long an **approval request** stays open before expiring. Opposite meaning
- **C** — the job has already started. A runner is allocated and billed for fifteen idle minutes, and
  cancelling mid-job is messier than never starting
- **D** — a schedule cannot be attached to a push-triggered release

</details>

---

## Q44

Which **two** configurations restrict production deployments to `main` and `v*` tags? (Choose two.)

- A. A deployment branch policy with `name="main"`
- B. `on: push: branches: [main]` in the workflow
- C. A branch protection rule on `main`
- D. A deployment branch policy with `name="v*"` and `type="tag"`
- E. `concurrency: group: main` on the deploy job

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-24.md`:** lines **74–75** and **183–185**.

```bash
gh api --method POST .../environments/production/deployment-branch-policies --field name="main"
gh api --method POST .../environments/production/deployment-branch-policies \
  --field name="v*" --field type="tag"
```

**Why the others fail**

- **B** — a trigger controls what **starts** the workflow. Someone could still add another trigger, or
  run it manually. The environment policy holds regardless of what the YAML says
- **C** — merge-time
- **E** — concurrency controls overlap

**Both together is the ideal answer in practice:** the trigger for everyday behaviour, the
environment policy as the enforcement that survives a YAML edit.

</details>

---

## Q45

Which configuration meets the overlap requirement?

- A. `concurrency` with `cancel-in-progress: true`
- B. `dependsOn` between the two deployment jobs
- C. An exclusive lock with `lockBehavior: runLatest`
- D. `concurrency` with `cancel-in-progress: false`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-24.md`:** lines **452–454**.

The requirement has two halves: never overlap, **and** never interrupt.

**Why the others fail**

- **A** — prevents overlap by **cancelling** the in-flight deployment. Violates the second half
- **B** — orders jobs **within one run**. Two separate runs are unaffected. **This is your recurring
  error**
- **C** — `runLatest` also cancels older queued runs. It prevents overlap but is the Azure Pipelines
  analogue of `cancel-in-progress: true`

**The Azure Pipelines answer to the same requirement is `lockBehavior: sequential`** (line 431).

</details>

---

## Q46

Which check meets the incident requirement?

- A. Azure Monitor alerts check on the `contoso-production` environment
- B. Business hours check on the `contoso-production` environment
- C. A `curl` health check as the first step of the deploy job
- D. Approval check with instructions to look for active alerts

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-24.md`:** lines **366–377**.

```text
# The check queries Azure Monitor for active alerts on the target resource
# If there are active Sev0 or Sev1 alerts, the deployment is blocked
```

**Why the others fail**

- **B** — meets the *Friday* requirement, not this one. An incident at 11am Tuesday passes it
- **C** — a health endpoint returning 200 does not mean there is no active alert. A partial outage or
  a downstream dependency failing can both be healthy at `/health` and Sev1 in Azure Monitor. And the
  deployment has already begun
- **D** — relies on a human remembering to look. The requirement is automatic

</details>

---

## Q47

Which check meets the "no Friday or weekend deployments" requirement?

- A. A `condition` on the stage testing the day of week
- B. A scheduled pipeline that only runs on weekdays
- C. Business hours check, Monday to Thursday
- D. Exclusive lock with `lockBehavior: sequential`

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-24.md`:** lines **343–348**.

```text
Business Hours check:
  Time zone: Eastern Time (US and Canada)
  Days: Monday through Thursday
  Start time: 09:00
  End time: 17:00
```

The check **holds** the deployment until the next allowed window rather than failing it.

**Why the others fail**

- **A** — a stage condition **skips** the stage. The deployment is silently not done, and the run goes
  green. A hold and a skip are very different outcomes
- **B** — schedules start pipelines; they do not gate a push-triggered release
- **D** — overlap, not timing

**Why "hold" beats "fail":** a Friday afternoon release simply waits until Monday morning. Nobody has
to remember to re-run it, and nothing is silently dropped.

</details>

---

## Q48

Which configuration meets the database and log-level requirements?

- A. Repository secrets with `_STAGING` and `_PROD` name suffixes for both values
- B. Environment secrets for the connection string, environment variables for the log level
- C. Workflow-level `env` blocks per job holding both the connection string and the log level
- D. A single variable group used by both stages holding both connection strings and log levels

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-24.md`:** lines **486–502** and **507–520**.

```bash
gh secret   set DATABASE_URL --env staging    --body "...staging-db..."
gh secret   set DATABASE_URL --env production --body "...prod-db..."
gh variable set LOG_LEVEL    --env staging    --body "debug"
gh variable set LOG_LEVEL    --env production --body "warning"
```

```yaml
    environment: production
    steps:
      - run: echo "Log level: ${{ vars.LOG_LEVEL }}"
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

**One workflow line, two values.** The `environment:` declaration resolves both.

**Why the others fail**

- **A** — **works, and is worse.** Suffixed repository secrets are visible to **every** job, including
  ones with no environment at all. A staging job could read the production connection string. The
  environment scope is the isolation
- **C** — `env` blocks are plaintext in the workflow file. Never for secrets
- **D** — one group cannot hold two different values for the same name. That is why Azure Pipelines
  uses two stage-scoped groups (lines 533 and 549)

**The decision rule, again:** *sensitive?* → secret. *Differs per environment?* → environment scope.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Missing `environment:` on the job** | Q1, Q27, Q38 | No environment declared = no protection rules evaluated |
| **Environment name case mismatch** | Q2, Q26, Q38 | Case-sensitive. A typo creates a new unprotected environment |
| **Branch protection offered for deployment** | Q1, Q17, Q25, Q30, Q42, Q44 | Environments gate deployments. Branch rules gate merges |
| **`dependsOn` used to prevent overlap** | Q5, Q20, Q29, Q45 | Intra-run ordering only. Use exclusive lock or concurrency |
| **`cancel-in-progress: true` in production** | Q7, Q25, Q45 | Interrupting a deploy leaves it half-updated. Queue instead |
| **Concurrency group too broad** | Q8, Q21 | A group is a lock name shared by anything using it |
| **`wait_timer` vs `timeout`** | Q11, Q16, Q43 | Timer delays the deployment. Timeout expires the approval |
| **Health check offered instead of an alerts check** | Q46 | 200 OK does not mean no active incident |
| **Stage `condition` instead of a business-hours check** | Q47 | A condition **skips**; a check **holds** |
| **Suffixed repository secrets** | Q48 | Visible to every job. Environment scope is the isolation |
| **`minRequiredApprovers` offered for GitHub** | Q42 | That is Azure DevOps. GitHub uses a reviewer list |
| **GitHub App reviewer vs custom protection rule** | Q18, Q10 | An App gates via a protection rule, never as a reviewer |

---

# The blocks to memorise

Line numbers are in `challenge-24.md`.

```json
// 1. GitHub production environment  (lines 57-71)
{
  "wait_timer": 15,
  "prevent_self_review": true,
  "reviewers": [
    {"type": "User", "id": 12345},
    {"type": "Team", "id": 67890}
  ],
  "deployment_branch_policy": {
    "protected_branches": false,
    "custom_branch_policies": true
  }
}
```

```bash
# 2. Deployment branch and tag policy  (lines 74-75, 183-185)
gh api --method POST .../environments/production/deployment-branch-policies \
  --field name="main"
gh api --method POST .../environments/production/deployment-branch-policies \
  --field name="v*" --field type="tag"

# 3. Custom protection rule registration  (lines 300-302)
gh api --method POST \
  repos/contoso/contoso-api/environments/production/deployment_protection_rules \
  --field integration_id=<github-app-id>

# 4. Environment secrets and variables  (lines 486-502)
gh secret   set DATABASE_URL --env production --body "postgresql://prod-db..."
gh variable set LOG_LEVEL    --env production --body "warning"
```

```yaml
# 5. Attaching the gate  (lines 149-151) - miss this and NOTHING is enforced
  deploy-production:
    needs: smoke-tests
    environment:
      name: production          # lowercase, exactly as created
      url: https://contoso-api.azurewebsites.net

# 6. Concurrency  (lines 452-462, 638)
concurrency:
  group: production-deploy
  cancel-in-progress: false     # production: queue, never interrupt
# safest default group name:
  group: ${{ github.workflow }}-${{ github.ref }}

# 7. Exclusive lock  (lines 429-434)
  - stage: DeployProduction
    lockBehavior: sequential    # or runLatest
    jobs:
      - deployment: Production
        environment: "contoso-production"

# 8. Tag-only release trigger  (lines 199-214)
on:
  push:
    tags:
      - "v[0-9]+.[0-9]+.[0-9]+"
...
        run: echo "version=${GITHUB_REF_NAME#v}" >> $GITHUB_OUTPUT
```

**The full Azure Pipelines check list — know all eight:**

| Check | Enforces |
|---|---|
| Approval | A human authorises |
| Branch control | Approved source branch only |
| Business hours | Time window |
| Evaluate artifact | Container image passes a Rego policy |
| Exclusive lock | One run at a time |
| Invoke Azure Function | Your own logic returns a verdict |
| Invoke REST API | An external system returns a verdict |
| Query Azure Monitor alerts | No active incident |
| Required template | Pipeline extends an approved template |

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 24 is exam-ready. Section 03c is complete |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 1 and 5, then retake this |
| Below 30 | Redo the challenge, configuring a real environment with a reviewer and a branch policy |

Record your result in `AZ-400-Learning-Log.md` under Challenge 24.

:::tip Two sentences to carry into the exam

**Environments gate deployments. Branch protection gates merges.**

**`dependsOn` orders stages within one run. Exclusive lock and concurrency coordinate between runs.**

The second one has cost you marks in more than one mock. Say it out loud before you close this file.

:::
