---
sidebar_position: 2.5
toc_max_heading_level: 2
title: "Challenge 08: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 08 — AZ-400 exam questions

**48 questions** built only from what Challenge 08 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-08.md`**.

:::danger Read this before you start

Two vocabularies for one idea, and the exam swaps them constantly.

**GitHub** calls them **branch protection rules** (legacy) or **rulesets** (modern).
**Azure Repos** calls them **branch policies**, configured with `az repos policy <type> create`.

Learn them as pairs. "Dismiss stale reviews on push" ↔ "Reset code reviewer votes when there are new
changes". "Required status checks" ↔ "Build validation policy". Same intent, different product.

**And keep the two GitHub layers apart from each other.** A *ruleset* rule is
`{"type": "pull_request", "parameters": {...}}`; legacy protection is a flat object with
`required_pull_request_reviews`. The field names differ —
`dismiss_stale_reviews_on_push` in a ruleset, `dismiss_stale_reviews` in legacy.

**The single most-tested fact here: a required status check is a string match against the job's `name:`,
not the workflow's.** Get it wrong and the check sits pending forever and the PR can never merge.

**And CODEOWNERS is last-match-wins**, like `.gitignore` — the *most specific* pattern is the one you
write *last*.

The scenario at line 20 is two production incidents from unreviewed code, and a SQL injection that
shipped because nobody looked at the query changes.

:::

---

# Section A — Multiple choice

---

## Q1

Branch protection requires two approving reviews and the `ci/test` check. A developer pushes a new commit
after receiving both approvals. What happens?

- A. The PR can still merge — it already has two approvals
- B. The approvals are dismissed and two new reviews are needed
- C. Only one new approval is needed
- D. The PR is closed automatically

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-08.md`:** line **47**.

```json
        "dismiss_stale_reviews_on_push": true,
```

**An approval is a statement about specific code.** New commits mean the code the reviewers approved no
longer exists, so the approval no longer applies to anything.

**Why C is the answer that feels reasonable and is not offered by the platform.** GitHub dismisses **all**
approvals; there is no partial credit for "they saw most of it".

**And that is the correct behaviour for the scenario at line 20.** The SQL injection reached production
because nobody reviewed the query changes — a system that kept stale approvals would let exactly that
happen after an initial clean review.

</details>

---

## Q2

In Azure Repos, what does "Reset code reviewer votes when there are new changes" do?

- A. Removes all reviewers from the PR
- B. Resets every vote to "No vote", requiring re-approval
- C. Resets only the votes of reviewers whose files changed
- D. Moves the PR back to draft

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-08.md`:** lines **290** and **605**.

```bash
  --reset-on-source-push true \
```

**This is GitHub's `dismiss_stale_reviews` under a different name** — the pairing worth memorising.

**Note what "every vote" means in Azure Repos**, which has more vote types than GitHub: Approve, Approve
with suggestions, Wait for author and Reject all reset to No vote. **A Reject is cleared too**, which is
occasionally surprising.

**Why C is a genuinely appealing idea that neither platform implements.** Per-file vote tracking does not
exist; the vote applies to the pull request.

</details>

---

## Q3

Given these CODEOWNERS entries in order, who must review `/src/api/billing/invoice.ts`?

```text
*                    @contoso/platform-team
/src/api/            @contoso/backend-team
/src/api/billing/    @contoso/billing-team @sarah-lead
```

- A. `@contoso/platform-team` only — first match wins
- B. `@contoso/backend-team` only
- C. `@contoso/billing-team` and `@sarah-lead` — last match wins
- D. All three

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-08.md`:** lines **121–125** and **616**.

```text
CODEOWNERS uses a last-match-wins rule, similar to .gitignore.
```

**Last match wins, so the most specific pattern must be written last.** Reverse the file's order and the
`*` on line 121 would own everything.

**Why D is the mental model people bring from other tools.** Owners do not accumulate — the winning
pattern's owners **replace** the others, they do not join them.

**Which has a practical consequence worth noticing.** `/src/api/billing/` does **not** inherit
`@contoso/backend-team`; if you want both teams, you list both on that line — exactly as line 125 lists a
team and an individual.

</details>

---

## Q4

What is the primary purpose of a merge queue?

- A. To limit how many PRs can be open
- B. To merge PRs in creation order
- C. To batch-test approved PRs together against `main` before merging, preventing concurrent merges from
  breaking the build
- D. To resolve merge conflicts automatically

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-08.md`:** line **627**.

```text
A merge queue solves the "semantic conflict" problem where two PRs individually pass CI against main,
but when both are merged, they break each other.
```

**"Semantic conflict" is the term to know.** Git reports no textual conflict — one PR renames a function,
the other adds a caller, and both are green against the `main` they branched from.

**Strict status checks (Challenge 01's `strict: true`) solve this by forcing serial rebasing.** A merge
queue solves it by testing the **merged result** without making every author rebase by hand — which is
why it is the answer on a busy repository.

**And when a group fails, the queue bisects** to find the offending PR and ejects it (line 627), so one
bad change does not block everything behind it.

</details>

---

## Q5

A required `ci/build` check sits pending forever. What is the cause?

- A. The check context does not match the job's `name:`
- B. The runner is offline
- C. The PR is from a fork
- D. The workflow has no `permissions` block

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** lines **510–511** and **518**.

```text
**Fix**: The required status check name `ci/build` must match the `name:` field of the job, not the
workflow name.
```

```yaml
jobs:
  build:
    name: ci/build  # This must match the required status check context
```

**Note the three different names in play**: the job **key** (`build`), the job **`name:`**
(`ci/build`), and the workflow **name** (`CI Pipeline`, line 167). **Branch protection matches the job's
`name:`.**

**And "pending forever" is the signature.** A check that runs and fails shows red; a check that was never
reported shows **pending**, because the platform is still waiting for a context that will never arrive.

**Both fixes are valid** (lines 520–536): rename the job to match the rule, or PATCH the rule to match
the jobs.

</details>

---

## Q6

CODEOWNERS is not requesting reviews from `@contoso/billing-team`. What must be checked?

- A. The team has at least write access, and `require_code_owner_reviews` is enabled
- B. The file is at the repository root
- C. The team has more than one member
- D. Every owner has approved once before

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** lines **552–557** and **564**.

```text
**Fix**: The team must have at least write access to the repository, and code owner reviews must be
enabled
```

**Two independent causes producing one symptom.** No access means the entry is **silently ignored**; no
requirement means the review is requested but not enforced.

**Why B is a real rule stated wrongly** (line 549): `.github/CODEOWNERS`, `CODEOWNERS` or
`docs/CODEOWNERS` are all valid.

**Check access first.** A silently ignored entry looks identical to no entry at all, so it is the harder
of the two to spot.

</details>

---

## Q7

Which ruleset rule requires signed commits?

- A. `required_signatures`
- B. `non_fast_forward`
- C. `deletion`
- D. `pull_request`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** lines **64–72**.

```json
    { "type": "required_signatures" },
    { "type": "non_fast_forward" },
    { "type": "deletion" }
