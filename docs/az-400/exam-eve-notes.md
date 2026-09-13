---
sidebar_position: 6
toc_max_heading_level: 3
title: "Exam-eve notes (short & fun)"
sidebar_label: "Exam-eve notes"
---

# AZ-400 exam-eve notes 🍿

> DRAFT - the 15-minute cram page is still being written and fact-checked. Refresh this file shortly.

## 🏁 Part 1 — Domain 1 · Processes & communications

Warm-up lap! 🏎️ This domain is worth 10–15% of the exam and covers how work flows, gets tracked, gets documented and gets shouted about. One theme wins almost every question: a rule nobody enforces is just a suggestion, so pick the option where doing the work updates the record by itself.

### 🌿 01 · Pick your branching dance

A DJ plays for the crowd, not the club's name: branching strategy depends on how often you ship and how many versions you support, never on the word "GitHub". Ship continuously → GitHub Flow; several supported versions → GitFlow; mature team behind feature flags → trunk-based. Then hire a bouncer: branch protection with CI jobs listed by exact name as required checks.

**🧠 Lock these in:**
- `strict: true` = branch must be up to date with main before merging.
- `enforce_admins: true` = admins can't push straight to main either.
- `dismiss_stale_reviews` (classic) = `dismiss_stale_reviews_on_push` (ruleset): new commits cancel approvals.
- Ruleset `non_fast_forward` blocks force pushes; `deletion` blocks deletes.
- Required check names must EXACTLY match CI job names, or the gate silently does nothing.
- `allow_merge_commit=false` = linear history.

**🪤 Trap:**
- They offer a CI job to block direct pushes → wrong, because CI runs after the push is accepted.
- They offer `require_last_push_approval` for stale approvals → wrong, because it's about who approves, not new commits.

**🎯 Question says → you pick:**
- "PR merged although CI failed" → check names don't match what CI reports.
- "Two green PRs broke main" → `strict: true`.
- "Merge the moment everything passes" → auto-merge, which bypasses nothing.

**💡 Hook:** The bouncer fails if VIPs skip the queue (`enforce_admins`) or the guest list is misspelled (check names).

### 📋 02 · One board, many views

Contoso tracks work on sticky notes, like a café taking orders on napkins. The fix is one system (GitHub Projects v2 and/or Azure Boards) where doing the work updates the record. Remember two splits: a field stores data while a view displays it, and `AB#1234` only links while `Fixes AB#1234` also moves the item on merge.

**🧠 Lock these in:**
- Sprint = `ITERATION`, Story Points = `NUMBER` (sums into velocity), Priority = `SINGLE_SELECT`.
- One project, three views: board (devs), table (PMs), roadmap (leadership).
- `Fixes AB#` needs the Azure Boards GitHub App AND the GitHub connection in Azure DevOps.
- Area path = WHO; iteration path = WHEN; `@CurrentIteration` never needs editing.
- WIQL `UNDER` includes child areas; `=` doesn't.
- CODEOWNERS only requests; `require_code_owner_reviews` enforces.

**🪤 Trap:**
- They offer `Closes AB#1234` → wrong, because Closes is GitHub Issues vocabulary; Boards uses `Fixes`.
- They offer `GITHUB_TOKEN` for an org project → wrong, because it's repo-scoped; use `secrets.PROJECT_TOKEN`.

**🎯 Question says → you pick:**
- "Work item links but never changes state" → add `Fixes`.
- "Query empty after stories moved to a sub-area" → `UNDER`, not `=`.
- "No blank issues allowed" → `blank_issues_enabled: false`.

**💡 Hook:** CODEOWNERS is a doorbell, not a lock; branch protection locks the door. 🔔

### 🔗 03 · Follow the parcel

Traceability is parcel tracking: work item → PR → commit → build → deployment. People write the labels (`#87`, `AB#2100`) and machines scan the barcode (the merge commit SHA). When production breaks, scan backwards from the deployment SHA. The rule: each link is enforced by a required check, not requested by convention.

**🧠 Lock these in:**
- `feat` = minor; `fix` and `perf` = patch; `docs`/`chore`/`ci` = no bump.
- Breaking = `feat(api)!:` or a `BREAKING CHANGE:` footer; no "breaking" type exists.
- commitlint severity: 0 off, 1 warning, 2 error; subject max 72.
- Husky `commit-msg` is skippable (`--no-verify`), so make commitlint a required CI check too.
- `fetch-depth: 0` = full history (default is depth 1).
- `core.setFailed` fails the job; `::warning::` doesn't.

**🪤 Trap:**
- They offer a shallow checkout "to keep CI fast" → wrong, because the commit range vanishes and the linter passes while checking nothing.
- They offer the audit log as prevention → wrong, because it only records; branch protection prevents.

**🎯 Question says → you pick:**
- "Issue stays open after merge" → PR merged into a non-default branch.
- "Find branch protection bypasses" → `action:protected_branch.policy_override`.
- "Evidence must outlive retention" → Azure DevOps audit streaming to Log Analytics.

**💡 Hook:** Reference = sticky note, keyword = rubber stamp; Feat is Minor, Fix and Perf are Patches, a Bang (!) is Big.

### 📊 04 · DORA's four numbers

In a pizza shop 🍕, throughput is how often pizzas go out (deployment frequency) and order-to-doorstep time (lead time); stability is how many arrive wrong (change failure rate) and how fast replacements land (MTTR). A pizza burnt in the kitchen never left, so a failed CI build is NOT a change failure. Convert to the table's units and read off the level.

**🧠 Lock these in:**
- Deploy frequency: Elite on demand (multiple per day); High once per day to once per week.
- Lead time: Elite less than 1 hour; High 1 day to 1 week.
- MTTR: Elite less than 1 hour; High less than 1 day.
- Change failure rate: Elite 0-15%; High 16-30%.
- Widgets (Burndown, Velocity, Cycle Time) show FLOW metrics, not DORA.
- Analytics 401 = extension missing or PAT lacks Analytics (read); 400 = bad syntax.

**🪤 Trap:**
- They offer "3 deploys a week" as Elite → wrong, because Elite is on demand; that's High.
- They offer more manual approvals to cut failures → wrong, because testing, progressive rollouts and feature flags are the direct fix.

**🎯 Question says → you pick:**
- "Deploy count is zero despite deployments" → environment name case mismatch.
- "Lead time skewed by stale PRs" → use the median.
- "Query portable across Agile/Scrum/CMMI" → filter on `StateCategory`.

**💡 Hook:** Two "under 1 hour" rows, and Elite can fail up to 15%, not zero.

### 📚 05 · Docs that stay true

Docs are a fridge list, and the exam asks: how does it stay true? Best is a fridge that prints its own list (generated OpenAPI, changelog from commits), then docs-as-code reviewed in the PR, then a wiki, and last a Visio file on a share. Pick what stays current as a by-product of work already happening.

**🧠 Lock these in:**
- Code wiki `--type codeWiki` + `--mapped-path "/docs"`: any branch, PR review.
- Provisioned wiki `--type projectWiki`: one branch, no PR review; suits non-engineer onboarding.
- Azure DevOps Wiki Mermaid uses `::: mermaid` ... `:::`; GitHub uses backticks.
- release-drafter reads PR LABELS, keeps a DRAFT, runs on `GITHUB_TOKEN`.
- `conventional-changelog` reads commit TYPES and needs `fetch-depth: 0`.
- `gh release create vX --generate-notes` = zero config, flat list.

**🪤 Trap:**
- They offer a committed PNG "because it's versioned" → wrong, because binary isn't diffable.
- They offer `--generate-notes` for "grouped by type" → wrong, because it's a flat list.

**🎯 Question says → you pick:**
- "Draft release is empty" → PRs lack matching labels; add a catch-all category and the autolabeler.
- "Always-current API docs" → `@openapi` annotations → `swagger-jsdoc` → Redocly → GitHub Pages.
- "Pages 404 with workflow build" → missing `pages: write` or `id-token: write`.

**💡 Hook:** LABELS for the DRAFTer, TYPES for the CHANGElog; colons for ADO, backticks for GitHub.

### 📞 06 · Who calls whom?

Integrations are phone calls: who dials, and when? A webhook is GitHub calling YOU (outbound POST), so check caller ID: the HMAC signature. `repository_dispatch` is an external system calling GitHub with a token and `client_payload`; `workflow_dispatch` is a human pressing Run. `AB#` links need the Boards app installed AND the repo connected in Azure DevOps.

**🧠 Lock these in:**
- `X-Hub-Signature-256` = HMAC-SHA256 of the body with the secret; signs, doesn't encrypt.
- Compare with `crypto.timingSafeEqual`, never `===`.
- `if (secret && signature)` fails open when `GITHUB_WEBHOOK_SECRET` is unset.
- Caller sends `event_type`; workflow reads `github.event.action`, listed under `types:`.
- Service hook: `publisherId` (`pipelines`/`tfs`), `eventType`, consumer `webHooks`/`httpRequest`.
- Teams: Adaptive Card is current; MessageCard is legacy.

**🪤 Trap:**
- They offer `workflow_dispatch` for a system trigger → wrong, because it's the manual button.
- They offer POST to rotate a secret → wrong, because it adds a second hook with the old secret live; use PATCH.

**🎯 Question says → you pick:**
- "Dispatch accepted, nothing runs" → `event_type` missing from `types:`.
- "Deliveries return 401" → secret mismatch with `GITHUB_WEBHOOK_SECRET`.
- "Externally triggered prod deploy stays gated" → `environment: production` plus protection rules.

**💡 Hook:** Webhook = they call you (check caller ID); dispatch = you call them (bring the PIN).

---


## 🏁 Part 2 — Domain 2 · Source control

Worth 10–15% of the exam: a small slice with a lot of Git flags. The big theme is **match the Git tool to the real problem** (cadence, access, file size, history, secrets) **and let the platform enforce it**, not good intentions. Fingers on the keyboard. 🧤

### 🚌 07 · Branching: cadence picks the strategy

Think buses. Daily or weekly is a city bus: branches live less than 24 hours and feature flags hide unfinished work (trunk-based). Bi-weekly is a coach with a ticket check: PR, merge, deploy (GitHub Flow). Monthly, quarterly or old passengers still aboard is a long-haul train dragging an old carriage (release branching). The rule: cadence picks the strategy, and only release branching supports a shipped old version.

**🧠 Lock these in:**

