---
sidebar_position: 5.5
toc_max_heading_level: 2
title: "Challenge 11: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 11 — AZ-400 exam questions

**48 questions** built only from what Challenge 11 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-11.md`**.

:::danger Read this before you start

Two opposite operations, one toolbox, and only one of them is safe.

**Recovery is local and additive.** `git reflog` finds where you were, `git fsck` finds dangling commits,
`git branch <name> <sha>` puts a label back. Nothing is destroyed and nobody else is affected.

**Removal rewrites everything.** `git filter-repo` and BFG change **every commit SHA** from the rewrite
point forward. That means a force push, and every other clone must be **deleted and re-cloned** — never
pulled.

**And the single most important idea in this challenge is the last task, not the first:** *removing a
secret from history does not un-revoke it.* The AWS keys sat in 47 commits across multiple branches for
**three weeks** (line 20). Forks, existing clones, CI caches and anyone who browsed the file still have
them.

**Rotate first. Rewrite second.** Line 309 says it in the challenge's own words: removing the data from
history does not revoke compromised credentials.

The other pair worth fixing now: **`git revert` adds a commit that undoes a change and leaves the
original visible.** It is the right answer for a bad *change* and the wrong answer for a leaked *secret*.

:::

---

# Section A — Multiple choice

---

## Q1

A database password was committed 50 commits ago and exists across three branches. Which tool removes it
from **all** history?

- A. `git filter-repo --replace-text` with rules
- B. `git reset --hard HEAD~50` to drop the commits
- C. `git rm` on the file followed by a commit
- D. `git revert` applied to the offending commit

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-11.md`:** lines **213** and **436**.

```bash
git filter-repo --replace-text expressions.txt
```

**Only `filter-repo` walks every commit on every branch and tag.** The other three touch the tip.

**Why each wrong option fails differently, and this is worth holding onto.** `git rm` deletes the file
**going forward** — every historical commit still contains it. `git reset --hard HEAD~50` discards fifty
commits of legitimate work and does nothing to the other two branches. **`git revert` is the most
dangerous of the three**, because it *looks* like a fix: it adds a commit undoing the change while the
password stays perfectly visible in the original.

</details>

---

## Q2

After `git filter-repo`, what must every other team member do?

- A. Delete the local clone and re-clone from the remote
- B. `git pull --rebase` onto the rewritten history
- C. `git fetch --all` then `git reset --hard origin/main`
- D. Run `git filter-repo` locally with the same flags

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-11.md`:** lines **291–299**.

```text
2. Delete your local clone: rm -rf platform-api
3. Re-clone: git clone https://github.com/contoso/platform-api.git

DO NOT run `git pull` on your existing clone - it will create duplicate history.
```

**Why B is explicitly called out as harmful.** Pulling merges the old history into the new one, producing
duplicate commits — **and reintroducing the very blobs you removed** into that developer's clone, from
which they can be pushed back.

**Why C is closer and still risky.** It fixes `main` and leaves every other local branch, stash and
reflog entry pointing at old objects. **A re-clone is a guarantee; a reset is a hope.**

**Note step 1** (line 294): save uncommitted work first. The procedure is destructive by design.

</details>

---

## Q3

A branch was deleted from local and remote two days ago and nobody has a clone with it. How can the
commits be recovered?

- A. They are permanently lost after the deletion
- B. `git reflog` on any machine that had it checked out
- C. `git fsck --lost-found` run directly on the server
- D. GitHub support — deleted commits are kept for a time

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-11.md`:** lines **66** and **458**.

```bash
# Ask GitHub support for branch restoration (within 90 days) or check:
```

**Read the question's constraint: *nobody has a clone*.** That is what eliminates B, which is otherwise
the right answer and the one this challenge teaches first (line 30).

**Dangling commits survive on the server for roughly 90 days** before garbage collection, so the objects
usually still exist — they simply have no ref pointing at them.

**And line 458 adds the platform difference worth knowing**: on **Azure DevOps**, a deleted branch can be
restored **through the UI** within its retention period, with no support ticket.

</details>

---

## Q4

What is the difference between `git cherry-pick` and `git rebase`?

- A. Cherry-pick preserves the original SHA; rebase changes it
- B. Cherry-pick copies single commits; rebase replays a series
- C. Cherry-pick applies only to merge commits, not others
- D. Cherry-pick moves the commit; rebase copies it

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-11.md`:** line **469**.

**Both create new SHAs, because a commit's hash includes its parent.** The difference is **scope and
intent**: cherry-pick is surgical and one-at-a-time; rebase moves a whole series.

**And the second half of line 469 is the part people miss.** Cherry-pick **leaves the original in place**
— you now have the change in two branches, as two commits. Rebase moves the branch pointer away from the
originals, so the old commits become unreferenced.

**Why A is the most common misconception**, and it matters: if cherry-pick preserved SHAs, Git could tell
that a commit had already been applied. It cannot, which is why cherry-picking the same fix twice
produces a conflict rather than a no-op.

</details>

---

## Q5

Which command finds commits from a deleted branch when the reflog entry is gone?

- A. `git log --all` across every reachable ref
- B. `git branch -a` listing local and remote branches
- C. `git fsck --no-reflogs` to find dangling commits
- D. `git stash list` showing saved working trees

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-11.md`:** lines **40–42**.

```bash
git fsck --no-reflogs | grep "dangling commit"
# dangling commit jkl3456789abcdef...
```

**"Dangling" means the object exists and nothing references it.** `fsck` walks the object database
directly rather than following refs, which is why it finds what `git log --all` cannot.

**`--no-reflogs` is what makes the result meaningful.** Without it, reflog entries count as references,
so commits you could already find are not reported as dangling.

**And recovery is one command** (line 48): `git branch <name> <sha>` puts a label back on the object.

</details>

---

## Q6

A developer made commits in detached HEAD, then checked out `main`. How are the commits recovered?

- A. `git stash pop` to restore the detached work
- B. `git reset --hard` back to the last commit
- C. The commits are lost once HEAD moves
- D. `git reflog` for the SHA, then `git branch`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-11.md`:** lines **82–93**.

```bash
git reflog
# mno7890 HEAD@{1}: commit: fix: critical auth bypass in v1.5.0
# pqr1234 HEAD@{2}: commit: fix: SQL injection in user search

git branch hotfix/recovered-auth-fix mno7890
```

**Detached HEAD is not a lost state — it is an unnamed one.** The commits exist; nothing points at them
except the reflog.

**Two recovery routes, and they differ in where the work lands.** `git branch` recreates the whole line
of work as a branch; `git cherry-pick pqr1234 mno7890` (line 93) copies the individual commits onto
wherever you are now.

**And the reflog is *local*.** If that developer's machine is gone, so is the entry — which is Q3.

</details>

---

## Q7

Which command removes an entire file from all history?

- A. `git filter-repo --path <file>`
- B. `git filter-repo --invert-paths --path <file>`
- C. `git rm --cached <file>` followed by a commit
- D. `git clean -fdx` across the working tree

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-11.md`:** line **204**.

