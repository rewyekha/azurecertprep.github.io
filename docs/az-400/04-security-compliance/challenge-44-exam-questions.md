---
sidebar_position: 6.5
toc_max_heading_level: 2
title: "Challenge 44: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 44 — AZ-400 exam questions

**48 questions** built only from what Challenge 44 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-44.md`**.

:::danger Read this before you start

GitHub Advanced Security has **four scanners**, and every question here is really asking *which one*.

**CodeQL** — flaws in **code you wrote**: injection, path traversal, hard-coded URLs.
**Dependabot** — flaws in **code you imported**: vulnerable packages, including transitive ones.
**Secret scanning** — **credentials** in the repository, with **push protection** as the blocking half.
**Container scanning** — flaws in the **image**: base OS packages, installed binaries.

They do not overlap. A vulnerable npm package is invisible to CodeQL. An SQL injection is invisible to
Dependabot. A leaked token is invisible to both.

And one structural fact that answers several questions on its own: **the Security tab renders SARIF.**
Any scanner that emits SARIF can publish into it through `codeql-action/upload-sarif` — which is how
Trivy results sit next to CodeQL results.

The scenario at line 18 is what happens with none of it: a patch available for six months, an outage,
and nobody ever saw an alert.

:::

---

# Section A — Multiple choice

---

## Q1

Contoso has repositories in both GitHub and Azure DevOps and wants CodeQL on both. What is correct?

- A. CodeQL only works with GitHub repositories
- B. Use the GitHub CodeQL action for GitHub repos and `AdvancedSecurity-Codeql-*` tasks for Azure
  DevOps repos
- C. Mirror all Azure DevOps repos to GitHub for scanning
- D. Use a third-party tool, since CodeQL cannot run in Azure DevOps

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-44.md`:** lines **88–107** and **300–318**.

```yaml
  - task: AdvancedSecurity-Codeql-Init@1
    inputs:
      languages: 'csharp,javascript'
      querysuite: 'security-extended'
```

**GitHub Advanced Security for Azure DevOps — GHAzDO — is the same engine and the same query suites**,
delivered as pipeline tasks instead of an action.

**Why C is the answer teams actually build before they know GHAzDO exists.** Mirroring means two copies
of every repository, alerts raised against the mirror rather than where developers work, and no PR
annotations in Azure DevOps (line 326).

**Note the naming symmetry, because the exam tests it directly:** `init` → build → `analyze` on both
platforms, plus `AdvancedSecurity-Publish@1` in Azure DevOps.

</details>

---

## Q2

A Dependabot alert reports a critical vulnerability in a **transitive** dependency. The direct
dependency has no fix yet. What should Contoso do?

- A. Dismiss the alert — it is the upstream maintainer's problem
- B. Override the transitive version with npm `overrides` or yarn `resolutions`
- C. Remove the direct dependency entirely
- D. Wait for the direct dependency to publish an update

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-44.md`:** the scenario at line **18**.

**This is the exact failure that caused the outage.** A patch existed for six months in a transitive
dependency, and nobody acted because the direct dependency had not moved.

**`overrides` forces the resolved version of a nested package** without waiting for anyone upstream —
immediate mitigation while functionality is preserved.

**Why D is the trap, and it is a trap because it is what most teams do.** "Waiting" is a decision with
a duration, and here that duration was six months of known exposure.

**Why A is worse than waiting** — dismissing removes the alert, so the next person sees a clean
dashboard over a live vulnerability. **Dismissal is for "not exploitable here"**, with a reason
recorded (lines 205–208).

</details>

---

## Q3

Which secret scanning feature prevents secrets **entering** the repository?

- A. Secret scanning alerts
- B. Custom secret patterns
- C. Push protection
- D. Security advisories

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-44.md`:** lines **36–37**.

```json
    "secret_scanning": { "status": "enabled" },
    "secret_scanning_push_protection": { "status": "enabled" }
```

**Two settings, two roles.** `secret_scanning` is **detective** — it alerts on what is already
committed. `secret_scanning_push_protection` is **preventive** — the push is rejected at the server.

**Why B is a modifier rather than a mechanism.** A custom pattern (line 125) extends *what* both
features look for. It does not change whether they alert or block.

</details>

---

## Q4

Contoso wants container scan results beside CodeQL results in the Security tab. How?

- A. Use a scanner that emits SARIF and upload with `codeql-action/upload-sarif`
- B. Enable Dependabot for the Docker ecosystem only
- C. Run `docker scan` and parse the text output
- D. Container results cannot appear in the Security tab

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-44.md`:** lines **400–412**.

```yaml
      - name: Run Trivy vulnerability scanner
        with:
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy scan results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'
          category: 'container-scanning'
```

**SARIF is the contract.** The Security tab does not know what Trivy is; it knows how to render SARIF,
so any scanner that emits it becomes a first-class citizen.

**`category:` is what keeps the two result sets separate** in the UI — without it, one upload can
appear to supersede another.

**Why B is a real and different control.** Dependabot's docker ecosystem (lines 178–183) watches the
**base image tag in your Dockerfile**. Trivy scans **the built image**, including every OS package
inside it. Both are worth having, and they see different things.

</details>

---

## Q5

CodeQL completes but reports zero results and warns that no source code was found. The language is
C#. Why?

- A. The repository has no C# files
- B. CodeQL must observe the build for compiled languages, and the build step is missing or failed
- C. `security-events: write` is missing
- D. The matrix language value is misspelled

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-44.md`:** lines **427–429** and **95–99**.

```yaml
      # For compiled languages, the build step is required
      - name: Build application (for compiled languages)
        if: matrix.language == 'csharp'
        run: |
          dotnet build src/Contoso.Web/Contoso.Web.csproj
```

**CodeQL builds its database by watching the compiler.** No compilation, no database — and the run
still succeeds, which is what makes this dangerous. A green tick over an empty analysis.

**And the build must sit between `init` and `analyze`** (lines 438–448). Running it before `init` means
CodeQL was not yet instrumenting anything.

**Why C would fail differently** — missing `security-events: write` fails at the *upload* step with a
permissions error, after the analysis produced results.

</details>

---

## Q6

Which languages need an explicit build step, and which use autobuild?

- A. All languages need a build step
- B. Compiled languages such as C# and Java need a build; interpreted languages such as JavaScript and
  Python use autobuild
- C. Only Java needs a build
- D. Autobuild works for every language

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-44.md`:** lines **95–104**.

```yaml
      # For interpreted languages, autobuild handles it
      - name: Autobuild
        if: matrix.language == 'javascript'
        uses: github/codeql-action/autobuild@v3
