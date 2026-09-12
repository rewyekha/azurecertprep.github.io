---
sidebar_position: 3.5
toc_max_heading_level: 2
title: "Challenge 18: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 18 — AZ-400 exam questions

**48 questions** built only from what Challenge 18 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-18.md`**.

:::danger Read this before you start

**Coverage measures which lines *ran*. It does not measure whether anything *checked* them.**

A test with no assertions at all executes the code and reports full coverage. That is not a technicality —
it is the reason coverage is a **floor**, not a goal. It tells you what was definitely **not** tested; it
can never tell you that what was tested was tested well.

**Three numbers, three different policies, and only one of them changes behaviour today.**

**An absolute threshold** — 80% overall — is satisfied on day one by legacy code and says nothing about
the change in front of you. **A ratchet** — never go below the last recorded baseline — stops erosion.
**Diff coverage** — 90% of the *lines this pull request added* — is the only one that makes the author of
a change write a test. Contoso's policy at lines 26–28 uses all three, deliberately.

**And then there is the plumbing, which is where the marks actually are.**

Every "0% coverage" in this challenge is a **plumbing failure, not a testing failure**: a reporter that
was never requested, a file written to a path nobody reads, a collector package that was never installed,
a source map that was never configured. Tests passing and coverage reporting are **two separate
pipelines**, and the second one fails silently.

**One habit will carry you through Section B.** Every check in this challenge is wrapped in
`if [ -f ... ]`. Ask each time: **what happens when the file is missing?** In this workflow, the answer is
almost always "the check is skipped and the gate goes green".

Three stacks, at lines 32–34: **Checkout API** on Node and Jest, **Inventory service** on Python and
pytest, **Payment gateway** on .NET and xUnit. Learn which format each one emits and half the paper
follows.

:::

---

# Section A — Multiple choice

---

## Q1

The log reads "42 tests passed" and every coverage metric reads 0%. What class of problem is this?

- A. A plumbing problem — output missing or written elsewhere
- B. The tests are not asserting anything meaningful
- C. The runner does not support coverage instrumentation
- D. Passing tests report 0% until a baseline exists

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-18.md`:** lines **666–673**, and the knowledge check at line **818**.

**Running tests and collecting coverage are two independent processes.** The first produces pass and fail
results; the second instruments the code and writes a data file. The first can succeed completely while
the second never happens.

**All four root causes are of that kind** (lines 683–697): a reporter that was never requested, a file
written to the wrong path, a collector package that was never referenced, and instrumentation applied to
transpiled output instead of source.

**Why B produces the opposite symptom.** Assertion-free tests report *high* coverage — the lines all ran.
That is Q45, and it is the reason coverage cannot be trusted as a measure of test quality.

**Why C is the answer the exam offers to make you doubt the environment.** Coverage is instrumentation
inside the test process; the runner has no opinion about it.

**The diagnostic to memorise: after running the tests locally, `ls` for the coverage file.** Line 713 does
exactly that. If the file is not there, no configuration downstream matters.

</details>

---

## Q2

Which Jest setting determines that a source file with **no tests at all** still appears in the coverage
report?

- A. `testMatch`
- B. `coverageDirectory`
- C. `coverageReporters`
- D. `collectCoverageFrom`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-18.md`:** lines **53–58**.

```javascript
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/**/__tests__/**',
    '!src/**/index.js',
    '!src/migrations/**',
  ],
