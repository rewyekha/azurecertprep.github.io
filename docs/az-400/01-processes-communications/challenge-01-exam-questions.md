---
sidebar_position: 1.5
toc_max_heading_level: 2
title: "Challenge 01: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 01 — AZ-400 exam questions

**48 questions** built only from what Challenge 01 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-01.md`**.

:::danger Read this before you start

Branching-strategy questions are decided by **release cadence**, never by which platform is named.

**GitHub Flow** — one long-lived branch, `main`, always deployable. Short-lived feature branches, PR,
merge, deploy from `main`. For services you ship continuously.
**GitFlow** — `main` **and** `develop`, plus release and hotfix branches. For **packaged software with
formal version numbers and support windows** — mobile apps, boxed products.
**Trunk-based** — commit to `main` or branches that live less than a day, behind **feature flags**.
Needs mature CI and flag infrastructure.

**The trap you must beat: the word "GitHub" in a question does not mean GitHub Flow.** A team shipping
a mobile app with three supported versions needs GitFlow whether their code sits in GitHub, Azure Repos
or anywhere else. Read the **cadence** and the **number of versions kept alive**, then choose.

The second idea runs through every task after Task 1: **a strategy nobody enforces is a suggestion.**
Contoso already "had" a strategy. Branch protection, required checks and auto-merge are what turn the
decision at line 86 into behaviour.

The scenario at line 20 is what the absence costs: five teams, merges taking **two to three days**,
`main` broken **at least once per sprint**, and nobody able to say which branch is production.

:::

---

# Section A — Multiple choice

---

## Q1

In GitHub Flow, what is the role of the `main` branch?

- A. It collects finished features until a release branch is cut from it
- B. It receives merges only inside a scheduled release window
- C. It stays deployable at all times and is what production ships from
- D. It tracks the `develop` branch and is resynchronised each sprint

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-01.md`:** line **44**.

```text
- `main` is always deployable; deployments happen from `main`
```

**"Always deployable" is not an aspiration in GitHub Flow — it is the load-bearing assumption.** There
is no other branch to deploy from, so the moment `main` breaks, the team cannot ship anything.

**Why the others fail, and each describes a different strategy**

- **A** — that is GitFlow's `develop` branch (line 48). GitHub Flow has no integration branch
- **B** — release windows are GitFlow's model; GitHub Flow assumes continuous deployment (line 66)
- **D** — there is no `develop` branch to mirror

**Which is why the scenario is a crisis, not an inconvenience.** `main` broken once per sprint (line 20)
means the one deployable branch was undeployable, repeatedly.

</details>

---

## Q2

Which branch protection setting stops a PR merging when new commits have landed on `main` since the PR
branch was created?

- A. Require branches to be up to date before merging, known as strict mode
- B. Require status checks to pass before merging on the pull request branch
- C. Require review from Code Owners for the paths the pull request touches
- D. Dismiss stale pull request approvals when new commits are pushed

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-01.md`:** line **180**.

```bash
  --field required_status_checks='{"strict":true,"contexts":["ci/build","ci/test"]}' \
```

**`strict: true` forces the branch to be current with `main` before it can merge**, which is what
prevents **merge skew** — two PRs that each pass in isolation and break when combined.

**Why B is the answer most people give, and why it is not enough.** Required checks prove the PR branch
is green. They say nothing about whether `main` moved underneath it. Strict mode is the extra clause
that re-runs the checks against the *merged* result.

**Why D is a different protection.** Dismissing stale approvals (line 182) invalidates a **human
review** after new commits; strict mode invalidates a **build** after `main` changes. Both are in this
challenge and they guard different things.

</details>

---

## Q3

When should you choose GitFlow over GitHub Flow?

- A. When every merge to `main` should deploy straight to production
- B. When one team commits to trunk many times a day behind feature flags
- C. When you want the fewest possible branch types and the simplest model
- D. When you ship versioned releases and support several of them at once

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-01.md`:** line **52**.

```text
- Best for: software with formal release cycles (mobile apps, packaged software)
```

**GitFlow's release and hotfix branches exist to keep several shipped versions alive at once.** If you
must patch v2.3 while v3.0 is in development, you need a branch that represents v2.3 — and GitHub Flow
does not have one.

**Why the others fail**

- **A** — continuous deployment is GitHub Flow's own use case (line 66)
- **B** — that describes trunk-based development (line 60)
- **C** — GitFlow is the **most** complex of the three: four branch types instead of one

**The exam's version of this question always hides the answer in the support model.** "We maintain the
last three releases" is GitFlow. "We deploy on merge" is GitHub Flow.

</details>

---

## Q4

What does enabling auto-merge with squash accomplish?

- A. It merges once all required conditions pass, squashed into one commit
- B. The PR merges at once, with required checks recorded but not enforced
- C. The PR is queued and merged on the next scheduled nightly rebase run
- D. The PR merges ahead of pending reviews if the author has write access

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-01.md`:** lines **315–320**.

```bash
gh pr merge --auto --squash

# The PR will merge automatically when:
# 1. All required status checks pass
# 2. Required reviews are approved
# 3. Branch is up to date with main (if strict mode enabled)
```

**Auto-merge waits; it does not bypass.** It removes the delay between "everything passed" and
"somebody noticed and clicked merge" — which on a five-team repository is often hours.

**Why B and D are the same misreading, and it is worth killing now.** Nothing in this challenge lets a
PR skip branch protection. Auto-merge is the *opposite* of a bypass: it is a promise to merge **only**
when every condition the repository requires is already satisfied.

</details>

---

## Q5

A developer pushes directly to `main` and expects it to be rejected, but the push succeeds. They are a
repository administrator. What setting is wrong?

- A. `allow_force_pushes` is set to true on the protected branch
- B. `enforce_admins` is not enabled on the protected branch
- C. `required_approving_review_count` is set to 0 for the branch
- D. `allow_deletions` is set to true on the protected branch

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-01.md`:** lines **440–442**.

```bash
gh api repos/{owner}/{repo}/branches/main/protection \
  --jq '.enforce_admins.enabled'
# Should be true - admins are also subject to protection
```

**Without `enforce_admins`, branch protection applies to everyone except the people most able to break
production.** That is not a rule; it is a convention with an exception list.

**And on a five-team repository, "who is an admin" grows quietly.** The setting matters most a year
after it was configured.

**Why A is the tempting wrong answer.** A force push *rewrites* history on `main`. This developer made
an ordinary commit and pushed it — which is a different operation, blocked by the pull-request
requirement rather than by the force-push setting.

</details>

---

## Q6

A PR merges even though CI failed. The required status checks are configured. What is the most likely
cause?

- A. Auto-merge was enabled on the pull request before CI had reported
- B. `strict` mode is disabled, so the branch was never brought up to date
- C. The reviewer approved the pull request before CI finished running
- D. The required check names do not match the contexts CI actually reports

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-01.md`:** lines **453** and **459–467**.

```text
The status check context names must match exactly what your CI reports.
```

```bash
# List recent check runs to see their exact names
gh api repos/{owner}/{repo}/commits/main/check-runs \
  --jq '.check_runs[].name'
