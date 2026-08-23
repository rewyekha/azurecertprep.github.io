---
sidebar_position: 5.5
toc_max_heading_level: 2
title: "Challenge 05: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 05 — AZ-400 exam questions

**48 questions** built only from what Challenge 05 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-05.md`**.

:::danger Read this before you start

Documentation questions are graded on **one property: how does this stay true?**

Rank the options by staleness risk before you read them properly.

**Generated from the source** — an OpenAPI spec built from code annotations, a changelog built from
commit history. Cannot drift, because there is nothing to update separately.
**Docs-as-code in the repository** — Markdown and Mermaid beside the code, reviewed in the same pull
request. Drifts only if a reviewer lets it.
**A wiki** — versioned, and separate from the code. Drifts quietly.
**A document on a file share** — the Visio diagram and the SharePoint Word file at line 22. Already
wrong.

**Mermaid is on this exam because it is text.** A diagram that is text can be diffed in a pull request,
so a reviewer can see that the architecture changed and the picture did not. A PNG cannot.

And one platform detail that is pure exam material: **GitHub renders Mermaid inside triple backticks;
Azure DevOps Wiki uses `::: mermaid`** (line 809).

The scenario at line 22 is four different staleness failures at once — two weeks of questions per new
starter, release notes written **from memory**, API docs in Word, and diagrams **three years** out of
date.

:::

---

# Section A — Multiple choice

---

## Q1

What is the key difference between a provisioned wiki and a code wiki in Azure DevOps?

- A. Provisioned wikis support Markdown; code wikis do not
- B. A provisioned wiki is managed internally by Azure DevOps; a code wiki is published from a repository
  folder and follows standard Git workflows
- C. Code wikis are read-only
- D. Provisioned wikis require a paid licence

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-05.md`:** lines **110–117**.

```text
| Storage | Azure DevOps internal Git repo | Your repository |
| PR reviews for changes | Not built-in | Standard PR process |
| Access control | Wiki permissions | Repository permissions |
```

**The row that matters most is "PR reviews for changes".** A code wiki's edits go through the same
review as code, which is what keeps documentation honest — a reviewer sees the doc change beside the
code change.

**And the trade-off is real, not one-sided.** A provisioned wiki is far easier for a non-engineer to
edit: no clone, no branch, no PR. **Team knowledge base → provisioned. Technical docs beside code →
code wiki** (line 117).

**Why A is refuted by both columns** — both are Markdown.

</details>

---

## Q2

What is the advantage of Mermaid over an image-based diagram?

- A. Better visual quality
- B. It is text, so it can be version-controlled, diffed in pull requests, and edited without external
  tools
- C. It supports more shapes than Visio
- D. It renders faster

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-05.md`:** lines **123** and **881**.

```text
They are ideal for keeping diagrams in sync with code because they are text-based and diffable.
```

**"Diffable" is the whole answer.** When the architecture changes, the diagram's diff shows exactly what
changed — and a reviewer who sees the code change without a diagram change can ask why.

**Why A is honestly false.** A hand-drawn Visio diagram usually looks better. Mermaid trades polish for
**truth over time**, which is the trade the scenario at line 22 is making after three years of Visio
drift.

**Why C is a claim nobody needs to defend.** Mermaid has fewer shapes; that is not the argument.

</details>

---

## Q3

How does release-drafter decide which category a pull request belongs to?

- A. It analyses the code changes
- B. It matches PR **labels** against the `categories` configuration
- C. It reads the PR title prefix
- D. It uses the assigned milestone

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-05.md`:** lines **284–307**.

```yaml
categories:
  - title: 'New features'
    labels:
      - 'enhancement'
      - 'feature'
  - title: 'Bug fixes'
    labels:
      - 'bug'
      - 'bugfix'
```

**Labels in, sections out.** Which is why Break scenario 2 exists at line 820: PRs with no labels
produce an empty release note.

**Why C is the tempting answer for anyone who just did Challenge 03.** Conventional Commit prefixes
drive `conventional-changelog` (Task 4); **labels** drive release-drafter (Task 3). **Two tools, two
inputs, and the exam pairs them to see whether you know which is which.**

**Note the catch-all at lines 305–307**: a category matching `'*'` collects everything unlabelled, which
is the defence against the empty-notes failure.

</details>

---

## Q4

What does `conventional-changelog` do?

- A. It enforces commit message format at commit time
- B. It generates a structured `CHANGELOG.md` by parsing Conventional Commit messages from Git history
- C. It validates that commits reference work items
- D. It creates GitHub releases

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-05.md`:** lines **403–404** and **472–495**.

```bash
npx conventional-changelog -p angular -i CHANGELOG.md -s -r 0
```

```markdown
### Features
* **auth:** implement SSO login with Microsoft Entra ID (a1b2c3d)
### Bug Fixes
* **payments:** correct decimal precision in currency conversion (f0a1b2c)
### Breaking Changes
* **api:** change authentication endpoint response format (e9f0a1b)
```

**Read the output: the sections are the commit *types* from Challenge 03.** `feat` becomes Features,
`fix` becomes Bug Fixes, and a `!` or `BREAKING CHANGE` footer becomes Breaking Changes.

**Why A is the deliberate pairing.** Enforcing the format at commit time is **commitlint** (Challenge
03, line 112). `conventional-changelog` is the *consumer* of that format — which is why the convention
was worth enforcing in the first place.

</details>

---

## Q5

Mermaid diagrams do not render in an Azure DevOps Wiki page. What is the likely cause?

- A. Azure DevOps Wiki uses `::: mermaid` rather than triple backticks
- B. Mermaid is not supported in Azure DevOps
- C. The wiki must be a code wiki
- D. Diagrams must be uploaded as images

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **809–816**.

```text
Azure DevOps uses `::: mermaid` syntax (not triple backticks) to render diagrams in wiki pages.
```

```text
    ::: mermaid
    flowchart TD
        A --> B
    :::
```

**The same Mermaid source, two different fences** — and this is exactly the kind of platform detail the
exam likes, because it only bites people who have used both.

**And the second half of that fix is worth carrying** (line 809): Azure DevOps Mermaid support **lags
behind GitHub's**, so a newer diagram type that renders on GitHub may not render in the wiki. Test in
the wiki editor rather than assuming parity.

</details>

---

## Q6

Release-drafter produces an empty draft although PRs are merging. What is the cause?

- A. PRs have no labels matching any configured category
- B. The workflow lacks `contents: write`
- C. The tag template is wrong
- D. Releases are disabled

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **827–836**.

```bash
gh pr list --state merged --limit 10 --json number,labels \
  --jq '.[] | {pr: .number, labels: [.labels[].name]}'
