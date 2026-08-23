---
sidebar_position: 5.5
toc_max_heading_level: 2
title: "Challenge 43: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 43 — AZ-400 exam questions

**48 questions** built only from what Challenge 43 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-43.md`**.

:::danger Read this before you start

Leak prevention is graded by **where the control runs**, and the exam almost always wants the layer
that cannot be skipped.

**Client-side** — `.gitignore`, pre-commit hooks. Helpful, and **bypassable**: `git commit --no-verify`,
a laptop where nobody ran `pre-commit install`, a new starter.
**Server-side** — GitHub push protection. The secret **never enters** the repository.
**After the fact** — CI scanning, log review. Finds what is already committed, which is a **detection**,
not a prevention.

The same split governs pipelines. **Automatic masking** covers values the platform already knows are
secret. **Anything you fetch at run time is unknown to it** until you register it — `::add-mask::` in
GitHub Actions, `isSecret=true` in Azure Pipelines.

And one hard fact behind every question here: **masking a leaked value does not un-leak it.** The
scenario at line 17 had secrets in commit history for three days. The only real remedy is rotation.

:::

---

# Section A — Multiple choice

---

## Q1

A pipeline must deploy using an SSH key that developers must not see and that must not be in source
control. Where should the key live?

- A. A pipeline variable marked as secret
- B. Azure Key Vault as a secret
- C. An Azure Pipelines secure file with restricted pipeline permissions
- D. A private Git repository with limited access

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-43.md`:** lines **30**, **54–64**, **92–94**.

```yaml
  - task: DownloadSecureFile@1
    name: sshKey
    inputs:
      secureFile: 'contoso-prod-deploy.key'
```

**Secure files exist for exactly this shape of secret: a *file*, not a string.** They are encrypted at
rest, downloadable only by the `DownloadSecureFile@1` task, invisible to developers in the Library, and
they carry their own pipeline permissions and approval checks.

**Why A fails on the data type.** A secret variable holds a value; an SSH private key is a multi-line
file with strict formatting and permissions. Pushing it through a variable means reassembling it on the
agent, which is fragile and usually ends with the key echoed while someone debugs it.

**Why B is defensible but not best here.** Key Vault can store a certificate, and for a value the app
reads at run time it would win. For a file a **pipeline task** consumes, secure files are the purpose-
built mechanism — with pipeline permissions and approvals attached (lines 93–94).

**Why D fails outright.** "Private repository" is not encryption. Anyone with read access has the key,
and it is in history forever.

</details>

---

## Q2

A workflow retrieves a secret dynamically through an API call. How do you keep it out of the logs?

- A. Use the `secrets` context, which masks everything automatically
- B. `echo "::add-mask::$SECRET_VALUE"` before using the value
- C. Set `ACTIONS_STEP_DEBUG` to false
- D. Redirect all output to `/dev/null`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-43.md`:** lines **131–139**.

```bash
          # Mask the value so it never appears in logs
          echo "::add-mask::$DB_CONN"
```

**A is true of repository secrets and false of this one.** `secrets.MY_TOKEN` is masked because GitHub
put it there and knows the string. A value fetched from Key Vault or an API at run time is **just text
on the runner** until you register it.

**That distinction is the entire question**, and it is why line 132 exists at all.

**Why the others fail** — C only controls debug verbosity, and the secret would still print from a
normal `echo`. D discards *all* output, including the diagnostics you need, and it does nothing about
output from a tool you did not wrap.

</details>

---

## Q3

Contoso wants to stop developers pushing secrets to **any** repository in the organisation. Which gives
the most comprehensive protection?

- A. Branch protection rules requiring pull-request reviews
- B. GitHub secret scanning push protection at the organisation level
- C. Pre-commit hooks on each developer machine
- D. A CI workflow that scans for secrets after each push

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-43.md`:** lines **330–336** and **367–372**.

```bash
gh api orgs/contoso -X PATCH --input - <<< '{
  "security_and_analysis": {
    "secret_scanning_push_protection": {"status": "enabled"}
  }
}'
```

**Push protection runs on the server, so the secret never enters the repository.** No client
configuration, no opt-in, no way to forget.

**Why C is the answer most people choose, and why it is second-best.** A pre-commit hook is excellent —
it gives the developer feedback in two seconds instead of two minutes. It is also **defeated by
`git commit --no-verify`, by a fresh clone where nobody ran `pre-commit install`, and by any commit
made in a web UI or by a bot.**

**Why D is detection, not prevention.** By the time CI reports it, the secret is in the repository, in
every clone that has fetched, and must be rotated.

**Why A misses on target.** Reviewers approve *intent*; a long diff with one changed config line is
exactly what human review does worst.

</details>

---

## Q4

A variable group is linked to Key Vault. A pipeline prints all variables. Do the secret values appear?

- A. Yes, variable group variables are always visible
- B. No, Key Vault-linked variables are treated as secret and masked
- C. Only if "Allow access to all pipelines" is enabled
- D. Only if the developer holds Key Vault Secrets User

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-43.md`:** lines **387** and Challenge 42's line 211.

```text
      # Azure DevOps automatically masks variables marked as secret
      # but dynamically generated values need explicit masking
```

**Key Vault-linked variables arrive pre-marked as secret**, so Azure Pipelines masks them everywhere —
even when a script deliberately echoes one.

**Read the second line of that comment carefully, because it is the other half of the challenge.**
Masking is automatic for values the platform *knows*. A token fetched with `curl` in a script (line
391) is unknown until `isSecret=true` registers it.

**Why C and D are dials from other challenges.** Pipeline permissions decide *which pipelines may use
the group*; the Key Vault role decides *whether the fetch succeeds*. Neither changes masking behaviour
once the value has arrived.

</details>

---

## Q5

A developer adds `echo $CONNECTION_STRING` for debugging and the password appears in the build log.
What is the cause?

- A. The variable was not marked as secret
- B. Debug logging was enabled
- C. The variable came from Key Vault
- D. The agent was self-hosted

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-43.md`:** line **465**.

**If Azure DevOps does not know a value is secret, it has no reason to mask it.** Masking is a lookup
against a known-secret list, not an analysis of the string's contents.

**Why C is the reversal worth noticing.** A Key Vault-linked variable **would** have been masked (Q4).
The fact that this one printed tells you it was *not* sourced that way.

</details>

---

## Q6

What is the correct **immediate** response to a secret found in a pipeline log?

- A. Delete the run so the log is removed, then rotate the secret
- B. Edit the log to remove the line
- C. Mark the variable as secret so future runs mask it
- D. Nothing; logs are only visible to the team

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-43.md`:** lines **471–475**.

```bash
az pipelines runs delete --id <run-id> --yes
```

**Deleting the run is containment, and it is not the fix.** The challenge presents it as the
*immediate* action alongside a *preventive* one (line 478) — and the exam wants you to know the
containment step is incomplete on its own.

**Rotation is what actually resolves it**, because you cannot know who read the log first. The
challenge's own scenario makes the point: three days of exposure at line 17.

**Why C is preventive-only.** It protects the next run. The value already published stays valid until
somebody changes it.

**Why B is impossible** — pipeline logs are immutable. You delete the run or you do not.

</details>

---

## Q7

What does `isSecret=true` do in a logging command?

- A. Encrypts the variable at rest
- B. Registers the value for masking in all subsequent log output
- C. Restricts the variable to the current step
- D. Prevents the variable being passed to other tasks

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-43.md`:** lines **170–174**.

