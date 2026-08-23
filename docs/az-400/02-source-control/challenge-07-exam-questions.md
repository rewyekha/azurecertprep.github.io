---
sidebar_position: 1.5
toc_max_heading_level: 2
title: "Challenge 07: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 07 — AZ-400 exam questions

**48 questions** built only from what Challenge 07 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-07.md`**.

:::danger Read this before you start

**Cadence chooses the strategy.** Not the platform, not team preference, not what the last company did.

**Daily or weekly** → **trunk-based**. Branches under a day, feature flags to hide incomplete work,
continuous deployment from `main`.
**Every week or two** → **GitHub Flow**. Feature branches that live days, PR, merge, deploy.
**Monthly, with older versions still supported** → **release branching**. A long-lived
`release/x.y` that can be patched while `main` moves on.

**The single fact that decides the hardest questions: do you still support a shipped version?** If yes,
you need a branch that represents it, and only release branching has one. If no, a release branch is
pure overhead — long-lived, drifting, and conflict-prone.

Two more that carry marks.

**Rebase rewrites; merge records.** Rebasing replays your commits on top of the target, producing
**new SHAs**. Merging creates a merge commit and keeps both histories.

**Drift is a function of time, not size.** A release branch three weeks out of sync conflicts badly
whether it holds one commit or fifty — which is Break scenario 1.

The scenario at line 19 is three teams, three cadences, and hotfixes that **never make it back**.

:::

---

# Section A — Multiple choice

---

## Q1

A team releases daily, needs `main` always deployable, and uses feature flags to hide incomplete work.
Which strategy fits?

- A. Git Flow with `develop` and release branches
- B. Trunk-based development with short-lived branches
- C. Release branching with long-lived support branches
- D. GitHub Flow with feature branches lasting one to two weeks

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-07.md`:** line **25**.

```text
Trunk-based development keeps all developers working on a single branch with short-lived feature
branches (less than 24 hours) and feature flags to hide incomplete work.
```

**Two of the three clues in the question are trunk-based's defining requirements**: daily releases and
feature flags. The third — `main` always deployable — is what feature flags make possible, because
unfinished code can ship disabled.

**Why D is the closest wrong answer and worth understanding.** GitHub Flow also keeps `main`
deployable, and it does **not** require feature flags — its branches live days, so incomplete work stays
on the branch. At a daily cadence that becomes a bottleneck.

**Why A and C both add branches this team has no use for.** Line 197 in the decision matrix: supporting
old versions is **No** for trunk-based and GitHub Flow, **Yes** only for release branching.

</details>

---

## Q2

What is the primary risk of long-lived release branches?

- A. Disk space
- B. They drift from `main`, producing increasingly complex merge conflicts
- C. They block other developers from branching
- D. They slow down clones

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-07.md`:** lines **200** and **382**.

```text
| Integration risk | Low (always merged)| Medium | High (long-lived) |
```

**The decision matrix's last row is the trade you are accepting** when you choose release branching.
Long-lived branches buy you version support and cost you integration risk.

**And the mitigation is forward-integration, not avoidance** (lines 207–211): merge `main` into the
release branch on a schedule, so the divergence never grows large enough to be frightening.

**Why A and D describe branches as though they were copies.** A Git branch is a pointer to a commit; it
costs almost nothing to store.

</details>

---

## Q3

When you rebase a feature branch instead of merging, what happens to its history?

- A. The commits are replayed on top of the target branch, producing new SHAs
- B. A merge commit combines both histories
- C. The commits are replaced by a single squash commit
- D. The target branch is rewritten chronologically

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** lines **269–275** and **393**.

```bash
# Rebase onto latest main (replays commits on top)
git fetch origin
git rebase origin/main

# Fast-forward merge (no merge commit)
git checkout main
git merge feature/payment-gateway --ff-only
```

**New SHAs, because a commit's hash includes its parent** — change the parent and every commit
downstream gets a new identity, even though the changes are identical.

**Which is the reason for the rule "never rebase a shared branch".** Anyone who has pulled those commits
now holds ones that no longer exist upstream, and their next pull produces duplicates.

**Why B is what `merge` does** and **why C is what `--squash` does** — three distinct operations, and
the exam offers all three.

</details>

---

## Q4

A team uses GitHub Flow. A critical production bug appears. What is the correct procedure?

- A. Branch from the release tag, fix, merge to the release branch, cherry-pick to `main`
- B. Branch from `main`, fix, open a PR, merge, deploy
- C. Commit directly to `main` and deploy immediately
- D. Revert `main` to the last known good state and re-apply features

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-07.md`:** line **404**.

```text
In GitHub Flow, main is the single source of truth. All changes go through the same process...
Hotfixes follow the same workflow but with expedited review. There are no separate release branches
in GitHub Flow.
```

**Why A is the trap, and it is a very good one.** That procedure is *correct* — for **release
branching** (lines 162–173). The question specifies GitHub Flow, where the branch it tells you to fix
does not exist.

**"Expedited, not exempt" is the phrase to carry into the exam.** A hotfix compresses the review, it
does not skip the pull request — which is why C is wrong even under pressure.

**Read the strategy named in the question before you pick a hotfix procedure.** The exam relies on
candidates recognising the release-branch hotfix dance and choosing it regardless of context.

</details>

---

## Q5

Team Gamma fixes a bug on `release/v2.1`. What must happen next, and why?

- A. Cherry-pick the fix to `main`, so the next release does not regress
- B. Nothing — the fix ships with v2.1
- C. Merge `release/v2.1` into `main`
- D. Rebase `main` onto the release branch

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** lines **170–173**.

```bash
# After merging to release, cherry-pick to main
git checkout main
git cherry-pick <commit-sha>
git push origin main
```

**This is the step the scenario says Contoso skips** — line 19: *"hotfixes that never make it back to
feature branches"*. The consequence is a bug fixed in v2.1 and **reintroduced** in v2.2, because `main`
never received the fix.

**Why C is the plausible alternative that causes a different problem.** Merging the whole release branch
into `main` brings back everything on it, including version-specific changes that should not be in the
next release.

**Cherry-pick moves one commit**, which is exactly the granularity a hotfix needs.

</details>

---

## Q6

A developer pushed three commits directly to `main`. How does the challenge recover?

- A. Branch from `main` to preserve the work, reset `main` back three commits, force-with-lease, then
  open a PR from the branch
- B. Revert each commit
- C. Delete and recreate `main`
- D. Leave them and add branch protection

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** lines **345–356**.

