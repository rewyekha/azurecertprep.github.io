---
sidebar_position: 1.5
toc_max_heading_level: 2
title: "Challenge 13: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 13 — AZ-400 exam questions

**48 questions** built only from what Challenge 13 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-13.md`**.

:::danger Read this before you start

**A feed is where packages live. A view is which *quality* of package a consumer sees. An upstream source
is where the feed goes when it does not have something.** Three ideas, and almost every question here is
one of them.

**The highest-yield idea is upstream sources.** Point every build at your feed, list npmjs and nuget.org
as upstreams, and the feed **fetches and caches** anything public on first request (line 283). You get
one registry URL in every config, resilience when the public registry is down or a package is yanked, and
one place to audit what your organisation actually consumes.

**Views are the other half.** Azure Artifacts lets one feed serve `@prerelease` to the team and
`@release` to consumers, with promotion between them (line 270). **GitHub Packages has no equivalent** —
it inherits repository and organisation permissions instead.

**And two authentication facts that decide the Break scenario.** GitHub Packages requires the package
**scope to match the repository owner** — `@contoso/auth-sdk` must live under `contoso`. And a workflow
publishing a package needs **`packages: write`** declared explicitly.

The scenario at line 21: 15 microservices, 4 shared libraries, and teams **copying source code between
repositories**.

:::

---

# Section A — Multiple choice

---

## Q1

Which package types does GitHub Packages support?

- A. npm and Docker container images only
- B. npm, Maven, NuGet, Docker and RubyGems
- C. npm packages only, no other ecosystem
- D. npm, Maven and pip, but not NuGet

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-13.md`:** line **425**.

```text
GitHub Packages supports npm, Maven, NuGet, Docker (container images), and RubyGems.
Python (pip) packages are not natively supported by GitHub Packages.
```

**Five ecosystems, and the exam tests the gap rather than the list.** **pip is the absent one**, which
matters for Contoso only if a Python service appears later — and is exactly the kind of constraint that
decides a platform choice.

**Why D is the distractor built from that gap.** It swaps a real ecosystem for the missing one, which is
the shape of a well-made wrong answer.

</details>

---

## Q2

What is the maximum number of upstream sources in a single Azure Artifacts feed?

- A. 1 upstream source per feed
- B. 5 upstream sources per feed
- C. 10 upstream sources per feed
- D. No hard limit per feed

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-13.md`:** line **436**.

```text
Azure Artifacts does not impose a hard limit on the number of upstream sources per feed.
```

**No hard limit — with a performance caveat.** Each upstream is consulted in priority order when a
package is not found locally, so a long list means a long miss path.

**Which is why priority order matters more than count** (lines 290–309): put the internal feed first so
an internal package always wins over a public one with the same name.

</details>

---

## Q3

Which platform provides native package vulnerability scanning?

- A. Azure Artifacts with Defender for Cloud only
- B. GitHub Packages with Dependabot only
- C. Both platforms, through different tooling
- D. Neither platform without a third-party scanner

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-13.md`:** line **447**.

```text
Both platforms provide native vulnerability scanning. GitHub uses Dependabot for security alerts and
automated PRs. Azure integrates with Microsoft Defender for DevOps to scan for vulnerable dependencies.
```

**The scenario asks for vulnerability scanning as a requirement** (line 23), so this is the question that
does **not** differentiate the platforms — and knowing that is the point.

**The difference is in what happens after detection.** Dependabot raises **pull requests** that upgrade
the dependency (Challenge 44); Defender for DevOps surfaces findings in a **dashboard** across
repositories (Challenge 45). **Detect on both; remediate automatically on one.**

</details>

---

## Q4

What happens the first time a package from an upstream source is requested?

- A. A copy is saved into the local feed cache
- B. It is fetched from upstream on every request
- C. It appears only in the prerelease view
- D. It requires manual approval before use

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-13.md`:** lines **283–286**.

```text
- First request: Package is fetched from upstream and saved to the local feed
- Subsequent requests: Package is served from the local feed cache
- If the upstream goes offline, cached packages remain available
- New versions from upstream are fetched on demand
```

**Read the third line — that is the operational payoff.** If npmjs is unreachable, or a package is
unpublished, or a version is yanked, **your builds keep working** because the feed already holds a copy.

**And the fourth line is the limit.** Caching is per **version**; a new version you have never installed
still requires the upstream to be reachable.

</details>

---

## Q5

`npm publish` to GitHub Packages returns 403. Which cause relates to the package name?

- A. The version already exists in the registry
- B. The registry is temporarily unavailable
- C. The package tarball exceeds the size limit
- D. The scope does not match the repository owner

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-13.md`:** lines **360–366**.

```bash
cat package.json | grep name
# Must output: "@contoso/auth-sdk" where "contoso" matches the org/user owning the repo
```

**GitHub Packages derives ownership from the scope**, so `@contoso/auth-sdk` can only be published from a
repository owned by `contoso`. Publish `@acme/auth-sdk` from a Contoso repository and you get 403 — not
404, because the request is understood and refused.

**This is the difference from Azure Artifacts worth noting.** An Azure Artifacts feed has no naming
constraint tied to the project; the **feed URL** determines where the package goes.

</details>

---

## Q6

A workflow fails to publish with 403. Which permission is missing?

- A. `contents: write`
- B. `packages: write`
- C. `id-token: write`
- D. `actions: write`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-13.md`:** lines **135–137** and **353–358**.

```yaml
    permissions:
      contents: read
      packages: write
```

```text
Without this, the `GITHUB_TOKEN` defaults to read-only for packages in workflows triggered by pull
requests from forks.
```

**Note `contents` stays `read`.** A publish job reads the source and writes a package; it has no business
writing to the repository.

**And the fork caveat at line 358 is the subtlety.** Default permissions differ by trigger — a workflow
that publishes happily on `release` can fail on a fork PR, which is why the block is declared explicitly
rather than relied upon.

</details>

---

## Q7

What does the `publishConfig.registry` field in `package.json` do?

- A. It sets the registry used for installs
- B. It authenticates the client to the registry
- C. It directs `npm publish` to that registry
- D. It sets the package's visibility level

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-13.md`:** lines **52–54**.

```json
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  }
```

**It binds the *publish* target to the package rather than to the machine.** Without it, `npm publish`
uses whatever registry the local `.npmrc` or global config happens to name — and a mis-set default
publishes an internal library to **npmjs.com, publicly**.