```bash
git filter-repo --invert-paths --path config/aws-credentials.json
```

**`--invert-paths` is the critical flag, and A is the trap.** Without it, `--path` means **keep only**
that path — so option A would delete the entire repository apart from the credentials file.

**That inversion catches people who type the command from memory**, and the consequence is spectacular:
a repository containing one file, and it is the one you were trying to remove.

</details>

---

## Q8

What does `git filter-repo --replace-text` do?

- A. It renames files matching a pattern across history
- B. It replaces text in the working tree only
- C. It replaces matching text in every file in history
- D. It rewrites commit messages matching the rules

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-11.md`:** lines **208–221**.

```text
regex:AKIA[A-Z0-9]{16}==>REMOVED_AWS_KEY
```

```text
AKIAIOSFODNN7EXAMPLE==>***REMOVED***
```

**Two forms in the same syntax.** A line starting `regex:` is a pattern; a plain line is a literal, and
`==>` separates the match from the replacement.

**Why the regex form is the safer choice for credentials.** `AKIA[A-Z0-9]{16}` catches **every** AWS key
ID in the history — including ones nobody knew about — while a literal only removes the one you found.

**And it replaces rather than deletes**, so the file's structure survives and the diff stays readable.

</details>

---

## Q9

Why must BFG run against a `--mirror` clone?

- A. It runs faster against a bare repository
- B. It needs a backup copy of the repo to work from
- C. It cannot read a checked-out working tree
- D. BFG works on bare repos containing every ref

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-11.md`:** lines **240–242**.

```bash
# Clone a fresh mirror of the repo (BFG works on bare repos)
git clone --mirror https://github.com/contoso/platform-api.git platform-api-mirror.git
```

**`--mirror` fetches *every* ref — all branches, all tags, all notes — into a bare repository.** A normal
clone gives you one working branch and remote-tracking refs, so a rewrite would miss history reachable
only from other branches.

**Which matters directly for the scenario**: the keys are in 47 commits **across multiple branches**
(line 20). Rewriting only `main` leaves them live.

</details>

---

## Q10

Which BFG option removes every file over 100 MB from history?

- A. `--strip-blobs-bigger-than 100M`
- B. `--delete-files "*.psd"`
- C. `--delete-folders assets`
- D. `--replace-text passwords.txt`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-11.md`:** line **250**.

```bash
java -jar ../bfg.jar --strip-blobs-bigger-than 100M
```

**Size-based rather than name-based**, which catches large files whose extension nobody listed — the same
idea as `git lfs migrate import --above=1mb` in Challenge 10.

**The other three are the name-based selectors**: `--delete-files` by pattern (line 253),
`--delete-folders` for a directory (line 257), and `--replace-text` for content (line 247).

**Note BFG never touches your latest commit.** It protects the current tip by design, on the assumption
that the present state is the one you want — so the file must already be deleted there.

</details>

---

## Q11

What must follow a BFG run?

- A. `reflog expire` then `gc --prune=now`
- B. `git fsck --full` to verify the object database
- C. `git stash` to save the working tree
- D. Nothing further; BFG completes the cleanup

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-11.md`:** lines **259–261**.

```bash
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

**BFG rewrites the commits; it does not delete the old objects.** They remain in the pack, reachable
through the reflog, until both commands run.

**Which means the secret is still physically present** in that repository until the garbage collection
completes — the same trap as Challenge 10's migration, with much higher stakes here.

**And the verification at line 264** lists objects by size to confirm the large blobs are gone.

</details>

---

## Q12

How do you verify a secret is gone from history?

- A. `git status` reports a clean working tree with no changes
- B. The file is absent from the current working tree
- C. `git log -S` finds nothing and a blob scan finds nothing
- D. `git diff` shows no pending changes in the tree

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-11.md`:** lines **275–283**.

```bash
git log --all -S "AKIAIOSFODNN7EXAMPLE" --oneline

git rev-list --objects --all | while read hash path; do
  if git cat-file -p "$hash" 2>/dev/null | grep -q "AKIA"; then
    echo "FOUND in: $hash $path"
  fi
done
```

**Two checks at two levels, and both are needed.** `log -S` searches **commit diffs** — it finds where the
string was added or removed. The blob scan reads **every object**, which catches content in a blob that
no diff introduced.

**`-S` is the "pickaxe"**, and it is worth knowing by name: it finds commits where the *count* of a
string changed.

**Why B is the mistake that ends investigations early.** A clean working tree tells you about one commit;
the history is what leaks.

</details>

---

## Q13

`git filter-repo` refuses to run: "does not look like a fresh clone". What are the options?

- A. Delete `.git` and reinitialise the repository
- B. Run `git gc` before attempting the rewrite
- C. Upgrade `git-filter-repo` to the latest version
- D. Use `--force`, or a `--no-local` fresh clone

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-11.md`:** lines **408–415**.

```bash
git filter-repo --invert-paths --path secrets.json --force

# Option 2: Create a fresh clone and work from there (recommended)
git clone --no-local platform-api platform-api-clean
```

**The refusal is a safety feature.** `filter-repo` wants to be sure you are not destroying a working
repository with uncommitted work, stashes and local branches you would lose.

**`--no-local` is the flag that makes the second option work.** A local clone normally hardlinks objects
for speed; `--no-local` forces a real copy, so the new repository is genuinely independent of the
original.

**And the challenge marks option 2 "recommended"** — the original survives as your undo.

</details>

---

## Q14

What is the **final** and most critical step after removing the credentials?

- A. Force-push the cleaned history to every remote
- B. Rotate the credentials and check the audit log
- C. Notify every developer that they must re-clone
- D. Add the credentials file to `.gitignore`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-11.md`:** lines **307–321**.

```text
The final critical step - removing the data from history does not revoke compromised credentials
```

```bash
aws iam create-access-key --user-name contoso-platform-svc
aws iam delete-access-key --user-name contoso-platform-svc \
  --access-key-id AKIAIOSFODNN7EXAMPLE
```

**Note the order in those two commands: create the new key *before* deleting the old one.** Deleting
first means every service using it breaks immediately; creating first gives you a window to roll over.

**And the CloudTrail lookup at lines 318–321 is the question nobody wants to ask** — was the key used
during the three-week exposure? — and the only way to know whether this is a cleanup or an incident.

**Why A, C and D are all real steps that resolve nothing.** The keys were public for three weeks; a
history rewrite does not reach forks, existing clones or anyone who copied them.

</details>

---

## Q15

What does the pre-commit hook at Task 8 detect?