```

**A required check that never reports is not a failing check — it is an absent one**, and branch
protection has nothing to wait for.

**Look at the mismatch the challenge itself sets up.** The protection at line 180 requires `ci/build`
and `ci/test`; the workflow at lines 342 and 361 defines jobs named `build` and `test`. The corrected
command at line 467 uses `["build","test","validate-branch-name"]`.

**This is a silent failure and that is what makes it exam material.** Nothing errors. The PR page shows
green, the rule appears configured, and the gate is doing nothing.

</details>

---

## Q7

A reviewer approves a PR. The developer then pushes two more commits. What setting ensures the approval
no longer counts?

- A. `require_code_owner_reviews` on the paths the commits changed
- B. `dismiss_stale_reviews` on the required pull request reviews
- C. `require_last_push_approval` on the required pull request reviews
- D. `strict` status checks on the required status checks object

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-01.md`:** lines **182** and **480–484**.

```bash
gh api repos/{owner}/{repo}/branches/main/protection/required_pull_request_reviews \
  --jq '.dismiss_stale_reviews'
# Must be true
```

**An approval is a statement about specific code.** Once the code changes, the statement is about
something that no longer exists.

**Why C is the closest wrong answer, and the distinction is real.** `require_last_push_approval`
(line 221) requires that someone **other than the last pusher** approves — it targets a person, not
staleness. `dismiss_stale_reviews` targets the *commits*. This challenge sets `require_last_push_approval`
to `false` and relies on dismissal.

</details>

---

## Q8

Which rule type in a ruleset prevents force pushes?

- A. `deletion`, which rejects updates that remove the branch entirely
- B. `required_status_checks`, which gates on the contexts CI reports
- C. `pull_request`, which requires changes to arrive through a review
- D. `non_fast_forward`, which rejects updates that discard history

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-01.md`:** lines **235–239**.

```json
    {
      "type": "deletion"
    },
    {
      "type": "non_fast_forward"
    }
```

**A force push is by definition not a fast-forward** — it moves the branch pointer to a commit that is
not a descendant of the current tip, discarding history. Blocking non-fast-forward updates blocks it.

**And `deletion` is the sibling rule**, protecting against the other way a branch disappears. The
equivalents in the classic API are `allow_force_pushes=false` and `allow_deletions=false` (lines
184–185) — same two protections, different vocabulary.

</details>

---

## Q9

Why does the ADR record **Consequences** as well as the decision?

- A. To satisfy the evidence requirements of an external compliance audit
- B. To list the tooling the team must purchase before the decision lands
- C. To record what the team is accepting, so it is not reopened later
- D. To assign a named owner to each follow-up task the decision creates

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-01.md`:** lines **89–93**.

```text
## Consequences
- All teams follow one workflow
- Main is always deployable
- No long-lived branches (feature flags for incomplete work)
- Requires branch protection and CI gates
```

**Read the third line: "no long-lived branches" is a cost, not a benefit.** A team that wanted a
two-week feature branch is being told no — and six months from now, someone will ask why.

**The ADR's four sections each answer a different question.** Status = is this current. Context = what
problem existed (line 82). Decision = what we chose. Consequences = what we accepted, including the
parts nobody likes.

**Why A is a real side benefit and not the purpose.** An ADR is written for the next engineer, not for
an auditor.

</details>

---

## Q10

The CI workflow rejects a branch named `add-user-service`. Why?

- A. It does not begin with one of the approved prefixes the pattern lists
- B. It exceeds the maximum branch name length the validation job allows
- C. It contains a hyphen, which the character class does not permit
- D. It contains no slash, which the pattern requires as a separator

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-01.md`:** lines **385–389**.

```bash
          PATTERN="^(feature|bugfix|hotfix|chore|docs)/[a-z0-9._-]+$"
          if [[ ! "$BRANCH_NAME" =~ $PATTERN ]]; then
            echo "::error::Branch name '$BRANCH_NAME' does not match pattern: $PATTERN"
            echo "Valid prefixes: feature/, bugfix/, hotfix/, chore/, docs/"
```

**The slash is required *because* of the prefix**, so D is a consequence rather than the rule — and a
branch named `stuff/thing` also has a slash and is also rejected.

**Why the convention earns its keep on a shared repository.** With five teams in one monorepo, the
prefix is the only thing that tells you at a glance whether `hotfix/payment-timeout` should jump the
queue. Hyphens are explicitly allowed by `[a-z0-9._-]`.

</details>

---

## Q11

The `enforce-no-long-lived-branches` job finds a branch 12 days old. What happens?

- A. The pull request is blocked until the branch is recreated from `main`
- B. The branch is deleted automatically once the job finishes running
- C. A warning is emitted to the log and the job still finishes green
- D. The build fails with a non-zero exit from the validation step

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-01.md`:** lines **405–406**.

```bash
          if [ "$DAYS_OLD" -gt 7 ]; then
            echo "::warning::This branch is $DAYS_OLD days old. GitHub Flow recommends short-lived branches (< 7 days)."
```

**`::warning::` surfaces the finding without failing the job**, and there is no `exit 1` — compare with
the branch-name check at line 390, which does exit non-zero.

**The design choice is deliberate and worth understanding.** Branch age is a **guideline**; a genuinely
complex change might legitimately take nine days, and blocking it would push people to game the date by
recreating the branch. Naming is a **rule** and can be enforced absolutely.

**Note also `fetch-depth: 0`** at line 399 — the job needs full history to find when the branch started,
which a shallow clone would not provide.

</details>

---

## Q12

Why does the CI workflow trigger on both `pull_request` and `push` to `main`?

- A. Because `pull_request` on its own will not run the workflow at all
- B. To validate a PR before merge and confirm `main` is healthy after it
- C. Because branch protection requires both triggers to be configured
- D. To double the run count so the pipeline statistics stay meaningful

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-01.md`:** lines **335–339**.

```yaml
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
```

**Two triggers, two questions.** The PR run asks "is this change safe to merge?"; the push run asks "is
`main` still deployable?" — which matters enormously when `main` is the only thing you deploy from
(Q1).

**And the two can disagree**, which is the entire reason for strict mode (Q2): the PR was green against
an older `main`, and the merged result is not.

**Note the `validate-branch-name` and branch-age jobs are PR-only** (lines 380 and 395), because a
branch name is meaningless once the branch is merged and gone.

</details>

---

## Q13

Which merge settings does Task 5 configure to keep `main` history clean?

- A. Merge commits only, with branches retained after the pull request lands
- B. All three merge types enabled, with branches deleted once merged
- C. Rebase only, with branches retained so the history can be re-read
- D. Squash and rebase allowed, merge commits off, branches deleted on merge

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-01.md`:** lines **303–306**.

```bash
  --field delete_branch_on_merge=true \
  --field allow_squash_merge=true \
  --field allow_merge_commit=false \
  --field allow_rebase_merge=true \
```

**Disabling merge commits is what actually produces linear history.** Leaving it enabled means one
developer's habit reintroduces the merge bubbles the setting was meant to remove.

**And `delete_branch_on_merge` is a hygiene control with a real payoff here.** On a monorepo with five
teams, undeleted branches accumulate until nobody can tell which are live — which is a smaller version
of the scenario's "nobody knows which branch represents the production state" (line 20).

</details>

---

## Q14

What do `squash_merge_commit_title="PR_TITLE"` and `squash_merge_commit_message="PR_BODY"` achieve?