**Why A is the separate mechanism.** Installs are directed by the scope line in `.npmrc` (line 63):
`@contoso:registry=...`. **Publishing and consuming are configured in different places**, and the exam
tests which is which.

</details>

---

## Q8

What does `@contoso:registry=https://npm.pkg.github.com` accomplish?

- A. Every package resolves to GitHub Packages, not npmjs
- B. It authenticates the `@contoso` scope
- C. It creates the `@contoso` scope on the registry
- D. Only `@contoso` packages resolve there; others default

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-13.md`:** line **63**.

**Scope-based routing is what lets one project consume internal and public packages side by side.**
`@contoso/auth-sdk` goes to GitHub Packages; `lodash` goes to npmjs.

**Contrast with the Azure Artifacts `.npmrc` at line 195**, which sets `registry=` with **no scope** —
everything goes to the feed, and public packages arrive through the upstream (Q4).

**Two philosophies, and this is the cleanest way to see them.** GitHub Packages routes **by scope**;
Azure Artifacts routes **everything through the feed** and proxies outward.

</details>

---

## Q9

What does `always-auth=true` do in the Azure Artifacts `.npmrc`?

- A. Sends credentials on every request, reads included
- B. Caches the token between npm invocations
- C. Refreshes the token when it is close to expiry
- D. Requires multi-factor authentication for publish

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-13.md`:** line **196**.

```ini
registry=https://pkgs.dev.azure.com/contoso/.../npm/registry/
always-auth=true
```

**npm normally authenticates only for publishing.** A private Azure Artifacts feed requires
authentication for **reads** too, so without this flag `npm install` gets 401 while `npm publish` works —
a confusing asymmetry.

**And it is required precisely because the feed is the *default* registry here** (Q8): every install goes
through it, including public packages fetched via upstream.

</details>

---

## Q10

Which Azure Artifacts feed role can consume packages **and** save packages from upstream sources, but
cannot publish?

- A. Reader, consume only
- B. Collaborator, consume and cache
- C. Contributor, publish too
- D. Owner, full control

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-13.md`:** lines **242–245**.

```text
- **Reader**: Can consume packages from the feed
- **Collaborator**: Can consume packages and save packages from upstream sources
- **Contributor**: Can publish new packages and versions
- **Owner**: Full control including feed deletion and permission management
```

**Collaborator exists because pulling a package through an upstream *writes* to your feed** (Q4) — a
Reader cannot trigger that save, so a Reader's `npm install` of a new public package fails.

**Which makes Reader narrower than it looks**, and Collaborator the right default for a consuming team
that must not publish.

</details>

---

## Q11

What do Azure Artifacts **views** provide?

- A. A separate feed for each deployment environment
- B. Package vulnerability scanning scoped to a view
- C. Exposing only promoted versions from one feed
- D. Access control applied per individual package

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-13.md`:** lines **249–274**.

```bash
    "name": "prerelease",
    "visibility": "private"
...
    "name": "release",
    "visibility": "organization"
```

**One feed, two audiences.** Every build publishes into the feed; only promoted versions reach the
`release` view, which is `organization`-visible while `prerelease` stays private.

**Promotion is an explicit act** (lines 270–274) — adding the version to a view — which is what makes it
a quality gate rather than a naming convention.

**And GitHub Packages has no equivalent**, which is one of the few genuine feature differences in this
challenge.

</details>

---

## Q12

How is a package version promoted to the `release` view?

- A. Republishing it under a new version number
- B. Copying the version into a separate feed
- C. Editing the version field in `package.json`
- D. A POST adding `release` to the version's views

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-13.md`:** lines **270–274**.

```json
    "views": { "op": "add", "path": "/views/-", "value": "release" }
```

**The version is not moved or rebuilt — a label is added.** The same immutable artifact is now visible in
both views, which is exactly what you want: **the thing you tested is the thing you ship.**

**Why A is the anti-pattern this prevents.** Rebuilding for release produces a different artifact from
the one that passed testing — the same principle as Challenge 28's build-once-deploy-many.

</details>

---

## Q13

Why place an internal upstream source **before** a public one?

- A. So an internal package beats a same-named public one
- B. For faster resolution of the most common packages
- C. To reduce the cost of upstream bandwidth
- D. It is required by the upstream configuration syntax

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-13.md`:** lines **290** and **296–308**.

```text
Place your internal feed first so internal packages take precedence over public ones with the same name
```

**This is dependency-confusion defence.** If an attacker publishes `@contoso/auth-sdk` to npmjs and your
feed checks public first, your build installs theirs.

**Ordering is the control**, and it is one line of configuration — which is why the exam likes it: a
security property expressed as an array order.

</details>

---

## Q14

What does `--scope project` do when creating a feed?

- A. It sets the npm scope used for publishing
- B. It scopes the feed to one project, not the org
- C. It limits which package types the feed accepts
- D. It sets the default permissions on the feed

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-13.md`:** lines **155–159**.

```bash
az artifacts feed create \
  --name contoso-packages \
  --project ContosoServices \
  --scope project
```

**Project-scoped feeds inherit project permissions and are visible within that project.**
Organisation-scoped feeds are shared across every project — which is the right choice for the four
libraries Contoso's 15 services all consume.

**Why the distinction is worth flagging in a recommendation.** Contoso's shared libraries span services
that may live in different projects; a project-scoped feed would need explicit sharing.

</details>

---

## Q15

How does the NuGet push authenticate to Azure Artifacts?

- A. A NuGet API key generated on the feed
- B. Azure AD interactive login on the agent
- C. A PAT on the source, with `--api-key az` on push
- D. No authentication for a project-scoped feed

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-13.md`:** lines **211–225**.

```bash
dotnet nuget add source "..." --name contoso-packages \
  --username contoso --password $AZURE_DEVOPS_PAT --store-password-in-clear-text

dotnet nuget push ./nupkgs/*.nupkg --source contoso-packages --api-key az
```

**`--api-key az` is a placeholder, not a real key.** The `dotnet` CLI requires the argument to be present;
the actual credential is the PAT stored on the source. The literal `az` is a convention, and the exam
quotes it.