```

**By default Jest reports coverage only for files a test actually loaded.** A module nobody imports is
absent from the report entirely — so it neither raises nor lowers the percentage, and a service with two
tested files and forty untested ones can report 95%.

**`collectCoverageFrom` sets the denominator explicitly.** Every file matching the pattern is measured,
imported or not, and the untested ones arrive as zeros.

**This is the single most consequential line in `jest.config.js`**, and the exam likes it because it is
counter-intuitive: adding this setting makes coverage **drop**, and the drop is the truth.

**Why A selects which files are *tests***, not which are *measured* — and note that `!src/**/__tests__/**`
excludes the tests themselves from the denominator, because measuring your tests' coverage is circular.

**Why B is where output goes and C is what formats it is written in** — both about the report, neither
about the population being measured.

</details>

---

## Q3

Which coverage format does `PublishCodeCoverageResults@2` accept?

- A. LCOV trace files
- B. HTML browsable reports
- C. Cobertura or JaCoCo XML
- D. `json-summary` totals

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-18.md`:** lines **589–592**, and the knowledge check at line **796**.

```yaml
          - task: PublishCodeCoverageResults@2
            inputs:
              summaryFileLocation: '$(System.DefaultWorkingDirectory)/services/checkout-api/coverage/cobertura-coverage.xml'
```

**Cobertura is the lingua franca here**, which is why all three stacks are configured to emit it: Jest via
the `cobertura` reporter (line 60), pytest via `--cov-report=xml` (line 150), and .NET via the coverlet
`Format=cobertura` run setting (line 272).

**Why A is the format GitHub tooling prefers** and Azure DevOps will not read. `lcov.info` is what the
Node diff-coverage script parses at line 527 — right file, wrong consumer.

**Why B is for humans.** The HTML report at line 283 is a browsable artifact, not machine input.

**Why D is Jest-specific and read by `jq`** (lines 323, 377), not by any Azure DevOps task.

**Read the whole challenge through this lens: match the format to the reader.** Cobertura for
Azure Pipelines, lcov for GitHub tooling and the diff parser, `json-summary` for shell scripts, HTML and
`text` for people.

</details>

---

## Q4

Every metric in `coverageThreshold` is set to 80. Which one can sit at 100% while an untested `else`
path ships to production?

- A. `lines`
- B. `branches`
- C. `functions`
- D. `statements`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-18.md`:** lines **61–68**.

```javascript
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
```

**A line containing a condition is marked covered the moment the line executes**, regardless of which way
the condition went. So a single test entering an `if` gives you 100% line coverage of that line and 50%
branch coverage.

**`branches` is the demanding metric**, and it is the one that forces both sides of every condition to be
exercised. When a scenario says "an untested error path shipped despite high coverage", the answer is
almost always branch coverage.

**Which makes Contoso's own policy worth noticing.** Line 26 says *line* coverage, and the gate at line
323 reads `.total.lines.pct`. The policy is written against the forgiving metric even though the config
enforces all four locally.

**Why C counts functions entered at least once** — a function with ten branches counts once — **and why D
is statement-level, close to lines** and equally blind to untaken paths.

</details>

---

## Q5

The Python job's test step is just `pytest`, with no flags. Where do the coverage options come from?

- A. Environment variables set by `setup-python`
- B. A `.coveragerc` file in the repository root
- C. `requirements-dev.txt` listing `pytest-cov`
- D. `addopts` under `[tool.pytest.ini_options]`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-18.md`:** lines **148–150** and **200**.

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--cov=src --cov-report=xml:coverage/coverage.xml --cov-report=html:coverage/html --cov-report=term-missing --cov-fail-under=80"
```

**`addopts` is prepended to every pytest invocation.** That is why the workflow step at line 200 is one
bare word, and it is deliberate: the same command runs identically on a laptop and in CI, because the
configuration lives with the code rather than in the pipeline.

**Contrast it with the Node service**, where the flags sit partly in `jest.config.js` and partly on the
command line (line 107), and with Azure Pipelines at line 604, where the flags are typed into the pipeline
again — and can therefore drift from what developers run locally.

**Why B is the older, still-valid alternative.** `.coveragerc`, `setup.cfg` and `pyproject.toml` are three
places coverage settings may live; this challenge uses `pyproject.toml` for both pytest and
`[tool.coverage.*]`.

**Why A and C are the plausible neighbours.** `setup-python` installs an interpreter and restores a cache;
`requirements-dev.txt` provides `pytest-cov` so `--cov` exists at all — necessary, not sufficient.

</details>

---

## Q6

What does `--cov-fail-under=80` do?

- A. Skips tests when coverage is below 80%
- B. Excludes files below 80% coverage from the report
- C. Makes pytest exit non-zero below 80% total coverage
- D. Warns in the log without changing the exit code

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-18.md`:** line **150**, with `fail_under = 80` repeated at line **168**.

**A non-zero exit fails the step, which fails the job.** This is the Python equivalent of Jest's
`coverageThreshold` — the enforcement that happens *inside the test run*, before any artifact exists.

**Note that the setting appears twice**, on the command line via `addopts` and in `[tool.coverage.report]`.
Both are real; the `addopts` form is what the challenge's job actually triggers.

**And notice the layering.** Coverage in this challenge is enforced at three separate places: inside the
test run (`coverageThreshold`, `fail_under`), in the aggregating gate job (line 358), and by the PR
comment action's `thresholdAll` (line 214). The first is fastest to fail; the second is the one wired to
branch protection.

**Why D is the behaviour when the flag is absent** — coverage is printed and nothing happens, which is
the report-versus-gate split again.

</details>

---

## Q7

`[tool.coverage.report]` lists `exclude_lines` including `pragma: no cover` and `pass`. What is the effect
on the reported percentage?

- A. The lines leave the denominator; the percentage rises
- B. Those lines are counted as covered lines
- C. Those lines fail the build when executed
- D. Nothing — `exclude_lines` only affects the HTML report

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-18.md`:** lines **160–167**.

```toml
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "if __name__ == .__main__.",
    "raise NotImplementedError",
    "pass",
]
```

**Exclusion changes the denominator.** An excluded line is neither covered nor uncovered; it is not
measured. That is legitimate for genuinely untestable lines — a `__main__` guard, a `__repr__`, an
abstract stub — and it is exactly how a coverage number gets inflated when abused.

**`pragma: no cover` is the one to watch.** It is a comment a developer can add to any line, and a PR that
fixes a coverage failure by adding pragmas has moved the number without changing the testing.

**Which is why the same idea appears in all three stacks** and is worth grouping in your head:
`collectCoverageFrom` negations for Jest (lines 55–57), `omit` and `exclude_lines` for coverage.py (lines
154–167), `ExcludeByFile` for coverlet (line 273).

**Why B is materially different.** Counting as covered inflates the numerator *and* the denominator;
excluding removes both. Both raise the percentage, and only exclusion is what these settings do.

</details>

---

## Q8

`dotnet test --collect:"XPlat Code Coverage"` runs, tests pass, and no coverage file is produced. Why?

- A. The results directory does not exist on the agent
- B. `--no-restore` prevented the collector loading
- C. Cobertura output is not supported on Linux agents
- D. `coverlet.collector` is not referenced by the project

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-18.md`:** lines **691–693**, with the package reference at line **233**.

```xml
    <PackageReference Include="coverlet.collector" Version="6.0.2" />
```

**`XPlat Code Coverage` is a *name* the test platform resolves to a data collector.** If no package
provides that collector, `dotnet test` does not error — it runs the tests, reports them, and writes no
coverage. **Silently** is the operative word at line 693.

**This is the .NET member of the 0% family**, and it is the hardest to spot because the command line looks
completely correct. Nothing about `--collect:"XPlat Code Coverage"` reveals that it depends on a NuGet
reference.

**The fix at line 736 is one command** — `dotnet add package coverlet.collector` — and line 742 shows the
fuller form with `PrivateAssets` and `IncludeAssets`, which keeps the collector out of the published
output while still available at test time.

**Why A would produce an error rather than silence**, and **why B affects package restore only** — a
missing restore fails the build, loudly.

</details>

---

## Q9

ReportGenerator is run with `-reporttypes:"Html;Cobertura;JsonSummary"`. Which output does the coverage
gate job actually read?

- A. The HTML report for browsing
- B. `Summary.json` from `JsonSummary`
- C. The merged Cobertura XML
- D. `lcov.info` from the Node service

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-18.md`:** lines **278–283** and **347–348**.

```bash
DOTNET_COV=$(jq '.summary.linecoverage' ./coverage-artifacts/coverage-dotnet/Summary.json)
```

**Three report types, three audiences.** HTML is for a person browsing the artifact, Cobertura is for
`PublishCodeCoverageResults@2` on the Azure Pipelines side, and `JsonSummary` exists purely so a shell
script can read one number with `jq`.

**That is why ReportGenerator is in the pipeline at all.** Coverlet emits per-assembly
`coverage.cobertura.xml` files under a GUID-named directory; ReportGenerator **merges** them and
normalises the output. Without it the gate would have to find and combine several XML files itself.

**Note the key name: `.summary.linecoverage`** — line coverage again (Q4), and expressed as a percentage
here rather than a fraction, which is why there is no multiplication by 100 on this branch of the script.

**Why C and A are produced and not consumed by this job**, and **why D belongs to the Node service**.

</details>

---

## Q10

The Python branch of the gate parses `coverage.xml` and computes
`float(root.attrib['line-rate']) * 100`. Why the multiplication?

- A. To convert a raw count into a percentage figure
- B. To normalise across the three services
- C. Because Cobertura's `line-rate` is a 0–1 fraction
- D. Because `bc` cannot compare decimal values

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-18.md`:** lines **333–338**, and the same conversion in PowerShell at line **648**.

```python
print(float(root.attrib['line-rate']) * 100)
```

**Cobertura stores rates, not percentages.** A `line-rate` of `0.8534` means 85.34%. Forget the
multiplication and every service reads as below 1, so the gate reports "coverage 0.85%" and fails
everything — a failure that looks like catastrophic coverage rather than a unit error.

**Compare the three branches of the same script deliberately.** Jest's `json-summary` gives
`.total.lines.pct` — already a percentage. ReportGenerator's `Summary.json` gives `.summary.linecoverage` —
already a percentage. Only the raw Cobertura XML gives a fraction.

**Line 648 is the same trap in PowerShell:**

```powershell
$lineRate = [math]::Round([double]$xml.coverage.'line-rate' * 100, 2)
```

**Why D reverses cause and effect** — `bc -l` exists precisely because these values are decimal (Q11).

</details>

---

## Q11

Why does the gate use `bc -l` for its comparisons instead of a plain shell test?

- A. Shell arithmetic is integer-only, not decimal
- B. `bc` is faster than the shell's own built-ins
- C. `bc` is required to parse the JSON output
- D. To avoid spawning a subshell for each check

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-18.md`:** lines **325**, **340**, **350**, **458**.

```bash
if (( $(echo "$NODE_COV < 80" | bc -l) )); then
```

**`[ "$NODE_COV" -lt 80 ]` on a value of `79.6` is a syntax error**, and `(( 79.6 < 80 ))` is an
arithmetic error. Bash has no floating point. `bc -l` performs the comparison and prints `1` or `0`, which
the surrounding `(( ))` then evaluates as a plain integer.

**The reason this appears on the exam is the failure mode.** In a script without `set -e`, the error goes
to stderr, the condition evaluates as false, and **the gate passes** — a coverage check that quietly stops
checking as soon as the number is not a whole number.

**Which puts it in the same family as everything in Q12**: this workflow's gates are much better at
passing than at failing.

**Why C is a category error** — `jq` and Python read the JSON and XML; `bc` only does arithmetic — and
**why B is irrelevant** at this scale.

</details>

---

## Q12

Each service's check in the gate job is wrapped in `if [ -f "<artifact path>" ]; then ... fi`. What
happens if the Node artifact is missing?

- A. The gate fails with a clear error
- B. The job errors on the missing file
- C. The download step retries automatically
- D. The check is skipped and the gate passes

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-18.md`:** lines **319–361**.

```bash
FAILED=false
if [ -f "./coverage-artifacts/coverage-node/coverage-summary.json" ]; then
  ...
fi
...
if [ "$FAILED" = true ]; then
  exit 1
fi
```

**The gate can only fail on evidence it has.** No file, no comparison, no failure — so every way of losing
coverage data becomes a pass: a job that failed before uploading, a renamed artifact, a changed path, a
reporter that stopped emitting `json-summary`.

**This is the most important idea in the challenge and the reason Q26 is a "No".** A gate built from
`if [ -f ... ]` is a gate that **fails open**, and failing open is indistinguishable from working
correctly for as long as nothing goes wrong.

**The fix in one line:** require the file. `if [ ! -f "$FILE" ]; then echo "::error::missing coverage
data"; exit 1; fi` before the comparison, so absent evidence is a failure rather than a silence.

**Why B describes `-f` incorrectly** — testing for a file that is not there is the normal, non-error case —
and **why C invents behaviour** `download-artifact` does not have here.

</details>

---

## Q13

`coverage-gate.yml` is a separate workflow file whose job declares
`needs: [coverage-node, coverage-python, coverage-dotnet]`. Those jobs live in `coverage.yml`. What is the
result?

- A. The gate waits for the other workflow to finish
- B. GitHub matches jobs by name across workflows
- C. `needs:` is scoped to one file, so it is invalid
- D. The gate runs first and the other jobs wait

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-18.md`:** lines **294–306**, against the job definitions at lines **84**, **175** and
**246**.

**`needs:` is scoped to one workflow.** Names are resolved within the file; a name that does not exist
there is a validation error, not a cross-file link.

**And the same boundary bites the artifacts.** `download-artifact` at line 313 retrieves artifacts from
**the current run**. A different workflow is a different run, so even with the dependency fixed, the
download needs the other run's ID and a token — or the jobs need to be in one workflow.

**The two ways out, and both are exam-worthy.** Move the gate job into `coverage.yml` so `needs:` and the
artifacts resolve naturally, or trigger the gate with `workflow_run` and fetch artifacts from the
triggering run explicitly.

**Why A is what everyone assumes and GitHub does not do**, and **why B would be a very different product**
— job names are not global identifiers.

</details>

---

## Q14

What does Contoso's "ratchet" policy require?

- A. Coverage must rise by at least 5% per pull request
- B. Coverage must never fall below the last `main` baseline
- C. Coverage must reach 100% by an agreed date
- D. Only new repositories must meet the threshold

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-18.md`:** line **27**, implemented at lines **449–464**, and the knowledge check at line
**807**.

```bash
BASELINE=$(tail -1 .coverage-history/trend.jsonl | jq '.checkout_api')
if (( $(echo "$CURRENT < $BASELINE" | bc -l) )); then
  exit 1
fi
```

**A ratchet only turns one way.** Each merge to `main` records the number (line 418); each pull request
compares against the last recorded value and must equal or beat it.

**Its purpose is to make legacy code affordable.** A service at 62% cannot reach 80% in one pull request,
and demanding that means the policy gets waived. A ratchet says: **you may not make it worse**, and
combined with a diff-coverage rule the number climbs on its own as the code changes.

**Why A is a policy nobody can satisfy repeatedly** — 5% per PR runs out of code — and **why C confuses a
direction with a destination.**

**And note where the baseline lives**: a JSONL file appended on every merge (line 418) and stored in the
Actions cache under `coverage-baseline-<sha>` (line 424). One line per merge is also what makes
requirement 4, the trend, possible.

</details>

---

## Q15

What is diff coverage, and which requirement does it satisfy?

- A. Coverage of the PR's changed lines — requirement 3
- B. Coverage of the whole codebase on the base branch
- C. The difference between branch and line coverage
- D. Coverage excluding the test files themselves

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-18.md`:** line **28**, implemented at lines **485–489** and **512–556**, and the knowledge
check at line **785**.

```bash
diff-cover coverage/coverage.xml \
  --compare-branch=origin/main \
  --fail-under=90
```

**Diff coverage ignores the legacy debt entirely.** A service sitting at 61% overall can still be required
to test 90% of every new line, because the metric is computed only over the lines the pull request
touched.

**It is the only one of the three policies that changes what an author does today.** An overall threshold
is met or missed by history; a ratchet says do no harm; **diff coverage asks for a test with this change**.

**The Python side uses `diff-cover`** against the Cobertura XML; the Node side hand-rolls the equivalent by
parsing `lcov.info` for the changed files (lines 525–553), and the `orgoro/coverage` action expresses it
declaratively as `thresholdNew: 0.90` (line 215).

**Why C is a real distinction and a different one** (Q4), and **why D describes an exclusion setting.**

</details>

---

## Q16

Every coverage job checks out with `fetch-depth: 0`. Why?

- A. To speed up the checkout on a large repository
- B. To include submodules alongside the checkout
- C. Because coverage tools read the Git history
- D. The ratchet and diff need history and a base

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-18.md`:** lines **93–94**, **185**, **255**, with the comparisons at lines **484**,
**486** and **517**.

```bash
git fetch origin main:refs/remotes/origin/main
CHANGED_FILES=$(git diff --name-only origin/main...HEAD -- 'services/checkout-api/src/**/*.js')
```

**The default checkout is `fetch-depth: 1`** — one commit, no history, no other branches. `origin/main...HEAD`
needs the **merge base**, and a shallow clone does not have one.

**The failure is the recurring one across this certification: it is green.** A diff against a base that
cannot be resolved produces an empty file list, an empty list produces zero changed lines, and zero
changed lines produce a diff coverage the script treats as 100% (line 547). **The gate reports success and
has measured nothing.**

**Note that `fetch-depth: 0` alone is not enough.** The explicit `git fetch origin main:...` at lines 484
and 516 is still required, because a pull-request checkout gives you the merge commit and not necessarily
a remote-tracking ref for the base branch.

**Why A inverts the cost** — full history is slower — and **why C is true of no coverage tool here**; it is
the *comparison* that needs history, not the measurement.

</details>

---

# Section B — Multiple answer

---

## Q17

Contoso's policy has four clauses. Which **three** are enforced by a mechanism that can fail a pull
request? (Choose three.)

- A. Coverage trends must be visible across sprints
- B. Overall line coverage at or above 80%
- C. Coverage reports must be uploaded as artifacts
- D. Coverage must not decrease against the baseline
- E. Every service must publish an HTML report
- F. New code in the pull request must reach 90%

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-18.md`:** lines **26–29**, enforced at lines **358**, **461** and **487**.