```

**Interpreted languages have no compile step to observe**, so CodeQL parses the source directly.

**A nuance worth carrying in.** `autobuild` will *attempt* compiled languages too, by guessing the
build system — and it silently produces an empty or partial database when it guesses wrong. That is why
the challenge writes the explicit `dotnet build` rather than trusting it.

</details>

---

## Q7

What does `queries: +security-extended,security-and-quality` do?

- A. Replaces the default query suite with these two
- B. Adds these suites **to** the default suite
- C. Runs only queries tagged `security`
- D. Enables custom queries from the repository

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-44.md`:** lines **92–93**.

```yaml
          queries: +security-extended,security-and-quality
```

**The leading `+` means append.** Without it, the listed suites **replace** the default set.

**That single character is the question.** `queries: security-extended` — no plus — narrows your
coverage while looking like it broadens it, because the default suite's high-precision queries are
dropped.

**And the same `+` appears at line 359** for custom queries: `queries: +./.github/codeql/queries/`
adds the Contoso query pack alongside everything else.

</details>

---

## Q8

Why does the CodeQL workflow include a `schedule:` trigger?

- A. To catch newly published CodeQL queries and advisories against unchanged code
- B. Because push triggers are unreliable
- C. To reduce Actions minutes
- D. To satisfy branch protection

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-44.md`:** lines **66–67**.

```yaml
  schedule:
    - cron: '30 6 * * 1'  # Weekly Monday 6:30 UTC
```

**Code that has not changed can still become vulnerable**, because the *queries* improve and new
advisories appear. A repository with no commits for three months would otherwise never be re-analysed.

**Monday 06:30 UTC is deliberate** — before the working week, so findings are waiting rather than
arriving mid-sprint.

</details>

---

## Q9

Which permission does the CodeQL job need to publish results?

- A. `contents: write`
- B. `security-events: write`
- C. `actions: write`
- D. `packages: write`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-44.md`:** lines **73–76**.

```yaml
    permissions:
      actions: read
      contents: read
      security-events: write
```

**Read the whole block, because each line has a job.** `security-events: write` publishes alerts,
`contents: read` reads the code, and `actions: read` lets CodeQL inspect workflow runs.

**Note that `contents` is `read`, not `write`.** A scanner has no business modifying the repository —
and the same block appears at lines 388–391 for the container scan.

</details>

---

## Q10

What does `open-pull-requests-limit: 10` control?

- A. The total number of Dependabot PRs allowed in the repository's lifetime
- B. How many Dependabot PRs may be open at once for that ecosystem
- C. How many commits each PR may contain
- D. How many reviewers are assigned

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-44.md`:** line **153**.

**It is the throttle that keeps Dependabot usable.** A monorepo with 300 npm packages would otherwise
open dozens of PRs the first Monday and the team would mute the bot.

**Which is what `groups:` at lines 160–166 is really for** — bundling all minor and patch updates into
one PR instead of forty:

```yaml
    groups:
      production-dependencies:
        patterns:
          - "*"
        update-types:
          - "minor"
          - "patch"
```

**Note the limit is per ecosystem entry**, so npm's 10 and NuGet's 5 (line 173) are independent.

</details>

---

## Q11

What is the difference between Dependabot **alerts** and **version updates**?

- A. Alerts fire on known vulnerabilities; version updates keep dependencies current regardless of
  vulnerabilities
- B. They are the same feature under two names
- C. Alerts are for npm only
- D. Version updates only apply to GitHub Actions

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-44.md`:** line **213**.

```text
Dependabot version updates keep dependencies current regardless of known vulnerabilities
```

**Alerts are security; version updates are hygiene** — and hygiene is what makes security actionable.
A repository three major versions behind cannot take a security patch without a migration project,
which is how a six-month-old fix goes unapplied.

**They are enabled differently too.** Alerts come from `gh api .../vulnerability-alerts -X PUT` (line
198); version updates come from `.github/dependabot.yml`.

</details>

---

## Q12

Which Dependabot configuration prevents major version bumps of `express` while still allowing patches?

- A. `ignore: [{dependency-name: "express", update-types: ["version-update:semver-major"]}]`
- B. `allow: [{dependency-type: "production"}]`
- C. `open-pull-requests-limit: 0`
- D. `groups: {patch-updates: {patterns: ["express"]}}`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-44.md`:** lines **223–227**.

```yaml
    ignore:
      - dependency-name: "express"
        update-types: ["version-update:semver-major"]
```

**`ignore` is per-update-type, not per-package.** Ignoring the major bump still lets minor and patch
updates through, which is exactly what you want for a framework whose major versions require
migration work.

**Why C is the blunt version.** `open-pull-requests-limit: 0` disables the ecosystem entirely — no
patches either.

**Why B is a different axis.** `allow: dependency-type: production` restricts to production
dependencies, so dev-only packages are skipped regardless of version.

</details>

---

## Q13

Which condition ensures the auto-merge workflow only acts on Dependabot's own PRs?

- A. `if: github.actor == 'dependabot[bot]'`
- B. `if: github.event_name == 'pull_request'`
- C. `if: contains(github.head_ref, 'dependabot')`
- D. `if: github.repository_owner == 'contoso'`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-44.md`:** line **256**.

```yaml
    if: github.actor == 'dependabot[bot]'
```

**This guard is a security control, not a filter.** Without it, any contributor could open a PR that
the workflow would auto-merge — turning a convenience into a way to merge arbitrary code without review.

**Why C is the plausible-looking weak version.** A branch name is attacker-controlled: anyone can push
a branch called `dependabot/npm/evil` and satisfy it. `github.actor` is set by GitHub.

</details>

---

## Q14

Which update type does the auto-merge workflow merge?

- A. `version-update:semver-patch`
- B. `version-update:semver-minor`
- C. `version-update:semver-major`
- D. All security updates regardless of type

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-44.md`:** lines **264–265**.

```yaml
      - name: Auto-merge patch updates
        if: steps.metadata.outputs['update-type'] == 'version-update:semver-patch'
```

**Patch only, and the reasoning is semantic versioning.** A patch release promises no API change, so
the risk of auto-merging without human review is low. Minor releases add features, and major releases
break things.

**And `--auto` at line 268 matters** — it queues the merge *pending required checks*, rather than
merging immediately. CI still has to pass.

</details>

---

## Q15

Which Azure DevOps task publishes Advanced Security results?

- A. `AdvancedSecurity-Publish@1`
- B. `PublishTestResults@2`
- C. `AdvancedSecurity-Codeql-Analyze@1`
- D. `PublishBuildArtifacts@1`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-44.md`:** lines **314–319**.

```yaml
  - task: AdvancedSecurity-Codeql-Analyze@1
  - task: AdvancedSecurity-Publish@1
```

**Analyze produces the findings; Publish surfaces them** in the Repos > Advanced Security tab and as PR
annotations (lines 324–326).

