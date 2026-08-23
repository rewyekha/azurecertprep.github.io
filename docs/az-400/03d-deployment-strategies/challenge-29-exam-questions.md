---
sidebar_position: 5.5
toc_max_heading_level: 2
title: "Challenge 29: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 29 — AZ-400 exam questions

**48 questions** built only from what Challenge 29 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-29.md`**.

| Section | Shape | Questions |
|---|---|---|
| A | Single answer | 1–16 |
| B | Multiple answer (choose two / three) | 17–23 |
| C | Repeated scenario — "Does this meet the goal?" | 24–26 |
| D | Yes/No statement grid | 27–30 |
| E | Drag and drop | 31–35 |
| F | Hot area — complete the configuration | 36–41 |
| G | Case study | 42–48 |

The **trap index**, the **blocks to memorise**, and **scoring** are at the end.

:::danger The one thing databases change about everything else

Slots, revisions and flags all roll back because the previous version still exists. **A dropped column
does not.** Every answer in this challenge follows from that: migrations must be **additive and
backward-compatible**, applied **before** the code that needs them, and reversed by **rolling
forward** rather than back.

:::

---

# Section A — Single answer

---

## Q1

Which EF Core command produces a script suitable for a CI/CD pipeline?

- A. `dotnet ef database update`
- B. `dotnet ef migrations script --idempotent`
- C. `dotnet ef migrations add`
- D. `dotnet ef dbcontext scaffold`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-29.md`:** lines **37–42**, and again at **86–92**.

```bash
dotnet ef migrations script \
  --project src/ContosoApi \
  --startup-project src/ContosoApi \
  --idempotent \
  --output migrations.sql
```

**`--idempotent` is the important flag.** It wraps every migration in a check against the
`__EFMigrationsHistory` table, so re-running the script is safe. That matters when a pipeline is
retried, or when the same script runs against environments at different versions.

**Why the others fail**

- **A** — `database update` applies migrations **directly from the build agent**, which means the
  agent needs a live connection and DDL rights, and nothing is reviewable beforehand. A generated
  script can be inspected, approved and archived
- **C** — creates a migration during development (line 32). It does not apply anything
- **D** — reverse-engineers a model from an existing database

**The reviewable-artifact argument is the exam answer:** generate SQL in build, apply it in a gated
deployment stage.

</details>

---

## Q2

An application throws `Invalid object name 'CustomerPreferences'` after deployment.

What is the cause?

- A. The migration stage and the application stage ran in parallel
- B. The connection string is wrong
- C. The application was built against the wrong framework version
- D. The table was created in the wrong schema

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-29.md`:** Break & fix Exercise 2, lines **705–724**.

> *"The pipeline stages did not have proper `dependsOn` configuration, allowing them to run in
> parallel."*

```yaml
stages:
  - stage: DeployDatabase
    dependsOn: Build
  - stage: DeployApplication
    dependsOn: DeployDatabase          # must wait for the migration
    condition: succeeded('DeployDatabase')
```

**This is the incident from the scenario** (line 15): a developer deployed an API expecting a table
that did not exist, and it took 45 minutes and a DBA to recover.

**Why the others fail**

- **B** — a bad connection string gives a login or network error, not a missing-object error. The app
  clearly connected
- **C** — a framework mismatch fails at startup, not on a specific table
- **D** — possible in principle, but the documented cause is ordering. And `Invalid object name` names
  the object, which is the ordering signature

**Note this is Challenge 22's `dependsOn` lesson with data at stake.** Missing ordering there made a
pipeline slow; here it makes production fail.

</details>

---

## Q3

In what order must a database change and its application code be deployed?

- A. Application first, then database
- B. Database first (additive), then application
- C. Simultaneously
- D. Order does not matter with idempotent scripts

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-29.md`:** lines **269–276**.

```text
1. Run database migrations (additive/non-breaking)
2. Verify migrations applied successfully
3. Deploy new application version to staging slot
4. Validate staging with new schema
5. Swap staging to production
6. (Optional) Remove deprecated columns/tables in next release cycle
```

**Why schema first works — and only because it is additive.** Adding a nullable column or a new table
is invisible to the currently running code. The old version keeps working; the new version finds what
it needs when it arrives.

If the migration were **destructive**, schema-first would break the running application immediately.
That is why step 1 says "additive/non-breaking" and step 6 defers removal to a later release.

**Why the others fail**

- **A** — the new code arrives before the objects it needs. That is the incident
- **C** — a race with no winner
- **D** — idempotency makes a script **safe to re-run**. It does not make objects exist earlier

</details>

---

## Q4

Which rollback strategy does the challenge prefer for a bad migration in production?

- A. Forward-fix with a compensating migration
- B. Point-in-time restore
- C. Restoring from a pre-migration backup copy
- D. Manually reversing the SQL

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-29.md`:** lines **374–381**.

```bash
# If V003 introduced a bug, create V004 to fix it (preferred in production)
dotnet ef migrations add FixOrdersIndexes
```

**Why forward-fix wins:** it keeps the migration history linear and truthful, loses no data written
since the bad migration, and needs no downtime. Every other option throws away the transactions that
happened in between.

**Why the others fail**

- **B** — a restore reverts the database to a moment in time, discarding **every write since then**.
  Reserved for catastrophic corruption (line 383 calls it exactly that)
- **C** — same data-loss problem, plus the backup is only as fresh as the copy
- **D** — manual SQL in production is unreviewed, untested and unrecorded in migration history

**The general principle:** *databases roll forward.* Rolling back is a recovery action, not a
deployment step.

</details>

---

## Q5

What does the expand-contract pattern's **expand** phase do?

- A. Adds new schema while keeping the old, without breaking changes
- B. Removes deprecated columns
- C. Increases database storage
- D. Copies data to a new database

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-29.md`:** lines **437–448**.

```sql
-- V005__Expand_add_full_name_column.sql
ALTER TABLE dbo.Customers ADD FullName NVARCHAR(512) NULL;

UPDATE dbo.Customers
SET FullName = FirstName + ' ' + LastName
WHERE FullName IS NULL;
```

**Two details make it non-breaking:** the column is `NULL`-able, so existing inserts that omit it
still succeed; and the backfill populates it for existing rows. The old code never notices.

**Why the others fail**

- **B** — that is **contract**, phase 3 (line 468), and it belongs to a later release
- **C** and **D** — unrelated to the pattern

**The name is the pattern:** expand the schema so old and new can coexist, migrate the application,
then contract once nothing uses the old shape.

</details>

---

## Q6

In the expand-contract timeline, when are the old columns dropped?