**Requirement 4 is a reporting requirement, not a gate.** It is satisfied by appending a line to
`trend.jsonl` on every merge (line 418) and by the PR comments — none of which refuses anything.

**Which is fine, and worth stating clearly**: not every policy clause needs teeth. The mistake the exam
tests is the reverse — assuming that because something is *visible*, it is *enforced*.

**Why C and E are the mechanics of requirement 4**, doing the same job. An artifact preserves the data; an
HTML report makes it readable. Neither has an exit code.

**The three that do bite** each fail in a different job: the threshold inside the test run and again in the
gate, the ratchet in `compare-coverage`, and diff coverage in the Python and Node diff steps.

</details>

---

## Q18

Which **three** settings in this challenge raise the reported coverage percentage **without a single test
being written**? (Choose three.)

- A. `!src/migrations/**` in `collectCoverageFrom`
- B. `coverageThreshold` set to 80 on all metrics
- C. `omit` under `[tool.coverage.run]`
- D. `fetch-depth: 0` on the checkout step
- E. `exclude_lines` under `[tool.coverage.report]`
- F. `--cov-report=term-missing` in `addopts`

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-18.md`:** lines **57**, **154–158**, **160–167**.

**All three shrink the denominator.** Excluded files and excluded lines are not counted as uncovered —
they are not counted at all, so the percentage rises immediately.

**And a fourth belongs to the same family**: coverlet's `ExcludeByFile="**/Migrations/**"` at line 273.
Every stack has an exclusion mechanism, and every one of them is a legitimate tool and a plausible way to
cheat.

**The judgement to apply: exclude what cannot be meaningfully tested, never what is merely inconvenient.**
Generated migrations, `__repr__`, `__main__` guards — reasonable. A module that is hard to test is
exactly the module the number should be complaining about.

**Why B is enforcement, not measurement.** A threshold decides what to do with the number; it never
changes it.

**Why F adds the uncovered line numbers to the terminal output** — more information, same percentage — and
**why D is about Git history** (Q16).

</details>

---

## Q19

The `orgoro/coverage` action is configured with two thresholds. Which **two** statements are correct?
(Choose two.)

- A. Both values are percentages, so 0.80 means 0.8%
- B. `thresholdAll: 0.80` is the overall coverage requirement
- C. `thresholdNew` applies to newly added files only
- D. `thresholdNew: 0.90` is the diff-coverage requirement
- E. The action replaces `--cov-fail-under` entirely

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-18.md`:** lines **208–215**.

```yaml
        uses: orgoro/coverage@v3.2
        with:
          coverageFile: services/inventory-service/coverage/coverage.xml
          token: ${{ secrets.GITHUB_TOKEN }}
          thresholdAll: 0.80
          thresholdNew: 0.90
```

**One action, both policies, expressed as fractions.** `0.80` is 80% — which is why A is the trap: this
action takes fractions, while `--cov-fail-under=80` takes a percentage and Cobertura's `line-rate`
attribute is a fraction again (Q10). **Units change between neighbouring lines in this challenge.**

**Why C narrows "new" incorrectly.** Diff coverage measures **new and modified lines**, wherever they are —
a two-line change in a five-year-old file is measured.

**Why E is the layering point from Q6.** `--cov-fail-under` fails the *test run*; this action comments on
the *pull request*. Different moments, different audiences, and the `token:` input is the tell — it needs
API access because its output is a comment.

</details>

---

## Q20

Jest is configured with five reporters. Which **three** are consumed by something automated in this
challenge? (Choose three.)