**The whole GHAzDO sequence in order:** Dependency-Scanning, Codeql-Init, **build**, Codeql-Analyze,
Publish. The build sits in the middle for the same reason as in GitHub Actions (Q5).

</details>

---

## Q16

What does `exit-code: '1'` do on the second Trivy step?

- A. Fails the workflow when critical vulnerabilities are found
- B. Retries the scan once
- C. Suppresses output
- D. Uploads results to the Security tab

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-44.md`:** lines **414–420**.

```yaml
      - name: Fail on critical vulnerabilities
        with:
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL'
```

**Note that the workflow scans twice, on purpose, and the two runs do different jobs.** The first
(lines 400–406) emits SARIF for `CRITICAL,HIGH` so everything is **visible** in the Security tab. The
second emits a table and **fails the build** on `CRITICAL` only.

**That split is the design point: report broadly, block narrowly.** Failing on HIGH as well would block
most builds most days, and a gate that always fails gets bypassed.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** does GitHub Advanced Security provide? (Choose three.)

- A. CodeQL code scanning
- B. Secret scanning with push protection
- C. Dependency review and Dependabot alerts
- D. Container image signing
- E. Runtime intrusion detection
- F. Network firewall rules

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-44.md`:** lines **33–39**, **58–110**, **139–202**.

**Three scanners, three sources of risk: code you wrote, credentials you leaked, code you imported.**

**Why D, E and F are all real security capabilities that live elsewhere.** Signing is a supply-chain
control (Sigstore, Notary); intrusion detection is Defender for Cloud; firewalls are network
configuration. GHAS is **static analysis of a repository** — that framing answers most "which product"
questions.

</details>

---

## Q18

Which **three** are required for CodeQL to analyse a C# project? (Choose three.)

- A. `github/codeql-action/init@v3` with `languages: csharp`
- B. A build step between init and analyze
- C. `github/codeql-action/analyze@v3`
- D. `github/codeql-action/autobuild@v3`
- E. `packages: write` permission
- F. A published NuGet feed

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-44.md`:** lines **438–448**.

```yaml
- name: Initialize CodeQL
  uses: github/codeql-action/init@v3
  with:
    languages: csharp

# This step is required for compiled languages
- name: Build
  run: dotnet build src/Contoso.Web.sln

- name: Perform CodeQL Analysis
  uses: github/codeql-action/analyze@v3
```

**Init, build, analyze — and the order is the answer.** Init instruments the environment; the build
generates the database; analyze runs the queries against it.

**Why D is not *required*.** Autobuild is an alternative to the explicit build, and for compiled
languages it guesses. The challenge uses it only for JavaScript (line 103).

</details>

---

## Q19

Which **three** appear in the container scanning workflow? (Choose three.)

- A. Building the image locally before scanning
- B. Emitting SARIF and uploading it to the Security tab
- C. A second scan that fails the build on critical findings
- D. Pushing the image to a registry before scanning
- E. Signing the image
- F. Dependabot scanning the base image

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-44.md`:** lines **396–420**.

**Scanning the locally built image is the point** — a vulnerable image never reaches a registry where
someone could deploy it. Compare Challenge 38, which scans the filesystem before building for the same
reason.

**Why F is a real control that is not in this workflow.** Dependabot's docker ecosystem (line 178)
raises a PR when the **base image tag** has a newer version. Different mechanism, different trigger,
and complementary (Q4).

</details>

---

## Q20

Which **two** are true of push protection bypass? (Choose two.)

- A. It can be restricted to specific roles or teams
- B. A reason can be required when bypassing
- C. Any repository admin can always bypass
- D. Bypassing disables scanning for the repository
- E. Bypassing is impossible once enabled

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-44.md`:** lines **133–137**.

```text
2. Push protection > Who can bypass push protection for secret scanning:
   - Select "Specific roles or teams"
   - Add: Security team only
3. Require a reason when bypassing: Enable
```

**A path must exist, because false positives are real** — and a control with no escape valve gets
disabled wholesale, which is worse.

**The two settings together are what make it governable:** a short list of actors, and a recorded
reason for every use.

**Why E is the misconception that makes people resist enabling it**, and why D is the reverse — a
bypass is per-push, not a switch.

</details>

---

## Q21

Which **two** resolutions are valid for a secret scanning alert? (Choose two.)

- A. `false_positive`
- B. `revoked`
- C. `merged`
- D. `deferred`
- E. `archived`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-44.md`:** lines **119–122**.

```bash
gh api repos/contoso/webapp/secret-scanning/alerts/1 -X PATCH \
  --field state="resolved" \
  --field resolution="false_positive"
```

**`revoked` is the one that matters operationally**, and the honest one: the string was a real
credential, and it has been rotated. `false_positive` says it was never a credential at all.

**Closing a real leak as `false_positive` is the failure mode to avoid** — it clears the dashboard
while the credential stays live, which is the Challenge 43 lesson restated.

**Why C, D and E belong to pull requests and repositories**, not alerts.

</details>

---

## Q22

Which **two** are true about Dependabot alert dismissal? (Choose two.)

- A. A dismissal reason such as `not_used` must be supplied
- B. A comment can record the justification
- C. Dismissal patches the vulnerability
- D. Dismissed alerts are deleted permanently
- E. Only organisation owners can dismiss

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-44.md`:** lines **205–208**.

```bash
gh api repos/contoso/webapp/dependabot/alerts/5 -X PATCH \
  --field state="dismissed" \
  --field dismissed_reason="not_used" \
  --field dismissed_comment="This dependency is only in dev dependencies and not deployed"
```

**The reason and the comment are the audit trail.** Six months later, when someone asks why a critical
alert is closed, the answer is in the record rather than in somebody's memory.

**And the comment shown is a genuinely valid justification** — a dev-only dependency is not in the
deployed artifact, so the vulnerability is not reachable in production. **Dismissal is for
unreachability, not inconvenience.**

</details>

---

## Q23

Which **two** does `AdvancedSecurity-Dependency-Scanning@1` provide in Azure DevOps? (Choose two.)

- A. Detection of vulnerable dependencies in the repository
- B. Results surfaced in the Advanced Security tab and as PR annotations
- C. Automatic pull requests upgrading dependencies
- D. CodeQL analysis of source code
- E. Container image scanning

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-44.md`:** lines **296–297** and **322–326**.

```text
1. Repos > Advanced Security (tab) to view alerts
3. Alerts appear as PR annotations on pull requests
```

**PR annotations are the feature that fixes the scenario at line 18** — "developers never saw the
alert". A finding in a dashboard nobody opens is not visibility; a comment on the pull request is.

**Why C is the gap between the platforms, and the exam knows it.** GHAzDO **detects** vulnerable
dependencies; it does not open Dependabot-style upgrade PRs. That automation is GitHub-side.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must gain visibility into vulnerabilities across 45 repositories in code,
dependencies, secrets and container images, ensure developers actually see findings, and prevent a
repeat of a six-month-old unpatched transitive dependency.