- A. They enforce a specific conventional-commit format on the squashed subject
- B. They prevent an empty commit message when the branch has no description
- C. They make the squashed commit carry the PR's title and body instead
- D. They append the pull request number to the squashed commit subject

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-01.md`:** lines **307–308**.

**The default squash message is every commit subject on the branch stitched together** — including
"wip", "fix typo" and "actually fix typo". Setting these fields makes the permanent record the
**reviewed description** rather than the developer's working notes.

**Which matters more than tidiness.** In GitHub Flow the squashed commit is the *only* record of the
change on `main`. It is what a future engineer reads during an incident, and what release notes are
generated from (Challenge 05).

</details>

---

## Q15

Which is a genuine reason GitHub Flow suits Contoso, according to the challenge?

- A. They maintain four parallel release branches that must stay in sync
- B. They deploy web services continuously with no formal release windows
- C. They must support the three most recently shipped product versions
- D. They deploy on a fixed monthly date agreed with the business

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-01.md`:** lines **64–69**.

```text
- They deploy web services continuously (no formal release windows)
- Teams need a simple, consistent model everyone can follow
- They want CI validation before any code reaches main
- They do not need the complexity of release branches
```

**Read the last line: the justification is partly about what they *do not* need.** Choosing the simpler
strategy when the complex one buys you nothing is the actual skill being tested.

**Why A is the trap, and it is a good one.** The scenario *does* mention a team maintaining four
parallel release branches (line 20) — as a **symptom of the chaos**, not a requirement. The exam
regularly quotes the current broken state back at you as though it were a constraint.

</details>

---

## Q16

The PR template's "Type of change" section lists Breaking change as an option. What is its purpose in
this workflow?

- A. It bumps the package version automatically when the box is ticked
- B. It blocks the merge until a second reviewer approves the change
- C. It notifies the security team through a CODEOWNERS assignment
- D. It makes the author declare compatibility impact for reviewers

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-01.md`:** lines **259–264**.

```text
## Type of change

- [ ] Bug fix (non-breaking change that fixes an issue)
- [ ] New feature (non-breaking change that adds functionality)
- [ ] Breaking change (fix or feature that would break existing functionality)
```

**A template is a prompt, not an enforcement mechanism** — nothing in this challenge validates the
checkbox. Its value is that the question gets asked at the moment the author still remembers the
answer.

**Why A is the near-miss worth separating.** Automatic version bumping from declared intent is real —
it is what conventional commits and `BREAKING CHANGE` do (Challenge 03), and what GitVersion consumes
(Challenge 14). A checkbox in Markdown is read by humans only.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** describe GitHub Flow? (Choose three.)

- A. A single long-lived branch, `main`, that every change returns to
- B. A permanent `develop` branch where features integrate first
- C. Short-lived feature branches created from `main` and merged back
- D. Release branches cut and maintained for each shipped version
- E. `main` is always deployable and is what production deploys from
- F. Commits land directly on `main` behind feature flags

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-01.md`:** lines **39–44**.

```text
- Single long-lived branch: `main`
- Developers create short-lived feature branches from `main`
- Pull requests are opened early for discussion
- CI runs on every push to the PR branch
- After review and green CI, the PR merges to `main`
- `main` is always deployable; deployments happen from `main`
```

**B and D are GitFlow** (lines 48–50). **F is trunk-based** (lines 57–58).

**All three wrong options are real practices from the same page**, which is exactly how the exam builds
this question — you cannot eliminate by plausibility, only by knowing which list each line came from.

</details>

---

## Q18

Which **three** are true of trunk-based development? (Choose three.)

- A. It uses a `develop` branch to integrate work before release
- B. Branches live less than one day, or work goes straight to `main`
- C. It relies heavily on feature flags to hide incomplete work
- D. It is designed around formal, scheduled release cycles
- E. It requires comprehensive automated test coverage to be safe
- F. It forbids pull requests in favour of direct commits

<details>
<summary>Show answer</summary>

### Answer: B, C, E

**In `challenge-01.md`:** lines **56–60**.

```text
- Developers commit directly to `main` or use extremely short-lived branches (less than one day)
- Relies heavily on feature flags for incomplete work
- Requires comprehensive automated testing
- Best for: high-performing teams with mature CI/CD and feature flag infrastructure
```

**C and E are prerequisites, not benefits, and that is the point of the question.** Trunk-based without
feature flags means shipping half-finished work; trunk-based without comprehensive tests means shipping
it unverified. Line 60 says so directly: *high-performing teams with mature CI/CD and feature flag
infrastructure*.

**Why F is the misconception.** Trunk-based does not forbid PRs — it forbids **long-lived** branches. A
PR that opens and merges the same day is entirely compatible with it.

</details>

---

## Q19

Which **three** guardrails does the branch protection configuration apply? (Choose three.)

- A. At least one approving review, with stale approvals dismissed
- B. A maximum pull request size measured in changed lines
- C. A required branch naming pattern checked before merge
- D. Required status checks that must pass, running in strict mode
- E. Automatic deployment to production once the PR merges
- F. Force pushes and branch deletions blocked on `main`

<details>
<summary>Show answer</summary>

### Answer: A, D, F

**In `challenge-01.md`:** lines **180–185**.

```bash
  --field required_status_checks='{"strict":true,"contexts":["ci/build","ci/test"]}' \
  --field enforce_admins=true \
  --field required_pull_request_reviews='{"required_approving_review_count":1,"dismiss_stale_reviews":true,"require_code_owner_reviews":true}' \
  --field restrictions=null \
  --field allow_force_pushes=false \
  --field allow_deletions=false
```

**Why C is the precise near-miss.** Branch naming **is** enforced in this challenge — by a **CI job**
(lines 378–391), not by branch protection. It becomes a guardrail only if `validate-branch-name` is
also listed as a required check, which is exactly what the corrected command at line 467 does.

**That distinction is the whole lesson.** Branch protection enforces *outcomes* (a review happened, a
check passed). CI enforces *content*. Wiring the CI job into the required list is what joins them.

</details>

---

## Q20

Which **two** conditions must be satisfied before an auto-merge PR merges, beyond passing checks?
(Choose two.)

- A. An administrator confirms the merge from the pull request page
- B. The pull request body matches the repository template exactly
- C. Required reviews on the pull request have all been approved
- D. The branch has existed for more than seven calendar days
- E. The branch is up to date with `main`, if strict mode is on

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-01.md`:** lines **317–320**.

```bash
# The PR will merge automatically when:
# 1. All required status checks pass
# 2. Required reviews are approved
# 3. Branch is up to date with main (if strict mode enabled)
```

**Note the conditional in item 3.** Strict mode is what *creates* the up-to-date requirement — without
it, an auto-merge PR can merge against a `main` that has moved, which is precisely the merge skew Q2
describes.

**Why A inverts the feature.** Auto-merge exists to remove the human from the *merge click*, not to add
one.

</details>

---

## Q21

Which **two** problems from the scenario does branch protection directly address? (Choose two.)

- A. Merges routinely taking two to three days to resolve
- B. `main` broken at least once per sprint by incoming changes
- C. Five development teams working in a single monorepo
- D. Nobody knows which branch represents the production state
- E. Each team having invented its own branching strategy

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-01.md`:** line **20**, with the protection at lines **177–185**.

**B is fixed by required status checks** — broken code cannot reach `main` if a green build is a
precondition. **D is fixed by having one deployable branch and enforcing it**: once `main` is the only
merge target, "which branch is production" has a single answer.