- A. In the same release that adds the new column
- B. In the release after the application stops writing to them
- C. Immediately after the backfill completes
- D. Never

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-29.md`:** lines **482–486**.

| Release | Database change | Application change |
|---|---|---|
| v2.1 | **Expand**: add `FullName` | Write to both old and new |
| v2.2 | None | Read new only, stop writing old |
| v2.3 | **Contract**: drop old columns | Remove old references |

**Three releases, and the middle one is the point.** v2.2 changes no schema at all — it exists so that
every running instance stops using the old columns before v2.3 removes them.

Line 472 states the gate: *"Only run after ALL application instances use FullName."*

**Why the others fail**

- **A** — during v2.1 the old code is still running and still reading those columns
- **C** — the backfill copies data; it does not migrate the code
- **D** — leaving them forever is real schema debt, and the pattern has a defined end

**Why the middle release matters for rollback:** if v2.2 has to be rolled back, the old columns are
still there. Skip it and you have no safe way back.

</details>

---

## Q7

A DACPAC publish fails with `Rows were detected. The schema update is terminating because data loss
might occur.`

What is the recommended fix?

- A. Set `/p:BlockOnPossibleDataLoss=false`
- B. Use the expand-contract pattern instead of dropping columns directly
- C. Delete the affected rows first
- D. Switch to `deployType: SqlTask`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-29.md`:** Break & fix Exercise 3, lines **729–746**.

> *"Fix: Use the expand-contract pattern instead of dropping columns directly."*

`/p:BlockOnPossibleDataLoss=true` (line 649) is a **safety net working correctly** — it noticed you
were about to drop a column containing data.

**Why the others fail**

- **A** — line 741 lists it as "use with caution" for a reason: it silently deletes production data.
  Disabling a guard is almost never the exam answer
- **C** — deleting the rows *is* the data loss, just done manually
- **D** — a raw SQL script would drop the column with no warning at all. That is worse, not better

**The habit worth carrying:** when a safety mechanism blocks you, ask whether it is right before you
turn it off. It usually is.

</details>

---

## Q8

A migration fails with `The server principal is not able to access the database under the current
security context.`

What is missing?

- A. The pipeline identity has no database user or DDL role
- B. The SQL firewall blocks the runner
- C. The connection string uses the wrong port
- D. The database is paused

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-29.md`:** Break & fix Exercise 1, lines **675–700**.

```sql
CREATE USER [contoso-pipeline-sp] FROM EXTERNAL PROVIDER;
ALTER ROLE db_ddladmin  ADD MEMBER [contoso-pipeline-sp];
ALTER ROLE db_datareader ADD MEMBER [contoso-pipeline-sp];
ALTER ROLE db_datawriter ADD MEMBER [contoso-pipeline-sp];
```

**Two layers of authorisation, and Azure RBAC is only the first.** A service principal with Owner on
the SQL *resource* still cannot run DDL **inside** the database until a database user exists for it.
`FROM EXTERNAL PROVIDER` is what creates that user from the Microsoft Entra identity.

`db_ddladmin` is the role that permits `CREATE TABLE` and `ALTER TABLE` — the minimum for migrations,
without granting `db_owner`.

**Why the others fail**

- **B** — a firewall block fails at **connection** time with a different message. This error means the
  connection succeeded and the identity was rejected inside the database
- **C** — a wrong port fails to connect
- **D** — a paused serverless database resumes automatically

**Same shape as Challenge 27 and 28:** control plane versus data plane. Managing the resource is not
the same as accessing what is inside it.

</details>

---

## Q9

What does `baselineOnMigrate = true` do in a Flyway configuration?

- A. Creates a baseline of an existing database so Flyway can start managing it
- B. Rolls back all migrations to the baseline
- C. Validates that migration checksums match
- D. Allows migrations to run out of order

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-29.md`:** lines **196–200**.

```toml
[flyway]
locations = ["filesystem:sql"]
baselineOnMigrate = true
outOfOrder = false
validateOnMigrate = true
```

**It solves the adoption problem.** Point Flyway at a database that already has tables and it would
otherwise refuse, seeing a non-empty schema it has no history for. `baselineOnMigrate` marks the
current state as the starting point and applies only later versions.

**Why the others fail**

- **B** — Flyway's undo is a separate command, and rollback is not what this setting does
- **C** — that is `validateOnMigrate = true`, which checks that already-applied files have not been
  edited
- **D** — that is `outOfOrder`, set to `false` here

**All three settings together describe a disciplined pipeline:** adopt an existing database, refuse
edited migrations, refuse out-of-order application.

</details>

---

## Q10

What does `validateOnMigrate = true` protect against?

- A. Migrations running in the wrong order
- B. A previously applied migration file being modified
- C. Migrations running against the wrong database
- D. Data loss during a migration

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-29.md`:** line **200**.

Flyway stores a **checksum** of every applied migration. On the next run it re-checksums the files
and fails if one has changed.

**Why this matters more than it sounds:** editing `V002__Add_customer_preferences.sql` after it has
run in production means production and every other environment now disagree, and nothing would tell
you. Migrations are **immutable once applied** — fix them with a new `V003`.

**Why the others fail**

- **A** — that is `outOfOrder = false`
- **C** — the connection URL decides that
- **D** — Flyway does not evaluate data loss. That is a DACPAC feature (line 649)

</details>

---

## Q11

What does the `R__` prefix mean in a Flyway migration filename?

- A. Rollback migration
- B. Repeatable migration, re-applied whenever its checksum changes
- C. Required migration
- D. Reference data migration

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-29.md`:** line **184**.

```text
    V001__Create_customers_table.sql
    V002__Add_customer_preferences.sql
    V003__Add_orders_indexes.sql
    R__Create_reporting_views.sql
```

**Versioned (`V`) migrations run once, in order.** Repeatable (`R__`) migrations run **after** all
pending versioned ones, and re-run whenever the file changes.

That fits objects you define declaratively and can safely recreate: views, stored procedures,
functions — anything written as `CREATE OR ALTER`. You edit the file rather than adding `V004`,
`V005`, `V006` for the same view.

**Why the others fail** — Flyway's undo migrations use `U`; `R` is not "required" or "reference data".

</details>

---

## Q12

Which `SqlAzureDacpacDeployment@1` argument prevents a schema change that would delete data?

- A. `/p:BlockOnPossibleDataLoss=true`
- B. `/p:DropObjectsNotInSource=false`
- C. `deploymentAction: 'Publish'`
- D. `authenticationType: 'servicePrincipal'`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-29.md`:** line **649**.

```yaml
                    additionalArguments: '/p:BlockOnPossibleDataLoss=true /p:DropObjectsNotInSource=false'