---

## Q24

**Proposed solution:** Enable Advanced Security, secret scanning and push protection organisation-wide.
Add a CodeQL workflow on push, pull request and a weekly schedule, with a build step for compiled
languages and `+security-extended`. Configure `.github/dependabot.yml` for npm, NuGet, docker and
github-actions with grouped minor and patch updates, and enable Dependabot security updates. Add
container scanning emitting SARIF to the Security tab plus a second scan failing on critical. Restrict
push-protection bypass to the security team with a required reason.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-44.md`:** lines **42–48**, **58–110**, **141–192**, **198**, **396–420**, **133–137**.

| Risk | Scanner |
|---|---|
| Flaws in code you wrote | CodeQL, on PR and weekly |
| Flaws in code you imported | Dependabot alerts + version updates |
| Credentials in the repository | Secret scanning + push protection |
| Flaws in the image | Trivy → SARIF → Security tab |
| Developers never see alerts | PR-triggered scanning and PR annotations |
| Stale dependencies blocking patches | Grouped minor/patch version updates |

**The last row is what actually prevents the outage recurring.** Alerts alone told nobody anything for
six months; keeping dependencies current is what makes a security patch a small change rather than a
migration.

</details>

---

## Q25

**Proposed solution:** Enable secret scanning alerts only. Run CodeQL on a monthly schedule with
`queries: security-extended`. Enable Dependabot alerts but no version updates. Scan containers with
`docker scan` and read the text output in the log.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures, one per scanner.**

**Alerts without push protection means secrets still enter the repository** (Q3). Detection where
prevention was available.

**Monthly CodeQL, with no PR trigger, means a developer learns about an injection flaw up to four
weeks after writing it** — and `queries: security-extended` **without the `+`** replaces the default
suite instead of extending it (Q7), so coverage is narrower than the default.

**Alerts without version updates is exactly the scenario at line 18.** The alert existed; the upgrade
path did not.

**Text output in a log is not a security dashboard.** No SARIF means nothing in the Security tab,
nothing tracked over time, no severity filtering, and no way for the security team to see across 45
repositories.

</details>

---

## Q26

**Proposed solution:** Enable Advanced Security, secret scanning and push protection organisation-wide.
Add CodeQL on push, pull request and weekly, with `+security-extended` and a build step for C#.
Configure Dependabot for all four ecosystems with grouped updates. Add container scanning with SARIF
upload and a critical-severity gate. Allow any organisation member to bypass push protection, so
security scanning never blocks a release.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**Unrestricted bypass turns the only preventive control into an advisory one.** Push protection's
entire value is that it cannot be skipped; make it skippable by anyone and it becomes a dialog people
click through under deadline pressure.

**The stated reasoning is also wrong on the facts.** Push protection blocks **pushes containing
secrets** — it does not block releases. The scenario it prevents is a credential entering history, which
is what Challenge 43's incident cost three days of exposure.

**Line 134's setting is the correct shape of the same intent:** a named team, and a required reason. The
release is never blocked for long; it is blocked until someone accountable says why.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — CodeQL

| # | Statement | Answer |
|---|---|---|
| 1 | Compiled languages need a build between init and analyze |  |
| 2 | A missing build causes the workflow to fail loudly |  |
| 3 | `queries: +security-extended` extends the default suite |  |
| 4 | CodeQL detects vulnerable npm packages |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Compiled languages need a build between init and analyze | **Yes** |
| 2 | A missing build causes the workflow to fail loudly | **No** |
| 3 | `queries: +security-extended` extends the default suite | **Yes** |
| 4 | CodeQL detects vulnerable npm packages | **No** |

**In `challenge-44.md`:** lines **429**, **427**, **92**, **139**.

Row 2 is why this is a Break scenario: **zero results, a green tick, and a warning nobody reads**.

Row 4 is the boundary between scanners. **CodeQL analyses code you wrote; Dependabot analyses code you
imported.**

</details>

---

## Q28 — Dependabot

| # | Statement | Answer |
|---|---|---|
| 1 | Alerts and version updates are separate features |  |
| 2 | `groups:` reduces the number of pull requests |  |
| 3 | `ignore` with `semver-major` still allows patch updates |  |
| 4 | Dependabot fixes transitive dependencies without upstream action |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Alerts and version updates are separate features | **Yes** |
| 2 | `groups:` reduces the number of pull requests | **Yes** |
| 3 | `ignore` with `semver-major` still allows patch updates | **Yes** |
| 4 | Dependabot fixes transitive dependencies without upstream action | **No** |

**In `challenge-44.md`:** lines **213**, **160–166**, **223–227**, **18**.

Row 4 is Q2. Dependabot **reports** the transitive vulnerability; forcing a patched nested version is
your package manager's job — `overrides` or `resolutions`.

</details>

---

## Q29 — secret scanning

| # | Statement | Answer |
|---|---|---|
| 1 | Push protection blocks the push at the server |  |
| 2 | Custom patterns can be defined organisation-wide |  |
| 3 | Alerts alone prevent secrets entering the repository |  |
| 4 | Bypass can be restricted to specific teams with a required reason |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Push protection blocks the push at the server | **Yes** |
| 2 | Custom patterns can be defined organisation-wide | **Yes** |
| 3 | Alerts alone prevent secrets entering the repository | **No** |
| 4 | Bypass can be restricted to specific teams with a required reason | **Yes** |

**In `challenge-44.md`:** lines **37**, **125–128**, **36**, **133–137**.

Row 2 is what covers Contoso's own key format, which no vendor pattern knows:

```bash
  --field pattern="contoso_[a-zA-Z0-9]{32}"