**And `--store-password-in-clear-text` is a genuine smell.** It writes the PAT into `NuGet.Config` in
plain text — acceptable on an ephemeral CI agent, and something to replace with a credential provider or
a service connection on a developer machine.

</details>

---

## Q16

Which setting makes a GitHub package visible to all organisation members?

- A. `visibility=public`
- B. `visibility=private`
- C. `visibility=organization`
- D. `visibility=internal`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-13.md`:** lines **111–112**.

```bash
gh api --method PUT /orgs/contoso/packages/npm/auth-sdk/visibility \
  -f visibility=internal
```

**Three values, three audiences.** `private` is the package's linked repository's collaborators,
**`internal` is everyone in the organisation**, and `public` is the world.

**`internal` is the correct setting for Contoso's shared libraries** — all 15 service teams need them,
and nobody outside should.

**Why C is the plausible invention.** The word for "everyone in the org" is `internal`, not
`organization` — which is the sort of vocabulary detail the exam checks.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** package types does GitHub Packages support? (Choose three.)

- A. pip (Python)
- B. npm (JavaScript)
- C. Cargo (Rust)
- D. NuGet (.NET)
- E. Composer (PHP)
- F. Docker (containers)

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-13.md`:** line **425**.

**Maven and RubyGems are the other two** — five in total.

**And the exam's angle is always the absence.** pip is not natively supported, so a Python service in
Contoso's future would need Azure Artifacts (which supports Python feeds) or a third-party registry.

</details>

---

## Q18

Which **three** are Azure Artifacts feed roles? (Choose three.)

- A. Reader
- B. Publisher
- C. Collaborator
- D. Consumer
- E. Contributor
- F. Maintainer

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-13.md`:** lines **242–245**.

**Owner is the fourth.** Four roles, and the one that carries information is **Collaborator** — consume,
plus save from upstream, without publishing (Q10).

**Why B, D and F are plausible-sounding inventions.** They are the words you would guess if you had not
read the list, which is precisely why they are offered.

</details>

---

## Q19

Which **three** are true of upstream sources? (Choose three.)

- A. They require manual approval per package
- B. The first request fetches from upstream and caches
- C. They are limited to five sources per feed
- D. Cached packages survive an upstream outage
- E. They only work for the npm ecosystem
- F. Priority order decides which source wins a name

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-13.md`:** lines **283–285**, **290**, **436**.

**B and D are the resilience argument; F is the security argument** (Q13).

**Why E is refuted by the challenge itself** — nuget.org is configured as an upstream at lines 180–187,
alongside npmjs at 167–174.

**And why A matters as a distraction.** Nothing approves a package on first fetch. If you want approval,
that is a separate control — which is Challenge 15's allow and deny lists.

</details>

---

## Q20

Which **two** distinguish GitHub Packages from Azure Artifacts in this challenge? (Choose two.)

- A. Only Azure Artifacts supports npm packages
- B. Azure Artifacts has views for promoting quality
- C. Only GitHub Packages supports NuGet packages
- D. GitHub Packages requires scope to match the owner
- E. Only GitHub Packages offers vulnerability scanning

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-13.md`:** lines **247–274** and **362**.

**These are the two real differences.** Views are a capability Azure Artifacts has and GitHub Packages
does not; scope-to-owner is a constraint GitHub Packages has and Azure Artifacts does not.

**Why A, C and E are all false** — both support npm and NuGet (lines 150, 425), and both scan (Q3).

**Which is the honest framing for the recommendation**: the platforms overlap heavily, so the decision
rests on **where the consumers and permissions already are**, not on a feature matrix.

</details>

---

## Q21

Which **two** authenticate npm to a private registry? (Choose two.)

- A. An `_authToken` line in `.npmrc` for the registry host
- B. `publishConfig.registry` in `package.json`
- C. `NODE_AUTH_TOKEN` supplied to `npm publish` in CI
- D. The `@scope:registry` line in `.npmrc`
- E. `npm login --scope` on the developer machine

<details>
<summary>Show answer</summary>

### Answer: A, C

**In `challenge-13.md`:** lines **64** and **147**.

```ini
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

```yaml
        env:
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Two forms of the same credential**, chosen by context: a file on a developer machine, an environment
variable in CI.

**Why B and D are *routing*, not authentication** (Q7, Q8) — they say where to go, not who you are. That
separation is the cleanest way to reason about a 401 versus a 404.

**And `actions/setup-node` with `registry-url`** (line 143) is what writes the `.npmrc` for the runner, so
`NODE_AUTH_TOKEN` has somewhere to be injected.

</details>

---

## Q22

Which **two** does the publish workflow require? (Choose two.)

- A. `contents: write` in the permissions block
- B. `permissions: packages: write` on the job
- C. `id-token: write` in the permissions block
- D. A PAT stored as a repository secret
- E. `registry-url` on `actions/setup-node`

<details>
<summary>Show answer</summary>

### Answer: B, E

**In `challenge-13.md`:** lines **135–147**.

**B grants the built-in token the right to publish; E configures npm to know where.** Without E, npm
publishes to the default registry — or fails, depending on `publishConfig`.

**Why D is unnecessary here and necessary elsewhere.** `secrets.GITHUB_TOKEN` suffices because the
package belongs to **this repository's** organisation. Publishing to a **different** organisation's
registry would need a PAT (Challenge 40's reach rule).

</details>

---

## Q23

Which **two** causes of a GitHub Packages 403 are configuration in the repository? (Choose two.)

- A. GitHub Packages being temporarily offline
- B. The package tarball exceeding the size limit
- C. Missing `packages: write` in the workflow
- D. Rate limiting on the publish endpoint
- E. Package scope not matching the repository owner

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-13.md`:** lines **340–342**.

```text
1. Missing `packages: write` permission in the workflow
2. Package name does not match the repository owner scope
3. `.npmrc` pointing to wrong registry
4. Token does not have `write:packages` scope
```

**Four listed causes; C and E live in the repository, while 3 and 4 live in the developer's environment.**

**That split is the diagnostic order.** If it fails in CI, check the workflow's permissions and the
package name. If it fails locally, check `.npmrc` and the token scopes (line 406).

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must replace copy-pasted source with a centralised package solution offering private
hosting, vulnerability scanning, per-team access control, and support for npm **and** NuGet across 15
services.

---

## Q24

