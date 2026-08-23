---
sidebar_position: 2.5
toc_max_heading_level: 2
title: "Challenge 02: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 02 — AZ-400 exam questions

**48 questions** built only from what Challenge 02 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-02.md`**.

:::danger Read this before you start

Work-tracking questions are decided by **two distinctions**, and almost every wrong option confuses one
of them.

**Field vs view.** A **field** is data stored on an item — Priority, Sprint, Story Points. A **view** is
a saved way of *looking* at items — board, table, roadmap, with grouping and filters. Adding a field
changes what you know; adding a view changes who can see it usefully.

**Link vs transition.** `AB#1234` on its own creates a **link**. `Fixes AB#1234` creates a link **and
moves the work item** when the PR merges. The keyword is the trigger, not the number.

And one structural fact behind Azure Boards: **`@CurrentIteration` is a macro, not a value.** A query
written with it keeps working next sprint; a query with a hard-coded sprint path is stale the moment
the sprint rolls over.

The scenario at line 21 is what none of this costs: spreadsheets **outdated within hours**, developers
on **sticky notes**, QA learning about features only when code **reaches staging**, and sprint planning
that is **guesswork because nobody has velocity data**.

:::

---

# Section A — Multiple choice

---

## Q1

In GitHub Projects v2, what is the difference between a view and a field?

- A. Views are for admins only; fields are visible to everyone
- B. Fields define the data stored on each item; views define how items are displayed and filtered
- C. Views are permanent; fields can be changed
- D. Fields only apply to issues; views apply to pull requests

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-02.md`:** lines **58–81** and **88–103**.

```bash
gh project field-create $PROJECT_NUMBER --name "Priority" --data-type "SINGLE_SELECT" ...
gh project field-create $PROJECT_NUMBER --name "Sprint" --data-type "ITERATION"
```

```bash
gh project view-create $PROJECT_NUMBER --title "Sprint Board" --layout "board"
gh project view-create $PROJECT_NUMBER --title "Backlog Table" --layout "table"
gh project view-create $PROJECT_NUMBER --title "Roadmap" --layout "roadmap"
```

**The commands are different verbs on purpose: `field-create` stores data, `view-create` presents it.**

**And the three views in this challenge are three audiences** (lines 87, 93, 99): a board for
developers, a table for project managers, a roadmap for leadership — **the same items, three
presentations**. That is the answer to the scenario's "every stakeholder has real-time visibility":
one dataset, not three spreadsheets.

**Why D is worth ruling out explicitly.** Both fields and views apply to issues, pull requests and
draft items alike.

</details>

---

## Q2

Which syntax in a PR description automatically transitions an Azure Boards work item to the
resolved state when the PR merges?

- A. `AB#1234`
- B. `Linked AB#1234`
- C. `Fixes AB#1234`
- D. `Closes AB#1234`

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-02.md`:** lines **438** and **442**.

```text
Fixes AB#1234
Related to AB#1200
```

```text
The `Fixes AB#1234` syntax will transition the work item to the "Done" state when the PR merges.
```

**Why A is the trap and it is the single most-tested fact in this challenge.** `AB#1234` alone creates
a **link** — the work item shows the PR, and its state does not move. Break scenario 2 at line 660 says
it plainly: use the keyword `Fixes`, **not just `AB#`**.

**Why D is the near-miss that catches people who know GitHub.** `Closes #42` is the **GitHub Issues**
keyword (line 415) and it works there. Azure Boards uses `Fixes`.

**Keep the two vocabularies apart:** GitHub Issues take `Fixes`, `Closes`, `Resolves` with a plain `#`;
Azure Boards takes `Fixes` with `AB#`.

</details>

---

## Q3

What is the purpose of the CODEOWNERS file?

- A. It restricts who can clone the repository
- B. It automatically assigns reviewers to PRs based on which files are modified
- C. It defines who can merge pull requests
- D. It configures repository access permissions

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-02.md`:** lines **459–477**.

```text
# Backend team owns API and services
/src/api/ @contoso-org/backend-team

# Security team must review auth changes
/src/auth/ @contoso-org/security-team
```

**Path in, reviewer out.** Touch `/src/auth/` and the security team is requested automatically — the
routing does not depend on the author knowing who owns what.

**Why C is the subtle wrong answer.** CODEOWNERS **requests** a review. It becomes a *merge*
requirement only when branch protection has "require review from Code Owners" enabled (line 687) —
which is Break scenario 3 exactly. **The file assigns; the protection rule enforces.**

**And the last line of that file is the highest-value one.** Routing auth changes to the security team
automatically is how you stop the review being skipped by whoever happened to be free.

</details>

---

## Q4

In an Azure Boards query, what does `@CurrentIteration` do?

- A. Returns work items from all past and current iterations
- B. Resolves dynamically to whichever iteration contains today's date, for the team's schedule
- C. Shows only the next upcoming iteration
- D. Returns the iteration with the most work items

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-02.md`:** line **226**.

```sql
WHERE [System.AreaPath] UNDER 'Contoso Web Platform\Backend'
  AND [System.IterationPath] = @CurrentIteration
  AND [System.State] <> 'Closed'
```

**A macro, not a value — which is what makes the query survive the sprint boundary.** Hard-code
`Sprint 1` and the "current sprint" query is wrong on the first Monday of Sprint 2, silently, and every
dashboard built on it is wrong too.

**"For the team's configured iteration schedule" is the clause worth noticing.** Different teams can
have different iteration paths, so `@CurrentIteration` can resolve differently depending on the team
context the query runs in.

</details>

---

## Q5

New issues are created but never appear on the GitHub Project board. The automation workflow runs.
What is the most likely cause?

- A. The token in `PROJECT_TOKEN` lacks the `project` scope
- B. The project has no views
- C. Issues cannot be added to organisation projects
- D. The repository is private

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **629–637**.

```text
# Verify the PROJECT_TOKEN secret has correct scopes
# The token needs: project (read/write), issues (read)
```

```text
**Fix:** Ensure the Personal Access Token or GitHub App token stored in `PROJECT_TOKEN` has the
`project` scope. Organization projects require `org:read` scope as well.
```

**"The workflow runs" is the diagnostic detail.** The job executes and the API call inside it is
refused — so the run may even be green while nothing is added.

**And this is where Challenge 40's rule bites.** A project is an **organisation-level** object, so the
built-in `GITHUB_TOKEN` cannot reach it — the workflow uses a separate `PROJECT_TOKEN` (line 553) for
exactly that reason. The `org:read` requirement in the fix is the giveaway.

</details>

---

## Q6

`Fixes AB#1234` in a merged PR is not moving the work item. What are the two things to check?

- A. The Azure Boards GitHub App is installed, and the keyword `Fixes` is used
- B. The work item is assigned
- C. The PR was squash-merged
- D. The branch was protected

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **648–660**.