```

Row 3 is the detective-versus-preventive distinction the exam returns to in every security challenge.

</details>

---

## Q30 — platforms and publishing

| # | Statement | Answer |
|---|---|---|
| 1 | CodeQL runs in Azure DevOps through Advanced Security tasks |  |
| 2 | The Security tab renders SARIF from any scanner |  |
| 3 | GHAzDO opens Dependabot-style upgrade pull requests |  |
| 4 | Advanced Security findings appear as PR annotations in Azure DevOps |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | CodeQL runs in Azure DevOps through Advanced Security tasks | **Yes** |
| 2 | The Security tab renders SARIF from any scanner | **Yes** |
| 3 | GHAzDO opens Dependabot-style upgrade pull requests | **No** |
| 4 | Advanced Security findings appear as PR annotations in Azure DevOps | **Yes** |

**In `challenge-44.md`:** lines **300–318**, **408–412**, **296**, **326**.

Row 3 is the one genuine capability gap between the two platforms, and it is exactly the kind of
detail the exam uses to separate people who read the docs from people who ran the lab.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each finding to the scanner that detects it.

| Finding | Scanner |
|---|---|
| SQL injection in a controller |  |
| A vulnerable version of `lodash` |  |
| An AWS key committed in `config.js` |  |
| A CVE in the base image's OpenSSL |  |
| A hard-coded production API URL |  |
| A stale base image tag in the Dockerfile |  |

**Options:** CodeQL · Container scanning (Trivy) · Custom CodeQL query · Dependabot · Dependabot, docker ecosystem · Secret scanning

<details>
<summary>Show answer</summary>

| Finding | Scanner |
|---|---|
| SQL injection in a controller | **CodeQL** |
| A vulnerable version of `lodash` | **Dependabot** |
| An AWS key committed in `config.js` | **Secret scanning** |
| A CVE in the base image's OpenSSL | **Container scanning (Trivy)** |
| A hard-coded production API URL | **Custom CodeQL query** |
| A stale base image tag in the Dockerfile | **Dependabot, docker ecosystem** |

**In `challenge-44.md`:** lines **58–110**, **139–202**, **112–128**, **400–406**, **332–349**,
**178–183**.

**The last two rows are the pair that catches people.** A CVE **inside** the built image is Trivy's; a
**newer tag available** for the base image is Dependabot's. One inspects the artifact, the other reads
your Dockerfile.

</details>

---

## Q32

Match each configuration value to its effect.

| Value | Effect |
|---|---|
| `queries: +security-extended` |  |
| `open-pull-requests-limit: 10` |  |
| `ignore: version-update:semver-major` |  |
| `exit-code: '1'` |  |
| `format: 'sarif'` |  |
| `category: 'container-scanning'` |  |

**Options:** Adds to the default query suite · Blocks major bumps, allows minor and patch · Caps concurrent Dependabot PRs for that ecosystem · Fails the build on matching findings · Keeps result sets separate in the Security tab · Produces output the Security tab can render

<details>
<summary>Show answer</summary>

| Value | Effect |
|---|---|
| `queries: +security-extended` | **Adds to the default query suite** |
| `open-pull-requests-limit: 10` | **Caps concurrent Dependabot PRs for that ecosystem** |
| `ignore: version-update:semver-major` | **Blocks major bumps, allows minor and patch** |
| `exit-code: '1'` | **Fails the build on matching findings** |
| `format: 'sarif'` | **Produces output the Security tab can render** |
| `category: 'container-scanning'` | **Keeps result sets separate in the Security tab** |

**In `challenge-44.md`:** lines **92**, **153**, **224–225**, **419**, **404**, **412**.

</details>

---

## Q33

Arrange the CodeQL steps for a C# repository.

**Items:** Perform CodeQL Analysis · Checkout repository · Build the project · Initialize CodeQL

<details>
<summary>Show answer</summary>

### Answer

1. Checkout repository — line **86**
2. Initialize CodeQL — line **88**
3. Build the project — line **99**
4. Perform CodeQL Analysis — line **107**

**Steps 2 and 3 are the pair the exam inverts.** Init must run **before** the build, because init is
what instruments the environment so CodeQL can watch the compiler. Build first and CodeQL observes
nothing — Break scenario 1, with a green tick and zero results.

**For JavaScript, step 3 becomes `autobuild`** (line 104) or can be omitted entirely, since there is
nothing to compile.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| CodeQL reports zero results for C# |  |
| CodeQL coverage narrower than expected |  |
| CodeQL fails when uploading results |  |
| Trivy findings never reach the Security tab |  |
| Dependabot opens 40 PRs on Monday |  |
| A critical alert stays open for months |  |

**Options:** Alerts enabled, version updates not · No build step, or the build failed · No `groups:` and a high PR limit · Output not SARIF, or not uploaded · `queries:` without the leading `+` · `security-events: write` missing

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| CodeQL reports zero results for C# | **No build step, or the build failed** |
| CodeQL coverage narrower than expected | **`queries:` without the leading `+`** |
| CodeQL fails when uploading results | **`security-events: write` missing** |
| Trivy findings never reach the Security tab | **Output not SARIF, or not uploaded** |
| Dependabot opens 40 PRs on Monday | **No `groups:` and a high PR limit** |
| A critical alert stays open for months | **Alerts enabled, version updates not** |

**In `challenge-44.md`:** lines **429**, **92**, **76**, **404–411**, **160–166**, **213**.

**The first two share a property that makes them dangerous: neither fails.** The workflow is green in
both cases, and the coverage gap is invisible unless you compare finding counts before and after.

</details>

---

## Q35

Match each control to whether it prevents or detects.

| Control | Type |
|---|---|
| Push protection |  |
| Secret scanning alerts |  |
| CodeQL on pull request |  |
| Trivy with `exit-code: '1'` |  |
| Dependabot alerts |  |
| Dependabot version updates |  |

**Options:** Detects · Detects, before merge · Prevents · Prevents — blocks the build · Prevents — keeps patching possible

<details>
<summary>Show answer</summary>

| Control | Type |
|---|---|
| Push protection | **Prevents** |
| Secret scanning alerts | **Detects** |
| CodeQL on pull request | **Detects, before merge** |
| Trivy with `exit-code: '1'` | **Prevents — blocks the build** |
| Dependabot alerts | **Detects** |
| Dependabot version updates | **Prevents — keeps patching possible** |

**In `challenge-44.md`:** lines **37**, **36**, **64–65**, **419**, **198**, **213**.

**"CodeQL on pull request" is deliberately the middle case.** It detects rather than blocks — unless
the check is required by branch protection, at which point the detective control becomes a gate. **A
scanner becomes preventive when something refuses to proceed on its result.**

</details>

---

# Section F — Hot area

---

## Q36

```yaml
    permissions:
      actions: [BLANK 1]
      contents: [BLANK 2]
      [BLANK 3]: write
```

Requirement: CodeQL analyses the repository and publishes alerts, with least privilege.

- **BLANK 1:** `read` / `write` / `none`
- **BLANK 2:** `read` / `write` / `none`
- **BLANK 3:** `security-events` / `issues` / `checks` / `packages`

<details>
<summary>Show answer</summary>

### Answer: `read`, `read`, `security-events`

**In `challenge-44.md`:** lines **73–76**.

**Only one scope needs write, and it is the one that publishes findings.** A scanner that could write
repository contents could modify the code it is scanning.

**`actions: read` is the least obvious of the three** — CodeQL reads workflow run metadata to correlate
analyses across runs.

</details>

---

## Q37

```yaml
      - name: Initialize CodeQL
        uses: github/codeql-action/[BLANK 1]@v3
        with:
          languages: ${{ matrix.language }}
          queries: [BLANK 2]security-extended,security-and-quality