```

**Three parameterless rules, three different protections.** `required_signatures` proves **who** wrote a
commit cryptographically; `non_fast_forward` blocks force pushes; `deletion` blocks branch deletion.

**Signed commits matter for the scenario's audit question.** Branch protection proves a review happened;
a signature proves the commit came from the identity it claims — which is what an auditor asks after an
incident.

</details>

---

## Q8

What does `require_last_push_approval: true` enforce?

- A. Someone other than the last person to push must approve
- B. The last reviewer must approve again
- C. Approvals expire after the last push
- D. Only the author may push last

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** line **49**.

**It closes the self-approval loop.** Without it, a reviewer with write access can approve a PR, push a
change to it, and merge — so the final state has no independent eyes on it.

**Why C is the neighbouring setting.** `dismiss_stale_reviews_on_push` (line 47) invalidates approvals
after a push; `require_last_push_approval` requires the **new** approval to come from someone who did not
make that push. **Together they are much stronger than either alone.**

</details>

---

## Q9

What does `required_review_thread_resolution` require?

- A. All review conversations must be resolved before merging
- B. All reviewers must respond
- C. Threads are deleted on merge
- D. Comments must be replied to within 24 hours

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** line **50**, with the legacy equivalent at **107**.

```json
        "required_review_thread_resolution": true
```

```json
  "required_conversation_resolution": true
```

**Two names for the same rule** — `required_review_thread_resolution` in a ruleset,
`required_conversation_resolution` in legacy protection. **Expect either.**

**And it addresses a real failure mode.** A reviewer raises "this looks like it concatenates user input
into SQL", the author does not reply, someone else approves, and the thread merges unanswered — which is
recognisably the incident at line 20.

**The Azure Repos equivalent is `az repos policy comment-required create`** (line 315).

</details>

---

## Q10

In the Azure Repos approver-count policy, what does `--creator-vote-counts false` mean?

- A. The PR author's own approval does not count toward the minimum
- B. The author cannot vote at all
- C. The author's vote counts double
- D. Votes are anonymous

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** line **288**.

```bash
  --minimum-approver-count 2 \
  --creator-vote-counts false \
```

**Without this, an author could supply one of their own two required approvals** — which turns a
two-reviewer policy into a one-reviewer policy.

**It is the Azure Repos analogue of GitHub's self-approval protections** (Q8), and it is the setting
most often left at its permissive default.

**And `--allow-downvotes false` on the next line** means a single Reject does not hard-block the PR —
the policy counts approvals rather than treating any rejection as a veto.

</details>

---

## Q11

What does `--queue-on-source-update-only true` do on a build validation policy?

- A. The validation build runs only when the source branch changes, not when the target moves
- B. The build queues only once per PR
- C. Builds are queued nightly
- D. The build runs only on merge

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** line **309**.

**It avoids re-running validation every time somebody else merges to `main`.** On a busy repository that
would rebuild every open PR many times a day for changes the PR did not make.

**And the trade-off is the same one strict status checks make** (Challenge 01 Q2): you are choosing not
to re-validate against the newest `main`. The compensating control is `--valid-duration 720` on the next
line — the passing build **expires after 720 minutes**, so a stale result cannot merge a day later.

</details>

---

## Q12

What does the Azure Repos merge-strategy policy at Task 5 enforce?

- A. Squash only — no fast-forward, no rebase
- B. Merge commits only
- C. Rebase only
- D. Any strategy the author chooses

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** lines **327–330**.

```bash
  --allow-squash true \
  --allow-no-fast-forward false \
  --allow-rebase false \
  --allow-rebase-merge false \
```

**One `true` and three `false` — the policy is a whitelist.** Every strategy must be explicitly permitted.

**`--allow-no-fast-forward` is the confusing name**: it is Azure Repos's term for a **merge commit**
(a merge performed with no fast-forward). Setting it `false` blocks merge commits.

**The GitHub equivalent is the repository's merge-button settings** (Challenge 01, lines 304–306) —
`allow_squash_merge`, `allow_merge_commit`, `allow_rebase_merge`.

</details>

---

## Q13

Which Azure Repos policy has no direct GitHub branch protection equivalent?

- A. Work item linking
- B. Minimum approver count
- C. Build validation
- D. Comment resolution

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** lines **295–300**.

```bash
az repos policy work-item-linking create \
  --blocking true \
  --enabled true
```

**Azure Repos can *require* a linked work item as a merge condition.** GitHub has no equivalent built-in
rule — which is why Challenge 03 enforced it with a **workflow** (`core.setFailed`) plus a required
status check.

**That contrast is worth carrying into the exam.** When a question asks how to require traceability
before merge, the answer differs by platform: a **policy** in Azure Repos, a **check** in GitHub.

</details>

---

## Q14

What does `grouping_strategy: "ALLGREEN"` mean in the merge queue configuration?

- A. A batch merges only if every PR in the group passes
- B. Only green-labelled PRs are queued
- C. PRs are grouped by author
- D. The queue merges the first passing PR only

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** line **243**.

```json
      "grouping_strategy": "ALLGREEN",
      "max_entries_to_build": 5,
      "max_entries_to_merge": 5,
```

**The group is tested as a unit, so it succeeds or fails as a unit** — which is the whole point (Q4). If
the batch fails, the queue works out which entry broke it and removes that one.

**And `min_entries_to_merge_wait_minutes: 5`** (line 248) is the batching window: wait up to five minutes
to accumulate entries rather than testing every PR alone, which is where the efficiency comes from.

</details>

---

## Q15

Which trigger does the merge queue workflow use?

- A. `merge_group` with `types: [checks_requested]`
- B. `pull_request`
- C. `push`
- D. `workflow_dispatch`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** lines **261–263**.

```yaml
on:
  merge_group:
    types: [checks_requested]
