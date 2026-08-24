---
sidebar_position: 5
toc_max_heading_level: 2
title: "Final revision: all 51 challenges on one page"
sidebar_label: "Final Revision (all 51)"
---

# AZ-400 final revision — everything, on one page

:::tip How to use this page

**Do not read it like a book.** You already tried that, and tomorrow it feels new again. That is normal — long text does not build recall.

Do this instead:

1. **Read Part 1 and Part 2 first.** They are short. Most exam questions are one of those rows.
2. **Then one domain a day** from Part 4. Each challenge is one small block.
3. For each block: read it, **cover it, say the bold line out loud**. That is the whole revision.
4. On exam morning read **Part 7** (how to answer) and **Part 8** (the 12 laws). Nothing else.

You do not need to remember the whole challenge. You need **the bold line** plus **"does this inform, or does it block?"**

:::

---

# Part 1 — Confusion pairs

**This is the highest-scoring page in the document.** AZ-400 rarely asks "what is X". It gives you two things that both sound right and asks which one fits. Learn the **separator**, not the definitions.

| A | B | The separator |
|---|---|---|
| **Blue-green** | **Canary** | Instant rollback → blue-green. Validate on real traffic → canary |
| **Lead time** | **Deployment frequency** | A **duration** vs a **rate** |
| **Cycle time** | **Lead time** | Work started → done, vs commit → production |
| **Git LFS** | **Scalar / partial clone** | Large **files** vs large **history** |
| **`--depth=1`** | **`--filter=blob:none`** | Cuts **history** (breaks blame) vs cuts the **download** |
| **Rebase** | **Merge** | **Rewrites** SHAs vs **records** both histories |
| **`AB#1234`** | **`Fixes AB#1234`** | Links vs links **and moves** the work item |
| **Field** | **View** | Stores data vs **displays** it |
| **SemVer** | **CalVer** | Communicates **compatibility** vs **age** |
| **Dependabot alerts** | **Dependabot version updates** | Immediate on a CVE vs **scheduled** |
| **CodeQL** | **Dependency scanning** | Code **you wrote** vs code **you imported** |
| **Approval** | **Gate** | Asks a **person once** vs asks a **system repeatedly** |
| **Wait timer** | **Gate `period`** | A fixed delay, once vs a re-evaluation interval |
| **Environments** | **Branch protection** | Gate **deployments** vs gate **merges** |
| **Line coverage** | **Branch coverage** | Line **ran** vs **both sides** of the `if` were taken |
| **Threshold** | **Diff coverage** | Met by history vs makes **this author** write a test |
| **Managed identity** | **Workload identity federation** | **Only on Azure compute** vs from a **trusted OIDC issuer** |
| **`AADSTS70021`** | **`AuthorizationFailed`** | Subject mismatch vs **missing RBAC role** |
| **`GITHUB_TOKEN`** | **GitHub App token** | This **repository only** vs the **installation's** repos |
| **403** | **404** | Reached and **refused** vs **outside the scope** (invisible) |
| **Key Vault Secrets User** | **Key Vault Reader** | Reads **values** vs metadata **only** |
| **Push protection** | **Secret scanning** | **Blocks** entry vs **alerts** after entry |
| **Posture (Defender)** | **Findings** | A **configuration** vs a **vulnerability** |
| **`Microsoft.Targeting`** | **`Microsoft.Percentage`** | **Sticky per user** vs random per evaluation |
| **Liveness probe** | **Readiness probe** | **Restarts** the container vs **removes it** from the Service |
| **`AcrPull`** | **`AcrPush`** | Read vs read **and write** |
| **`${{ }}`** | **`$[ ]`** | **Compile** time vs **runtime** (crosses stages) |
| **`always()`** | **`succeededOrFailed()`** | Includes **skipped and cancelled** vs does not |
| **`fail-fast: false`** | **`continue-on-error`** | Every leg **finishes honestly** vs a failing leg turns **green** |
| **`status`** | **`conclusion`** | Finished vs **how** it finished |
| **Composite action** | **Reusable workflow** | **Steps**, caller's runner vs **jobs**, own runner |
| **`dependsOn` chained** | **`dependsOn` shared** | **Serial** vs **parallel** |
| **Repository Insights** | **Projects Insights** | The **code** vs the **work items** |
| **Application Insights** | **VM Insights** | The **code** vs the **machine** |
| **Smart detection** | **Dynamic threshold alert** | On by default, **notifies only** vs must be **created**, can act |
| **`join kind=inner`** | **`join kind=leftanti`** | In **both** vs **only new** |

---

# Part 2 — Keyword → answer

When the question contains the phrase on the left, the answer is almost always on the right.

| The question says | Go straight to |
|---|---|
| "without storing a credential" / "no secret" | **OIDC / workload identity federation** |
| "runs on an Azure VM" and needs Azure access | **Managed identity** |
| "on-premises" / "outside Azure" / "cannot use OIDC" | **Service principal + secret** |
| "prevent the merge" | A failing job **+ a required status check** |
| "prevent the deployment" | **Environment** protection rules |
| "must not be bypassable by developers" | **Server-side**: push protection, branch policy, `enforce_admins` |
| "we still support version 2.x" | **Release branching / GitFlow** |
| "several deployments a day" | **Trunk-based** (CD + feature flags) |
| "instant rollback" | **Blue-green** (slot swap) |
| "test with a small percentage of real users" | **Canary** |
| "turn it off without redeploying" | **Feature flag** |
| "the previous version must stay available" | Slots, revisions, rings — **and never for the database** |
| "no code changes" (monitoring) | **Auto-instrumentation** + restart |
| "trace one request across services" | `operation_Id` + **`traceparent` propagated at every hop** |
| "how urgently should we act?" | **Error budget burn rate** |
| "one pane of glass across GitHub and Azure DevOps" | **Defender for Cloud DevOps Security** |
| "which setting is not enabled?" | **Defender posture** |
| "SQL injection" / "hard-coded value in code" | **CodeQL** |
| "vulnerable package", incl. transitive | **Dependabot / dependency scanning** |
| "CVE inside the built image" | **Container scan (Trivy)** |
| "out-of-date base image tag in the Dockerfile" | **Dependabot (docker ecosystem)** |
| "must be visible on the pull request" | An **annotation / comment** — and remember it does **not** block |
| "audit / traceable to a work item" | `Fixes AB#`, work-item-linking policy, audit log |
| "the metric must not be gamed by one slow outlier" | **Median / percentile**, not mean |
| "why is coverage 0% when tests pass?" | **Plumbing** — reporter, path, collector, or source maps |
| "the pipeline is stuck / Waiting" | An unmet **gate**: criteria, missing reviewer team, or a required post-merge check |
| "this passes even though the data is missing" | A **fail-open** guard — make missing evidence a failure |

---

# Part 3 — Where the marks are

| Domain | Weight | Challenges | Your status |
|---|---|---|---|
| 3 · Build and release pipelines | **50–55%** | 13–38 | Strong in mocks — protect it |
| 1 · Processes and communications | 10–15% | 1–6 | Was weak, now fixed |
| 2 · Source control | 10–15% | 7–12 | Mid |
| 4 · **Security and compliance** | 10–15% | 39–45 | **Your weakest — read twice** |
| 5 · Instrumentation | 5–10% | 46–50 | Strong, and cheap marks |

**Pass = 700/1000.** Domain 3 alone is over half the paper, so **challenges 13–38 are worth more than everything else combined**. Domain 4 is where your score is leaking.

---


# Part 4 — All 51 challenges, one block each

Read one block. Cover it. Say the bold line out loud. Move on.

---

## Domain 1 — Processes and communications (10–15%)

---

### 01 · Design flow of work

> **Cadence chooses the branching strategy. The word "GitHub" chooses nothing.**

| What the question says | Answer |
|---|---|
| Ships continuously, only one version live | **GitHub Flow** |
| Packaged product, old versions still supported | **GitFlow** / release branching |
| Ships daily, branches live under a day | **Trunk-based** — needs CD **and** feature flags |

**Also know**

- A check becomes a gate **only** when its name is listed in `contexts`.
- The required-check name must match the **job's** `name:`, not the workflow's. Mismatch = pending forever.
- `enforce_admins` — a rule with an exception list is not a rule.
- `strict` = the PR must be green **against today's `main`**. Green + green can still be red.
- Auto-merge **waits for** conditions. It never removes them.

**Trap:** a PR template, an ADR or an audit log **records**. Only branch protection **prevents**.

---

### 02 · Feedback cycles and work tracking

> **A field stores data. A view displays it. A keyword moves the work item.**

| Write this | It does this |
|---|---|
| `AB#1234` | Links only |
| `Fixes AB#1234` | Links **and moves the item to Done** |
| `#87` | Links a GitHub issue |
| `Fixes #87` | Closes it — **default branch only** |

**Also know**

- Story points must be a `NUMBER` field. A `SINGLE_SELECT` cannot be summed, so velocity breaks.
- Sprint must be an `ITERATION` field — it knows dates. A select is just a list.
- `@CurrentIteration` is a **macro**. A hard-coded sprint path is stale next sprint.
- Area path: use `UNDER`, not `=`. `=` misses every sub-area.
- Azure Boards linking needs **two halves**: the GitHub App installed **and** the repo connected in Azure DevOps. One alone fails silently.

