---
sidebar_position: 94
title: "Challenge 28: exam questions"
---

# Challenge 28 — AZ-400 exam questions

**48 questions** built only from what Challenge 28 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-28.md`**.

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

:::tip Three registries, two orchestrators, one pipeline

This challenge covers ACR **and** GHCR, deploying to Container Apps **and** AKS. The exam asks you to
pick the right one for a stated constraint, so notice *which* platform each command belongs to as you
read.

:::

---

# Section A — Single answer

---

## Q1

A GitHub Actions workflow fails with `unauthorized: authentication required` when pushing to ACR. The
service principal can already pull images.

What should you do?

- A. Enable the ACR admin account
- B. Assign the `AcrPush` role to the service principal
- C. Assign the `Contributor` role on the resource group
- D. Regenerate the ACR access keys

### Answer: B

**In `challenge-28.md`:** Break & fix Exercise 1, lines **606–634**.

```bash
az role assignment create \
  --assignee $SP_ID \
  --role AcrPush \
  --scope ".../registries/acrcontosoprod"
```

> *"The service principal in the AZURE_CREDENTIALS secret only has `AcrPull` role, not `AcrPush`."*

**The ACR data-plane roles, in order:**

| Role | Can |
|---|---|
| `AcrPull` | Pull images |
| `AcrPush` | Pull **and** push |
| `AcrDelete` | Delete images |
| `AcrImageSigner` | Sign images with content trust |

**Why the others fail**

- **A** — line 141 sets `--admin-enabled false` deliberately. The admin account is a shared username
  and password with full access, which is exactly what identity-based auth replaces
- **C** — Contributor is control-plane. Same split as Challenge 27's App Configuration: it can delete
  the registry but is not a data-plane push role
- **D** — keys belong to the admin account, which is disabled

---

## Q2

Which `build-push-action` setting builds the image on a pull request but only pushes on merge?

- A. `push: true`
- B. `push: ${{ github.event_name != 'pull_request' }}`
- C. `if: github.ref == 'refs/heads/main'`
- D. `load: true`

### Answer: B

**In `challenge-28.md`:** line **115**.

```yaml
      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: src/ProductCatalog
          push: ${{ github.event_name != 'pull_request' }}
```

**Why this pattern is right:** a pull request should prove the image **builds** — that is real
validation. It should not publish an artifact, because the code has not been reviewed or merged yet.
One expression gives you both.

**Why the others fail**

- **A** — pushes an unreviewed image from every PR, including from forks
- **C** — a job-level `if` would skip the **whole job**, so PRs would not build at all and you would
  lose the validation
- **D** — `load: true` loads the image into the local Docker daemon for testing. Useful, but it does
  not control pushing

---

## Q3

A Container App revision never becomes active, and logs show repeated restarts. The Dockerfile
exposes 8080 but ingress targets port 80.

What is the fix?

- A. Change the Dockerfile to expose port 80
- B. Set the ingress target port to 8080 and `ASPNETCORE_URLS` to listen on 8080
- C. Increase the minimum replica count
- D. Restart the Container Apps environment

### Answer: B

**In `challenge-28.md`:** Break & fix Exercise 2, lines **638–674**.

```bash
az containerapp update --set-env-vars "ASPNETCORE_URLS=http://+:8080"
az containerapp ingress update --target-port 8080
```

**Three things must agree** — and the exam likes breaking exactly one of them:

1. The application **listens** on the port (`ASPNETCORE_URLS`)
2. The Dockerfile **documents** it (`EXPOSE 8080`, line 45)
3. The platform **routes** to it (`--target-port 8080`, line 245)

**Why the others fail**

- **A** — possible, but it means changing the image and rebuilding to fix a routing setting. And 8080
  is deliberate: it is above 1024, so a **non-root** user can bind it (lines 48–49)
- **C** — more replicas that all fail health checks
- **D** — the environment is fine; the app's port is wrong

**Note `EXPOSE` alone does nothing.** It is documentation. The container listens where the process
binds.

---

## Q4

Which Kubernetes rolling update setting guarantees no reduction in capacity during a deployment?

- A. `maxSurge: 1, maxUnavailable: 0`
- B. `maxSurge: 0, maxUnavailable: 1`
- C. `maxSurge: 1, maxUnavailable: 1`
- D. `replicas: 3`

### Answer: A

**In `challenge-28.md`:** lines **353–357**.

```yaml
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

**`maxUnavailable: 0` is the guarantee.** Kubernetes must add a new pod (surge to 4) and wait for it
to become **ready** before removing an old one. Capacity never drops below 3.

**Why the others fail**

- **B** — the opposite trade-off: no extra pods, so one is removed first and you run at 2 of 3 during
  the roll. Cheaper, briefly degraded
- **C** — allows both at once. Faster, and capacity can still dip
- **D** — the desired count, not the update behaviour

**The cost of A:** you need headroom for one extra pod. This is the same trade-off as Challenge 26's
`maxBatchInstancePercent` and Challenge 25's 2x blue-green cost — availability is bought with spare
capacity.

---

## Q5

What is the difference between a Kubernetes liveness probe and a readiness probe?

- A. Liveness restarts an unhealthy container; readiness removes a pod from the Service endpoints
- B. Liveness runs once at startup; readiness runs continuously
- C. Liveness checks the node; readiness checks the container
- D. They are interchangeable

### Answer: A

**In `challenge-28.md`:** lines **375–386**.

```yaml
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

**The two questions they ask:**

| Probe | Asks | On failure |
|---|---|---|
| Liveness | *Is this process wedged?* | **Restart the container** |
| Readiness | *Can it serve right now?* | **Remove it from the Service** — no restart |

Readiness is what makes `maxUnavailable: 0` meaningful: a new pod only counts as available once it
passes readiness, so the roll waits for it.

**Getting them backwards is dangerous.** If your liveness probe checks a database connection, a brief
database outage restarts every pod in the cluster at once. Liveness should check *the process*;
readiness checks *dependencies*.

**Why the others fail** — B, C and D all misstate the mechanism. Both probes run continuously, both
target the container.

---

## Q6

Which Azure Container Apps command shifts all traffic to a previous revision?

- A. `az containerapp update --image <previous>`
- B. `az containerapp ingress traffic set --revision-weight "<revision>=100"`
- C. `az containerapp restart`
- D. `az containerapp revision restart`

### Answer: B

**In `challenge-28.md`:** lines **316–328**.

```bash
          PREVIOUS_REVISION=$(az containerapp revision list \
            --query "[-2].name" -o tsv)

          az containerapp ingress traffic set \
            --revision-weight "$PREVIOUS_REVISION=100"