- Trunk-based REQUIRES feature flags AND CD; rollback = flag off
- GitHub Flow: no release branches; a hotfix is the same PR flow with faster review
- Release fix → `git cherry-pick <sha>` onto main; never merge the whole release branch back
- Forward-integrate with `git merge main --no-edit`; never rebase a shared branch (new SHAs)
- `--no-ff` forces a merge commit; `--ff-only` refuses unless it can fast-forward
- Direct push to main: `git branch` → `git reset --hard HEAD~3` → `git push --force-with-lease` → PR
- Drift job warns when BEHIND is greater than 50 and needs `fetch-depth: 0`

**🪤 Trap:**

- They offer one strategy for every team → wrong, because each team's cadence picks its own.
- They offer feature flags to support an old version → wrong, because flags can't rebuild a prior release.

**🎯 Question says → you pick:**

- "patch shipped version while next-release work continues" → release branching
- "drift job reports zero drift everywhere" → shallow checkout, set `fetch-depth: 0`

**💡 Hook:** How often does the bus leave, and are old passengers still aboard?

### 🚪 08 · Pull requests: only the bouncer counts

Picture a nightclub door. Unreviewed code (a SQL injection included) strolled straight into Contoso's main, so now every change queues at the PR door: rulesets or branch protection on GitHub, branch policies on Azure Repos. The rule: a control only counts if the platform actually blocks the merge.

**🧠 Lock these in:**

- Required check = the job's `name:` (e.g. `ci/build`), not the workflow name; mismatch = pending forever
- CODEOWNERS: last match wins and replaces earlier owners; team needs write access + `require_code_owner_review`
- `dismiss_stale_reviews_on_push`: a new push cancels ALL approvals
- `require_last_push_approval`: the last pusher can't be the approver
- Azure: `--blocking true`, `--creator-vote-counts false`, `--valid-duration` in minutes (720 = 12 hours)
- Merge queue needs a `merge_group` trigger (`types: [checks_requested]`) as well as `pull_request`

**🪤 Trap:**

- They offer `*` at the bottom of CODEOWNERS → wrong, because last match wins, so `*` owns everything.
- They offer "merge queue replaces PR checks" → wrong, because you need both triggers.

**🎯 Question says → you pick:**

- "required check stuck pending forever" → job `name:` doesn't match
- "two green PRs merge and break main" → merge queue + `merge_group` workflow
- "require a linked work item" → Azure `work-item-linking`; GitHub needs a workflow + required check

**💡 Hook:** A bouncer without `--blocking true` is just a greeter.

### 🏢 09 · Repo management: narrowest badge, proper certificates

Contoso gave 200 people identical access, an intern edited `/deploy/prod/`, and prod fell over. Think office building: Read is a visitor badge, Triage sorts the mail, Write gets a desk, Maintain redecorates the lobby, Admin holds the deed. The rule: give the narrowest role that fits, protect paths with CODEOWNERS (GitHub) or a path deny (Azure Repos), and enforce annotated `v*` tags so CI can read the version.

**🧠 Lock these in:**

- API values: `pull` (Read), `triage`, `push` (Write), `maintain`, `admin`
- Only Admin changes visibility, deletes the repo or manages access; no role bypasses branch protection by default
- Azure path deny: full ref + `//` before the path, `--deny-bit 4`; explicit Deny beats Allow
- `git tag -a` = annotated (tagger, date, message); bare `git tag` = lightweight pointer
- `git describe` returns the NEAREST tag; CI needs `fetch-depth: 0` + push rights (`contents: write` / `persistCredentials: true`)
- Tag ruleset: target `tag`, exclude `refs/tags/v*`, rule `creation`

**🪤 Trap:**

- They offer `permission=write` → wrong, because the API value is `push`.
- They offer deleting and re-creating a wrong release tag → wrong, because clones keep the old target; ship a new tag.

**🎯 Question says → you pick:**

- "manage topics and wiki, not visibility or deletion" → Maintain
- "intern triages issues, must not push code" → Triage
- "CI versions from v0.0.0, run is green" → shallow clone, set `fetch-depth: 0`

**💡 Hook:** Annotated tag = certificate, lightweight tag = sticky note, and CI only reads certificates.

### 🧥 10 · Git LFS: the coat check

Contoso's game repo hit 50 GB of `.psd` and `.fbx` files, so clones take over 4 hours. Git LFS is a coat check: the heavy coat goes on the rack while Git keeps a tiny 3-line ticket (the pointer). The rule: large binary FILES → Git LFS (with locking); deep HISTORY or too many files → Scalar, partial clone or sparse-checkout. They stack; neither replaces the other.

**🧠 Lock these in:**

- `.gitattributes` MUST be committed; `git lfs install` runs once per user
- `*.psd filter=lfs diff=lfs merge=lfs -text` (`-text` stops CRLF corrupting binaries)
- `git lfs migrate info` is read-only; `migrate import --everything` rewrites history → force push + team re-clones
- Shrink it: `git reflog expire --expire-unreachable=now --all`, then `git gc --prune=now`
- `GIT_LFS_SKIP_SMUDGE=1` or checkout `lfs: false` leaves pointer files (that checkout only)
- `--lockable` → read-only until `git lfs lock`; `git lfs unlock --force` needs maintain/admin
- git-fat (S3/rsync backend) has NO file locking

**🪤 Trap:**

- They offer LFS for 500,000 commits → wrong, because LFS doesn't shrink history; use Scalar or partial clone.
- They offer a pre-push hook as enforcement → wrong, because `--no-verify` skips it; use a server-side/CI check.

**🎯 Question says → you pick:**

- "files contain `version https://git-lfs.github.com/spec/v1`" → `git lfs pull`
- "repo still 50 GB after migration" → reflog expire + `gc --prune=now` never ran
- "binaries must live in S3 you control" → git-fat

**💡 Hook:** Heavy coats → coat check (LFS); too many guests → Scalar.

### 🔑 11 · Recover & scrub: change the locks first

Two opposite jobs. A deleted branch is a torn-off name tag with the box still on the shelf, so find it (`git reflog` or `git fsck --no-reflogs`) and relabel it (`git branch <name> <sha>`). A leaked key is a lost house key, and sweeping up the copies never changes the lock. The rule: rotate the credential FIRST, then rewrite history, force push, and everyone re-clones.

**🧠 Lock these in:**

- `git filter-repo --invert-paths --path <file>`; without `--invert-paths` it keeps ONLY that file
- `--replace-text`: `regex:` for patterns, `==>` separates match from replacement
- BFG runs on a `git clone --mirror`; filter-branch is deprecated
- Then `git reflog expire --expire=now --all` + `git gc --prune=now --aggressive`, push `--force --all` and `--force --tags`
- Verify: `git log --all -S "<secret>"` finds nothing, plus an object scan
- Rotate: create the new key before deleting the old; audit CloudTrail
- Prevent: push protection (server-side, unskippable) + pre-commit hook

**🪤 Trap:**

- They offer `git revert` or `git rm` → wrong, because old commits still hold the secret.
- They offer `git pull` after the rewrite → wrong, because it duplicates history and brings the secret back.

**🎯 Question says → you pick:**

- "FIRST action for leaked credentials" → rotate + audit
- "deleted branch, nobody has a clone" → GitHub support (~90 days); Azure DevOps restores from the UI
- "fold in a commit, discard its message" → `fixup` (`squash` keeps both)

**💡 Hook:** Change the locks, then sweep the copies.

### 🏬 12 · Mono vs multi: fix the door, keep the warehouse

Contoso has 15 microservices in an 8 GB repo with a 25-minute clone. A mono-repo is one shared warehouse (atomic cross-service changes); multi-repo is 15 corner shops (clear ownership, independent releases). The rule: choose by whether services change together, name your model's cost, and fix size with engineering. Nobody demolishes the warehouse because the front door is slow.

**🧠 Lock these in:**

- Fastest clone, least disk: `git clone --filter=blob:none --sparse` + `git sparse-checkout set <paths>`
- Partial clone cuts DOWNLOAD (keeps history); sparse-checkout cuts WORKING TREE; `--depth=1` cuts history
- `sparse-checkout set` replaces, `add` extends, `disable` restores everything
- Scalar = standard Git config: FSMonitor, commit-graph, multi-pack index, background maintenance
- Submodules: `update --init --recursive` → pinned commit; `update --remote` → latest
- Azure: `resources.repositories` + `checkout: <alias>` + explicit `checkout: self`; Actions private repo needs `token:`
- `trigger.paths` gates the whole pipeline; `dorny/paths-filter` gates individual jobs

**🪤 Trap:**

- They offer Scalar for a 500 MB binary → wrong, because that's Git LFS.
- They offer multi-repo to fix clone time → wrong, because you pay in coordinated PRs and version drift forever.

**🎯 Question says → you pick:**

- "minimal download AND full history" → `--filter=blob:none`
- "pipeline can't find its own source" → add `checkout: self`
- "mono-repo cost with no mitigation" → limited per-service access control

**💡 Hook:** DOWNLOAD = filter, DISK = sparse, HISTORY = depth, FILES = LFS.

---


## 🏁 Part 3 — Domain 3a + 3b · Package management & testing

Welcome to the heavyweight round: this part sits inside Domain 3, worth a whopping 50–55% of the exam. One theme runs through all six challenges: **a control only counts if it can say no**. Dashboards and PR comments are commentary; non-zero exits, required checks and protection rules actually stop you.

### 📦 13 · One pantry for every package

Contoso's 15 microservices copy-paste shared code like neighbours photocopying one recipe. The fix is a company pantry: the **feed** holds packages, **views** are shelves (prerelease in the back room, release out front), and the **upstream** is the supermarket, bought from once and then cached. Need npm AND NuGet in one feed, with upstreams and views? Pick Azure Artifacts, because GitHub Packages has neither.

**🧠 Lock these in:**

- GitHub Packages supports npm, Maven, NuGet, Docker, RubyGems. No pip.
- Feed roles: Reader (consume), Collaborator (+ save from upstream), Contributor (publish), Owner (full control).
- Upstream cache survives a public outage. List the internal upstream FIRST.
- Promotion ADDS an existing version to a view. Nothing is rebuilt.
- GitHub scope must match the repo owner, or 403. Workflow needs `packages: write`.
- Azure `.npmrc`: bare `registry=` plus `always-auth=true`. `setup-node` with `registry-url` expects `NODE_AUTH_TOKEN` (not `NPM_TOKEN`).

**🪤 Trap:**

- They offer Reader for consuming teams → wrong, because Reader can't save from upstream, so new public packages fail.
- They offer the public upstream first → wrong, because that invites dependency confusion.

**🎯 Question says → you pick:**

- "npm install 401, publish works (Azure feed)" → `always-auth=true` is missing.
- "only tested versions reach consumers" → views plus promotion.
- "404 after a team moved project" → project-scoped feed; go org-scoped.

