---
sidebar_position: 4.5
toc_max_heading_level: 2
title: "Challenge 10: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 10 — AZ-400 exam questions

**48 questions** built only from what Challenge 10 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-10.md`**.

:::danger Read this before you start

**Git LFS solves ONE problem: large binary FILES.** It replaces each tracked file with a small text
**pointer** in the repository and stores the real bytes on an LFS server.

**It does not solve a repository that is large because it has too much HISTORY or too many files.** That
is what **Scalar**, partial clone and sparse-checkout are for (Challenge 12). They are not alternatives
to each other and the exam builds distractors from the confusion.

**Three facts carry most of the marks here.**

**`.gitattributes` is the configuration, and it must be committed.** `git lfs track` only edits that
file. If it is not committed, other clones have no tracking rules and commit raw binaries.

**Migrating existing files rewrites history.** Every commit that touched a migrated file gets a new SHA,
so it needs a force push and everyone must re-clone.

**Locking exists because binaries cannot be merged.** Two people editing a `.psd` is a lost-work
problem, not a conflict problem — Git cannot merge them, so one edit is simply discarded.

The scenario at line 19 is a **50 GB** repository, `.psd` files at **200 MB**, `.fbx` at **500 MB**,
clones taking **over four hours**, push timeouts, and CI runners out of disk.

:::

---

# Section A — Multiple choice

---

## Q1

A developer clones a repository with `GIT_LFS_SKIP_SMUDGE=1`. What do the `.psd` files contain?

- A. Empty files
- B. Text pointer files with the LFS object ID and size
- C. Corrupted binary data
- D. Nothing — the files are absent

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-10.md`:** lines **139–143** and **501**.

```text
version https://git-lfs.github.com/spec/v1
oid sha256:4d7a214614...
size 214958080
```

**Three lines: spec version, SHA-256 of the content, and the byte size.** That is what is actually stored
in the Git object database for every LFS-tracked file — the repository holds pointers, the LFS server
holds bytes.

**The "smudge" filter is what swaps a pointer for content on checkout.** Skipping it leaves the pointer
on disk, and `git lfs pull` later fetches the real thing.

**Why this is deliberately useful rather than a failure** (line 274): a CI job that compiles code does not
need 50 GB of textures, so skipping the smudge makes the checkout fast and cheap.

</details>

---

## Q2

After `git lfs migrate import --include="*.fbx" --everything`, what must the rest of the team do?

- A. Run `git lfs install`
- B. Run `git pull`
- C. Re-clone, because history was rewritten
- D. Delete local `.fbx` files and check out again

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-10.md`:** lines **122–123** and **512**.

```bash
# Migrate existing files to LFS (rewrites history)
# WARNING: This rewrites git history - coordinate with entire team
```

**`--everything` rewrites every commit that touched a migrated file type**, so every SHA downstream
changes — the same mechanic as `git filter-repo` in Challenge 11.

**Why B actively makes it worse.** A `git pull` against rewritten history tries to merge two unrelated
versions of the same project, producing a tangle of duplicate commits.

**Note the less disruptive alternative at line 127**: `--include-ref=refs/heads/main` migrates one branch
rather than all history — still a rewrite, smaller blast radius.

</details>

---

## Q3

What is the primary advantage of git-fat over Git LFS?

- A. Better performance on large files
- B. Native GitHub and Azure DevOps integration
- C. Any storage backend you control — S3, rsync, custom — without depending on the host's LFS
  implementation
- D. Built-in file locking

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-10.md`:** lines **321** and **523**.

```text
| Backend storage | GitHub/Azure DevOps LFS server | S3, rsync, any remote store |
```

**You own the storage, so you own the cost model and the quotas** — which matters when 50 GB of assets
would otherwise be billed per gigabyte of bandwidth (line 239).

**Why B and D are LFS's advantages, offered as git-fat's.** Line 322 gives native integration to LFS;
line 323 gives file locking to LFS and explicitly **No** to git-fat.

**And the trade is stated plainly in the table**: git-fat is self-managed setup, self-managed bandwidth,
self-managed maintenance, with custom CI scripting (lines 324–328). **You are exchanging convenience for
control.**

</details>

---

## Q4

An artist locks `character.fbx` and goes on holiday. Another artist needs it urgently. What is correct?

- A. Delete and recreate the file to bypass the lock
- B. `git lfs unlock --path="character.fbx" --force`, which needs maintain or admin permission
- C. Disable LFS locking on the repository
- D. Edit locally and wait

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-10.md`:** lines **361–362** and **534**.

```bash
# Force unlock someone else's lock (requires admin/maintain permission)
git lfs unlock assets/textures/hero_diffuse.psd --force
```

**`--force` is the documented emergency procedure**, and the permission requirement is what stops it
being routine — breaking someone's lock is a decision, not a convenience.

**Why C is the over-correction.** Disabling locking to solve one stuck lock removes the protection for
every binary in the repository.

**And line 534 adds the governance point worth remembering**: the team should agree a **lock duration
policy** and a delegation path, so the emergency is rare.

</details>

---

## Q5

Cloned `.psd` files contain pointer text. What is the fix?

- A. `git lfs pull`, or install LFS then `git lfs fetch --all` and `git lfs checkout`
- B. Re-clone with `--depth 1`
- C. Run `git gc`
- D. Delete `.gitattributes`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** lines **432–441**.

```bash
git lfs pull