- A. `lcov` for the diff parser
- B. `text` for the log
- C. `cobertura` for Azure
- D. `text-summary` for the log
- E. `json-summary` for `jq`
- F. `html` for browsing

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-18.md`:** line **60**, consumed at lines **527**, **592** and **323**.

| Reporter | Read by | Line |
|---|---|---|
| `lcov` | The Node diff-coverage script parsing `SF:` and `DA:` records | 527 |
| `cobertura` | `PublishCodeCoverageResults@2` | 592 |
| `json-summary` | `jq` in the gate and the PR comment | 323, 377 |
| `text` / `text-summary` | A person reading the job log | 60 |

**Five reporters is not excess.** Each one exists because a different consumer cannot read the others, and
deleting the one you do not recognise is how Issue 1 happens (line 685).

**Why F is not in the list at all** for Jest here — the HTML report in this challenge comes from pytest
(line 150) and ReportGenerator (line 283).

**The exam question built on this** is always "which reporter must be added for X to work", and the answer
follows from the consumer: Azure DevOps needs Cobertura, a shell script needs `json-summary`, a diff
parser needs `lcov`.

</details>

---

## Q21

Which **three** parts of this workflow will report success when the underlying data is missing? (Choose
three.)

- A. `--cov-fail-under=80` inside the test run
- B. The `if [ -f ... ]` guards around each threshold check
- C. Jest's `coverageThreshold` inside the test run
- D. The ratchet check when `trend.jsonl` does not exist
- E. The `exit 1` at the end of the gate job
- F. The Node diff-coverage script on an empty file list

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-18.md`:** lines **322–354**, **465–467**, **519–522** and **547**.

**Three different mechanisms, one shape: no evidence, no failure.**

**B** skips the comparison entirely (Q12). **D** prints "No baseline found. Using current as initial
measurement." and continues — reasonable on the very first run, and indistinguishable from a cache that
silently stopped restoring. **F** exits 0 on an empty file list at line 521, and even when it proceeds, a
`totalLines` of zero yields `pct = 100` at line 547.

**Why A, C and E fail closed.** A threshold inside the test run has the data by definition, and the gate's
`exit 1` is the one place a decision is actually enforced.

**Carry the question into every gate you review: what does this do when it has nothing to measure?** A
gate that answers "passes" is decoration with an exit code.

</details>

---

## Q22

Coverage reports 0% on all three services. Which **three** root causes does the solution identify?
(Choose three.)

- A. A reporter that produces console output only, no XML
- B. The tests are not asserting anything at all
- C. Coverage written to a path the publish task ignores
- D. Branch coverage measured instead of line coverage
- E. `coverlet.collector` missing, so nothing is collected
- F. The runner lacking instrumentation support

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-18.md`:** lines **683–693**, with a fourth cause at lines **695–697**.

**Three plumbing failures and one instrumentation failure**, and the fourth is worth learning beside them:
coverage collected against **transpiled** output rather than source, so the report describes files nobody
wrote (line 697).

**They fail in three different places.** Issue 1 is a config omission in the repository, Issue 2 is a
mismatch between two files that never see each other, Issue 3 is a missing NuGet package, and Issue 4 is a
build-tooling interaction. **The only thing they share is the symptom.**

**Why B produces high coverage, not zero** (Q45), and **why D changes which number you read**, not whether
one exists.

**Fix 4 at lines 750–757 is the one to remember**, because it names the modern escape hatch:

```javascript
  coverageProvider: 'v8',
```

**V8 native coverage bypasses Babel instrumentation entirely**, which sidesteps the whole source-map class
of problem.

</details>

---

## Q23

Which **two** components make up the Azure Pipelines implementation of the coverage policy? (Choose two.)

- A. `PublishCodeCoverageResults@2` with a Cobertura file
- B. `PublishTestResults@2` with JUnit format
- C. `PublishPipelineArtifact@1` with the lcov file
- D. A `PowerShell@2` script parsing `line-rate`, exit 1
- E. A branch policy that measures coverage directly

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-18.md`:** lines **589–592**, **607–610**, **626–629**, and **638–657**.

**Publishing and gating are separate, as they are on GitHub.** The publish task renders coverage in the
pipeline's Code Coverage tab — visibility. The PowerShell stage is the only thing that can fail the run.

```powershell
$lineRate = [math]::Round([double]$xml.coverage.'line-rate' * 100, 2)
if ($lineRate -lt $threshold) {
  Write-Error "Coverage $lineRate% is below threshold $threshold%"
  exit 1
}
```

**Note it searches recursively across `$(Pipeline.Workspace)`** for every `coverage.cobertura.xml` (line
644) — which means it inherits the same weakness as its bash counterpart: **find nothing, check nothing,
pass**.

**Why B handles test results, not coverage** — the distinction the knowledge check at line 796 makes
explicitly — and **why E does not exist**: Azure DevOps branch policies require a *build* to succeed; the
coverage judgement has to happen inside it.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso's engineering director has found that low-coverage services suffer three times more
production incidents. Policy: no merge below 80% overall line coverage, coverage must not decrease against
the base branch, new code must reach 90% coverage, and trends must be visible across sprints — across a
Node service, a Python service and a .NET service.

---

## Q24

