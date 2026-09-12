---
sidebar_position: 2.5
toc_max_heading_level: 2
title: "Challenge 14: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 14 — AZ-400 exam questions

**48 questions** built only from what Challenge 14 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-14.md`**.

:::danger Read this before you start

**SemVer communicates COMPATIBILITY. CalVer communicates AGE.** That single sentence decides most of
this paper.

**MAJOR** = an incompatible API change. **MINOR** = backward-compatible new functionality. **PATCH** =
a backward-compatible fix. A consumer reading `2.1.0 → 2.2.0` knows the upgrade is safe **without
reading the changelog** — that is the entire value, and it is why libraries use SemVer.

**CalVer** answers "when was this released?" and says nothing about compatibility. Right for
**applications** on a time-based cadence, wrong for a **library** other teams depend on.

**Two mechanical rules the exam checks every time.**

**A higher bump resets everything below it.** A minor bump zeroes patch, so `2.1.0` plus a feature **and**
a fix is `2.2.0`, not `2.2.1`.

**A release always outranks its own pre-releases.** `1.0.0-alpha < 1.0.0-beta < 1.0.0-rc < 1.0.0`. And
**build metadata after `+` does not affect precedence at all.**

**And the race condition is the operational heart of the challenge.** Two pipelines that read the same
latest tag compute the same next version, and the second publish fails — because **a published version is
immutable**.

The scenario at line 21: dates, random build numbers, and three teams not versioning at all.

:::

---

# Section A — Multiple choice

---

## Q1

Which version has the highest precedence?

- A. `1.0.0-alpha`
- B. `1.0.0-beta.2`
- C. `1.0.0-rc.1`
- D. `1.0.0`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-14.md`:** line **49**.

```text
Precedence order: 1.0.0-alpha.1 < 1.0.0-beta.1 < 1.0.0-rc.1 < 1.0.0
```

**A release always outranks every pre-release of the same `MAJOR.MINOR.PATCH`.** The pre-release
identifier means "not this yet", so removing it is the final step up.

**Which has a practical consequence worth knowing.** A consumer with `^1.0.0` will **not** install
`1.0.0-rc.1` by default — pre-releases are opted into, not defaulted into, precisely because they sort
below the release.

**And the alphabetical accident helps you remember it**: alpha, beta, rc — three identifiers that happen
to sort correctly as strings.

</details>

---

## Q2

A team releases an internal API gateway monthly and wants the version to say **when** rather than **what
changed**. Which strategy?

- A. CalVer with a `YYYY.MM` scheme
- B. SemVer with pre-release tags
- C. Auto-incrementing build numbers
- D. The commit SHA as the version

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-14.md`:** lines **118–125**.

```text
CalVer works well for:
- Applications (not libraries) where API compatibility is not the primary concern
- Products with time-based release trains (Ubuntu uses YY.MM: 24.04)
- Internal services where "when was this deployed" matters more than "what changed"
```

**All three bullets match the question.** An internal gateway on a monthly train, where the operational
question is "which month's build is running?"

**Why B is right for a library and wrong here.** SemVer's value is telling a consumer whether an upgrade
breaks them — and a deployed service has no consumers reading its version to decide that.

**Why C and D fail the "when" requirement.** Build 4,271 and `a1b2c3d` both identify a build uniquely and
neither tells you anything without a lookup.

</details>

---

## Q3

Which Azure Pipelines expression provides an **atomic** auto-incrementing integer that prevents
collisions between parallel runs?

- A. `$(Build.BuildId)`, unique per organisation
- B. `$(Rev:r)`, the daily revision suffix
- C. `$[counter(variables['prefix'], 0)]`
- D. `$(System.JobAttempt)`, the retry count

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-14.md`:** lines **269** and **471**.

```yaml
  patchVersion: $[counter(format('{0}.{1}', variables['majorVersion'], variables['minorVersion']), 0)]
```

**Atomic is the operative word.** The counter is evaluated server-side and increments once per run, so two
simultaneous pipelines get different values — which is the whole fix for the Break scenario.

**Why A is the near-miss that fails the *sequential* requirement** (line 471). `Build.BuildId` is unique
across the **organisation**, so it is a large arbitrary number that jumps unpredictably between your
package's versions.