```bash
git branch feature/recover-direct-push
git reset --hard HEAD~3
git push origin main --force-with-lease
```

**Preserve first, then reset.** Creating the branch before the reset is what makes the work recoverable
— `reset --hard` discards it otherwise.

**`--force-with-lease` rather than `--force` is the detail worth learning.** It refuses the push if
someone else has updated `main` since you last fetched, so you cannot silently destroy a colleague's
commits. **Plain `--force` has no such check.**

**Why B is the safer choice on a busy shared branch**, and it is not what this challenge does. Reverting
adds three new commits and rewrites nothing — preferable when others have already pulled.

</details>

---

## Q7

What does `--no-ff` do on a merge?

- A. Forces a merge commit even when a fast-forward is possible
- B. Prevents the merge if there are conflicts
- C. Skips the commit message editor
- D. Merges without fetching

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** lines **256–258**.

```bash
git merge feature/payment-gateway --no-ff
# Creates: "Merge branch 'feature/payment-gateway' into main"
```

**The merge commit is the record that a branch existed.** Without `--no-ff`, a branch whose parent has
not moved simply fast-forwards, and the history shows the commits as though they were always on `main`.

**Which is the argument for it: `--no-ff` preserves the shape of the work.** The counter-argument is at
line 275 — `--ff-only` after a rebase gives a clean linear history with no merge bubbles.

**Both appear in Task 6 because they are the two ends of the same decision** (Q3).

</details>

---

## Q8

What does `--ff-only` guarantee?

- A. The merge succeeds only if it can fast-forward, otherwise it fails
- B. It always creates a merge commit
- C. It squashes the branch
- D. It rebases automatically

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** lines **274–275**.

**It is a safety assertion, not a merge strategy.** You have just rebased, so the branch should be a
direct descendant of `main` — `--ff-only` fails loudly if that assumption is wrong, rather than silently
creating the merge commit you were trying to avoid.

**Use it as a check on the rebase.** If `--ff-only` fails, `main` moved while you were rebasing, and you
need to fetch and rebase again.

</details>

---

## Q9

Which release cadence does the decision matrix associate with GitHub Flow?

- A. Daily or weekly
- B. Bi-weekly
- C. Monthly or quarterly
- D. Continuous

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-07.md`:** line **193**.

```text
| Release cadence | Daily/weekly | Bi-weekly | Monthly/quarterly |
```

**Three columns, three cadences — and this row alone answers most strategy questions.** Trunk-based
daily/weekly, GitHub Flow bi-weekly, release branching monthly/quarterly.

**Which is exactly how the scenario assigns the three teams** (line 19): Alpha weekly → trunk-based,
Beta bi-weekly → GitHub Flow, Gamma monthly → release branching.

</details>

---

## Q10

Which strategy **requires** deployment automation according to the matrix?

- A. Trunk-based
- B. GitHub Flow
- C. Release branching
- D. All three

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** line **195**.

```text
| Deployment automation | Required (CD) | Recommended | Optional |
```

**Trunk-based without CD is just a shared branch with no safety net.** Every merge is meant to be
deployable, and if deployment is manual, the "always deployable" claim is never tested.

**Note the gradient across the row**: Required, Recommended, Optional. **The faster the cadence, the more
the strategy depends on automation** — which is a genuinely useful thing to be able to say in a case
study.

</details>

---

## Q11

Which strategy needs feature flags?

- A. Trunk-based
- B. GitHub Flow
- C. Release branching
- D. None

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** lines **196** and **56–62**.

```json
{
  "RATE_LIMIT_ENABLED": {
    "enabled": false,
    "rollout_percentage": 0,
    "description": "Enable API rate limiting (in progress)"
  }
}
```

**Trunk-based merges incomplete work to `main` deliberately**, so something must hide it from users. The
flag is what makes a sub-day branch possible on a multi-day feature.

**GitHub Flow says "Optional"** (line 196) because the branch itself hides the work. **Release branching
says "No"** — an unfinished feature simply misses the release train.

**And note `rollout_percentage`** alongside `enabled` — the same flag that hides work can later expose it
gradually, which is Challenge 27's progressive delivery.

</details>

---

## Q12

What is trunk-based development's rollback strategy in the matrix?

- A. Turn the feature flag off
- B. Revert the commit
- C. Deploy the prior release
- D. Reset `main`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** line **199**.

```text
| Rollback strategy | Feature flag off | Revert commit | Deploy prior release |
```

**Three strategies, three rollbacks — and they get progressively slower.** A flag toggle is instant and
needs no deployment. A revert needs a commit, a build and a deploy. Redeploying a prior release needs
that release to still exist and to still be compatible.

**Which is the hidden argument for feature flags.** They are not only about hiding incomplete work; they
are the fastest rollback mechanism available, because the code is already deployed and merely disabled.

</details>

---

## Q13

How does the drift-detection workflow decide a branch needs syncing?

- A. When it is more than 50 commits **behind** `main`
- B. When it is more than 50 commits ahead
- C. When it is older than 30 days
- D. When a conflict is detected

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** lines **233–237**.

```bash
            AHEAD=$(git rev-list --count "origin/main..${branch}")
            BEHIND=$(git rev-list --count "${branch}..origin/main")
            ...
            if [ "$BEHIND" -gt 50 ]; then
```

**Read the two ranges carefully, because they are easy to invert.** `main..branch` counts commits on the
branch that `main` lacks — that is *ahead*. `branch..main` counts commits on `main` the branch lacks —
that is *behind*.

**And *behind* is the one that predicts pain.** A release branch that has fallen behind `main` will
conflict when you forward-integrate; being ahead simply means it has hotfixes to cherry-pick back
(Q5).

</details>

---

## Q14

What does the drift workflow do when the threshold is exceeded?

- A. Emits a warning and opens a maintenance issue
- B. Fails the workflow
- C. Merges `main` automatically
- D. Deletes the branch

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** lines **237–240**.

```bash
              echo "::warning::${branch} is ${BEHIND} commits behind main - sync needed"
              gh issue create --title "Branch drift: ${branch} is ${BEHIND} commits behind" \
                --body "Please merge main into ${branch} to reduce drift." \
                --label "maintenance"