**Trap:** CODEOWNERS **requests** a reviewer. Branch protection **requires** one. A CODEOWNERS team with no repo access is ignored with no error.

---

### 03 · Source, bug and quality traceability

> **Traceability is a chain. Every question asks which link is missing.**

```text
bug -> work item -> pull request -> commit SHA -> build -> deployment
      TEXT joins the top          THE SHA joins the bottom
```

**Conventional Commits**

| Type | Version bump |
|---|---|
| `feat` | MINOR |
| `fix` | PATCH |
| `perf` | PATCH — the one people miss |
| `docs` `style` `refactor` `test` `chore` `ci` | none |
| `feat!:` or a `BREAKING CHANGE:` footer | MAJOR |

**Also know**

- There is **no `breaking` type**. It is a `!` or a footer.
- Trace an incident **backwards**: deployment → build → merge commit → PR → work item.
- Anything that reads history needs `fetch-depth: 0`. `0` means unlimited, not shallow.

**Trap:** a git hook is not enforcement — `--no-verify`, fresh clones and web commits all skip it.

---

### 04 · DevOps metrics and dashboards

> **Four DORA metrics, two pairs. Throughput trades against stability.**

| Pair | Metrics |
|---|---|
| **Throughput** | Deployment frequency (a **rate**) · Lead time for changes (a **duration**) |
| **Stability** | Change failure rate · MTTR |

| | Elite | High | Medium |
|---|---|---|---|
| Deploy frequency | **multiple per day** | daily–weekly | weekly–monthly |
| Lead time | **under 1 hour** | 1 day–1 week | 1 week–1 month |
| MTTR | **under 1 hour** | under 1 day | 1 day–1 week |
| Change failure rate | **0–15%** | 16–30% | 31–45% |

**Also know**

- Elite change failure rate is **not 0%**.
- A build failing in CI is **not** a change failure. A change failure degrades **service** and needs remediation.
- Use the **median**, not the mean — one stale outlier moves a mean.
- Environment names are **case-sensitive**. Wrong case returns a silent zero.

**Trap:** burndown, velocity and cycle time are **flow** metrics, not DORA.

---

### 05 · Documentation and collaboration

> **Grade every doc option by one question: how does this stay true?**

```text
1  GENERATED from source   OpenAPI from code, changelog from commits  - cannot drift
2  DOCS-AS-CODE            Markdown + Mermaid in the repo, reviewed in the PR
3  WIKI                    versioned, but separate from the code - drifts quietly
4  FILE ON A SHARE         Visio / Word - already wrong
```

**Also know**

- Mermaid fence: **GitHub** uses triple backticks; **Azure DevOps wiki** uses `::: mermaid`.
- Provisioned wiki = no built-in PR review. **Code wiki** lives in your repo and gets normal PRs.
- **Labels** feed release-drafter. **Commit types** feed conventional-changelog.
- release-drafter maintains a **draft**. It does not publish.
- Mermaid wins because it is **text** — a reviewer can see the diagram did not change.

**Trap:** a quarterly review is not a freshness control. Twelve of them produced a three-year-old diagram.

---

### 06 · Integrations and webhooks

> **Ask: who starts it, and on what event?**

| Direction | Mechanism | Auth |
|---|---|---|
| Platform → you (outbound) | **webhook** | HMAC **signature** |
| Outside → GitHub (inbound) | **`repository_dispatch`** | token, carries `client_payload` |
| A person clicks Run | **`workflow_dispatch`** | declared inputs |
| Both ways | Azure Boards ↔ GitHub | the installed GitHub App |

**Also know**

- Verify `X-Hub-Signature-256` with a **constant-time** compare. `===` leaks the digest by timing.
- An "unguessable URL" is not security — URLs sit in logs and config.
- `insecure_ssl: "0"` means **verify**. `"1"` disables it.
- Deployment status has `state`. A workflow run has `conclusion`.
- Use `PATCH` to update a webhook. `POST` creates a second one and the old secret stays live.

**Trap:** subscribing to every event. Volume is exactly why the current alerts get ignored.

---

## Domain 2 — Source control (10–15%)

---

### 07 · Branching strategies

> **How often do they ship? And do they still support a shipped version?**

| | Trunk-based | GitHub Flow | Release branching |
|---|---|---|---|
| Cadence | daily / weekly | every 1–2 weeks | monthly / quarterly |
| CD | **required** | recommended | optional |
| Feature flags | **yes** | optional | no |
| **Supports old versions** | no | no | **YES** ← the decider |
| Rollback | flag off | revert commit | deploy prior release |

**Also know**

- No feature flags → **not** trunk-based. Both CD and flags are hard prerequisites.
- **Rebase rewrites; merge records.** Never rebase a branch anyone else has.
- Fix on a release branch → **cherry-pick** it back to `main`. Never merge the branch back.
- Skip the cherry-pick and the bug **returns** in the next release.
- Drift is caused by **time**, not size.

**Trap:** a shallow checkout in a drift-measuring job makes every count `0` — green and wrong.

---

### 08 · Pull request workflows

> **A required status check matches the JOB's `name:`. Get it wrong and the PR is pending forever.**

| GitHub | Azure Repos |
|---|---|
| Required approving reviews | `az repos policy approver-count create` |
| Dismiss stale reviews on push | `--reset-on-source-push true` |
| Author self-approval | `--creator-vote-counts false` |
| Required status checks | `az repos policy build create` |
| Required conversation resolution | `az repos policy comment-required create` |
| *(no equivalent)* | `az repos policy work-item-linking create` |

**Also know**

- Every Azure Repos policy needs `--blocking true`, or it is **advisory only**.
- CODEOWNERS is **last match wins** (like `.gitignore`). General first, specific last.
- The winning CODEOWNERS pattern **replaces** the owners; it does not add to them.
- A merge queue needs a `merge_group` workflow, or the queue waits for checks nobody runs.
- `--valid-duration` is in **minutes**. 720 = twelve hours.

**Trap:** size labels and PR templates are information, never enforcement.

---

### 09 · Repository management

> **Pick the narrowest role. Write pushes code; Maintain manages the repository.**

| Role | Can do | API value |
|---|---|---|
| Read | clone, view | **`pull`** |
| Triage | manage issues/PRs, no code | `triage` |
| Write | push to unprotected branches | **`push`** |
| Maintain | repo settings — **not** visibility, delete or access | `maintain` |
| Admin | everything | `admin` |

**Also know**

- `git tag -a` = **annotated** — has a tagger, date, message, can be signed. Plain `git tag` = a pointer.
- Only **annotated** tags are found by `git describe`.
- `git describe` returns `` `<tag>-<commits-ahead>-g<short-sha>` `` and picks the **nearest** tag, not the newest.
- A shallow clone makes `git describe` fall back to `v0.0.0` — green and wrong.
- Azure Repos **Contribute** = GitHub **Write**. Azure Repos also has **path-level** security, GitHub does not.

**Trap:** `.gitignore` controls what is **tracked**, never who has **access**.

---

### 10 · Large file management

> **Large FILES → Git LFS. Large HISTORY → Scalar. They are not alternatives.**

**Also know**

- LFS puts a small **pointer** in Git and the bytes on an LFS server.
- `.gitattributes` **is** the configuration and it **must be committed** — otherwise other clones commit raw binaries.
- `git lfs install` is **per machine**. Without it, that machine commits a 200 MB blob silently.
- Use `-text` in the attribute line. Plain `text` turns on CRLF conversion and **corrupts binaries**.
- Migration **rewrites history** → force push, everyone re-clones. Space is only freed after `reflog expire` + `gc --prune=now`.
- **Locking exists because binaries cannot be merged.** One side is simply discarded.
- Storage is paid once; **bandwidth is paid on every clone**.

**Trap:** `--depth 1` limits history. It does nothing about file size.

---

### 11 · Git advanced operations

> **Rotate first. Rewrite second.**

Removing a secret from history does **not** revoke it. Forks, old clones, CI caches and anyone who looked still have it.

| Recover (safe, local, additive) | Remove (rewrites everything) |
|---|---|
| `git reflog` | `git filter-repo` (`filter-branch` is deprecated) |
| `git fsck --no-reflogs` → dangling commits | BFG, on a `--mirror` clone |
| `git branch <name> <sha>` | new SHA for every downstream commit |
| nothing is destroyed | force push `--all` **and** `--tags` |
| reflog is **local** to one machine | every clone deleted and **re-cloned** |

**Also know**

- **Never `git pull` after a rewrite** — you get duplicate history and the secret comes back.
- `--path` without `--invert-paths` keeps **only** that path. Catastrophic typo.
- `git revert` adds an undo commit and leaves the original readable — right for a bad change, wrong for a secret.
- Create the new key **before** deleting the old one, or every consumer breaks.

**Trap:** cherry-pick makes a **new** SHA and the original commit stays where it was.

---

### 12 · Mono-repo vs multi-repo

> **Atomic cross-service change versus independent ownership. Name the cost of whichever you pick.**

| | Mono-repo | Multi-repo |
|---|---|---|
| Wins | one commit changes 15 services; no version drift | ownership, independent releases, small clones |
| Costs | slow clone, everyone triggers everything, **limited per-service permissions** | coordinated PRs, **diamond dependencies**, version drift |

**Four tools, four problems — do not mix them up**