**Note the `$[ ]` syntax** — runtime evaluation. `$( )` would be a macro and `${{ }}` compile-time, and
the counter must be evaluated when the run starts (Challenge 22's timing rules).

</details>

---

## Q4

A library at `2.1.0` receives a backward-compatible new method **and** a bug fix. What is the next
version?

- A. `2.1.1`
- B. `2.2.0`
- C. `3.0.0`
- D. `2.1.0-patch.1`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-14.md`:** lines **35–37** and **482**.

```text
When changes include a new backward-compatible feature (new method), the MINOR version increments.
The bug fix would normally be a PATCH increment, but since MINOR is being bumped, the patch resets to 0.
```

**The highest-order change wins, and everything below it resets.** A feature plus a fix is a MINOR bump,
and the patch goes to zero — not `2.2.1`.

**Why A is the arithmetic mistake** of applying only the smaller change, and why C would be correct only
if the new method **broke** an existing one.

**The same rule with a major change**: `2.4.7` plus a breaking change is `3.0.0`, not `3.4.7` (Challenge
09 Q40).

</details>

---

## Q5

Two pipelines merge to `main` seconds apart, both compute `1.3.0`, and the second publish fails with 403.
What is the root cause?

- A. The registry was temporarily unavailable at publish
- B. The publishing token had expired before the push
- C. The package name was misspelled in `package.json`
- D. Both runs read the same state before either tagged

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-14.md`:** line **378**.

```text
Race conditions in tag-based versioning occur when parallel pipelines read the same "latest" tag before
either has pushed a new one.
```

**The underlying rule is that a published version is immutable** — the error at line 372 says it
directly: *"You cannot publish over the previously published versions."*

**Which is a feature, not an obstacle.** If a version could be overwritten, a consumer's locked
`1.3.0` could silently become different code.

**And note this is a read-then-write race**, so it gets more likely as the team gets busier — exactly when
you least want to debug it.

</details>

---

## Q6

Which fix eliminates the race **entirely** rather than working around it?

- A. Retry the publish with an incremented patch number
- B. Publishing more slowly to avoid overlap
- C. GitVersion `ContinuousDeployment` mode per commit
- D. Manual version bumps by the release manager

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-14.md`:** lines **423–433**.

```text
v1.3.0 tag on main
  -> commit A: 1.3.1-ci.1
  -> commit B: 1.3.1-ci.2  (always unique)
```

```text
This avoids the race entirely because each commit produces a distinct version.
```

**The version is derived from the commit, not from a shared counter** — so two commits cannot produce the
same version no matter how close together they land.

**Why A is listed as Fix 3 and marked below Fix 4** (line 423 says "recommended"). Retry works and it
publishes a version nobody predicted, so the tag, the changelog and the artifact can disagree.

**Why B is not a fix.** It reduces the probability of a race without removing it, which is the definition
of a latent bug.

</details>

---

## Q7

What does build metadata after `+` affect?

- A. It bumps the major version component
- B. Nothing — it does not affect precedence
- C. It changes the pre-release ordering
- D. It sets the package's visibility level

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-14.md`:** lines **53–57**.

```text
Build metadata is appended with a plus sign and does not affect version precedence:

1.0.0+20240615
1.0.0-beta.1+build.42
```

**`1.0.0+build.1` and `1.0.0+build.2` are the *same version* to any SemVer-aware tool.** The metadata
travels with the artifact for traceability and is ignored when resolving.

**Which is exactly why it is useful for artifacts** (line 359): `1.2.0+build.42.sha.a1b2c3d` tells you
which run and which commit produced this binary, **without** creating a version consumers must reason
about.

**And note the second example**: metadata can follow a pre-release identifier, so both suffixes coexist.

</details>

---

## Q8

Which npm command produces `2.1.0-beta.0` from `2.0.0`?

- A. `npm version prerelease --preid=beta`
- B. `npm version minor --preid=beta`
- C. `npm version prepatch --preid=beta`
- D. `npm version preminor --preid=beta`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-14.md`:** lines **82–86**.

```bash
# Pre-release: 2.0.0 -> 2.1.0-beta.0
npm version preminor --preid=beta

# Increment pre-release: 2.1.0-beta.0 -> 2.1.0-beta.1
npm version prerelease
```

**`preminor` bumps the minor **and** starts a pre-release**; `prerelease` only advances an existing one.

**Which is the two-step pattern the challenge demonstrates.** Start the pre-release series once with
`preminor`, then use `prerelease` for every subsequent candidate.

**And `--preid` names the identifier** — without it you get `2.1.0-0` rather than `2.1.0-beta.0`.

</details>

---

## Q9

How do you override a NuGet package version at build time?

- A. Editing the `.csproj` before every build
- B. `dotnet nuget push --version` at publish time
- C. `dotnet pack /p:Version=1.2.0-beta.1`
- D. An environment variable named `VERSION`

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-14.md`:** line **109**.

```bash
dotnet pack --configuration Release /p:Version=1.2.0-beta.1
```

**`/p:` sets an MSBuild property**, overriding the `<Version>` in the project file (line 98) without
touching it.

**Which is what makes CI versioning possible.** The `.csproj` holds a sensible default for a local build;
the pipeline supplies the computed version at pack time — the same value GitVersion produces at line 255.

**Why A is what teams do first and why it fails.** Editing the file in CI means committing it, which
means a commit per build, which triggers the build again.

</details>

---

## Q10

What does `versioningScheme: byEnvVar` do in the `DotNetCoreCLI@2` pack task?

- A. It uses the pipeline's build number as the version
- B. It reads the version from the named variable
- C. It uses the current date as the version
- D. It increments the version automatically each run

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-14.md`:** lines **281–282**.

```yaml
      versioningScheme: byEnvVar
      versionEnvVar: packageVersion
```

**Two settings that work as a pair**: the scheme says *where to look*, and `versionEnvVar` says *which
variable*.

**And `packageVersion` is composed at line 270** from the major, minor and the atomic counter — so the
whole chain is: counter → variable → pack task → package version.

**The alternative schemes exist and are worth knowing by shape** — `byPrereleaseNumber`,
`byBuildNumber`, `off` — but `byEnvVar` is the one that lets you compute the version yourself.

</details>

---

## Q11

In the branch-based pre-release pipeline, what suffix does a `feature/` branch produce?

- A. `-alpha.$(Build.BuildId)`
- B. `-rc.$(Build.BuildId)`
- C. `-dev.$(Build.BuildId)`
- D. Empty

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-14.md`:** lines **295–302**.

```yaml
  ${{ if eq(variables['Build.SourceBranch'], 'refs/heads/main') }}:
    versionSuffix: ''
  ${{ elseif startsWith(variables['Build.SourceBranch'], 'refs/heads/feature/') }}:
    versionSuffix: '-alpha.$(Build.BuildId)'
  ${{ elseif startsWith(variables['Build.SourceBranch'], 'refs/heads/release/') }}:
    versionSuffix: '-rc.$(Build.BuildId)'
```

**Four branches, four maturities: main is a release, feature is alpha, release is rc, everything else is
dev.** That mapping is the branch model expressed as version identifiers.

**And `main` gets an *empty* suffix** — the only branch that produces a clean release version, which is
what makes `main` the source of shipped packages.

**Note the `${{ }}` syntax**: these are **compile-time** conditionals, evaluated when the YAML is
expanded (Challenge 22).

</details>

---

## Q12

What does `##vso[build.updatebuildnumber]` do?

- A. It sets the package version for the pack task
- B. It increments the pipeline's build counter
- C. It renames the pipeline run to the version
- D. It creates a git tag carrying the version

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-14.md`:** line **307**.

```bash
      echo "##vso[build.updatebuildnumber]$VERSION"
      echo "##vso[task.setvariable variable=packageVersion]$VERSION"
```

**Two logging commands, two different jobs, on consecutive lines.** The first changes what the **run is
called** in the UI; the second sets a **variable** later steps consume.

**And the first is worth more than it looks operationally.** A run list showing `1.2.0-rc.44` instead of
`20240615.3` means you can find the build that produced a given version by looking, not by querying.

</details>

---

## Q13

What does GitVersion's `mode: ContinuousDeployment` provide?

- A. Deployment automation to each environment
- B. Automatic tagging of every commit on `main`
- C. Continuous integration triggers on push
- D. A unique version per commit via commit count

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-14.md`:** lines **203** and **425–431**.

**"Continuous deployment" here describes the *versioning* mode, not a deployment feature** — which is
exactly why it is a good exam distractor.

**Every commit gets a distinct version** (`1.3.1-ci.1`, `1.3.1-ci.2`), which removes the race (Q6) and
means any commit can be published and deployed without a human choosing a number.

**And the configuration at lines 204–221 maps branches to increments**: `main` patch, `feature` minor,
`release` none, `hotfix` patch — with `tag:` supplying the pre-release identifier.

</details>

---

## Q14

Why does the GitVersion workflow use `fetch-depth: 0`?

- A. To fetch submodules alongside the checkout
- B. GitVersion reads full history and tags to compute
- C. To speed up the checkout on a large repository
- D. To include every branch for merge detection

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-14.md`:** lines **235–237**.

```yaml
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
```

**GitVersion's entire method is analysing branch history and tags** — how far you are from the last tag,
which branch you are on, what merged in. A shallow clone gives it one commit and no tags.

**And it fails in the most confusing way possible**: it computes *a* version, usually `0.1.0`, and the
build succeeds. **Green, and wrong** — the same trap as Challenges 03, 05, 07 and 09.

</details>

---

## Q15

Which GitVersion output should feed `dotnet pack`?

- A. `nuGetVersion`
- B. `semVer`
- C. `informationalVersion`
- D. `assemblySemVer`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-14.md`:** lines **250–255**.

```yaml
          echo "SemVer: ${{ steps.gitversion.outputs.semVer }}"
          echo "NuGetVersion: ${{ steps.gitversion.outputs.nuGetVersion }}"
```

```yaml
        run: dotnet pack /p:Version=${{ steps.gitversion.outputs.nuGetVersion }}
```

**GitVersion emits several formats because different ecosystems accept different characters.**
`nuGetVersion` is normalised for NuGet's rules; `semVer` is strict SemVer; `informationalVersion`
includes the commit metadata for embedding in the assembly.

**Using the wrong one usually still works and produces a subtly different string** in the package
metadata — which is how two artifacts of the same build end up labelled differently.

</details>

---

## Q16

Which artifact versioning strategy gives the best **traceability** to source?

- A. The `latest` tag, updated on every push
- B. The build date alone as the image tag
- C. A random GUID generated for each build
- D. `$(Build.BuildId)-<sha>` as the image tag

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-14.md`:** lines **340–343**.

```bash
      SHORT_SHA=$(echo $(Build.SourceVersion) | cut -c1-7)
      IMAGE_TAG="$(Build.BuildId)-${SHORT_SHA}"
```

**Two identifiers, two questions answered.** The build ID finds the pipeline run — logs, artifacts,
approvals. The SHA finds the source — the commit, the PR, the work item (Challenge 03's chain).

**Why A is the tag that makes rollback impossible.** `latest` is a moving pointer; "redeploy what we had
yesterday" has no answer if that is all you tagged.

**And note strategy 1 at line 331 pushes *both***: the SemVer tag **and** `latest`. Immutable for
rollback, mutable for convenience.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** describe SemVer components? (Choose three.)

- A. MAJOR increments for incompatible API changes
- B. MAJOR increments on a fixed monthly schedule
- C. MINOR increments for backward-compatible features
- D. PATCH increments once for every build run
- E. PATCH increments for backward-compatible bug fixes
- F. MINOR encodes the month of the release date

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-14.md`:** lines **35–37**.

**Every component describes **compatibility**, not time and not effort.** A one-line change that breaks an
API is a MAJOR; a two-month feature that breaks nothing is a MINOR.

**Why B, D and F are CalVer and build-number thinking** grafted onto SemVer — and they are how the
scenario's teams ended up with `20240115` and random build numbers (line 21).

</details>

---

## Q18

Which **three** are true of pre-release versions? (Choose three.)

- A. They sort above the corresponding release
- B. They are appended after a hyphen separator
- C. They are ignored by package managers
- D. `alpha < beta < rc` in precedence order
- E. They cannot be published to a registry
- F. A release outranks any pre-release of it

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-14.md`:** lines **41–49**.

**A is the exact inversion of F**, and it is the option that catches people who reason "release candidate
sounds later than release".

**Why C overstates a real behaviour.** Package managers do not **ignore** pre-releases — they do not
**select** them by default for a range like `^1.0.0`. Ask for `1.0.0-rc.1` explicitly and you get it.

</details>

---

## Q19

Which **three** situations suit CalVer? (Choose three.)

- A. Applications where API compatibility is secondary
- B. A shared library consumed by fifteen services
- C. Products released on a time-based release train
- D. An SDK with a public, documented API surface
- E. Internal services where deploy timing matters most
- F. A package with automated dependency updates

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-14.md`:** lines **122–125**.

**B, D and F are all the same case: something with consumers who must reason about upgrade risk.**

**F is the sharpest of the three.** Automated dependency updates — Dependabot's `semver-major`,
`semver-minor`, `semver-patch` classification (Challenge 44) — **require** SemVer. A CalVer package
gives the bot no way to distinguish a safe patch from a breaking change, so every update looks the same.

**Which is the scenario's third complaint** at line 25: *"Inability to set up automated dependency update
policies."*

</details>

---

## Q20

Which **two** produce a unique version per pipeline run without a race? (Choose two.)

- A. Reading the latest git tag and adding one
- B. Azure Pipelines `counter()` expression
- C. A hard-coded version in `package.json`
- D. GitVersion in `ContinuousDeployment` mode
- E. The current calendar date as the version

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-14.md`:** lines **382–389** and **423–433**.

**Two different mechanisms, same guarantee.** The counter is **atomic server-side state**; GitVersion is
**derived from the commit**, which is unique by construction.

**Why A is the Break scenario itself** (Q5), and **why E fails on the same day** — two runs on 15 June
both compute `2024.06.15`, which is why line 132 appends the run number.

</details>

---

## Q21

Which **two** are true of build metadata? (Choose two.)

- A. It replaces the pre-release identifier
- B. It increments the patch component
- C. It follows a `+` sign in the version
- D. It is required by the SemVer specification
- E. It does not affect version precedence

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-14.md`:** lines **53–57**.

**Why A is refuted by the second example on line 57**: `1.0.0-beta.1+build.42` carries both, and they are
independent — the pre-release sets precedence, the metadata carries traceability.

**And the practical use is at line 359**: `1.2.0+build.42.sha.a1b2c3d` identifies exactly which run and
which commit produced an artifact, while remaining **version 1.2.0** to every resolver.

</details>

---

## Q22

Which **two** does the Azure Pipelines version block compose? (Choose two.)

- A. `patchVersion` from an atomic `counter()`
- B. The container image tag for the registry
- C. The pipeline's own run build number
- D. `packageVersion` from major, minor and patch
- E. The published artifact's storage path

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-14.md`:** lines **267–270**.

```yaml
  majorVersion: 1
  minorVersion: 3
  patchVersion: $[counter(format('{0}.{1}', variables['majorVersion'], variables['minorVersion']), 0)]
  packageVersion: $(majorVersion).$(minorVersion).$(patchVersion)
```

**The counter's *seed* is the major and minor** — `format('{0}.{1}', ...)` — which is the detail worth
noticing. **Bump the minor and the counter resets**, because it is now keyed on a different prefix.

**That is SemVer's reset rule** (Q4) implemented by the counter's scoping, rather than by arithmetic.

</details>

---

## Q23

Which **two** artifact tagging practices does the challenge recommend together? (Choose two.)

- A. Only a `latest` tag on every push
- B. An immutable SemVer tag on the image
- C. A random GUID generated for every build
- D. A mutable `latest` tag alongside it
- E. Overwriting the SemVer tag on rebuild

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-14.md`:** lines **330–333**.

```bash
docker push contoso.azurecr.io/auth-service:1.2.0
docker push contoso.azurecr.io/auth-service:latest
```

**Two tags for two purposes.** The SemVer tag is what a deployment records and what a rollback names; the
`latest` tag is a convenience for local pulls.

**And E is the practice that breaks rollback silently.** Retagging `1.2.0` onto a different image means
"redeploy 1.2.0" now deploys something else — with nothing in the deployment record showing that
anything changed.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must unify versioning across 15 teams so that a deployed version can be identified,
dependency conflicts are avoidable, automated dependency updates are possible, and artifact lineage is
auditable.

---

## Q24

**Proposed solution:** Use SemVer for the shared libraries so consumers can reason about upgrade risk, and
CalVer for the deployed applications where release timing matters. Derive library versions with GitVersion
in `ContinuousDeployment` mode over full history, mapping branches to pre-release identifiers. In Azure
Pipelines, compose the version from major, minor and an atomic `counter()` seeded on the major and minor.
Tag container images with the SemVer version **and** a build-ID-plus-short-SHA tag, pushing `latest`
alongside.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-14.md`:** lines **34–37**, **122–125**, **203–221**, **267–270**, **330–343**.

| Complaint at line 21 | Mechanism |
|---|---|
| Cannot identify the deployed version | Immutable SemVer/CalVer tags on artifacts |
| Dependency conflicts on shared majors | SemVer MAJOR signals incompatibility |
| No automated dependency updates | SemVer classification (Q19) |
| Untraceable artifact lineage | Build ID + short SHA on every image |

**Using both schemes is the mature answer, not a compromise.** Libraries have consumers who must reason
about compatibility; applications have operators who need to know when something shipped.

</details>

---

## Q25

**Proposed solution:** Standardise every team on CalVer in `YYYY.MM.DD` format, including the four shared
libraries, since it is simple and needs no judgement. Tag container images `latest` only, since that is
always the newest. Let each pipeline read the newest git tag and add one to compute the next version.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures.**

**CalVer on a library removes the only signal consumers have.** A service upgrading from `2024.06.15` to
`2024.07.01` cannot tell whether the API changed — which is the dependency-conflict complaint at line 24,
unfixed.

**And it makes automated dependency updates impossible** (Q19). Dependabot classifies updates as major,
minor or patch; CalVer offers no such classification, so every update is unclassifiable.

**`latest` only** means no deployed version can be identified and no rollback has a target — the first
complaint at line 23, restated.

**And reading the newest tag and adding one is the Break scenario** (Q5): two merges seconds apart compute
the same version and the second publish fails.

</details>

---

## Q26

**Proposed solution:** SemVer for the libraries, CalVer for the applications. GitVersion in
`ContinuousDeployment` mode over full history. Atomic `counter()` in Azure Pipelines. SemVer plus
build-ID-and-SHA tags on images. When a published library version turns out to contain a defect, republish
the corrected code under the **same** version number so consumers pick up the fix without changing their
dependency ranges.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**A published version is immutable, and the registry enforces it.** The error at line 372 is exactly this
attempt: *"You cannot publish over the previously published versions: 1.3.0."*

**And the reasoning — "consumers pick up the fix without changing their ranges" — describes the danger
rather than a benefit.** If it worked, a consumer's locked `1.3.0` would silently become different code:
their lockfile hash would no longer match, their reproducible build would stop reproducing, and a version
they tested would differ from the one they deploy.

**That is the property versioning exists to provide.** `1.3.0` must mean one specific artifact forever,
which is what lets a rollback target it and an audit trust it — the first and fourth complaints at lines
23 and 26.

**The correct response is a new version**: `1.3.1` for a fix, and if `1.3.0` is dangerous, **deprecate**
it so consumers are warned while the artifact remains intact for anyone auditing what shipped.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — SemVer rules

| # | Statement | Answer |
|---|---|---|
| 1 | MAJOR signals an incompatible API change |  |
| 2 | A MINOR bump resets PATCH to zero |  |
| 3 | `1.0.0-rc.1` outranks `1.0.0` |  |
| 4 | Build metadata after `+` affects precedence |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | MAJOR signals an incompatible API change | **Yes** |
| 2 | A MINOR bump resets PATCH to zero | **Yes** |
| 3 | `1.0.0-rc.1` outranks `1.0.0` | **No** |
| 4 | Build metadata after `+` affects precedence | **No** |

**In `challenge-14.md`:** lines **35**, **482**, **49**, **53**.

Row 2 is the arithmetic the exam checks most (Q4); row 3 is the inversion it offers as a distractor.

</details>

---

## Q28 — CalVer

| # | Statement | Answer |
|---|---|---|
| 1 | CalVer communicates when, not what changed |  |
| 2 | CalVer suits applications more than libraries |  |
| 3 | CalVer supports automated dependency update policies |  |
| 4 | `YYYY.MM.DD` alone is unique per day only |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | CalVer communicates when, not what changed | **Yes** |
| 2 | CalVer suits applications more than libraries | **Yes** |
| 3 | CalVer supports automated dependency update policies | **No** |
| 4 | `YYYY.MM.DD` alone is unique per day only | **Yes** |

**In `challenge-14.md`:** lines **114**, **123**, **25**, **130–132**.

Row 3 is the requirement CalVer cannot satisfy (Q19), and row 4 is why line 132 appends the run number.

</details>

---

## Q29 — pipeline versioning

| # | Statement | Answer |
|---|---|---|
| 1 | `counter()` is atomic across concurrent runs |  |
| 2 | `Build.BuildId` is sequential per package |  |
| 3 | GitVersion needs `fetch-depth: 0` |  |
| 4 | `versioningScheme: byEnvVar` reads a named variable |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `counter()` is atomic across concurrent runs | **Yes** |
| 2 | `Build.BuildId` is sequential per package | **No** |
| 3 | GitVersion needs `fetch-depth: 0` | **Yes** |
| 4 | `versioningScheme: byEnvVar` reads a named variable | **Yes** |

**In `challenge-14.md`:** lines **382**, **471**, **237**, **281–282**.

Row 2 is the distinction that makes `counter()` the answer to Q3: unique but not sequential is not the
same as sequential.

Row 3's failure is silent — GitVersion computes `0.1.0` and the build goes green (Q14).

</details>

---

## Q30 — artifacts and immutability

| # | Statement | Answer |
|---|---|---|
| 1 | A published package version cannot be overwritten |  |
| 2 | `latest` is sufficient for rollback |  |
| 3 | Build ID plus short SHA links an artifact to its source |  |
| 4 | Build metadata is useful on artifacts for traceability |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A published package version cannot be overwritten | **Yes** |
| 2 | `latest` is sufficient for rollback | **No** |
| 3 | Build ID plus short SHA links an artifact to its source | **Yes** |
| 4 | Build metadata is useful on artifacts for traceability | **Yes** |

**In `challenge-14.md`:** lines **372**, **331**, **340–342**, **359**.

Row 1 is the rule Q26 breaks; row 2 is the practice that makes the scenario's first complaint permanent.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each artifact to the versioning scheme that fits.

| Artifact | Scheme |
|---|---|
| A shared library used by 15 services |  |
| An internal service deployed monthly |  |
| A release candidate for testing |  |
| A container image needing source traceability |  |
| A binary you must trace to a commit and a run |  |
| A distro-style time-boxed product |  |

**Options:** Build ID + short SHA · CalVer · CalVer — Ubuntu's `24.04` · SemVer · SemVer + build metadata · SemVer pre-release — `2.2.0-rc.1`

<details>
<summary>Show answer</summary>

| Artifact | Scheme |
|---|---|
| A shared library used by 15 services | **SemVer** |
| An internal service deployed monthly | **CalVer** |
| A release candidate for testing | **SemVer pre-release — `2.2.0-rc.1`** |
| A container image needing source traceability | **Build ID + short SHA** |
| A binary you must trace to a commit and a run | **SemVer + build metadata** |
| A distro-style time-boxed product | **CalVer — Ubuntu's `24.04`** |

**In `challenge-14.md`:** lines **34–37**, **122–125**, **44–46**, **340–342**, **359**, **124**.

**Rows 1 and 2 are the whole decision.** Does anything **depend** on this and need to reason about
upgrade risk? SemVer. Is it a thing you deploy on a schedule? CalVer.

</details>

---

## Q32

Match each change to the version it produces from `2.1.0`.

| Change | Next version |
|---|---|
| A backward-compatible bug fix |  |
| A backward-compatible new method |  |
| A new method **and** a bug fix |  |
| A breaking API change |  |
| A release candidate for the next minor |  |
| A rebuild of the same code |  |

**Options:** 2.1.0+build.43 · 2.1.1 · 2.2.0 · 2.2.0-rc.1 · 3.0.0

<details>
<summary>Show answer</summary>

| Change | Next version |
|---|---|
| A backward-compatible bug fix | **2.1.1** |
| A backward-compatible new method | **2.2.0** |
| A new method **and** a bug fix | **2.2.0** |
| A breaking API change | **3.0.0** |
| A release candidate for the next minor | **2.2.0-rc.1** |
| A rebuild of the same code | **2.1.0+build.43** |

**In `challenge-14.md`:** lines **35–37**, **482**, **44–46**, **56**.

**Row 3 is the one the exam asks** (Q4): the higher-order change wins and the patch resets.

**And row 6 is the metadata case** — the same version, a different build, and no precedence change (Q7).

</details>

---

## Q33

Arrange the steps to make library versions automatic and race-free.

**Items:** Feed the computed version into `dotnet pack` · Check out with full history · Add
`GitVersion.yml` mapping branches to increments and pre-release tags · Install and execute GitVersion in
the workflow

<details>
<summary>Show answer</summary>

### Answer

1. Add `GitVersion.yml` mapping branches to increments and pre-release tags — lines **202–221**
2. Check out with full history — lines **235–237**
3. Install and execute GitVersion in the workflow — lines **239–246**
4. Feed the computed version into `dotnet pack` — line **255**

**Step 2 before step 3 is the requirement that fails silently** (Q14). GitVersion on a shallow clone does
not error — it computes a wrong version and the build succeeds.

**And step 1 first because it is the *policy*.** The configuration says what a `feature/` branch means and
what `main` means; without it GitVersion applies defaults that may not match your branching model
(Challenge 07).

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| `403 - You cannot publish over the previously published versions` |  |
| Every build versions as `0.1.0` |  |
| Dependabot cannot classify an update |  |
| A rollback has no version to target |  |
| A minor bump produced `2.2.1` |  |
| Two artifacts of one build carry different version strings |  |

**Options:** CalVer on a library · Different GitVersion outputs used · Only `latest` was tagged · Patch not reset on the higher-order bump · Shallow clone — GitVersion sees no tags · Two runs computed the same version

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| `403 - You cannot publish over the previously published versions` | **Two runs computed the same version** |
| Every build versions as `0.1.0` | **Shallow clone — GitVersion sees no tags** |
| Dependabot cannot classify an update | **CalVer on a library** |
| A rollback has no version to target | **Only `latest` was tagged** |
| A minor bump produced `2.2.1` | **Patch not reset on the higher-order bump** |
| Two artifacts of one build carry different version strings | **Different GitVersion outputs used** |

**In `challenge-14.md`:** lines **372**, **237**, **25**, **331**, **482**, **250–255**.

**Rows 2 and 6 are the quiet ones.** Both produce a successful build with a version that is merely wrong,
which is far harder to notice than a failed publish.

</details>

---

## Q35

Match each mechanism to what guarantees uniqueness.

| Mechanism | Guarantee |
|---|---|
| `counter()` |  |
| GitVersion `ContinuousDeployment` |  |
| `Build.BuildId` |  |
| `GITHUB_RUN_NUMBER` |  |
| Reading the latest tag and adding one |  |
| `date +%Y.%m.%d` |  |

**Options:** Atomic server-side increment · Derived from the commit — unique by construction · No guarantee — collides within a day · No guarantee — races · Unique, but not sequential per package · Unique per workflow

<details>
<summary>Show answer</summary>

| Mechanism | Guarantee |
|---|---|
| `counter()` | **Atomic server-side increment** |
| GitVersion `ContinuousDeployment` | **Derived from the commit — unique by construction** |
| `Build.BuildId` | **Unique, but not sequential per package** |
| `GITHUB_RUN_NUMBER` | **Unique per workflow** |
| Reading the latest tag and adding one | **No guarantee — races** |
| `date +%Y.%m.%d` | **No guarantee — collides within a day** |

**In `challenge-14.md`:** lines **382–389**, **425–431**, **471**, **393**, **378**, **130–132**.

**The bottom two are the two ways the scenario's teams broke it** — and both look correct until two
pipelines run at once.

</details>

---

# Section F — Hot area

---

## Q36

```bash
# 1.0.0 -> ?
npm version [BLANK 1]      # backward-compatible bug fix
npm version [BLANK 2]      # backward-compatible new feature
npm version [BLANK 3]      # breaking change
```

- **BLANK 1:** `minor` / `major` / `patch` / `prerelease`
- **BLANK 2:** `patch` / `minor` / `preminor` / `major`
- **BLANK 3:** `premajor` / `minor` / `breaking` / `major`

<details>
<summary>Show answer</summary>

### Answer: `patch`, `minor`, `major`

**In `challenge-14.md`:** lines **73–80**.

**Fix → patch, feature → minor, break → major.** Three words, and they map exactly onto the three
components (Q17).

**And the `pre*` variants start a pre-release series** rather than releasing — `preminor` bumps the minor
**and** opens `-beta.0` (Q8).

</details>

---

## Q37

```yaml
variables:
  majorVersion: 1
  minorVersion: 3
  patchVersion: $[[BLANK 1](format('{0}.{1}', variables['majorVersion'], variables['minorVersion']), 0)]
  packageVersion: $(majorVersion).$(minorVersion).$([BLANK 2])
```

- **BLANK 1:** `increment` / `buildId` / `counter` / `rev`
- **BLANK 2:** `counter` / `patchVersion` / `Build.BuildId` / `Rev:r`

<details>
<summary>Show answer</summary>

### Answer: `counter`, `patchVersion`

**In `challenge-14.md`:** lines **269–270**.

**Note the two syntaxes on adjacent lines.** `$[ ]` for the runtime-evaluated counter, `$( )` for the
macro that substitutes an already-computed variable — Challenge 22's timing distinction, applied.

**And the counter's seed matters** (Q22): keyed on major and minor, so bumping the minor resets the patch
automatically.

</details>

---

## Q38

```yaml
  ${{ if eq(variables['Build.SourceBranch'], 'refs/heads/main') }}:
    versionSuffix: '[BLANK 1]'
  ${{ elseif startsWith(variables['Build.SourceBranch'], 'refs/heads/release/') }}:
    versionSuffix: '[BLANK 2]'
```

- **BLANK 1:** `-release` / `-main` / `` (empty) / `-stable`
- **BLANK 2:** `-alpha.$(Build.BuildId)` / `-rc.$(Build.BuildId)` / `-beta` / `` (empty)

<details>
<summary>Show answer</summary>

### Answer: empty, `-rc.$(Build.BuildId)`

**In `challenge-14.md`:** lines **295–300**.

**`main` produces a *clean* version with no suffix** — the only branch that does, which is what makes it
the source of shipped packages (Q11).

**And `-rc` for `release/` matches the precedence order** at line 49: alpha for feature work, rc for a
release branch, nothing for main.

</details>

---

## Q39

```yaml
mode: [BLANK 1]
branches:
  main:
    increment: [BLANK 2]
  feature:
    tag: [BLANK 3]
    increment: Minor
  release:
    tag: rc
    increment: [BLANK 4]
```

- **BLANK 1:** `ContinuousDelivery` / `Mainline` / `ContinuousDeployment` / `Automatic`
- **BLANK 2:** `Minor` / `Patch` / `Major` / `None`
- **BLANK 3:** `beta` / `rc` / `dev` / `alpha`
- **BLANK 4:** `Patch` / `Minor` / `None` / `Major`

<details>
<summary>Show answer</summary>

### Answer: `ContinuousDeployment`, `Patch`, `alpha`, `None`

**In `challenge-14.md`:** lines **203–216**.

**`increment: None` on `release/` is the one worth understanding.** A release branch is stabilising a
version that has already been decided — it produces `1.4.0-rc.1`, `-rc.2`, `-rc.3`, not a climbing minor.

**And `ContinuousDeployment` is the mode that guarantees a distinct version per commit** (Q13), which is
what removes the race.

</details>

---

## Q40

```bash
SHORT_SHA=$(echo $(Build.SourceVersion) | cut -c1-[BLANK 1])
IMAGE_TAG="$(Build.BuildId)-${SHORT_SHA}"
echo "##vso[[BLANK 2]]$IMAGE_TAG"
```

- **BLANK 1:** `8` / `7` / `40` / `12`
- **BLANK 2:** `build.updatebuildnumber` / `task.logissue type=warning` /
  `task.setvariable variable=imageTag` / `artifact.upload`

<details>
<summary>Show answer</summary>

### Answer: `7`, `task.setvariable variable=imageTag`

**In `challenge-14.md`:** lines **341–343**.

**Seven characters is Git's conventional short SHA** — long enough to be unambiguous in practice, short
enough to read in a tag.

**And `setvariable` versus `updatebuildnumber` is the pair from Q12.** This one makes the tag available
to the later Docker task (line 352); the other renames the run.

</details>

---

## Q41

```bash
VERSION="1.2.0[BLANK 1]build.${GITHUB_RUN_NUMBER}.sha.${GITHUB_SHA:0:7}"
# Output: 1.2.0+build.42.sha.a1b2c3d
```

- **BLANK 1:** `-` / `.` / `+` / `_`

<details>
<summary>Show answer</summary>

### Answer: `+`

**In `challenge-14.md`:** line **359**.

**`+` is build metadata and is ignored for precedence** (Q7, Q21) — so this artifact is still **version
1.2.0** to every resolver, while carrying the run and the commit for audit.

**Use `-` instead and you have created a pre-release**, which sorts **below** `1.2.0` and will not be
selected by a consumer asking for `^1.2.0`. **One character, opposite meaning.**

</details>

---

# Section G — Case study

## Case study: Contoso versioning unification

### Background

Contoso has **inconsistent versioning across 15 microservice teams**. The auth team uses dates like
`20240115`, the payments team uses **random build numbers**, and **three teams do not version at all**.
This causes **rollback failures** because nobody can identify what is deployed, **dependency conflicts**
when incompatible libraries share a major version, **no automated dependency update policies**, and
**audit failures from untraceable artifact lineage**.

### Requirements

**Libraries**

- A consumer must be able to tell from the version alone whether an upgrade is safe
- Automated dependency update tooling must be able to classify each update
- A published version must always refer to the same artifact

**Applications**

- The version should communicate when the build was released
- Two runs on the same day must produce different versions

**Pipelines and artifacts**

- Concurrent runs must never compute the same version
- Every container image must be traceable to a commit and a pipeline run
- A rollback must have a specific version to target

---

## Q42

Which scheme should the four shared libraries use, and why?

- A. CalVer — it is simpler and needs no engineering judgement
- B. Build numbers — they are always unique
- C. SemVer — it tells consumers whether an upgrade is safe
- D. The commit SHA — it is exact and unambiguous

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-14.md`:** lines **34–37** and **25**.

**Two of the three library requirements are the same requirement stated twice** — a human reading the
version and a bot reading it both need the compatibility signal, and only SemVer carries one.

**Why A is genuinely simpler and fails both.** "Needs no judgement" is the appeal and also the defect: the
judgement — *is this breaking?* — is the information consumers need.

**Why B and D identify a build uniquely and communicate nothing.** Uniqueness is necessary; it is not
sufficient.

</details>

---

## Q43

Which scheme should the deployed applications use?

- A. CalVer alone, with the date as the whole version
- B. CalVer with a run number appended for same-day runs
- C. SemVer, so that consumers can classify upgrades
- D. `latest`, updated on every successful deploy

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-14.md`:** lines **122–125** and **130–132**.

```bash
CALVER=$(date +%Y.%m.%d)
BUILD_NUMBER=${GITHUB_RUN_NUMBER:-0}
VERSION="${CALVER}.${BUILD_NUMBER}"
```

**Both application requirements are in the answer**: the date communicates *when*, and the run number
satisfies *two runs on the same day must differ*.

**Why A fails the second requirement** — and it fails it on exactly the busy days when it matters.

**And note the `:-0` default** on line 131: outside CI, `GITHUB_RUN_NUMBER` is unset, so the script
produces a valid version locally rather than a malformed one.

</details>

---

## Q44

How is "concurrent runs must never compute the same version" satisfied?

- A. An atomic `counter()` or GitVersion per commit
- B. Reading the newest tag and adding one to it
- C. Publishing packages strictly sequentially
- D. Retrying the publish whenever it returns 403

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-14.md`:** lines **382–389** and **423–433**.

**Both options remove the race rather than surviving it** (Q20) — one with atomic server state, the other
by deriving the version from the commit.

**Why D is the challenge's Fix 3 and marked below Fix 4** (line 423). Retrying publishes a version nobody
predicted, so the git tag, the changelog and the published package disagree about what shipped.

**Why B is the Break scenario** and C is a policy, not a control — it holds until someone runs a pipeline
manually.

</details>

---

## Q45

Which **two** make an image traceable and rollback-able? (Choose two.)

- A. A tag combining the build ID and the short SHA
- B. A `latest` tag only, with no other tag
- C. A random GUID applied as the only image tag
- D. An immutable SemVer or CalVer tag alongside `latest`
- E. Retagging the version onto each rebuild

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-14.md`:** lines **330–333** and **340–342**.

**A answers *where did this come from*; D answers *what do I redeploy*.** Both requirements are on the
list and neither tag serves both purposes.

**Why E is the practice that quietly destroys rollback** (Q23). If `1.2.0` can be retagged, "redeploy
1.2.0" is not a deterministic instruction — and nothing in the deployment record shows it changed.

</details>

---

## Q46

How is "a published version must always refer to the same artifact" enforced?

- A. A code review rule against republishing
- B. Deleting and carefully republishing the version
- C. A naming convention marking published versions
- D. Registry immutability — ship a new version instead

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-14.md`:** line **372**.

```text
npm ERR! You cannot publish over the previously published versions: 1.3.0
```

**The registry enforces it, which is why it holds under pressure.** A convention would last until the
first urgent defect (Q26).

**Why B is the workaround that breaks consumers.** Even where a registry permits unpublishing, a consumer
whose lockfile pins `1.3.0` now has a hash that matches nothing — their reproducible build stops
reproducing.

**The correct response to a bad version is `1.3.1`**, and deprecation of `1.3.0` if it is dangerous —
warning consumers while leaving the artifact intact for audit.

</details>

---

## Q47

Eight months in, the platform team notices that every package from the `contoso-data-models` pipeline has
been publishing as `0.1.0-ci.N` for six weeks, overwriting nothing because the pre-release identifier
differs each time. Consumers pinned to `^2.0.0` stopped receiving updates. The pipeline is green on every
run, GitVersion is configured correctly, and CI checkout settings were recently standardised across all
repositories.

What is the most likely cause?

- A. The `GitVersion.yml` was deleted from the repository
- B. Standardisation applied a shallow checkout, so no tags
- C. The registry rejected the 2.x versions as duplicates
- D. Consumers changed their declared version ranges

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-14.md`:** lines **235–237**.

```yaml
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
```

**"Green on every run" plus "GitVersion configured correctly" points at its *input*, not its
configuration.** GitVersion computes from history and tags; a depth-1 clone has one commit and no tags, so
it falls back to a `0.1.0` baseline.

**And the `-ci.N` suffix is why nothing failed.** In `ContinuousDeployment` mode every commit produces a
distinct version (Q13), so each publish **succeeded** — there was no 403 to notice, and no alert.

**The consumer-side symptom is the one that eventually surfaces**, and it is silent too: a range of
`^2.0.0` simply never matches a `0.x` version, so consumers stop receiving updates **without an error**.
Nothing breaks; things merely stop improving.

**This is the fourth challenge in this course where `fetch-depth: 0` is the answer** — commitlint,
changelog generation, drift detection, and now version derivation. **Any tool that reads history needs
the history**, and the failure is almost always green.

</details>

---

## Q48

A year on, any deployed artifact can be traced to a commit and a run, consumers upgrade libraries with
confidence, and Dependabot opens classified update pull requests automatically.

Which explanation best accounts for the change?

- A. The version became a fact derived from the change
- B. Teams agreed to be more consistent about versioning
- C. A versioning policy document was published
- D. Releases were made less frequent across the board

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-14.md`:** lines **21–26**, **34–37**, **122–132**, **382–433**, **330–342**.

**Take the four complaints at line 21 in turn.**

*Rollback failures* — because `latest` and random build numbers name nothing you can redeploy. An
immutable SemVer or CalVer tag is a target.

*Dependency conflicts on shared majors* — because a version that does not encode compatibility cannot warn
you. MAJOR is that warning, and it is the only one a package manager understands.

*No automated update policies* — because Dependabot classifies by SemVer (Q19). Without it, every update
is unclassifiable and therefore unautomatable.

*Untraceable artifact lineage* — because a tag with no commit in it cannot be traced. Build ID plus short
SHA closes the chain to Challenge 03's traceability.

**What actually changed is that versions stopped being *authored*.** The scenario describes three
different humans making three different choices, plus three teams making none — and no amount of policy
fixes that, because the decision was manual and manual decisions diverge.

**The graded idea: a good version is *derived*, not chosen.** GitVersion derives it from branch and tag
history; `counter()` derives it from atomic pipeline state; CalVer derives it from the clock. **Once it is
derived, consistency is free** — which is why the answer to "how do we get 15 teams to version the same
way" is never "tell them to".

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Pre-release assumed to outrank the release** | Q1, Q18, Q27 | alpha < beta < rc < release |
| **Patch not reset on a higher-order bump** | Q4, Q27, Q32, Q34 | `2.1.0` + feature + fix = **2.2.0** |
| **CalVer used on a library** | Q19, Q25, Q42 | No compatibility signal; breaks update automation |
| **`Build.BuildId` offered as sequential** | Q3, Q29, Q35 | Unique, not sequential per package |
| **`-` used where `+` was meant** | Q41 | `-` is a pre-release and sorts BELOW. `+` is metadata |
| **Build metadata assumed to affect precedence** | Q7, Q21, Q28 | Ignored entirely by resolvers |
| **Reading the newest tag and adding one** | Q5, Q20, Q25, Q44 | The race. Two merges, one version, 403 |
| **Retry-on-403 treated as the fix** | Q6, Q44 | Publishes a version nobody predicted |
| **Republishing over a version** | Q26, Q46 | Immutable. Ship `1.3.1` and deprecate |
| **`latest` as the only image tag** | Q16, Q23, Q45 | No rollback target |
| **Retagging a SemVer tag onto a rebuild** | Q23, Q45 | Breaks rollback with no record |
| **Shallow clone with GitVersion** | Q14, Q34, Q47 | Versions from `0.1.0`, and the build is GREEN |
| **`increment: Minor` on a release branch** | Q39 | `None` — the version is already decided |
| **Wrong GitVersion output fed to pack** | Q15, Q34 | `nuGetVersion` for NuGet |

---

# What to memorise

**In `challenge-14.md`:** lines **34–58**, **114–132**, **202–221**, **262–318**, **378–433**.

```text
SEMVER communicates COMPATIBILITY        CALVER communicates AGE
  MAJOR  incompatible API change           YYYY.MM.DD   daily / rolling
  MINOR  backward-compatible FEATURE       YYYY.MM.MICRO monthly + patch count
  PATCH  backward-compatible FIX           YYYY.MINOR.MICRO yearly major
  a HIGHER bump RESETS everything below:  2.1.0 + feature + fix = 2.2.0  (NOT 2.2.1)
                                          2.4.7 + breaking      = 3.0.0  (NOT 3.4.7)

  pre-release  1.0.0-alpha.1 < 1.0.0-beta.1 < 1.0.0-rc.1 < 1.0.0    (release ALWAYS wins)
  metadata     1.0.0+20240615   1.0.0-beta.1+build.42
               after '+' -> DOES NOT AFFECT PRECEDENCE. '-' would make it a PRE-RELEASE

  LIBRARY -> SemVer (consumers + Dependabot need the compatibility signal)
  APPLICATION on a release train -> CalVer (+ a run number, or two builds a day collide)
```

```bash
# npm                                              (lines 73-86)
npm version patch | minor | major
npm version preminor --preid=beta     # 2.0.0 -> 2.1.0-beta.0   (starts the series)
npm version prerelease                # 2.1.0-beta.0 -> beta.1  (advances it)

# NuGet - override at build time, do NOT edit the csproj in CI    (line 109)
dotnet pack --configuration Release /p:Version=1.2.0-beta.1

# CalVer                                           (lines 130-132)
VERSION="$(date +%Y.%m.%d).${GITHUB_RUN_NUMBER:-0}"    # the run number prevents same-day collisions

# Artifact tagging                                 (lines 330-359)
docker push registry/svc:1.2.0        # IMMUTABLE - the rollback target
docker push registry/svc:latest       # mutable convenience. NEVER the only tag
IMAGE_TAG="$(Build.BuildId)-$(echo $SHA | cut -c1-7)"   # run + commit = traceability
VERSION="1.2.0+build.42.sha.a1b2c3d"                    # still version 1.2.0 to any resolver
```

```yaml
# Azure Pipelines - the ATOMIC counter             (lines 266-289)
variables:
  majorVersion: 1
  minorVersion: 3
  patchVersion: $[counter(format('{0}.{1}', variables['majorVersion'], variables['minorVersion']), 0)]
  packageVersion: $(majorVersion).$(minorVersion).$(patchVersion)
#   $[ ] runtime | $( ) macro | ${{ }} compile-time      counter is seeded on major.minor -> resets
steps:
  - task: DotNetCoreCLI@2
    inputs: {command: pack, versioningScheme: byEnvVar, versionEnvVar: packageVersion}

# branch -> maturity                               (lines 295-307)
  main            ''                    <- the ONLY clean release version
  feature/        -alpha.$(Build.BuildId)
  release/        -rc.$(Build.BuildId)
  else            -dev.$(Build.BuildId)
echo "##vso[build.updatebuildnumber]$VERSION"          # renames the RUN
echo "##vso[task.setvariable variable=packageVersion]$VERSION"   # sets a VARIABLE
```

```yaml
# GitVersion - derives the version from history    (lines 202-256)
mode: ContinuousDeployment      # a DISTINCT version per commit -> the race cannot happen
branches:
  main:     {increment: Patch,  tag: ''}
  feature:  {increment: Minor,  tag: alpha}
  release:  {increment: None,   tag: rc}     # None - the version is already decided
  hotfix:   {increment: Patch,  tag: beta}

- uses: actions/checkout@v4
  with: {fetch-depth: 0}        # MANDATORY - shallow => no tags => versions from 0.1.0, GREEN
- uses: gittools/actions/gitversion/setup@v1
- uses: gittools/actions/gitversion/execute@v1
  id: gitversion
- run: dotnet pack /p:Version=${{ steps.gitversion.outputs.nuGetVersion }}
#   outputs: semVer | nuGetVersion | informationalVersion | assemblySemVer - use the right one

# THE RACE (lines 366-433)
# two merges seconds apart read the same latest tag -> both compute 1.3.0 -> second publish 403
#   "You cannot publish over the previously published versions"   <- versions are IMMUTABLE
# fixes: counter() (atomic) | GitVersion ContinuousDeployment (per-commit) | run number
#        retry-on-403 works and publishes a version nobody predicted
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 15 |
| 38–43 | Re-read the trap index and the SemVer reset rule, then move on |
| 30–37 | Write the three components, the precedence order and the race fixes from memory, then retake |
| Below 30 | Redo Tasks 1, 3 and 4 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 14.

:::danger The two sentences

**SemVer communicates compatibility; CalVer communicates age.** Libraries need the first because consumers
and update bots read it. Applications on a release train need the second.

**A published version is immutable, and a good version is derived rather than chosen.** GitVersion derives
it from history, `counter()` from atomic pipeline state, CalVer from the clock — and once it is derived,
15 teams are consistent for free.

And a higher bump resets everything below it: `2.1.0` plus a feature and a fix is **2.2.0**.

:::