**Proposed solution:** Configure Jest with `collectCoverageFrom`, a full reporter list including
`cobertura`, `lcov` and `json-summary`, and a `coverageThreshold` of 80. Configure pytest through
`addopts` with an XML report and `--cov-fail-under=80`. Reference `coverlet.collector` in the .NET test
project and merge its output with ReportGenerator. Check out with `fetch-depth: 0`. Add one gate job in
the same workflow that requires each service's coverage file to exist, compares each figure against 80 and
exits non-zero on any breach, and make it a required status check. Add a ratchet comparison against a
baseline appended on every merge to `main`, and a diff-coverage check with a 90% floor on the changed
lines.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-18.md`:** lines **53–68**, **148–168**, **233**, **278–283**, **93–94**, **317–361**,
**449–464**, **485–489**.

| Requirement | Enforced by | Line |
|---|---|---|
| 80% overall | `coverageThreshold`, `--cov-fail-under`, and the gate's comparison | 61, 150, 325 |
| No decrease | The ratchet against the last baseline | 458 |
| 90% on new code | `diff-cover --fail-under=90`, `thresholdNew` | 487, 215 |
| Visible trends | `trend.jsonl` appended per merge, plus PR comments | 418, 135 |

**Two clauses are doing quiet work.** "In the same workflow" is what makes `needs:` and the artifact
download resolve (Q13). "Requires each service's coverage file to exist" is what stops the gate failing
open (Q12) — and it is the sentence Q26 removes.

**And the three policies compose deliberately.** The threshold sets a floor, the ratchet stops erosion,
and diff coverage does the work of getting from one to the other.

</details>

---

## Q25

**Proposed solution:** Run each service's tests with `--coverage` and upload the HTML reports as build
artifacts so reviewers can browse them. Add `pragma: no cover` to modules the team considers hard to test,
and remove `collectCoverageFrom` so only files with tests are measured. Post the overall percentage as a
PR comment. Review the trend in the sprint retrospective.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures, and each one is a category.**

**Nothing here can fail a pull request.** HTML artifacts, a comment and a retrospective are all
visibility. Requirement 1 says "no pull request merges" — that needs a non-zero exit wired to a required
status check (line 358).

**Removing `collectCoverageFrom` makes the number meaningless upward.** Only files a test imported are
measured, so the forty untested modules vanish from the denominator and coverage reads high precisely
because testing is poor (Q2).

**`pragma: no cover` on inconvenient modules is the same move in the other stack** (Q7). Both adjust the
measurement rather than the code, and both make the number stop tracking the thing it was chosen to
track — which is the incident rate at line 24.

**And there is no diff coverage and no ratchet at all**, so requirements 2 and 3 are simply absent. An
overall percentage cannot express either: it is a fact about history, not about this change.

</details>

---

## Q26

**Proposed solution:** Configure Jest, pytest and coverlet with full reporter sets and per-run thresholds.
Merge the .NET output with ReportGenerator. Check out with `fetch-depth: 0`. Add one gate job in the same
workflow that compares each service against 80 and exits non-zero on any breach, and make it a required
status check. Add the ratchet and the 90% diff-coverage check. Because most pull requests touch only one
service, wrap each service's comparison in a check that skips it when that service's coverage file is not
present.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**"Skip when the file is not present" cannot distinguish "not built" from "failed to build".** A job that
crashed before uploading, an artifact renamed, a reporter dropped from the config — every one of them
looks exactly like "this PR did not touch that service", and every one produces a green gate.

**And the stated reasoning does not need the exception.** The real requirement is "do not penalise a PR
for services it did not change", and that is what **path filters** are for: run each service's coverage job
only when its directory changed, and have the gate require a result from every job that ran.

**The distinction to keep: "no data" is not "no problem".** Absent evidence must be a failure, because the
whole point of a gate is that it refuses when it cannot confirm.

**Which is doubly true here.** This gate is a required status check, so it is the last thing between a
change and `main` — and it has just been configured to pass whenever it knows nothing.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — what coverage does and does not prove

| # | Statement | Answer |
|---|---|---|
| 1 | Coverage measures which lines executed during the test run |  |
| 2 | A test with no assertions still produces coverage |  |
| 3 | 100% line coverage proves the behaviour is verified |  |
| 4 | Branch coverage requires both sides of a condition to execute |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Coverage measures which lines executed during the test run | **Yes** |
| 2 | A test with no assertions still produces coverage | **Yes** |
| 3 | 100% line coverage proves the behaviour is verified | **No** |
| 4 | Branch coverage requires both sides of a condition to execute | **Yes** |

**In `challenge-18.md`:** lines **666–673**, **61–68**, **26**, **63**.

Row 2 is the reason coverage is a floor rather than a target — execution is not verification.

Row 4 is why `branches` is the demanding metric and why a policy written against `lines` (line 26) is the
lenient reading of the same rule.

</details>

---

## Q28 — the three policies

| # | Statement | Answer |
|---|---|---|
| 1 | An absolute threshold can be satisfied without testing the current change |  |
| 2 | A ratchet compares against the last baseline recorded on `main` |  |
| 3 | Diff coverage measures only lines added or modified in the pull request |  |
| 4 | Diff coverage and overall coverage are the same number computed by different tools |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | An absolute threshold can be satisfied without testing the current change | **Yes** |
| 2 | A ratchet compares against the last baseline recorded on `main` | **Yes** |
| 3 | Diff coverage measures only lines added or modified in the pull request | **Yes** |
| 4 | Diff coverage and overall coverage are the same number computed by different tools | **No** |

**In `challenge-18.md`:** lines **26**, **449–464**, **785**, **779–785**.

Row 1 is why the other two exist: a service already at 84% passes clause 1 forever while adding untested
code every week.

Row 4 is the exam's own distractor, verbatim from line 782.

</details>

---

## Q29 — formats and plumbing

| # | Statement | Answer |
|---|---|---|
| 1 | `PublishCodeCoverageResults@2` accepts Cobertura or JaCoCo |  |
| 2 | LCOV is accepted by `PublishCodeCoverageResults@2` |  |
| 3 | Cobertura's `line-rate` is a fraction, not a percentage |  |
| 4 | `--collect:"XPlat Code Coverage"` works without `coverlet.collector` |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `PublishCodeCoverageResults@2` accepts Cobertura or JaCoCo | **Yes** |
| 2 | LCOV is accepted by `PublishCodeCoverageResults@2` | **No** |
| 3 | Cobertura's `line-rate` is a fraction, not a percentage | **Yes** |
| 4 | `--collect:"XPlat Code Coverage"` works without `coverlet.collector` | **No** |

**In `challenge-18.md`:** lines **796**, **793**, **337**, **691–693**.

Row 2 is the format trap: lcov is the natural Node output and is useless to Azure DevOps.

Row 4 fails **silently**, which is what makes it Issue 3 rather than a build error.

</details>

---

## Q30 — the 0% break

| # | Statement | Answer |
|---|---|---|
| 1 | Passing tests confirm coverage was collected |  |
| 2 | A reporter list of `['text']` alone produces no XML file |  |
| 3 | Instrumenting transpiled output can report 0% for the original sources |  |
| 4 | `coverageProvider: 'v8'` avoids Babel instrumentation |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Passing tests confirm coverage was collected | **No** |
| 2 | A reporter list of `['text']` alone produces no XML file | **Yes** |
| 3 | Instrumenting transpiled output can report 0% for the original sources | **Yes** |
| 4 | `coverageProvider: 'v8'` avoids Babel instrumentation | **Yes** |

**In `challenge-18.md`:** lines **666**, **685**, **695–697**, **756**.

Row 1 is the whole break in one line: two pipelines, one of which is silent when it fails.

Row 3 is the subtlest of the four causes, and the one that survives every "the config looks right" review.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each service to its coverage toolchain.

| Service | Toolchain |
|---|---|
| Checkout API (Node.js) |  |
| Inventory service (Python) |  |
| Payment gateway (.NET) |  |
| All three, for Azure DevOps |  |
| All three, for the shell gate |  |
| Python, for the pull request |  |

**Options:** A Cobertura XML passed to `PublishCodeCoverageResults@2` · A single number extracted by `jq` or a small parser · `diff-cover` against `origin/main` with a 90% floor · Jest, configured by `jest.config.js`, emitting lcov, cobertura and json-summary · pytest with `pytest-cov`, configured by `addopts` in `pyproject.toml` · xUnit with `coverlet.collector`, merged by ReportGenerator

<details>
<summary>Show answer</summary>

| Service | Toolchain |
|---|---|
| Checkout API (Node.js) | **Jest, configured by `jest.config.js`, emitting lcov, cobertura and json-summary** |
| Inventory service (Python) | **pytest with `pytest-cov`, configured by `addopts` in `pyproject.toml`** |
| Payment gateway (.NET) | **xUnit with `coverlet.collector`, merged by ReportGenerator** |
| All three, for Azure DevOps | **A Cobertura XML passed to `PublishCodeCoverageResults@2`** |
| All three, for the shell gate | **A single number extracted by `jq` or a small parser** |
| Python, for the pull request | **`diff-cover` against `origin/main` with a 90% floor** |

**In `challenge-18.md`:** lines **46–70**, **147–170**, **222–241** and **275–283**, **589–629**,
**322–354**, **485–489**.

**Three ecosystems, one convergence point.** Whatever the language, the pipeline needs a Cobertura file
for publishing and one scalar for gating — and every stack has a different route to both.

</details>

---

## Q32

Match each policy clause to its implementation.

| Clause | Implementation |
|---|---|
| 80% overall line coverage |  |
| Coverage must not decrease |  |
| 90% coverage on new code |  |
| Trends visible across sprints |  |
| Blocking the merge itself |  |
| Making the number visible to the author |  |

**Options:** A `github-script` comment on the pull request · `coverageThreshold`, `--cov-fail-under=80`, and the gate's comparison · `diff-cover --fail-under=90` and `thresholdNew: 0.90` · One JSON line appended to `trend.jsonl` on every merge to `main` · The gate job configured as a required status check · The ratchet against `trend.jsonl`, exiting non-zero on a drop

<details>
<summary>Show answer</summary>

| Clause | Implementation |
|---|---|
| 80% overall line coverage | **`coverageThreshold`, `--cov-fail-under=80`, and the gate's comparison** |
| Coverage must not decrease | **The ratchet against `trend.jsonl`, exiting non-zero on a drop** |
| 90% coverage on new code | **`diff-cover --fail-under=90` and `thresholdNew: 0.90`** |
| Trends visible across sprints | **One JSON line appended to `trend.jsonl` on every merge to `main`** |
| Blocking the merge itself | **The gate job configured as a required status check** |
| Making the number visible to the author | **A `github-script` comment on the pull request** |

**In `challenge-18.md`:** lines **61–68** and **150** and **325**, **449–464**, **487** and **215**,
**410–418**, **294–306**, **115–140**.

**The last two rows are the pair the exam separates.** A comment tells the author; only a required status
check stops the merge. Everything else in this table computes a number — these two decide what happens to
it.

</details>

---

## Q33

Arrange what must happen for the Node diff-coverage check to produce a meaningful number.

**Items:** Compare the covered proportion against the 90% floor · Check out with full history ·
Parse `lcov.info` for only those files · Fetch the base branch as a remote-tracking ref · List the files
changed against the merge base

<details>
<summary>Show answer</summary>

### Answer

1. Check out with full history — lines **93–94**
2. Fetch the base branch as a remote-tracking ref — line **516**
3. List the files changed against the merge base — line **517**
4. Parse `lcov.info` for only those files — lines **525–545**
5. Compare the covered proportion against the 90% floor — lines **547–552**

**Steps 1 and 2 are both required and neither is sufficient.** `fetch-depth: 0` gives you history;
`git fetch origin main:refs/remotes/origin/main` gives you the ref to compare against.

**And every one of the first three fails *upward*.** No history, no ref or no changed files each produce
an empty list, and an empty list gives `totalLines = 0`, which line 547 turns into **100%**. The check
reports a perfect score for having measured nothing.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| 42 tests pass, coverage reads 0% in Azure DevOps |  |
| The coverage file exists but the publish task finds nothing |  |
| `dotnet test` succeeds and writes no coverage at all |  |
| Coverage reports on files nobody wrote |  |
| Coverage reads 95% on a barely tested service |  |
| The gate passes on a run where the upload step failed |  |

**Options:** An `if [ -f ... ]` guard skipping the check · An output path that does not match `summaryFileLocation` · `collectCoverageFrom` absent, so untested files are never counted · `coverlet.collector` not referenced by the test project · Instrumentation applied to transpiled output without source maps · Only the `text` reporter configured — no XML file produced

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| 42 tests pass, coverage reads 0% in Azure DevOps | **Only the `text` reporter configured — no XML file produced** |
| The coverage file exists but the publish task finds nothing | **An output path that does not match `summaryFileLocation`** |
| `dotnet test` succeeds and writes no coverage at all | **`coverlet.collector` not referenced by the test project** |
| Coverage reports on files nobody wrote | **Instrumentation applied to transpiled output without source maps** |
| Coverage reads 95% on a barely tested service | **`collectCoverageFrom` absent, so untested files are never counted** |
| The gate passes on a run where the upload step failed | **An `if [ -f ... ]` guard skipping the check** |

**In `challenge-18.md`:** lines **683–685**, **687–689**, **691–693**, **695–697**, **53–58**, **322**.

**The first four are the break scenario; the last two are the design.** All six share one property: the
pipeline is green and the information is wrong.

</details>

---

## Q35

Match each configuration key to its effect.

| Key | Effect |
|---|---|
| `collectCoverageFrom` |  |
| `coverageThreshold` |  |
| `omit` under `[tool.coverage.run]` |  |
| `exclude_lines` |  |
| `ExcludeByFile` |  |
| `-reporttypes:"Html;Cobertura;JsonSummary"` |  |

**Options:** Fails the Jest run below the configured percentages · Produces one output for people, one for Azure DevOps and one for `jq` · Removes files from measurement entirely · Removes matching lines from the denominator · Sets which files are measured, including ones no test imports · The coverlet equivalent, applied through run settings

<details>
<summary>Show answer</summary>

| Key | Effect |
|---|---|
| `collectCoverageFrom` | **Sets which files are measured, including ones no test imports** |
| `coverageThreshold` | **Fails the Jest run below the configured percentages** |
| `omit` under `[tool.coverage.run]` | **Removes files from measurement entirely** |
| `exclude_lines` | **Removes matching lines from the denominator** |
| `ExcludeByFile` | **The coverlet equivalent, applied through run settings** |
| `-reporttypes:"Html;Cobertura;JsonSummary"` | **Produces one output for people, one for Azure DevOps and one for `jq`** |

**In `challenge-18.md`:** lines **53–58**, **61–68**, **154–158**, **160–167**, **273**, **283**.

**Five of these six change the number; one changes who can read it.** Sorting a coverage setting into
"measurement", "enforcement" or "presentation" answers most questions about it before you know the tool.

</details>

---

# Section F — Hot area

---

## Q36

```javascript
module.exports = {
  [BLANK 1]: [
    'src/**/*.js',
    '!src/**/__tests__/**',
    '!src/migrations/**',
  ],
  coverageReporters: ['text', 'text-summary', 'lcov', '[BLANK 2]', 'json-summary'],
  [BLANK 3]: {
    global: { branches: 80, functions: 80, lines: 80, statements: 80 },
  },
};
```

<details>
<summary>Show answer</summary>

### Answer

| Blank | Value |
|---|---|
| 1 | `collectCoverageFrom` |
| 2 | `cobertura` |
| 3 | `coverageThreshold` |

**In `challenge-18.md`:** lines **53–68**.

**Blank 1 is measurement, blank 3 is enforcement, blank 2 is presentation** — the three roles from Q35, in
one file.

**Blank 2 is Issue 1 in advance.** Drop `cobertura` and `PublishCodeCoverageResults@2` has nothing to
publish (line 685). Drop `json-summary` and the shell gate has nothing to read (line 323). Drop `lcov` and
the diff-coverage parser has nothing to parse (line 527). **Each reporter is load-bearing for exactly one
consumer.**

</details>

---

## Q37

```toml
[tool.pytest.ini_options]
[BLANK 1] = "--cov=src --cov-report=xml:coverage/coverage.xml --cov-report=term-missing [BLANK 2]=80"

