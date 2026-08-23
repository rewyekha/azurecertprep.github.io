---
sidebar_position: 3.5
toc_max_heading_level: 2
title: "Challenge 15: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 15 — AZ-400 exam questions

**48 questions** built only from what Challenge 15 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-15.md`**.

:::danger Read this before you start

**Dependabot is two independent features, and the exam separates them in almost every question.**

**Security updates** fire **immediately** when a CVE is published against something in your dependency
tree. They are the emergency channel.
**Version updates** run **on a schedule** from `.github/dependabot.yml` and keep the tree current whether
or not anything is vulnerable.

**You need both, and for different reasons.** Alerts tell you about the fire; version updates are why the
fix is a one-line bump rather than a migration project.

**Three more distinctions carry the rest of the paper.**

**Vulnerability scanning ≠ license scanning.** One asks *is this dangerous*; the other asks *are we
allowed to ship it*. Different tools, different failure, and the GPL problem at line 25 is the second.

**Detect ≠ prevent.** An alert tells you a vulnerable package is **already there**. The
`dependency-review-action` refuses the **pull request that would introduce one** — and source mapping
refuses the **download**.

**And the vulnerability is almost never in a package you chose.** It is transitive (line 24), which is why
the dependency graph matters more than your `package.json`.

The scenario at line 21: **3 of 15** services on a critical CVE, found by a **quarterly** audit, with
remediation taking **weeks** because nobody knew which services were affected.

:::

---

# Section A — Multiple choice

---

## Q1

Dependabot is set to `weekly` with `open-pull-requests-limit: 5`, and 8 dependencies are outdated. What
happens?

- A. 8 PRs open at once
- B. 5 PRs open and the remaining 3 are queued
- C. 5 PRs open and the other 3 are ignored permanently
- D. The configuration is invalid

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-15.md`:** line **480**.

```text
Dependabot opens PRs up to the limit and queues the remaining updates. They will be opened as existing
PRs are merged or closed, or on the next scheduled run if capacity is available.
```

**The limit is a *concurrency* cap, not a filter.** Nothing is dropped; the queue drains as PRs close.

**Which makes the limit a throughput control on the team, not on Dependabot.** Set it to 5 and never
review anything, and you have five permanently stale PRs and a queue that never moves.

**Why C is the misreading that leads teams to set the limit high** — and then to mute the bot when 40 PRs
arrive.

</details>

---

## Q2

Which action blocks a pull request that introduces a known vulnerability?

- A. `actions/codeql-action`
- B. `actions/dependency-review-action`
- C. `github/dependabot-action`
- D. `actions/security-scan`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-15.md`:** lines **296–299**.

```yaml
      - name: Dependency review
        uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: high
```

**This is the *preventive* control, and it is the one the scenario is missing.** Dependabot alerts tell you
about a vulnerability that is **already merged**; dependency review refuses the merge.

**Why A is the wrong scanner.** CodeQL analyses **code you wrote** (Challenge 44); this analyses
**dependencies you added**.

**And it compares two refs** (lines 304–305) — base against head — so it reports only what **this PR**
introduces, not the pre-existing backlog.

</details>

---

## Q3

With NuGet package source mapping configured, what happens to a package matching no pattern?

- A. It is downloaded from all sources
- B. The restore **fails**
- C. It falls back to nuget.org
- D. It uses the first source

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-15.md`:** line **502**.

```text
When NuGet package source mapping is configured and a package matches no pattern, the restore fails.
This is a security feature that prevents unintended package sources from being used.
```

**Fail-closed by design.** A silent fallback would defeat the point — the mapping exists so that
`Contoso.*` can **only** come from the internal feed and nothing else can arrive from anywhere.

**Which is the strongest defence against dependency confusion in this challenge.** An attacker publishing
`Contoso.AuthSdk` to nuget.org cannot be resolved, because that pattern is mapped elsewhere (line 335).

**Why C is the behaviour without mapping**, and why the failure feels like a bug the first time: adding a
new legitimate package now requires adding a pattern.

</details>

---

## Q4

A team wants critical CVEs remediated automatically and everything else reviewed by hand. Which
configuration achieves it?

- A. `interval: daily` with `open-pull-requests-limit: 1`
- B. Enable Dependabot **security updates**, and configure **version updates** with `ignore` rules
- C. `fail-on-severity: critical` in the dependency review action
- D. `groups` batching all non-critical updates

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-15.md`:** line **513**.

```text
Dependabot has two independent features: security updates (triggered immediately by CVE publications,
always automatic) and version updates (scheduled, configurable).
```

**The two features answer the two halves of the requirement.** Security updates are automatic and
immediate; version updates are where you apply judgement.

**Why C is the right idea in the wrong place.** Dependency review gates a **pull request**; it does not
remediate an existing vulnerability, and it does nothing on the default branch.

**And note "always automatic"** — security updates do not wait for your `schedule`. That is precisely why
they are the emergency channel.

</details>

---

## Q5

A Dependabot PR bumping `@contoso/auth-sdk` from 1.2.0 to 2.0.0 breaks the build. What is the root cause?

- A. A breaking API change across a major version boundary
- B. A network failure
- C. A lockfile conflict
- D. A missing peer dependency

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** lines **372** and **381**.

```text
TypeError: AuthClient.validateToken is not a function
```

```text
The `validateToken` method was renamed to `verifyToken` in version 2.0.0.
```

**The major bump was the warning, and it was working correctly.** SemVer's MAJOR component exists to say
"this will break you" (Challenge 14) — Dependabot proposed it, CI caught it, and the process functioned.

**The defect is not that the PR was opened; it is that it was opened *unlabelled and un-isolated* among
12 others** (line 376), so nobody could tell at a glance which one carried the risk.

</details>

---

## Q6

Which Dependabot setting stops major bumps of a named package?

- A. `ignore` with `update-types: ["version-update:semver-major"]`
- B. `open-pull-requests-limit: 0`
- C. `schedule.interval: monthly`
- D. `allow: production`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** lines **399–401**.

```yaml
    ignore:
      - dependency-name: "@contoso/auth-sdk"
        update-types: ["version-update:semver-major"]
```

**`ignore` is per update-type, so minor and patch keep flowing.** That is the point: you are deferring a
migration, not freezing the package.

**Why B is the blunt instrument** — it disables the ecosystem entirely, including the security-relevant
patches.

**And the same construct appears at line 79** for `Microsoft.Extensions.*`, using a **wildcard** in
`dependency-name` — one rule covering a whole family.

</details>

---

## Q7

What is the difference between `~1.2.0` and `^1.2.0`?

- A. Tilde allows patch updates; caret allows minor and patch
- B. Tilde allows minor; caret allows major
- C. They are identical
- D. Tilde pins exactly

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** lines **406–423**.

```json
    "@contoso/auth-sdk": "~1.2.0"