# If LFS wasn't installed before clone, install and fetch:
git lfs install
git lfs fetch --all
git lfs checkout
```

**Two causes, two fixes.** If LFS is installed and the smudge was skipped, `git lfs pull` is enough. If
LFS was **never installed**, Git had no filter to run at all, so you install it and then fetch and
checkout explicitly.

**And `file` is the verification** (line 444): `Adobe Photoshop Image` means real content, not a pointer.

**Why D would make it permanent.** Deleting `.gitattributes` removes the tracking rules, so the next
commit stores raw binaries in Git.

</details>

---

## Q6

A push fails with "This repository is over its data quota". Which options address it?

- A. Buy data packs, restrict fetch with include and exclude, use a custom LFS server, or cache LFS
  objects in CI
- B. Delete the repository history
- C. Convert LFS files back to normal files
- D. Increase the runner disk size

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** lines **469–485**.

```bash
git config lfs.fetchinclude "assets/textures/*, assets/models/*"
git config lfs.fetchexclude "assets/cinematics/*"
git config lfs.url "https://lfs.contoso.internal/game-studio"
```

**Four options across two categories: pay more, or transfer less.** Data packs and a custom server change
the supply; fetch filters and CI caching change the demand.

**The CI cache at lines 480–485 is usually the biggest win**, because CI is what repeatedly downloads the
same objects:

```yaml
# - uses: actions/cache@v4
#   with:
#     path: .git/lfs
```

**Why C reintroduces the original problem** — 50 GB of binaries back in Git history.

</details>

---

## Q7

Where are LFS tracking rules stored?

- A. `.gitattributes`
- B. `.gitignore`
- C. `.lfsconfig`
- D. `.git/config`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** lines **98–107**.

```text
*.psd filter=lfs diff=lfs merge=lfs -text
```

```bash
# IMPORTANT: Commit the .gitattributes file
git add .gitattributes
git commit -m "chore: configure Git LFS tracking for binary assets"
```

**`git lfs track` is just an editor for that file** — and the challenge shouts about committing it for a
reason. **An uncommitted `.gitattributes` means every other clone has no rules**, so their commits store
raw binaries and the repository grows again.

**Read the attribute line's four parts:** `filter=lfs` swaps content for pointers, `diff=lfs` and
`merge=lfs` stop Git attempting textual operations, and **`-text` disables line-ending conversion** —
which would corrupt a binary.

</details>

---

## Q8

What does `--lockable` add to a tracking pattern?

- A. Matching files are checked out read-only until locked
- B. The files cannot be deleted
- C. The files are encrypted
- D. Only admins can edit them

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** lines **336–341** and **371–378**.

```text
*.psd filter=lfs diff=lfs merge=lfs -text lockable
```

```bash
ls -la assets/textures/hero_diffuse.psd
# -r--r--r-- (read-only until locked)
```

**The read-only bit is the whole mechanism.** It makes the workflow discoverable — an artist opening a
`.psd` in Photoshop is told the file is read-only, which prompts them to lock it before they have done
any work.

**Compare the alternative: a policy document nobody reads.** The permission change turns the convention
into something you trip over at exactly the right moment.

</details>

---

## Q9

Why does locking exist for binary files specifically?

- A. Binary files cannot be merged, so concurrent edits mean one person's work is lost
- B. Binaries are too large to merge quickly
- C. Git refuses to store two versions
- D. LFS servers do not support branching

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** line **332**.

```text
Prevent merge conflicts on binary files by implementing file locking:
```

**A text conflict is recoverable — Git shows you both sides and you reconcile them.** A binary conflict
is a choice between two whole files: one artist's four hours of work survives and the other's does not.

**Which is why locking is *pessimistic* concurrency control.** For text, Git's optimistic model works
because merging is possible. For a 200 MB `.psd` it is not, so you prevent the concurrent edit instead
of resolving it afterwards.

</details>

---

## Q10

What does `git lfs migrate info --everything` do?

- A. Reports which file types consume the most space across history, without changing anything
- B. Migrates all files to LFS
- C. Uploads files to the LFS server
- D. Deletes large files

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** lines **115–120**.

```bash
git lfs migrate info --everything
# Output shows file types sorted by total size in history
```

**`info` is read-only; `import` is the one that rewrites.** Running `info` first is how you decide which
extensions are worth migrating, rather than guessing.

**And `--above=1mb`** (line 130) is the alternative selector: migrate by **size** rather than by
extension, which catches large files whose type you did not think of.

</details>

---

## Q11

What does `git lfs ls-files` show?

- A. Files currently tracked by LFS, with their object IDs
- B. All files in the repository
- C. Locked files
- D. Files pending upload

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** lines **133–136**.

```bash
git lfs ls-files
# abc1234567 * assets/textures/hero_diffuse.psd
# def8901234 * models/character/protagonist.fbx
```

**It is the post-migration verification.** After `migrate import`, this is how you confirm the files are
now pointers rather than blobs.

**Why C is a different command** — `git lfs locks` (line 349) lists locks with owner and timestamp.

**And the second verification is at line 139**: `cat` the file and look for the three pointer lines.

</details>

---

## Q12

What do `git reflog expire` and `git gc --prune=now` accomplish after a migration?

- A. They remove the old large objects from the local repository so the size actually drops
- B. They upload objects to LFS
- C. They rebuild the index
- D. They verify the migration

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** lines **155–162**.

```bash
git reflog expire --expire-unreachable=now --all
git gc --prune=now

git count-objects -vH
# Before: size-pack: 50.2 GiB
# After:  size-pack: 1.8 GiB (only code + LFS pointers)
```

**Migration makes the old objects unreachable; it does not delete them.** The reflog still references
the pre-rewrite commits, so Git keeps every blob until both the reflog is expired and garbage collection
runs.

**Which is why a migration can appear to have done nothing.** The developer runs `migrate import`,
checks the folder size, sees 50 GB, and concludes it failed.

**50.2 GiB to 1.8 GiB is the scenario's problem solved** — and the remainder is code plus a few kilobytes
of pointers.

</details>

---

## Q13

What does `lfs: false` on `actions/checkout` do?

- A. It skips downloading LFS content during checkout
- B. It disables LFS for the repository
- C. It removes `.gitattributes`
- D. It fails the build if LFS files exist

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** lines **261–268**.

```yaml
      - uses: actions/checkout@v4
        with:
          lfs: false  # Don't fetch all LFS files

      - name: Fetch only needed LFS files
        run: |
          git lfs pull --include="src/**" --exclude="assets/cinematics/**"
```

**Skip everything, then pull only what the job needs.** On a 50 GB asset repository that is the
difference between a CI run that works and one that fills the runner's disk (line 19).

**And the include/exclude pair is the granularity that makes it practical** — source assets in, cinematic
video out.

</details>

---

## Q14

What is the difference between `git lfs fetch` and `git lfs pull`?

- A. `fetch` downloads objects into `.git/lfs`; `pull` also replaces the working-copy pointers with
  content
- B. They are identical
- C. `fetch` is for remotes, `pull` is for local
- D. `pull` uploads

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** lines **438–441**.

```bash
git lfs install
git lfs fetch --all
git lfs checkout
```

**That three-command sequence is the answer written out.** `pull` is `fetch` plus `checkout`, exactly as
`git pull` is `git fetch` plus `git merge`.

**Which is why the recovery uses the long form.** When LFS was never installed, you may want the objects
downloaded and then explicitly checked out — and separating the steps makes the failure point obvious if
one of them errors.

</details>

---

## Q15

Which storage limits does the challenge cite for GitHub LFS on the free tier?

- A. 1 GB storage and 1 GB bandwidth per month
- B. 5 GB storage, unlimited bandwidth
- C. 50 GB storage
- D. Unlimited for public repositories

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** lines **237–243**.

```text
# - Free: 1 GB storage, 1 GB bandwidth/month
# - Data packs: $5/month per 50 GB storage + 50 GB bandwidth
# Azure DevOps LFS limits:
# - Free tier: 1 GB per repository
```

**Both platforms start at 1 GB**, and the scenario needs 50 — so this is not a repository that can run on
free LFS regardless of platform.

**And the fact that bites hardest is that bandwidth is metered too.** Storage is paid once; **bandwidth is
paid every time CI clones**, which is why the optimisations at lines 261–274 matter more than they first
appear.

</details>

---

## Q16

What does the pre-push hook at Task 7 enforce?

- A. It blocks a push that modifies a lockable file the pusher does not hold a lock on
- B. It locks files automatically
- C. It warns and continues
- D. It rejects all binary changes

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** lines **388–403**.

```bash
    if [ -z "$LOCK_OWNER" ]; then
      echo "WARNING: $file was modified without a lock!"
      exit 1
    elif [ "$LOCK_OWNER" != "$CURRENT_USER" ]; then
      echo "ERROR: $file is locked by $LOCK_OWNER, not you!"
      exit 1
    fi