**Why A is fixed by something else, and this is the useful distinction.** Two-to-three-day merges are
caused by **long-lived branches diverging**, and the cure is short-lived branches plus frequent
integration — a *practice*, nudged by the branch-age warning at line 406. No protection rule shortens
a merge conflict.

**Why E is what the ADR addresses** (lines 75–94): a written, agreed decision.

</details>

---

## Q22

Which **two** are true of the ruleset approach compared with classic branch protection? (Choose two.)

- A. It is the approach recommended for organisation-wide use
- B. It removes the need for a CI system to report checks
- C. It can only be applied to tags, never to branch names
- D. It expresses the same guardrails as typed rules with parameters
- E. It cannot require status checks the way protection can

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-01.md`:** lines **197** and **214–240**.

```text
Using the newer branch ruleset approach (recommended for organizations):
```

```json
  "rules": [
    { "type": "pull_request", "parameters": { ... } },
    { "type": "required_status_checks", "parameters": { ... } },
    { "type": "deletion" },
    { "type": "non_fast_forward" }
  ]
```

**The parallels are exact and worth learning as pairs**: `pull_request` ↔ `required_pull_request_reviews`,
`required_status_checks` ↔ `required_status_checks`, `non_fast_forward` ↔ `allow_force_pushes=false`,
`deletion` ↔ `allow_deletions=false`.

**Why E is refuted by line 225** — the rule type is right there. And note `conditions.ref_name.include`
at line 210 targets `refs/heads/main`, which is how a ruleset can cover a **pattern** of branches rather
than one named branch.

</details>

---

## Q23

Which **two** jobs in the challenge's CI workflow run only on pull requests? (Choose two.)

- A. `build`, which compiles the application and uploads artifacts
- B. `test`, which runs the unit suite and publishes results
- C. `validate-branch-name`, which checks the branch prefix pattern
- D. `lint`, which checks formatting across the whole repository
- E. `enforce-no-long-lived-branches`, which checks the branch age

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-01.md`:** lines **380** and **395**.

```yaml
    if: github.event_name == 'pull_request'
```

**Both guard *branch* properties, and a branch stops existing at merge** — with `delete_branch_on_merge`
enabled (line 303), it is gone seconds later. Running them on a push to `main` would evaluate a name and
an age that no longer mean anything.

**Why `build` and `test` run on both** — Q12. They validate the *code*, which persists.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must unify five teams on GitHub Flow, guarantee `main` stays deployable, prevent
direct pushes, stop PRs merging against a stale `main`, invalidate approvals when code changes, and
record the decision so it is not relitigated.

---

## Q24