```

**Both flags on that line are guards, and they guard different things:**

| Flag | Prevents |
|---|---|
| `BlockOnPossibleDataLoss=true` | Changes that would **delete row data** — dropping a populated column, narrowing a type |
| `DropObjectsNotInSource=false` | Deleting objects that exist in the database but not in the DACPAC |

`DropObjectsNotInSource=false` matters when the database holds objects created outside the project —
a reporting view, a DBA's index. Left at `true`, a publish would remove them.

**Why the others fail**

- **C** — `Publish` is the action being performed
- **D** — how the task authenticates

</details>

---

## Q13

Which `deployType` runs a plain SQL migration script rather than a DACPAC?

- A. `DacpacTask`
- B. `SqlTask`
- C. `InlineSqlTask`
- D. `ScriptTask`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-29.md`:** lines **667–668**.

```yaml
                    deployType: 'SqlTask'
                    sqlFile: '$(Pipeline.Workspace)/drop/migrations.sql'
```

**Choosing between them is a genuine design decision:**

| | DACPAC | SQL script |
|---|---|---|
| Model | **Declarative** — describe the desired state | **Imperative** — describe the steps |
| Change plan | Generated by comparing to the live database | Written by you |
| Data-loss guard | Built in (`BlockOnPossibleDataLoss`) | None |
| Fits | SQL Server database projects | EF Core, Flyway, Liquibase |

The challenge's EF Core path produces a script (line 92), so `SqlTask` is what applies it.

**Why the others fail** — `InlineSqlTask` is a real third option for inline SQL text; `ScriptTask`
does not exist.

</details>

---

## Q14

Which GitHub Actions action applies a SQL script to Azure SQL in the challenge's workflow?

- A. `azure/sql-action@v2.3`
- B. `azure/CLI@v1`
- C. `microsoft/sql-deploy@v1`
- D. `flyway/flyway-action@v1`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-29.md`:** lines **116–120**.

```yaml
      - name: Run database migrations
        uses: azure/sql-action@v2.3
        with:
          connection-string: ${{ secrets.SQL_CONNECTION_STRING }}
          path: ./publish/migrations.sql
```

Note it runs in the `deploy-database` job, which declares `environment: production` (line 103) — so
approvals and environment secrets apply to the migration, not just to the app deployment.

**Why the others fail**

- **B** — could work via `sqlcmd`, but it is not the purpose-built action
- **C** — does not exist
- **D** — real (line 245) and used for the **Flyway** path, which is Task 2's alternative approach

</details>

---

## Q15

Why does the pipeline generate the migration script during the **build** job rather than the deploy
job?

- A. To produce a reviewable, versioned artifact before anything touches the database
- B. Because the build agent has database access
- C. To reduce deployment time
- D. Because EF Core tools cannot run in a deployment job

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-29.md`:** lines **86–98**, consumed at lines **105–120**.

```yaml
      - name: Generate idempotent migration script
        run: dotnet ef migrations script --idempotent --output ./publish/migrations.sql
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
```

**Three things follow from generating early:**

1. The SQL is an **artifact** — attached to the run, downloadable, auditable
2. A human can **read it** during the environment approval, before it executes
3. The **exact same script** goes to every environment, so staging genuinely rehearses production

**Why the others fail**

- **B** — the reverse. Generating a script needs **no** database connection, which is the point:
  the build agent stays credential-free
- **C** — a marginal effect, not the reason
- **D** — they can; it is a design choice

</details>

---

## Q16

Which strategy does the challenge reserve for **catastrophic** migration failures?

- A. Forward-fix with a compensating migration
- B. Point-in-time restore
- C. Re-running the idempotent script
- D. Dropping and recreating the schema

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-29.md`:** lines **383–405**.

```bash
az sql db restore \
  --dest-name ContosoWebDB-restored \
  --name ContosoWebDB \
  --time "2025-01-15T10:00:00Z"

az sql db rename --name ContosoWebDB          --new-name ContosoWebDB-old
az sql db rename --name ContosoWebDB-restored --new-name ContosoWebDB
```

**Note the shape of the recovery:** restore to a **new** database, verify it, then rename to swap
them. The original is kept as `-old` rather than deleted — the same "keep the previous version alive"
instinct as slots and revisions.

**The cost is unavoidable:** every transaction after the restore point is gone. That is why this is a
last resort and forward-fix is the default.

**Why the others fail**

- **A** — the preferred **normal** path, not the catastrophic one
- **C** — re-running an idempotent script changes nothing; it is already applied
- **D** — destroys all data with no recovery at all

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** phases make up the expand-contract pattern? (Choose three.)

- A. Expand — add new schema alongside the old
- B. Migrate — the application uses both old and new
- C. Contract — remove the old schema
- D. Restore — revert to a backup
- E. Baseline — mark the current state
- F. Validate — check migration checksums

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-29.md`:** lines **437**, **450**, **468**.

```sql
-- Phase 1: Expand  (line 441)
ALTER TABLE dbo.Customers ADD FullName NVARCHAR(512) NULL;
```

```csharp
// Phase 2: Migrate  (line 463)
return FullName ?? $"{FirstName} {LastName}";     // prefer new, fall back to old
```

```sql
-- Phase 3: Contract  (line 473) - a LATER release
ALTER TABLE dbo.Customers DROP COLUMN FirstName;
```

**Why the others fail** — restore is a rollback strategy (line 383); baseline and validate are Flyway
settings (lines 198–200).

**The pattern's real product is a window** in which both schemas work, so deployment and rollback are
both safe at any moment inside it.

</details>

---

## Q18

Which **two** make a migration safe to apply **before** the application deploys? (Choose two.)

- A. It only adds tables, columns or indexes
- B. New columns are nullable or have defaults
- C. It drops unused columns in the same release
- D. It renames columns to match the new model
- E. It is wrapped in a transaction

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-29.md`:** line **270** and lines **441–447**.

**The test is simple: can the *currently running* code survive this change?** Adding a nullable column
is invisible to it. Dropping or renaming one is not.

**Why the others fail**

- **C** — the running application still reads those columns. This is what triggers
  `BlockOnPossibleDataLoss` in Break & fix Exercise 3
- **D** — **a rename is a drop plus an add.** The old code looks for the old name and fails
  immediately. Renames are the most common accidental breaking change
- **E** — a transaction makes the migration atomic, which is good and unrelated. An atomic breaking
  change is still breaking

</details>

---

## Q19

Which **two** Flyway settings enforce migration discipline? (Choose two.)

- A. `validateOnMigrate = true`
- B. `outOfOrder = false`
- C. `baselineOnMigrate = true`
- D. `locations = ["filesystem:sql"]`
- E. `schemas = ["dbo"]`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-29.md`:** lines **199–200**.