**💡 Hook:** Reader takes from the shelf; Collaborator can also run to the supermarket.

### 🔢 14 · Version numbers that tell the truth

SemVer is an allergy label: consumers read it to see if upgrading is safe (MAJOR = contains nuts!). CalVer is a birth certificate: it says when, not what's inside. So libraries get SemVer and scheduled apps get CalVer. Versions are calculated by a machine, never typed by hand, and never reprinted once published.

**🧠 Lock these in:**

- MAJOR = breaking, MINOR = compatible feature, PATCH = compatible fix. Higher bumps reset lower: 2.1.0 + feature + fix = 2.2.0.
- Precedence: alpha, beta, rc, then the release (highest). `+metadata` is ignored.
- `$[counter(prefix, 0)]` is atomic, so parallel runs never collide. `Build.BuildId` is unique but not sequential.
- `dotnet pack /p:Version=1.2.0-beta.1`, or `DotNetCoreCLI@2` with `versioningScheme: byEnvVar`.
- GitVersion `mode: ContinuousDeployment` = one version per commit. Needs `fetch-depth: 0`; feed `nuGetVersion` to pack.
- `##vso[build.updatebuildnumber]` renames the run; `##vso[task.setvariable]` sets a variable.
- Images: `$(Build.BuildId)-<shortSHA>` for traceability, plus an immutable SemVer tag for rollback.

**🪤 Trap:**

- They offer CalVer for a shared library → wrong, because Dependabot can't classify updates.
- They offer republishing the fix as 1.3.0 → wrong, because versions are immutable. Ship 1.3.1, deprecate 1.3.0.

**🎯 Question says → you pick:**

- "every build is 0.1.0, pipeline green" → shallow clone; set `fetch-depth: 0`.
- "eliminates the race entirely" → GitVersion ContinuousDeployment.

**💡 Hook:** SemVer is a safety label, CalVer is a birth date, and nobody reprints either.

### 🛡️ 15 · Smoke alarms vs locked doors

A quarterly audit found 3 of 15 services on a critical CVE, and fixing it took weeks. Dependabot alerts are the smoke alarm, dependency review is the locked door, NuGet source mapping is a bouncer with a guest list, and licence checks are the lawyer asking "may we ship it?". The rule: if something must be refused before it gets in, pick the preventive control, not an alert.

**🧠 Lock these in:**

- Security updates fire immediately, no `dependabot.yml` needed. Version updates follow that file's schedule.
- `open-pull-requests-limit` queues extras: limit 5 + 8 outdated = 5 open, 3 queued.
- `ignore` with `update-types: ["version-update:semver-major"]` blocks majors only; `groups` makes majors stand out.
- `dependency-review-action@v4`: `fail-on-severity: high` (high + critical), `warn-only: false`, AND a required check.
- `deny-licenses` blocks GPL; `license-checker --failOn` blocks copyleft, and `--unknown` plus `exit 1` fails undeclared licences.
- NuGet `packageSourceMapping` fails closed: no pattern match, restore fails.
- `MicrosoftSecurityDevOps@1` detects but opens no upgrade PRs.

**🪤 Trap:**

- They offer `warn-only: true` for urgent work → wrong, because the gate becomes a report. Use `allow-ghsas` for that one advisory.
- They offer CodeQL for vulnerable packages → wrong, because CodeQL checks code you wrote.

**🎯 Question says → you pick:**

- "which services are affected, one call" → org `dependabot/alerts` API via `gh api`.
- "event-stream must never appear" → grep the lockfile, `exit 1`.

**💡 Hook:** An alarm detects, a lock prevents, and a lock that only beeps isn't a lock.

### 🍳 16 · The test pyramid kitchen

Unit tests taste each ingredient, integration tests cook on a real stove (a real PostgreSQL, not a mock), and load tests are the Friday-night rush. The pyramid is about cost, so chain layers with `needs:` and never pay for the pricey layer after a cheap failure. Only Jest `coverageThreshold` and k6 `thresholds` can send a plate back, because they exit non-zero; everyone else just writes reviews.

**🧠 Lock these in:**

- `fail-fast: false` lets every matrix leg finish; `max-parallel` controls concurrency.
- Service container: `--health-cmd="pg_isready"` plus an explicit `pg_isready` loop, then migrate → seed → test.
- `PublishTestResults@2` sends JUnit to the Tests tab (`mergeTestResults: true`).
- Reporting steps: `if: always()` / `condition: always()`. `succeededOrFailed()` misses cancellation.
- k6 `stages` = load profile, `thresholds` = pass/fail. No thresholds, k6 always exits 0.
- Server wait: `npm start &`, poll `/health`, end with `curl -f ... || exit 1`.

**🪤 Trap:**

- They offer `continue-on-error` → wrong, because it paints a failing leg green.
- They offer a longer `sleep` or auto re-runs → wrong, because timing failures keep coming back, now hidden.

**🎯 Question says → you pick:**

- "ECONNREFUSED 5432 in CI only" → `pg_isready` retry loop before migrations.
- "load test passed for weeks despite a slowdown" → `thresholds` block missing.

**💡 Hook:** Check the thermometer, don't count to five, and walk out if the oven never heats.

### 🚦 17 · Gates that actually say no

Picture an airport. An approval is the passport officer who checks you once, a wait timer is a queue that asks nothing, and an Azure Pipelines gate is the agent re-checking the weather until it's safe to board. SARIF uploads and artifacts are CCTV: they record everything and stop nobody. The rule: a gate is a non-zero exit or an environment protection rule, and only Azure Pipelines gates poll.

**🧠 Lock these in:**

- Reviewers, `wait_timer`, `deployment_branch_policy` live ON the environment; YAML only names it.
- Deploy runs only when `needs:` succeeded AND `if:` is true AND environment rules pass.
- Trivy blocks only with `exit-code: '1'`. SARIF upload needs `security-events: write` and `if: always()`.
- Composite action: every run step needs `shell:`. Outputs block nothing; `exit 1` does.
- Azure gates retry every `period` and fail only at `timeout`.
- Branch protection approves the CODE; environment reviewers approve the DEPLOYMENT.

**🪤 Trap:**

- They offer a wait timer for "pause while alerting, resume automatically" → wrong, because it doesn't poll. Use the Azure Monitor alerts gate.
- They offer the push-only deploy job as a required check → wrong, because it never runs on PRs, so it sits at "Expected" forever.

**🎯 Question says → you pick:**

- "Defender fails but findings still deploy" → `defender-scan` missing from `needs:`.
- "no reviewer notified, job pending" → renamed or deleted team, silently skipped.

**💡 Hook:** The passport officer checks once, the gate agent keeps checking, CCTV stops nobody.

### 📏 18 · The bathroom-scale metric

Coverage is a bathroom scale: it records that you stepped on, not that you exercised. It shows which lines ran, not whether a test checked them. Contoso's school rules: 80% overall is the pass mark, the ratchet says "your grade can't go down", and 90% diff coverage says "today's homework must be done", the only rule that makes a PR author write a test. Missing data must fail, and only a required status check blocks a merge.

**🧠 Lock these in:**

- `PublishCodeCoverageResults@2` takes Cobertura or JaCoCo only; its tab is visibility, not a gate.
- Azure gate: `PowerShell@2` reads `line-rate` (a 0–1 fraction, ×100), then `exit 1`.
- Jest: `collectCoverageFrom` counts unimported files; `coverageThreshold` fails the run.
- `--collect:"XPlat Code Coverage"` silently writes nothing without `coverlet.collector`.
- `diff-cover` without `--fail-under=90` only reports.
- Diff jobs need `fetch-depth: 0` AND `git fetch origin main:refs/remotes/origin/main`.

**🪤 Trap:**

- They offer 60% for a legacy service → wrong, because that's a waiver. Use a ratchet plus diff coverage.
- They offer an `if [ -f ]` guard → wrong, because missing data then passes.

**🎯 Question says → you pick:**

- "42 tests passed, coverage 0%" → plumbing: reporter, path, `coverlet.collector` or source maps.
- "tests assert nothing, coverage 96%" → no coverage rule catches it; mutation testing and review do.

**💡 Hook:** The scale says you stood on it, not that you ran. No ID, no entry.

---


## 🏁 Part 4 — Domain 3c · Pipeline fundamentals

Welcome to the engine room 🚂. Domain 3 is the heavyweight at 50–55% of the exam, and this part covers how pipelines run: where, in what order, and who gets through the door. One big theme: the platform already has a built-in keyword for almost every problem, so pick it over any clever workaround.

### 🧩 19 · GitHub Actions: badge, not master key

Picture a restaurant kitchen: each station (job) cooks on its own stove, and a plate only moves on when the ticket (`needs`) says so. Data travels as job `outputs`, and `permissions` decides which doors the built-in `GITHUB_TOKEN` badge opens. The one rule: if a built-in feature (token permission, environment, composite action) solves it, don't add a PAT, a stored secret or a merged job. Read the constraint in the question's last line first.

**🧠 Lock these in:**

- Job-to-job data: step `id` + `$GITHUB_OUTPUT` + job `outputs:` + `needs:`. Miss one → empty string, no error.
- Only `needs` sets order; `if: success()` waits for nothing. `$GITHUB_ENV` stops at the job boundary.
- GHCR push: `packages: write` AND `docker/login-action` with `secrets.GITHUB_TOKEN`.
- Azure with no stored secret: `id-token: write` + `azure/login@v2` (client-id, tenant-id, subscription-id) = OIDC.
- Dropdown manual run: `workflow_dispatch` + `inputs` + `type: choice` + `options`.
- Composite action: `using: "composite"`, every `run` step needs `shell:`, called under `steps:`.
- `vars.NAME` = Settings variables; `env.NAME` = YAML-only values.

**🪤 Trap:**

- They offer a PAT for a GHCR push → wrong, because `packages: write` fixes the built-in token.
- They offer `github.base_ref == 'main'` for push-to-main → wrong, because `base_ref` is only filled on pull requests.

**🎯 Question says → you pick:**

- "downstream job reads needs.build.outputs.x but gets empty" → job-level `outputs:` on build.
- "no long-lived Azure credential" → OIDC with `id-token: write`.
- "every production deploy must be approved" → required reviewer on the `production` environment.

**💡 Hook:** Steps whisper, jobs publish, `needs` listens.

### 🏗️ 20 · Azure Pipelines YAML: does the value exist yet?

Building a house 🏠: `${{ }}` is the architect's blueprint (compile time, can add or remove rooms), `$[ ]` is the doorman checking the list as each stage or job opens, and `$( )` is a sticky note slapped on a task just before it runs. Most questions boil down to "does this value exist yet?" The rule: a value for a later stage = output variable, files = artifact, approvals = environment on a `deployment` job.