- A. Staged content matching key or password patterns
- B. Files larger than a configured size threshold
- C. Commits that add code without matching tests
- D. Commit messages that break the format convention

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-11.md`:** line **335**.

```bash
if git diff --cached --diff-filter=ACM | grep -qE '(AKIA[A-Z0-9]{16}|-----BEGIN (RSA|DSA|EC|OPENSSH) PRIVATE KEY-----|password\s*=\s*["\x27][^"\x27]+["\x27])'; then
```

**Three patterns for three shapes of secret**: an AWS key ID, a PEM private key header, and a quoted
password assignment.

**`--cached` scans the *staged* diff**, which is the only correct scope for a pre-commit hook, and
`--diff-filter=ACM` limits it to Added, Copied and Modified files — a **deleted** file's removed content
should not block the commit that removes it.

**And it is a local hook** (line 332): fast feedback, bypassable with `--no-verify`, which is why line
346 also enables **push protection** server-side.

</details>

---

## Q16

Which server-side control does Task 8 enable?

- A. Branch protection requiring a review
- B. CODEOWNERS routing for the config path
- C. Secret scanning with push protection
- D. Required status checks on the branch

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-11.md`:** lines **345–346**.

```bash
gh api repos/contoso/platform-api --method PATCH \
  -f security_and_analysis='{"secret_scanning":{"status":"enabled"},"secret_scanning_push_protection":{"status":"enabled"}}'
```

**Two settings, two roles** (Challenge 44 Q3): `secret_scanning` **detects** what is already committed,
`secret_scanning_push_protection` **blocks** the push.

**And this is the pairing that makes the whole task coherent.** The hook is client-side and skippable;
push protection runs on the server and cannot be. **Together they are fast feedback plus a guarantee.**

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** recover lost commits? (Choose three.)

- A. `git reset --hard` to the previous commit
- B. `git reflog` to find the SHA of the lost tip
- C. `git clean -fd` to clear untracked files
- D. `git fsck --no-reflogs` to find dangling commits
- E. `git revert` on the deleting commit
- F. `git branch <name> <sha>` to put a ref back

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-11.md`:** lines **30**, **40**, **45**.

**Find, then label.** B and D are two ways of finding an unreferenced commit; F is what makes it reachable
again.

**Why A and C are the opposite operation.** `reset --hard` discards work and `clean -fd` deletes untracked
files — both are how commits get lost, not how they are found.

**Why E belongs to a different problem.** `revert` undoes a *change* in a commit that is still perfectly
present.

</details>

---

## Q18

Which **three** are true of history rewriting? (Choose three.)

- A. Every commit SHA from the rewrite point changes
- B. Existing tags are unaffected by the rewrite
- C. It requires a force push to the remote
- D. `git pull` safely reconciles the new history
- E. Other clones must be deleted and re-cloned
- F. The old objects are removed immediately

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-11.md`:** lines **447**, **286–287**, **294–296**.

**Why D is called out in capitals at line 299** — pulling creates duplicate history **and** restores the
removed blobs into that clone.

**Why B is false and consequential.** Tags point at specific commits; after a rewrite those commits are
orphaned, so the tags reference history no longer on any branch. **That is why line 287 force-pushes
`--tags` as well.**

**Why F is the BFG and `filter-repo` trap** (Q11): the old objects survive until `reflog expire` and
`gc --prune=now`.

</details>

---

## Q19

Which **three** does the challenge use to remove secrets from history? (Choose three.)

- A. `git rm --cached` on the credentials file
- B. `git filter-repo --invert-paths --path <file>`
- C. `git revert` on the commit that added it
- D. `git filter-repo --replace-text <rules file>`
- E. A `.gitignore` entry for the credentials file
- F. BFG `--replace-text <passwords file>`

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-11.md`:** lines **204**, **213**, **247**.

**Two tools, three approaches: delete the file, replace the text, or replace with BFG.**

**Why A, C and E are all in this challenge and none of them removes history.** `.gitignore` (line 324)
prevents the *next* commit; `git rm --cached` untracks going forward; `revert` adds an undo commit. **All
three are legitimate follow-ups and none is the fix.**

</details>

---

## Q20

Which **two** are true of interactive rebase? (Choose two.)

- A. It preserves the original commit SHAs
- B. It is safe to run on shared branches
- C. `squash` combines into the previous, keeping messages
- D. It cannot reorder commits in the sequence
- E. `fixup` combines but discards the fixup's message

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-11.md`:** lines **168–184**.

```text
# Alternative: Use fixup instead of squash (discards the fixup commit message)
# fixup h3 wip: trying different approach
```

**Use `fixup` when the message is "wip" or "fix typo" — nobody wants it in the final history.** Use
`squash` when both messages contain something worth keeping.

**Why A and B are the rules from Challenge 07.** Rebase rewrites, so SHAs change and shared branches must
not be rebased — which is why line 179 force-pushes with lease **"only for feature branches, never
main"**.

**And `--autosquash`** (lines 186–187) automates the reordering: `git commit --fixup=<sha>` creates a
marked commit, and the rebase moves it next to its target automatically.

</details>

---

## Q21

Which **two** cherry-pick behaviours does the challenge demonstrate? (Choose two.)

- A. Cherry-pick preserves the original commit SHA
- B. `git cherry-pick abc1234..def5678` applies a range
- C. Cherry-pick moves the commit off the source branch
- D. `git cherry-pick -m 1 <sha>` picks a merge by parent
- E. Cherry-pick can never produce a conflict

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-11.md`:** lines **123** and **132**.

```bash
git cherry-pick abc1234..def5678
git cherry-pick -m 1 <merge-commit-sha>
```

**`-m 1` exists because a merge commit has two parents**, so "the change this commit introduced" is
ambiguous — you must say which parent to diff against. **`-m 1` means the first parent, normally the
branch that was merged into.**

**Why E is refuted at lines 115–120**, where the challenge walks through resolving a cherry-pick conflict
with `git add` then `--continue`, and `--abort` at line 129 if you change your mind.

</details>

---

## Q22

Which **two** verify that a secret has been removed? (Choose two.)

- A. `git status` reporting a clean working tree
- B. The file being absent from the working directory
- C. `git log --all -S "<secret>"` returning nothing
- D. `git diff HEAD` showing no changes at all
- E. A loop over `rev-list --objects --all` finding no blob

<details>
<summary>Show answer</summary>

### Answer: C, E

**In `challenge-11.md`:** lines **275–283**.

**Diff-level and object-level, and they can disagree.** `-S` searches commit **diffs**; the blob scan
reads the **object database**, which catches a blob still referenced by a tag or a stash.

**And the honest caveat at lines 302–303** is worth carrying: even after both checks pass locally, the
**server** may still hold loose objects until its own garbage collection runs — which is why the
challenge suggests contacting support for repository maintenance.

</details>

---

## Q23

Which **two** prevent a recurrence? (Choose two.)

- A. A pre-commit hook scanning for secret patterns
- B. Adding the credentials file to `.gitignore`
- C. Rotating the credentials that were exposed
- D. GitHub secret scanning with push protection
- E. Training the team on credential hygiene

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-11.md`:** lines **332–341** and **345–346**.