```

```text
Or use a caret range to allow minor and patch but not major:
```

**Neither crosses a major boundary**, which is the property that matters — both would have refused the
2.0.0 bump that broke the build.

**The choice between them is a risk appetite.** Tilde accepts only fixes; caret accepts new features too,
trusting the publisher's SemVer discipline.

**And a range in `package.json` is a *policy*, while `ignore` in `dependabot.yml` is a *bot instruction*.**
The range protects every install; the ignore rule only stops the PR being opened.

</details>

---

## Q8

Which Dependabot ecosystems does the configuration cover?

- A. npm, NuGet, Docker, GitHub Actions
- B. npm only
- C. npm and NuGet
- D. npm, pip, Maven

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** lines **44**, **73**, **83**, **92**.

**`github-actions` as an ecosystem is the one teams forget** (line 92), and it is a genuine supply-chain
surface: a compromised or abandoned action runs with whatever permissions your workflow grants it.

**And `docker` watches the base image tag in your Dockerfile** — the complement to scanning the built
image (Challenge 44 Q4).

**Note each ecosystem has its own `directory:`** — `/` for npm, `/src/Contoso.Api` for NuGet (line 74) —
because the manifest lives where the project lives.

</details>

---

## Q9

What does `groups` accomplish in the Dependabot configuration?

- A. It batches related updates into one PR, isolating the ones that need attention
- B. It groups repositories
- C. It groups reviewers
- D. It sets priority

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** lines **60–70** and **450–464**.

```yaml
    groups:
      dev-dependencies:
        dependency-type: "development"
        update-types: ["minor", "patch"]
```

```text
This ensures major version bumps appear as individual PRs that are easy to identify and defer.
```

**Read the last line — that is the real purpose.** Grouping the routine updates means the **ungrouped**
PRs are exactly the ones carrying risk.

**Which is the answer to the Break scenario's real complaint** (line 376): twelve open PRs with no way to
tell which was dangerous. Group the minors and patches, and the major stands alone.

**And the two groups at lines 61–70 split by `dependency-type`** — development and production — so a
reviewer can merge dev-tooling churn without thinking hard about it.

</details>

---

## Q10

What does `dismissed_reason=tolerable_risk` record?

- A. A justified dismissal, with a comment explaining why the alert does not apply
- B. That the vulnerability is fixed
- C. That the package was removed
- D. That the alert was a duplicate

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** lines **152–155**.

```bash
gh api --method PATCH /repos/contoso/auth-service/dependabot/alerts/42 \
  --field state=dismissed \
  --field dismissed_reason=tolerable_risk \
  --field dismissed_comment="This code path is not reachable in our configuration"
```

**The comment is the audit trail**, and it is what makes the dismissal defensible six months later when
someone asks why a critical alert is closed.

**"This code path is not reachable" is a legitimate reason** — a vulnerability in a function you never
call cannot be exploited through your service.

**But it is a claim with a shelf life.** The next refactor may start calling it, and the dismissal will not
reopen itself — which is why the reason must be specific enough to re-evaluate.

</details>

---

## Q11

Which severity filter returns only critical and high open alerts?

- A. `/orgs/contoso/dependabot/alerts?severity=critical,high&state=open`
- B. `--jq 'select(.severity > 7)'`
- C. `?filter=high`
- D. `--severity high`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** lines **145–146**.

```bash
gh api "/orgs/contoso/dependabot/alerts?severity=critical,high&state=open" \
  --jq '.[] | "\(.repository.name): \(.dependency.package.name) - \(.security_advisory.severity) - \(.security_advisory.cve_id)"'
```

**Note the endpoint is `/orgs/`, not `/repos/`.** That single word answers the scenario's fourth complaint
(line 26): *"nobody knows which services are affected"* — one query, every repository.

**And the output names the repository first**, which is what turns a list of alerts into a remediation
plan.

</details>

---

## Q12

What does `fail-on-severity: high` do in the dependency review action?

- A. Fails the check when the PR introduces a vulnerability of high severity or above
- B. Warns on high severity
- C. Ignores anything below high
- D. Sets the alert threshold

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** lines **299** and **303**.

```yaml
          fail-on-severity: high
          ...
          warn-only: false
```

**`warn-only: false` is what makes it a gate**, and it is stated explicitly rather than left to default —
which is the habit worth copying.

**And "high or above" means high and critical.** Setting it to `critical` would let a high-severity
vulnerability merge, which is usually the wrong trade for a threshold nobody revisits.

**As always, the check gates only if it is a required status check** (Challenge 08).

</details>

---

## Q13

What does `deny-licenses: GPL-2.0-only, GPL-3.0-only, AGPL-3.0-only` prevent?

- A. Merging a PR that introduces a dependency under a copyleft licence
- B. A vulnerable dependency
- C. Publishing the package
- D. Using the licence in your own code

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** line **300**.

**This is the *legal* half of dependency review, and it addresses the scenario's third complaint** (line
25): GPL code accidentally shipped inside proprietary services.

**Why it is a separate concern from vulnerabilities.** A GPL package can be perfectly secure and still be
unusable — the risk is a licensing obligation, not an exploit.

**And AGPL is on the list for a specific reason.** Its network clause extends copyleft to software offered
**as a service**, which is precisely what Contoso's 15 microservices are.

</details>

---

## Q14

What does `allow-ghsas: GHSA-xxxx-yyyy-zzzz` do?

- A. Permits a specific advisory to pass, as a documented exception
- B. Allows all advisories
- C. Ignores GitHub Security Advisories
- D. Blocks that advisory

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** line **301**.

**A per-advisory exception, named explicitly.** It is the same idea as `dismissed_reason=tolerable_risk`
(Q10) applied at the gate rather than at the alert.

**And naming a specific GHSA is what makes it reviewable.** Lowering `fail-on-severity` to let one
advisory through would weaken the gate for **everything**; an allow-list weakens it for exactly one
known, argued case.

**The same principle as Challenge 43's gitleaks allowlist**: narrow the exemption to the case, never to
the category.

</details>

---

## Q15

What does the npm allow-list `.npmrc` at Task 6 accomplish?

- A. All packages resolve through approved feeds rather than public npm directly
- B. It blocks specific packages
- C. It scans for vulnerabilities
- D. It pins versions

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** lines **315–319**.

```ini
@contoso:registry=https://npm.pkg.github.com
registry=https://pkgs.dev.azure.com/contoso/ContosoServices/_packaging/contoso-packages/npm/registry/
```

```text
This prevents developers from accidentally pulling packages from public npm directly.
```

**Two lines, two routes: Contoso-scoped packages to GitHub Packages, everything else to the Azure
Artifacts feed** — which reaches public npm through an upstream (Challenge 13).

**The gain is a chokepoint.** Every package the organisation consumes passes through a feed you control,
so it can be cached, audited and scanned in one place.

**Why B is a different control** — that is the banned-package check at lines 353–361.

</details>

---

## Q16

What does the banned-package check at Task 6 do?

- A. Fails the build if a named package appears in the lockfile
- B. Removes the package
- C. Warns the developer
- D. Updates the package

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** lines **353–361**.

```bash
    BANNED_PACKAGES=("event-stream" "flatmap-stream" "ua-parser-js@0.7.29")
    ...
      if grep -q "\"$pkg\"" "$LOCKFILE"; then
        echo "ERROR: Banned package found: $pkg"
        exit 1