```

**Both branches `exit 1`, so despite the word "WARNING" it blocks.** No lock and someone else's lock are
both refused.

**And the honest limitation, which is the exam-relevant part: this is a *local* hook.** It lives in
`.git/hooks/` (line 385), is not committed, and is bypassed with `--no-verify`. **It is fast feedback,
not enforcement** — the same distinction as Challenge 43.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** are true of Git LFS? (Choose three.)

- A. It replaces tracked files with text pointers in the repository
- B. Tracking rules live in `.gitattributes` and must be committed
- C. It provides built-in file locking
- D. It reduces the number of commits in history
- E. It speeds up a repository that is large because of history depth
- F. It requires a custom storage backend

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-10.md`:** lines **139–143**, **98–107**, **323**.

**Why D and E are the Scalar confusion, and this is the paper's central trap.** LFS shrinks the
repository by moving **file content** out of it. A repository that is slow because it has 500,000 commits
or a million files is unaffected — that is Scalar, partial clone and sparse-checkout (Challenge 12).

**Why F is git-fat's model** (line 321), not LFS's — LFS uses the host's server by default.

</details>

---

## Q18

Which **three** does `.gitattributes` do for an LFS-tracked type? (Choose three.)

- A. `filter=lfs` swaps content for a pointer on commit and back on checkout
- B. `diff=lfs` and `merge=lfs` stop Git attempting textual diff or merge
- C. `-text` disables line-ending conversion
- D. It compresses the file
- E. It sets the file read-only
- F. It uploads the file

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-10.md`:** line **173**.

```text
*.fbx filter=lfs diff=lfs merge=lfs -text
```

**Four attributes, three jobs, and `-text` is the one people overlook.** Without it, Git may treat the
file as text and rewrite CRLF to LF on checkout — which **corrupts a binary silently**.

**Why E is the separate `lockable` attribute** (line 341), appended to the same line when you use
`git lfs track --lockable`.

</details>

---

## Q19

Which **three** reduce LFS bandwidth in CI? (Choose three.)

- A. `lfs: false` on checkout, then a targeted `git lfs pull`
- B. `GIT_LFS_SKIP_SMUDGE: 1` for jobs that need no assets
- C. Caching `.git/lfs` between runs
- D. Increasing the runner size
- E. Cloning with `--depth 1`
- F. Deleting `.gitattributes` in CI

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-10.md`:** lines **261–274** and **480–485**.

**Three techniques, three different savings.** A downloads a subset, B downloads nothing, C downloads
nothing **again**.

**Why E is the Challenge 12 confusion in miniature.** `--depth 1` limits how much **history** you fetch;
it does nothing about the size of the LFS objects for the commit you did check out.

**Why F would break the checkout** — without the rules, the pointer files are not recognised and never
resolved.

</details>

---

## Q20

Which **two** are true of LFS file locking? (Choose two.)

- A. `--lockable` makes files read-only until locked
- B. `git lfs unlock --force` requires maintain or admin permission
- C. Locks expire automatically after 24 hours
- D. Locking works for text files only
- E. Locks prevent cloning

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-10.md`:** lines **371–378** and **361–362**.

**Why C is the feature people assume exists**, and line 534 is careful about it: the team should
*consider* lock expiration **"if supported by their LFS server"**. It is not a guaranteed capability, and
assuming it is how a lock survives a two-week holiday.

**Why D inverts the purpose entirely.** Locking exists **because** binaries cannot be merged (Q9); text
files need no lock precisely because Git can merge them.

</details>

---

## Q21

Which **two** are advantages of Git LFS over git-fat, per the comparison table? (Choose two.)

- A. Native integration with GitHub and Azure DevOps
- B. Built-in file locking
- C. Any storage backend
- D. No bandwidth quotas
- E. Lower storage cost

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-10.md`:** lines **322–323** and **327**.

```text
| Hosting integration | Native GitHub/ADO support | Self-managed |
| File locking | Yes (built-in) | No |
| CI integration | Native (actions/checkout lfs) | Custom scripts needed |
```

**Three rows favour LFS: integration, locking and CI.** Three favour git-fat: backend choice, bandwidth
control and cost model.

**Which makes the choice a genuine trade rather than a right answer.** LFS if you want it to work with no
infrastructure; git-fat if 50 GB of bandwidth per month makes the provider's pricing unacceptable.

</details>

---

## Q22

Which **two** are required after `git lfs migrate import --everything`? (Choose two.)

- A. A force push, coordinated with the team
- B. Every other clone re-cloned or hard-reset
- C. Running `git lfs install` on the server
- D. Deleting `.gitattributes`
- E. Re-tagging every release

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-10.md`:** lines **145–149** and **512**.

```bash
git push origin main --force-with-lease

# Team members must re-clone or run:
git lfs pull
```

**`--force-with-lease` rather than `--force`** — the same safety property as Challenge 07 Q21: it refuses
if the remote moved since your last fetch.

**And E is the consequence worth thinking about even though it is not required.** Rewriting history
changes commit SHAs, so **existing tags now point at commits that are no longer on any branch**. The
migration does not break the tags; it orphans what they reference — which is why a migration is a
coordinated event rather than a Tuesday afternoon.

</details>

---

## Q23

Which **two** verify that a migration worked? (Choose two.)

- A. `git lfs ls-files` lists the files with object IDs
- B. `cat` on a tracked file shows the three-line pointer
- C. `git lfs locks` returns entries
- D. `git status` shows no changes
- E. `git log` shows fewer commits

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-10.md`:** lines **133–143**.

**Two views of the same fact: the index knows the file is LFS, and the object store holds a pointer.**

**And the third check is `git count-objects -vH`** (line 160) — 50.2 GiB down to 1.8 GiB, which is the
one that proves the *point* of the exercise rather than just its mechanics.

