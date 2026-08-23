---
sidebar_position: 2.5
toc_max_heading_level: 2
title: "Challenge 40: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 40 — AZ-400 exam questions

**48 questions** built only from what Challenge 40 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-40.md`**.

:::danger Read this before you start

Every GitHub authentication question is decided by **two** facts.

**How far does it reach?** `GITHUB_TOKEN` = **this repository only**. GitHub App = **every repository
the installation covers**. Fine-grained PAT = **the repositories you ticked**. Classic PAT = **every
repository the user can see**.

**Who owns it?** `GITHUB_TOKEN` and GitHub Apps belong to the **automation**. PATs belong to a
**person** — and die when that person leaves.

The scenario at line 17 is that second fact biting: three tokens, a developer gone six months, and 15
repositories still depending on them.

:::

---

# Section A — Multiple choice

---

## Q1

A workflow must create a deployment in the current repository **and** trigger a workflow in
`contoso/production-infra`. Which authentication should it use?

- A. `GITHUB_TOKEN` with `permissions: contents: write`
- B. A classic PAT with `repo` scope stored as a secret
- C. A GitHub App installation token
- D. A fine-grained PAT owned by the team lead

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-40.md`:** lines **72** and **116–122**.

```yaml
      - name: Trigger deployment in target repo
        run: |
          gh workflow run deploy.yml \
            --repo contoso/production-infra \
            --ref main
        env:
          GH_TOKEN: ${{ steps['app-token'].outputs.token }}
```

**A is the trap, and it is a hard wall rather than a permissions problem.** Line 72 states it plainly:
`GITHUB_TOKEN` **cannot access other repositories in the organization**. No value in the `permissions`
key changes that — `permissions` tunes what the token may do *here*, never *where* it may go.

**Why the others fail**

- **B** — a classic `repo` PAT would work, and it is exactly what the scenario is replacing. It reaches
  every repository the owner can see, and it dies with the owner's account
- **D** — a fine-grained PAT can be scoped to both repositories and would technically function, but it
  is still **tied to a person**. GitHub Apps are the answer for automation because they are owned by
  the organisation and carry their own audit identity

</details>

---

## Q2

Which statement about `GITHUB_TOKEN` is correct?

- A. It can trigger other workflows in the same repository
- B. Its permissions are fixed and cannot be customised per workflow
- C. It persists across runs so it can be reused for caching
- D. It cannot push commits when branch protection requires pull-request reviews

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-40.md`:** lines **66–72** and Break scenario 1 at **300–302**.

```text
- Expires when the job completes
- Permissions configurable via the `permissions` key
- Cannot trigger other workflows (prevents recursive triggers)
- Cannot access other repositories in the organization
```

**`GITHUB_TOKEN` respects branch protection like any other actor.** It has no bypass.

**Why the others fail — each is the exact inverse of a line in that list**

- **A** — it deliberately cannot, so a workflow that pushes cannot trigger itself forever
- **B** — the `permissions` key exists precisely to customise it (lines 38–41)
- **C** — it expires when the job completes. There is nothing to persist

</details>

---

## Q3

Contoso wants to guarantee that **no automation token lives longer than 90 days**, enforced by the
organisation. Which token type supports an org-enforced maximum lifetime?

- A. Classic PATs
- B. Fine-grained PATs
- C. Both classic and fine-grained PATs
- D. `GITHUB_TOKEN` and fine-grained PATs

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-40.md`:** lines **154–159**.

```text
2. Restrict access via personal access tokens (classic): Do not allow
3. Require approval of fine-grained personal access tokens: Enable
```

**Read what the organisation can actually do to each type.** For classic PATs the only lever is
**block or allow** — the expiry is the user's choice, which is how the scenario ended up with three
tokens set to *no expiration* (line 17). For fine-grained PATs the organisation can require approval
**and** cap the lifetime.

**Why D is close but wrong.** `GITHUB_TOKEN` does expire — every job — but that is not an
*org-enforced lifetime policy*; it is built-in behaviour with nothing to configure. The question asks
which type the organisation can enforce a maximum on.

</details>

---

## Q4

A security engineer must view and dismiss Dependabot alerts across all repositories but must not be
able to change code. Which role fits?

- A. Organization Owner
- B. Organization Member with Write on every repository
- C. Security Manager
- D. Triage on each repository

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-40.md`:** line **252**.

```text
| Security manager | Read access to all repos, manage security alerts | Security team |
```

**Read to all repositories, plus alert management, and no write.** It is the only role that separates
those two things.

**Why the others fail**

- **A** — Owner can do everything, including delete the organisation. Wildly over-privileged for
  reading alerts
- **B** — grants exactly the write access the requirement forbids
- **D** — Triage manages issues and pull requests (line 274), not security alerts, and it would have to
  be granted repository by repository

</details>

---

## Q5

An auto-format workflow using `GITHUB_TOKEN` fails with *"refusing to allow a GitHub App to create or
update workflow files."* What is happening?

- A. The token lacks `contents: write`
- B. `GITHUB_TOKEN` can never modify files under `.github/workflows/`
- C. The runner has no git credentials
- D. The branch does not exist

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-40.md`:** line **302**.

```text
**Cause:** By default, GITHUB_TOKEN cannot push to branches with branch protection rules that
require PR reviews, and it can never modify workflow files under .github/workflows/.
```

**This is a deliberate, unconditional restriction**, and it exists for an obvious reason: a token that
could rewrite workflow files could rewrite its own permissions and then do anything.

**Why A is the plausible wrong answer.** Adding `contents: write` fixes ordinary pushes. It does
**not** unlock `.github/workflows/` — that is a separate hard rule, which is why the fix at lines
308–327 swaps in a **GitHub App token**.

</details>

---

## Q6

Which is the correct fix for the failing formatter workflow?

- A. Add `permissions: contents: write` and retry
- B. Use a GitHub App token and add the app to the branch protection bypass list
- C. Disable branch protection on `main`
- D. Commit with `--no-verify`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-40.md`:** lines **308–329**.

```yaml
- name: Generate token with bypass
  id: app-token
  uses: actions/create-github-app-token@v1
  with:
    app-id: ${{ vars.FORMATTER_APP_ID }}
    private-key: ${{ secrets.FORMATTER_APP_PRIVATE_KEY }}