```

**Those three names are real supply-chain incidents**, and that is why the list exists: some packages are
not "vulnerable" in a CVE sense — they were **deliberately compromised**, and no version of them should
ever appear.

**It scans the lockfile, not `package.json`**, which is what catches a **transitive** occurrence — the
scenario's second complaint at line 24.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** are true of Dependabot's two features? (Choose three.)

- A. Security updates trigger immediately on a CVE publication
- B. Version updates run on the configured schedule
- C. Security updates are automatic and not schedule-driven
- D. Version updates only fire for vulnerabilities
- E. Security updates require `.github/dependabot.yml`
- F. They are the same feature

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-15.md`:** lines **118** and **513**.

```text
Security alerts differ from version updates. They trigger immediately when a new CVE is published that
affects your dependency tree.
```

**E is the misconception that leaves teams unprotected.** Security alerts are enabled at the repository or
organisation level (lines 106, 123–132) — **the config file is only for version updates.** Delete
`dependabot.yml` and alerts keep working.

**And D inverts the purpose of version updates.** They keep the tree current **regardless** of
vulnerabilities, which is what makes the eventual security fix a small change.

</details>

---

## Q18

Which **three** Dependabot behaviours can `.github/dependabot.yml` control? (Choose three.)

- A. Schedule and timezone
- B. Concurrent PR limit
- C. `ignore` rules by dependency and update type
- D. Whether CVEs are detected
- E. The severity of an advisory
- F. Which repositories are scanned

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-15.md`:** lines **46–51** and **78–80**.

**Plus reviewers, labels, commit-message prefix and groups** (lines 52–70) — the file is entirely about
**how updates are proposed**.

**D and E are properties of the advisory database**, not of your configuration. You cannot configure a
vulnerability into or out of existence; you can only choose how to respond.

**F is organisation-level** (lines 129–132), including
`dependabot_alerts_enabled_for_new_repositories` — which is the setting that stops the next repository
starting unprotected.

</details>

---

## Q19

Which **three** are preventive controls rather than detective ones? (Choose three.)

- A. `dependency-review-action` with `warn-only: false`
- B. NuGet package source mapping
- C. The banned-package lockfile check
- D. Dependabot alerts
- E. The quarterly security audit
- F. The org-wide alert listing query

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-15.md`:** lines **296–303**, **333–342**, **353–361**.

**All three refuse something**: a merge, a download, a build. D, E and F all report on something that has
already happened.

**And that split is the scenario's whole problem.** Contoso had only detection — a **quarterly** audit —
so three services ran a critical CVE for up to three months (line 21).

**Detection is necessary and it is not a control.** The exam grades whether you know which of the two a
given mechanism provides.

</details>

---

## Q20

Which **two** address the license compliance requirement? (Choose two.)

- A. `license-checker --failOn "GPL-2.0-only;GPL-3.0-only;AGPL-3.0-only;SSPL-1.0"`
- B. `deny-licenses` on the dependency review action
- C. Dependabot security updates
- D. Trivy
- E. CodeQL

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-15.md`:** lines **234–235** and **300**.

**Two layers for one requirement**: a dedicated check on the manifests, and the same policy enforced
inside dependency review.

**And the `--unknown` check at lines 240–245 is the subtler half.** A package with **no declared licence**
is not permitted-by-default — it is unassessable, and shipping it is a legal unknown rather than a
legal risk.

**Why C, D and E all scan for the wrong thing.** Vulnerability tools ask *is this dangerous*; licensing
asks *may we ship it* (Q13).

</details>

---

## Q21

Which **two** stop a package arriving from an unapproved source? (Choose two.)

- A. NuGet `packageSourceMapping` patterns
- B. An `.npmrc` that routes all installs through approved feeds
- C. `fail-on-severity: high`
- D. `open-pull-requests-limit`
- E. Dependabot `ignore` rules

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-15.md`:** lines **333–342** and **315–316**.

**Both make the registry a chokepoint rather than a default.**

**And source mapping is the stronger of the two** because it fails closed (Q3): a package matching no
pattern **cannot be restored at all**, whereas `.npmrc` routing can be overridden by a command-line
`--registry`.

**Which is the argument for enforcing it in CI as well as locally** — a developer's machine is
configurable; the pipeline's `nuget.config` is committed.

</details>

---

## Q22

Which **two** did the Break scenario's team lack? (Choose two.)

- A. Grouping, so routine updates do not hide the risky one
- B. `ignore` rules for major bumps on critical internal packages
- C. Dependabot itself
- D. A test suite
- E. A vulnerability scanner

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-15.md`:** lines **389–401** and **450–464**.

**Twelve open PRs with no visual distinction** (line 376) is the failure. Grouping the minors and patches
makes the major bump the only ungrouped PR — impossible to miss.

**Why C and D are refuted by the scenario itself.** Dependabot opened the PR and **CI caught the break**
(line 372) — both worked. The gap was triage.

**And that is a genuinely useful framing for the exam**: a control can function perfectly and still fail,
if its output is unreadable.

</details>

---

## Q23

Which **two** does the Microsoft Security DevOps task provide in Azure Pipelines? (Choose two.)

- A. Dependency scanning via the `dependencies` category
- B. Results published as a build artifact from `.gdn`
- C. Automatic dependency update PRs
- D. License compliance
- E. CodeQL analysis

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-15.md`:** lines **183–192**.

```yaml
  - task: MicrosoftSecurityDevOps@1
    inputs:
      categories: 'dependencies'
      tools: 'eslint,trivy'

  - task: PublishBuildArtifacts@1
    inputs:
      pathToPublish: $(System.DefaultWorkingDirectory)/.gdn
```

**Publishing `.gdn` is what makes the findings visible** — without it the scan runs and the results stay
on the agent (Challenge 45 Q6).

**Why C is the genuine platform gap.** Azure DevOps **detects** vulnerable dependencies; it does not open
Dependabot-style upgrade PRs. **That automation is GitHub-side**, and it is the difference that matters
when choosing where a repository lives.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must detect vulnerable dependencies automatically, understand transitive risk, enforce
a license policy, and know immediately which of 15 services are affected by a new CVE.

---

## Q24