**Why E is false.** A migration rewrites commits; it does not remove them. The count is identical and
every SHA is different.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso's 50 GB game repository must stop timing out on push, clone in a usable time, and
stop filling CI runners — without leaving Git, and without artists losing work to concurrent binary
edits.

---

## Q24

**Proposed solution:** Install Git LFS, track the binary extensions and commit `.gitattributes`. Run
`git lfs migrate info` to identify the heaviest types, then `migrate import` for those, force-push with
lease and have the team re-clone. Expire the reflog and garbage-collect to reclaim space. Mark `.psd`,
`.fbx` and `.blend` as `--lockable` so they check out read-only, and use `git lfs lock` before editing.
In CI, check out with `lfs: false` and pull only the paths the job needs, caching `.git/lfs` between
runs.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-10.md`:** lines **42–107**, **115–149**, **155–162**, **336–345**, **261–274**,
**480–485**.

| Problem at line 19 | Mechanism |
|---|---|
| 50 GB repository | Migrate binaries to LFS; reflog expire + gc |
| Four-hour clones | Pointers instead of blobs |
| Push timeouts | Git no longer diffs and compresses binaries |
| CI runners out of disk | `lfs: false` + targeted pull + cache |
| Artists overwriting each other | `--lockable` + `git lfs lock` |

**The `gc` step is the one candidates leave out** (Q12), and without it the migration appears to have
achieved nothing.

</details>

---

## Q25

**Proposed solution:** Install Git LFS and run `git lfs track` for the binary types. Ask everyone to run
the same commands locally. Leave existing history alone, since rewriting is risky. Rely on Git's merge
resolution when two artists edit the same asset. Clone with `--depth 1` in CI to keep it fast.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures.**

**Tracking without committing `.gitattributes`** means each developer's rules are local. Anyone who did
not run the commands commits raw binaries, and the repository grows exactly as before (Q7).

**Leaving history alone leaves the 50 GB.** LFS applies to *new* commits; the existing blobs are still
in the pack, so clones still take four hours. Migration is the only way to shrink it (Q23).

**"Rely on Git's merge resolution" is not available for binaries** (Q9). Git will present a conflict it
cannot help you resolve, and one artist's work is discarded.

**And `--depth 1` addresses history depth, not object size** (Q19). The CI runner still downloads the LFS
content for the commit it checked out — which is what fills the disk.

</details>

---

## Q26

**Proposed solution:** Install Git LFS, track the binary types and commit `.gitattributes`. Migrate the
heaviest types with `migrate import`, force-push with lease, have the team re-clone, then expire the
reflog and garbage-collect. Mark binaries `--lockable` and lock before editing. Optimise CI with
`lfs: false`, targeted pulls and a cache. Because history was rewritten and old tags now point at
orphaned commits, delete the affected release tags and recreate them on the equivalent new commits.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right until the last sentence — and it is the one that turns a controlled migration into
a permanent loss of traceability.

**"The equivalent new commit" is a judgement, not a fact.** The rewrite changed every SHA; nothing
records which new commit corresponds to which old one once the reflog is expired and `gc` has run
(line 156–157). **You are re-tagging by inspection**, and any mistake is silent.

**And deleting a published release tag breaks everything that referenced it** — Challenge 09's rule at
its line 280. Deployment records, artifacts and incident reports all cite `v2.1.0`; recreating it
elsewhere quietly changes what those citations mean, and clones that already fetched keep the old target
because Git does not update an existing local tag on fetch.

**The correct handling is to preserve the tags as they are and accept that they reference pre-migration
history.** Cut new tags from the rewritten history going forward, and record in the release notes that
the migration happened on a given date. **The rewrite is a fact about the repository's past, not
something to paper over.**

**And this is why the challenge insists on coordination** (line 123). A migration is a one-way, scheduled
event precisely because its consequences reach beyond the repository.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — what LFS does

| # | Statement | Answer |
|---|---|---|
| 1 | LFS stores a text pointer in Git and the bytes on an LFS server |  |
| 2 | LFS shrinks a repository that is large because of deep history |  |
| 3 | `.gitattributes` must be committed for tracking to apply to clones |  |
| 4 | `-text` prevents line-ending conversion on binaries |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | LFS stores a text pointer in Git and the bytes on an LFS server | **Yes** |
| 2 | LFS shrinks a repository that is large because of deep history | **No** |
| 3 | `.gitattributes` must be committed for tracking to apply to clones | **Yes** |
| 4 | `-text` prevents line-ending conversion on binaries | **Yes** |

**In `challenge-10.md`:** lines **139–143**, **19**, **105–107**, **173**.

Row 2 is the Scalar-versus-LFS boundary and the highest-yield fact in the paper. **Large FILES → LFS.
Large HISTORY or breadth → Scalar and partial clone.**

</details>

---

## Q28 — migration

| # | Statement | Answer |
|---|---|---|
| 1 | `migrate info` reports without changing anything |  |
| 2 | `migrate import --everything` rewrites history |  |
| 3 | Repository size drops immediately after migration |  |
| 4 | Team members must re-clone or hard-reset |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `migrate info` reports without changing anything | **Yes** |
| 2 | `migrate import --everything` rewrites history | **Yes** |
| 3 | Repository size drops immediately after migration | **No** |
| 4 | Team members must re-clone or hard-reset | **Yes** |

**In `challenge-10.md`:** lines **116**, **122–124**, **155–162**, **512**.

Row 3 is Q12: the old objects remain until `reflog expire` and `gc --prune=now`.

</details>

---

## Q29 — locking

| # | Statement | Answer |
|---|---|---|
| 1 | `--lockable` makes files read-only until locked |  |
| 2 | Locking exists because binaries cannot be merged |  |
| 3 | `--force` unlock requires elevated permission |  |
| 4 | The pre-push hook cannot be bypassed |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `--lockable` makes files read-only until locked | **Yes** |
| 2 | Locking exists because binaries cannot be merged | **Yes** |
| 3 | `--force` unlock requires elevated permission | **Yes** |
| 4 | The pre-push hook cannot be bypassed | **No** |

**In `challenge-10.md`:** lines **371–378**, **332**, **361**, **385**.

Row 4: it is a **local** hook in `.git/hooks/`, uncommitted and skippable with `--no-verify` (Q16).

</details>

---

## Q30 — quotas and CI

| # | Statement | Answer |
|---|---|---|
| 1 | GitHub LFS free tier is 1 GB storage and 1 GB bandwidth monthly |  |
| 2 | LFS bandwidth is metered as well as storage |  |
| 3 | `lfs: false` disables LFS for the repository |  |
| 4 | Caching `.git/lfs` between CI runs reduces bandwidth |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | GitHub LFS free tier is 1 GB storage and 1 GB bandwidth monthly | **Yes** |
| 2 | LFS bandwidth is metered as well as storage | **Yes** |
| 3 | `lfs: false` disables LFS for the repository | **No** |
| 4 | Caching `.git/lfs` between CI runs reduces bandwidth | **Yes** |

**In `challenge-10.md`:** lines **238**, **239**, **263**, **480–485**.

Row 3: it skips the download **for that checkout**; the repository configuration is untouched.

Row 2 is the one that surprises teams — storage is paid once, bandwidth every time CI clones.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each repository problem to the right tool.

| Problem | Tool |
|---|---|
| 200 MB `.psd` files bloating the pack |  |
| A repository slow because of 500,000 commits |  |
| A working tree with a million files you do not need |  |
| Binaries must live in S3 you control |  |
| Two artists overwriting the same asset |  |
| CI runners filling their disk with assets |  |

**Options:** Git LFS · git-fat · `lfs: false` + targeted pull + cache · LFS file locking · **Scalar / partial clone** (Challenge 12) · **Sparse-checkout** (Challenge 12)

<details>
<summary>Show answer</summary>

| Problem | Tool |
|---|---|
| 200 MB `.psd` files bloating the pack | **Git LFS** |
| A repository slow because of 500,000 commits | **Scalar / partial clone** (Challenge 12) |
| A working tree with a million files you do not need | **Sparse-checkout** (Challenge 12) |
| Binaries must live in S3 you control | **git-fat** |
| Two artists overwriting the same asset | **LFS file locking** |
| CI runners filling their disk with assets | **`lfs: false` + targeted pull + cache** |

**In `challenge-10.md`:** lines **19**, **321**, **332**, **261–274**.

**Rows 1, 2 and 3 are the distinction the exam builds this whole topic on.** LFS moves **file content**
out; Scalar and sparse-checkout limit **how much of the repository you materialise**. They stack; they do
not substitute.

</details>

---

## Q32

Match each command to what it does.

| Command | Does |
|---|---|
| `git lfs track "*.psd"` |  |
| `git lfs migrate info` |  |
| `git lfs migrate import` |  |
| `git lfs ls-files` |  |
| `git lfs pull` |  |
| `git lfs prune` |  |

**Options:** Adds a rule to `.gitattributes` · Fetch plus checkout of LFS content · Lists LFS-tracked files with object IDs · Removes local objects no branch references · Reports history usage, changes nothing · Rewrites history, converting files to pointers

<details>
<summary>Show answer</summary>

| Command | Does |
|---|---|
| `git lfs track "*.psd"` | **Adds a rule to `.gitattributes`** |
| `git lfs migrate info` | **Reports history usage, changes nothing** |
| `git lfs migrate import` | **Rewrites history, converting files to pointers** |
| `git lfs ls-files` | **Lists LFS-tracked files with object IDs** |
| `git lfs pull` | **Fetch plus checkout of LFS content** |
| `git lfs prune` | **Removes local objects no branch references** |

**In `challenge-10.md`:** lines **59**, **116**, **124**, **133**, **432**, **552**.

**`prune` is local housekeeping only** — it clears `.git/lfs` of objects your branches no longer need,
and it does not touch the server.

</details>

---

## Q33

Arrange the steps to migrate a 50 GB repository to LFS.

**Items:** Force-push with lease and have the team re-clone · Expire the reflog and garbage-collect ·
Commit `.gitattributes` · Run `git lfs migrate info` to find the heaviest types · Run `git lfs migrate
import` for those types · Install and initialise Git LFS

<details>
<summary>Show answer</summary>

### Answer

1. Install and initialise Git LFS — lines **42–46**
2. Run `git lfs migrate info` to find the heaviest types — lines **115–120**
3. Run `git lfs migrate import` for those types — line **124**
4. Commit `.gitattributes` — lines **105–107**
5. Force-push with lease and have the team re-clone — lines **145–149**
6. Expire the reflog and garbage-collect — lines **155–160**

**Step 2 before step 3 is the difference between a decision and a guess.** `migrate info` tells you which
extensions actually consume the 50 GB; migrating everything binary-looking rewrites more history than
necessary.

**And step 6 last is the one people omit** (Q12). Until the reflog is expired and `gc --prune=now` has
run, the old blobs are still on disk and `git count-objects` still reports 50 GB — which reads as a
failed migration.

**Note `migrate import` writes `.gitattributes` itself**, so step 4 is confirming and committing it
rather than authoring it.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| `.psd` opens as three lines of text |  |
| Repository still 50 GB after migration |  |
| A colleague's commits store raw binaries |  |
| Push rejected: over data quota |  |
| An artist's four hours of work vanished |  |
| A binary is corrupted after checkout on Windows |  |

**Options:** Concurrent binary edit with no lock · `.gitattributes` never committed · LFS storage or bandwidth limit reached · Missing `-text`, so CRLF conversion ran · Reflog not expired and `gc` not run · Smudge skipped, or LFS not installed at clone

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| `.psd` opens as three lines of text | **Smudge skipped, or LFS not installed at clone** |
| Repository still 50 GB after migration | **Reflog not expired and `gc` not run** |
| A colleague's commits store raw binaries | **`.gitattributes` never committed** |
| Push rejected: over data quota | **LFS storage or bandwidth limit reached** |
| An artist's four hours of work vanished | **Concurrent binary edit with no lock** |
| A binary is corrupted after checkout on Windows | **Missing `-text`, so CRLF conversion ran** |

**In `challenge-10.md`:** lines **420**, **155–162**, **105**, **454**, **332**, **173**.

**The last row is the quietest failure in the challenge.** Nothing errors; the file simply no longer
opens in the tool that made it, and the cause is four characters missing from a `.gitattributes` line.

</details>

---

## Q35

Match each LFS-versus-git-fat row to the winner.

| Criterion | Winner |
|---|---|
| Native GitHub and Azure DevOps integration |  |
| File locking |  |
| Native CI integration |  |
| Choice of storage backend |  |
| Bandwidth cost control |  |
| Setup and maintenance effort |  |

**Options:** Git LFS · **Git LFS** (lower) · git-fat

<details>
<summary>Show answer</summary>

| Criterion | Winner |
|---|---|
| Native GitHub and Azure DevOps integration | **Git LFS** |
| File locking | **Git LFS** |
| Native CI integration | **Git LFS** |
| Choice of storage backend | **git-fat** |
| Bandwidth cost control | **git-fat** |
| Setup and maintenance effort | **Git LFS** (lower) |

**In `challenge-10.md`:** lines **319–328**.

**Four to two in favour of LFS on convenience; git-fat wins where you need to own the infrastructure.**
The decision is usually made by the bandwidth bill (Q15) rather than by features.

</details>

---

# Section F — Hot area

---

## Q36

```text
*.fbx filter=[BLANK 1] diff=lfs merge=lfs [BLANK 2] [BLANK 3]
```

Requirement: LFS-tracked, no line-ending conversion, and read-only until locked.

- **BLANK 1:** `lfs` / `fat` / `binary` / `large`
- **BLANK 2:** `-text` / `text` / `binary` / `nodiff`
- **BLANK 3:** `lockable` / `locked` / `readonly` / `exclusive`

<details>
<summary>Show answer</summary>

### Answer: `lfs`, `-text`, `lockable`

**In `challenge-10.md`:** lines **173** and **341**.

**`-text` with the leading minus *disables* the text attribute.** Writing `text` would do the opposite —
mark it as text and enable exactly the CRLF conversion that corrupts a binary (Q34).

**And `filter=fat`** (line 304) is the git-fat equivalent of `filter=lfs`, which is how the two tools
coexist in the same `.gitattributes` if you migrate between them.

</details>

---

## Q37

```bash
git lfs migrate [BLANK 1] --include="*.psd,*.fbx,*.png" --everything
git lfs migrate [BLANK 2] --include="*.psd,*.fbx,*.png,*.wav" --everything
git lfs migrate import --[BLANK 3]=1mb --everything
```

- **BLANK 1:** `info` / `import` / `export` / `status`
- **BLANK 2:** `import` / `info` / `push` / `track`
- **BLANK 3:** `above` / `over` / `min-size` / `larger-than`

<details>
<summary>Show answer</summary>

### Answer: `info`, `import`, `above`

**In `challenge-10.md`:** lines **120**, **124**, **130**.

**`info` first, always.** It is the read-only survey that tells you whether `.psd` or `.wav` is actually
the problem, and it costs nothing.

**And `--above=1mb` is the selector for files whose extension you did not anticipate** — a
`assets/raw/dump.bin` nobody listed still gets migrated if it is large enough.

</details>

---

## Q38

```bash
git lfs track --[BLANK 1] "*.psd"
git lfs [BLANK 2] assets/textures/hero_diffuse.psd
git lfs [BLANK 3]
git lfs unlock assets/textures/hero_diffuse.psd --[BLANK 4]
```

Requirement: make the type lockable, take a lock, list all locks, then break someone else's lock.

- **BLANK 1:** `lockable` / `lock` / `exclusive` / `binary`
- **BLANK 2:** `lock` / `claim` / `reserve` / `hold`
- **BLANK 3:** `locks` / `lock --list` / `status` / `ls-locks`
- **BLANK 4:** `force` / `admin` / `override` / `steal`

<details>
<summary>Show answer</summary>

### Answer: `lockable`, `lock`, `locks`, `force`

**In `challenge-10.md`:** lines **336**, **345**, **349**, **362**.

**`git lfs locks` (plural) lists; `git lfs lock` (singular) takes one.** The plural form outputs ID, path,
owner and timestamp (lines 351–353) — the ID is what `--id=1234` unlocks (line 365).

**And `--force` needs maintain or admin** (Q4), which is what makes breaking a lock a governed action.

</details>

---

## Q39

```yaml
      - uses: actions/checkout@v4
        with:
          lfs: [BLANK 1]

      - run: git lfs pull --[BLANK 2]="src/**" --[BLANK 3]="assets/cinematics/**"