```

**Two halves, and both are required** (line 329): the App token gives an identity that *may* be
excepted, and the bypass list is where you actually grant the exception. The token alone still hits
the protection rule.

**Why the others fail**

- **A** — Q5's rule
- **C** — removes the control instead of granting a narrow, named, auditable exception to it
- **D** — `--no-verify` skips **local** git hooks. Branch protection is enforced server-side and never
  sees it

</details>

---

## Q7

A GitHub App installation token returns **403** for a repository in the same organisation. What is the
most likely cause?

- A. The app's private key has expired
- B. The installation's repository access is "Selected repositories" and this repo is not included
- C. Installation tokens cannot access private repositories
- D. The app needs `admin:org` scope

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-40.md`:** lines **333–341**.

```bash
gh api /app/installations/{installation_id}/repositories --jq '.repositories[].full_name'
```

**Installation access and app permissions are two different dials**, and this is the one people forget.
The app can hold every permission it needs and still be unable to touch a repository that was never
added to the installation.

**403 versus 404 is the tell.** 403 means *recognised but not allowed here*; a fine-grained PAT
outside its scope returns **404** instead (line 151), because GitHub hides what the token cannot see.

**Why the others fail**

- **A** — an expired key fails at token **generation**, before any API call
- **C** — installation tokens access private repositories routinely; that is their job
- **D** — `admin:org` is classic-PAT vocabulary. Apps use named permissions such as Members: read

</details>

---

## Q8

Which permissions does a GitHub App need to publish CI build status, and nothing more?

- A. Checks: write, Contents: read
- B. Contents: write, Pull requests: write
- C. Administration: write
- D. Deployments: write, Environments: read

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-40.md`:** line **167**.

```text
| CI build status | Checks: write, Contents: read | None |
```

**Checks: write to publish the result, Contents: read to see the code it checked.** Nothing else.

**Why the others fail** — B is the auto-merge row (line 168), D is the deployment row (line 169), and C
appears in none of them. **Administration is repository settings**: it could remove branch protection,
which is not something a status reporter should ever be able to do.

</details>

---

## Q9

Which permission set matches a **deployment** app?

- A. Deployments: write, Contents: read, Environments: read
- B. Contents: write, Checks: write
- C. Members: read
- D. Security events: read, Contents: read

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-40.md`:** line **169**.

**Note that Environments is read, not write.** The app deploys **to** environments; it does not create
or reconfigure them. Environment protection rules are the thing gating the deployment, so letting the
deployer edit them would defeat the gate — the same principle as Challenge 37, where a pipeline names
an environment but never edits it.

**Why the others fail** — C is the team-notifications row (line 170), D is the security-scanning row
(line 171), and B mixes build status with write access it does not need.

</details>

---

## Q10

Fine-grained PAT `contoso-ci-readonly` is scoped to `contoso/webapp` and `contoso/api`. A script calls
`gh api repos/contoso/other-repo`. What happens?

- A. 200 with the repository data
- B. 403 Forbidden
- C. 404 Not Found
- D. 401 Unauthorized

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-40.md`:** lines **149–151**.

```bash
# This should fail (not in token scope)
gh api repos/contoso/other-repo --jq '.full_name'
# Expected: 404 Not Found
```

**404, not 403, and the difference is intentional.** GitHub does not confirm the existence of
resources a credential is not scoped to — a 403 would leak the fact that `other-repo` exists.

**Learn the three codes together, because the exam uses them to test whether you actually ran the
lab.** 401 = the token is invalid or missing. 403 = valid token, recognised resource, action refused
(Q7). 404 = out of scope, and GitHub is not telling you anything more.

</details>

---

## Q11

Which organisation setting **removes** classic PATs as an access route?

- A. Restrict access via personal access tokens (classic): **Do not allow**
- B. Require approval of fine-grained personal access tokens: Enable
- C. Workflow permissions: Read repository contents
- D. Restrict access via fine-grained personal access tokens: Allow

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-40.md`:** line **157**.

**This is the single setting that ends the scenario at line 17.** The three orphaned tokens stop
working the moment it is applied, whether or not anyone remembers they exist.

**Why the others fail** — B governs the *replacement* token type, D **enables** the replacement route,
and C is about `GITHUB_TOKEN`'s defaults, an unrelated dial. All three are correct things to configure,
and none of them revokes a classic PAT.

</details>

---

## Q12

Which two settings appear under Organization Settings > Actions > General to harden the default token?

- A. Workflow permissions, and the create-and-approve-pull-requests toggle
- B. Fine-grained PAT approval and classic PAT restriction
- C. Branch protection and required reviewers
- D. Secret scanning and push protection

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-40.md`:** lines **239–243**.

```text
2. Workflow permissions: Read repository contents and packages permissions
3. Uncheck "Allow GitHub Actions to create and approve pull requests"
```

**Why the second toggle matters more than it looks.** If Actions can both create *and approve* a pull
request, a workflow can manufacture its own approval and satisfy a review requirement without a human
ever looking. Unchecking it closes that self-approval loop.

**Why B is the near-miss** — those are the right controls, on the wrong page. PAT policy lives under
Organization Settings > Personal access tokens (line 156).

</details>

---

## Q13

A workflow declares `permissions: contents: read` at the top, and its `test` job declares
`permissions: contents: read, checks: write`. What does the `test` job get?

- A. `contents: read` only
- B. `contents: read` and `checks: write`
- C. All scopes, because the job overrides with write
- D. The workflow-level block is invalid when a job also declares one

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-40.md`:** lines **192–208**.

```yaml
permissions:
  contents: read

jobs:
  test:
    permissions:
      contents: read
      checks: write
```

**A job-level `permissions` block replaces the workflow-level one entirely** — it does not merge with
it. Which is why `contents: read` is **repeated** inside the job: leave it out and the job would have
no contents access at all, and `actions/checkout` would fail.

**That repetition is not redundancy. It is the whole mechanic**, and the exam tests it by deleting the
repeated line and asking what breaks.

</details>

---

## Q14

Why does the `deploy` job add `id-token: write`?