| Setting | Enforces |
|---|---|
| `validateOnMigrate = true` | Applied migrations have not been **edited** |
| `outOfOrder = false` | Versions apply **in order**, with no gaps filled later |

**Why `outOfOrder = false` matters:** with it `true`, a `V002` merged after `V003` already ran would
still be applied — so two environments could end up with the same "version" reached by different
paths. Setting it `false` means a late-arriving migration fails loudly instead.

**Why the others fail**

- **C** — adoption behaviour for an existing database
- **D** and **E** — where migrations live and which schema to target. Configuration, not discipline

</details>

---

## Q20

Which **two** rollback options lose data written since the migration? (Choose two.)

- A. Point-in-time restore
- B. Restoring a pre-migration database copy
- C. Forward-fix with a compensating migration
- D. Re-running the idempotent script
- E. Swapping the App Service slot back

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-29.md`:** lines **383–405** and **408–428**.

Both rewind the database to an earlier state, discarding every transaction after it. For Contoso's
API that means real customer orders disappearing.

**Why the others fail**

- **C** — rolls the schema **forward** to a corrected state. Nothing is discarded
- **D** — an idempotent script skips what is already applied. It changes nothing
- **E** — **worth stating plainly.** A slot swap reverts the **application** and does not touch the
  database at all. That is exactly why the schema must stay backward-compatible: after a swap, old
  code runs against the new schema

**That last point connects the whole domain.** Challenges 25–28 make code reversible. Only additive
migrations make the *combination* reversible.

</details>

---

## Q21

Which **two** database roles must the pipeline identity hold to run migrations? (Choose two.)

- A. `db_ddladmin`
- B. `db_datawriter`
- C. `db_owner`
- D. `db_denydatareader`
- E. `sysadmin`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-29.md`:** lines **697–700**.

```sql
CREATE USER [contoso-pipeline-sp] FROM EXTERNAL PROVIDER;
ALTER ROLE db_ddladmin   ADD MEMBER [contoso-pipeline-sp];
ALTER ROLE db_datareader ADD MEMBER [contoso-pipeline-sp];
ALTER ROLE db_datawriter ADD MEMBER [contoso-pipeline-sp];
```

`db_ddladmin` permits `CREATE`/`ALTER`/`DROP` on schema objects. `db_datawriter` is needed because
migrations frequently move data — the backfill at line 445 is an `UPDATE`.

**Why the others fail**

- **C** — `db_owner` would work and grants far more than needed, including permission management. The
  exam rewards least privilege
- **D** — explicitly **denies** reads
- **E** — a server-level role that does not exist in Azure SQL Database in that form

</details>

---

## Q22

Which **two** are true about `dotnet ef migrations script --idempotent`? (Choose two.)

- A. The script can be safely re-run against a partially migrated database
- B. It requires no database connection to generate
- C. It applies migrations directly to the target database
- D. It reverses the most recent migration
- E. It must be regenerated per environment

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-29.md`:** lines **37–42** and **86–92**.

**A** — every migration is guarded by a check against `__EFMigrationsHistory`, so already-applied ones
are skipped.

**B** — the script is generated from the **model and migration files**, not from the live database.
That is why it can run in the build job with no credentials and no network path to production.

**Why the others fail**

- **C** — that is `dotnet ef database update`
- **D** — reversal is `database update <PreviousMigration>`
- **E** — **the opposite is the value.** One script for every environment is what makes staging a real
  rehearsal for production

</details>

---

## Q23

Which **two** guards should a DACPAC publish keep enabled in production? (Choose two.)

- A. `/p:BlockOnPossibleDataLoss=true`
- B. `/p:DropObjectsNotInSource=false`
- C. `/p:BlockOnPossibleDataLoss=false`
- D. `/p:DropObjectsNotInSource=true`
- E. `deploymentAction: 'Script'`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-29.md`:** line **649**.

**A** stops a change that would delete row data. **B** stops the publish removing objects that exist
in the database but not in the project — reporting views, DBA-created indexes, anything added outside
the DACPAC.

**Why the others fail**

- **C** and **D** — the same two flags inverted, which is precisely how the exam phrases the wrong
  answer
- **E** — `deploymentAction: 'Script'` generates the change script **without applying it**. That is
  genuinely useful for review, and it is not a guard on a real publish

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must add a `CustomerPreferences` table and deploy a new API version that uses
it. The deployment must not cause errors for the running application, and a bad release must be
reversible without losing customer data.

---

## Q24