```

**Container Apps keeps every revision.** Rolling back is a **traffic weight change**, not a
deployment — the old revision is still there, so the switch is immediate.

Note `[-2]` — the second from the end of the list, i.e. the revision before the current one.

**Why the others fail**

- **A** — works, but it creates a **new revision** with the old image. Slower, and it clutters the
  revision history
- **C** and **D** — restarting reruns the same broken image

**This is the same shape as an App Service slot swap:** the previous version survives, so reverting is
a routing decision. And the same mechanism gives you canary — `--revision-weight` with a split.

---

## Q7

Which Azure Pipelines task deploys manifests to AKS and substitutes the image tag?

- A. `Kubernetes@1`
- B. `KubernetesManifest@1`
- C. `Docker@2`
- D. `HelmDeploy@0`

### Answer: B

**In `challenge-28.md`:** lines **463–475**.

```yaml
                - task: KubernetesManifest@1
                  inputs:
                    action: 'deploy'
                    manifests: |
                      k8s/product-catalog/deployment.yml
                    containers: |
                      $(acrName).azurecr.io/$(imageRepository):$(tag)
```

The `containers:` input is the substitution: the manifest at line 365 says `:latest`, and this
replaces it with the built tag at deploy time — so the manifest never needs editing per build.

**Why the others fail**

- **A** — `Kubernetes@1` runs raw kubectl commands. The challenge uses it at line 477 for
  `rollout status`, which is a different job
- **C** — `Docker@2` builds and pushes images (line 441)
- **D** — deploys Helm charts, not plain manifests

**`KubernetesManifest@1` also does** `bake`, `promote`, `reject` and `createSecret`, and supports
`canary` deployment strategies. The `action` input is what selects behaviour.

---

## Q8

Which command verifies that an AKS deployment finished rolling out?

- A. `kubectl get pods`
- B. `kubectl rollout status deployment/product-catalog-api --timeout=300s`
- C. `kubectl describe deployment`
- D. `kubectl logs deployment/product-catalog-api`

### Answer: B

**In `challenge-28.md`:** lines **477–486**.

```yaml
                - task: Kubernetes@1
                  inputs:
                    command: 'rollout'
                    arguments: 'status deployment/product-catalog-api --timeout=300s'
```

**`rollout status` blocks until the rollout completes or the timeout expires**, and it exits non-zero
on failure — which is what fails the pipeline stage.

**Why the others fail — all three return immediately and always succeed**

- **A** — prints pods, exit code 0 even if every pod is crash-looping
- **C** — prints detail, exit code 0 regardless
- **D** — prints logs, exit code 0 regardless

**The general lesson:** a verification step must **fail** when the thing it verifies fails. A command
that only prints information verifies nothing, no matter how useful the output is to a human.

---

## Q9

Which ACR SKU supports geo-replication and content trust?

- A. Basic
- B. Standard
- C. Premium
- D. All SKUs

### Answer: C

**In `challenge-28.md`:** lines **137–146**.

```bash
az acr create --sku Premium --admin-enabled false
az acr config content-trust update --registry $ACR_NAME --status enabled
```

**Premium-only features worth knowing:** geo-replication, content trust (image signing), private
endpoints, customer-managed keys, and higher throughput and storage.

**Why the others fail** — Basic and Standard differ mainly in storage and throughput; neither offers
these.

**Note `--admin-enabled false`.** The admin account is a single shared credential with full registry
access. Disabling it forces identity-based authentication, which is the pattern the whole challenge
follows.

---

## Q10

Which ACR retention setting deletes untagged manifests after seven days?

- A. `az acr config retention update --status enabled --days 7 --type UntaggedManifests`
- B. `az acr purge --ago 7d`
- C. `az acr repository delete --untagged`
- D. `az acr config content-trust update --days 7`

### Answer: A

**In `challenge-28.md`:** lines **170–174**.

```bash
az acr config retention update \
  --registry $ACR_NAME \
  --status enabled \
  --days 7 \
  --type UntaggedManifests
```

**Untagged manifests are the invisible cost.** Every time you push a new image with an existing tag,
the old manifest loses its tag but keeps its layers — and you keep paying for them. This policy
collects that garbage automatically.

**Why the others fail**

- **B** — `acr purge` is real (line 179) but runs as a **task**, and it handles *tagged* images by age
  and count. Different tool, different target
- **C** — deletes a repository, not a policy
- **D** — content trust is image signing

**The two together (lines 170–180)** are a complete lifecycle: retention for untagged manifests, purge
keeping the last 10 tagged versions over 30 days old.

---

## Q11

Which Trivy setting makes the workflow fail when a CRITICAL vulnerability is found?

- A. `severity: 'CRITICAL,HIGH'`
- B. `exit-code: '1'`
- C. `format: 'sarif'`
- D. `ignore-unfixed: true`

### Answer: B

**In `challenge-28.md`:** lines **502–509**.

```yaml
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: '...'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
```

**`severity` decides what is *reported*; `exit-code` decides whether it *blocks*.** Without
`exit-code: '1'`, Trivy prints findings and the step passes — a scan that is a report rather than a
gate.

**Why the others fail**

- **A** — filters which severities appear. On its own it blocks nothing
- **C** — output format for the Security tab
- **D** — real and useful: it hides vulnerabilities with no available fix. That is a policy choice,
  not the blocking mechanism

---

## Q12

Where do Trivy SARIF results appear, and which step sends them there?

- A. Pipeline artifacts, via `upload-artifact`
- B. The GitHub Security tab, via `github/codeql-action/upload-sarif`
- C. Azure Monitor, via `azure/login`
- D. The workflow summary, automatically

### Answer: B

**In `challenge-28.md`:** lines **511–515**.

```yaml
      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