```

Requirement: skip the bulk download, then fetch only source assets.

- **BLANK 1:** `false` / `true`
- **BLANK 2:** `include` / `only` / `path` / `filter`
- **BLANK 3:** `exclude` / `ignore` / `skip` / `not`

<details>
<summary>Show answer</summary>

### Answer: `false`, `include`, `exclude`

**In `challenge-10.md`:** lines **263** and **268**.

**Skip everything, then pull a subset** — and the include/exclude pair also exists as persistent
configuration (lines 473–474):

```bash
git config lfs.fetchinclude "assets/textures/*, assets/models/*"
git config lfs.fetchexclude "assets/cinematics/*"
```

**The `config` form applies to every fetch in that clone**, which suits a developer machine; the
command-line form suits a CI job that needs different paths per workflow.

</details>

---

## Q40

```bash
# After migration, reclaim the space
git reflog expire --expire-unreachable=[BLANK 1] --all
git gc --[BLANK 2]

git count-objects -[BLANK 3]
```

- **BLANK 1:** `now` / `never` / `30.days` / `all`
- **BLANK 2:** `prune=now` / `aggressive` / `auto` / `force`
- **BLANK 3:** `vH` / `a` / `s` / `l`

<details>
<summary>Show answer</summary>

### Answer: `now`, `prune=now`, `vH`

**In `challenge-10.md`:** lines **156–160**.

**Both `now` values are required, and each removes a different protection.** The reflog expiry drops the
references keeping old commits reachable; `--prune=now` overrides `gc`'s default grace period, which
otherwise keeps recent unreachable objects for safety.

**`-vH` is verbose plus human-readable sizes** — the flag pair that produces `size-pack: 50.2 GiB` rather
than a byte count.

</details>

---

## Q41

```bash
cat > .gitfat << 'EOF'
[[BLANK 1]]
bucket = contoso-game-assets
region = us-east-1
prefix = git-fat/
EOF

