---
sidebar_position: 3.5
toc_max_heading_level: 2
title: "Challenge 09: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 09 — AZ-400 exam questions

**48 questions** built only from what Challenge 09 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-09.md`**.

:::danger Read this before you start

Two subjects, and each has one deciding idea.

**Permissions: pick the narrowest role that satisfies the requirement.** GitHub's five, in order —
**Read**, **Triage**, **Write**, **Maintain**, **Admin**. The pair the exam tests is **Write versus
Maintain**: Write pushes **code**; Maintain manages **the repository** (topics, wiki, description) and
still cannot change visibility, delete it or manage access.

**And the CLI names differ from the UI labels.** Read is `pull`, Write is `push`, and the API takes
those. Triage, maintain and admin keep their names.

**Tags: annotated or lightweight, and it is not a style choice.** `git tag -a` creates a **full object**
with a tagger, a date, a message, and the ability to be signed. `git tag` alone creates a **pointer**.
Only annotated tags are found by `git describe`, which is what CI uses to derive a version.

**The one fact that ties the whole challenge together: `git describe` returns
`{tag}-{commits-ahead}-g{short-sha}`.** `v1.2.0-47-g2414721` means 47 commits after `v1.2.0`, at commit
`2414721`. The `g` is for "git", not part of the hash.

The scenario at line 20 is 200 contributors on one access level, an intern who changed
`/deploy/prod/`, and tags in four incompatible formats.

:::

---

# Section A — Multiple choice

---

## Q1

What is the difference between GitHub's **Maintain** and **Write** roles?

- A. Maintain can merge pull requests; Write cannot merge at all
- B. Maintain can push to protected branches; Write cannot push
- C. They are aliases for the same underlying permission set
- D. Maintain manages settings; Write pushes code and manages PRs

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-09.md`:** lines **40** and **49**, with the explanation at **555**.

```bash
# Maintain: Tech leads (manage issues, PRs, but can't change settings)
# Write: Senior developers (push to non-protected branches)
```

**Maintain is "manage the repository without owning it".** Description, topics, wiki, interaction limits
— and **not** visibility, deletion or access management, which stay with Admin.

**Why B is the misconception worth killing.** No role bypasses branch protection by default. That is what
`enforce_admins` and bypass lists control (Challenge 08), not the role.

**Read the comment on line 49 carefully**: Write pushes to **non-protected** branches. On a repository
where `main` is protected, Write means "push a feature branch and open a PR".

</details>

---

## Q2

What is the key difference between an annotated and a lightweight tag?

- A. Annotated tags are full objects; lightweight are pointers
- B. Annotated tags can be pushed; lightweight tags cannot be
- C. Annotated tags are encrypted; lightweight tags are plain
- D. Annotated tags require a GPG key; lightweight tags do not

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-09.md`:** lines **231–232**.

```bash
git cat-file -t v1.2.0                # Output: tag (annotated - full object)
git cat-file -t build-2024.01.15-rc1  # Output: commit (lightweight - just a ref)
```

**`git cat-file -t` is the proof, and it is worth remembering as the diagnostic.** An annotated tag's type
is `tag`; a lightweight tag's type is `commit`, because the ref points straight at the commit with no
tag object in between.

**Why D is close but wrong.** Annotated tags **can** be signed (`git tag -v` at line 224 verifies one),
and signing is optional. The `-a` flag creates the object; `-s` signs it.

**And the consequence that matters** (line 234): only annotated tags appear in `git describe`, which is
why every release tag in this challenge uses `-a`.

</details>

---

## Q3

How do you restrict a group from modifying a specific **path** in an Azure Repos repository?

- A. A `.gitignore` entry excluding the protected path
- B. A separate repository for the protected files
- C. Path-level security with a path-scoped token
- D. Branch policies with file-pattern exclusions

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-09.md`:** lines **163–167**.

```bash
az devops security permission update \
  --namespace-id "2e9eb7ed-3c0a-47d4-87c1-0ffdd275fd87" \
  --subject "vssgp.intern-group-descriptor" \
  --token "repoV2/project-id/repo-id/refs/heads/main//deploy/prod" \
  --deny-bit 4
```

**Read the token's structure — it is the whole answer.**
`repoV2/{project}/{repo}/refs/heads/{branch}//{path}`. The **double slash** separates the ref from the
path, and that is what makes the permission path-scoped rather than repository-scoped.

**Why A is a category error.** `.gitignore` stops files being **tracked**; it has nothing to do with who
may change tracked files.

**This is a genuine Azure Repos capability with no GitHub equivalent.** GitHub achieves the same intent
with CODEOWNERS plus required code owner review — which is the fix in Break scenario 1 (line 469).

</details>

---

## Q4

`git describe --tags` returns `v1.2.0-47-g2414721`. What does it mean?

- A. HEAD is 47 commits past `v1.2.0`, SHA `2414721`
- B. 47 files changed since the `v1.2.0` tag was cut
- C. Build number 47 with hash prefix `2414721`
- D. `v1.2.0` was released 47 days before the build

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-09.md`:** lines **235–236** and **588**.

```bash
git describe --tags
# Output: v1.2.0-14-g2414721 (14 commits after v1.2.0)
```

**The format is `{tag}-{commits-ahead}-g{short-sha}`, and the `g` stands for "git"** — it is a prefix,
not part of the hash.

**And the special case is worth knowing**: if HEAD is **exactly on** a tag, `git describe` returns just
the tag name with no suffix. **That is how CI distinguishes "this is a release build" from "this is 47
commits past a release".**

</details>

---

## Q5

An intern's change to `/deploy/prod/kubernetes.yaml` merged without review. What is the fix?

- A. Remove the intern's write access to the repository
- B. Add the deploy path to CODEOWNERS with owner review
- C. Move deployment manifests to a separate repository
- D. Add a `.gitignore` entry for the deployment path

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-09.md`:** lines **465–472**.

```bash
echo '/deploy/prod/ @contoso/platform-admins @contoso/sre-team' >> .github/CODEOWNERS
```

**The cause is stated at line 445: CODEOWNERS did not cover deployment manifests.** The protection
existed; the path was outside it.

**And the diagnosis at lines 453–458 is the pair to remember** — check whether the path is covered, then
check whether code owner review is actually required. **Either alone is insufficient** (Challenge 08 Q6).

**Why A over-corrects.** Interns need write access to do their work; the requirement is that *this path*
needs senior eyes, not that this person needs no access.

</details>

---

## Q6

`git describe --tags` returns `release-20231115-47-gabc1234` instead of a semver tag. Why, and what is
the fix?