```

**A distinct event, because a merge group is not a pull request.** It is a temporary branch containing
`main` plus the queued PRs, and CI must run against **that** to detect semantic conflicts.

**Which means a repository with a merge queue needs both triggers.** The `pull_request` workflow (line
169) validates the PR in isolation; the `merge_group` workflow validates the combination. **Omit the
second and the queue has nothing to wait for.**

</details>

---

## Q16

What does the PR size labeller do for a PR with 600 changed lines?

- A. Applies `size/XL` and comments asking for it to be split
- B. Blocks the PR
- C. Applies `size/L`
- D. Closes the PR

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** lines **376–379**.

```javascript
            } else {
              label = 'size/XL';
              comment = 'This PR has over 500 lines changed. Large PRs are difficult to review
              thoroughly and increase the risk of bugs. Please split this into smaller, focused PRs.';
            }
```

**Label and nudge, do not block** — and the reasoning is in the comment text itself: large PRs are
reviewed *worse*, which is the mechanism behind the incident at line 20.

**Why blocking would be wrong here.** Some large changes are legitimate — a generated file, a
dependency bump, a formatting sweep. A hard limit would be routed around; a visible label makes the
review cost obvious to everyone.

**Note it removes existing `size/` labels first** (lines 382–395), so a PR that shrinks is relabelled
rather than accumulating both.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** review protections does the ruleset's `pull_request` rule configure? (Choose three.)

- A. Two required approving reviews
- B. Stale reviews dismissed on push
- C. Code owner review required
- D. Signed commits required
- E. Force pushes blocked
- F. Branch deletion blocked

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-08.md`:** lines **46–50**.

```json
        "required_approving_review_count": 2,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": true,
        "require_last_push_approval": true,
        "required_review_thread_resolution": true
```

**Five parameters in that block, and D, E and F are separate rule *types*** (lines 65–72) rather than
parameters of this one.

**That structural difference is the ruleset model.** Some protections are parameters on a rule; others
are rules with no parameters at all. The exam tests whether you know which is which.

</details>

---

## Q18

Which **three** required status checks does the branch protection reference? (Choose three.)

- A. `ci/build`
- B. `ci/test`
- C. `security/scan`
- D. `ci/lint`
- E. `ci/deploy`
- F. `merge_group`

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-08.md`:** lines **57–61** and **174**, **186**, **206**.

```json
          { "context": "ci/build" },
          { "context": "ci/test" },
          { "context": "security/scan" }
```

**And every one of them matches a job `name:` in the workflow** — `name: ci/build` at line 174,
`name: ci/test` at line 186, `name: security/scan` at line 206.

**That correspondence is the thing to verify in any real repository** (Q5). Three contexts, three job
names, spelled identically — including the slash, which is part of the name and not a path.

</details>

---

## Q19

Which **three** does the `security/scan` job run? (Choose three.)

- A. CodeQL initialisation
- B. CodeQL analysis
- C. TruffleHog secret scanning with `--only-verified`
- D. Dependabot
- E. A coverage check
- F. Integration tests

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-08.md`:** lines **210–218**.

```yaml
      - uses: github/codeql-action/init@v3
      - uses: github/codeql-action/analyze@v3
      - uses: trufflesecurity/trufflehog@main
        with:
          extra_args: --only-verified
```

**Two scanners for two different risks** — CodeQL finds flaws in code you wrote (the SQL injection at
line 20), TruffleHog finds credentials.

**`--only-verified` is the flag that makes it usable.** It reports secrets TruffleHog has **confirmed are
live** by testing them against the provider, which removes most false positives — the difference between
a scanner people act on and one they mute.

**Why E is in the `ci/test` job** (lines 197–203) and F is in the merge-queue workflow (line 274).

</details>

---

## Q20

Which **two** Azure Repos policies correspond to GitHub required status checks and stale-review
dismissal? (Choose two.)

- A. `az repos policy build create`
- B. `az repos policy approver-count create --reset-on-source-push true`
- C. `az repos policy work-item-linking create`
- D. `az repos policy comment-required create`
- E. `az repos policy merge-strategy create`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-08.md`:** lines **303–312** and **283–292**.

**Build validation ↔ required status checks; reset-on-source-push ↔ dismiss stale reviews.**

**Why C is the one with no GitHub equivalent** (Q13), and why D maps to
`required_conversation_resolution` (Q9) — a third pairing worth knowing, just not the one asked for.

</details>

---

## Q21

Which **two** are true of merge queues? (Choose two.)

- A. They test batched PRs against `main` together before merging
- B. They need a workflow triggered on `merge_group`
- C. They replace required status checks
- D. They merge PRs in creation order regardless of result
- E. They resolve textual conflicts automatically

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-08.md`:** lines **238–250** and **261–263**.

**Why C is the misconception worth killing.** The queue does not remove the PR-level checks — it **adds**
a second validation of the combined result. A repository with a merge queue runs CI twice: once on the
PR, once on the group.

**Why E confuses semantic conflicts with textual ones.** A merge queue catches changes that are
individually valid and jointly broken; Git already reports textual conflicts, and nothing here resolves
them.

</details>

---

## Q22

Which **two** make the PR size labeller idempotent? (Choose two.)

- A. Listing existing labels and removing any starting with `size/`
- B. Adding the newly computed label afterwards
- C. Running only on `opened`
- D. Using `fetch-depth: 0`
- E. Commenting on every run

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-08.md`:** lines **382–403**.

```javascript
            const existingLabels = (await github.rest.issues.listLabelsOnIssue({...}))
              .data.map(l => l.name).filter(n => n.startsWith('size/'));

            for (const l of existingLabels) { await github.rest.issues.removeLabel({...}); }

            await github.rest.issues.addLabels({ labels: [label] });
```

**Remove then add, so a PR that shrinks from XL to M ends up with one label rather than two.**

**And the `size/` prefix is what makes the removal safe** — it targets only this workflow's labels and
leaves `bug`, `enhancement` and the rest untouched. The same namespacing idea as Challenge 02's
`priority/` labels.

**Why C would break it.** The workflow runs on `synchronize` too (line 344), which is precisely when the
size changes.

</details>

---

## Q23

Which **two** does the hotfix PR template capture that the standard template does not? (Choose two.)

- A. A severity level
- B. An incident link and root cause
- C. A type-of-change checklist
- D. Testing notes
- E. Screenshots

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-08.md`:** lines **469–475**.