**Proposed solution:** Generate an idempotent migration script in the build job. Run it in a
`deploy-database` stage that the `deploy-application` stage depends on. If the release is bad, swap
the App Service slot back and fix forward with a new migration.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-29.md`:** lines **86–92**, **100–133**, **376–381**.

Each requirement is met by one decision:

| Requirement | Mechanism |
|---|---|
| No errors for running code | The migration is **additive** — a new table (line 221) |
| Correct ordering | `deploy-application` declares `needs: deploy-database` (line 132) |
| Reversible without data loss | Slot swap back for the app, forward-fix for the schema |

`CREATE TABLE dbo.CustomerPreferences` is invisible to the old code, so the running application is
unaffected by the migration and unaffected again if the app is rolled back.

</details>

---

## Q25

**Proposed solution:** Deploy the application first, then run migrations from the application's
startup code using `Database.Migrate()`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Ordering fails outright.** The application starts, receives traffic, and queries a table that does
not exist yet — the exact 45-minute incident in the scenario.

**And it is worse than that at scale.** With multiple instances or a rolling deployment, several
instances run `Database.Migrate()` **simultaneously** against the same database, racing on the same
DDL.

**Startup migration also removes every control the pipeline provides:** no reviewable script, no
environment approval, no separate identity — the *application's* runtime credential now needs
`db_ddladmin` permanently, rather than a pipeline identity holding it only during deployment.

**Where it is acceptable:** local development and single-instance test environments. Never
production.

</details>

---

## Q26

**Proposed solution:** Generate an idempotent script, run it in a `deploy-database` stage that
`deploy-application` depends on, and include `DROP COLUMN` statements for columns the new version no
longer uses.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Ordering is correct. The **additive** requirement is not.

The migration runs **before** the application deploys, so for that window the **old** code is running
against a schema whose columns have been removed. It breaks immediately — the same failure as Q25,
arriving from the opposite direction.

It also destroys reversibility: swapping the slot back restores old code that needs columns which no
longer exist.

**The fix is expand-contract** (lines 468–478): drop the columns in a **later** release, once no
running instance references them.

**Q25 and Q26 together are the whole lesson.** Right order with a breaking change fails; additive
change in the wrong order fails. You need **both**.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — migration ordering

| # | Statement | Answer |
|---|---|---|
| 1 | Additive migrations should run before the application deploys |  |
| 2 | Destructive migrations should run in the same release as the code change |  |
| 3 | `dependsOn` between stages guarantees the migration finishes first |  |
| 4 | An idempotent script makes ordering unnecessary |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Additive migrations should run before the application deploys | **Yes** |
| 2 | Destructive migrations should run in the same release as the code change | **No** |
| 3 | `dependsOn` between stages guarantees the migration finishes first | **Yes** |
| 4 | An idempotent script makes ordering unnecessary | **No** |

**In `challenge-29.md`:** lines **269–276**, **472**, **714–724**.

Row 2 is expand-contract stated as a rule: destructive changes wait for a **later** release, after
every instance has stopped using the old shape.

Row 4 is the tempting misread. Idempotency means **safe to re-run**, not **applied sooner**. The app
still fails if it arrives first.

</details>

---

## Q28 — rollback

| # | Statement | Answer |
|---|---|---|
| 1 | Forward-fix is preferred for a bad migration in production |  |
| 2 | Point-in-time restore loses data written after the restore point |  |
| 3 | An App Service slot swap rolls back database changes |  |
| 4 | Disabling a feature flag reverts a schema change |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Forward-fix is preferred for a bad migration in production | **Yes** |
| 2 | Point-in-time restore loses data written after the restore point | **Yes** |
| 3 | An App Service slot swap rolls back database changes | **No** |
| 4 | Disabling a feature flag reverts a schema change | **No** |

**In `challenge-29.md`:** lines **376–381**, **385–393**.

**Rows 3 and 4 are the connection to the rest of the domain.** Every fast rollback you have learned —
slot swap, revision weight, feature flag — reverts **code only**. The database keeps moving forward.

That asymmetry is the reason for every rule in this challenge: the schema must remain compatible with
**both** the old and the new code for as long as either might run.

</details>

---

## Q29 — Flyway

| # | Statement | Answer |
|---|---|---|
| 1 | `V` migrations run once, in version order |  |
| 2 | `R__` migrations re-run when their checksum changes |  |
| 3 | Editing an applied migration is safe when `validateOnMigrate` is true |  |
| 4 | `baselineOnMigrate` lets Flyway adopt an existing database |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `V` migrations run once, in version order | **Yes** |
| 2 | `R__` migrations re-run when their checksum changes | **Yes** |
| 3 | Editing an applied migration is safe when `validateOnMigrate` is true | **No** |
| 4 | `baselineOnMigrate` lets Flyway adopt an existing database | **Yes** |

**In `challenge-29.md`:** lines **178–185**, **198–200**.

Row 3 inverts the setting's purpose: `validateOnMigrate = true` makes editing an applied migration
**fail the next run**. That is the protection — the checksum mismatch tells you environments have
diverged.

Row 2 is why views and stored procedures belong in `R__` files: you edit the definition rather than
accumulating `V004`, `V005`, `V006` for the same object.

</details>

---

## Q30 — permissions and tooling

| # | Statement | Answer |
|---|---|---|
| 1 | An Azure RBAC role alone lets a pipeline run DDL inside a database |  |
| 2 | `CREATE USER ... FROM EXTERNAL PROVIDER` creates a user from an Entra identity |  |
| 3 | `db_ddladmin` permits `CREATE TABLE` and `ALTER TABLE` |  |
| 4 | `BlockOnPossibleDataLoss=true` should be disabled to unblock a deployment |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | An Azure RBAC role alone lets a pipeline run DDL inside a database | **No** |
| 2 | `CREATE USER ... FROM EXTERNAL PROVIDER` creates a user from an Entra identity | **Yes** |
| 3 | `db_ddladmin` permits `CREATE TABLE` and `ALTER TABLE` | **Yes** |
| 4 | `BlockOnPossibleDataLoss=true` should be disabled to unblock a deployment | **No** |

**In `challenge-29.md`:** lines **692–700**, **736–746**.

Row 1 is the two-layer model: Azure RBAC governs the **resource**; database roles govern what happens
**inside** it. Owner on the SQL server still cannot create a table until a database user exists.

Row 4: the guard fired because a column with data was about to be dropped. Turning it off deletes the
data. The answer is expand-contract.

</details>

---

# Section E — Drag and drop

---

## Q31

Arrange the database-aware deployment sequence in order.

**Items:** Swap staging to production · Deploy the application to staging · Run additive migrations ·
Validate staging against the new schema · Verify migrations applied

<details>
<summary>Show answer</summary>

### Answer

1. Run additive migrations — line **270**
2. Verify migrations applied — line **271**
3. Deploy the application to staging — line **272**
4. Validate staging against the new schema — line **273**
5. Swap staging to production — line **274**

Step 6 in the challenge is *"(Optional) Remove deprecated columns/tables in next release cycle"* —
the **contract** phase, deliberately in a different release.

**Why step 2 exists as its own step:** a migration command exiting 0 is not proof the schema is
correct. Verifying before deploying code means a partial migration is caught while only the database
has changed.

</details>

---

## Q32

Match each rollback strategy to when it should be used.

| Strategy | Use when |
|---|---|
| Forward-fix with a compensating migration |  |
| Point-in-time restore |  |
| Pre-migration database copy |  |
| Slot swap back |  |
| Expand-contract |  |

**Options:** **A high-risk migration**, taken as insurance beforehand · **Catastrophic corruption**, data loss acceptable · **Normal production bug** — preferred · **Planned** removal of schema, over several releases · The **application** is bad; the schema is fine

<details>
<summary>Show answer</summary>

| Strategy | Use when |
|---|---|
| Forward-fix with a compensating migration | **Normal production bug** — preferred |
| Point-in-time restore | **Catastrophic corruption**, data loss acceptable |
| Pre-migration database copy | **A high-risk migration**, taken as insurance beforehand |
| Slot swap back | The **application** is bad; the schema is fine |
| Expand-contract | **Planned** removal of schema, over several releases |

**In `challenge-29.md`:** lines **376**, **385**, **419**, **433**.

**Ranked by data loss:** forward-fix and slot swap lose nothing. A pre-migration copy loses everything
since the copy. A point-in-time restore loses everything since the restore point.

**The order to try them in is that same order.**

</details>

---

## Q33

Arrange the expand-contract releases with their changes.

**Items:** Drop old columns · Add the new nullable column and backfill · Read new only, stop writing
old

<details>
<summary>Show answer</summary>

### Answer

| Release | Database | Application |
|---|---|---|
| **v2.1** | Add `FullName` nullable, backfill | Write to **both** old and new |
| **v2.2** | *(none)* | Read **new only**, stop writing old |
| **v2.3** | Drop `FirstName`, `LastName`; make `FullName NOT NULL` | Remove old references |

**In `challenge-29.md`:** lines **482–486**, with the SQL at **441–447** and **470–477**.

**v2.2 changes no schema at all and cannot be skipped.** It is the release that guarantees no running
instance writes to the old columns, which is the precondition line 472 states for the contract.

**Note the last statement in v2.3:** `ALTER COLUMN FullName ... NOT NULL`. The constraint is only
tightened once every row is populated and every instance writes it — tightening it in v2.1 would have
broken inserts from the old code.

</details>

---

## Q34

Arrange the pipeline jobs in the database-aware workflow, in order.

**Items:** `deploy-application` · `build` · `deploy-database`

<details>
<summary>Show answer</summary>

### Answer

1. `build` — line **66**: compiles, generates the idempotent script, uploads the artifact
2. `deploy-database` — line **100**: `needs: build`, `environment: production`
3. `deploy-application` — line **130**: `needs: deploy-database`, `environment: production`

```yaml
  deploy-database:
    needs: build
    environment: production
  deploy-application:
    needs: deploy-database
    environment: production