```

```text
**Fix:** Release-drafter groups PRs by label. If PRs have no labels, they fall into "Other changes"
(if configured) or are omitted. Enable autolabeler in `.github/release-drafter.yml` or manually label
PRs.
```

**The diagnosis is one command: list the labels actually on recent PRs.** If that array is empty, you
have found it.

**And the structural fix is the `autolabeler` block** (lines 325–340), which applies labels
automatically from the branch name and the changed files — so the release notes stop depending on
anyone remembering to label.

**Why B would fail the job**, not empty it. This job succeeds and produces nothing.

</details>

---

## Q7

What does the `autolabeler` section apply labels from?

- A. Changed file paths and branch names
- B. Commit message types
- C. The PR author
- D. The milestone

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **325–340**.

```yaml
autolabeler:
  - label: 'documentation'
    files:
      - '*.md'
      - 'docs/**'
  - label: 'bug'
    branch:
      - '/^bugfix\//'
      - '/^fix\//'
```

**Two signals the developer produces anyway** — where they branched from and what they touched — turned
into the labels release-drafter needs.

**Which is the pattern this whole challenge teaches: derive the metadata from the work rather than
asking for it.** Branch naming is already enforced in Challenge 01 (line 385 there), so `bugfix/` and
`feature/` prefixes are guaranteed to exist.

**Why B is the near-miss again.** Commit types drive `conventional-changelog`; branch and file paths
drive the autolabeler (Q3).

</details>

---

## Q8

What does `version-resolver` in the release-drafter config control?

- A. Which label causes a major, minor or patch version bump
- B. The Node.js version
- C. The release-drafter action version
- D. The changelog format

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **309–323**.

```yaml
version-resolver:
  major:
    labels:
      - 'breaking-change'
  minor:
    labels:
      - 'enhancement'
      - 'feature'
  patch:
    labels:
      - 'bug'
      - 'bugfix'
      - 'documentation'
      - 'dependencies'
  default: patch
```

**This is SemVer derived from labels**, which is the same idea as deriving it from commit types
(Challenge 03, lines 52–61) through a different input.

**`default: patch` is the safety net** — an unlabelled PR still moves the version, conservatively. And
it feeds `$RESOLVED_VERSION` in the name and tag templates at lines 273–274.

</details>

---

## Q9

What does `--generate-notes` do on `gh release create`?

- A. GitHub builds release notes automatically from merged pull requests
- B. It generates a changelog from commit messages
- C. It creates an empty notes file
- D. It publishes to GitHub Pages

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **380–384**.

```bash
# GitHub can auto-generate notes from merged PRs
gh release create v2.4.0 \
  --title "v2.4.0 - Authentication Overhaul" \
  --generate-notes \
  --target main
```

**This is the zero-configuration option, and knowing it exists is worth marks.** It needs no
release-drafter, no config file and no labels — GitHub lists the PRs merged since the last release.

**The trade is control.** Release-drafter gives you categories, version resolution and exclusions;
`--generate-notes` gives you a flat list. **Choose by whether anyone reads the notes for structure.**

**And `--notes-file` at line 389 is the third option**: fully hand-written, for the release where the
narrative matters more than the list.

</details>

---

## Q10

Which trigger does the changelog workflow use?

- A. `push` on tags matching `v*`
- B. `push` to `main`
- C. `pull_request`
- D. `schedule`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **429–432**.

```yaml
on:
  push:
    tags:
      - 'v*'
```

**A changelog is a release artifact, so it is generated when a release is tagged** — not on every merge,
which would rewrite the file dozens of times a week.

**And note `fetch-depth: 0` at line 443.** `conventional-changelog` reads the **entire commit history**
to group by type; a shallow clone gives it one commit and an empty changelog — the same trap as
Challenge 03 Q4.

</details>

---

## Q11

Why does the changelog commit step end with `|| true`?

- A. So the job does not fail when there is nothing to commit
- B. To ignore push errors
- C. To skip the commit entirely
- D. To force the commit

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** line **461**.

```bash
          git commit -m "docs: update changelog for ${{ github.ref_name }}" || true
          git push origin HEAD:main
```

**`git commit` exits non-zero when there is nothing staged**, which would fail the job on any tag whose
changelog output happened to be identical.

**A pattern worth recognising, and worth being slightly wary of.** `|| true` also swallows *real*
failures — a stricter form is `git diff --staged --quiet || git commit ...`, which commits only when
there is something to commit rather than ignoring the error either way.

</details>

---

## Q12

Where does the OpenAPI specification come from in Task 5?

- A. JSDoc `@openapi` annotations in the route files, assembled by `swagger-jsdoc`
- B. A hand-written `openapi.yaml`
- C. Generated at runtime by the API
- D. Exported from Postman

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **511–546** and **632**.

```javascript
/**
 * @openapi
 * /api/users:
 *   get:
 *     summary: List all users
 */