**Proposed solution:** Create an Azure Artifacts feed. Add npmjs.org and nuget.org as public upstream
sources, with the internal shared feed listed first in priority order. Point every service's `.npmrc` and
NuGet source at the feed so all packages resolve through it. Grant consuming teams Collaborator and
publishing teams Contributor. Create `prerelease` and `release` views and promote versions into `release`
after testing. Rely on Defender for DevOps for dependency scanning.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-13.md`:** lines **155–159**, **167–187**, **290–309**, **195–226**, **242–245**,
**249–274**, **447**.

| Requirement | Mechanism |
|---|---|
| Private hosting | The feed itself |
| npm **and** NuGet | One feed, both protocols |
| Per-team access control | Reader / Collaborator / Contributor / Owner |
| Vulnerability scanning | Defender for DevOps |
| No copy-pasted source | Shared libraries published once, consumed by version |
| Consumers see only tested versions | `release` view with explicit promotion |

**The upstream-first ordering is the clause that is easy to omit and expensive to miss** (Q13) — it is
what stops a public package impersonating an internal one.

</details>

---

## Q25

**Proposed solution:** Use GitHub Packages. Have each team publish their libraries under their own scope,
such as `@backend/auth-sdk` and `@platform/logging-sdk`. Configure each consuming service's `.npmrc` with
one registry line pointing at GitHub Packages for everything. Keep using npmjs directly for public
packages. Skip vulnerability scanning for now, since the libraries are internal.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures.**

**Per-team scopes will not publish.** GitHub Packages requires the scope to match the **repository
owner** (Q5), and `@backend` is a team, not an organisation — so every publish returns 403.

**A single unscoped registry line sends *every* package to GitHub Packages**, including `lodash`, which
is not there. Scope-based routing (line 63) is what makes GitHub Packages work alongside npmjs.

**"Keep using npmjs directly" forfeits the audit and resilience** an upstream would provide — and it
means two registries to configure and two to trust.

**And "internal libraries do not need scanning" misunderstands the risk.** The vulnerability is almost
never in the four libraries; it is in **their transitive dependencies**, which are public. The
requirement at line 23 asks for scanning **on dependencies**.

</details>

---

## Q26

**Proposed solution:** Create an Azure Artifacts feed with npmjs and nuget.org upstreams, internal feed
first. Point every service at the feed. Grant Collaborator to consumers and Contributor to publishers.
Create `prerelease` and `release` views. Rely on Defender for DevOps for scanning. To keep the process
simple, have CI publish straight into the `release` view so consumers always get the newest build without
a promotion step.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**Publishing straight to `release` deletes the only thing views are for.** The `release` view exists to
mean *this version was tested and promoted*; if every CI build lands there, it means *this version
exists*, which the feed already told you.

**And the practical consequence lands on the 15 consuming services.** A library author pushes
`1.4.0-experimental`, every consumer resolving `^1.0.0` picks it up on their next install, and a change
that was never meant to leave the author's branch is now in fifteen builds.

**The `visibility` difference makes it worse** (lines 255, 263): `prerelease` is `private` and `release`
is `organization`-visible. Publishing into `release` makes every build immediately visible
organisation-wide.

**Promotion is one API call** (lines 270–274) and it adds a label to an **existing** version — it does not
rebuild or republish, so it costs nothing but the decision. **That decision is the control.**

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — feeds and views

| # | Statement | Answer |
|---|---|---|
| 1 | A view exposes a subset of a feed's package versions |  |
| 2 | Promotion adds the existing version to a view |  |
| 3 | GitHub Packages has an equivalent view concept |  |
| 4 | A feed can serve both npm and NuGet |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A view exposes a subset of a feed's package versions | **Yes** |
| 2 | Promotion adds the existing version to a view | **Yes** |
| 3 | GitHub Packages has an equivalent view concept | **No** |
| 4 | A feed can serve both npm and NuGet | **Yes** |

**In `challenge-13.md`:** lines **249–264**, **270–274**, **247**, **150**.

Row 3 is one of only two real capability differences in this challenge (Q20).

Row 4 is why one Azure Artifacts feed satisfies the "npm **and** NuGet" requirement at line 25.

</details>

---

## Q28 — upstream sources

| # | Statement | Answer |
|---|---|---|
| 1 | The first request caches the package locally |  |
| 2 | Cached packages survive an upstream outage |  |
| 3 | Public sources should be listed before internal ones |  |
| 4 | New upstream versions are fetched on demand |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | The first request caches the package locally | **Yes** |
| 2 | Cached packages survive an upstream outage | **Yes** |
| 3 | Public sources should be listed before internal ones | **No** |
| 4 | New upstream versions are fetched on demand | **Yes** |

**In `challenge-13.md`:** lines **283–286** and **290**.

Row 3 is inverted, and the inversion is a **security** defect: a public package with an internal name
would win (Q13).

</details>

---

## Q29 — GitHub Packages

| # | Statement | Answer |
|---|---|---|
| 1 | The package scope must match the repository owner |  |
| 2 | `visibility=internal` exposes it to all organisation members |  |
| 3 | `@scope:registry` in `.npmrc` routes installs for that scope |  |
| 4 | pip packages are supported |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | The package scope must match the repository owner | **Yes** |
| 2 | `visibility=internal` exposes it to all organisation members | **Yes** |
| 3 | `@scope:registry` in `.npmrc` routes installs for that scope | **Yes** |
| 4 | pip packages are supported | **No** |

**In `challenge-13.md`:** lines **362**, **111–112**, **63**, **425**.

Row 1 is the 403 that reads as a permissions problem and is a **naming** problem.

Row 4 is the gap the exam builds its distractor from (Q1).

</details>

---

## Q30 — authentication

| # | Statement | Answer |
|---|---|---|
| 1 | `always-auth=true` sends credentials on reads as well as publishes |  |
| 2 | `--api-key az` is a placeholder, not a real key |  |
| 3 | `publishConfig.registry` authenticates the publish |  |
| 4 | A workflow publishing a package needs `packages: write` |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `always-auth=true` sends credentials on reads as well as publishes | **Yes** |
| 2 | `--api-key az` is a placeholder, not a real key | **Yes** |
| 3 | `publishConfig.registry` authenticates the publish | **No** |
| 4 | A workflow publishing a package needs `packages: write` | **Yes** |

**In `challenge-13.md`:** lines **196**, **225**, **52–54**, **137**.

Row 3 is the routing-versus-authentication split (Q21): `publishConfig` says **where**, `_authToken` and
`NODE_AUTH_TOKEN` say **who**.

Row 1 is why a private feed can publish successfully and fail on `npm install`.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each requirement to its mechanism.

| Requirement | Mechanism |
|---|---|
| Private hosting for internal libraries |  |
| Public packages available through one URL |  |
| Builds survive a public registry outage |  |
| Consumers see only tested versions |  |
| A team may consume but not publish |  |
| An internal name cannot be hijacked publicly |  |

**Options:** A feed, or a scoped GitHub package · Collaborator role · Internal upstream listed first · The upstream cache · Upstream sources · Views with explicit promotion

<details>
<summary>Show answer</summary>

| Requirement | Mechanism |
|---|---|
| Private hosting for internal libraries | **A feed, or a scoped GitHub package** |
| Public packages available through one URL | **Upstream sources** |
| Builds survive a public registry outage | **The upstream cache** |
| Consumers see only tested versions | **Views with explicit promotion** |
| A team may consume but not publish | **Collaborator role** |
| An internal name cannot be hijacked publicly | **Internal upstream listed first** |

**In `challenge-13.md`:** lines **155–159**, **167–174**, **285**, **249–274**, **243**, **290**.

**Rows 2, 3 and 6 are all the same feature answering three different questions** — which is why upstream
sources are the highest-yield idea in this challenge.

</details>

---

## Q32

Match each Azure Artifacts role to what it may do.

| Role | May |
|---|---|
| Reader |  |
| Collaborator |  |
| Contributor |  |
| Owner |  |

**Options:** Consume, and save packages from upstream · Consume packages already in the feed · Everything, including deletion and permissions · Publish new packages and versions

<details>
<summary>Show answer</summary>

| Role | May |
|---|---|
| Reader | **Consume packages already in the feed** |
| Collaborator | **Consume, and save packages from upstream** |
| Contributor | **Publish new packages and versions** |
| Owner | **Everything, including deletion and permissions** |

**In `challenge-13.md`:** lines **242–245**.

**Reader is narrower than it sounds.** Pulling a public package through an upstream **writes** to the
feed, so a Reader's first `npm install lodash` fails (Q10) — Collaborator is the practical minimum for a
consuming team.

</details>

---

## Q33

Arrange the steps to publish and consume an internal npm library through GitHub Packages.

**Items:** Add `@contoso:registry` and `_authToken` to the consumer's `.npmrc` · Scope the package name to
the organisation and set `publishConfig` · Publish from a workflow with `packages: write` · Set the
package visibility to `internal` · `npm install @contoso/auth-sdk`

<details>
<summary>Show answer</summary>

### Answer

1. Scope the package name to the organisation and set `publishConfig` — lines **44–54**
2. Publish from a workflow with `packages: write` — lines **135–147**
3. Set the package visibility to `internal` — lines **111–112**
4. Add `@contoso:registry` and `_authToken` to the consumer's `.npmrc` — lines **96–97**
5. `npm install @contoso/auth-sdk` — line **103**

**Step 1 first, because it is the precondition for step 2 succeeding at all.** A name that does not match
the repository owner returns 403 no matter how the workflow is configured (Q5).

**And step 3 before step 4 matters on a first publish.** A package defaults to the linked repository's
visibility; a consumer in another repository gets 404 until it is `internal` — **a 404, not a 403**,
because GitHub does not confirm the existence of packages you cannot see.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| `403 Forbidden` on publish from CI |  |
| `403 Forbidden` on publish locally |  |
| `npm install` of a public package 401s |  |
| `npm install lodash` fails for one team |  |
| An internal name resolves to a public package |  |
| A consumer cannot see a published package |  |

**Options:** `always-auth=true` missing on a private feed · `packages: write` missing from the workflow · Public upstream listed before internal · They hold Reader, not Collaborator · Token lacks `write:packages`, or the scope is wrong · Visibility still private

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| `403 Forbidden` on publish from CI | **`packages: write` missing from the workflow** |
| `403 Forbidden` on publish locally | **Token lacks `write:packages`, or the scope is wrong** |
| `npm install` of a public package 401s | **`always-auth=true` missing on a private feed** |
| `npm install lodash` fails for one team | **They hold Reader, not Collaborator** |
| An internal name resolves to a public package | **Public upstream listed before internal** |
| A consumer cannot see a published package | **Visibility still private** |

**In `challenge-13.md`:** lines **353**, **401**, **196**, **243**, **290**, **111**.

**The two 403s look identical and are diagnosed in different places** — CI's is in the workflow file,
local's is in the token (Q23).

</details>

---

## Q35

Match each configuration line to its job.

| Line | Job |
|---|---|
| `@contoso:registry=https://npm.pkg.github.com` |  |
| `//npm.pkg.github.com/:_authToken=...` |  |
| `publishConfig.registry` in `package.json` |  |
| `always-auth=true` |  |
| `registry-url` on `actions/setup-node` |  |
| `NODE_AUTH_TOKEN` |  |

