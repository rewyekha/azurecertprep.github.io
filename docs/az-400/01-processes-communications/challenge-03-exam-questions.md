---
sidebar_position: 3.5
toc_max_heading_level: 2
title: "Challenge 03: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 03 — AZ-400 exam questions

**48 questions** built only from what Challenge 03 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-03.md`**.

:::danger Read this before you start

Traceability is a **chain**, and every question in this paper is really asking **which link is missing**.

**Bug report → work item → pull request → commit → build → deployment.**

The chain is joined by two things and nothing else. **Text references** — `#87`, `AB#2100`, `Fixes` —
join the human end. **The commit SHA** joins the machine end: the merge commit is built, the build
produces an artifact stamped with that SHA, and the deployment records the SHA it shipped.

Two facts do most of the work here.

**A reference links; a keyword transitions.** `AB#2045` creates an association. `Fixes AB#2100` creates
the association **and moves the work item**. Same for GitHub: `#87` links, `Fixes #87` closes.

**Conventional Commits exist to make commit messages machine-readable.** `feat` bumps minor, `fix`
bumps patch, and a `!` or a `BREAKING CHANGE:` footer bumps major. That is what turns commit history
into a changelog and a version number instead of a diary.

The scenario at line 20 is the cost of a broken chain: the symptom found **in minutes**, and **four
hours** to answer which commit, who approved it, and which work item authorised it.

:::

---

# Section A — Multiple choice

---

## Q1

What is the difference between `AB#1234` and `Fixes AB#1234` in a commit message?

- A. `AB#1234` works only in PR descriptions; `Fixes AB#1234` works in commits
- B. `AB#1234` links Azure Boards; `Fixes AB#1234` links GitHub Issues
- C. `AB#1234` creates a link; `Fixes AB#1234` also transitions the item
- D. They behave identically in commits and in pull request descriptions

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-03.md`:** lines **144–158**.

```bash
# Simple reference (creates link, no state change)
...
AB#2045"

# Fix reference (creates link AND transitions to Done/Resolved)
...
Fixes AB#2100"
```

**The comments in the source say it outright**, and this is the most-tested fact in the challenge.

**Why the distinction matters operationally.** A board full of work items sitting in Active while the
code is already in production is a board nobody trusts — and that distrust is how Contoso ended up
spending four hours reconstructing what happened (line 20).

**Why B mixes the platforms.** `AB#` is Azure Boards in both forms. GitHub Issues use a bare `#`
(line 195).

</details>

---

## Q2

Which component of Conventional Commits indicates a breaking change?

- A. The `breaking` type prefix used in place of `feat` or `fix`
- B. Capitalising the entire subject line of the commit message
- C. Adding the text `[BREAKING]` anywhere in the commit body
- D. A `!` after the type or scope, or a `BREAKING CHANGE:` footer

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-03.md`:** lines **66–68**.

```text
feat(api)!: change authentication endpoint response format

BREAKING CHANGE: The /auth/token endpoint now returns a JSON object
```

**Either signal works, and both trigger a major version bump.** The `!` is compact; the footer explains
what consumers must change — and the example does both, which is the practice worth copying.

**Why A is the trap for anyone reading the type table quickly.** Look at lines 52–61: there is no
`breaking` type. Breaking is a **modifier** on a type, not a type of its own — `feat(api)!` is still a
feature.

**And read the rest of that footer block** (lines 72–73): `Reviewed-by: security-team` and
`Refs: AB#2001`. Footers are where the traceability metadata lives.

</details>

---

## Q3

In a full traceability chain, what connects a merged PR to its deployed artifact?

- A. The developer tags the deployment with the pull request number
- B. The merge commit SHA matches the build that produced the artifact
- C. Azure Boards records deployments against work items automatically
- D. CODEOWNERS maps each pull request to its deployment environment

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-03.md`:** lines **301–306**.

```bash
# Step 5: Build triggered by merge
gh run list --branch main --limit 5 --json databaseId,conclusion,headSha

# Step 6: Deployment from that build
gh api repos/{owner}/{repo}/deployments \
  --jq '.[] | select(.sha == "MERGE_COMMIT_SHA") | {id, environment, created_at}'
```

**The SHA is the join key**, and the query at line 306 is that join written out: select the deployment
whose `sha` equals the merge commit.

**Why A is what teams do when the chain is broken**, and why it fails: a manual tag is applied by
someone who remembers, at 2 AM, under pressure — which is exactly when it will not happen.

**The whole chain in one sentence:** issue number links to work item, work item links to PR, PR
contains commits, merge produces a SHA, the SHA identifies the build, the build's artifact is
deployed. **Text references at the top, SHA at the bottom.**

</details>

---

## Q4

Why does the commitlint workflow use `fetch-depth: 0`?

- A. To fetch full history so commitlint can read the PR range
- B. To enable shallow cloning so the checkout runs faster
- C. To download every branch in the repository, not just one
- D. To include submodules in the checkout alongside the repo

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-03.md`:** lines **448–450** and **480–483**.

```yaml
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
```

```bash
          npx commitlint \
            --from ${{ github.event.pull_request.base.sha }} \
            --to ${{ github.event.pull_request.head.sha }} \
```

**`actions/checkout` clones shallow by default — depth 1, one commit.** The linter is asked to inspect
every commit from base to head, and those commits are not in the clone.

**Why B is the exact inverse**, and it is the option that catches people who half-remember the flag.
`fetch-depth: 0` means **unlimited**, not shallow. Shallow is the default it is overriding.

**The same line appears on the traceability workflow** (line 327) and the work-item check (line 490),
for the same reason — both run `git log` over a commit range.

</details>

---

## Q5

`AB#1234` in commit messages is not creating links in Azure Boards. What is the cause?

- A. The commits in the pull request were squashed on merge
- B. The `AB#` syntax must be written entirely in lowercase
- C. Commit messages cannot link work items, only PR bodies can
- D. The Boards GitHub App is missing or the repo is not connected

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-03.md`:** line **538**.

```text
**Fix:** The Azure Boards GitHub App must be installed on the repository, and the repository must be
linked in the Azure DevOps project settings under Boards > GitHub connections. The `AB#` syntax only
works for repositories that are explicitly connected.
```

**Two sides of one handshake, and both are required.** The app grants GitHub's side; the connection
under Boards > GitHub connections grants Azure DevOps's side.

**Read the last sentence again: "explicitly connected".** A repository the organisation owns but has
not connected will happily accept `AB#2045` in a commit message and do nothing with it — no error, no
warning, just a reference that never becomes a link.

**Why C is refuted by lines 144–168**, where three commit messages carry work item references.

</details>

---

## Q6

Commitlint rejects `Merge branch 'main' into feature/xyz`. What is the correct fix?

