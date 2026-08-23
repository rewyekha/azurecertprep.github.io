---
sidebar_position: 7.5
toc_max_heading_level: 2
title: "Challenge 45: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 45 — AZ-400 exam questions

**48 questions** built only from what Challenge 45 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-45.md`**.

:::danger Read this before you start

Defender for Cloud DevOps Security is graded on **one distinction**, and almost every wrong option in
this paper ignores it.

**Posture** = **configuration**. Is code scanning switched on? Is the default branch protected? Are
reviewers required? Is a service connection over-permissive? Defender checks these itself.

**Findings** = **code**. An SQL injection, a vulnerable package, a leaked token. **Defender does not
find these.** CodeQL, Dependabot and secret scanning do — Defender **aggregates** what they report.

So when a question asks "which would Defender automatically detect", the answer is a **setting**, never
a vulnerability.

The second distinction runs through Section G: **Defender annotates; branch protection blocks.** A PR
comment is information. A required status check is a gate. The exam offers the comment as the answer to
"prevent the merge" every single time.

The scenario at line 17 names the actual deliverable: **one pane of glass** across 30 GitHub and 15
Azure DevOps repositories — aggregation and governance, not a new scanner.

:::

---

# Section A — Multiple choice

---

## Q1

Contoso wants findings from GitHub and Azure DevOps repositories in a single dashboard. What should
they configure?

- A. Export alerts from each platform to a shared inbox
- B. Connect both platforms to Microsoft Defender for Cloud with DevOps security connectors
- C. Mirror all code to one platform and scan there
- D. Use a third-party SIEM to aggregate alerts

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-45.md`:** lines **38–49** and **65–77**.

```text
# 1. Microsoft Defender for Cloud > Environment settings
# 2. Add environment > GitHub
```

**Native connectors on both sides**, with consistent severity ratings and remediation guidance in one
inventory (lines 165–167).

**Why C is the workaround teams build before they know connectors exist.** Two copies of every
repository, alerts raised against the mirror rather than where people work, and no PR annotations on
the real pull requests — the same objection as Challenge 44 Q1.

**Why D is not wrong so much as premature.** A SIEM aggregates *alerts*; it does not perform **posture
assessment**, and it gives you no governance rules, no Azure Policy integration and no correlation with
cloud resource posture (line 427).

</details>

---

## Q2

After connecting GitHub, which finding would **Defender** automatically detect?

- A. An SQL injection vulnerability in application code
- B. A repository without branch protection on its default branch
- C. An expired SSL certificate on a web server
- D. A misconfigured network security group on an Azure VM

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-45.md`:** lines **136–144**.

```text
| Branch protection missing | Default branch unprotected | GitHub, Azure DevOps |
```

**Posture management inspects configuration, not code.** Every row in that table is a setting: code
scanning not enabled, secret scanning not enabled, Dependabot not enabled, no required reviewers,
excessive permissions, stale repository access.

**Why A is the trap this whole challenge is built around.** An SQL injection is found by **CodeQL**.
Defender's contribution is noticing that CodeQL was **never switched on** — which is a different and,
across 45 repositories, more valuable observation.

**Why C and D are Defender for Cloud's other job.** Cloud security posture management covers Azure
resources. **DevOps** security posture covers repositories. Same product, different plane.

</details>

---

## Q3

Contoso wants critical findings in a pull request to **block the merge** in GitHub. What achieves that?

- A. Configure Defender PR annotations with "Block" behaviour
- B. A required status check running `microsoft/security-devops-action` that exits non-zero on critical
  findings
- C. Branch protection requiring Defender approval
- D. Configure Dependabot to block merges

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-45.md`:** lines **183** and **232–236**.

```text
4. Annotation behavior: Comment only (do not block merge)
```

**Line 183 is the answer written out in the challenge itself.** Defender annotations comment. They do
not gate.

**Blocking is always the same two-part pattern:** a job that **fails** on the condition you care about,
plus **branch protection** that lists that job as a **required status check**. The failing job alone
does nothing; the required check with a passing job does nothing.

**Why C is the phrasing that sounds authoritative and does not exist.** Branch protection requires
**status checks** and **reviewers**. There is no "Defender approval" concept.

</details>

---

## Q4

What is the primary benefit of connecting Azure DevOps to Defender for Cloud, compared with GHAzDO
alone?

- A. Defender provides CodeQL scanning that GHAzDO does not
- B. Defender gives a unified cross-platform view and correlates DevOps findings with cloud posture
- C. Defender is free while GHAzDO is licensed
- D. Defender scans faster

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-45.md`:** line **17** and lines **283–301**.

**GHAzDO scans; Defender aggregates and governs.** The additions are cross-platform visibility,
governance rules with owners and SLAs (lines 287–296), Azure Policy integration (lines 267–280), and
correlation between a repository finding and the cloud resources it deploys to.

**Why A is exactly backwards.** GHAzDO **is** CodeQL inside Azure DevOps (Challenge 44 Q1). Defender
adds no scanner of its own for code.

**That inversion is the single most useful sentence for this challenge:** Defender is not a better
scanner, it is the layer **above** the scanners.

</details>

---

## Q5

The GitHub connector shows "Disconnected" and no new findings arrive. What is the most likely cause?

- A. The Azure subscription was suspended
- B. The Microsoft Defender for Cloud GitHub App was uninstalled or its authorisation revoked
- C. The repositories have no findings
- D. Auto-discovery is disabled

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-45.md`:** lines **323–325**.

**The connector is a GitHub App installation** (line 47), so it fails the way any app installation fails
— someone removed it, or an admin revoked consent.

**The fix is Reauthorize plus reinstall if needed** (lines 343–346), and the verification step at line
346 is the one worth remembering: check **Organization Settings > Installed GitHub Apps** on the GitHub
side. A connector can look wrong in Azure when the actual change happened in GitHub.

**Why D would produce a different symptom.** With auto-discovery off, **existing** repositories keep
reporting and only **new** ones are missed — a gap, not a disconnection.

</details>

---

## Q6

`MicrosoftSecurityDevOps@1` runs successfully but no PR annotations appear. What is the cause?

- A. The pipeline triggers on push to `main` rather than on pull requests
- B. The task requires a paid licence
- C. Annotations only work in GitHub
- D. The agent is Microsoft-hosted

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-45.md`:** lines **352–354** and **362–367**.

```yaml
trigger: none  # Do not run on push
pr:
  branches:
    include:
      - main