**Options:** Authenticates reads as well as writes · Authenticates to that host · Routes installs for that scope · Routes `npm publish` · Supplies the credential in CI · Writes the runner's `.npmrc`

<details>
<summary>Show answer</summary>

| Line | Job |
|---|---|
| `@contoso:registry=https://npm.pkg.github.com` | **Routes installs for that scope** |
| `//npm.pkg.github.com/:_authToken=...` | **Authenticates to that host** |
| `publishConfig.registry` in `package.json` | **Routes `npm publish`** |
| `always-auth=true` | **Authenticates reads as well as writes** |
| `registry-url` on `actions/setup-node` | **Writes the runner's `.npmrc`** |
| `NODE_AUTH_TOKEN` | **Supplies the credential in CI** |

**In `challenge-13.md`:** lines **63**, **64**, **53**, **196**, **143**, **147**.

**Three route, three authenticate.** Getting a 404 usually means routing; a 401 or 403 means
authentication — and knowing which half a line belongs to halves the search.

</details>

---

# Section F — Hot area

---

## Q36

```json
{
  "name": "[BLANK 1]",
  "version": "1.0.0",
  "[BLANK 2]": {
    "registry": "https://npm.pkg.github.com"
  }
}
```

Requirement: publish to GitHub Packages under the `contoso` organisation.