**Client-side speed plus server-side guarantee** (Q16) — the layered pattern from Challenge 43.

**Why B is genuinely useful and narrower than it looks.** `.gitignore` prevents committing **that named
file**; it does nothing about the same key pasted into `config.ts`.

**Why C is the *response*, not the prevention** — essential, and it fixes the incident rather than the
class.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must remove AWS keys present in 47 commits across multiple branches for three
weeks, recover a deleted branch containing three weeks of unmerged work, and do both with minimal
disruption to 15 developers.

---

## Q24

**Proposed solution:** Recover the branch from the reflog or a dangling commit and push it. **Rotate the
AWS keys immediately and check CloudTrail for use during the exposure window.** Then clone a fresh mirror,
run `git filter-repo --replace-text` with a regex covering the key format, verify with `log -S` and a
blob scan, expire the reflog and garbage-collect, force-push all branches and tags, and instruct every
developer to delete their clone and re-clone. Add the credential paths to `.gitignore`, install a
pre-commit secret scan, and enable secret scanning with push protection.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-11.md`:** lines **30–51**, **312–321**, **240–247**, **275–283**, **259–261**,
**286–299**, **324–346**.

| Requirement | Mechanism |
|---|---|
| Branch recovered | reflog or `fsck` → `git branch` → push |
| Exposure ended | **Key rotation, first** |
| Was it used? | CloudTrail lookup over the window |
| History cleaned | `filter-repo --replace-text` on a mirror |
| Verified | `log -S` **and** an object scan |
| Team disrupted minimally | One coordinated re-clone with clear instructions |
| Recurrence prevented | Hook + push protection |

**Rotation being second in the list — before the rewrite — is what makes this answer correct.** Every
other step is housekeeping on a secret that is already public.

</details>

---

## Q25

**Proposed solution:** Run `git revert` on the commit that added the credentials so the change is undone.
Add the file to `.gitignore`. Tell developers to `git pull` to get the fix. Recover the deleted branch by
asking the developer to recreate the work. Rotate the keys next sprint once the cleanup is verified.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Five failures, and the first two are the same misunderstanding.**

**`git revert` leaves the secret fully visible** in the original commit (Q1). Anyone can `git show` it.

**`.gitignore` prevents nothing that has already happened**, and it does not cover the key pasted
elsewhere.

**"Tell developers to `git pull`" is correct here only because nothing was rewritten** — which is the
point: no rewrite happened, so the history still contains the keys on all 15 machines.

**Recreating three weeks of work by hand** discards a recoverable branch. The reflog or `fsck` finds it in
seconds (lines 30–48).

**And "rotate next sprint" inverts the entire priority.** Line 309 is explicit: removing the data does not
revoke the credentials. **Three weeks of exposure is already an incident; deferring rotation extends
it.**

</details>

---

## Q26

**Proposed solution:** Recover the branch from the reflog and push it. Rotate the AWS keys immediately and
check CloudTrail. Clone a fresh mirror, run `git filter-repo --replace-text`, verify with `log -S` and a
blob scan, expire the reflog and garbage-collect, then force-push all branches and tags. Add the paths to
`.gitignore`, install the pre-commit hook, and enable push protection. To avoid disrupting 15 developers
mid-sprint, tell them to run `git pull --rebase` on their existing clones rather than re-cloning.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**`git pull --rebase` against rewritten history replays each developer's local commits onto the new
base — and their local history still contains the old commits.** The result is a clone holding **both**
histories: the cleaned one from the remote and the original, secret-bearing one they never deleted.

**Which means the secret comes back.** The next time one of those 15 developers pushes a branch based on
their old commits, the blobs containing the AWS keys return to the server — and the entire rewrite is
undone by one push.

**Line 299 states it in capitals for exactly this reason:**

```text
DO NOT run `git pull` on your existing clone - it will create duplicate history.
```

**And the stated justification is the trap.** Avoiding disruption is a real concern, and it is precisely
why the challenge scripts the announcement (lines 291–300) with a stash step: **the disruption is a
scheduled ten minutes, not a risk to be avoided.**

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — recovery

| # | Statement | Answer |
|---|---|---|
| 1 | The reflog is local to a machine |  |
| 2 | `git fsck --no-reflogs` finds dangling commits |  |
| 3 | A deleted remote branch is immediately unrecoverable |  |
| 4 | `git branch <name> <sha>` restores a reference |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | The reflog is local to a machine | **Yes** |
| 2 | `git fsck --no-reflogs` finds dangling commits | **Yes** |
| 3 | A deleted remote branch is immediately unrecoverable | **No** |
| 4 | `git branch <name> <sha>` restores a reference | **Yes** |

**In `challenge-11.md`:** lines **26**, **40**, **66**, **45**.

Row 1 is what makes Q3's answer "contact support" rather than "check the reflog".

Row 3: dangling commits survive roughly 90 days, and Azure DevOps offers UI restoration.

</details>

---

## Q28 — rewriting

| # | Statement | Answer |
|---|---|---|
| 1 | `filter-repo` changes every downstream commit SHA |  |
| 2 | A force push of branches **and tags** is required |  |
| 3 | `git pull` is a safe way to receive rewritten history |  |
| 4 | Old objects vanish without `reflog expire` and `gc` |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `filter-repo` changes every downstream commit SHA | **Yes** |
| 2 | A force push of branches **and tags** is required | **Yes** |
| 3 | `git pull` is a safe way to receive rewritten history | **No** |
| 4 | Old objects vanish without `reflog expire` and `gc` | **No** |

**In `challenge-11.md`:** lines **447**, **286–287**, **299**, **259–261**.

Row 3 is Q26's failure, and row 4 is the reason a "completed" cleanup can still contain the secret.

</details>

---

## Q29 — the tools

| # | Statement | Answer |
|---|---|---|
| 1 | `--invert-paths` is required to *remove* a path |  |
| 2 | BFG needs a `--mirror` clone |  |
| 3 | `git revert` removes a secret from history |  |
| 4 | `filter-branch` is the recommended modern tool |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `--invert-paths` is required to *remove* a path | **Yes** |
| 2 | BFG needs a `--mirror` clone | **Yes** |
| 3 | `git revert` removes a secret from history | **No** |
| 4 | `filter-branch` is the recommended modern tool | **No** |

**In `challenge-11.md`:** lines **204**, **240–241**, **436**, **192**.

Row 1 is the flag whose absence keeps only that file (Q7).

Row 4: line 192 calls `filter-repo` **"the modern replacement for the deprecated `git filter-branch`"**.

</details>

---

## Q30 — the aftermath

| # | Statement | Answer |
|---|---|---|
| 1 | Rewriting history revokes the exposed credentials |  |
| 2 | Rotation should create the new key before deleting the old |  |
| 3 | CloudTrail can show whether the key was used |  |
| 4 | A pre-commit hook alone prevents future leaks |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Rewriting history revokes the exposed credentials | **No** |
| 2 | Rotation should create the new key before deleting the old | **Yes** |
| 3 | CloudTrail can show whether the key was used | **Yes** |
| 4 | A pre-commit hook alone prevents future leaks | **No** |

**In `challenge-11.md`:** lines **309**, **313–315**, **318–321**, **332–346**.

Row 1 is the sentence this whole challenge builds to.

Row 4: local hooks are bypassable, which is why push protection is enabled alongside them (Q16).

</details>

---

# Section E — Drag and drop

---

## Q31

Match each situation to the right command.

| Situation | Command |
|---|---|
| A deleted branch, and you had it checked out |  |
| A deleted branch, nobody has a clone |  |
| Commits made in detached HEAD |  |
| Commits with no reflog entry left |  |
| One fix needed on another branch |  |
| A whole messy branch tidied before review |  |

**Options:** Contact support — dangling commits kept ~90 days · `git cherry-pick <sha>` · `git fsck --no-reflogs` · `git rebase -i` · `git reflog` → `git branch <name> <sha>` · `git reflog` → `git branch` or `git cherry-pick`

<details>
<summary>Show answer</summary>

| Situation | Command |
|---|---|
| A deleted branch, and you had it checked out | **`git reflog` → `git branch <name> <sha>`** |
| A deleted branch, nobody has a clone | **Contact support — dangling commits kept ~90 days** |
| Commits made in detached HEAD | **`git reflog` → `git branch` or `git cherry-pick`** |
| Commits with no reflog entry left | **`git fsck --no-reflogs`** |
| One fix needed on another branch | **`git cherry-pick <sha>`** |
| A whole messy branch tidied before review | **`git rebase -i`** |

**In `challenge-11.md`:** lines **30–45**, **66**, **82–93**, **40**, **113**, **153**.

**The first two rows differ by one fact only: does anyone still have the objects locally?** That is what
separates a two-command fix from a support ticket.

</details>

---

## Q32

Match each removal tool to what it does.

| Tool | Does |
|---|---|
| `git filter-repo --invert-paths --path X` |  |
| `git filter-repo --replace-text rules` |  |
| BFG `--replace-text passwords.txt` |  |
| BFG `--strip-blobs-bigger-than 100M` |  |
| `git rm --cached` |  |
| `git revert` |  |

**Options:** Adds an undo commit — the original stays visible · Removes large blobs by size · Removes path X from all history · Replaces matching text everywhere · Same, faster, on a mirror clone · Untracks going forward — history untouched

<details>
<summary>Show answer</summary>

| Tool | Does |
|---|---|
| `git filter-repo --invert-paths --path X` | **Removes path X from all history** |
| `git filter-repo --replace-text rules` | **Replaces matching text everywhere** |
| BFG `--replace-text passwords.txt` | **Same, faster, on a mirror clone** |
| BFG `--strip-blobs-bigger-than 100M` | **Removes large blobs by size** |
| `git rm --cached` | **Untracks going forward — history untouched** |
| `git revert` | **Adds an undo commit — the original stays visible** |

**In `challenge-11.md`:** lines **204**, **213**, **247**, **250**, **436**, **436**.

**The last two rows are the two wrong answers the exam offers most**, and they are wrong in different
ways: one stops the file being tracked, the other announces the change in a new commit.

</details>

---

## Q33

Arrange the response to leaked credentials in Git history.

**Items:** Force-push all branches and tags · Rotate the credentials and check the audit log · Verify with
`log -S` and an object scan · Instruct the team to delete and re-clone · Run `filter-repo` on a fresh
mirror · Expire the reflog and garbage-collect · Enable push protection and a pre-commit scan

<details>
<summary>Show answer</summary>

### Answer

1. **Rotate the credentials and check the audit log** — lines **312–321**
2. Run `filter-repo` on a fresh mirror — lines **240–247**
3. Verify with `log -S` and an object scan — lines **275–283**
4. Expire the reflog and garbage-collect — lines **259–261**
5. Force-push all branches and tags — lines **286–287**
6. Instruct the team to delete and re-clone — lines **291–299**
7. Enable push protection and a pre-commit scan — lines **332–346**

**Rotation is first, and it is first by a wide margin.** Steps 2 to 6 take hours and coordinate 15 people;
step 1 takes two commands and **ends the exposure immediately**. Everything after it is cleaning up
something that can no longer be used.

**Step 4 before step 5 matters too.** Force-pushing before the local garbage collection pushes a
repository that still contains the old objects.

**And step 6 before step 7 is deliberate**: enable push protection after the rewrite, or the cleaned
history's own push may be flagged by the scanner it is meant to protect you from.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| The repository now contains only the credentials file |  |
| The secret reappears after the cleanup |  |
| The rewrite finished but the pack is unchanged |  |
| The keys are gone but were used by an attacker |  |
| The secret survives on another branch |  |
| A cherry-picked fix conflicts on reapplication |  |

**Options:** Cherry-pick creates a new SHA; Git cannot tell · No `reflog expire` and `gc --prune=now` · `--path` without `--invert-paths` · Rewrote a normal clone, not a `--mirror` · Rotation deferred until after the rewrite · Someone ran `git pull` instead of re-cloning

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| The repository now contains only the credentials file | **`--path` without `--invert-paths`** |
| The secret reappears after the cleanup | **Someone ran `git pull` instead of re-cloning** |
| The rewrite finished but the pack is unchanged | **No `reflog expire` and `gc --prune=now`** |
| The keys are gone but were used by an attacker | **Rotation deferred until after the rewrite** |
| The secret survives on another branch | **Rewrote a normal clone, not a `--mirror`** |
| A cherry-picked fix conflicts on reapplication | **Cherry-pick creates a new SHA; Git cannot tell** |

**In `challenge-11.md`:** lines **204**, **299**, **259–261**, **309**, **240–241**, **469**.

**Rows 2 and 5 are the two ways a "completed" removal quietly fails**, and neither produces an error —
the secret is simply still reachable somewhere.

</details>

---

## Q35

Match each control to whether it recovers, removes or prevents.

| Control | Role |
|---|---|
| `git reflog` / `git fsck` |  |
| `git filter-repo` / BFG |  |
| Credential rotation |  |
| `.gitignore` |  |
| Pre-commit secret scan |  |
| Secret scanning push protection |  |

**Options:** Ends the exposure · Prevents locally — bypassable · Prevents server-side — not bypassable · Prevents that named file recurring · Recovers · Removes from history

<details>
<summary>Show answer</summary>

| Control | Role |
|---|---|
| `git reflog` / `git fsck` | **Recovers** |
| `git filter-repo` / BFG | **Removes from history** |
| Credential rotation | **Ends the exposure** |
| `.gitignore` | **Prevents that named file recurring** |
| Pre-commit secret scan | **Prevents locally — bypassable** |
| Secret scanning push protection | **Prevents server-side — not bypassable** |

**In `challenge-11.md`:** lines **30–48**, **204–247**, **312–315**, **324**, **332**, **346**.

**Only one row actually makes the leaked key harmless**, and it is not the one that took the most work.

</details>

---

# Section F — Hot area

---

## Q36

```bash
git reflog | grep "payment-gateway-v2"
git fsck --[BLANK 1] | grep "dangling commit"
git [BLANK 2] feature/payment-gateway-v2 abc1234
git push origin feature/payment-gateway-v2
```

- **BLANK 1:** `lost-found` / `full` / `no-reflogs` / `unreachable`
- **BLANK 2:** `checkout` / `branch` / `reset` / `restore`

<details>
<summary>Show answer</summary>

### Answer: `no-reflogs`, `branch`

**In `challenge-11.md`:** lines **40** and **45**.

**`--no-reflogs` stops reflog entries counting as references**, so commits you could already reach are
not reported — leaving only the genuinely orphaned ones.

**And `git branch <name> <sha>` creates the ref without switching to it**, which is what you want when
you are recovering from another branch. `checkout -b` would move you there.

</details>

---

## Q37

```bash
# Remove an entire file from all history
git filter-repo --[BLANK 1] --path config/aws-credentials.json