- A. The repository holds too many tags to search
- B. `git describe` only ever considers annotated tags
- C. Mixed formats; filter with `--match "v[0-9]*"`
- D. The semver tag was deleted from the remote

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-09.md`:** lines **496–509**.

```bash
git describe --tags --match "v[0-9]*"
```

**`git describe` returns the *nearest reachable* tag, not the newest semver one.** With `1.0.0`, `V1.1.0`,
`v1.2.0`, `release-20231115` and `v1.3` all in the repository (lines 488–494), whichever is closest wins
— and CI derives the wrong version from it.

**`--match` is the immediate fix; consistency is the real one.** The tag ruleset at lines 524–539
prevents non-compliant tags being created at all.

**Note the four distinct problems in that tag list**: a missing `v`, an uppercase `V`, a date format, and
a two-component version. **All four break a parser that expects `v{MAJOR}.{MINOR}.{PATCH}`.**

</details>

---

## Q7

Which GitHub permission value does the API use for **Read**?

- A. `read`
- B. `pull`
- C. `view`
- D. `triage`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-09.md`:** line **74**.

```bash
gh api orgs/contoso/teams/stakeholders/repos/contoso/platform-monorepo \
  --method PUT -f permission="pull"
```

**Read is `pull` and Write is `push`** (line 56) — the API keeps Git's vocabulary while the UI shows
friendlier labels.

**The other three match their labels**: `triage` (line 65), `maintain` (line 47), `admin` (line 38).
**Two renamed, three not** — which is exactly the sort of asymmetry the exam tests.

</details>

---

## Q8

Which role suits junior developers who manage issues but must not push code?

- A. Read
- B. Write
- C. Maintain
- D. Triage

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-09.md`:** lines **58–65**.

```bash
# Triage: Junior developers (manage issues, can't push code)
```

**Triage is the "helpful but harmless" role**: label, assign, close and reopen issues and pull requests,
with no write access to the code.

**Why A is too narrow.** Read cannot manage issues at all, so a junior could not triage the backlog.

**And note the scenario's outcome** (line 20): giving everyone the same level is what let an intern push
to `/deploy/prod/`. **Triage is the level that would have prevented it** — combined with path
protection for the people who do need write access.

</details>

---

## Q9

What does `parent_team_id` accomplish when creating a team?

- A. It nests the team so it inherits the parent's access
- B. It merges the two teams into a single team
- C. It copies the parent team's members into the child
- D. It sets the parent team's lead as the maintainer

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-09.md`:** lines **100–112**.

```bash
PARENT_ID=$(gh api orgs/contoso/teams/engineering --jq '.id')

gh api orgs/contoso/teams --method POST \
  -f name="billing-team" \
  -f parent_team_id="$PARENT_ID"
```

**Nesting is how you grant baseline access once.** Give `engineering` read on the monorepo and all three
child teams have it, without three separate grants to maintain.

**Access flows down, membership does not.** A member of `billing-team` is **not** automatically a member
of `engineering` for other purposes — but they receive `engineering`'s repository permissions.

**Which is what makes 200 contributors across 8 teams manageable** (line 20): one hierarchy instead of
eight independent permission sets.

</details>

---

## Q10

What is the difference between team roles `maintainer` and `member`?

- A. `maintainer` has admin access to the team's repositories
- B. `maintainer` manages team membership; `member` cannot
- C. `member` is read-only on every repository the team has
- D. They are the same role under two different labels

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-09.md`:** lines **121–123**.

```bash
gh api orgs/contoso/teams/billing-team/memberships/sarah-billing --method PUT -f role="maintainer"
gh api orgs/contoso/teams/billing-team/memberships/dev-bob --method PUT -f role="member"
```

**Team role and repository permission are two different dials, and the exam relies on the confusion.**
A team maintainer manages **who is in the team**; the team's **repository** permission is set separately
(line 38 onwards).

**So a team maintainer of a read-only team still has read-only access to the code** — they simply decide
who else gets it.

</details>

---

## Q11

Which tag type does the convention require for production releases?

- A. Lightweight, so they can be created quickly
- B. Either type, at the release manager's choice
- C. Signed only, using the release GPG key
- D. Annotated, with a descriptive message

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-09.md`:** line **277**.

```text
- All production releases MUST use annotated tags with descriptive messages
- Pre-releases follow SemVer pre-release syntax
- CI/build tags may be lightweight (auto-generated, high volume)
- Never delete or move a published release tag
```

**Four rules, and the fourth is the one that causes incidents when broken.** A published tag is a
contract — someone has deployed from it, an artifact references it, an audit cites it. Moving it
silently changes what those things mean.

**And "CI/build tags may be lightweight" is a deliberate exception** (line 279): they are auto-generated
in high volume, nobody reads their metadata, and creating full objects for thousands of builds is waste.

</details>

---

## Q12

Which tag format does the convention define for pre-releases?

- A. `v{X}.{Y}.{Z}-{pre}.{N}`, e.g. `v2.2.0-beta.1`
- B. `beta-{X}.{Y}.{Z}`, for example `beta-2.2.0`
- C. `v{X}.{Y}.{Z}b{N}`, for example `v2.2.0b1`
- D. `{X}.{Y}.{Z}-pre`, for example `2.2.0-pre`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-09.md`:** lines **251–253** and **271**.

```bash
git tag -a v2.2.0-alpha.1 -m "Alpha: new dashboard (unstable)"
git tag -a v2.2.0-beta.1 -m "Beta: new dashboard (feature complete, testing)"
git tag -a v2.2.0-rc.1 -m "Release candidate: final testing"
```

**Hyphen then identifier then dot then number — that is SemVer's pre-release syntax**, and it sorts
correctly: `alpha` < `beta` < `rc`, and `beta.1` < `beta.2`.

**And a pre-release version sorts *before* its release**: `v2.2.0-rc.1` precedes `v2.2.0`. That ordering
is why the syntax matters to tooling rather than being cosmetic.

</details>

---

## Q13

In the auto-tag workflow, what determines the version bump?

- A. The number of commits since the last tag
- B. The name of the branch being tagged
- C. A manual input supplied at dispatch time
- D. Conventional commit messages since the tag

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-09.md`:** lines **315–329**.

```bash
          COMMITS=$(git log "$LATEST_TAG"..HEAD --pretty=format:"%s")

          if echo "$COMMITS" | grep -q "^BREAKING CHANGE\|^.*!:"; then
            MAJOR=$((MAJOR + 1)); MINOR=0; PATCH=0
          elif echo "$COMMITS" | grep -q "^feat"; then
            MINOR=$((MINOR + 1)); PATCH=0
          elif echo "$COMMITS" | grep -q "^fix"; then
            PATCH=$((PATCH + 1))
```

**Challenge 03's convention paying off.** The commit messages were made machine-readable so that exactly
this could be automated.

**Note the reset behaviour**: a major bump zeroes minor *and* patch; a minor bump zeroes patch. **That is
SemVer, and getting it wrong produces versions like `2.1.3` → `3.1.3`.**

**And the `else` at line 326 sets `skip=true`** — a release with only `docs` and `chore` commits creates
no tag at all, which is correct (Challenge 03 Q17).

</details>

---

## Q14