- **BLANK 1:** `contoso-auth-sdk` / `@auth/contoso-sdk` / `@contoso/auth-sdk` / `auth-sdk`
- **BLANK 2:** `publish` / `publishConfig` / `registry` / `distConfig`

<details>
<summary>Show answer</summary>

### Answer: `@contoso/auth-sdk`, `publishConfig`

**In `challenge-13.md`:** lines **44** and **52**.

**The scope must be the organisation** (Q5) — `@auth/contoso-sdk` inverts it and returns 403, and an
unscoped name cannot be published to GitHub Packages at all.

**And `publishConfig` binds the publish target to the package**, so a developer with a different default
registry cannot accidentally push an internal library to npmjs (Q7).

</details>

---

## Q37

```ini
[BLANK 1]=https://npm.pkg.github.com
//npm.pkg.github.com/:[BLANK 2]=${GITHUB_TOKEN}
```

Requirement: only Contoso-scoped packages come from GitHub Packages; everything else from npmjs.

- **BLANK 1:** `registry` / `scope` / `@contoso` / `@contoso:registry`
- **BLANK 2:** `token` / `_authToken` / `auth` / `password`

<details>
<summary>Show answer</summary>

### Answer: `@contoso:registry`, `_authToken`

**In `challenge-13.md`:** lines **63–64**.

**A bare `registry=` would send *everything* to GitHub Packages** (Q25), including `lodash`, which is not
there — so installs fail for every public dependency.

**And `_authToken` with the leading underscore** is the npm convention; the host prefix `//host/:` scopes
the credential to that registry so one `.npmrc` can hold several.

</details>

---

## Q38

```ini
registry=https://pkgs.dev.azure.com/contoso/ContosoServices/_packaging/contoso-packages/npm/registry/
[BLANK 1]=true
//pkgs.dev.azure.com/.../npm/registry/:_authToken=${AZURE_DEVOPS_PAT}
```

- **BLANK 1:** `strict-ssl` / `save-exact` / `always-auth` / `auth-required`

<details>
<summary>Show answer</summary>

### Answer: `always-auth`

**In `challenge-13.md`:** line **196**.

**Note the `registry=` here has *no scope*** — deliberately. Azure Artifacts is the **default** registry
and public packages arrive through upstreams (Q8), which is the opposite routing philosophy to the GitHub
Packages `.npmrc` above.

**And `always-auth` is required precisely because of that**: every install, public or private, now goes
through an authenticated feed.

</details>

---

## Q39

```yaml
    permissions:
      contents: [BLANK 1]
      packages: [BLANK 2]
    steps:
      - uses: actions/setup-node@v4
        with:
          [BLANK 3]: https://npm.pkg.github.com/
      - run: npm publish
        env:
          [BLANK 4]: ${{ secrets.GITHUB_TOKEN }}
```

- **BLANK 1:** `write` / `read`
- **BLANK 2:** `read` / `write`
- **BLANK 3:** `registry` / `npm-registry` / `scope` / `registry-url`
- **BLANK 4:** `NPM_TOKEN` / `GITHUB_TOKEN` / `NODE_AUTH_TOKEN` / `AUTH_TOKEN`

<details>
<summary>Show answer</summary>

### Answer: `read`, `write`, `registry-url`, `NODE_AUTH_TOKEN`

**In `challenge-13.md`:** lines **136–147**.

**`NODE_AUTH_TOKEN` is the name `setup-node` writes into the generated `.npmrc`.** Calling it `NPM_TOKEN`
is the common mistake — the variable exists, npm never reads it, and the publish fails with 401.

**And `contents: read` is deliberate least privilege** — a publish job has no reason to write to the
repository.

</details>

---

## Q40

```bash
az rest --method post \
  --uri ".../feeds/contoso-packages/[BLANK 1]?api-version=7.1-preview.1" \
  --body '{
    "name": "npmjs",
    "protocol": "npm",
    "location": "https://registry.npmjs.org/",
    "[BLANK 2]": "public"
  }'
```

- **BLANK 1:** `views` / `upstreamsources` / `permissions` / `packages`
- **BLANK 2:** `type` / `visibility` / `upstreamSourceType` / `scope`

<details>
<summary>Show answer</summary>

### Answer: `upstreamsources`, `upstreamSourceType`

**In `challenge-13.md`:** lines **168–173**.

**`upstreamSourceType` takes `public` or `internal`** — and the internal form (line 302) points at
another Azure Artifacts feed, which is how one feed chains to another.

**Compare the `views` endpoint at line 251**, which takes `type` and `visibility` instead. **Different
endpoint, different body shape** — and the exam swaps them.

</details>

---

## Q41

```bash
az rest --method post \
  --uri ".../feeds/contoso-packages/npm/@contoso/auth-sdk/versions/1.0.0?api-version=7.1-preview.1" \
  --body '{
    "views": { "op": "[BLANK 1]", "path": "/views/-", "value": "[BLANK 2]" }
  }'
```

Requirement: promote a tested version so organisation consumers can see it.

- **BLANK 1:** `replace` / `move` / `add` / `copy`
- **BLANK 2:** `prerelease` / `latest` / `stable` / `release`

<details>
<summary>Show answer</summary>

### Answer: `add`, `release`

**In `challenge-13.md`:** lines **270–274**.

**`add` with `/views/-` appends to the array** — the version stays in `prerelease` **and** becomes
visible in `release`. Promotion is additive, not a move.

**Which is why `release` is `organization`-visible and `prerelease` is `private`** (lines 255, 263): the
same artifact, two audiences, one label apart.

</details>

---

# Section G — Case study

## Case study: Contoso package management

### Background

Contoso Ltd has **15 microservices sharing 4 internal libraries** — `auth-sdk`, `logging-sdk`,
`data-models` and `api-contracts`. Teams currently **copy source code between repositories**. The VP of
Engineering wants a centralised solution with **private package hosting**, **vulnerability scanning on
dependencies**, **access control per team**, and **support for npm and NuGet**.

### Requirements

**Hosting**

- Internal libraries must be published once and consumed by version, not copied
- Both npm and NuGet must be served
- Public packages must reach builds through a single, auditable path

**Access**

- Teams that only consume must not be able to publish
- Consuming teams must still be able to pull new public packages
- Only tested versions of the internal libraries may reach consuming services