- A. Disable commitlint for the repository altogether
- B. Add an `ignores` rule for commits starting with `Merge`
- C. Rewrite the merge commit message by hand each time
- D. Ban merge commits entirely with `allow_merge_commit`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-03.md`:** lines **552–559**.

```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
  ignores: [(commit) => commit.startsWith('Merge')],
  rules: {
    // existing rules
  }
};
```

**A merge commit's message is generated by Git, not written by a developer** — so holding it to a
human-authored convention punishes people for a message they did not choose.

**Why A is the over-correction the exam offers** whenever a control produces friction. Turning off the
linter to accommodate one generated message removes it for every real one.

**Why D is a defensible policy that is not this fix.** Banning merge commits is Challenge 01's
`allow_merge_commit=false` (line 305 there) — a different lever, and it would not help a developer who
merged `main` into their own branch locally.

</details>

---

## Q7

The traceability check fails on Dependabot PRs. What is the fix?

- A. Disable the traceability check for all pull requests
- B. Ask Dependabot to include issue numbers in its PR body
- C. Skip the check for bot accounts using `github.actor`
- D. Merge Dependabot pull requests manually without the check

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-03.md`:** lines **575–576**.

```yaml
      - name: Check PR links to issue or work item
        if: github.actor != 'dependabot[bot]' && github.actor != 'github-actions[bot]'
```

**A Dependabot PR has no authorising work item because no human requested it** — the trigger was a
published advisory. Demanding a reference invents one.

**Why `github.actor` and not the branch name.** A branch name is attacker-controlled; `github.actor` is
set by GitHub. This is the same guard as Challenge 44's auto-merge job.

**And note the exclusion is narrow: two named bots.** A blanket "skip for all bots" would let any
integration bypass traceability.

</details>

---

## Q8

Which commit type triggers a **minor** version bump?

- A. `fix`, which corrects behaviour without adding capability
- B. `perf`, which improves speed without changing behaviour
- C. `chore`, which covers maintenance with no consumer impact
- D. `feat`, which adds capability consumers can start using

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-03.md`:** lines **52–53**.

```text
| `feat` | New feature | Minor version bump |
| `fix` | Bug fix | Patch version bump |
```

**Feature = minor, fix = patch, breaking = major.** Three types drive the version; the other seven in
the table drive nothing.

**Why this table is worth memorising rather than reasoning about.** `perf` is the one people get wrong:
line 57 gives it a **patch** bump, because a performance improvement is a behaviour-preserving change
that users should still receive.

**And `docs`, `style`, `refactor`, `test`, `build`, `ci` and `chore` all bump nothing** — they change
the repository without changing what consumers get.

</details>

---

## Q9

What does `'type-enum': [2, 'always', [...]]` mean in the commitlint config?

- A. Severity 2 (error), always applied, restricted to those types
- B. Two commit types out of the list are permitted per commit
- C. The rule begins running from the second commit in the range
- D. It emits a warning only after two separate violations occur

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-03.md`:** lines **92–95**.

```javascript
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style', 'refactor',
      'perf', 'test', 'build', 'ci', 'chore', 'revert'
    ]],
```

**The first element is severity: 0 disabled, 1 warning, 2 error.** That number is the whole question.

**Compare with `scope-enum` on the next line** (line 96): severity **1**. Scopes are a *warning* — an
unlisted scope is flagged and the commit is allowed — while an unlisted **type** is an error and blocks
the commit. The team chose to be strict about types and lenient about scopes, and the difference is one
digit.

**`references-empty` at line 102 is also severity 1**, with `'never'`: warn when a commit has no
reference, but do not block.

</details>

---

## Q10

Which hook does husky create to validate a commit message?

- A. `pre-commit`, which runs before the message is written
- B. `post-merge`, which runs after a merge completes
- C. `commit-msg`, which receives the message file path
- D. `pre-push`, which runs before refs are sent upstream

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-03.md`:** lines **111–113**.

```bash
echo 'npx --no -- commitlint --edit "$1"' > .husky/commit-msg
chmod +x .husky/commit-msg
```

**`commit-msg` receives the path to the message file as `$1`**, which is what `--edit "$1"` reads. No
other hook is handed the message.

**Why A is the plausible near-miss.** `pre-commit` runs **before** the message exists — it is where
linters and formatters go. At that point there is nothing for commitlint to inspect.

</details>

---

## Q11

Which three keywords close a GitHub issue when a PR merges?

- A. `done`, `complete` and `finished`, in any capitalisation
- B. `AB#`, `WI#` and `REF#`, followed by the item number
- C. `merge`, `ship` and `deploy`, followed by the issue number
- D. `closes`, `fixes` and `resolves`, in any capitalisation

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-03.md`:** lines **195–199**.

```text
- `closes #123`
- `fixes #123`
- `resolves #123`
```

```text
Case-insensitive variations all work: `Close`, `FIXES`, `Resolves`.
```

**Case-insensitive, and the variants conjugate** — `close`, `closed`, `closes` all work, which is why
the workflow's regex at line 336 is written as `close[sd]?|fix(e[sd])?|resolve[sd]?`.

**One constraint the exam likes:** the keyword closes the issue only when the PR merges **to the default
branch** (line 193). Merging into a release branch links without closing.

</details>

---

## Q12

What does the traceability workflow do when a PR has no issue or work item reference?

- A. It fails the check by calling `core.setFailed` with a message
- B. It logs a warning to the run log and lets the check pass
- C. It adds a label to the pull request marking it untraceable
- D. It closes the pull request and comments on the issue thread

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-03.md`:** lines **340–344**.

```javascript
            if (!hasIssueRef && !hasABRef && !hasLinkedIssue) {
              core.setFailed(
                'PR must reference at least one issue (#123) or work item (AB#123). ' +
                'Add "Fixes #<number>" or "AB#<number>" to the PR description.'
              );
```

**`core.setFailed` fails the job**, which is what makes this a gate rather than a report — provided the
job is a required status check (Challenge 01 Q46).

**Contrast it with the deliberately softer check in the same challenge** at lines 500–501, which uses
`::warning::` for missing work-item references **in commits**. The PR-level reference is mandatory; the
commit-level one is encouraged.

**And note the message tells the developer exactly what to type.** An error that names the fix is worth
three that describe the problem.

</details>

---

## Q13

What does the `hasLinkedIssue` check add beyond `hasIssueRef`?