Why does the auto-tag workflow need `fetch-depth: 0`?

- A. To fetch every branch rather than only the default
- B. To speed up the checkout on a large monorepo
- C. `git describe` and `git log` need the full history
- D. To include submodules in the checkout as well

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-09.md`:** lines **299–301**.

**Two operations depend on it.** `git describe --tags --abbrev=0` (line 307) needs the tags, and
`git log "$LATEST_TAG"..HEAD` (line 315) needs the commits between them.

**On a shallow clone the fallback at line 307 fires** — `|| echo "v0.0.0"` — so the workflow silently
decides the repository has never been released and starts numbering from zero.

**A green run producing `v0.1.0` on a mature repository** is the symptom, and it is the same
history-dependent trap as Challenges 03, 05 and 07.

</details>

---

## Q15

What does `persistCredentials: true` do in the Azure Pipelines equivalent?

- A. It stores the credentials inside the repository itself
- B. It keeps the credential so later git commands can push
- C. It caches the checkout between pipeline runs
- D. It enables a shallow fetch for the checkout step

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-09.md`:** lines **376–378**.

```yaml
  - checkout: self
    fetchDepth: 0
    persistCredentials: true
```

**By default the checkout task removes the credential after cloning**, so a later `git push` in a script
step fails with an authentication error.

**And the pair on those three lines is the whole precondition for automated tagging in Azure Pipelines**:
full history to read the tags, and a retained credential to push the new one.

**The GitHub equivalent is `permissions: contents: write`** (line 297) — different mechanism, same
requirement: a job that writes to the repository must be granted the ability to.

</details>

---

## Q16

What does the tag ruleset in Break scenario 2 enforce?

- A. It blocks creation of tags not matching `refs/tags/v*`
- B. It deletes any non-compliant tags already present
- C. It renames non-compliant tags to the convention
- D. It requires every new tag to be annotated and signed

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-09.md`:** lines **526–537**.

```json
  "target": "tag",
  "conditions": {
    "ref_name": {
      "include": ["~ALL"],
      "exclude": ["refs/tags/v*"]
    }
  },
  "rules": [ { "type": "creation" } ]
```

**Read the logic carefully, because it is inverted from how it first appears.** The condition matches
**all** tags **except** those starting with `v`, and the `creation` rule **blocks creation** of anything
matching. So `v2.1.0` is allowed and `release-20240115` is refused.

**`target: "tag"`** is what makes this a tag ruleset rather than a branch one — the same rulesets API,
pointed at a different ref type.

**And it is preventive, which is the point.** Cleaning up existing tags (lines 513–521) fixes history
once; the ruleset stops the problem returning.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** are GitHub repository permission levels used in Task 1? (Choose three.)

- A. Owner
- B. Triage
- C. Contribute
- D. Maintain
- E. Manage
- F. Admin

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-09.md`:** lines **38**, **47**, **65**.

**Five levels exist — Read, Triage, Write, Maintain, Admin** — and the API values are `pull`, `triage`,
`push`, `maintain`, `admin`.

**Why A is organisation vocabulary, not repository.** **Owner** is an organisation role (Challenge 40);
a repository's top level is **Admin**.

**Why C is Azure Repos vocabulary.** Its equivalent of Write is **Contribute** (line 177) — which is
exactly the kind of cross-platform swap the exam builds distractors from.

</details>

---

## Q18

Which **three** are true of annotated tags? (Choose three.)

- A. They store tagger name, email, date and message
- B. They cannot be pushed to a remote repository
- C. They can be signed with a GPG or SSH key
- D. They are created with a bare `git tag` command
- E. They are found and used by `git describe`
- F. They point at a commit with no intermediate object

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-09.md`:** lines **566**, **224**, **234**.

**Why D and F describe *lightweight* tags** (line 232), and why B is false of both — line 204 pushes an
annotated tag and line 208 pushes a lightweight one.

**The `git describe` property at E is the operationally important one.** A release marked with a
lightweight tag is invisible to version derivation, so CI computes the wrong version and nobody notices
until a build is published under it.

</details>

---

## Q19

Which **three** appear in Contoso's tag naming convention table? (Choose three.)

- A. `hotfix-{ticket}` — annotated
- B. `v{MAJOR}.{MINOR}.{PATCH}` — annotated
- C. `snapshot-{sha}` — lightweight
- D. `release-{YYYY.MM.DD}` — annotated
- E. `main-{N}` — annotated
- F. `ci-build-{N}` — lightweight

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-09.md`:** lines **270–274**.

```text
| `v{MAJOR}.{MINOR}.{PATCH}` | Versioned releases | v2.1.0 | Annotated |
| `release-{YYYY.MM.DD}` | Date-based releases| release-2024.01.15| Annotated |
| `ci-build-{N}` | CI build artifacts | ci-build-1847 | Lightweight|
```

**Note that `release-{YYYY.MM.DD}` is annotated** even though it is date-based — the **type** is decided
by whether a human will read the tag later, not by the naming pattern.

**And `deploy-{env}-{date}.{N}`** (line 274) is the fifth row: a lightweight **deployment marker**, which
is how you record what shipped where without polluting the version namespace.

</details>

---

## Q20

Which **two** does the auto-tag workflow produce? (Choose two.)

- A. A commit updating `CHANGELOG.md` on `main`
- B. A deployment to the staging environment
- C. An annotated tag with a changelog in its message
- D. A pull request proposing the version bump
- E. A GitHub release with auto-generated notes

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-09.md`:** lines **342–351** and **356–358**.

```bash
          CHANGELOG=$(git log ${{ steps.version.outputs.latest_tag }}..HEAD \
            --pretty=format:"- %s (%h)" --no-merges)
          git tag -a "${{ steps.version.outputs.new_tag }}" -m "Release ... $CHANGELOG"
```

```bash
          gh release create "${{ steps.version.outputs.new_tag }}" --generate-notes
```

**Two changelogs from two sources, deliberately.** The tag message is built from **commit subjects**;
the release notes come from **merged pull requests** via `--generate-notes` (Challenge 05 Q9).

**And `--no-merges` on line 343 keeps merge commits out of the tag message** — otherwise every entry
would be "Merge pull request #123 from ...".

</details>

---

## Q21

Which **two** prevent a repeat of the `/deploy/prod/` incident? (Choose two.)

- A. A `.gitignore` entry for the production manifests
- B. A CODEOWNERS entry for `/deploy/prod/` with owner review
- C. Removing all intern accounts from the organisation
- D. An Azure Repos path-level deny for the intern group
- E. A longer pull request template with a checklist

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-09.md`:** lines **465–469** and **163–167**.

**Two platforms, two mechanisms for the same intent** — which is the pairing this challenge exists to
teach. GitHub routes the review; Azure Repos denies the write.

**They differ in kind, and it is worth stating.** CODEOWNERS makes the change **reviewable by the right
people**; a path deny makes it **impossible**. The first suits a path anyone may propose changes to; the
second suits a path only one group should touch at all.