```

**SARIF** is the standard format for static analysis results, which is why a CodeQL action can upload
findings from a completely different tool.

**Note `if: always()`.** The scan step exits 1 on a finding, so without this the upload is skipped
precisely when there are results worth seeing. Same pattern as `PublishTestResults` with
`condition: always()` in Challenge 20.

**Why the others fail** — A stores a file nobody opens; C and D are wrong destinations.

---

## Q13

Which **two** actions are needed for a multi-architecture build? (Pick the single best answer.)

- A. `docker/setup-qemu-action` and `docker/setup-buildx-action`
- B. `docker/setup-buildx-action` only
- C. A self-hosted ARM64 runner
- D. `docker/metadata-action` with a platform tag

### Answer: A

**In `challenge-28.md`:** lines **562–580**.

```yaml
      - name: Set up QEMU (for cross-platform builds)
        uses: docker/setup-qemu-action@v3
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      ...
        with:
          platforms: linux/amd64,linux/arm64
```

**QEMU is the emulator** that lets an amd64 runner execute arm64 instructions. **Buildx is the
builder** that produces a multi-platform manifest. You need both.

**Why the others fail**

- **B** — buildx alone cannot execute arm64 binaries on an amd64 host
- **C** — a real alternative and much faster than emulation, but it requires managing runners
  (Challenge 21). Emulation needs no infrastructure
- **D** — metadata generates tags, not platforms

**The trade-off:** emulated arm64 builds are slow — often several times slower than native. For
occasional builds that is fine; for frequent ones, native ARM runners win.

---

## Q14

Which Container Apps setting lets the app pull from ACR without a stored credential?

- A. `--registry-identity system`
- B. `--registry-username` and `--registry-password`
- C. `--admin-enabled true` on the registry
- D. An image pull secret in the environment

### Answer: A

**In `challenge-28.md`:** lines **243–244**.

```bash
az containerapp create \
  --registry-server acrcontosoprod.azurecr.io \
  --registry-identity system
```

The Container App's **system-assigned managed identity** authenticates to ACR. Grant it `AcrPull` and
nothing is stored anywhere.

**Why the others fail**

- **B** — stored credentials that must be rotated
- **C** — the admin account, deliberately disabled at line 141
- **D** — the Kubernetes pattern, not Container Apps

**Same reasoning as your OIDC block and Challenge 27's Data Reader role:** an identity beats a stored
secret every time, and "no stored credential" in a question is the phrase that selects it.

---

## Q15

Why does the Dockerfile create and switch to a non-root user?

- A. To reduce image size
- B. To limit the impact if the container is compromised
- C. To allow binding to port 80
- D. To enable multi-architecture builds

### Answer: B

**In `challenge-28.md`:** lines **47–49**.

```dockerfile
# Run as non-root user
RUN adduser --disabled-password --gecos "" appuser
USER appuser
```

A container escape or an in-container exploit inherits the process's privileges. Root inside the
container means root-level capability against the container's filesystem and, with certain
misconfigurations, a path toward the host.

**Why the others fail**

- **A** — adding a user slightly *increases* size
- **C** — **backwards, and this is the point.** Ports below 1024 require root. Running as non-root is
  precisely why the app uses **8080** rather than 80 (line 45)
- **D** — unrelated

**The chain to see:** non-root → cannot bind 80 → use 8080 → ingress must target 8080 (line 245). Get
the last step wrong and you get Break & fix Exercise 2.

---

## Q16

Which registry does `${{ secrets.GITHUB_TOKEN }}` authenticate to without extra configuration?

- A. Azure Container Registry
- B. GitHub Container Registry
- C. Docker Hub
- D. Any OCI registry

### Answer: B

**In `challenge-28.md`:** lines **192–203**.

```yaml
    permissions:
      contents: read
      packages: write
    steps:
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```

`GITHUB_TOKEN` is a GitHub credential, so it works with GitHub's own registry and needs
`packages: write` (Challenge 19 Q3).

**Why the others fail** — ACR needs an Azure identity (`azure/login` then `az acr login`, lines
91–97); Docker Hub and other registries need their own credentials.

**Choosing between them:** GHCR is free, needs no extra identity, and lives beside the code. ACR
supports private endpoints, geo-replication, content trust and Defender integration. Contoso uses ACR
for production and GHCR for convenience — which is why both appear here.

---

# Section B — Multiple answer

---

## Q17

Which **three** permissions does the container build job declare? (Choose three.)

- A. `contents: read`
- B. `packages: write`
- C. `security-events: write`
- D. `id-token: write`
- E. `deployments: write`
- F. `actions: write`

### Answer: A, B, C

**In `challenge-28.md`:** lines **80–83**.

```yaml
    permissions:
      contents: read          # check out the repository
      packages: write         # push to a registry
      security-events: write  # upload SARIF to the Security tab
```

**Each permission maps to one thing the job actually does.** That is least privilege in practice —
you can read the job's capabilities off its permission block.

**Why the others fail**

- **D** — `id-token: write` would be required if this job used **OIDC**. It uses
  `secrets.AZURE_CREDENTIALS` instead (line 94), which is the less secure choice. On an exam question
  that says "no stored credential", you would add `id-token: write` and remove that secret
- **E** and **F** — not needed here

---

## Q18

Which **three** tag types does the metadata action generate for the ACR image? (Choose three.)

- A. `type=semver,pattern={{version}}`
- B. `type=sha,prefix=,format=short`
- C. `type=raw,value=latest,enable={{is_default_branch}}`
- D. `type=schedule,pattern=nightly`
- E. `type=ref,event=pr`

### Answer: A, B, C

**In `challenge-28.md`:** lines **105–109**.

```yaml
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=,format=short
            type=raw,value=latest,enable={{is_default_branch}}
