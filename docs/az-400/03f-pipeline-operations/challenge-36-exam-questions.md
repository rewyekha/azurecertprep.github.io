---
sidebar_position: 93
title: "Challenge 36: exam questions"
---

# Challenge 36 — AZ-400 exam questions

**48 questions** built only from what Challenge 36 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-36.md`**.

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

:::tip Retention is always tiered

Nothing here has one answer. **CI artifacts days, main-branch artifacts weeks, release artifacts a
year.** Every question gives you a class of artifact and expects the matching tier — and the wrong
answers apply one policy to everything.

:::

---

# Section A — Single answer

---

## Q1

Contoso must keep production release artifacts for 1 year but dev/test artifacts for 7 days.

Which approach meets both?

- A. A single 365-day project retention policy
- B. Tiered retention — short defaults plus a retention lease on release runs
- C. A single 7-day policy with manual copies of releases
- D. Disable retention and delete manually

### Answer: B

**In `challenge-36.md`:** the tiers at lines **66–70** and the lease at lines **219–239**.

```text
# - Days to keep artifacts: 30 (default)
# - Days to keep pull request runs: 10
# - Days to keep runs with release artifacts: 365
```

**The default is short; the exception is explicit.** A retention **lease** extends a specific run
beyond the policy, which is what "keep this one for a year" means without keeping everything for a
year.

**Why the others fail**

- **A** — keeps *everything* for a year. The 200 GB of pipeline artifacts becomes the problem the
  scenario is trying to solve
- **C** — manual copies are unauditable and get forgotten. Compliance requires a guarantee
- **D** — the current situation, which is why storage grows 50 GB a month

---

## Q2

What is the default GitHub Actions artifact retention period, and the maximum?

- A. 30 days default, 90 maximum
- B. 90 days default, 400 maximum
- C. 7 days default, 90 maximum
- D. 365 days default, unlimited

### Answer: B

**In `challenge-36.md`:** lines **113–114**.

```text
# Organization/repo level: Settings > Actions > General > Artifact and log retention
# Default: 90 days, Maximum: 400 days
```

**90 days is a long default**, and it is why untouched repositories accumulate storage: every CI run
keeps its artifacts for three months whether anyone wants them or not.

**Note the maximum is 400, not unlimited** — which matters for the 1-year compliance requirement.
365 fits inside 400, so `retention-days: 365` (line 155) works. Anything longer needs a **GitHub
Release** instead (Q6).

**Why the others fail** — A, C and D all misstate one or both figures.

---

## Q3

Which `upload-artifact` input sets per-artifact retention?

- A. `retention-days`
- B. `expires-in`
- C. `ttl`
- D. `keep-days`

### Answer: A

**In `challenge-36.md`:** lines **137**, **146**, **155**.

```yaml
          retention-days: 3    # PR artifacts
          retention-days: 30   # main branch
          retention-days: 365  # release branch
```

**Three uploads, one workflow, three tiers** — selected by `if:` conditions on the branch (lines 132,
141, 150). That is tiered retention expressed in YAML rather than in settings.

**The value cannot exceed the repository or organisation maximum.** Setting 500 where the cap is 400
does not error loudly; it is silently clamped.

**Why the others fail** — none exist as inputs.

---

## Q4

Which Azure DevOps mechanism protects a specific pipeline run from the retention policy?

- A. A retention lease
- B. A branch policy
- C. A pipeline variable
- D. An exclusive lock

### Answer: A

**In `challenge-36.md`:** lines **86–105**.

```powershell
            $body = @{
              daysValid = 365
              definitionId = $(System.DefinitionId)
              ownerId = "User:$(Build.RequestedForId)"
              protectPipeline = $true
              runId = $(Build.BuildId)
            } | ConvertTo-Json
            $uri = "$(System.CollectionUri)$(System.TeamProject)/_apis/build/retention/leases?api-version=7.1"
```

**A lease is a per-run override.** The project policy stays short, and individual runs that matter —
the ones that went to production — get an explicit extension.

**`protectPipeline`** decides whether the **pipeline definition** itself is also protected from
deletion, not just the run. Line 234 sets it `false` for the production deployment, because the run's
artifacts are what compliance cares about.

**Why the others fail** — B governs merging, C is a value, D prevents concurrent deployments
(Challenge 24).

---

## Q5

A retention lease is created but artifacts are still deleted after 30 days.

Which **two** defects are present? (Pick the single best answer.)

- A. The authorization header is missing and the body is not a JSON array
- B. `daysValid` is too low
- C. The pipeline lacks permissions to publish artifacts
- D. The artifact name is wrong

### Answer: A

**In `challenge-36.md`:** Break & fix Exercise 1, lines **556–606**.

```powershell
      # ERROR: Missing authorization header
      Invoke-RestMethod -Uri $uri -Method POST -Body $body
      # Also ERROR: Body must be an array, not a single object
```

**Fixed (lines 591–606):**

```powershell
      $headers = @{
        Authorization = "Bearer $(System.AccessToken)"
        "Content-Type" = "application/json"
      }
      $body = ConvertTo-Json @( @{ daysValid = 365; ... } )