**Proposed solution:** Enable Dependabot alerts and security updates organisation-wide, including for new
repositories. Add `.github/dependabot.yml` per repository covering npm, NuGet, docker and github-actions,
with grouped minor and patch updates so major bumps stand alone, and `ignore` rules for majors on
critical internal packages. Add the `dependency-review-action` on pull requests with `fail-on-severity:
high`, `deny-licenses` for copyleft licences and `warn-only: false`. Add a license check on manifest
changes that also fails on unknown licences. Route all installs through approved feeds with NuGet source
mapping. Query the organisation-wide alerts endpoint filtered to critical and high.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-15.md`:** lines **123–132**, **44–98**, **60–70**, **296–305**, **234–245**, **333–342**,
**145–146**.

| Complaint at line 21 | Mechanism |
|---|---|
| No automated detection | Alerts org-wide, plus new repositories by default |
| Transitive risk invisible | The dependency graph; the lockfile-scanning checks |
| No license policy | `deny-licenses` + `license-checker`, including unknowns |
| Nobody knows which services are affected | `/orgs/.../dependabot/alerts` in one query |
| Weeks to remediate | Version updates keep the tree close enough to patch cheaply |

**The last row is the one that is easy to omit and does the most work.** Alerts alone tell you about a fix
you cannot cheaply apply.

</details>

---

## Q25

**Proposed solution:** Keep the quarterly security audit and add a spreadsheet listing each service's
dependencies. Enable Dependabot version updates only, with `open-pull-requests-limit: 20` so nothing is
missed. Ask developers to check licences during code review. Pin every dependency to an exact version so
nothing changes unexpectedly.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures.**

**A quarterly audit plus a spreadsheet is the current state** (line 21), with a manually maintained
inventory added — and an inventory maintained by hand is wrong within a sprint.

**Version updates without security alerts** removes the immediate channel (Q17). A CVE published on Tuesday
waits for the weekly schedule, or longer if the package is not otherwise outdated.

**Licence review by humans does not scale to transitive dependencies.** The GPL problem at line 25 was
**accidental** — nobody chose to include GPL code; it arrived several levels deep, where no reviewer looks.

**And pinning every version freezes the tree**, which sounds safe and is the opposite: security patches
stop arriving, and the eventual forced upgrade spans multiple majors — the "weeks to remediate" at
line 26, made permanent.

</details>

---

## Q26

**Proposed solution:** Enable Dependabot alerts and security updates organisation-wide. Add
`.github/dependabot.yml` for all four ecosystems with grouping and `ignore` rules. Add the
`dependency-review-action` with `deny-licenses`, a license check including unknowns, and NuGet source
mapping. Query alerts organisation-wide. Set the dependency review action to `warn-only: true` so
developers see the findings without a red check blocking urgent work.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**`warn-only: true` turns the only preventive control in the design back into a detective one.** The
action still runs, still comments on the PR (line 302), and **the merge proceeds** — so a PR introducing a
critical CVE lands exactly as it did before any of this was configured.

**And the stated justification is the one that always accompanies it.** "Without blocking urgent work" is
persuasive because it is occasionally true — and the correct response to a genuine emergency is
`allow-ghsas` naming that specific advisory (Q14), or a documented bypass, **not** disabling the gate for
every future PR.

**The failure is also invisible in the right way to be dangerous.** Nothing errors, the summary comment
still appears, and the team believes dependency review is protecting them. **A warning that nobody must
act on is indistinguishable from no control at all**, and it will be scrolled past inside a fortnight.

**Line 303 sets it explicitly to `false`** for that reason — the value is stated rather than defaulted,
because it is the line that decides whether this is a gate.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — the two Dependabot features

| # | Statement | Answer |
|---|---|---|
| 1 | Security updates trigger immediately on a CVE |  |
| 2 | Version updates run on a schedule |  |
| 3 | Security alerts require `dependabot.yml` |  |
| 4 | `open-pull-requests-limit` permanently drops excess updates |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Security updates trigger immediately on a CVE | **Yes** |
| 2 | Version updates run on a schedule | **Yes** |
| 3 | Security alerts require `dependabot.yml` | **No** |
| 4 | `open-pull-requests-limit` permanently drops excess updates | **No** |

**In `challenge-15.md`:** lines **118**, **46–47**, **106**, **480**.

Row 3 is the misconception that leaves a repository unprotected while looking configured.

Row 4: the excess is **queued** (Q1).

</details>

---

## Q28 — gating

| # | Statement | Answer |
|---|---|---|
| 1 | `dependency-review-action` compares base and head refs |  |
| 2 | `warn-only: false` makes it a blocking check |  |
| 3 | It scans code for injection flaws |  |
| 4 | `deny-licenses` blocks copyleft dependencies |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `dependency-review-action` compares base and head refs | **Yes** |
| 2 | `warn-only: false` makes it a blocking check | **Yes** |
| 3 | It scans code for injection flaws | **No** |
| 4 | `deny-licenses` blocks copyleft dependencies | **Yes** |

**In `challenge-15.md`:** lines **304–305**, **303**, **491**, **300**.

Row 3 is the CodeQL boundary — **dependencies you added** versus **code you wrote** (Q2).

Row 1 is why it reports only what the PR introduces rather than the whole backlog.

</details>

---

## Q29 — source control of packages

| # | Statement | Answer |
|---|---|---|
| 1 | NuGet source mapping fails the restore for an unmapped package |  |
| 2 | An `.npmrc` registry line routes installs through an approved feed |  |
| 3 | Source mapping falls back to nuget.org |  |
| 4 | The banned-package check scans the lockfile |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | NuGet source mapping fails the restore for an unmapped package | **Yes** |
| 2 | An `.npmrc` registry line routes installs through an approved feed | **Yes** |
| 3 | Source mapping falls back to nuget.org | **No** |
| 4 | The banned-package check scans the lockfile | **Yes** |

**In `challenge-15.md`:** lines **502**, **315–316**, **502**, **354–357**.

Row 3 is the fail-closed property that makes row 1 a security control rather than an inconvenience.

Row 4 is what catches a **transitive** occurrence, which `package.json` would not show.

</details>

---

## Q30 — licences

| # | Statement | Answer |
|---|---|---|
| 1 | Licence scanning and vulnerability scanning answer different questions |  |
| 2 | A package with an unknown licence should fail the check |  |
| 3 | AGPL is treated the same as MIT by the policy |  |
| 4 | Licence risk can be transitive |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Licence scanning and vulnerability scanning answer different questions | **Yes** |
| 2 | A package with an unknown licence should fail the check | **Yes** |
| 3 | AGPL is treated the same as MIT by the policy | **No** |
| 4 | Licence risk can be transitive | **Yes** |

**In `challenge-15.md`:** lines **234–235**, **240–245**, **300**, **25**.

Row 2 is the half people omit — unknown is **unassessable**, not permitted.

Row 4 is the scenario's own wording: the GPL code was **accidentally** included, which only happens
transitively.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each requirement to its mechanism.

| Requirement | Mechanism |
|---|---|
| Know immediately when a CVE affects us |  |
| Keep the tree current so patching is cheap |  |
| Stop a PR introducing a vulnerable package |  |
| Stop a copyleft licence entering the product |  |
| Stop a package arriving from an unapproved source |  |
| Know which of 15 services are affected |  |

**Options:** `deny-licenses` + `license-checker` · Dependabot security alerts · Dependabot version updates · `dependency-review-action` · NuGet source mapping / `.npmrc` routing · `/orgs/.../dependabot/alerts`

<details>
<summary>Show answer</summary>

| Requirement | Mechanism |
|---|---|
| Know immediately when a CVE affects us | **Dependabot security alerts** |
| Keep the tree current so patching is cheap | **Dependabot version updates** |
| Stop a PR introducing a vulnerable package | **`dependency-review-action`** |
| Stop a copyleft licence entering the product | **`deny-licenses` + `license-checker`** |
| Stop a package arriving from an unapproved source | **NuGet source mapping / `.npmrc` routing** |
| Know which of 15 services are affected | **`/orgs/.../dependabot/alerts`** |

**In `challenge-15.md`:** lines **118**, **46–47**, **297**, **300** with **234**, **333–342**,
**145–146**.

**Rows 1 and 2 are the pair the exam separates constantly.** Alerts tell you about the fire; version
updates are why the extinguisher is within reach.

</details>

---

## Q32

Match each control to whether it detects or prevents.

| Control | Type |
|---|---|
| Dependabot alerts |  |
| Dependabot security updates |  |
| `dependency-review-action` with `warn-only: false` |  |
| NuGet `packageSourceMapping` |  |
| Banned-package lockfile check |  |
| The quarterly audit |  |

**Options:** Detect · Detect, slowly · Prevent · Prevent — fails the build · Prevent — fails the restore · Remediate — a PR, still merged by a human

<details>
<summary>Show answer</summary>

| Control | Type |
|---|---|
| Dependabot alerts | **Detect** |
| Dependabot security updates | **Remediate — a PR, still merged by a human** |
| `dependency-review-action` with `warn-only: false` | **Prevent** |
| NuGet `packageSourceMapping` | **Prevent — fails the restore** |
| Banned-package lockfile check | **Prevent — fails the build** |
| The quarterly audit | **Detect, slowly** |

**In `challenge-15.md`:** lines **112**, **513**, **303**, **502**, **358**, **21**.

**Only three of the six refuse anything.** The scenario had the detective half only — which is why a
critical CVE lived in production for up to a quarter.

</details>

---

## Q33

Arrange the steps to close the gap the audit found.

**Items:** Add `dependency-review-action` gating pull requests · Query the organisation alerts endpoint to
find every affected service · Enable Dependabot alerts and security updates org-wide · Add
`dependabot.yml` with grouping and ignore rules · Add license and source-mapping controls

<details>
<summary>Show answer</summary>

### Answer

1. Enable Dependabot alerts and security updates org-wide — lines **123–132**
2. Query the organisation alerts endpoint to find every affected service — lines **145–146**
3. Add `dependency-review-action` gating pull requests — lines **296–305**
4. Add `dependabot.yml` with grouping and ignore rules — lines **44–98**, **450–464**
5. Add license and source-mapping controls — lines **234–245**, **333–342**

**Steps 1 and 2 come first because they answer the immediate question**: three services are on a critical
CVE **right now**, and the org-wide query is how you find the third one you did not know about.

**Step 3 before step 4 is deliberate.** Gating stops the problem **growing** while you work; the schedule
and grouping configuration is about steady-state hygiene and can wait a day.

**And step 5 last because it is the slowest to land** — source mapping requires enumerating every package
pattern, and it fails closed (Q3), so a hasty rollout breaks restores.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| A CVE published Tuesday is not noticed until the weekly run |  |
| Twelve PRs open and nobody spots the breaking one |  |
| A restore fails for a newly added package |  |
| GPL code ships in a proprietary service |  |
| A PR introducing a critical CVE merges cleanly |  |
| The eventual security patch takes weeks |  |

**Options:** No grouping — routine and risky look alike · No licence policy, and it arrived transitively · Security alerts not enabled · Source mapping with no matching pattern · The tree was pinned and is majors behind · `warn-only: true`, or the check is not required

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| A CVE published Tuesday is not noticed until the weekly run | **Security alerts not enabled** |
| Twelve PRs open and nobody spots the breaking one | **No grouping — routine and risky look alike** |
| A restore fails for a newly added package | **Source mapping with no matching pattern** |
| GPL code ships in a proprietary service | **No licence policy, and it arrived transitively** |
| A PR introducing a critical CVE merges cleanly | **`warn-only: true`, or the check is not required** |
| The eventual security patch takes weeks | **The tree was pinned and is majors behind** |

**In `challenge-15.md`:** lines **118**, **376**, **502**, **25**, **303**, **26**.

**Rows 5 and 6 are the two that make the whole programme decorative** — one lets the vulnerability in, the
other makes the fix unaffordable.

</details>

---

## Q35

Match each scanner to what it inspects.

| Scanner | Inspects |
|---|---|
| Dependabot |  |
| `dependency-review-action` |  |
| `license-checker` |  |
| Banned-package check |  |
| Microsoft Security DevOps |  |
| CodeQL |  |

**Options:** **Code you wrote** (Challenge 44) · Declared licences of production dependencies · Dependencies in an Azure Pipeline, with eslint and trivy · The lockfile, for named packages · What this PR adds, base against head · Your dependency tree against advisories

<details>
<summary>Show answer</summary>

| Scanner | Inspects |
|---|---|
| Dependabot | **Your dependency tree against advisories** |
| `dependency-review-action` | **What this PR adds, base against head** |
| `license-checker` | **Declared licences of production dependencies** |
| Banned-package check | **The lockfile, for named packages** |
| Microsoft Security DevOps | **Dependencies in an Azure Pipeline, with eslint and trivy** |
| CodeQL | **Code you wrote** (Challenge 44) |

**In `challenge-15.md`:** lines **112**, **304–305**, **234**, **354–357**, **183–187**, **491**.

**Five of these six read *dependencies*; one reads *your source*.** Keeping that boundary is the recurring
exam question across Challenges 15, 44 and 45.

</details>

---

# Section F — Hot area

---

## Q36

```yaml
version: 2
updates:
  - package-ecosystem: "[BLANK 1]"
    directory: "/"
    schedule:
      interval: "[BLANK 2]"
    [BLANK 3]: 10