```text
**Severity**: [ ] P1 - Service down [ ] P2 - Major degradation [ ] P3 - Minor issue

**Incident link**: <!-- Link to incident or on-call ticket -->

## Root cause
```

**A hotfix template asks different questions because it is written under time pressure**, and the answers
matter for the post-incident review rather than for the code review.

**And the risk assessment at lines 481–485 is the reviewer's checklist**: is the change minimal, is there
a rollback plan, have the dashboards been checked. **That is what a reviewer needs at 2 AM**, and it is
what the general template's style-guideline checkbox cannot provide.

**Note it lives at `.github/PULL_REQUEST_TEMPLATE/hotfix.md`** (line 466) — the plural directory holds
multiple named templates, while `.github/pull_request_template.md` (line 421) is the default.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must prevent direct pushes to `main`, guarantee that code is reviewed by the right
people, run automated build, test and security checks before merge, and stop concurrent merges breaking
the build.

---

## Q24

**Proposed solution:** Create a ruleset on `main` requiring two approving reviews with stale dismissal,
code owner review, last-push approval and thread resolution; required status checks for `ci/build`,
`ci/test` and `security/scan` with strict policy; signed commits; and no force pushes or deletions. Add a
CODEOWNERS file ordered general to specific, with the owning teams granted write access. Name the CI jobs
to match the check contexts exactly. Enable a merge queue with an `ALLGREEN` grouping strategy and a
`merge_group` workflow. Mirror the policies in Azure Repos with approver-count, build validation,
comment-required and merge-strategy policies.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-08.md`:** lines **31–82**, **117–150**, **174–218**, **227–275**, **283–332**.

| Requirement | Mechanism |
|---|---|
| No direct pushes | `pull_request` rule with required reviews |
| Reviewed by the right people | CODEOWNERS + `require_code_owner_review` + team write access |
| Automated gates | Three required status checks, names matched to jobs |
| Concurrent merges do not break `main` | Merge queue with `merge_group` CI |
| Same guarantees in Azure Repos | Equivalent branch policies |

**"Ordered general to specific" is the clause that makes CODEOWNERS work** (Q3), and "named to match
exactly" is the clause that makes the checks fire (Q5). **Both are the kind of detail an exam answer
either has or does not.**

</details>

---

## Q25

**Proposed solution:** Enable branch protection requiring one review. Add a CODEOWNERS file with the
specific paths listed first and the `*` catch-all last. Require status checks named after the workflow
files. Rely on strict status checks instead of a merge queue. Skip Azure Repos, since the team mostly
uses GitHub.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures.**

**One review with no last-push-approval requirement** lets a reviewer approve, push, and merge their own
change — the self-approval loop `require_last_push_approval` exists to close (Q8).

**CODEOWNERS ordered specific-first is inverted.** Last match wins (Q3), so the `*` line at the bottom
would claim **every** file and the specific team entries would never apply — including
`/src/auth/ @contoso/security-team`, which is exactly the path the SQL injection would have touched.

**Checks named after workflow files never report.** The context matches the job's `name:` (Q5), so all
three checks sit pending and no PR can merge at all.

**And "skip Azure Repos" ignores half the requirement.** The VP mandated protection **across both**
platforms (line 20).

</details>

---

## Q26

**Proposed solution:** Create the ruleset with two reviews, stale dismissal, code owner review, last-push
approval, thread resolution, the three required checks, signed commits, and no force pushes or deletions.
Order CODEOWNERS general to specific with teams granted write access. Match job names to check contexts.
Enable a merge queue with `ALLGREEN`. Mirror everything in Azure Repos. Because the merge queue already
validates the combined result, drop the `pull_request`-triggered CI workflow and run checks only on
`merge_group`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**The three required status checks are evaluated on the pull request**, and with no `pull_request`
trigger nothing ever reports them. Every PR sits pending forever, unable to enter the queue in the first
place — **the same symptom as Break scenario 1, caused deliberately.**

**And it inverts where feedback belongs.** The PR-level run tells an author within minutes that their own
change is broken. The merge-group run happens **after approval**, at the front of the queue, where a
failure ejects the PR and wastes everyone's place in line.

**The two runs answer different questions and both are needed** (Q21): *is this change correct on its
own*, and *is it compatible with what else is about to merge*. Line 169 and line 262 are separate
triggers for exactly that reason.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — GitHub protection

| # | Statement | Answer |
|---|---|---|
| 1 | `dismiss_stale_reviews_on_push` invalidates all approvals after a push |  |
| 2 | `require_last_push_approval` requires approval from someone who did not push last |  |
| 3 | A required check matches the workflow's name |  |
| 4 | `required_signatures` blocks force pushes |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `dismiss_stale_reviews_on_push` invalidates all approvals after a push | **Yes** |
| 2 | `require_last_push_approval` requires approval from someone who did not push last | **Yes** |
| 3 | A required check matches the workflow's name | **No** |
| 4 | `required_signatures` blocks force pushes | **No** |

**In `challenge-08.md`:** lines **47**, **49**, **518**, **65–68**.

Row 3 is the highest-yield fact in the challenge. Row 4 confuses two parameterless rules —
`non_fast_forward` blocks force pushes.

</details>

---

## Q28 — CODEOWNERS

| # | Statement | Answer |
|---|---|---|
| 1 | The last matching pattern wins |  |
| 2 | Owners from all matching patterns are combined |  |
| 3 | A team without repository access is silently ignored |  |
| 4 | Valid locations include `docs/CODEOWNERS` |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | The last matching pattern wins | **Yes** |
| 2 | Owners from all matching patterns are combined | **No** |
| 3 | A team without repository access is silently ignored | **Yes** |
| 4 | Valid locations include `docs/CODEOWNERS` | **Yes** |

**In `challenge-08.md`:** lines **616**, **125**, **564**, **549**.

Row 2 is the misconception that makes people order the file wrongly (Q25).

Row 3 is the failure with no error message — check access before syntax.

</details>

---

## Q29 — Azure Repos policies

| # | Statement | Answer |
|---|---|---|
| 1 | `--creator-vote-counts false` excludes the author's own approval |  |
| 2 | `--reset-on-source-push` mirrors dismiss-stale-reviews |  |
| 3 | Azure Repos can require a linked work item to merge |  |
| 4 | GitHub branch protection has a built-in work item requirement |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `--creator-vote-counts false` excludes the author's own approval | **Yes** |
| 2 | `--reset-on-source-push` mirrors dismiss-stale-reviews | **Yes** |
| 3 | Azure Repos can require a linked work item to merge | **Yes** |
| 4 | GitHub branch protection has a built-in work item requirement | **No** |

**In `challenge-08.md`:** lines **288**, **290**, **295–300**, and Q13.

Rows 3 and 4 are the genuine capability gap between the platforms, and the exam uses it.

</details>

---

## Q30 — merge queue

| # | Statement | Answer |
|---|---|---|
| 1 | A merge queue prevents semantic conflicts between concurrent PRs |  |
| 2 | It requires a `merge_group`-triggered workflow |  |
| 3 | It replaces PR-level required status checks |  |
| 4 | `ALLGREEN` means the batch merges only if every entry passes |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A merge queue prevents semantic conflicts between concurrent PRs | **Yes** |
| 2 | It requires a `merge_group`-triggered workflow | **Yes** |
| 3 | It replaces PR-level required status checks | **No** |
| 4 | `ALLGREEN` means the batch merges only if every entry passes | **Yes** |

**In `challenge-08.md`:** lines **627**, **262**, **238–250**, **243**.

Row 3 is Q26's failure stated as a fact. **Two validations, two questions.**

</details>

---

# Section E — Drag and drop

---

## Q31

Match each GitHub protection to its Azure Repos equivalent.

| GitHub | Azure Repos |
|---|---|
| Required approving reviews |  |
| Dismiss stale reviews on push |  |
| Required status checks |  |
| Required conversation resolution |  |
| Merge button settings |  |
| *(no equivalent)* |  |

**Options:** `az repos policy approver-count create` · `az repos policy build create` · `az repos policy comment-required create` · `az repos policy merge-strategy create` · `az repos policy work-item-linking create` · `--reset-on-source-push true`

<details>
<summary>Show answer</summary>

| GitHub | Azure Repos |
|---|---|
| Required approving reviews | **`az repos policy approver-count create`** |
| Dismiss stale reviews on push | **`--reset-on-source-push true`** |
| Required status checks | **`az repos policy build create`** |
| Required conversation resolution | **`az repos policy comment-required create`** |
| Merge button settings | **`az repos policy merge-strategy create`** |
| *(no equivalent)* | **`az repos policy work-item-linking create`** |

**In `challenge-08.md`:** lines **283–332**.

**Five pairs and one gap.** Requiring a linked work item is an Azure Repos policy with no GitHub
counterpart — on GitHub you build it as a workflow check (Challenge 03).

</details>

---

## Q32

Match each ruleset element to what it does.

| Element | Does |
|---|---|
| `required_approving_review_count` |  |
| `dismiss_stale_reviews_on_push` |  |
| `require_last_push_approval` |  |
| `required_review_thread_resolution` |  |
| `required_signatures` |  |
| `non_fast_forward` |  |
| `deletion` |  |

**Options:** Blocks branch deletion · Blocks force pushes · Blocks merge on unresolved conversations · Invalidates approvals when new commits land · Requires signed commits · Sets the number of approvals · Stops the last pusher self-approving

<details>
<summary>Show answer</summary>

| Element | Does |
|---|---|
| `required_approving_review_count` | **Sets the number of approvals** |
| `dismiss_stale_reviews_on_push` | **Invalidates approvals when new commits land** |
| `require_last_push_approval` | **Stops the last pusher self-approving** |
| `required_review_thread_resolution` | **Blocks merge on unresolved conversations** |
| `required_signatures` | **Requires signed commits** |
| `non_fast_forward` | **Blocks force pushes** |
| `deletion` | **Blocks branch deletion** |

**In `challenge-08.md`:** lines **46–72**.

**The first four are *parameters* of the `pull_request` rule; the last three are *rule types* of their
own.** That structural split is what the exam checks when it asks where a setting lives.

</details>

---

## Q33

Arrange the steps to make a required status check actually gate merges.

**Items:** Add the context to the required status checks · Set the job's `name:` to the context value ·
Open a PR and confirm the check reports · Write the CI workflow triggered on `pull_request`

<details>
<summary>Show answer</summary>

### Answer

1. Write the CI workflow triggered on `pull_request` — lines **166–170**
2. Set the job's `name:` to the context value — lines **173–174**
3. Add the context to the required status checks — lines **57–61**
4. Open a PR and confirm the check reports — lines **504–508**

**Step 3 must come after steps 1 and 2, and that ordering is the answer.** Add a required context before
anything reports it and every open PR blocks immediately on a check that does not exist — which is how
teams end up deleting the rule to unstick themselves.

**And step 4 is not ceremony.** The only proof that the context string matches is a PR where the check
turns green; `gh run list` plus the protection query (lines 505–508) is the diagnostic pair when it does
not.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| A required check is pending forever |  |
| CODEOWNERS assigns the wrong team |  |
| CODEOWNERS assigns nobody |  |
| Two green PRs merge and break `main` |  |
| A reviewer approves, pushes, and merges alone |  |
| A PR merges with an unanswered security comment |  |

**Options:** Context does not match the job's `name:` · No merge queue and no strict checks · Patterns ordered specific-first — last match wins · `require_last_push_approval` not set · The team lacks repository access · Thread resolution not required

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| A required check is pending forever | **Context does not match the job's `name:`** |
| CODEOWNERS assigns the wrong team | **Patterns ordered specific-first — last match wins** |
| CODEOWNERS assigns nobody | **The team lacks repository access** |
| Two green PRs merge and break `main` | **No merge queue and no strict checks** |
| A reviewer approves, pushes, and merges alone | **`require_last_push_approval` not set** |
| A PR merges with an unanswered security comment | **Thread resolution not required** |

**In `challenge-08.md`:** lines **518**, **616**, **564**, **627**, **49**, **50**.

**Rows 2 and 3 produce different symptoms from the same file** — one assigns the wrong owners, the other
assigns none — and the diagnosis differs accordingly.

</details>

---

## Q35

Match each control to what it prevents.

| Control | Prevents |
|---|---|
| Two required reviews |  |
| Code owner review |  |
| `security/scan` as a required check |  |
| Merge queue |  |
| PR size labelling |  |
| PR template |  |

**Options:** A known vulnerability class shipping · Concurrent merges breaking the build · Nothing — it makes review cost visible · Nothing — it prompts the author · The wrong people reviewing security-sensitive paths · Unreviewed code reaching `main`

<details>
<summary>Show answer</summary>

| Control | Prevents |
|---|---|
| Two required reviews | **Unreviewed code reaching `main`** |
| Code owner review | **The wrong people reviewing security-sensitive paths** |
| `security/scan` as a required check | **A known vulnerability class shipping** |
| Merge queue | **Concurrent merges breaking the build** |
| PR size labelling | **Nothing — it makes review cost visible** |
| PR template | **Nothing — it prompts the author** |

**In `challenge-08.md`:** lines **46**, **48** with **140**, **60** with **205–214**, **240–250**,
**376–379**, **420–461**.

**Two of the six prevent nothing**, and both are the ones people name when asked how to stop large
unreviewed changes. **A label is information; a required check is a gate.**

</details>

---

# Section F — Hot area

---

## Q36

```json
{
  "type": "[BLANK 1]",
  "parameters": {
    "required_approving_review_count": 2,
    "[BLANK 2]": true,
    "require_code_owner_review": true,
    "[BLANK 3]": true
  }
}
```

Requirement: two approvals, invalidated by new commits, from code owners, and the person who pushed last
cannot be the approver.

- **BLANK 1:** `pull_request` / `required_reviews` / `branch_protection` / `approval`
- **BLANK 2:** `dismiss_stale_reviews_on_push` / `dismiss_stale_reviews` / `reset_on_push` /
  `stale_reviews`
- **BLANK 3:** `require_last_push_approval` / `require_signed_commits` / `require_linear_history` /
  `block_self_merge`

<details>
<summary>Show answer</summary>

### Answer: `pull_request`, `dismiss_stale_reviews_on_push`, `require_last_push_approval`

**In `challenge-08.md`:** lines **44–49**.

**BLANK 2 is the ruleset-versus-legacy trap.** A ruleset uses `dismiss_stale_reviews_on_push` (line 47);
legacy protection uses `dismiss_stale_reviews` (line 98). **Both appear in this challenge, three dozen
lines apart.**

</details>

---

## Q37

```yaml
jobs:
  build:
    [BLANK 1]: ci/build
    runs-on: ubuntu-latest