| Tool | Reduces |
|---|---|
| `--filter=blob:none` (partial clone) | the **download** — history stays, blobs come on demand |
| sparse-checkout | the **working tree** only |
| `--depth=1` (shallow) | **history** — breaks blame and bisect |
| **Git LFS** | large **files** — nothing to do with history |

**Also know**

- **Scalar** turns on a bundle: partial clone, FSMonitor, commit-graph, background maintenance.
- `git submodule update` = the **pinned** commit. `--remote` = latest on the branch.
- Clone with `--recurse-submodules`, or you get empty directories.
- Adding any `checkout:` in Azure Pipelines stops the implicit one — you must add `checkout: self`.

**Trap:** splitting a repo to fix a 25-minute clone buys 25 minutes and pays with coordinated PRs forever.

---


## Domain 3a — Package management

---

### 13 · GitHub Packages and Azure Artifacts

> **Feed = where packages live. View = which quality a consumer sees. Upstream = where the feed goes when it doesn't have something.**

**Also know**

- Point every build at **your feed**, list npmjs / nuget.org as **upstreams**. First request fetches and **caches**. Public registry down? Cached packages still work.
- **Internal upstream must be listed FIRST.** Public first = dependency confusion.
- Azure Artifacts views: `@prerelease` → `@release`. **Promotion is the control.** Never publish straight into `release`.
- **GitHub Packages has no views.** It inherits repo and org permissions.
- GitHub Packages: the **scope must match the repository owner** — `@contoso/auth-sdk` must live under `contoso`, else 403.
- A publishing workflow needs **`packages: write`**.
- Use `@scope:registry=`, not a bare `registry=` — a bare one sends `lodash` there too.

**Feed roles (Azure Artifacts)**

| Role | Can |
|---|---|
| Reader | consume what the feed already holds |
| **Collaborator** | consume **+ save from upstream** ← minimum for a consuming team |
| Contributor | publish new packages |
| Owner | everything |

**Trap:** giving a consuming team **Reader**. The first public install fails, because it cannot save from upstream.

---

### 14 · Versioning strategies

> **SemVer communicates COMPATIBILITY. CalVer communicates AGE.**

| | Use for |
|---|---|
| **SemVer** `MAJOR.MINOR.PATCH` | **Libraries** — consumers and Dependabot read the compatibility signal |
| **CalVer** `YYYY.MM.DD` | **Applications** on a release train |

**The two arithmetic rules**

- A higher bump **resets everything below**: `2.1.0` + feature + fix = **`2.2.0`** (not `2.2.1`); `2.4.7` + breaking = **`3.0.0`** (not `3.4.7`).
- Precedence: `1.0.0-alpha` < `1.0.0-beta` < `1.0.0-rc` < **`1.0.0`**. A release always beats its own pre-releases.
- After `+` is **build metadata** — ignored for precedence. After `-` is a **pre-release** — sorts below.

**Also know**

- A published version is **immutable**. Never republish — ship `1.3.1` and deprecate.
- Reading the newest tag and adding one is a **race**: two merges compute the same version, second publish 403s.
- Retry-on-403 is not a fix — it publishes a version nobody predicted.
- **GitVersion with a shallow clone** starts from `0.1.0` and the build stays **green**.
- Never tag only `latest` — you have no rollback target.

**Trap:** `Build.BuildId` is unique, **not** sequential per package.

---

### 15 · Dependency management and vulnerability scanning

> **Dependabot is two separate features. And detect is not prevent.**

| Feature | When | Needs `dependabot.yml`? |
|---|---|---|
| **Security updates** | **Immediately** on a new CVE | **No** — enabled at repo/org level |
| **Version updates** | On a **schedule** | **Yes** |

You need both: alerts find the fire, version updates make the fix a one-line bump instead of a migration.

**Detect vs prevent**

| Detect | Prevent |
|---|---|
| Dependabot alerts | `dependency-review-action` with `warn-only: false` **and a required check** |
| Quarterly audit | NuGet `packageSourceMapping` (**fails closed**) |
| Security tab | a banned-package check that calls `exit 1` |

**Also know**

- **Vulnerability ≠ licence.** "Is this dangerous?" vs "may we ship it?" — different tools.
- The GPL problem arrives **transitively**, where no human reviewer looks.
- The vulnerable package is almost never one you chose — it is transitive.
- **CodeQL scans code you wrote.** Dependency scanning covers code you imported.
- An **unknown** licence is unassessed, not approved.

**Trap:** `warn-only: true` turns the only gate back into a report.

---

## Domain 3b — Testing in pipelines

---

### 16 · Testing strategy in pipelines

> **The pyramid is about cost and speed, not importance. Put each check at the lowest layer that can see the failure.**

| Layer | Needs | Finds |
|---|---|---|
| **Unit** (many) | nothing | logic; runtimes, via a matrix |
| **Integration** (fewer) | a real database, migrations, seeds | queries, migrations, contracts |
| **Load** (fewest) | a running app + concurrency | latency and error rate under load |

Chain them with `needs:` so an expensive layer never runs after a cheap failure.

**Also know**

- **A test that cannot fail the build is a report.** Only two things gate here: Jest `coverageThreshold` and k6 `thresholds` — both are non-zero exits.
- `PublishTestResults@2`, artifacts and PR comments are **visibility**, never enforcement.
- **Never wait with `sleep`.** Poll for readiness, and make the loop **fail** when it runs out.
- A container health check does **not** guarantee readiness — add an explicit `pg_isready` loop.
- `curl` needs **`-f`**, or a 500 response still exits zero.
- `fail-fast: false` lets every matrix leg finish. `continue-on-error` marks a failing leg **green** — not the same.
- Use `p(95)`, never `avg` — an average hides the tail users feel.

**Trap:** a shared staging database for integration tests. Concurrent runs make failures intermittent, and people learn to re-run instead of read.

---

### 17 · Quality and release gates

> **An approval asks a person, once. A gate asks a system, repeatedly.**

| Mechanism | Asks | How often |
|---|---|---|
| **Approval** (environment reviewers, PR reviews) | a person | once |
| **Wait timer** | nobody — a fixed delay | once |
| **Status check** | a build | once per commit |
| **Gate** (Azure Pipelines only) | a system | **repeatedly** — `initialDelay` → `period` → `timeout` |

Only a gate can say "wait until production is healthy, then continue on its own."

**Where things live — this is why the break is invisible**

```text
the YAML          only NAMES the environment
the ENVIRONMENT   reviewers, wait timer, deployment branch policy
BRANCH PROTECTION required contexts, review count, enforce_admins
none of these three can see the others
```

**Also know**

- A scanner gates only with a non-zero exit — Trivy needs `exit-code: '1'`.
- `if: always()` on the SARIF upload, or you lose results from exactly the runs that had findings.
- A failing job blocks a **merge** only if it is a required status check; it blocks a **deploy** only if something has it in `needs:`.
- A **post-merge** job can never be a required status check → the PR waits forever. Deadlock.
- Composite action `run` steps need **`shell:`**.
- A gate criterion must be checked against the **real** response. Wrong criteria look exactly like a slow system, for 24 hours.

**Trap:** "Waiting" is not a diagnosis. It is the state of every unmet gate.

---

### 18 · Code coverage analysis

> **Coverage measures which lines RAN, not whether anything CHECKED them.**

A test with no assertions covers everything and proves nothing. Coverage is a **floor**, not a score.

**Three policies — only one changes behaviour today**

| Policy | Meaning |
|---|---|
| Threshold (80% overall) | satisfied by history; says nothing about this PR |
| Ratchet | never below the last baseline on `main` — stops erosion (`<`, so equal passes) |
| **Diff coverage** (90% of new lines) | **the one that makes the author write a test** |

**"0% coverage but tests pass" — always plumbing, always silent**

1. Reporter list has no `cobertura` → console output only, no XML.
2. Output path ≠ `summaryFileLocation` → the file exists, elsewhere.
3. `coverlet.collector` not referenced → `XPlat Code Coverage` emits **nothing**.
4. Babel/ts-jest without source maps → 0% for the real sources. Fix: `coverageProvider: 'v8'`.

**Also know**

- `PublishCodeCoverageResults@2` takes **Cobertura or JaCoCo** — never LCOV or HTML.
- Cobertura `line-rate` is a **fraction** — multiply by 100.
- **Branch** coverage is the demanding metric; a covered line can have an untaken `else`.
- Without `collectCoverageFrom`, untested files vanish from the denominator and coverage looks high.
- Anything comparing to a base branch needs `fetch-depth: 0` **and** an explicit fetch of the base branch.

**Trap:** ask every gate what it does with **no data**. A missing artifact, a missing baseline and an empty changed-file list all report **success**.

---

## Domain 3c — Pipeline fundamentals

---

### 19 · GitHub Actions fundamentals

> **When the built-in token fails, add a permission — not a secret.**

**Also know**

- OIDC block, memorise it: `permissions: id-token: write` + `contents: read`, then `azure/login@v2` with `client-id`, `tenant-id`, `subscription-id` — **no secret stored**.
- **Managed identity does not exist on a GitHub-hosted runner.** The runner is not an Azure resource.
- **Composite action** = steps, same runner. **Reusable workflow** = jobs, its own runner.
- **Starter workflow** = copied once. **Composite action** = referenced live, updates flow.
- `if:` does **not** create ordering. Only `needs:` does.
- Matrix legs share nothing — own runner, own workspace. That is why `checkout` repeats.
- Job outputs are a 3-link chain: step `id` → job `outputs` → consumer `needs`. Any break gives an **empty string**, no error.
- Docker build context = the repo **minus `.dockerignore`** (not `.gitignore`).
- Tag `latest` only with `enable={{is_default_branch}}`, or a feature branch overwrites production's tag.