```

**Warn and create work; do not act.** A scheduled job that failed every Monday would be muted within a
month, and one that merged automatically would resolve conflicts nobody reviewed.

**The issue is the mechanism that matters** — it is assignable and closable, so the drift becomes tracked
work rather than a log line (the same idea as Challenge 04's weekly metrics issue).

</details>

---

## Q15

Why does the drift workflow use `fetch-depth: 0`?

- A. `git rev-list --count` needs full history to count commits between refs
- B. To fetch all tags
- C. To speed up the checkout
- D. To include submodules

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** lines **227–229**.

```yaml
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
```

**A shallow clone has one commit, so every count returns 0** — and the workflow reports that every
release branch is perfectly in sync while they diverge.

**A green run reporting no drift is worse than no run at all**, because it creates confidence. This is
the same trap as Challenge 03's commitlint and Challenge 05's changelog: **any job that reads history
needs the history.**

</details>

---

## Q16

What does interactive rebase accomplish before opening a PR?

- A. It squashes messy work-in-progress commits into a few meaningful ones
- B. It merges the branch
- C. It resolves conflicts automatically
- D. It rewrites the target branch

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** lines **281–289**.

```text
# pick abc1234 feat: add Stripe integration
# squash def5678 wip: stripe config
# squash ghi9012 fix: typo in stripe key
# pick jkl3456 feat: add payment webhook handler
# squash mno7890 fix: webhook signature validation
```

**Five commits become two, and the two that survive are the ones a reviewer wants to read.** "wip: stripe
config" and "fix: typo" are working notes, not history.

**And it is safe here specifically because the branch has not been pushed or shared.** Interactive rebase
rewrites history (Q3) — doing it on a branch others have pulled causes the duplicate-commit problem.

**Note the alternative the challenge also uses**: `gh pr merge --squash` (line 49) achieves a similar
result at merge time, without the developer running a rebase at all.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** describe trunk-based development? (Choose three.)

- A. Feature branches live less than 24 hours
- B. Feature flags hide incomplete work
- C. Continuous deployment is required
- D. Long-lived release branches are maintained
- E. Old versions are supported from branches
- F. Code review is always a formal PR gate

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-07.md`:** lines **25**, **195–196**.

**Why F is the row people get wrong.** Line 198 says code review is **"Optional (pair)"** for
trunk-based — pairing is treated as review, so a formal PR is not mandatory. GitHub Flow and release
branching both say **"Required (PR)"**.

**That does not mean trunk-based skips review.** It means the review can happen *while* the code is
written rather than after, which is what makes a sub-day branch achievable.

</details>

---

## Q18

Which **three** are true of release branching? (Choose three.)

- A. Older versions can be supported from their branches
- B. Integration risk is high
- C. Hotfixes must be cherry-picked back to `main`
- D. Feature flags are required
- E. It suits daily releases
- F. Deployment automation is required

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-07.md`:** lines **197**, **200**, **170–173**.

**A is the *only* reason to choose it**, and B is the price. If nobody is running v2.1 any more, the
branch is cost with no benefit.

**C is the discipline that makes it work**, and the one Contoso skips (line 19).

**Why D, E and F are inverted rows** — feature flags "No" (line 196), cadence monthly/quarterly (line
193), automation "Optional" (line 195).

</details>

---

## Q19

Which **two** distinguish merge from rebase? (Choose two.)

- A. Merge creates a merge commit and preserves both histories
- B. Rebase replays commits and produces new SHAs
- C. Merge produces new SHAs
- D. Rebase creates a merge commit
- E. Both are safe on shared branches

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-07.md`:** lines **256–258**, **269–271**, **393**.

**Why E is the practical rule that follows from B.** Rebasing rewrites history, so a branch others have
pulled must not be rebased — their commits and yours are now different objects containing the same
changes.

**A useful way to hold it: merge is additive, rebase is destructive-and-recreative.** Merge only ever
adds a commit; rebase replaces every commit it touches.

</details>

---

## Q20

Which **two** prevent release branch drift? (Choose two.)

- A. Scheduled forward-integration merges of `main` into the release branch
- B. An automated drift check that raises an issue past a threshold
- C. Rebasing the release branch onto `main`
- D. Deleting the release branch weekly
- E. Squashing the release branch

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-07.md`:** lines **207–211** and **216–243**.

**A is the cure; B is the detection.** Merging on a schedule keeps the divergence small; the Monday check
catches the branch nobody merged.

**Why C is technically possible and wrong here.** `release/v2.1` is a **shared, published** branch —
rebasing it rewrites history everyone has pulled (Q19). Forward-integration uses `merge` at line 209 for
exactly that reason.

</details>

---

## Q21

Which **two** are correct about the direct-push recovery? (Choose two.)

- A. Create a branch at the current tip before resetting, to preserve the commits
- B. Use `--force-with-lease` rather than `--force`
- C. Use `git revert` to remove them from history
- D. `--force` and `--force-with-lease` behave identically
- E. Reset the branch, then recover from the reflog

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-07.md`:** lines **346–351**.

**A is ordering: `reset --hard` discards the commits, so the branch must exist first.**

**B is the safety property.** `--force-with-lease` compares the remote to what you last fetched and
refuses if someone else has pushed — so you cannot destroy work you never saw. Plain `--force`
overwrites unconditionally.

**Why E would work and is a worse plan.** The reflog is local, time-limited, and recovering from it under
pressure is error-prone. Creating a branch first is deliberate rather than hopeful.

</details>

---

## Q22

Which **two** does the drift-detection job compute? (Choose two.)

- A. Commits the branch is **behind** `main`, with `branch..origin/main`
- B. Commits the branch is **ahead** of `main`, with `origin/main..branch`
- C. The age of the branch in days
- D. The number of conflicting files
- E. The branch's author

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-07.md`:** lines **233–234**.

```bash
            AHEAD=$(git rev-list --count "origin/main..${branch}")
            BEHIND=$(git rev-list --count "${branch}..origin/main")
```

**Read a double-dot range as "reachable from the right, not from the left".** `main..branch` = what the
branch has that `main` does not = ahead.

**Both numbers are reported** (line 235), and they mean different things: **behind** predicts conflict
pain, **ahead** counts hotfixes that may still need cherry-picking back (Q13).

</details>

---

## Q23

Which **two** problems from the scenario does a per-team strategy solve? (Choose two.)

- A. Broken builds when branches diverge for weeks
- B. Hotfixes that never reach `main`
- C. Slow clone times
- D. Missing test coverage
- E. Unreviewed code reaching production

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-07.md`:** line **19**, with **207–243** and **170–173**.

**A is solved by matching branch lifetime to cadence** — plus drift detection for the one strategy that
still has long-lived branches.

**B is solved by making the cherry-pick a defined step** of the release-branch workflow rather than
something someone might remember.