```

```json
"required_status_checks": [ { "[BLANK 2]": "ci/build" } ]
```

- **BLANK 1:** `name` / `id` / `label` / `context`
- **BLANK 2:** `context` / `name` / `job` / `check`

<details>
<summary>Show answer</summary>

### Answer: `name`, `context`

**In `challenge-08.md`:** lines **174** and **58**.

**The two sides use different keywords for the same string.** The workflow calls it `name:`; the rule
calls it `context`. **They must be identical in value** even though the keys differ.

**And the job *key* (`build`) is irrelevant to the match** — it is the `name:` that becomes the reported
context (Q5).

</details>

---

## Q38

```bash
az repos policy approver-count create \
  --branch main \
  --minimum-approver-count [BLANK 1] \
  --creator-vote-counts [BLANK 2] \
  --reset-on-source-push [BLANK 3] \
  --blocking true --enabled true
```

Requirement: two independent approvals that do not survive new commits.

- **BLANK 1:** `2` / `1` / `0` / `3`
- **BLANK 2:** `false` / `true`
- **BLANK 3:** `true` / `false`

<details>
<summary>Show answer</summary>

### Answer: `2`, `false`, `true`

**In `challenge-08.md`:** lines **286–290**.

**"Independent" is what `--creator-vote-counts false` delivers** (Q10) — without it, the author supplies
one of the two.

**And `--blocking true` on line 291 is the one that turns the policy into a gate.** A non-blocking policy
is advisory: the vote is recorded, and the PR can complete anyway.

</details>

---

## Q39

```bash
az repos policy build create \
  --build-definition-id 42 \
  --queue-on-source-update-only [BLANK 1] \
  --valid-duration [BLANK 2] \
  --blocking true --enabled true