```

**Three tags, three purposes** — and a good tagging strategy has all three:

| Tag | Answers |
|---|---|
| `1.2.3` semver | Which **release** is this? |
| `a1b2c3d` short SHA | Which **commit** produced it? |
| `latest` | What should a casual pull get? |

Note `{{major}}.{{minor}}` too: a `1.2` tag that follows every patch, so consumers can track a minor
line.

**Why the others fail** — `type=schedule` only applies to scheduled runs; `type=ref,event=pr` tags PR
builds, which this workflow does not push (line 115).

---

## Q19

Which **two** guarantee an AKS deployment never drops below its desired capacity? (Choose two.)

- A. `maxUnavailable: 0`
- B. A readiness probe on the container
- C. `maxSurge: 0`
- D. `replicas: 3`
- E. A liveness probe on the container

### Answer: A, B

**In `challenge-28.md`:** lines **356–357** and **381–386**.

**They only work together.** `maxUnavailable: 0` tells Kubernetes to wait for a new pod to become
**available** before removing an old one — and "available" means **readiness probe passing**. Without
a readiness probe a pod is considered ready the moment it starts, so Kubernetes removes an old pod
while the new one is still initialising, and capacity really does dip.

**Why the others fail**

- **C** — `maxSurge: 0` forbids extra pods, so with `maxUnavailable: 0` nothing could ever roll
- **D** — the target count, not the update behaviour
- **E** — liveness restarts a wedged container. It plays no part in rollout accounting

---

## Q20

Which **two** are Premium-only ACR features used in this challenge? (Choose two.)

- A. Content trust (image signing)
- B. Geo-replication
- C. Retention policies for untagged manifests
- D. Webhooks
- E. The admin account

### Answer: A, B

**In `challenge-28.md`:** lines **136–146**.

```bash
# "Create ACR with Premium SKU (supports geo-replication, content trust)"
az acr create --sku Premium --admin-enabled false
az acr config content-trust update --status enabled
```

**Why the others fail**

- **C** — retention is available below Premium
- **D** — webhooks work on all SKUs
- **E** — available on all SKUs and deliberately **disabled** here

**What content trust buys you:** images are signed, and clients can be configured to refuse unsigned
images. That is supply-chain integrity — the same concern as artifact attestation in Challenge 19's
gap list.

---

## Q21

Which **two** stop a vulnerable image reaching production? (Choose two.)

- A. Trivy with `exit-code: '1'` in the build workflow
- B. Microsoft Defender for Containers on the registry
- C. Uploading SARIF to the Security tab
- D. `severity: 'CRITICAL,HIGH'` alone
- E. `.trivyignore` listing known CVEs

### Answer: A, B

**In `challenge-28.md`:** lines **502–509** (A) and **518–530** (B).

```bash
az security pricing create --name Containers --tier Standard
```

**Two layers at two moments:** Trivy blocks at **build time** so a bad image never reaches the
registry; Defender scans images **already in the registry**, catching vulnerabilities disclosed after
the build.

That second layer matters more than it looks — an image built clean in January can be vulnerable by
March without a single line changing.

**Why the others fail**

- **C** — reporting. Valuable for triage, blocks nothing
- **D** — filters what is reported. Without `exit-code: '1'` the step passes
- **E** — the opposite: it **allows** listed CVEs through

---

## Q22

Which **two** are true about Azure Container Apps revisions? (Choose two.)

- A. Traffic can be split across revisions by weight
- B. Rolling back is a traffic weight change, not a redeployment
- C. Only one revision can exist at a time
- D. A revision must be deleted before a new one is created
- E. Revisions are only available in the Premium tier

### Answer: A, B

**In `challenge-28.md`:** lines **320–328**.

```bash
az containerapp ingress traffic set --revision-weight "$PREVIOUS_REVISION=100"
```

**Revisions give you canary and instant rollback with one mechanism.** Split 90/10 to canary; set the
previous revision to 100 to roll back. No image rebuild, no redeploy.

**Why the others fail**

- **C** and **D** — multiple revisions coexist by design. Line 320 queries the list precisely because
  history is retained
- **E** — no such tier restriction

**Compare the three rollback mechanisms you have now seen:** App Service swaps **slots**, Container
Apps shifts **revision weights**, Kubernetes rolls back a **deployment**. All three keep the previous
version alive, which is what makes reversal fast.

---

## Q23

Which **two** correctly describe `KubernetesManifest@1` with `action: 'deploy'`? (Choose two.)

- A. It applies the listed manifests to the cluster
- B. The `containers` input substitutes the image reference in the manifests
- C. It waits for the rollout to complete and fails on timeout
- D. It builds the container image before deploying
- E. It requires a kubeconfig file in the repository

### Answer: A, B

**In `challenge-28.md`:** lines **463–475**.

```yaml
                    action: 'deploy'
                    manifests: |
                      k8s/product-catalog/deployment.yml
                    containers: |
                      $(acrName).azurecr.io/$(imageRepository):$(tag)