```

**A pull request annotation needs a pull request run.** A pipeline triggered by `trigger:` on `main`
executes *after* the merge — there is nothing to annotate.

**And the second half of the fix is easy to miss** (lines 374–378): the results under `.gdn` must be
**published** for annotations to appear.

```yaml
  - task: PublishBuildArtifacts@1
    inputs:
      pathToPublish: '$(System.DefaultWorkingDirectory)/.gdn'
```

**Note `trigger: none` explicitly.** In Azure Pipelines, omitting `trigger:` means "trigger on every
push to every branch" — the opposite of what a PR-only pipeline wants.

</details>

---

## Q7

Which categories can `MicrosoftSecurityDevOps@1` scan?

- A. `code,artifacts,IaC,containers`
- B. `code` only
- C. `dependencies,secrets`
- D. `infrastructure,network`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-45.md`:** lines **204–207**.

```yaml
    inputs:
      categories: 'code,artifacts,IaC,containers'
      # Tools included: Bandit, BinSkim, ESlint, Template Analyzer,
      # Terrascan, Trivy, AntiMalware
```

**It is a wrapper around several open-source scanners**, not a scanner itself — which is the theme of
the whole challenge.

**Worth knowing which tool serves which category**, because the exam names them: Bandit (Python code),
ESLint (JavaScript code), BinSkim (compiled artifacts), Template Analyzer and Terrascan (IaC), Trivy
(containers), AntiMalware (artifacts).

**And note what is *not* in the list: CodeQL.** Microsoft Security DevOps complements GHAS; it does not
replace it.

</details>

---

## Q8

Which permission does the Defender GitHub Actions workflow need to publish SARIF?

- A. `contents: write`
- B. `security-events: write`
- C. `packages: write`
- D. `actions: write`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-45.md`:** lines **221–223**.

```yaml
permissions:
  contents: read
  security-events: write
  id-token: write
```

**`security-events: write` is what allows an upload into the Security tab**, exactly as in Challenge 44.

**And `id-token: write` is here for a different reason** — OIDC authentication to Azure, so the workflow
can report into Defender without a stored credential. Two write scopes, two purposes.

</details>

---

## Q9

What does `${{ steps.msdo.outputs.sarifFile }}` reference?

- A. A file path the Defender action published as a step output
- B. A GitHub secret
- C. A repository variable
- D. An artifact from a previous workflow

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-45.md`:** lines **232–241**.

```yaml
        uses: microsoft/security-devops-action@v1
        id: msdo
        with:
          categories: 'code,artifacts,IaC,containers'

      - name: Upload results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: ${{ steps.msdo.outputs.sarifFile }}
```

**The `id: msdo` is what makes the reference resolvable.** Remove it and the expression evaluates to
empty, the upload silently receives no file, and nothing appears in the Security tab — with a green
workflow.

**And the pattern is the same one from Challenge 44 Q4:** any scanner that emits SARIF publishes through
`codeql-action/upload-sarif`.

</details>

---

## Q10

Which Defender feature auto-assigns findings with an owner and a remediation deadline?

- A. Governance rules
- B. Auto-discovery
- C. PR annotations
- D. Security connectors

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-45.md`:** lines **287–296**.

```text
   - Conditions: Severity = Critical
   - Owner: security-team@contoso.com
   - Remediation timeframe: 7 days
   - Grace period: 3 days
```

**This is what turns a dashboard into a process.** A finding with no owner and no date is a finding
nobody closes — the failure mode Challenge 44's scenario documented over six months.

**The grace period is worth understanding rather than memorising.** Findings become **overdue** at 7
days, and the 3-day grace period is how long before they count against the compliance metric — so a
team gets a warning window before the number turns red.

</details>

---

## Q11

What does auto-discovery do on a connector?

- A. Automatically scans newly created repositories without manual onboarding
- B. Discovers Azure resources referenced in pipelines
- C. Finds secrets in commit history
- D. Detects unused service connections

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-45.md`:** lines **128–131**.

```text
# - Auto-discovery: On (new repos automatically scanned)
# - Scanning frequency: Every 24 hours (default)
```

**Across 45 repositories with new ones appearing regularly, this is the difference between a coverage
programme and a one-off audit.** Without it, every new repository is invisible until somebody remembers
to add it.

**Note the 24-hour scan cadence**, because it sets expectations: enable code scanning on a repository
now and the posture recommendation clears within a day, not immediately.

</details>

---

## Q12

Which posture check applies to **Azure DevOps** but not GitHub in the challenge's table?

- A. Excessive permissions on service connections
- B. Branch protection missing
- C. No required reviewers
- D. Code scanning not enabled

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-45.md`:** line **142**.

```text
| Excessive permissions | Over-permissive service connections | Azure DevOps |
```

**Service connections are an Azure DevOps concept**, so the check has no GitHub equivalent — and this
is Challenge 41's `Azure-All` finding, surfaced automatically instead of by audit.

**The mirror-image rows are worth noting too.** Secret scanning, Dependabot and inactive-repository
checks are listed as **GitHub** only (lines 139, 140, 144), while branch protection, required reviewers
and code scanning span both.

</details>

---

## Q13

Which severity threshold does the challenge configure for GitHub PR annotations?

- A. All severities
- B. High and Critical
- C. Critical only
- D. Medium and above

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-45.md`:** line **182**.

```text
3. Severity threshold: High and Critical (ignore Medium/Low in PRs)
```

**The parenthesis explains the reasoning: a pull request is a scarce channel.** Annotate everything and
developers stop reading; annotate what is genuinely worth interrupting for, and they keep reading.

**Medium and Low still exist** — they are in the dashboard, tracked and triaged. They just do not
interrupt a code review.

</details>

---

## Q14

Which command lists security recommendations originating from DevOps sources?

- A. `az security assessment list --query "[?contains(resourceDetails.source, 'DevOps')]"`
- B. `az security alert list`
- C. `az devops security permission list`
- D. `az policy state list`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-45.md`:** lines **125–126** and **150–151**.

```bash
az security assessment list \
  --query "[?contains(resourceDetails.source, 'DevOps')].{name:displayName, status:status.code, resource:resourceDetails.id}" -o table
```

**Assessments are *posture* — the recommendations.** `az security alert list` (line 159) returns
**alerts**, which are detections of active threat activity.

**That distinction is worth carrying in.** A recommendation says "this repository has no branch
protection". An alert says "something happened". The exam uses both nouns precisely.

</details>

---

## Q15

What does the ARM connector template's `offeringType` value indicate?

- A. The billing tier
- B. Which Defender capability the connector provides — here, CSPM monitoring for Azure DevOps
- C. The region
- D. The number of repositories

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-45.md`:** lines **108–112**.