[tool.coverage.report]
[BLANK 3] = [
    "pragma: no cover",
    "raise NotImplementedError",
]
```

<details>
<summary>Show answer</summary>

### Answer

| Blank | Value |
|---|---|
| 1 | `addopts` |
| 2 | `--cov-fail-under` |
| 3 | `exclude_lines` |

**In `challenge-18.md`:** lines **148–150** and **160–167**.

**Blank 1 is why the CI step is a bare `pytest`** (line 200) — the configuration travels with the code, so
local runs and CI runs cannot drift.

**Blank 2 is the gate that fires first**, inside the test process, before any artifact exists. It takes a
**percentage** — while the `orgoro/coverage` action on the same service takes a **fraction** (line 214).
Check the units on every threshold you write.

**Blank 3 shrinks the denominator** (Q7), and `xml:coverage/coverage.xml` in blank 1 is the explicit output
path that prevents Issue 2 (line 721).

</details>

---

## Q38

```bash
dotnet test \
  --collect:"[BLANK 1]" \
  --results-directory ./coverage \
  -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=[BLANK 2]

reportgenerator \
  -reports:"coverage/**/coverage.cobertura.xml" \
  -targetdir:"coverage/report" \
  -reporttypes:"Html;Cobertura;[BLANK 3]"
```

<details>
<summary>Show answer</summary>

### Answer

| Blank | Value |
|---|---|
| 1 | `XPlat Code Coverage` |
| 2 | `cobertura` |
| 3 | `JsonSummary` |

**In `challenge-18.md`:** lines **266–283**.

**Blank 1 names a collector that must be supplied by a package.** Without `coverlet.collector` in the
`.csproj` (line 233) this exact command runs, passes, and produces nothing (Q8).

**Blank 3 is what the gate reads.** ReportGenerator's `JsonSummary` writes `Summary.json`, and line 348
pulls `.summary.linecoverage` out of it.

**And ReportGenerator earns its place by merging.** Coverlet writes one
`coverage.cobertura.xml` per test assembly under a GUID directory — hence the `**` in `-reports:` — and a
gate that read only the first file would report one assembly's coverage as the whole service's.

</details>

---

## Q39

```bash
FAILED=false

if [ -f "./coverage-artifacts/coverage-node/coverage-summary.json" ]; then
  NODE_COV=$([BLANK 1] '.total.lines.pct' ./coverage-artifacts/coverage-node/coverage-summary.json)
  if (( $(echo "$NODE_COV < 80" | [BLANK 2]) )); then
    echo "::error::Checkout API coverage below threshold"
    FAILED=true
  fi
fi

if [ "$FAILED" = true ]; then
  [BLANK 3]
fi
```

<details>
<summary>Show answer</summary>

### Answer

| Blank | Value |
|---|---|
| 1 | `jq` |
| 2 | `bc -l` |
| 3 | `exit 1` |

**In `challenge-18.md`:** lines **319–359**.

**Blank 2 exists because bash has no floating point** (Q11), and blank 3 is the only line in the whole job
that stops a merge.

**But read the shape, not just the blanks.** `FAILED` starts as `false` and is only ever set by a
comparison that runs. **Every path that skips the comparison ends at `exit 0`** — which is why Q26 is a
"No" and why the fix is to require the file rather than test for it.

</details>

---

## Q40

```bash
CURRENT=$(jq '.total.lines.pct' ./current-coverage/coverage-summary.json)

if [ -f ".coverage-history/trend.jsonl" ]; then
  BASELINE=$([BLANK 1] .coverage-history/trend.jsonl | jq '.checkout_api')

  if (( $(echo "$CURRENT [BLANK 2] $BASELINE" | bc -l) )); then
    echo "::error::Coverage decreased. Coverage ratchet policy violated."
    [BLANK 3]
  fi
fi
```

<details>
<summary>Show answer</summary>

### Answer

| Blank | Value |
|---|---|
| 1 | `tail -1` |
| 2 | `<` |
| 3 | `exit 1` |

**In `challenge-18.md`:** lines **451–461**.

**Blank 1 is what makes an append-only file a baseline.** `trend.jsonl` gains one line per merge (line
418); the last line is the current state of `main`, and the whole file is requirement 4's trend.

**Blank 2 is `<` and not `<=` for a reason.** Equal coverage passes — a ratchet forbids **going backwards**,
not standing still. Requiring an increase on every pull request would block every bug fix that adds no
lines.

**And note what happens when the `if` is false**: line 466 prints "No baseline found" and the job succeeds
(Q21). A cache that silently stops restoring turns the ratchet off with no visible change.

</details>

---

## Q41

```bash
git fetch origin main:refs/remotes/origin/main

diff-cover coverage/coverage.xml \
  --compare-branch=[BLANK 1] \
  --[BLANK 2]=90 \
  --markdown-report coverage/diff-cover.md