</details>

---

## Q22

Which **two** settings does Task 7 use to keep history clean? (Choose two.)

- A. `allow_merge_commit: false`
- B. `has_wiki: false`
- C. `has_discussions: true`
- D. `delete_branch_on_merge: true`
- E. `allow_auto_merge: true`

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-09.md`:** lines **411** and **415**.

```json
  "allow_squash_merge": true,
  "allow_merge_commit": false,
  "allow_rebase_merge": true,
  "delete_branch_on_merge": true,
```

**Disabling merge commits is what actually produces linear history** (Challenge 01 Q13); deleting merged
branches keeps the branch list meaningful on a 200-contributor repository.

**Why E is convenience rather than hygiene**, and why B and C are feature toggles — including
`web_commit_signoff_required: true` at line 436, which is a **compliance** control requiring a DCO
sign-off on web edits.

</details>

---

## Q23

Which **two** does Task 7 enable for supply-chain security? (Choose two.)

- A. Repository topics for discoverability
- B. `vulnerability-alerts` on the repository
- C. Discussions enabled for the community
- D. Projects enabled for sprint planning
- E. `automated-security-fixes` on the repository

<details>
<summary>Show answer</summary>

### Answer: B, E

**In `challenge-09.md`:** lines **422–423**.

```bash
gh api repos/contoso/platform-monorepo/vulnerability-alerts --method PUT
gh api repos/contoso/platform-monorepo/automated-security-fixes --method PUT
```

**Alerts detect; automated fixes act.** The second raises Dependabot pull requests for the alerts the
first produces — which is Challenge 44's alerts-versus-updates distinction in its two-line form.

**Note both are `PUT` with no body.** They are switches, not configuration — enabled by the presence of
the call.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must give 200 contributors appropriate access rather than one shared level, protect
production deployment manifests, and make tags consistent enough for CI to derive versions
automatically.

---

## Q24

**Proposed solution:** Create five teams mapped to Read, Triage, Write, Maintain and Admin, nested under
an `engineering` parent for baseline access. Add `/deploy/prod/` to CODEOWNERS with the platform admins
and SRE team, and require code owner review. In Azure Repos, deny write on the `/deploy/prod/` path token
for the intern group. Standardise on annotated `v{MAJOR}.{MINOR}.{PATCH}` tags for releases with
lightweight tags for CI builds, add a tag ruleset blocking non-`v*` tag creation, and add an auto-tag
workflow using full history that derives the bump from conventional commits.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-09.md`:** lines **31–74**, **94–118**, **465–469**, **163–167**, **270–280**,
**524–539**, **286–361**.

| Requirement | Mechanism |
|---|---|
| Appropriate access at scale | Five roles, nested teams for the baseline |
| Production manifests protected | CODEOWNERS + code owner review; path deny in Azure Repos |
| Tags consistent | Convention + tag ruleset blocking non-compliant creation |
| CI derives versions | Annotated tags + `git describe` + conventional-commit bump |

**The ruleset is what makes the convention durable.** Documented rules are followed until someone is in a
hurry; a rule that refuses the push is followed always.

</details>

---

## Q25

**Proposed solution:** Give all 200 contributors Write access and rely on branch protection to stop
mistakes. Ask the team to use consistent tags and document the convention in the wiki. Use lightweight
tags everywhere for simplicity. Have CI read the version from a `VERSION` file that developers update by
hand.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures.**

**Write for everyone is the current state.** Line 20 says everyone has the same access level, and branch
protection did not prevent the incident — the intern's change reached `/deploy/prod/` through a **merged
pull request**, not a direct push.

**A documented convention is what Contoso has been failing to follow**, producing the four incompatible
formats at lines 488–494. The tag ruleset is what enforces it.

**Lightweight tags everywhere breaks `git describe`** (Q2, Q18), so nothing can derive a version — which
is the automation the requirement asks for.

**And a hand-edited `VERSION` file reintroduces the manual step** the conventional-commit bump removes.
It also drifts: nothing ties it to what was actually released, so the file and the tags disagree.

</details>

---

## Q26

**Proposed solution:** Create five teams mapped to the five roles, nested under `engineering`. Add
`/deploy/prod/` to CODEOWNERS with code owner review required, and deny the path in Azure Repos.
Standardise on annotated `v` tags with a tag ruleset. Add the auto-tag workflow deriving the bump from
conventional commits. Where a release tag turns out to be wrong, delete it and re-create it on the
correct commit so the version history stays tidy.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**Line 280 states the rule the sentence breaks:** *"Never delete or move a published release tag."*

**A published tag is referenced by things you do not control.** A deployment recorded `v1.2.0`; an
artifact was built from it and stored under that name; an incident report cites it; a customer's
support ticket quotes it. **Moving the tag silently changes what every one of those references means**,
and nothing in the audit trail records that it moved.

**The failure is also asymmetric across clones.** Developers who already fetched keep the old tag
pointing at the old commit, and Git does **not** update an existing local tag on fetch by default. So
half the team's `v1.2.0` is one commit and half is another — with no warning on either side.

**The correct response is a new tag.** If `v1.2.0` was cut from the wrong commit, ship `v1.2.1` from the
right one. **Tags are append-only in practice**, which is why Break scenario 2's re-tagging (lines
513–521) is presented as a clean-up of *inconsistent legacy* tags requiring **coordination with the
team** (line 511), not as routine practice.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — GitHub permissions

| # | Statement | Answer |
|---|---|---|
| 1 | Maintain can manage repository settings |  |
| 2 | Maintain can change repository visibility |  |
| 3 | Triage can manage issues without pushing code |  |
| 4 | The API value for Write is `push` |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Maintain can manage repository settings | **Yes** |
| 2 | Maintain can change repository visibility | **No** |
| 3 | Triage can manage issues without pushing code | **Yes** |
| 4 | The API value for Write is `push` | **Yes** |

**In `challenge-09.md`:** lines **555**, **555**, **58**, **56**.

Row 2 is the boundary between Maintain and Admin — visibility, deletion and access management stay with
Admin.

Row 4 is the naming asymmetry: `pull` and `push` are renamed, the other three are not (Q7).

</details>

---

## Q28 — tags

| # | Statement | Answer |
|---|---|---|
| 1 | `git tag -a` creates a full tag object |  |
| 2 | `git cat-file -t` on a lightweight tag returns `commit` |  |
| 3 | `git describe` finds lightweight tags by default |  |
| 4 | A published release tag may be moved if it was wrong |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `git tag -a` creates a full tag object | **Yes** |
| 2 | `git cat-file -t` on a lightweight tag returns `commit` | **Yes** |
| 3 | `git describe` finds lightweight tags by default | **No** |
| 4 | A published release tag may be moved if it was wrong | **No** |

**In `challenge-09.md`:** lines **192**, **232**, **234**, **280**.