```json
        "offerings": [
          {
            "offeringType": "CspmMonitorAzureDevOps"
          }
        ]
```

**CSPM = Cloud Security Posture Management**, and the name tells you what the connector does: monitor
**posture**. Not scan code.

**And `hierarchyIdentifier` at line 107 is the organisation's ID** — the connector is bound to one
Azure DevOps organisation, the same way the GitHub connector is bound to one GitHub organisation.

</details>

---

## Q16

Why deploy the connector through an ARM template rather than the portal?

- A. It is the only supported method
- B. It makes connector creation repeatable, reviewable and deployable through a pipeline
- C. It is faster to click through the portal
- D. The portal cannot create Azure DevOps connectors

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-45.md`:** lines **83–87**.

```bash
az deployment group create \
  --resource-group rg-contoso-defender-devops \
  --template-file defender-devops-connector.json \
  --parameters organizationName=contoso
```

**The security configuration itself becomes infrastructure as code** — reviewed in a pull request,
version-controlled, and redeployable identically into another subscription.

**Why A and D are both false**, and the challenge shows the portal path first (lines 65–77). The
template is the *repeatable* option, not the *only* one — and the exam likes offering "only supported
method" when the honest answer is "better practice".

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** posture checks does Defender perform on GitHub repositories? (Choose three.)

- A. Code scanning not enabled
- B. Secret scanning not enabled
- C. Branch protection missing on the default branch
- D. SQL injection in application code
- E. An unpatched Azure VM
- F. An expired TLS certificate

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-45.md`:** lines **138–141**.

**Every posture check is a *setting*.** Read the table's own description column: "Repos without CodeQL",
"Repos without secret scanning", "Default branch unprotected".

**Why D is the trap the entire challenge turns on.** Defender does not analyse code. It notices that the
thing that *would* analyse code is switched off.

**Why E and F belong to Defender for Cloud's other half** — cloud resource posture, not DevOps posture.

</details>

---

## Q18

Which **three** does Defender for Cloud add on top of GHAzDO? (Choose three.)

- A. A unified view across GitHub, Azure DevOps and cloud resources
- B. Governance rules with owners and remediation timeframes
- C. Azure Policy integration for DevOps governance
- D. CodeQL analysis
- E. Faster scanning
- F. Free licensing

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-45.md`:** lines **17**, **287–296**, **267–280**.

**Aggregation, accountability, and enforcement.** Those three words cover everything Defender adds.

**Why D is the inversion worth stating out loud.** GHAzDO **is** CodeQL in Azure DevOps. Defender
consumes what GHAzDO produces; it does not duplicate it.

</details>

---

## Q19

Which **two** are required for PR annotations to appear in Azure DevOps? (Choose two.)

- A. The pipeline triggers on pull requests
- B. Scan results under `.gdn` are published
- C. The pipeline runs on a self-hosted agent
- D. The repository has branch protection
- E. Defender's GitHub App is installed

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-45.md`:** lines **362–378**.

```yaml
trigger: none  # Do not run on push
pr:
  branches:
    include:
      - main
```

**Run on the pull request, then publish the results.** Missing either produces the same symptom — a
successful pipeline and a silent pull request.

**Why E is the cross-platform confusion the exam plants.** The GitHub App connects **GitHub**. Azure
DevOps uses the **Microsoft Security DevOps extension** (line 189) and its own connector consent (line
77).

</details>

---

## Q20

Which **two** does a governance rule define? (Choose two.)

- A. An owner responsible for remediation
- B. A remediation timeframe and grace period
- C. Which scanners run
- D. The severity of the underlying finding
- E. Whether the pull request is blocked

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-45.md`:** lines **288–296**.

**A governance rule assigns and schedules; it does not scan or gate.** Severity is a property of the
finding, and the rule **filters on** it (line 292) rather than setting it.

**Why E is the recurring Section G error.** Blocking is branch protection plus a failing required check
(Q3). Nothing in Defender's governance model touches a merge.

</details>

---

## Q21

Which **two** tools does `MicrosoftSecurityDevOps@1` include for IaC scanning? (Choose two.)

- A. Template Analyzer
- B. Terrascan
- C. CodeQL
- D. Dependabot
- E. BinSkim

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-45.md`:** lines **206–207**.

```text
      # Tools included: Bandit, BinSkim, ESlint, Template Analyzer,
      # Terrascan, Trivy, AntiMalware
```

**Template Analyzer handles ARM and Bicep; Terrascan handles Terraform.** Two IaC formats, two tools —
which is why the `IaC` category covers both.

**Why E is the near-miss** — BinSkim analyses **compiled binaries**, so it belongs to the `artifacts`
category.

</details>

---

## Q22

Which **two** verify a connector's health? (Choose two.)

- A. `az security security-connector list --query "[?environmentName=='GitHub']"`
- B. `az security security-connector show --name contoso-github-connector --query "properties.environmentData"`
- C. `az devops project list`
- D. `az policy assignment list`
- E. `gh api repos/contoso/webapp --jq '.security_and_analysis'`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-45.md`:** lines **52–59** and **330–334**.

```bash
az security security-connector show \
  --name contoso-github-connector \
  --resource-group rg-contoso-defender-devops \
  --query "properties.environmentData"
```

**List confirms the connector exists; show reports its state** — which is the diagnostic path for Break
scenario 1.

**Why E is the useful adjacent check that answers a different question.** It reports whether **GHAS** is
enabled on a repository (Challenge 44 line 51), not whether **Defender** is connected. A repository can
have GHAS on and no connector, or a healthy connector and GHAS off — and the second is exactly what a
posture recommendation would flag.

</details>

---

## Q23

Which **two** are true about Defender email notifications as configured? (Choose two.)

- A. They can be limited to High and Critical severity
- B. Frequency can differ by severity — real-time for Critical, daily digest for High
- C. They replace the need for an action group
- D. They notify on every severity by default
- E. They can only be sent to subscription owners

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-45.md`:** lines **258–262**.

```text
# - Notification types: High and Critical severity
# - Frequency: Real-time for Critical, Daily digest for High
```

**Two channels for two urgencies.** Critical interrupts; High accumulates into a digest that gets read
once a day. Sending everything in real time trains people to filter the sender.

**Why C misses that they solve different problems.** The action group (lines 250–255) drives
**webhooks** — a Teams channel, an incident tool — alongside email. Notifications configure who Defender
emails directly.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso's CISO needs one aggregate risk view across 30 GitHub and 15 Azure DevOps
repositories, consistent policy enforcement, findings that reach developers, and accountability for
remediating critical issues.

---

## Q24