```

<details>
<summary>Show answer</summary>

### Answer

| Blank | Value |
|---|---|
| 1 | `origin/main` |
| 2 | `fail-under` |

**In `challenge-18.md`:** lines **484–489**.

**Two blanks, and the third element is the one already written for you.** `diff-cover` takes the **same
Cobertura XML** the overall coverage came from — the difference is not the data, it is which lines are
counted.

**Blank 2 is what makes it a gate.** Without `--fail-under`, `diff-cover` prints a report and exits zero,
and the 90% requirement becomes a suggestion in a PR comment.

**And the `git fetch` above it is not optional.** The comparison needs `origin/main` to resolve; without
it, `diff-cover` cannot compute a diff and the step fails — which, unusually for this challenge, is the
safe direction.

</details>

---

# Section G — Case study

**Contoso Ltd — three services, one policy.** Low-coverage services see three times the production
incidents. No merge below 80% overall line coverage; coverage may not decrease against the base branch;
new code must reach 90%; trends must be visible across sprints. The stacks are Node with Jest, Python with
pytest, and .NET with xUnit. Coverage runs in GitHub Actions, with an Azure Pipelines equivalent for the
teams still on Azure DevOps.

---

## Q42

A service reports 96% line coverage. A reviewer notices its test file asserts nothing — it calls each
function and discards the result. Which policy clause would have caught this?

- A. The 80% overall line-coverage threshold
- B. The ratchet against the recorded baseline
- C. The 90% diff-coverage floor on new lines
- D. None — coverage measures execution only

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-18.md`:** lines **26–28** against the definition at lines **666–673**.

**All three clauses are computed from the same signal: did this line run.** An assertion-free test runs
every line, so it satisfies the threshold, satisfies the ratchet, and satisfies diff coverage at 100%.

**This is the honest limit of the whole challenge** and the reason coverage policy is paired with code
review rather than replacing it. Coverage answers "what did nobody execute?" — a genuinely useful
question — and cannot answer "was the behaviour checked?"

**Why the exam still asks it.** The distractors are all real policies, and the reflex is to pick the
strictest one. The discipline is to notice that **strictness on the wrong metric changes nothing.**

**What does catch it: mutation testing, and review.** Neither is in this challenge; knowing that coverage
does not substitute for them is the point.

</details>

---

## Q43

The Python service sits at 61% overall and cannot realistically reach 80% this quarter. The team still
wants every new change well tested. What do you configure?

- A. Lower the overall threshold to 60% for this service
- B. Exclude the untested modules with `omit` for now
- C. A ratchet plus a 90% diff floor on changed lines
- D. Require 100% coverage on newly added files only

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-18.md`:** lines **27–28**, **449–464**, **485–489**.

**The ratchet stops the number falling; diff coverage forces every change to arrive tested.** Together
they raise the figure as a by-product of ordinary work, with no cliff and no waiver.

**Why A is a waiver with a number on it.** It unblocks the team and removes the pressure entirely, and the
threshold ratchets *downward* the next time someone finds it inconvenient.

**Why B is the same move disguised as configuration** (Q18). The untested modules are exactly the ones the
metric exists to point at, and omitting them makes 61% become 85% with no change to the risk at line 24.

**Why D sounds strict and covers almost nothing.** Most changes modify existing files. Diff coverage
measures **new and modified lines wherever they are**, which is the version that actually bites.

</details>

---

## Q44

A developer reports that the Node diff-coverage step always passes, even on a pull request that adds a
hundred untested lines. Looking at Task 6, what is wrong?

- A. The 90% threshold is set far too low
- B. `CHANGED_FILES` refers to a nonexistent step output
- C. `lcov.info` is not being generated by Jest
- D. `--fail-under` is missing from the diff-cover call

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-18.md`:** lines **512–556**.

```yaml
        env:
          CHANGED_FILES: ${{ steps['changed-files'].outputs.files }}
```

**There is no step with the id `changed-files`.** The file list is computed in the shell at line 517 as a
local variable, which the `node -e` script cannot see. The environment variable resolves to an empty
string, `split` yields one empty entry, `filter(Boolean)` removes it, and the loop matches nothing.

**Then line 547 does the damage:**

```javascript
const pct = totalLines > 0 ? ((coveredLines / totalLines) * 100).toFixed(2) : 100;
```

**Zero measured lines is reported as 100%.** The check passes, prints a perfect score, and has evaluated
nothing.

**The fix is to write the list where the script can read it** — either export it in the same `run` block,
or publish it as a real step output and reference that id.

**And notice the family resemblance.** This is the same defect as the empty-diff case in Q33 and the
missing-artifact case in Q12: **the absence of data being scored as success.**

</details>

---

## Q45

The .NET team adds `ExcludeByFile="**/Migrations/**"` and coverage jumps from 74% to 83%, passing the
gate. Is this acceptable?

- A. It depends — defensible, but reviewed like a code change
- B. Yes, unconditionally — exclusions are a supported feature
- C. No — exclusions are never legitimate in a gated pipeline
- D. Yes, because line coverage is the wrong metric anyway

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-18.md`:** line **273**, alongside the equivalents at lines **57** and **154–158**.

**Generated migrations are a reasonable exclusion.** They are produced by a tool, they run once against a
database, and unit tests of them assert that the generator generated what the generator generated.

**And the jump from 74% to 83% is the part to look at.** Nine points arrived with no test written and no
defect prevented. If that is the difference between blocked and merged, the gate has been satisfied by a
configuration change — which is a decision, and decisions belong in review.

**Why B and C are both absolutes the exam offers to skip the judgement.** Every stack here ships an
exclusion mechanism precisely because some code cannot be usefully tested; every one of them is also the
easiest way to make a red gate green.

**The practical rule: exclusions belong in the repository, in a reviewed file, with a reason** — never as
a flag added to a pipeline to unblock a release.

</details>

---

## Q46

A pull request touching only the Python service is blocked because the Node coverage artifact is
missing — its job did not run. Which change fixes this **without** weakening the gate?

- A. Wrap the Node check in a file-existence test
- B. Remove the Node check from the gate entirely
- C. Path-filter each job; the gate needs each that ran
- D. Set the Node threshold to 0 until it is fixed

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-18.md`:** lines **306–354**, and the design discussed at Q26.

**Path filters express the actual intent.** "This PR did not touch the checkout API" is a fact the
workflow can determine from the diff — and it is a completely different fact from "the coverage file is
absent", which is what a file-existence test observes.

**Why A is exactly what Q26 rejects.** It fixes this symptom and silently accepts every other cause of a
missing artifact, including a failed upload on a service the PR *did* change.

**Why B and D delete the requirement rather than scope it.** Both leave the Node service permanently
ungated, which is the outcome the policy at line 24 exists to prevent.

**The general shape, worth carrying past this exam: scope a gate by *what changed*, never by *what data
happens to be present*.** The first is a decision; the second is an accident.

</details>

---

## Q47

The Azure DevOps team publishes coverage with `PublishCodeCoverageResults@2` and reports that the Code
Coverage tab is empty for the Node service, though the job succeeds. What do you check first?

- A. Whether the tests passed on the build agent
- B. Whether `cobertura` is listed and the file exists
- C. Whether the agent supports coverage collection
- D. Whether branch coverage is enabled in the config

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-18.md`:** lines **60**, **589–592**, **683–689**, **709–713**.

**Issues 1 and 2 in one check.** The tab is empty either because the Cobertura file was never produced
(wrong reporter list) or because it was produced somewhere `summaryFileLocation` does not point.

**The verification is two commands and settles it:**

```bash
npx jest --coverage
ls -la coverage/cobertura-coverage.xml
```

**File missing, it is Issue 1. File present, it is Issue 2** — and the fix is either the path in the task
or the output path in the config (lines 716–730).

**Why A is already answered by "the job succeeds"**, and why it is the wrong instinct anyway: passing tests
say nothing about coverage collection (Q1).

**Why D changes which numbers are shown, not whether any are** — and note that the publish task and the
gate read the same file for different purposes, so an empty tab often means the gate is reading nothing
either.

</details>

---

## Q48

Which single sentence best states why coverage gates are worth building, given everything above?

- A. High coverage guarantees fewer production defects
- B. It finds code no test runs and stops that set growing
- C. Coverage replaces code review for well-tested services
- D. Coverage is a management metric with no engineering value

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-18.md`:** lines **24–29**, with the limits at lines **666–673**.

**The director's observation at line 24 is a correlation** — low-coverage services see three times the
incidents — and the policy built on it is careful about what it claims. It does not assert that 80%
prevents defects. It asserts that **untested code is where defects concentrate**, and it stops that region
expanding.

**Which is why the design is three policies and a comment, not one number.** The threshold sets a floor,
the ratchet stops erosion, diff coverage taxes new code, and the PR comment puts the figure in front of
the author while the change is still cheap to fix.