**🧠 Lock these in:**

- Stage-to-stage: `isOutput=true` + step `name:` + `dependsOn` + `$[ stageDependencies.Build.BuildJob.outputs['setVersion.buildVersion'] ]`.
- `AzureKeyVault@2` secrets are plain `$(SecretName)`, masked in logs.
- `deployment:` job needs `environment:` + `strategy` (runOnce, rolling, canary), auto-downloads artifacts to `$(Pipeline.Workspace)`, does NOT check out source.
- `and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))`: a custom condition REPLACES the implicit `succeeded()`.
- `Build.SourceBranch` = `refs/heads/main`; `Build.SourceBranchName` = `main`.
- Restrict values: `type: string` + `values:`. Yes/no is `boolean`, not bool.

**🪤 Trap:**

- They offer `${{ stageDependencies... }}` → wrong, because it's resolved before any stage runs, so it's silently empty.
- They offer `continueOnError: true` on tests → wrong, because it hides failures; put `condition: always()` on `PublishTestResults@2`.
- They offer `pr:` paths exclude for "docs must never trigger a build" → wrong, because that filters PR validation; use `trigger:` paths exclude.

**🎯 Question says → you pick:**

- "files / a string needed by a later stage" → pipeline artifact / output variable.
- "template in another GitHub repo" → `resources: repositories:` (type github + endpoint), then `file.yml@alias`.
- "repo file missing inside a deployment job" → add `- checkout: self`.

**💡 Hook:** NIDS: Name the step, IsOutput=true, DependsOn, `$[ stageDependencies ]`.

### 🚕 21 · Runners & agents: ride-share or company car?

A hosted runner is a ride-share 🚗: clean car every trip, zero upkeep, public roads only, and the macOS model costs 10x Linux. A self-hosted runner is the company car: it reaches the private staff car park (on-prem network) and keeps gear in the boot (warm Docker cache), but you pay the mechanic. Start hosted; go self-hosted only for private network access, heavy macOS volume, persistent caches or compliance.

**🧠 Lock these in:**

- Self-hosted is outbound only (HTTPS 443): no inbound port.
- `--ephemeral` = ONE job, then the runner de-registers (VMSS: tear down after every use).
- `runs-on: [self-hosted, linux, on-prem]` = runner needs ALL labels.
- Runner group = WHO may use runners; label = WHICH runner.
- Azure DevOps demand: `- Agent.OS -equals Linux`. `pool: vmImage:` = Microsoft-hosted.
- Elastic: VMSS pool (Azure DevOps, scales to 0) vs ARC on Kubernetes (GitHub).
- Offline + `Http response code: 403` → token expired; re-register with `--replace`.

**🪤 Trap:**

- They offer allow-listing GitHub-hosted IPs → wrong, because the ranges are shared and change often.
- They offer extra labels to restrict repos → wrong, because any repo can target labels.
- They offer ephemeral runners for a warm Docker cache → wrong, because ephemeral destroys the cache.

**🎯 Question says → you pick:**

- "on-premises SQL / behind corporate firewall" → self-hosted runners inside the network.
- "Xcode builds and macOS cost must fall" → self-hosted macOS hardware.
- "capacity must cost nothing when idle" → VMSS pool with Minimum agents 0.

**💡 Hook:** Label = where the car goes; runner group = who has the keys.

### 🏃 22 · Triggers & order: same starting gun, side by side

Think relay race 🏁. Runners waiting for the same starting gun (same upstream) run side by side; pass the baton (chained `dependsOn`) and they run one after another: no error, just slower. Triggers decide whether a run starts; `dependsOn`/`needs`, matrices and conditions decide order. Another race can't pass you a baton, so use `resources: pipelines:` (Azure) or `workflow_run` (GitHub).

**🧠 Lock these in:**

- GitHub: `paths` + `paths-ignore` together = never fires; use `!` negation. Azure: include + exclude is fine.
- Azure schedules: `always: false` = only if code changed. `trigger: none` does NOT stop schedules.
- `workflow_run` `types: [completed]` fires on failure too: check `conclusion == 'success'`, check out `head_sha`.
- No `dependsOn` = waits for previous stage; `dependsOn: []` = start now.
- Only `always()` runs after a SKIPPED dependency, not `succeededOrFailed()`.
- `fail-fast` defaults to true; set `false` so every leg reports.
- Job outputs are strings (`== 'true'`); `type: boolean` inputs are real booleans.

**🪤 Trap:**

- They offer `Api dependsOn: WebApp` for "run in parallel" → wrong, because it serialises; give both `dependsOn: Build`.
- They offer `succeededOrFailed()` on Notify → wrong, because it's skipped when a dependency was skipped.
- They offer `continue-on-error: true` → wrong, because the failing leg shows green.

**🎯 Question says → you pick:**

- "workflow starts, detects changes, then skips; reduce cost" → trigger path filter.
- "shared/ change builds frontend but not backend" → add `shared/**` to backend `paths`.
- "notify whether deploy succeeded or failed" → `always()` + `contains(needs.*.result, 'failure')`.

**💡 Hook:** `always()` is the announcer who speaks even if a runner never showed.

### 📦 23 · Reusable bits: recipe card or caterer?

Fifteen services copy-pasted one pipeline, so one policy change meant fifteen edits 😩. A composite action or Azure steps template is a recipe card in your own kitchen: steps, same runner. A reusable workflow or Azure jobs/stages template is a caterer with their own kitchens: whole jobs, own runners. Share steps → recipe card, share a job graph → caterer, and always pin to a tag.

**🧠 Lock these in:**

- Reusable workflow: `on: workflow_call`, called with `uses:` at JOB level + `with:` + `secrets:`; no `runs-on` on the caller.
- Secrets never flow automatically: `secrets: inherit` or explicit mapping. Composite can't read `secrets`; pass as input.
- Outputs: reusable workflow uses `jobs.<job>.outputs.<name>`; composite uses `steps.<id>.outputs.<name>`.
- Azure template types: steps, jobs, stages, variables. The root key decides where it fits.
- Cross-repo: `path.yml@alias` with `ref: refs/tags/v2.1.0` (plain `ref: main` fails).
- Task groups: classic-only, step level only, no YAML.

**🪤 Trap:**

- They offer `@main` as a stable pin → wrong, because every push hits every consumer.
- They offer a starter workflow → wrong, because it's copied once, so fixes never propagate.
- They offer `type: choice` + `options` in an Azure template → wrong, because Azure uses `type: string` + `values:`.

**🎯 Question says → you pick:**

- "share steps in the same job / no extra runner" → composite action.
- "least privilege for secrets" → explicit `secrets:` mapping, not inherit.
- "one deploy stage for staging and production" → one parameterised stage template, referenced twice.

**💡 Hook:** Order from a dated menu (`@v2`), never "whatever the chef fancies today" (`@main`).

### 🚦 24 · Checks & approvals: bouncer at the front door

Nightclub time 🪩. Branch protection is the kitchen-door bouncer (merges); the environment is the front-door bouncer checking every guest, every night (deployments). One kitchen check doesn't get you through the front door fifty times. In GitHub and Azure, gates live on the ENVIRONMENT in settings, not YAML; the job just names it.

**🧠 Lock these in:**

- No `environment:` = no gate, no warning. Names are case-sensitive: `Production` silently creates a new, unprotected environment.
- GitHub rules: required reviewers (`User`/`Team` only), `wait_timer` (minutes, before the job starts), branch/tag policy, `prevent_self_review: true`.
- Custom GitHub rule = GitHub App POSTing `approved`/`rejected` to `deployment_callback_url`.
- Azure approval: `minRequiredApprovers`, `blockedApprovers`, `timeout` (43200 min = 30 days).
- Azure `lockBehavior`: `sequential` queues, `runLatest` cancels older queued runs.
- GitHub `concurrency`: same group name = shared lock. `cancel-in-progress: false` for production.
- Azure checks: Business hours, Azure Monitor alerts (Sev0/Sev1), Evaluate artifact (Rego), Required template.

**🪤 Trap:**

- They offer branch protection for deploy approval → wrong, because it gates merges only.
- They offer `dependsOn`/`needs` to stop overlapping deploys → wrong, because they order work inside one run.
- They offer a day-of-week stage condition → wrong, because it skips (silent green run); Business hours holds it.

**🎯 Question says → you pick:**

- "reviewer configured but deploy went through unapproved" → job missing `environment:`.
- "never overlap AND never interrupt a deploy" → `cancel-in-progress: false` / `lockBehavior: sequential`.
- "give the team N minutes to cancel" → `wait_timer`.

**💡 Hook:** Kitchen door = merges, front door = deploys, rope line = concurrency.

---


## 🏁 Part 5 — Domain 3d · Deployment strategies

Domain 3 is worth a whopping 50–55% of the exam, and this is where releases stop being scary. One theme runs through all six challenges: **always keep a way back**. Swap back, route back, toggle off or roll forward, but never ship without an exit.

### 🎭 25 · Blue-green vs canary

Blue-green is a theatre with two stages: rehearse backstage (staging slot), spin the stage round (swap), spin it back if the show flops. Canary is a food tasting: 10% of diners get a sample via Traffic Manager, but anyone holding an old menu (cached DNS) keeps ordering from it until the TTL expires. Zero downtime never picks the answer; the requirement does: instant rollback = blue-green, real user traffic = canary, separate audiences = rings, "without deploying" = feature flags.

**🧠 Lock these in:**

- `az webapp deployment slot swap --slot staging --target-slot production` deploys AND rolls back
- Slots need Standard (S1) or higher; Free and Basic have none
- Content swaps, slot settings stay: `--slot-settings` for connection strings
- Swap warm-up: `WEBSITE_SWAP_WARMUP_PING_PATH` + `WEBSITE_SWAP_WARMUP_PING_STATUSES=200`
- Canary: Traffic Manager `--routing-method Weighted` (90/10), split per DNS answer, not per request
- Rollback step `if: failure()`; approval = `environment: production` + required reviewer

**🪤 Trap:**

- They offer canary for "instant rollback" → wrong, because cached DNS holds until the TTL expires.
- They offer `WEBSITE_HEALTHCHECK_PATH` for swap warm-up → wrong, because it handles ongoing instance health, not the swap.

**🎯 Question says → you pick:**

- "10% canary gets about 50% of traffic" → DNS caching; fix with `--ttl 30`
- "production hits the staging database after swap" → connection string not a slot setting
- "roll back automatically if 5xx spikes" → metric alert on Http5xx + rollback action group