```

Requirement: weekly npm updates, at most ten open pull requests at a time.

- **BLANK 1:** `npm` / `nodejs` / `javascript` / `yarn`
- **BLANK 2:** `weekly` / `daily` / `monthly` / `hourly`
- **BLANK 3:** `open-pull-requests-limit` / `max-prs` / `pr-limit` / `concurrency`

<details>
<summary>Show answer</summary>

### Answer: `npm`, `weekly`, `open-pull-requests-limit`

**In `challenge-15.md`:** lines **44–51**.

**`package-ecosystem` names the *package manager*, not the language** — the same rule as Challenge 44.
The four here are `npm`, `nuget`, `docker` and `github-actions`.

**And the limit queues rather than drops** (Q1), so it is a review-capacity setting rather than a filter.

</details>

---

## Q37

```yaml
    [BLANK 1]:
      - dependency-name: "Microsoft.Extensions.*"
        [BLANK 2]: ["version-update:[BLANK 3]"]
```

Requirement: allow minor and patch updates to that family, but never a major bump.

- **BLANK 1:** `ignore` / `deny` / `exclude` / `block`
- **BLANK 2:** `update-types` / `versions` / `semver` / `levels`
- **BLANK 3:** `semver-major` / `semver-minor` / `major` / `breaking`

<details>
<summary>Show answer</summary>

### Answer: `ignore`, `update-types`, `semver-major`

**In `challenge-15.md`:** lines **78–80**.

**`ignore` is per update-type, which is what keeps the minors and patches flowing** (Q6) — the wildcard in
`dependency-name` covers the whole `Microsoft.Extensions.*` family in one rule.

**The full vocabulary is `version-update:semver-major`, `-minor` and `-patch`**, and it is the same
classification Dependabot uses to label its PRs.

</details>

---

## Q38

```yaml
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: [BLANK 1]
          deny-licenses: GPL-2.0-only, GPL-3.0-only, AGPL-3.0-only
          [BLANK 2]: false
          base-ref: ${{ github.event.pull_request.[BLANK 3] }}