```

**Why B matters:** the manifest hardcodes `:latest` (line 365). Substitution replaces it with the
build's real tag, so the manifest is a **template** rather than something a pipeline must rewrite.

**Why the others fail**

- **C** — **the important one.** The deploy task does not verify the rollout, which is exactly why the
  challenge adds a separate `rollout status` step at lines 477–486. Deploying and verifying are two
  steps
- **D** — `Docker@2` does that (line 441)
- **E** — auth comes from the Azure service connection (`connectionType: 'azureResourceManager'`)

---

# Section C — Repeated scenario

**Scenario:** Contoso must deploy the Product Catalog API to Azure Container Apps. No credential may
be stored for registry access, a vulnerable image must never reach production, and a bad deployment
must be reversible without rebuilding.

---

## Q24

**Proposed solution:** Create the Container App with `--registry-identity system` and grant it
`AcrPull`. Run Trivy with `exit-code: '1'` before pushing. Roll back with
`az containerapp ingress traffic set --revision-weight "<previous>=100"`.

Does this meet the goal? **Yes**

### Answer: Yes

Each requirement maps to one mechanism:

| Requirement | Mechanism |
|---|---|
| No stored credential | `--registry-identity system` + `AcrPull` (lines 243–244) |
| No vulnerable image | Trivy with `exit-code: '1'` before push (lines 502–509) |
| Reversible without rebuild | Revision weight change (lines 325–328) |

The ordering matters too: scanning **before** pushing means a vulnerable image never enters the
registry at all, so it cannot be deployed by accident later.

---

## Q25

**Proposed solution:** Enable the ACR admin account and store the username and password as GitHub
secrets. Run Trivy in report-only mode and review findings weekly. Roll back by rebuilding the
previous commit.

Does this meet the goal? **No**

### Answer: No

**All three requirements fail.**

**Stored credential:** the admin account is a shared username and password — exactly what
`--admin-enabled false` (line 141) exists to prevent. It is also a *single* credential for the whole
registry, so it cannot be scoped or attributed.

**Vulnerable images:** report-only means no `exit-code: '1'`. The image is pushed regardless, and a
weekly review is not a gate.

**Rollback:** rebuilding takes minutes and requires the build to still succeed. The previous revision
is sitting right there, ready.

**Each wrong choice is the "easy" one**, which is what makes this a realistic distractor. Convenience
in all three places, correctness in none.

---

## Q26

**Proposed solution:** Use `--registry-identity system` with `AcrPull`, run Trivy with
`exit-code: '1'`, and roll back with `az containerapp update --image <previous-tag>`.

Does this meet the goal? **No**

### Answer: No

The first two requirements are met. The third is not — narrowly, and worth understanding.

`az containerapp update --image <previous-tag>` **creates a new revision** pointing at the old image.
No rebuild is needed, so it satisfies the letter of "without rebuilding" — but it is a **deployment**,
with a cold start while the new revision provisions, and it pollutes the revision history with
duplicates.

The previous revision already exists and is warm. Shifting traffic to it is immediate.

**The distinction the exam is testing:** *creating a new revision with old content* is not the same as
*routing back to the revision you still have*. The same distinction separates a slot swap from a
redeploy.

---

# Section D — Yes/No statement grid

---

## Q27 — registries and authentication

| # | Statement | Answer |
|---|---|---|
| 1 | `AcrPull` allows pushing images | **No** |
| 2 | `--admin-enabled false` forces identity-based authentication | **Yes** |
| 3 | `secrets.GITHUB_TOKEN` can authenticate to ACR | **No** |
| 4 | Content trust requires the Premium SKU | **Yes** |

**In `challenge-28.md`:** lines **622–633**, **141**, **91–97**, **136–146**.

Row 1 is Break & fix Exercise 1 stated as a fact. The roles are strictly ordered: `AcrPull` reads,
`AcrPush` reads and writes.

Row 3: `GITHUB_TOKEN` is a GitHub credential. ACR needs an Azure identity, which is why the workflow
runs `azure/login` and then `az acr login`.

---

## Q28 — Kubernetes deployment

| # | Statement | Answer |
|---|---|---|
| 1 | `maxUnavailable: 0` prevents capacity dropping during a roll | **Yes** |
| 2 | A liveness probe failure removes the pod from the Service | **No** |
| 3 | A readiness probe failure restarts the container | **No** |
| 4 | `rollout status` fails the pipeline when the rollout does not complete | **Yes** |

**In `challenge-28.md`:** lines **356–357**, **375–386**, **486**.

**Rows 2 and 3 are the probes swapped**, and it is the most common Kubernetes mistake on this exam:

- **Liveness fails → restart the container**
- **Readiness fails → remove from Service endpoints, no restart**

Practical consequence: a liveness probe that checks a database means a database blip restarts every
pod simultaneously, turning a partial outage into a total one. Check dependencies in **readiness**.

---

## Q29 — Container Apps

| # | Statement | Answer |
|---|---|---|
| 1 | Multiple revisions can receive traffic simultaneously | **Yes** |
| 2 | `--registry-identity system` avoids storing registry credentials | **Yes** |
| 3 | The ingress target port must match the port the app listens on | **Yes** |
| 4 | Rolling back requires rebuilding the image | **No** |

**In `challenge-28.md`:** lines **325–328**, **244**, **661–674**, **320–328**.

Row 3 is Break & fix Exercise 2, and it needs three things to agree: `ASPNETCORE_URLS`, `EXPOSE`, and
`--target-port`. `EXPOSE` alone is only documentation — it does not make anything listen.

Row 1 is what enables canary on Container Apps: weights across revisions, per request, no DNS
involved.

---

## Q30 — scanning and supply chain

| # | Statement | Answer |
|---|---|---|
| 1 | `severity` alone causes the build to fail on findings | **No** |
| 2 | `exit-code: '1'` makes Trivy block the pipeline | **Yes** |
| 3 | SARIF results can be uploaded to the GitHub Security tab | **Yes** |
| 4 | Defender for Containers scans images already in the registry | **Yes** |

**In `challenge-28.md`:** lines **508–509**, **511–515**, **518–524**.

Row 1 is the difference between a **report** and a **gate**, and it recurs across these challenges —
the same distinction as `continue-on-error` on a test step.

Row 4 is why both layers exist. Build-time scanning uses the vulnerability database *as it was on
build day*. Registry scanning re-evaluates continuously, so an image that was clean in January is
flagged when a CVE is disclosed in March.

---

# Section E — Drag and drop

---

## Q31

Arrange the container build and push workflow steps in order.

**Items:** Build and push the image · Log in to the registry · Check out the repository · Generate
image metadata · Set up Docker Buildx

### Answer

1. Check out the repository — line **86**
2. Set up Docker Buildx — line **88**
3. Log in to the registry — lines **91–97**
4. Generate image metadata — lines **99–109**
5. Build and push the image — lines **111–119**

**Two hard rules, same as Challenge 19 Q34:** login before push, and metadata before whatever consumes
`steps.meta.outputs.tags`.

Buildx must precede the build because `cache-from: type=gha` (line 118) needs the buildx driver — the
default Docker builder cannot use that cache backend.

**Note ACR login is two steps** (lines 91–97): `azure/login` to get an Azure identity, then
`az acr login` to exchange it for a registry token. GHCR needs only one (line 199).

---

## Q32

Match each rollback mechanism to its platform.

| Mechanism | Platform |
|---|---|
| Swap the staging and production slots | **App Service** |
| Set a previous revision to 100% traffic | **Container Apps** |
| `kubectl rollout undo` on the deployment | **AKS** |
| Set a Traffic Manager endpoint weight to 0 | **Traffic Manager (DNS)** |
| Disable a feature flag | **App Configuration** |

**In `challenge-28.md`:** lines **325–328**; the others are Challenges 25–27.

**The common property:** every one keeps the previous version **alive**, so reversal is a routing or
configuration decision rather than a deployment. That is why they are fast.

**And the one that is different:** disabling a feature flag reverts *behaviour* without touching the
deployed artifact at all — which is why Challenge 25's table rates it instant at 1x cost.

---

## Q33

Match each image tag to what it identifies.

| Tag | Identifies |
|---|---|
| `1.2.3` | The **release version** (semver) |
| `1.2` | The **minor line**, following every patch |
| `a1b2c3d` | The exact **commit** |
| `latest` | The most recent build **on the default branch** |

**In `challenge-28.md`:** lines **105–109**.

```yaml
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=,format=short
            type=raw,value=latest,enable={{is_default_branch}}