```bash
        echo "##vso[task.setvariable variable=API_KEY;isSecret=true;isOutput=true]$SECRET"

        # This will print *** in logs
        echo "The API key is: $SECRET"
```

**Those two lines together are the whole demonstration.** After registration, even a deliberate echo
prints `***`.

**Why D is subtly wrong and worth separating.** A secret variable **is not automatically mapped into
the environment** of later steps — which is why line 183–184 passes it explicitly via `env:`. That is a
real behaviour, and it is not what `isSecret` *does*; it is a consequence.

</details>

---

## Q8

What does `isOutput=true` add?

- A. It makes the variable available to later steps and jobs via the step's name
- B. It writes the variable to the build summary
- C. It masks the value
- D. It publishes the variable as a pipeline artifact

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-43.md`:** lines **171–180**.

```bash
        echo "##vso[task.setvariable variable=API_KEY;isSecret=true;isOutput=true]$SECRET"
```

```yaml
    name: fetchSecrets
```

```bash
      echo "Deploying with key: $(fetchSecrets.API_KEY)"
```

**Three things must line up, and the exam removes one of them.** `isOutput=true`, a `name:` on the
step, and the reference written as `$(stepName.variableName)`.

**Drop the `name:` and there is nothing to qualify the reference with.** Drop `isOutput=true` and the
variable exists only within the same job's later steps as `$(API_KEY)` — a different scope entirely.

**Note line 394 versus line 397 in the challenge**: `bearer_token` is set with `isSecret` alone for use
in the same job; `AUTH_TOKEN` adds `isOutput` because a later step references it as `$(auth.AUTH_TOKEN)`.

</details>

---

## Q9

Why does the secure-file cleanup step use `condition: always()`?

- A. To run even if an earlier step failed, so credentials never remain on the agent
- B. To run before the deployment step
- C. To force the pipeline to succeed
- D. To retry the cleanup if it fails

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-43.md`:** lines **452–456**.

```yaml
  - script: |
      rm -f ~/.ssh/deploy.key
      rm -f ~/.kube/config
    displayName: 'Clean up credentials'
    condition: always()
```

**A failed deployment is exactly when credentials get left behind**, and it is also when someone is
most likely to re-run, connect to, or investigate the agent.

**Why this matters more on a self-hosted agent than a Microsoft-hosted one.** A hosted agent is
destroyed after the run, so the residue disappears with it. A self-hosted agent persists — and the next
pipeline, possibly from another team, runs on the same filesystem.

**Note the challenge's own comment at line 82:** secure files are cleaned up automatically. The `rm`
targets the **copies** the script made at lines 71–72 — and copies are what the platform cannot track.

</details>

---

## Q10

What does `chmod 600` accomplish on the downloaded key?

- A. It encrypts the key
- B. It restricts read and write to the file's owner, which SSH requires
- C. It marks the file as a secret
- D. It deletes the file after use

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-43.md`:** lines **436–440**.

```bash
      cp $(sshKey.secureFilePath) ~/.ssh/deploy.key
      chmod 600 ~/.ssh/deploy.key
```

**SSH refuses to use a private key that other users can read** — it fails with "permissions are too
open" rather than warning. So this line is not decoration; without it the deployment does not run.

**And on a shared self-hosted agent it is a genuine control**, because other processes on that machine
are exactly the threat.

</details>

---

## Q11

Which `.gitignore` entry catches Terraform state files that contain secrets?

- A. `*.tf`
- B. `*.tfstate` and `*.tfstate.*`
- C. `terraform/`
- D. `*.tfvars.example`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-43.md`:** lines **212–215**.

```bash
# Terraform state (contains secrets)
*.tfstate
*.tfstate.*
.terraform/
```

**State files record resource attributes in plain text**, including generated passwords, connection
strings and keys — which is why Challenge 32 puts state in a storage account rather than the repository.

**Why A is actively wrong.** `*.tf` is your source code; ignoring it would exclude the infrastructure
definitions you are trying to version.

**And `*.tfstate.*` catches the backups** — `terraform.tfstate.backup` is the file people forget, and it
contains the same secrets.

</details>

---

## Q12

A `.env` file was committed three days ago. Adding it to `.gitignore` now — what does that achieve?

- A. It removes the file from history
- B. It stops future commits of the file, but the committed secrets remain in history and must be
  rotated
- C. It masks the values in the GitHub UI
- D. Nothing; `.gitignore` does not work on `.env` files

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-43.md`:** lines **17**, **237–240**, **583–585**.

```bash
git rm --cached .env
git commit -m "fix: remove tracked secret files"
```

**`.gitignore` only affects files git is not already tracking.** A tracked file keeps being tracked,
which is why `git rm --cached` is needed to stop it — and even that only removes it from the *current*
commit.

**History is the part people miss.** Every clone made in those three days still has the file. Removing
it needs `git filter-repo` and a force push (lines 585–588), which rewrites every commit hash and
disrupts everyone.

**And after all that, the secrets are still compromised.** They were readable for three days. **Rotate
first, clean history second** — in that order, because rotation is what actually ends the exposure.

</details>

---

## Q13

Which command finds sensitive files already tracked by git?

- A. `git status --ignored`
- B. `git ls-files | grep -iE '\.(env|pem|key|pfx|p12)$'`
- C. `git log --all --grep=secret`
- D. `git diff HEAD~1`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-43.md`:** lines **233–235**.

```bash
git ls-files | grep -iE '\.(env|pem|key|pfx|p12)$'
git ls-files | grep -iE '(secret|credential|password|apikey)'
```

**`git ls-files` lists what is *tracked*** — which is the question. The second line catches files named
by purpose rather than extension, like `credentials` or `api-keys.json`.

**Why A answers the opposite question.** `git status --ignored` shows what is being **ignored** — useful
to verify the fix at line 243, useless for finding what is already in.

**Why C searches commit messages**, and nobody writes "adding secret" in a commit message.

</details>

---

## Q14

What does the `[allowlist]` section in `.gitleaks.toml` do?