**Proposed solution:** Create Defender for Cloud security connectors for the GitHub organisation and the
Azure DevOps organisation with auto-discovery on. Enable PR annotations at High and Critical on both.
Add `microsoft/security-devops-action` to GitHub workflows and `MicrosoftSecurityDevOps@1` to PR-triggered
pipelines, publishing SARIF and `.gdn` results. Create a governance rule assigning Critical findings to
the security team with a 7-day remediation timeframe. Assign Azure Policy for required code scanning
and secret scanning. Configure email notifications at High and Critical.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-45.md`:** lines **38–49**, **65–77**, **128–130**, **180–189**, **232–241**, **287–296**,
**267–280**, **258–262**.

| Requirement | Mechanism |
|---|---|
| One aggregate view | Two connectors, one Defender inventory |
| New repositories covered | Auto-discovery |
| Findings reach developers | PR annotations, High and Critical |
| Consistent policy | Azure Policy assignments |
| Accountability | Governance rule: owner, timeframe, grace period |
| Escalation | Email notifications by severity |

**Note what this design does *not* claim to do: block merges.** Nothing in the requirements asked for
it, and adding it would need the separate required-status-check pattern from Q3.

</details>

---

## Q25

**Proposed solution:** Enable GHAS on all GitHub repositories and GHAzDO on all Azure DevOps
repositories. Have each team review its own platform dashboard weekly. Email findings to the CISO
monthly.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures, and the first is the scenario's own sentence.**

**Two dashboards is the problem, not the solution.** Line 17 says it directly: each platform has its own
dashboard, "making it impossible to get an aggregate risk view".

**Per-team weekly review gives no consistency.** Each team applies its own severity judgement, its own
triage, and its own definition of done — so the aggregate number, if anyone computed one, would not mean
anything.

**Nothing enforces policy.** A repository with code scanning switched off is invisible to this design,
because a scanner that is not running produces no findings to review. **That absence is precisely what
posture management detects** (Q2).

**And a monthly email to the CISO is a report, not accountability.** No owner, no deadline, no
tracking — the governance rule at lines 287–296 exists because reports do not close findings.

</details>

---

## Q26

**Proposed solution:** Create connectors for both platforms with auto-discovery. Enable PR annotations at
High and Critical. Add the Microsoft Security DevOps scans to PR-triggered pipelines with results
published. Assign Azure Policy for code scanning and secret scanning. Configure email notifications.
Rely on PR annotations to block merges containing critical findings, so no separate gate is needed.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical
apart from having a governance rule rather than a blocking claim.

**PR annotations do not block anything.** Line 183 states the configured behaviour explicitly:

```text
4. Annotation behavior: Comment only (do not block merge)
```

**A comment on a pull request is information delivered at the right moment.** It is read by whoever is
looking, and merged past by whoever is in a hurry.

**Blocking requires the two-part pattern:** a job that exits non-zero on critical findings, and branch
protection listing that job as a **required status check**. The gate is enforced by the *platform's
merge rules*, not by the scanner's opinion.

**And the design has now also lost its accountability layer**, because it substituted a blocking claim
for the governance rule. Nothing owns the finding and nothing has a deadline.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — what Defender does

| # | Statement | Answer |
|---|---|---|
| 1 | Defender detects repositories without branch protection |  |
| 2 | Defender finds SQL injection in application code |  |
| 3 | Defender aggregates findings from GHAS and GHAzDO |  |
| 4 | Defender replaces the need for CodeQL |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Defender detects repositories without branch protection | **Yes** |
| 2 | Defender finds SQL injection in application code | **No** |
| 3 | Defender aggregates findings from GHAS and GHAzDO | **Yes** |
| 4 | Defender replaces the need for CodeQL | **No** |

**In `challenge-45.md`:** lines **141**, **405**, **17**, **421**.

Rows 2 and 4 are the same fact from two directions. **Defender is the layer above the scanners**, and
its unique value is noticing which scanners are missing.

</details>

---

## Q28 — connectors

| # | Statement | Answer |
|---|---|---|
| 1 | The GitHub connector installs a GitHub App |  |
| 2 | Auto-discovery scans newly created repositories |  |
| 3 | A connector can be deployed with an ARM template |  |
| 4 | Removing the GitHub App leaves the connector healthy |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | The GitHub connector installs a GitHub App | **Yes** |
| 2 | Auto-discovery scans newly created repositories | **Yes** |
| 3 | A connector can be deployed with an ARM template | **Yes** |
| 4 | Removing the GitHub App leaves the connector healthy | **No** |

**In `challenge-45.md`:** lines **47**, **130**, **92–116**, **325**.

Row 4 is Break scenario 1, and the diagnostic instinct it teaches: **when an Azure connector looks
wrong, check the other platform.** The cause is usually a change made in GitHub.

</details>

---

## Q29 — PR annotations

| # | Statement | Answer |
|---|---|---|
| 1 | Annotations can be limited to High and Critical |  |
| 2 | Annotations block the merge |  |
| 3 | Azure DevOps annotations need a PR-triggered pipeline |  |
| 4 | Results must be published for annotations to appear |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Annotations can be limited to High and Critical | **Yes** |
| 2 | Annotations block the merge | **No** |
| 3 | Azure DevOps annotations need a PR-triggered pipeline | **Yes** |
| 4 | Results must be published for annotations to appear | **Yes** |

**In `challenge-45.md`:** lines **182**, **183**, **364–367**, **374–378**.

Row 2 is the highest-yield single fact in this challenge, and the exam tests it in at least two shapes
per case study.

Rows 3 and 4 together are Break scenario 2 — one symptom, two possible causes.

</details>

---

## Q30 — governance and policy

| # | Statement | Answer |
|---|---|---|
| 1 | A governance rule assigns an owner and a remediation timeframe |  |
| 2 | A governance rule changes a finding's severity |  |
| 3 | Azure Policy can require code scanning on repositories |  |
| 4 | Assessments and alerts are the same object |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A governance rule assigns an owner and a remediation timeframe | **Yes** |
| 2 | A governance rule changes a finding's severity | **No** |
| 3 | Azure Policy can require code scanning on repositories | **Yes** |
| 4 | Assessments and alerts are the same object | **No** |

**In `challenge-45.md`:** lines **288–296**, **292**, **268–273**, **150–160**.

Row 2: the rule **filters on** severity. A severity override is a separate manual action on the finding
itself (line 172).

Row 4 is the vocabulary the exam uses precisely. **Assessment = a posture recommendation. Alert = a
detection of activity.** Different commands, different meanings.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each capability to the product that provides it.

| Capability | Product |
|---|---|
| Finding an SQL injection in source code |  |
| Reporting that a repository has no code scanning |  |
| Blocking a push containing a secret |  |
| One dashboard across GitHub, Azure DevOps and Azure |  |
| Assigning a critical finding an owner and a 7-day deadline |  |
| Requiring code scanning across an organisation |  |

**Options:** Azure Policy · CodeQL (GHAS / GHAzDO) · Defender DevOps posture · Defender for Cloud · Defender governance rule · GitHub push protection

<details>
<summary>Show answer</summary>

| Capability | Product |
|---|---|
| Finding an SQL injection in source code | **CodeQL (GHAS / GHAzDO)** |
| Reporting that a repository has no code scanning | **Defender DevOps posture** |
| Blocking a push containing a secret | **GitHub push protection** |
| One dashboard across GitHub, Azure DevOps and Azure | **Defender for Cloud** |
| Assigning a critical finding an owner and a 7-day deadline | **Defender governance rule** |
| Requiring code scanning across an organisation | **Azure Policy** |

**In `challenge-45.md`:** lines **421**, **138**, Challenge 44's line 37, line **17**, lines **288–296**,
lines **268–273**.

**Read the first two rows together and the whole challenge falls into place.** CodeQL finds the flaw;
Defender notices CodeQL was never enabled. Neither does the other's job.

</details>

---

## Q32

Match each Microsoft Security DevOps category to its tools.

| Category | Tools |
|---|---|
| `code` |  |
| `artifacts` |  |
| `IaC` |  |
| `containers` |  |

**Options:** Bandit, ESLint · BinSkim, AntiMalware · Template Analyzer, Terrascan · Trivy

<details>
<summary>Show answer</summary>

| Category | Tools |
|---|---|
| `code` | **Bandit, ESLint** |
| `artifacts` | **BinSkim, AntiMalware** |
| `IaC` | **Template Analyzer, Terrascan** |
| `containers` | **Trivy** |

**In `challenge-45.md`:** lines **205–207**.

**Template Analyzer reads ARM and Bicep; Terrascan reads Terraform** — which is why one category needs
two tools.

**And note again what is absent: CodeQL.** Microsoft Security DevOps runs alongside GHAS, not instead of
it.

</details>

---

## Q33

Arrange the steps to unify security visibility across both platforms.

**Items:** Create a governance rule for critical findings · Register the `Microsoft.Security` provider ·
Enable PR annotations on both connectors · Create the GitHub connector and authorise the app · Create
the Azure DevOps connector and grant consent · Create a resource group for the connectors

<details>
<summary>Show answer</summary>

### Answer

1. Register the `Microsoft.Security` provider — line **33**
2. Create a resource group for the connectors — line **36**
3. Create the GitHub connector and authorise the app — lines **40–49**
4. Create the Azure DevOps connector and grant consent — lines **67–77**
5. Enable PR annotations on both connectors — lines **180–189**
6. Create a governance rule for critical findings — lines **287–296**

**The last two steps are the order that matters.** Annotations put findings **in front of developers**;
the governance rule makes someone **accountable** for the ones that are not fixed in a pull request.
Doing governance first means assigning owners to findings nobody can see yet.

**And step 1 is the one people skip until a deployment fails** with a resource-provider error.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Connector shows "Disconnected" |  |
| Azure DevOps PR shows no annotations |  |
| New repositories are never scanned |  |
| SARIF upload produces no alerts |  |
| A repository reports no findings at all |  |

**Options:** Auto-discovery disabled · GitHub App uninstalled or authorisation revoked · `id:` missing on the scan step, so the path is empty · No scanner is enabled — a posture finding, not a clean repo · Pipeline not PR-triggered, or `.gdn` not published

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| Connector shows "Disconnected" | **GitHub App uninstalled or authorisation revoked** |
| Azure DevOps PR shows no annotations | **Pipeline not PR-triggered, or `.gdn` not published** |
| New repositories are never scanned | **Auto-discovery disabled** |
| SARIF upload produces no alerts | **`id:` missing on the scan step, so the path is empty** |
| A repository reports no findings at all | **No scanner is enabled — a posture finding, not a clean repo** |

**In `challenge-45.md`:** lines **325**, **354**, **130**, **234–241**, **138**.

**The last row is the mindset the exam rewards.** "No findings" and "nothing scanning" look identical on
a dashboard, and only posture management distinguishes them.

</details>

---

## Q35

Match each control to what it achieves.

| Control | Achieves |
|---|---|
| PR annotation |  |
| Required status check on a failing scan job |  |
| Governance rule |  |
| Azure Policy assignment |  |
| Email notification |  |
| Auto-discovery |  |

**Options:** Assigns ownership and a deadline · Blocks the merge · Enforces a configuration standard · Escalates by severity · Informs the developer in the pull request · Keeps coverage current as repositories are created

<details>
<summary>Show answer</summary>

| Control | Achieves |
|---|---|
| PR annotation | **Informs the developer in the pull request** |
| Required status check on a failing scan job | **Blocks the merge** |
| Governance rule | **Assigns ownership and a deadline** |
| Azure Policy assignment | **Enforces a configuration standard** |
| Email notification | **Escalates by severity** |
| Auto-discovery | **Keeps coverage current as repositories are created** |

**In `challenge-45.md`:** lines **183**, **411**, **288–296**, **268–273**, **258–262**, **130**.

**Rows 1 and 2 are the pair the exam separates in every case study.** Inform, or block. Only one of them
stops a merge, and it is not the one Defender configures.

</details>

---

# Section F — Hot area

---

## Q36

```bash
az provider register --namespace [BLANK 1]