router.get('/', async (req, res) => {
```

```javascript
  apis: ['./src/api/*.js']
```

**The annotation sits directly above the route it documents**, which is what makes it likely to be
updated when the route changes — the two are three lines apart in the same file.

**That is rung one of the staleness ladder**: generated from the source. Compare the scenario's Word
documents on SharePoint (line 22), which have no relationship to the code at all.

**And `apis:` is the glob that tells `swagger-jsdoc` where to look.** Add a route file outside
`./src/api/` and it is silently undocumented.

</details>

---

## Q13

What triggers the API documentation workflow?

- A. Pushes to `main` that touch `src/api/**` or `openapi.yaml`
- B. Every push to `main`
- C. Tag pushes
- D. Pull requests

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **645–650**.

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'src/api/**'
      - 'openapi.yaml'
```

**A path filter is what keeps documentation current without rebuilding it on every unrelated commit.**
Change the CSS and the docs job does not run; change a route and it does.

**And the path list must match where the annotations live** (Q12). Move the API to `src/routes/` without
updating this filter and the published documentation silently freezes at its last build — which looks
exactly like documentation that is up to date.

</details>

---

## Q14

Which permissions does the API docs workflow need to publish to GitHub Pages?

- A. `pages: write` and `id-token: write`
- B. `contents: write`
- C. `packages: write`
- D. `actions: write`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **652–655**.

```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```

**`pages: write` publishes; `id-token: write` is the OIDC token the deployment uses to prove the
artifact came from this workflow run.**

**And `contents` is deliberately `read`** — a documentation build has no business writing to the
repository.

**This is also the fix in Break scenario 3** (line 854): a Pages 404 with `build_type: workflow` is
usually a missing permission rather than a missing file.

</details>

---

## Q15

How is the HTML documentation generated from the OpenAPI spec?

- A. `npx @redocly/cli build-docs openapi.json --output docs/api/index.html`
- B. `npx swagger-ui`
- C. `npm run docs`
- D. Pages renders the JSON directly

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **681–683**.

```bash
          npx @redocly/cli build-docs openapi.json \
            --output docs/api/index.html \
            --title "Contoso API Reference"
```

**Three steps in one job: assemble the spec from annotations, render it to HTML, upload it as a Pages
artifact** (lines 673–688).

**Why D is worth ruling out.** GitHub Pages serves static files; it does not know what OpenAPI is. The
`build-docs` step is what turns a machine-readable spec into something a human can read.

</details>

---

## Q16

What does the `deploy-docs` job's `environment` block accomplish?

- A. It names the `github-pages` environment and surfaces the published URL on the run
- B. It requires an approval
- C. It sets environment variables
- D. It selects the runner

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **693–695**.

```yaml
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
```

**The `url:` is what puts a clickable link to the published site on the workflow run and the
environment page.**

**And naming an environment is also what *allows* protection rules to apply** — if someone later adds a
reviewer to `github-pages`, this job starts pausing for approval with no YAML change. That is the same
mechanic as Challenge 41: **the YAML names the environment; the rules live on it.**

**Why B is not automatic.** Naming an environment does not create an approval; it creates the place one
could be configured.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** distinguish a code wiki from a provisioned wiki? (Choose three.)

- A. It is stored in your own repository
- B. Changes go through the standard pull request process
- C. Access is governed by repository permissions
- D. It cannot use Markdown
- E. It supports only a single branch
- F. It requires a separate licence

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-05.md`:** lines **112–116**.

```text
| Storage | Azure DevOps internal Git repo | Your repository |
| Branch support | Single branch | Any branch |
| PR reviews for changes | Not built-in | Standard PR process |
| Access control | Wiki permissions | Repository permissions |
```

**E is inverted — the *provisioned* wiki is single-branch** (line 114); a code wiki can publish from any
branch, which is how you keep a `docs` branch for a release line.

**And row C has a consequence people miss.** Repository permissions mean anyone who can read the code
can read the wiki, and anyone who can push can propose a doc change. That is usually what you want for
technical docs and usually **not** what you want for an HR-adjacent knowledge base.

</details>

---

## Q18

Which **three** Mermaid diagram types does the challenge use? (Choose three.)

- A. `flowchart`
- B. `sequenceDiagram`
- C. `graph` with `subgraph`
- D. `gantt`
- E. `pie`
- F. `erDiagram`

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-05.md`:** lines **129**, **149**, **174–201**.

```text
flowchart TD          <- deployment pipeline, with decision nodes
sequenceDiagram       <- authentication flow, participant by participant
graph TB + subgraph   <- system architecture, grouped by tier
```

**Each type answers a different question.** A flowchart answers "what happens next, and what decides";
a sequence diagram answers "who talks to whom, in what order"; a subgraph architecture answers "what
lives where".

**The authentication sequence at lines 155–166 is worth reading as revision** — it is the OIDC flow from
Challenge 39 drawn out: redirect to authorize, auth code back, code exchanged for tokens, token in an
HTTP-only cookie.

</details>

---

## Q19

Which **three** are true of release-drafter? (Choose three.)

- A. It groups PRs into categories by label
- B. It can resolve the next version from labels
- C. It can apply labels automatically from branch names and changed files
- D. It parses Conventional Commit types
- E. It publishes the release automatically
- F. It requires a PAT

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-05.md`:** lines **284–307**, **309–323**, **325–340**.

**Why D is the pairing this challenge sets up twice.** Conventional Commit **types** are parsed by
`conventional-changelog` (Task 4). Release-drafter reads **labels**.

**Why E is a genuinely useful nuance.** Release-drafter maintains a **draft**; a human publishes it.
That is deliberate — the draft accumulates as PRs merge, and someone reviews the notes before they
become public.

**Why F is refuted at line 369** — it runs on `secrets.GITHUB_TOKEN`, with `contents: write` granted at
job level (line 364), because everything it touches is in this repository.

</details>

---

## Q20

Which **two** make documentation resist going stale? (Choose two.)

- A. Generating the API reference from annotations in the route files
- B. Generating the changelog from commit history
- C. Storing diagrams as PNG exports
- D. A quarterly documentation review meeting
- E. A SharePoint folder with an owner

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-05.md`:** lines **511–546** and **403–404**.

**Both are *derived*, so there is nothing to forget to update.** The API reference changes when the
route changes; the changelog changes when commits are written.

**Why D is the control Contoso probably already nominally has**, and why it fails: a quarterly review
depends on someone doing an unrewarded task on a schedule, and the scenario's Visio diagrams are three
years old (line 22) — twelve reviews that did not happen.

**Why C is the specific regression to avoid.** A PNG is text's opposite: not diffable, not searchable,
and requiring the original tool to change.

</details>

---

## Q21

Which **two** are required for the Pages deployment to succeed? (Choose two.)

- A. `pages: write` and `id-token: write` permissions
- B. An uploaded Pages artifact
- C. `contents: write`
- D. A `gh-pages` branch
- E. A paid plan

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-05.md`:** lines **652–655** and **685–688**.

```yaml
      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: docs/api
```

**Two jobs and an artifact between them.** `build-docs` uploads; `deploy-docs` (with `needs:` at line
691) deploys what was uploaded.

**Why D is the older model and a good distractor.** Publishing from a `gh-pages` branch is the classic
approach; this workflow uses `build_type: workflow` (line 718), where nothing is committed to a branch
at all.

</details>

---

## Q22

Which **two** does the incident-response runbook contain? (Choose two.)

- A. A severity table with response times and escalation paths
- B. A Mermaid flowchart of the response steps
- C. An OpenAPI specification
- D. A changelog
- E. A list of contributors

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-05.md`:** lines **764–769** and **773–787**.

```text
| SEV1 | Service down | 15 minutes | VP Engineering |
| SEV2 | Major degradation | 30 minutes | Engineering Manager |
```

**A runbook is documentation that is read under stress**, which is why both halves matter: a table you
can scan in ten seconds, and a diagram that shows the decision points without prose.

**And note the flowchart's structure** (lines 777–786): assess severity, branch on SEV1/SEV2, open an
incident channel and page a commander — then, at the end, **write a post-mortem**. The last node is the
one teams skip, and putting it in the diagram is how it stops being optional.

</details>

---

## Q23

Which **two** problems from the scenario does docs-as-code with automation solve? (Choose two.)

- A. Release notes written from memory
- B. API documentation that nobody updates
- C. Slow build times
- D. Merge conflicts
- E. Missing test coverage

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-05.md`:** line **22**, with **272–344** and **511–546**.

**A is solved by deriving notes from merged PRs**, so the PM writes nothing from memory. **B is solved
by generating the reference from annotations beside the code.**

**The two remaining complaints in that scenario are solved by different tasks in this challenge**: two
weeks of new-starter questions by the onboarding wiki page (lines 71–89), and three-year-old diagrams by
Mermaid in the repository (lines 219–259).

**Why C, D and E belong to other challenges** — 35, 01 and 18 respectively.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must replace ad-hoc documentation with material that lives beside the code,
updates automatically where possible, follows a consistent structure, and does not depend on anyone
remembering to maintain it.

---

## Q24

**Proposed solution:** Use a code wiki published from the repository's `/docs` folder so doc changes go
through pull requests. Draw architecture, pipeline and sequence diagrams in Mermaid and commit them.
Configure release-drafter with categories, a version resolver and an autolabeler, and let it maintain a
draft release. Generate `CHANGELOG.md` from Conventional Commits on tag pushes with full history.
Annotate routes with `@openapi`, build the spec with `swagger-jsdoc`, render it with Redocly, and
publish to GitHub Pages on changes under `src/api/**`, granting `pages: write` and `id-token: write`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-05.md`:** lines **96–106**, **219–259**, **272–344**, **426–463**, **511–546**,
**642–700**.

| Failure at line 22 | Mechanism |
|---|---|
| No standards, two weeks of questions | Structured docs in the repository, reviewed |
| Release notes from memory | Release-drafter over merged PRs |
| API docs in Word, never updated | Generated from annotations, published on change |
| Visio diagrams three years old | Mermaid, diffable in the PR that changes the code |

**The autolabeler is the clause that makes the release notes durable** (Q6). Without it the whole chain
depends on someone labelling every PR, and the first busy week produces an empty release.

</details>

---

## Q25

**Proposed solution:** Create a provisioned wiki and ask each team to keep their section current. Export
the existing Visio diagrams as PNG and attach them to wiki pages. Have the PM continue writing release
notes but store them in the wiki. Publish the API documentation quarterly from an exported Postman
collection.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures, and every one of them keeps a human in the update loop.**

**"Ask each team to keep their section current" is what Contoso does now.** The scenario's problem is not
that nobody was asked.

**PNG exports are strictly worse than the Visio files they came from.** They are still not diffable, and
now the editable original lives somewhere else — so the next change needs Visio *and* an export step.

**Release notes from memory in a wiki are release notes from memory.** Moving the artifact does not
change how it is produced.

**And quarterly API documentation is stale for up to three months by design**, from a Postman collection
that is itself maintained by hand.

**Nothing here is generated from the source**, so every item decays at the rate the scenario already
describes.

</details>

---

## Q26

**Proposed solution:** Use a code wiki published from `/docs` so changes go through pull requests. Commit
Mermaid diagrams. Configure release-drafter with categories, version resolver and autolabeler. Generate
`CHANGELOG.md` from Conventional Commits on tag pushes. Annotate routes with `@openapi`, build with
`swagger-jsdoc`, render with Redocly, publish to Pages with the right permissions. Use the default
shallow checkout in the changelog job, since only the newest commits are needed for the current release.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**The changelog command is `conventional-changelog -p angular -i CHANGELOG.md -s -r 0`** (line 454), and
`-r 0` means **regenerate every release from the entire history**. A shallow clone provides one commit,
so the tool has nothing to parse and writes an empty or near-empty file over the existing one.

**The failure is destructive rather than merely absent.** The job succeeds, commits the truncated
`CHANGELOG.md`, and pushes it to `main` (lines 460–462) — so the run **overwrites** the accumulated
history with nothing.

**And the stated reasoning is wrong on its own terms.** Even generating only the newest release requires
history back to the previous tag, which a depth-1 clone does not have.

**Line 443 sets `fetch-depth: 0` for exactly this reason**, and it is the same trap as Challenge 03 Q26:
**a tool that reads a commit range cannot run on a shallow clone**, and it usually fails quietly rather
than loudly.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — wikis

| # | Statement | Answer |
|---|---|---|
| 1 | A code wiki is published from a folder in a repository |  |
| 2 | A provisioned wiki supports pull request review natively |  |
| 3 | A provisioned wiki is limited to a single branch |  |
| 4 | Code wiki access follows repository permissions |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A code wiki is published from a folder in a repository | **Yes** |
| 2 | A provisioned wiki supports pull request review natively | **No** |
| 3 | A provisioned wiki is limited to a single branch | **Yes** |
| 4 | Code wiki access follows repository permissions | **Yes** |

**In `challenge-05.md`:** lines **94**, **115**, **114**, **116**.

Row 2 is the decisive property. **If you want documentation reviewed with the code, you need a code
wiki.**

</details>

---

## Q28 — Mermaid

| # | Statement | Answer |
|---|---|---|
| 1 | Mermaid diagrams are diffable in pull requests |  |
| 2 | GitHub renders Mermaid inside triple backticks |  |
| 3 | Azure DevOps Wiki uses the same triple-backtick syntax |  |
| 4 | Azure DevOps Mermaid support can lag behind GitHub |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Mermaid diagrams are diffable in pull requests | **Yes** |
| 2 | GitHub renders Mermaid inside triple backticks | **Yes** |
| 3 | Azure DevOps Wiki uses the same triple-backtick syntax | **No** |
| 4 | Azure DevOps Mermaid support can lag behind GitHub | **Yes** |

**In `challenge-05.md`:** lines **123**, **128**, **809**, **809**.

Rows 3 and 4 come from the same sentence in Break scenario 1, and both are the sort of detail that
separates people who have used both platforms.

</details>

---

## Q29 — release notes and changelogs

| # | Statement | Answer |
|---|---|---|
| 1 | Release-drafter categorises by PR label |  |
| 2 | `conventional-changelog` parses commit types |  |
| 3 | Release-drafter publishes the release automatically |  |
| 4 | `--generate-notes` builds notes from merged PRs |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Release-drafter categorises by PR label | **Yes** |
| 2 | `conventional-changelog` parses commit types | **Yes** |
| 3 | Release-drafter publishes the release automatically | **No** |
| 4 | `--generate-notes` builds notes from merged PRs | **Yes** |

**In `challenge-05.md`:** lines **284–307**, **404**, **272–344**, **380–383**.

Rows 1 and 2 are the pairing to keep straight: **labels for release-drafter, commit types for
conventional-changelog.**

Row 3: it maintains a **draft**, which a human publishes.

</details>

---

## Q30 — API docs and Pages

| # | Statement | Answer |
|---|---|---|
| 1 | The OpenAPI spec is assembled from `@openapi` JSDoc annotations |  |
| 2 | The docs workflow runs only when API files change |  |
| 3 | Pages deployment needs `pages: write` and `id-token: write` |  |
| 4 | GitHub Pages renders `openapi.json` without a build step |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | The OpenAPI spec is assembled from `@openapi` JSDoc annotations | **Yes** |
| 2 | The docs workflow runs only when API files change | **Yes** |
| 3 | Pages deployment needs `pages: write` and `id-token: write` | **Yes** |
| 4 | GitHub Pages renders `openapi.json` without a build step | **No** |

**In `challenge-05.md`:** lines **511–546**, **648–650**, **652–655**, **681–683**.

Row 4 is the step people assume away. **Pages serves static files**; Redocly is what turns the spec into
a page.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each documentation need to its mechanism.

| Need | Mechanism |
|---|---|
| Docs reviewed alongside the code change |  |
| A team knowledge base non-engineers edit |  |
| An architecture diagram that cannot drift silently |  |
| Release notes without anyone writing them |  |
| A changelog grouped by change type |  |
| An always-current API reference |  |

**Options:** Code wiki, or `/docs` in the repository · `conventional-changelog` over commit history · Mermaid committed to the repository · `@openapi` annotations → spec → Redocly → Pages · Provisioned wiki · Release-drafter over merged PRs

<details>
<summary>Show answer</summary>

| Need | Mechanism |
|---|---|
| Docs reviewed alongside the code change | **Code wiki, or `/docs` in the repository** |
| A team knowledge base non-engineers edit | **Provisioned wiki** |
| An architecture diagram that cannot drift silently | **Mermaid committed to the repository** |
| Release notes without anyone writing them | **Release-drafter over merged PRs** |
| A changelog grouped by change type | **`conventional-changelog` over commit history** |
| An always-current API reference | **`@openapi` annotations → spec → Redocly → Pages** |

**In `challenge-05.md`:** lines **96–106**, **44–50**, **219–259**, **272–344**, **403–404**,
**511–546** with **642–700**.

**Read the list top to bottom: each row moves further from "someone maintains it" toward "it is
derived".** That direction is the whole challenge.

</details>

---

## Q32

Match each tool to its input.

| Tool | Input |
|---|---|
| Release-drafter |  |
| `conventional-changelog` |  |
| Autolabeler |  |
| `swagger-jsdoc` |  |
| Redocly `build-docs` |  |
| `gh release create --generate-notes` |  |

**Options:** Branch names and changed file paths · Conventional Commit types in Git history · Merged pull requests since the last release · `@openapi` JSDoc annotations in route files · Pull request labels · The generated `openapi.json`

<details>
<summary>Show answer</summary>

| Tool | Input |
|---|---|
| Release-drafter | **Pull request labels** |
| `conventional-changelog` | **Conventional Commit types in Git history** |
| Autolabeler | **Branch names and changed file paths** |
| `swagger-jsdoc` | **`@openapi` JSDoc annotations in route files** |
| Redocly `build-docs` | **The generated `openapi.json`** |
| `gh release create --generate-notes` | **Merged pull requests since the last release** |

**In `challenge-05.md`:** lines **284–307**, **404**, **325–340**, **632**, **681**, **380**.

**Rows 1 and 2 are the pair the exam swaps** (Q3, Q19). Labels feed release-drafter; commit types feed
the changelog.

</details>

---

## Q33

Arrange the API documentation pipeline.

**Items:** Render HTML with Redocly · Deploy to GitHub Pages · Annotate the route with `@openapi` ·
Upload the Pages artifact · Assemble `openapi.json` with `swagger-jsdoc`

<details>
<summary>Show answer</summary>

### Answer

1. Annotate the route with `@openapi` — lines **511–546**
2. Assemble `openapi.json` with `swagger-jsdoc` — lines **671–677**
3. Render HTML with Redocly — lines **681–683**
4. Upload the Pages artifact — lines **685–688**
5. Deploy to GitHub Pages — lines **690–699**

**Step 1 is the only human step, and that is the design.** Everything after it is a transformation, so
the published reference is a function of the code rather than a document with its own lifecycle.

**Steps 4 and 5 are separate jobs joined by `needs:`** (line 691) — build and deploy split so the deploy
job can carry the `github-pages` environment and its URL (Q16).

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Mermaid renders on GitHub, not in the ADO wiki |  |
| The draft release is empty |  |
| The changelog is overwritten with almost nothing |  |
| Pages returns 404 on a workflow build |  |
| A new route is undocumented |  |
| The changelog job fails on a re-tag |  |

**Options:** `git commit` with nothing staged · It sits outside the `apis:` glob or the path filter · Missing `pages: write` / `id-token: write` · PRs carry no labels matching a category · Shallow checkout with `-r 0` · Wiki needs `::: mermaid`, not backticks

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| Mermaid renders on GitHub, not in the ADO wiki | **Wiki needs `::: mermaid`, not backticks** |
| The draft release is empty | **PRs carry no labels matching a category** |
| The changelog is overwritten with almost nothing | **Shallow checkout with `-r 0`** |
| Pages returns 404 on a workflow build | **Missing `pages: write` / `id-token: write`** |
| A new route is undocumented | **It sits outside the `apis:` glob or the path filter** |
| The changelog job fails on a re-tag | **`git commit` with nothing staged** |

**In `challenge-05.md`:** lines **809**, **836**, **443** with **454**, **854**, **632** and **648–650**,
**461**.

**Rows 3 and 5 are the quiet ones.** One destroys existing content while reporting success; the other
publishes documentation that is confidently incomplete.

</details>

---

## Q35

Rank each artifact by how it goes stale.

| Artifact | Staleness |
|---|---|
| OpenAPI reference generated from annotations |  |
| Changelog generated from commit history |  |
| Mermaid diagram in the repository |  |
| Provisioned wiki page |  |
| Word document on SharePoint |  |

**Options:** Already wrong · Cannot drift — derived from commits · Cannot drift — derived from the code · Drifts only if a reviewer allows it · Drifts quietly — separate from the code

<details>
<summary>Show answer</summary>

| Artifact | Staleness |
|---|---|
| OpenAPI reference generated from annotations | **Cannot drift — derived from the code** |
| Changelog generated from commit history | **Cannot drift — derived from commits** |
| Mermaid diagram in the repository | **Drifts only if a reviewer allows it** |
| Provisioned wiki page | **Drifts quietly — separate from the code** |
| Word document on SharePoint | **Already wrong** |

**In `challenge-05.md`:** lines **511–546**, **403–404**, **123**, **42**, **22**.

**This ranking is the answer to almost every "which should Contoso use" question in the paper.** Prefer
derived; then reviewed-with-the-code; then versioned-but-separate; never a file on a share.

</details>

---

# Section F — Hot area

---

## Q36

```bash
az devops wiki create \
  --name "Contoso API Docs" \
  --type [BLANK 1] \
  --repository "contoso-webapp" \
  --[BLANK 2] "/docs" \
  --branch "main"
```

Requirement: publish documentation from a folder in an existing repository so changes go through pull
requests.

- **BLANK 1:** `codeWiki` / `projectWiki` / `gitWiki` / `repoWiki`
- **BLANK 2:** `mapped-path` / `path` / `folder` / `root`

<details>
<summary>Show answer</summary>

### Answer: `codeWiki`, `mapped-path`

**In `challenge-05.md`:** lines **98–105**.

**`projectWiki` is the provisioned type** (line 48) — Azure DevOps stores it in an internal repository
you do not control, so there is no pull request to review.

**`--mapped-path` is what makes it a *folder* publish** rather than a whole-repository one, which is why
the same repository can hold code and a wiki without the wiki containing `src/`.

</details>

---

## Q37

```yaml
categories:
  - title: 'New features'
    [BLANK 1]:
      - 'enhancement'
      - 'feature'
  - title: 'Other changes'
    [BLANK 1]:
      - '[BLANK 2]'

exclude-labels:
  - '[BLANK 3]'
```

Requirement: every merged PR appears somewhere, except ones explicitly marked to be left out.

- **BLANK 1:** `labels` / `types` / `prefixes` / `branches`
- **BLANK 2:** `*` / `other` / `none` / `misc`
- **BLANK 3:** `skip-changelog` / `wontfix` / `draft` / `internal`

<details>
<summary>Show answer</summary>

### Answer: `labels`, `*`, `skip-changelog`

**In `challenge-05.md`:** lines **286**, **305–307**, **342–343**.

**The `'*'` catch-all is the fix for Break scenario 2** (Q6) — with it, an unlabelled PR lands in "Other
changes" instead of vanishing.

**And `exclude-labels` is the deliberate opposite**: a way to keep something out on purpose, so the
catch-all does not force noise into the notes.

</details>

---

## Q38

```yaml
version-resolver:
  major:
    labels: ['[BLANK 1]']
  minor:
    labels: ['enhancement', 'feature']
  patch:
    labels: ['bug', 'bugfix', 'documentation', 'dependencies']
  default: [BLANK 2]
```

- **BLANK 1:** `breaking-change` / `major` / `breaking` / `incompatible`
- **BLANK 2:** `patch` / `minor` / `major` / `none`

<details>
<summary>Show answer</summary>

### Answer: `breaking-change`, `patch`

**In `challenge-05.md`:** lines **311–323**.

**`default: patch` means an unlabelled PR still advances the version conservatively**, which is the safe
direction — under-bumping a breaking change would be far worse than over-bumping a docs fix.

**Note the label name is `breaking-change` with a hyphen**, and it is a *label*, not a commit footer.
The commit-message equivalent is `BREAKING CHANGE:` (Challenge 03, line 68) — same idea, different
system.

</details>

---

## Q39

```bash
npx conventional-changelog -p [BLANK 1] -i CHANGELOG.md -s -r [BLANK 2]
```

Requirement: regenerate the whole changelog from the full history, writing in place.

- **BLANK 1:** `angular` / `conventional` / `semver` / `github`
- **BLANK 2:** `0` / `1` / `all` / `-1`

<details>
<summary>Show answer</summary>

### Answer: `angular`, `0`

**In `challenge-05.md`:** lines **404** and **454**.

**`-r 0` means "regenerate all releases"**, which is why the job needs `fetch-depth: 0` (Q26). `-r 1`
would produce only the newest release and append it.

**`-p angular` selects the preset that understands Conventional Commits** — the `feat`/`fix`/`perf`
vocabulary from Challenge 03. **`-i` is the input file and `-s` means write it in place.**

</details>

---

## Q40

```yaml
on:
  push:
    branches: [main]
    [BLANK 1]:
      - 'src/api/**'
      - 'openapi.yaml'

permissions:
  contents: read
  [BLANK 2]: write
  [BLANK 3]: write
```

- **BLANK 1:** `paths` / `files` / `include` / `filters`
- **BLANK 2:** `pages` / `contents` / `packages` / `deployments`
- **BLANK 3:** `id-token` / `actions` / `checks` / `issues`

<details>
<summary>Show answer</summary>

### Answer: `paths`, `pages`, `id-token`

**In `challenge-05.md`:** lines **648–655**.

**`contents` stays `read`** — the docs build has no reason to write to the repository, and the
least-privilege habit is what the exam rewards.

**`id-token: write` is easy to omit** because nothing in the YAML obviously asks for a token. It is the
OIDC identity the Pages deployment uses, and its absence is the commonest cause of the 404 in Break
scenario 3 (line 854).

</details>

---

## Q41

```text
[BLANK 1] mermaid
flowchart TD
    A --> B
[BLANK 2]
```

Requirement: render a diagram on an **Azure DevOps Wiki** page.

- **BLANK 1:** `:::` / three backticks / `~~~` / `<mermaid>`
- **BLANK 2:** `:::` / three backticks / `</mermaid>` / *(nothing)*

<details>
<summary>Show answer</summary>

### Answer: `:::` and `:::`

**In `challenge-05.md`:** lines **813–816**.

```text
    ::: mermaid
    flowchart TD
        A --> B
    :::
```

**The diagram source is identical; only the fence differs.** Triple backticks are GitHub's form (line
128) and produce a plain code block in an Azure DevOps wiki — the text renders, the picture does not.

**Which is exactly what Break scenario 1 describes**, and it is a favourite exam detail because both
platforms appear in the same challenge.

</details>

---

# Section G — Case study

## Case study: Contoso documentation standards

### Background

Contoso Ltd has **no documentation standards**. A new developer spends their **first two weeks asking
questions** that onboarding docs should answer. Release notes are **manual emails the PM writes from
memory**. API documentation is **outdated Word documents on SharePoint**. Architecture diagrams were
drawn in **Visio three years ago** and no longer reflect reality. The Engineering Director wants
documentation that **lives alongside code, updates automatically where possible, and follows a
consistent structure**.

### Requirements

**Structure**

- Onboarding material must be editable by non-engineers and easy to find
- Technical documentation must be reviewed in the same pull request as the code it describes
- Diagrams must show what changed when the system changes

**Automation**

- Release notes must be produced without anyone writing them from memory
- A changelog grouped by change type must be generated from history
- The API reference must be regenerated and republished whenever the API changes

**Operations**

- The published reference must be reachable at a stable URL
- Nothing may depend on a person remembering a quarterly task

---

## Q42

Where should the onboarding guide live, and where should the API design notes live?

- A. Onboarding in a provisioned wiki; API notes in a code wiki or `/docs` in the repository
- B. Both in a provisioned wiki
- C. Both in the repository
- D. Both on SharePoint

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **42**, **94** and **117**.

```text
| Best for | Team knowledge base | Technical docs alongside code |
```

**The two requirements pull in opposite directions and the answer honours both.** Onboarding is edited
by people who will not open a pull request; API notes must be reviewed with the code that changes them.

**Why C is the answer an engineer gives and the Director would reject.** Putting the access-request table
(lines 84–88) behind a clone, a branch and a PR means it will not be updated by the people who know when
the approver changes.

</details>

---

## Q43

How should architecture diagrams satisfy "show what changed when the system changes"?

- A. Mermaid committed to the repository, so the diagram diffs in the same pull request
- B. Visio files committed to the repository
- C. PNG exports attached to wiki pages
- D. Diagrams redrawn at each quarterly review

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **123** and **219–259**.

**"Show what changed" is a requirement about *diffs*, and only text diffs.**

**Why B is the near-miss that catches people.** Committing the Visio file gives you version history and
a binary blob — Git can store it and cannot show you what changed inside it, so a reviewer sees "the
diagram changed" and not how.

**Why D is the control that already failed** (line 22): three years, twelve quarterly reviews, no
update.

</details>

---

## Q44

How should release notes be produced?

- A. Release-drafter with categories, a version resolver and an autolabeler, maintaining a draft
- B. The PM continues writing them, stored in the wiki
- C. `--generate-notes` on every release, with no configuration
- D. A weekly email summarising merges

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **272–370**.

**The autolabeler is what removes the last human step** (Q7). Without it the categories depend on
labelling discipline, and Break scenario 2 is the result.

**Why C is a defensible second choice worth understanding.** `--generate-notes` needs no config at all
and produces a flat list of merged PRs. It satisfies "without anyone writing them" — it just gives up
categories, version resolution and exclusions. **If the exam adds "grouped by type", C stops being
sufficient.**

**Why B is the failure being replaced**, relocated.

</details>

---

## Q45

Which **two** keep the API reference current? (Choose two.)

- A. `@openapi` annotations in the route files, assembled by `swagger-jsdoc`
- B. A workflow triggered by pushes touching `src/api/**`
- C. A quarterly export from Postman
- D. A manually maintained `openapi.yaml`
- E. A link to the staging Swagger UI

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-05.md`:** lines **511–546** and **645–650**.

**A ties the documentation to the code; B ties the *publish* to the change.** Either alone leaves a gap:
perfect annotations that are never rebuilt, or a rebuild of a spec nobody updated.

**Why E is genuinely useful and fails the "stable URL" requirement.** A staging Swagger UI is current and
points at an environment that may be down, redeployed or unreachable to the API's consumers.

</details>

---

## Q46

How is the documentation made reachable at a stable URL?

- A. GitHub Pages with `build_type: workflow`, deploying the rendered artifact
- B. A link to the repository folder
- C. An attachment on a wiki page
- D. A shared drive

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **714–727** and **690–699**.

```bash
gh api repos/{owner}/{repo}/pages --jq '{url: .html_url, status: .status, build_type: .build_type}'
```

**Pages gives a fixed public URL that survives every rebuild**, and the environment block at lines
693–695 surfaces it on each run.

**Why B fails for the audience.** A `/docs` folder renders raw Markdown for people with repository
access — which excludes the API consumers the reference exists for.

</details>

---

## Q47

Nine months later, the team notices that `CHANGELOG.md` contains only the two most recent releases,
although it once held two years of history. The changelog job is green on every tag. The platform team
recently trimmed CI checkout times across all workflows.

What happened, and what is the fix?

- A. The changelog job now runs on a shallow clone, so `-r 0` regenerates from one commit and the job
  commits the truncated file over the real one — restore `fetch-depth: 0`
- B. `conventional-changelog` was upgraded
- C. Commits stopped following the convention
- D. The tag pattern no longer matches

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **443**, **454** and **460–462**.

```bash
          npx conventional-changelog -p angular -i CHANGELOG.md -s -r 0
...
          git add CHANGELOG.md
          git commit -m "docs: update changelog for ${{ github.ref_name }}" || true
          git push origin HEAD:main
```

**"Green on every tag" is what makes this dangerous.** The tool runs, finds one commit's worth of
history, writes a short file, and the job **commits and pushes it to `main`** — so each release destroys
a little more of the record while reporting success.

**`-r 0` is the amplifier.** It means *regenerate all releases*, so the output is not appended to what
exists — it **replaces** it.

**Why B and C would leave the file intact.** A parsing failure produces missing *entries*, not a
truncated *history*, and D would mean the job never ran at all.

**The durable lesson: a workflow that writes back to the repository is a workflow whose inputs must be
complete.** Optimising checkout depth is safe for a build and unsafe for anything that reads history —
the same rule as commitlint in Challenge 03.

</details>

---

## Q48

A year on, a new developer is productive in two days, the release notes write themselves, and the API
reference is trusted enough that support links to it.

Explain what each piece contributed, and what actually changed.

- A. The provisioned wiki answered the questions new starters ask; Mermaid in the repository made
  diagrams change with the system; release-drafter derived notes from PRs people were opening anyway;
  `conventional-changelog` turned commit messages into a grouped history; and annotations plus Pages made
  the API reference a function of the code — every artifact is now derived from work that was already
  happening
- B. The team started caring about documentation
- C. A technical writer was hired
- D. Quarterly reviews were made mandatory

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-05.md`:** lines **71–89**, **219–259**, **272–344**, **403–404**, **511–546**,
**642–700**.

**Take the four failures at line 22 in turn.**

*Two weeks of questions* — the onboarding page (lines 74–88) answers the first-week checklist and the
access-request table, which is most of what a new starter actually asks.

*Release notes from memory* — the PM writes nothing. The notes are assembled from pull requests that
were opened anyway, categorised by labels the autolabeler applied from branch names.

*API docs in Word* — the reference is now three transformations away from the route handler, and every
one of them is automatic.

*Three-year-old Visio* — the diagram lives four lines from the pipeline it documents, and changing one
without the other shows up as a diff a reviewer can see.

**What actually changed is not attitude.** The scenario does not describe a team that dislikes
documentation; it describes documentation that had **its own separate lifecycle** — a Word file, a
Visio file, an email — each requiring a deliberate act by a specific person on a schedule nobody owned.

**The graded idea is that documentation survives only when it is a by-product.** Notes derived from
PRs, changelogs derived from commits, references derived from annotations, diagrams reviewed with the
code. **Anything that needs its own maintenance task will be as old as the last person who felt like
doing it** — which, at Contoso, was three years.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Triple backticks in an Azure DevOps wiki** | Q5, Q28, Q41 | `::: mermaid` there |
| **Provisioned wiki assumed to support PR review** | Q1, Q17, Q27 | Not built-in. Code wiki does |
| **Provisioned wiki for docs that change with code** | Q42 | Reviewed-with-the-code needs the repository |
| **Labels vs commit types** | Q3, Q7, Q19, Q29, Q32 | Labels → release-drafter; types → conventional-changelog |
| **Release-drafter assumed to publish** | Q19, Q29 | It maintains a **draft** |
| **Empty release notes blamed on the workflow** | Q6, Q34, Q37 | No matching labels. Add `'*'` and the autolabeler |
| **Shallow clone with `-r 0`** | Q10, Q26, Q34, Q47 | Truncates history and commits it over the real file |
| **`fetch-depth` treated as a CI optimisation everywhere** | Q26, Q47 | Anything reading history needs `0` |
| **PNG or Visio committed as "version-controlled"** | Q20, Q35, Q43 | Stored is not diffable |
| **Pages 404 blamed on missing files** | Q14, Q21, Q30, Q40 | Usually `pages: write` / `id-token: write` |
| **Pages assumed to render `openapi.json`** | Q15, Q30 | It serves static files. Redocly builds the HTML |
| **A route outside the `apis:` glob or path filter** | Q12, Q13, Q34 | Silently undocumented, silently unpublished |
| **`|| true` treated as good practice** | Q11 | It also swallows real failures |
| **Quarterly review as the freshness control** | Q20, Q25, Q43 | Twelve of them produced a three-year-old diagram |

---

# What to memorise

**In `challenge-05.md`:** lines **108–117**, **272–344**, **395–454**, **642–700**.

```text
STALENESS LADDER - rank the options before reading them properly
  1  GENERATED from source     OpenAPI from @openapi annotations | changelog from commits
  2  DOCS-AS-CODE              Markdown + Mermaid in the repo, reviewed in the same PR
  3  WIKI                      versioned, but separate from the code - drifts quietly
  4  FILE ON A SHARE           the Visio/Word in the scenario - already wrong

WIKI TYPES                                          (lines 108-117)
                     Provisioned (projectWiki)        Code wiki (codeWiki)
  storage            ADO internal Git repo            YOUR repository
  branches           single                           any
  PR review          NOT built-in                     standard PR process
  access control     wiki permissions                 repository permissions
  best for           team knowledge base              technical docs beside code
  az devops wiki create --type codeWiki --repository R --mapped-path "/docs" --branch main

MERMAID
  GitHub          ```mermaid ... ```
  ADO Wiki        ::: mermaid ... :::        <- DIFFERENT FENCE. and ADO support LAGS GitHub
  types used      flowchart TD | sequenceDiagram | graph TB + subgraph
  why it wins     TEXT -> diffable in a PR -> a reviewer sees the diagram did NOT change
```

```yaml
# release-drafter - LABELS in, sections out          (lines 272-344)
categories:
  - {title: 'New features', labels: ['enhancement','feature']}
  - {title: 'Other changes', labels: ['*']}       # catch-all, or unlabelled PRs VANISH
version-resolver:
  major: {labels: ['breaking-change']}
  minor: {labels: ['enhancement','feature']}
  patch: {labels: ['bug','bugfix','documentation','dependencies']}
  default: patch
autolabeler:                                       # removes the last human step
  - {label: 'documentation', files: ['*.md','docs/**']}
  - {label: 'bug',           branch: ['/^bugfix\//','/^fix\//']}
exclude-labels: ['skip-changelog']
# it maintains a DRAFT. a human publishes.
# zero-config alternative:  gh release create vX --generate-notes   (flat list, no categories)
```

```bash
# conventional-changelog - COMMIT TYPES in            (lines 395-463)
npx conventional-changelog -p angular -i CHANGELOG.md -s -r 0
#   -p angular  the preset that understands feat/fix/perf
#   -i FILE     input      -s  write in place      -r 0  REGENERATE ALL RELEASES
#   trigger:    on push tags 'v*'          checkout: fetch-depth: 0  (REQUIRED for -r 0)
#   output sections: Features | Bug Fixes | Performance | Breaking Changes
#   git commit ... || true    - no-op when nothing changed (but it hides real errors too)
```

```yaml
# API docs pipeline - one human step, four automatic  (lines 499-700)
@openapi JSDoc above the route
  -> swagger-jsdoc  (apis: ['./src/api/*.js'])  -> openapi.json
  -> npx @redocly/cli build-docs openapi.json --output docs/api/index.html
  -> actions/upload-pages-artifact@v3
  -> actions/deploy-pages@v4

on:
  push:
    branches: [main]
    paths: ['src/api/**','openapi.yaml']    # a route outside this is silently unpublished
permissions:
  contents: read        # a docs build never writes to the repo
  pages: write          # publish
  id-token: write       # OIDC for the deployment - MISSING THIS = the Pages 404
environment:
  name: github-pages
  url: ${{ steps.deployment.outputs.page_url }}    # the clickable link on the run
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 06 |
| 38–43 | Re-read the trap index and the staleness ladder, then move on |
| 30–37 | Rewrite the wiki comparison and the two tool inputs from memory, then retake |
| Below 30 | Redo Tasks 2, 3 and 5 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 05.

:::danger The one question

**How does this stay true?**

Derived from the source beats reviewed with the code, which beats a versioned wiki, which beats a file
on a share. Rank the options that way before you read them carefully.

And two details that are pure marks: **`::: mermaid` in an Azure DevOps wiki**, and **labels feed
release-drafter while commit types feed conventional-changelog**.

Contoso's diagrams were three years old after twelve quarterly reviews. Nothing that needs its own
maintenance task survives.

:::