```

Requirement: block the merge on high or critical vulnerabilities.

- **BLANK 1:** `high` / `critical` / `moderate` / `low`
- **BLANK 2:** `warn-only` / `fail-fast` / `continue-on-error` / `strict`
- **BLANK 3:** `base.sha` / `head.sha` / `merge_commit_sha` / `number`

<details>
<summary>Show answer</summary>

### Answer: `high`, `warn-only`, `base.sha`

**In `challenge-15.md`:** lines **299–304**.

**`warn-only: false` is the line that makes it a gate** (Q12, Q26), and it is set explicitly rather than
left to a default — which is the habit to copy for anything that decides whether a merge proceeds.

**And `base.sha` against `head.sha`** is what scopes the review to this PR's additions rather than the
repository's existing backlog.

</details>

---

## Q39

```bash
license-checker --production --[BLANK 1] \
  "GPL-2.0-only;GPL-3.0-only;AGPL-3.0-only;SSPL-1.0" \
  --summary

UNKNOWN=$(license-checker --production --[BLANK 2] | wc -l)
if [ "$UNKNOWN" -gt 0 ]; then exit 1; fi
```

- **BLANK 1:** `failOn` / `deny` / `banned` / `exclude`
- **BLANK 2:** `unknown` / `missing` / `unlicensed` / `null`

<details>
<summary>Show answer</summary>

### Answer: `failOn`, `unknown`

**In `challenge-15.md`:** lines **234** and **240**.

**`--production` matters on both invocations.** A GPL dev-dependency used only by a test runner is not
shipped and is usually acceptable; the same package in production is not.

**And the unknown check is the half people leave out** (Q30). A package with no declared licence has not
passed — it has not been **assessed**, which is a different and worse position to defend.

</details>

---

## Q40

```xml
  <[BLANK 1]>
    <packageSource key="contoso-packages">
      <package pattern="[BLANK 2]" />
    </packageSource>
    <packageSource key="nuget.org">
      <package pattern="Microsoft.*" />
    </packageSource>
  </[BLANK 1]>
```

Requirement: internal packages may come only from the internal feed.

- **BLANK 1:** `packageSourceMapping` / `packageSources` / `packageRestore` / `sourceMapping`
- **BLANK 2:** `Contoso.*` / `*` / `Microsoft.*` / `Contoso`

<details>
<summary>Show answer</summary>

### Answer: `packageSourceMapping`, `Contoso.*`

**In `challenge-15.md`:** lines **333–341**.

**Note `<clear />` at line 329 immediately before the sources.** It discards any inherited configuration
from a machine-level `NuGet.Config`, so the pipeline's sources are exactly these two and nothing a
developer's machine adds leaks in.

**And `Contoso` without the wildcard would match one package literally** — the `.*` is what covers the
family and closes the dependency-confusion gap.

</details>

---

## Q41

```bash
gh api --method PATCH /repos/contoso/auth-service/dependabot/alerts/42 \
  --field state=[BLANK 1] \
  --field [BLANK 2]=tolerable_risk \
  --field dismissed_comment="This code path is not reachable in our configuration"
```

- **BLANK 1:** `dismissed` / `closed` / `resolved` / `ignored`
- **BLANK 2:** `dismissed_reason` / `reason` / `justification` / `resolution`

<details>
<summary>Show answer</summary>

### Answer: `dismissed`, `dismissed_reason`

**In `challenge-15.md`:** lines **152–155**.

**The reason and the comment together are the audit trail** (Q10) — six months later, the comment is what
lets someone re-evaluate rather than guess.

**And `tolerable_risk` is one of a fixed set of reasons.** Choosing an accurate one matters: dismissing a
real, reachable vulnerability as `tolerable_risk` clears the dashboard and leaves the service exposed —
the same failure as marking a real secret `false_positive` in Challenge 44.

</details>

---

# Section G — Case study

## Case study: Contoso dependency security

### Background

Contoso's security team runs a **quarterly audit** and finds **3 of 15 production microservices** depend
on a library with a **critical CVE** — an Express.js path traversal. **No automated process detects
vulnerable dependencies.** Developers are **unaware which transitive dependencies carry risk**. There is
**no licence policy**, and some teams have **accidentally shipped GPL code in proprietary services**.
**Remediation takes weeks** because nobody knows which services are affected.

### Requirements

**Detection**

- A new CVE affecting any service must be known immediately, not quarterly
- It must be possible to list every affected service in one operation
- Transitive dependencies must be in scope

**Prevention**

- A pull request that introduces a high or critical vulnerability must not merge
- A copyleft-licensed dependency must not enter a proprietary service
- Packages must only come from approved sources

**Remediation**

- Applying a security patch must be a small change, not a migration
- A breaking major update must be distinguishable at a glance from routine updates

---

## Q42

How is "a new CVE must be known immediately" satisfied?

- A. Dependabot **security alerts**, enabled organisation-wide including for new repositories
- B. Dependabot version updates on a daily schedule
- C. The quarterly audit, run monthly
- D. A dependency review action

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** lines **118** and **129–132**.

```bash
gh api --method PUT /orgs/contoso \
  --field dependabot_security_updates_enabled_for_new_repositories=true \
  --field dependency_graph_enabled_for_new_repositories=true \
  --field dependabot_alerts_enabled_for_new_repositories=true