Row 3 is why the challenge insists on annotated tags for releases — `--tags` is what makes `describe`
consider lightweight ones at all.

Row 4 is Q26, and it is the rule with the widest blast radius when broken.

</details>

---

## Q29 — Azure Repos

| # | Statement | Answer |
|---|---|---|
| 1 | Path-level permissions use a `repoV2/.../refs/heads/main//path` token |  |
| 2 | The Git Repositories namespace ID is `2e9eb7ed-3c0a-47d4-87c1-0ffdd275fd87` |  |
| 3 | Azure Repos calls the Write equivalent "Contribute" |  |
| 4 | GitHub has an equivalent path-level permission |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Path-level permissions use a `repoV2/.../refs/heads/main//path` token | **Yes** |
| 2 | The Git Repositories namespace ID is `2e9eb7ed-3c0a-47d4-87c1-0ffdd275fd87` | **Yes** |
| 3 | Azure Repos calls the Write equivalent "Contribute" | **Yes** |
| 4 | GitHub has an equivalent path-level permission | **No** |

**In `challenge-09.md`:** lines **166**, **137**, **177**, and Q3.

Row 4 is the capability difference that decides which mechanism a question wants — a **path deny** in
Azure Repos, **CODEOWNERS plus required review** on GitHub.

</details>

---

## Q30 — automation

| # | Statement | Answer |
|---|---|---|
| 1 | The auto-tag workflow needs `fetch-depth: 0` |  |
| 2 | It needs `contents: write` |  |
| 3 | Azure Pipelines needs `persistCredentials: true` to push a tag |  |
| 4 | A release with only `docs` commits still creates a tag |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | The auto-tag workflow needs `fetch-depth: 0` | **Yes** |
| 2 | It needs `contents: write` | **Yes** |
| 3 | Azure Pipelines needs `persistCredentials: true` to push a tag | **Yes** |
| 4 | A release with only `docs` commits still creates a tag | **No** |

**In `challenge-09.md`:** lines **301**, **297**, **378**, **326–328**.

Row 4 is the `else` branch setting `skip=true` — no version-bumping commit type, no release (Q13).

Rows 1–3 are the three preconditions for a job that writes back to the repository, and each fails
differently: no history, no permission, no credential.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each person to the narrowest role that fits.

| Person | Role |
|---|---|
| DevOps lead who configures branch protection |  |
| Tech lead who manages topics and the wiki |  |
| Senior developer pushing feature branches |  |
| Intern triaging the issue backlog |  |
| External auditor reading the code |  |
| Business stakeholder tracking progress |  |

**Options:** Admin · Maintain · Read · Triage · Write

<details>
<summary>Show answer</summary>

| Person | Role |
|---|---|
| DevOps lead who configures branch protection | **Admin** |
| Tech lead who manages topics and the wiki | **Maintain** |
| Senior developer pushing feature branches | **Write** |
| Intern triaging the issue backlog | **Triage** |
| External auditor reading the code | **Read** |
| Business stakeholder tracking progress | **Read** |

**In `challenge-09.md`:** lines **30**, **40**, **49**, **58**, **67**.

**Pick the narrowest role that satisfies the requirement** — the same instinct as every permission
question on this exam. The tech lead does not need Admin to rename a topic.

</details>

---

## Q32

Match each API permission value to its label.

| API value | Label |
|---|---|
| `pull` |  |
| `triage` |  |
| `push` |  |
| `maintain` |  |
| `admin` |  |

**Options:** Admin · Maintain · Read · Triage · Write

<details>
<summary>Show answer</summary>

| API value | Label |
|---|---|
| `pull` | **Read** |
| `triage` | **Triage** |
| `push` | **Write** |
| `maintain` | **Maintain** |
| `admin` | **Admin** |

**In `challenge-09.md`:** lines **74**, **65**, **56**, **47**, **38**.

**Only two are renamed, and both keep Git's vocabulary.** `pull` and `push` describe what the permission
lets you do to the repository, which is why they survived into the API.

</details>

---

## Q33

Arrange the steps to make CI derive versions automatically from tags.

**Items:** Add a tag ruleset blocking non-compliant tag creation · Retag legacy tags into the `v` format ·
Add the auto-tag workflow with full history and `contents: write` · Agree the naming convention and
record it

<details>
<summary>Show answer</summary>

### Answer

1. Agree the naming convention and record it — lines **266–280**
2. Retag legacy tags into the `v` format — lines **513–521**
3. Add a tag ruleset blocking non-compliant tag creation — lines **524–539**
4. Add the auto-tag workflow with full history and `contents: write` — lines **286–361**

**Step 2 before step 3 is the ordering the exam tests.** Enable the ruleset first and the clean-up itself
is blocked — you cannot create `v1.0.0` to replace `1.0.0` if the rule refuses tag creation while you are
still working out the exceptions.

**And step 4 last, because it depends on the other three.** The workflow reads the newest tag with
`git describe`; run it against the mixed tags at lines 488–494 and it derives a version from
`release-20231115` (Q6).

**Step 2 is also the only one requiring coordination** (line 511) — deleting and recreating tags changes
what other people's clones hold.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| `git describe` returns a date-based tag |  |
| CI starts versioning from `v0.0.0` |  |
| A release tag does not appear in `git describe` |  |
| An intern's change reaches a protected path |  |
| The Azure Pipelines tag push fails to authenticate |  |
| Two developers disagree on what `v1.2.0` points at |  |

**Options:** A published tag was moved · CODEOWNERS does not cover that path · It is lightweight, not annotated · Mixed tag formats; no `--match` filter · `persistCredentials` not set · Shallow clone — no tags, fallback fired

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| `git describe` returns a date-based tag | **Mixed tag formats; no `--match` filter** |
| CI starts versioning from `v0.0.0` | **Shallow clone — no tags, fallback fired** |
| A release tag does not appear in `git describe` | **It is lightweight, not annotated** |
| An intern's change reaches a protected path | **CODEOWNERS does not cover that path** |
| The Azure Pipelines tag push fails to authenticate | **`persistCredentials` not set** |
| Two developers disagree on what `v1.2.0` points at | **A published tag was moved** |

**In `challenge-09.md`:** lines **497**, **307**, **234**, **445**, **378**, **280**.

**The last row is the one with no error message anywhere.** Git does not update an existing local tag on
fetch, so each clone keeps whichever version it saw first — and both developers are confident.

</details>

---

## Q35

Match each tag pattern to its type and purpose.

| Pattern | Type — purpose |
|---|---|
| `v2.1.0` |  |
| `v2.2.0-beta.1` |  |
| `release-2024.01.15` |  |
| `ci-build-1847` |  |
| `deploy-prod-2024.01.15.1` |  |

**Options:** Annotated — date-based release · Annotated — pre-release · Annotated — versioned release · Lightweight — CI artifact · Lightweight — deployment marker

<details>
<summary>Show answer</summary>