```

**Which one should a deployment reference?** The **SHA**. It is immutable and traceable to a commit.
`latest` moves under you, and even a semver tag can be re-pushed. That is why the Container Apps
deploy workflow derives its tag from `workflow_run.head_sha` (lines 284–295).

---

## Q34

Arrange the AKS deployment stages in order.

**Items:** Verify rollout status · Build and push the image to ACR · Deploy the manifests ·
Substitute the image tag into the manifests

### Answer

1. Build and push the image to ACR — lines **441–451**
2. Substitute the image tag — the `containers:` input, lines **474–475**
3. Deploy the manifests — `action: 'deploy'`, lines **463–473**
4. Verify rollout status — lines **477–486**

Steps 2 and 3 are one task; the substitution happens as part of the deploy.

**Step 4 is separate and non-optional.** `KubernetesManifest@1` returns once the manifests are
**applied**, not once the pods are **running**. Without `rollout status --timeout=300s`, a deployment
whose pods crash-loop forever still reports success.

---

## Q35

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| `unauthorized: authentication required` pushing to ACR | **Service principal has `AcrPull`, not `AcrPush`** |
| A Container App revision restarts repeatedly | **Ingress target port does not match the listening port** |
| The build fails on a CVE with no available fix | **Trivy `exit-code: '1'` with no `.trivyignore` entry** |
| The pipeline reports success but pods crash-loop | **No `rollout status` verification step** |
| An arm64 build fails on a GitHub-hosted runner | **QEMU not set up** |

**In `challenge-28.md`:** lines **622**, **661**, **687–706**, **477–486**, **562**.

**Read the failure's layer:** authorisation, networking, policy, verification, tooling. Naming the
layer usually names the fix.

---

# Section F — Hot area

---

## Q36

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
...
FROM mcr.microsoft.com/dotnet/[BLANK 1]:8.0 AS runtime
WORKDIR /app
EXPOSE [BLANK 2]

RUN adduser --disabled-password --gecos "" appuser
[BLANK 3] appuser
```

- **BLANK 1:** `aspnet` / `sdk` / `runtime-deps` / `runtime`
- **BLANK 2:** `8080` / `80` / `443` / `5000`
- **BLANK 3:** `USER` / `RUN AS` / `SETUSER` / `ENV USER`

### Answer: `aspnet`, `8080`, `USER`

**In `challenge-28.md`:** lines **43–49**.

`sdk` is the build image and is far larger — shipping it would include compilers in production.
`runtime` runs console apps; `aspnet` adds the ASP.NET Core runtime. `runtime-deps` is for
self-contained apps.

**8080 rather than 80 is a consequence of `USER appuser`:** ports below 1024 require root.

---

## Q37

```yaml
      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: src/ProductCatalog
          push: [BLANK 1]
          cache-from: type=[BLANK 2]
          cache-to: type=[BLANK 2],mode=[BLANK 3]
```

- **BLANK 1:** `${{ github.event_name != 'pull_request' }}` / `true` / `false` /
  `${{ github.ref == 'refs/heads/main' }}`
- **BLANK 2:** `gha` / `registry` / `local` / `inline`
- **BLANK 3:** `max` / `min` / `all` / `full`

### Answer: `${{ github.event_name != 'pull_request' }}`, `gha`, `max`

**In `challenge-28.md`:** lines **115–119**.

`mode=max` caches **intermediate** layers, which matters for a multi-stage build where the expensive
work happens in the build stage.

The `github.ref` option is close but wrong: it would also block pushes from release branches, and it
does not express the PR intent as directly.

---

## Q38

```bash
az acr create --sku [BLANK 1] --admin-enabled [BLANK 2]

az acr config retention update \
  --status enabled --days 7 --type [BLANK 3]
```

- **BLANK 1:** `Premium` / `Basic` / `Standard` / `Free`
- **BLANK 2:** `false` / `true`
- **BLANK 3:** `UntaggedManifests` / `TaggedImages` / `AllManifests` / `Repositories`

### Answer: `Premium`, `false`, `UntaggedManifests`

**In `challenge-28.md`:** lines **137–174**.

Premium is required for content trust and geo-replication. `--admin-enabled false` forces
identity-based auth. `UntaggedManifests` is the only valid retention type — those are the orphaned
layers you keep paying for after a tag moves.

---

## Q39

```yaml
  strategy:
    type: [BLANK 1]
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: [BLANK 2]
```

Requirement: capacity must never drop below the desired replica count.

- **BLANK 1:** `RollingUpdate` / `Recreate` / `BlueGreen` / `Canary`
- **BLANK 2:** `0` / `1` / `25%` / `100%`

### Answer: `RollingUpdate`, `0`

**In `challenge-28.md`:** lines **353–357**.

`Recreate` is the other **built-in** strategy: it terminates every pod, then starts the new ones —
guaranteed downtime, which is what Contoso is escaping. `BlueGreen` and `Canary` are patterns you
build with extra tooling, not `strategy.type` values.

---

## Q40

```yaml
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          format: '[BLANK 1]'
          severity: 'CRITICAL,HIGH'
          [BLANK 2]: '1'
          trivyignores: '[BLANK 3]'
```

- **BLANK 1:** `sarif` / `json` / `table` / `template`
- **BLANK 2:** `exit-code` / `fail-on` / `severity-threshold` / `block`
- **BLANK 3:** `.trivyignore` / `.securityignore` / `trivy-exclusions.txt` / `.cveignore`

### Answer: `sarif`, `exit-code`, `.trivyignore`