**Proposed solution:** Write an ADR recording the decision and its consequences. Configure branch
protection on `main` with one required approving review, dismiss stale reviews, required status checks
in strict mode, `enforce_admins` true, and force pushes and deletions blocked. Add a CI workflow
triggered on pull requests and on pushes to `main`, with build, test and a branch-name validation job,
and list the job names in the required checks exactly as CI reports them. Enable auto-merge with squash,
disable merge commits, and delete branches on merge.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-01.md`:** lines **75–94**, **177–185**, **332–408**, **459–467**, **300–308**.

| Requirement | Mechanism |
|---|---|
| Unify five teams | ADR recording decision + consequences |
| `main` stays deployable | Required status checks, plus CI on push to `main` |
| No direct pushes | Required PR reviews + `enforce_admins` |
| No merging against stale `main` | `strict: true` |
| Approvals invalidated by new commits | `dismiss_stale_reviews: true` |
| Consistent branch naming | CI job, **listed as a required check** |

**The clause that makes it work is "exactly as CI reports them."** Every other line is standard; that
one is the difference between a configured gate and an operating one (Q6).

</details>

---

## Q25

**Proposed solution:** Announce GitHub Flow in a team meeting. Configure branch protection requiring one
review, leaving `enforce_admins` off so leads can unblock urgent work. Require status checks named
`ci/build` and `ci/test`. Enable auto-merge. Let each team decide its own branch naming.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures, and they compound.**

**An announcement is not a decision record.** Nothing captures *why*, so the first team that wants a
release branch reopens the argument — and the scenario at line 20 shows five teams are perfectly capable
of inventing their own answer.

**`enforce_admins` off means the rule has an exception list** (Q5), and "leads can unblock urgent work"
is the exact phrasing under which `main` gets broken. The people with the power to bypass are the people
under the most pressure to.

**`ci/build` and `ci/test` do not match the jobs the workflow defines** — `build` and `test` (lines 342,
361). Those required checks will never report, so the PR either blocks forever or, if the rule is later
"fixed" by deletion, merges unchecked.

**And no strict mode** means two green PRs can still combine into a broken `main`, which is the specific
failure the scenario complains about.

</details>

---

## Q26

**Proposed solution:** Write an ADR recording the decision and its consequences. Configure branch
protection on `main` with one required approving review, dismiss stale reviews, required status checks in
strict mode, `enforce_admins` true, and force pushes and deletions blocked. Add a CI workflow triggered
on pull requests and on pushes to `main`, with build, test and branch-name validation, and list the job
names exactly as CI reports them. Enable auto-merge with squash and delete branches on merge. Because
teams complained that strict mode forces constant rebasing on a busy monorepo, disable it and rely on the
required status checks alone.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**Without `strict`, a PR is validated against the `main` it branched from, not the `main` it will become
part of.** Two developers can each open a green PR, and merging both produces a `main` that neither PR
ever tested.

**On this repository that is not a theoretical risk — it is the reported symptom.** Line 20 says `main`
is broken at least once per sprint with five teams merging into one monorepo. The more concurrent PRs,
the more often the merged result differs from anything that was tested.

**And the complaint driving the change is real**, which is what makes the option so plausible. Strict
mode *does* force rebasing when `main` moves often. But the correct responses are **smaller, faster PRs**
(the short-lived branches at line 40, nudged by the age warning at line 406) or a **merge queue**, which
tests the merged result without making every author rebase by hand.

**Turning off the check because it keeps catching something is the anti-pattern the exam is testing.**
The friction is the control working.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — GitHub Flow

| # | Statement | Answer |
|---|---|---|
| 1 | `main` is always deployable |  |
| 2 | Feature branches are long-lived |  |
| 3 | Deployments happen from `main` |  |
| 4 | A `develop` branch integrates features first |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `main` is always deployable | **Yes** |
| 2 | Feature branches are long-lived | **No** |
| 3 | Deployments happen from `main` | **Yes** |
| 4 | A `develop` branch integrates features first | **No** |

**In `challenge-01.md`:** lines **44**, **40**, **44**, **48**.

Rows 2 and 4 are GitFlow's properties offered as GitHub Flow's. **The exam mixes the three strategies'
bullet lists**, and the only defence is knowing which list each line belongs to.

</details>

---

## Q28 — branch protection

| # | Statement | Answer |
|---|---|---|
| 1 | `enforce_admins` makes protection apply to administrators |  |
| 2 | `strict` requires the branch to be up to date before merging |  |
| 3 | Required check names must match the CI job names exactly |  |
| 4 | `dismiss_stale_reviews` invalidates approvals after new commits |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `enforce_admins` makes protection apply to administrators | **Yes** |
| 2 | `strict` requires the branch to be up to date before merging | **Yes** |
| 3 | Required check names must match the CI job names exactly | **Yes** |
| 4 | `dismiss_stale_reviews` invalidates approvals after new commits | **Yes** |

**In `challenge-01.md`:** lines **181**, **180**, **453**, **182**.

**All four are Yes, which the exam does use** — and it is exactly when a candidate pattern-matching on
"there must be a No in here" talks themselves out of a correct row.

Row 3 is the one that fails silently in production (Q6): no error, no warning, just a gate that never
evaluates.

</details>

---

## Q29 — auto-merge and merge settings

| # | Statement | Answer |
|---|---|---|
| 1 | Auto-merge bypasses required reviews |  |
| 2 | `allow_merge_commit=false` prevents merge-commit history |  |
| 3 | `delete_branch_on_merge` removes the branch after merging |  |
| 4 | Squash merge preserves every individual commit on `main` |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Auto-merge bypasses required reviews | **No** |
| 2 | `allow_merge_commit=false` prevents merge-commit history | **Yes** |
| 3 | `delete_branch_on_merge` removes the branch after merging | **Yes** |
| 4 | Squash merge preserves every individual commit on `main` | **No** |

**In `challenge-01.md`:** lines **317–320**, **305**, **303**, **121**.

Row 4 is the definition of squashing — the branch's commits become **one** commit, which is why the title
and body settings at lines 307–308 matter so much (Q14).

Row 1 is the misconception worth stating twice: auto-merge **waits for** conditions, it does not remove
them.

</details>

---

## Q30 — CI enforcement

| # | Statement | Answer |
|---|---|---|
| 1 | The branch-name job fails the run on a bad name |  |
| 2 | The branch-age job fails the run after seven days |  |
| 3 | `fetch-depth: 0` is needed to determine branch age |  |
| 4 | A CI job becomes a gate only when listed as a required check |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | The branch-name job fails the run on a bad name | **Yes** |
| 2 | The branch-age job fails the run after seven days | **No** |
| 3 | `fetch-depth: 0` is needed to determine branch age | **Yes** |
| 4 | A CI job becomes a gate only when listed as a required check | **Yes** |

**In `challenge-01.md`:** lines **390**, **405–406**, **399**, **467**.

Rows 1 and 2 are the deliberate asymmetry (Q11): **naming is a rule, age is a guideline.**

Row 4 is the sentence to carry out of this challenge. A job that runs and reports is information; a job
named in `contexts` is a gate.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each team situation to the branching strategy.

| Situation | Strategy |
|---|---|
| Web service deployed on every merge |  |
| Mobile app supporting three shipped versions |  |
| Mature team merging many times a day behind flags |  |
| Boxed software with formal release windows |  |
| Five teams needing one simple consistent model |  |

**Options:** GitFlow · GitHub Flow · Trunk-based

<details>
<summary>Show answer</summary>

| Situation | Strategy |
|---|---|
| Web service deployed on every merge | **GitHub Flow** |
| Mobile app supporting three shipped versions | **GitFlow** |
| Mature team merging many times a day behind flags | **Trunk-based** |
| Boxed software with formal release windows | **GitFlow** |
| Five teams needing one simple consistent model | **GitHub Flow** |

**In `challenge-01.md`:** lines **44**, **52**, **57–58**, **52**, **67**.

**Only two facts decide every row: how often you ship, and how many versions you keep alive.** Nothing
about the hosting platform appears in the answer — which is the trap Q3 and Q15 attack from two
directions.

</details>

---

## Q32

Match each requirement to the setting that enforces it.

| Requirement | Setting |
|---|---|
| No direct commits to `main` |  |
| Rules apply to administrators too |  |
| PR must be current with `main` |  |
| Approval dies when new commits land |  |
| History cannot be rewritten |  |
| Branch is removed after merge |  |

**Options:** `allow_force_pushes: false` / `non_fast_forward` · `delete_branch_on_merge: true` · `dismiss_stale_reviews: true` · `enforce_admins: true` · Required pull request reviews · `strict: true`

<details>
<summary>Show answer</summary>

| Requirement | Setting |
|---|---|
| No direct commits to `main` | **Required pull request reviews** |
| Rules apply to administrators too | **`enforce_admins: true`** |
| PR must be current with `main` | **`strict: true`** |
| Approval dies when new commits land | **`dismiss_stale_reviews: true`** |
| History cannot be rewritten | **`allow_force_pushes: false`** / `non_fast_forward` |
| Branch is removed after merge | **`delete_branch_on_merge: true`** |

**In `challenge-01.md`:** lines **182**, **181**, **180**, **182**, **184** and **238**, **303**.

**Two vocabularies for the same intent** — classic protection fields on the left of the last row,
ruleset rule types on the right. Expect the exam to use either.

</details>

---

## Q33

Arrange the steps to move Contoso from the current chaos to enforced GitHub Flow.

**Items:** Configure branch protection with required reviews and checks · Write the ADR recording the
decision · Add the CI workflow that produces the checks · List the CI job names as required status
checks · Enable auto-merge and squash-only merging

<details>
<summary>Show answer</summary>

### Answer

1. Write the ADR recording the decision — lines **75–94**
2. Add the CI workflow that produces the checks — lines **332–408**
3. Configure branch protection with required reviews and checks — lines **177–185**
4. List the CI job names as required status checks — line **467**
5. Enable auto-merge and squash-only merging — lines **300–308**

**Step 2 comes before step 4, and that ordering is the answer the exam wants.** A required check can
only reference a job that has actually reported at least once — name a check that does not exist and
every PR blocks forever with nothing to click.

**Step 1 is first for a human reason.** Turning on branch protection before the teams have agreed why
produces a support queue, not a workflow. The ADR is what makes step 3 defensible.

**Step 5 is last because auto-merge is only safe once the conditions it waits for are correct.** Enable
it while the required checks are misnamed and you have automated merging with no gate (Q25).

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| An admin pushes straight to `main` and it works |  |
| A PR merges with a red build |  |
| Two green PRs merge and break `main` |  |
| An approval survives three new commits |  |
| A PR is blocked by a check that never runs |  |
| Branch name check passes on `main` pushes |  |

**Options:** A required check that no job reports · `dismiss_stale_reviews` is off · `enforce_admins` is off · Required check names do not match CI job names · `strict` mode disabled · The job is `pull_request`-only, correctly

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| An admin pushes straight to `main` and it works | **`enforce_admins` is off** |
| A PR merges with a red build | **Required check names do not match CI job names** |
| Two green PRs merge and break `main` | **`strict` mode disabled** |
| An approval survives three new commits | **`dismiss_stale_reviews` is off** |
| A PR is blocked by a check that never runs | **A required check that no job reports** |
| Branch name check passes on `main` pushes | **The job is `pull_request`-only, correctly** |

**In `challenge-01.md`:** lines **441**, **453**, **180**, **484**, **459–467**, **380**.

**Rows 2 and 5 are the same misconfiguration producing opposite symptoms**, and which one you get
depends on whether the check was ever required. That is why the diagnosis at line 460 lists the *actual*
check-run names first — you cannot fix the rule until you know what CI really calls itself.

</details>

---

## Q35

Match each artifact to what it enforces.

| Artifact | Enforces |
|---|---|
| ADR |  |
| Branch protection |  |
| CI workflow |  |
| Required status check list |  |
| PR template |  |
| Auto-merge |  |

**Options:** Content: naming, build, tests · Nothing — it prompts the author · Nothing — it records the decision and its cost · Nothing — it removes the delay after conditions pass · Outcomes: a review happened, a check passed · That a CI job becomes a gate

<details>
<summary>Show answer</summary>

| Artifact | Enforces |
|---|---|
| ADR | **Nothing — it records the decision and its cost** |
| Branch protection | **Outcomes: a review happened, a check passed** |
| CI workflow | **Content: naming, build, tests** |
| Required status check list | **That a CI job becomes a gate** |
| PR template | **Nothing — it prompts the author** |
| Auto-merge | **Nothing — it removes the delay after conditions pass** |

**In `challenge-01.md`:** lines **75–94**, **177–185**, **332–408**, **467**, **254–285**, **315**.

**Three of the six enforce nothing, and knowing which three is the point.** The exam offers a template
or an ADR as the answer to "how do you prevent…", and prevention only ever comes from protection rules
plus a required check.

</details>

---

# Section F — Hot area

---

## Q36

```bash
gh api repos/OWNER/REPO/branches/main/protection \
  --method PUT \
  --field required_status_checks='[BLANK 1]' \
  --field [BLANK 2]=true \
  --field allow_force_pushes=[BLANK 3]