```bash
gh api repos/{owner}/{repo}/installations --jq '.[].app_slug'
# Should include "azure-boards"
```

```text
**Fix:** Ensure the Azure Boards GitHub App is installed on the repository and that the keyword
`Fixes` (not just `AB#`) is used. The connection must be configured in Azure DevOps under
Project Settings > GitHub connections.
```

**Three things really, and the third is the one people forget:** the connection must also exist on the
**Azure DevOps side**, under Project Settings > GitHub connections. Installing the app on GitHub alone
is half a handshake.

**Why C is worth rejecting explicitly.** Squash merging does not break the link — the keyword is read
from the **PR description**, not from individual commit messages.

</details>

---

## Q7

PRs touching `/src/api/` do not request a backend review. CODEOWNERS exists at `.github/CODEOWNERS`.
What should you check?

- A. That branch protection has `require_code_owner_reviews` enabled and the team has repository access
- B. That the file is at the repository root
- C. That the author is on the backend team
- D. That the PR is not a draft

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **671–687**.

```bash
gh api repos/{owner}/{repo}/branches/main/protection/required_pull_request_reviews \
  --jq '.require_code_owner_reviews'
```

```text
**Fix:** Branch protection must have "Require review from Code Owners" enabled. The team referenced
in CODEOWNERS must have at least read access to the repository.
```

**The team-access half is the one that fails silently.** A CODEOWNERS entry naming a team with no
access to the repository is simply ignored — no error, no warning, no review request.

**Why B is a real rule and not the problem here.** Valid locations are `.github/CODEOWNERS`,
`CODEOWNERS` or `docs/CODEOWNERS` (line 676) — and the file is already in the first of those.

</details>

---

## Q8

Which field data type suits a sprint in GitHub Projects v2?

- A. `ITERATION`
- B. `SINGLE_SELECT`
- C. `NUMBER`
- D. `TEXT`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **65–68**.

```bash
gh project field-create $PROJECT_NUMBER \
  --name "Sprint" \
  --data-type "ITERATION"
```

**`ITERATION` is date-aware — it understands sprint boundaries and durations**, which a single-select
list of sprint names does not.

**Why B is what people build first, and what goes wrong.** A single-select "Sprint" field needs a new
option added by hand every fortnight, has no notion of which sprint is current, and cannot drive a
roadmap view.

**The other three field types in this challenge each match their data:** Priority and Team are
`SINGLE_SELECT` (fixed option lists), Story Points is `NUMBER` (it must be summable for velocity).

</details>

---

## Q9

Why is Story Points created as a `NUMBER` field rather than a single-select?

- A. So values can be summed and averaged for velocity
- B. Because single-select has an option limit
- C. Because numbers sort correctly
- D. Because it is required by the roadmap view

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **78–81**.

**The scenario asks for velocity data** (line 21: sprint planning is guesswork because nobody has data
on velocity or remaining work). **Velocity is a sum**, and a text or select field cannot be summed.

**Why C is true and insufficient.** Correct sorting is a side effect; the requirement is arithmetic.

**Note the Azure Boards equivalent** at line 199: `Microsoft.VSTS.Scheduling.StoryPoints=5`. Same
concept, and it is a real numeric field on the User Story type.

</details>

---

## Q10

Which Azure Boards structure separates work by team?

- A. Area paths
- B. Iteration paths
- C. Work item types
- D. Shared queries

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **152–157**.

```bash
az boards area project create --name "Backend"
az boards area project create --name "Frontend"
az boards area project create --name "Platform"
```

**Area = who owns it. Iteration = when it is being done.** Two independent axes, and the exam swaps
them constantly.

**The query at line 226 uses both together**, which is what makes the distinction concrete:

```sql
WHERE [System.AreaPath] UNDER 'Contoso Web Platform\Backend'
  AND [System.IterationPath] = @CurrentIteration
```

**Note `UNDER` on the area path and `=` on the iteration.** `UNDER` picks up child areas, so a nested
`Backend\API` is included.

</details>

---

## Q11

What is the correct work item hierarchy created in Task 2?

- A. Epic → Feature → User Story, linked with the `Parent` relation
- B. Feature → Epic → User Story
- C. User Story → Task → Bug
- D. Epic → User Story only

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **178–217**.

```bash
az boards work-item relation add --id 2 --relation-type "Parent" --target-id 1
az boards work-item relation add --id 3 --relation-type "Parent" --target-id 2
```

**Read the two commands as a chain: item 2's parent is item 1, item 3's parent is item 2.** The Epic
was created first (id 1), then the Feature (id 2), then the User Stories.

**Why the hierarchy is the point rather than a detail.** It is what lets leadership see the Epic's
progress roll up from stories nobody at that level tracks individually — the "real-time visibility for
every stakeholder" the Director asked for.

**This is the Agile process template's hierarchy.** Scrum uses Epic → Feature → Product Backlog Item,
and the prerequisite at line 28 names Agile explicitly.

</details>

---

## Q12

What does `blank_issues_enabled: false` accomplish?

- A. It forces contributors to choose a template instead of opening an empty issue
- B. It disables issues entirely
- C. It hides the issue tab
- D. It requires a label

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **351–352**.

```yaml
blank_issues_enabled: false
contact_links:
  - name: Security vulnerability
    url: https://github.com/contoso-org/contoso-webapp/security/advisories/new
```

**A blank issue is how "the app is broken" gets filed with no severity, no steps and no environment** —
and then someone spends a day getting those details by conversation.

**The `contact_links` beside it are the escape valve**, and the first one matters for security: a
vulnerability should go to a private advisory, **not** a public issue where it is disclosed to everyone
before it is fixed.

</details>

---

## Q13

In the bug report template, what does `validations: required: true` do?

- A. It prevents the issue being submitted with that field empty
- B. It validates the format of the input
- C. It assigns a reviewer
- D. It adds a label

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **267–268**.

```yaml
    validations:
      required: true