```

**Security alerts are event-driven, not scheduled** (Q17) — they fire when the advisory is published.

**Why B is the near-miss.** A daily schedule is still a **poll**, and it only notices a CVE if the package
also happens to have a newer version to move to.

**And the `_for_new_repositories` flags are the durable part**: the sixteenth service is protected on
creation, without anyone remembering.

</details>

---

## Q43

How do you find every affected service in one operation?

- A. `gh api "/orgs/contoso/dependabot/alerts?severity=critical,high&state=open"`
- B. Query each of the 15 repositories in turn
- C. Read the spreadsheet
- D. Ask each team

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** lines **145–146**.

**The `/orgs/` endpoint is the difference between minutes and weeks**, and it answers the scenario's
fourth complaint directly (line 26).

**Why B works and is not "one operation"** — and it misses any repository nobody remembered to include,
which is how the audit found a third affected service nobody expected.

**And the output at line 146 formats repository, package, severity and CVE per line**, which is a
remediation worklist rather than a data dump.

</details>

---

## Q44

Which **two** prevent a vulnerable or copyleft dependency entering the codebase? (Choose two.)

- A. `dependency-review-action` with `fail-on-severity: high` and `warn-only: false`
- B. `deny-licenses` on the same action, plus a `license-checker` job on manifest changes
- C. Dependabot alerts
- D. The quarterly audit
- E. Grouping updates

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-15.md`:** lines **296–305** and **214–245**.

**Both are gates on the pull request**, which is the only point where "entering the codebase" can be
prevented.

**Why C is the control that runs one step too late.** An alert fires **after** the dependency is in the
default branch — useful, and by definition not prevention (Q19).

**And note the licence job's path filter** (lines 217–221): it runs when `package.json`, the lockfile or a
`.csproj` changes, so it is not paying for itself on every documentation commit.

</details>

---

## Q45

How is "applying a security patch must be a small change" satisfied?

- A. Dependabot version updates keeping the tree current, with `ignore` rules only for majors on critical
  internal packages
- B. Pinning every dependency to an exact version
- C. Updating only when a CVE appears
- D. Quarterly bulk upgrades

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** lines **44–70** and **399–401**.

**A tree that is weeks behind takes a version bump; a tree that is years behind takes a project.** That is
the whole argument for routine version updates, and it is the scenario's "weeks to remediate" (line 26).

**Why B is the intuition that causes the problem** (Q25). Exact pins feel safe and guarantee the tree
drifts further behind every month — so the eventual mandatory security fix spans several majors.

**And the `ignore` rules are deliberately narrow**: majors, on named critical packages only. Everything
else keeps flowing.

</details>

---

## Q46

How is a breaking major update made distinguishable at a glance?

- A. Group minor and patch updates so a major bump is the only ungrouped PR
- B. Label every PR
- C. Reduce the PR limit
- D. Review PRs daily

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** lines **450–464**.

```text
This ensures major version bumps appear as individual PRs that are easy to identify and defer.
```

**Grouping works by making the risky PR *structurally* different**, not by adding information a reviewer
must read.

**Why B is genuinely useful and insufficient.** Dependabot already labels its PRs (lines 54–56); twelve
labelled PRs still look alike in a list.

**And that is the Break scenario's actual failure** (Q22): both the bot and CI worked perfectly, and the
output was unreadable.

</details>

---

## Q47

Seven months in, the security team notices that no dependency-review check has failed on any repository
for four months, although Dependabot alerts show fourteen high-severity vulnerabilities were introduced
and later fixed in that period. The workflow runs on every pull request and reports a summary comment.

What happened, and what is the fix?

- A. `warn-only` was set to `true`, or the job was never added as a required status check — so the review
  reports and never blocks; restore `warn-only: false` and require the check in branch protection
- B. The action version is outdated
- C. `fail-on-severity` was set to `critical`
- D. Dependabot alerts were misconfigured

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** lines **303** and Challenge 08's required-checks rule.

**"Runs on every PR and reports a summary comment" is the detail that identifies it.** The action is
healthy and doing half its job — the half that informs.

**And the giveaway is the shape of the evidence: vulnerabilities were *introduced and later fixed*.** If
the gate were working, they would never have been introduced; the alerts fired afterwards, which means
the merge went through.

**Two causes produce the identical symptom**, and both must be checked. `warn-only: true` makes the job
succeed regardless of findings. And even with `warn-only: false`, a **failing job that is not a required
status check** leaves a red mark someone can merge past — Challenge 08's recurring lesson.

**Why C would have blocked *some* of the fourteen.** With `fail-on-severity: critical`, the high-severity
ones pass and the critical ones fail — so a run of four months with **zero** failures is not consistent
with C.

**The durable lesson: for any control, ask what a *failure* would look like.** If the answer is "a
comment", it is not a gate.

</details>

---

## Q48

A year on, a newly published CVE is triaged the same morning, every affected service is named in one
query, and no GPL dependency has entered a proprietary service.

Explain what each control contributed, and what actually changed.

- A. Security alerts made detection event-driven rather than quarterly; the org-wide query turned "which
  services?" into one operation; dependency review and licence checks moved the decision to the pull
  request, before the dependency exists in the branch; source mapping made the registry a chokepoint; and
  routine version updates kept the tree close enough that a patch is a bump — the programme moved from
  **finding** problems to **refusing** them