az security security-connector list \
  --query "[?environmentName=='[BLANK 2]']" -o table
```

- **BLANK 1:** `Microsoft.Security` / `Microsoft.DevOps` / `Microsoft.Defender` /
  `Microsoft.OperationalInsights`
- **BLANK 2:** `GitHub` / `GitHubEnterprise` / `Git` / `SourceControl`

<details>
<summary>Show answer</summary>

### Answer: `Microsoft.Security`, `GitHub`

**In `challenge-45.md`:** lines **33** and **52–53**.

**`Microsoft.Security` is the provider behind Defender for Cloud**, which is why every command in this
challenge is `az security ...` rather than `az defender ...`.

**And `environmentName` takes `GitHub` or `AzureDevOps`** (line 81) — the same two values used in the
ARM template at line 103.

</details>

---

## Q37

```json
{
  "type": "Microsoft.Security/[BLANK 1]",
  "properties": {
    "environmentName": "AzureDevOps",
    "[BLANK 2]": "<azdo-org-id>",
    "offerings": [{ "offeringType": "[BLANK 3]" }]
  }
}
```

- **BLANK 1:** `securityConnectors` / `devopsConnectors` / `assessments` / `alerts`
- **BLANK 2:** `hierarchyIdentifier` / `organizationName` / `tenantId` / `scope`
- **BLANK 3:** `CspmMonitorAzureDevOps` / `DefenderForDevOps` / `CodeQLScanning` /
  `CspmMonitorGitHub`

<details>
<summary>Show answer</summary>

### Answer: `securityConnectors`, `hierarchyIdentifier`, `CspmMonitorAzureDevOps`

**In `challenge-45.md`:** lines **98–111**.

**`CspmMonitorGitHub` is the correct distractor** — it is a real offering type, for the *other*
platform. The offering must match the `environmentName`.

**And `hierarchyIdentifier` binds the connector to one organisation**, which is why a second
organisation needs a second connector rather than a setting change.

</details>

---

## Q38

```yaml
trigger: [BLANK 1]
[BLANK 2]:
  branches:
    include:
      - main