```

- **BLANK 1:** `init` / `analyze` / `autobuild` / `upload-sarif`
- **BLANK 2:** `+` / `-` / `*` / *(nothing)*

<details>
<summary>Show answer</summary>

### Answer: `init`, `+`

**In `challenge-44.md`:** lines **88–92**.

**One character decides your coverage.** With `+`, these suites are added to the default. Without it,
they replace it — and the default suite contains the high-precision queries that produce the fewest
false positives.

**This is a favourite exam question precisely because both forms are valid YAML** and both run
successfully. Only the finding count differs, and only if you were counting.

</details>

---

## Q38

```yaml
  - package-ecosystem: "[BLANK 1]"
    directory: "/"
    schedule:
      interval: "weekly"
    [BLANK 2]: 10
    groups:
      production-dependencies:
        patterns: ["*"]
        [BLANK 3]: ["minor", "patch"]
```

- **BLANK 1:** `npm` / `nodejs` / `javascript` / `package.json`
- **BLANK 2:** `open-pull-requests-limit` / `max-prs` / `pr-limit` / `concurrency`
- **BLANK 3:** `update-types` / `versions` / `semver` / `levels`

<details>
<summary>Show answer</summary>

### Answer: `npm`, `open-pull-requests-limit`, `update-types`

**In `challenge-44.md`:** lines **146–166**.

**`package-ecosystem` names the *package manager*, not the language** — which is why `npm` is right and
`javascript` is not. The four in this challenge are `npm`, `nuget`, `docker` and `github-actions`.

**And `github-actions` as an ecosystem is worth remembering** (line 186): it keeps your own workflow
action versions patched, which is a supply-chain surface most teams forget entirely.

</details>

---

## Q39

```yaml
jobs:
  auto-merge:
    if: github.[BLANK 1] == 'dependabot[bot]'
    steps:
      - uses: dependabot/fetch-metadata@v2
        id: metadata
      - if: steps.metadata.outputs['update-type'] == '[BLANK 2]'
        run: gh pr merge "${{ github.event.pull_request.number }}" [BLANK 3] --squash
```

- **BLANK 1:** `actor` / `author` / `triggering_user` / `head_ref`
- **BLANK 2:** `version-update:semver-patch` / `version-update:semver-major` /
  `security-update` / `patch`
- **BLANK 3:** `--auto` / `--admin` / `--force` / `--now`

<details>
<summary>Show answer</summary>

### Answer: `actor`, `version-update:semver-patch`, `--auto`

**In `challenge-44.md`:** lines **256–268**.

**`--auto` is the safety-critical one.** It enables auto-merge *pending required checks*, so CI still
gates the merge. `--admin` would merge **immediately, bypassing branch protection** — an auto-merge
workflow with `--admin` is a way to merge unreviewed, untested code on a schedule.

**And `github.actor` rather than `head_ref`** because a branch name is attacker-controlled (Q13).

</details>

---

## Q40

```yaml
      - name: Run Trivy vulnerability scanner
        with:
          image-ref: 'contoso-webapp:${{ github.sha }}'
          format: '[BLANK 1]'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - uses: github/codeql-action/[BLANK 2]@v3
        with:
          sarif_file: 'trivy-results.sarif'
          [BLANK 3]: 'container-scanning'
```

- **BLANK 1:** `sarif` / `table` / `json` / `template`
- **BLANK 2:** `upload-sarif` / `analyze` / `init` / `publish`
- **BLANK 3:** `category` / `name` / `tool` / `label`

<details>
<summary>Show answer</summary>

### Answer: `sarif`, `upload-sarif`, `category`

**In `challenge-44.md`:** lines **400–412**.

**`category` separates this scanner's results from CodeQL's** in the Security tab. Upload two different
tools' results without distinct categories and GitHub treats the later upload as a **replacement** for
the earlier one — so your CodeQL alerts silently disappear.

**Note that the second Trivy step uses `format: 'table'`** (line 418) because it is not uploading
anything; it exists to print a readable list and fail the build.

</details>

---

## Q41

```yaml
  - task: [BLANK 1]
    inputs:
      languages: 'csharp,javascript'
      querysuite: '[BLANK 2]'

  - task: DotNetCoreCLI@2
    inputs:
      command: 'build'

  - task: AdvancedSecurity-Codeql-Analyze@1
  - task: [BLANK 3]
```

- **BLANK 1:** `AdvancedSecurity-Codeql-Init@1` / `CodeQL-Init@1` / `AdvancedSecurity-Publish@1` /
  `AdvancedSecurity-Dependency-Scanning@1`
- **BLANK 2:** `security-extended` / `+security-extended` / `default` / `all`
- **BLANK 3:** `AdvancedSecurity-Publish@1` / `PublishBuildArtifacts@1` / `PublishTestResults@2` /
  `AdvancedSecurity-Codeql-Init@1`

<details>
<summary>Show answer</summary>

### Answer: `AdvancedSecurity-Codeql-Init@1`, `security-extended`, `AdvancedSecurity-Publish@1`

**In `challenge-44.md`:** lines **300–319**.

**Note the difference from GitHub Actions and do not carry the habit across.** In the Azure DevOps
task, `querysuite` takes a plain suite name — there is **no leading `+`**. The `+` syntax belongs to the
`queries:` input of `codeql-action/init`.

**And the build sits between init and analyze here too** (lines 307–314), for the reason in Q5.

</details>

---

# Section G — Case study

## Case study: Contoso security scanning rollout

### Background

Contoso Ltd's security team has **no visibility into code vulnerabilities across 45 repositories**.
Last quarter, a **production outage** was caused by a known vulnerability in a **transitive dependency
that had a patch available for six months**. **Developers never saw the alert** because no scanning was
configured.

### Requirements

**Coverage**

- Vulnerabilities must be detected in application code, dependencies, secrets and container images
- Repositories exist in both GitHub and Azure DevOps
- Contoso's own internal API key format must be detected

**Visibility**

- Findings must reach developers where they work, not only a central dashboard
- All scanner results must be viewable in one place
- Code that has not changed must still be re-analysed as new advisories appear

**Prevention**

- Secrets must be blocked from entering repositories, with bypass restricted and recorded
- Critical container vulnerabilities must block the build
- Dependencies must stay current enough that a security patch is a small change

---

## Q42

How should Contoso scan code across both platforms?

- A. `github/codeql-action` in GitHub Actions and `AdvancedSecurity-Codeql-*` tasks in Azure Pipelines
- B. Mirror Azure DevOps repositories into GitHub and scan there
- C. A third-party SAST tool on both
- D. CodeQL in GitHub, and manual review for Azure DevOps

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-44.md`:** lines **88–107** and **300–319**.

**Same engine, same query suites, native to each platform** — and native matters because of the
visibility requirement: PR annotations appear where the developer is working (line 326).

**Why B fails the visibility requirement rather than the scanning one.** The scanning would work.
Alerts would land on a mirror nobody opens, and no annotation would appear on the Azure DevOps pull
request.