- B. The team became more careful about dependencies
- C. More frequent audits were scheduled
- D. Fewer third-party packages were used

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-15.md`:** lines **118**, **145–146**, **296–305**, **333–342**, **44–70**.

**Take the four complaints at line 21 in turn.**

*No automated detection* — a quarterly audit is a **sampling** process, and its worst case is a full
quarter of exposure. Alerts are event-driven, so the worst case is the time between publication and
someone reading the notification.

*Transitive risk invisible* — the dependency graph and the lockfile-scanning checks look at what is
**actually resolved**, not at what was declared. That is where the Express CVE was.

*No licence policy* — and note the word in the scenario: *accidentally*. Nobody chose GPL code; it arrived
several levels deep, where review does not reach. Only an automated check operates at that depth.

*Weeks to remediate* — two causes, and the design addresses both: nobody could enumerate the affected
services (now one query), and the fix was a large upgrade (now a bump, because the tree is current).

**What actually changed is the point in the lifecycle where the decision happens.** Before, every control
ran **after** the dependency was in production, so every finding was an incident. Now the expensive
controls run at the **pull request**, where rejecting a package costs nothing.

**The graded idea: detection tells you how bad it is; prevention decides how often it happens.** A
security programme built only from alerts measures its own failure rate — and Contoso's measured it four
times a year.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Security alerts and version updates conflated** | Q4, Q17, Q27, Q42 | Immediate and automatic vs scheduled and configurable |
| **Alerts assumed to need `dependabot.yml`** | Q17, Q27 | Enabled at repo or org level. The file is for version updates |
| **`open-pull-requests-limit` read as a filter** | Q1, Q27 | Excess is queued, not dropped |
| **`warn-only: true`** | Q12, Q26, Q47 | Turns the only gate back into a report |
| **A failing job assumed to block a merge** | Q47 | Only when it is a required status check |
| **CodeQL offered for dependency vulnerabilities** | Q2, Q35 | Code you wrote vs code you imported |
| **Licence risk treated as a security scan** | Q13, Q20, Q30 | Different question, different tool |
| **Unknown licences treated as acceptable** | Q30, Q39 | Unassessed, not approved |
| **Source mapping assumed to fall back** | Q3, Q29, Q40 | Fails closed. That is the security property |
| **Pinning everything to exact versions** | Q25, Q45 | The tree drifts; the eventual fix spans majors |
| **Major bumps blamed on Dependabot** | Q5, Q22 | It warned you. The gap was triage |
| **Labels assumed sufficient to spot a risky PR** | Q46 | Grouping makes it structurally distinct |
| **Dismissing a reachable vulnerability as tolerable** | Q41 | Clears the dashboard, not the risk |
| **Azure DevOps expected to open upgrade PRs** | Q23 | It detects. Dependabot PRs are GitHub-side |

---

# What to memorise

**In `challenge-15.md`:** lines **40–99**, **116–156**, **274–306**, **308–363**.

```text
DEPENDABOT IS TWO FEATURES
  SECURITY UPDATES   fire IMMEDIATELY on a CVE. automatic. enabled at REPO or ORG level.
                     do NOT need dependabot.yml
  VERSION UPDATES    run on a SCHEDULE from .github/dependabot.yml. keep the tree CURRENT
                     -> this is why the eventual security fix is a bump, not a migration

DETECT vs PREVENT - the exam grades which one a mechanism is
  DETECT   Dependabot alerts | the quarterly audit | org-wide alert query
  PREVENT  dependency-review-action (warn-only: false, AND a required check)
           NuGet packageSourceMapping (FAILS CLOSED)
           banned-package lockfile check (exit 1)

VULNERABILITY vs LICENCE - different questions, different tools
  "is this dangerous?"      Dependabot | dependency review | Defender for DevOps
  "may we ship it?"         deny-licenses | license-checker | dotnet-project-licenses
  the GPL problem is ACCIDENTAL and TRANSITIVE - only an automated check reaches that depth
```

```yaml
# .github/dependabot.yml                            (lines 40-99)
version: 2
updates:
  - package-ecosystem: "npm"          # npm | nuget | docker | github-actions (the MANAGER)
    directory: "/"                    # where the manifest lives
    schedule: {interval: "weekly", day: "monday", time: "09:00", timezone: "America/New_York"}
    open-pull-requests-limit: 10      # a CONCURRENCY cap - excess is QUEUED, not dropped
    reviewers: ["contoso/backend-team"]
    labels: ["dependencies", "automated"]
    commit-message: {prefix: "deps", include: "scope"}
    groups:                           # batch the routine so the RISKY one stands alone
      dev-dependencies:  {dependency-type: "development", update-types: ["minor","patch"]}
      production-minor:  {dependency-type: "production",  update-types: ["minor","patch"]}
    ignore:
      - dependency-name: "Microsoft.Extensions.*"      # wildcards allowed
        update-types: ["version-update:semver-major"]  # minor+patch keep flowing

# enable org-wide, including future repositories     (lines 129-132)
dependabot_alerts_enabled_for_new_repositories | dependabot_security_updates_... | dependency_graph_...

# find every affected service in ONE call            (lines 145-146)
gh api "/orgs/contoso/dependabot/alerts?severity=critical,high&state=open"

# dismiss with an audit trail                        (lines 152-155)
--field state=dismissed --field dismissed_reason=tolerable_risk --field dismissed_comment="..."
```

```yaml
# THE GATE                                           (lines 280-306)
- uses: actions/dependency-review-action@v4
  with:
    fail-on-severity: high            # high AND critical
    deny-licenses: GPL-2.0-only, GPL-3.0-only, AGPL-3.0-only
    allow-ghsas: GHSA-xxxx-yyyy-zzzz  # a NAMED exception, not a lowered threshold
    comment-summary-in-pr: always
    warn-only: false                  # <- THE LINE THAT MAKES IT A GATE
    base-ref: ${{ github.event.pull_request.base.sha }}    # only what THIS PR adds
    head-ref: ${{ github.event.pull_request.head.sha }}
# and it still only blocks if it is a REQUIRED STATUS CHECK (Challenge 08)
```

```text
LICENCE + SOURCE CONTROL                             (lines 214-363)
license-checker --production --failOn "GPL-2.0-only;GPL-3.0-only;AGPL-3.0-only;SSPL-1.0" --summary
license-checker --production --unknown        # UNKNOWN = unassessed, not approved -> exit 1
dotnet-project-licenses --banned-license-types ./banned-licenses.json

.npmrc      @contoso:registry=<github packages>
            registry=<azure artifacts feed>      # everything else through an approved feed

nuget.config  <clear />                          # discard machine-level sources
              <packageSourceMapping>
                <packageSource key="contoso-packages"><package pattern="Contoso.*" /></packageSource>
                <packageSource key="nuget.org"><package pattern="Microsoft.*" /></packageSource>
              </packageSourceMapping>
# a package matching NO pattern -> THE RESTORE FAILS. no fallback. that is the security feature.

banned packages: grep the LOCKFILE (catches TRANSITIVE), exit 1
  event-stream | flatmap-stream | ua-parser-js@0.7.29   <- real supply-chain compromises

AZURE PIPELINES                                      (lines 183-192)
- task: MicrosoftSecurityDevOps@1
  inputs: {categories: 'dependencies', tools: 'eslint,trivy'}
- task: PublishBuildArtifacts@1
  inputs: {pathToPublish: $(System.DefaultWorkingDirectory)/.gdn}   # or the results stay on the agent
# ADO DETECTS vulnerable dependencies. it does NOT open upgrade PRs - that is Dependabot, GitHub-side.
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Package management complete. Move to Challenge 16 |
| 38–43 | Re-read the trap index and the detect-vs-prevent table, then move on |
| 30–37 | Write the two Dependabot features and the three preventive controls from memory, then retake |
| Below 30 | Redo Tasks 1, 5 and 6 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 15.

:::danger The three splits

**Security updates vs version updates.** Immediate and automatic; scheduled and configurable. You need
both — alerts find the fire, version updates make the fix cheap.

**Detect vs prevent.** An alert fires after the dependency is merged. `dependency-review-action` with
`warn-only: false` **and a required check** refuses the merge. Source mapping refuses the download.

**Vulnerability vs licence.** *Is this dangerous* and *may we ship it* are different questions with
different tools — and the GPL arrived transitively, where no human reviewer was looking.

:::