**Why E is Challenge 08's problem.** Branch protection and required reviews are the mechanism; a
branching strategy describes *where* work happens, not *who approves it*.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso's three teams ship weekly, bi-weekly and monthly. `main` must always be deployable,
each team needs a strategy suited to its cadence, and hotfixes must reach `main`.

---

## Q24

**Proposed solution:** Team Alpha uses trunk-based development with branches under a day, feature flags,
squash merges and continuous deployment on merge to `main`. Team Beta uses GitHub Flow with feature
branches, reviewed PRs and a scheduled bi-weekly release. Team Gamma cuts `release/vX.Y` from `main`,
fixes bugs on the release branch and cherry-picks each fix to `main`, tagging patch releases. A weekly
job reports how far each release branch is behind `main` and opens an issue past the threshold.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-07.md`:** lines **27–86**, **92–144**, **150–183**, **216–243**.

| Team | Cadence | Strategy | Why |
|---|---|---|---|
| Alpha | Weekly | Trunk-based | Flags hide incomplete work; CD keeps `main` proven |
| Beta | Bi-weekly | GitHub Flow | Branches live days; PR gate; scheduled release |
| Gamma | Monthly | Release branching | Only strategy that can patch a shipped version |

**The cherry-pick step is what fixes the scenario's named failure**, and the drift job is what stops the
one long-lived branch becoming the integration problem it was before.

</details>

---

## Q25

**Proposed solution:** Standardise all three teams on release branching so the process is consistent.
Every team cuts a release branch, works on it until the release ships, and merges it back to `main`
afterwards. Skip drift checks, since branches are short-lived by policy.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures, and the first is the reasoning itself.**

**Consistency is not the requirement.** Line 19 says each team needs a strategy **suited to their
cadence** — a weekly team running monthly ceremony spends more time on branch management than on the
release.

**Working on the release branch inverts the model.** Development belongs on `main`; the release branch
is cut *when feature-complete* (line 151) and receives only fixes.

**Merging the release branch back to `main`** brings version-specific changes with it, which is why the
challenge cherry-picks individual fixes instead (line 172).

**And "short-lived by policy" is the assumption drift detection exists to test.** A branch is short-lived
until the release slips — at which point nobody is checking.

</details>

---

## Q26

**Proposed solution:** Team Alpha uses trunk-based with feature flags and CD. Team Beta uses GitHub Flow
with reviewed PRs and a scheduled release. Team Gamma cuts release branches, fixes bugs there and
cherry-picks to `main`, with a weekly drift job. To keep the release branch's history clean and linear,
rebase `release/v2.1` onto `main` during each weekly sync instead of merging.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**`release/v2.1` is pushed and shared** (line 155), so rebasing it rewrites commits that other developers
and the CI system already have. Every hotfix commit gets a new SHA, and anyone with the branch checked
out gets duplicate commits on their next pull.

**And it breaks the thing the release branch exists for.** The tags `v2.1.0` and `v2.1.1` (lines 177,
181) point at specific commits. After a rebase those commits are orphaned, so the tags reference
history that is no longer on the branch — and "deploy the prior release" (line 199), release
branching's entire rollback strategy, now deploys something that does not match the branch.

**The stated goal is also unachievable in principle.** A release branch **diverges by design** — that is
what makes it a release branch. Linear history and long-lived parallel branches are mutually exclusive.

**Line 209 uses `git merge main --no-edit` deliberately**: forward-integration on a shared branch is a
merge, always.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — strategy selection

| # | Statement | Answer |
|---|---|---|
| 1 | Trunk-based suits daily or weekly releases |  |
| 2 | GitHub Flow supports older shipped versions |  |
| 3 | Release branching suits monthly or quarterly cadence |  |
| 4 | Trunk-based requires deployment automation |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Trunk-based suits daily or weekly releases | **Yes** |
| 2 | GitHub Flow supports older shipped versions | **No** |
| 3 | Release branching suits monthly or quarterly cadence | **Yes** |
| 4 | Trunk-based requires deployment automation | **Yes** |

**In `challenge-07.md`:** lines **193**, **197**, **193**, **195**.

Row 2 is the decisive question in most strategy scenarios. **Only release branching answers Yes to
"support old versions".**

</details>

---

## Q28 — merge and rebase

| # | Statement | Answer |
|---|---|---|
| 1 | Rebase produces new commit SHAs |  |
| 2 | `--no-ff` forces a merge commit |  |
| 3 | `--ff-only` creates a merge commit |  |
| 4 | Rebasing a shared branch is safe |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Rebase produces new commit SHAs | **Yes** |
| 2 | `--no-ff` forces a merge commit | **Yes** |
| 3 | `--ff-only` creates a merge commit | **No** |
| 4 | Rebasing a shared branch is safe | **No** |

**In `challenge-07.md`:** lines **393**, **257**, **275**, and Q19.

Row 3 is the opposite of `--no-ff` — it **refuses** unless the merge can fast-forward, so no merge commit
is ever created.

Row 4 follows from row 1 and is the rule behind Q26.

</details>

---

## Q29 — release branch hygiene

| # | Statement | Answer |
|---|---|---|
| 1 | A fix on the release branch must be cherry-picked to `main` |  |
| 2 | Forward-integration merges `main` into the release branch |  |
| 3 | `branch..origin/main` counts how far the branch is behind |  |
| 4 | The drift job fails the workflow past the threshold |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A fix on the release branch must be cherry-picked to `main` | **Yes** |
| 2 | Forward-integration merges `main` into the release branch | **Yes** |
| 3 | `branch..origin/main` counts how far the branch is behind | **Yes** |
| 4 | The drift job fails the workflow past the threshold | **No** |

**In `challenge-07.md`:** lines **170–173**, **207–211**, **234**, **237–240**.

Row 4: it warns and opens an issue (Q14). **A recurring maintenance check that fails gets muted.**

</details>

---

## Q30 — recovery

| # | Statement | Answer |
|---|---|---|
| 1 | Create the recovery branch before resetting |  |
| 2 | `--force-with-lease` refuses if the remote moved |  |
| 3 | `--force` performs the same check |  |
| 4 | `reset --hard` discards uncommitted and unreferenced work |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Create the recovery branch before resetting | **Yes** |
| 2 | `--force-with-lease` refuses if the remote moved | **Yes** |
| 3 | `--force` performs the same check | **No** |
| 4 | `reset --hard` discards uncommitted and unreferenced work | **Yes** |

**In `challenge-07.md`:** lines **346–351**.

Rows 1 and 4 are the same fact from two sides, and the ordering in the challenge exists because of it.

Row 3 is the distinction worth internalising: **`--force-with-lease` is `--force` with a seatbelt.**

</details>

---

# Section E — Drag and drop

---

## Q31

Match each team to its strategy and the deciding factor.

| Team | Strategy — deciding factor |
|---|---|
| Alpha, ships weekly |  |
| Beta, ships bi-weekly |  |
| Gamma, ships monthly and patches v2.1 |  |
| A team with no CD pipeline |  |
| A team that cannot use feature flags |  |

**Options:** GitHub Flow — branches live days, PR gate · Not trunk-based — automation is required · Not trunk-based — flags are required · Release branching — a shipped version needs a branch · Trunk-based — cadence too fast for long branches

<details>
<summary>Show answer</summary>

| Team | Strategy — deciding factor |
|---|---|
| Alpha, ships weekly | **Trunk-based — cadence too fast for long branches** |
| Beta, ships bi-weekly | **GitHub Flow — branches live days, PR gate** |
| Gamma, ships monthly and patches v2.1 | **Release branching — a shipped version needs a branch** |
| A team with no CD pipeline | **Not trunk-based — automation is required** |
| A team that cannot use feature flags | **Not trunk-based — flags are required** |

**In `challenge-07.md`:** lines **19**, **193–196**.

**The last two rows are the disqualifiers.** Trunk-based is not a choice you can make by preference; it
has two hard prerequisites, and a team lacking either must use GitHub Flow instead.

</details>

---

## Q32

Match each matrix row to what it tells you.

| Row | Tells you |
|---|---|
| Release cadence |  |
| Support old versions |  |
| Feature flags needed |  |
| Rollback strategy |  |
| Integration risk |  |
| Deployment automation |  |

**Options:** Flag off / revert commit / deploy prior release · Low / medium / high — the price of long branches · Required / recommended / optional · Required for trunk-based, optional for GitHub Flow · Which column you are in before anything else · Yes only for release branching

<details>
<summary>Show answer</summary>

| Row | Tells you |
|---|---|
| Release cadence | **Which column you are in before anything else** |
| Support old versions | **Yes only for release branching** |
| Feature flags needed | **Required for trunk-based, optional for GitHub Flow** |
| Rollback strategy | **Flag off / revert commit / deploy prior release** |
| Integration risk | **Low / medium / high — the price of long branches** |
| Deployment automation | **Required / recommended / optional** |

**In `challenge-07.md`:** lines **193–200**.

**Two rows decide almost every question: cadence and old-version support.** The rest describe the
consequences of that choice.

</details>

---

## Q33

Arrange the release-branching workflow for a monthly team.

**Items:** Cherry-pick the fix to `main` · Tag the patch release · Cut `release/v2.1` from `main` · Fix
the bug on a branch off `release/v2.1` · Tag `v2.1.0` · Continue next-release work on `main`

<details>
<summary>Show answer</summary>

### Answer

1. Cut `release/v2.1` from `main` — lines **152–155**
2. Continue next-release work on `main` — lines **158–160**
3. Tag `v2.1.0` — lines **176–178**
4. Fix the bug on a branch off `release/v2.1` — lines **163–168**
5. Cherry-pick the fix to `main` — lines **171–173**
6. Tag the patch release — lines **181–182**

**Steps 4 and 5 are inseparable, and the exam separates them to see whether you notice.** A fix that stops
at step 4 ships in v2.1.1 and is **absent from v2.2** — the regression the scenario describes at line
19.

**And step 2 is what makes the whole strategy worthwhile.** `main` never waits for the release; the two
proceed in parallel, which is the entire reason for cutting a branch.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| A fix present in v2.1.1 reappears as a bug in v2.2 |  |
| A three-week-old release branch conflicts badly |  |
| Colleagues see duplicate commits after a pull |  |
| The drift job reports zero drift on every branch |  |
| A force push silently destroyed a colleague's work |  |
| A daily-release team spends more time branching than shipping |  |

**Options:** A shared branch was rebased · Fix never cherry-picked to `main` · `--force` instead of `--force-with-lease` · No forward-integration merges · Release branching at the wrong cadence · Shallow checkout — `rev-list --count` sees one commit

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| A fix present in v2.1.1 reappears as a bug in v2.2 | **Fix never cherry-picked to `main`** |
| A three-week-old release branch conflicts badly | **No forward-integration merges** |
| Colleagues see duplicate commits after a pull | **A shared branch was rebased** |
| The drift job reports zero drift on every branch | **Shallow checkout — `rev-list --count` sees one commit** |
| A force push silently destroyed a colleague's work | **`--force` instead of `--force-with-lease`** |
| A daily-release team spends more time branching than shipping | **Release branching at the wrong cadence** |

**In `challenge-07.md`:** lines **19** with **172**, **207–211**, **393**, **229**, **351**, **193**.

**Rows 4 and 1 are the silent ones.** A drift job reporting no drift builds confidence in a branch that
is diverging, and a missing cherry-pick is invisible until the next release regresses.

</details>

---

## Q35

Match each rollback approach to its strategy and speed.

| Rollback | Strategy — speed |
|---|---|
| Turn the feature flag off |  |
| Revert the commit |  |
| Deploy the prior release |  |
| Reset and force-push `main` |  |

**Options:** GitHub Flow — needs build and deploy · Recovery from a mistake, not a rollback strategy · Release branching — needs that release to still exist · Trunk-based — instant, no deployment

<details>
<summary>Show answer</summary>

| Rollback | Strategy — speed |
|---|---|
| Turn the feature flag off | **Trunk-based — instant, no deployment** |
| Revert the commit | **GitHub Flow — needs build and deploy** |
| Deploy the prior release | **Release branching — needs that release to still exist** |
| Reset and force-push `main` | **Recovery from a mistake, not a rollback strategy** |

**In `challenge-07.md`:** lines **199** and **345–356**.

**The row that matters for the exam is the last one.** `reset --hard` plus force-push is how you undo an
**accidental direct push**; it is never the answer to "how do we roll back a bad release", because it
rewrites shared history.

</details>

---

# Section F — Hot area

---

## Q36

```bash
git checkout -b feature/add-rate-limiting
git commit -m "feat: add rate limiting middleware"