# Replace text everywhere
cat > expressions.txt << 'EOF'
[BLANK 2]:AKIA[A-Z0-9]{16}[BLANK 3]REMOVED_AWS_KEY
EOF
git filter-repo --replace-text expressions.txt
```

- **BLANK 1:** `remove-paths` / `delete-path` / `exclude` / `invert-paths`
- **BLANK 2:** `pattern` / `regex` / `match` / `re`
- **BLANK 3:** `=>` / `->` / `==>` / `:`

<details>
<summary>Show answer</summary>

### Answer: `invert-paths`, `regex`, `==>`

**In `challenge-11.md`:** lines **204** and **209**.

```text
regex:AKIA[A-Z0-9]{16}==>REMOVED_AWS_KEY
```

**Omit `--invert-paths` and the command means "keep only this path"** — the single most destructive typo
in this challenge (Q7).

**And `==>` is two equals and a chevron.** A plain line with no `regex:` prefix is a literal match
(line 217), which is the safer form when the secret contains regex metacharacters.

</details>

---

## Q38

```bash
git clone --[BLANK 1] https://github.com/contoso/platform-api.git platform-api-mirror.git
cd platform-api-mirror.git
java -jar ../bfg.jar --[BLANK 2] ../passwords.txt
git reflog expire --expire=now --all
git gc --[BLANK 3]
```

- **BLANK 1:** `bare` / `mirror` / `depth 1` / `no-local`
- **BLANK 2:** `delete-files` / `strip-blobs-bigger-than` / `replace-text` / `clean`
- **BLANK 3:** `auto` / `force` / `all` / `prune=now --aggressive`

<details>
<summary>Show answer</summary>

### Answer: `mirror`, `replace-text`, `prune=now --aggressive`

**In `challenge-11.md`:** lines **241**, **247**, **261**.

**`--mirror` brings **every** ref**, which is the requirement when the secret spans multiple branches
(Q9). `--bare` gives a bare repository without mirroring all refs.

**And `--prune=now` is what overrides `gc`'s safety grace period** for recently-unreachable objects —
without it the blobs stay on disk (Q11).

</details>

---

## Q39

```bash
# Did the secret survive?
git log --all -[BLANK 1] "AKIAIOSFODNN7EXAMPLE" --oneline