```

Requirement: avoid rebuilding every open PR when `main` moves, but do not let a passing build be trusted
for more than twelve hours.

- **BLANK 1:** `true` / `false`
- **BLANK 2:** `720` / `12` / `43200` / `1`

<details>
<summary>Show answer</summary>

### Answer: `true`, `720`

**In `challenge-08.md`:** lines **309–310**.

**`--valid-duration` is in *minutes*** — 720 minutes is twelve hours. Reading it as hours is the trap,
and `12` would expire the build after twelve minutes.

**The two settings are a deliberate pair** (Q11): one reduces rebuild churn, the other bounds how stale a
result may be. Set the first without the second and a build from last week can merge today.

</details>

---

## Q40

```json
{
  "type": "merge_queue",
  "parameters": {
    "grouping_strategy": "[BLANK 1]",
    "merge_method": "[BLANK 2]",
    "min_entries_to_merge_wait_minutes": 5
  }
}
```

- **BLANK 1:** `ALLGREEN` / `HEADGREEN` / `SEQUENTIAL` / `ANY`
- **BLANK 2:** `squash` / `merge` / `rebase` / `fast-forward`

<details>
<summary>Show answer</summary>

### Answer: `ALLGREEN`, `squash`

**In `challenge-08.md`:** lines **243** and **246**.

**`ALLGREEN` means the whole group must pass** (Q14). The queue then merges the batch, or bisects to find
the entry that broke it.

**And `merge_method: "squash"` keeps `main` at one commit per PR**, matching the linear-history habit
from Challenge 07 — the queue performs the merge, so the method is configured here rather than on the
merge button.

</details>

---

## Q41

```yaml
on:
  [BLANK 1]:
    types: [checks_requested]

jobs:
  ci:
    steps:
      - run: npm test
      - run: npm run test:integration