git fetch origin
git [BLANK 1] origin/main

git push origin feature/add-rate-limiting
gh pr create --base main
gh pr merge --[BLANK 2] --delete-branch
```

Requirement: trunk-based development with a linear history on `main`.

- **BLANK 1:** `rebase` / `merge` / `cherry-pick` / `reset`
- **BLANK 2:** `squash` / `merge` / `rebase` / `admin`

<details>
<summary>Show answer</summary>

### Answer: `rebase`, `squash`

**In `challenge-07.md`:** lines **39–49**.

**Rebase before pushing keeps the branch a direct descendant of `main`; squash on merge keeps `main` one
commit per change.** Together they produce the clean linear log trunk-based development depends on.

**And rebasing here is safe** because the branch is unpushed at that point (line 40 precedes the push at
line 43) — the shared-branch rule in Q26 does not apply.

**Compare Team Beta at line 115**, which uses `gh pr merge --merge` — a merge commit, deliberately, for a
branch that carried several days of related commits worth preserving.

</details>

---

## Q37

```bash
# Cut the release branch
git checkout main
git checkout -b [BLANK 1]
git push origin release/v2.1

# Fix on the release branch, then...
git checkout main
git [BLANK 2] <commit-sha>
```

- **BLANK 1:** `release/v2.1` / `hotfix/v2.1` / `develop` / `main-v2.1`
- **BLANK 2:** `cherry-pick` / `merge` / `rebase` / `revert`

<details>
<summary>Show answer</summary>

### Answer: `release/v2.1`, `cherry-pick`

**In `challenge-07.md`:** lines **154** and **172**.

**`cherry-pick` moves one commit; `merge` would bring the whole branch.** That granularity is the point —
you want the tax fix on `main`, not v2.1's version bumps and release-specific configuration.

**Why `revert` is the opposite operation.** It creates a commit that undoes changes; here you are
propagating them forward.

</details>

---

## Q38

```bash
git merge feature/payment-gateway --[BLANK 1]
# Creates a merge commit even when fast-forward is possible