- A. To sign the container image
- B. To request an OIDC token for `azure/login`
- C. To write to the GitHub Packages registry
- D. To create a deployment status

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-40.md`:** lines **224–236**.

```yaml
    permissions:
      contents: read
      id-token: write
      deployments: write
    environment: production
    steps:
      - name: Azure Login (OIDC)
```

**`id-token: write` is what lets the job ask GitHub for an OIDC token** — the workload identity
federation flow from Challenge 39. Omit it and the login fails at the token request, before Azure is
ever contacted.

**And `deployments: write` is the separate one** that records the deployment against the repository —
option D's job, and it is already in the block.

**The absence of `client-secret`** under `azure/login` is the confirmation that this is OIDC rather
than a stored credential.

</details>

---

## Q15

Which characteristic makes `GITHUB_TOKEN` unsuitable for a workflow that must trigger a second workflow
in the **same** repository?

- A. It expires when the job completes
- B. It cannot trigger other workflows
- C. It is scoped to the repository
- D. Its permissions cannot include `actions: write`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-40.md`:** line **71**.

```text
- Cannot trigger other workflows (prevents recursive triggers)
```

**The reason is in the parenthesis.** A workflow that pushes a commit would re-trigger itself on
`push`, forever. GitHub blocks the loop at the token level rather than trusting every author to avoid
it.

**Why C is the wrong reading of the right fact.** Repository scope is real, but it is not the obstacle
here — the target workflow **is** in this repository. The obstacle is the anti-recursion rule.

**The fix, when you genuinely need the chain**, is a GitHub App token — or `workflow_run`, which is
designed for exactly this and is not blocked (Challenge 21).

</details>

---

## Q16

Contoso must let a contractor work on `contoso/webapp` and nothing else. Which is correct?

- A. Add them as an Organization Member
- B. Add them as an Outside collaborator on that repository
- C. Grant them the Security Manager role
- D. Make them a Billing manager

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-40.md`:** lines **253** and **262–264**.

```bash
gh api repos/contoso/webapp/collaborators/vendor-user -X PUT \
  --field permission=push
```

```text
| Outside collaborator | Access to specific repositories only | Contractors, vendors |
```

**Outside collaborator is the only role that is not organisation-wide.** It grants access to named
repositories and gives no organisation membership, no team visibility, and no view of the member list.

**Why A is the common mistake.** Member is "default access" (line 250) — it makes the contractor part
of the organisation, visible in it, and eligible for every team-based grant. That is broader than the
requirement, and it survives long after the contract ends unless someone remembers to remove it.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** are true of `GITHUB_TOKEN`? (Choose three.)

- A. It is created automatically for every workflow run
- B. It expires when the job completes
- C. Its permissions are set with the `permissions` key
- D. It can access other repositories in the organisation
- E. It can trigger other workflows
- F. It must be created in Settings before first use

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-40.md`:** lines **66–72**.

```text
- Automatically created, no manual setup required
- Scoped to the current repository only
- Expires when the job completes
- Permissions configurable via the `permissions` key
- Cannot trigger other workflows (prevents recursive triggers)
- Cannot access other repositories in the organization
```

**D and E are the two "cannot" lines flipped**, and they are the two the exam reuses most. F contradicts
"no manual setup required" — there is nothing to create, which is precisely why it is the default
choice for single-repository automation.

</details>

---

## Q18

Which **two** are advantages of a GitHub App over a personal access token for automation? (Choose two.)

- A. It is owned by the organisation, not by a user account
- B. Its installation token is short-lived and scoped to the installation
- C. It never needs any permissions configured
- D. It can bypass all branch protection rules by default
- E. It works without any credential of any kind

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-40.md`:** lines **74–76**, **103–113**, and line **17**.

**A is the whole reason the scenario exists.** A developer left six months ago and their tokens still
authenticate 15 repositories. An app is an organisation object — nobody's departure orphans it.

**B is the operational half:** `actions/create-github-app-token@v1` mints a token at the start of the
run, scoped to what the installation covers, and it expires shortly after. There is no long-lived
secret in a variable.

**Why the others fail**

- **C** — apps need permissions explicitly (lines 86–88 and the table at 165–171). That is a feature
- **D** — an app bypasses nothing by default; it must be **added to the bypass list** (line 329)
- **E** — the app still has a **private key** (`secrets.CONTOSO_APP_PRIVATE_KEY`, line 108). It is one
  well-protected credential in place of many scattered ones, not zero credentials

</details>

---

## Q19

Which **three** should Contoso configure to end the orphaned-classic-PAT problem? (Choose three.)

- A. Restrict access via classic PATs: Do not allow
- B. Require approval of fine-grained PATs
- C. Replace cross-repo automation with a GitHub App
- D. Extend the classic PATs' expiry to 12 months
- E. Share one fine-grained PAT across all 15 repositories
- F. Give every developer Owner so nobody is blocked

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-40.md`:** lines **157**, **158**, and **74–76**.

**Each one closes a different half of the finding.** A revokes the existing route. B controls what
replaces it. C removes the *need* for a user-owned token in the first place — because the cross-repo
requirement is what drove someone to a `repo`-scoped classic PAT originally.

**Why the others fail**

- **D** — keeps the credential that a departed employee created. Expiry was never the only problem
- **E** — reproduces the original failure with newer vocabulary: one shared credential, 15
  repositories, one blast radius
- **F** — the opposite of least privilege, and line 249 caps Owners at two or three people

</details>

---

## Q20

Which **two** organisation-level roles grant access without granting the ability to write code? (Choose
two.)

- A. Billing manager
- B. Security manager
- C. Member
- D. Owner
- E. Outside collaborator with `push`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-40.md`:** lines **251–253**.

**Billing manager sees billing and nothing else. Security manager reads all repositories and manages
alerts, with no write.** They are the two "wide but shallow" roles.

**Why the others fail**

- **C** — Member is the *default* level and can be added to teams that carry write
- **D** — Owner is full administrative access
- **E** — `push` **is** write, and the `--field permission=push` at line 264 spells it out

</details>

---

## Q21

Which **two** are true about fine-grained PATs as configured in Task 3? (Choose two.)

- A. Metadata: Read-only is always required
- B. Repository access can be limited to selected repositories
- C. They are owned by the organisation rather than a user
- D. They cannot be given write permissions
- E. They have no expiry

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-40.md`:** lines **136–141**.