| Pattern | Type — purpose |
|---|---|
| `v2.1.0` | **Annotated — versioned release** |
| `v2.2.0-beta.1` | **Annotated — pre-release** |
| `release-2024.01.15` | **Annotated — date-based release** |
| `ci-build-1847` | **Lightweight — CI artifact** |
| `deploy-prod-2024.01.15.1` | **Lightweight — deployment marker** |

**In `challenge-09.md`:** lines **270–274**.

**The type is decided by whether a human reads it later, not by the pattern.** Releases carry a message
someone will read during an incident; build and deployment markers are machine bookkeeping in high
volume.

</details>

---

# Section F — Hot area

---

## Q36

```bash
gh api orgs/contoso/teams/tech-leads/repos/contoso/platform-monorepo \
  --method [BLANK 1] -f permission="[BLANK 2]"
```

Requirement: tech leads manage repository settings such as topics and the wiki, but cannot change
visibility or delete the repository.

- **BLANK 1:** `POST` / `PATCH` / `PUT` / `GET`
- **BLANK 2:** `admin` / `push` / `write` / `maintain`

<details>
<summary>Show answer</summary>

### Answer: `PUT`, `maintain`

**In `challenge-09.md`:** lines **46–47**.

**`write` is the trap in BLANK 2** — it is the **label**, and the API value is `push` (Q7). Sending
`write` is rejected.

**And `PUT` because adding a team to a repository is idempotent**: repeating it updates the permission
rather than erroring.

</details>

---

## Q37

```bash
PARENT_ID=$(gh api orgs/contoso/teams/engineering --jq '.id')

gh api orgs/contoso/teams --method POST \
  -f name="billing-team" \
  -f [BLANK 1]="$PARENT_ID"

gh api orgs/contoso/teams/billing-team/memberships/sarah-billing \
  --method PUT -f [BLANK 2]="maintainer"
```

- **BLANK 1:** `parent` / `parent_team_id` / `team_parent` / `nested_under`
- **BLANK 2:** `permission` / `level` / `role` / `access`

<details>
<summary>Show answer</summary>

### Answer: `parent_team_id`, `role`

**In `challenge-09.md`:** lines **106** and **121**.

**BLANK 2 is the dial-confusion this challenge sets up.** `role` on a *membership* is `maintainer` or
`member` — who manages the team. `permission` on a *repository* grant is `pull`/`push`/`admin` — what the
team can do to the code (Q10).

**Same word family, different objects.** A team maintainer of a read-only team still has read-only access.

</details>

---

## Q38

```bash
az devops security permission update \
  --namespace-id "2e9eb7ed-3c0a-47d4-87c1-0ffdd275fd87" \
  --subject "vssgp.intern-group-descriptor" \
  --token "repoV2/project-id/repo-id/[BLANK 1]//deploy/prod" \
  --[BLANK 2] 4
```

Requirement: deny the intern group write access to `/deploy/prod/` on `main`.

- **BLANK 1:** `main` / `branches/main` / `heads/main` / `refs/heads/main`
- **BLANK 2:** `allow-bit` / `deny-bit` / `remove-bit` / `set-bit`

<details>
<summary>Show answer</summary>

### Answer: `refs/heads/main`, `deny-bit`

**In `challenge-09.md`:** lines **166–167**.

**The token needs the **full ref**, and the **double slash** separates ref from path** — both are easy to
get wrong and both cause the permission to apply somewhere other than intended.

**And `--deny-bit` matters more than it looks.** In Azure DevOps an explicit **Deny beats Allow** across
every group membership, so this denies the interns regardless of what their other groups grant — which
is exactly what the scenario needs (line 20).

</details>

---

## Q39

```bash
# Release tag
git tag [BLANK 1] v1.2.0 -m "Release v1.2.0 - Q1 2024 billing engine update"

# CI build tag
git tag [BLANK 2] ci-build-1847

# Retroactively tag an older commit
git tag -a v1.1.5 [BLANK 3] -m "Patch release v1.1.5"
```

- **BLANK 1:** `-s` / `-l` / `-a` / *(nothing)*
- **BLANK 2:** `-a` / `-f` / *(nothing)* / `-d`
- **BLANK 3:** `HEAD` / `abc1234` / `main` / `-c abc1234`

<details>
<summary>Show answer</summary>

### Answer: `-a`, *(nothing)*, `abc1234`

**In `challenge-09.md`:** lines **193**, **207**, **211**.

**`-a` for releases, nothing for CI builds** — the convention at lines 277–279.

**And BLANK 3 shows a detail worth knowing: a commit-ish after the tag name tags *that* commit**, not
HEAD. That is how you tag a release retroactively, and it is how the re-tagging in Break scenario 2
works — `$(git rev-list -n 1 1.0.0)` at line 513 resolves the old tag to its commit first.

</details>

---

## Q40

```bash
LATEST_TAG=$(git describe --tags --[BLANK 1] 2>/dev/null || echo "v0.0.0")
COMMITS=$(git log "$LATEST_TAG"[BLANK 2]HEAD --pretty=format:"%s")

if echo "$COMMITS" | grep -q "^BREAKING CHANGE\|^.*!:"; then
  MAJOR=$((MAJOR + 1)); MINOR=[BLANK 3]; PATCH=0
```

- **BLANK 1:** `long` / `always` / `abbrev=0` / `dirty`
- **BLANK 2:** `...` / `..` / `^` / `~`
- **BLANK 3:** `MINOR` / `1` / `$((MINOR + 1))` / `0`

<details>
<summary>Show answer</summary>

### Answer: `abbrev=0`, `..`, `0`

**In `challenge-09.md`:** lines **307**, **315**, **318–319**.

**`--abbrev=0` returns the bare tag name** with no `-47-g2414721` suffix — which is what you want when
the value feeds a version parser rather than a human.

**`..` is the two-dot range**: commits on HEAD not on the tag (Challenge 07 Q22).

**And zeroing MINOR on a major bump is SemVer.** `2.4.7` with a breaking change becomes `3.0.0`, not
`3.4.7` — the reset is at lines 318–319 and it is the arithmetic the exam checks.

</details>

---

## Q41

```json
{
  "name": "Tag naming enforcement",
  "target": "[BLANK 1]",
  "conditions": {
    "ref_name": { "include": ["~ALL"], "exclude": ["refs/tags/[BLANK 2]"] }
  },
  "rules": [ { "type": "[BLANK 3]" } ]
}
```

Requirement: block creation of any tag that does not begin with `v`.

- **BLANK 1:** `branch` / `ref` / `tag` / `push`
- **BLANK 2:** `*` / `release-*` / `v*` / `~ALL`
- **BLANK 3:** `deletion` / `update` / `required_signatures` / `creation`

<details>
<summary>Show answer</summary>

### Answer: `tag`, `v*`, `creation`

**In `challenge-09.md`:** lines **527–536**.