steps:
  - task: MicrosoftSecurityDevOps@1
  - task: PublishBuildArtifacts@1
    inputs:
      pathToPublish: '$(System.DefaultWorkingDirectory)/[BLANK 3]'
```

Requirement: PR annotations appear on pull requests targeting `main`.

- **BLANK 1:** `none` / `main` / `- main` / *(omit the line)*
- **BLANK 2:** `pr` / `pull_request` / `schedules` / `resources`
- **BLANK 3:** `.gdn` / `.sarif` / `results` / `TestResults`

<details>
<summary>Show answer</summary>

### Answer: `none`, `pr`, `.gdn`

**In `challenge-45.md`:** lines **363–377**.

**`trigger: none` must be explicit.** Omitting `trigger:` in Azure Pipelines means *trigger on every
push to every branch* — so the pipeline would run twice per change and annotate nothing.

**And `pr:` is the Azure Pipelines keyword**; `pull_request:` is GitHub Actions. Mixing the two
platforms' vocabulary is one of the most reliable ways the exam separates people who have written both.

</details>

---

## Q39

```yaml
      - name: Run Microsoft Security DevOps
        uses: microsoft/security-devops-action@v1
        [BLANK 1]: msdo
        with:
          categories: '[BLANK 2]'

      - uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: ${{ steps.msdo.outputs.[BLANK 3] }}
```

- **BLANK 1:** `id` / `name` / `key` / `step-id`
- **BLANK 2:** `code,artifacts,IaC,containers` / `all` / `security` / `code`
- **BLANK 3:** `sarifFile` / `output` / `results` / `file`

<details>
<summary>Show answer</summary>

### Answer: `id`, `code,artifacts,IaC,containers`, `sarifFile`

**In `challenge-45.md`:** lines **232–241**.

**`id:` — not `name:` — is what makes `steps.msdo.*` resolvable.** `name:` is the display label; `id:` is
the reference handle. Swap them and the expression evaluates to empty, the upload receives no file, and
the workflow still reports success.

**That silent-empty failure mode is worth internalising**, because it is the same shape as Challenge
44's missing-build problem: green tick, no findings.

</details>

---

## Q40

```text
Governance rule
  Conditions: Severity = [BLANK 1]
  Owner: security-team@contoso.com
  Remediation timeframe: [BLANK 2] days
  Grace period: [BLANK 3] days
```

- **BLANK 1:** `Critical` / `Low` / `Informational` / `All`
- **BLANK 2:** `7` / `90` / `365` / `1`
- **BLANK 3:** `3` / `30` / `0` / `14`

<details>
<summary>Show answer</summary>

### Answer: `Critical`, `7`, `3`

**In `challenge-45.md`:** lines **292–295**.

**Understand what the two numbers do rather than memorising them.** The **timeframe** is when the
finding becomes overdue. The **grace period** is how long after that before it counts against the
compliance score — a warning window, so a team sees the deadline pass before the metric turns red.

**A grace period of 0 makes overdue and non-compliant simultaneous**, which is defensible for Critical
and unforgiving in practice.

</details>

---

## Q41

```bash
az monitor action-group create \
  --name ag-security-critical \
  --short-name SecCritical \
  --action [BLANK 1] security-team security-team@contoso.com \
  --action [BLANK 2] security-webhook "https://contoso.webhook.office.com/webhookb2/..."
```

- **BLANK 1:** `email` / `sms` / `voice` / `azureapp`
- **BLANK 2:** `webhook` / `logicapp` / `function` / `automationrunbook`

<details>
<summary>Show answer</summary>

### Answer: `email`, `webhook`

**In `challenge-45.md`:** lines **250–255**.

**Two channels for two audiences.** Email reaches individuals; the webhook posts into a Teams channel
where the whole security team sees it and can act together.

**And the action group is reusable** — the same group can be attached to metric alerts, activity log
alerts and Defender alerts, so notification routing is configured once rather than per alert.

</details>

---

# Section G — Case study

## Case study: Contoso unified security posture

### Background

Contoso Ltd's **CISO wants a single pane of glass** for all security findings across **30 GitHub
repositories and 15 Azure DevOps repositories**. Each platform currently has its own security
dashboard, **making an aggregate risk view impossible** and preventing consistent policy enforcement.

### Requirements

**Visibility**

- All findings from both platforms must appear in one inventory with consistent severity
- Newly created repositories must be covered without manual onboarding
- The CISO must be able to report on posture trends over time

**Developer experience**

- High and Critical findings must appear in pull requests
- Medium and Low findings must not interrupt code review
- Critical findings must **block** the merge

**Governance**

- Critical findings must have a named owner and a remediation deadline
- Code scanning and secret scanning must be required, not optional
- The security team must be notified in real time for Critical findings

---

## Q42

How should Contoso achieve the unified inventory?

- A. Defender for Cloud security connectors for the GitHub organisation and the Azure DevOps
  organisation, with auto-discovery enabled
- B. Export both platforms' alerts to a shared mailbox
- C. Migrate all 15 Azure DevOps repositories to GitHub
- D. Build a custom dashboard from both platforms' APIs

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-45.md`:** lines **38–49**, **65–77**, **128–130**.

**One connector per organisation, and auto-discovery covers the "without manual onboarding"
requirement** — new repositories are scanned within the 24-hour cycle.

**Why C is a large project that solves a reporting problem with a migration**, and it abandons every
Azure Pipelines integration those 15 repositories depend on.

**Why D is what teams build when they do not know connectors exist.** It produces a dashboard and none
of the rest: no posture assessment, no governance rules, no Azure Policy, no PR annotations.