git rev-list --objects --all | while read hash path; do
  if git [BLANK 2] -p "$hash" 2>/dev/null | grep -q "AKIA"; then
    echo "FOUND in: $hash $path"
  fi
done
```

- **BLANK 1:** `G` / `p` / `S` / `n`
- **BLANK 2:** `show` / `cat-file` / `blame` / `describe`

<details>
<summary>Show answer</summary>

### Answer: `S`, `cat-file`

**In `challenge-11.md`:** lines **275** and **280**.

**`-S` is the pickaxe: it finds commits where the *number of occurrences* of a string changed.** `-G` is
the near-miss — it matches commits whose **diff text** contains the pattern, which also catches commits
that merely moved the line around.

**And `git cat-file -p` prints an object's contents by hash**, which is how you inspect a blob directly
rather than through a commit.

</details>

---

## Q40

```bash
aws iam [BLANK 1] --user-name contoso-platform-svc
aws iam [BLANK 2] --user-name contoso-platform-svc \
  --access-key-id AKIAIOSFODNN7EXAMPLE

aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=[BLANK 3],AttributeValue=AKIAIOSFODNN7EXAMPLE
```

- **BLANK 1:** `delete-access-key` / `update-access-key` / `create-access-key` / `list-access-keys`
- **BLANK 2:** `create-access-key` / `delete-access-key` / `deactivate-key` / `rotate-key`
- **BLANK 3:** `Username` / `EventName` / `ResourceName` / `AccessKeyId`

<details>
<summary>Show answer</summary>

### Answer: `create-access-key`, `delete-access-key`, `AccessKeyId`

**In `challenge-11.md`:** lines **313–319**.

**Create then delete, in that order** (Q14). Reversing them breaks every service using the key with no
replacement available.

**And the CloudTrail lookup by `AccessKeyId` answers the only question that matters after a three-week
exposure**: was it used, and by whom.

</details>

---

## Q41

```bash
git rebase -i HEAD~8
# pick   h2 feat: add Stripe SDK integration
# [BLANK 1] h3 wip: trying different approach
# [BLANK 2] h5 fix: linting errors