```text
# - Repository access: Only select repositories > contoso/webapp, contoso/api
# - Permissions:
#   - Contents: Read-only
#   - Metadata: Read-only (always required)
#   - Pull requests: Read and write
```

**Metadata is the implicit baseline** — without it the token cannot even resolve a repository name, so
GitHub adds it to every fine-grained PAT.

**Why the others fail**

- **C** — still a **personal** access token. The resource *owner* is the organisation (line 136), which
  is what brings it under org policy, but the token belongs to the user. This distinction is the exact
  reason GitHub Apps win in Q1
- **D** — line 141 grants Pull requests: Read and write
- **E** — line 135 sets 30 days, described as the maximum recommended for automation

</details>

---

## Q22

Which **two** conditions cause a `GITHUB_TOKEN` push to fail? (Choose two.)

- A. The target branch requires pull-request reviews
- B. The commit touches a file under `.github/workflows/`
- C. The workflow declares `permissions: contents: write`
- D. The repository is private
- E. The runner is `ubuntu-latest`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-40.md`:** line **302**.

**Both come from the same sentence, and they are different in kind.** A is *conditional* — it depends
on how the branch is protected, so it disappears on an unprotected branch. B is *absolute* — no
setting, permission or branch makes it work.

**Why C is the trap.** `contents: write` is what you would add, and it is necessary for a normal push.
It is simply not sufficient for either of these two.

</details>

---

## Q23

Which **two** permissions belong in the `deploy` job of the least-privilege workflow? (Choose two.)

- A. `id-token: write`
- B. `deployments: write`
- C. `checks: write`
- D. `packages: write`
- E. `administration: write`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-40.md`:** lines **224–227**.

```yaml
    permissions:
      contents: read
      id-token: write
      deployments: write
```

**Three permissions for three distinct needs:** read the code, get an OIDC token for Azure, record the
deployment.

**Why C is the near-miss** — `checks: write` is in the **test** job (line 208), for publishing test
results. Correct permission, wrong job, and moving it into `deploy` would widen that job for no reason.

**D and E appear nowhere** in the challenge. Note that `administration: write` would let a job edit the
repository's own protection rules — never a deployment job's business.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must remove three orphaned classic PATs with `repo` and `admin:org` scope and no
expiry, shared across 15 repositories. The replacement must support cross-repository automation, must
not be tied to any individual, must expire, and must be auditable.

---

## Q24