```

- **BLANK 1:** `merge_group` / `pull_request` / `push` / `workflow_run`

<details>
<summary>Show answer</summary>

### Answer: `merge_group`

**In `challenge-08.md`:** lines **261–263**.

**A merge group is its own event because it is its own ref** — a temporary branch of `main` plus the
queued PRs.

**And note this workflow runs the *integration* tests** (line 274) that the PR workflow does not. **That
is the design**: fast checks on every PR, the slower combined suite once at the front of the queue.

</details>

---

# Section G — Case study

## Case study: Contoso pull request enforcement

### Background

Contoso Ltd had **two production incidents last month caused by unreviewed code**. Junior developers are
**pushing directly to `main`**, bypassing review. A **SQL injection vulnerability reached production**
because nobody reviewed the database query changes. The VP of Engineering has mandated that all code go
through **pull request review with automated checks** before merging, **across both GitHub and Azure
Repos**.

### Requirements

**Review**

- No change may reach `main` without independent approval
- Security-sensitive paths must always be reviewed by the security team
- An approval must not survive a subsequent code change
- Unresolved review conversations must block the merge

**Automation**

- Build, test and security scans must pass before merge, enforced by the platform
- Concurrent merges must not be able to break `main`

**Parity**

- Azure Repos must enforce equivalent controls

---

## Q42

How is "no change without independent approval" enforced on GitHub?

- A. A `pull_request` rule with two required approvals, stale dismissal and `require_last_push_approval`
- B. Two required approvals
- C. A PR template with a review checklist
- D. CODEOWNERS

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** lines **46–49**.

**The word "independent" is doing the work.** Two approvals alone can be satisfied by a reviewer who
approves, then pushes a change, then merges — so `require_last_push_approval` is what makes the approval
independent of the final state.

**And stale dismissal is what ties the approval to the code**, so "approved" always refers to what is
about to merge (Q1).

**Why C prompts and enforces nothing** (Q35).

</details>

---

## Q43

How are security-sensitive paths guaranteed a security team review?

- A. CODEOWNERS entries for `/src/auth/`, `/src/crypto/` and `**/security*.yml`, plus
  `require_code_owner_review`, with the team granted write access
- B. CODEOWNERS entries alone
- C. A required status check
- D. A label

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** lines **139–142**, **48**, **564**.

```text
/src/auth/ @contoso/security-team
/src/crypto/ @contoso/security-team
**/security*.yml @contoso/security-team
```

**Three parts, and each fails silently on its own.** The entries route the request;
`require_code_owner_review` makes it mandatory; the write access makes the entry resolve at all.

**And this is the control that would have caught the incident at line 20** — a change to database query
code under an owned path cannot merge without the owning team looking at it.

**Note the glob on the third line.** `**/security*.yml` matches at any depth, which is how you own a
*kind* of file rather than a directory.

</details>

---

## Q44

Why must the CI job names match the required check contexts exactly?

- A. Branch protection waits for a context string; an unmatched name means the check never reports and the
  PR blocks forever
- B. It is a naming convention
- C. GitHub derives the job from the context
- D. It affects run ordering

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** lines **510–518**.

**"Blocks forever" is the specific consequence**, and it is worse than a failure: a red check tells you
what to fix, a pending one tells you nothing.

**And the failure mode when someone "fixes" it is the dangerous part.** Under pressure, an administrator
removes the stuck context from the rule — and the repository silently loses a gate. **That is how a
security scan stops being required** without anyone deciding it should.

</details>

---

## Q45

Which **two** stop concurrent merges breaking `main`? (Choose two.)

- A. A merge queue with `ALLGREEN` grouping
- B. A `merge_group`-triggered CI workflow
- C. Required approving reviews
- D. CODEOWNERS
- E. PR size labels

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-08.md`:** lines **238–250** and **261–263**.

**The queue orchestrates; the workflow is what actually tests the combination.** Enable the rule without
the `merge_group` workflow and the queue waits for checks nobody runs.

**Why C and D address a different failure.** Reviews and ownership catch *bad* changes; semantic conflicts
are two *good* changes that are incompatible (Q4).

</details>

---

## Q46

How is parity achieved in Azure Repos?

- A. Approver-count with reset-on-source-push and creator-vote-counts false, build validation,
  comment-required, and merge-strategy policies, all blocking
- B. A single approver-count policy
- C. A pipeline that checks the rules
- D. Documentation telling the team to follow the same process

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** lines **283–332**.

**Four policies mapping onto the four GitHub protections** (Q31), and every one of them carries
`--blocking true`.

**That flag is the parity requirement in one word.** A non-blocking policy in Azure Repos is
**advisory** — it evaluates, it displays, and the PR can complete regardless. **A non-blocking policy is
the Azure Repos equivalent of a check that is not required.**

</details>

---

## Q47

Four months in, the security team notices that PRs touching `/src/auth/` have merged without their
review for several weeks. CODEOWNERS still lists them, `require_code_owner_review` is still enabled, and
other paths are still routing correctly. The organisation recently restructured its teams.

What happened, and what is the fix?

- A. The security team was renamed or lost repository access, so its CODEOWNERS entry resolves to nothing
  and is silently skipped — restore access, or update the entry to the new team slug
- B. The ruleset was deleted
- C. `require_code_owner_review` was disabled
- D. The PRs were merged by administrators

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** lines **552–557** and **564**.

```bash
gh api orgs/contoso/teams/billing-team/repos/contoso/platform-api
```

**"Other paths still route correctly" is the detail that localises it.** The file is being parsed and the
mechanism is working — one entry has stopped resolving.

**A CODEOWNERS entry naming a non-existent or access-less team is ignored without any error.** There is
no annotation on the pull request, no warning in the checks, and no entry in the log. The path simply
falls through to whatever earlier pattern matched — here, `*` and `@contoso/platform-team` on line 121.

**Which makes the symptom "reviewed by the wrong people", not "unreviewed"** — the PR still had an
approval, so nothing looked broken.

**The durable lesson: a team rename is a change to every CODEOWNERS file that mentions it.** Validate the
file after any org restructure — `github-codeowners validate` (line 158) catches syntax, and the team
lookup at line 553 catches resolution.

</details>

---

## Q48

A year on, no change reaches `main` unreviewed, the security team sees every auth change, and `main` has
not been broken by a merge collision.

Explain what each control contributed, and what actually changed.

- A. The `pull_request` rule made review structurally unavoidable; stale dismissal and last-push approval
  made the approval mean the final code; CODEOWNERS with code-owner review put the right eyes on the
  right paths; three required checks named to match their jobs made build, test and security scanning
  conditions of merge rather than suggestions; and the merge queue tested combinations no single PR could
  test — every one enforced by the platform rather than by discipline