</details>

---

## Q43

Which finding would Defender surface that **neither** platform's own dashboard would?

- A. A repository where code scanning has never been enabled
- B. A cross-site scripting flaw in a React component
- C. A vulnerable transitive npm dependency
- D. An exposed API key in commit history

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-45.md`:** line **138**.

```text
| Code scanning not enabled | Repos without CodeQL or equivalent | GitHub, Azure DevOps |
```

**This is the whole argument for posture management, and it is worth stating carefully.** B, C and D
are findings that **appear on a platform dashboard when scanning is running**. A is what you cannot see
from that dashboard at all — because a repository with no scanner produces **no findings**, and an empty
security tab looks identical to a clean one.

**Across 45 repositories, the gap is invisible without something counting the repositories rather than
the findings.**

</details>

---

## Q44

How should High and Critical findings reach developers without interrupting on Medium and Low?

- A. Enable PR annotations with a severity threshold of High and Critical
- B. Enable PR annotations for all severities and let developers filter
- C. Email findings to the repository owner
- D. Post all findings to a Teams channel

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-45.md`:** lines **181–183**.

```text
3. Severity threshold: High and Critical (ignore Medium/Low in PRs)
```

**The threshold is the requirement, expressed as a setting.**

**Why B fails on the second half of the requirement** and, more practically, on attention. A pull
request with 30 Medium annotations trains reviewers to collapse the bot's comments — after which the
Critical one is invisible too.

**Why C and D route findings away from the pull request**, which is the moment the code is still fresh
and the fix is cheap.

</details>

---

## Q45

Which **two** are needed for Critical findings to **block** the merge? (Choose two.)

- A. A scan job that exits non-zero on critical findings
- B. Branch protection listing that job as a required status check
- C. PR annotations set to "Block" behaviour
- D. A governance rule with a 7-day timeframe
- E. Azure Policy requiring code scanning

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-45.md`:** lines **183** and **411**.

**Two halves, and neither works alone.** A failing job with no required check is a red mark nobody has
to act on; a required check on a job that always passes gates nothing.

**Why C is the option the exam offers every time**, and line 183 is the refutation in the challenge's
own words: *Comment only (do not block merge)*.

**Why D and E are real controls answering different questions.** The governance rule makes someone
accountable **after** the merge; the policy makes scanning mandatory **before** any of this applies.

</details>

---

## Q46

How should Critical findings be made accountable?

- A. A governance rule scoped to all DevOps connectors, condition Severity = Critical, with an owner,
  a 7-day remediation timeframe and a 3-day grace period
- B. A monthly report to the CISO
- C. A Teams channel where findings are posted
- D. Assigning every finding to the repository's most recent committer

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-45.md`:** lines **287–296**.

**Owner plus deadline is what "accountable" means operationally.** Everything else is publication.

**Why D is the automation that feels fair and is not.** The most recent committer may have changed a
README. Ownership belongs with a team that can act — here, the security team, who can triage and route.

**Why B is what Contoso would fall back to**, and the reason the governance rule exists: a report with
no owner and no date describes a problem rather than assigning one.

</details>

---

## Q47

Nine months after rollout, the CISO's posture trend shows a steady improvement, but the security team
notices that **four** repositories have reported **zero findings of any kind since onboarding** —
including zero posture recommendations. The connector reports healthy, and other repositories in the
same organisation report normally.

What is the most likely explanation, and what should be checked first?

- A. Those repositories were excluded when the connector's repository selection was set to specific
  repositories rather than all — check the app installation's repository access
- B. They genuinely have no security issues
- C. Auto-discovery is disabled organisation-wide
- D. The governance rule suppressed their findings

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-45.md`:** line **49** and Break scenario 1's verification at line **346**.

```text
# - Choose repositories: All repositories (or select specific ones)
```

**"Zero posture recommendations" is the diagnostic detail.** A scanned repository with nothing wrong
still produces *assessments* — checks that passed. **Zero of everything means the repository was never
assessed**, which points at scope rather than at health.

**Why C would affect newly created repositories only**, and these four were onboarded. Existing
repositories in a selective installation are the ones missed by scope.

**Why B is the conclusion a dashboard invites and the one to distrust.** This is Q43's lesson turned
into an operational habit: **a clean repository and an unscanned repository look the same from above.**

**What to check first** is the GitHub side — Organization Settings > Installed GitHub Apps > Microsoft
Defender for Cloud > repository access (line 346) — for the same reason Break scenario 1 does: the
Azure object looks healthy because the change was made on the other platform.

</details>

---

## Q48

At the annual audit, Contoso is asked to demonstrate that its security posture is improving and that
critical findings are being remediated within policy.

What can Contoso produce, and what does this illustrate?

- A. A Workbook trending `SecurityRecommendation` over time, the exported posture assessment data, and
  governance-rule compliance showing critical findings closed within the 7-day timeframe
- B. Screenshots of the GitHub and Azure DevOps security tabs
- C. The count of open alerts today
- D. The list of enabled scanners

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-45.md`:** lines **300–316**.

```kusto
SecurityRecommendation
| where RecommendationName contains "DevOps" or RecommendationName contains "GitHub" or RecommendationName contains "Azure DevOps"
| summarize count() by RecommendationName, RecommendationState, bin(TimeGenerated, 1d)
| render timechart
```

**Three artifacts for three different questions.** The Workbook shows the **trend**; the exported
assessment JSON (lines 304–306) is the **evidence** an auditor can inspect; governance-rule compliance
proves findings were closed **within the agreed time**, not merely closed.

**Why C is the number a dashboard shows and the least useful of the four.** A count today says nothing
about direction. A hundred open findings falling week on week is a healthy programme; ten rising is not.

**What it illustrates: security posture is a time series, not a state.** The CISO's original request at
line 17 was for an aggregate risk view, and "aggregate" implies both **across platforms** and **across
time**.