- A. Permits specific paths and patterns to bypass detection
- B. Lists which secrets are approved for commit
- C. Defines which developers may bypass the hook
- D. Whitelists IP addresses

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-43.md`:** lines **275–280**.

```toml
[allowlist]
paths = [
  '''(.*)_test\.go''',
  '''(.*)/testdata/''',
  '''(.*)\.md'''
]
```

**Test fixtures and documentation are the classic false positives**, and Break scenario 2 (line 499) is
exactly that complaint.

**Why this needs care rather than enthusiasm.** An allowlist is a hole you cut in your own detection.
`'''(.*)\.md'''` exempts **every markdown file in the repository** — so a README with a real
connection string in an example passes silently. Scope allowlists as narrowly as the false positive
requires.

</details>

---

## Q15

What is the difference between `gitleaks detect` and `gitleaks protect --staged`?

- A. `detect` scans history; `protect --staged` scans staged changes for pre-commit use
- B. `detect` is faster
- C. `protect` scans remote branches
- D. They are aliases

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-43.md`:** lines **283–287**.

```bash
gitleaks detect --source . --verbose
gitleaks protect --staged --verbose
```

**Two different jobs.** `detect` audits what is already committed — the right tool for onboarding a
repository or a scheduled CI scan. `protect --staged` inspects what is about to be committed, which is
fast enough to sit in a pre-commit hook.

**Running `detect` in a hook would scan the entire history on every commit** and developers would
uninstall it by Thursday.

</details>

---

## Q16

What does `detect-secrets scan > .secrets.baseline` accomplish?

- A. It removes existing secrets from the repository
- B. It records currently detected findings so the hook flags only *new* ones
- C. It encrypts the findings
- D. It uploads findings to GitHub

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-43.md`:** lines **307–315**.

```bash
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

**A baseline makes an existing repository adoptable.** Without it, a repository with 40 historical
findings blocks every commit until all 40 are resolved, so the tool gets removed.

**And the risk is the mirror image of Q14's.** A baseline generated without reviewing it **blesses real
secrets as known**. Generate it, read it, rotate anything genuine, then commit it.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** are true of Azure Pipelines secure files? (Choose three.)

- A. They are stored encrypted and downloaded only by `DownloadSecureFile@1`
- B. Pipeline permissions can restrict which pipelines may use them
- C. Approvals and checks can be added to them
- D. They are stored in the repository under `.azuredevops/`
- E. Developers can view their contents in the Library
- F. They must be re-uploaded on every run

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-43.md`:** lines **30**, **92–94**.

```text
1. Pipelines > Library > Secure files
2. Select the file > Pipeline permissions: restrict to specific pipelines
3. Select the file > Approvals and checks: add approvers for production certificates
```

**B and C are the same governance model as service connections and environments** (Challenge 41) —
which is the pattern worth carrying into the exam: **anything in the Library is a protected resource
with its own permissions and checks.**

**Why the others fail** — D would put the secret in source control, which is the problem; E is false
(the Library shows the name and metadata, not the contents); F is false, since files persist until
deleted.

</details>

---

## Q18

Which **three** layers does Contoso's design use to stop secrets reaching the repository? (Choose
three.)

- A. `.gitignore` entries for secret file types
- B. Pre-commit hooks running gitleaks
- C. GitHub push protection at the organisation level
- D. Pipeline log deletion
- E. Secure files
- F. Branch protection requiring reviews

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-43.md`:** lines **192–228**, **297–312**, **330–336**.

**Three layers, each catching what the previous one misses.** `.gitignore` stops the accident; the hook
catches a secret pasted into a tracked file; push protection catches everything the first two missed,
including commits from a machine with no hooks.

**Why D and E belong to the other half of the challenge.** They protect **pipeline** output and files,
not source control. Keeping the two halves separate is what makes Section G answerable.

**Why F is not a leak control** — see Q3.

</details>

---

## Q19

Which **two** values need explicit masking? (Choose two.)