```

**Two things worth noticing.** Both deployment jobs declare `environment: production`, so the
**migration** is gated by the same approval as the app — a human sees the SQL before it runs.

And the script is produced in `build`, so `deploy-database` only *applies* a reviewed artifact.

</details>

---

## Q35

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| `Invalid object name 'CustomerPreferences'` |  |
| `The server principal is not able to access the database` |  |
| `Rows were detected... data loss might occur` |  |
| Flyway fails with a checksum mismatch |  |
| Two instances race on DDL at startup |  |

**Options:** An already-applied migration file was edited · Dropping a populated column with `BlockOnPossibleDataLoss=true` · Migrations run from application startup code · Missing `dependsOn` — app deployed before the migration · No database user or `db_ddladmin` for the pipeline identity

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| `Invalid object name 'CustomerPreferences'` | **Missing `dependsOn` — app deployed before the migration** |
| `The server principal is not able to access the database` | **No database user or `db_ddladmin` for the pipeline identity** |
| `Rows were detected... data loss might occur` | **Dropping a populated column with `BlockOnPossibleDataLoss=true`** |
| Flyway fails with a checksum mismatch | **An already-applied migration file was edited** |
| Two instances race on DDL at startup | **Migrations run from application startup code** |

**In `challenge-29.md`:** lines **707**, **692**, **736**, **200**, and Q25's discussion.

**Read the failure's layer:** ordering, authorisation, data safety, immutability, concurrency. Each
one has a different fix and a different owner.

</details>

---

# Section F — Hot area

---

## Q36

```bash
dotnet ef migrations [BLANK 1] \
  --project src/ContosoApi \
  --[BLANK 2] \
  --output ./publish/migrations.sql
```

Requirement: produce a re-runnable SQL artifact during the build.

- **BLANK 1:** `script` / `add` / `remove` / `list`
- **BLANK 2:** `idempotent` / `dry-run` / `no-build` / `verbose`

<details>
<summary>Show answer</summary>

### Answer: `script`, `idempotent`

**In `challenge-29.md`:** lines **86–92**.

`add` creates a migration during development. `--idempotent` is what guards each migration against
`__EFMigrationsHistory` so re-running is safe.

</details>

---

## Q37

```yaml
  deploy-database:
    needs: [BLANK 1]
    environment: production
    steps:
      - uses: [BLANK 2]
        with:
          connection-string: ${{ secrets.SQL_CONNECTION_STRING }}
          path: ./publish/migrations.sql

  deploy-application:
    needs: [BLANK 3]
```

- **BLANK 1:** `build` / `deploy-application` / `test` / `none`
- **BLANK 2:** `azure/sql-action@v2.3` / `azure/CLI@v1` / `azure/webapps-deploy@v3` /
  `flyway/flyway-action@v1`
- **BLANK 3:** `deploy-database` / `build` / `deploy-application` / `test`

<details>
<summary>Show answer</summary>

### Answer: `build`, `azure/sql-action@v2.3`, `deploy-database`

**In `challenge-29.md`:** lines **102**, **117**, **132**.

BLANK 3 is the one that matters. `needs: build` there would let the app deploy **in parallel** with
the migration — Break & fix Exercise 2, and the incident in the scenario.

</details>

---

## Q38

```toml
[flyway]
locations = ["filesystem:sql"]
[BLANK 1] = true
[BLANK 2] = false
[BLANK 3] = true
```

Requirements: adopt an existing database, forbid out-of-order migrations, detect edited files.

- **BLANK 1:** `baselineOnMigrate` / `cleanDisabled` / `mixed` / `group`
- **BLANK 2:** `outOfOrder` / `validateOnMigrate` / `baselineOnMigrate` / `ignoreMissing`
- **BLANK 3:** `validateOnMigrate` / `outOfOrder` / `placeholderReplacement` / `skipDefault`

<details>
<summary>Show answer</summary>

### Answer: `baselineOnMigrate`, `outOfOrder`, `validateOnMigrate`

**In `challenge-29.md`:** lines **198–200**.

Each maps to one requirement in order: adopt, forbid gaps, detect edits.

</details>

---

## Q39

```sql
-- Phase 1  (v2.1)
ALTER TABLE dbo.Customers ADD FullName NVARCHAR(512) [BLANK 1];

UPDATE dbo.Customers
SET FullName = FirstName + ' ' + LastName
WHERE FullName IS NULL;

-- Phase 3  (v2.3)
ALTER TABLE dbo.Customers [BLANK 2] COLUMN FirstName;
ALTER TABLE dbo.Customers ALTER COLUMN FullName NVARCHAR(512) [BLANK 3];
```

- **BLANK 1:** `NULL` / `NOT NULL` / `DEFAULT ''` / `UNIQUE`
- **BLANK 2:** `DROP` / `REMOVE` / `DELETE` / `TRUNCATE`
- **BLANK 3:** `NOT NULL` / `NULL` / `SPARSE` / `DEFAULT ''`

<details>
<summary>Show answer</summary>

### Answer: `NULL`, `DROP`, `NOT NULL`

**In `challenge-29.md`:** lines **441**, **473**, **477**.

**BLANK 1 must be `NULL` in phase 1.** The old code inserts rows without a `FullName`; a `NOT NULL`
column with no default would make every one of those inserts fail the moment the migration ran.

**BLANK 3 tightens it in phase 3**, once the backfill is done and every instance writes the column.
Same constraint, two releases apart, and the gap is the entire point.

</details>

---

## Q40

```yaml
                - task: SqlAzureDacpacDeployment@1
                  inputs:
                    deployType: '[BLANK 1]'
                    deploymentAction: 'Publish'
                    dacpacFile: '$(Pipeline.Workspace)/drop/ContosoApi.dacpac'
                    additionalArguments: '/p:[BLANK 2]=true /p:DropObjectsNotInSource=[BLANK 3]'