- B. The developers became more careful after the incidents
- C. More senior reviewers were added to the team
- D. Deployment frequency was reduced

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-08.md`:** lines **44–72**, **139–142**, **57–61**, **238–263**.

**Take the two incidents at line 20 in turn.**

*Junior developers pushing directly to `main`* was not a training problem — it was possible. The
`pull_request` rule removes the path entirely, and `enforce_admins` (line 95) removes the exception list.

*A SQL injection reviewed by nobody* had two causes and both are now closed: **CODEOWNERS** guarantees the
security team is asked for `/src/auth/` and `/src/crypto/`, and **`security/scan`** as a required check
means CodeQL's opinion is a merge condition rather than an annotation someone can scroll past.

**And the merge queue closes a failure the incidents had not yet produced** but that two protected,
reviewed, individually-green PRs will eventually cause.

**What actually changed is not care.** The scenario does not describe reckless engineers; it describes a
repository where **the correct behaviour was optional** — review was available, scanning was available,
and neither was required.

**The graded idea, and it recurs across this whole certification: a control is only a control when
something refuses to proceed.** CODEOWNERS without `require_code_owner_review` requests. A scan without a
required context reports. A policy without `--blocking` advises. **The gate is the part that says no.**

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Check context matched to the workflow name** | Q5, Q25, Q27, Q37, Q44 | It matches the job's `name:`. Pending forever otherwise |
| **CODEOWNERS ordered specific-first** | Q3, Q25, Q28, Q34 | Last match wins. General first, specific last |
| **CODEOWNERS owners assumed to accumulate** | Q3, Q28 | The winning pattern replaces, it does not add |
| **CODEOWNERS entry with no team access** | Q6, Q28, Q34, Q47 | Silently ignored. No error anywhere |
| **`dismiss_stale_reviews` vs `dismiss_stale_reviews_on_push`** | Q36 | Legacy vs ruleset field names |
| **Two approvals assumed to be independent** | Q8, Q42 | Add `require_last_push_approval` |
| **`--creator-vote-counts` left at default** | Q10, Q38 | The author supplies one of the approvals |
| **`--valid-duration` read as hours** | Q39 | Minutes. 720 = twelve hours |
| **Non-blocking Azure Repos policy** | Q38, Q46 | Advisory only. The PR completes anyway |
| **Merge queue without a `merge_group` workflow** | Q15, Q21, Q45 | The queue waits for checks nobody runs |
| **Merge queue assumed to replace PR checks** | Q21, Q26 | Two validations, two questions |
| **`required_signatures` confused with `non_fast_forward`** | Q7, Q27 | Signing vs force-push blocking |
| **Size labels or templates treated as gates** | Q16, Q35 | Information, not enforcement |
| **Work item requirement assumed available on GitHub** | Q13, Q29 | Azure Repos policy; a workflow check on GitHub |

---

# What to memorise

**In `challenge-08.md`:** lines **31–110**, **283–332**, **227–275**.

```text
TWO VOCABULARIES - learn them as PAIRS
  GitHub (rulesets / branch protection)        Azure Repos (branch policies)
  required approving reviews                   az repos policy approver-count create
  dismiss stale reviews on push                  --reset-on-source-push true
  (author self-approval)                         --creator-vote-counts false
  required status checks                       az repos policy build create
  required conversation resolution             az repos policy comment-required create
  merge button settings                        az repos policy merge-strategy create
  -- no equivalent --                          az repos policy work-item-linking create
  every Azure policy needs --blocking true, or it is ADVISORY ONLY
```

```json
// RULESET rule types                              (lines 42-72)
{"type":"pull_request","parameters":{
   "required_approving_review_count": 2,
   "dismiss_stale_reviews_on_push": true,      // legacy name: dismiss_stale_reviews
   "require_code_owner_review": true,
   "require_last_push_approval": true,         // the last pusher cannot be the approver
   "required_review_thread_resolution": true   // legacy name: required_conversation_resolution
}}
{"type":"required_status_checks","parameters":{
   "strict_status_checks_policy": true,
   "required_status_checks":[{"context":"ci/build"},{"context":"ci/test"},{"context":"security/scan"}]}}
{"type":"required_signatures"}   {"type":"non_fast_forward"}   {"type":"deletion"}
//  first four are PARAMETERS of one rule. last three are RULE TYPES.
```

```yaml
# THE MATCH THAT BREAKS EVERYTHING                  (lines 173-174, 518)
jobs:
  build:              # job KEY - irrelevant to the match
    name: ci/build    # <- THIS is the reported context
# rule:  {"context": "ci/build"}
# mismatch = check PENDING FOREVER (not red). PR can never merge.
# fix either side:  rename the job, or PATCH contexts to ["build","test","security-scan"]
```

```text
CODEOWNERS - LAST MATCH WINS (like .gitignore)      (lines 117-150)
  *                     @contoso/platform-team      <- general FIRST
  /src/api/             @contoso/backend-team
  /src/api/billing/     @contoso/billing-team @sarah-lead   <- specific LAST, and it REPLACES
  /src/auth/            @contoso/security-team
  **/security*.yml      @contoso/security-team      <- glob owns a KIND of file at any depth
  locations: .github/CODEOWNERS | CODEOWNERS | docs/CODEOWNERS
  needs: require_code_owner_review = true  AND  the team has WRITE access
         (no access -> the line is SILENTLY skipped, and an earlier pattern wins)
```

```yaml
# MERGE QUEUE - for semantic conflicts              (lines 238-263)
{"type":"merge_queue","parameters":{
   "grouping_strategy":"ALLGREEN",         // batch passes as a unit, or bisect and eject
   "merge_method":"squash",
   "max_entries_to_build":5, "max_entries_to_merge":5,
   "min_entries_to_merge_wait_minutes":5}}    // the batching window

on:
  merge_group:                    # a DIFFERENT event - main + queued PRs on a temp branch
    types: [checks_requested]
# you need BOTH workflows: pull_request (is this change ok?) and merge_group (are these compatible?)
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 09 |
| 38–43 | Re-read the trap index and the vocabulary pairs, then move on |
| 30–37 | Write the GitHub/Azure Repos pairs and the CODEOWNERS rule from memory, then retake |
| Below 30 | Redo Tasks 1, 2 and 5 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 08.

:::danger The three facts

**A required check matches the job's `name:`** — not the workflow's. A mismatch leaves the PR **pending
forever**, and the usual "fix" is to delete the rule.

**CODEOWNERS is last-match-wins**, and an entry naming a team without repository access is **silently
ignored**.

**Azure Repos policies need `--blocking true`**, or they are advisory.

Contoso's problem was never carelessness. The correct behaviour was optional.

:::