```

**Required, not validated — the distinction is in the name and the exam uses it.** GitHub checks the
field is non-empty; it does not check that the steps to reproduce actually reproduce anything.

**Which is why the template shapes the *shape* of the report rather than its quality**: severity
(dropdown, line 258), description, reproduction steps, expected behaviour, environment. That list is
what the QA team at line 21 currently receives none of.

</details>

---

## Q14

Which label scheme does the challenge use, and why?

- A. Prefixed namespaces — `priority/`, `team/`, `status/`
- B. Plain single words
- C. Numeric codes
- D. One label per milestone

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **367–375**.

```bash
gh label create "priority/critical" --color "B60205" --description "Requires immediate attention"
gh label create "team/backend" --color "1D76DB" --description "Backend team"
gh label create "status/blocked" --color "B60205" --description "Blocked by dependency"
```

**The prefix turns a flat list into groups you can filter and reason about** — `label:priority/*` is a
meaningful query, and it is obvious at a glance that an issue is missing a priority.

**And the automation depends on it.** The workflow at lines 565–570 maps `priority/critical` to the
project's Critical field value — a mapping only possible because the labels follow a predictable shape.

</details>

---

## Q15

What triggers the `add-to-project` job?

- A. An issue being opened
- B. Any issue event
- C. A pull request opening
- D. A push to `main`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **539–547**.

```yaml
on:
  issues:
    types: [opened, closed, labeled]
```

```yaml
  add-to-project:
    if: github.event_name == 'issues' && github.event.action == 'opened'
```

**The workflow listens to three issue actions and four PR actions; each job narrows to the one it
wants.** That is the pattern worth learning — a single workflow with several `if:`-gated jobs rather
than four workflow files.

**Why B is the careless read.** `labeled` and `closed` also fire the workflow; they are handled by
`set-priority-on-label` (line 557) and by the project's own built-in automation.

</details>

---

## Q16

Which built-in project automation moves an item to Done?

- A. Pull request merged
- B. Item added to project
- C. Item reopened
- D. Label applied

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **128–131**.

```text
#   - Item added to project -> Set Status to "Backlog"
#   - Pull request merged -> Set Status to "Done"
#   - Item reopened -> Set Status to "In Progress"
```

**These are project *workflows*, configured on the project, not in YAML** — which is why the workflow
file's `close-linked-issues` job at line 604 only logs a message: *"PR merged - project automation will
move items to Done"*.

**The division of labour is the exam-relevant idea.** Built-in automation handles the common status
transitions; the Actions workflow handles what the built-ins cannot, such as setting a custom field
from a label.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** custom fields does the challenge add to the GitHub project? (Choose three.)

- A. Priority, as a single select
- B. Sprint, as an iteration
- C. Story Points, as a number
- D. Area path
- E. Milestone
- F. Assignee

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-02.md`:** lines **58–81**.

**Team is the fourth** (line 71), also a single select. Four fields, three data types, each matched to
what the data does: a fixed list, a date range, a summable number.

**Why D is the Azure Boards vocabulary** (line 152) — GitHub Projects has no area path. **Why E and F
already exist**: milestones and assignees are built-in issue properties, not custom project fields.

</details>

---

## Q18

Which **three** views does the challenge create, and who is each for? (Choose three.)

- A. Board — for developers
- B. Table — for project managers
- C. Roadmap — for leadership
- D. Gantt — for finance
- E. Calendar — for QA
- F. Burndown — for scrum masters

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-02.md`:** lines **87–103**.

```bash
# Create a board view for developers
# Create a table view for project managers
# Create a roadmap view for leadership
```

**Three audiences, one dataset — that is the answer to the scenario's core failure.** The PMs' outdated
spreadsheet (line 21) exists because the PM view and the developer view were different *systems*. Here
they are different *lenses* on the same items, so neither can go stale relative to the other.

**Why D, E and F are plausible and not present.** Board, table and roadmap are the layouts this
challenge uses.

</details>

---

## Q19

Which **three** are true about linking work? (Choose three.)

- A. `Fixes #42` closes a GitHub issue when the PR merges
- B. `AB#1234` creates a link to an Azure Boards work item without transitioning it
- C. `Fixes AB#1234` transitions the work item to Done when the PR merges
- D. `Relates to #38` closes issue 38
- E. `AB#1234` requires no integration to be installed
- F. Commit messages cannot reference work items

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-02.md`:** lines **407**, **431**, **442**.

```text
Fixes #42
Relates to #38
```

```text
AB#1234
```

```text
The `Fixes AB#1234` syntax will transition the work item to the "Done" state when the PR merges.
```

**Why D is the deliberate contrast sitting two lines from A in the source.** `Relates to` links without
closing — it is how you reference context that this PR does *not* resolve.

**Why E is false and is Break scenario 2** (line 660): the Azure Boards GitHub App must be installed
and the connection configured in Azure DevOps.

**Why F is refuted by line 401** — the commit at that line carries `Fixes #42` in its message.

</details>

---

## Q20

Which **two** must be true for CODEOWNERS to produce a required review? (Choose two.)

- A. Branch protection has `require_code_owner_reviews` enabled
- B. The team named has at least read access to the repository
- C. The file is at the repository root only
- D. Every contributor is on a team
- E. The PR is not a draft

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-02.md`:** lines **672–687**.

**A makes it *required*; B makes it *work at all*.** Without A you get a requested review nobody has to
give; without B the entry is ignored entirely.

**Why C is a real rule stated wrongly.** Three locations are valid (line 676): `.github/CODEOWNERS`,
`CODEOWNERS`, or `docs/CODEOWNERS`. "Root only" is false.

**The order to check them in a real incident** is B then A — a silently ignored entry looks identical
to no entry.

</details>

---

## Q21

Which **two** does the Azure DevOps notification subscription filter on? (Choose two.)

- A. Work item type equals Bug
- B. Priority less than or equal to 2
- C. Assigned user
- D. Iteration path
- E. Story points

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-02.md`:** lines **499–515**.

```json
{ "fieldName": "System.WorkItemType", "operator": "=",     "value": "Bug" },
{ "fieldName": "Microsoft.VSTS.Common.Priority", "operator": "<=", "value": "2" },
{ "fieldName": "System.AreaPath", "operator": "Under", "value": "Contoso Web Platform\\Backend" }
```

**Area path is the third clause** and the one that scopes the noise to one team.

**Three clauses is the design, and it is the point.** A subscription on "any bug" would page the
backend team about frontend cosmetic issues; narrowing by **type**, **priority** and **area** is what
makes the notification worth reading — which is the difference between a feedback loop and a mail
filter rule.

</details>

---

## Q22

Which **two** jobs in the automation workflow act on pull requests? (Choose two.)

- A. `move-pr-to-review`
- B. `close-linked-issues`
- C. `add-to-project`
- D. `set-priority-on-label`
- E. All four

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-02.md`:** lines **580** and **597**.

```yaml
    if: github.event_name == 'pull_request' && github.event.action == 'ready_for_review'
```

```yaml
    if: github.event_name == 'pull_request' && github.event.action == 'closed' && github.event.pull_request.merged == true
```

**Note the extra clause on `close-linked-issues`: `merged == true`.** A `closed` pull request event
fires whether the PR was merged **or abandoned** — without that check, closing a PR without merging
would mark its issues Done.

**That is a genuine bug pattern**, and the exam likes it because the condition looks redundant until you
think about the abandoned case.

</details>

---

## Q23

Which **two** problems from the scenario does a unified project with views solve? (Choose two.)

- A. PM spreadsheets outdated within hours
- B. QA learning about features only at staging
- C. Merge conflicts
- D. Broken builds
- E. Slow clone times

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-02.md`:** line **21**, with views at lines **87–103** and automation at **539–547**.

**A is solved by removing the second source of truth.** A table view *is* the PM's spreadsheet, backed
by the same items developers move on the board.

**B is solved by items appearing on the board when the issue opens** (line 547), not when the code
ships. QA sees the work in Backlog, weeks before staging.

**Why C, D and E belong to other challenges** — 01, 08 and 10 respectively. The exam mixes symptoms
across domains and expects you to attribute each to its own mechanism.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must replace spreadsheets and sticky notes with one tracked system, give every
stakeholder a view suited to them, capture enough structure to compute velocity, link code to work
automatically, and route notifications so the right people hear about the right things.

---

## Q24

**Proposed solution:** Create an organisation project with Priority, Sprint, Team and Story Points
fields, and board, table and roadmap views. In Azure Boards, create area paths per team, iteration paths
per sprint, and an Epic/Feature/User Story hierarchy with story points, plus shared queries using
`@CurrentIteration`. Add issue forms with required fields and disable blank issues. Link work with
`Fixes #n` in GitHub and `Fixes AB#n` with the Azure Boards app installed. Add CODEOWNERS and enable
require-code-owner-reviews. Create a notification subscription filtered by work item type, priority and
area path. Automate the board with a workflow using a token that carries the `project` scope.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-02.md`:** lines **58–103**, **152–233**, **247–360**, **407–442**, **459–487**,
**499–515**, **539–605**, **637**.

| Requirement | Mechanism |
|---|---|
| One tracked system | Project items replace the spreadsheet |
| A view per stakeholder | Board, table, roadmap |
| Velocity is computable | Story Points as a `NUMBER`, iteration field |
| Queries survive the sprint boundary | `@CurrentIteration` |
| Reports arrive usable | Issue forms with required fields, blank issues off |
| Code linked to work | `Fixes #n` / `Fixes AB#n` |
| The right reviewer, automatically | CODEOWNERS + require-code-owner-reviews |
| Notifications worth reading | Type + priority + area filter |

**The token clause is the one that makes the automation real** rather than a workflow that runs green
and adds nothing (Q5).

</details>

---

## Q25

**Proposed solution:** Keep the spreadsheets as the master plan but mirror the sprint into a GitHub
project weekly. Track sprints with a single-select field listing sprint names. Use `AB#1234` in PR
descriptions so work items are linked. Subscribe the whole engineering group to all work item changes so
nothing is missed. Add CODEOWNERS.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Five failures, and the first one poisons the rest.**

**A weekly mirror of a spreadsheet is two sources of truth with a scheduled lie between them.** The
scenario's complaint is that spreadsheets go stale *within hours* (line 21); syncing weekly guarantees
the board is wrong for most of the week.

**A single-select Sprint field must be extended by hand every fortnight** and has no notion of the
current sprint (Q8) — so no query can ask "this sprint" and no roadmap can be drawn.

**`AB#1234` links but does not transition** (Q2). The board will show work items sitting in Active long
after the code shipped, which is how it stops being believed.

**Subscribing everyone to everything is alert fatigue by design.** Within a fortnight there is a mail
rule, and the priority-1 backend bug lands in the same folder as 400 routine updates. The subscription
at lines 499–515 filters on **three** clauses for exactly this reason.

**And CODEOWNERS with no branch protection requirement requests reviews nobody must give** (Q3).

</details>

---

## Q26

**Proposed solution:** Create an organisation project with Priority, Sprint, Team and Story Points fields
and board, table and roadmap views. In Azure Boards, create area and iteration paths, the
Epic/Feature/User Story hierarchy with story points, and shared queries using `@CurrentIteration`. Add
issue forms with required fields and disable blank issues. Link work with `Fixes #n` and `Fixes AB#n`
with the app installed. Add CODEOWNERS and enable require-code-owner-reviews. Filter the notification
subscription by type, priority and area. Automate the board with a workflow using the built-in
`GITHUB_TOKEN`, since it needs no setup and is scoped to the repository automatically.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**A GitHub Project at organisation level is not a repository resource**, and the built-in token is
scoped to **this repository only**. It cannot add an item to an organisation project, so
`actions/add-to-project` fails on every issue.

**The failure mode is what makes this the worst option in the paper.** Depending on how the action
handles the error, the workflow may report **green** while adding nothing — and the symptom is Break
scenario 1 at line 618: *"New issues are created but do not show up in the GitHub Project."*

**The reasoning offered is also exactly backwards.** "Scoped to the repository automatically" is
presented as a convenience; it is the **limitation**. Line 637 states the requirement plainly: the token
needs the `project` scope, and organisation projects additionally need `org:read` — which is why the
workflow references `secrets.PROJECT_TOKEN` (line 553) rather than the built-in one.

**This is Challenge 40's rule arriving early:** `GITHUB_TOKEN` cannot leave its repository, and no
permissions block changes that.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — GitHub Projects v2

| # | Statement | Answer |
|---|---|---|
| 1 | Fields store data on items; views control presentation |  |
| 2 | An `ITERATION` field understands sprint dates |  |
| 3 | Story Points must be a single-select to be summable |  |
| 4 | Built-in workflows can set Status to Done when a PR merges |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Fields store data on items; views control presentation | **Yes** |
| 2 | An `ITERATION` field understands sprint dates | **Yes** |
| 3 | Story Points must be a single-select to be summable | **No** |
| 4 | Built-in workflows can set Status to Done when a PR merges | **Yes** |

**In `challenge-02.md`:** lines **58–103**, **67**, **78–81**, **130**.

Row 3 is inverted: a **number** is summable; a select is not (Q9).

Row 4 is the automation you do not have to write — and knowing it exists is what stops you writing a
workflow to duplicate it.

</details>

---

## Q28 — Azure Boards

| # | Statement | Answer |
|---|---|---|
| 1 | Area paths separate work by team |  |
| 2 | Iteration paths separate work by time |  |
| 3 | `@CurrentIteration` must be updated each sprint |  |
| 4 | The `Parent` relation builds the Epic → Feature → Story hierarchy |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Area paths separate work by team | **Yes** |
| 2 | Iteration paths separate work by time | **Yes** |
| 3 | `@CurrentIteration` must be updated each sprint | **No** |
| 4 | The `Parent` relation builds the Epic → Feature → Story hierarchy | **Yes** |

**In `challenge-02.md`:** lines **152–157**, **160–171**, **226**, **209–217**.

Row 3 is the whole reason the macro exists (Q4). A query that needs editing every sprint will not be
edited every sprint.

Rows 1 and 2 are the two axes the exam swaps — **who** versus **when**.

</details>

---

## Q29 — linking and transitions

| # | Statement | Answer |
|---|---|---|
| 1 | `AB#1234` alone transitions the work item |  |
| 2 | `Fixes AB#1234` transitions it on merge |  |
| 3 | The Azure Boards GitHub App must be installed |  |
| 4 | `Closes #42` works for Azure Boards work items |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `AB#1234` alone transitions the work item | **No** |
| 2 | `Fixes AB#1234` transitions it on merge | **Yes** |
| 3 | The Azure Boards GitHub App must be installed | **Yes** |
| 4 | `Closes #42` works for Azure Boards work items | **No** |

**In `challenge-02.md`:** lines **431**, **442**, **660**, **415**.

Rows 1 and 4 are the two halves of the same confusion: **the keyword belongs to the platform**. `Closes`
and `Fixes` with a bare `#` are GitHub Issues; `Fixes` with `AB#` is Azure Boards.

</details>

---

## Q30 — notifications and automation

| # | Statement | Answer |
|---|---|---|
| 1 | A subscription can filter on type, priority and area together |  |
| 2 | The built-in `GITHUB_TOKEN` can add items to an organisation project |  |
| 3 | `close-linked-issues` checks that the PR was actually merged |  |
| 4 | CODEOWNERS alone makes a review mandatory |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A subscription can filter on type, priority and area together | **Yes** |
| 2 | The built-in `GITHUB_TOKEN` can add items to an organisation project | **No** |
| 3 | `close-linked-issues` checks that the PR was actually merged | **Yes** |
| 4 | CODEOWNERS alone makes a review mandatory | **No** |

**In `challenge-02.md`:** lines **499–515**, **637**, **597**, **687**.

Row 3 is the condition that looks redundant and is not (Q22) — a closed-unmerged PR must not mark work
Done.

Row 4 is the recurring separation in this challenge: **the file assigns, the protection rule enforces.**

</details>

---

# Section E — Drag and drop

---

## Q31

Match each requirement to its mechanism.

| Requirement | Mechanism |
|---|---|
| Developers see a kanban of the sprint |  |
| PMs need a sortable, filterable list |  |
| Leadership needs dates and sequencing |  |
| Velocity must be computable |  |
| A query that still works next sprint |  |
| Auth changes always reviewed by security |  |

**Options:** Board view · CODEOWNERS + require-code-owner-reviews · `@CurrentIteration` · Roadmap view · Story Points as a `NUMBER` field · Table view

<details>
<summary>Show answer</summary>

| Requirement | Mechanism |
|---|---|
| Developers see a kanban of the sprint | **Board view** |
| PMs need a sortable, filterable list | **Table view** |
| Leadership needs dates and sequencing | **Roadmap view** |
| Velocity must be computable | **Story Points as a `NUMBER` field** |
| A query that still works next sprint | **`@CurrentIteration`** |
| Auth changes always reviewed by security | **CODEOWNERS + require-code-owner-reviews** |

**In `challenge-02.md`:** lines **88–103**, **78–81**, **226**, **476** with **687**.

**Three views, one dataset.** That is the structural answer to "every stakeholder has real-time
visibility" — not three tools kept in sync, one tool seen three ways.

</details>

---

## Q32

Match each syntax to what it does.

| Syntax | Does |
|---|---|
| `Fixes #42` |  |
| `Relates to #38` |  |
| `AB#1234` |  |
| `Fixes AB#1234` |  |
| `Closes #42` |  |

**Options:** Closes GitHub issue 42 on merge · Links and transitions the work item to Done · Links issue 38 without closing it · Links the Azure Boards work item, no transition

<details>
<summary>Show answer</summary>

| Syntax | Does |
|---|---|
| `Fixes #42` | **Closes GitHub issue 42 on merge** |
| `Relates to #38` | **Links issue 38 without closing it** |
| `AB#1234` | **Links the Azure Boards work item, no transition** |
| `Fixes AB#1234` | **Links and transitions the work item to Done** |
| `Closes #42` | **Closes GitHub issue 42 on merge** |

**In `challenge-02.md`:** lines **407**, **408**, **431**, **438**, **415**.

**The pair to memorise is rows 3 and 4.** Same number, same platform, one word of difference, entirely
different behaviour — and it is the most-tested fact in this challenge.

</details>

---

## Q33

Arrange the steps to make `Fixes AB#1234` actually transition a work item.

**Items:** Write `Fixes AB#1234` in the PR description · Install the Azure Boards GitHub App on the
repository · Configure the GitHub connection in Azure DevOps project settings · Merge the pull request

<details>
<summary>Show answer</summary>

### Answer

1. Install the Azure Boards GitHub App on the repository — line **649**
2. Configure the GitHub connection in Azure DevOps project settings — line **660**
3. Write `Fixes AB#1234` in the PR description — line **438**
4. Merge the pull request — line **442**

**Steps 1 and 2 are two sides of one handshake**, and the exam tests whether you know both exist. The
app grants GitHub's side; the connection under Project Settings > GitHub connections grants Azure
DevOps's side. Do only the first and the syntax sits inert in PR descriptions.

**And the order matters operationally:** write the syntax before the integration exists and the link is
never created retroactively — the PR merges, the work item stays Active, and nobody notices until the
sprint review.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Issues never appear on the project board |  |
| Work item links but never moves |  |
| Backend PRs get no backend review |  |
| The "current sprint" query returns nothing after a sprint rolls |  |
| A closed-but-unmerged PR marks issues Done |  |
| Everyone ignores work item email |  |

**Options:** A hard-coded iteration path · A subscription with no filter clauses · `AB#` used without the `Fixes` keyword · Missing the `merged == true` check · `PROJECT_TOKEN` lacks the `project` scope · `require_code_owner_reviews` off, or team lacks access

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| Issues never appear on the project board | **`PROJECT_TOKEN` lacks the `project` scope** |
| Work item links but never moves | **`AB#` used without the `Fixes` keyword** |
| Backend PRs get no backend review | **`require_code_owner_reviews` off, or team lacks access** |
| The "current sprint" query returns nothing after a sprint rolls | **A hard-coded iteration path** |
| A closed-but-unmerged PR marks issues Done | **Missing the `merged == true` check** |
| Everyone ignores work item email | **A subscription with no filter clauses** |

**In `challenge-02.md`:** lines **637**, **660**, **687**, **226**, **597**, **499–515**.

**Four of these six fail silently.** No error, no red run — just a board that quietly stops reflecting
reality, which is precisely how Contoso ended up trusting spreadsheets instead.

</details>

---

## Q35

Match each artifact to what it enforces.

| Artifact | Enforces |
|---|---|
| Issue form with `required: true` |  |
| `blank_issues_enabled: false` |  |
| CODEOWNERS |  |
| `require_code_owner_reviews` |  |
| Built-in project workflows |  |
| Notification subscription |  |

**Options:** A non-empty field before submission · A template must be chosen · Nothing — it delivers, it does not gate · Nothing on its own — it requests reviewers · Status transitions on project events · That the requested review must be given

<details>
<summary>Show answer</summary>

| Artifact | Enforces |
|---|---|
| Issue form with `required: true` | **A non-empty field before submission** |
| `blank_issues_enabled: false` | **A template must be chosen** |
| CODEOWNERS | **Nothing on its own — it requests reviewers** |
| `require_code_owner_reviews` | **That the requested review must be given** |
| Built-in project workflows | **Status transitions on project events** |
| Notification subscription | **Nothing — it delivers, it does not gate** |

**In `challenge-02.md`:** lines **267–268**, **352**, **459–477**, **687**, **128–131**,
**488–525**.

**Two of the six enforce nothing**, and both are the ones people name when asked "how do you make sure
a review happens" or "how do you make sure the team knows". **Delivery is not enforcement.**

</details>

---

# Section F — Hot area

---

## Q36

```bash
gh project field-create $PROJECT_NUMBER \
  --owner "contoso-org" \
  --name "Sprint" \
  --data-type "[BLANK 1]"

gh project field-create $PROJECT_NUMBER \
  --owner "contoso-org" \
  --name "Story Points" \
  --data-type "[BLANK 2]"
```

- **BLANK 1:** `ITERATION` / `SINGLE_SELECT` / `DATE` / `TEXT`
- **BLANK 2:** `NUMBER` / `SINGLE_SELECT` / `TEXT` / `ITERATION`

<details>
<summary>Show answer</summary>

### Answer: `ITERATION`, `NUMBER`

**In `challenge-02.md`:** lines **67–68** and **80–81**.

**`DATE` is the interesting distractor for BLANK 1.** A sprint is not a date — it is a **span** with a
start and an end, which is what `ITERATION` models and what a roadmap view needs to draw a bar.

**And `NUMBER` for story points is the requirement, not a preference** (Q9): velocity is a sum.

</details>

---

## Q37

```sql
SELECT [System.Id], [System.Title], [System.State]
FROM WorkItems
WHERE [System.AreaPath] [BLANK 1] 'Contoso Web Platform\Backend'
  AND [System.IterationPath] = [BLANK 2]
  AND [System.State] <> 'Closed'
```

- **BLANK 1:** `UNDER` / `=` / `CONTAINS` / `IN`
- **BLANK 2:** `@CurrentIteration` / `'Sprint 1'` / `@Today` / `@Me`

<details>
<summary>Show answer</summary>

### Answer: `UNDER`, `@CurrentIteration`

**In `challenge-02.md`:** line **226**.

**`UNDER` includes child area paths**, so a nested `Backend\API` is picked up. `=` would match that one
node exactly and silently miss every sub-team.

**`@Today` is the distractor worth naming** — it is a real macro, for **date** fields such as
`ChangedDate`, not for iteration paths. `@Me` resolves to the current user.

</details>

---

## Q38

```yaml
  - type: dropdown
    id: severity
    attributes:
      label: Severity
      options:
        - Critical (system down)
        - High (major feature broken)
    [BLANK 1]:
      [BLANK 2]: true
```

- **BLANK 1:** `validations` / `rules` / `constraints` / `checks`
- **BLANK 2:** `required` / `enforced` / `mandatory` / `strict`

<details>
<summary>Show answer</summary>

### Answer: `validations`, `required`

**In `challenge-02.md`:** lines **258–268**.

```yaml
    validations:
      required: true
```

**The key is `validations` even though the only thing under it is a presence check** — which is exactly
why the exam asks. The field name suggests format validation; the behaviour is "not empty".

</details>

---

## Q39

```text
# .github/CODEOWNERS
*                     @contoso-org/engineering-leads
/src/api/             @contoso-org/backend-team
/src/auth/            [BLANK 1]
```

Requirement: every change under `/src/auth/` must be reviewed by the security team.

- **BLANK 1:** `@contoso-org/security-team` / `security-team` / `@security-team` /
  `contoso-org/security-team`

<details>
<summary>Show answer</summary>

### Answer: `@contoso-org/security-team`

**In `challenge-02.md`:** lines **475–476**.

```text
# Security team must review auth changes
/src/auth/ @contoso-org/security-team
```

**A team reference needs the `@` and the organisation prefix.** Drop either and the entry does not
resolve — and it fails **silently**, which is half of Break scenario 3.

**Note the ordering rule that makes this file work at all: the last matching pattern wins.** `*` on line
461 assigns engineering leads to everything, and the more specific paths below override it.

</details>

---

## Q40

```yaml
  close-linked-issues:
    if: github.event_name == 'pull_request'
        && github.event.action == '[BLANK 1]'
        && github.event.pull_request.[BLANK 2] == true
```

- **BLANK 1:** `closed` / `merged` / `completed` / `done`
- **BLANK 2:** `merged` / `closed` / `success` / `state`

<details>
<summary>Show answer</summary>

### Answer: `closed`, `merged`

**In `challenge-02.md`:** line **597**.

**There is no `merged` action — that is the whole trap.** GitHub fires `closed` for both a merge and an
abandonment, so the merge is a **property of the payload**, checked separately.

**Get it backwards and the job never runs** (waiting for an action that does not exist); omit the second
clause and abandoned PRs mark their issues Done.

</details>

---

## Q41

```yaml
      - name: Add issue to project
        uses: actions/add-to-project@v1
        with:
          project-url: https://github.com/orgs/contoso-org/projects/5
          github-token: [BLANK 1]
```

Requirement: add issues to an **organisation** project.

- **BLANK 1:** the `secrets.PROJECT_TOKEN` expression / the `secrets.GITHUB_TOKEN` expression /
  the `github.token` expression / `none`

<details>
<summary>Show answer</summary>

### Answer: the `secrets.PROJECT_TOKEN` expression

**In `challenge-02.md`:** lines **552–553**.

**Look at the `project-url`: it starts `/orgs/`.** An organisation project sits outside the repository,
and the built-in token cannot reach outside the repository — which is Q26's failure and Challenge 40's
rule.

**The scopes that token needs are named at line 637**: `project` read/write, `issues` read, and
`org:read` for organisation projects.

</details>

---

# Section G — Case study

## Case study: Contoso work-tracking unification

### Background

Contoso Ltd's product development is in disarray. Project managers maintain **spreadsheets that are
outdated within hours**. Developers track work on **sticky notes or personal to-do lists**. The QA team
finds out about new features **only when code hits staging**. Sprint planning is **guesswork because
nobody has data on velocity or remaining work**. The Director of Engineering wants a unified work
tracking system with proper feedback loops so every stakeholder has real-time visibility.

### Requirements

**Tracking**

- One system of record; no parallel spreadsheet
- Developers, project managers and leadership each need a suitable presentation of the same work
- Story points must be aggregable so velocity can be measured
- Sprint queries must not need editing when the sprint rolls over

**Intake**

- Bug reports must arrive with severity, reproduction steps, expected behaviour and environment
- Contributors must not be able to file an empty issue
- Security vulnerabilities must not be filed publicly

**Flow**

- Code must link to work, and merged work must move without anyone updating a board
- Changes to authentication code must always be reviewed by the security team
- The backend team must hear about their high-priority bugs and nothing else

---

## Q42

How should the three audiences be served?

- A. One project with board, table and roadmap views over the same items
- B. Three projects, one per audience
- C. A project for developers and a spreadsheet for PMs
- D. A weekly exported report

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **87–103**.

**"One system of record" and "each needs a suitable presentation" are two requirements that only a
views model satisfies together.**

**Why B fails on the first requirement.** Three projects is three systems of record with three sets of
stale data — the spreadsheet problem with a nicer interface.

**Why C is what Contoso has today** (line 21), and **why D re-creates the staleness the scenario opens
with**: a report is accurate at the moment it is exported and decays from then on.

</details>

---

## Q43

How is velocity made measurable?

- A. A `NUMBER` Story Points field plus an `ITERATION` Sprint field
- B. Counting issues closed per sprint
- C. A `SINGLE_SELECT` Story Points field with values 1, 2, 3, 5, 8
- D. Counting commits

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **65–81**, with the Azure Boards equivalent at **199**.

**Velocity is points-per-sprint, so it needs a summable number *and* a sprint boundary to sum within.**
One without the other gives you a total with no period, or a period with nothing to total.

**Why C is the near-miss that breaks the arithmetic.** A single-select stores the *label* "5", not the
number 5 — so it groups and filters and cannot be summed. This is the same class of error as putting a
measurement in a string field (Challenge 47).

**Why B and D are proxies that mislead.** Issues closed counts small issues and large ones equally;
commits measure typing.

</details>

---

## Q44

How should bug reports be made usable on arrival?

- A. Issue forms with required severity, description, reproduction, expected behaviour and environment,
  with blank issues disabled and a private route for security
- B. A Markdown issue template
- C. A note in CONTRIBUTING.md
- D. Triage the reports manually and ask for details

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **247–307** and **351–359**.

**All three intake requirements are satisfied by one configuration.** Required fields guarantee the
content; `blank_issues_enabled: false` closes the bypass; the `contact_links` security advisory URL
keeps vulnerabilities out of public issues.

**Why B is the older, weaker form and the best distractor.** A Markdown template pre-fills text the
author can delete. **A form's `required: true` cannot be deleted** — the issue will not submit.

**Why D is the current state.** Manual triage is what the team does now, and it is why QA hears about
things late.

</details>

---

## Q45

Which **two** ensure auth changes are always reviewed by the security team? (Choose two.)

- A. A CODEOWNERS entry mapping `/src/auth/` to the security team
- B. Branch protection with require-code-owner-reviews enabled
- C. A label named `security`
- D. A notification subscription for the security team
- E. An issue template

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-02.md`:** lines **475–476** and **687**.

**A routes the request; B makes it mandatory.** The word in the requirement is "always", and A alone
delivers "usually, if someone remembers not to merge without it".

**Why D is the most tempting wrong answer.** A subscription tells the security team a PR exists — which
is genuinely useful and does not stop the merge. **Notification is not enforcement** (Q35).

**And do not forget the quiet precondition** from Break scenario 3: the security team must have
repository access, or the CODEOWNERS line is ignored with no error.

</details>

---

## Q46

How should the backend team's notifications be scoped?

- A. A subscription filtered on work item type Bug, priority ≤ 2, and area path under Backend
- B. A subscription on all work item changes
- C. Everyone watches the repository
- D. A daily digest of the whole board

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **499–515**.

**Three clauses, three narrowings: what kind of item, how urgent, and whose.** The requirement says
"their high-priority bugs and nothing else", and each clause removes one category of noise.

**Why B is how the requirement is actually violated in practice.** It technically delivers the
high-priority bugs — inside a stream so large the team mutes it, at which point the delivery rate is
effectively zero.

**Priority `<= 2` is worth reading carefully**: lower numbers are more urgent in Azure Boards, so `<= 2`
means priority 1 and 2.

</details>

---

## Q47

Eight months after the rollout, the PM notices the "Current Sprint - Backend" query has been returning
an empty list for about three weeks, while the board itself looks healthy. The team recently created a
new sub-area, `Backend\Payments`, and moved most stories into it.

What happened, and what is the fix?

- A. Nothing is wrong with the query's iteration clause — but if the area clause had used `=` instead of
  `UNDER`, the sub-area would be excluded; confirm the clause is `UNDER 'Contoso Web Platform\Backend'`
- B. `@CurrentIteration` expired and must be refreshed
- C. The shared query was deleted
- D. Story points were removed

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** line **226**.

```sql
WHERE [System.AreaPath] UNDER 'Contoso Web Platform\Backend'
```

**"The board looks healthy" is the detail that localises it.** The work items exist and are in the
current sprint; the *query* is not selecting them — so the problem is a clause, and the thing that
changed is the area path.

**`UNDER` versus `=` is the entire difference.** `UNDER` matches the node and everything beneath it, so
`Backend\Payments` is included. `=` matches that exact node only, and every story moved into the new
sub-area silently drops out of the sprint query, the dashboard built on it, and any notification
subscription written the same way.

**Why B is a plausible-sounding fiction.** `@CurrentIteration` is evaluated at query time and never
expires — that is its entire purpose (Q4).

**The durable lesson: reorganising area paths is a change to every query, dashboard and subscription
that references them.** Use `UNDER` by default, and treat an area-path change like a schema change.

</details>

---

## Q48

A year on, the PM's spreadsheet is gone, QA joins sprint planning with the same board the developers
use, and velocity is a number nobody argues about.

Explain what each piece contributed, and what actually changed.

- A. One project with three views removed the second source of truth; `ITERATION` plus a `NUMBER` field
  made velocity computable; `@CurrentIteration` kept queries valid across sprints; issue forms made
  reports usable on arrival; `Fixes AB#`/`Fixes #` moved work without anyone updating a board; and
  CODEOWNERS with required reviews put the right eyes on the right code — every one of them enforced by
  the tool rather than by a person remembering
- B. The team became more disciplined about updating the board
- C. The PM started exporting the board to a spreadsheet weekly
- D. More meetings were added to sprint planning

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-02.md`:** lines **87–103**, **65–81**, **226**, **247–307**, **407–442**, **475–487**.

**Take the failures at line 21 one at a time.**

The **spreadsheet went stale within hours** because it was a copy. Three **views** over one dataset
means there is nothing to copy — the PM's table and the developer's board cannot disagree.

**Sprint planning was guesswork** because story points lived on sticky notes. A `NUMBER` field inside an
`ITERATION` makes velocity arithmetic rather than opinion.

**QA found out at staging** because nothing told them earlier. An issue on the board from the moment it
opens (line 547) is visible weeks before code exists.

**And the board stayed accurate** because `Fixes AB#1234` moves the work item on merge — nobody has to
remember to drag a card, which is the step that always gets skipped under pressure.

**What actually changed is not diligence.** The scenario does not describe lazy people; it describes
people maintaining **parallel records by hand**, which is a task that fails the moment anyone is busy.
**The fix was to make the record a by-product of the work itself** — the commit links it, the merge
moves it, the view presents it.

**That is the sentence to give any exam question about work tracking and feedback cycles.** A tracking
system that requires manual updating is a spreadsheet with better fonts. **The graded design is the one
where doing the work updates the record.**

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **`AB#1234` assumed to transition** | Q2, Q19, Q25, Q29, Q32 | The keyword `Fixes` is the trigger |
| **`Closes #n` used for Azure Boards** | Q2, Q29 | Bare `#` is GitHub Issues; `AB#` is Boards |
| **Field vs view confused** | Q1, Q27, Q31 | Fields store data; views present it |
| **`SINGLE_SELECT` for story points** | Q9, Q27, Q43 | A select is not summable. Velocity is a sum |
| **`SINGLE_SELECT` for sprint** | Q8, Q36 | `ITERATION` is date-aware; a select is a list |
| **Hard-coded iteration path** | Q4, Q28, Q34 | `@CurrentIteration` survives the sprint boundary |
| **`=` instead of `UNDER` on area path** | Q10, Q37, Q47 | `UNDER` includes sub-areas |
| **CODEOWNERS assumed to enforce** | Q3, Q20, Q35, Q45 | It requests. Branch protection enforces |
| **CODEOWNERS team without repo access** | Q7, Q20 | Silently ignored, no error |
| **Built-in `GITHUB_TOKEN` for an org project** | Q5, Q26, Q41 | Repository-scoped. Needs `project` + `org:read` |
| **`merged` used as a PR action** | Q40 | The action is `closed`; `merged` is a payload field |
| **Missing the `merged == true` check** | Q22, Q30, Q34 | Abandoned PRs would mark issues Done |
| **Unfiltered notification subscriptions** | Q21, Q25, Q46 | Delivered and muted is not delivered |
| **A Markdown template read as enforcement** | Q44 | Text can be deleted; `required: true` cannot |

---

# What to memorise

**In `challenge-02.md`:** lines **58–103**, **152–233**, **407–442**, **459–487**.

```text
FIELD vs VIEW      field = data ON the item     view = how items are DISPLAYED
  fields  Priority SINGLE_SELECT | Sprint ITERATION | Team SINGLE_SELECT | Story Points NUMBER
  views   board (developers) | table (project managers) | roadmap (leadership)
  ONE dataset, three lenses -> nothing to keep in sync -> no stale spreadsheet

LINK vs TRANSITION
  GitHub Issues   Fixes #42 | Closes #42 | Resolves #42   -> closes on merge
                  Relates to #38                           -> links only
  Azure Boards    AB#1234                                  -> LINKS ONLY
                  Fixes AB#1234                            -> links AND moves to Done
  requires BOTH:  Azure Boards GitHub App installed on the repo
                  + connection in Azure DevOps > Project Settings > GitHub connections
```

```text
AZURE BOARDS STRUCTURE                              (lines 152-233)
  AREA path       = WHO owns it   (Backend / Frontend / Platform)
  ITERATION path  = WHEN it is done (Sprint 1, 2, 3 with start/finish dates)
  hierarchy       Epic -> Feature -> User Story   via  --relation-type "Parent"
                  (Agile template. Scrum uses Product Backlog Item)
  story points    --fields "Microsoft.VSTS.Scheduling.StoryPoints=5"

WIQL macros
  @CurrentIteration  resolves to the sprint containing TODAY, per team - NEVER needs editing
  @Today  (date fields)      @Me  (current user)
  [System.AreaPath] UNDER 'Project\Backend'    <- UNDER includes sub-areas.  = does NOT
  [System.IterationPath] = @CurrentIteration
```

```yaml
# Issue form - the required-field mechanism        (lines 247-307)
  - type: dropdown            # also: input, textarea, markdown, checkboxes
    id: severity
    attributes:
      label: Severity
      options: [Critical (system down), High (major feature broken), ...]
    validations:
      required: true          # cannot submit empty. a MARKDOWN template can just be deleted

# config.yml                                        (lines 351-360)
blank_issues_enabled: false   # forces a template to be chosen
contact_links:
  - name: Security vulnerability
    url: .../security/advisories/new     # keeps CVEs out of public issues
```

```text
CODEOWNERS                                          (lines 459-477)
  .github/CODEOWNERS  |  CODEOWNERS  |  docs/CODEOWNERS      <- three valid locations
  *          @org/engineering-leads                          <- LAST MATCH WINS
  /src/api/  @org/backend-team
  /src/auth/ @org/security-team
  needs BOTH:  require_code_owner_reviews = true  (branch protection)
               + the team has at least READ access, or the line is SILENTLY ignored

NOTIFICATION SUBSCRIPTION - filter or be muted      (lines 499-515)
  System.WorkItemType             =      Bug
  Microsoft.VSTS.Common.Priority  <=     2        (lower number = MORE urgent)
  System.AreaPath                 Under  Project\Backend

PROJECT AUTOMATION                                  (lines 539-605)
  built-in workflows: item added -> Backlog | PR merged -> Done | reopened -> In Progress
  Actions workflow for what built-ins cannot do (setting a custom field from a label)
  if: github.event.action == 'closed' && github.event.pull_request.merged == true
      ^ there is NO 'merged' action. closed fires for merge AND abandon
  github-token: secrets.PROJECT_TOKEN   <- org project. built-in GITHUB_TOKEN CANNOT reach it
      scopes: project (read/write), issues (read), org:read
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 03 |
| 38–43 | Re-read the trap index and the link-vs-transition table, then move on |
| 30–37 | Rewrite the field/view table and the AB# rules from memory, then retake |
| Below 30 | Redo Tasks 1, 2 and 4 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 02.

:::danger The two distinctions

**Field or view?** A field stores data on an item. A view decides who can see it usefully.

**Link or transition?** `AB#1234` links. **`Fixes AB#1234`** links *and moves the work item*. The
keyword is the trigger.

And the design that works is the one where **doing the work updates the record** — not the one that asks
people to update it afterwards.

:::
