---
sidebar_position: 3.5
toc_max_heading_level: 2
title: "Challenge 21: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 21 — AZ-400 exam questions

**48 questions** built only from what Challenge 21 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-21.md`** where the detail lives.

| Section | Shape | Questions |
|---|---|---|
| A | Single answer | 1–16 |
| B | Multiple answer (choose two / three) | 17–23 |
| C | Repeated scenario — "Does this meet the goal?" | 24–26 |
| D | Yes/No statement grid | 27–30 |
| E | Drag and drop | 31–35 |
| F | Hot area — complete the configuration | 36–41 |
| G | Case study | 42–48 |

The **trap index**, the **things to memorise**, and **scoring** are at the end.

:::warning This challenge is mostly judgement, not syntax

Challenge 21 has less YAML than 19 or 20. The exam tests **choosing** — hosted or self-hosted, VMSS
or fixed VMs, ephemeral or persistent. Almost every question hides the deciding constraint in one
clause. Find that clause before you read the options.

:::

---

# Section A — Single answer

---

## Q1

Contoso's integration tests must query an on-premises SQL Server behind a corporate firewall.

Which runner configuration meets the requirement?

- A. GitHub-hosted runners with an allow-list on the firewall
- B. Self-hosted runners inside the corporate network
- C. GitHub-hosted runners with a larger runner size
- D. GitHub-hosted runners with a longer job timeout

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-21.md`:** line **37** ("Network access: Public internet only" vs "Can access private
networks"), line **398** (decision matrix), and line **155** where the workflow reaches
`sql-server.contoso.internal:1433`.

```yaml
  integration-tests:
    runs-on: [self-hosted, linux, on-prem]     # line 150
    steps:
      - run: npm run test:integration
        env:
          SQL_SERVER: sql-server.contoso.internal:1433
```

**Private network access is the number-one technical reason for self-hosted runners.** Cost is a
secondary reason; connectivity is the one with no alternative.

**Why the others fail**

- **A** — GitHub-hosted runners come from large, changing public IP ranges. Allow-listing them would
  mean opening your firewall to a huge chunk of Azure, which no security team accepts
- **C** — a bigger runner has more CPU and RAM. It is still on the public internet
- **D** — a longer timeout does not create a network route. The connection fails, it does not
  time out slowly

</details>

---

## Q2

What does the `--ephemeral` flag do when configuring a self-hosted GitHub Actions runner?

- A. The runner deletes its registration after a configurable timeout
- B. The runner accepts exactly one job and then de-registers itself
- C. The runner uses temporary storage that is cleared between jobs
- D. The runner does not write logs to disk

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-21.md`:** lines **307–312**.

```bash
./config.sh --ephemeral \
  --url https://github.com/contoso \
  --token <TOKEN> \
  --name ephemeral-runner-01
```

One job, then the runner removes itself. Your automation starts a fresh one. That gives self-hosted
infrastructure the **same clean-environment guarantee** as a hosted runner.

**Why the others fail**

- **A** — there is no timeout involved. It is job-count based, and the count is one
- **C** — plausible and wrong. The *whole runner* goes away, not just its storage. A runner that
  merely wiped its workspace would still hold cached credentials in memory and on disk
- **D** — logging is unrelated

**Why it matters (line 41):** a persistent runner is a **shared** runner. Job A can leave credentials,
containers or files that job B — possibly from a different repository — can read.

</details>

---

## Q3

An Azure DevOps pipeline fails with "No agent found in pool matching demands".

What is the cause?

- A. The agent name does not match the pipeline configuration
- B. The agent does not advertise a capability the pipeline demands
- C. The agent is in a different Azure subscription
- D. The agent OS does not match the `vmImage` value

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-21.md`:** Break & fix Exercise 2, lines **430–463**.

```yaml
pool:
  name: contoso-linux-pool
  demands:
    - dotnet8                    # the agent never advertised this
    - docker
    - Agent.OS -equals Linux
```

Azure DevOps uses a **capabilities and demands** system. Agents advertise capabilities; pipelines
declare demands. A job is only assigned when **every** demand is satisfied.

**The fix (lines 452–458):**

```bash
echo 'export dotnet8=/usr/share/dotnet' >> ~/.bashrc   # auto-detected as a capability
# or add it in Organization Settings > Agent pools > Agents > Capabilities
sudo ./svc.sh stop && sudo ./svc.sh start              # restart to re-advertise
```

**Why the others fail**

- **A** — agent names are labels for humans. Nothing matches on them
- **C** — subscriptions are irrelevant to job assignment
- **D** — `vmImage` is only for **Microsoft-hosted** agents (line 211). A self-hosted pool uses
  `name:` and `demands:`, and the two forms are mutually exclusive

</details>

---

## Q4

What is the primary advantage of a VMSS-backed Azure DevOps agent pool?

- A. VMSS agents cost less per minute than Microsoft-hosted agents
- B. VMSS scales agent count with queue demand and can scale to zero when idle
- C. VMSS agents have faster network connectivity than hosted agents
- D. VMSS provides built-in secret management for agents

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-21.md`:** lines **220–241**.

```bash
az vmss create \
  --instance-count 0 \        # line 225 - start at zero
  ...
# Pool settings (lines 238-241):
#   Minimum agents: 0
#   Maximum agents: 10
#   Idle timeout: 30 minutes
#   Desired idle agents: 2
```

**Scale to zero is the headline.** No queued jobs → no VMs → no cost. It sits between always-on
self-hosted VMs (expensive when idle) and hosted agents (no private network, no custom image).

**Why the others fail**

- **A** — not automatically. A VMSS agent still costs VM time plus the $15/parallel-job licence
  (line 47). Cheaper only at sustained volume
- **C** — network performance is not the point. Network *reach* might be, but that is a self-hosted
  property in general, not a VMSS one
- **D** — secrets come from Key Vault or variable groups. VMSS provides compute

</details>

---

## Q5

Which cost multiplier applies to GitHub-hosted macOS runners compared with Linux?

- A. 2x
- B. 5x
- C. 10x
- D. 20x

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-21.md`:** line **34** and lines **367–370**.

```text
  Linux:   $0.008/min
  Windows: $0.016/min     # 2x Linux
  macOS:   $0.08/min      # 10x Linux
```

Windows is **2x**, macOS is **10x**. Both numbers are testable.

**Why this matters (line 397):** it is why the decision matrix sends iOS builds to self-hosted Macs.
At 10x, macOS minutes dominate a bill fast — in Contoso's estimate, 1,000 macOS minutes cost $80
while 2,000 Linux minutes cost $16 (lines 373–374).

</details>

---

## Q6

How many parallel jobs does an Azure DevOps organization get for free on Microsoft-hosted agents?

- A. 0
- B. 1
- C. 5
- D. 10

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-21.md`:** line **47**.

```text
| Cost | 1 free parallel job, then $40/parallel job/month | $15/parallel job/month + infra |
```

One free Microsoft-hosted parallel job, with 1,800 minutes per month. After that, **$40 per parallel
job per month** for hosted, or **$15 per parallel job per month** for self-hosted.

**Why the numbers matter:** the self-hosted licence being cheaper than the hosted one is the reason
"buy more parallel jobs" is sometimes the wrong exam answer — self-hosted plus a licence can cost
less at scale, and Challenge 35 revisits this as a pipeline-optimisation trade-off.

</details>

---

## Q7

A workflow job must run on a self-hosted runner that has Docker and sits on the internal network.

Which `runs-on` value is correct?

- A. `runs-on: self-hosted`
- B. `runs-on: [self-hosted, linux, docker]`
- C. `runs-on: contoso-runner-linux-01`
- D. `runs-on: docker`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-21.md`:** the labels are set at line **85**, consumed at line **158**.

```bash
./config.sh --labels linux,docker,on-prem     # line 85
```

```yaml
  docker-build:
    runs-on: [self-hosted, linux, docker]     # line 158
```

An **array means AND**. The job needs a runner carrying *every* label listed.

**Why the others fail**

- **A** — valid syntax, but it matches **any** self-hosted runner, including the macOS one. The job
  could land somewhere without Docker
- **C** — `runs-on` matches labels, not runner names. A name is not a label
- **D** — omits `self-hosted`, so it would try to match a GitHub-hosted runner label. There is no
  hosted label called `docker`

</details>

---

## Q8

What is a GitHub runner group used for?

- A. Load balancing jobs across runners
- B. Controlling which repositories and workflows may use a set of runners
- C. Grouping runners by operating system for billing
- D. Defining the labels a runner advertises

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-21.md`:** lines **123–131**, and `--runnergroup` at line **86**.

```bash
gh api --method POST /orgs/contoso/actions/runner-groups \
  -f name="internal-network" \
  -f visibility="selected" \
  -F selected_repository_ids[]="<repo-id-1>" \
  -F allows_public_repositories=false
```

It is an **access control** boundary. `visibility: selected` limits the group to named repositories,
and `allows_public_repositories=false` blocks public repos entirely.

**Why the others fail**

- **A** — load balancing across matching runners happens automatically. That is not what a group does
- **C** — billing is per-minute on hosted runners. Self-hosted runners are not billed by GitHub
- **D** — labels are set with `--labels` (line 85). A group and a label are different mechanisms:
  **group = who may use it. Label = which job matches it.**

**Note:** runner groups require GitHub Team or Enterprise (line 122).

</details>

---

## Q9

Why does the runner VM in Challenge 21 specify `--public-ip-address ""`?

- A. To reduce the VM's cost
- B. To keep the runner off the public internet, reachable only inside the VNet
- C. Because GitHub runners cannot use public IPs
- D. To force the runner into ephemeral mode

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-21.md`:** line **69**.

```bash
az vm create \
  --vnet-name contoso-runners-vnet \
  --subnet runners-subnet \
  --public-ip-address ""          # no inbound public address
```

The runner still reaches GitHub **outbound** over 443 — that is what the NSG rule at lines 326–335
allows. But nothing on the internet can reach the runner.

**This is the key architectural fact about self-hosted runners:** the connection is **outbound only**.
The runner polls GitHub or Azure DevOps for work. You never open an inbound firewall port, which is
what makes the pattern acceptable to security teams.

**Why the others fail**

- **A** — a public IP is a trivial cost. Security is the reason
- **C** — runners can have public IPs; it is just poor practice
- **D** — ephemeral is `--ephemeral` on `config.sh` (line 309), unrelated to networking

</details>

---

## Q10

Which outbound port must a self-hosted runner be able to reach?

- A. 22
- B. 443
- C. 5986
- D. 8080

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-21.md`:** lines **326–335**.

```bash
az network nsg rule create \
  --name AllowGitHub \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 443 \
  --destination-address-prefixes "140.82.112.0/20" "143.55.64.0/20"
```

HTTPS outbound to GitHub's address ranges. Note `--direction Outbound` — there is no inbound rule at
all.

**Why the others fail**

- **A** — SSH is for **you** to administer the VM. The runner does not use it
- **C** — WinRM over HTTPS, used for Windows remote management, not runner communication
- **D** — a common application port, not used here

</details>

---

## Q11

What is Actions Runner Controller (ARC)?

- A. A GitHub-hosted service that manages runner scaling
- B. A Kubernetes controller that runs GitHub Actions runners as auto-scaling pods
- C. An Azure DevOps agent pool type
- D. A CLI tool for registering runners in bulk

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-21.md`:** lines **271–300**.

```bash
helm install contoso-runners \
  --namespace arc-runners \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --set githubConfigUrl="https://github.com/contoso" \
  --set minRunners=1 \
  --set maxRunners=10
```

```yaml
jobs:
  build:
    runs-on: arc-runner-set     # line 296 - the scale set NAME, not a label list
```

**ARC is the GitHub answer to VMSS.** Both give elastic, scale-to-near-zero self-hosted capacity;
ARC does it on Kubernetes, VMSS does it on Azure VMs.

**Why the others fail**

- **A** — ARC runs in **your** cluster. GitHub does not host it
- **C** — VMSS is the Azure DevOps scale-set pool type. ARC is GitHub-only
- **D** — it is a controller that continuously reconciles runner count, not a one-shot tool

**Note line 296:** with ARC, `runs-on` takes the **scale set name**, not a label array.

</details>

---

## Q12

A self-hosted runner shows as "Offline" and its log reports `Http response code: 403`.

What is the most likely cause?

- A. The runner service is stopped
- B. The registration token expired or the runner registration was removed
- C. The VM has no outbound internet access
- D. The runner has the wrong labels

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-21.md`:** Break & fix Exercise 1, lines **405–428**.

```bash
sudo ./svc.sh status
# Output: active (running)         <- the service IS running

cat _diag/Runner_*.log | tail -50
# Shows: "Failed to connect. Http response code: 403"
```

**The fix:**

```bash
./config.sh remove --token <REMOVAL_TOKEN>
./config.sh --url https://github.com/contoso --token <NEW_REGISTRATION_TOKEN> \
  --name contoso-runner-linux-01 --labels linux,docker,on-prem --replace
sudo ./svc.sh start
```

**Why the others fail**

- **A** — the question's own diagnostic shows `active (running)`. Read the evidence given
- **C** — no network would produce a connection or DNS error, not an HTTP **403**. A 403 means the
  server answered and refused you — so connectivity works and authorisation does not
- **D** — wrong labels mean jobs never match. The runner would still show **Online** and simply sit
  idle

**Read the error code.** 403 = reached the server, rejected. Timeout = never reached it. That
distinction is worth exam points.

</details>

---

## Q13

Which Azure DevOps pool configuration targets a self-hosted pool rather than Microsoft-hosted agents?

- A. `pool: vmImage: "ubuntu-latest"`
- B. `pool: name: contoso-linux-pool`
- C. `pool: ubuntu-latest`
- D. `pool: self-hosted`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-21.md`:** lines **202–211**.

```yaml
pool:
  name: contoso-linux-pool      # self-hosted pool
  demands:
    - docker
    - Agent.OS -equals Linux
    - dotnet8

pool:
  vmImage: "ubuntu-latest"      # Microsoft-hosted
```

`name` and `vmImage` are **mutually exclusive**. One or the other decides which world you are in.

**Why the others fail**

- **A** — that is the Microsoft-hosted form
- **C** — `pool: ubuntu-latest` is invalid. This is the classic Azure Pipelines YAML error the exam
  tips call out: `pool` needs a nested key, not a bare string
- **D** — `self-hosted` is a **GitHub Actions** label. Azure DevOps has no such value

</details>

---

## Q14

An agent must only run jobs on Linux. Which demand expresses that?

- A. `- Agent.OS -equals Linux`
- B. `- os == linux`
- C. `- runs-on: linux`
- D. `- Agent.OS: Linux`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-21.md`:** line **206**.

```yaml
  demands:
    - docker                        # exists demand: capability must be present
    - Agent.OS -equals Linux        # equals demand: capability must have this value
```

Two demand forms:

- **exists** — just name the capability: `- docker`
- **equals** — `- <capability> -equals <value>`

**Why the others fail**

- **B** — `==` is not Azure Pipelines demand syntax
- **C** — `runs-on` is GitHub Actions
- **D** — a colon makes it a YAML mapping. Demands are strings in a list

</details>

---

## Q15

Which statement best describes the cost trade-off in Challenge 21's break-even analysis?

- A. Self-hosted is always cheaper than hosted
- B. Hosted is cheaper until macOS usage grows large, and maintenance time is a real cost
- C. Hosted is always cheaper because there is no infrastructure to run
- D. Cost is identical; only capability differs

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-21.md`:** lines **366–391**.

```text
Total hosted:   $96/month

Self-hosted alternative:
  Azure VM (Standard_D4s_v5):
    On-demand: ~$140/month (always on)
  Maintenance overhead: ~$500/month (engineer time)

Break-even:
  - Hosted is cheaper until ~50 hours/month on macOS
```

**Maintenance is $500/month — over three times the VM cost.** That is the number people forget, and
the exam rewards remembering that engineer time is the dominant cost of self-hosting.

**Why the others fail**

- **A** and **C** — both absolute. Absolutes are almost always wrong on trade-off questions
- **D** — the analysis shows a clear cost difference

**When self-hosted wins (lines 386–390):** private network access, very high Linux volume, macOS
above ~1,000 minutes/month, custom hardware or persistent caches.

</details>

---

## Q16

Which authentication method does a self-hosted Azure DevOps agent use during `config.sh`?

- A. A personal access token
- B. A service principal secret
- C. A managed identity
- D. An SSH key

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-21.md`:** lines **184–192**.

```bash
./config.sh \
  --unattended \
  --url https://dev.azure.com/contoso \
  --auth pat \
  --token <PAT_TOKEN> \
  --pool "contoso-linux-pool" \
  --agent contoso-agent-linux-01 \
  --acceptTeeEula \
  --replace
```

**Important nuance the exam likes:** the PAT is used **only during registration**. Once configured,
the agent holds its own credential and the PAT can be revoked. The GitHub equivalent is the
short-lived **registration token** at line 83.

**Why the others fail**

- **B** and **C** — service principals and managed identities authenticate to **Azure**, not to
  Azure DevOps agent registration
- **D** — SSH gets *you* onto the VM (line 72). The agent does not use it

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** are genuine advantages of self-hosted runners over GitHub-hosted runners? (Choose
three.)

- A. Access to private network resources
- B. Persistent local caches on the filesystem
- C. Automatic OS security patching
- D. Full control over installed software
- E. A guaranteed clean environment for every job
- F. Zero maintenance effort

<details>
<summary>Show answer</summary>

### Answer: A, B, D

**In `challenge-21.md`:** lines **31–41**.

| Factor | Hosted | Self-hosted |
|---|---|---|
| Network access | Public internet only | **Private networks** (A) |
| Caching | `actions/cache` round-trip | **Local filesystem** (B) |
| Customization | Pre-installed tools only | **Full control** (D) |

**Why the others fail — all three are hosted advantages, inverted**

- **C** — line 35: hosted runners are auto-updated by GitHub. Self-hosted means **you** patch
- **E** — line 36: hosted gets a fresh VM every job. Self-hosted is persistent unless you make it
  ephemeral
- **F** — line 35 again: self-hosted is self-managed, and line 382 prices that at ~$500/month

**The exam pattern:** it lists real hosted advantages as if they were self-hosted ones. Know the
comparison table in both directions.

</details>

---

## Q18

Which **two** options provide elastic, scale-to-near-zero self-hosted capacity? (Choose two.)

- A. Azure VM Scale Set agent pool in Azure DevOps
- B. Actions Runner Controller on Kubernetes
- C. A fixed pool of always-on Azure VMs
- D. Microsoft-hosted agents
- E. A macOS Mac Mini in the office

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-21.md`:** VMSS at lines **220–241**, ARC at lines **271–288**.

```bash
az vmss create --instance-count 0 ...       # Azure DevOps side
# Minimum agents: 0, Maximum agents: 10

helm install contoso-runners ... \
  --set minRunners=1 --set maxRunners=10    # GitHub side
```

**Why the others fail**

- **C** — always-on is the opposite of elastic. It is the ~$140/month option at line 379
- **D** — hosted agents *are* elastic, but they are not self-hosted, so they fail the private-network
  and custom-image requirements
- **E** — physical hardware does not scale to zero

**The pairing to remember:** *Azure DevOps → VMSS. GitHub Actions → ARC.*

</details>

---

## Q19

Which **two** settings make a self-hosted runner behave like a hosted one for security purposes?
(Choose two.)

- A. `--ephemeral` on `config.sh` for GitHub runners
- B. "Automatically tear down virtual machines after every use" for VMSS pools
- C. Running the runner service as root
- D. Adding more labels to the runner
- E. Increasing the idle timeout

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-21.md`:** lines **307–315**.

```bash
./config.sh --ephemeral ...                                    # GitHub
# Azure DevOps VMSS: "Automatically tear down VMs after every use" = Yes
```

Both give a **one-job lifetime**, which is what makes hosted runners safe: nothing survives to leak
into the next job.

**Why the others fail**

- **C** — the opposite. Line 321 says run as a **non-root** user with minimal permissions. Root means
  any compromised workflow owns the machine
- **D** — labels route jobs. They are not a security control
- **E** — a longer idle timeout keeps the machine alive *longer*, increasing exposure. It is a cost
  and latency setting

</details>

---

## Q20

Which **three** are valid runner and agent security hardening measures from Challenge 21? (Choose
three.)

- A. Run the runner as a non-root user
- B. Restrict outbound network access with NSG rules
- C. Limit a runner group to specific repositories
- D. Store a long-lived PAT on the runner for cloud authentication
- E. Give every repository access to every runner
- F. Disable audit logging to reduce noise

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-21.md`:** lines **320–341**.

```bash
# 1. Run agent as non-root user with minimal permissions        (A)
useradd -m -s /bin/bash agentuser

# 2. Restrict network access with firewall rules                (B)
az network nsg rule create --direction Outbound --destination-port-ranges 443 ...

# 3. Limit runner group to specific repositories                (C)
# 4. Use short-lived registration tokens
# 5. Enable audit logging for runner activity
# 6. Use just-in-time runner provisioning (ephemeral)
```

**Why the others fail**

- **D** — the opposite of the guidance. Lines 346–359 show **OIDC** precisely so no long-lived
  credential sits on the runner
- **E** — the opposite of C, and of `allows_public_repositories=false` at line 131
- **F** — line 339 says **enable** audit logging

</details>

---

## Q21

Which **two** are required for a self-hosted runner to authenticate to Azure without stored secrets?
(Choose two.)

- A. `permissions: id-token: write` on the job
- B. `azure/login@v2` with `client-id`, `tenant-id` and `subscription-id`
- C. A service principal client secret in a repository secret
- D. A managed identity assigned to the runner VM
- E. `permissions: packages: write` on the job

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-21.md`:** lines **346–359**.

```yaml
  deploy:
    runs-on: [self-hosted, linux]
    permissions:
      id-token: write               # A
      contents: read
    steps:
      - uses: azure/login@v2        # B
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

**Why the others fail**

- **C** — that is the stored secret the requirement forbids
- **D** — **the interesting one.** The VM *does* have a managed identity available. But the OIDC
  token here is issued by **GitHub** to the workflow, and the federated credential trusts GitHub's
  issuer plus a subject like `repo:contoso/api:ref:refs/heads/main`. Using the VM's identity instead
  would authenticate *the machine*, not *the workflow* — so every workflow on that runner would share
  one identity. On a self-hosted runner, workflow identity beats machine identity
- **E** — `packages: write` is for GHCR (Challenge 19)

**This block is identical to Challenge 19 Q41.** It repeats because security auth is your weakest
domain — and because it is genuinely the same answer wherever the runner lives.

</details>

---

## Q22

Which **two** situations justify GitHub-hosted runners over self-hosted? (Choose two.)

- A. Simple CI such as lint and unit tests
- B. Teams that want zero infrastructure maintenance
- C. Builds needing on-premises database access
- D. iOS builds at high volume
- E. Builds requiring a persistent Docker layer cache

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-21.md`:** decision matrix, lines **395–401**.

| Requirement | Recommendation |
|---|---|
| Simple CI (lint, unit test) | **GitHub-hosted** — low cost, zero maintenance |
| On-premises SQL access | Self-hosted |
| iOS builds (macOS) | Self-hosted — 10x multiplier |
| Docker builds with cache | Self-hosted — local cache |

**Why the others fail**

- **C** — hosted cannot reach private networks (line 37)
- **D** — the 10x macOS multiplier makes volume expensive (line 34)
- **E** — hosted runners are fresh every job, so no local cache survives (line 40)

**Default posture for the exam:** start hosted, move to self-hosted only when a **specific** blocker
appears — network reach, cost at volume, custom hardware, or compliance.

</details>

---

## Q23

Which **two** statements about GitHub runner labels are correct? (Choose two.)

- A. An array in `runs-on` means the runner must have **all** listed labels
- B. Labels are assigned with `--labels` during `config.sh`
- C. Labels control which repositories may use a runner
- D. `runs-on` can match a runner by its name
- E. Labels are automatically derived from installed software

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-21.md`:** line **85** (B), lines **144–158** (A).

```bash
./config.sh --labels linux,docker,on-prem     # line 85
```

```yaml
    runs-on: [self-hosted, macOS, xcode-15]      # line 144 - AND
    runs-on: [self-hosted, linux, on-prem]       # line 150
    runs-on: [self-hosted, linux, docker]        # line 158
```

**Why the others fail**

- **C** — that is a **runner group** (lines 123–131). Group = who may use it. Label = which job
  matches it. Keep them apart
- **D** — `runs-on` matches labels only. The runner's `--name` is for humans
- **E** — labels are declared by you. This is where GitHub and Azure DevOps genuinely differ: Azure
  DevOps agents **do** auto-detect capabilities from environment variables (line 454), GitHub runners
  do not

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso's data engineering team runs integration tests that must query an on-premises
SQL Server. Builds currently fail because the database is unreachable. The team also wants each job
to start from a clean environment.

---

## Q24

**Proposed solution:** Deploy self-hosted runners inside the corporate network, configured with
`--ephemeral`, and target them with `runs-on: [self-hosted, linux, on-prem]`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

Both requirements are satisfied:

```bash
./config.sh --ephemeral --labels linux,on-prem ...   # clean environment (line 309)
```

```yaml
    runs-on: [self-hosted, linux, on-prem]           # network reach (line 150)
```

- **Network reach** — the runner sits inside the network, so `sql-server.contoso.internal` resolves
- **Clean environment** — ephemeral means one job per runner, then de-registration

You would also need automation to start a replacement runner. That is implied by the pattern, not a
gap in the design.

</details>

---

## Q25

**Proposed solution:** Keep GitHub-hosted runners and add the GitHub runner IP ranges to the
corporate firewall allow-list.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

It fails **both** requirements, and the first failure is the serious one.

**Network:** GitHub-hosted runners come from large, frequently changing Azure IP ranges. Allow-listing
them opens your internal SQL Server to a huge shared address space that any GitHub customer's
workflow can run in. That is not access control; it is exposure.

**Clean environment:** this half would actually pass — hosted runners are fresh every job (line 36).

**On the exam:** when a proposal solves a connectivity problem by widening a firewall to a public
cloud range, it is wrong. The intended answer is to move the compute **inside** the network, not to
open the network to the compute.

</details>

---

## Q26

**Proposed solution:** Deploy persistent self-hosted runners inside the corporate network without
`--ephemeral`.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Network reach: solved.** **Clean environment: not solved.**

Line 36 states self-hosted runners are persistent and you must manage cleanup yourself. Line 41 adds
the consequence: *"Shared runner risk if not ephemeral."*

Concretely, without ephemeral the next job inherits:

- files left in the workspace
- Docker images, containers and volumes
- cached credentials and environment changes

**A partial solution is still No.** Both requirements were stated, so both must be met.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — hosted versus self-hosted

| # | Statement | Answer |
|---|---|---|
| 1 | GitHub-hosted runners can reach private network resources |  |
| 2 | Self-hosted runners keep a local cache between jobs |  |
| 3 | GitHub patches the OS of self-hosted runners |  |
| 4 | Hosted runners provide a fresh VM for every job |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | GitHub-hosted runners can reach private network resources | **No** |
| 2 | Self-hosted runners keep a local cache between jobs | **Yes** |
| 3 | GitHub patches the OS of self-hosted runners | **No** |
| 4 | Hosted runners provide a fresh VM for every job | **Yes** |

**In `challenge-21.md`:** lines **35–40**.

Row 3 is the cost nobody budgets for. GitHub auto-updates the **runner application**, but the
operating system, Docker, the .NET SDK, Node and everything else you installed at lines 99–114 are
yours to patch — the ~$500/month at line 382.

Row 2 is the performance argument: line 40 contrasts `actions/cache` (a network round-trip) with a
local filesystem cache.

</details>

---

## Q28 — ephemeral runners

| # | Statement | Answer |
|---|---|---|
| 1 | An ephemeral runner runs one job then de-registers |  |
| 2 | Ephemeral runners are recommended for public repositories |  |
| 3 | Ephemeral means the workspace is wiped but the runner stays online |  |
| 4 | Azure DevOps VMSS pools have an equivalent setting |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | An ephemeral runner runs one job then de-registers | **Yes** |
| 2 | Ephemeral runners are recommended for public repositories | **Yes** |
| 3 | Ephemeral means the workspace is wiped but the runner stays online | **No** |
| 4 | Azure DevOps VMSS pools have an equivalent setting | **Yes** |

**In `challenge-21.md`:** lines **307–316**.

Row 2 matters and is worth understanding, not memorising: on a **public** repository, anyone can open
a pull request, and a workflow that runs on PR code executes **untrusted code on your machine**. A
persistent runner would let that code read the next job's secrets and files. Ephemeral limits the
blast radius to one job.

Row 3 is the distractor from Q2 restated as a grid row. The **runner** goes away, not just its files.

</details>

---

## Q29 — Azure DevOps agents

| # | Statement | Answer |
|---|---|---|
| 1 | `pool: name:` and `pool: vmImage:` can be used together |  |
| 2 | Demands must all be satisfied for a job to be assigned |  |
| 3 | Agents advertise capabilities automatically from environment variables |  |
| 4 | A PAT is required every time the agent runs a job |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `pool: name:` and `pool: vmImage:` can be used together | **No** |
| 2 | Demands must all be satisfied for a job to be assigned | **Yes** |
| 3 | Agents advertise capabilities automatically from environment variables | **Yes** |
| 4 | A PAT is required every time the agent runs a job | **No** |

**In `challenge-21.md`:** lines **202–211** (row 1), lines **437–441** (row 2), line **454** (row 3),
lines **184–192** (row 4).

Row 3 is the genuine platform difference: setting `export dotnet8=/usr/share/dotnet` makes the agent
advertise a `dotnet8` capability after a restart. GitHub runners have no equivalent — you declare
labels by hand.

Row 4: the PAT authenticates **registration only**. Afterwards the agent holds its own credential, so
the PAT can be revoked. Leaving a long-lived PAT on the machine is the security failure the exam
tests.

</details>

---

## Q30 — scaling

| # | Statement | Answer |
|---|---|---|
| 1 | A VMSS agent pool can scale to zero agents |  |
| 2 | ARC runs GitHub runners as Kubernetes pods |  |
| 3 | VMSS pools work with GitHub Actions |  |
| 4 | ARC requires a Kubernetes cluster you operate |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A VMSS agent pool can scale to zero agents | **Yes** |
| 2 | ARC runs GitHub runners as Kubernetes pods | **Yes** |
| 3 | VMSS pools work with GitHub Actions | **No** |
| 4 | ARC requires a Kubernetes cluster you operate | **Yes** |

**In `challenge-21.md`:** lines **225–241** (row 1), lines **271–288** (rows 2 and 4).

Row 3 is the one to get right. VMSS-backed pools are an **Azure DevOps** pool type. GitHub Actions
has no VMSS integration — its scaling story is ARC.

Row 4 is the hidden cost of ARC: you now operate a Kubernetes cluster. Cheap if you already run AKS,
expensive if this would be your first cluster. The exam sometimes phrases this as "the team has no
Kubernetes experience", which pushes the answer toward VMSS or hosted.

</details>

---

# Section E — Drag and drop

---

## Q31

Arrange the steps to bring a self-hosted GitHub runner online, in order.

**Items:** Install the runner as a service · Download the runner package · Create the VM ·
Run `config.sh` with a registration token · Start the service

<details>
<summary>Show answer</summary>

### Answer

1. Create the VM — line **59**
2. Download the runner package — line **76**
3. Run `config.sh` with a registration token — line **81**
4. Install the runner as a service — line **91**
5. Start the service — line **92**

```bash
az vm create --name contoso-runner-linux-01 ...        # 1
curl -o actions-runner-linux-x64-2.321.0.tar.gz -L ... # 2
./config.sh --url ... --token ... --labels ...         # 3
sudo ./svc.sh install                                  # 4
sudo ./svc.sh start                                    # 5
```

**Why the order is fixed:** `config.sh` writes the runner's credential and settings to disk.
`svc.sh install` reads those files to create the service unit, so configuration must come first.

</details>

---

## Q32

Match each capability to the platform that provides it.

| Capability | Platform |
|---|---|
| VMSS-backed elastic agent pool |  |
| Actions Runner Controller (ARC) |  |
| Capabilities auto-detected from env vars |  |
| Runner groups with repository visibility |  |
| `demands` in the pool declaration |  |
| Labels array in `runs-on` |  |

**Options:** Azure DevOps · GitHub Actions

<details>
<summary>Show answer</summary>

| Capability | Platform |
|---|---|
| VMSS-backed elastic agent pool | **Azure DevOps** |
| Actions Runner Controller (ARC) | **GitHub Actions** |
| Capabilities auto-detected from env vars | **Azure DevOps** |
| Runner groups with repository visibility | **GitHub Actions** |
| `demands` in the pool declaration | **Azure DevOps** |
| Labels array in `runs-on` | **GitHub Actions** |

**In `challenge-21.md`:** VMSS **220–241**, ARC **271–288**, capabilities **454**, runner groups
**123–131**, demands **204–207**, labels **144–158**.

**The mental model:** both platforms solve the same four problems — matching, access control,
scaling, cleanliness — with different names.

| Problem | GitHub | Azure DevOps |
|---|---|---|
| Which runner? | labels | pool + demands |
| Who may use it? | runner group | pool permissions |
| Elastic scale | ARC | VMSS |
| One-job lifetime | `--ephemeral` | tear down after use |

</details>

---

## Q33

Match each Contoso requirement to its runner recommendation.

| Requirement | Recommendation |
|---|---|
| iOS builds requiring Xcode |  |
| Integration tests against on-prem SQL |  |
| Docker builds needing a warm layer cache |  |
| Lint and unit tests |  |
| Data residency compliance |  |

**Options:** GitHub-hosted · Self-hosted in the required region · Self-hosted inside the corporate network · Self-hosted macOS · Self-hosted with local Docker cache

<details>
<summary>Show answer</summary>

| Requirement | Recommendation |
|---|---|
| iOS builds requiring Xcode | **Self-hosted macOS** |
| Integration tests against on-prem SQL | **Self-hosted inside the corporate network** |
| Docker builds needing a warm layer cache | **Self-hosted with local Docker cache** |
| Lint and unit tests | **GitHub-hosted** |
| Data residency compliance | **Self-hosted in the required region** |

**In `challenge-21.md`:** decision matrix, lines **395–401**.

Each row has a **different** driver: cost multiplier, network reach, cache locality, simplicity,
regulation. The exam gives you one of these drivers in a sentence and expects the matching answer.

</details>

---

## Q34

Arrange these runner options from lowest to highest maintenance burden.

**Items:** Self-hosted VM (always on) · GitHub-hosted runner · ARC on Kubernetes ·
VMSS-backed agent pool

<details>
<summary>Show answer</summary>

### Answer

1. **GitHub-hosted** — line 35, fully managed
2. **VMSS-backed pool** — image and scaling config to maintain, but Azure manages the VM lifecycle
3. **Self-hosted VM (always on)** — you patch the OS and tools, and it runs 24/7
4. **ARC on Kubernetes** — everything above plus operating a cluster

**The trade-off in one line:** maintenance rises exactly as control rises. Line 382 prices the
maintenance at ~$500/month, which is more than the VM itself (~$140/month at line 379).

</details>

---

## Q35

Match each cost figure to what it describes.

| Figure | Meaning |
|---|---|
| $0.008/min |  |
| $0.016/min |  |
| $0.08/min |  |
| $40/month |  |
| $15/month |  |
| ~$500/month |  |

**Options:** Extra Azure DevOps **Microsoft-hosted** parallel job · Extra Azure DevOps **self-hosted** parallel job · GitHub-hosted **Linux** · GitHub-hosted **macOS** (10x) · GitHub-hosted **Windows** (2x) · **Maintenance** — engineer time

<details>
<summary>Show answer</summary>

| Figure | Meaning |
|---|---|
| $0.008/min | GitHub-hosted **Linux** |
| $0.016/min | GitHub-hosted **Windows** (2x) |
| $0.08/min | GitHub-hosted **macOS** (10x) |
| $40/month | Extra Azure DevOps **Microsoft-hosted** parallel job |
| $15/month | Extra Azure DevOps **self-hosted** parallel job |
| ~$500/month | **Maintenance** — engineer time |

**In `challenge-21.md`:** lines **367–370**, line **47**, line **382**.

**The two numbers most likely to be tested:** macOS is **10x** Linux, and Azure DevOps gives **one**
free hosted parallel job.

</details>

---

# Section F — Hot area

---

## Q36

```bash
./config.sh \
  --url https://github.com/contoso \
  --token <REGISTRATION_TOKEN> \
  --[BLANK 1] linux,docker,on-prem \
  --[BLANK 2] contoso-internal \
  --[BLANK 3]
```

- **BLANK 1:** `labels` / `tags` / `demands` / `capabilities`
- **BLANK 2:** `runnergroup` / `pool` / `team` / `scope`
- **BLANK 3:** `ephemeral` / `persistent` / `once` / `single`

<details>
<summary>Show answer</summary>

### Answer: `labels`, `runnergroup`, `ephemeral`

**In `challenge-21.md`:** lines **85–86** and line **309**.

`demands` and `capabilities` are **Azure DevOps** words; `pool` too. `tags` belongs to Azure
resources. `persistent`, `once` and `single` do not exist as flags.

</details>

---

## Q37

```yaml
jobs:
  integration-tests:
    runs-on: [BLANK 1]
```

- **BLANK 1:** `[self-hosted, linux, on-prem]` / `self-hosted` / `contoso-runner-linux-01` /
  `[linux]`

<details>
<summary>Show answer</summary>

### Answer: `[self-hosted, linux, on-prem]`

**In `challenge-21.md`:** line **150**.

The array means **all** labels must be present. Bare `self-hosted` could land the job on the macOS
runner. A runner **name** never matches. `[linux]` alone would look for a hosted runner label.

</details>

---

## Q38

```yaml
pool:
  [BLANK 1]: contoso-linux-pool
  [BLANK 2]:
    - docker
    - Agent.OS [BLANK 3] Linux
```

- **BLANK 1:** `name` / `vmImage` / `runs-on` / `group`
- **BLANK 2:** `demands` / `labels` / `capabilities` / `requires`
- **BLANK 3:** `-equals` / `==` / `:` / `-is`

<details>
<summary>Show answer</summary>

### Answer: `name`, `demands`, `-equals`

**In `challenge-21.md`:** lines **202–206**.

`vmImage` would make it a Microsoft-hosted pool and cannot appear alongside `name`. `labels` is
GitHub. `capabilities` is what the **agent** advertises; `demands` is what the **pipeline** asks for —
two ends of the same handshake, and the exam swaps them.

</details>

---

## Q39

```bash
az vmss create \
  --name contoso-agent-vmss \
  --instance-count [BLANK 1] \
  ...

# Pool settings:
#   Minimum agents: [BLANK 2]
#   Maximum agents: 10
#   Desired idle agents: 2
```

- **BLANK 1:** `0` / `1` / `2` / `10`
- **BLANK 2:** `0` / `1` / `2` / `10`

<details>
<summary>Show answer</summary>

### Answer: `0`, `0`

**In `challenge-21.md`:** lines **225** and **239**.

**Scale to zero is the whole value proposition.** No queued work, no VMs, no cost.

Note the tension with "Desired idle agents: 2" (line 241): that keeps two warm agents ready for fast
job starts. Set it above zero and you trade money for latency. The exam asks this as
"minimise cost" (idle 0) versus "minimise queue time" (idle above 0).

</details>

---

## Q40

```yaml
  deploy:
    runs-on: [self-hosted, linux]
    permissions:
      [BLANK 1]: write
      contents: read
    steps:
      - uses: [BLANK 2]
```

- **BLANK 1:** `id-token` / `packages` / `deployments` / `actions`
- **BLANK 2:** `azure/login@v2` / `docker/login-action@v3` / `actions/checkout@v4` /
  `azure/webapps-deploy@v3`

<details>
<summary>Show answer</summary>

### Answer: `id-token`, `azure/login@v2`

**In `challenge-21.md`:** lines **350–355**.

Same block as Challenge 19 Q41. It repeats deliberately — this is the pattern that has cost you marks
in every mock, and it is identical whether the runner is hosted or self-hosted.

</details>

---

## Q41

```bash
az network nsg rule create \
  --name AllowGitHub \
  --direction [BLANK 1] \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges [BLANK 2]
```

- **BLANK 1:** `Outbound` / `Inbound` / `Both` / `Any`
- **BLANK 2:** `443` / `22` / `80` / `5986`

<details>
<summary>Show answer</summary>

### Answer: `Outbound`, `443`

**In `challenge-21.md`:** lines **331** and **334**.

**The architectural point:** a runner's connection is **outbound only**. It polls GitHub or Azure
DevOps for work. You never open an inbound port — which is exactly why security teams accept
self-hosted runners inside a private network.

If an exam option proposes an **inbound** rule so the platform can "reach the runner", it is wrong.

</details>

---

# Section G — Case study

## Case study: Contoso Ltd engineering teams

### Background

Contoso has four teams with different build needs. Everything currently runs on GitHub-hosted
runners, which is slow, expensive and cannot reach internal systems.

### Current problems

- Mobile team builds iOS apps needing macOS and Xcode. macOS minutes dominate the bill
- Data engineering runs integration tests against an on-premises SQL Server behind a firewall
- Platform team builds Docker images and re-pulls base images every build
- Builds are slow because no cache persists between jobs

### Requirements

- Integration tests must reach the on-premises SQL Server
- iOS build cost must fall
- Docker builds must reuse a warm layer cache
- Lint and unit test jobs must stay zero-maintenance
- Runners must not be reachable from the public internet
- A compromised workflow must not affect the next job
- Capacity must cost nothing when no builds are running

---

## Q42

Which configuration meets the integration-test requirement?

- A. Self-hosted runners in the corporate network, labelled `on-prem`
- B. GitHub-hosted runners with a firewall allow-list
- C. A VPN gateway between GitHub and the corporate network
- D. Larger GitHub-hosted runners

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-21.md`:** lines **150–155**, line **398**.

**Why the others fail**

- **B** — allow-listing GitHub's public ranges exposes internal SQL to a shared public cloud space
- **C** — you cannot terminate a VPN on a GitHub-hosted runner. You do not control it
- **D** — more CPU does not create a network route

</details>

---

## Q43

Which option meets the iOS cost requirement?

- A. Self-hosted macOS hardware
- B. GitHub-hosted macOS runners with a longer timeout
- C. Cross-compiling iOS builds on Linux
- D. Running iOS builds less frequently

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-21.md`:** line **34** (10x multiplier), line **397** (decision matrix).

At $0.08/min versus $0.008/min, macOS minutes cost ten times Linux. Line 389 puts the break-even at
about **1,000 macOS minutes per month**.

**Why the others fail**

- **B** — a longer timeout permits *more* expensive minutes
- **C** — iOS builds require Xcode, which only runs on macOS. This is a licensing and platform
  constraint, not a technical preference
- **D** — reducing build frequency degrades the delivery process to save money. The exam never wants
  that as the answer

</details>

---

## Q44

Which **two** requirements does `--ephemeral` satisfy? (Choose two.)

- A. A compromised workflow must not affect the next job
- B. Capacity must cost nothing when idle
- C. Runners must not be reachable from the public internet
- D. Each job starts from a clean environment
- E. Docker builds must reuse a warm cache

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-21.md`:** lines **307–312**, line **41**.

One job per runner, then de-registration. Nothing carries over — which is both the security property
(A) and the cleanliness property (D).

**Why the others fail**

- **B** — that is **scaling** (VMSS at line 225 or ARC at line 287), a separate concern
- **C** — that is **networking** (`--public-ip-address ""` at line 69)
- **E** — **the important conflict.** Ephemeral destroys the local cache, which directly opposes the
  Docker cache requirement

**Contoso's real answer is a split fleet:** ephemeral runners for untrusted or public-facing work,
persistent labelled runners for the Docker cache. Note that the requirements as written *do* conflict
— and noticing a conflict rather than forcing one answer is itself an exam skill.

</details>

---

## Q45

Which option meets the "capacity must cost nothing when idle" requirement?

- A. A VMSS-backed agent pool with minimum agents set to 0
- B. Three always-on self-hosted VMs
- C. Reserved 1-year VM pricing
- D. Spot VMs running continuously

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-21.md`:** lines **225** and **239**.

```bash
--instance-count 0        # line 225
#   Minimum agents: 0     # line 239
```

**Why the others fail**

- **B** — ~$140/month each, running whether used or not (line 379)
- **C** — reserved pricing at ~$89/month (line 381) is a **discount** on always-on, not zero
- **D** — spot at ~$28/month (line 380) is cheap but still continuous, and it adds interruption risk
  mid-build

**The distinction:** reserved and spot make idle capacity *cheaper*. Only scale-to-zero makes it
*free*.

</details>

---

## Q46

The platform team's Docker builds must reuse a warm layer cache. Which runner configuration is
required?

- A. Persistent self-hosted runners with a local Docker cache
- B. Ephemeral self-hosted runners
- C. GitHub-hosted runners with `actions/cache`
- D. VMSS agents with tear-down enabled

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-21.md`:** line **40** and line **399**.

A warm Docker layer cache **is** state that survives between jobs. That requires persistence, by
definition.

**Why the others fail**

- **B** and **D** — both destroy the machine after each job, taking the cache with it
- **C** — `actions/cache` works, but line 40 notes it is a **network round-trip**. For multi-gigabyte
  Docker base images, upload and download can cost more time than rebuilding

**The trade-off to state plainly:** persistence buys cache speed and costs isolation. Restrict those
runners to trusted internal repositories via a runner group (line 128).

</details>

---

## Q47

Contoso wants the internal-network runners usable only by two specific repositories.

What should you configure?

- A. A runner group with `visibility: selected`
- B. Additional labels on the runners
- C. Branch protection on both repositories
- D. A separate GitHub organization

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-21.md`:** lines **123–131**.

```bash
gh api --method POST /orgs/contoso/actions/runner-groups \
  -f name="internal-network" \
  -f visibility="selected" \
  -F selected_repository_ids[]="<repo-id-1>" \
  -F selected_repository_ids[]="<repo-id-2>" \
  -F allows_public_repositories=false
```

**Why the others fail**

- **B** — **the trap.** Labels are **routing**, not permission. Any repository in the org can write
  `runs-on: [self-hosted, on-prem]` and its jobs will land on those runners. Labels answer "which
  runner?", groups answer "who may?"
- **C** — branch protection governs merging code
- **D** — a whole new org to scope a runner pool is disproportionate, and it fragments everything else

</details>

---

## Q48

After the migration, a lint job that previously took 40 seconds now takes 4 minutes on a self-hosted
runner, and the runner is often idle.

What is the most likely explanation, and what should Contoso do?

- A. The runner is undersized; move lint jobs back to GitHub-hosted runners
- B. The runner lacks a label; add one
- C. The runner is ephemeral and re-provisions before each job
- D. The runner group is misconfigured

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-21.md`:** line **38** (startup time), lines **307–312** (ephemeral), line **241**
(desired idle agents).

An ephemeral runner de-registers after each job, so a fresh machine must boot, install the runner and
register before work begins. For a **40-second** job that overhead dominates.

**The fixes, in order of preference:**

1. Keep a warm pool — "Desired idle agents: 2" (line 241) or `minRunners` above 0 (line 287)
2. Send trivial jobs back to hosted runners — line 400 recommends exactly that for lint and unit
   tests

**Why the others fail**

- **A** — half right for the wrong reason. Moving lint to hosted **is** sensible (line 400), but the
  cause is provisioning latency, not CPU. The exam wants the diagnosis, not just an action
- **B** — a missing label means the job never runs at all, not that it runs slowly
- **D** — a group problem produces a permission error

**The general lesson:** self-hosting helps **long** builds with **big** caches. Short jobs are pure
overhead. Match the runner to the job, not to the org.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **Hosted advantages listed as self-hosted** | Q17 | Learn the comparison table both directions |
| **Firewall allow-list "fixes" connectivity** | Q25, Q42 | Move compute inside the network; never open it to a cloud range |
| **Labels confused with runner groups** | Q8, Q23, Q47 | Label = which runner. Group = who may use it |
| **Ephemeral vs "wipes the workspace"** | Q2, Q28 | The whole runner de-registers |
| **Ephemeral kills the cache** | Q44, Q46 | Cache needs persistence. The two requirements conflict |
| **`pool: name` vs `pool: vmImage`** | Q13, Q38 | Mutually exclusive. Self-hosted vs Microsoft-hosted |
| **Capabilities vs demands** | Q3, Q38 | Agent advertises capabilities; pipeline declares demands |
| **403 read as a network problem** | Q12 | 403 = reached and refused. Timeout = never reached |
| **VMSS offered for GitHub Actions** | Q30 | VMSS is Azure DevOps. ARC is GitHub |
| **Reserved or spot instead of scale-to-zero** | Q45 | Discounts are not zero |
| **Self-hosted assumed always cheaper** | Q15 | Maintenance is ~$500/month, more than the VM |
| **Machine identity vs workflow identity** | Q21 | OIDC identifies the workflow, not the VM |

---

# The things to memorise

Not much YAML in this challenge — memorise the **numbers** and the **pairings**.

```text
# Cost multipliers (lines 367-370)
Linux    $0.008/min      1x
Windows  $0.016/min      2x
macOS    $0.08/min      10x        <- most tested

# Azure DevOps parallel jobs (line 47)
1 free Microsoft-hosted parallel job (1,800 min/month)
$40/month  extra Microsoft-hosted parallel job
$15/month  extra self-hosted parallel job

# Maintenance (line 382)
~$500/month engineer time  - more than the VM itself (~$140/month)
```

```text
# Platform pairings
Problem            GitHub Actions        Azure DevOps
-----------------  --------------------  ---------------------
Which runner?      labels in runs-on     pool + demands
Who may use it?    runner group          pool permissions
Elastic scaling    ARC (Kubernetes)      VMSS agent pool
One-job lifetime   --ephemeral           tear down after use
Registration       registration token    PAT (registration only)
```

```bash
# Runner registration (lines 81-92)
./config.sh --url <url> --token <TOKEN> --labels linux,docker,on-prem \
            --runnergroup contoso-internal --ephemeral --replace
sudo ./svc.sh install && sudo ./svc.sh start
```

```yaml
# Job targeting - array means ALL labels (line 150)
    runs-on: [self-hosted, linux, on-prem]

# Azure DevOps equivalent (lines 202-207)
pool:
  name: contoso-linux-pool
  demands:
    - docker
    - Agent.OS -equals Linux
```

**The four reasons self-hosted wins** (lines 386–390): private network access, very high Linux
volume, macOS above ~1,000 min/month, custom hardware or persistent caches. If a question shows none
of these, the answer is hosted.

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 21 is exam-ready. Move to Challenge 22 |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Re-read Tasks 1 and 7 (the comparison and cost tables), then retake |
| Below 30 | Redo the challenge, focusing on the decision matrix at lines 395–401 |

Record your result in `AZ-400-Learning-Log.md` under Challenge 21, and note **which trap** caught you.

:::tip The one thing

This challenge is judgement, not syntax. For every question, find the **deciding constraint** —
private network, macOS cost, warm cache, zero idle cost, clean environment — and let it pick the
answer. That habit is worth more than any single fact here.

:::