git rebase origin/main
git merge feature/payment-gateway --[BLANK 2]
# Fails if a fast-forward is not possible
```

- **BLANK 1:** `no-ff` / `ff-only` / `squash` / `no-commit`
- **BLANK 2:** `ff-only` / `no-ff` / `abort` / `strategy=ours`

<details>
<summary>Show answer</summary>

### Answer: `no-ff`, `ff-only`

**In `challenge-07.md`:** lines **257** and **275**.

**Two opposite assertions about the same situation.** `--no-ff` says "I want a merge commit even though I
do not need one"; `--ff-only` says "I expect no merge commit, and fail if that is not true".

**`--ff-only` after a rebase is a check on the rebase** (Q8). If it fails, `main` moved and you must
fetch and rebase again — better than discovering it in a surprise merge commit.

</details>

---

## Q39

```bash
for branch in $(git branch -r | grep 'release/'); do
  AHEAD=$(git rev-list --count "[BLANK 1]")
  BEHIND=$(git rev-list --count "[BLANK 2]")
  if [ "$BEHIND" -gt [BLANK 3] ]; then
    echo "::warning::${branch} is ${BEHIND} commits behind main - sync needed"
  fi
done
```

- **BLANK 1:** `origin/main..${branch}` / `${branch}..origin/main` / `origin/main...${branch}` /
  `${branch}`
- **BLANK 2:** `${branch}..origin/main` / `origin/main..${branch}` / `HEAD..${branch}` / `main`
- **BLANK 3:** `50` / `5` / `500` / `0`

<details>
<summary>Show answer</summary>

### Answer: `origin/main..${branch}`, `${branch}..origin/main`, `50`

**In `challenge-07.md`:** lines **233–236**.

**Reverse the two ranges and the workflow warns about branches that are *ahead*** — which is the normal,
healthy state of a release branch carrying hotfixes, so it would alert constantly and be ignored.

**Read the range as "reachable from the right, excluding the left".** `origin/main..branch` = what the
branch has and `main` does not.

</details>

---

## Q40

```bash
# A developer pushed 3 commits directly to main
git checkout main
git [BLANK 1] feature/recover-direct-push

git reset --hard [BLANK 2]
git push origin main --[BLANK 3]
```

- **BLANK 1:** `branch` / `checkout -b` / `switch` / `tag`
- **BLANK 2:** `HEAD~3` / `HEAD` / `origin/main` / `HEAD^^`
- **BLANK 3:** `force-with-lease` / `force` / `no-verify` / `mirror`

<details>
<summary>Show answer</summary>

### Answer: `branch`, `HEAD~3`, `force-with-lease`

**In `challenge-07.md`:** lines **347–351**.

**`git branch` creates the pointer without switching to it**, so the next command still operates on
`main`. `checkout -b` would switch, and the reset would then hit the wrong branch.

**`HEAD~3` moves back three commits**, matching the three that were pushed.

**And `--force-with-lease` is the difference between a controlled correction and a second incident**
(Q21).

</details>

---

## Q41

```yaml
on:
  schedule:
    - cron: '[BLANK 1]'   # Every Monday at 9 AM