**The logic is a denylist with a carve-out, and it reads backwards at first.** Include everything, exclude
the compliant pattern, then block **creation** of whatever remains matched — so `v2.1.0` is exempt and
everything else is refused.

**`target: "tag"`** is what points the rulesets API at tag refs rather than branches — the same API as
Challenge 08's branch protection, aimed elsewhere.

</details>

---

# Section G — Case study

## Case study: Contoso monorepo governance

### Background

Contoso Ltd's monorepo has **200 contributors across 8 teams**, and **everyone has the same access
level**. Last sprint an **intern pushed a configuration change to `/deploy/prod/`**, causing an outage.
Git tags are inconsistent — `v1.2.3`, `release-20240115`, and **lightweight tags with no metadata**. The
DevOps team must implement access controls and a tagging strategy that **integrates with CI/CD**.

### Requirements

**Access**

- Roles must match responsibility, not be uniform across 200 people
- Baseline access must be granted once, not per team
- Production deployment manifests must not be changeable without senior sign-off, on either platform

**Tags**

- Release tags must carry a tagger, a date and a message, and must be verifiable
- CI must be able to derive the next version without a human editing anything
- Non-compliant tags must be impossible to create

---

## Q42

How should the 200 contributors be structured?

- A. One team holding Write access for every contributor
- B. Individual collaborator grants for each person
- C. Five role-mapped teams nested under `engineering`
- D. Everyone as Admin so that nobody is ever blocked

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-09.md`:** lines **31–74** and **94–118**.

**Both access requirements are met by that one structure.** The five teams match role to responsibility;
the nesting grants the baseline once (Q9).

**Why B is unmaintainable at 200 people** and unauditable — nobody can answer "who can push?" without
enumerating collaborators.

**Why A is the current state** (line 20), and **why D is the failure mode it would become**.

</details>

---

## Q43

How are production manifests protected on **each** platform?

- A. GitHub: CODEOWNERS with owner review; Azure: a path deny
- B. Both platforms: a CODEOWNERS file covering the path
- C. Both platforms: a separate repository for manifests
- D. GitHub: branch protection; Azure Repos: branch policies

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-09.md`:** lines **465–469** and **163–167**.

**Two platforms, two mechanisms** (Q21) — and the requirement says "on either platform", so a single
answer will not do.

**Why D misses the granularity.** Branch protection and branch policies operate on a **branch**; the
requirement is about a **path** within it. `main` is already protected and the intern's change still
merged (line 445).

**Note the difference in strength.** CODEOWNERS makes the change **reviewable by the right people**; the
Azure deny makes it **impossible** for that group.

</details>

---

## Q44

Which tag properties satisfy "must carry a tagger, a date and a message, and must be verifiable"?

- A. Lightweight tags with descriptive, consistent names
- B. Annotated tags only, with no signature applied
- C. GitHub releases created without an underlying tag
- D. Annotated tags, GPG-signed and checked with `-v`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-09.md`:** lines **192–201** and **224**.

**"Tagger, date and message" is the definition of an annotated tag** (line 566). **"Verifiable" is the
word that adds signing** — `git tag -v` verifies a signature, and only an annotated tag can carry one
(Q2).

**Why B satisfies three of the four clauses.** Without a signature there is nothing to verify; the tag's
metadata says who *claims* to have made it.

**Why A fails on all of them.** A lightweight tag stores no metadata at all, however carefully it is
named.

</details>

---

## Q45

Which **two** let CI derive the next version with no human input? (Choose two.)

- A. A `VERSION` file maintained by hand by developers
- B. `git describe --tags --abbrev=0` with full history
- C. The pipeline's incrementing build number
- D. Bump rules driven by commit types since the last tag
- E. The name of the branch that was built

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-09.md`:** lines **307** and **315–329**.

**B finds where you are; D decides where to go next.** Neither alone completes the derivation.

**And both have preconditions this challenge is explicit about.** B needs `fetch-depth: 0` and annotated
tags; D needs Challenge 03's commit convention to actually be enforced.

**Why A reintroduces the human** the requirement excludes, and why C and E carry no compatibility
meaning — a build number never tells a consumer whether an upgrade is safe.

</details>

---

## Q46

How is "non-compliant tags must be impossible to create" satisfied?

- A. Documenting the convention in the contributing guide
- B. A CI job that deletes tags not matching the pattern
- C. A ruleset excluding `refs/tags/v*`, with `creation`
- D. Renaming non-compliant tags on a weekly schedule

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-09.md`:** lines **524–539**.

**"Impossible" is the word that selects a ruleset.** Anything else detects or corrects after the fact.

**Why B is the design teams reach for and why it is worse than it sounds.** Deleting a tag someone has
already fetched leaves their clone holding a tag the server no longer has — and it violates the
never-delete rule (line 280) as routine practice.

**Why A is the state that produced four formats** (lines 488–494).

</details>

---

## Q47

Eight months later, the platform team finds that the auto-tag workflow has been producing versions like
`v0.1.0`, `v0.2.0` and `v0.3.0` for the past six weeks, although the repository has been on `v3.x` for a
year. The workflow is green on every run, and the tags it creates are annotated and correctly formatted.
CI checkout times were recently optimised across all workflows.

What is the most likely cause?

- A. The tag ruleset is rejecting the correctly formatted tags
- B. Conventional commits stopped being used by the team
- C. `contents: write` was removed from the workflow
- D. A shallow clone makes the `v0.0.0` fallback fire every run

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-09.md`:** lines **299–307**.

```bash
LATEST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0")
```

**That fallback is the amplifier.** It exists so the very first run on a fresh repository has somewhere
to start — and on a shallow clone it makes **every** run look like the first one.

**"Green on every run" follows directly.** Nothing errors: `git describe` fails, `2>/dev/null` swallows
the message, the `||` supplies a default, and the workflow proceeds to create a perfectly valid tag with
a completely wrong number.

**And the damage compounds.** Six weeks of `v0.x` tags now sit alongside the real `v3.x` ones, so
`git describe` on a *full* clone may also pick a wrong tag depending on reachability — the mixed-tag
problem from Break scenario 2, self-inflicted.

**Why A and C would fail loudly** — a rejected tag push and a permissions error respectively — and why B
would set `skip=true` and create no tag at all (Q13).

**The durable lesson: a fallback that hides a missing precondition converts a loud failure into a silent
wrong answer.** If the fallback is worth having, log when it fires.

</details>

---

## Q48

A year on, permissions match responsibility, `/deploy/prod/` has not been changed without sign-off, and
CI has produced every release version without anyone typing one.

Which explanation best accounts for the change?