**Trap:** read the **last line** of the question first. "No long-lived credential" / "no PAT" eliminates most options instantly.

---

### 20 · Azure Pipelines YAML

> **`${{ }}` is compile time. `$[ ]` is runtime. `$( )` is the agent at execution.**

| Syntax | When it resolves | Can read a previous stage's output? |
|---|---|---|
| `${{ variables.x }}` | **compile time**, before anything runs | **No** |
| `$[ dependencies.A.outputs['s.v'] ]` | **runtime** | **Yes** |
| `$(varName)` | on the agent, in the script | – |

**Also know**

- A custom `condition:` **replaces** the default. Always write `and(succeeded(), ...)`.
- `Build.SourceBranch` = `refs/heads/main` (full ref). Short name = `Build.SourceBranchName`.
- **Deployment jobs do not check out source.** Add `- checkout: self` or download an artifact.
- Parameter type is **`boolean`**, not `bool`. Compare to `true`, never `'true'`.
- Stage variables do **not** cross stages — only output variables do.
- Only `name:` makes a step referenceable. `displayName:` does not.
- External templates need the alias: `template: path.yml@repoAlias`.
- `${{ if }}` **removes** the step at compile time; `condition:` **skips** it at runtime.
- Environments do not provision anything — they are a tracking and approval record.

**Trap:** `needs`, `if`, `type: choice`, `secrets.` are **GitHub Actions** keywords offered as Azure Pipelines distractors.

---

### 21 · Runner and agent infrastructure

> **Private network access → self-hosted. Never open the network to a cloud IP range.**

**Also know**

- **Label** = which runner picks the job. **Runner group** = who is allowed to use it.
- **Ephemeral** runner = the whole runner **de-registers** after one job — not just a wiped workspace.
- Ephemeral and **caching conflict** — a cache needs persistence.
- Azure Pipelines: `pool: vmImage` (Microsoft-hosted) and `pool: name` (self-hosted) are **mutually exclusive**.
- The agent advertises **capabilities**; the pipeline declares **demands**.
- **403** = reached and refused (permission/token). **Timeout** = never reached (network).
- Scale sets: **VMSS** is Azure DevOps. **ARC** (Actions Runner Controller) is GitHub.
- OIDC identifies the **workflow**, not the machine.

**Trap:** self-hosted is not automatically cheaper — maintenance costs more than the VM.

---

### 22 · Triggers and execution order — **your repeated mistake lives here**

> **Same upstream = parallel. Chained = serial.**

```text
PARALLEL                          SERIAL (the mistake)
  A                                 A
 / \                                |
B   C   <- both dependsOn: A        B   <- dependsOn: A
 \ /                                |
  D     <- dependsOn: [B, C]        C   <- dependsOn: B   <-- kills parallelism
```

Chaining `C dependsOn B` when both only needed `A` destroys the parallelism the requirement asked for. **Draw the graph before answering.**

**Also know**

- `dependsOn` / `needs` are **intra-pipeline only**. Across pipelines use `resources: pipelines:` or `workflow_run`.
- `always()` covers **skipped**; `succeededOrFailed()` does not.
- `workflow_run` fires on **failure too** — check `conclusion == 'success'`.
- `workflow_run` checks out the **default branch** — use `head_sha`.
- GitHub **forbids** `paths` + `paths-ignore` together (use `!`). Azure Pipelines **expects** include + exclude.
- `trigger: none` disables CI only — **schedules still run**.
- A scheduled run only works on a branch the pipeline actually tracks.
- **Path filter first** — it is free. A job condition burns an agent before skipping.

**Trap:** every consumer of a shared folder must list that shared path in its filter, or a `libs/` change builds nothing.

---

### 23 · Reusable pipeline elements

> **Steps → composite action. A job graph → reusable workflow.**

| | Composite action | Reusable workflow |
|---|---|---|
| Unit | steps | jobs |
| Runner | the **caller's** | its **own** |
| Secrets | **cannot** read `secrets` — pass as an input | pass explicitly or `secrets: inherit` |
| Outputs read from | `steps` | `jobs` |

**Also know**

- Pin to a **tag**, not `@main`, for anything consumers depend on.
- `ref:` needs the full path — `refs/heads/main`, not `main`.
- `@alias` refers to the `repository:` **alias**, not the repo name.
- Templates resolve at **compile time** — a `checkout` cannot fetch them.
- Azure Pipelines parameters use `type: string` + `values:`. `type: choice` is GitHub.
- Task groups are **classic only**, and step-level only.
- Path asymmetry: workflows must be in `.github/workflows/`; actions can live anywhere in the repo.

**Trap:** a reusable-workflow call takes no `runs-on` — the calling job runs no steps of its own.

---

### 24 · Checks and approvals

> **Environments gate DEPLOYMENTS. Branch protection gates MERGES.**

**Also know**

- No `environment:` on the job = **no protection rules evaluated at all**.
- Environment names are **case-sensitive** — a typo silently creates a new, unprotected environment.
- `dependsOn` cannot prevent two runs overlapping — that is **exclusive lock** (Azure DevOps) or **concurrency** (GitHub).
- Never use `cancel-in-progress: true` for production — an interrupted deploy leaves it half-updated. **Queue instead.**
- A concurrency group is a **lock name** — too broad and unrelated work blocks.
- `wait_timer` delays the deployment. **Timeout** expires the approval request.
- A stage `condition` **skips**; a check **holds** and re-evaluates.
- `200 OK` from a health endpoint does **not** mean there is no active incident — that needs an alerts check.
- Environment-scoped secrets are the isolation. Suffixed repo secrets (`PROD_KEY`) are visible to **every** job.

**Trap:** `minRequiredApprovers` is **Azure DevOps**. GitHub uses a reviewer list.

---


## Domain 3d — Deployment strategies

---

### 25 · Blue-green and canary — **your confirmed weak spot**

> **Both give zero downtime. So the separator is always in the requirement.**

| The requirement says | Answer |
|---|---|
| **Instant rollback**, low complexity | **Blue-green** (swap back) |
| **Validate with real user traffic** first | **Canary** |
| Gradual, group by group, stop on failure | **Ring-based** |
| Turn it off for users without redeploying | **Feature flag** |

| Strategy | Rollback speed | Cost |
|---|---|---|
| Blue-green | instant (swap back) | 2× |
| Canary | fast (route traffic away) | 1.1× |
| Rolling | moderate | 1× |
| Ring-based | fast (stop promotion) | 1.2–1.5× |
| Feature flags | instant (toggle) | 1× |

**Also know**

- **Slot swap: content swaps, slot settings stay put.** A "slot setting" is sticky to the slot.
- Slots need **Standard tier or higher**. F1 and B1 have none.
- Traffic Manager is **DNS-level** and probabilistic — TTL delays every change. `az webapp traffic-routing` is per request.
- Weight `0` stops DNS answers; the endpoint stays configured and probed.
- Warm-up before a swap = `WEBSITE_SWAP_WARMUP_PING_PATH`. `WEBSITE_HEALTHCHECK_PATH` is ongoing instance health.
- **Rollback steps use `failure()`, never `always()`** — `always()` undoes good releases too.
- A failed ring means **roll back**, never promote.

**Trap:** one release can legitimately need blue-green **and** canary. The strategies are not ranked.

---

### 26 · Rolling deployments and slot swaps

> **Batch size is how many at once. Unhealthy threshold is when to abort.**

**Also know**

- `maxBatchInstancePercent=100` = one batch of everything = the original outage.
- **Auto-swap fires inside App Service** — the pipeline never sees it, so it removes your validation step.
- Auto-swap also needs **Standard or higher**.
- Three warm-up settings, keep them apart:
  - `healthCheckPath` → **ongoing** instance health
  - `WEBSITE_SWAP_WARMUP_PING_PATH` → **during a swap**
  - `WEBSITE_WARMUP_PATH` → **on start**
- A readiness endpoint that always returns 200 makes warm-up theatre — it must check dependencies.
- The **App Insights key must be a slot setting**, or staging telemetry pollutes production dashboards.
- `AzureWebApp@1` **deploys**. `AzureAppServiceManage@0` **swaps**.

**Trap:** `continue-on-error` on validation turns a real failure into a green run.

---

### 27 · Feature flags

> **Targeting is sticky per user. Percentage is random per evaluation.**

| Filter | Behaviour |
|---|---|
| `Microsoft.Targeting` | **Same user always gets the same answer** — use for a real audience |
| `Microsoft.Percentage` | Random **each time it is evaluated** — a user can flip on refresh |
| `Microsoft.TimeWindow` | On between two times |

**Also know**

- Reading flags needs the **data-plane** role `App Configuration Data Reader`. **Contributor cannot read flags.**
- You need **both**: `AddAzureAppConfiguration()` (service) **and** `UseAzureAppConfiguration()` (middleware, before `MapControllers()`).
- Register `IHttpContextAccessor`, or every user is anonymous with no groups.
- **Label mismatch** makes a separate record. An empty label is not `production`.
- **Deploy dark, verify, then enable.** Never enable before the code is out.
- Deleting a flag before removing the code = it evaluates as **false**.
- Flags decide **who sees** a feature. Slots decide **how code arrives**. Flags never roll back a schema change.