**In `challenge-28.md`:** lines **502–509** and **690–706**.

`sarif` is what the Security tab accepts (line 512). `exit-code: '1'` is what converts a report into
a gate.

**A `.trivyignore` entry should always carry a reason and a tracking reference** — line 692 does
exactly that: `# No fix available - tracked in issue #1234`. An unexplained ignore is how a real
vulnerability gets permanently suppressed.

---

## Q41

```yaml
      - name: Set up [BLANK 1]
        uses: docker/setup-qemu-action@v3
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Build and push multi-arch image
        uses: docker/build-push-action@v6
        with:
          [BLANK 2]: linux/amd64,linux/arm64
```

- **BLANK 1:** `QEMU` / `Kubernetes` / `Helm` / `Docker Compose`
- **BLANK 2:** `platforms` / `architectures` / `targets` / `variants`

### Answer: `QEMU`, `platforms`

**In `challenge-28.md`:** lines **562–580**.

QEMU emulates the foreign architecture; buildx assembles the multi-platform manifest. Both are
required, and emulated builds are noticeably slower than native — a native ARM runner is the faster
alternative if build time matters.

---

# Section G — Case study

## Case study: Contoso container platform

### Background

Contoso is breaking a monolithic e-commerce application into three microservices: Product Catalog
API, Order Processing Service and Notification Service. Product Catalog runs on **Azure Container
Apps**; Order Processing runs on **AKS**.

### Requirements

**Build and registry**

- Images must be built on every pull request but published only on merge to `main`
- Every image must carry a semantic version, the commit SHA, and `latest` on the default branch
- Images must be signed, and the registry must be geo-replicated
- No registry username or password may be stored anywhere

**Security**

- An image with a CRITICAL vulnerability must not reach the registry
- Findings must be visible to the security team in GitHub
- Images already in the registry must be re-assessed as new CVEs are disclosed

**Deployment**

- The AKS rollout must never reduce capacity
- The pipeline must fail if pods do not become healthy
- A bad Container Apps release must be reversible without a rebuild

---

## Q42

Which configuration meets the build-on-PR, publish-on-merge requirement?

- A. `push: ${{ github.event_name != 'pull_request' }}` on the build step
- B. A job-level `if: github.ref == 'refs/heads/main'`
- C. Two separate workflows, one per event
- D. `push: true` with a branch filter on the trigger

### Answer: A

**In `challenge-28.md`:** line **115**, with the trigger at lines **62–70**.

**Why the others fail**

- **B** — skips the entire job on a PR, so the image is never built and the validation is lost
- **C** — duplicated definitions that drift apart. This is the problem Challenge 23 solves with
  reuse, not something to introduce deliberately
- **D** — a branch filter on the trigger stops PRs firing the workflow at all

---

## Q43

Which **two** meet the registry requirements? (Choose two.)

- A. Premium SKU with content trust enabled
- B. `--admin-enabled false` with identity-based authentication
- C. Standard SKU with the admin account enabled
- D. Basic SKU with an access token stored as a secret
- E. GHCR with `secrets.GITHUB_TOKEN`

### Answer: A, B

**In `challenge-28.md`:** lines **137–146**.

```bash
az acr create --sku Premium --admin-enabled false
az acr config content-trust update --status enabled
```

Signing and geo-replication both require **Premium**; "no stored username or password" requires the
admin account **off**.

**Why the others fail**

- **C** and **D** — wrong SKU for signing, and both store credentials
- **E** — **the interesting one.** GHCR needs no stored secret, which satisfies one requirement. But
  it offers neither content trust nor geo-replication, so it fails the other two. A partially correct
  option is still wrong

---

## Q44

Which **two** meet the security requirements? (Choose two.)

- A. Trivy with `exit-code: '1'` before the push step
- B. Microsoft Defender for Containers enabled on the registry
- C. Trivy running after the push, in report-only mode
- D. A weekly manual review of the registry
- E. `.trivyignore` covering all CRITICAL findings

### Answer: A, B

**In `challenge-28.md`:** lines **502–509** and **518–524**.

**Order matters for A.** Scanning **before** the push means a vulnerable image never enters the
registry. Scanning after means it is already there, and someone can deploy it.

**B covers the third requirement** — continuous re-assessment as new CVEs are disclosed, which
build-time scanning structurally cannot do.

**Why the others fail**

- **C** — wrong order and not a gate
- **D** — not automatic, and slower than the CVE feed
- **E** — suppresses exactly what must block

---

## Q45

Which configuration meets the AKS capacity requirement?

- A. `maxSurge: 1, maxUnavailable: 0` with a readiness probe
- B. `maxSurge: 0, maxUnavailable: 1`
- C. `strategy: type: Recreate`
- D. `replicas: 5`

### Answer: A

**In `challenge-28.md`:** lines **353–357** and **381–386**.

The readiness probe is not optional here. `maxUnavailable: 0` waits for a new pod to be **available**,
and availability is defined by readiness. Without the probe, a pod counts as ready the instant it
starts and capacity really does dip.

**Why the others fail**

- **B** — removes a pod first. Capacity drops to 2 of 3
- **C** — `Recreate` terminates everything before starting anything. Guaranteed downtime
- **D** — more replicas, same percentage dip

---

## Q46

Which configuration makes the pipeline fail when pods do not become healthy?

- A. `KubernetesManifest@1` with `action: 'deploy'`
- B. `Kubernetes@1` running `rollout status --timeout=300s`
- C. `kubectl get pods` after deployment
- D. A liveness probe on the container

### Answer: B

**In `challenge-28.md`:** lines **477–486**.

**Why the others fail**

- **A** — returns once manifests are **applied**. Applying a manifest that produces crash-looping pods
  is still a successful apply
- **C** — prints pods and exits 0 regardless
- **D** — a liveness probe restarts unhealthy containers indefinitely. It never tells the pipeline
  anything

**The general rule, worth carrying:** *applying* is not *verifying*. Any deployment step needs a
companion that **fails** when the result is wrong.

---

## Q47

Which configuration reverses a bad Container Apps release without a rebuild?