**And the deepest point of the whole challenge sits underneath that.** Contoso started with two
dashboards showing findings from repositories that had scanning enabled. What it could never see was
the repositories that did not — so the risk picture was not merely fragmented, it was **systematically
optimistic**. Posture management measures the coverage, and only once you are measuring coverage does a
trend line mean anything at all.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Defender expected to find code vulnerabilities** | Q2, Q17, Q27, Q43 | Posture = configuration. Code is CodeQL's job |
| **PR annotations expected to block merges** | Q3, Q20, Q26, Q29, Q45 | *Comment only (do not block merge)*, line 183 |
| **Defender assumed to replace GHAzDO** | Q4, Q18, Q27 | GHAzDO scans. Defender aggregates and governs |
| **Pipeline triggered on push, not PR** | Q6, Q19, Q38 | No pull request, nothing to annotate |
| **`.gdn` results not published** | Q6, Q19 | Annotations need published results |
| **`name:` instead of `id:` on the scan step** | Q39 | `steps.msdo.*` needs `id:`. Silent empty upload |
| **Omitting `trigger:` instead of `trigger: none`** | Q38 | Omission means every push to every branch |
| **`pull_request:` in Azure Pipelines** | Q38 | `pr:` there. `pull_request:` is Actions |
| **"No findings" read as "no problems"** | Q34, Q43, Q47 | An unscanned repo looks exactly like a clean one |
| **Governance rule assumed to gate or set severity** | Q20, Q30 | It assigns an owner and a deadline. It filters on severity |
| **Assessment and alert used interchangeably** | Q14, Q30 | Assessment = recommendation. Alert = detection |
| **Wrong offering type for the platform** | Q37 | `CspmMonitorGitHub` vs `CspmMonitorAzureDevOps` |
| **Two dashboards proposed as the fix** | Q25 | That is the problem statement at line 17 |
| **Auto-discovery blamed for missing existing repos** | Q47 | Auto-discovery covers *new* repos. Scope covers existing ones |

---

# What to memorise

**In `challenge-45.md`:** lines **136–144**, **180–189**, **204–207**, **287–296**.

```text
THE ONE DISTINCTION
  POSTURE   = configuration    Defender checks it itself
             "code scanning not enabled" | "branch protection missing" | "no required reviewers"
             "secret scanning not enabled" | "Dependabot not enabled" | "excessive permissions"
             "inactive repos with access"
  FINDINGS  = code             Defender only AGGREGATES these
             SQL injection -> CodeQL | vulnerable package -> Dependabot | leaked token -> secret scanning

  Asked "what would Defender automatically detect?" -> the answer is a SETTING, never a vulnerability.

PLATFORM SPLIT in the posture table
  GitHub + Azure DevOps : code scanning | branch protection | required reviewers
  GitHub only           : secret scanning | Dependabot | inactive repos
  Azure DevOps only     : excessive permissions on SERVICE CONNECTIONS

DEFENDER ADDS (over GHAS/GHAzDO):  unified view | governance rules | Azure Policy | cloud correlation
DEFENDER DOES NOT ADD:             a code scanner. GHAzDO *is* CodeQL in Azure DevOps
```

```text
INFORM vs BLOCK   (asked in every case study)
  PR annotation                 -> comments.  "Comment only (do not block merge)"   line 183
  BLOCK a merge                 -> BOTH of:
       1. a scan job that EXITS NON-ZERO on critical findings
       2. branch protection listing that job as a REQUIRED STATUS CHECK
  Governance rule               -> owner + deadline, AFTER the merge
  Azure Policy                  -> makes the scanner mandatory in the first place
```

```bash
# Connectors                                        (lines 33-59, 92-116)
az provider register --namespace Microsoft.Security
az security security-connector list --query "[?environmentName=='GitHub']" -o table
az security security-connector show --name <c> --resource-group <rg> --query "properties.environmentData"
# ARM: type Microsoft.Security/securityConnectors
#      environmentName  GitHub | AzureDevOps
#      hierarchyIdentifier  <org id>        offeringType  CspmMonitorAzureDevOps | CspmMonitorGitHub
# GitHub connector = a GitHub App installation -> uninstalled/revoked = "Disconnected"
# Auto-discovery ON -> new repos scanned.  Scan cadence: every 24h
# Existing repos missing -> the installation's REPOSITORY SELECTION, not auto-discovery

# Posture vs activity                               (lines 125, 159)
az security assessment list --query "[?contains(resourceDetails.source,'DevOps')]"   # RECOMMENDATIONS
az security alert list --query "[?alertType=='DevOps']"                              # DETECTIONS
az security assessment list ... -o json > devops-security-posture.json               # audit evidence
```

```yaml
# Azure Pipelines - PR annotations                  (lines 202-207, 362-378)
trigger: none          # MUST be explicit. omitting it = every push to every branch
pr:                    # 'pr:' here. 'pull_request:' is GitHub Actions
  branches: {include: [main]}
steps:
  - task: MicrosoftSecurityDevOps@1
    inputs:
      categories: 'code,artifacts,IaC,containers'
      # code: Bandit, ESLint | artifacts: BinSkim, AntiMalware
      # IaC: Template Analyzer (ARM/Bicep), Terrascan (Terraform) | containers: Trivy
      # NOT CodeQL - this complements GHAS, it does not replace it
  - task: PublishBuildArtifacts@1
    inputs: {pathToPublish: '$(System.DefaultWorkingDirectory)/.gdn'}   # required for annotations

# GitHub Actions                                    (lines 221-241)
permissions:
  contents: read
  security-events: write      # publish SARIF
  id-token: write             # OIDC to Azure
steps:
  - uses: microsoft/security-devops-action@v1
    id: msdo                  # 'id:' NOT 'name:' - or steps.msdo.* is empty and the upload is silent
  - uses: github/codeql-action/upload-sarif@v3
    with: {sarif_file: "${{ steps.msdo.outputs.sarifFile }}"}
```

```text
GOVERNANCE + NOTIFICATION                           (lines 258-296)
Governance rule   scope: all DevOps connectors | condition: Severity = Critical
                  owner: security-team | remediation timeframe: 7 days | grace period: 3 days
                  timeframe = when it goes OVERDUE.  grace = warning window before it counts non-compliant
PR annotations    severity threshold: High and Critical  (Medium/Low stay in the dashboard)
Notifications     High + Critical | real-time for Critical, daily digest for High
Action group      --action email <name> <addr>   --action webhook <name> <url>
Azure Policy      "repositories should have code scanning / secret scanning enabled"
Workbook          SecurityRecommendation | summarize count() by ..., bin(TimeGenerated, 1d) | render timechart
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Domain 4 is complete. Move to Challenge 46 |
| 38–43 | Re-read the trap index and the posture/findings split, then move on |
| 30–37 | Rewrite the posture table and the inform-vs-block rule from memory, then retake |
| Below 30 | Redo Tasks 3, 5 and 7 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 45.

:::danger The two questions

**Is this a setting or a vulnerability?** Defender detects settings. CodeQL, Dependabot and secret
scanning detect vulnerabilities. Defender's unique contribution is noticing the scanner was never
switched on.

**Does this inform, or does it block?** An annotation comments. A required status check on a failing
job blocks. The exam offers the annotation as the answer to "prevent the merge" every time.

:::