```

Requirement: checks must pass, the branch must be current with `main`, and the rules must bind
administrators.

- **BLANK 1:** `{"strict":false,"contexts":[]}` / `{"required":true}` /
  `{"strict":true,"contexts":["ci/build","ci/test"]}` / `{"checks":"all"}`
- **BLANK 2:** `restrictions` / `enforce_admins` / `allow_deletions` / `required_signatures`
- **BLANK 3:** `true` / `false`

<details>
<summary>Show answer</summary>

### Answer: the `strict:true` form, `enforce_admins`, `false`

**In `challenge-01.md`:** lines **180–184**.

```bash
  --field required_status_checks='{"strict":true,"contexts":["ci/build","ci/test"]}' \
  --field enforce_admins=true \
  ...
  --field allow_force_pushes=false \
```

**`strict` is what turns "checks passed" into "checks passed against current `main`"** — the difference
Q2 and Q26 both turn on.

**And `restrictions` is the distractor worth knowing.** It limits *who may push* to the branch and is
set to `null` at line 183 — meaning no push allowlist, because the pull-request requirement is doing
that job instead.

</details>

---

## Q37

```json
{
  "type": "[BLANK 1]",
  "parameters": {
    "required_approving_review_count": 1,
    "[BLANK 2]": true,
    "require_code_owner_reviews": true
  }
}
```

- **BLANK 1:** `required_reviews` / `branch_protection` / `merge_policy` / `pull_request`
- **BLANK 2:** `dismiss_stale_reviews` / `require_review` / `dismiss_stale_reviews_on_push` /
  `strict_reviews`

<details>
<summary>Show answer</summary>

### Answer: `pull_request`, `dismiss_stale_reviews_on_push`

**In `challenge-01.md`:** lines **216–222**.

```json
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 1,
        "dismiss_stale_reviews_on_push": true,
```

**The field name differs between the two APIs and the exam exploits it.** Classic protection calls it
`dismiss_stale_reviews` (line 182); a ruleset calls it `dismiss_stale_reviews_on_push`. Same behaviour,
different spelling — and BLANK 2's first option is the *other* API's name.

</details>

---

## Q38

```yaml
  validate-branch-name:
    runs-on: ubuntu-latest
    if: github.event_name == '[BLANK 1]'
    steps:
      - name: Check branch naming convention
        run: |
          BRANCH_NAME="[BLANK 2]"
          PATTERN="^(feature|bugfix|hotfix|chore|docs)/[a-z0-9._-]+$"
          if [[ ! "$BRANCH_NAME" =~ $PATTERN ]]; then
            echo "::[BLANK 3]::Branch name '$BRANCH_NAME' does not match pattern"
            exit 1
          fi
```

- **BLANK 1:** `push` / `pull_request` / `workflow_dispatch` / `schedule`
- **BLANK 2:** the `github.ref` expression / the `github.base_ref` expression /
  the `github.head_ref` expression / the `github.sha` expression
- **BLANK 3:** `notice` / `warning` / `debug` / `error`

<details>
<summary>Show answer</summary>

### Answer: `pull_request`, `github.head_ref`, `error`

**In `challenge-01.md`:** lines **380–390**.

**`head_ref` is the *source* branch of the PR; `base_ref` is the target.** Using `base_ref` here would
validate `main` against the pattern on every run — and `main` has no prefix, so every PR would fail.

**And `github.ref` on a pull-request event points at the merge ref**, not the branch name, so it would
not match either. This is the sort of detail you only get right after running the lab once.

**`::error::` pairs with the `exit 1`** at line 390 — compare the age check at line 406, which uses
`::warning::` and no exit (Q11).

</details>

---

## Q39

```bash
gh api repos/OWNER/REPO \
  --method PATCH \
  --field allow_auto_merge=[BLANK 1] \
  --field allow_squash_merge=true \
  --field allow_merge_commit=[BLANK 2] \
  --field squash_merge_commit_title="[BLANK 3]"
```

Requirement: linear history, and squashed commits that carry the reviewed description.

- **BLANK 1:** `false` / `true`
- **BLANK 2:** `true` / `false`
- **BLANK 3:** `COMMIT_OR_PR_TITLE` / `DEFAULT` / `PR_TITLE` / `BRANCH_NAME`

<details>
<summary>Show answer</summary>

### Answer: `true`, `false`, `PR_TITLE`

**In `challenge-01.md`:** lines **302–307**.

**`allow_merge_commit=false` is the line that actually delivers linear history.** Leaving it `true`
means the setting is a preference, and one developer choosing "Create a merge commit" reintroduces the
bubbles.

**`PR_TITLE` makes the permanent record the reviewed title** rather than the branch's last commit
subject (Q14).

</details>

---

## Q40

```bash
# Diagnose why a PR merged with a failing build
gh api repos/OWNER/REPO/commits/main/[BLANK 1] \
  --jq '.check_runs[].[BLANK 2]'
```

- **BLANK 1:** `statuses` / `protection` / `check-runs` / `pulls`
- **BLANK 2:** `conclusion` / `name` / `id` / `status`

<details>
<summary>Show answer</summary>

### Answer: `check-runs`, `name`

**In `challenge-01.md`:** lines **460–461**.

```bash
gh api repos/{owner}/{repo}/commits/main/check-runs \
  --jq '.check_runs[].name'
```

**You are asking what CI actually calls its jobs**, because the whole failure is a name mismatch (Q6).
`conclusion` would tell you whether they passed — useful, and not the question when the gate never
evaluated in the first place.

</details>

---

## Q41

```bash
gh api repos/OWNER/REPO/branches/main/protection/[BLANK 1] \
  --method PATCH \
  --field strict=true \
  --field contexts='[BLANK 2]'
```

Requirement: correct the required checks so they match the workflow's actual job names.

- **BLANK 1:** `required_pull_request_reviews` / `enforce_admins` / `required_status_checks` /
  `restrictions`
- **BLANK 2:** `["ci/build","ci/test"]` / `["build","test","validate-branch-name"]` / `["*"]` / `[]`

<details>
<summary>Show answer</summary>

### Answer: `required_status_checks`, `["build","test","validate-branch-name"]`

**In `challenge-01.md`:** lines **464–467**.

```bash
gh api repos/{owner}/{repo}/branches/main/protection/required_status_checks \
  --method PATCH \
  --field strict=true \
  --field contexts='["build","test","validate-branch-name"]'