**Trap:** "Store it in Key Vault" is not "no secret". A hidden secret is still a secret — use an identity.

---

### 28 · Container-based deployments

> **Applying is not verifying. Reporting is not gating.**

**Also know**

- `AcrPull` reads. **`AcrPush` reads and writes.** Pushing with `AcrPull` fails.
- Turn the admin account **off** (`--admin-enabled false`) and use a managed identity.
- **Liveness probe restarts** the container. **Readiness probe removes it from the Service.**
- `maxUnavailable: 0` is meaningless without a readiness probe — availability is *defined* by readiness.
- `KubernetesManifest@1` **applies**. `kubectl rollout status` **verifies**.
- Trivy `severity` only **filters**. `exit-code: '1'` **blocks**.
- **Scan before the push**, or the bad image is already pullable.
- Container Apps: `--image` creates a **new revision**; `--revision-weight` routes traffic back to an old one.
- Multi-arch needs **buildx** (builds) **and** QEMU (emulates).
- `push: true` on pull requests publishes untrusted images — build on PR, push on merge.

**Trap:** the port must agree in three places — the app listens, the Dockerfile documents, the ingress routes.

---

### 29 · Database deployment automation

> **Code rolls back. Data does not.**

Slots, revisions and flags all revert because the old version still exists. **A dropped column does not.**

**Expand–contract, always**

```text
1  EXPAND    add the new column, NULLABLE. deploy.
2  MIGRATE   backfill. both old and new code work.
3  CONTRACT  drop the old column - IN A LATER RELEASE
```

**Also know**

- The **migration runs before** the code that needs it — wire it with `dependsOn` / `needs`.
- A **rename is a drop plus an add** to running code. Never "just rename".
- `NOT NULL` in the expand phase breaks old inserts. Tighten later.
- Never disable `BlockOnPossibleDataLoss` — the guard is right; use expand-contract.
- **Forward-fix** for an ordinary bug. Restore is for catastrophe only.
- A slot swap reverts **code only**. Schema keeps moving forward.
- Azure RBAC is the **control plane** — it does not grant DDL. Create a database user, and use `db_ddladmin`, not `db_owner`.
- Migrations at application startup = no review, no gate, and instances racing on DDL.
- An applied migration is **immutable** — add a new one.

**Trap:** idempotent is not ordered. Safe to re-run ≠ applied sooner.

---

### 30 · Hotfix paths and resiliency

> **A hotfix REDUCES gates. It never removes them.**

| Survives | Dropped |
|---|---|
| Security scan (reduced to SAST) | Full integration suite |
| Deploy to staging, then swap | Progressive rings |
| One approver (an **on-call team**) | Multi-stage approvals |
| Cherry-pick back to `main` | – |

**Also know**

- **Branch from the release tag**, not `main` — `main` has unreleased work in it.
- Branching from the *last-good* tag is a **rollback**, not a hotfix.
- Deploy to **staging and swap**, or you throw away the rollback path.
- `traffic-routing clear` **before** swapping — leftover routing survives the swap.
- **Cherry-pick the fix to `main`**, or the bug returns next release.
- A swap **exchanges** — the third swap restores the bug. You get one free rollback.
- Circuit breaker counts to a **threshold**; transients happen.
- `GITHUB_TOKEN` pushes do **not** trigger workflows — use an App token or PAT.

**Trap:** naming an individual as the hotfix approver deadlocks the emergency path at 3 AM.

---

## Domain 3e — Infrastructure as Code

---

### 31 · Infrastructure as Code strategy

> **Plan on the PR. Apply on merge. Never apply on a pull request.**

**Also know**

- **Bicep** for Azure-only. **Terraform** when you need drift detection or multi-cloud.
- `validate` = "is this acceptable?" **`what-if`** = "what will change?"
- One **state file per environment**. Never share state.
- A stuck Terraform lock: verify the lease, then `force-unlock`. **Never delete state.**
- Activity Log records **operations**, not **differences** — it is not drift detection.
- An environment parameter should have **no default** — force an explicit choice.
- A `@secure()` parameter with a default puts the secret in git and in deployment history.
- `github.ref` is `refs/heads/main`, not `main`.
- `bicep build` checks **syntax**. checkov / PSRule check **security**. Different jobs.
- Pin module versions — publish to a registry and reference a version.

**Trap:** `if: always()` on the SARIF upload — the scan fails the step, so without it the upload is skipped.

---

### 32 · Desired state configuration

> **Audit tells you. AuditAndSet + ApplyAndAutoCorrect fixes it — and it needs an identity with a role.**

**Three things must all be right, or you get a dashboard instead of enforcement**

1. The **package** must be `AuditAndSet` (not `Audit`).
2. The **policy** effect must be `ApplyAndAutoCorrect` (`ApplyAndMonitor` applies once, then only reports).
3. The **assignment** needs a managed identity **and** an RBAC role — the role grant is a separate step.

**Also know**

- Compliance **"Pending"** means never reported — check the identity and the extension, not the policy.
- `trigger-scan` **evaluates**. A **remediation task** fixes.
- Bicep configures the **resource**; Machine Configuration configures **inside the VM**.
- Azure Automation State Configuration is **legacy** — Machine Configuration replaces it.
- The agent connects **outbound only** — no inbound access needed.
- Bundle dependent modules with `-FilesToInclude`.
- Linux needs its **own** package and policy — `-Platform Windows` misses half a mixed fleet.

**Trap:** a SAS token expiry — every assignment breaks silently the day it expires.

---

### 33 · Azure Deployment Environments

> **The developer REQUESTS. The platform identity DEPLOYS. Two different principals.**

| Who | Role | Does |
|---|---|---|
| Developer | `Deployment Environments User` (project-scoped) | **asks for** an environment |
| Project environment type's **managed identity** | Contributor on the target subscription | **actually deploys** |

A developer with the correct role still fails if that identity has no permission — and the error is `AuthorizationFailed`.

**Also know**

- **Catalogs belong to the Dev Center**, not the project.
- The **project environment type** maps the subscription, not the Dev Center.
- `environment.yaml` is the developer contract — parameters live there, not in `main.bicep`.
- Keyword by platform: Azure DevOps `values`, GitHub `options`, ADE **`allowed`**.
- Name the environment after the **PR number**, not `Build.BuildId` — the PR number is stable across pushes.
- Check whether the environment exists first, or the second push fails.
- **Poll `provisioningState` until `Succeeded`** — never a fixed sleep.
- Idle cleanup is not built in: a policy tags, a runbook deletes. Neither alone is enough.
- A `modify` policy acts on **create and update** — existing resources need a remediation task.

**Trap:** read the error. "Not found in catalog" = a sync problem. `AuthorizationFailed` = a permission problem.

---


## Domain 3f — Pipeline operations

---

### 34 · Pipeline health monitoring

> **A retry that leaves no trace turns a visible problem into a permanent one.**

Right answer = **retry AND record** (annotation, artifact or tracking issue). Every wrong answer skips the second half.

**Also know**

- `retryTimes` makes Jest publish a clean "passed" — flakiness detection needs comparison or platform-side reporting.
- Never `|| true` on a test step. Let it fail; publish with `always()`; fail honestly.
- `continue-on-error` turns a failure into a warning nobody reads.
- In a retry loop, **the last command's exit code wins** — usually the `echo`.
- **Quarantine** a flaky test; never delete it. Deleting loses the coverage.
- Alert on a **pattern with an owner**, not on every failure.
- Read **P95** duration, not the average — P95 is what developers actually feel.
- **Queue time counts.** Total duration includes waiting for an agent.
- `status` = finished. `conclusion` = how it finished.

**Trap:** pipeline MTTR and DORA MTTR are the same word at different scope.

---

### 35 · Pipeline optimization

> **Four levers: cache, parallelise, skip, shrink. Optimise before you buy capacity.**

**Also know**

- Cache key on the **lock file** (`package-lock.json`), never `package.json`. Add `restore-keys`.
- `setup-node`'s cache covers **`~/.npm` only**, not `node_modules`.
- `cache-hit` is the **string** `'true'`, not a boolean.
- Each shard needs a **unique artifact name**, then merge. Same name = they overwrite.
- `fail-fast: true` on a test matrix cancels the rest after one failure — you lose the full picture.
- Upload only the deployable output. `path: .` uploads the whole workspace.
- **Trigger/path filters are free. A job condition costs a runner.** Filter first.
- `affected`-style tooling needs at least `fetch-depth: 2`.
- **Parallelism cannot beat the critical path** — duration is bounded by the longest chain.
- GitHub bills per **minute**; Azure DevOps bills per **parallel job**. Do not mix the models.

**Trap:** self-hosting before optimising. Optimisation changes the break-even point.

---

### 36 · Retention strategies

> **Tier retention by value. PRs days, `main` weeks, releases a year.**

**Also know**

- Purge containers with **`--untagged` first**. Deleting by age alone removes tags you still reference.
- A **retention lease** needs the auth header **and** a JSON array body — verify it exists afterwards.
- Scope leases to **production deployments**, not every merge.
- A cleanup script that ignores leases still deletes: deleting a run deletes its artifacts.
- Schedules need `always: true`, or storage grows in quiet weeks.
- The **90-day default** is generous and nobody chose it.
- Scope feed cleanup to `@prerelease`. **Promotion to `@release` is how you protect a package.**
- `--paginate` or you only process the first page.
- **Releases never expire** — they persist until deleted and need their own pruning.