jobs:
  check-drift:
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: [BLANK 2]
```

- **BLANK 1:** `0 9 * * 1` / `9 0 * * 1` / `0 9 1 * *` / `* 9 * * 1`
- **BLANK 2:** `0` / `1` / `50` / `100`

<details>
<summary>Show answer</summary>

### Answer: `0 9 * * 1`, `0`

**In `challenge-07.md`:** lines **221** and **229**.

**Cron order is minute, hour, day-of-month, month, day-of-week** — so `0 9 * * 1` is minute 0, hour 9,
any day of month, any month, weekday 1 (Monday). `0 9 1 * *` would be the **first of the month**, which
is the bi-weekly release schedule's shape at line 125.

**And `fetch-depth: 0` is not optional** (Q15): a shallow clone makes every drift count zero, and the job
reports perfect health for branches that are diverging.

</details>

---

# Section G — Case study

## Case study: Contoso three-cadence platform

### Background

Contoso Ltd has **three development teams** on a shared platform. **Team Alpha ships weekly** (mobile
API), **Team Beta ships bi-weekly** (web dashboard), **Team Gamma ships monthly** (billing engine). Each
adopted a different branching approach **without coordination**. The result: **integration pain every
release**, **broken builds when branches diverge for weeks**, and **hotfixes that never make it back to
feature branches**. The CTO has mandated that **`main` must always be deployable** and that each team
needs a strategy suited to its cadence.

### Requirements

**Per team**

- Alpha must be able to merge several times a day without exposing unfinished work
- Beta must keep a formal review gate and ship on a predictable fortnightly schedule
- Gamma must be able to patch the shipped billing release while next-release work continues

**Shared**

- `main` must always be deployable
- A fix applied to a shipped version must also reach `main`
- Long-lived branches must not be allowed to drift unnoticed
- A mistaken direct push to `main` must be recoverable without losing the work

---

## Q42

Which strategy should Team Alpha use, and what are its prerequisites?

- A. Trunk-based — requires feature flags and continuous deployment
- B. GitHub Flow — requires a PR gate
- C. Release branching — requires version tags
- D. Git Flow — requires a `develop` branch

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** lines **25**, **195–196**.

**"Merge several times a day without exposing unfinished work" names both prerequisites in one sentence**
— the merging is trunk-based, the hiding is feature flags.

**And the CD requirement is what keeps `main` deployable *provably*.** Every merge deploys, so a broken
`main` is discovered in minutes rather than at the next release.

**Why B would work and slow them down.** GitHub Flow's day-long branches are fine at Beta's cadence and
become a queue at Alpha's.

</details>

---

## Q43

Which strategy should Team Gamma use, and why is Beta's not sufficient?

- A. Release branching — GitHub Flow has no branch representing the shipped version, so it cannot be
  patched while `main` moves on
- B. GitHub Flow — it is simpler
- C. Trunk-based — feature flags can hide the old version
- D. Git Flow — it has a `develop` branch

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** lines **148**, **197**.

```text
Release branching maintains long-lived release branches for supporting older versions while main
moves forward.
```

**"Patch the shipped release while next-release work continues" is the requirement**, and only a branch
that represents v2.1 can carry a v2.1.1 patch.

**Why C is the interesting wrong answer.** Feature flags hide **unreleased** work from users; they cannot
reconstruct a previous version's code for a customer still running it.

**And the cost is stated in the matrix** (line 200): high integration risk — which the drift job exists
to manage.

</details>

---

## Q44

How is "a fix on a shipped version must also reach `main`" satisfied?

- A. Cherry-pick the fix commit from the release branch to `main`
- B. Merge the release branch into `main`
- C. Rebase `main` onto the release branch
- D. Re-apply the fix by hand on `main`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** lines **170–173**.

**Cherry-pick moves exactly the fix**, leaving version-specific changes behind (Q37).

**Why B brings too much.** The release branch also carries version bumps, release configuration and
anything else specific to v2.1.

**Why D is what happens when the process is undocumented**, and why it fails: a hand-applied fix drifts
from the original, so the two versions behave subtly differently and the divergence is invisible.

</details>

---

## Q45

Which **two** stop long-lived branches drifting unnoticed? (Choose two.)

- A. Scheduled `git merge main` into the release branch
- B. A weekly job counting commits behind and opening an issue past a threshold
- C. Rebasing the release branch weekly
- D. Deleting release branches after 30 days
- E. Requiring PR review on the release branch

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-07.md`:** lines **207–211** and **216–243**.

**A keeps the gap small; B makes the gap visible when A does not happen.** Neither alone is sufficient —
a sync process nobody performs is exactly what needs detecting.

**Why C is the trap this paper builds twice** (Q20, Q26). The release branch is shared and tagged;
rebasing it orphans the tags and duplicates commits for everyone.

**Why D would destroy the ability to patch v2.1**, which is the only reason the branch exists.

</details>

---

## Q46

How should a mistaken direct push to `main` be recovered?

- A. Create a branch at the current tip, reset `main` back, push with `--force-with-lease`, then open a
  PR from the branch
- B. Revert the three commits on `main`
- C. Force-push an older `main`
- D. Leave the commits and enable branch protection

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** lines **345–356**.

**"Without losing the work" is the clause that orders the steps.** The branch must exist before the
reset, and the PR is what puts the work back through the process it skipped.

**Why B is genuinely defensible and not what is asked.** Reverting is safer on a branch others have
already pulled, and it leaves the mistaken commits in history — which fails "recover the work into a
PR" as a *process* correction.

**Why D is the preventive control, needed as well.** Branch protection (Challenge 01) stops the next
one; it does nothing about the three commits already on `main`.

</details>

---

## Q47

Five months in, Team Gamma reports that a tax bug fixed in v2.1.1 has reappeared in v2.2. The drift job
has been green every week, the cherry-pick step is in the runbook, and the fix commit is present on
`release/v2.1`.

What happened, and what should change?

- A. The cherry-pick to `main` was skipped for that fix — the drift job only measures *behind*, so a
  release branch **ahead** of `main` with un-propagated hotfixes raises no warning; report and act on
  the ahead count too
- B. The drift job's threshold is too high
- C. `main` was force-pushed
- D. The v2.2 release branch was cut from the wrong commit

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** lines **233–236** and **170–173**.

```bash
            AHEAD=$(git rev-list --count "origin/main..${branch}")
            BEHIND=$(git rev-list --count "${branch}..origin/main")
            echo "Branch ${branch}: ${BEHIND} commits behind, ${AHEAD} commits ahead"
            if [ "$BEHIND" -gt 50 ]; then
```

**Look at what the `if` tests: only `BEHIND`.** The job *prints* the ahead count at line 235 and never
acts on it — so a release branch carrying three un-cherry-picked hotfixes is reported as healthy every
Monday.

**"Green every week" is therefore consistent with the failure**, which is what makes this the realistic
version of the scenario's complaint at line 19.

**And the runbook being correct is not the same as the runbook being followed.** A step that depends on
memory fails eventually; the fix is to make the *omission* detectable.

**The change: alert on the ahead count as well**, or better, list the commits — `git log --oneline
origin/main..release/v2.1` names exactly which hotfixes have not reached `main`.

</details>

---

## Q48

A year on, all three teams ship on their own cadence, `main` has not been broken in months, and no fix
has regressed between releases.

Explain what each choice contributed, and what actually changed.

- A. Cadence chose each strategy, so branch lifetime matched release rhythm; feature flags let Alpha
  merge unfinished work safely; the PR gate gave Beta review without slowing a fortnightly train; the
  release branch let Gamma patch a shipped version; cherry-picking closed the regression path; and drift
  detection made the one long-lived branch's divergence visible — the process is now enforced by tooling
  rather than by coordination