git commit --[BLANK 3]=h2
git rebase -i --autosquash HEAD~5
```

Requirement: fold `h3` in keeping both messages; fold `h5` discarding its message.

- **BLANK 1:** `fixup` / `squash` / `drop` / `edit`
- **BLANK 2:** `squash` / `reword` / `fixup` / `pick`
- **BLANK 3:** `squash` / `fixup` / `amend` / `autosquash`

<details>
<summary>Show answer</summary>

### Answer: `squash`, `fixup`, `fixup`

**In `challenge-11.md`:** lines **168–186**.

**`squash` keeps both messages for you to edit; `fixup` discards the second silently** (Q20) — the whole
distinction between the two verbs.

**And `git commit --fixup=<sha>`** creates a commit titled `fixup! <original subject>`, which
`--autosquash` then moves next to its target automatically — so the rebase editor opens with the
ordering already correct.

</details>

---

# Section G — Case study

## Case study: Contoso credential exposure and branch loss

### Background

Contoso Ltd's platform team has two urgent problems. A developer committed **AWS access keys**
(`AKIA...`) **three weeks ago**; the credentials are in **47 commits across multiple branches**. The
security team found them by automated scanning, and **the keys are still in the history**. Separately, a
senior developer deleted `feature/payment-gateway-v2`, containing **three weeks of unmerged work** with
**no PR**. Both must be resolved while **minimising disruption to 15 developers**.

### Requirements

**Credentials**

- The exposure must be ended as quickly as possible
- Whether the keys were used during the window must be determined
- The keys must be removed from every commit on every branch and tag
- Removal must be verifiable, not assumed

**Branch**

- The three weeks of work must be recovered without recreating it

**Team**

- The 15 developers must end up on the cleaned history with no path back to the secret
- Recurrence must be prevented by something that cannot be skipped

---

## Q42

What is the **first** action, and why?

- A. Run `filter-repo` to clean the history first
- B. Force-push the cleaned branches to the remote
- C. Notify the team to stop pushing to the repo
- D. Rotate the AWS keys and check CloudTrail first

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-11.md`:** lines **307–321**.

```text
The final critical step - removing the data from history does not revoke compromised credentials
```

**Three weeks of public exposure means the keys must be assumed compromised.** Every minute spent on the
history rewrite first is a minute they remain usable.

**And it is cheap.** Two `aws iam` commands versus a multi-hour coordinated rewrite.

**Why C is sensible operationally and not the priority.** Freezing pushes helps the rewrite go smoothly;
it does nothing about a key someone may already be using.

</details>

---

## Q43

How should the branch be recovered?

- A. Ask the developer to redo the three weeks of work
- B. `reflog` or `fsck` for the tip, then branch it
- C. `git revert` the commit that deleted the branch
- D. Restore the repository from a nightly backup

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-11.md`:** lines **30–51**.

**"Without recreating it" is the requirement**, and the commits still exist as unreferenced objects — the
branch deletion removed a *label*, not the work.

**Two routes depending on what survives.** If someone had it checked out, the reflog names the tip. If
not, `fsck` finds it as a dangling commit — and if neither, the objects are on the server for about 90
days (Q3).

**Why C is a category error.** `revert` undoes a **commit**; deleting a branch is not a commit.

</details>

---

## Q44

Which removal approach covers "every commit on every branch and tag"?

- A. `filter-repo` on a mirror; push `--all`, `--tags`
- B. `git rm --cached` on the file, then a commit
- C. `filter-repo` on the working clone of `main` only
- D. `git revert` applied to each of the 47 commits

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-11.md`:** lines **240–247** and **286–287**.

**The mirror is what makes it cover everything** (Q9): all branches, all tags, in one bare repository.

**And the two force pushes are both required** — `--all` for branches and `--tags` separately, because
tags are refs that a branch push does not touch.

**Why C leaves the keys reachable** on every other branch, which the scenario explicitly says exist.

**Why D would create 47 more commits** and leave all 47 originals intact and readable.

</details>

---

## Q45

Which **two** make removal verifiable? (Choose two.)

- A. Checking the file is absent from the working tree
- B. `git log --all -S "<key>" --oneline` returning nothing
- C. `git status` reporting a clean working tree
- D. The file missing from the `main` branch tip
- E. Scanning every object with `rev-list` and `cat-file`

<details>
<summary>Show answer</summary>

### Answer: B, E

**In `challenge-11.md`:** lines **275–283**.

**Diff-level and object-level.** A commit-diff search can miss content in a blob no diff introduced; an
object scan cannot.

**And "verifiable, not assumed" is the requirement's own wording** — which rules out every option that
inspects the current state rather than the history.

</details>

---

## Q46

Which **two** ensure the 15 developers have no path back to the secret? (Choose two.)

- A. Every developer deletes their clone and re-clones
- B. `git pull --rebase` on each existing clone
- C. `git fetch` then `git reset --hard origin/main`
- D. An instruction not to `git pull` on the old clone
- E. Emailing the new history's tip SHA to the team

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-11.md`:** lines **291–299**.

**A removes the old objects; D stops the one action that would put them back.** Both appear in the
announcement, and the second is in capitals for a reason.

**Why C leaves every other local branch, stash and reflog entry** pointing at old objects — a push from
any of them restores the secret (Q26).

**And note the announcement's step 1**: stash or copy uncommitted work first. **The procedure is
destructive by design**, and telling people that in advance is what keeps the disruption to ten minutes.

</details>

---

## Q47

Four months later, the security scanner flags the same AWS key pattern in the repository again. The
history was cleaned and verified at the time, push protection is enabled, and the pre-commit hook is in
the repository's documentation. The finding is on a long-lived feature branch created five months ago
and pushed for the first time last week.

What is the most likely cause?

- A. `filter-repo` missed a branch during the original rewrite
- B. The branch predates the rewrite; old commits were pushed
- C. Push protection was disabled on the repository
- D. The pre-commit hook was never installed by the author

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-11.md`:** lines **291–299** and **346**.

**"Created five months ago, pushed last week" is the whole diagnosis.** The branch was cut from history
that still contained the keys, kept locally through the rewrite, and pushed afterwards — carrying the old
blobs back onto the server.

**This is the failure mode line 299 exists to prevent**, arriving four months late from a developer who
never deleted their clone.

**Why push protection did not stop it, and this is worth understanding.** Push protection scans for
**known secret-provider patterns in the push** — and it may well have flagged this one, which is
plausibly *how* the scanner found it. **What it cannot do is undo the rewrite's assumption** that no old
clones remain.

**Why A is refutable from the evidence.** The rewrite ran on a `--mirror` (Q9) and was verified with a
full object scan (Q45); a missed branch would have shown up then, not five months later on a branch that
did not exist on the server.

**The durable lesson: a history rewrite is only complete when every clone has been replaced**, and that is
a people problem, not a Git one. Track it — a list of developers who have confirmed the re-clone is
worth more than the rewrite itself.

</details>

---

## Q48

A year on, the branch was recovered without losing a day's work, the keys were rotated within an hour,
and no secret has reached the repository since.

Which explanation best accounts for the change?