```

- **BLANK 1:** `DacpacTask` / `SqlTask` / `InlineSqlTask` / `ScriptTask`
- **BLANK 2:** `BlockOnPossibleDataLoss` / `IgnoreDataLoss` / `AllowIncompatiblePlatform` /
  `VerifyDeployment`
- **BLANK 3:** `false` / `true`

<details>
<summary>Show answer</summary>

### Answer: `DacpacTask`, `BlockOnPossibleDataLoss`, `false`

**In `challenge-29.md`:** lines **646–649**.

Both flags are guards: block changes that would delete row data, and do not remove objects that exist
in the database but not in the project.

</details>

---

## Q41

```sql
CREATE USER [contoso-pipeline-sp] FROM [BLANK 1];
ALTER ROLE [BLANK 2] ADD MEMBER [contoso-pipeline-sp];
```

Requirement: let the pipeline identity create and alter tables, with least privilege.

- **BLANK 1:** `EXTERNAL PROVIDER` / `LOGIN` / `CERTIFICATE` / `ASYMMETRIC KEY`
- **BLANK 2:** `db_ddladmin` / `db_owner` / `db_datareader` / `db_securityadmin`

<details>
<summary>Show answer</summary>

### Answer: `EXTERNAL PROVIDER`, `db_ddladmin`

**In `challenge-29.md`:** lines **697–698**.

`FROM EXTERNAL PROVIDER` creates a database user from a **Microsoft Entra** identity — a managed
identity or service principal — so no SQL password exists.

`db_owner` would work and grants far more, including permission management. `db_ddladmin` is the
least-privilege answer for schema changes.

</details>

---

# Section G — Case study

## Case study: Contoso database-aware delivery

### Background

Contoso's API deployments break because schema changes are not coordinated with code. Last week a new
API version expected a `CustomerPreferences` table that did not exist, causing 500 errors for **45
minutes** until a DBA ran the migration by hand.

The application is .NET 8 with Entity Framework Core against **Azure SQL Database**.

### Requirements

**Ordering and safety**

- Schema changes must be applied before the code that needs them
- The currently running application must never break during a migration
- The migration SQL must be reviewable before it executes

**Permissions**

- The pipeline must run DDL with least privilege
- No SQL username or password may be stored

**Change management**

- Customers' `FirstName` and `LastName` must become a single `FullName` column
- No customer data may be lost
- A bad application release must be reversible

---

## Q42

Which **two** meet the ordering and reviewability requirements? (Choose two.)

- A. Generate an idempotent script in the build job and upload it as an artifact
- B. A `deploy-database` stage that `deploy-application` depends on
- C. `Database.Migrate()` in the application's startup code
- D. A DBA applying migrations manually before each release
- E. Running migrations in the same job as the deployment

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-29.md`:** lines **86–98** and **130–132**.

**A** makes the SQL an artifact — attached to the run, downloadable, and visible to whoever approves
the `production` environment.

**B** makes the ordering structural rather than incidental.

**Why the others fail**

- **C** — the app starts before the schema exists, and multiple instances race on DDL
- **D** — a manual step is what caused the 45-minute incident
- **E** — one job means no separate environment gate on the migration, and no clean failure boundary

</details>

---

## Q43

Which **two** meet the permission requirements? (Choose two.)

- A. `CREATE USER [pipeline-sp] FROM EXTERNAL PROVIDER`
- B. `ALTER ROLE db_ddladmin ADD MEMBER [pipeline-sp]`
- C. `ALTER ROLE db_owner ADD MEMBER [pipeline-sp]`
- D. A SQL admin username and password in a GitHub secret
- E. Contributor on the SQL server resource

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-29.md`:** lines **697–698**.

**A** creates the database user from an Entra identity, so no password exists anywhere. **B** grants
exactly the DDL rights migrations need.

**Why the others fail**

- **C** — works, grants far more than required
- **D** — a stored credential, which the requirement forbids
- **E** — **the one worth understanding.** Contributor is an Azure **control-plane** role. It can
  delete the entire server and still cannot create a table inside a database. Azure RBAC and database
  roles are two separate systems, and migrations need the second

</details>

---

## Q44

Which approach merges `FirstName` and `LastName` into `FullName` without data loss?

- A. Expand-contract across three releases
- B. `ALTER TABLE ... DROP COLUMN` and `ADD COLUMN` in one migration
- C. `sp_rename` on `FirstName` to `FullName`
- D. Create a new table and copy the data

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-29.md`:** lines **433–486**.

**Why the others fail**

- **B** — the old code reads `FirstName` and breaks the moment the migration runs. It would also trip
  `BlockOnPossibleDataLoss`
- **C** — **a rename is a drop plus an add** from the running code's point of view. It also cannot
  merge two columns into one
- **D** — a new table changes every query, every foreign key and every index, and the two tables
  diverge during the switchover. Far more disruptive than adding a column

</details>

---

## Q45

During the expand phase, why must `FullName` be nullable?

- A. Because the old application inserts rows without it
- B. To reduce storage
- C. Because `NVARCHAR(512)` cannot be `NOT NULL`
- D. To allow the backfill to run

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-29.md`:** line **441**, with the constraint tightened later at line **477**.

The old code has no knowledge of `FullName`. A `NOT NULL` column with no default would reject every
insert it makes — turning an "additive, non-breaking" migration into an immediate outage.

**Why the others fail**

- **B** — nullability is not a storage optimisation here
- **C** — `NVARCHAR(512) NOT NULL` is perfectly valid; it is just wrong *at this point*
- **D** — **close, and inverted.** The backfill is an `UPDATE` and works either way. The nullability
  is about the **old application's inserts**, not the backfill

</details>

---

## Q46

A migration succeeds but the new API version has a bug. What is the correct response?

- A. Swap the App Service slot back; leave the schema in place
- B. Restore the database to a point before the migration
- C. Drop the new table and redeploy
- D. Disable a feature flag to revert the schema

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-29.md`:** lines **162–168**, and the additive migration at **221**.

**The schema is additive, so it is harmless.** `CustomerPreferences` is a new table the old code
ignores. Swapping the slot back restores the working application in seconds, and the table simply
sits unused until the fixed version ships.

**Why the others fail**

- **B** — discards every transaction since the restore point to fix a code bug
- **C** — unnecessary, and it would break the fixed version when it arrives
- **D** — a flag toggles **code paths**, never schema (Challenge 27 Q30)