```

**Why it fails silently.** The REST call errors, the PowerShell task may still exit 0 depending on
error handling, and the pipeline goes green. Nobody discovers the lease was never created until 30
days later when the artifact disappears.

**`$(System.AccessToken)`** is the pipeline's own OAuth token — it must be explicitly passed as a
header; it is not applied automatically.

**Why the others fail** — B, C and D would produce different, visible failures.

---

## Q6

Why are GitHub Releases suitable for long-term artifact retention?

- A. They are not subject to Actions artifact retention policies
- B. They are compressed more efficiently
- C. They cost less per gigabyte
- D. They are automatically replicated

### Answer: A

**In `challenge-36.md`:** lines **245–251**.

```bash
gh release create v1.5.0 dist/*.zip \
  --title "v1.5.0 - Payment processing update"

# GitHub Releases are not subject to artifact retention policies
# They persist until explicitly deleted
```

**This is the escape hatch from the 400-day cap.** Workflow artifacts expire; release assets do not.
For anything with a genuine multi-year retention requirement, a Release is the correct container.

**The trade-off:** because they persist forever, they need their own cleanup. Line 257 prunes old
**pre-releases**, keeping the last five, while stable releases are left alone.

**Why the others fail** — B, C and D are all untrue; the storage is the same underlying service.

---

## Q7

Which Azure Artifacts view holds promoted stable versions that are excluded from cleanup?

- A. `@local`
- B. `@prerelease`
- C. `@release`
- D. `@latest`

### Answer: C

**In `challenge-36.md`:** lines **309–314**.

```text
# @local - all versions (default retention applies)
# @prerelease - pre-release versions
# @release - promoted stable versions (longer retention)
```

**Promotion is the protection mechanism.** A package version sitting only in `@local` is subject to
the feed's count limit; promoting it to `@release` takes it out of the cleanup set.

**That is why the feed retention policy at line 333 scopes itself to `["@prerelease"]`** — it deletes
old pre-release versions and never touches promoted ones.

**Why the others fail** — A is the default view where everything lands, B is what gets cleaned up,
D does not exist as an Azure Artifacts view.

---

## Q8

Which feed retention setting caps how many versions of each package are kept?

- A. `countLimit`
- B. `daysToKeepRecentlyCreatedPackages`
- C. `packageTypes`
- D. `views`

### Answer: A

**In `challenge-36.md`:** lines **328–341**.

```json
{
  "countLimit": 5,
  "daysToKeepRecentlyCreatedPackages": 30,
  "packageTypes": ["npm", "NuGet"],
  "views": ["@prerelease"]
}
```

**Two independent guards, and both must be satisfied before a version is deleted.** `countLimit: 5`
keeps the newest five; `daysToKeepRecentlyCreatedPackages: 30` protects anything published in the last
30 days even if it is the sixth version.

That second setting is what stops a burst of releases immediately evicting a version someone is still
consuming.

**Why the others fail** — C scopes which package types the policy applies to, D scopes which views.

---

## Q9

Which ACR command deletes untagged manifests older than 7 days?

- A. `acr purge --filter 'contoso-api:.*' --untagged --ago 7d`
- B. `az acr repository delete --name contoso-api`
- C. `az acr manifest list-metadata`
- D. `az acr build --no-cache`

### Answer: A

**In `challenge-36.md`:** lines **415–418**.

```bash
az acr run --registry contosoregistry --cmd "acr purge \
  --filter 'contoso-api:.*' \
  --untagged \
  --ago 7d" /dev/null
```

**Untagged manifests are the invisible cost.** Every time a tag like `latest` is re-pushed, the
previous manifest keeps its layers but loses its tag — invisible in the portal's tag list, fully
billed.

**Note `az acr run ... /dev/null`** — purge executes as an ACR **task** inside the registry, not on
your machine, so no image pulling is involved. The `/dev/null` is an empty build context.

**Why the others fail** — B deletes the entire repository, C lists metadata, D is a build command.

---

## Q10

An automated ACR purge deleted images production was running.

What was wrong?

- A. The purge had no `--untagged` flag and no tag filter, so it deleted tagged images in use
- B. The schedule ran too often
- C. `--ago 7d` was too long
- D. The registry was the wrong SKU

### Answer: A

**In `challenge-36.md`:** Break & fix Exercise 2, lines **612–634**.

```bash
# BROKEN
az acr run --registry contosoregistry --cmd "acr purge \
  --filter 'contoso-api:.*' \
  --ago 7d" /dev/null
# This deletes the 'latest' and 'v1.4.2' tags that production is using!
```

```bash
# FIXED
  --filter 'contoso-api:sha-.*' \
  --ago 7d \
  --untagged
```

**Two changes, both necessary.** `--untagged` restricts deletion to manifests nothing points at, and
the narrower filter `sha-.*` targets only the commit-tagged CI images rather than every tag including
`latest` and semver releases.

**The general rule:** *purge by what nothing references, not by age alone.* Age tells you nothing about
whether something is in use — a stable release image from last year may be exactly what production
runs.

---

## Q11

Which ACR purge flag retains the most recent N tags?

- A. `--keep 10`
- B. `--untagged`
- C. `--ago 30d`
- D. `--filter`

### Answer: A

**In `challenge-36.md`:** lines **420–424**.

```bash
az acr run --registry contosoregistry --cmd "acr purge \
  --filter 'contoso-api:.*' \
  --keep 10 \
  --ago 30d" /dev/null
```

**`--keep` and `--ago` compose as an AND.** A tag is deleted only if it is **both** older than 30 days
**and** outside the newest 10. That combination is what makes it safe: a repository with only three
tags loses nothing regardless of age.

**Why the others fail** — B targets untagged manifests, C is the age threshold, D selects which
repositories and tags are in scope.

---

## Q12

Which command creates a scheduled, self-running ACR cleanup?

- A. `az acr task create` with `--schedule`
- B. `az acr run` with `--schedule`
- C. An Azure Automation runbook only
- D. A GitHub Actions cron workflow only

### Answer: A

**In `challenge-36.md`:** lines **426–433**.

```bash
az acr task create \
  --name purge-old-images \
  --registry contosoregistry \
  --cmd "acr purge --filter 'contoso-api:.*' --untagged --ago 7d && \
         acr purge --filter 'contoso-api:.*' --keep 20 --ago 90d" \
  --schedule "0 3 * * *" \
  --context /dev/null
```

**Note the two-stage policy chained with `&&`:** aggressive on untagged manifests (7 days),
conservative on tagged ones (keep 20, older than 90 days). Different classes of image, different
tiers — the theme of the whole challenge.

**Why A beats C and D:** the task runs **inside ACR**. No pipeline agent, no credentials to manage, no
external scheduler to keep alive. Both alternatives work and add moving parts.

**Why B fails** — `az acr run` executes once. `az acr task create` registers a recurring task.

---

## Q13

Contoso's storage is 500 GB at $0.30/GB/month. What is the projected saving after retention policies?

- A. ~$36/month
- B. ~$114/month
- C. ~$150/month
- D. ~$1,368/month

### Answer: B

**In `challenge-36.md`:** lines **438–446**.

```text
- Current total: 500 GB at ~$0.30/GB/month = $150/month
- After retention policies:
  - Pipeline artifacts: 200 GB -> 30 GB
  - Package feed:       180 GB -> 50 GB
  - Container images:   120 GB -> 40 GB
- New total: 120 GB at $0.30/GB/month = $36/month
- Monthly savings: $114/month ($1,368/year)
```

**Read the distractors carefully:** $36 is the **new cost**, $150 the **old cost**, $1,368 the
**annual** saving. Only $114 is the monthly saving.

**And note where the reduction comes from:** pipeline artifacts fall 85%, the feed 72%, images 67%.
The biggest absolute win is the artifacts, which is also the easiest — it is a policy setting, not a
cleanup script.

---

## Q14

Which schedule setting makes a weekly cleanup pipeline run even when no code has changed?

- A. `always: true`
- B. `batch: true`
- C. `trigger: none`
- D. `enabled: true`

### Answer: A

**In `challenge-36.md`:** lines **347–355**.

```yaml
schedules:
  - cron: "0 2 * * 0"  # Weekly on Sunday at 2 AM
    displayName: "Weekly artifact cleanup"
    branches:
      include: [main]
    always: true

trigger: none
```

**Without `always: true`, the schedule is skipped when nothing changed** — and a cleanup pipeline that
only runs after code changes is exactly backwards. Storage grows regardless of commits.

**`trigger: none`** (line 354) is the companion: this pipeline should never run on a push, only on its
schedule.

**Why the others fail** — B batches CI triggers, C disables CI triggering, D is not a schedule
property. This is the same pair as Challenge 22 Q2.

---

## Q15

Which GitHub CLI pattern deletes untagged container versions in GHCR?

- A. Filter versions where `metadata.container.tags | length == 0` and DELETE by id
- B. `gh release delete`
- C. `gh api --method DELETE /repos/{owner}/{repo}/actions/artifacts`
- D. `docker rmi`

### Answer: A

**In `challenge-36.md`:** lines **297–301**.

```bash
gh api /orgs/contoso/packages/container/contoso-api/versions \
  --paginate \
  --jq '.[] | select(.metadata.container.tags | length == 0) | .id' | \
  xargs -I {} gh api --method DELETE /orgs/contoso/packages/container/contoso-api/versions/{}
```

**`tags | length == 0` is the GHCR equivalent of ACR's `--untagged`** — same concept, expressed as a
jq filter because GHCR has no purge command.

**`--paginate` matters:** without it you get only the first page, so a repository with hundreds of
versions is barely touched and the script appears to have worked.

**Why the others fail** — B deletes releases, C deletes **Actions artifacts** (a different store),
D removes a local image.

---

## Q16

Which Azure DevOps setting controls how long pull request runs are kept?

- A. Days to keep pull request runs
- B. Days to keep artifacts
- C. Minimum days to keep
- D. Days to keep runs with release artifacts

### Answer: A

**In `challenge-36.md`:** lines **66–70**.

```text
# - Days to keep artifacts: 30 (default)
# - Minimum days to keep: 1
# - Days to keep pull request runs: 10
# - Days to keep runs with release artifacts: 365
```

**Four separate dials, and knowing which does what is the exam question.** PR runs are the highest
volume and shortest value — the pull request is merged or closed within days — so they get the
shortest tier.

**"Minimum days to keep"** is a **floor**: a run cannot be deleted before this, even by a manual
cleanup. It protects against a misconfigured policy wiping today's builds.

**Why the others fail** — B is the general default, C is the floor, D is the release tier.

---

# Section B — Multiple answer

---

## Q17

Which **three** storage areas does Contoso's retention strategy address? (Choose three.)

- A. Pipeline run artifacts
- B. Azure Artifacts package feed
- C. Container images in ACR
- D. Source code repositories
- E. Work item attachments
- F. Wiki pages

### Answer: A, B, C

**In `challenge-36.md`:** the scenario at lines **20–23**.

```text
- Pipeline run artifacts (build outputs, test results): 200 GB
- Azure Artifacts feed (npm packages): 180 GB (including 3 years of pre-release versions)
- Container images in ACR: 120 GB
```

**Each needs a different mechanism**, which is why the challenge has seven tasks:

| Store | Mechanism |
|---|---|
| Pipeline artifacts | Retention policy + leases |
| Package feed | `countLimit` + views |
| Container images | `acr purge` |

**Why the others fail** — D, E and F consume storage and are not part of this strategy. Git history is
not something you prune on a schedule.

---

## Q18

Which **two** are needed for a retention lease to work? (Choose two.)

- A. An `Authorization: Bearer $(System.AccessToken)` header
- B. A request body that is a JSON **array**
- C. `protectPipeline: true`
- D. A personal access token stored as a secret
- E. `daysValid` under 30

### Answer: A, B

**In `challenge-36.md`:** Break & fix Exercise 1, lines **573–606**.

**Why the others fail**

- **C** — a real property, and `false` is what the production example uses (line 234). It protects the
  **pipeline definition**, not the run's artifacts
- **D** — `$(System.AccessToken)` is the pipeline's built-in OAuth token. Storing a PAT is the
  less-secure alternative, and the same identity-over-stored-secret principle as everywhere else
- **E** — the whole point is a **long** value, 365

**Note the failure is silent.** Both defects produce a green pipeline and an artifact that vanishes a
month later.

---

## Q19

Which **two** correctly tier GitHub Actions artifact retention? (Choose two.)

- A. `retention-days: 3` for pull request builds
- B. `retention-days: 365` for release branch builds
- C. `retention-days: 400` for everything
- D. No `retention-days`, relying on the 90-day default
- E. `retention-days: 0` for CI builds

### Answer: A, B

**In `challenge-36.md`:** lines **131–155**.

**Why the others fail**

- **C** — the maximum applied to everything. PR artifacts nobody will ever download, kept 400 days
- **D** — **the status quo that causes the problem.** 90 days is the default, and it is why untouched
  repositories accumulate storage without anyone deciding to
- **E** — not a valid retention value

**The selection is by branch** (lines 132, 141, 150), so the tier follows the artifact's purpose
automatically rather than depending on anyone remembering.

---

## Q20

Which **two** protect a package version from feed cleanup? (Choose two.)

- A. Promoting it to the `@release` view
- B. Being within the newest `countLimit` versions
- C. Deleting the `@prerelease` view
- D. Setting `packageTypes` to an empty list
- E. Publishing it as a pre-release

### Answer: A, B

**In `challenge-36.md`:** lines **309–314** and **330**.

**A is the deliberate protection**; the policy scopes itself to `["@prerelease"]` (line 333), so
promoted versions are simply out of scope.

**B is automatic** — the newest five survive by definition. Note `daysToKeepRecentlyCreatedPackages: 30`
is a third protection, covering versions that are both old-ranked and recently published.

**Why the others fail**

- **C** — deleting a view does not protect its contents
- **D** — would make the policy apply to nothing, which is disabling cleanup rather than protecting a
  version
- **E** — pre-release is exactly what the policy targets

---

## Q21

Which **two** make an ACR purge safe for production? (Choose two.)

- A. `--untagged` so only unreferenced manifests are deleted
- B. A narrow `--filter` such as `contoso-api:sha-.*`
- C. `--ago 1d` for faster cleanup
- D. Removing `--keep`
- E. Purging every tag older than 7 days

### Answer: A, B

**In `challenge-36.md`:** Break & fix Exercise 2, lines **630–634**.

**Age alone is not a safety signal.** A stable release image tagged `v1.4.2` from six months ago may
be exactly what production runs. `--untagged` restricts deletion to manifests nothing references;
the filter restricts it to the CI-generated commit tags.

**Why the others fail**

- **C** — makes it more aggressive, not safer
- **D** — `--keep 10` (line 423) is a guard; removing it deletes more
- **E** — the exact broken command that deleted production images

---

## Q22

Which **two** are true about GitHub Releases as a retention mechanism? (Choose two.)

- A. They are exempt from Actions artifact retention policies
- B. They persist until explicitly deleted
- C. They expire after 400 days
- D. They are automatically created for every build
- E. They cannot hold binary assets

### Answer: A, B

**In `challenge-36.md`:** lines **250–251**.

**The consequence is that they need their own lifecycle.** Line 257 prunes old **pre-releases**,
keeping the last five, while stable releases are untouched:

```bash
gh release list --json tagName,isPrerelease --jq '.[] | select(.isPrerelease) | .tagName' | \
  tail -n +6 | xargs -I {} gh release delete {} --yes --cleanup-tag
```

**`--cleanup-tag` deletes the git tag too**, which matters — otherwise you accumulate tags pointing at
releases that no longer exist.

**Why the others fail** — C is the *artifact* maximum, D requires an explicit `gh release create`,
E is false (line 246 attaches `dist/*.zip`).

---

## Q23

Which **two** should a cleanup pipeline configure? (Choose two.)

- A. `schedules` with `always: true`
- B. `trigger: none`
- C. `trigger` on every push to main
- D. `always: false` to skip when nothing changed
- E. A `pr` trigger

### Answer: A, B

**In `challenge-36.md`:** lines **347–355**.

**They are complementary.** `always: true` runs the cleanup whether or not code changed — storage grows
independently of commits. `trigger: none` stops it running on pushes, which would be pointless and
would compete for agents with real builds.

**Why the others fail**

- **C** and **E** — cleanup on every push wastes agent time and, at worst, deletes artifacts a
  concurrent run is still using
- **D** — the inverse of the requirement. A week with no commits is still a week of accumulated
  storage

---

# Section C — Repeated scenario

**Scenario:** Contoso must reduce 500 GB of storage while guaranteeing production release artifacts
are kept for 1 year and dev/test artifacts for 7 days, without deleting anything production depends
on.

---

## Q24

**Proposed solution:** Set the project default to 7 days for CI runs and 10 for PR runs, create a
365-day retention lease on runs that deploy to production, cap the feed at 5 versions per package
scoped to `@prerelease`, and schedule an ACR purge of untagged manifests keeping the last 20 tags.

Does this meet the goal? **Yes**

### Answer: Yes

**In `challenge-36.md`:** lines **66–70**, **219–239**, **328–341**, **426–433**.

| Requirement | Mechanism |
|---|---|
| Dev/test 7 days | Project default retention |
| Production 1 year | Retention lease with `daysValid: 365` |
| Reduce feed storage | `countLimit: 5` on `@prerelease` only |
| Reduce image storage | `acr purge --untagged` + `--keep 20` |
| Delete nothing in use | Promoted `@release` versions and tagged images are out of scope |

**The last row is what makes it a Yes.** Every deletion path is scoped to something nothing references
— untagged manifests and unpromoted pre-releases.

---

## Q25

**Proposed solution:** Set a single 7-day retention policy across the project and schedule a daily ACR
purge of everything older than 7 days.

Does this meet the goal? **No**

### Answer: No

**Both halves are actively dangerous.**

**Production artifacts are destroyed.** A 7-day blanket policy deletes the release artifacts
compliance requires for a year. There is no lease and no exception.

**The purge deletes running images.** This is Break & fix Exercise 2 verbatim — without `--untagged`,
`--ago 7d` removes `latest` and `v1.4.2`, which production is pulling. The next pod restart or scale-out
fails to pull its image.

**Storage would fall dramatically**, which is why the proposal is tempting. It achieves the number by
deleting the things that matter.

---

## Q26

**Proposed solution:** Set the project default to 7 days, create a 365-day retention lease on
production runs, cap the feed at 5 versions per package, and schedule an ACR purge with
`--untagged --ago 7d`. Configure the lease with `daysValid: 365` but omit the `Authorization` header
from the REST call.

Does this meet the goal? **No**

### Answer: No

Everything is designed correctly, and the one part that guarantees compliance **silently does not
work**.

Without the header (Break & fix Exercise 1, line 573), the lease call is rejected. The pipeline goes
green, the deployment succeeds, and the run looks retained.

**Thirty days later the production artifact is deleted** — by the 7-day-plus-grace project policy that
is otherwise correct. The failure surfaces a month after the mistake, during an audit or an incident
when someone needs that build.

**The pattern:** a tightened default plus a broken exception is worse than a loose default, because it
looks compliant. Verify the lease exists after deployment rather than assuming the API call worked.

---

# Section D — Yes/No statement grid

---

## Q27 — Azure Pipelines retention

| # | Statement | Answer |
|---|---|---|
| 1 | A retention lease overrides the project policy for one run | **Yes** |
| 2 | "Minimum days to keep" is a floor that manual deletion respects | **Yes** |
| 3 | Pull request runs use the same retention as main-branch runs | **No** |
| 4 | `$(System.AccessToken)` must be passed explicitly as a header | **Yes** |

**In `challenge-36.md`:** lines **86–105**, **68**, **69**, **592**.

Row 3: PR runs have their own dial — 10 days at line 69 — because they are the highest volume and the
shortest-lived value.

Row 4 is Break & fix Exercise 1's first defect. The token exists in the pipeline; it is not applied to
outbound REST calls automatically.

---

## Q28 — GitHub Actions retention

| # | Statement | Answer |
|---|---|---|
| 1 | The default artifact retention is 90 days | **Yes** |
| 2 | The maximum is 400 days | **Yes** |
| 3 | GitHub Releases expire with artifact retention | **No** |
| 4 | `retention-days` can be set per upload | **Yes** |

**In `challenge-36.md`:** lines **113–114**, **250–251**, **137**.

Row 3 is the mechanism behind Q6: Releases are a different store with no expiry, which is what makes
them right for multi-year retention — and what makes them need their own pruning.

Row 1 explains the drift: 90 days is generous, and nobody chooses it. It is simply what happens.

---

## Q29 — package feeds

| # | Statement | Answer |
|---|---|---|
| 1 | `@release` holds promoted versions excluded from cleanup | **Yes** |
| 2 | `countLimit` caps versions kept per package | **Yes** |
| 3 | Recently published versions can be protected regardless of count | **Yes** |
| 4 | Deleting a package version also deletes its downstream consumers' locks | **No** |

**In `challenge-36.md`:** lines **312**, **330**, **331**.

Row 3 is `daysToKeepRecentlyCreatedPackages: 30`, the second guard that stops a burst of releases
evicting something still in use.

Row 4 matters operationally: a lock file referencing a deleted version means every consumer's
`npm ci` fails. That is why the two guards exist and why `@release` promotion is the durable answer for
anything shipped.

---

## Q30 — container registry

| # | Statement | Answer |
|---|---|---|
| 1 | `--untagged` deletes manifests nothing references | **Yes** |
| 2 | `--keep 10` and `--ago 30d` compose as an AND | **Yes** |
| 3 | Age alone is a safe deletion criterion | **No** |
| 4 | `az acr task create` can run a purge on a schedule | **Yes** |

**In `challenge-36.md`:** lines **417**, **420–424**, **616–620**, **427–433**.

Row 3 is Break & fix Exercise 2 stated as a principle, and it is the most transferable idea in the
challenge: **delete by reference, not by age.** A year-old image may be what production runs; a
one-day-old untagged manifest is genuinely garbage.

Row 2 is why the combination is safe — a repository with three tags loses nothing however old they
are.

---

# Section E — Drag and drop

---

## Q31

Match each artifact class to its retention tier.

| Artifact class | Tier |
|---|---|
| Pull request build output | **3–10 days** |
| Main-branch build output | **30 days** |
| Release build output | **365 days** |
| Untagged container manifest | **7 days** |
| Pre-release package version | **Newest 5, or 30 days if recent** |
| Production release asset | **Indefinite — GitHub Release** |

**In `challenge-36.md`:** lines **69**, **146**, **155**, **417**, **330–331**, **250**.

**Volume and value move in opposite directions.** PR artifacts are the most numerous and the least
valuable; release artifacts are rare and must survive an audit. Any single policy is wrong for one end
or the other.

---

## Q32

Match each store to its cleanup mechanism.

| Store | Mechanism |
|---|---|
| Azure Pipelines run artifacts | **Project retention policy + retention leases** |
| GitHub Actions artifacts | **`retention-days` per upload; 90-day default, 400 max** |
| Azure Artifacts feed | **`countLimit` + views, scoped to `@prerelease`** |
| GitHub Packages | **`gh api` DELETE by version id** |
| Azure Container Registry | **`acr purge --untagged --keep N --ago Nd`** |
| Long-term release assets | **GitHub Releases — exempt from retention** |

**In `challenge-36.md`:** lines **66–105**, **113–155**, **328–341**, **292–301**, **415–433**,
**250**.

**Five stores, five mechanisms, no shared setting.** That is why the scenario's 500 GB needed seven
tasks — reducing it is not one policy change.

---

## Q33

Arrange the storage reduction by size of saving, largest first.

**Items:** Container images 120 → 40 GB · Package feed 180 → 50 GB · Pipeline artifacts 200 → 30 GB

### Answer

| Rank | Store | Saving |
|---|---|---|
| 1 | Pipeline artifacts | **170 GB** (85%) |
| 2 | Package feed | **130 GB** (72%) |
| 3 | Container images | **80 GB** (67%) |

**In `challenge-36.md`:** lines **442–446**.

**Total 380 GB, from 500 to 120** — $150/month down to $36, a $114 monthly saving.

**The biggest win is also the easiest.** Pipeline artifact retention is a **settings change** plus
leases on release runs. The feed and registry need cleanup scripts and careful filters. Start where
the ratio of saving to risk is best.

---

## Q34

Arrange the ACR purge safety checks from most to least important.

**Items:** Restrict the tag filter to CI-generated patterns · Add `--untagged` · Set `--keep N` ·
Set `--ago Nd`

### Answer

1. **`--untagged`** — delete only what nothing references
2. **Restrict the filter** — scope to `sha-.*`, not every tag
3. **`--keep N`** — retain the newest tags regardless of age
4. **`--ago Nd`** — the age threshold

**In `challenge-36.md`:** lines **630–634** and **420–424**.

**The order is the order of protection.** `--untagged` alone makes the command almost safe;
`--ago` alone makes it dangerous — which is exactly the Break & fix Exercise 2 failure, where age was
the *only* criterion.

---

## Q35

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Release artifacts deleted after 30 days | **Retention lease failed — missing header or non-array body** |
| Production pods fail to pull their image | **Purge without `--untagged` deleted a tagged image** |
| Storage grows despite a weekly cleanup pipeline | **`always: false`, so the schedule skips quiet weeks** |
| A consumer's `npm ci` fails after cleanup | **A package version still referenced was evicted** |
| GHCR cleanup deletes almost nothing | **`--paginate` omitted, so only the first page is processed** |

**In `challenge-36.md`:** lines **573**, **620**, **352**, **330**, **299**.

**The first and last are both silent.** A failed lease and a single-page cleanup both leave a green
pipeline and an unchanged outcome — you discover them by measuring storage, not by reading logs.

---

# Section F — Hot area

---

## Q36

```text
Project Settings > Pipelines > Retention
- Days to keep artifacts: [BLANK 1]
- Days to keep pull request runs: [BLANK 2]
- Days to keep runs with release artifacts: [BLANK 3]
```

- **BLANK 1:** `30` / `1` / `365` / `400`
- **BLANK 2:** `10` / `30` / `90` / `365`
- **BLANK 3:** `365` / `30` / `10` / `1`

### Answer: `30`, `10`, `365`

**In `challenge-36.md`:** lines **67–70**.

Three tiers ordered by value: PR runs shortest, general artifacts in the middle, release artifacts
longest. `400` is the **GitHub** maximum, offered here as a cross-platform distractor.

---

## Q37

```powershell
      $headers = @{
        [BLANK 1] = "Bearer $(System.AccessToken)"
        "Content-Type" = "application/json"
      }
      $body = ConvertTo-Json [BLANK 2](
        @{
          daysValid = [BLANK 3]
          definitionId = $(System.DefinitionId)
          runId = $(Build.BuildId)
        }
      )
```

- **BLANK 1:** `Authorization` / `Token` / `ApiKey` / `Bearer`
- **BLANK 2:** `@` / `$` / `%` / *(omit)*
- **BLANK 3:** `365` / `30` / `7` / `1`

### Answer: `Authorization`, `@`, `365`

**In `challenge-36.md`:** lines **592–604**.

Both blanks 1 and 2 are Break & fix Exercise 1's defects. `ConvertTo-Json @( ... )` produces a JSON
**array**, which the leases API requires — omitting `@(` sends a single object and the call fails.

---

## Q38

```yaml
      - name: Upload release artifact
        if: startsWith(github.ref, '[BLANK 1]')
        uses: actions/upload-artifact@v4
        with:
          name: release-${{ github.ref_name }}-${{ github.sha }}
          path: dist/
          [BLANK 2]: [BLANK 3]
```

- **BLANK 1:** `refs/heads/release/` / `release/` / `refs/tags/` / `main`
- **BLANK 2:** `retention-days` / `expires-in` / `ttl` / `keep`
- **BLANK 3:** `365` / `3` / `90` / `400`

### Answer: `refs/heads/release/`, `retention-days`, `365`

**In `challenge-36.md`:** lines **150–155**.

`github.ref` holds the **full ref** — the same value trap as Challenge 31 Q41. `400` is the maximum
rather than the requirement; compliance asks for one year.

---

## Q39

```json
{
  "[BLANK 1]": 5,
  "daysToKeepRecentlyCreatedPackages": 30,
  "packageTypes": ["npm", "NuGet"],
  "views": ["[BLANK 2]"]
}
```

- **BLANK 1:** `countLimit` / `maxVersions` / `versionLimit` / `keepCount`
- **BLANK 2:** `@prerelease` / `@release` / `@local` / `@latest`

### Answer: `countLimit`, `@prerelease`

**In `challenge-36.md`:** lines **330** and **333**.

Scoping to `@prerelease` is what protects promoted versions: anything in `@release` is out of the
policy's scope entirely. Targeting `@release` would delete the stable versions consumers depend on.

---

## Q40

```bash
az acr run --registry contosoregistry --cmd "acr purge \
  --filter 'contoso-api:[BLANK 1]' \
  --[BLANK 2] \
  --ago 7d" /dev/null
```

Requirement: remove CI images that nothing references, without touching `latest` or semver tags.

- **BLANK 1:** `sha-.*` / `.*` / `latest` / `v.*`
- **BLANK 2:** `untagged` / `force` / `all` / `dry-run`

### Answer: `sha-.*`, `untagged`

**In `challenge-36.md`:** lines **631–634**.

`.*` is the broken filter that deleted production images. `--untagged` is the primary safety flag —
delete by reference, not by age.

---

## Q41

```yaml
schedules:
  - cron: "[BLANK 1]"
    displayName: "Weekly artifact cleanup"
    branches:
      include: [main]
    [BLANK 2]: true

[BLANK 3]: none
```

Requirement: run every Sunday at 02:00 whether or not code changed, and never on a push.

- **BLANK 1:** `0 2 * * 0` / `2 0 * * 0` / `0 2 * * *` / `0 2 0 * *`
- **BLANK 2:** `always` / `batch` / `enabled` / `force`
- **BLANK 3:** `trigger` / `pr` / `schedule` / `pool`

### Answer: `0 2 * * 0`, `always`, `trigger`

**In `challenge-36.md`:** lines **348–354**.

Minute, hour, day-of-month, month, day-of-week — so `0 2` is 02:00 and `0` in the fifth field is
Sunday. `always: true` is essential: a cleanup that skips quiet weeks defeats its own purpose.

---

# Section G — Case study

## Case study: Contoso storage reduction

### Background

Contoso's Azure DevOps project holds **500 GB**, growing **50 GB per month**, costing **$150/month**
at ~$0.30/GB.

| Store | Size |
|---|---|
| Pipeline run artifacts | 200 GB |
| Azure Artifacts npm feed | 180 GB, including 3 years of pre-release versions |
| Container images in ACR | 120 GB |
| Release artifacts | retained indefinitely, growing unchecked |

### Requirements

**Compliance**

- Production release artifacts must be retained for **1 year**
- Dev and test artifacts need only **7 days**
- Nothing currently deployed may be deleted

**Cost**

- Monthly storage spend must fall below **$50**
- Growth must be bounded, not merely reduced once

**Operations**

- Cleanup must run without manual intervention
- The team must be able to keep a specific build beyond the default

---

## Q42

Which **two** meet the tiered compliance requirement? (Choose two.)

- A. A short project default with a 365-day retention lease on production runs
- B. `retention-days` set per upload by branch in GitHub Actions
- C. A single 365-day policy for all runs
- D. A single 7-day policy for all runs
- E. Manually downloading release artifacts to a file share

### Answer: A, B

**In `challenge-36.md`:** lines **66–105** (Azure DevOps) and **131–155** (GitHub).

**Both express the same idea on the two platforms:** default short, extend the exceptions.

**Why the others fail**

- **C** — keeps 200 GB of CI artifacts for a year. Cost requirement broken
- **D** — deletes the production artifacts compliance requires
- **E** — unauditable, manual, and it will be forgotten. Compliance needs a guarantee

---

## Q43

Which configuration reduces the 180 GB feed without breaking consumers?

- A. `countLimit: 5` scoped to `@prerelease`, with promoted versions in `@release`
- B. `countLimit: 5` applied to all views
- C. Deleting every package older than 30 days
- D. Deleting the feed and recreating it

### Answer: A

**In `challenge-36.md`:** lines **309–314** and **328–341**.

**Why the others fail**

- **B** — would evict stable versions consumers reference in their lock files. Their next `npm ci`
  fails, and the failure appears in *their* pipeline, not yours
- **C** — age says nothing about use. A stable dependency published two years ago may be in every
  lock file in the organisation
- **D** — destroys everything, including versions in active use

**`daysToKeepRecentlyCreatedPackages: 30`** is the second guard, protecting a version that is both
outside the newest five and recently published.

---

## Q44

Which **two** safely reduce the 120 GB of container images? (Choose two.)

- A. `acr purge --untagged --ago 7d`
- B. `acr purge --keep 20 --ago 90d`
- C. `acr purge --ago 7d` with no other flags
- D. `az acr repository delete` for old repositories
- E. Reducing the ACR SKU

### Answer: A, B

**In `challenge-36.md`:** lines **415–424** and **430–431**.

**Two tiers again**, chained in the scheduled task at line 430: aggressive on untagged manifests,
conservative on tagged ones.

**Why the others fail**

- **C** — Break & fix Exercise 2. Deletes `latest` and `v1.4.2` while production is using them
- **D** — deletes whole repositories, including current images
- **E** — **the tempting one.** A lower SKU does not reduce storage; it reduces included storage,
  throughput and features — and may increase the overage bill. It also loses geo-replication and
  content trust (Challenge 28)

---

## Q45

Which configuration bounds **future** growth rather than reducing storage once?

- A. A scheduled ACR task and a scheduled cleanup pipeline, both with `always: true`
- B. A one-time manual purge
- C. Increasing the storage quota
- D. Archiving old artifacts to blob storage

### Answer: A

**In `challenge-36.md`:** lines **347–355** and **426–433**.

**The requirement says "bounded, not merely reduced once"**, and that word is what selects the answer.
A one-time cleanup takes 500 GB to 120 GB, and at 50 GB a month it is back to 500 within eight months.

`always: true` matters here specifically: a cleanup that only runs after code changes would skip quiet
periods, which are exactly when nobody notices growth.

**Why the others fail**

- **B** — a one-time fix for a continuous problem
- **C** — pays for the growth
- **D** — **plausible and worth stating.** Moving artifacts to cheap blob storage does lower cost per
  gigabyte, and it does nothing about growth — you now have an unbounded, cheaper pile, plus a
  restore process nobody has tested

---

## Q46

Which mechanism lets a team keep one specific build beyond the default?

- A. A retention lease created via the REST API
- B. Renaming the artifact
- C. Changing the project default temporarily
- D. Re-running the pipeline monthly

### Answer: A

**In `challenge-36.md`:** lines **86–105**.

```powershell
              daysValid = 365
              runId = $(Build.BuildId)
```

**Per-run, not per-policy.** The lease names a specific `runId`, so the project default stays short
while one build survives.

**Why the others fail**

- **B** — the name has no bearing on retention
- **C** — changing the default affects **every** run, and changing it back does not re-protect
  anything already past the new threshold
- **D** — re-running produces a **new** build, not the one being preserved. The original artifact is
  still deleted, and the rebuilt one may not even be identical

---

## Q47

Six months after implementing retention, storage sits at 130 GB — but 90 GB of it is pipeline
artifacts, despite a 30-day policy. Investigation finds hundreds of runs with retention leases created
by a deployment pipeline that runs on every merge to main.

What went wrong?

- A. The lease is applied to every main-branch run, not only production deployments
- B. The project policy is not being enforced
- C. The leases have expired but artifacts remain
- D. The 30-day policy is too long

### Answer: A

**In `challenge-36.md`:** compare the **conditional** release job at line **78** with the production
deployment lease at lines **219–239**.

```yaml
  - job: Release
    condition: startsWith(variables['Build.SourceBranch'], 'refs/heads/release/')
```

**The lease must be as selective as the tier it grants.** In the challenge it sits inside the
`DeployProd` stage, which is itself gated on `refs/heads/main` **and** success of the dev stage
(line 207). Applying it to every merge grants 365-day retention to every CI build.

**The symptom is diagnostic:** hundreds of leases where you expected a handful of production releases.
Count them against your actual release cadence.

**Why the others fail**

- **B** — the policy **is** enforced; leases legitimately override it
- **C** — an expired lease stops protecting, so the artifact would be deleted
- **D** — 30 days is not the problem; the exception is

---

## Q48

A year on, an auditor asks for the artifact from a production release deployed 11 months ago. The
retention lease was created correctly with `daysValid: 365`. The artifact is gone.

What is the most likely explanation?

- A. `daysValid` counts from run creation, so an 11-month-old run is close to expiry and the lease may
  have been removed or the run deleted by a cleanup script
- B. Retention leases do not work
- C. The auditor is looking in the wrong project
- D. Artifacts cannot be kept longer than 90 days

### Answer: A

**In `challenge-36.md`:** the lease at lines **96–101** and the run-deletion script at lines
**454–470**.

**Two plausible mechanisms, and both are worth knowing.**

**Expiry is measured from the run**, not from when you last needed it. A 365-day lease on a run created
11 months ago has roughly one month left — comfortably inside "gone" if anything shortened it.

**And the cleanup script is the more likely culprit.** Lines 456–470 delete old **pipeline runs**,
keeping the last 100:

```bash
  --query "sort_by([],&id)|[:-100].id"
```

That script does not check for leases. A pipeline running 20 times a day passes 100 runs in a week, so
an 11-month-old run was deleted long before its lease expired — **deleting the run deletes its
artifacts regardless of retention**.

**The lesson, and it generalises:** a retention guarantee is only as strong as every deletion path that
can reach the data. Automation you wrote yourself is not bound by the policy that protects it.

**What Contoso should do:** exclude leased runs from the cleanup script, and for genuine multi-year
compliance move release artifacts somewhere outside the pipeline's lifecycle — a GitHub Release
(line 250) or immutable blob storage.

**Why the others fail** — B is contradicted by the design, C is not a technical answer, D is the
GitHub *default*, not a cap on Azure DevOps.

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **One policy for every artifact class** | Q1, Q19, Q25, Q42 | Tier by value: PR days, main weeks, release a year |
| **Purging by age alone** | Q10, Q21, Q30, Q44 | `--untagged` first. Delete by reference, not age |
| **Retention lease failing silently** | Q5, Q18, Q26 | Auth header **and** a JSON array body. Verify it exists |
| **Lease applied too broadly** | Q47 | Scope it to production deployments, not every merge |
| **Cleanup script ignoring leases** | Q48 | Deleting a run deletes its artifacts regardless |
| **`always: false` on a cleanup schedule** | Q23, Q35, Q45 | Storage grows in quiet weeks too |
| **90-day default left unchanged** | Q19, Q28 | It is generous, and nobody chose it |
| **Feed cleanup scoped to all views** | Q20, Q43 | Scope to `@prerelease`; promote to `@release` to protect |
| **`--paginate` omitted** | Q15, Q35 | Only the first page is processed |
| **Reducing the SKU to cut cost** | Q44 | It cuts features, not gigabytes |
| **Archiving mistaken for bounding** | Q45 | Cheaper storage still grows |
| **Releases assumed to expire** | Q6, Q22, Q28 | They persist until deleted — and need their own pruning |

---

# The blocks to memorise

Line numbers are in `challenge-36.md`.

```text
# 1. Azure DevOps retention dials  (lines 67-70)
Days to keep artifacts                  30   (default)
Minimum days to keep                     1   (floor)
Days to keep pull request runs          10
Days to keep runs with release artifacts 365

# 2. GitHub Actions artifact retention  (lines 113-114)
Default 90 days | Maximum 400 days | per-upload override with retention-days
GitHub Releases: NOT subject to retention - persist until deleted
```

```powershell
# 3. Retention lease - both defects fixed  (lines 591-606)
      $headers = @{ Authorization = "Bearer $(System.AccessToken)"
                    "Content-Type" = "application/json" }
      $body = ConvertTo-Json @( @{        # MUST be an array
          daysValid = 365
          definitionId = $(System.DefinitionId)
          ownerId = "User:$(Build.RequestedForId)"
          protectPipeline = $false
          runId = $(Build.BuildId)
      } )
      POST $(System.CollectionUri)$(System.TeamProject)/_apis/build/retention/leases?api-version=7.1
```

```yaml
# 4. Tiered GitHub retention  (lines 131-155)
          retention-days: 3     # PR
          retention-days: 30    # main
          retention-days: 365   # release/**

# 5. Scheduled cleanup  (lines 347-355)
schedules:
  - cron: "0 2 * * 0"
    always: true          # run even with no changes
trigger: none
```

```json
// 6. Feed retention  (lines 328-341)
{ "countLimit": 5,
  "daysToKeepRecentlyCreatedPackages": 30,
  "views": ["@prerelease"] }        // @release is protected by promotion
```

```bash
# 7. ACR purge - safe form  (lines 415-433, 630-634)
acr purge --filter 'contoso-api:sha-.*' --untagged --ago 7d      # unreferenced
acr purge --filter 'contoso-api:.*'     --keep 20   --ago 90d    # tagged, conservative
az acr task create --name purge-old-images --schedule "0 3 * * *" --context /dev/null

# 8. The numbers  (lines 440-446)
500 GB @ $0.30/GB = $150/month
  artifacts 200 -> 30 | feed 180 -> 50 | images 120 -> 40
120 GB = $36/month   ->  saving $114/month = $1,368/year
```

**Views:** `@local` (everything), `@prerelease` (cleaned up), `@release` (promoted, protected).

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 36 is exam-ready. Move to Challenge 37 |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 1 and 5, then retake this |
| Below 30 | Redo the challenge, writing the four Azure DevOps retention dials from memory |

Record your result in `AZ-400-Learning-Log.md` under Challenge 36.

:::tip The one thing

**Delete by reference, not by age — and tier everything.**

Untagged manifests, unpromoted pre-releases and expired PR artifacts are safe to remove because
nothing points at them. A year-old production image is not safe to remove just because it is old.

:::