**Operations**

- A public registry outage must not stop builds
- A public package must not be able to shadow an internal one
- Dependency vulnerabilities must be detected

---

## Q42

Which platform best fits the requirements, and what is the deciding factor?

- A. Azure Artifacts — one feed, upstreams, views for gating
- B. GitHub Packages — it is the simpler of the two platforms
- C. Both platforms, split by language ecosystem
- D. Neither; a third-party registry for both ecosystems

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-13.md`:** lines **150**, **167–187**, **247–274**, **242–245**.

**Three requirements point the same way.** npm **and** NuGet from one feed; a **single auditable path**
for public packages, which is what upstreams provide; and **only tested versions** reaching consumers,
which is what views provide and GitHub Packages cannot (Q20).

**Why B is defensible on the hosting requirement alone** and fails the other two. GitHub Packages hosts
both ecosystems perfectly well — it has no upstream proxying and no views.

**Why C is the answer that doubles the operational surface**: two registries to configure, two credential
types, two audit trails, for no capability gained.

</details>

---

## Q43

How should public packages reach builds through "a single, auditable path"?

- A. Let each project use npmjs and nuget.org directly
- B. Mirror the public registries nightly into the feed
- C. Upstreams on the feed; every project points at it
- D. Vendor every dependency into each repository

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-13.md`:** lines **167–187** and **195–197**.

**One registry URL in every configuration, and every public package arrives cached and recorded.** That
is the audit trail — the feed's contents *are* the list of what the organisation consumes.

**Why B is real work that solves less.** A nightly mirror copies packages nobody uses and lags behind
ones they need; an upstream fetches **on demand** and caches exactly what was asked for.

**And D is the practice being replaced**, in a new form — vendoring is copy-paste with a build step.

</details>

---

## Q44

Which **two** satisfy "consumers cannot publish, but can still pull new public packages"? (Choose two.)

- A. Grant consuming teams the Reader role
- B. Grant consuming teams the Collaborator role
- C. Grant every team the Contributor role
- D. Grant publishing teams the Contributor role
- E. Grant consuming teams the Owner role

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-13.md`:** lines **242–245**.

**Collaborator is the exact role the requirement describes** — consume, plus save from upstream, without
publishing.

**Why A fails the second half, and this is the subtle part.** A Reader can install what the feed already
holds. The **first** person to request a new public package triggers a save into the feed, and Reader
cannot do that — so their `npm install` of a new dependency fails with a permissions error that reads
like an outage (Q10).

</details>

---

## Q45

How is "only tested versions may reach consuming services" enforced?

- A. Publish only builds that have already been tested
- B. Publish every build; promote to `release` when tested
- C. Delete untested versions from the feed nightly
- D. Use a naming convention to mark tested versions

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-13.md`:** lines **249–274**.

**Publish freely, expose deliberately.** The `prerelease` view is private and holds everything; `release`
is organisation-visible and holds only promoted versions.

**Why A sounds equivalent and is not.** If only tested builds are published, the library team has nowhere
to share a candidate with an early adopter — so they either publish it anyway or share a tarball, which
is copy-paste again.

**Why C is destructive and unnecessary.** Promotion is additive (Q41); nothing needs deleting.

</details>

---

## Q46

Which **two** protect builds operationally? (Choose two.)

- A. The upstream cache, so an outage does not stop builds
- B. Pinning every dependency to an exact version
- C. A nightly full mirror of the public registries
- D. Internal upstream sources listed before public ones
- E. Vendoring dependencies into each repository

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-13.md`:** lines **285** and **290**.

**A is the availability requirement; D is the integrity one** — the public registry being down, and the
public registry being hostile.

**D is dependency confusion in one line of configuration.** If `@contoso/auth-sdk` also exists on npmjs
and public is checked first, your build installs a stranger's package under your own name.

**Why B is good practice that addresses neither.** A pinned version still has to be **fetched** from
somewhere, and a pin does not tell you which registry answered.

</details>

---

## Q47

Ten months in, a build for `payment-service` fails with `404 Not Found` for `@contoso/data-models@2.1.0`.
The feed holds the package, other services resolve it fine, and the version exists. The payment team was
recently moved into a new Azure DevOps project.

What is the most likely cause?

- A. The version was unpublished from the feed
- B. The upstream source was removed from the feed config
- C. The team's personal access token expired
- D. The feed is project-scoped; the new project cannot see it

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-13.md`:** lines **155–159**.

```bash
az artifacts feed create \
  --name contoso-packages \
  --project ContosoServices \
  --scope project
```

**"Other services resolve it fine" localises it to that team**, and "moved into a new project" names what
changed.

**A project-scoped feed inherits the project's permissions** (Q14), so a team in a different project has
no path to it — and Azure Artifacts returns **404 rather than 403**, because it does not confirm the
existence of feeds you cannot see. **That is why the error reads like a missing package.**

**Why C would give 401**, and why A and B would break **every** consumer, not one team.

**The fix depends on intent.** If the libraries are genuinely organisation-wide — and four libraries
shared by 15 services are — the feed should be **organisation-scoped**. If the scoping is deliberate,
grant the new project's team the Collaborator role explicitly.

**The durable lesson: feed scope is a decision about who the consumers are**, and "15 services across an
unknown number of projects" answers it.

</details>

---

## Q48

A year on, no team copies source between repositories, a public registry outage passes unnoticed, and
consuming services only ever see promoted library versions.

Which explanation best accounts for the change?