- A. A token fetched with `curl` inside a script
- B. A secret read from Key Vault by an inline CLI script
- C. A GitHub repository secret referenced as `secrets.AZURE_CLIENT_ID`
- D. A variable from a Key Vault-linked variable group
- E. A pipeline variable already marked as secret in the UI

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-43.md`:** lines **126–139**, **391–394**.

```bash
      TOKEN=$(curl -s https://auth.contoso.com/token | jq -r '.access_token')
      echo "##vso[task.setvariable variable=bearer_token;isSecret=true]$TOKEN"
```

**The rule is one sentence: if the platform did not hand you the value, it does not know to mask it.**

C, D and E all arrive **through** the platform, already labelled. A and B are produced **on the agent**
at run time, and are plain text until you register them.

</details>

---

## Q20

Which **two** correctly verify a secret loaded without revealing it? (Choose two.)

- A. Print the value's length
- B. Print a SHA256 hash of the value
- C. Print the first four characters
- D. Print the value and delete the log afterwards
- E. Print the value only on failure

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-43.md`:** lines **487–490**.

```bash
      echo "Connection string length: ${#CONNECTION_STRING}"
      echo "Connection string SHA256: $(echo -n "$CONNECTION_STRING" | sha256sum | cut -d' ' -f1)"
```

**Both answer "did the right value arrive?" without disclosing it**, and the hash is the stronger one:
you can compare it against the expected hash and confirm the *exact* value, not just its size.

**Why C is a genuine leak in small clothing.** A prefix is often the identifying part — `sk_live_`,
`pk_live_` (line 382), an account name — and four characters materially narrows a brute-force.

**Why E is the worst of the set.** Failures are precisely when logs get copied into tickets, chats and
screenshots.

</details>

---

## Q21

Which **two** happen when GitHub push protection blocks a push? (Choose two.)

- A. The push is rejected before the commit reaches the repository
- B. The message names the commit, path, line and secret type
- C. The secret is automatically rotated
- D. The repository is locked
- E. The commit is accepted and an alert is raised

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-43.md`:** lines **350–360**.

```text
remote:   locations:
remote:     - commit: abc123def
remote:       path: src/config.js:3
remote:       secret type: Azure Storage Account Key
```

**The detail in the message is what makes the control usable.** The developer gets the file, the line
and what was detected — enough to fix it in one attempt.

**Why E describes secret *scanning* without push protection**, which is the important contrast: alert
after the fact versus block before entry. Both are useful; only one prevents.

**Why C is the wish rather than the behaviour.** GitHub can notify some providers, but rotation is
yours to perform.

</details>

---

## Q22

Which **two** options does a blocked developer have? (Choose two.)

- A. Remove the secret and amend the commit
- B. Bypass with a documented reason, if policy allows
- C. Force push to override the block
- D. Disable secret scanning for the repository
- E. Push to a different branch

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-43.md`:** lines **363–365** and **372**.

```text
4. Allow actors to bypass push protection: Restrict (require review from security team)
```

**The bypass is deliberate and governed.** False positives are real, so a path exists — restricted to
named actors and reviewed, so it leaves a record.

**Why C and E both fail on the same fact.** Push protection is evaluated on the **server**, for
**every** ref. A different branch and a force push both go through it.

**Why D would work and is why line 372 exists.** If any developer can turn scanning off, the control is
advisory. Restricting who may bypass — and who may disable — is what makes it a control.

</details>

---

## Q23

Which **two** reduce false positives from gitleaks without weakening real protection? (Choose two.)

- A. Allowlist specific test paths such as `tests/` and `testdata/`
- B. Allowlist a regex matching an obvious test pattern such as `test_api_key_[a-z0-9]+`
- C. Allowlist `(.*)` to stop all blocking
- D. Remove the hook from the developers who complain
- E. Set `useDefault = false` to disable the built-in rules

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-43.md`:** lines **507–518**.

```toml
[allowlist]
paths = ['''tests/''', '''(.*)_test\.(go|py|js|ts)''', '''testdata/''', '''fixtures/''']

[[allowlist.commits]]
regexes = ['''test_api_key_[a-z0-9]+''']
```

**Both are narrow, and narrowness is the whole point.** A path allowlist for `tests/` still scans every
production file; a regex for `test_api_key_` still catches `sk_live_`.

**Why E is the quiet catastrophe.** `useDefault = true` at line 261 is what brings in gitleaks'
library of AWS keys, GitHub tokens, private key headers and the rest. Turning it off leaves you with
only the two custom Contoso rules — a scanner that finds almost nothing and reports success.

</details>

---

# Section C — Repeated scenario

**Scenario:** After a connection string leaked into pipeline output and a `.env` file with production
API keys sat in history for three days, Contoso must prevent secrets entering source control, prevent
them appearing in pipeline logs, and handle deployment credentials that are files rather than strings.

---

## Q24

**Proposed solution:** Add secret file patterns to `.gitignore` and remove tracked files with
`git rm --cached`. Install gitleaks as a pre-commit hook with a reviewed baseline and narrow
allowlists. Enable secret scanning and push protection organisation-wide, restricting who may bypass.
Store deployment certificates and kubeconfigs as secure files with pipeline permissions. Mask
dynamically fetched values with `::add-mask::` and `isSecret=true`. Rotate every secret that was
exposed.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-43.md`:** lines **192–240**, **297–315**, **330–336**, **367–372**, **424–432**,
**132**, **171**.

| Requirement | Layer |
|---|---|
| Accidental add | `.gitignore` |
| Deliberate paste, caught early | Pre-commit gitleaks |
| Everything else | Server-side push protection |
| Bypass is governed | Restricted actors, security review |
| File credentials | Secure files with pipeline permissions |
| Run-time values in logs | `::add-mask::` / `isSecret=true` |
| Already-exposed values | **Rotation** |

**The last row is the one that is graded hardest.** Every other line prevents the *next* leak; only
rotation resolves the one that already happened.

</details>

---

## Q25

**Proposed solution:** Add `.env` to `.gitignore`, install pre-commit hooks on all developer machines,
delete the pipeline runs that contain the leaked connection string, and instruct developers not to echo
secrets.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures, and they are all the same shape: each control depends on someone doing the right
thing.**

**`.gitignore` does not remove what is already tracked.** The `.env` file stays in history until
`git rm --cached` and a history rewrite (lines 237–240, 585).

**Pre-commit hooks are client-side and bypassable** — `--no-verify`, a fresh clone, a new starter.
Without server-side push protection there is no guaranteed layer at all (Q3).

**Deleting runs is containment, not remediation.** The secrets were readable for three days and by
whoever opened that build log. They remain valid.

**And "instruct developers not to echo secrets" is the control that already failed**, in the first
sentence of the scenario.

**Nothing here rotates anything**, which means every exposed credential is still live.

</details>

---

## Q26

**Proposed solution:** Add secret patterns to `.gitignore` and untrack existing files. Install gitleaks
pre-commit hooks with a reviewed baseline. Enable push protection organisation-wide. Store certificates
as secure files. Mask dynamic values. Rotate exposed secrets. To stop developer friction, allow any
member to bypass push protection and add `'''(.*)'''` to the gitleaks allowlist paths.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right until the final sentence — and compare with Q24, which is otherwise identical.

**`'''(.*)'''` matches every path in the repository.** Gitleaks now scans nothing, while continuing to
run, pass, and appear in the pre-commit output as a green check. **A control that reports success
without inspecting anything is worse than no control**, because it removes the sense that anything is
missing.

**And unrestricted bypass removes the one guaranteed layer.** Line 372's setting exists precisely to
prevent this: bypass is for false positives, reviewed by the security team, leaving a record.

**Together the two changes undo every layer except `.gitignore`** — which the original incident already
proved insufficient.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — masking behaviour

| # | Statement | Answer |
|---|---|---|
| 1 | GitHub repository secrets are masked automatically |  |
| 2 | A value fetched by `curl` at run time is masked automatically |  |
| 3 | Key Vault-linked variable group values are masked automatically |  |
| 4 | `::add-mask::` masks occurrences printed before it ran |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | GitHub repository secrets are masked automatically | **Yes** |
| 2 | A value fetched by `curl` at run time is masked automatically | **No** |
| 3 | Key Vault-linked variable group values are masked automatically | **Yes** |
| 4 | `::add-mask::` masks occurrences printed before it ran | **No** |

**In `challenge-43.md`:** lines **118–120**, **391–394**, **387**, **131–132**.

Rows 1–3 are one rule: **the platform masks what it gave you.**

Row 4 is the ordering rule. Masking applies from registration onwards, so **mask immediately after
retrieval**, before anything can print.

</details>

---

## Q28 — source control protection

| # | Statement | Answer |
|---|---|---|
| 1 | `.gitignore` prevents committing files git already tracks |  |
| 2 | `git rm --cached` stops tracking without deleting the local file |  |
| 3 | Push protection is enforced on the server |  |
| 4 | Pre-commit hooks can be bypassed with `--no-verify` |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `.gitignore` prevents committing files git already tracks | **No** |
| 2 | `git rm --cached` stops tracking without deleting the local file | **Yes** |
| 3 | Push protection is enforced on the server | **Yes** |
| 4 | Pre-commit hooks can be bypassed with `--no-verify` | **Yes** |

**In `challenge-43.md`:** lines **237–243**, **330–336**, **363–365**.

Rows 3 and 4 together are Q3's answer stated as facts — and they are the reason a serious design has
both layers rather than choosing between them.

</details>

---

## Q29 — secure files

| # | Statement | Answer |
|---|---|---|
| 1 | Secure files are removed from the agent after the pipeline completes |  |
| 2 | Copies the script made are also removed automatically |  |
| 3 | Secure files support pipeline permissions and approvals |  |
| 4 | Secure files are visible to any developer in the Library |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Secure files are removed from the agent after the pipeline completes | **Yes** |
| 2 | Copies the script made are also removed automatically | **No** |
| 3 | Secure files support pipeline permissions and approvals | **Yes** |
| 4 | Secure files are visible to any developer in the Library | **No** |

**In `challenge-43.md`:** lines **82–86**, **92–94**.

Row 2 is why the explicit `rm -f` with `condition: always()` exists (lines 452–456). **The platform
tracks the file it placed; it cannot track what your script did with it.**

</details>

---

## Q30 — logging commands

| # | Statement | Answer |
|---|---|---|
| 1 | `isSecret=true` masks the value in subsequent log output |  |
| 2 | `isOutput=true` requires the step to have a `name:` |  |
| 3 | An output variable is referenced as `$(stepName.variableName)` |  |
| 4 | A secret variable is automatically available as an environment variable in later steps |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `isSecret=true` masks the value in subsequent log output | **Yes** |
| 2 | `isOutput=true` requires the step to have a `name:` | **Yes** |
| 3 | An output variable is referenced as `$(stepName.variableName)` | **Yes** |
| 4 | A secret variable is automatically available as an environment variable in later steps | **No** |

**In `challenge-43.md`:** lines **171**, **175**, **180**, **183–184**.

Row 4 is the behaviour that produces "it works locally but the variable is empty in the pipeline". The
fix is at lines 183–184:

```yaml
    env:
      API_KEY: $(fetchSecrets.API_KEY)
```

**Secrets are deliberately not injected into the environment**, so a child process cannot pick one up
by accident. You pass them where you mean to.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each sensitive item to where it should be stored.

| Item | Storage |
|---|---|
| SSH private key used by a deployment task |  |
| Kubeconfig for the production cluster |  |
| Database connection string read by the app |  |
| Azure credentials for the pipeline to log in |  |
| Local developer settings |  |

**Options:** Azure Pipelines secure file · Ignored by `.gitignore`, never committed · Key Vault, via a Key Vault reference · Nothing stored — OIDC

<details>
<summary>Show answer</summary>

| Item | Storage |
|---|---|
| SSH private key used by a deployment task | **Azure Pipelines secure file** |
| Kubeconfig for the production cluster | **Azure Pipelines secure file** |
| Database connection string read by the app | **Key Vault, via a Key Vault reference** |
| Azure credentials for the pipeline to log in | **Nothing stored — OIDC** |
| Local developer settings | **Ignored by `.gitignore`, never committed** |

**In `challenge-43.md`:** lines **424–432**, Challenge 42's line 261, lines **105–120**, **207–210**.

**The deciding question is who consumes it, and in what form.** A *file* consumed by a *pipeline task*
→ secure file. A *value* consumed by the *app* → Key Vault reference. A *login* → no secret at all.

</details>

---

## Q32

Match each leak vector to its control.

| Vector | Control |
|---|---|
| Secret echoed in a pipeline log |  |
| `.env` file committed by accident |  |
| Secret pasted into tracked source |  |
| Credential left on a self-hosted agent |  |
| Secret already in commit history |  |
| Certificate needed by a deployment task |  |

**Options:** Cleanup step with `condition: always()` · GitHub push protection · `.gitignore` + pre-commit hook · `isSecret=true` / `::add-mask::` · Rotate, then rewrite history · Secure file with pipeline permissions

<details>
<summary>Show answer</summary>

| Vector | Control |
|---|---|
| Secret echoed in a pipeline log | **`isSecret=true` / `::add-mask::`** |
| `.env` file committed by accident | **`.gitignore` + pre-commit hook** |
| Secret pasted into tracked source | **GitHub push protection** |
| Credential left on a self-hosted agent | **Cleanup step with `condition: always()`** |
| Secret already in commit history | **Rotate, then rewrite history** |
| Certificate needed by a deployment task | **Secure file with pipeline permissions** |

**In `challenge-43.md`:** lines **171**, **194–228**, **330–336**, **452–456**, **585**, **92–93**.

**Note that only one row is a *response* rather than a prevention.** Once a secret is in history the
prevention window has closed, and rotation comes first because history rewriting takes days to
propagate.

</details>

---

## Q33

Arrange the response to a secret discovered in commit history.

**Items:** Rewrite history with `git filter-repo` · Rotate the exposed credential · Enable push
protection to prevent recurrence · Identify what was exposed and for how long · Remove the file from
tracking and add it to `.gitignore`

<details>
<summary>Show answer</summary>

### Answer

1. Identify what was exposed and for how long — line **17**
2. **Rotate the exposed credential** — the exposure ends here, not later
3. Remove the file from tracking and add it to `.gitignore` — lines **237–240**
4. Rewrite history with `git filter-repo` — line **585**
5. Enable push protection to prevent recurrence — lines **330–336**

**Rotation is second, and that placement is the whole question.** History rewriting is slow, disruptive
and imperfect — forks, existing clones, cached views and any CI system that fetched the commit all
still hold the value. **The moment you rotate, none of that matters.**

**Step 4 is genuinely disruptive**, so it is worth knowing what it costs: every commit hash changes,
every open branch and pull request needs rebasing, and everyone must re-clone. Do it because you should
not leave secrets in a repository — not because it fixes the exposure.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Secret prints in full in an Azure Pipelines log |  |
| Secret prints in a GitHub Actions log |  |
| `$(fetchSecrets.API_KEY)` resolves to nothing |  |
| Gitleaks blocks a legitimate test fixture |  |
| SSH fails with "permissions are too open" |  |

**Options:** Missing `chmod 600` on the key · No allowlist for test paths · Step has no `name:`, or `isOutput=true` missing · Value fetched at run time, never `::add-mask::`d · Variable not marked as secret

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| Secret prints in full in an Azure Pipelines log | **Variable not marked as secret** |
| Secret prints in a GitHub Actions log | **Value fetched at run time, never `::add-mask::`d** |
| `$(fetchSecrets.API_KEY)` resolves to nothing | **Step has no `name:`, or `isOutput=true` missing** |
| Gitleaks blocks a legitimate test fixture | **No allowlist for test paths** |
| SSH fails with "permissions are too open" | **Missing `chmod 600` on the key** |

**In `challenge-43.md`:** lines **465**, **132**, **171–175**, **507–514**, **437**.

**The third row is the one that costs time**, because nothing errors — the variable is simply empty,
and the failure surfaces later as a confusing 401 from whatever API you called.

</details>

---

## Q35

Match each control to the layer it runs at.

| Control | Layer |
|---|---|
| `.gitignore` |  |
| Pre-commit hook |  |
| Push protection |  |
| CI secret scan |  |
| Log masking |  |
| Secure file permissions |  |

**Options:** After the commit — detection · Client-side, advisory · Client-side, bypassable · Platform, before download · Run time, pipeline output · Server-side, enforced

<details>
<summary>Show answer</summary>

| Control | Layer |
|---|---|
| `.gitignore` | **Client-side, advisory** |
| Pre-commit hook | **Client-side, bypassable** |
| Push protection | **Server-side, enforced** |
| CI secret scan | **After the commit — detection** |
| Log masking | **Run time, pipeline output** |
| Secure file permissions | **Platform, before download** |

**In `challenge-43.md`:** lines **192**, **312**, **330–336**, **283–284**, **171**, **93**.

**Rank them by "can a developer skip this?"** Only push protection and secure file permissions answer
no. That ranking is how the exam decides between two otherwise-correct options.

</details>

---

# Section F — Hot area

---

## Q36

```yaml
  - task: [BLANK 1]
    name: sshKey
    inputs:
      [BLANK 2]: 'contoso-prod-deploy.key'

  - script: |
      cp $([BLANK 3]) ~/.ssh/deploy.key
      chmod 600 ~/.ssh/deploy.key
```

- **BLANK 1:** `DownloadSecureFile@1` / `DownloadPipelineArtifact@2` / `AzureKeyVault@2` /
  `CopyFiles@2`
- **BLANK 2:** `secureFile` / `fileName` / `artifact` / `path`
- **BLANK 3:** `sshKey.secureFilePath` / `sshKey.path` / `secureFile.location` / `Agent.TempDirectory`

<details>
<summary>Show answer</summary>

### Answer: `DownloadSecureFile@1`, `secureFile`, `sshKey.secureFilePath`

**In `challenge-43.md`:** lines **424–437**.

**`secureFilePath` is an output of the task, qualified by the step's `name:`.** Without `name: sshKey`
there is nothing to write in front of the dot — the same pattern as output variables in Q8.

**Why `DownloadPipelineArtifact@2` is the substantive distractor.** Artifacts are pipeline **outputs**,
visible to anyone who can view the run. A secure file is a pipeline **input**, encrypted, with its own
permissions.

</details>

---

## Q37

```bash
SECRET=$(az keyvault secret show --vault-name kv-... --name ApiKey --query value -o tsv)
echo "##vso[task.setvariable variable=API_KEY;[BLANK 1];[BLANK 2]]$SECRET"
```

Requirement: mask the value in logs **and** make it available to a later step as `$(fetchSecrets.API_KEY)`.

- **BLANK 1:** `isSecret=true` / `isReadOnly=true` / `isMasked=true` / `secret=yes`
- **BLANK 2:** `isOutput=true` / `isGlobal=true` / `scope=job` / `persist=true`

<details>
<summary>Show answer</summary>

### Answer: `isSecret=true`, `isOutput=true`

**In `challenge-43.md`:** line **171**.

```bash
        echo "##vso[task.setvariable variable=API_KEY;isSecret=true;isOutput=true]$SECRET"
```

**Two properties, two jobs: `isSecret` masks, `isOutput` exports.** They are independent — line 394 sets
`bearer_token` with `isSecret` alone, because nothing outside that step needs it.

**And the step still needs `name: fetchSecrets`** (line 175) for the reference to resolve.

</details>

---

## Q38

```yaml
  - script: |
      rm -f ~/.ssh/deploy.key
      rm -f ~/.kube/config
    displayName: 'Clean up credentials'
    condition: [BLANK 1]
```

- **BLANK 1:** `always()` / `succeeded()` / `failed()` / `succeededOrFailed()`

<details>
<summary>Show answer</summary>

### Answer: `always()`

**In `challenge-43.md`:** line **456**.

**`always()` runs even when the pipeline was cancelled**, which `succeededOrFailed()` does not.

**That difference is the exam's target.** A cancelled deployment — someone hit stop mid-rollout — is
exactly when credentials are sitting on a self-hosted agent, and it is the one case
`succeededOrFailed()` misses.

</details>

---

## Q39

```toml
[extend]
useDefault = [BLANK 1]

[[rules]]
id = "azure-storage-key"
[BLANK 2] = '''(?i)AccountKey=[A-Za-z0-9+/=]{86,88}'''

[[BLANK 3]]
paths = ['''(.*)/testdata/''']
```

- **BLANK 1:** `true` / `false`
- **BLANK 2:** `regex` / `pattern` / `match` / `expression`
- **BLANK 3:** `allowlist` / `ignore` / `exclude` / `skip`

<details>
<summary>Show answer</summary>

### Answer: `true`, `regex`, `allowlist`

**In `challenge-43.md`:** lines **260–279**.

**`useDefault = true` is the one that matters most.** It brings in gitleaks' maintained rule library —
AWS keys, GitHub tokens, private key headers, Slack webhooks. The two custom Contoso rules **extend**
that set; they do not replace it.

**Set it to `false` and the scanner only knows about `contoso_api_` and `AccountKey=`** — and it will
pass a commit containing a live AWS key while reporting no findings (Q23).

</details>

---

## Q40

```bash
gh api orgs/contoso -X [BLANK 1] --input - <<< '{
  "security_and_analysis": {
    "secret_scanning": {"status": "[BLANK 2]"},
    "[BLANK 3]": {"status": "enabled"}
  }
}'
```

Requirement: detect secrets **and** block pushes containing them, organisation-wide.

- **BLANK 1:** `PATCH` / `POST` / `PUT` / `GET`
- **BLANK 2:** `enabled` / `disabled` / `auto` / `true`
- **BLANK 3:** `secret_scanning_push_protection` / `push_rules` / `branch_protection` /
  `secret_scanning_validity_checks`

<details>
<summary>Show answer</summary>

### Answer: `PATCH`, `enabled`, `secret_scanning_push_protection`

**In `challenge-43.md`:** lines **331–335**.

**Two separate settings, and the exam offers one where two are needed.** `secret_scanning` **detects**
and alerts; `secret_scanning_push_protection` **blocks**. Enabling only the first gives you an alert
after the secret is in the repository — Q21's contrast.

**`PATCH` because you are modifying one section** of an existing organisation, not creating it.

</details>

---

## Q41

```bash
# Stop tracking a file that was already committed
git [BLANK 1] .env
git commit -m "fix: remove tracked secret files"

# Verify it is now ignored
git status [BLANK 2] | grep .env
```

- **BLANK 1:** `rm --cached` / `rm` / `reset` / `checkout`
- **BLANK 2:** `--ignored` / `--short` / `--untracked-files=no` / `--porcelain`

<details>
<summary>Show answer</summary>

### Answer: `rm --cached`, `--ignored`

**In `challenge-43.md`:** lines **238–243**.

**`git rm --cached` removes the file from the index and leaves it on disk** — which is what you want,
since developers still need their local `.env` to run the application. Plain `git rm` deletes it and
breaks everyone's environment.

**And neither command touches history.** After this the file is gone from `HEAD` and still present in
every earlier commit (Q12).

</details>

---

# Section G — Case study

## Case study: Contoso leak prevention

### Background

A Contoso developer **logged a database connection string** in pipeline output while debugging a failed
deployment. The same week, another developer **committed a `.env` file containing production API keys**.
The secrets were **exposed in commit history for three days** before anyone noticed.

### Requirements

**Source control**

- Secrets must not be committable, regardless of what any individual developer has configured locally
- Legitimate test fixtures must not be blocked
- Files already committed must be removed from tracking and from history

**Pipelines**

- Values fetched at run time must never appear in logs
- Deployment credentials that are *files* — SSH keys, kubeconfigs — must not be visible to developers
  and must not remain on the agent
- Verification that a secret loaded must not reveal it

**Incident response**

- Exposed credentials must be treated as compromised
- Bypasses of the controls must be governed and recorded

---

## Q42

Which control guarantees a secret cannot enter the repository?

- A. GitHub secret scanning push protection at the organisation level
- B. A pre-commit hook running gitleaks
- C. A comprehensive `.gitignore`
- D. A CI job that scans each push

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-43.md`:** lines **330–336** and **367–371**.

**"Regardless of what any individual developer has configured locally" is the requirement**, and it
eliminates every client-side option by definition.

**Why B is the right second layer and the wrong answer here.** Hooks give immediate feedback and cost
nothing to run — keep them. They are just not a guarantee, because `--no-verify` exists and because a
hook only runs on a machine where someone installed it.

**Why D is detection.** A CI job runs after the push has been accepted.

</details>

---

## Q43

How should the SSH key and kubeconfig be handled?

- A. Secure files with pipeline permissions, downloaded by `DownloadSecureFile@1`, with a cleanup step
  using `condition: always()`
- B. Base64-encoded into secret pipeline variables
- C. Stored in a private repository the pipeline clones
- D. Committed encrypted with a passphrase in a variable

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-43.md`:** lines **424–456**.

**Three requirements, three parts of the answer:** secure files keep them out of developers' hands,
pipeline permissions limit which pipelines may download them, and the always-run cleanup removes the
copies the script made.

**Why B is the workaround people actually build.** It works — and it turns a structured file into a
long opaque string that gets reassembled on the agent, so any debugging step that echoes it leaks the
whole key at once. Secure files exist so that nobody needs this.

**Why C and D put credentials in source control** in different costumes. D also leaves the passphrase
in a variable, so the repository and the pipeline together hold everything an attacker needs.

</details>

---

## Q44

How should a run-time token fetched by `curl` be handled in Azure Pipelines?

- A. `##vso[task.setvariable variable=AUTH_TOKEN;isOutput=true;isSecret=true]$TOKEN`, then pass it to
  later steps through `env:`
- B. Echo it to confirm it was retrieved, then use it
- C. Write it to a file on the agent and read it back
- D. Rely on automatic masking

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-43.md`:** lines **391–408**.

```bash
      echo "##vso[task.setvariable variable=AUTH_TOKEN;isOutput=true;isSecret=true]$TOKEN"
```

```yaml
    env:
      AUTH_TOKEN: $(auth.AUTH_TOKEN)
```

**Why D is the trap that connects this challenge to the last one.** Automatic masking is real — for
Key Vault-linked variables and variables marked secret in the UI (Q4). A value produced by `curl` on
the agent is **not** one of those, and assuming otherwise is how the original incident happened.

**Why C is worse than it looks.** A file on the agent persists past the step, is readable by anything
else running there, and is not covered by masking at all.

</details>

---

## Q45

Which **two** satisfy "verification must not reveal the secret"? (Choose two.)

- A. Print the value's character length
- B. Print a SHA256 hash of the value
- C. Print the first and last four characters
- D. Print the value once and delete the run afterwards
- E. Compare the value against the expected value and print the result of the comparison

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-43.md`:** lines **487–490**.

**E is deliberately included because it is genuinely good practice** — and it is not what lines 487–490
implement, and it presumes you already have the expected value on the agent to compare against, which
is a second secret to manage. **A and B are what the challenge does**, and B is the strong one: a hash
confirms the exact value without disclosing any of it.

**Why C leaks the identifying part** (Q20) and **why D is the behaviour that caused the incident.**

</details>

---

## Q46

How should the already-committed `.env` file be handled?

- A. Rotate the keys immediately, untrack the file, add it to `.gitignore`, then rewrite history with
  `git filter-repo`
- B. Add it to `.gitignore` and move on
- C. Delete the repository and start a new one
- D. Rewrite history first, then decide whether rotation is needed

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-43.md`:** lines **237–243**, **583–585**, **17**.

**Rotation first, because the exposure ends at rotation and not at the rewrite.** Three days of history
means forks, clones, CI caches and anyone who browsed the file — none of which a rewrite reaches.

**Why D inverts the order and is the most common real-world mistake.** Teams spend a day on
`filter-repo`, force-push, ask everyone to re-clone, and only then ask whether the keys were used in the
meantime.

**Why C would destroy issues, pull requests, history and CI configuration** to solve something
`filter-repo` handles.

</details>

---

## Q47

Four months later, a scheduled pipeline that has been deploying successfully begins failing. The log
shows `Permission denied (publickey)`. The team recently migrated from Microsoft-hosted agents to a
self-hosted pool, and a colleague mentions the deployment key file "was already there" on the agent
before the migration finished.

What is the most likely cause, and what should the response be?

- A. A previous run left the copied key on the persistent agent; the pipeline was relying on that
  residue and it has since been cleaned. Restore the `DownloadSecureFile@1` step and the always-run
  cleanup
- B. The secure file expired and must be re-uploaded
- C. Push protection is blocking the deployment
- D. The `chmod 600` is no longer needed on self-hosted agents

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-43.md`:** lines **424–440** and **452–456**.

**This is the failure mode that makes the cleanup step matter.** On a Microsoft-hosted agent the
machine is destroyed after each run, so a missing download step fails immediately and obviously. On a
**self-hosted** agent the filesystem persists — so a pipeline that stopped downloading the key kept
working, silently, on a leftover copy, until something removed it.

**Two problems, not one.** The pipeline has a hidden dependency on agent state, and a production
deployment key was sitting readable on a shared machine for months — available to every other pipeline
that ran there.

**The response is both halves:** restore the download and the `condition: always()` cleanup, and treat
the key as exposed and rotate it.

**Why the others fail** — secure files do not expire (Q17), push protection governs pushes rather than
deployments, and `chmod 600` matters **more** on a shared agent, not less.

</details>

---

## Q48

Six months after the remediation, a developer opens a pull request adding
`'''(.*)\.json'''` to the gitleaks allowlist, explaining that a test fixture keeps being flagged.

What should the reviewer say, and what does this illustrate?

- A. Reject it and allowlist the specific fixture path instead — a repository-wide JSON exemption
  silently disables scanning for the most common config format, while the tool keeps reporting success
- B. Approve it; false positives waste developer time
- C. Approve it and rely on push protection as the real control
- D. Reject it and remove gitleaks entirely, since push protection is sufficient

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-43.md`:** lines **275–280** and **507–514**.

```toml
[allowlist]
paths = ['''tests/''', '''(.*)_test\.(go|py|js|ts)''', '''testdata/''', '''fixtures/''']
```

**Look at what the challenge itself allowlists: paths and file-naming conventions that identify test
code.** Not a file *format*. `appsettings.json`, `local.settings.json` and every service account key
file are JSON — the exemption covers exactly the files most likely to hold a real secret.

**What makes this dangerous rather than merely wrong is that nothing changes visibly.** Gitleaks still
runs on every commit, still takes two seconds, still prints no findings. The team's confidence stays
where it was while the coverage collapses — the same failure as Q26, arriving through a reasonable
pull request instead of a bad design.

**Why C is the argument that will be made in the review**, and why it is wrong: push protection detects
**known secret-provider patterns** — issued tokens with recognisable formats. It does not catch a
Contoso-internal key format, a database password, or a private key in a config file. **The two tools
have different coverage; neither is a superset.** That is why line 261 keeps `useDefault = true` *and*
adds custom rules.

**And why D throws away the fast feedback loop** — the hook is what stops a developer waiting for a
push to learn they made a mistake.

**The lesson to carry into the exam:** when an option weakens a control to reduce friction, the correct
answer is almost always the one that **narrows the exemption to the specific case** rather than
generalising it.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Pre-commit hooks as the guarantee** | Q3, Q25, Q28, Q42 | Client-side. `--no-verify`, fresh clones, bots |
| **`.gitignore` fixing an already-tracked file** | Q12, Q28, Q46 | Needs `git rm --cached`, and history still holds it |
| **Masking assumed automatic for run-time values** | Q2, Q19, Q27, Q44 | The platform masks what it gave you. Nothing else |
| **`::add-mask::` after the value could print** | Q27, Q40 | Not retroactive. Mask immediately after retrieval |
| **Deleting the run instead of rotating** | Q6, Q25, Q46 | Containment, not remediation |
| **Rewriting history before rotating** | Q33, Q46 | Rotation ends the exposure; the rewrite never fully does |
| **`isOutput=true` without a step `name:`** | Q8, Q30, Q37 | The reference has nothing to qualify |
| **Secret variables auto-injected as env vars** | Q30, Q44 | They are not. Pass them via `env:` |
| **Secure-file copies cleaned up automatically** | Q9, Q29, Q47 | Only the original. Your copies need `always()` |
| **`succeededOrFailed()` for cleanup** | Q38 | Misses cancellation. Use `always()` |
| **Broad gitleaks allowlists** | Q14, Q23, Q26, Q48 | Narrow to the fixture path, never a format or `(.*)` |
| **`useDefault = false`** | Q23, Q39 | Discards the entire built-in rule library |
| **`secret_scanning` alone** | Q40 | Alerts after entry. Push protection blocks |
| **Printing a prefix to "verify"** | Q20, Q45 | The prefix is the identifying part. Length or hash |

---

# What to memorise

**In `challenge-43.md`:** lines **171**, **132**, **192–228**, **330–336**, **452–456**.

```text
WHERE DOES THE CONTROL RUN?   (this decides most questions)
  .gitignore          client   advisory      does nothing to tracked files
  pre-commit hook     client   BYPASSABLE    --no-verify, fresh clone, bot commits
  push protection     SERVER   ENFORCED      the secret never enters the repo
  CI secret scan      after    DETECTION     already committed by then
  log masking         runtime  pipeline output only
  secure file perms   platform before the file is ever downloaded

MASKING RULE:  the platform masks what the PLATFORM gave you.
  masked automatically   GitHub repo secrets | Key Vault-linked variable groups | UI secret variables
  NOT masked             anything curl/az/jq produced on the agent  -> register it yourself
```

```yaml
# Azure Pipelines - register a run-time value          (lines 171-184)
- script: |
    SECRET=$(az keyvault secret show ... --query value -o tsv)
    echo "##vso[task.setvariable variable=API_KEY;isSecret=true;isOutput=true]$SECRET"
  name: fetchSecrets            # REQUIRED for the $(fetchSecrets.API_KEY) reference
- script: |
    curl -H "X-API-Key: $(fetchSecrets.API_KEY)" https://api.contoso.com/deploy
  env:
    API_KEY: $(fetchSecrets.API_KEY)     # secrets are NOT auto-injected into the environment

# isSecret=true   -> mask in logs
# isOutput=true   -> expose as $(stepName.varName)  (needs name: on the step)
```

```bash
# GitHub Actions - register a run-time value          (lines 132-139)
DB_CONN=$(az keyvault secret show ... --query value -o tsv)
echo "::add-mask::$DB_CONN"                  # MASK FIRST - not retroactive
echo "db-connection=$DB_CONN" >> $GITHUB_OUTPUT
# mask sub-values too - a password extracted from a connection string is a separate string
```

```yaml
# Secure files - for FILE credentials                 (lines 424-456)
- task: DownloadSecureFile@1
  name: sshKey
  inputs:
    secureFile: 'contoso-prod-deploy.key'
- script: |
    cp $(sshKey.secureFilePath) ~/.ssh/deploy.key
    chmod 600 ~/.ssh/deploy.key         # SSH refuses a world-readable key
- script: rm -f ~/.ssh/deploy.key
  condition: always()                   # always() also covers CANCELLED; succeededOrFailed() does not
# Library > Secure files > Pipeline permissions + Approvals and checks
```

```text
SOURCE CONTROL                                        (lines 192-243, 330-336)
.gitignore    .env* | *.pem *.key *.p12 *.pfx *.cer | *.tfstate* .terraform/
              appsettings.Development.json local.settings.json | .azure/ .aws/ credentials
find tracked  git ls-files | grep -iE '\.(env|pem|key|pfx|p12)$'
untrack       git rm --cached .env        (leaves the local file - plain git rm deletes it)
verify        git status --ignored | grep .env
history       git filter-repo --path .env --invert-paths      then force push

gh api orgs/contoso -X PATCH  "secret_scanning": enabled              <- DETECT + alert
                              "secret_scanning_push_protection": enabled   <- BLOCK
Org Settings > Code security: enable for all repos; RESTRICT who may bypass

gitleaks      useDefault = true          <- keeps the built-in rule library. never false
              detect --source .          history / CI scan
              protect --staged           pre-commit
              [allowlist] paths          narrow: tests/ testdata/ fixtures/ *_test.*
                                         NEVER a file format, never (.*)
detect-secrets scan > .secrets.baseline  READ IT before committing - it blesses what it finds
```

```text
INCIDENT ORDER                                        (lines 17, 237-240, 585)
1  what was exposed, and for how long
2  ROTATE                              <- the exposure ends HERE
3  git rm --cached  +  .gitignore
4  git filter-repo  +  force push      (every hash changes; everyone re-clones)
5  enable push protection

Immediate containment for a leaked LOG:  az pipelines runs delete --id <id> --yes
...but that is containment. Rotation is the fix.
Verify a secret loaded WITHOUT printing it:  ${#VAR}  or  sha256sum
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 44 |
| 38–43 | Re-read the trap index and the layer table, then move on |
| 30–37 | Rewrite the layer table and the masking rule from memory, then retake |
| Below 30 | Redo Tasks 1, 3 and 5 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 43.

:::danger The two questions

**Can a developer skip this control?** If yes, it is a helpful layer, not a guarantee. The exam wants
the server-side one.

**Did the platform give me this value?** If yes, it is masked. If you fetched it yourself, it is plain
text until you register it.

And whatever else you do — **a leaked secret is rotated, not hidden.**

:::