**Proposed solution:** Create a GitHub App owned by the organisation with the minimum permissions per
use case, install it on the required repositories, generate installation tokens in workflows with
`actions/create-github-app-token@v1`, set classic PATs to **Do not allow**, and require approval for
fine-grained PATs.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-40.md`:** lines **74–89**, **103–113**, **157–158**, **165–171**.

| Requirement | Mechanism |
|---|---|
| Cross-repository | App installation covering both repositories |
| Not tied to an individual | The app is an organisation object |
| Expires | Installation tokens are short-lived |
| Auditable | Actions appear as the app, not as a person |
| Old route closed | Classic PATs set to Do not allow |

**The last row is what makes it a fix rather than an addition.** Building the replacement without
revoking the originals leaves three unexpiring `admin:org` tokens live — the finding stays open.

</details>

---

## Q25

**Proposed solution:** Create one fine-grained PAT under a shared service account with access to all 15
repositories, 90-day expiry, and Contents plus Pull requests write. Store it as an organisation secret.
Leave classic PATs allowed for backward compatibility.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Three failures, and they compound.**

**"Leave classic PATs allowed" alone sinks it.** The three orphaned tokens keep working. Nothing about
the finding has changed.

**A shared service account is a person-shaped object wearing a costume.** It has credentials someone
must hold, an owner who can be disabled or offboarded, and a login that cannot be attributed to a
specific automation. The audit trail says "svc-contoso did it" for all 15 repositories.

**One credential across 15 repositories is the original blast radius**, reduced only from *every*
repository to *fifteen*. The Q19 trap, restated.

</details>

---

## Q26

**Proposed solution:** Create a GitHub App with the minimum permissions, install it on all 15
repositories, use installation tokens in workflows, and set classic PATs to **Do not allow**. Grant the
app Administration: write so it can never be blocked by a repository setting.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and that sentence is bigger than the rest of the
solution.

**Administration: write is repository-settings control.** An app holding it can remove branch
protection, change merge requirements, add collaborators, and reconfigure the very controls that make
the other answers safe.

**The reasoning offered — "so it can never be blocked" — is the exact anti-pattern the challenge is
about.** The permission table at lines 165–171 assigns the narrowest set per use case, and none of the
five rows contains Administration.

**The correct handling of "blocked by a setting" is Break scenario 1's answer** (line 329): grant a
**named, auditable exception** on the specific protection rule, not a blanket power to edit rules.

**Compare with Q24, which is identical minus this line.** One added permission turns a passing design
into a failing one — and the exam builds Yes/No triplets exactly this way.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — GITHUB_TOKEN behaviour

| # | Statement | Answer |
|---|---|---|
| 1 | It is created automatically for every workflow run |  |
| 2 | It can read a second repository if given `contents: read` |  |
| 3 | It expires when the job completes |  |
| 4 | It can update files under `.github/workflows/` |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | It is created automatically for every workflow run | **Yes** |
| 2 | It can read a second repository if given `contents: read` | **No** |
| 3 | It expires when the job completes | **Yes** |
| 4 | It can update files under `.github/workflows/` | **No** |

**In `challenge-40.md`:** lines **67**, **72**, **69**, **302**.

Row 2 is the single most-tested misconception in this challenge. **Permissions control *what*, never
*where*.**

Row 4 has no exception, no setting, and no permission that unlocks it.

</details>

---

## Q28 — tokens and ownership

| # | Statement | Answer |
|---|---|---|
| 1 | A fine-grained PAT is owned by the user who created it |  |
| 2 | A GitHub App is owned by the organisation |  |
| 3 | An organisation can enforce a maximum lifetime on classic PATs |  |
| 4 | An organisation can block classic PATs entirely |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A fine-grained PAT is owned by the user who created it | **Yes** |
| 2 | A GitHub App is owned by the organisation | **Yes** |
| 3 | An organisation can enforce a maximum lifetime on classic PATs | **No** |
| 4 | An organisation can block classic PATs entirely | **Yes** |

**In `challenge-40.md`:** lines **130–136**, **79**, **157**.

Rows 3 and 4 together are the whole classic-PAT policy story: **you cannot govern them, you can only
switch them off.**

Row 1 is why Q1 chose an app over a fine-grained PAT even though both can reach two repositories.

</details>

---

## Q29 — roles

| # | Statement | Answer |
|---|---|---|
| 1 | Security Manager gives read across all repositories |  |
| 2 | Triage can push code |  |
| 3 | Maintain can delete the repository |  |
| 4 | Outside collaborator is limited to named repositories |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Security Manager gives read across all repositories | **Yes** |
| 2 | Triage can push code | **No** |
| 3 | Maintain can delete the repository | **No** |
| 4 | Outside collaborator is limited to named repositories | **Yes** |

**In `challenge-40.md`:** lines **252**, **274**, **276**, **253**.

Row 3 is the definition of Maintain: *manage repo settings (**no destructive actions**)*. Deleting is
Admin.

Row 2: Triage manages issues and pull requests without code write — the "helpful but harmless" role.

</details>

---

## Q30 — the permissions key

| # | Statement | Answer |
|---|---|---|
| 1 | A job-level `permissions` block replaces the workflow-level one |  |
| 2 | Omitting `contents: read` in a job that overrides permissions breaks `checkout` |  |
| 3 | `id-token: write` is required for OIDC login to Azure |  |
| 4 | Organisation defaults can force read-only workflow permissions |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A job-level `permissions` block replaces the workflow-level one | **Yes** |
| 2 | Omitting `contents: read` in a job that overrides permissions breaks `checkout` | **Yes** |
| 3 | `id-token: write` is required for OIDC login to Azure | **Yes** |
| 4 | Organisation defaults can force read-only workflow permissions | **Yes** |

**In `challenge-40.md`:** lines **192–208**, **226**, **241–242**.

All four are Yes, which the exam does use — and it is exactly when a candidate who is pattern-matching
on "there must be a No in here" talks themselves out of a correct row.

Row 4 is the organisation acting as a floor: the default becomes read-only, and any workflow needing
more must **ask for it in writing** in its own `permissions` block.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each automation requirement to the correct credential.

| Requirement | Credential |
|---|---|
| Create an issue in the current repository |  |
| Trigger a workflow in another repository |  |
| A developer's local script reading two repositories |  |
| Push to a branch that requires reviews |  |
| Publish a check run from CI |  |

**Options:** Fine-grained PAT · GitHub App installation token · GitHub App token + bypass entry · `GITHUB_TOKEN` · `GITHUB_TOKEN` with `checks: write`

<details>
<summary>Show answer</summary>

| Requirement | Credential |
|---|---|
| Create an issue in the current repository | **`GITHUB_TOKEN`** |
| Trigger a workflow in another repository | **GitHub App installation token** |
| A developer's local script reading two repositories | **Fine-grained PAT** |
| Push to a branch that requires reviews | **GitHub App token + bypass entry** |
| Publish a check run from CI | **`GITHUB_TOKEN` with `checks: write`** |

**In `challenge-40.md`:** lines **49–56**, **116–122**, **129–141**, **308–329**, **167**.

**The routing question is always the same: does it leave this repository, and is a human behind it?**
Stay inside with no human → `GITHUB_TOKEN`. Leave the repository → App. A human at a keyboard →
fine-grained PAT.

</details>

---

## Q32

Match each GitHub App use case to its minimum permissions.

| Use case | Permissions |
|---|---|
| CI build status |  |
| Auto-merge pull requests |  |
| Deployment |  |
| Team notifications |  |
| Security scanning |  |

**Options:** Checks: write, Contents: read · Deployments: write, Contents: read, Environments: read · **Members: read** (organisation) · Pull requests: write, Contents: write · Security events: read, Contents: read

<details>
<summary>Show answer</summary>

| Use case | Permissions |
|---|---|
| CI build status | **Checks: write, Contents: read** |
| Auto-merge pull requests | **Pull requests: write, Contents: write** |
| Deployment | **Deployments: write, Contents: read, Environments: read** |
| Team notifications | **Members: read** (organisation) |
| Security scanning | **Security events: read, Contents: read** |

**In `challenge-40.md`:** lines **165–171**.

**Two patterns worth noticing.** Only auto-merge needs Contents: **write** — because merging writes to
the branch. And only team notifications needs an **organisation** permission; everything else is
repository-scoped.

</details>

---

## Q33

Arrange the steps to replace a classic PAT with a GitHub App for cross-repository deployment.

**Items:** Add the app-token step to the workflow · Create the GitHub App with minimum permissions ·
Set classic PATs to Do not allow · Install the app on the required repositories · Store the private key
as a secret and the app ID as a variable

<details>
<summary>Show answer</summary>

### Answer

1. Create the GitHub App with minimum permissions — lines **79–88**
2. Install the app on the required repositories — line **88**
3. Store the private key as a secret and the app ID as a variable — lines **107–108**
4. Add the app-token step to the workflow — lines **103–109**
5. Set classic PATs to **Do not allow** — line **157**

**Revoking comes last, and that ordering is the answer the exam wants.** Blocking classic PATs while
15 repositories still depend on them takes every pipeline down. Build the replacement, prove it works,
then close the old door.

**Note the split at step 3:** the app ID goes in `vars`, the private key in `secrets`. The ID is not
sensitive; the key is the credential.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| 404 from `gh api` on a repository that exists |  |
| 403 from an installation token |  |
| "refusing to allow a GitHub App to ... workflow files" |  |
| A workflow cannot reach a sibling repository |  |
| Automation broke when an employee left |  |

**Options:** A user-owned PAT · `GITHUB_TOKEN` cannot write `.github/workflows/` · `GITHUB_TOKEN` is repository-scoped · Repository not in the app's installation · Repository outside the fine-grained PAT's scope

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| 404 from `gh api` on a repository that exists | **Repository outside the fine-grained PAT's scope** |
| 403 from an installation token | **Repository not in the app's installation** |
| "refusing to allow a GitHub App to ... workflow files" | **`GITHUB_TOKEN` cannot write `.github/workflows/`** |
| A workflow cannot reach a sibling repository | **`GITHUB_TOKEN` is repository-scoped** |
| Automation broke when an employee left | **A user-owned PAT** |

**In `challenge-40.md`:** lines **151**, **335**, **302**, **72**, **17**.

**The first two rows are the pair to memorise.** Fine-grained PAT out of scope → **404**, because
GitHub will not confirm the repository exists. App installation missing the repository → **403**,
because the identity is recognised and simply not permitted there.

</details>

---

## Q35

Match each person to the correct role.

| Person | Role |
|---|---|
| Platform lead who administers the organisation |  |
| Every developer |  |
| Finance analyst tracking Actions spend |  |
| Security team reviewing Dependabot alerts |  |
| Contractor working on one repository |  |

**Options:** Billing manager · Member · Outside collaborator · Owner · Security manager

<details>
<summary>Show answer</summary>

| Person | Role |
|---|---|
| Platform lead who administers the organisation | **Owner** |
| Every developer | **Member** |
| Finance analyst tracking Actions spend | **Billing manager** |
| Security team reviewing Dependabot alerts | **Security manager** |
| Contractor working on one repository | **Outside collaborator** |

**In `challenge-40.md`:** lines **249–253**.

**Line 249 caps Owners at two or three people**, and that number is testable. The exam likes offering
"add all senior engineers as Owners" and expecting you to reject it.

</details>

---

# Section F — Hot area

---

## Q36

```yaml
permissions:
  contents: [BLANK 1]
  issues: [BLANK 2]