</details>

---

## Q43

How should the six-month-unpatched transitive dependency be prevented from recurring?

- A. Dependabot alerts **and** version updates, with grouped minor and patch PRs, plus package-manager
  overrides when the direct dependency lags
- B. Dependabot alerts only, reviewed monthly by the security team
- C. A quarterly manual dependency audit
- D. Pin all dependencies to fixed versions to avoid surprises

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-44.md`:** lines **141–166**, **198**, **213**, and the scenario at **18**.

**Three parts, because the failure had three causes.** Alerts create the signal; version updates keep
the codebase close enough to current that patching is trivial; overrides handle the case where the
direct dependency has not moved.

**Why B is what Contoso would have had, and it still fails.** An alert existed. It was the *acting* on
it that never happened — which is why grouped PRs matter more than the alert.

**Why D is the intuitive answer that causes the problem.** Pinning everything means nothing updates,
including security patches. Combined with no version updates, it is precisely how a project ends up
three majors behind and unable to take a fix.

</details>

---

## Q44

How should Contoso detect its own internal API key format?

- A. An organisation-level custom secret scanning pattern
- B. A custom CodeQL query
- C. A grep step in every workflow
- D. Rely on the default provider patterns

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-44.md`:** lines **125–128**.

```bash
gh api orgs/contoso/secret-scanning/custom-patterns -X POST \
  --field name="Contoso Internal API Key" \
  --field pattern="contoso_[a-zA-Z0-9]{32}" \
  --field scope="organization"
```

**Organisation scope covers all 45 repositories from one definition**, including repositories created
next year.

**Why B is the interesting near-miss.** A custom CodeQL query *could* find a matching string literal
(lines 332–349) — but it runs on a schedule and on push, it does not **block the push**, and it does
not appear in the secret scanning alert stream. **Custom patterns feed push protection; custom queries
do not.**

**Why D fails on definition** — provider patterns cover credentials whose format a vendor registered
with GitHub. Contoso's internal format is known only to Contoso.

</details>

---

## Q45

Which **two** ensure developers actually see findings? (Choose two.)

- A. CodeQL triggered on `pull_request`
- B. Advanced Security PR annotations in Azure DevOps
- C. A weekly email digest to the security team
- D. A central dashboard reviewed monthly
- E. Dismissing low-severity alerts to reduce noise

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-44.md`:** lines **64–65** and **326**.

**Both put the finding in the pull request** — the one place a developer is already looking, at the one
moment the code is still fresh.

**Why C and D describe the state that produced the outage.** "Developers never saw the alert" (line 18)
is not a claim that no alert existed anywhere; it is a claim about **where** it existed.

**Why E is the tempting operational answer.** Noise reduction is real and necessary — but dismissal is
for unreachable findings with a recorded reason (Q22), not for volume management. Grouping and
scoping reduce noise; dismissal hides it.

</details>

---

## Q46

How should container scanning be configured?

- A. Build the image, scan for `CRITICAL,HIGH` emitting SARIF into the Security tab, and scan again
  failing the build on `CRITICAL`
- B. Scan only after pushing to the registry
- C. Fail the build on any finding of any severity
- D. Scan weekly on a schedule instead of on push

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-44.md`:** lines **396–420**.

**Report broadly, block narrowly** — the split at lines 400–420 exists for a reason. Everything at HIGH
and above is **visible** and tracked; only CRITICAL **stops** the build.

**Why C is the design that gets disabled within a month.** A base image typically carries dozens of low
and medium findings with no available fix. A gate that fails every build teaches people to bypass it,
and then nothing is gated.

**Why B loses the point of the gate.** Once the image is in the registry it is deployable, and the
scan becomes a report about something already shipped.

</details>

---

## Q47

Eight months into the rollout, the security team notices that CodeQL alerts for `contoso/api` have
stopped appearing entirely, though the workflow runs green every day. The repository migrated from
JavaScript to TypeScript compiled through a custom build script six months ago.

What happened, and what is the fix?

- A. The build no longer produces a database CodeQL can observe — replace `autobuild` with the explicit
  build command between `init` and `analyze`
- B. `security-events: write` was removed
- C. The query suite expired
- D. GitHub Advanced Security was disabled for the repository

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-44.md`:** lines **95–104** and **427–429**.

**"Runs green, reports nothing" is the signature of Break scenario 1**, and it is the most dangerous
failure in this entire challenge because it looks exactly like success.

**What changed underneath is subtle.** `autobuild` handled the plain JavaScript fine — there was nothing
to compile, so CodeQL parsed the source. With a custom TypeScript build, autobuild guesses, guesses
wrong, and produces an empty or near-empty database. **No error, no failed step, no alerts.**

**Six months of silent non-coverage is worse than no scanning**, because the dashboard said the
repository was clean.

**The detection worth taking into practice:** watch for a **finding count that drops to zero and stays
there**. A healthy scanner produces some noise.

**Why B and D would both fail loudly** — a permissions error on upload, or the workflow refusing to run.

</details>

---

## Q48

A year after the rollout, the security team is asked whether Contoso would catch the original incident
today — a critical vulnerability in a transitive dependency with a patch available.

What is the honest answer, and what does this illustrate?

- A. Yes, and by four independent mechanisms: the Dependabot alert, the grouped version-update PR
  keeping the tree current, PR annotations putting it in front of developers, and the container scan
  catching it in the built image if it reached one
- B. Yes, because CodeQL analyses all dependencies
- C. No, because Dependabot cannot see transitive dependencies
- D. Only if the direct dependency publishes a fix

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-44.md`:** lines **198**, **160–166**, **326**, **400–406**, **18**.

**Walk the four in the order they would fire.**

The **Dependabot alert** raises the transitive CVE with its severity. The **grouped version-update PR**
means the dependency tree is weeks behind rather than years, so the fix is a version bump rather than a
migration. **PR annotations** put it in front of the developer who is already in that pull request,
which is the failure the original incident actually was. And if it still slipped through, the
**container scan** finds the vulnerable package in the built image and fails the build on CRITICAL.

**What it illustrates: the original outage was not a detection failure.** The vulnerability was known
publicly, and a patch existed for six months. It was a **routing** failure — nothing carried the
information to a person who could act, at a moment when acting was cheap.

**That is the sentence to give any exam case study about security scanning.** Coverage is the easy half;
the graded half is **where the finding lands and how expensive it is to act on when it does.** Four
scanners feeding a dashboard nobody opens would have changed nothing.