```

**Compare with the original at line 180: `ci/build` and `ci/test`.** Those prefixes were never real —
the workflow's jobs are `build` (line 342) and `test` (line 361).

**And note what the fix adds: `validate-branch-name`.** Promoting the naming job into the required list
is what converts it from a report into a gate (Q19, Q35).

</details>

---

# Section G — Case study

## Case study: Contoso branching unification

### Background

Contoso Ltd has **five development teams working on a single monorepo**. Each invented its own branching
strategy: long-lived feature branches, direct commits to `main`, and one team maintaining **four parallel
release branches**. Merges regularly take **two to three days** to resolve conflicts, `main` is broken
**at least once per sprint**, and **nobody knows which branch represents the production state**. The VP
of Engineering has mandated a unified workflow based on **GitHub Flow**, with guardrails that prevent
direct pushes and broken merges.

### Requirements

**Strategy**

- One workflow all five teams follow, with the reasoning recorded so it is not reopened
- `main` must always be deployable and must be what production comes from
- No long-lived branches; incomplete work is handled without them

**Guardrails**

- Direct pushes to `main` must be impossible, including for administrators
- No PR may merge without a passing build and at least one current approval
- Two independently-green PRs must not be able to combine into a broken `main`
- Branch names must follow an agreed convention, enforced rather than requested

**Flow**

- A PR that satisfies every condition should not wait for someone to notice
- `main` history must stay linear and readable
- Merged branches must not accumulate

---

## Q42

How should the strategy decision be recorded?

- A. A message in the engineering channel, pinned for visibility
- B. A page in the team wiki, owned and maintained by the VP
- C. An ADR recording Status, Context, Decision and Consequences
- D. A section in the README describing the chosen workflow

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-01.md`:** lines **73–94**.

```text
## Consequences
- All teams follow one workflow
- Main is always deployable
- No long-lived branches (feature flags for incomplete work)
- Requires branch protection and CI gates
```

**"So it is not reopened" is the requirement**, and only the Consequences section satisfies it — it
states what was *given up*, which is what the next argument will be about.

**And it lives at `docs/decisions/001-branching-strategy.md`** (line 75), inside the repository, so it
is versioned with the code it governs and travels with a clone.

**Why B and C decay.** A channel message is unfindable in six months; a wiki page drifts from the
repository and has no reviewer.

</details>

---

## Q43

Which strategy meets the Strategy requirements, and why is the obvious alternative wrong?

- A. GitFlow — the team already maintains four parallel release branches
- B. Trunk-based — the simplest model available to a team this size
- C. Let each team keep its own model, but standardise branch names
- D. GitHub Flow — continuous deployment, one model, no release branches

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-01.md`:** lines **64–69**.

**Why A is the strongest distractor in this paper.** The four parallel release branches are real — and
they are listed at line 20 as a **symptom of the chaos being fixed**, not as a requirement to preserve.
Nothing in the scenario says Contoso supports four shipped versions; one team invented the branches.
**The exam quotes the broken state back at you and waits to see whether you treat it as a constraint.**

**Why B fails on readiness rather than on principle.** Trunk-based needs feature flags and comprehensive
automated testing (lines 58–59). A team whose `main` breaks every sprint has neither — adopting it would
remove the PR gate that is the entire fix.

**Why C is the status quo with better labels.**

</details>

---

## Q44

How is "direct pushes must be impossible, including for administrators" satisfied?

- A. Required pull request reviews on `main`, with no other setting
- B. A CI job that rejects any push landing directly on `main`
- C. Required pull request reviews with `enforce_admins` set to true
- D. Removing write access from everyone except the release team

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-01.md`:** lines **181–182** and **440–442**.

**Both halves are named in the requirement, and A supplies only one.** Without `enforce_admins`,
protection has a standing exception for the people under the most delivery pressure (Q5).

**Why B cannot work, and the reason is worth internalising.** CI runs *after* a push is accepted. It can
report that `main` is broken; it cannot prevent the push. **Only the server-side rule prevents.**

**Why D breaks the workflow it is protecting** — nobody could merge a PR either.

</details>

---

## Q45

Which **two** prevent two independently-green PRs from combining into a broken `main`? (Choose two.)

- A. `dismiss_stale_reviews` on the required review settings
- B. `strict: true` on the required status checks object
- C. `delete_branch_on_merge` on the repository settings
- D. CI triggered on pushes to `main` as well as pull requests
- E. The pull request template's "Type of change" section

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-01.md`:** lines **180** and **335–339**.

**B is the prevention; D is the detection.** Strict mode stops the second PR merging until it has been
validated against the first one's result. The push trigger on `main` catches anything that still slips
through, so the team learns within one build rather than at the next deployment.

**Why A guards a different failure.** Stale approvals are about the *reviewer's* knowledge going out of
date, not `main` moving.

**Why the pairing matters for the exam.** When a question asks how to stop a class of failure, a
complete answer usually has a preventive control **and** a way to find out when prevention was
insufficient.

</details>

---

## Q46

How should branch naming be "enforced rather than requested"?

- A. A CI job validating the pattern, reporting on every pull request
- B. A CI job validating the pattern, listed as a required status check
- C. A line in the pull request template asking authors to check it
- D. A note in the ADR recording the agreed naming convention

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-01.md`:** lines **378–390** and **467**.

**A is the answer that is 90% right and fails the requirement's last word.** The job runs, reports and
fails — and a PR can still merge unless the job is in the `contexts` list. That promotion at line 467 is
the difference between *requested* and *enforced*.

**Why C and D enforce nothing** (Q35). A template prompts; an ADR records.

**This is the single most transferable idea in Challenge 01**, and it recurs across the whole
certification: **a check becomes a gate only when something refuses to proceed on its result.**

</details>

---

## Q47

Nine months after the rollout, the team notices that PRs have been merging with a red
`validate-branch-name` job for about six weeks. Branch protection still lists it as required, and the
job still runs and still fails on bad names. The workflow file was recently refactored, and the job now
reads `validate-branch:`.

What is the most likely cause?

- A. `enforce_admins` was disabled at some point during the refactor
- B. Auto-merge began bypassing the configured required checks
- C. The rename broke the context match, so nothing reports that name
- D. `strict` mode expired and is no longer applied to the branch

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-01.md`:** lines **453** and **459–467**.

```text
The status check context names must match exactly what your CI reports.
```

**"The job still runs and still fails" is the detail that identifies it.** CI is healthy. The *link*
between CI and the rule is what broke, and a renamed job is the commonest way to break it.

**Why the symptom is merging rather than blocking is worth thinking through.** A required context that
never reports normally leaves a PR pending forever. In practice teams hit that within a day, someone
with admin rights edits the required list to unstick it, and the protection quietly loses a check —
which is how six weeks pass with a red job and green merges.

**The durable lesson: a required status check is a string match against a job name, and renaming a job
is a change to a security control.** Treat workflow refactors as touching branch protection, and re-run
the check-run listing at line 460 afterwards.

**Why the others fail** — A would allow direct pushes, not bad merges; B contradicts Q4; and nothing in
branch protection expires.

</details>

---

## Q48

A year on, Contoso's five teams merge into one monorepo without the old pain. `main` has not been broken
in four months.

Which explanation best accounts for the change?

- A. The five teams became more disciplined once the new model was explained
- B. Platform-enforced controls closed each path to a broken `main` in turn
- C. The pull request template made authors more careful about their changes
- D. Fewer developers were working in the monorepo than a year previously

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-01.md`:** lines **75–94**, **177–185**, **180**, **300–308**, **467**.