jobs:
  demo:
    steps:
      - name: Create an issue using GITHUB_TOKEN
        run: gh issue create --title "..." --body "..."
        env:
          GH_TOKEN: ${{ secrets.[BLANK 3] }}
```

- **BLANK 1:** `read` / `write` / `none` / `admin`
- **BLANK 2:** `write` / `read` / `none` / `triage`
- **BLANK 3:** `GITHUB_TOKEN` / `PAT` / `APP_TOKEN` / `ACTIONS_TOKEN`

<details>
<summary>Show answer</summary>

### Answer: `read`, `write`, `GITHUB_TOKEN`

**In `challenge-40.md`:** lines **38–40** and **56**.

```yaml
permissions:
  contents: read
  issues: write
```

**Least privilege per scope, not one level for the whole block.** The job reads code and writes an
issue, so `contents` stays at `read` — creating an issue does not modify the repository's contents.

**And `secrets.GITHUB_TOKEN` needs no setup.** It appears in the `secrets` context automatically, which
is why the trap options all look like things somebody would have had to create.

</details>

---

## Q37

```yaml
      - name: Generate installation token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          [BLANK 1]: ${{ vars.CONTOSO_APP_ID }}
          [BLANK 2]: ${{ secrets.CONTOSO_APP_PRIVATE_KEY }}
          [BLANK 3]: contoso
```

- **BLANK 1:** `app-id` / `client-id` / `application` / `installation-id`
- **BLANK 2:** `private-key` / `password` / `client-secret` / `token`
- **BLANK 3:** `owner` / `repository` / `org-id` / `tenant`

<details>
<summary>Show answer</summary>

### Answer: `app-id`, `private-key`, `owner`

**In `challenge-40.md`:** lines **105–109**.

**`owner: contoso` is the one people leave out**, and it is what makes the token span the organisation's
installation rather than just the current repository. Omit it and cross-repository access — the entire
point of Task 2 — quietly disappears.

**And notice the split again:** the ID comes from `vars`, the key from `secrets`.

</details>

---

## Q38

```bash
gh api repos/contoso/webapp/collaborators/vendor-user -X [BLANK 1] \
  --field permission=[BLANK 2]
```

Requirement: a contractor must push code to this one repository and nothing else.

- **BLANK 1:** `PUT` / `POST` / `PATCH` / `GET`
- **BLANK 2:** `push` / `admin` / `pull` / `maintain`

<details>
<summary>Show answer</summary>

### Answer: `PUT`, `push`

**In `challenge-40.md`:** lines **263–264**.

**`push` is the API name for the Write role** (line 275): push to non-protected branches and merge pull
requests. `pull` would be read-only and would not meet "must push"; `admin` and `maintain` exceed it.

**`PUT` because the call is idempotent** — adding a collaborator who is already there is not an error,
which matters when the command runs from a script.

</details>

---

## Q39

```bash
gh api orgs/contoso/custom-repository-roles -X POST \
  --field name="Release Manager" \
  --field [BLANK 1]="write" \
  --field permissions[]="[BLANK 2]" \
  --field permissions[]="manage_releases"
```

- **BLANK 1:** `base_role` / `parent_role` / `inherits` / `template`
- **BLANK 2:** `manage_deploy_keys` / `delete_repository` / `admin` / `bypass_branch_protection`

<details>
<summary>Show answer</summary>

### Answer: `base_role`, `manage_deploy_keys`

**In `challenge-40.md`:** lines **283–289**.

```bash
  --field base_role="write" \
  --field permissions[]="manage_deploy_keys" \
  --field permissions[]="manage_releases" \
  --field permissions[]="edit_repo_metadata"
```

**A custom role is a base role plus named additions.** It starts from Write and adds exactly three
capabilities — it cannot subtract from the base, so choose the smallest base that works.

**Why the wrong options are wrong in kind, not just in name.** `delete_repository` is destructive,
`admin` is a role rather than a permission, and `bypass_branch_protection` would let a Release Manager
push straight past review — undoing Break scenario 1's careful, named exception.

</details>

---

## Q40

```bash
# Verify the app's granted permissions
gh api [BLANK 1] --jq '.permissions'