**This question is the payoff for making migrations additive.** Because the schema tolerates both
versions, application rollback stays a simple, fast, lossless operation.

</details>

---

## Q47

Contoso wants staging to rehearse production faithfully.

Which practice achieves this?

- A. Apply the same generated script artifact to staging and production
- B. Regenerate the migration script separately per environment
- C. Use `dotnet ef database update` directly against each database
- D. Maintain a separate migration folder per environment

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-29.md`:** lines **86–98** and **105–120**.

One artifact, produced once in `build`, downloaded by each deployment job. Staging executes the
**identical bytes** that production will.

**Why the others fail**

- **B** — regeneration means the scripts can differ, so staging tests something production will not
  run
- **C** — applies from the model rather than a fixed artifact, and there is nothing to review
- **D** — environments diverge by design, which is the opposite of a rehearsal

**This is the "build once, deploy many" principle applied to the database.** The same rule as
container images: build the artifact once, promote the same one through environments.

</details>

---

## Q48

Two years on, Contoso has 180 EF Core migrations. A new environment takes 40 minutes to build because
every migration runs in sequence, and several early ones fail against the current SQL Server version.

What should they do, and what does this illustrate?

- A. Consolidate the migration history into a new baseline, keeping the old migrations archived
- B. Delete all migrations and recreate the database from the current model
- C. Set `outOfOrder = true` so failing migrations are skipped
- D. Increase the pipeline timeout

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-29.md`:** the Flyway equivalent is `baselineOnMigrate` at line **198**.

**The problem:** migration history is a **replay log**. Every new environment re-executes two years of
history, including migrations written against an older SQL Server that may no longer be valid.

**Squashing** means generating a single script representing the current schema, marking it as the new
baseline, and archiving the originals in source control for reference. New environments start from
the baseline; existing ones are already past it.

**Why the others fail**

- **B** — deleting migrations without a baseline breaks every **existing** database, because their
  `__EFMigrationsHistory` still references migrations that no longer exist
- **C** — `outOfOrder` allows late-arriving versions to apply; it does not skip failures, and skipping
  a failed migration silently would leave environments inconsistent
- **D** — pays the 40 minutes forever and does not address the failures

**What it illustrates:** migrations accumulate the same way feature flags do (Challenge 27 Q48). Both
need periodic, deliberate cleanup, and both break in confusing ways if you clean up carelessly.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Application deployed before the migration** | Q2, Q25, Q42 | `dependsOn` / `needs` on the database stage |
| **Destructive migration in the same release** | Q18, Q26, Q44 | Expand-contract. Drop in a later release |
| **Rename treated as safe** | Q18, Q44 | A rename is a drop plus an add to running code |
| **`NOT NULL` in the expand phase** | Q39, Q45 | Old inserts omit the column. Tighten later |
| **Disabling `BlockOnPossibleDataLoss`** | Q7, Q23, Q30 | The guard is right. Use expand-contract |
| **Restore used for an ordinary bug** | Q4, Q20, Q46 | Forward-fix. Restore is for catastrophe |
| **Slot swap assumed to revert schema** | Q20, Q28, Q46 | It reverts code only. Schema keeps moving forward |
| **Azure RBAC assumed to grant DDL** | Q8, Q30, Q43 | Control plane vs database roles. Create the user |
| **`db_owner` instead of `db_ddladmin`** | Q21, Q43 | Least privilege |
| **Migrations at application startup** | Q25 | No review, no gate, and instances race on DDL |
| **Idempotency confused with ordering** | Q3, Q27 | Safe to re-run is not applied sooner |
| **Editing an applied migration** | Q10, Q29 | Immutable once applied. Add a new one |

---

# The blocks to memorise

Line numbers are in `challenge-29.md`.

```text
# 1. The deployment sequence  (lines 269-276) - the single most important thing here
1. Run database migrations (additive / non-breaking)
2. Verify migrations applied successfully
3. Deploy new application version to staging slot
4. Validate staging with new schema
5. Swap staging to production
6. (Optional) Remove deprecated columns in the NEXT release

# 2. Expand-contract timeline  (lines 482-486)
v2.1  Expand:   add FullName NULL + backfill   | app writes BOTH old and new
v2.2  none                                      | app reads new only, stops writing old
v2.3  Contract: drop old columns, set NOT NULL  | app removes old references
```

```bash
# 3. Idempotent script in the build job  (lines 86-92)
dotnet ef migrations script --idempotent --output ./publish/migrations.sql
```

```yaml
# 4. Strict ordering  (lines 100-132)
  deploy-database:
    needs: build
    environment: production
    steps:
      - uses: azure/sql-action@v2.3
        with:
          connection-string: ${{ secrets.SQL_CONNECTION_STRING }}
          path: ./publish/migrations.sql
  deploy-application:
    needs: deploy-database        # <- the line that prevents the incident
    environment: production

# 5. DACPAC guards  (lines 646-649)
                    deployType: 'DacpacTask'
                    additionalArguments: '/p:BlockOnPossibleDataLoss=true /p:DropObjectsNotInSource=false'
```

```sql
-- 6. Least-privilege pipeline identity  (lines 697-700)
CREATE USER [contoso-pipeline-sp] FROM EXTERNAL PROVIDER;
ALTER ROLE db_ddladmin   ADD MEMBER [contoso-pipeline-sp];
ALTER ROLE db_datareader ADD MEMBER [contoso-pipeline-sp];
ALTER ROLE db_datawriter ADD MEMBER [contoso-pipeline-sp];

-- 7. Expand phase - nullable, then backfill  (lines 441-447)
ALTER TABLE dbo.Customers ADD FullName NVARCHAR(512) NULL;
UPDATE dbo.Customers SET FullName = FirstName + ' ' + LastName WHERE FullName IS NULL;
```

```toml
# 8. Flyway discipline  (lines 196-200)
[flyway]
locations = ["filesystem:sql"]
baselineOnMigrate = true     # adopt an existing database
outOfOrder = false           # no gap-filling
validateOnMigrate = true     # applied migrations are immutable
```

**Rollback, ranked by data loss:** forward-fix (none) → slot swap (none, code only) → pre-migration
copy (since the copy) → point-in-time restore (since the restore point).

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 29 is exam-ready. Move to Challenge 30 |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 3 and 5, then retake this |
| Below 30 | Redo the challenge, writing the expand-contract timeline from memory first |

Record your result in `AZ-400-Learning-Log.md` under Challenge 29.

:::danger The one thing

**Code rolls back. Data does not.**

Slots, revisions and flags all revert because the previous version still exists. A dropped column does
not. That single asymmetry produces every rule in this challenge — additive first, contract later,
forward-fix always.

:::