**Walk the controls in the order a bad change meets them.**

The **ADR** means nobody is still arguing about which model to use, so all five teams are producing the
same shape of change. **Required reviews plus `enforce_admins`** mean the only route to `main` is a pull
request, for everyone. **Required status checks** mean that pull request must build and test green.
**Strict mode** means it must be green *against the current `main`*, which is what stops two innocent
PRs combining into a broken one. And **squash plus auto-merge** mean the change lands the moment it
qualifies, with one readable commit, and the branch disappears.

**What actually changed is not that the teams became more careful.** The scenario at line 20 does not
describe careless engineers — it describes five reasonable groups following five reasonable
conventions with nothing to reconcile them. **The fix was to move the workflow out of people's habits
and into the repository's rules**, where it applies identically to every team, every day, including the
week before a deadline.

**That is the sentence to give any exam question about flow of work.** Choosing the strategy is the easy
half; the graded half is **what makes the choice hold when nobody is watching** — and the answer is
always a server-side rule with a check wired into it.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **"GitHub" read as "GitHub Flow"** | Q3, Q31, Q43 | Cadence and versions-kept-alive choose the strategy |
| **The broken state quoted as a requirement** | Q15, Q43 | Four release branches are the symptom, not the need |
| **Strategy bullets mixed between the three models** | Q17, Q18, Q27 | Know which list each line came from |
| **`enforce_admins` left off** | Q5, Q25, Q28, Q44 | A rule with an exception list is not a rule |
| **Required check names not matching job names** | Q6, Q19, Q34, Q40, Q41, Q47 | Silent. List the real check-run names |
| **A CI job assumed to be a gate** | Q19, Q35, Q46 | It gates only when listed in `contexts` |
| **`strict` disabled to reduce rebasing** | Q26, Q45 | Green + green can still be red |
| **Auto-merge read as a bypass** | Q4, Q20, Q29 | It waits for conditions; it never removes them |
| **`dismiss_stale_reviews` vs `strict`** | Q2, Q7 | One invalidates a review, one invalidates a build |
| **`dismiss_stale_reviews` vs `require_last_push_approval`** | Q7 | Staleness vs who pushed last |
| **`base_ref` instead of `head_ref`** | Q38 | `head_ref` is the PR's source branch |
| **Template or ADR offered as prevention** | Q16, Q35, Q42, Q46 | Neither enforces anything |
| **CI proposed to block a push** | Q44 | CI runs after the push is accepted |
| **Branch-age check assumed to fail the build** | Q11, Q30 | `::warning::`, no `exit 1` |

---

# What to memorise

**In `challenge-01.md`:** lines **37–60**, **177–185**, **300–308**, **459–467**.

```text
THE THREE STRATEGIES - chosen by CADENCE and VERSIONS KEPT ALIVE, never by platform
  GitHub Flow   main only, always deployable, short-lived branches, PR -> merge -> deploy from main
                for: continuously deployed services
  GitFlow       main + develop, plus release/* and hotfix/*
                for: PACKAGED SOFTWARE, formal versions, support windows (mobile, boxed)
  Trunk-based   main, branches < 1 DAY, FEATURE FLAGS, comprehensive automated tests
                for: mature CI/CD + flag infrastructure

Contoso picks GitHub Flow because: continuous deploys | one simple model | CI before main
                                   | DOES NOT NEED release branches
```

```bash
# Branch protection - classic API                    (lines 177-185)
gh api repos/OWNER/REPO/branches/main/protection --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["build","test"]}' \
  --field enforce_admins=true \
  --field required_pull_request_reviews='{"required_approving_review_count":1,"dismiss_stale_reviews":true,"require_code_owner_reviews":true}' \
  --field restrictions=null \
  --field allow_force_pushes=false \
  --field allow_deletions=false

#  strict            = branch must be UP TO DATE with main   -> stops green+green = red
#  enforce_admins    = admins are not exempt                 -> or the rule has an exception list
#  dismiss_stale     = approval dies when new commits land
#  restrictions=null = no push allowlist (the PR requirement does that job)
#  CONTEXTS MUST MATCH THE JOB NAMES EXACTLY. a check nothing reports never evaluates.
```

```json
// Ruleset equivalents - same intent, different names   (lines 214-240)
{ "type": "pull_request",           "parameters": { "dismiss_stale_reviews_on_push": true } }
{ "type": "required_status_checks", "parameters": { "strict_required_status_checks_policy": true } }
{ "type": "deletion" }          // == allow_deletions: false
{ "type": "non_fast_forward" }  // == allow_force_pushes: false
// conditions.ref_name.include: ["refs/heads/main"]
```

```bash
# Merge hygiene                                       (lines 300-308)
allow_auto_merge=true            # WAITS for conditions - never a bypass
delete_branch_on_merge=true
allow_squash_merge=true
allow_merge_commit=false         # <- this line is what makes history linear
allow_rebase_merge=true
squash_merge_commit_title="PR_TITLE"     squash_merge_commit_message="PR_BODY"

gh pr merge --auto --squash
#  merges when: checks pass + reviews approved + branch up to date (if strict)

# Diagnose "PR merged with a red build"               (lines 459-467)
gh api repos/OWNER/REPO/commits/main/check-runs --jq '.check_runs[].name'   # the REAL names
gh api repos/OWNER/REPO/branches/main/protection/required_status_checks --method PATCH \
  --field strict=true --field contexts='["build","test","validate-branch-name"]'
```

```yaml
# CI that enforces the strategy                       (lines 332-408)
on:
  pull_request: {branches: [main]}    # is this change safe to merge?
  push:         {branches: [main]}    # is main still deployable?

  validate-branch-name:               # RULE  -> ::error:: + exit 1
    if: github.event_name == 'pull_request'
    # head_ref = the PR's SOURCE branch (base_ref would be main and always fail)
    # PATTERN="^(feature|bugfix|hotfix|chore|docs)/[a-z0-9._-]+$"

  enforce-no-long-lived-branches:     # GUIDELINE -> ::warning::, NO exit
    steps: [{uses: actions/checkout@v4, with: {fetch-depth: 0}}]   # full history to date the branch
```

```text
WHAT ENFORCES WHAT
  ADR                    nothing - records the decision AND ITS COST (Consequences)
  PR template            nothing - prompts the author
  auto-merge             nothing - removes the delay after conditions pass
  branch protection      OUTCOMES  - a review happened, a check passed
  CI workflow            CONTENT   - naming, build, tests
  required status check  the JOIN  - turns a CI job into a gate
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 02 |
| 38–43 | Re-read the trap index and the three-strategy table, then move on |
| 30–37 | Rewrite the three-strategy table and the protection block from memory, then retake |
| Below 30 | Redo Tasks 1, 3 and 6 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 01.

:::danger The two questions

**How often do they ship, and how many versions stay alive?** That chooses the strategy. The word
"GitHub" in the question chooses nothing.

**Does anything refuse to proceed on this?** A CI job that reports is information. A CI job named in
`contexts` is a gate.

Contoso's teams were not careless — they were unreconciled. The fix was moving the workflow out of
habits and into rules.

:::