- A. It confirms the referenced issue actually exists in the repo
- B. It accepts a bare `#123` mention with no closing keyword
- C. It checks the pull request title as well as the body text
- D. It requires an Azure Boards reference as well as the issue

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-03.md`:** lines **336–338**.

```javascript
            const hasIssueRef = /(close[sd]?|fix(e[sd])?|resolve[sd]?)\s+#\d+/i.test(prBody);
            const hasABRef = /AB#\d+/i.test(prBody) || /AB#\d+/i.test(prTitle);
            const hasLinkedIssue = prBody.match(/#\d+/);
```

**Three progressively looser tests, joined by OR at line 340.** The check demands *some* traceability,
not necessarily a closing one — because a PR that is part of a larger issue should reference it without
closing it.

**Note the asymmetry worth spotting: `hasABRef` also searches the PR title**, while the issue checks
look only at the body. Azure Boards references are conventionally put in the title, GitHub references
in the body.

</details>

---

## Q14

What does the commit-message validation step in the traceability workflow compare against?

- A. The `origin/main..HEAD` range of the branch's own commits
- B. The entire history of the repository from the root
- C. The last ten commits on the pull request branch
- D. Only the tip commit of the repository default branch

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-03.md`:** line **353**.

```bash
          COMMITS=$(git log --format="%s" origin/main..HEAD)
```

**The double-dot range means "commits on HEAD that are not on `origin/main`"** — precisely the PR's own
commits, and nothing the developer inherited.

**Why that scoping is essential rather than tidy.** Linting the entire history would fail on every
pre-convention commit in the repository, forever, and the check would be deleted within a day.

**`--format="%s"` takes only the subject line**, which is what the pattern at line 354 validates.

</details>

---

## Q15

Which audit-log query finds branch protection overrides?

- A. `-f phrase='action:repo'` to list repository-level events
- B. `-f phrase='action:protected_branch.policy_override repo:...'`
- C. `-f phrase='actor:username'` to list one person's actions
- D. `--jq '.[] | select(.action == "push")'` to filter pushes

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-03.md`:** lines **394–397**.

```bash
gh api orgs/{org}/audit-log \
  --method GET \
  -f phrase='action:protected_branch.policy_override repo:contoso-org/contoso-payments' \
  --jq '.[] | {actor, action, created_at}'
```

**A policy override is somebody using a bypass**, and it is the single most audit-relevant event on a
protected branch — the answer to "why was this allowed to merge?" from the scenario at line 20.

**The `phrase` syntax composes with spaces**, so `action:` and `repo:` narrow together. That is how the
same endpoint serves the three different queries at lines 380, 387 and 394.

**And the endpoint is organisation-scoped and admin-only** — a repository-level token will not see it.

</details>

---

## Q16

What does Azure DevOps audit **streaming** provide over querying the audit log?

- A. Faster queries against the audit log held in Azure DevOps
- B. Longer retention of the audit log inside Azure DevOps itself
- C. Alerting whenever a work item changes state on the board
- D. Continuous export to a destination such as Log Analytics

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-03.md`:** lines **412–427**.

```json
{
  "consumerType": "AzureMonitorLogs",
  "consumerInputs": {
    "WorkspaceId": "<workspace-id>",
    "SharedKey": "<workspace-key>"
  },
  "displayName": "Contoso Audit Stream"
}
```

**Querying is pull and bounded; streaming is push and continuous.** The query at line 409 takes an
explicit start and end time because the log inside Azure DevOps has a limited window.

**And the destination is the point.** Once audit events are in Log Analytics they can be queried with
KQL alongside deployment and application telemetry (Challenge 49) — which is how "who changed this, and
did it correlate with the incident?" becomes one query instead of three systems.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** commit types trigger a version bump? (Choose three.)

- A. `feat`, which produces a minor version bump
- B. `docs`, which documents without changing code
- C. `chore`, which covers repository maintenance
- D. `fix`, which produces a patch version bump
- E. `style`, which changes formatting alone
- F. `perf`, which produces a patch version bump

<details>
<summary>Show answer</summary>

### Answer: A, D, F

**In `challenge-03.md`:** lines **52–61**.

```text
| `feat` | New feature | Minor version bump |
| `fix` | Bug fix | Patch version bump |
| `perf` | Performance improvement | Patch version bump |
```

**`perf` is the one candidates miss.** It reads like housekeeping and it is not — a performance change
alters runtime behaviour users should receive, so it earns a patch release.

**The seven that bump nothing** — `docs`, `style`, `refactor`, `test`, `build`, `ci`, `chore` — change
the repository without changing the artifact consumers install.

</details>

---

## Q18

Which **three** are steps in the full traceability chain the challenge builds? (Choose three.)

- A. A spreadsheet of releases kept by the release manager
- B. A bug report raised as a GitHub issue by the QA team
- C. A work item in Azure Boards linked to that issue
- D. A weekly status email summarising what shipped
- E. A deployment selected by matching the merge SHA
- F. A deployment tag applied by hand after release

<details>
<summary>Show answer</summary>

### Answer: B, C, E

**In `challenge-03.md`:** lines **289–306**.

```bash
# Step 1: Bug is reported (GitHub Issue #87)
# Step 2: Work item created/linked (Azure Boards AB#2100)
# Step 3: PR created referencing both (#42)
# Step 4: Commits in the PR
# Step 5: Build triggered by merge
# Step 6: Deployment from that build
```

**Six steps, and each is one command** — which is the demonstration that matters. If any link needs a
human to remember something, the chain breaks under pressure, which is what produced the four-hour
investigation at line 20.

**Why A, D and F are the alternatives teams build when the chain is broken**, and all three depend on
somebody maintaining them by hand.

</details>

---

## Q19

Which **two** signals mark a breaking change? (Choose two.)

- A. A `breaking` type in place of `feat` or `fix`
- B. A `!` placed after the type or the scope
- C. An upper-case subject line on the commit
- D. A `BREAKING CHANGE:` footer in the body
- E. A `major` scope such as `feat(major):`

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-03.md`:** lines **66–70**.

```text
feat(api)!: change authentication endpoint response format

BREAKING CHANGE: The /auth/token endpoint now returns a JSON object
```

**Either is sufficient; the example uses both because they do different jobs.** The `!` is the machine
signal that drives the major bump; the footer is the human explanation of what consumers must change.

**Why A is the trap worth stating twice** (Q2): there is no `breaking` type in the table at lines
52–61. Breaking is a modifier on an existing type.

</details>

---

## Q20

Which **two** are true about GitHub closing keywords? (Choose two.)

- A. They close the issue on merge into any target branch
- B. They also transition Azure Boards work items directly
- C. `closes`, `fixes` and `resolves` all close the issue
- D. Only `closes` works when placed in a pull request body
- E. They are case-insensitive in every documented form

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-03.md`:** lines **195–199**.

```text
Case-insensitive variations all work: `Close`, `FIXES`, `Resolves`.
```

**Why A is the constraint the exam tests.** Line 193 says "when the PR merges **to the default
branch**". Merge the same PR into `release/2.4` and the issue is linked but stays open — which is
correct behaviour, since the fix has not reached production.

**Why B crosses the platforms.** Azure Boards needs `AB#` and the `Fixes` keyword together (Q1).

</details>

---

## Q21

Which **two** checks does the commit-lint workflow run? (Choose two.)

- A. A unit test suite run over the changed packages
- B. `commitlint` run over the pull request's commit range
- C. A dependency security scan of the lockfile
- D. A search for work item references, warning if absent
- E. A coverage gate comparing against the base branch

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-03.md`:** lines **445–483** and **485–502**.

```bash
          npx commitlint --from ${{ github.event.pull_request.base.sha }} --to ${{ github.event.pull_request.head.sha }} --verbose
```

```bash
          if echo "$COMMITS" | grep -qE "(AB#[0-9]+|#[0-9]+|Fixes|Closes|Resolves)"; then
            echo "Work item reference found in commits"
          else
            echo "::warning::No work item reference found in commits."
```

**Two jobs, two severities, and the asymmetry is deliberate.** Message **format** is an error — it is
mechanical and there is no excuse. A work item reference **in every commit** is a warning, because a
refactoring commit inside a referenced PR legitimately has none.

**The mandatory reference is enforced at PR level instead** (Q12), which is the right granularity.

</details>

---

## Q22

Which **two** are needed for `AB#` links to work? (Choose two.)

- A. A PAT for Azure DevOps stored in the repository secrets
- B. The Azure Boards GitHub App installed on the repository
- C. Commitlint configured to validate the `AB#` reference
- D. Branch protection enabled on the default branch
- E. The repository connected under Boards > GitHub connections

<details>
<summary>Show answer</summary>

### Answer: B, E

**In `challenge-03.md`:** line **538**.

**Two sides, one handshake** (Q5). The app is the GitHub half; the connection is the Azure DevOps half.

**Why the failure is so hard to spot.** With only one half configured, `AB#2045` sits in commit messages
looking exactly like a working reference. Nothing errors — the link simply never appears, and it is
usually noticed weeks later during a release review.

</details>

---

## Q23

Which **two** does the audit configuration provide? (Choose two.)

- A. Blocking policy overrides on protected branches
- B. Querying the GitHub organisation audit log by action and actor
- C. Automatic rollback of unauthorised configuration changes
- D. Streaming Azure DevOps audit events to Log Analytics
- E. Commit signing enforced across the organisation

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-03.md`:** lines **380–397** and **412–427**.

**Both are detective controls, and it is worth being precise about that.** They record what
happened; they prevent nothing.

**Why A is the specific misconception.** `protected_branch.policy_override` is an event **type you can
search for** (line 396) — evidence that a bypass occurred, which is only useful because the bypass was
possible. Preventing it is branch protection's job (Challenge 01).

**And that pairing is the answer to the scenario's second question.** "Understand why it was approved"
needs the audit log; "stop it being approvable" needs the protection rule.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must be able to trace any production error to the commit that caused it, the pull
request that introduced it, the review that approved it, and the work item that authorised it —
automatically, with no gaps.

---

## Q24

**Proposed solution:** Adopt Conventional Commits and enforce them with commitlint via a husky
`commit-msg` hook locally and a required CI check over the PR's commit range with `fetch-depth: 0`.
Install the Azure Boards GitHub App and connect the repository in Azure DevOps. Require every PR to
reference an issue or work item, failing the check when none is present, and skip the check for bots.
Link work with `Fixes #n` and `Fixes AB#n`. Correlate build and deployment by merge commit SHA. Query
the GitHub audit log for policy overrides and stream Azure DevOps audit events to Log Analytics.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-03.md`:** lines **88–113**, **437–483**, **538**, **340–344**, **575–576**,
**249–276**, **301–306**, **394–397**, **412–427**.

| Question the incident could not answer | Mechanism |
|---|---|
| Which commit? | Deployment SHA → build → merge commit → PR |
| Which change was it? | Conventional commit type and scope |
| Which work item authorised it? | `Fixes AB#n`, enforced at PR level |
| Who approved it, and were rules bypassed? | Audit log, `policy_override` |
| Will the next one be traceable too? | Required checks, not conventions |

**The local hook plus the CI check is deliberate redundancy, not duplication.** The hook gives the
developer feedback in two seconds; the CI check is the one that cannot be skipped with `--no-verify`.

</details>

---

## Q25

**Proposed solution:** Ask developers to follow Conventional Commits and to mention the work item number
somewhere in the PR. Install the Azure Boards GitHub App. Use `AB#2100` in commit messages. Review the
audit log if an incident occurs.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures, and every one of them is the difference between a convention and a control.**

**"Ask developers to follow" is what the team already does.** Nothing validates the format, so the
changelog and version derivation the convention exists for cannot be automated.

**"Mention somewhere" is not a reference.** The check at line 336 looks for specific patterns; free text
saying "this is for the auth work" links nothing and is invisible to any query.

**Installing the app without connecting the repository in Azure DevOps** leaves `AB#` inert (Q5, Q22) —
and it fails silently, so the team believes traceability is working.

**And "review the audit log if an incident occurs" is the current state**, which took four hours
(line 20). The audit log answers *who bypassed what*; it does not reconstruct the chain from deployment
back to work item.

</details>

---

## Q26

**Proposed solution:** Adopt Conventional Commits enforced by commitlint locally and in CI with
`fetch-depth: 0`. Install the Azure Boards GitHub App and connect the repository. Require a work item or
issue reference on every PR and skip the check for bots. Link with `Fixes #n` and `Fixes AB#n`.
Correlate build and deployment by merge commit SHA. Stream audit events to Log Analytics. To keep the CI
fast on a repository with deep history, use the default shallow checkout in the commitlint job.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**A shallow checkout is depth 1: one commit.** Commitlint is asked to validate the range
`base.sha..head.sha` (lines 481–482), and on a shallow clone those commits are not present — so the
command either errors on an unknown revision or validates nothing at all.

**The second outcome is the dangerous one.** A linter that inspects zero commits reports success. The
job is green, the required check passes, and every commit message on the branch is unvalidated — which
is exactly the state the challenge is trying to leave.

**And the stated justification is wrong on the facts, which is why the option is convincing.** The
concern about deep history is real, and it is why `fetch-depth: 0` is not a default. But the correct
answer to slow checkout is a **fetch depth large enough for the range** — not a depth that excludes the
data the job exists to read.

**The same line matters in three jobs here**: commitlint (line 450), the traceability commit check
(line 327), and the work-item scan (line 490). All three run `git log` over a range.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — Conventional Commits

| # | Statement | Answer |
|---|---|---|
| 1 | `feat` triggers a minor version bump |  |
| 2 | `perf` triggers a patch version bump |  |
| 3 | There is a `breaking` commit type |  |
| 4 | `BREAKING CHANGE:` in a footer signals a major bump |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `feat` triggers a minor version bump | **Yes** |
| 2 | `perf` triggers a patch version bump | **Yes** |
| 3 | There is a `breaking` commit type | **No** |
| 4 | `BREAKING CHANGE:` in a footer signals a major bump | **Yes** |

**In `challenge-03.md`:** lines **52**, **57**, **52–61**, **68**.

Row 2 is the one that is missed. Row 3 is the trap that survives into the exam because "breaking" feels
like it ought to be a type.

</details>

---

## Q28 — linking

| # | Statement | Answer |
|---|---|---|
| 1 | `AB#2045` links without transitioning |  |
| 2 | `Fixes AB#2100` transitions the work item |  |
| 3 | `fixes #123` is case-sensitive |  |
| 4 | Closing keywords act on merge to the default branch |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `AB#2045` links without transitioning | **Yes** |
| 2 | `Fixes AB#2100` transitions the work item | **Yes** |
| 3 | `fixes #123` is case-sensitive | **No** |
| 4 | Closing keywords act on merge to the default branch | **Yes** |

**In `challenge-03.md`:** lines **144**, **152**, **199**, **193**.

Row 4 is the constraint that turns up as a distractor: merging into a release branch links the issue and
leaves it open, correctly.

</details>

---

## Q29 — enforcement

| # | Statement | Answer |
|---|---|---|
| 1 | A husky hook can be bypassed with `--no-verify` |  |
| 2 | `core.setFailed` fails the workflow job |  |
| 3 | `::warning::` fails the job |  |
| 4 | A CI job blocks a merge only when it is a required check |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A husky hook can be bypassed with `--no-verify` | **Yes** |
| 2 | `core.setFailed` fails the workflow job | **Yes** |
| 3 | `::warning::` fails the job | **No** |
| 4 | A CI job blocks a merge only when it is a required check | **Yes** |

**In `challenge-03.md`:** lines **112**, **341**, **500**, and Challenge 01's line 467.

Rows 1 and 4 together are why the design has both layers. **The hook is fast and skippable; the required
check is slow and not.**

Row 3 is the deliberate softness on commit-level work item references (Q21).

</details>

---

## Q30 — audit and traceability

| # | Statement | Answer |
|---|---|---|
| 1 | The merge commit SHA joins build to deployment |  |
| 2 | The GitHub audit log can be filtered by action and repository |  |
| 3 | Audit streaming prevents unauthorised changes |  |
| 4 | `fetch-depth: 0` is needed to lint a PR's commit range |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | The merge commit SHA joins build to deployment | **Yes** |
| 2 | The GitHub audit log can be filtered by action and repository | **Yes** |
| 3 | Audit streaming prevents unauthorised changes | **No** |
| 4 | `fetch-depth: 0` is needed to lint a PR's commit range | **Yes** |

**In `challenge-03.md`:** lines **301–306**, **394–396**, **412–427**, **448–450**.

Row 3 is the preventive-versus-detective line the exam draws in every domain. **Streaming records;
branch protection prevents.**

</details>

---

# Section E — Drag and drop

---

## Q31

Match each link in the chain to what joins it to the next.

| Link | Joined by |
|---|---|
| Bug report → work item |  |
| Work item → pull request |  |
| Pull request → commits |  |
| Commits → build |  |
| Build → deployment |  |
| Any of these → who approved it |  |

**Options:** An `AB#` reference, or a linked issue · `Fixes AB#n` in the PR description · The audit log · The deployment's recorded SHA · The merge commit SHA · The PR's commit list

<details>
<summary>Show answer</summary>

| Link | Joined by |
|---|---|
| Bug report → work item | **An `AB#` reference, or a linked issue** |
| Work item → pull request | **`Fixes AB#n` in the PR description** |
| Pull request → commits | **The PR's commit list** |
| Commits → build | **The merge commit SHA** |
| Build → deployment | **The deployment's recorded SHA** |
| Any of these → who approved it | **The audit log** |

**In `challenge-03.md`:** lines **289–306** and **394–397**.

**Text references at the top of the chain, the SHA at the bottom.** That split is the structure of every
traceability question on this exam.

</details>

---

## Q32

Match each commit type to its version effect.

| Type | Effect |
|---|---|
| `feat` |  |
| `fix` |  |
| `perf` |  |
| `docs`, `style`, `refactor`, `test`, `build`, `ci`, `chore` |  |
| `feat(api)!` or a `BREAKING CHANGE:` footer |  |

**Options:** Major · Minor · None · Patch

<details>
<summary>Show answer</summary>

| Type | Effect |
|---|---|
| `feat` | **Minor** |
| `fix` | **Patch** |
| `perf` | **Patch** |
| `docs`, `style`, `refactor`, `test`, `build`, `ci`, `chore` | **None** |
| `feat(api)!` or a `BREAKING CHANGE:` footer | **Major** |

**In `challenge-03.md`:** lines **52–61** and **66–68**.

**Three rows carry the whole table.** Feature, fix and performance move the version; everything else
moves the repository.

</details>

---

## Q33

Arrange the traceability chain from production symptom back to authorisation.

**Items:** Find the pull request containing that commit · Find the deployment that shipped to production
· Find the work item the PR references · Find the build whose SHA matches the deployment · Find the
merge commit the build ran on

<details>
<summary>Show answer</summary>

### Answer

1. Find the deployment that shipped to production — line **305**
2. Find the build whose SHA matches the deployment — line **302**
3. Find the merge commit the build ran on — line **299**
4. Find the pull request containing that commit — line **296**
5. Find the work item the PR references — line **293**

**Walk it backwards, because that is the direction an incident runs.** You start with a symptom in
production and you need the authorisation — which is the reverse of the order the chain was built in
(lines 289–306).

**And every step is a lookup by an identifier the previous step handed you.** The deployment gives a
SHA, the SHA gives a build and a commit, the commit gives a PR, the PR gives a work item. **No step
requires anyone to remember anything**, which is the entire difference from the four-hour investigation
at line 20.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| `AB#` references create no links |  |
| Commitlint passes but validates nothing |  |
| Work item stays Active after the fix ships |  |
| Every PR from Dependabot fails traceability |  |
| A generated merge commit fails the linter |  |
| An issue stays open after merge |  |

**Options:** `AB#n` used without the `Fixes` keyword · App not installed, or repo not connected in Azure DevOps · Merged into a non-default branch · No bot exclusion on the check · No `ignores` rule for `Merge` commits · Shallow checkout — the commit range is absent

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| `AB#` references create no links | **App not installed, or repo not connected in Azure DevOps** |
| Commitlint passes but validates nothing | **Shallow checkout — the commit range is absent** |
| Work item stays Active after the fix ships | **`AB#n` used without the `Fixes` keyword** |
| Every PR from Dependabot fails traceability | **No bot exclusion on the check** |
| A generated merge commit fails the linter | **No `ignores` rule for `Merge` commits** |
| An issue stays open after merge | **Merged into a non-default branch** |

**In `challenge-03.md`:** lines **538**, **448–450**, **144**, **576**, **555**, **193**.

**The first two are the silent ones.** Nothing errors in either case — one produces references that
never become links, the other produces a green check that inspected nothing.

</details>

---

## Q35

Match each control to whether it prevents or records.

| Control | Type |
|---|---|
| Husky `commit-msg` hook |  |
| Commitlint as a required CI check |  |
| PR traceability check with `core.setFailed` |  |
| Commit-level work item scan with `::warning::` |  |
| GitHub audit log |  |
| Azure DevOps audit streaming |  |

**Options:** Prevents — not bypassable · Prevents locally — bypassable with `--no-verify` · Prevents, when required · Records · Records — informational only · Records, and retains off-platform

<details>
<summary>Show answer</summary>

| Control | Type |
|---|---|
| Husky `commit-msg` hook | **Prevents locally — bypassable with `--no-verify`** |
| Commitlint as a required CI check | **Prevents — not bypassable** |
| PR traceability check with `core.setFailed` | **Prevents, when required** |
| Commit-level work item scan with `::warning::` | **Records — informational only** |
| GitHub audit log | **Records** |
| Azure DevOps audit streaming | **Records, and retains off-platform** |

**In `challenge-03.md`:** lines **112**, **478–483**, **341**, **500**, **380**, **412–427**.

**Rank the top three by skippability, because that is what the exam grades.** A hook is a courtesy; a
required check is a control; an audit log is evidence that the control was or was not applied.

</details>

---

# Section F — Hot area

---

## Q36

```text
[BLANK 1](api)[BLANK 2]: change authentication endpoint response format

[BLANK 3]: The /auth/token endpoint now returns a JSON object
with 'access_token' and 'refresh_token' fields instead of a flat token string.

Refs: AB#2001
```

Requirement: a new feature that breaks existing API consumers.

- **BLANK 1:** `fix` / `breaking` / `feat` / `chore`
- **BLANK 2:** `*` / `!` / `#` / *(nothing)*
- **BLANK 3:** `BREAKING` / `MAJOR` / `INCOMPATIBLE` / `BREAKING CHANGE`

<details>
<summary>Show answer</summary>

### Answer: `feat`, `!`, `BREAKING CHANGE`

**In `challenge-03.md`:** lines **66–73**.

**The type stays `feat` because the change *is* a feature** — the `!` is what escalates it to a major
bump. There is no `breaking` type (Q2, Q19).

**And the footer keyword is `BREAKING CHANGE` with a space**, not an underscore or a hyphen. Tooling
matches it literally.

</details>

---

## Q37

```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [[BLANK 1], 'always', ['feat', 'fix', ...]],
    'scope-enum': [[BLANK 2], 'always', ['api', 'auth', 'db', ...]],
    'subject-max-length': [2, 'always', [BLANK 3]]
  }
};
```

Requirement: an unlisted type must block the commit; an unlisted scope should only warn.

- **BLANK 1:** `1` / `2` / `0`
- **BLANK 2:** `2` / `0` / `1`
- **BLANK 3:** `50` / `100` / `72` / `120`

<details>
<summary>Show answer</summary>

### Answer: `2`, `1`, `72`

**In `challenge-03.md`:** lines **92–99**.

**Severity is the first element: 0 off, 1 warning, 2 error.** The requirement maps directly onto the two
numbers, and the challenge's own config makes exactly this choice.

**`100` is the distractor because it appears twice nearby** — as `body-max-line-length` and
`footer-max-line-length` (lines 100–101). The **subject** limit is 72.

</details>

---

## Q38

```bash
# .husky/[BLANK 1]
npx --no -- commitlint --[BLANK 2] "$1"
```

- **BLANK 1:** `pre-commit` / `pre-push` / `post-commit` / `commit-msg`
- **BLANK 2:** `read` / `edit` / `file` / `message`

<details>
<summary>Show answer</summary>

### Answer: `commit-msg`, `edit`

**In `challenge-03.md`:** line **112**.

```bash
echo 'npx --no -- commitlint --edit "$1"' > .husky/commit-msg
```

**`commit-msg` is the only hook handed the message file path**, and `$1` is that path. `--edit` tells
commitlint to read the message from a file rather than from stdin.

**Do not forget `chmod +x`** on the next line — a hook without the execute bit is silently skipped, which
is a favourite lab failure.

</details>

---

## Q39

```yaml
      - uses: actions/checkout@v4
        with:
          fetch-depth: [BLANK 1]

      - name: Lint commits in PR
        run: |
          npx commitlint \
            --from ${{ github.event.pull_request.[BLANK 2] }} \
            --to ${{ github.event.pull_request.head.sha }}
```

- **BLANK 1:** `1` / `10` / `0` / `50`
- **BLANK 2:** `merge_commit_sha` / `base.sha` / `head.ref` / `number`

<details>
<summary>Show answer</summary>

### Answer: `0`, `base.sha`

**In `challenge-03.md`:** lines **450** and **481**.

**`0` means unlimited history, not "no history"** — the inversion that makes this a good exam question
(Q4, Q26).

**And `base.sha` is where the PR diverged**, so `base..head` is exactly the set of commits the author
added. `merge_commit_sha` does not exist until the PR merges.

</details>

---

## Q40

```javascript
const hasIssueRef = /(close[sd]?|fix(e[sd])?|resolve[sd]?)\s+#\d+/i.test(prBody);
const hasABRef = /AB#\d+/i.test(prBody) || /AB#\d+/i.test([BLANK 1]);

if (!hasIssueRef && !hasABRef && !hasLinkedIssue) {
  core.[BLANK 2]('PR must reference at least one issue or work item');
}
```

- **BLANK 1:** `prBody` / `context.sha` / `prTitle` / `github.actor`
- **BLANK 2:** `warning` / `info` / `notice` / `setFailed`

<details>
<summary>Show answer</summary>

### Answer: `prTitle`, `setFailed`

**In `challenge-03.md`:** lines **337** and **341**.

**Only the Azure Boards check also searches the title** (Q13) — a deliberate asymmetry, because `AB#`
references are conventionally placed there.

**`core.setFailed` is what makes it a gate.** `core.warning` would print a yellow annotation and let the
PR merge, which is the softer treatment applied to the *commit-level* scan at line 500.

</details>

---

## Q41

```bash
gh api orgs/{org}/audit-log \
  --method GET \
  -f phrase='[BLANK 1] repo:contoso-org/contoso-payments' \
  --jq '.[] | {actor, action, created_at}'
```

Requirement: find occasions where branch protection was bypassed.

- **BLANK 1:** `action:push` / `actor:admin` / `action:repo.create` /
  `action:protected_branch.policy_override`

<details>
<summary>Show answer</summary>

### Answer: `action:protected_branch.policy_override`

**In `challenge-03.md`:** line **396**.

**A policy override is the audit event for "somebody used a bypass"**, and it answers the second of the
scenario's three questions: *understand why it was approved*.

**Note the phrase composes two filters separated by a space** — the action and the repository — which is
how the same endpoint serves the actor query at line 388 and the branch-protection query here.

</details>

---

# Section G — Case study

## Case study: Contoso payments traceability

### Background

Last Thursday at **2:47 AM**, Contoso's production payment service began returning **500 errors for 12%
of transactions**. The on-call engineer identified the symptom **within minutes**, but it took **four
hours** to trace the error to a specific commit, understand why it was approved, and determine which
work item authorised the change. The root cause was a **database migration that passed all tests in
isolation** but conflicted with a concurrent schema change from another team. The CTO has mandated
end-to-end traceability with **no gaps in the audit chain**.

### Requirements

**Commit hygiene**

- Commit messages must be machine-readable, so changelogs and versions can be derived
- The format must be validated before the commit exists locally, and again where it cannot be skipped
- Generated merge commits must not be treated as violations

**Linking**

- Every pull request must reference an issue or work item, enforced
- Merging a fix must move its work item without anyone updating a board
- Automated dependency PRs must not be blocked by the reference requirement

**Audit**

- It must be possible to show who approved a change and whether any protection was bypassed
- Audit evidence must survive beyond the platform's own retention window

---

## Q42

How should commit format be enforced?

- A. A husky `commit-msg` hook on each developer's machine alone
- B. A note in CONTRIBUTING.md describing the required format
- C. A husky hook locally plus commitlint as a required CI check
- D. A commitlint CI check on the pull request range alone

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-03.md`:** lines **107–113** and **437–483**.

**The requirement asks for both, in its own words: "before the commit exists locally, and again where it
cannot be skipped."**

**Why A alone fails.** A hook lives in the developer's working copy and is bypassed by `--no-verify`, by
a fresh clone where nobody ran `husky init`, and by any commit made through a web UI.

**Why D alone is worse for the developer.** Discovering a bad message after pushing means an interactive
rebase to fix history — the hook catches it in two seconds, before it exists.

**Together they are fast feedback plus a guarantee**, which is the same pattern as Challenge 43's
pre-commit hooks and push protection.

</details>

---

## Q43

How is "merging a fix must move its work item" satisfied?

- A. `Fixes AB#2100`, with the app installed and repo connected
- B. A nightly job that syncs board state from merged PRs
- C. A workflow that calls the Azure Boards API on merge
- D. A bare `AB#2100` reference in the commit message

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-03.md`:** lines **152–158** and **538**.

**D is the near-miss that defines this challenge.** A bare `AB#2100` creates the link and leaves the
work item exactly where it was — so the board says Active while the fix is in production.

**Why C is real, unnecessary and worse.** You would be writing and maintaining code, with a credential
to manage, to reproduce behaviour the integration already provides.

**And do not miss the second half of A.** Without the connection configured on the Azure DevOps side,
even the correct syntax does nothing (Q5, Q22).

</details>

---

## Q44

How should a production error be traced back to its authorisation?

- A. Search the commit messages for the text of the error
- B. Deployment SHA → build → merge commit → PR → work item
- C. Ask the team which of them deployed to production last
- D. Check the release notes published alongside the release

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-03.md`:** lines **289–306**.

**Each step consumes an identifier the previous step produced** (Q33), which is what makes the trace
deterministic rather than a search.

**Why A and C are what happened on the night** and why it took four hours. Asking people reconstructs
memory; searching text finds what someone happened to write.

**Why D is downstream of the chain, not a substitute for it.** Release notes are generated *from* the
commits (Challenge 05) — they are only as complete as the convention that produced them.

</details>

---

## Q45

Which **two** satisfy "every PR must reference an issue or work item, enforced"? (Choose two.)

- A. A pull request template asking for the reference
- B. A workflow step failing with `core.setFailed` when absent
- C. A `::warning::` annotation when the reference is missing
- D. Listing that workflow job as a required status check
- E. A rule written in the repository's CONTRIBUTING.md

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-03.md`:** lines **340–344**, with the required-check pattern from Challenge 01.

**B produces the failure; D makes the failure matter.** The word in the requirement is "enforced", and a
failing job that is not required is a red mark someone can merge past.

**Why A is genuinely useful and not enforcement.** A template prompts (Challenge 02 Q35); the text can
be deleted.

**Why C is the deliberate contrast inside this challenge.** The commit-level scan at line 500 warns on
purpose, because not every individual commit needs a reference. The **PR-level** check fails, because
every PR does.

</details>

---

## Q46

How should the audit requirements be met?

- A. Query the organisation audit log when an incident occurs
- B. Enable branch protection on every production repository
- C. Require signed commits across the whole organisation
- D. Query for `policy_override` and stream events to Log Analytics

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-03.md`:** lines **394–397** and **412–427**.

**Two requirements, two mechanisms.** The override query answers "was anything bypassed?"; streaming to
Log Analytics answers "does the evidence outlive the platform's retention window?"

**Why A fails the second requirement.** Querying only works while the events are still inside the
platform — the query at line 409 takes an explicit start and end time for exactly that reason.

**Why B is preventive rather than evidential.** Branch protection is what makes an override *rare*; the
audit log is what makes it *visible*. The scenario needs both, and this question asks about the second.

</details>

---

## Q47

Seven months later, a release manager notices that the automatically generated changelog for the last
three releases contains almost nothing, although development has been busy. Commitlint is a required
check and is passing on every PR. The repository recently adopted squash merging with the PR title as
the commit subject.

What is the most likely cause?

- A. Commitlint stopped running as a required check on the repo
- B. The changelog generation tool was misconfigured recently
- C. Squash merging writes the PR title, unchecked by commitlint
- D. `fetch-depth: 0` was removed from the commitlint workflow

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-03.md`:** lines **353–354**, **480–483**, and Challenge 01's squash settings.

**"Commitlint is passing" is the detail that localises it.** The linter validates the range
`base.sha..head.sha` — the commits **on the branch**. Squash merging discards those and writes a single
new commit on `main`.

**And with `squash_merge_commit_title="PR_TITLE"` (Challenge 01, line 307), the subject of that commit
is the pull request title** — a field nothing in this challenge validates. Authors write "Payment
timeout fix" instead of `fix(payments): handle gateway timeout gracefully`, the linter never sees it,
and the changelog generator finds no parseable types on `main`.

**The result is a control that is genuinely working and genuinely not protecting the thing it was
bought for.** Every branch commit is perfect; every commit that survives is not.

**The fix is to validate what actually lands**: add a PR-title check against the same pattern
(line 354). Note the challenge's own `hasABRef` already inspects `prTitle` (line 337) — the title is
part of the traceability surface, and format should be too.

</details>

---

## Q48

A year on, an on-call engineer answers "which change caused this, who approved it, and what authorised
it" in under ten minutes.

Which explanation best accounts for it?

- A. Each link is created by doing the work, not recording it
- B. The team documented their releases in more detail
- C. The on-call engineer simply had more experience by then
- D. The organisation audit log became easier to search

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-03.md`:** lines **289–306**, **340–344**, **394–397**.

**Walk it backwards, as an incident runs.**

The alert names a service and a time. The **deployment** for that environment records a **SHA**. The
SHA names a **build** and the **merge commit** it ran on. The commit belongs to a **pull request** —
and that PR exists only because it carried a reference, since the check at line 341 fails without one.
The reference names the **work item**, which carries the discussion and the approval. And the **audit
log** answers whether the merge used a bypass.

**What actually changed is not the tooling.** Every one of these systems existed on the night of the
incident — GitHub had the commits, Azure Boards had the work items, the deployment had a SHA.

**What was missing was that the links were optional.** A commit could omit a reference, a PR could omit
a work item, and a message could be free text — so reconstructing the chain meant reading, guessing and
asking people, which is what consumed four hours at 2:47 AM.

**That is the sentence to give any exam question about traceability.** The chain is not built by
documenting more; it is built by **making each link a by-product of the work and a condition of
merging**. A link that depends on someone remembering will be missing on precisely the night you need
it.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **`AB#n` assumed to transition** | Q1, Q25, Q28, Q34, Q43 | `Fixes AB#n` transitions; `AB#n` only links |
| **A `breaking` commit type** | Q2, Q19, Q27, Q36 | It is a `!` modifier or a footer, not a type |
| **`perf` assumed to bump nothing** | Q8, Q17, Q27 | Patch bump |
| **`fetch-depth: 0` read as shallow** | Q4, Q26, Q39 | `0` = unlimited. Shallow is the default it overrides |
| **Shallow checkout with a commit range** | Q26, Q34 | The linter validates nothing and reports success |
| **App installed but repo not connected** | Q5, Q22, Q25 | Two halves. Silent failure |
| **Hook treated as enforcement** | Q29, Q35, Q42 | `--no-verify`, fresh clones, web commits |
| **`::warning::` expected to block** | Q12, Q21, Q29, Q45 | Only `core.setFailed` / `exit 1` gates |
| **Failing job assumed to block a merge** | Q45 | Only when it is a required status check |
| **Closing keywords on any branch** | Q11, Q20, Q28 | Default branch only |
| **Disabling the linter for merge commits** | Q6 | Add an `ignores` rule instead |
| **Blocking bot PRs on traceability** | Q7 | Exclude named bots by `github.actor` |
| **Audit log treated as preventive** | Q23, Q30, Q46 | It records. Branch protection prevents |
| **Squash merge bypassing commit validation** | Q47 | The PR title becomes the commit subject |

---

# What to memorise

**In `challenge-03.md`:** lines **40–74**, **144–168**, **289–306**, **394–427**.

```text
THE CHAIN - and what joins each link
  bug report  --#n / AB#n-->  work item  --Fixes AB#n-->  PR  --commits-->  SHA
              --build-->  artifact  --deployment records the SHA-->  environment
  TEXT REFERENCES at the top.  THE COMMIT SHA at the bottom.
  Trace an incident BACKWARDS: deployment -> build -> merge commit -> PR -> work item.

LINK vs TRANSITION
  AB#2045          link only, no state change
  Fixes AB#2100    link AND transition to Done/Resolved
  #87              GitHub issue link
  Fixes/Closes/Resolves #87   closes it - CASE-INSENSITIVE, DEFAULT BRANCH ONLY
  needs BOTH: Azure Boards GitHub App installed + repo connected in ADO (Boards > GitHub connections)
```

```text
CONVENTIONAL COMMITS                                (lines 40-74)
  <type>[optional scope]: <description>
  [body]
  [footer(s)]

  feat   MINOR      docs/style/refactor/test/build/ci/chore   none
  fix    PATCH      revert                                     -
  perf   PATCH      <- the one people miss

  BREAKING:  feat(api)!: ...        or a  BREAKING CHANGE: footer     -> MAJOR
             there is NO 'breaking' TYPE
  footers carry metadata:  Reviewed-by: security-team    Refs: AB#2001
```

```javascript
// commitlint - severity is the FIRST element: 0 off | 1 warning | 2 error   (lines 88-105)
rules: {
  'type-enum':           [2, 'always', ['feat','fix','docs','style','refactor',
                                        'perf','test','build','ci','chore','revert']],
  'scope-enum':          [1, 'always', ['api','auth','db','payments','notifications','ui']],
  'subject-max-length':  [2, 'always', 72],     // body/footer are 100
  'references-empty':    [1, 'never']
},
ignores: [(commit) => commit.startsWith('Merge')]   // generated merge commits are not violations
```

```yaml
# CI enforcement                                    (lines 437-502, 314-369)
- uses: actions/checkout@v4
  with: {fetch-depth: 0}          # 0 = UNLIMITED. default is shallow (depth 1) and breaks ranges
- run: npx commitlint --from ${{ github.event.pull_request.base.sha }} \
                      --to   ${{ github.event.pull_request.head.sha }} --verbose

# PR must reference something - HARD fail
core.setFailed('PR must reference at least one issue (#123) or work item (AB#123)')
  hasIssueRef  /(close[sd]?|fix(e[sd])?|resolve[sd]?)\s+#\d+/i   on the BODY
  hasABRef     /AB#\d+/i                                          on body OR TITLE
# commit-level work item scan - SOFT warn
echo "::warning::No work item reference found in commits."

# bots have no authorising work item
if: github.actor != 'dependabot[bot]' && github.actor != 'github-actions[bot]'

# local hook - fast feedback, BYPASSABLE with --no-verify
.husky/commit-msg    npx --no -- commitlint --edit "$1"      + chmod +x
```

```bash
# AUDIT - records, never prevents                   (lines 380-427)
gh api orgs/ORG/audit-log -f phrase='action:protected_branch.policy_override repo:ORG/REPO' \
  --jq '.[] | {actor, action, created_at}'
#   phrase composes with spaces:  action: | actor: | repo:
#   org-scoped, admin only

# Azure DevOps: QUERY is pull and bounded; STREAMING is push and continuous
az devops invoke --area audit --resource streams --http-method POST ...
  consumerType: "AzureMonitorLogs"   -> Log Analytics -> KQL alongside app telemetry (Ch.49)
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 04 |
| 38–43 | Re-read the trap index and the chain diagram, then move on |
| 30–37 | Draw the chain and write the type-to-bump table from memory, then retake |
| Below 30 | Redo Tasks 1, 2 and 4 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 03.

:::danger The two facts

**A reference links; a keyword transitions.** `AB#2045` links. `Fixes AB#2100` links **and moves the
work item**. Same for `#87` and `Fixes #87`.

**Text joins the top of the chain; the SHA joins the bottom.** Deployment → build → merge commit → PR →
work item, each step a lookup, none of them requiring anyone to remember.

Four hours at 2:47 AM was not a tooling gap. It was a chain whose links were optional.

:::