**💡 Hook:** Blue-green = spin the stage back; canary = a tasting slowed down by old menus.

### 🏃 26 · Rolling updates and slot swaps

Think relay pit stop: pull one runner off the track, change their shoes, send them back, watch their first lap. Contoso updated all 4 instances at once and ate a 3-minute outage. The rule: never update an instance while it serves traffic, and never let it serve traffic before it is warm.

**🧠 Lock these in:**

- `maxBatchInstancePercent=25` with 4 instances = 1 at a time, 75% capacity kept
- `maxUnhealthyUpgradedInstancePercent` halts a bad release; fix the app, then `az vmss rolling-upgrade start`
- Hooks: preDeploy (drain) → deploy → routeTraffic (back in LB) → postRouteTraffic (verify)
- `AzureAppServiceManage@0` with `action: 'Swap Slots'` swaps; `AzureWebApp@1` only deploys
- Auto-swap needs Standard+, skips smoke tests and approvals, so dev/test only
- `az webapp traffic-routing set --distribution staging=10` = per-request split, instant

**🪤 Trap:**

- They offer `maxUnhealthyInstancePercent` for batch size → wrong, because it is only an abort threshold.
- They offer `--sticky-settings` → wrong, because it does not exist; use `--slot-settings`.

**🎯 Question says → you pick:**

- "auto-swap never fires" → plan is Free or Basic; upgrade to Standard (S1)+
- "production telemetry spikes when staging is tested" → App Insights key not a slot setting
- "VMSS extension reporting app health" → ApplicationHealthLinux

**💡 Hook:** Only a quarter of the team is ever in the pit.

### 🎚️ 27 · Feature flags

A feature flag is a light switch wired in before the tenants move in: code deploys with the switch off, then you flip it on for staff first, beta testers next. Sparks fly? Flip it off, nobody rewires anything. The rule: a named group of people or an A/B test = Microsoft.Targeting; a kill switch with no deployment = disable the flag in App Configuration.

**🧠 Lock these in:**

- App Configuration **Standard** tier + App Configuration Data Reader role (Reader/Contributor are control plane)
- Targeting = sticky per user; Percentage = random every check; TimeWindow = Start/End in UTC
- Precedence: named users → groups → DefaultRolloutPercentage
- Refresh: `app.UseAzureAppConfiguration()` BEFORE `MapControllers()`; set `CacheExpirationInterval` to 30 seconds
- `[FeatureGate("Name")]` = whole endpoint (404 when off); `IsEnabledAsync` = pick a code path
- Labels = environments; kill switch `az appconfig feature disable`

**🪤 Trap:**

- They offer Microsoft.Percentage for an A/B test → wrong, because users flip between variants.
- They offer traffic routing for "internal team first" → wrong, because it picks random requests, not users.

**🎯 Question says → you pick:**

- "works in staging, not production; portal shows enabled" → label mismatch
- "internal team at 100% but can't see it" → `AddHttpContextAccessor()` missing, everyone is anonymous
- "deleted old flag, app misbehaves" → remove the code first, deploy, then delete

**💡 Hook:** Targeting is a fixed seat number; Percentage is musical chairs.

### 📦 28 · Containers: ACR, Container Apps, AKS

Picture a restaurant kitchen: plates on the pass are not a happy customer. Applying is not verifying (KubernetesManifest@1 applies, `rollout status` verifies), and reporting is not gating (Trivy severity reports, `exit-code: '1'` blocks). "No stored credential" means an identity, never the ACR admin account.

**🧠 Lock these in:**

- ACR roles: AcrPull → AcrPush → AcrDelete → AcrImageSigner; push auth error = needs AcrPush
- Premium only: geo-replication, content trust, private endpoints; admin account off with `--admin-enabled false`
- Build on PR, push on merge: `push: ${{ github.event_name != 'pull_request' }}`
- Multi-arch = setup-qemu-action + setup-buildx-action
- Container Apps: `--registry-identity system` + AcrPull; rollback `--revision-weight "<previous>=100"`
- AKS: `maxSurge: 1`, `maxUnavailable: 0` PLUS a readiness probe

**🪤 Trap:**

- They offer `severity: 'CRITICAL,HIGH'` alone → wrong, because nothing blocks without `exit-code: '1'`.
- They offer `az containerapp update --image <previous-tag>` → wrong, because it makes a NEW cold revision; revision weight is instant.

**🎯 Question says → you pick:**

- "pipeline succeeds but pods crash-loop" → Kubernetes@1 `rollout status --timeout=300s`
- "app on 8080, ingress on 80, restarts" → `--target-port 8080`
- "re-scan registry images for new CVEs" → Defender for Containers

**💡 Hook:** Liveness = chef fainted (restart him); readiness = chef still prepping (no orders yet).

### 🗄️ 29 · Database deployments

Code rolls back, data does not: a swap restores old code, but a dropped column is gone. Think moving house: build the new room (expand), move furniture while both work (migrate), knock down the old room a later release (contract). The rule: idempotent script generated in build, applied in a gated stage the app depends on, fixed by rolling forward.

**🧠 Lock these in:**

- `dotnet ef migrations script --idempotent`: no database connection, safe to re-run
- Same script artifact to staging and production
- Apply: `azure/sql-action@v2.3` (GitHub) or `SqlAzureDacpacDeployment@1` (Azure Pipelines)
- Keep `/p:BlockOnPossibleDataLoss=true` and `/p:DropObjectsNotInSource=false`
- Identity: `CREATE USER [sp] FROM EXTERNAL PROVIDER` + db_ddladmin
- Expand-contract = 3 releases; new column nullable in expand

```yaml
- stage: DeployApplication
  dependsOn: DeployDatabase
  condition: succeeded('DeployDatabase')   # GitHub: needs
```

**🪤 Trap:**

- They offer `Database.Migrate()` at startup → wrong, because instances race on DDL with no review gate.
- They offer Owner/Contributor for DDL → wrong, because Azure RBAC is control plane only.

**🎯 Question says → you pick:**

- "Invalid object name after deployment" → stages ran in parallel; add `dependsOn` / `needs`
- "bad migration, normal bug" → forward-fix with a compensating migration
- "Rows were detected… data loss might occur" → expand-contract, not `BlockOnPossibleDataLoss=false`

**💡 Hook:** Build the new room before knocking down the old one.

### 🚑 30 · Hotfix paths and resiliency

A hotfix is an ambulance: it runs red lights (integration tests, rings, second reviewer) but keeps the seatbelt (security scan) and reverse gear (staging swap). Normal pipeline about 2 hours; hotfix goal under 15 minutes. The rule: shrink gates, never remove them, and cherry-pick the fix to main.

**🧠 Lock these in:**