- A. The team was trained to be more careful with permissions
- B. The version became a function of history, not a decision
- C. The intern was removed from the repository entirely
- D. Releases were made less frequent so tagging mattered less

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-09.md`:** lines **31–74**, **163–167**, **192–201**, **270–280**, **286–361**.

**Take the two failures at line 20 in turn.**

*An intern changed `/deploy/prod/`* because 200 people shared one access level and the path had no owner.
**Both halves are now closed**, and closed differently on each platform — reviewable on GitHub,
impossible in Azure Repos.

*Tags in four formats* meant `git describe` returned whatever was nearest, so nothing could be automated
on top of them. **The convention made the format decidable, the ruleset made it enforced, and annotated
tags made it machine-readable.**

**What actually changed is that the version stopped being an opinion.** Before, a release number was
something a person chose and typed; now it is derived — the previous tag plus what the commits since then
declare themselves to be.

**And the same shift applies to access.** Before, "who can change production config" was answered by
trust; now it is answered by a CODEOWNERS line and a deny bit.

**The graded idea: repository governance works when the correct outcome is the *default* outcome.** A
convention people are asked to follow produces four tag formats. A rule that refuses the push produces
one.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **`write` sent as an API permission value** | Q7, Q36 | The value is `push`; Read is `pull` |
| **Maintain assumed to include visibility or deletion** | Q1, Q27 | Those stay with Admin |
| **Maintain assumed to bypass branch protection** | Q1 | No role does by default |
| **Team `role` confused with repository `permission`** | Q10, Q37 | Who manages the team vs what it can do |
| **Lightweight tags used for releases** | Q2, Q18, Q25, Q44 | Invisible to `git describe`, no metadata |
| **`git describe` assumed to pick the newest semver tag** | Q6, Q34 | It picks the **nearest**. Use `--match` |
| **Moving or deleting a published release tag** | Q26, Q28, Q46 | Other clones keep the old target, silently |
| **Shallow clone with `git describe`** | Q14, Q34, Q47 | The `|| echo "v0.0.0"` fallback fires. Green and wrong |
| **`persistCredentials` omitted in Azure Pipelines** | Q15, Q30 | The tag push fails to authenticate |
| **Path permissions attempted with `.gitignore`** | Q3, Q21 | `.gitignore` controls tracking, not access |
| **Path token missing the full ref or the double slash** | Q38 | `repoV2/proj/repo/refs/heads/main//path` |
| **Branch protection offered for a path requirement** | Q43 | Branch-level vs path-level |
| **Ruleset enabled before cleaning legacy tags** | Q33 | The clean-up itself gets blocked |
| **Major bump without zeroing minor and patch** | Q40 | `2.4.7` + breaking = `3.0.0` |

---

# What to memorise

**In `challenge-09.md`:** lines **28–75**, **187–237**, **266–280**, **286–361**.

```text
GITHUB REPOSITORY ROLES - narrowest that fits          API VALUE
  Read       clone, view issues and PRs                pull      <- renamed
  Triage     manage issues/PRs, NO code write          triage
  Write      push to non-protected branches            push      <- renamed
  Maintain   manage repo settings (topics, wiki)       maintain
             NOT visibility, deletion, or access
  Admin      everything                                admin

  team membership role = maintainer | member   (WHO manages the team)
  repository permission = pull|triage|push|maintain|admin   (WHAT the team can do)
  parent_team_id nests a team -> child INHERITS the parent's repository access

AZURE REPOS
  Contribute == GitHub Write.  Namespace "Git Repositories" = 2e9eb7ed-3c0a-47d4-87c1-0ffdd275fd87
  PATH-LEVEL security (no GitHub equivalent):
    --token "repoV2/{project}/{repo}/refs/heads/{branch}//{path}"   <- FULL ref + DOUBLE slash
    --deny-bit 4        (Deny beats Allow across every group)
  GitHub equivalent intent: CODEOWNERS + require_code_owner_reviews
```

```bash
# TAGS - annotated or lightweight is NOT a style choice   (lines 187-237)
git tag -a v1.2.0 -m "..."      # ANNOTATED: full object - tagger, date, message, signable
git tag ci-build-1847           # LIGHTWEIGHT: a pointer. no metadata
git tag -a v1.1.5 abc1234 -m "" # tag a PAST commit (commit-ish after the name)
git tag -v v1.2.0               # verify a signature (annotated only)
git cat-file -t v1.2.0          # -> "tag"     lightweight -> "commit"    <- the proof

git describe --tags             # {tag}-{commits-ahead}-g{short-sha}   the g means "git"
                                # exactly on a tag -> just the tag name
git describe --tags --abbrev=0  # bare tag name, for parsing
git describe --tags --match "v[0-9]*"   # picks the NEAREST tag - filter or it picks the wrong one

CONVENTION            v{MAJOR}.{MINOR}.{PATCH}      annotated   releases
                      v{X}.{Y}.{Z}-{pre}.{N}        annotated   v2.2.0-beta.1
                      release-{YYYY.MM.DD}          annotated   date-based releases
                      ci-build-{N}                  lightweight high volume, machine-only
                      deploy-{env}-{date}.{N}       lightweight deployment markers
RULE: NEVER delete or move a published release tag. Ship a new one instead.
```

```yaml
# AUTO-TAG ON MERGE                                   (lines 286-361)
permissions:
  contents: write                  # writes back to the repo
steps:
  - uses: actions/checkout@v4
    with: {fetch-depth: 0}         # REQUIRED - describe needs tags, git log needs the range
  - run: |
      LATEST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0")
      #                                             ^^^^^^^^^^ fires on a SHALLOW clone
      #                                             -> every run bumps from zero, GREEN
      COMMITS=$(git log "$LATEST_TAG"..HEAD --pretty=format:"%s")
      BREAKING / ^.*!:  -> MAJOR+1, MINOR=0, PATCH=0
      ^feat             -> MINOR+1, PATCH=0
      ^fix              -> PATCH+1
      else              -> skip=true, no tag at all

# Azure Pipelines equivalent
  - checkout: self
    fetchDepth: 0
    persistCredentials: true       # or the git push has no credential
```

```json
// TAG RULESET - make the convention impossible to break   (lines 524-539)
{ "target": "tag",
  "conditions": {"ref_name": {"include": ["~ALL"], "exclude": ["refs/tags/v*"]}},
  "rules": [ {"type": "creation"} ] }
// include everything, EXCLUDE the compliant pattern, block CREATION of the rest
// clean up legacy tags FIRST - the ruleset would block the clean-up
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 10 |
| 38–43 | Re-read the trap index and the role table, then move on |
| 30–37 | Write the five roles with API values and the tag types from memory, then retake |
| Below 30 | Redo Tasks 1, 4 and 6 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 09.

:::danger The three facts

**Pick the narrowest role.** Write pushes code; Maintain manages the repository; only Admin owns it. The
API says `pull` and `push`.

**Annotated for releases, lightweight for machines.** Only annotated tags carry metadata and only they
are found by `git describe` — which is where your version number comes from.

**`git describe` returns `{tag}-{commits-ahead}-g{short-sha}`**, picks the **nearest** tag, and needs the
full history to find any at all.

Contoso's outage was not carelessness. Everyone had the same access, and no path had an owner.

:::