echo "*.psd filter=[BLANK 2] -text" >> .gitattributes
git fat [BLANK 3]
```

- **BLANK 1:** `s3` / `rsync` / `aws` / `storage`
- **BLANK 2:** `fat` / `lfs` / `s3` / `binary`
- **BLANK 3:** `push` / `upload` / `sync` / `commit`

<details>
<summary>Show answer</summary>

### Answer: `s3`, `fat`, `push`

**In `challenge-10.md`:** lines **296–308**.

**`[rsync]` at line 290 is the other supported backend** — the same tool, a different section header, and
that flexibility is git-fat's entire selling point (Q3).

**And `filter=fat` sits in the same `.gitattributes` as LFS's `filter=lfs`**, so a repository can in
principle use both for different types — though in practice you pick one.

</details>

---

# Section G — Case study

## Case study: Contoso game studio assets

### Background

Contoso Ltd's game studio stores all binary assets in Git. The repository is **50 GB**, `.psd` files
average **200 MB** and `.fbx` models reach **500 MB**. Clones take **over four hours** on the office
network. Developers hit **push timeouts** because Git diffs and compresses binaries inefficiently. **CI
builds fail when runners run out of disk space.** The team must handle large binaries **without
abandoning Git**.

### Requirements

**Repository**

- New binary commits must not add to the pack
- The existing 50 GB of history must actually shrink, not merely stop growing
- Every developer must get the same tracking behaviour without configuring anything themselves

**Workflow**

- Two artists must not be able to overwrite each other's work on the same asset
- Breaking a colleague's lock must be possible in an emergency and must require permission

**CI**

- Build jobs that need no assets must not download them
- Jobs that need some assets must download only those
- Repeated runs must not re-download the same objects

---

## Q42

How is "new binary commits must not add to the pack" achieved for **every** developer?

- A. `git lfs track` for each type, with `.gitattributes` committed to the repository
- B. Each developer runs `git lfs track` locally
- C. A `.gitignore` entry for binaries
- D. A pre-commit hook on each machine

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** lines **59–107**.

**"Without configuring anything themselves" is the clause that selects A.** `.gitattributes` travels with
the clone, so the rules apply to anyone who checks out the repository.

**Why B is what the challenge warns about at line 105** in capitals. Local-only rules mean the developer
who joined last week commits a 200 MB `.psd` as a normal blob, and nobody notices until the pack grows.

**Why C removes the file from version control entirely** — the assets must be versioned, just not stored
inline.

</details>

---

## Q43

How does the existing 50 GB actually shrink?

- A. `git lfs migrate import` for the heaviest types, force-push with lease, team re-clones, then reflog
  expire and `gc --prune=now`
- B. `git lfs track` going forward
- C. `git gc --aggressive`
- D. Deleting old branches

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** lines **124–162**.

**Three phases, and all three are required.** Migration rewrites the history, the force push and re-clone
propagate it, and the reflog expiry plus prune reclaim the disk.

**Why B is the difference between "stop growing" and "shrink"** — the requirement is explicit about
wanting both.

**Why C alone does nothing.** The old blobs are still **reachable** through the reflog and, before the
rewrite, through the commits themselves — `gc` will not delete what is referenced.

</details>

---

## Q44

Which **two** protect artists from overwriting each other, and what makes it discoverable? (Choose two.)

- A. `git lfs track --lockable`, so files check out read-only
- B. `git lfs lock` before editing, with `git lfs locks` showing the owner
- C. Branch protection on `main`
- D. A PR template reminder
- E. `.gitignore` for assets

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-10.md`:** lines **336–353** and **371–378**.