- A. The team learned not to commit secrets again
- B. The repository was made private to the organisation
- C. Rotation ended exposure; tooling blocks the next attempt
- D. Access to the repository was restricted to fewer people

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-11.md`:** lines **312–321**, **240–247**, **275–283**, **291–299**, **332–346**.

**Take the two problems at line 20 in turn.**

*Three weeks of exposed AWS keys* had two independent parts, and the challenge is structured to make you
notice: **the secret in the history** and **the credential in AWS**. The rewrite fixes the first. Only
rotation fixes the second, and only CloudTrail tells you whether the second one already cost something.

*Three weeks of unmerged work deleted* was never lost at all — the commits were intact and unlabelled.
**Deleting a branch removes a pointer, not the objects**, and knowing that turns a fortnight of rework
into two commands.

**What actually changed is not caution.** A developer will commit a credential again; the scenario's
value is in what happens next.

**The graded idea, and it is the one to carry into the exam: for a leaked secret, the history is the
*second* problem.** Removing it is slow, coordinated and never quite complete — forks, caches and old
clones persist. **Rotation is fast, unilateral and total.** Do that first, then clean up at whatever pace
the team can manage.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **`git revert` offered to remove a secret** | Q1, Q19, Q25, Q29 | Adds an undo commit; the original stays readable |
| **`git rm` assumed to clean history** | Q1, Q19 | Removes going forward only |
| **`--path` without `--invert-paths`** | Q7, Q34, Q37 | Keeps ONLY that path. Catastrophic typo |
| **`git pull` after a rewrite** | Q2, Q18, Q26, Q46 | Duplicate history AND the blobs return |
| **`fetch` + `reset --hard` as a substitute for re-cloning** | Q2, Q46 | Other branches, stashes and reflog still hold old objects |
| **Rewriting a normal clone, not a `--mirror`** | Q9, Q34, Q44 | Other branches keep the secret |
| **Rewrite assumed to delete old objects** | Q11, Q18, Q28 | Needs `reflog expire` + `gc --prune=now` |
| **Rotation deferred until after the cleanup** | Q14, Q25, Q42, Q48 | The rewrite revokes nothing |
| **Deleting the old key before creating the new one** | Q14, Q40 | Breaks every consumer with no rollover |
| **Reflog assumed to be available anywhere** | Q3, Q27 | It is local to a machine |
| **`squash` and `fixup` confused** | Q20, Q41 | `fixup` discards the message |
| **Cherry-pick assumed to preserve the SHA** | Q4, Q34 | New parent, new hash — and the original stays |
| **`-G` used instead of `-S`** | Q39 | `-S` counts occurrences; `-G` matches diff text |
| **`filter-branch` treated as current** | Q29 | Deprecated. `filter-repo` replaces it |

---

# What to memorise

**In `challenge-11.md`:** lines **28–48**, **190–230**, **259–299**, **307–346**.

```text
RECOVER (local, additive)              vs      REMOVE (rewrites everything)
  git reflog                                     git filter-repo   (filter-branch is DEPRECATED)
  git fsck --no-reflogs  -> dangling commit      BFG (on a --mirror clone)
  git branch <name> <sha>                        -> new SHA for EVERY downstream commit
  git cherry-pick <sha>                          -> force push --all AND --tags
  nothing is destroyed                           -> every clone DELETED and RE-CLONED
  reflog is LOCAL. no clone -> ask support        NEVER git pull (duplicate history + blobs return)
  (dangling commits kept ~90 days; ADO restores deleted branches from the UI)
```

```bash
# REMOVING SECRETS                                 (lines 190-230)
git log --all -S "AKIAIOSFODNN7EXAMPLE" --oneline    # -S = pickaxe, counts occurrences
git filter-repo --invert-paths --path config/aws-credentials.json
#                ^^^^^^^^^^^^^ WITHOUT THIS, --path means KEEP ONLY THAT PATH

cat > expressions.txt <<'EOF'
regex:AKIA[A-Z0-9]{16}==>REMOVED_AWS_KEY        # regex: prefix, ==> separator
AKIAIOSFODNN7EXAMPLE==>***REMOVED***            # plain line = literal
EOF
git filter-repo --replace-text expressions.txt

# BFG - needs a MIRROR clone (all refs)
git clone --mirror <url> repo.git && cd repo.git
java -jar bfg.jar --replace-text ../passwords.txt
java -jar bfg.jar --strip-blobs-bigger-than 100M | --delete-files "*.psd" | --delete-folders ".terraform"

# ALWAYS AFTER EITHER TOOL - or the blobs stay on disk
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# VERIFY - two levels
git log --all -S "<secret>" --oneline                    # commit DIFFS
git rev-list --objects --all | while read hash path; do  # every OBJECT
  git cat-file -p "$hash" 2>/dev/null | grep -q "AKIA" && echo "FOUND $hash $path"; done

# "not a fresh clone" refusal
git filter-repo ... --force            |   git clone --no-local <repo> <clean>   (recommended)
```

```bash
# THE ORDER THAT MATTERS                           (lines 307-346)
1  ROTATE          aws iam create-access-key ...   <- CREATE the new one FIRST
                   aws iam delete-access-key --access-key-id AKIA...
2  AUDIT           aws cloudtrail lookup-events --lookup-attributes AttributeKey=AccessKeyId,...
3  rewrite on a mirror     4  verify     5  expire+gc     6  force push --all --tags
7  team deletes clone and RE-CLONES (stash first; DO NOT PULL)
8  prevent:  .gitignore (that file only)
             .git/hooks/pre-commit  scanning the STAGED diff  -> local, --no-verify skips it
             secret_scanning + secret_scanning_push_protection -> SERVER-SIDE, cannot be skipped

REMOVING THE SECRET FROM HISTORY DOES NOT REVOKE IT.  Rotate first, always.
```

```bash
# REBASE / CHERRY-PICK                             (lines 99-188)
git cherry-pick <sha>              # copies ONE commit - NEW sha, original stays on its branch
git cherry-pick abc..def           # a range
git cherry-pick -m 1 <merge-sha>   # a merge commit needs the parent number
git cherry-pick --continue | --abort | --no-commit

git rebase -i HEAD~8
   pick    keep as-is
   squash  fold into previous, KEEP both messages to edit
   fixup   fold into previous, DISCARD this message
git commit --fixup=<sha>  +  git rebase -i --autosquash    # auto-orders the fixups
git push origin feature/x --force-with-lease    # FEATURE BRANCHES ONLY, never main
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 12 |
| 38–43 | Re-read the trap index and the response order, then move on |
| 30–37 | Write the recover-vs-remove table and the eight-step order from memory, then retake |
| Below 30 | Redo Tasks 1, 5 and 8 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 11.

:::danger The one rule

**Rotate first. Rewrite second.**

Removing a secret from Git history does not revoke it. The keys were public for three weeks — forks,
clones and CI caches still have them, and no rewrite reaches those. Two `aws iam` commands end the
exposure; the history cleanup is housekeeping on something that can no longer be used.

And a rewrite is only finished when **every clone has been replaced** — one `git pull` on an old clone
brings the secret straight back.

:::
