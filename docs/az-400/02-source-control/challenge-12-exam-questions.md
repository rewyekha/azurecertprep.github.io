---
sidebar_position: 6.5
toc_max_heading_level: 2
title: "Challenge 12: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 12 — AZ-400 exam questions

**48 questions** built only from what Challenge 12 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-12.md`**.

:::danger Read this before you start

**The mono-repo trade in one line: atomic cross-service change versus independent ownership.**

Mono-repo buys you **one commit that renames a shared type and updates all 15 services** (line 30).
Multi-repo buys you **clear ownership, independent releases and small clones** (lines 57–60). Neither is
correct; the exam wants you to name the cost of whichever you pick.

**And the three scaling tools solve three different problems. This is where marks are won and lost.**

**Partial clone** — `--filter=blob:none` — reduces what you **download**. History arrives; file contents
arrive on demand.
**Sparse-checkout** — reduces what you **materialise** on disk. Fewer files in the working tree.
**Scalar** — turns on a **bundle** of Git optimisations at once: partial clone, FSMonitor, commit-graph,
multi-pack index and background maintenance.

**None of them is Git LFS** (Challenge 10). LFS moves large **file content** out of the repository.
These three make a repository that is large because of **history and breadth** usable. **Large files →
LFS. Large history → Scalar.**

**And shallow clone is a fourth, different thing.** `--depth=1` limits **history depth** and still
downloads every file at that depth.

The scenario at line 19: 15 microservices, **8 GB**, **5 years**, **50,000 commits**, and a **25-minute**
clone.

:::

---

# Section A — Multiple choice

---

## Q1

A developer works only on `order-service` in the mono-repo and needs the fastest clone with minimal disk
use. Which combination?

- A. `git clone --depth=1` to limit history
- B. `git clone`, then delete the unwanted directories
- C. `--filter=blob:none --sparse`, then sparse-checkout
- D. `git clone --single-branch --branch=main`

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-12.md`:** lines **189–192** and **624**.

```bash
git clone --filter=blob:none --sparse https://github.com/contoso/platform-monorepo.git
cd platform-monorepo
git sparse-checkout set services/user-service libs/auth-middleware
# Only downloads blobs for the sparse paths (not entire repo history)
```

**Two filters on two axes: `--filter=blob:none` cuts the download, `--sparse` cuts the working tree.**
Together they address both the 8 GB transfer and the disk footprint.

**Why A is the near-miss the exam relies on.** A shallow clone limits **history depth** — you still
download every file in the tree at that depth, across all 15 services. And it breaks `git blame`, `git
log` and anything else needing history.

**Why B downloads everything first**, which is the 25 minutes you were trying to avoid, and **why D still
fetches all blobs** on that one branch.

</details>

---

## Q2

What does `scalar register` enable?

- A. Uploads the repository to a Scalar server
- B. Converts the repository to a Scalar-specific format
- C. Enables server-side partial clone for all clones
- D. A bundle of standard Git performance optimisations

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-12.md`:** lines **103–108** and **635**.

```text
# - Partial clone (only download needed objects)
# - Filesystem monitor (FSMonitor for faster git status)
# - Commit-graph (faster git log and traversal)
# - Multi-pack index (faster object lookups)
# - Background maintenance (prefetch, gc, commit-graph updates)
```

**Every one of those is standard Git and simply off by default.** Scalar is a configuration bundle, not a
new format — which is why line 635 stresses that **the repository stays a normal Git repository any
client can use**.

**Why B is the misconception that makes people avoid it.** Nothing becomes incompatible; `scalar
unregister` (line 134) removes the settings and leaves the repository intact.

**Why C confuses local with server.** Partial clone is requested by the client; Scalar configures **this
clone**, not the server's policy.

</details>

---

## Q3

In multi-repo, team A releases `shared-libs` v2.4.0 with a breaking change. What is the primary
challenge?

- A. Each consumer must independently update, test and release
- B. All other repositories update automatically and may break
- C. Submodules prevent adopting the new version at all
- D. `shared-libs` must be forked for every consuming team

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-12.md`:** lines **66–67** and **646**.

```text
- Cross-service changes require coordinated PRs across repos
- Shared library versioning creates diamond dependency problems
```

**Version drift is the durable cost.** Some services sit on v2.3.0, others move to v2.4.0, and the ones
in between hit the **diamond dependency** problem — two dependencies demanding incompatible versions of
the same library.

**Why B inverts the model.** Nothing updates automatically; a pinned dependency is pinned. That is the
*advantage* of multi-repo (independent release cycles, line 58) and the source of the problem.

**And the mono-repo contrast at line 646 is the whole argument**: the breaking change and every consumer
update land in **one atomic commit**.

</details>

---

## Q4

A developer changes `libs/shared-types/index.ts`. Which Azure Pipelines behaviour is correct?