**A is what makes it discoverable.** The read-only bit surfaces at the moment the artist opens the file
— before any work has been done — rather than at push time when four hours are already invested.

**B is the workflow itself**, and `git lfs locks` gives the answer to "who has this?" without asking in
chat.

**Why C guards the wrong thing.** Branch protection stops unreviewed code reaching `main`; it cannot stop
two people editing the same binary in parallel, because both edits are legitimate changes to the branch.

</details>

---

## Q45

How should the CI requirements be met?

- A. `lfs: false` on checkout, `git lfs pull` with include and exclude for jobs that need assets,
  `GIT_LFS_SKIP_SMUDGE` for jobs that do not, and a cache of `.git/lfs`
- B. Increase the runner disk size
- C. Clone with `--depth 1`
- D. Disable LFS in CI

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** lines **261–274** and **480–485**.

**Three requirements, three mechanisms in one answer.** Skip by default, pull selectively, cache between
runs.

**Why B treats the symptom at a cost that scales with the asset library** — and the scenario's failure is
disk exhaustion (line 19), which a larger disk defers rather than fixes.

**Why C is the recurring confusion** (Q19): depth limits **history**, not the size of the objects for the
commit you checked out.

</details>

---

## Q46

The team is considering git-fat instead. Which requirement would they lose?

- A. File locking
- B. Versioned binaries
- C. Pointer files in Git
- D. The ability to use S3

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** line **323**.

```text
| File locking | Yes (built-in) | No |
```

**And it is the requirement the scenario cares about most**, because two artists overwriting each other
is a named workflow requirement, not a nice-to-have.

**Why B and C are false of git-fat.** It also stores pointers and versions the files; the difference is
**where the bytes live** (Q3).

**Which makes this the deciding trade for a game studio specifically.** git-fat's storage flexibility is
attractive at 50 GB — and losing locking would cost them the thing LFS was chosen to provide.

</details>

---

## Q47

Six months after the migration, the repository has grown back to 22 GB. `.gitattributes` is present and
correct, LFS is working for most of the team, and `git lfs ls-files` shows thousands of tracked files. A
new studio was acquired and its artists joined three months ago.

What happened, and what is the fix?

- A. The new artists cloned before LFS was installed on their machines, so the smudge and clean filters
  never ran and their commits stored raw binaries — install LFS, migrate the affected commits, and add a
  server-side check
- B. `.gitattributes` was deleted
- C. LFS quota was exceeded
- D. `gc` was never run

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** lines **42–43** and **105**.

```bash
git lfs install
# Output: Updated git hooks. Git LFS initialized.
```

**Read what `git lfs install` actually does: it updates *git hooks*, per user.** `.gitattributes` declares
which files should be filtered; **the filter itself only exists if LFS is installed on that machine.**

**So a developer without LFS sees the rules, cannot act on them, and commits the raw file.** Git accepts
it — there is no error, because from Git's perspective a `filter` that is not configured is simply not
applied.

**"Working for most of the team" is the detail that identifies it.** A missing `.gitattributes` would
affect everyone; a quota problem would block pushes rather than accept them.

**The fix has three parts.** Install LFS on the affected machines, migrate the commits they contributed
(`migrate import --include-ref` scoped to the affected range), and — the durable part — **add a
server-side or CI check** that fails when a commit adds a large blob that should have been a pointer.
`.gitattributes` is a *declaration*; only a check on the server is *enforcement*.

</details>

---

## Q48

A year on, clones take four minutes, pushes never time out, CI runs on a small runner, and no artist has
lost work to an overwrite.

Explain what each piece contributed, and what actually changed.

- A. LFS moved file **content** out of the pack while keeping the files versioned; migration plus reflog
  expiry shrank the existing 50 GB; committed `.gitattributes` made the behaviour universal rather than
  per-machine; lockable tracking turned a merge problem Git cannot solve into a coordination problem it
  does not need to; and CI-side skipping, filtering and caching stopped the same objects being downloaded
  repeatedly