**Why B is wrong on the boundary between scanners** — CodeQL analyses code you wrote (Q27, row 4).
**Why C is factually wrong** — transitive dependencies are exactly what Dependabot reads the lockfile
to find. **Why D is the six-month wait** the design exists to eliminate (Q2).

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **`queries:` without the leading `+`** | Q7, Q25, Q34, Q37 | Replaces the default suite instead of extending it |
| **Missing build for a compiled language** | Q5, Q18, Q27, Q33, Q47 | Zero results, green tick. Build between init and analyze |
| **Build placed before `init`** | Q33 | Init instruments the environment. Nothing observed otherwise |
| **`autobuild` trusted for compiled languages** | Q6, Q47 | It guesses, and fails silently when wrong |
| **Alerts assumed to prevent** | Q3, Q25, Q29, Q35 | Scanning detects. Push protection prevents |
| **Version updates confused with alerts** | Q11, Q28, Q43 | Alerts = vulnerabilities. Version updates = currency |
| **Dependabot expected to fix transitives** | Q2, Q28, Q48 | It reports. `overrides` / `resolutions` force the fix |
| **Dismissing a real alert as a false positive** | Q21, Q22, Q45 | `revoked` for real, `false_positive` for not-a-secret |
| **Unrestricted push-protection bypass** | Q20, Q26 | Named team, required reason |
| **SARIF upload without `category:`** | Q32, Q40 | One tool's results replace the other's |
| **Failing the build on every severity** | Q16, Q46 | Report at HIGH, block at CRITICAL |
| **`--admin` on the auto-merge** | Q39 | Bypasses branch protection. Use `--auto` |
| **`head_ref` as the Dependabot guard** | Q13, Q39 | Branch names are attacker-controlled. Use `github.actor` |
| **`+` carried into GHAzDO `querysuite`** | Q41 | Plain suite name there. The `+` is Actions-only |
| **GHAzDO expected to open upgrade PRs** | Q23, Q30 | It detects. Dependabot PRs are GitHub-side |
| **Mirroring Azure DevOps repos to GitHub** | Q1, Q42 | GHAzDO is native. Mirrors break PR annotations |

---

# What to memorise

**In `challenge-44.md`:** lines **33–48**, **88–110**, **141–192**, **396–420**, **296–319**.

```text
FOUR SCANNERS - they do NOT overlap
  CodeQL             flaws in code you WROTE        injection, traversal, hard-coded values
  Dependabot         flaws in code you IMPORTED     vulnerable packages, incl. TRANSITIVE
  Secret scanning    CREDENTIALS in the repo        + push protection = the blocking half
  Container scan     flaws in the IMAGE             base OS packages, installed binaries

  Dockerfile base image tag out of date  -> Dependabot (docker ecosystem)
  CVE inside the built image             -> Trivy

SECURITY TAB RENDERS SARIF.  Any scanner emitting SARIF publishes via codeql-action/upload-sarif.
PREVENT vs DETECT:  push protection, exit-code:'1', version updates PREVENT.
                    alerts, CodeQL, dependency scanning DETECT (unless a required check gates on them).
```

```yaml
# CodeQL - GitHub Actions                            (lines 61-110)
on:
  push: {branches: [main, develop]}
  pull_request: {branches: [main]}      # <- developers see it in the PR
  schedule: [{cron: '30 6 * * 1'}]      # <- new queries/advisories vs unchanged code
permissions:
  actions: read
  contents: read
  security-events: write                # <- the ONLY write. publishes alerts
steps:
  - uses: github/codeql-action/init@v3
    with:
      languages: ${{ matrix.language }}
      queries: +security-extended,security-and-quality   # THE + MEANS ADD. no + = REPLACE
  - run: dotnet build ...          # COMPILED languages: explicit build, BETWEEN init and analyze
  - uses: github/codeql-action/autobuild@v3    # interpreted languages only
  - uses: github/codeql-action/analyze@v3
    with: {category: "/language:${{ matrix.language }}"}
# no build -> zero results, GREEN TICK, a warning nobody reads
```

```yaml
# Dependabot                                         (lines 143-192, 223-237)
version: 2
updates:
  - package-ecosystem: "npm"          # npm | nuget | docker | github-actions  (the MANAGER, not the language)
    directory: "/"
    schedule: {interval: "weekly"}
    open-pull-requests-limit: 10      # per ecosystem
    groups:                           # one PR instead of forty
      production-dependencies:
        patterns: ["*"]
        update-types: ["minor", "patch"]
    ignore:
      - dependency-name: "express"
        update-types: ["version-update:semver-major"]   # majors blocked, patches still flow
    allow:
      - dependency-type: "production"

gh api repos/<org>/<repo>/vulnerability-alerts -X PUT     # ALERTS (security)
.github/dependabot.yml                                     # VERSION UPDATES (currency)
dismiss: --field dismissed_reason="not_used" --field dismissed_comment="..."   # unreachable, not inconvenient
TRANSITIVE with no upstream fix -> npm overrides / yarn resolutions
```

```yaml
# Secret scanning                                    (lines 33-48, 125-137)
gh api orgs/contoso -X PATCH  "advanced_security": enabled
                              "secret_scanning": enabled                 <- DETECT
                              "secret_scanning_push_protection": enabled <- BLOCK
custom pattern:  --field pattern="contoso_[a-zA-Z0-9]{32}" --field scope="organization"
bypass:  Specific roles or teams (security team) + Require a reason
resolutions:  revoked (it was real - rotated)  |  false_positive (never a credential)

# Container scan - two passes on purpose            (lines 396-420)
  pass 1  severity: 'CRITICAL,HIGH'  format: 'sarif'  -> upload-sarif  category: 'container-scanning'
  pass 2  severity: 'CRITICAL'       format: 'table'  exit-code: '1'   -> fails the build
# category: keeps result sets apart. without it, one upload REPLACES the other
```

```yaml
# GHAzDO - Azure Pipelines                           (lines 296-326)
- task: AdvancedSecurity-Dependency-Scanning@1
- task: AdvancedSecurity-Codeql-Init@1
    inputs: {languages: 'csharp,javascript', querysuite: 'security-extended'}   # NO leading +
- task: DotNetCoreCLI@2   # build, between init and analyze
- task: AdvancedSecurity-Codeql-Analyze@1
- task: AdvancedSecurity-Publish@1
# Repos > Advanced Security tab + PR ANNOTATIONS
# GHAzDO DETECTS vulnerable dependencies. It does NOT open upgrade PRs - that is GitHub-side
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 45 |
| 38–43 | Re-read the trap index and the four-scanner table, then move on |
| 30–37 | Rewrite the four scanners and the CodeQL step order from memory, then retake |
| Below 30 | Redo Tasks 2, 4 and 8 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 44.

:::danger The two questions

**Which scanner sees this?** Code you wrote, code you imported, a credential, or the image. They do not
overlap.

**Where does the finding land?** A dashboard is coverage. A pull request annotation is visibility.

Contoso's outage was never a detection failure — the patch had been public for six months.

:::