- A. Every pipeline in the project triggers on the push
- B. No pipeline triggers because the path is a library
- C. Only pipelines whose `paths.include` matches trigger
- D. The pipeline triggers but skips the build stage

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-12.md`:** lines **501–504** and **657**.

```yaml
  paths:
    include:
      - services/order-service/**
      - libs/shared-types/**
```

**A path filter is what makes per-service CI possible in a mono-repo** — without it, all 50 developers
trigger every pipeline on every push (line 40).

**Note that this pipeline includes both its own service and the shared library.** That is deliberate: a
change to `shared-types` **can** break `order-service`, so its pipeline must run.

**Which is the design tension worth understanding.** Filter too narrowly and you miss real breakage;
filter too broadly and you rebuild everything.

</details>

---

## Q5

Sparse-checkout is set to `services/order-service` and the build fails with "Cannot find module
`@contoso/shared-types`". What is the fix?

- A. `git sparse-checkout disable` to restore the tree
- B. `git sparse-checkout add libs/shared-types` and utils
- C. Re-clone the repository without sparse-checkout
- D. Install `@contoso/shared-types` from the npm registry

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-12.md`:** lines **546–547**.

```bash
git sparse-checkout add libs/shared-types libs/common-utils
```

**`add` extends the current set; `set` replaces it** (line 182 shows `set` removing a path). Using `set`
here would have to re-list `services/order-service` or lose it.

**Why A works and gives up the benefit.** Disabling brings back all 8 GB of working tree — correct as a
last resort, wasteful as a fix.

**And the durable answer is the sparse profile at lines 559–564**: record the service's full path
dependencies in a file so the next developer does not rediscover them.

</details>

---

## Q6

A submodule shows as modified after `git pull`, and its directory contains old code. What is the fix?

- A. `git submodule update --init --recursive`
- B. `git pull` run inside the submodule directory
- C. Delete the submodule and add it again from scratch
- D. `git reset --hard HEAD` in the parent repository

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-12.md`:** lines **594–598**.

```bash
git submodule update --init --recursive
```

**The parent repository stores a *commit pointer*, and `git pull` updates the pointer without moving the
submodule's working tree.** So `git status` reports a difference between what the parent expects and what
is checked out.

**Why B moves it to the wrong place.** Pulling inside the submodule advances it to the **latest** commit
on its branch — which is probably **not** the pinned commit the parent expects, so the difference remains
and now points the other way.

**And that is the distinction between the two commands** (line 606): `update` moves to the **pinned**
commit; `update --remote` moves to the **latest** and is followed by committing the new pointer.

</details>

---

## Q7

What does `--filter=blob:none` do?

- A. It skips all history beyond the most recent commit
- B. It defers blobs while fetching every commit and tree
- C. It excludes binary files from the clone entirely
- D. It filters out blobs larger than a size threshold

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-12.md`:** lines **189–192** and **624**.

**You get the full commit graph and every tree — so `git log`, `git blame` and `git bisect` work — and
blobs arrive lazily when something actually reads a file.**

**Which is exactly why it beats a shallow clone for a developer** (Q1). Shallow saves history and breaks
the tools that need it; partial clone keeps history and defers the bulk.

**The trade is that lazy fetching needs the remote to be reachable.** A partial clone offline will fail
when it tries to materialise a blob it never downloaded.

</details>

---

## Q8

What does `git sparse-checkout init --cone` provide over pattern mode?

- A. More flexible matching with arbitrary glob patterns
- B. Automatic detection of a service's path dependencies
- C. Faster performance, using directory-level patterns
- D. Server-side filtering of the objects that are sent

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-12.md`:** lines **161–162**.

```bash
# Initialize sparse-checkout in cone mode (faster than pattern mode)
git sparse-checkout init --cone
```

**Cone mode restricts patterns to whole directories**, which lets Git decide inclusion by walking the
tree rather than testing every path against every pattern.

**On a repository with 50,000 commits and 15 services that difference is measurable** — pattern mode gets
slower as the file count grows, cone mode does not.

**Why B is the gap this challenge leaves to you.** Nothing detects that `order-service` needs
`shared-types`; that is Break scenario 1, and the answer is a documented profile (Q5).

</details>

---

## Q9

Which Scalar command generates a diagnostic bundle?

- A. `scalar run`
- B. `scalar list`
- C. `scalar cache-server`
- D. `scalar diagnose`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-12.md`:** lines **132–134**.

```bash
scalar diagnose    # Generate diagnostic zip for troubleshooting
scalar cache-server --set https://cache.contoso.internal  # Use a cache server
scalar unregister  # Remove Scalar from this repo
```

**Four commands, four jobs.** `run` executes the maintenance tasks on demand (line 128), `list` shows
registered repositories (line 115), `cache-server` points at a nearby object cache, and `unregister`
removes the configuration.

**The cache server is the one worth remembering for a distributed team** — it puts objects closer to the
developer than the origin, which matters when a clone is 8 GB.

</details>

---

## Q10

Which maintenance tasks does `scalar run` execute?

- A. `gc`, `fsck`, `prune-packed`, `pack-refs`
- B. `prefetch`, `commit-graph`, `loose-objects`, `incremental-repack`
- C. `fetch --all`, `pull --rebase`, `push --mirror`, `remote prune`
- D. `commit-graph`, `pack-refs`, `reflog expire`, `gc --auto`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-12.md`:** lines **128–129**.

```bash
scalar run
# Runs: prefetch, commit-graph, loose-objects, incremental-repack
```

**All four are incremental by design, which is the point.** `maintenance.strategy=incremental` (line 125)
means small frequent work rather than one long `gc` that blocks the developer.

**`prefetch` is the one with the biggest felt effect** — it downloads new objects in the background, so
the developer's next `git fetch` is nearly instant instead of pulling a day of commits.

**And `maintenance.auto=false`** (line 124) turns off Git's own opportunistic garbage collection, because
the scheduled tasks now handle it.

</details>

---

## Q11

What does `.gitmodules` record?

- A. The submodule's path, URL and tracked branch
- B. The submodule's currently pinned commit SHA
- C. The submodule's files and their contents
- D. The parent repository's own dependencies

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-12.md`:** lines **238–242**.

```text
[submodule "libs/shared"]
    path = libs/shared
    url = https://github.com/contoso/shared-libs.git
    branch = main
```

**The *commit* is not in `.gitmodules` — it is stored in the parent's tree as a special entry**, which is
why `git add libs/shared` (line 248) commits a pointer rather than files.

**That split explains Break scenario 2** (Q6). `.gitmodules` says *where* the submodule comes from; the
tree entry says *which commit*, and only the tree entry moves when you pin a version.

</details>

---

## Q12

How do you pin a submodule to a specific version?

- A. Edit the `branch` entry in `.gitmodules` to the tag
- B. `git submodule update --remote` run in the parent
- C. Use a tag name instead of a branch in `.gitmodules`
- D. Check out the tag inside, then `git add` the path

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-12.md`:** lines **245–249**.

```bash
cd libs/shared
git checkout v2.3.0
cd ..
git add libs/shared
git commit -m "chore: pin shared-libs to v2.3.0"
```

**Move the submodule's HEAD, then record it in the parent.** Both halves are required — checking out the
tag alone leaves the parent still pointing at the old commit.

**Why B does the opposite.** `--remote` moves to the **latest** on the tracked branch, which is the
un-pinning operation (Q6).

**And why D is the trap.** `branch = main` in `.gitmodules` only tells `--remote` where to look; **the
pinned commit is what a clone actually gets.**

</details>

---

## Q13

What does `git clone --recurse-submodules` do?

- A. Clones only the submodules, not the parent
- B. Clones the parent and updates all submodules in one step
- C. Clones the parent and leaves submodule directories empty
- D. Converts each submodule into a normal directory

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-12.md`:** line **252**.

**Without it, submodule directories are created empty** — and the build fails with missing files,
which is the commonest first-day experience on a submodule repository.

**The recovery is two commands** (lines 255–256): `git submodule init` then `git submodule update`, or
the combined `update --init --recursive` from Q6.

</details>

---

## Q14

In Azure Pipelines, how do you check out a second repository?

- A. Declare it under `resources.repositories`, then `checkout:`
- B. A second `checkout: self` step pointing at the other repo
- C. A `git clone` command run inside a script step
- D. A submodule reference committed to the parent

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-12.md`:** lines **287–318**.

```yaml
resources:
  repositories:
    - repository: shared-libs
      type: git
      name: Contoso-Platform/shared-libs
      ref: refs/tags/v2.3.0
```

```yaml
  - checkout: self
    path: s/payment-service
  - checkout: shared-libs
    path: s/shared-libs
```

**Declare, then check out** — and note that once you check out anything other than `self`, **you must
also explicitly check out `self`**, because the implicit checkout stops applying.

**`ref: refs/tags/v2.3.0` pins the second repository to a tag**, which is the multi-repo equivalent of
pinning a submodule (Q12).

**And `type: github` with an `endpoint`** (lines 298–300) is how you reach a GitHub repository from Azure
Pipelines — the endpoint names a service connection.

</details>

---

## Q15

In GitHub Actions, how do you check out a **private** second repository?

- A. `actions/checkout` with `repository:` alone, no token
- B. A submodule reference pointing at the private repository
- C. `actions/checkout` with `repository:`, `path:`, `token:`
- D. `git clone` with the default `GITHUB_TOKEN`

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-12.md`:** lines **374–379**.

```yaml
      - uses: actions/checkout@v4
        with:
          repository: contoso/infrastructure
          token: ${{ secrets.CROSS_REPO_TOKEN }}
          path: infrastructure
```

**Compare with the public repository above it** (lines 368–372), which needs **no token** at all.

**And this is Challenge 40's rule appearing again: `GITHUB_TOKEN` cannot leave its repository.** No
`permissions` value changes that, which is why D fails and why a separate credential — a fine-grained PAT
or a GitHub App token — is required.

**`path:` is mandatory once you check out more than one repository**, or the second overwrites the first.

</details>

---

## Q16

What does `dorny/paths-filter` provide in the path-triggered workflow?

- A. It skips the checkout step when no watched paths match
- B. It caches dependencies between workflow runs
- C. It merges the changed branches before building
- D. Per-filter boolean outputs for downstream `if:` gates

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-12.md`:** lines **411–437**.

```yaml
    outputs:
      order-service: ${{ steps.changes.outputs['order-service'] }}
```

```yaml
  build-order-service:
    needs: detect-changes
    if: needs['detect-changes'].outputs['order-service'] == 'true'
```

**One detection job, many gated build jobs.** The filter evaluates once and every service job reads its
own boolean.

**Why this is more capable than Azure Pipelines' `trigger.paths`** (line 501): a path trigger decides
whether the **whole pipeline** runs; `paths-filter` decides **which jobs within one run** execute — so a
change touching two services builds exactly those two.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** are mono-repo advantages? (Choose three.)

- A. Fine-grained per-service access control
- B. Atomic cross-service changes in one commit
- C. Independent release cycles per service
- D. Single source of truth for shared libraries
- E. Small, fast clones for every developer
- F. Unified CI/CD configuration across services

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-12.md`:** lines **30–32**.

**A, C and E are the multi-repo column** (lines 58–60), and each is listed as a mono-repo
**disadvantage** on the other side: permission granularity is limited (line 41), all teams share a
branching strategy (line 45), and 8 GB takes 25 minutes (line 39).

**The lists mirror each other on purpose.** Every mono-repo advantage has a multi-repo disadvantage
opposite it, which is why "which is better" is never the question — **"what does this organisation need
most" is.**

</details>

---

## Q18

Which **three** are multi-repo disadvantages? (Choose three.)

- A. Cross-service changes need coordinated PRs across repos
- B. Repository size makes every clone slow
- C. Shared library versioning creates diamond dependencies
- D. Merge conflicts on files many teams share
- E. Refactoring across service boundaries is painful
- F. All teams must agree on one branching strategy

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-12.md`:** lines **66–72**.

**B, D and F are the mono-repo disadvantages** (lines 39–45) — the same mirroring as Q17, offered in the
other direction.

**And all three correct answers are the same underlying cost:** a change that spans services is cheap in
one repository and expensive in fifteen.

</details>

---

## Q19

Which **three** optimisations does Scalar enable? (Choose three.)

- A. Git LFS for large binary asset files
- B. FSMonitor for faster `git status`
- C. Server-side filtering of the objects sent
- D. Commit-graph for faster history traversal
- E. Automatic sparse-checkout profiles
- F. Multi-pack index for faster object lookup

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-12.md`:** lines **104–108**.

**Partial clone and background maintenance are the other two** — five optimisations in the bundle.

**Why A is the boundary this challenge and Challenge 10 both defend.** LFS is a separate tool for large
**files**; Scalar is for large **history and breadth**. Enabling Scalar does nothing about a 500 MB `.fbx`
and enabling LFS does nothing about 50,000 commits.

**Why E is the gap you fill yourself** (Q8, Q26) — the team profiles at lines 198–224 are hand-written
scripts.

</details>

---

## Q20

Which **two** reduce what a clone **downloads**? (Choose two.)

- A. `git sparse-checkout set` on the clone
- B. `scalar unregister` after cloning
- C. `--filter=blob:none` on the clone
- D. `--recurse-submodules` on the clone
- E. `--depth=1` on the clone command

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-12.md`:** lines **189** and **624**.

**Both reduce transfer, on different axes.** `--filter=blob:none` skips **file contents**; `--depth=1`
skips **history**.

**Why A is the third axis and the reason the question says "download".** Sparse-checkout reduces what is
**written to disk**, not what is fetched — a full clone with sparse-checkout has still downloaded
everything.

**Which is why Q1's answer combines a filter *and* sparse-checkout**: one for the network, one for the
disk.

</details>

---

## Q21

Which **two** are true of `git submodule update --remote`? (Choose two.)

- A. It moves the submodule to the parent's pinned commit
- B. It moves the submodule to the latest on its branch
- C. It updates the `.gitmodules` file automatically
- D. The new pointer must be committed in the parent
- E. It requires `--init` to be passed on every invocation

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-12.md`:** lines **259–261**.

```bash
git submodule update --remote libs/shared
git add libs/shared
git commit -m "chore: update shared-libs to latest"
```

**Two steps, and forgetting the second leaves the submodule ahead of what the repository records** — so
every colleague still gets the old commit.

**Why A is `update` *without* `--remote`** (line 596), which is the recovery in Break scenario 2. **The
two flags move in opposite directions**, and that pairing is the most testable thing about submodules.

</details>

---

## Q22

Which **two** are required to check out multiple repositories in Azure Pipelines? (Choose two.)

- A. A `resources.repositories` entry per external repository
- B. A submodule reference for each external repository
- C. A separate pipeline definition per repository
- D. An explicit `checkout: self` alongside the others
- E. `persistCredentials: true` on every checkout

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-12.md`:** lines **287–318**.

**D is the one that catches people.** Azure Pipelines checks out `self` implicitly — **until you add any
`checkout:` step**, at which point nothing is implicit and `self` must be listed too.

**The symptom is a build that cannot find its own source**, which reads as a path problem and is not.

**And `path:` on each** (lines 308–318) is what puts them side by side under `$(Pipeline.Workspace)/s/`
so one can reference another (line 333).

</details>

---

## Q23

Which **two** does the path-filtered workflow do when `libs/**` changes? (Choose two.)

- A. It builds nothing, since no service folder changed
- B. It skips the change-detection job entirely
- C. It sets the `shared-libs` output to `true`
- D. It rebuilds only `order-service` and nothing else
- E. It runs a matrix job rebuilding all dependent services

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-12.md`:** lines **429–430** and **471–482**.

```yaml
  # If shared libs change, rebuild ALL dependent services
  build-all-on-shared-change:
    if: needs['detect-changes'].outputs['shared-libs'] == 'true'
    strategy:
      matrix:
        service: [order-service, payment-service, user-service, catalog-service, shipping-service]
```

**This is the honest half of path-based CI.** Filtering is an optimisation for the common case; when the
shared library moves, the optimisation must **stand down** and rebuild everything that depends on it.

**A path-filtered pipeline with no shared-library fan-out is quietly unsafe** — it will happily merge a
`libs/` change that breaks four services it never built.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must decide between mono-repo and multi-repo for 15 services, and make the chosen
approach workable at 8 GB, 50,000 commits and a 25-minute clone.

---

## Q24

**Proposed solution:** Keep the mono-repo for atomic cross-service changes. Have developers clone with
`--filter=blob:none --sparse` and set a team sparse profile covering their service plus its library
dependencies. Register the repository with Scalar for FSMonitor, commit-graph, multi-pack index and
background maintenance. Use `dorny/paths-filter` so each service builds only when its own paths or its
libraries change, with a matrix job rebuilding all dependent services when `libs/**` changes.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-12.md`:** lines **30–36**, **189–192**, **198–224**, **101–108**, **411–437**,
**471–482**.

| Problem at line 19 | Mechanism |
|---|---|
| 8 GB, 25-minute clone | Partial clone + sparse-checkout |
| Slow local Git operations | Scalar: FSMonitor, commit-graph, multi-pack index |
| Everyone triggers every build | `paths-filter` gating per service |
| A shared-library change could break consumers | Matrix rebuild of all dependents |
| Cross-service refactoring | Retained — it is why the mono-repo was kept |

**The last row of the matrix job is what makes the filtering safe** (Q23). Without it the optimisation
becomes a hole.

</details>

---

## Q25

**Proposed solution:** Split into 15 repositories immediately. Share code with submodules pinned to tags.
Let each team choose its own branching strategy and tooling. Coordinate cross-service changes with a
spreadsheet of PR links. Clone shallowly with `--depth=1` to keep it fast.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures, and the first is that the decision is unargued.**

**The CTO asked for a data-driven recommendation** (line 19). "Split immediately" answers the question
without weighing the 8 GB clone against atomic refactoring — and the challenge's two analyses exist
precisely to make that comparison.

**Submodules pinned to tags reintroduce the diamond dependency problem** (line 67). Fifteen repositories
pinned to different `shared-libs` versions is version drift by construction.

**"Coordinate with a spreadsheet"** is the multi-repo disadvantage at line 66 restated as a plan. A
cross-service rename becomes 15 PRs that must merge in an order nobody can enforce.

**And `--depth=1` breaks history for every developer** — no `blame`, no `bisect`, no `log` past the tip —
which is a large price for a saving that partial clone provides without it (Q20).

</details>

---

## Q26

**Proposed solution:** Keep the mono-repo. Clone with `--filter=blob:none --sparse` and set a sparse
profile per team. Register with Scalar. Use `paths-filter` per service with a matrix rebuild when
`libs/**` changes. Because sparse-checkout already limits what each developer sees, let each developer
set their own paths as they discover what they need, rather than maintaining shared profiles.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**"As they discover what they need" is Break scenario 1 as a policy.** The developer sets
`services/order-service`, the build fails with `Cannot find module '@contoso/shared-types'` (line 531),
and they debug a **missing-file error that looks like a broken dependency**.

**And the failure is worse than an inconvenience, because the symptom points the wrong way.** `npm run
build` reports a missing module — so the developer's first instinct is to reinstall packages, check
`package.json`, or suspect the registry. Nothing in that error mentions sparse-checkout.

**Multiply it by 15 services and 50 developers** (line 40) and it recurs on every onboarding, every time
someone switches team, and every time a service gains a new library dependency.

**The challenge's answer is a committed profile** (lines 559–564), and the reasoning is in its own
comment: *"Document dependencies in a sparse profile for the team."* **A path list is a dependency
declaration**, and it belongs in the repository beside the code that needs it — not in each developer's
local configuration.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — the trade

| # | Statement | Answer |
|---|---|---|
| 1 | A mono-repo allows atomic cross-service changes |  |
| 2 | A mono-repo gives fine-grained per-service access control |  |
| 3 | Multi-repo gives independent release cycles |  |
| 4 | Multi-repo makes cross-service refactoring easier |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A mono-repo allows atomic cross-service changes | **Yes** |
| 2 | A mono-repo gives fine-grained per-service access control | **No** |
| 3 | Multi-repo gives independent release cycles | **Yes** |
| 4 | Multi-repo makes cross-service refactoring easier | **No** |

**In `challenge-12.md`:** lines **30**, **41**, **58**, **72**.

Rows 2 and 4 are each column's headline weakness, and the exam offers them as strengths of the other
model.

</details>

---

## Q28 — scaling tools

| # | Statement | Answer |
|---|---|---|
| 1 | `--filter=blob:none` keeps full commit history |  |
| 2 | `--depth=1` reduces the number of files downloaded |  |
| 3 | Sparse-checkout reduces the working tree, not the download |  |
| 4 | Scalar changes the repository format |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `--filter=blob:none` keeps full commit history | **Yes** |
| 2 | `--depth=1` reduces the number of files downloaded | **No** |
| 3 | Sparse-checkout reduces the working tree, not the download | **Yes** |
| 4 | Scalar changes the repository format | **No** |

**In `challenge-12.md`:** lines **624**, **624**, **192**, **635**.

Row 2 is the distinction Q1 turns on: **shallow limits history depth and still fetches every file at that
depth**.

Row 4: a Scalar repository is a normal Git repository, and `scalar unregister` reverses it.

</details>

---

## Q29 — submodules

| # | Statement | Answer |
|---|---|---|
| 1 | `.gitmodules` stores the path, URL and tracked branch |  |
| 2 | `.gitmodules` stores the pinned commit |  |
| 3 | `git submodule update` moves to the parent's pinned commit |  |
| 4 | `git submodule update --remote` moves to the latest on the branch |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `.gitmodules` stores the path, URL and tracked branch | **Yes** |
| 2 | `.gitmodules` stores the pinned commit | **No** |
| 3 | `git submodule update` moves to the parent's pinned commit | **Yes** |
| 4 | `git submodule update --remote` moves to the latest on the branch | **Yes** |

**In `challenge-12.md`:** lines **238–242**, **248**, **596**, **259**.

Row 2 is the fact that explains Break scenario 2: **the commit lives in the parent's tree**, which is why
`git add libs/shared` commits a pointer.

Rows 3 and 4 are the opposite-direction pair (Q21).

</details>

---

## Q30 — pipelines

| # | Statement | Answer |
|---|---|---|
| 1 | Adding any `checkout:` step means `self` must be listed explicitly |  |
| 2 | A public second repository needs a token in GitHub Actions |  |
| 3 | A private second repository needs a PAT or App token |  |
| 4 | Azure Pipelines `trigger.paths` gates individual jobs |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Adding any `checkout:` step means `self` must be listed explicitly | **Yes** |
| 2 | A public second repository needs a token in GitHub Actions | **No** |
| 3 | A private second repository needs a PAT or App token | **Yes** |
| 4 | Azure Pipelines `trigger.paths` gates individual jobs | **No** |

**In `challenge-12.md`:** lines **306–318**, **368–372**, **374–378**, **501**.

Row 4 is the platform difference (Q16): a path trigger gates **the whole pipeline**; `paths-filter` gates
**jobs within one run**.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each problem to the tool that solves it.

| Problem | Tool |
|---|---|
| 8 GB download on every clone |  |
| A working tree with 15 services you do not need |  |
| `git status` takes seconds on a huge tree |  |
| `git log` is slow across 50,000 commits |  |
| A 500 MB binary asset in the repository |  |
| Every push builds all 15 services |  |

**Options:** Git LFS (Challenge 10) · Partial clone — `--filter=blob:none` · Path filters · Scalar — commit-graph · Scalar — FSMonitor · Sparse-checkout

<details>
<summary>Show answer</summary>

| Problem | Tool |
|---|---|
| 8 GB download on every clone | **Partial clone — `--filter=blob:none`** |
| A working tree with 15 services you do not need | **Sparse-checkout** |
| `git status` takes seconds on a huge tree | **Scalar — FSMonitor** |
| `git log` is slow across 50,000 commits | **Scalar — commit-graph** |
| A 500 MB binary asset in the repository | **Git LFS** (Challenge 10) |
| Every push builds all 15 services | **Path filters** |

**In `challenge-12.md`:** lines **189**, **165**, **105**, **106**, **19**, **421–433**.

**Row 5 is the boundary the exam attacks.** Everything else in this table addresses **history and
breadth**; only LFS addresses **file size**.

</details>

---

## Q32

Match each mono-repo cost to its mitigation.

| Cost | Mitigation |
|---|---|
| 25-minute clone |  |
| Slow local Git operations |  |
| Every push triggers every build |  |
| A shared-library change may break consumers |  |
| Developers do not know which paths they need |  |
| Limited per-service access control |  |

**Options:** Committed sparse profiles · Matrix rebuild of dependents · Partial clone + sparse-checkout · `paths-filter` / `trigger.paths` · Scalar · Unmitigated — a genuine trade-off

<details>
<summary>Show answer</summary>

| Cost | Mitigation |
|---|---|
| 25-minute clone | **Partial clone + sparse-checkout** |
| Slow local Git operations | **Scalar** |
| Every push triggers every build | **`paths-filter` / `trigger.paths`** |
| A shared-library change may break consumers | **Matrix rebuild of dependents** |
| Developers do not know which paths they need | **Committed sparse profiles** |
| Limited per-service access control | **Unmitigated — a genuine trade-off** |

**In `challenge-12.md`:** lines **189–192**, **101–108**, **411–437**, **471–482**, **559–564**,
**41**.

**The last row matters.** Five of the six costs have technical mitigations; **permission granularity does
not**, and an honest recommendation to the CTO says so rather than pretending otherwise.

</details>

---

## Q33

Arrange the steps for a developer to start work on `order-service` in the mono-repo.

**Items:** `git sparse-checkout set` the service and its libraries · `scalar register` · Clone with
`--filter=blob:none --sparse` · Read the team's sparse profile

<details>
<summary>Show answer</summary>

### Answer

1. Clone with `--filter=blob:none --sparse` — line **189**
2. Read the team's sparse profile — lines **198–207**
3. `git sparse-checkout set` the service and its libraries — line **165**
4. `scalar register` — line **101**

**Step 2 before step 3 is the whole lesson of Break scenario 1.** Setting the paths from memory produces
a working tree missing `libs/shared-types`, and a build error that names a module rather than a path
(Q26).

**And `--sparse` at clone time already puts you in sparse mode** with only the root directory checked
out — so step 3 is broadening a narrow tree, not narrowing a full one.

**Step 4 can come at any point**, but registering after the clone means the background maintenance starts
against a repository that already exists locally.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| `Cannot find module '@contoso/shared-types'` |  |
| A submodule shows modified with old content |  |
| A pipeline cannot find its own source |  |
| A cross-repo checkout fails on a private repository |  |
| A `libs/` change merges and breaks four services |  |
| `git blame` returns nothing useful |  |

**Options:** `checkout: self` omitted after adding others · No matrix rebuild of dependents · No PAT or App token supplied · Pointer updated, submodule not updated · Shallow clone instead of partial clone · Sparse-checkout missing the library path

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| `Cannot find module '@contoso/shared-types'` | **Sparse-checkout missing the library path** |
| A submodule shows modified with old content | **Pointer updated, submodule not updated** |
| A pipeline cannot find its own source | **`checkout: self` omitted after adding others** |
| A cross-repo checkout fails on a private repository | **No PAT or App token supplied** |
| A `libs/` change merges and breaks four services | **No matrix rebuild of dependents** |
| `git blame` returns nothing useful | **Shallow clone instead of partial clone** |

**In `challenge-12.md`:** lines **531**, **576**, **306**, **378**, **471**, **624**.

**The first and last are the two that mislead most.** A missing module reads as a dependency problem, and
a broken `blame` reads as a Git bug — neither points at the clone strategy that caused it.

</details>

---

## Q35

Match each checkout mechanism to its platform and syntax.

| Mechanism | Platform — syntax |
|---|---|
| Declare an external repository |  |
| Check out a declared repository |  |
| Check out another repository |  |
| Reach a private repository |  |
| Reach GitHub from Azure Pipelines |  |
| Pin the external repository |  |

**Options:** Azure Pipelines — `- checkout: <alias>` · Azure Pipelines — `resources.repositories` · GitHub Actions — `actions/checkout` with `repository:` · GitHub Actions — `token: secrets.CROSS_REPO_TOKEN` · `ref: refs/tags/v2.3.0` · `type: github` + `endpoint:`

<details>
<summary>Show answer</summary>

| Mechanism | Platform — syntax |
|---|---|
| Declare an external repository | **Azure Pipelines — `resources.repositories`** |
| Check out a declared repository | **Azure Pipelines — `- checkout: <alias>`** |
| Check out another repository | **GitHub Actions — `actions/checkout` with `repository:`** |
| Reach a private repository | **GitHub Actions — `token: secrets.CROSS_REPO_TOKEN`** |
| Reach GitHub from Azure Pipelines | **`type: github` + `endpoint:`** |
| Pin the external repository | **`ref: refs/tags/v2.3.0`** |

**In `challenge-12.md`:** lines **287–292**, **312**, **368–371**, **378**, **298–300**, **292**.

**Two platforms, one intent** — and the exam swaps the syntax. Azure Pipelines **declares then checks
out**; Actions checks out directly with a `path:` for each.

</details>

---

# Section F — Hot area

---

## Q36

```bash
git clone --[BLANK 1] --[BLANK 2] https://github.com/contoso/platform-monorepo.git
cd platform-monorepo
git sparse-checkout [BLANK 3] services/user-service libs/auth-middleware
```

Requirement: minimal download **and** minimal disk, with full history preserved.

- **BLANK 1:** `depth=1` / `single-branch` / `filter=blob:none` / `bare`
- **BLANK 2:** `recurse-submodules` / `mirror` / `no-checkout` / `sparse`
- **BLANK 3:** `add` / `set` / `init` / `list`

<details>
<summary>Show answer</summary>

### Answer: `filter=blob:none`, `sparse`, `set`

**In `challenge-12.md`:** lines **189–191**.

**`depth=1` is the distractor that fails the "full history" clause** (Q20). It saves the history you were
told to keep.

**And `set` versus `add`** (Q5): `set` **replaces** the path list, which is correct on a fresh clone
where the list is empty. Use `add` to extend an existing selection.

</details>

---

## Q37

```bash
scalar [BLANK 1]
scalar [BLANK 2]
# Runs: prefetch, commit-graph, loose-objects, incremental-repack
scalar [BLANK 3]
# Generate diagnostic zip for troubleshooting
```

- **BLANK 1:** `install` / `enable` / `register` / `init`
- **BLANK 2:** `maintain` / `run` / `gc` / `sync`
- **BLANK 3:** `debug` / `doctor` / `report` / `diagnose`

<details>
<summary>Show answer</summary>

### Answer: `register`, `run`, `diagnose`

**In `challenge-12.md`:** lines **101**, **128**, **132**.

**`register` configures an existing repository; `scalar clone`** (line 111) **does both at once** for a
new one. Knowing there are two entry points is worth a mark.

**And `unregister`** (line 134) removes the configuration without touching the repository — which is the
answer to "how do we back this out".

</details>

---

## Q38

```bash
git sparse-checkout init --[BLANK 1]
git sparse-checkout set services/order-service libs/shared-types
git sparse-checkout [BLANK 2] services/payment-service
git sparse-checkout [BLANK 3]
```

Requirement: fast directory-based mode, then temporarily include another service, then return to the full
tree.

- **BLANK 1:** `pattern` / `cone` / `full` / `strict`
- **BLANK 2:** `set` / `include` / `add` / `append`
- **BLANK 3:** `reset` / `disable` / `clear` / `off`

<details>
<summary>Show answer</summary>

### Answer: `cone`, `add`, `disable`

**In `challenge-12.md`:** lines **162**, **179**, **186**.

**`set` in BLANK 2 would *replace* the list** and remove `order-service` — the exact mistake that makes
sparse-checkout feel unpredictable.

**And `disable` restores the entire working tree** (line 186), which on this repository means all 15
services reappear.

</details>

---

## Q39

```bash
cd order-service
git submodule add https://github.com/contoso/shared-libs.git libs/shared

cd libs/shared
git checkout v2.3.0
cd ..
git [BLANK 1] libs/shared
git commit -m "chore: pin shared-libs to v2.3.0"

# Later, someone else clones:
git clone --[BLANK 2] https://github.com/contoso/order-service.git
```

- **BLANK 1:** `submodule update` / `commit` / `add` / `push`
- **BLANK 2:** `submodules` / `recurse-submodules` / `with-submodules` / `init-submodules`

<details>
<summary>Show answer</summary>

### Answer: `add`, `recurse-submodules`

**In `challenge-12.md`:** lines **248** and **252**.

**`git add` on a submodule path stages the *commit pointer*, not files** — which is the mechanic behind
`.gitmodules` holding the URL and the tree holding the SHA (Q11).

**And without `--recurse-submodules` the directory clones empty** (Q13), which is the commonest
first-day failure on a submodule repository.

</details>

---

## Q40

```yaml
resources:
  [BLANK 1]:
    - repository: shared-libs
      type: git
      name: Contoso-Platform/shared-libs
      [BLANK 2]: refs/tags/v2.3.0

steps:
  - checkout: [BLANK 3]
    path: s/payment-service
  - checkout: shared-libs
    path: s/shared-libs
```

- **BLANK 1:** `repos` / `sources` / `containers` / `repositories`
- **BLANK 2:** `branch` / `version` / `ref` / `tag`
- **BLANK 3:** `primary` / `self` / `main` / `source`

<details>
<summary>Show answer</summary>

### Answer: `repositories`, `ref`, `self`

**In `challenge-12.md`:** lines **288**, **292**, **307**.

**BLANK 3 is the one that breaks builds.** Azure Pipelines checks out `self` implicitly **until you add
any explicit checkout** — at which point omitting it leaves the pipeline with no source of its own
(Q22).

**And `ref:` takes a full ref**, so `refs/tags/v2.3.0` for a tag and `refs/heads/main` for a branch (line
296) — not a bare name.

</details>

---

## Q41

```yaml
      - uses: dorny/paths-filter@v3
        id: changes
        with:
          filters: |
            order-service:
              - 'services/order-service/**'
              - '[BLANK 1]'

  build-order-service:
    needs: detect-changes
    if: needs['detect-changes'].outputs['order-service'] == '[BLANK 2]'
```

Requirement: rebuild `order-service` when its own code **or** the shared types change.

- **BLANK 1:** `libs/**` / `libs/shared-types/**` / `services/**` / `**`
- **BLANK 2:** `changed` / `yes` / `true` / `1`

<details>
<summary>Show answer</summary>

### Answer: `libs/shared-types/**`, `true`

**In `challenge-12.md`:** lines **423–425** and **437**.

**`libs/**` would be too broad** — it would rebuild `order-service` whenever any unrelated library
changed, which erodes the saving the filter exists for.

**And the output is the *string* `'true'`**, which is why the comparison uses quotes. A boolean comparison
without them silently never matches.

</details>

---

# Section G — Case study

## Case study: Contoso e-commerce repository strategy

### Background

Contoso Ltd operates **15 microservices**. Some teams want a **mono-repo** for easier cross-service
refactoring and atomic changes; others want **separate repositories** for clear ownership, independent
deployments and smaller clones. The repository is **8 GB** with **5 years of history and 50,000
commits**, and cloning takes **25 minutes**. The CTO wants a **data-driven recommendation** with
implementation details.

### Requirements

**Decision**

- The recommendation must state what is gained and what is given up
- The costs of the chosen model must have named mitigations, and any cost with no mitigation must be
  stated as such

**Developer experience**

- A developer working on one service must not download or materialise all 15
- Local Git operations must stay fast at 50,000 commits
- A new starter must not have to discover which paths their service needs

**CI**

- A push must build only the services it can affect
- A change to a shared library must rebuild every dependent service

---

## Q42

Which model should Contoso choose, and what is the deciding factor?

- A. Mono-repo — atomic refactoring, most costs mitigated
- B. Multi-repo — clones are faster for every single developer
- C. Mono-repo — it is simpler for the platform team
- D. Multi-repo — teams prefer their own autonomy

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-12.md`:** lines **30–36**, **39–45**, and the mitigations at **189–192**, **101–108**,
**411–482**.

**The requirement asks for gains, losses and mitigations** — so an answer that names only a benefit fails
regardless of which model it picks.

**And the honest part is the exception.** Clone size, Git performance and CI cost all have technical
answers; **per-service access control does not** (line 41), and saying so is what makes the
recommendation credible.

**Why B and D are the arguments rather than the analysis.** Both are true statements from the multi-repo
column, offered without weighing the diamond dependency problem (line 67) that 15 services sharing
`shared-libs` will hit.

</details>

---

## Q43

How does a developer avoid downloading and materialising all 15 services?

- A. `git clone --depth=1` to limit history to one commit
- B. Clone everything and delete the other fourteen services
- C. `scalar register` on a full clone of the repository
- D. `--filter=blob:none --sparse`, then sparse-checkout

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-12.md`:** lines **189–192**.

**Two verbs in the requirement — "download" and "materialise" — and two mechanisms answer them** (Q20).

**Why C helps and does not answer this.** Scalar makes local operations fast (the *next* requirement) and
its `scalar clone` does include partial clone — but `register` on its own configures an existing clone
rather than avoiding the 8 GB.

**Why B is the 25 minutes you were avoiding**, and A loses history.

</details>

---

## Q44

How are local Git operations kept fast at 50,000 commits?

- A. `git gc --aggressive` run weekly on each clone
- B. Shallow clones for every developer machine
- C. `scalar register` for FSMonitor and commit-graph
- D. Deleting old branches to shrink the ref count

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-12.md`:** lines **101–108** and **124–129**.

**Each optimisation targets a different slow command.** FSMonitor for `git status`, commit-graph for
`git log` and traversal, multi-pack index for object lookup, prefetch for `git fetch`.

**Why A is the manual, blocking version of what Scalar schedules.** `maintenance.strategy=incremental`
(line 125) does the same work in small background pieces rather than one long pause.

**Why B trades the history developers need** for a speed-up partial clone provides without it.

</details>

---

## Q45

Which **two** satisfy "a new starter must not have to discover which paths their service needs"?
(Choose two.)

- A. Letting developers add paths as build errors reveal them
- B. Committed sparse profiles per team listing the paths
- C. Disabling sparse-checkout for all new starters
- D. A `.sparse-profiles/<service>.txt` file beside the code
- E. A wiki page listing each service's dependencies

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-12.md`:** lines **198–224** and **559–564**.

```text
services/order-service
libs/shared-types
libs/common-utils
infrastructure/kubernetes/order-service
```

**Both are the same idea at two levels of formality** — an executable profile script and a plain path
list — and both live **in the repository**, so they change with the dependencies they describe.

**Why A is Break scenario 1 as a policy** (Q26), and its failure mode misleads: the error names a
**module**, not a path.

**Why E decays.** A wiki page describing path dependencies is stale the first time a service gains a
library.

</details>

---

## Q46

Which **two** satisfy the CI requirements? (Choose two.)

- A. `dorny/paths-filter` producing per-service booleans for gating
- B. Building every service on every push to the repository
- C. Building only the service whose folder changed, always
- D. A matrix job rebuilding all dependents when `libs/**` changes
- E. Nightly full builds of every service in the repo

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-12.md`:** lines **411–437** and **471–482**.

**Two requirements, and C satisfies only the first while breaking the second.** A change to
`libs/shared-types` touches no service folder — so a strict per-folder rule builds **nothing**, and the
breakage is discovered by whichever service deploys next.

**That is why line 470's comment exists**: *"If shared libs change, rebuild ALL dependent services."*

**Why B is the state at line 40** — 50 developers triggering every build — and why E finds the problem
hours after the merge.

</details>

---

## Q47

Nine months in, a `libs/shared-types` change merges cleanly and breaks `catalog-service` in production.
The path filter ran, the shared-library matrix job ran and passed, and `catalog-service` was in the
matrix list. Two new services, `review-service` and `recommendation-service`, were added to the mono-repo
four months ago.

What is the most likely cause?

- A. The path filter did not match the changed file
- B. `paths-filter` was misconfigured for the library path
- C. Dependents' unit tests passed but their contract changed
- D. Sparse-checkout hid the changed file from the build

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-12.md`:** lines **475–490**.

```yaml
    strategy:
      matrix:
        service:
          - order-service
          - payment-service
          - user-service
          - catalog-service
          - shipping-service
```

**Two defects, and the question separates them deliberately.**

**The hard-coded list is the obvious one, and it is not the cause here** — `catalog-service` is on it. But
**`review-service` and `recommendation-service` are not**, and they were added four months ago. That is a
live, undetected gap: a `libs/` change today rebuilds five of fifteen services, and nobody has noticed
because nothing reports the omission.

**The cause of *this* failure is what the matrix job runs**: `npm ci && npm run build && npm test` (lines
487–490). A type change that compiles and passes `catalog-service`'s own unit tests can still break its
**runtime contract** with another service — which unit tests by definition do not exercise.

**The fix has two parts.** Generate the matrix from the repository (a discovery step listing `services/*`)
so new services cannot be forgotten, and add **integration tests** to the shared-library fan-out so the
job proves the services still work **together**, not merely that each compiles.

**The durable lesson: a hand-maintained list of dependents is a control that silently decays**, and in a
mono-repo the whole point is that the repository already knows what the services are.

</details>

---

## Q48

A year on, a developer clones in under two minutes, `git status` is instant, a push builds two services
rather than fifteen, and a shared-type rename still lands in one commit.

Which explanation best accounts for the change?

- A. The repository was split into fifteen separate repositories
- B. Size stopped being a reason to split the mono-repo
- C. History was rewritten to shrink the repository
- D. Every developer was issued a faster machine

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-12.md`:** lines **19**, **30–45**, **189–192**, **101–108**, **411–482**.

**Take the four numbers at line 19 in turn.**

*8 GB* was never really the problem — **downloading 8 GB was.** Partial clone defers blob transfer, so the
repository can stay large while the clone stops being.

*50,000 commits* made local operations slow because Git recomputes things it could cache. **Scalar's
commit-graph and multi-pack index are caches**, and FSMonitor stops `git status` walking the tree.

*25 minutes* was the compound of both, and it is now two.

*15 services* was the argument for splitting — and **path filters deliver the CI benefit of separate
repositories without giving up the atomic commit** that made the mono-repo worth keeping.

**What actually changed is that the mono-repo's costs stopped being structural and became configuration.**
The teams arguing for a split were not wrong about the symptoms; they were treating "8 GB is slow" as a
property of mono-repos rather than of **default Git settings**.

**The graded idea: decide the repository model on the *organisational* question — do these services change
together? — and treat scale as an engineering problem with known tools.** Splitting to fix a clone time
buys a 25-minute saving and pays for it with coordinated PRs and version drift, forever.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **`--depth=1` offered for repository size** | Q1, Q20, Q25, Q28, Q36 | Limits history, still fetches every file. Breaks blame |
| **Scalar or sparse-checkout offered for large binaries** | Q19, Q31 | That is Git LFS. Large history ≠ large files |
| **Sparse-checkout assumed to reduce the download** | Q20, Q28, Q43 | It reduces the working tree only |
| **`set` used where `add` was meant** | Q5, Q36, Q38 | `set` replaces the whole list |
| **Sparse paths left to individual discovery** | Q26, Q45 | The error names a module, not a path |
| **Scalar assumed to change the repo format** | Q2, Q28 | Standard Git settings. `unregister` reverses it |
| **`submodule update` vs `--remote` confused** | Q6, Q12, Q21, Q29 | Pinned commit vs latest on the branch |
| **Pinned commit assumed to be in `.gitmodules`** | Q11, Q29 | It is in the parent's tree. `git add` the path |
| **Cloning submodules without `--recurse-submodules`** | Q13, Q39 | Empty directories, missing-file build errors |
| **`checkout: self` omitted after adding others** | Q22, Q30, Q40 | Implicit checkout stops the moment you add one |
| **`GITHUB_TOKEN` for a private second repository** | Q15, Q30 | Repository-scoped. Needs a PAT or App token |
| **Path filter with no shared-library fan-out** | Q23, Q46, Q47 | A `libs/` change builds nothing and breaks consumers |
| **Hand-maintained matrix of dependent services** | Q47 | Decays silently as services are added |
| **Splitting a repo to fix a clone time** | Q25, Q48 | Pays with coordinated PRs and version drift, forever |

---

# What to memorise

**In `challenge-12.md`:** lines **29–72**, **99–135**, **156–193**, **395–491**.

```text
THE TRADE - name the cost of whichever you pick
  MONO-REPO   + ATOMIC cross-service change (one commit, 15 services)
              + one source of truth for shared libs, no version drift
              + unified CI config, consistent tooling
              - 8 GB / 25 min clone | everyone triggers every build
              - LIMITED PER-SERVICE PERMISSIONS  <- the cost with NO mitigation
              - conflicts on shared files | one branching strategy for all
  MULTI-REPO  + clear ownership, independent releases, small clones
              + fine-grained access, isolated failures
              - coordinated PRs for cross-service change
              - DIAMOND DEPENDENCIES + version drift on shared libs
              - painful refactoring across boundaries, harder discovery

FOUR TOOLS, FOUR PROBLEMS - do not mix them up
  --filter=blob:none   reduces the DOWNLOAD. keeps full history. blobs fetched lazily
  sparse-checkout      reduces the WORKING TREE. downloads unchanged
  --depth=1            reduces HISTORY. still downloads every file. BREAKS blame/bisect
  Git LFS (Ch.10)      large FILES. nothing to do with history depth
```

```bash
# The developer's clone                             (lines 189-192)
git clone --filter=blob:none --sparse <url>
git sparse-checkout init --cone     # directory-level patterns = faster than pattern mode
git sparse-checkout set services/order-service libs/shared-types libs/common-utils
git sparse-checkout add services/payment-service    # ADD extends; SET replaces
git sparse-checkout list | disable

# Scalar - a BUNDLE of standard Git settings        (lines 99-135)
scalar register        # configure an existing clone
scalar clone <url>     # clone + configure in one
scalar run             # prefetch, commit-graph, loose-objects, incremental-repack
scalar diagnose | scalar list | scalar cache-server --set <url> | scalar unregister
#   enables: partial clone | FSMonitor | commit-graph | multi-pack index | background maintenance
#   core.fsmonitor=true  core.multipackindex=true  fetch.writecommitgraph=true
#   maintenance.auto=false  maintenance.strategy=incremental
#   still a NORMAL git repository - any client can use it
```

```bash
# Submodules                                        (lines 231-274)
git submodule add <url> libs/shared          # writes .gitmodules: path, url, branch
cd libs/shared && git checkout v2.3.0 && cd ..
git add libs/shared                          # commits the COMMIT POINTER (in the tree, not .gitmodules)
git clone --recurse-submodules <url>         # or the directory clones EMPTY
git submodule update --init --recursive      # move to the PARENT'S PINNED commit
git submodule update --remote libs/shared    # move to LATEST on the tracked branch, then git add + commit
git submodule foreach 'git checkout main && git pull'
git submodule deinit libs/shared && git rm libs/shared && rm -rf .git/modules/libs/shared
```

```yaml
# Azure Pipelines multi-repo                        (lines 287-318)
resources:
  repositories:
    - {repository: shared-libs, type: git,    name: Project/shared-libs, ref: refs/tags/v2.3.0}
    - {repository: order,       type: github, name: contoso/order-service, endpoint: github-conn}
steps:
  - checkout: self          # REQUIRED once you add ANY checkout - implicit checkout stops
    path: s/payment-service
  - checkout: shared-libs
    path: s/shared-libs

# GitHub Actions multi-repo                         (lines 363-379)
- uses: actions/checkout@v4
  with: {path: order-service}
- uses: actions/checkout@v4
  with: {repository: contoso/shared-libs, ref: v2.3.0, path: shared-libs}   # PUBLIC: no token
- uses: actions/checkout@v4
  with: {repository: contoso/infrastructure, token: "${{ secrets.CROSS_REPO_TOKEN }}", path: infra}
#   PRIVATE needs a PAT or App token - GITHUB_TOKEN cannot leave its repo (Ch.40)
```

```yaml
# Path-based CI                                     (lines 408-491)
- uses: dorny/paths-filter@v3
  id: changes
  with:
    filters: |
      order-service:
        - 'services/order-service/**'
        - 'libs/shared-types/**'      # include the libraries it DEPENDS ON
build-order-service:
  if: needs['detect-changes'].outputs['order-service'] == 'true'    # a STRING 'true'

# AND the honest half - when libs change, rebuild every dependent
build-all-on-shared-change:
  if: needs['detect-changes'].outputs['shared-libs'] == 'true'
  strategy: {matrix: {service: [order-service, payment-service, ...]}}
#   a path filter with NO shared-lib fan-out silently ships breakage
#   GENERATE the service list - a hand-maintained matrix decays as services are added

# Azure Pipelines equivalent gates the WHOLE PIPELINE, not individual jobs
trigger: {paths: {include: ['services/order-service/**', 'libs/shared-types/**']}}
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Domain 2 complete. Move to Challenge 13 |
| 38–43 | Re-read the trap index and the four-tools table, then move on |
| 30–37 | Write the four tools and the mono/multi trade from memory, then retake |
| Below 30 | Redo Tasks 3, 4 and 8 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 12.

:::danger The two things

**Four tools, four problems.** `--filter=blob:none` cuts the **download**. Sparse-checkout cuts the
**working tree**. `--depth=1` cuts **history** and breaks blame. **Git LFS** is for large **files** and
belongs to a different challenge.

**Decide the model on whether the services change together**, not on clone time. Splitting to fix a slow
clone buys 25 minutes and pays with coordinated PRs and version drift, permanently.

:::