- B. The teams agreed to coordinate better
- C. Everyone moved to one strategy
- D. Releases were made less frequent

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-07.md`:** lines **19**, **193–200**, **170–173**, **216–243**.

**Take the three complaints at line 19 in turn.**

*Integration pain every release* came from branches living longer than the gap between releases. Matching
lifetime to cadence removes it structurally: Alpha's branches close the same day, Beta's within the
fortnight, and Gamma's release branch is forward-integrated weekly.

*Broken builds when branches diverge for weeks* is the same cause seen from CI. A branch that merges
daily cannot diverge for weeks.

*Hotfixes that never make it back* was not a Git problem at all — it was a **missing step in a process**,
and the fix is to define it (cherry-pick) and then to detect its omission (Q47).

**What actually changed is that the strategy stopped being a matter of team culture.** The scenario
describes three teams that each chose reasonably in isolation; the problem was that nothing reconciled
them or made the consequences visible.

**The graded idea: a branching strategy is a decision about *how long code may live unmerged*, and the
right answer is a function of how often you ship.** Everything else in this challenge — flags, PR gates,
release branches, cherry-picks, drift jobs — exists to make one of those three answers survivable.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Release-branch hotfix procedure applied to GitHub Flow** | Q4 | GitHub Flow has no release branch |
| **One strategy standardised across all cadences** | Q25, Q48 | Cadence chooses. Line 193 |
| **Trunk-based chosen without CD or feature flags** | Q10, Q11, Q31, Q42 | Both are Required, not optional |
| **Feature flags expected to support old versions** | Q43 | Flags hide unreleased work, not prior releases |
| **Rebasing a shared release branch** | Q20, Q26, Q28, Q45 | Rewrites SHAs, orphans tags, duplicates commits |
| **Merging the release branch back to `main`** | Q5, Q25, Q44 | Cherry-pick the fix, not the branch |
| **Skipping the cherry-pick** | Q5, Q33, Q34, Q47 | The fix regresses in the next release |
| **Drift measured only as *behind*** | Q47 | Un-propagated hotfixes show as *ahead* |
| **`origin/main..branch` vs `branch..origin/main`** | Q13, Q22, Q39 | Right minus left. Behind predicts conflict |
| **Shallow checkout in the drift job** | Q15, Q34, Q41 | Every count returns 0. Green and wrong |
| **`--force` instead of `--force-with-lease`** | Q21, Q30, Q40 | No check against the remote moving |
| **Reset before creating the recovery branch** | Q6, Q21, Q30 | `reset --hard` discards it |
| **`--no-ff` and `--ff-only` confused** | Q7, Q8, Q28, Q38 | Force a merge commit vs refuse one |
| **Working *on* the release branch** | Q25 | Cut when feature-complete; fixes only |

---

# What to memorise

**In `challenge-07.md`:** lines **191–200**, **150–183**, **216–243**, **245–290**.

```text
CADENCE CHOOSES THE STRATEGY                        (line 193)
                        Trunk-based        GitHub Flow      Release branching
  release cadence       daily/weekly       BI-WEEKLY        monthly/quarterly
  team size             any                small-medium     medium-large
  deployment automation REQUIRED (CD)      recommended      optional
  feature flags         YES                optional         no
  SUPPORT OLD VERSIONS  no                 no               YES   <- the decider
  code review gate      optional (pair)    required (PR)    required (PR)
  rollback              FLAG OFF           revert commit    deploy prior release
  integration risk      low (always merged) medium          HIGH (long-lived)

Two rows answer most questions:  CADENCE, and SUPPORT OLD VERSIONS.
Trunk-based has two hard prerequisites: CD and feature flags. No flags -> not trunk-based.
```

```bash
# Trunk-based                                       (lines 32-49)
git checkout -b feature/x        # < 24 HOURS
git fetch origin && git rebase origin/main    # safe: branch not pushed yet
gh pr merge --squash --delete-branch          # one commit per change on main
# feature flag hides incomplete work:  {"RATE_LIMIT_ENABLED": {"enabled": false, ...}}

# Release branching                                 (lines 150-183)
git checkout main && git checkout -b release/v2.1 && git push origin release/v2.1
#   ... main continues with next-release work IN PARALLEL - that is the point
git checkout release/v2.1 && git checkout -b hotfix/...      # fix ON the release branch
git checkout main && git cherry-pick <sha>                   # <- THE STEP TEAMS SKIP
git tag -a v2.1.0 ... ; git tag -a v2.1.1 ...                # tags pin the shipped commits

# Forward-integration - MERGE, never rebase          (lines 207-211)
git checkout release/v2.1 && git merge main --no-edit && git push origin release/v2.1
```

```bash
# MERGE vs REBASE                                    (lines 245-290)
git merge feature --no-ff     # ALWAYS a merge commit. records that a branch existed
git rebase origin/main        # replays commits -> NEW SHAs -> never on a SHARED branch
git merge feature --ff-only   # REFUSES unless fast-forward. a check on your rebase
git rebase -i HEAD~5          # squash wip commits before the PR (unpushed branches only)

# Recover an accidental direct push                  (lines 345-356)
git branch feature/recover-direct-push    # PRESERVE FIRST - reset --hard discards otherwise
git reset --hard HEAD~3
git push origin main --force-with-lease   # refuses if the remote moved. --force does NOT
gh pr create --base main                  # put the work back through the process
```

```bash
# Drift detection                                    (lines 216-243)
on: schedule: - cron: '0 9 * * 1'     # min hour dom month dow -> Monday 09:00
  checkout with fetch-depth: 0        # REQUIRED - shallow makes every count ZERO

AHEAD=$(git rev-list --count "origin/main..${branch}")   # branch has, main lacks
BEHIND=$(git rev-list --count "${branch}..origin/main")  # main has, branch lacks
if [ "$BEHIND" -gt 50 ]; then  ::warning::  + gh issue create --label maintenance
#   BEHIND predicts merge conflicts.  AHEAD = hotfixes possibly not cherry-picked yet.
#   warn + raise an issue. do NOT fail the job, and do NOT auto-merge.
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 08 |
| 38–43 | Re-read the trap index and the decision matrix, then move on |
| 30–37 | Rewrite the decision matrix from memory, then retake |
| Below 30 | Redo Tasks 3, 5 and 6 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 07.

:::danger The two questions

**How often do they ship?** That picks the column. **Do they still support a shipped version?** That is
the only question release branching answers Yes to.

And one rule that never bends: **rebase rewrites history, so never rebase a branch anyone else has.**
Forward-integration onto a release branch is always a merge.

Contoso's teams each chose sensibly in isolation. Nothing reconciled them, and nothing made the
divergence visible.

:::