# Check whether the installation covers all repositories or selected ones
gh api /app/installations --jq '.[].[BLANK 2]'
```

- **BLANK 1:** `/app` / `/user` / `/orgs/contoso` / `/installation`
- **BLANK 2:** `repository_selection` / `permissions` / `account.login` / `target_type`

<details>
<summary>Show answer</summary>

### Answer: `/app`, `repository_selection`

**In `challenge-40.md`:** lines **174–178**.

**These are the two diagnostic calls for a 403** (Q7), and they answer different halves of the question.
`/app` says **what the app may do**; `repository_selection` says **where it may do it** — returning
`all` or `selected`.

**If it returns `selected`, that is your answer**, and the follow-up at line 341 lists exactly which
repositories are in.

</details>

---

## Q41

```yaml
  test:
    permissions:
      contents: [BLANK 1]
      [BLANK 2]: write
    steps:
      - uses: actions/checkout@v4
      - run: npm test
      - name: Publish test results
        uses: dorny/test-reporter@v1
```

- **BLANK 1:** `read` / `write` / `none`
- **BLANK 2:** `checks` / `issues` / `pull-requests` / `statuses`

<details>
<summary>Show answer</summary>

### Answer: `read`, `checks`

**In `challenge-40.md`:** lines **206–218**.

**`contents: read` must be written out even though the workflow already declares it** — a job-level
block replaces the workflow-level one rather than merging with it (Q13). Delete that line and
`actions/checkout` fails.

**`checks: write` is what `dorny/test-reporter` needs** to publish a check run. `statuses: write` is the
older commit-status API and is not what this action uses.

</details>

---

# Section G — Case study

## Case study: Contoso token remediation

### Background

Contoso Ltd's automation depends on **three classic PATs** created by a developer who **left six months
ago**. They carry `repo` and `admin:org` scope with **no expiration**, and they are shared across **15
repositories** by custom integrations, scheduled workflows and a deployment bot.

### Requirements

**Authentication**

- Workflows acting only on their own repository must use no stored credential
- Cross-repository automation must not depend on any individual's account
- Every automation credential must expire
- A formatter workflow must push to `main`, which requires pull-request reviews

**Access**

- The security team must review alerts across all repositories without code write
- Contractors must reach one repository only
- A release team must manage releases and deploy keys without full administrative access

**Policy**

- Classic PATs must stop working
- Workflows must default to read-only unless they declare otherwise
- Actions must not be able to approve pull requests

---

## Q42

Which credential should single-repository workflows use?

- A. `GITHUB_TOKEN` with a per-job `permissions` block
- B. A fine-grained PAT stored as a repository secret
- C. The existing classic PAT until migration completes
- D. A GitHub App installation token

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-40.md`:** lines **66–70** and **192–208**.

**It satisfies "no stored credential" by existing without one.** It is created per run, scoped to this
repository, and expires with the job — three requirements met by doing nothing.

**Why the others fail**

- **B** — a stored credential, tied to a person, where none is needed
- **C** — the finding being remediated
- **D** — correct for cross-repository work and unnecessary overhead here. **The exam rewards the
  *narrowest* mechanism that meets the requirement**, not the most capable one

</details>

---

## Q43

Which credential should the cross-repository deployment bot use?

- A. A GitHub App installation token
- B. A fine-grained PAT under a shared service account
- C. `GITHUB_TOKEN` with `contents: write`
- D. A classic PAT with a 90-day expiry

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-40.md`:** lines **74–76** and **116–122**.

**Only an app satisfies both halves at once:** it reaches other repositories, and it is not tied to any
individual.

**Why the others fail**

- **B** — a service account is still an account, with a holder, an owner and a shared credential
  (Q25)
- **C** — `GITHUB_TOKEN` cannot leave the repository. No permission changes that
- **D** — reintroduces the exact object being removed, with a shorter fuse

</details>

---

## Q44

How should the formatter workflow push to a protected `main`?

- A. Use a GitHub App token and add the app to the branch protection bypass list
- B. Add `contents: write` to `GITHUB_TOKEN`
- C. Remove the review requirement from `main`
- D. Push to a side branch and let the bot approve its own pull request

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-40.md`:** lines **308–329**.

**Both parts are required**, and this is what the exam checks: the app token supplies an identity, the
bypass entry supplies the permission. Either alone fails.

**Why D is the most interesting wrong answer.** It sounds like good practice — raise a PR instead of
pushing — but the *bot approving its own PR* is precisely what the last requirement forbids, and it is
disabled at the organisation level by line 243. A pull request nobody looked at is not a review.

</details>

---

## Q45

Which **two** access decisions satisfy the Access requirements? (Choose two.)

- A. Security Manager for the security team
- B. Outside collaborator for contractors
- C. Organization Member with write on all repositories for the security team
- D. Owner for the release team
- E. Admin repository role for contractors

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-40.md`:** lines **252–253**.

**Both are the only roles that fit their constraint exactly** — wide read without write for the
security team, and narrow named access for contractors.

**Why the others fail**

- **C** — grants the write access the requirement explicitly excludes
- **D** — organisation-wide administration for a repository-level need, and it breaks the two-to-three
  Owner cap at line 249
- **E** — repository administration for a contractor, including deletion

</details>

---

## Q46

How should the release team's requirement be met?

- A. A custom repository role with `base_role: write` plus `manage_releases` and `manage_deploy_keys`
- B. The Admin repository role
- C. The Maintain repository role
- D. Organization Owner

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-40.md`:** lines **279–294**.

```bash
  --field base_role="write" \
  --field permissions[]="manage_deploy_keys" \
  --field permissions[]="manage_releases" \
```

**Custom roles exist for exactly this shape of request:** a built-in role that is *almost* right, plus
two named capabilities, without jumping to the next role up and inheriting everything else with it.

**Why C is the closest wrong answer.** Maintain covers repository settings but is a fixed bundle — it
grants more than releases and deploy keys, and you cannot trim it. B and D are progressively worse for
the same reason.

</details>

---

## Q47

Six months after migration, a scheduled workflow that clones a second repository starts failing with
403. Nothing in the workflow changed, but the platform team recently created a new repository and moved
the deployment manifests into it.

What happened, and what is the fix?