- A. `az containerapp ingress traffic set --revision-weight "<previous>=100"`
- B. `az containerapp update --image <previous-tag>`
- C. Re-running the previous successful pipeline
- D. `az containerapp restart`

### Answer: A

**In `challenge-28.md`:** lines **316–328**.

The previous revision already exists and is warm. Shifting traffic is immediate.

**Why the others fail**

- **B** — no rebuild, but it creates a **new revision** that must provision and warm up. Slower, and
  it duplicates history (Q26)
- **C** — a full pipeline run, including a build
- **D** — restarts the same broken image

**Note the rollback runs under `if: failure()`** (line 317) — the same pattern as Challenge 25's slot
swap-back. Detect, then react.

---

## Q48

Six months on, Contoso's ACR has grown to 800 GB. Most of it is untagged manifests from repeated
`latest` pushes, plus hundreds of old SHA-tagged images.

What should they configure, and why does the problem exist?

- A. A retention policy for untagged manifests plus an `acr purge` task; every re-push of a tag orphans
  the previous manifest
- B. Upgrade to a larger Premium tier
- C. Delete the repository and start again
- D. Disable SHA tagging so fewer images are created

### Answer: A

**In `challenge-28.md`:** lines **168–180**.

```bash
az acr config retention update --status enabled --days 7 --type UntaggedManifests

az acr run --cmd "acr purge --filter 'product-catalog-api:.*' --ago 30d --keep 10 --untagged" /dev/null
```

**Why it happens:** pushing a new image as `latest` does not delete the old one. The tag moves; the
old manifest and all its layers stay, now untagged and invisible in the portal's tag list — but fully
billed. Every merge to `main` adds another.

Two policies for two problems: **retention** collects untagged manifests automatically; **purge**
manages tagged images by age and count, keeping the last 10.

**Why the others fail**

- **B** — pays for the garbage instead of removing it. Storage cost scales linearly and never stops
- **C** — destroys images you may need to roll back to
- **D** — **the trap.** SHA tags are what make deployments traceable and rollbacks possible (Q33).
  Removing them to save storage trades a real capability for a cost you can fix with policy

**This is Challenge 36's subject in miniature** — retention strategy for artifacts, packages and
images.

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **`AcrPull` where `AcrPush` is needed** | Q1, Q27 | Pull reads. Push reads and writes |
| **Admin account as the easy auth answer** | Q1, Q25, Q43 | Shared credential. Use identity and `--admin-enabled false` |
| **Liveness and readiness swapped** | Q5, Q28 | Liveness restarts. Readiness removes from Service |
| **`maxUnavailable: 0` without a readiness probe** | Q19, Q45 | Availability is *defined* by readiness |
| **Applying assumed to be verifying** | Q8, Q23, Q46 | `KubernetesManifest@1` applies. `rollout status` verifies |
| **`severity` mistaken for a gate** | Q11, Q30 | `exit-code: '1'` blocks. `severity` only filters |
| **Port mismatch across three places** | Q3, Q29 | App listens, Dockerfile documents, ingress routes |
| **New revision vs routing to an old one** | Q6, Q26, Q47 | `--image` creates a revision. `--revision-weight` routes back |
| **QEMU omitted for multi-arch** | Q13, Q41 | Buildx builds; QEMU emulates. Both needed |
| **`push: true` on pull requests** | Q2, Q42 | Build on PR, publish on merge |
| **Scanning after the push** | Q44 | Scan before, or the bad image is already available |
| **Deleting SHA tags to save storage** | Q48 | Use retention and purge. SHA tags are traceability |

---

# The blocks to memorise

Line numbers are in `challenge-28.md`.

```dockerfile
# 1. Multi-stage, non-root  (lines 34-52)
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
RUN dotnet publish -c Release -o /app/publish /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
EXPOSE 8080
RUN adduser --disabled-password --gecos "" appuser
USER appuser
COPY --from=build /app/publish .
```

```yaml
# 2. Build on PR, push on merge  (lines 111-119)
      - uses: docker/build-push-action@v6
        with:
          push: ${{ github.event_name != 'pull_request' }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

# 3. Tagging strategy  (lines 105-109)
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=,format=short
            type=raw,value=latest,enable={{is_default_branch}}

# 4. Scan as a gate, then report  (lines 502-515)
      - uses: aquasecurity/trivy-action@master
        with:
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
          format: 'sarif'
      - uses: github/codeql-action/upload-sarif@v3
        if: always()

# 5. Kubernetes rolling update with no capacity loss  (lines 353-386)
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
...
          livenessProbe:   { httpGet: { path: /health/live,  port: 8080 } }
          readinessProbe:  { httpGet: { path: /health/ready, port: 8080 } }

# 6. Deploy then VERIFY  (lines 463-486)
                - task: KubernetesManifest@1
                  inputs:
                    action: 'deploy'
                    containers: | 
                      $(acrName).azurecr.io/$(imageRepository):$(tag)
                - task: Kubernetes@1
                  inputs:
                    command: 'rollout'
                    arguments: 'status deployment/product-catalog-api --timeout=300s'

# 7. Multi-arch  (lines 562-580)
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v6
        with:
          platforms: linux/amd64,linux/arm64
```

```bash
# 8. Registry hardening and lifecycle  (lines 137-180, 243-244, 325-328)
az acr create --sku Premium --admin-enabled false
az acr config content-trust update --status enabled
az acr config retention update --status enabled --days 7 --type UntaggedManifests
az acr run --cmd "acr purge --filter 'repo:.*' --ago 30d --keep 10 --untagged" /dev/null

az containerapp create --registry-identity system --target-port 8080 --ingress external
az containerapp ingress traffic set --revision-weight "<previous-revision>=100"   # rollback
```

**ACR roles:** `AcrPull` → `AcrPush` → `AcrDelete` → `AcrImageSigner`.

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 28 is exam-ready. Move to Challenge 29 |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 4 and 5, then retake this |
| Below 30 | Redo the challenge, writing the Kubernetes manifest from memory before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 28.

:::tip The one thing

**Applying is not verifying, and reporting is not gating.**

`KubernetesManifest@1` applies — `rollout status` verifies. Trivy's `severity` reports —
`exit-code: '1'` gates. Both pairs generate exam questions, and both are the same idea.

:::