**Trap:** dropping the SKU cuts features, not gigabytes. Archiving to cheaper storage still grows.

---

### 37 · Migrate classic to YAML

> **Approvals and gates do NOT live in YAML, and they do NOT migrate.**

A newly created environment has **no checks**, and the pipeline will deploy happily without them.

**Also know**

- Gates become **environment checks**, configured in the UI. There is no `gates:` keyword.
- Another pipeline's artifact needs `resources: pipelines:` and download **by alias** — classic did this implicitly.
- `pipeline:` is the **alias**; `source:` is the real pipeline name.
- Deployment groups → an **environment with `resourceType: VirtualMachine`**, not an agent pool.
- Export covers **builds only**. Release definitions are converted by hand.
- Migrate in phases: run both against a **shadow** environment, keep classic disabled for two weeks, then delete.
- Archive the classic JSON — it is the audit record of what the pipeline used to do.
- **Re-authorise** service connections and environments for the new pipeline.
- Reference the existing variable group; never inline secrets during a migration.

**Trap:** `type: boolean` compared to `true` — not `bool`, not `'true'`.

---

### 38 · End-to-end capstone

> **Gate on the merged result, not the parts. Verify the version, not the status code.**

**Also know**

- Gate on `coverage-gate`, not `test-unit` — **shards can each pass while the merged total fails.**
- A 200 after a swap can come from the code you just replaced. **Compare the version.**
- Warm up, swap, **then** verify — two separate loops.
- Approvals live on the **environment**. Staging is automatic; only production gates.
- Use OIDC (`id-token: write`), never a stored `AZURE_CREDENTIALS`.
- Suffix shard artifacts with the matrix value.
- Intermediate artifacts get **7 days**, not the 90-day default.
- **Scan the filesystem before building/publishing the image.**
- Optimise the **longest chain**, not the sum of jobs.

**Trap:** never lower a threshold to make a build pass. A moved gate is no gate.

---

## Domain 4 — Security and compliance (10–15%) — **your weakest domain, read twice**

---

### 39 · Authentication and identity — **the three sentences that earn the most marks**

> **1. Stored secret? · 2. Where can it run? · 3. What if it leaks?**

| | Service principal + secret | Managed identity | Workload identity federation (OIDC) |
|---|---|---|---|
| **Stored secret?** | **Yes** | **No** | **No** |
| **Where can it run?** | anywhere | **only Azure-hosted compute** | from a trusted OIDC issuer |
| **If it leaks?** | works anywhere until rotated | useless off its resource | valid for one repo/branch/environment only |

**Pick it in one step**

| Scenario | Answer |
|---|---|
| GitHub Actions → Azure, "no stored secret" | **Workload identity federation** |
| An Azure VM / App Service → Key Vault | **Managed identity** |
| On-prem agent, or a tool that cannot do OIDC | **Service principal + secret** |
| One identity shared by several Azure resources | **User-assigned managed identity** |

**Also know**

- **Managed identity does not exist on a GitHub-hosted runner.** It is not an Azure resource.
- Federation is **authentication only**. You still need an **RBAC role assignment**.
- **`AADSTS70021`** = subject mismatch (fix the federated credential). **`AuthorizationFailed`** = no RBAC role (create the assignment).
- The subject holds the **full git ref**: `repo:org/repo:ref:refs/heads/main` — not `branch:main`.
- **No wildcards.** One credential per subject. An environment subject needs `environment:` on the job.
- Forgetting `id-token: write` fails at the **token request**, with a different error.
- **Principal ID → RBAC. Client ID → the application.** Do not swap them.
- Control-plane roles (Contributor) cannot read a blob or a secret. That needs a **data-plane** role.

**Trap:** Owner is not "more secure" than Contributor — it adds privilege escalation.

---

### 40 · GitHub authentication

> **How far does it reach? And who owns it — the automation, or a person who can leave?**

| Credential | Reach | Owner | Expires |
|---|---|---|---|
| `GITHUB_TOKEN` | **this repository only** | automation | end of job |
| GitHub App token | the **installation's** repos | organisation | short-lived |
| Fine-grained PAT | the repositories you ticked | a person | set; org can cap |
| Classic PAT | **everything the user can see** | a person | optional ← ban it |

**`GITHUB_TOKEN` can never**

reach another repo · trigger another workflow · write `.github/workflows/` · bypass branch protection.

**Also know**

- Permissions control **what**, not **where**. Adding `contents: write` never reaches another repo.
- Writing workflow files needs a **GitHub App token plus a bypass entry** — no permission alone does it.
- An org can **block or allow** classic PATs. It cannot cap their lifetime.
- **404** = outside a PAT's scope. **403** = outside an installation.
- Job-level `permissions:` **replace** workflow-level — repeat the ones you still need.
- The app-token step needs `owner:`, or cross-repository reach disappears.
- Cap Owners at **2–3 people**. Use **Outside collaborator** for contractors — Member is org-wide and outlives the contract.
- Build and prove the replacement **before** revoking classic PATs.

**Trap:** a shared service account is still an account, still shared, still unattributable.

---

### 41 · Azure DevOps permissions and service connections

> **Four dials. Almost every wrong answer is the right action on the wrong dial.**

| Dial | Controls |
|---|---|
| **Access level** (licence) | what you can **see** — Stakeholder / Basic / Basic+Test Plans |
| **Security group** | what a **group of people** may do in a project |
| **Service connection** | what a **pipeline** may do in Azure (SP + RBAC scope) |
| **Pipeline permissions** | **which pipelines** may use that connection (`--enable-for-all false`) |

All four must line up. Basic licence but no group = sees nothing. Right group but unauthorised connection = resource authorization error.

**Also know**

- CLI names: `stakeholder` · **`express`** (= Basic) · **`advanced`** (= Basic + Test Plans).
- **Stakeholder is free and CAN create work items** — that is why it suits PMs.
- Custom groups **inherit nothing** — nest them inside Contributors.
- **Deny beats Allow everywhere.** Grant narrowly instead of denying.
- PAT policy is set at **Organisation Settings** and is not overridable per project.
- Only stages that declare `environment:` get checks — a `job:` instead of a `deployment:` gets **no checks and no error**.
- A YAML `condition` is not access control — anyone who edits the file removes it.
- Azure DevOps federated credentials use `vstoken.dev.azure.com` and a `sc://` subject, not GitHub's.

**Trap:** check the **licence** before you debug namespace bits.

---

### 42 · Secrets management

> **Ask: does a credential still exist after this change?**

```text
RUNG 0   secret in a plain-text pipeline variable        <- the audit finding
RUNG 1   secret in Key Vault, fetched by the pipeline
RUNG 2   secret in Key Vault, resolved by the APP        @Microsoft.KeyVault(...)
RUNG 3   NO SECRET EXISTS                                managed identity / OIDC
```

Take the **highest rung the scenario allows**. A third-party credential you cannot replace caps you at rung 2.

**Key Vault RBAC roles** — *Officer manages, User uses, Reader only sees it exists*

| Role | Can |
|---|---|
| Key Vault Administrator | everything |
| Secrets **Officer** | create / update / delete / read |
| Secrets **User** | **read secret values** ← pipelines and apps |
| Key Vault **Reader** | **metadata only — NOT values** |

**Also know**

- On an **RBAC-enabled vault, access policies are ignored** — and the 403 still says "access policy". Check `enableRbacAuthorization` first.
- **Never pin a secret version** — rotation silently stops propagating.
- Always set an **expiry date**, or `SecretNearExpiry` never fires.
- Azure Pipelines masks secrets it knows about. **GitHub Actions needs `::add-mask::`**, and masking is **not retroactive**.
- The **app** resolves a Key Vault reference — that is the whole point; the pipeline never sees the value.
- A managed identity's principal type is **`ServicePrincipal`**.
- A **system-assigned** identity dies with the resource. Use **user-assigned** for rebuilt infrastructure.

**Trap:** "store it in Key Vault" when another option removes the credential entirely. The second wins.

---

### 43 · Sensitive file handling and leak prevention

> **Where does the control run? And a leaked secret is ROTATED, not hidden.**

| Layer | Control | Strength |
|---|---|---|
| Client | `.gitignore`, pre-commit hook | **bypassable** — `--no-verify`, fresh clone, bot |
| **Server** | **push protection** | **the secret never enters the repo** |
| After | CI secret scan, log review | **detection**, already committed |

**Masking rule: the platform masks what the PLATFORM gave you.**

- Masked: repo secrets, Key Vault-linked variable groups, UI secret variables.
- **Not masked:** anything `curl` / `az` / `jq` produced on the agent — register it yourself, **immediately**.

**Also know**

- `.gitignore` does nothing to an **already-tracked** file — needs `git rm --cached`, and history still holds it.
- Deleting the run is **containment**, not remediation.
- **Rotate before rewriting history.** Rotation ends the exposure; a rewrite never fully does.
- `isOutput=true` needs a step **`name:`** to qualify the reference.
- Secret variables are **not** auto-injected as env vars — pass them via `env:`.
- Only the **original** secure file is cleaned up. Your copies need an `always()` cleanup step.
- Use `always()`, not `succeededOrFailed()` — the latter misses cancellation.
- gitleaks allowlists: narrow to the **fixture path**. Never a format or `(.*)`, and never `useDefault = false`.