- A. The app installation uses "Selected repositories" and the new repository was never added — add it
- B. The app's private key expired — regenerate it
- C. `GITHUB_TOKEN` lost `contents: read` — restore it
- D. The organisation blocked classic PATs — re-enable them

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-40.md`:** lines **333–348**.

```bash
gh api /app/installations/{installation_id}/repositories --jq '.repositories[].full_name'
```

**A new repository is not automatically in a "Selected repositories" installation.** The app's
permissions are unchanged and correct; its *reach* is what fell short — and 403 rather than 404 tells
you the identity was recognised.

**The operational lesson:** an installation scoped to selected repositories needs a step in whatever
process creates repositories, or this failure recurs every time. Choosing "All repositories" avoids it
at the cost of a much wider blast radius — a trade-off worth stating out loud rather than defaulting
into.

**Why the others fail**

- **B** — a bad key fails at token generation, before any clone
- **C** — the workflow uses an **app** token for the second repository; `GITHUB_TOKEN` could never have
  reached it
- **D** — classic PATs are not involved, and re-enabling them would reopen the original finding

</details>

---

## Q48

After the migration, an auditor asks Contoso to show who deployed to production last Tuesday and under
what authority.

What can Contoso now produce that it could not before, and what does this illustrate?

- A. An audit trail attributing the deployment to a named app installation with a scoped, expired token
- B. Nothing; app actions are not logged
- C. The workflow run log only, which is what it always had
- D. The classic PAT's usage history

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-40.md`:** the scenario at line **17** contrasted with lines **74–89** and **103–113**.

**Before, the honest answer was "we cannot tell you."** Three shared classic PATs, created by someone
who left, used by custom integrations, scheduled workflows and a bot across 15 repositories. Every
action in the log reads as the same departed user. Whether a deployment came from CI or from someone
who copied the token is unanswerable.

**After, each action is attributed to an app installation** whose permissions are declared (lines
165–171), whose reach is enumerable (line 341), and whose token expired shortly after the run — so the
credential that performed the action no longer exists to be misused.

**What it illustrates:** the reason to migrate is not only that PATs are risky to hold. It is that
**identity is what makes an audit possible**. A credential shared by many things can prove that
*something* happened; a credential belonging to one automation can prove *what*.

**And that is the sentence to give an exam case study that asks "why replace working PATs?"** — the
tokens were never the deliverable. Attribution was.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **`GITHUB_TOKEN` + permissions to reach another repo** | Q1, Q17, Q27, Q43 | Repository-scoped. Permissions control *what*, not *where* |
| **`contents: write` to fix the workflow-files error** | Q5, Q22 | Absolute rule. Only an App token plus a bypass entry works |
| **Org can cap classic PAT lifetime** | Q3, Q28 | Block or allow only. No lifetime policy |
| **403 vs 404** | Q7, Q10, Q34, Q47 | 404 = out of a PAT's scope. 403 = outside an installation |
| **Job permissions merge with workflow permissions** | Q13, Q30, Q41 | They **replace**. Repeat `contents: read` |
| **Shared service account as the PAT fix** | Q19, Q25, Q43 | Still an account, still shared, still unattributable |
| **Administration: write "so it is never blocked"** | Q8, Q26 | Lets the app edit the controls protecting it |
| **Revoking classic PATs first** | Q33 | Build and prove the replacement, then revoke |
| **Bot approving its own pull request** | Q44 | Disabled at org level. Not a review |
| **Member instead of Outside collaborator** | Q16, Q45 | Member is org-wide and outlives the contract |
| **Owner for anything short of org administration** | Q4, Q45, Q46 | Cap at 2–3 people |
| **Missing `owner:` on the app-token step** | Q37 | Without it, cross-repository reach disappears |

---

# What to memorise

**In `challenge-40.md`:** lines **66–72**, **165–171**, **247–253**, **271–277**.

```text
CREDENTIAL              REACH                     OWNER          EXPIRES
GITHUB_TOKEN            this repository only      automation     end of job
GitHub App token        the installation's repos  organisation   short-lived
Fine-grained PAT        selected repositories     a person       set, org can cap
Classic PAT             everything the user sees  a person       optional  <- ban it

GITHUB_TOKEN cannot:  reach another repo | trigger a workflow | write .github/workflows/
                      | bypass branch protection
```

```text
APP PERMISSIONS (minimum per use case)
CI build status      Checks: write, Contents: read
Auto-merge PRs       Pull requests: write, Contents: write   <- only row needing Contents: write
Deployment           Deployments: write, Contents: read, Environments: READ
Team notifications   Members: read                            <- only ORG permission
Security scanning    Security events: read, Contents: read

ORG ROLES     Owner (2-3 people) | Member | Billing manager | Security manager | Outside collaborator
REPO ROLES    Read | Triage (no code write) | Write | Maintain (no destructive) | Admin
              Custom role = base_role + named permissions
```

```yaml
# The two blocks worth writing from memory

permissions:            # workflow level = the default for every job
  contents: read

jobs:
  test:
    permissions:        # job level REPLACES the workflow level - repeat what you still need
      contents: read
      checks: write

      # deploy job adds:  id-token: write  (OIDC)   deployments: write  (record it)
```

```yaml
- uses: actions/create-github-app-token@v1
  id: app-token
  with:
    app-id: ${{ vars.CONTOSO_APP_ID }}              # vars - not sensitive
    private-key: ${{ secrets.CONTOSO_APP_PRIVATE_KEY }}   # secrets - the credential
    owner: contoso                                  # omit and you lose cross-repo reach
# then use it:  ${{ steps['app-token'].outputs.token }}
```

```text
DIAGNOSTICS
gh api /app --jq '.permissions'                                   what the app may do
gh api /app/installations --jq '.[].repository_selection'         all | selected
gh api /app/installations/{id}/repositories --jq '.repositories[].full_name'   where

POLICY  Org Settings > Personal access tokens:  classic = Do not allow
                                                fine-grained = require approval
        Org Settings > Actions > General:       workflow permissions = read
                                                uncheck "create and approve pull requests"
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 41 |
| 38–43 | Re-read the trap index and the reach table, then move on |
| 30–37 | Rewrite the four-row reach table and the permissions table from memory, then retake |
| Below 30 | Redo Tasks 1, 2 and 5 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 40.

:::danger The two sentences

**How far does it reach?** `GITHUB_TOKEN` = this repository. App = the installation. Fine-grained PAT =
the repositories you ticked. Classic PAT = everything the user can see.

**Who owns it?** The automation, or a person who can leave.

Answer those two and you have answered the question.

:::