- Branch from the broken tag `release/2.4.0`; tag the fix `release/2.4.1`
- Security scan never skipped: SAST only with CodeQL
- Tests: `--filter "Category=Critical|Category=Payment"`
- `production-hotfix` environment: single on-call approver (normal = 2 reviewers)
- Swap = ONE free rollback; a third swap brings the bug back
- `az webapp traffic-routing clear` BEFORE the swap; breaker step `if: failure()`
- Cherry-pick: `fetch-depth: 0` + PAT (GITHUB_TOKEN pushes don't trigger workflows)

**🪤 Trap:**

- They offer branching from `release/2.3.1` → wrong, because that is a rollback, not a hotfix.
- They offer `condition: always()` on the frontend stage → wrong, because it deploys even if the backend failed.

**🎯 Question says → you pick:**

- "swap done but some customers still see the bug" → leftover routing; `traffic-routing clear`
- "frontend deployed before API" → `dependsOn: DeployBackend`
- "same bug returns next release" → never cherry-picked to main

**💡 Hook:** Kill the old indicator before changing lanes: clear routing, then swap.

---


## 🏁 Part 6 — Domain 3e + 3f · Infrastructure as Code & pipeline operations

Still Domain 3, the heavyweight at 50–55% of the exam, so stay awake at the back. One theme runs through all eight: stop clicking, start committing. Infra, VM settings and environments live in git, and your pipelines stay honest, lean, tidy and fully YAML.

### 🏗️ 31 · Plan on PR, apply on merge

Clicking changes into the portal is building with no drawing: environments drift, nobody knows who did what, and prod falls over. IaC keeps one template set in git and a pipeline deploys it. The pull request is the architect's drawing (lint, validate, what-if or plan), and no wall moves until merge to main. Pick the tool by one constraint: Azure-only and simple → Bicep, multi-cloud or built-in drift detection → Terraform, existing JSON → ARM.

**🧠 Lock these in:**
- Cheapest check first: `az bicep build` (no Azure call) → `validate` (would Azure accept it) → `what-if` (what changes)
- `--confirm-with-what-if` prompts, then deploys, so it is not read-only
- GitHub OIDC: `azure/login@v2` (client-id, tenant-id, subscription-id) + `id-token: write`
- Terraform: `use_oidc = true` in backend AND provider, one state file per environment
- Stale lock: check the blob lease, then `terraform force-unlock`
- `terraform plan -detailed-exitcode` returns 2 = drift

**🪤 Trap:**
- They offer `terraform apply` on the PR → wrong, because that writes from unreviewed code.
- They offer Activity Log or a resource lock for drift → wrong, because one logs operations and the other blocks changes.

**🎯 Question says → you pick:**
- "post what-if results to the PR" → `pull-requests: write`
- "validate Terraform with no Azure credentials" → `terraform init -backend=false` + `terraform validate`
- "template must create resource groups" → `targetScope = 'subscription'`

**💡 Hook:** PR = the drawing, merge = the builder, OIDC = an expiring visitor badge, not a copied key.

### 🧹 32 · Machine Configuration: skills + orders + key card

Quick manual fixes inside VMs make the security baseline drift. Azure Machine Configuration (formerly Guest Configuration) uses Azure Policy to audit and fix settings inside the guest OS: registry, services, software. Think hotel housekeeper: the package is her skills, the policy mode her orders, the managed identity plus RBAC role her key card. To fix drift, not just report it, you need all three.

**🧠 Lock these in:**
- VM prereqs: `Microsoft.GuestConfiguration` provider + system-assigned identity + `AzurePolicyforWindows` / `AzurePolicyforLinux` extension
- Package `-Type`: only `Audit` or `AuditAndSet`
- Policy `-Mode`: `ApplyAndMonitor` applies once, `ApplyAndAutoCorrect` re-applies every evaluation
- Assign with `-IdentityType SystemAssigned` + an RBAC role at the scope
- `az policy state trigger-scan` = evaluate now, `az policy remediation create` = fix existing VMs
- Agent is outbound only: no public IP or inbound rule needed

**🪤 Trap:**
- They offer `ApplyAndMonitor` for auto-correct → wrong, because it applies once, then only reports.
- They offer Automation State Configuration → wrong, because it is legacy and reports to the Automation account, not Policy.

**🎯 Question says → you pick:**
- "compliance stuck at Pending" → missing managed identity and/or extension
- "dashboard says compliant, VMs clearly aren't" → package built as `Audit`
- "remediation fails with an authorisation error" → identity has no RBAC role

**💡 Hook:** Skills (`AuditAndSet`) + orders (`ApplyAndAutoCorrect`) + key card (identity + RBAC) = a clean room.

### 🏨 33 · Deployment Environments: developer requests, platform deploys

Developers stuck on 3-5 day IT tickets build shadow IT. Azure Deployment Environments (ADE) is a hotel: guests get a card for the front desk only (Deployment Environments User on the project), while staff hold the master key (the project environment type's managed identity, Contributor on the subscription). Rooms come from an approved menu: a Dev Center catalog plus `environment.yaml`. Most wrong answers put a setting on the wrong object.

**🧠 Lock these in:**
- Developers get Deployment Environments User at PROJECT scope, never on the subscription
- Project environment type maps a name to `--deployment-target-id` + `--identity-type SystemAssigned`
- Catalogs attach to the Dev Center (`--git-hub` or `--ado-git`), PAT in Key Vault
- Dev Center identity reads the catalog secret, project env type identity deploys
- `environment.yaml` = developer contract with `allowed` lists, `environmentName` is auto-filled
- No built-in expiry: `--max-dev-boxes-per-user` caps how many, Policy tag + runbook caps how long

**🪤 Trap:**
- They offer `@allowed` in `main.bicep` to block big SKUs → wrong, because it fails at deploy time and builds no dropdown.
- They offer attaching the catalog to the project → wrong, because catalogs belong to the Dev Center.

**🎯 Question says → you pick:**
- "right role, still AuthorizationFailed" → Contributor for the project env type identity
- "definition not found in catalog" → check `syncState` and `get-sync-error-details`
- "PR pipeline fails on the second push" → existence check before create

**💡 Hook:** Guests get the front-desk card, only staff hold the master key.

### 🚨 34 · Pipeline health: green build, visible debt

A pipeline failing 30% of the time is a smoke alarm going off for no reason: soon nobody reacts to red. Pulling the battery (silent retries) makes it quiet, not safe. Track failure rate, average and P95 duration, MTTR and flaky tests. The rule: retry only if you record which tests were flaky and still fail on real failures.

**🧠 Lock these in:**
- Targets 90 / 15 / 25 / 30: success above 90%, avg under 15 min, P95 under 25, MTTR under 30
- Flaky = passes and fails on the same commit. Azure DevOps `flakyDetectionType: "system"` works with any framework
- `PublishTestResults@2`: keep `condition: always()`, set `failTaskOnFailedTests` + `failTaskOnMissingResultsFile` to true
- GitHub: `workflow_run` with `types: [completed]`, filter on `conclusion`, needs `actions: read`
- Alert on 3 or more consecutive failures on main, annotate flaky tests with `::warning::`
- Azure DevOps: Analytics tab (no code), Service Hooks for alerts

**🪤 Trap:**
- They offer raising Jest `retryTimes` → wrong, because JUnit just says "passed" and flakiness hides deeper.
- They offer alerting on every failure → wrong, because at 30% that is alert fatigue.

**🎯 Question says → you pick:**
- "100% pass rate, bugs reach prod" → remove `|| true`, set `failTaskOnFailedTests: true`
- "queue spike 9-11 AM" → more capacity (parallel jobs, self-hosted), not a faster build
- "same flaky tests every week" → owners + quarantine, never delete

**💡 Hook:** Don't pull the battery: log it, assign it, keep it loud.

### ⚡ 35 · Optimisation: cache, split, skip, shrink

A 45-minute, ~$216/month pipeline must drop under 15 minutes and $100/month with no coverage lost. Pack like a pro: cache the toiletries bag, split packing across four people, skip ski gear for a beach trip, shrink the suitcase. Only then consider a bigger car (self-hosted runners, parallel jobs). The trip still takes as long as its longest leg: the critical path.

**🧠 Lock these in:**
- Cache key on `package-lock.json` + `restore-keys`. setup-node `cache: "npm"` caches `~/.npm` only
- `cache-hit` is a string: compare to `'true'`
- Shards: `jest --shard=N/4` + `fail-fast: false`. Azure: `strategy: parallel: 4`
- Coverage: unique artifact per shard, `merge-multiple: true`, `nyc merge`
- `dorny/paths-filter@v3` gates jobs, trigger path filters cost nothing
- Turbo/nx affected builds need `fetch-depth: 2`. Upload only `dist/`

**🪤 Trap:**
- They offer buying more parallel jobs → wrong, because the critical path caps duration.
- They offer self-hosting first → wrong, because optimised usage (6,600 min/month) sits below the ~17,500 break-even.

**🎯 Question says → you pick:**
- "coverage shows 25% after sharding" → unique artifact name per shard + merge
- "deploy skipped when only the API changed" → `always()` + accept `'skipped'`
- "Turborepo rebuilds everything" → `fetch-depth` too shallow

**💡 Hook:** Cache, split, skip, shrink, then shop for a bigger car.

### 🧊 36 · Retention: delete by reference, not by age

Storage piles up like a fridge nobody clears: 500 GB, growing 50 GB a month. Tier it: PR leftovers keep for days, main-branch groceries about a month, and the wedding cake (the production release) gets a labelled freezer spot for a year. Bin what has no label (untagged manifests, unpromoted pre-releases), never just what's old, because that old jar might be tonight's dinner.

**🧠 Lock these in:**
- Azure DevOps: artifacts 30 days, PR runs 10, runs with release artifacts 365
- Retention lease (REST) protects one runId: `Bearer $(System.AccessToken)` header + JSON array body
- GitHub artifacts: 90 days default, 400 max, `retention-days` per upload. Releases never expire
- Azure Artifacts: clean `@prerelease`, protect `@release`, cap with `countLimit`
- ACR: `acr purge --untagged`, `--keep N`, recurring via `az acr task create --schedule`
- Cleanup pipeline: `always: true` + `trigger: none`

**🪤 Trap:**
- They offer `acr purge --ago 7d` on every tag → wrong, because it deletes images prod is running.
- They offer archiving to blob storage → wrong, because growth stays unbounded.

**🎯 Question says → you pick:**
- "lease created, artifacts gone after 30 days" → missing Bearer header or non-array body
- "keep longer than 400 days on GitHub" → GitHub Release
- "GHCR cleanup deletes almost nothing" → add `--paginate`

**💡 Hook:** Label the wedding cake, bin only the unlabelled.

### 🚚 37 · Classic to YAML: the locks don't move with you

Classic GUI pipelines move to YAML to live in git and reuse templates, without breaking live deploys. It's moving house: furniture (steps, stages) goes into labelled boxes (YAML keywords), and variable groups and service connections ride along unchanged. But locks belong to the house: approvals and gates become environment checks set in the UI, and a new environment has none. Artifact sources must be declared in `resources: pipelines:`.

**🧠 Lock these in:**
- Task groups → `template:` + `parameters:`. Agent phases → `jobs:` with different `pool:`
- Deployment groups → `environment:` with `resourceType: VirtualMachine` + `tags:`
- `- pipeline:` = alias, `source:` = real name. Fetch with `download: <alias>`
- Export to YAML works for builds only. Releases are rewritten by hand
- Phases: 1 run both (shadow env), 2 classic triggers off but kept, 3 archive JSON then delete
- `strategy: rolling` + `maxParallel` = a few VMs at a time

**🪤 Trap:**
- They offer a `gates:` or `approvals:` YAML key → wrong, because none exists.
- They offer `download: current` in the CD pipeline → wrong, because it only finds this run's artifacts.

**🎯 Question says → you pick:**
- "prod deploys unapproved after migration" → Approvals check on the environment
- "unauthorised service connection" → grant pipeline permissions
- "replace deployment groups" → VM environment, not an agent pool

**💡 Hook:** Pack the furniture, fit new locks yourself, keep the old key for two weeks.

### 🍽️ 38 · Capstone: taste the whole plate

The Domain 3 finale: one GitHub Actions pipeline with a private npm package, quality gates, an ACR image, Bicep checks, automatic staging and approved blue-green production. At the restaurant pass the chef tastes the whole plate, and the waiter checks it's the new recipe, not just that food arrived. So: gate on the merged result, and verify the version, not just a 200.

**🧠 Lock these in:**
- `build-image` needs `[coverage-gate, test-integration, security-scan]`, push to main only
- Private npm: setup-node `registry-url` + `scope` + `NODE_AUTH_TOKEN` from `GITHUB_TOKEN`
- Coverage: `merge-multiple: true`, `nyc merge`, fail under 80%
- Trivy blocks with `exit-code: "1"`, `scan-type: "fs"` before the build
- Swap flow: slot deploy → warm up → swap → `sleep 15`, retry, compare version → swap back
- Reviewers live on the production environment, none on staging. Metrics job `if: always()`

**🪤 Trap:**
- They offer `needs: test-unit` → wrong, because shards can pass while the merged total is 78.5%.
- They offer `wait_timer` as approval → wrong, because it delays without asking anyone.

**🎯 Question says → you pick:**
- "prod says success, users see the old version" → `sleep 15`, retry, compare with `image_version`
- "pipeline too slow, which job?" → integration tests (19-min critical path)
- "post a coverage comment on the PR" → `pull-requests: write`

**💡 Hook:** Taste the whole plate, check the recipe, and rollback = swap the plates back.

---


## 🏁 Part 7 — Domain 4 · Security & compliance

Domain 4 is worth 10–15% of the exam, and it's the classic spot where marks quietly leak out of your score. The one big theme: pick the control that removes the risk, not the one that just hides it. No stored secret beats a well-guarded secret, and a server-side block beats a polite reminder.

### 🪪 39 · Secretless sign-in to Azure
A service principal secret is a copied door key: it works for anyone, anywhere, until you change the locks. A managed identity (MI) is a staff badge that only works inside the building (Azure), and workload identity federation (WIF) is a visitor pass from a trusted front desk (GitHub or Azure DevOps) with one exact name on it. The rule: Azure compute → MI, GitHub Actions or Azure Pipelines → WIF, anywhere else → service principal + rotated secret. Getting in isn't opening rooms: every identity still needs an RBAC role.

**🧠 Lock these in:**
- GitHub subject `repo:org/repo:ref:refs/heads/main`; Azure DevOps subject `sc://org/project/connection-name`
- Audience is ALWAYS `api://AzureADTokenExchange`; subjects match exactly, no wildcards
- Workflow needs `id-token: write`; `azure/login@v2` takes client-id, tenant-id, subscription-id, no secret
- User-assigned MI is shareable and survives deletion; system-assigned dies with its resource
- MI role assignment: `--assignee-object-id` + `--assignee-principal-type ServicePrincipal`
- Data access: Storage Blob Data Contributor or Key Vault Secrets User, not Contributor

**🪤 Trap:**
- They offer MI on a GitHub-hosted runner or Microsoft-hosted agent → wrong, because neither is an Azure resource.
- They offer Contributor to read secrets → wrong, because control-plane roles can't read data.

**🎯 Question says → you pick:**
- "AADSTS70021 No matching federated identity record found" → subject mismatch, fix the federated credential
- "Login succeeded, then AuthorizationFailed" → missing RBAC role assignment
- "Only approved production deployments get a token" → subject `repo:org/repo:environment:production`

**💡 Hook:** No pass = 70021; no room permission = AuthorizationFailed.

### 🏨 40 · GitHub tokens: whose key is it?
GITHUB_TOKEN is a hotel key card: it opens only your room (this repo) and dies at checkout (job end). A GitHub App is the company's master key, logged under the org's name; a fine-grained PAT is a personal key that leaves with the employee; a classic PAT is a skeleton key, so ban it. The rule: ask how far it reaches and who owns it. Same repo → GITHUB_TOKEN, cross-repo automation → GitHub App, a human's local script → fine-grained PAT.

**🧠 Lock these in:**
- `permissions` sets WHAT, never WHERE; a job-level block REPLACES the workflow one (repeat `contents: read`)
- GITHUB_TOKEN can never write `.github/workflows/` or trigger other workflows
- `actions/create-github-app-token@v1`: app-id in `vars`, private-key in `secrets`, plus `owner:`
- Org policy: classic PATs "Do not allow"; only fine-grained PATs get approval and a max lifetime
- 401 bad token · 403 repo not in App installation · 404 outside fine-grained PAT scope
- Security manager = read all repos + manage security alerts, no write

**🪤 Trap:**
- They offer `contents: write` so GITHUB_TOKEN reaches another repo → wrong, because permissions change what, not where.
- They offer an App token alone to push to protected main → wrong, because the app must also be on the bypass list.

**🎯 Question says → you pick:**
- "Not tied to any individual / employee left" → GitHub App
- "Manage releases and deploy keys without full admin" → custom repository role (base write + extras)

**💡 Hook:** Hotel card, company master key, personal key, skeleton key.

### 🏦 41 · Azure DevOps: four dials on the vault
Azure DevOps security is a bank vault with four dials. The access level (licence) is your building badge, the security group is your job title, the service connection's RBAC scope is which vault the key opens, and pipeline permissions list who may pick up that key. The rule: spot which dial the requirement is about before reading the options, because most wrong answers are real controls on the wrong dial.

**🧠 Lock these in:**
- One pipeline only: disable "Grant access permission to all pipelines", then add it under Pipeline permissions
- "Resource authorization problem" = pipeline not authorised; AuthorizationFailed inside a task = no RBAC role
- Custom groups inherit nothing, so nest them in Contributors; explicit Deny beats Allow
- PAT policies (90-day max lifetime, restrict full-scoped and global) live only in `Organization Settings > Policies`
- Approvals, Branch control, Business hours = environment checks, not YAML
- Checks only gate a `deployment:` job declaring `environment:`; a plain `job:` skips them silently

**🪤 Trap:**
- They offer a manual intervention task or YAML condition as approval → wrong, because anyone editing the file removes it.
- They offer a "Stakeholders" group to cut licence cost → wrong, because groups change permissions, not the bill.

**🎯 Question says → you pick:**
- "Production deployments must come from main" → Branch control check on the environment
- "Dev pipeline must not reach production" → one service principal per environment, scoped to its resource group
- "Executives need dashboards at no licence cost" → Stakeholder access level

**💡 Hook:** Badge, title, vault, key list: "which dial?"

### 🔐 42 · The secrets ladder
Picture a ladder of house keys. Step 0 is a key under the doormat (plain-text variable), step 1 a lockbox the courier opens (pipeline fetches from Key Vault), step 2 a lockbox only the homeowner opens (the app resolves a `@Microsoft.KeyVault(...)` reference itself), and step 3 a door that knows your face (managed identity, OIDC). The exam wants the highest step allowed, so ask "does a credential still exist?" Only an irreplaceable third-party key stops you at step 2.

**🧠 Lock these in:**
- New vault: `--enable-rbac-authorization true`; access policies are then ignored
- Secrets User reads values · Reader sees metadata only · Secrets Officer manages secrets · Administrator does everything
- Key Vault-linked variable group: reusable, fetches latest at run time, auto-masked
- `AzureKeyVault@2`: `SecretsFilter` lists needed names (not `'*'`), `RunAsPreJob: true`
- Rotation: `--expires` + Event Grid on `Microsoft.KeyVault.SecretNearExpiry`
- `AuditEvent` diagnostics to Log Analytics = who read which secret, when

**🪤 Trap:**
- They offer the SQL password in Key Vault when managed identity works → wrong, because you're protecting what you could eliminate.
- They offer `az keyvault set-policy` for a 403 on an RBAC vault → wrong, because policies are ignored; assign Secrets User.

**🎯 Question says → you pick:**
- "Rotation must propagate with no pipeline changes" → Key Vault-linked variable group
- "403 after a Bicep rebuild" → system-assigned identity recreated; grant the new principal Secrets User
- "Auditor lists secret names and expiry dates" → Key Vault Reader

**💡 Hook:** Officer manages, User uses, Reader only sees it exists. No expiry, no rotation.

### 🚪 43 · Leak-proofing logs and repos
Secrets leak into pipeline logs and into git, and the exam grades you on where the control runs. A bouncer at the club door (GitHub push protection, server-side) beats a sign on your own door (.gitignore, pre-commit hook), and CCTV (CI scanning) only shows who already got in. In logs, the platform blurs only values it handed you; anything you fetched yourself needs your own blur. Already leaked? Blurring won't help: rotate.

**🧠 Lock these in:**
- File credentials (SSH key, kubeconfig) → Secure files + Pipeline permissions, fetched by `DownloadSecureFile@1`
- Clean up script copies with `condition: always()` (covers cancelled runs); `chmod 600` the key
- Azure Pipelines: `##vso[task.setvariable variable=X;isSecret=true;isOutput=true]`; outputs need a step `name:`
- GitHub Actions: `echo "::add-mask::$VALUE"` right after fetching; it's not retroactive
- `secret_scanning` alerts; `secret_scanning_push_protection` blocks the push
- `.gitignore` can't untrack: `git rm --cached .env`, then `git filter-repo` for history

**🪤 Trap:**
- They offer pre-commit hooks as the guarantee → wrong, because `--no-verify`, bots and the web UI skip them.
- They offer deleting the pipeline run as the fix → wrong, because that's containment; rotation is the fix.

**🎯 Question says → you pick:**
- "Prevent secrets pushed to ANY repo, regardless of local config" → org-level push protection
- "`$(fetchSecrets.API_KEY)` is empty" → step missing `name:` or `isOutput=true`
- "Secret variable empty in a later step" → map it with `env:`

**💡 Hook:** Bouncer beats door sign, CCTV only watches, photographed key = new locks.

### 🛡️ 44 · GHAS: four guards, one report
GitHub Advanced Security is four guards who never swap jobs. CodeQL reads code you wrote, Dependabot checks packages you imported, push protection stops secrets at the door (alerts only record them), and Trivy X-rays the built image. They all report in SARIF to the Security tab. The rule: match the finding to the right guard, and land results in the pull request where developers look.

**🧠 Lock these in:**
- CodeQL: `init@v3` → build → `analyze@v3`; compiled languages need an explicit build, interpreted use `autobuild`
- `queries: +security-extended` adds; without `+` it REPLACES the default suite
- CodeQL job needs `security-events: write`
- Container scan: output `sarif` → `upload-sarif@v3` with a `category:`
- Transitive vuln, no upstream fix → npm `overrides` or yarn `resolutions`
- GHAzDO: Dependency-Scanning, Codeql-Init, build, Codeql-Analyze, Publish; no upgrade PRs

**🪤 Trap:**
- They offer missing `security-events: write` for zero CodeQL results → wrong, because that fails loudly; silent zero = missing build.
- They offer `gh pr merge --admin` for Dependabot auto-merge → wrong, because it bypasses branch protection; use `--auto`.

**🎯 Question says → you pick:**
- "Critical alert stays open for months" → alerts on, version updates off
- "Detect the company's internal API key format" → org-level custom secret scanning pattern
- "Block only critical container vulns, report more" → two Trivy passes (SARIF + exit-code 1)

**💡 Hook:** In CodeQL, "+" means plus; drop it and you get "instead".

### 🏗️ 45 · Defender for DevOps: the building inspector
Contoso's 30 GitHub and 15 Azure DevOps repos have two dashboards, so nobody sees total risk. Defender for Cloud DevOps Security is the building inspector: it walks every room on both platforms noting "no smoke detector installed", but scans no code itself. Rule 1: Defender detects a setting, never a vulnerability. Rule 2: its PR sticky note only informs; blocking needs a failing scan job plus a required status check.

**🧠 Lock these in:**
- Posture: code scanning off, branch protection missing, no required reviewers
- One connector per organisation; GitHub uses the "Microsoft Defender for Cloud" GitHub App
- Connector Disconnected → app uninstalled or revoked: Reauthorize
- Auto-discovery (on by default) covers NEW repos; missing EXISTING repos = app's repository selection
- Governance rule = owner + deadline; it doesn't set severity or block
- PR annotations: High and Critical, "Comment only (do not block merge)"
- Azure DevOps annotations: `MicrosoftSecurityDevOps@1`, `trigger: none` + `pr:`, publish `.gdn`

**🪤 Trap:**
- They offer "Defender finds SQL injection" → wrong, because that's CodeQL.
- They offer an expired TLS certificate as DevOps posture → wrong, because that's cloud resource posture (CSPM).

**🎯 Question says → you pick:**
- "Single pane of glass across GitHub and Azure DevOps" → Defender DevOps connectors for both
- "Require code scanning across the organisation" → Azure Policy assignment

**💡 Hook:** Setting or vulnerability? Inform or block?

---


## 🏁 Part 8 — Domain 5 · Instrumentation + the Capstone

Domain 5 is only 5–10% of the exam, so these are the cheap marks. Grab every one. The big theme: telemetry is useless until it's tied to a deployment and something acts on it. Then the capstone throws all five domains into one PCI-DSS blender. 🍹

### 🚨 46 · Alerts that actually do something

Contoso's error rate sat on a dashboard for 8 hours after the 2:15 PM deploy: a smoke alarm with no batteries. The fix is detect (metric or log alert), decide (threshold + window), act (action group). Application Insights never notices a deployment, so the pipeline must push an annotation. Webhooks carry no auth, so an Azure Function or Logic App must sit between alert and pipeline.

**🧠 Lock these in:**
- Annotation: `az rest --method put`, Annotations API `api-version=2015-05-01`, Category `Deployment`, time via `date -u`.
- Metric alert = pre-aggregated `Http5xx`, fast. Log alert (`scheduled-query`) = KQL over `exceptions`, heavier.
- `--window-size` = data per check. `--evaluation-frequency` = how often it checks.
- `webhook` = unauthenticated POST (Teams/Slack too). `azurefunction` = code that can authenticate. `useCommonAlertSchema` = one payload.
- Gate "Query Azure Monitor alerts" (filter Fired): Pre-deployment conditions › Gates, not YAML.
- `sleep 120` first. `exit 1` fails the stage (`logissue` only reports). Rollback = slot swap.

**🪤 Trap:**
- They offer Smart detection + email for auto-rollback → wrong, because it notifies but triggers nothing.
- They offer querying right after deploy → wrong, because ingestion lag shows 0 errors.

**🎯 Question says → you pick:**
- "webhook sent, pipeline never starts" → a Function or Logic App that authenticates.
- "annotations don't show" → wrong resource ID, non-UTC time, or no Contributor.
- "external system starts a GitHub workflow" → `repository_dispatch`.

**💡 Hook:** 📢 The alarm only shouts. The Function holds the phone and PIN to call the fire brigade.

### 🏥 47 · One hospital, three doctors

Contoso runs VMs, AKS and App Service, monitored three different ways (one not at all). Picture a hospital. Application Insights listens to the patient (your code), VM Insights checks the building (the machine), Container Insights watches the wards (the cluster). They add together in one Log Analytics workspace, and a patient is only followed across departments if everyone passes the wristband: `traceparent`.

**🧠 Lock these in:**
- Codeless App Service: `APPLICATIONINSIGHTS_CONNECTION_STRING`, `ApplicationInsightsAgent_EXTENSION_VERSION=~3`, `XDT_MicrosoftApplicationInsights_Mode=Recommended`, then restart.
- VM Insights = Azure Monitor Agent + DCR + DCR association. Streams: `Microsoft-InsightsMetrics` (Performance), `Microsoft-ServiceMap` (Map).
- AKS: `enable-addons --addons monitoring` + `--enable-azure-monitor-metrics` (managed Prometheus). Enable both.
- ConfigMap `container-azm-ms-agentconfig`: edit `exclude_namespaces`, keep `env_var` off, then restart the `omsagent` daemonset.
- Traces correlate on `operation_Id`. Give each service its own `OTEL_SERVICE_NAME`.
- `excludedTypes: "Event;Exception"` in adaptive sampling = every error kept.

**🪤 Trap:**
- They offer the SDK for "no code changes" → wrong, because codeless attach is app settings plus a restart.
- They offer a daily cap as the cost plan → wrong, because it's a hard stop that hits first on incident days.
- They offer `includedTypes` → wrong, because those get sampled, so exceptions are thrown away.

**🎯 Question says → you pick:**
- "agent healthy, DCR correct, no data" → the DCR isn't associated with the VM.
- "trace stops at one service" → `traceparent` not passed on.
- "which processes talk to external services" → VM Insights Map.

**💡 Hook:** 🩺 Doctor, engineer, ward manager: one shared chart, and always pass the wristband.

### 🐙 48 · GitHub's house of rooms

The manager can't say the average build time, which workflow fails most, or how often teams deploy. Treat GitHub like a house: code lives in Repository Insights, pipelines in the Actions API, work items in Projects Insights, deliveries in the Deployments API. For alerts, never put the smoke detector inside the oven. A step at the end of the build job doesn't run when the job dies early.

**🧠 Lock these in:**
- Projects Insights = burn-down, cycle time, items by assignee. Never Repository Insights.
- `status` = lifecycle (completed). `conclusion` = outcome (success, failure, cancelled, skipped).
- No duration field: `updatedAt` minus `startedAt`, successful runs only (`createdAt` includes queue time).
- Deployment frequency = `gh api repos/{o}/{r}/deployments` + `group_by(.environment)`.
- Traffic API needs push access (403 otherwise).
- Azure DevOps: failure + recovery = two notification subscriptions. `##vso[task.logissue type=warning]` reports without failing.

```yaml
notify-failure:
  runs-on: ubuntu-latest
  needs: [build]
  if: failure()
```

**🪤 Trap:**
- They offer `needs:` without `if: failure()` → wrong, because it fires every run and gets muted.
- They offer `continue-on-error: true` on build → wrong, because the job reports success, so `failure()` never fires.
- They offer averaging duration over all runs → wrong, because failed runs end early and fake a speed-up.

**🎯 Question says → you pick:**
- "burn-down / cycle time" → Projects Insights.
- "notify job green, Slack silent for weeks" → expired webhook: curl a test, new webhook, `gh secret set`.

**💡 Hook:** 🔥 Mount the smoke detector on the wall next door (a separate job), not inside the oven.

### 🕵️ 49 · KQL, the nightclub bouncer

The SRE team eyeballs App Insights after every deploy. KQL turns "did the deploy cause it?" into a query you can repeat and alert on. Most questions are "pick the operator", like a bouncer with four moves. Two silent killers: `customDimensions` needs `tostring()`, and `100` instead of `100.0` turns every rate below 1% into zero.

**🧠 Lock these in:**
- Slowest endpoints: `summarize percentile(duration, 95), count() by name` + `where requestCount > 10`. No `by name` = single requests.
- `toscalar()` = the baseline as one number, e.g. `currentRate > toscalar(baseline) * 3`.
- `join kind=leftanti` = what's NEW. `fullouter` = keeps one-sided rows. `inner` = both sides only.
- `countif(success == false)` next to `count()` = error rate in one pass.
- `bin(timestamp, 5m)` + `render timechart`. `serialize` before `prev()`.
- Tables: `requests` inbound, `dependencies` outbound, `customEvents` deployment annotations.
- Azure DevOps Analytics OData = aggregate history: `$filter`, `$select`, word operators `gt`/`lt`. Share workbooks via Monitoring Reader.

**🪤 Trap:**
- They offer `where success == false` before `summarize` → wrong, because it deletes the denominator (rate always 100%).
- They offer "not indexed" for an empty `customDimensions` filter → wrong, because it's dynamic: cast with `tostring()`.

**🎯 Question says → you pick:**
- "exception types that didn't exist before the deploy" → `distinct type` + `join kind=leftanti`.
- "error rate is a flat zero line" → use `100.0`.
- "alert keeps false-firing" → baseline excluding the present (`ago(2h)`), `requestCount > 20`, floor `> 1.0`.

**💡 Hook:** 🕺 `toscalar` is last week's crowd on the door, `leftanti` spots new faces, `countif` clicks only for the rowdy ones.

### 🐢 50 · Who's really slow?

After a deploy the app feels "slow". Think traffic jam: the car at the back waited longest, but the broken-down truck at the front is the cause, so read the trace from the bottom. Then subtract: request time minus dependency time is your own code. The error budget burn rate (not ticket volume) decides how fast you act.

**🧠 Lock these in:**
- Request 1500ms minus SQL 1200ms = 300ms of your code, so investigate SQL queries and indexes.
- Percentiles, not averages. P50 flat, P99 tripled = tail problem.
- Costliest dependency overall: `avgDuration * slowCallCount`.
- Before/after: `iff(timestamp < deployTime, "Before", "After")`, then summarize by period.
- `VMProcess` by `ProcessName` = which process. Available MBytes falling = memory leak.
- Burn rate 1.0 = budget gone exactly at window end. Fast burn 14.4x over 1h, slow burn 6x over 6h.

**🪤 Trap:**
- They offer the biggest span (Order 4500ms) → wrong, because it's waiting on the child that times out.
- They offer smart detection to trigger rollback → wrong, because it only notifies.
- They offer rollback after a DB schema migration → wrong, because old code on a migrated schema breaks. Hotfix.

**🎯 Question says → you pick:**
- "anomalies without manual configuration" → Smart detection.
- "80% of budget used after 15 of 30 days" → burn rate 1.6: fewer deploys, more testing, no freeze.
- "payment service shows as a separate trace" → `traceparent` not propagated.

**💡 Hook:** 🚗 Blame the truck at the front, not the car at the back.

### 🛫 51 · Capstone: Contoso Payments takes off

The capstone mixes all five domains into one PCI-DSS payment microservice. Think airport: every ticket is tied to a booking (traceability), and security stops bad bags at the gate (push protection). Crew wear one-day badges (OIDC), and a new plane carries 10% of passengers first (canary). When something crashes, read the black box: the `exceptions` table.

**🧠 Lock these in:**
- Traceability = required linked issues + squash merge + commit linting (`Refs: #issue`).
- Push protection BLOCKS the push: rewrite history (filter-branch/rebase) or request a justified bypass.
- OIDC: `azure/login@v2` with client-id, tenant-id, subscription-id, no secret. Federated credential scoped to repo, branch, environment.
- Production: 2 approvals (release-managers), 5-minute wait timer, `main` only. Staging: no approvals.
- Canary fails: 100% back to the previous revision, deactivate the failed one, open an incident issue, notify Teams.
- Hardening: pin actions to full SHA (not `@v4`), job-level least privilege, no self-hosted runners for production.

```yaml
permissions:
  id-token: write
  contents: read
```

**🪤 Trap:**
- They offer "signed commits + branch protection" for traceability → wrong, because they prove who, not which work item.
- They offer "OIDC is faster / more granular / removes RBAC" → wrong, because the win is no long-lived secret.

**🎯 Question says → you pick:**
- "500 spike root cause" → `exceptions` after deployment time, by `type` and `outerMessage`.
- "staging passed, prod `SqlException` invalid column" → add `dotnet ef database update` before the app deploys.
- "strip card numbers from telemetry" → custom `ITelemetryProcessor`.

**💡 Hook:** ✈️ Ticket, gate, badge, test flight, black box.

---