**Trap:** printing a "prefix to verify" — the prefix is the identifying part. Print a length or a hash.

---

### 44 · GitHub Advanced Security

> **Four scanners. They do not overlap. Every question is asking WHICH ONE.**

| Scanner | Finds |
|---|---|
| **CodeQL** | flaws in code **you wrote** — injection, path traversal |
| **Dependabot** | flaws in code **you imported** — incl. **transitive** |
| **Secret scanning** | **credentials** in the repo (+ **push protection** = the blocking half) |
| **Container scanning** | flaws in the **image** — base OS packages, binaries |

- Out-of-date **base image tag in a Dockerfile** → **Dependabot** (docker ecosystem).
- **CVE inside the built image** → **Trivy** / container scanning.

**Also know**

- **The Security tab renders SARIF.** Any scanner emitting SARIF publishes via `codeql-action/upload-sarif`.
- `queries:` needs a leading **`+`** to *extend* the default suite — without it you **replace** it. (In GHAzDO `querysuite`, no `+`.)
- **Compiled languages need a build between `init` and `analyze`** — otherwise zero results and a green tick. `autobuild` guesses and fails silently.
- Multiple SARIF uploads need a **`category:`**, or one tool's results replace the other's.
- Dependabot **reports** transitive fixes — `overrides` / `resolutions` force them.
- Dismiss with `revoked` for a real secret, `false_positive` for a non-secret.
- Report at HIGH, **block at CRITICAL** — failing on every severity gets the gate switched off.
- Guard Dependabot workflows on `github.actor`, not `head_ref` (branch names are attacker-controlled).
- Auto-merge with `--auto`, never `--admin` (which bypasses branch protection).

**Trap:** GHAzDO **detects**; Dependabot **PRs are GitHub-side**. And never mirror Azure DevOps repos to GitHub for scanning — GHAzDO is native.

---

### 45 · Defender for Cloud DevOps Security

> **Posture = CONFIGURATION. Findings = CODE. Defender detects settings, and aggregates the rest.**

Asked "what would Defender **automatically detect**?" → the answer is a **setting**, never a vulnerability.

| Defender finds (posture) | Defender only aggregates (findings) |
|---|---|
| code scanning not enabled | SQL injection → CodeQL |
| branch protection missing | vulnerable package → Dependabot |
| no required reviewers | leaked token → secret scanning |
| secret scanning / Dependabot not enabled | |
| **excessive permissions on service connections** (Azure DevOps only) | |
| inactive repos with access (GitHub only) | |

**Inform vs block — asked in every case study**

- A PR annotation **comments**. The setting is literally *"Comment only (do not block merge)"*.
- To **block a merge** you need **both**: a scan job that **exits non-zero**, **and** branch protection listing that job as a **required status check**.
- A **governance rule** assigns an owner and a deadline — **after** the merge.
- **Azure Policy** makes the scanner mandatory in the first place.

**Also know**

- Defender does **not** replace GHAzDO. GHAzDO scans; Defender aggregates and governs.
- Annotations need the pipeline to run **on the PR** (`pr:` in Azure Pipelines, not `pull_request:`) and the `.gdn` results **published**.
- The scan step needs **`id:`**, not just `name:` — `steps.msdo.*` silently uploads nothing otherwise.
- `trigger: none` — omitting `trigger:` means **every push to every branch**.
- **"No findings" is not "no problems."** An unscanned repo looks exactly like a clean one.
- Offering type: `CspmMonitorGitHub` vs `CspmMonitorAzureDevOps`.
- **Auto-discovery covers NEW repos.** Existing ones need the scope updated.

**Trap:** the deliverable is **one pane of glass**. Proposing two dashboards restates the problem.

---


## Domain 5 — Instrumentation (5–10%) — **cheapest marks on the exam**

---

### 46 · Azure Monitor integration with DevOps

> **Detect → Decide → Act. And Application Insights NEVER notices you deployed.**

| Step | Thing |
|---|---|
| **Detect** | metric alert (platform metric, cheap, 1 min) or log alert (KQL, expressive, 5 min) |
| **Decide** | `--window-size` = **how much data** · `--evaluation-frequency` = **how often you look** |
| **Act** | action group: email · webhook · **azurefunction** |

**Two facts answer half this challenge**

1. **The pipeline pushes the deployment annotation** via the REST API — in **UTC**, with the commit in its properties.
2. **Action group webhooks carry no authentication**, so they cannot start a pipeline. Put an **Azure Function or Logic App** in between.

**Also know**

- Window **larger than** frequency = overlapping windows, smooths noise, still reacts fast.
- **Smart detection notifies. It does not invoke** anything.
- Querying telemetry right after a deploy always passes — **ingestion lag**. Give the gate a minimum duration.
- Use a **rate**, not an absolute error count — counts scale with traffic.
- A rollback **is a deployment** — verify it.
- External trigger = `repository_dispatch` (carries `client_payload`), not `workflow_dispatch`.

**Trap:** a 2-minute in-pipeline check cannot see a memory leak. Slow degradation is caught by the **reactive** layer.

---

### 47 · Telemetry collection and insights

> **Choose by WHAT you are watching. They stack — they do not compete.**

| Watching | Use |
|---|---|
| Application code | **Application Insights** — requests, dependencies, exceptions, traces |
| A virtual machine | **VM Insights** — CPU/memory/disk + the **Map** (process dependencies) |
| A Kubernetes cluster | **Container Insights** — node/pod metrics, container stdout/stderr |
| Pod-exposed metrics | **Managed Prometheus** |
| Is the site reachable? | **Availability test** (multi-region) |

An app on a VM wants **both** App Insights and VM Insights.

**Also know**

- **"No code changes"** → auto-instrumentation: app settings + **restart**. The agent attaches at process start.
- **Instrumenting every service is NOT enough for distributed tracing** — `traceparent` must propagate at **every hop**, or the trace stops there and everything still looks healthy.
- Correlation is on **`operation_Id`**, not on being in the same resource.
- A **DCR created but never associated** = healthy agent, correct rule, **no data**.
- The Map needs `Microsoft-ServiceMap` — without it Performance works and the Map is empty.
- Adaptive sampling **drops exceptions** unless you list them in **`excludedTypes`** (excluded *from sampling* = kept).
- **Never enable `env_var` collection** — it copies secrets into a queryable store.
- A **daily cap is a hard stop**, not a cost strategy — it fires soonest on your worst day.
- Use the **connection string**; `APPINSIGHTS_INSTRUMENTATIONKEY` is deprecated.
- Numbers go in **`measurements`** (aggregate); strings in `properties` (filter).

**Trap:** percentiles, never averages.

---

### 48 · GitHub monitoring and alerts

> **Three insights surfaces. Name the wrong one and the answer is wrong.**

| Surface | Shows |
|---|---|
| **Repository Insights** | Pulse, Contributors, Traffic, Commits, Code frequency, Dependency graph → **the code** |
| **Actions (via API)** | run duration, success rate, billed minutes → **the pipelines** |
| **Projects Insights** | burn-down, cycle time, items by assignee → **the work** |

**The alerting pattern — memorise it**

```yaml
  notify-failure:
    needs: [build]      # gives the condition something to evaluate
    if: failure()       # the default is success()
```

A step at the end of a job **does not run when the job fails early** — the only case you wrote it for.

**Also know**

- `continue-on-error` makes the job report success, so `failure()` **never fires**.
- `needs:` without `if: failure()` notifies on **every** run, and the channel gets muted.
- **Traffic is views and clones** — not deployment frequency.
- A **run** is an attempt. A **deployment** records an environment. A **release** is just a tag.
- Average duration over all runs is wrong — **failures die early**. Use success-only, failures separately.
- `status` = lifecycle (`completed`). **`conclusion`** = outcome.
- **Cancelled** is neither success nor fault.
- `createdAt` includes queue time; `startedAt` does not.
- For PRs use **`merged`**, not `closed` (closed includes abandoned work).
- `if:` in GitHub Actions. `condition:` in Azure Pipelines.

**Trap:** a silent alert channel is an assumption until you have tested it.

---

### 49 · KQL for DevOps

> **Four operators carry most of the marks.**

| Operator | Use for |
|---|---|
| `summarize ... by name` | per **group**, not per row — "5 slowest **endpoints**" vs "5 slowest **requests**" |
| `toscalar()` | a baseline is **one number**, not a table |
| `join kind=leftanti` | **what is new** — rows on the left with no match on the right |
| `countif()` | a **rate in one pass** |

**Two silent killers — neither raises an error, both say "everything is fine"**

- `customDimensions` is **dynamic** → compare with **`tostring()`** or the indexer.
- **`100` instead of `100.0`** → integer division truncates every rate below 1% to **zero**.

**Also know**

- `where success == false` **before** the summarize removes the denominator — the rate is always 100%.
- A **baseline must exclude the present**, or the spike inflates its own comparison.
- A ratio threshold needs an **absolute floor** and a **minimum traffic** count — 1 failure in 4 requests is 25%.
- `prev()` needs `serialize` — row order is otherwise undefined.
- `inner` vs `leftanti` vs `fullouter`: **both** / **only new** / **new and resolved**.
- KQL words: **`by`**, **`project`**, **`join`** — never SQL's `group by`, `select`, `merge`.
- **Analytics (OData)** aggregates history server-side; the **REST API** returns individual records.
- OData filters use **word operators**: `gt ge lt le eq ne` — never `>` or `<`.
- Read-only monitoring access = **Monitoring Reader**, not Contributor.