- B. The team started committing fewer assets
- C. The office network was upgraded
- D. Larger CI runners were purchased

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-10.md`:** lines **19**, **139–162**, **105–107**, **332**, **261–274**.

**Take the four failures at line 19 in turn.**

*Four-hour clones and a 50 GB pack* — the repository now contains code plus a few kilobytes of pointers
(line 162). The bytes still exist; they are fetched **on demand and only when needed**.

*Push timeouts* — Git was trying to diff and delta-compress 500 MB binaries against each other, which is
work with no possible payoff. LFS uploads them as opaque objects instead.

*CI disk exhaustion* — the build job never downloads what it does not use, and a cache means it rarely
downloads anything twice.

*Artists overwriting each other* — this one is not a storage problem at all. **Git's optimistic
concurrency model assumes merging is possible**, and for a `.psd` it is not. Locking replaces the
assumption.

**What actually changed is not discipline or hardware.** The studio was using a tool designed for **text
that can be merged and delta-compressed** to store **binaries that can be neither**.

**The graded idea: choose the mechanism by the shape of the data.** Large binary files → LFS, with
locking. Large history or breadth → Scalar and sparse-checkout. **And whichever you choose, the
configuration must be committed and enforced server-side** — because a rule that lives on each
developer's machine is a rule that a new starter will not have.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **LFS offered for a deep-history problem** | Q17, Q27, Q31 | LFS = large FILES. Scalar/partial clone = large HISTORY |
| **`--depth 1` offered to reduce LFS traffic** | Q19, Q45 | Depth limits history, not object size |
| **`.gitattributes` not committed** | Q7, Q25, Q34, Q42 | Local rules only. New clones commit raw binaries |
| **LFS not installed on a machine** | Q5, Q47 | The filter is per-user. Git accepts the raw blob silently |
| **`text` instead of `-text`** | Q18, Q34, Q36 | Enables CRLF conversion. Corrupts binaries |
| **Migration assumed to shrink the repo immediately** | Q12, Q28, Q43 | Needs reflog expire + `gc --prune=now` |
| **`git pull` after a migration** | Q2 | History was rewritten. Re-clone |
| **Old tags recreated after a rewrite** | Q26 | "Equivalent commit" is a guess. Preserve them |
| **Git merge expected to resolve a binary conflict** | Q9, Q25 | It cannot. One side is discarded |
| **Lock expiry assumed** | Q20 | "If supported by their LFS server". Not guaranteed |
| **Pre-push hook treated as enforcement** | Q16, Q29 | Local, uncommitted, `--no-verify` |
| **Bandwidth forgotten in quota planning** | Q15, Q30 | Storage once; bandwidth every clone |
| **git-fat chosen where locking is required** | Q3, Q21, Q46 | git-fat has no file locking |
| **`lfs: false` read as disabling LFS** | Q13, Q30 | It skips that checkout only |

---

# What to memorise

**In `challenge-10.md`:** lines **98–107**, **115–162**, **319–328**, **330–378**.

```text
LFS SOLVES LARGE FILES. IT DOES NOT SOLVE LARGE HISTORY.
  large binary FILES        -> Git LFS (pointer in Git, bytes on the LFS server)
  deep HISTORY / breadth    -> Scalar, partial clone, sparse-checkout   (Challenge 12)
  they STACK. they are not alternatives.

THE POINTER - what Git actually stores
  version https://git-lfs.github.com/spec/v1
  oid sha256:4d7a214614...
  size 214958080

.gitattributes  IS the configuration, and it MUST BE COMMITTED
  *.fbx filter=lfs diff=lfs merge=lfs -text [lockable]
        ^swap content   ^no textual diff/merge   ^NO CRLF CONVERSION (corrupts binaries)
  git lfs install   is PER USER and installs the git hooks. without it the filter never runs
                    -> that machine commits RAW BINARIES, silently
```

```bash
# MIGRATION - rewrites history                       (lines 115-162)
git lfs migrate info --everything                    # READ-ONLY survey. do this first
git lfs migrate import --include="*.psd,*.fbx" --everything     # REWRITES every touched commit
git lfs migrate import --include-ref=refs/heads/main            # smaller blast radius
git lfs migrate import --above=1mb --everything                 # by SIZE, not extension
git push origin main --force-with-lease              # team must RE-CLONE
git reflog expire --expire-unreachable=now --all     # <- without these two the size does
git gc --prune=now                                   #    NOT drop. 50.2 GiB -> 1.8 GiB
git count-objects -vH                                # verbose + human-readable

# VERIFY
git lfs ls-files          # tracked files + object IDs
cat file.psd              # three pointer lines
file file.psd             # "Adobe Photoshop Image" = real content
```

```bash
# LOCKING - because binaries CANNOT be merged        (lines 330-378)
git lfs track --lockable "*.psd"     # -> checked out READ-ONLY until locked
git lfs lock <path>                  # -rw-r--r-- , you may now edit
git lfs locks                        # ID | path | owner | locked at
git lfs unlock <path>                # release
git lfs unlock <path> --force        # break someone else's - needs MAINTAIN/ADMIN
git lfs unlock --id=1234
# pre-push hook is LOCAL, uncommitted, and skippable with --no-verify

# CI BANDWIDTH                                       (lines 261-274, 469-485)
- uses: actions/checkout@v4
  with: {lfs: false}                 # skip the bulk download (does NOT disable LFS)
- run: git lfs pull --include="src/**" --exclude="assets/cinematics/**"
  env: {GIT_LFS_SKIP_SMUDGE: 1}      # jobs that need NO assets
git config lfs.fetchinclude / lfs.fetchexclude       # persistent, per clone
git config lfs.url "https://lfs.contoso.internal/..."  # custom server, no provider quota
actions/cache@v4 with path: .git/lfs                 # stop re-downloading the same objects
# quotas: GitHub free = 1 GB storage AND 1 GB bandwidth/month. data packs $5 / 50 GB
#         Azure DevOps free = 1 GB per repository.   BANDWIDTH IS METERED TOO.
```

```text
LFS vs git-fat                                       (lines 319-328)
                        Git LFS                        git-fat
  backend               provider's LFS server          S3 / rsync / anything you run
  hosting integration   NATIVE GitHub + ADO            self-managed
  FILE LOCKING          YES, built in                  NO
  bandwidth             provider quotas                your S3 bill
  setup                 low                            medium
  CI                    native (checkout lfs)          custom scripts
  choose git-fat when you must own the storage. choose LFS when you need LOCKING or low effort.
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Solid. Move to Challenge 11 |
| 38–43 | Re-read the trap index and the LFS-vs-Scalar boundary, then move on |
| 30–37 | Write the pointer format, the attribute line and the migration sequence from memory, then retake |
| Below 30 | Redo Tasks 2, 3 and 7 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 10.

:::danger The one boundary

**Large FILES → Git LFS. Large HISTORY or breadth → Scalar and sparse-checkout.** They are not
alternatives, and the exam builds its best distractors from that confusion.

Two details that decide questions: **`.gitattributes` must be committed** and **`git lfs install` is
per machine** — miss either and someone commits a 200 MB blob with no error at all.

And **binaries cannot be merged**, which is why locking exists.

:::