- A. The libraries became a versioned product, not a folder
- B. Teams were told to stop copying source between repos
- C. A code review rule banned duplicated files
- D. The libraries were merged into a single mono-repo

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-13.md`:** lines **150–159**, **167–187**, **242–245**, **249–274**, **283–290**.

**Start with why teams were copying** (line 21). Copying is what people do when there is **no supported
way to depend on someone else's code**. It is not laziness; it is the only mechanism available.

**A feed replaces the mechanism**, and everything else in the challenge exists to make that replacement
survivable.

*Upstreams* mean the feed is the **only** registry anyone configures — one URL, one credential, one audit
trail, and a cache that makes a public outage invisible.

*Priority order* means an internal package name cannot be claimed by a stranger.

*Views* mean a library author can publish a candidate without every consumer picking it up — the thing
that would otherwise push them back to sharing tarballs.

*Roles* mean a consuming team can pull a new public dependency without being able to publish into the
shared namespace.

**What actually changed is that the four libraries became versioned artifacts with a lifecycle**, rather
than directories. Copy-paste has no version, no changelog and no way to know who is running what;
`@contoso/auth-sdk@2.1.0` has all three.

**The graded idea: package management questions are asking how *consumers* get code, not where it is
stored.** The right answer is the one where consuming is easier than copying — and where what they
consume can be scanned, audited and promoted.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Scope not matching the repository owner** | Q5, Q25, Q34, Q36 | GitHub Packages derives ownership from the scope. 403 |
| **`packages: write` omitted from a workflow** | Q6, Q22, Q34, Q39 | Defaults to read on fork-triggered runs |
| **Bare `registry=` with GitHub Packages** | Q8, Q25, Q37 | Sends `lodash` there too. Use `@scope:registry` |
| **`always-auth` omitted on a private feed** | Q9, Q30, Q34 | Publish works, install 401s |
| **`NPM_TOKEN` instead of `NODE_AUTH_TOKEN`** | Q39 | `setup-node` writes an `.npmrc` expecting the latter |
| **Reader granted to a consuming team** | Q10, Q32, Q44 | Cannot save from upstream. First public install fails |
| **Public upstream listed before internal** | Q13, Q28, Q46 | Dependency confusion in one array |
| **Publishing straight into the `release` view** | Q26, Q45 | Promotion *is* the control |
| **Views assumed to exist in GitHub Packages** | Q20, Q27 | They do not |
| **pip assumed supported by GitHub Packages** | Q1, Q29 | npm, Maven, NuGet, Docker, RubyGems |
| **`--api-key az` read as a real key** | Q15, Q30 | Placeholder; the PAT on the source is the credential |
| **Project-scoped feed for organisation-wide libraries** | Q14, Q47 | Other projects get 404, not 403 |
| **Internal libraries assumed not to need scanning** | Q25 | The risk is in their transitive dependencies |
| **`upstreamsources` and `views` body shapes mixed** | Q40 | Different endpoints, different fields |

---

# What to memorise

**In `challenge-13.md`:** lines **42–64**, **155–187**, **241–274**, **283–309**.

```text
THREE IDEAS
  FEED      where packages live. one feed serves npm AND NuGet (and more)
  VIEW      which QUALITY of package a consumer sees. @prerelease / @release
            promotion = adding an EXISTING version to a view. GitHub Packages has NO equivalent
  UPSTREAM  where the feed goes when it does not have something
            1st request -> FETCH + CACHE locally    later -> served from cache
            upstream offline -> cached packages still work
            PRIORITY ORDER: internal FIRST, public second  <- dependency-confusion defence

AZURE ARTIFACTS FEED ROLES
  Reader        consume what the feed ALREADY holds
  Collaborator  consume + SAVE FROM UPSTREAM      <- the practical minimum for a consuming team
  Contributor   publish new packages and versions
  Owner         everything, including deletion and permissions
```

```ini
# GitHub Packages - routes BY SCOPE                (lines 63-64)
@contoso:registry=https://npm.pkg.github.com          # only @contoso goes here
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}      # authentication, per host
# package.json:  "name": "@contoso/auth-sdk"          <- SCOPE MUST MATCH THE REPO OWNER (else 403)
#                "publishConfig": {"registry": "https://npm.pkg.github.com"}
# visibility:  private (repo collaborators) | internal (WHOLE ORG) | public
# supports: npm, Maven, NuGet, Docker, RubyGems.   NOT pip.

# Azure Artifacts - routes EVERYTHING through the feed   (lines 195-197)
registry=https://pkgs.dev.azure.com/ORG/PROJ/_packaging/FEED/npm/registry/
always-auth=true                                      # or reads 401 while publish works
//pkgs.dev.azure.com/.../npm/registry/:_authToken=${AZURE_DEVOPS_PAT}
```

```yaml
# Publishing from Actions                           (lines 135-147)
permissions:
  contents: read          # least privilege - a publish job never writes the repo
  packages: write         # WITHOUT THIS: 403 (default is read on fork-triggered runs)
steps:
  - uses: actions/setup-node@v4
    with: {node-version: 20, registry-url: https://npm.pkg.github.com/}   # writes the runner .npmrc
  - run: npm publish
    env: {NODE_AUTH_TOKEN: "${{ secrets.GITHUB_TOKEN }}"}   # NOT NPM_TOKEN
```

```bash
# Azure Artifacts                                   (lines 155-274)
az artifacts feed create --name contoso-packages --project ContosoServices --scope project
#   --scope project  = project permissions.   organisation-scoped = shared across projects
#   wrong scope -> other projects get 404 (not 403) - it hides feeds you cannot see

# upstream source
POST .../feeds/FEED/upstreamsources
  {"name":"npmjs","protocol":"npm","location":"https://registry.npmjs.org/",
   "upstreamSourceType":"public"}          # or "internal" -> another ADO feed

# views
POST .../feeds/FEED/views
  {"name":"prerelease","type":"implicit","visibility":"private"}
  {"name":"release","type":"implicit","visibility":"organization"}
# promote an EXISTING version (additive - nothing is rebuilt or moved)
POST .../feeds/FEED/npm/@contoso/auth-sdk/versions/1.0.0
  {"views":{"op":"add","path":"/views/-","value":"release"}}

# NuGet
dotnet nuget add source "<feed>/nuget/v3/index.json" --name contoso-packages \
  --username contoso --password $PAT --store-password-in-clear-text
dotnet nuget push ./nupkgs/*.nupkg --source contoso-packages --api-key az    # 'az' is a PLACEHOLDER
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 14 |
| 38–43 | Re-read the trap index and the three ideas, then move on |
| 30–37 | Write the four feed roles and the upstream behaviour from memory, then retake |
| Below 30 | Redo Tasks 1, 2 and 3 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 13.

:::danger The three ideas

**Feed** — where packages live. **View** — which quality of package a consumer sees, promoted
deliberately. **Upstream** — one URL for everything, cached on first use, with **internal listed first**
so an internal name cannot be shadowed.

Two authentication facts decide the rest: GitHub Packages needs the **scope to match the repository
owner**, and a publishing workflow needs **`packages: write`**.

Contoso's teams were not copying code out of laziness. There was no supported way to depend on it.

:::