**Why A overclaims** in exactly the way Q42 disproves — coverage measures execution, not verification —
and **why C is the failure mode that follows from believing A.**

**Why D throws away a real signal.** A module at 0% is a fact worth knowing, and it is the one thing
coverage reports with complete reliability.

</details>

---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Coverage read as proof of testing** | Q27, Q42, Q48 | It measures execution; assertion-free tests score high |
| **Line coverage trusted over branch coverage** | Q4, Q27 | A covered line can have an untaken `else` |
| **`collectCoverageFrom` omitted** | Q2, Q25, Q34 | Untested files vanish from the denominator |
| **Exclusions used to fix a red gate** | Q7, Q18, Q25, Q45 | They shrink the denominator, not the risk |
| **`if [ -f ... ]` guards around every check** | Q12, Q21, Q26, Q46 | No data becomes a pass — require the file |
| **Missing baseline treated as success** | Q21, Q40 | A cache that stops restoring turns the ratchet off |
| **Empty changed-file list scored as 100%** | Q33, Q44 | Zero measured lines must fail, not pass |
| **`CHANGED_FILES` from a step id that does not exist** | Q44 | Shell variables are not step outputs |
| **`needs:` across two workflow files** | Q13 | It resolves within one file only |
| **Shallow checkout with a diff-based gate** | Q16, Q33 | `fetch-depth: 0` **and** an explicit base-branch fetch |
| **LCOV sent to `PublishCodeCoverageResults@2`** | Q3, Q29 | Cobertura or JaCoCo only |
| **Cobertura `line-rate` read as a percentage** | Q10, Q29 | It is a fraction — multiply by 100 |
| **Fraction and percentage thresholds mixed** | Q19, Q37 | `0.80` for the action, `80` for `--cov-fail-under` |
| **`--collect:"XPlat Code Coverage"` without coverlet** | Q8, Q29, Q38 | The collector is silent, not loud |
| **Passing tests taken as proof coverage ran** | Q1, Q22, Q30, Q47 | Two pipelines; the second fails quietly |
| **Transpiled output instrumented instead of source** | Q22, Q30 | Source maps, or `coverageProvider: 'v8'` |
| **A comment or artifact mistaken for a gate** | Q17, Q25, Q32 | Only a required status check blocks a merge |

---

# What to memorise

**In `challenge-18.md`:** lines **24–36**, **46–70**, **147–170**, **222–290**, **317–361**, **449–467**,
**480–557**, **683–769**.

```text
WHAT COVERAGE IS
  it measures which lines RAN. not whether anything CHECKED them.
  a test with no assertions -> full coverage, zero verification
  lines/statements  = forgiving      branches = the demanding one (both sides of every if)
  use it as a FLOOR: it reliably names code nobody executed

THREE POLICIES - and only one changes behaviour today
  THRESHOLD  80% overall    -> satisfied by history. says nothing about this PR
  RATCHET    never below the last baseline on main  -> stops erosion. "<" not "<=" (equal passes)
  DIFF       90% of the lines THIS PR added/modified -> the one that makes an author write a test
  requirement 4 (trends) is REPORTING - one JSON line per merge. it gates nothing.

THE 0% FAMILY - every one is PLUMBING, and every one is SILENT
  1  reporter list has no cobertura   -> console output only, no XML       (line 685)
  2  output path != summaryFileLocation -> the file exists, elsewhere      (line 689)
  3  coverlet.collector not referenced -> XPlat collector emits NOTHING    (line 693)
  4  babel/ts-jest without source maps -> 0% for the real sources          (line 697)
  diagnose it with:  run the tests, then ls for the coverage file
```

```javascript
// jest.config.js                                     (lines 46-70)
collectCoverageFrom: ['src/**/*.js', '!src/**/__tests__/**', '!src/migrations/**'],
//  WITHOUT THIS only files a test imported are measured -> untested modules are INVISIBLE
coverageReporters: ['text', 'text-summary', 'lcov', 'cobertura', 'json-summary'],
//  text=humans  lcov=diff parser + GitHub  cobertura=PublishCodeCoverageResults@2  json-summary=jq
coverageThreshold: { global: { branches: 80, functions: 80, lines: 80, statements: 80 } },
//  fails the JEST RUN. the earliest of the three enforcement points
coverageProvider: 'v8',   // the fix for Issue 4 - bypasses babel instrumentation
```

```toml
# pyproject.toml                                      (lines 147-170)
[tool.pytest.ini_options]
addopts = "--cov=src --cov-report=xml:coverage/coverage.xml --cov-report=term-missing --cov-fail-under=80"
#   addopts -> the CI step is a bare `pytest`. config travels with the code
#   xml:PATH -> the explicit output path that prevents Issue 2
#   --cov-fail-under takes a PERCENTAGE (80). orgoro/coverage takes a FRACTION (0.80)
[tool.coverage.run]
omit = ["src/migrations/*", "tests/*"]        # removes FILES from the denominator
[tool.coverage.report]
exclude_lines = ["pragma: no cover", "raise NotImplementedError"]   # removes LINES
```

```bash
# .NET                                                (lines 222-290)
<PackageReference Include="coverlet.collector" Version="6.0.2" />   # WITHOUT IT: silent, no output
dotnet test --collect:"XPlat Code Coverage" --results-directory ./coverage \
  -- ...DataCollector.Configuration.Format=cobertura
reportgenerator -reports:"coverage/**/coverage.cobertura.xml" \
                -targetdir:"coverage/report" -reporttypes:"Html;Cobertura;JsonSummary"
#   MERGES one cobertura file per test assembly. JsonSummary -> Summary.json -> .summary.linecoverage
```

```bash
# The gate                                            (lines 317-361)
NODE_COV=$(jq '.total.lines.pct' .../coverage-summary.json)          # already a PERCENTAGE
PY_COV=$(python3 -c "... float(root.attrib['line-rate']) * 100")     # cobertura = a FRACTION
DOTNET_COV=$(jq '.summary.linecoverage' .../Summary.json)            # already a PERCENTAGE
if (( $(echo "$NODE_COV < 80" | bc -l) )); then FAILED=true; fi
#   bc -l because bash has NO floating point
if [ "$FAILED" = true ]; then exit 1; fi        # the ONLY line that stops a merge
#   every check sits inside `if [ -f ... ]`  ->  NO FILE = NO CHECK = GREEN.  fail-open.
#   fix: require the file. missing evidence must be a FAILURE.

# Ratchet                                             (lines 449-467)
BASELINE=$(tail -1 .coverage-history/trend.jsonl | jq '.checkout_api')   # appended per merge to main
if (( $(echo "$CURRENT < $BASELINE" | bc -l) )); then exit 1; fi         # "<" - equal is allowed

# Diff coverage                                       (lines 480-557)
fetch-depth: 0                                  # AND an explicit fetch of the base branch:
git fetch origin main:refs/remotes/origin/main
git diff --name-only origin/main...HEAD         # three dots = changes since the MERGE BASE
diff-cover coverage/coverage.xml --compare-branch=origin/main --fail-under=90
#   an empty changed-file list -> totalLines 0 -> reported as 100%. measuring nothing SCORES PERFECT
```

```yaml
# Azure Pipelines                                     (lines 559-658)
- task: PublishCodeCoverageResults@2
  inputs: {summaryFileLocation: '$(System.DefaultWorkingDirectory)/.../cobertura-coverage.xml'}
#   COBERTURA or JACOCO only. NOT lcov, NOT html. and publishing is VISIBILITY, not a gate
- task: PowerShell@2                    # the gate: parse line-rate, x100, compare, exit 1
#   [math]::Round([double]$xml.coverage.'line-rate' * 100, 2)
#   it Get-ChildItem -Recurse for the files -> finds none, checks none, PASSES. same fail-open shape
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Domain 3 is finished — move on |
| 38–43 | Re-read the trap index and the three-policies block, then move on |
| 30–37 | Write the four causes of 0% and the three policies from memory, then retake |
| Below 30 | Redo Tasks 1, 4 and 6 hands-on, then work the break scenario before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 18.

:::danger The three rules

**Coverage is a floor, not a score.** It names code nobody executed, and that is genuinely worth knowing.
It cannot tell you whether anything was verified — an assertion-free test covers everything and proves
nothing.

**Pick the policy that changes what someone does today.** A threshold is met by history and a ratchet only
prevents harm; **diff coverage** is what makes the author of a change write a test, and it is the clause
that lets a 61% service improve without a waiver.

**Ask every gate what it does with no data.** In this challenge, a missing artifact, a missing baseline and
an empty changed-file list all report success. **A gate that cannot confirm must refuse** — otherwise the
first thing to break is the checking, and nothing tells you.

:::