**Trap:** a percentile over 3 data points means nothing. Always set a minimum request count.

---

### 50 · Performance analysis

> **Read the trace from the BOTTOM. Then subtract the dependency.**

```text
Frontend 50ms -> Gateway 30ms -> Order 4500ms -> Payment TIMEOUT
Order is not slow. Order is WAITING. The answer is PAYMENT.

request 1500ms - dependency 1200ms = 300ms of your code   -> the DATABASE is the answer
request 1500ms - dependency   50ms = 1450ms of your code  -> your code / undersized instance
```

**Order of investigation — each step narrows the next**

1. **Percentiles** — how bad, and for whom (P50 vs P99)
2. **Before / after** — did the deployment cause it?
3. **Dependencies** — by P95 **and** by total impact (duration × count)
4. **End-to-end trace** — inside one slow request (`union` on `operation_Id`)
5. **Infrastructure** — only if 1–4 point there
6. **Burn rate** — how urgently to act

**SLI / SLO / error budget**

| Term | Meaning |
|---|---|
| **SLI** | the measurement — successful ÷ total |
| **SLO** | the target — 99.9% |
| **Error budget** | `100% − SLO` → `total * (1 - sloTarget / 100.0)` ← **`100.0`, not `100`** |
| **Burn rate** | failed ÷ daily budget. **1.0** = spends it exactly at the window's end. **>1.0** = exhausted early |

- Multi-window: **fast burn 14.4× over 1 h**, **slow burn 6.0× over 6 h**.
- 80% of budget at 50% of window = burn 1.6 → **reduce frequency, raise rigour**. A freeze is for budget **exhausted**.
- **Lowering the SLO is never the answer** — then it constrains nothing.

**Rollback or hotfix?**

- **Rollback** — the fault is in your code and cleanly reversible.
- **Hotfix** — the fault is downstream, **or** the deploy included a **migration** a rollback would not undo.

**Also know**

- `Available MBytes` measures **free** memory — falling is the problem.
- Attribute VM CPU with `VMProcess` and `ProcessName`, not `Perf` totals.
- Capture evidence **before** restarting.
- **Smart detection is on by default and only notifies.** A **dynamic threshold alert must be created.**

**Trap:** ticket volume is not urgency. Burn rate is.

---

## Capstone

---

### 51 · Cross-domain capstone (Contoso Payments, PCI-DSS)

> **Every domain in one question. The compliance words tell you which control to pick.**

| The scenario says | It is asking for |
|---|---|
| "No secrets in source control" | OIDC / managed identity (rung 3), push protection, secret scanning |
| "Traceable to approved work items" | `Fixes AB#`, work-item-linking policy, required reviews |
| "Auditable" | branch protection + `enforce_admins`, audit log, retention leases |
| "Continuous monitoring, automated alerting" | App Insights + action group → Function → dispatch |
| "Payment gateway, card data" | least privilege everywhere; Key Vault; no admin accounts |

**How to answer a capstone question**

1. Name the **domain** the sentence belongs to.
2. Name the **control type** — detect or prevent, inform or block.
3. Pick the option that **enforces**, not the one that reports.

---

# Part 5 — The snippets you must be able to write from memory

---

**GitHub → Azure with no stored secret (OIDC)**

```yaml
permissions:
  id-token: write        # WITHOUT THIS: the token request itself fails
  contents: read
steps:
  - uses: azure/login@v2
    with:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

The federated credential subject is the **full ref**: `repo:org/repo:ref:refs/heads/main`.

**Notify only on failure — from a separate job**

```yaml
notify-failure:
  needs: [build, test]
  if: failure()
```

**Matrix that finishes every leg**

```yaml
strategy:
  matrix:
    node-version: [18, 20, 22]
  fail-fast: false        # NOT continue-on-error - that marks a failing leg GREEN
```

**Publish results even when the job failed**

```yaml
- uses: actions/upload-artifact@v4
  if: always()            # Azure Pipelines: condition: always()
```

**Azure Pipelines — runtime vs compile time**

```yaml
- stage: B
  variables:
    v: $[ dependencies.A.outputs['setvar.value'] ]    # RUNTIME - crosses stages
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
```

**Deployment job — it does NOT check out source**

```yaml
- deployment: Deploy
  environment: production        # the checks live ON the environment, not here
  strategy:
    runOnce:
      deploy:
        steps:
          - checkout: self       # or you have no source
```

**Wait for something to be ready — never `sleep`**

```bash
for i in $(seq 1 30); do curl -sf http://localhost:3000/health && break; sleep 2; done
curl -f http://localhost:3000/health || exit 1     # an EXHAUSTED loop must FAIL
```

**Make a scanner a gate**

```yaml
- uses: aquasecurity/trivy-action@0.28.0
  with:
    severity: 'CRITICAL,HIGH'
    exit-code: '1'        # WITHOUT THIS: findings, a SARIF file, and a green tick
```

**KQL — an error rate that is actually correct**

```kusto
requests
| where timestamp > ago(1h)
| summarize total = count(), failed = countif(success == false) by name
| where total > 100                         // no percentile on 3 data points
| extend errorRate = 100.0 * failed / total // 100.0 - NOT 100
| order by errorRate desc
```

---

# Part 6 — Numbers to memorise

| Number | What |
|---|---|
| **700 / 1000** | Passing score |
| **~120 min** | Exam duration |
| **50–55%** | Domain 3 weight — over half the exam |
| **10–15%** | Domains 1, 2 and 4 each |
| **5–10%** | Domain 5 |
| **Under 1 hour** | Elite lead time **and** Elite MTTR |
| **Multiple per day** | Elite deployment frequency |
| **0–15%** | Elite change failure rate (**not 0%**) |
| **90 days** | Default Azure DevOps retention (nobody chose it) |
| **`fetch-depth: 0`** | Unlimited history — the default is shallow |
| **720** | Minutes in `--valid-duration` = 12 hours |
| **14.4× / 1 h · 6.0× / 6 h** | Fast burn · slow burn |
| **Standard tier** | Minimum App Service tier for **slots** |
| **`api://AzureADTokenExchange`** | The OIDC audience |

---

# Part 7 — How to answer a question you have never seen

This is the part that matters most. Do it in this exact order, every time.

**Step 1 — Read the LAST sentence first.**
The constraint lives there: *"without storing a credential"*, *"minimise cost"*, *"with no code changes"*, *"the team must keep supporting version 2"*. This one habit fixes your biggest confirmed weakness.

**Step 2 — Say the constraint out loud in your own words.**
If you cannot say it, you have not read it.

**Step 3 — Ask: does this need to INFORM or to BLOCK?**

- Inform → a comment, an annotation, an artifact, a dashboard, an audit log, an alert.
- Block → a **non-zero exit code**, a **required status check**, an **environment protection rule**, a **branch policy with `--blocking true`**.

More than half of all wrong answers on this exam are a real control that only informs.

**Step 4 — Eliminate anything that ignores the constraint**, even if it is technically correct. A right answer to a different question is still wrong.

**Step 5 — Between the two survivors, pick the one that removes the risk rather than manages it.**

- No secret beats a stored secret.
- Server-side beats client-side.
- Derived beats maintained by hand.
- Narrowest permission that works beats a broader one.

**Step 6 — If you are still stuck, name the layer.**
Licence, group, connection, pipeline permission? Code, imported package, credential, image? Field or view? Approval or gate? Naming the layer usually eliminates two options instantly.

---

# Part 8 — The 12 laws (read these on exam morning)

1. **A check becomes a gate only when something refuses to proceed.** Exit code, required check, environment rule.
2. **Detect is not prevent.** Alerts, scans and audit logs report. Push protection, `exit 1` and branch policies stop.
3. **Identity beats a stored secret.** OIDC and managed identity remove the credential; Key Vault only hides it.
4. **Read the last line of the question first.** The constraint is the answer.
5. **Cadence chooses the branching strategy** — and "do they still support an old version?" is the decider.
6. **Same upstream = parallel. Chained = serial.** Draw the graph.
7. **Anything that reads history needs `fetch-depth: 0`** — and the failure is always green.
8. **Code rolls back. Data does not.** Additive first, contract later, forward-fix always.
9. **A gate with no data must refuse.** Missing artifact, missing baseline, empty diff — all report success by default.
10. **Environments gate deployments. Branch protection gates merges.** Never swap them.
11. **Percentiles, never averages.** P95 is what users feel.
12. **The narrowest permission that satisfies the requirement is the answer.** Owner is not "more secure".

---

# Scoring your revision

Cover the page. For each challenge block, say the **bold line** out loud.

| You could say | Do this |
|---|---|
| 45+ of 51 | You are ready. Re-read Part 8 and Domain 4 only. |
| 35–44 | Re-read Parts 1–3 and the blocks you missed. Then go again. |
| 25–34 | Two passes over Domain 3 (challenges 13–38), then retest. |
| Under 25 | One domain per day, in order. Do not jump around. |

:::tip The one thing to remember
You do not need to recall the whole challenge. You need the **bold line** plus **"does this inform or block?"**. Those two together answer most of this exam.
:::
