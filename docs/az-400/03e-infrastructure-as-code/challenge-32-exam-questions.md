---
sidebar_position: 2.5
toc_max_heading_level: 2
title: "Challenge 32: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 32 — AZ-400 exam questions

**48 questions** built only from what Challenge 32 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-32.md`**.

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

:::tip Two words decide most of this challenge

**Audit** reports. **AuditAndSet** enforces. Almost every wrong answer here is a configuration that
*observes* drift where the requirement was to *correct* it — including the challenge's own Break & fix
Exercise 2.

:::

---

# Section A — Single answer

---

## Q1

Which service does Azure Machine Configuration use to audit and enforce settings **inside** a VM?

- A. Azure Policy
- B. Azure Monitor
- C. Azure Automation
- D. Microsoft Defender for Cloud

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** line **33**.

> *"Azure Machine Configuration uses Azure Policy to audit and enforce settings inside VMs."*

**The division of labour matters.** Azure Policy normally governs the **resource** — its SKU, its
location, its tags. Machine Configuration extends Policy **inside the guest OS**, so registry keys,
services and installed software become policy-evaluable.

That is why everything downstream is Policy machinery: a policy **definition** (line 195), an
**assignment** (line 200), **compliance state** (line 215) and **remediation tasks** (line 304).

**Why the others fail**

- **B** — collects telemetry; it does not enforce configuration
- **C** — hosts the **older** Automation State Configuration (line 249), a different, pull-based
  mechanism
- **D** — assesses security posture and consumes policy results rather than driving guest
  configuration

</details>

---

## Q2

A VM shows "Pending" compliance status for 24 hours after policy assignment.

Which **two** prerequisites are most likely missing? (Pick the single best answer.)

- A. A system-assigned managed identity and the Guest Configuration extension
- B. A network security group rule and a public IP
- C. An Azure Monitor agent and a Log Analytics workspace
- D. A user-assigned identity and a service principal

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** Break & fix Exercise 1, lines **497–538**.

```bash
az vm show --query "identity.type"
# Returns: null  <-- ERROR: No managed identity assigned

az vm extension list --query "[?publisher=='Microsoft.GuestConfiguration']..."
# Returns: empty  <-- ERROR: Extension not installed
```

**Both are required and they do different jobs.** The **managed identity** is how the VM authenticates
to Azure to report its own compliance. The **extension** is the agent that actually evaluates the
configuration inside the guest.

**The fix, plus one extra step (line 535):**

```bash
az vm identity assign ...
az vm extension set --name AzurePolicyforWindows --publisher Microsoft.GuestConfiguration ...
az policy state trigger-scan --resource-group rg-contoso-vms --no-wait
```

`trigger-scan` forces immediate evaluation instead of waiting for the next cycle — which is why the
status sat at Pending rather than failing outright.

**Why the others fail** — B, C and D describe unrelated prerequisites. Machine Configuration needs no
inbound network access; the agent connects outbound.

</details>

---

## Q3

Which package type both **reports** and **corrects** configuration drift?

- A. `Audit`
- B. `AuditAndSet`
- C. `Set`
- D. `Enforce`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-32.md`:** line **143**, and Break & fix Exercise 2 at lines **547–569**.

```powershell
New-GuestConfigurationPackage `
    -Name 'ContosoSecurityBaseline' `
    -Configuration './compiled/ContosoWindows.mof' `
    -Type AuditAndSet `
    -Force
```

**Break & fix Exercise 2 is exactly this mistake**, and its symptom is worth memorising: *"the policy
shows all VMs as compliant when they clearly are not."* An `Audit` package only checks the settings it
was told to check — it never corrects, so drift persists silently while the dashboard looks healthy.

**Why the others fail**

- **A** — reports only. Correct for a baseline you are still measuring, wrong for one you are
  enforcing
- **C** and **D** — not valid `-Type` values. The two are `Audit` and `AuditAndSet`

</details>

---

## Q4

Which `New-GuestConfigurationPolicy` mode automatically corrects non-compliant machines?

- A. `Audit`
- B. `ApplyAndAutoCorrect`
- C. `ApplyAndMonitor`
- D. `Disabled`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-32.md`:** line **191**.

```powershell
New-GuestConfigurationPolicy `
    -ContentUri $packageUri `
    -Platform Windows `
    -Mode ApplyAndAutoCorrect `
    -Version '1.0.0'
```

**Two settings must agree**, and this is the pairing the exam tests: the **package** must be
`AuditAndSet` (line 143) *and* the **policy** must be `ApplyAndAutoCorrect`. An `AuditAndSet` package
assigned in audit mode still only reports.

**Why the others fail**

- **A** — reports only
- **C** — **real and different.** `ApplyAndMonitor` applies the configuration **once**, then only
  monitors afterwards. If someone changes the setting later it is reported, not corrected. That is a
  genuine middle option, and not "automatically corrects"
- **D** — not a mode here

</details>

---

## Q5

Why must the configuration package be uploaded to storage with a long-lived SAS token?

- A. So Azure Policy can retrieve the package when evaluating machines
- B. To back up the package
- C. To allow VMs to write compliance results
- D. To version the package

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** lines **166–186**.

```powershell
$sasToken = New-AzStorageBlobSASToken `
    -Permission r `
    -ExpiryTime (Get-Date).AddYears(3)

$packageUri = "$($storageAccount.Context.BlobEndPoint)guestconfiguration/ContosoSecurityBaseline.zip$sasToken"
```

**The URI is baked into the policy definition** (line 186). Every machine evaluating the policy fetches
the package from that URI, so it must stay reachable for the life of the assignment — hence three
years.

**Note `-Permission r`** — read only. The machines download; nothing writes back through this URI.

**The operational consequence worth knowing:** when the SAS token expires, every assignment using it
breaks at once. That expiry date is a real thing to track.

**Why the others fail** — B, C and D all misdescribe the purpose.

</details>

---

## Q6

Which policy assignment setting allows Azure Policy to remediate resources?

- A. `-IdentityType SystemAssigned`
- B. `-EnforcementMode Default`
- C. `-NotScopes`
- D. `-Metadata`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** lines **200–206**.

```powershell
New-AzPolicyAssignment `
    -Name 'contoso-baseline-assignment' `
    -PolicyDefinition $definition `
    -Scope "/subscriptions/{sub-id}/resourceGroups/rg-contoso-vms" `
    -IdentityType SystemAssigned `
    -Location 'eastus2'
```

**A policy that only audits needs no identity.** A policy that **changes** things — `DeployIfNotExists`
or `Modify` — must act as someone, and that someone is the assignment's managed identity.

That is also why `-Location` is required (line 206): a managed identity is a resource and needs a
region.

**The follow-on step people forget:** the identity needs an **RBAC role** at the scope, or remediation
fails with an authorisation error even though the assignment looks correct.

**Why the others fail**

- **B** — `EnforcementMode` decides whether the effect is applied or only logged (`DoNotEnforce` is
  "what-if" for policy). It does not grant permission
- **C** — excludes scopes
- **D** — descriptive metadata

</details>

---

## Q7

Which command forces immediate policy compliance evaluation instead of waiting for the next cycle?

- A. `az policy state trigger-scan`
- B. `az policy assignment update`
- C. `az policy remediation create`
- D. `az vm restart`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** lines **534–537**.

```bash
az policy state trigger-scan \
  --resource-group rg-contoso-vms \
  --no-wait
```

**Why it exists:** policy evaluation runs on a schedule — roughly every 24 hours, plus on resource
change. After fixing prerequisites you do not want to wait a day to learn whether it worked.

**Why the others fail**

- **B** — updating an assignment does trigger re-evaluation as a side effect, which makes it a
  plausible distractor. `trigger-scan` is the purpose-built command
- **C** — remediation **fixes** known non-compliant resources; it does not re-evaluate compliance
- **D** — restarting the VM does not schedule a policy scan

</details>

---

## Q8

Which DSC resource enforces that a Windows service is running and set to start automatically?

- A. `Registry`
- B. `Service`
- C. `WindowsFeature`
- D. `Script`

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-32.md`:** lines **113–117**.

```powershell
        Service 'WindowsFirewall' {
            Name        = 'MpsSvc'
            State       = 'Running'
            StartupType = 'Automatic'
        }
```

**Both properties are needed.** `State = 'Running'` starts it now; `StartupType = 'Automatic'` ensures
it starts after a reboot. Setting only the first means the firewall is running until the next restart.

**Why the others fail**

- **A** — `Registry` sets registry values, used for TLS at lines 88–110
- **C** — `WindowsFeature` installs roles and features (line 120)
- **D** — `Script` runs arbitrary Get/Set/Test blocks. It works and is the last resort, because you
  must write the idempotency logic yourself

</details>

---

## Q9

Which registry configuration disables TLS 1.0 on the server?

- A. `ValueData = '1'` under the TLS 1.0 Server key
- B. `ValueData = '0'` under the TLS 1.0 Server key
- C. Deleting the TLS 1.0 key
- D. `Ensure = 'Absent'` on the TLS 1.2 key

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-32.md`:** lines **88–102**.

```powershell
        Registry 'DisableTLS10' {
            Ensure    = 'Present'
            Key       = 'HKLM:\...\Protocols\TLS 1.0\Server'
            ValueName = 'Enabled'
            ValueType = 'DWord'
            ValueData = '0'
        }
```

**Note `Ensure = 'Present'` with `ValueData = '0'`** — the *value* must exist and be zero. That is
different from removing the key, and it is the part people misread.

**Why the others fail**

- **A** — `1` **enables** it, which is what line 109 does for TLS 1.2
- **C** — an absent key means "default", and the default is enabled. Deleting it does not disable
  anything
- **D** — disabling TLS 1.2 would break every modern client

**The pattern across lines 88–110:** disable 1.0, disable 1.1, enable 1.2 — three explicit settings,
because leaving any of them to a default leaves it to chance.

</details>

---

## Q10

Which command tests a Machine Configuration package **locally** before publishing?

- A. `Test-GuestConfigurationPackage`
- B. `New-GuestConfigurationPackage`
- C. `Start-DscConfiguration`
- D. `az policy state summarize`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** lines **150–154**.

```powershell
$result = Test-GuestConfigurationPackage `
    -Path './ContosoSecurityBaseline/ContosoSecurityBaseline.zip'

$result.resources | Format-Table ResourceName, InDesiredState, Reasons
```

**Read the output columns** — they are the exam-relevant part. `InDesiredState` tells you whether each
resource is compliant on this machine; **`Reasons`** explains *why not*, which is what turns a red
dashboard into an actionable finding.

**Why the others fail**

- **B** — builds the package; it does not evaluate it
- **C** — applies a DSC configuration directly, bypassing Machine Configuration entirely
- **D** — summarises compliance for **already-assigned** policies in Azure

**Why local testing matters here:** the publish path is long — upload, SAS, definition, assignment,
evaluation cycle. Catching a broken package before all of that saves a day.

</details>

---

## Q11

What is the relationship between Azure Automation State Configuration and Azure Machine Configuration?

- A. Automation State Configuration is the newer replacement
- B. Machine Configuration supersedes Automation State Configuration
- C. They are the same service under two names
- D. Automation State Configuration only works on Linux

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-32.md`:** line **251**.

> *"For VMs that need pull-based configuration (legacy approach, being superseded by Machine
> Configuration)"*

**The architectural difference:**

| | Automation State Configuration | Machine Configuration |
|---|---|---|
| Model | **Pull** — nodes poll a pull server | Policy-driven evaluation |
| Registration | `Register-AzAutomationDscNode` (line 276) | Managed identity + extension |
| Compliance view | Automation account | **Azure Policy compliance** |
| Direction | Legacy | Current |

**Why the others fail**

- **A** — reversed
- **C** — both use DSC configurations, and the delivery mechanism differs entirely
- **D** — Automation State Configuration is Windows-centric; Linux support was always more limited

</details>

---

## Q12

In `Register-AzAutomationDscNode`, what does `RefreshFrequencyMins 30` control?

- A. How often the node checks the pull server for a new configuration
- B. How often the node applies the configuration
- C. How often compliance is reported to Azure Policy
- D. The reboot interval

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** lines **282–286**.

```powershell
    -ConfigurationMode 'ApplyAndAutoCorrect' `
    -RebootNodeIfNeeded $true `
    -ActionAfterReboot 'ContinueConfiguration' `
    -RefreshFrequencyMins 30 `
    -ConfigurationModeFrequencyMins 15
```

**Two frequencies, two jobs — and this is the pair the exam separates:**

| Setting | Controls |
|---|---|
| `RefreshFrequencyMins` | How often the node **downloads** its configuration from the pull server |
| `ConfigurationModeFrequencyMins` | How often the node **evaluates and applies** the configuration it has |

So this node checks for a new configuration every 30 minutes and re-applies the current one every 15.

**Why the others fail**

- **B** — that is `ConfigurationModeFrequencyMins`
- **C** — Automation State Configuration reports to the Automation account, not Policy
- **D** — reboots are governed by `RebootNodeIfNeeded` and `ActionAfterReboot`

</details>

---

## Q13

Which command creates a task that fixes already non-compliant resources?

- A. `az policy remediation create`
- B. `az policy state trigger-scan`
- C. `az policy assignment create`
- D. `az policy definition create`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** lines **304–308**.

```bash
az policy remediation create \
  --name "remediate-contoso-baseline" \
  --policy-assignment "contoso-baseline-assignment" \
  --resource-group "rg-contoso-vms" \
  --resource-discovery-mode ReEvaluateCompliance
```

**Why remediation tasks exist at all:** a `DeployIfNotExists` policy acts on resources **as they are
created or updated**. Resources that already existed when you assigned the policy are marked
non-compliant and otherwise left alone. A remediation task is what goes back and fixes them.

**Why the others fail**

- **B** — evaluates compliance; it changes nothing
- **C** — creates the assignment, which is a prerequisite
- **D** — creates the definition

**And the dependency chain:** remediation requires the assignment to have a managed identity (Q6) with
an appropriate RBAC role.

</details>

---

## Q14

What does `--resource-discovery-mode ReEvaluateCompliance` do?

- A. Re-evaluates compliance before remediating, catching resources whose state has changed
- B. Skips resources already marked compliant
- C. Deletes non-compliant resources
- D. Runs remediation only on newly created resources

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** line **308**.

**The two modes:**

| Mode | Behaviour |
|---|---|
| `ExistingNonCompliant` (default) | Remediate resources currently **recorded** as non-compliant |
| `ReEvaluateCompliance` | **Re-scan first**, then remediate what is actually non-compliant |

**Why re-evaluating is safer:** compliance data can be up to 24 hours old. Without a fresh scan you
may remediate a machine someone already fixed, or miss one that drifted this morning.

**Why the others fail** — B, C and D all describe behaviours these modes do not have. Note especially
C: policy remediation **corrects** resources; it never deletes them.

</details>

---

## Q15

Which KQL table reports Machine Configuration compliance in Azure Resource Graph?

- A. `GuestConfigurationResources`
- B. `PolicyResources`
- C. `SecurityResources`
- D. `Resources`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** lines **235–246**.

```kusto
GuestConfigurationResources
| where type == "microsoft.guestconfiguration/guestconfigurationassignments"
| extend complianceStatus = properties.complianceStatus
| summarize
    Compliant = countif(complianceStatus == "Compliant"),
    NonCompliant = countif(complianceStatus == "NonCompliant"),
    Pending = countif(complianceStatus == "Pending")
    by configName
| extend ComplianceRate = round(todouble(Compliant) / todouble(Total) * 100, 1)
```

**Note the three states, not two.** `Pending` is its own category — machines that have never reported,
usually because of the missing identity or extension from Break & fix Exercise 1. Counting Pending as
compliant would hide exactly the problem you are looking for.

**`todouble()` before dividing** avoids integer truncation, the same reason Challenge 27 used `100.0`.

**Why the others fail** — `PolicyResources` holds policy definitions and assignments;
`SecurityResources` is Defender for Cloud; `Resources` is the general ARM inventory.

</details>

---

## Q16

Break & fix Exercise 2 has a second defect beyond the wrong package type. What is it?

- A. The dependent DSC module is not bundled into the package
- B. The MOF file is not compiled
- C. The SAS token has expired
- D. The policy is assigned to the wrong scope

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** lines **554–573**.

```powershell
# ALSO BROKEN: Missing module dependency in package
# The PSDscResources module is referenced but not bundled
```

```powershell
New-GuestConfigurationPackage `
    -Type AuditAndSet `
    -FilesToInclude @(
        (Get-Module PSDscResources -ListAvailable).ModuleBase
    )
```

**Why it fails only on the target machine.** The configuration imports `PSDscResources` (line 83),
which is installed on the **authoring workstation**. The target VM has never seen it. Without
`-FilesToInclude`, the package arrives referencing a module that is not there.

**Both defects produce the same misleading symptom** — a dashboard showing compliance that is not
real. One because nothing is enforced, the other because nothing can run.

**Why the others fail** — B, C and D are unrelated to this exercise.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** prerequisites must exist before Machine Configuration can evaluate a VM? (Choose
three.)

- A. The `Microsoft.GuestConfiguration` resource provider registered
- B. A system-assigned managed identity on the VM
- C. The Guest Configuration extension installed
- D. A public IP address on the VM
- E. A Log Analytics workspace
- F. An inbound NSG rule on port 443

<details>
<summary>Show answer</summary>

### Answer: A, B, C

**In `challenge-32.md`:** lines **37**, **43**, **56–61**.

```bash
az provider register --namespace Microsoft.GuestConfiguration          # A
az vm identity assign --resource-group rg-contoso-vms --name vm-...    # B
az vm extension set --name AzurePolicyforWindows \
  --publisher Microsoft.GuestConfiguration --version 1.1               # C
```

**Why the others fail — and this is the architectural point**

- **D** and **F** — the agent connects **outbound** only. No public IP, no inbound rule. That is what
  makes Machine Configuration acceptable on machines with no internet exposure, and it is the same
  outbound-only model as self-hosted runners in Challenge 21
- **E** — Log Analytics is used by Azure Monitor and VM Insights (Challenge 47), not by Machine
  Configuration

**At scale, B is applied by policy** (lines 49–53) rather than one VM at a time — a built-in policy
that adds a system-assigned identity to VMs.

</details>

---

## Q18

Which **two** settings must both be correct for a configuration to be **enforced** rather than merely
reported? (Choose two.)

- A. `New-GuestConfigurationPackage -Type AuditAndSet`
- B. `New-GuestConfigurationPolicy -Mode ApplyAndAutoCorrect`
- C. `New-GuestConfigurationPolicy -Mode Audit`
- D. `-Platform Windows`
- E. `-Version '1.0.0'`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-32.md`:** lines **143** and **191**.

**Both, or neither works.** An `AuditAndSet` package assigned in `Audit` mode reports only. An `Audit`
package in `ApplyAndAutoCorrect` mode has nothing to apply. The **package** says what it is capable
of; the **policy** says what it is allowed to do.

**Why the others fail**

- **C** — reports only, and is Break & fix Exercise 2's defect
- **D** — targets the OS platform
- **E** — versioning, which matters for updates but not for enforcement

</details>

---

## Q19

Which **two** are required before a policy remediation task can succeed? (Choose two.)

- A. The policy assignment has a managed identity
- B. That identity holds an appropriate RBAC role at the scope
- C. The VM has a public IP
- D. The policy effect is `Audit`
- E. Compliance data is less than one hour old

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-32.md`:** lines **205** and **304–308**.

```powershell
    -IdentityType SystemAssigned `
    -Location 'eastus2'
```

**A is visible in the code; B is the step that is easy to miss.** The assignment identity exists but
starts with **no permissions**. Remediation then fails with an authorisation error while the
assignment itself looks perfectly configured.

**Why the others fail**

- **C** — no inbound connectivity is required
- **D** — **the inverted one.** An `Audit` effect produces nothing to remediate. Remediation applies to
  `DeployIfNotExists` and `Modify`
- **E** — stale data is exactly why `ReEvaluateCompliance` exists (line 308); freshness is not a
  prerequisite

</details>

---

## Q20

Which **two** DSC resources appear in Contoso's security baseline? (Choose two.)

- A. `Registry` for TLS and password settings
- B. `Service` for the Windows Firewall
- C. `File` for audit log paths
- D. `User` for local accounts
- E. `Package` for antivirus installation

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-32.md`:** lines **88–132**.

```powershell
        Registry 'DisableTLS10' { ... }      # A - TLS 1.0 off
        Registry 'EnableTLS12'  { ... }      # A - TLS 1.2 on
        Service  'WindowsFirewall' { Name = 'MpsSvc'; State = 'Running' }   # B
        WindowsFeature 'MonitoringAgent' { ... }
        Registry 'MinPasswordLength' { ValueData = '14' }                    # A
```

**Why the others fail** — `File`, `User` and `Package` are all real DSC resources and none is used
here. The baseline covers TLS, firewall, a Windows feature and a password-length registry value.

**Note `MpsSvc`** — the Windows Firewall service's real name. Configuration resources address the
service name, not its display name, and getting that wrong produces a resource that silently manages
nothing.

</details>

---

## Q21

Which **two** distinguish Automation State Configuration from Machine Configuration? (Choose two.)

- A. Automation State Configuration uses a pull model with node registration
- B. Machine Configuration reports compliance through Azure Policy
- C. Machine Configuration requires an Automation account
- D. Automation State Configuration is the current recommended approach
- E. Machine Configuration cannot enforce settings

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-32.md`:** lines **251**, **276–286**, **215–223**.

```powershell
Register-AzAutomationDscNode `
    -NodeConfigurationName 'ContosoSecurityBaseline.ContosoWindows' `
    -ConfigurationMode 'ApplyAndAutoCorrect' `
    -RefreshFrequencyMins 30
```

**The compliance surface is the practical difference.** Automation State Configuration reports to the
Automation account, so you check node status there (line 291). Machine Configuration reports through
**Policy compliance**, which means guest settings appear alongside every other governance control on
one dashboard.

**Why the others fail**

- **C** — Machine Configuration needs no Automation account at all
- **D** — line 251 calls it legacy and superseded
- **E** — `AuditAndSet` plus `ApplyAndAutoCorrect` enforces

</details>

---

## Q22

Which **two** compliance queries would you use to find machines failing the baseline? (Choose two.)

- A. `az policy state list --filter "... and complianceState eq 'NonCompliant'"`
- B. A Resource Graph query over `GuestConfigurationResources`
- C. `az vm list`
- D. `az policy definition list`
- E. `az automation dsc node list`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-32.md`:** lines **220–223** and **235–246**.

**They answer different questions.** The `az policy state` query is scoped to **one policy** and gives
you the non-compliant resource IDs. The Resource Graph query aggregates **across configurations and
subscriptions**, producing the compliance-rate trend that goes on a workbook.

**Why the others fail**

- **C** — lists VMs with no compliance information
- **D** — lists definitions, not results
- **E** — real, and it queries **Automation State Configuration** nodes (line 291). Wrong service for
  a Machine Configuration baseline

</details>

---

## Q23

Which **two** describe the runbook-based scheduled remediation? (Choose two.)

- A. It authenticates with `Connect-AzAccount -Identity`
- B. It calls `Start-AzPolicyRemediation` only when non-compliant resources exist
- C. It applies DSC configurations directly to each VM
- D. It requires a stored service principal secret
- E. It runs on every VM locally

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-32.md`:** lines **332** and **340–348**.

```powershell
Connect-AzAccount -Identity

$nonCompliant = Get-AzPolicyState -Filter "complianceState eq 'NonCompliant'" ...

if ($nonCompliant.Count -gt 0) {
    Start-AzPolicyRemediation -Name "auto-remediate-$(Get-Date -Format 'yyyyMMdd-HHmmss')" ...
} else {
    Write-Output "All resources are compliant. No remediation needed."
}
```

**A is the identity pattern again** — the Automation account's managed identity, no secret stored.
Same reasoning as OIDC in Challenge 31.

**B is a guard worth noticing:** creating a remediation task when nothing is non-compliant produces
noise and consumes quota. The check makes the runbook idempotent and quiet.

**Note the timestamped name** — remediation task names must be unique, so a scheduled runbook that
used a fixed name would fail on its second run.

**Why the others fail** — C describes Automation State Configuration; D contradicts `-Identity`; E is
wrong, the runbook runs centrally against Azure.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must enforce TLS 1.2, an enabled firewall and a 14-character password policy on
70 VMs. Ad-hoc operator changes must be **corrected automatically**, and the security team needs
fleet-wide compliance visibility.

---

## Q24

**Proposed solution:** Build an `AuditAndSet` Machine Configuration package, publish it with
`-Mode ApplyAndAutoCorrect`, assign the policy with a system-assigned identity, and grant that
identity a role at the resource group scope.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-32.md`:** lines **143**, **191**, **205**.

| Requirement | Mechanism |
|---|---|
| Enforce, not just observe | `AuditAndSet` **and** `ApplyAndAutoCorrect` |
| Correct operator drift automatically | Auto-correct re-applies on each evaluation |
| Fleet-wide visibility | Policy compliance across the assigned scope |

The RBAC grant is the step that turns a correct-looking assignment into a working one.

</details>

---

## Q25

**Proposed solution:** Build an `Audit` Machine Configuration package, assign it, and review the
compliance dashboard weekly so operations can fix failures manually.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Visibility: yes. Automatic correction: no.**

This is Break & fix Exercise 2 as a design decision rather than a mistake. `Audit` reports and never
enforces, so the 40% failure rate in the scenario (line 27) becomes a number someone reads rather than
a number that falls.

**And the deeper problem:** the drift is caused by operators making ad-hoc changes during incidents.
A weekly manual fix cycle competes with a continuous drift source, and loses.

**When `Audit` is right:** while you are measuring a new baseline and do not yet know what enforcement
would break. It is a starting point, not a destination.

</details>

---

## Q26

**Proposed solution:** Build an `AuditAndSet` package, publish with `-Mode ApplyAndAutoCorrect`, and
assign the policy — but leave the assignment without a managed identity.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is correct except one line, and the failure is quiet.

Without `-IdentityType SystemAssigned` (line 205), the assignment has no identity to act as. Auditing
still works — machines report their state — so **the dashboard populates and looks healthy in the
sense that data is flowing**. But nothing is ever corrected, and remediation tasks fail with
authorisation errors.

**The symptom pattern to recognise:** compliance data appears, non-compliant machines stay
non-compliant indefinitely, and remediation tasks show failed deployments.

**This is the same shape as Challenge 24 Q1** — a job with no `environment:` still runs, it just has
no gate. Here an assignment with no identity still audits, it just cannot act.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — prerequisites

| # | Statement | Answer |
|---|---|---|
| 1 | VMs need a managed identity for Machine Configuration |  |
| 2 | The Guest Configuration extension is required |  |
| 3 | VMs need inbound network access on port 443 |  |
| 4 | The `Microsoft.GuestConfiguration` provider must be registered |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | VMs need a managed identity for Machine Configuration | **Yes** |
| 2 | The Guest Configuration extension is required | **Yes** |
| 3 | VMs need inbound network access on port 443 | **No** |
| 4 | The `Microsoft.GuestConfiguration` provider must be registered | **Yes** |

**In `challenge-32.md`:** lines **43**, **56**, **37**.

Row 3 is the architectural fact worth carrying: communication is **outbound only**. Machine
Configuration works on VMs with no public IP and no inbound rules — which is why it is acceptable on
locked-down production servers.

Rows 1 and 2 together are Break & fix Exercise 1, and their shared symptom is **"Pending"** rather
than a failure.

</details>

---

## Q28 — modes

| # | Statement | Answer |
|---|---|---|
| 1 | An `Audit` package corrects drift |  |
| 2 | `ApplyAndAutoCorrect` re-applies configuration on each evaluation |  |
| 3 | `ApplyAndMonitor` applies once, then only reports |  |
| 4 | The package type and policy mode must be compatible |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | An `Audit` package corrects drift | **No** |
| 2 | `ApplyAndAutoCorrect` re-applies configuration on each evaluation | **Yes** |
| 3 | `ApplyAndMonitor` applies once, then only reports | **Yes** |
| 4 | The package type and policy mode must be compatible | **Yes** |

**In `challenge-32.md`:** lines **143**, **191**, **551**.

Row 3 is the middle option the exam uses as a distractor. It is genuinely useful — apply a baseline
once, then observe deliberate deviations — and it is **not** continuous enforcement.

Row 4 is the pairing rule: capability comes from the **package**, permission from the **policy mode**.

</details>

---

## Q29 — remediation

| # | Statement | Answer |
|---|---|---|
| 1 | A remediation task fixes resources that were already non-compliant |  |
| 2 | An `Audit` effect policy can be remediated |  |
| 3 | The assignment identity needs an RBAC role to remediate |  |
| 4 | `ReEvaluateCompliance` re-scans before remediating |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A remediation task fixes resources that were already non-compliant | **Yes** |
| 2 | An `Audit` effect policy can be remediated | **No** |
| 3 | The assignment identity needs an RBAC role to remediate | **Yes** |
| 4 | `ReEvaluateCompliance` re-scans before remediating | **Yes** |

**In `challenge-32.md`:** lines **304–308**, **205**.

Row 1 explains why the feature exists: `DeployIfNotExists` acts on **create and update** events, so
pre-existing resources are flagged and otherwise untouched until a remediation task runs.

Row 2 follows from that: an `Audit` effect has no deployment to trigger, so there is nothing to
remediate.

</details>

---

## Q30 — Automation State Configuration

| # | Statement | Answer |
|---|---|---|
| 1 | Nodes pull their configuration from the Automation account |  |
| 2 | `ConfigurationModeFrequencyMins` controls how often configuration is applied |  |
| 3 | It is the recommended approach for new deployments |  |
| 4 | Compliance appears in Azure Policy compliance |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Nodes pull their configuration from the Automation account | **Yes** |
| 2 | `ConfigurationModeFrequencyMins` controls how often configuration is applied | **Yes** |
| 3 | It is the recommended approach for new deployments | **No** |
| 4 | Compliance appears in Azure Policy compliance | **No** |

**In `challenge-32.md`:** lines **276–286**, **251**, **291**.

Row 2 pairs with `RefreshFrequencyMins`, which controls how often the node **downloads** the
configuration. Download cadence and apply cadence are separate knobs.

Row 4 is the difference that matters operationally: node status lives in the **Automation account**
(line 291), not on the Policy compliance dashboard. Two services means two places to look, which is
part of why Machine Configuration supersedes it.

</details>

---

# Section E — Drag and drop

---

## Q31

Arrange the Machine Configuration authoring and publishing flow in order.

**Items:** Assign the policy with a managed identity · Compile the DSC configuration to MOF · Upload
the package and generate a SAS URI · Create the policy definition · Build the package with
`AuditAndSet` · Test the package locally

<details>
<summary>Show answer</summary>

### Answer

1. Compile the DSC configuration to MOF — line **137**
2. Build the package with `AuditAndSet` — line **140**
3. Test the package locally — line **151**
4. Upload the package and generate a SAS URI — lines **166–181**
5. Create the policy definition — line **185**
6. Assign the policy with a managed identity — line **200**

**Step 3 is the one people skip**, and it is the cheapest gate in the chain. Everything after it —
upload, SAS, definition, assignment, evaluation cycle — takes hours before you learn the package was
broken.

**Same principle as Challenge 31's ordering:** local checks first, cloud round-trips last.

</details>

---

## Q32

Match each mode to its behaviour.

| Mode or type | Behaviour |
|---|---|
| Package `-Type Audit` |  |
| Package `-Type AuditAndSet` |  |
| Policy `-Mode Audit` |  |
| Policy `-Mode ApplyAndMonitor` |  |
| Policy `-Mode ApplyAndAutoCorrect` |  |

**Options:** Applies on every evaluation · Applies once, then reports · Capable of reporting and correcting · Evaluates and reports · Reports compliance only

<details>
<summary>Show answer</summary>

| Mode or type | Behaviour |
|---|---|
| Package `-Type Audit` | **Reports compliance only** |
| Package `-Type AuditAndSet` | **Capable of reporting and correcting** |
| Policy `-Mode Audit` | **Evaluates and reports** |
| Policy `-Mode ApplyAndMonitor` | **Applies once, then reports** |
| Policy `-Mode ApplyAndAutoCorrect` | **Applies on every evaluation** |

**In `challenge-32.md`:** lines **143**, **191**, **551**.

**Read it as capability and permission.** The package type is what the configuration is *able* to do;
the policy mode is what it is *allowed* to do. Enforcement requires both — `AuditAndSet` **and**
`ApplyAndAutoCorrect`.

</details>

---

## Q33

Match each command to its purpose.

| Command | Purpose |
|---|---|
| `az policy state trigger-scan` |  |
| `az policy state summarize` |  |
| `az policy state list --filter ...NonCompliant` |  |
| `az policy remediation create` |  |
| `Test-GuestConfigurationPackage` |  |

**Options:** Aggregate compliance counts · Fix already non-compliant resources · Force immediate compliance evaluation · List failing resources · Validate a package locally before publishing

<details>
<summary>Show answer</summary>

| Command | Purpose |
|---|---|
| `az policy state trigger-scan` | **Force immediate compliance evaluation** |
| `az policy state summarize` | **Aggregate compliance counts** |
| `az policy state list --filter ...NonCompliant` | **List failing resources** |
| `az policy remediation create` | **Fix already non-compliant resources** |
| `Test-GuestConfigurationPackage` | **Validate a package locally before publishing** |

**In `challenge-32.md`:** lines **535**, **215**, **220**, **304**, **151**.

**The distinction that generates questions:** `trigger-scan` **evaluates**, `remediation create`
**fixes**. Evaluating a broken machine over and over changes nothing; remediating without a fresh
evaluation may act on stale data — which is why `ReEvaluateCompliance` combines both.

</details>

---

## Q34

Arrange the Break & fix Exercise 1 diagnosis and repair in order.

**Items:** Install the Guest Configuration extension · Trigger a compliance scan · Check for a managed
identity · Assign a managed identity · Check for the extension

<details>
<summary>Show answer</summary>

### Answer

1. Check for a managed identity — line **503**, returns `null`
2. Check for the extension — line **508**, returns empty
3. Assign a managed identity — line **522**
4. Install the Guest Configuration extension — line **527**
5. Trigger a compliance scan — line **535**

**Diagnose both before fixing either.** Assigning the identity alone leaves the machine still
Pending, so you would conclude the fix failed and start looking in the wrong place. Two missing
prerequisites, one symptom.

**Step 5 turns a 24-hour wait into a minute** — without it, the correct fix looks like it did not
work.

</details>

---

## Q35

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Compliance stuck at "Pending" for 24 hours |  |
| Dashboard shows compliant machines that clearly are not |  |
| Package works locally, fails on target machines |  |
| Remediation task fails with an authorisation error |  |
| Every assignment breaks on the same day |  |

**Options:** Assignment identity has no RBAC role · Dependent DSC module not bundled (`-FilesToInclude`) · Missing managed identity and/or extension · Package built as `Audit`, not `AuditAndSet` · The package SAS token expired

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| Compliance stuck at "Pending" for 24 hours | **Missing managed identity and/or extension** |
| Dashboard shows compliant machines that clearly are not | **Package built as `Audit`, not `AuditAndSet`** |
| Package works locally, fails on target machines | **Dependent DSC module not bundled (`-FilesToInclude`)** |
| Remediation task fails with an authorisation error | **Assignment identity has no RBAC role** |
| Every assignment breaks on the same day | **The package SAS token expired** |

**In `challenge-32.md`:** lines **501–511**, **551**, **554–573**, **205**, **179**.

**The last row is a scheduling trap.** A three-year SAS (line 179) feels permanent when you write it
and becomes an outage nobody predicted three years later. Put the expiry date in a calendar the day
you create it.

</details>

---

# Section F — Hot area

---

## Q36

```bash
az provider register --namespace [BLANK 1]

az vm identity assign --resource-group rg-contoso-vms --name vm-contoso-web-01

az vm extension set \
  --name [BLANK 2] \
  --publisher [BLANK 1] \
  --version 1.1
```

- **BLANK 1:** `Microsoft.GuestConfiguration` / `Microsoft.Compute` / `Microsoft.PolicyInsights` /
  `Microsoft.Automation`
- **BLANK 2:** `AzurePolicyforWindows` / `GuestConfigurationForWindows` / `DSCForWindows` /
  `AzureMonitorWindowsAgent`

<details>
<summary>Show answer</summary>

### Answer: `Microsoft.GuestConfiguration`, `AzurePolicyforWindows`

**In `challenge-32.md`:** lines **37** and **59–60**.

The namespace appears twice — as the provider to register and as the extension **publisher**. The
Linux equivalent extension is `AzurePolicyforLinux`.

</details>

---

## Q37

```powershell
New-GuestConfigurationPackage `
    -Name 'ContosoSecurityBaseline' `
    -Configuration './compiled/ContosoWindows.mof' `
    -Type [BLANK 1] `
    -FilesToInclude @( (Get-Module PSDscResources -ListAvailable).[BLANK 2] )
```

Requirement: the package must correct drift, and must work on machines that do not have the module
installed.

- **BLANK 1:** `AuditAndSet` / `Audit` / `Set` / `Enforce`
- **BLANK 2:** `ModuleBase` / `Path` / `Name` / `Version`

<details>
<summary>Show answer</summary>

### Answer: `AuditAndSet`, `ModuleBase`

**In `challenge-32.md`:** lines **143** and **572**.

Both halves of Break & fix Exercise 2 in one block. `Audit` reports without correcting; omitting
`-FilesToInclude` produces a package referencing a module the target machine has never seen.

</details>

---

## Q38

```powershell
New-GuestConfigurationPolicy `
    -ContentUri $packageUri `
    -Platform Windows `
    -Mode [BLANK 1] `
    -Version '1.0.0'

New-AzPolicyAssignment `
    -PolicyDefinition $definition `
    -Scope "/subscriptions/{sub-id}/resourceGroups/rg-contoso-vms" `
    -[BLANK 2] SystemAssigned `
    -Location 'eastus2'
```

- **BLANK 1:** `ApplyAndAutoCorrect` / `Audit` / `ApplyAndMonitor` / `Disabled`
- **BLANK 2:** `IdentityType` / `EnforcementMode` / `AssignIdentity` / `RoleDefinition`

<details>
<summary>Show answer</summary>

### Answer: `ApplyAndAutoCorrect`, `IdentityType`

**In `challenge-32.md`:** lines **191** and **205**.

`EnforcementMode` controls whether the effect is applied or only logged — a different concept that
sounds like the right answer. `-Location` is required alongside `-IdentityType` because a managed
identity is a regional resource.

</details>

---

## Q39

```powershell
        Registry 'DisableTLS10' {
            Ensure    = '[BLANK 1]'
            Key       = 'HKLM:\...\Protocols\TLS 1.0\Server'
            ValueName = 'Enabled'
            ValueType = 'DWord'
            ValueData = '[BLANK 2]'
        }

        Service 'WindowsFirewall' {
            Name        = '[BLANK 3]'
            State       = 'Running'
            StartupType = 'Automatic'
        }
```

- **BLANK 1:** `Present` / `Absent`
- **BLANK 2:** `0` / `1`
- **BLANK 3:** `MpsSvc` / `WinDefend` / `Firewall` / `WindowsFirewall`

<details>
<summary>Show answer</summary>

### Answer: `Present`, `0`, `MpsSvc`

**In `challenge-32.md`:** lines **89–93** and **114**.

`Ensure = 'Present'` with `ValueData = '0'` means the value must **exist and be zero** — explicitly
disabled. `Absent` would delete the value, and an absent value falls back to the default, which is
enabled.

`MpsSvc` is the service name; `WinDefend` is Microsoft Defender Antivirus, a different service.

</details>

---

## Q40

```bash
az policy remediation create \
  --name "remediate-contoso-baseline" \
  --policy-assignment "contoso-baseline-assignment" \
  --resource-discovery-mode [BLANK 1]

az policy state [BLANK 2] --resource-group rg-contoso-vms --no-wait
```

Requirements: re-check current state before fixing, and force an immediate evaluation afterwards.

- **BLANK 1:** `ReEvaluateCompliance` / `ExistingNonCompliant` / `AllResources` / `Incremental`
- **BLANK 2:** `trigger-scan` / `refresh` / `evaluate` / `summarize`

<details>
<summary>Show answer</summary>

### Answer: `ReEvaluateCompliance`, `trigger-scan`

**In `challenge-32.md`:** lines **308** and **535**.

`ExistingNonCompliant` is the **default** and acts on recorded state, which can be up to 24 hours old.

</details>

---

## Q41

```kusto
GuestConfigurationResources
| where type == "microsoft.guestconfiguration/guestconfigurationassignments"
| extend complianceStatus = properties.complianceStatus
| summarize
    Compliant = [BLANK 1](complianceStatus == "Compliant"),
    NonCompliant = [BLANK 1](complianceStatus == "NonCompliant"),
    Pending = [BLANK 1](complianceStatus == "Pending")
    by configName
| extend ComplianceRate = round([BLANK 2](Compliant) / [BLANK 2](Total) * 100, 1)
```

- **BLANK 1:** `countif` / `count` / `sumif` / `dcount`
- **BLANK 2:** `todouble` / `toint` / `tostring` / `tolong`

<details>
<summary>Show answer</summary>

### Answer: `countif`, `todouble`

**In `challenge-32.md`:** lines **241–246**.

`countif()` counts rows matching a condition — the same function as Challenge 27's A/B analysis.
`todouble()` prevents integer division truncating the rate to a whole number.

**And note the three buckets.** Counting only Compliant and NonCompliant would silently exclude
`Pending` machines — the ones with the missing identity or extension, which are precisely the ones you
need to find.

</details>

---

# Section G — Case study

## Case study: Contoso configuration compliance

### Background

Contoso runs **50 Windows Server VMs and 20 Linux VMs**. Operations staff make ad-hoc changes during
incidents, and the security team reports **40% of VMs fail the monthly compliance scan**.

### Required baseline

- TLS 1.2 enforced, TLS 1.0 and 1.1 disabled
- Windows Firewall enabled with specific rules
- Required software installed; prohibited software removed
- Password policy: minimum length 14, complexity enabled
- Audit logging configured

### Requirements

**Enforcement**

- Drift must be corrected automatically, not merely reported
- Configuration must be authored as code and deployed through a pipeline
- The same baseline must apply to all 70 VMs without per-machine setup

**Visibility**

- The security team needs a fleet-wide compliance percentage
- Machines that have never reported must be distinguishable from failing ones

**Operations**

- No credential may be stored for remediation
- Existing non-compliant VMs must be brought into compliance, not only new ones

---

## Q42

Which technology should Contoso use for the baseline?

- A. Azure Machine Configuration via Azure Policy
- B. Azure Automation State Configuration
- C. A scheduled PowerShell script on each VM
- D. Bicep templates

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** lines **33** and **251**.

**Why the others fail**

- **B** — line 251 calls it legacy and superseded. It also reports to the Automation account rather
  than to Policy compliance, which fails the fleet-wide visibility requirement
- **C** — a script per VM is per-machine setup, has no compliance model, and needs credentials
- **D** — **the important distinction.** Bicep configures the **resource** — VM size, disks,
  networking. It cannot reach **inside** the guest OS to set a registry key or start a service. Guest
  configuration is a different layer entirely

</details>

---

## Q43

Which **two** settings ensure drift is corrected rather than reported? (Choose two.)

- A. Package `-Type AuditAndSet`
- B. Policy `-Mode ApplyAndAutoCorrect`
- C. Package `-Type Audit`
- D. Policy `-Mode ApplyAndMonitor`
- E. `-EnforcementMode DoNotEnforce`

<details>
<summary>Show answer</summary>

### Answer: A, B

**In `challenge-32.md`:** lines **143** and **191**.

**Why the others fail**

- **C** — reports only. Break & fix Exercise 2
- **D** — applies **once**, then only reports. Since the drift source is ongoing operator activity,
  the very next incident undoes it and nothing corrects it again
- **E** — `DoNotEnforce` logs what *would* happen without doing it — policy's equivalent of what-if

</details>

---

## Q44

How should the same baseline reach all 70 VMs without per-machine setup?

- A. Assign the policy at resource group or subscription scope, and use a built-in policy to add
  identities
- B. Run `az vm extension set` on each VM individually
- C. Create one policy assignment per VM
- D. Add each VM to an Automation account as a DSC node

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** lines **49–53** and **204**.

```bash
az policy assignment create \
  --name "assign-vm-identity" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/3cf2ab00-..." \
  --scope "/subscriptions/{subscription-id}/resourceGroups/rg-contoso-vms"
```

**Policy at scope is what makes this scale**, and it applies to VMs created **later** as well — a
machine built next month inherits the baseline without anyone remembering.

**Why the others fail**

- **B** — 70 commands, and 71 when someone adds a VM
- **C** — 70 assignments to maintain
- **D** — the legacy service, plus per-node registration

</details>

---

## Q45

Which query gives the security team a fleet-wide compliance percentage that distinguishes
never-reported machines?

- A. A Resource Graph query over `GuestConfigurationResources` counting Compliant, NonCompliant and
  Pending
- B. `az vm list --output table`
- C. `az policy definition list`
- D. A Log Analytics query over `Heartbeat`

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** lines **235–246**.

**The three buckets are the requirement.** `Pending` means the machine has never reported — usually
the missing identity or extension from Break & fix Exercise 1. Folding it into either of the other
two hides a real gap: a fleet showing "60% compliant, 40% non-compliant" reads very differently from
"60% compliant, 25% non-compliant, 15% never reported".

**Why the others fail** — B lists VMs, C lists definitions, D shows agent heartbeats from a different
service entirely.

</details>

---

## Q46

Which configuration remediates the VMs that are **already** non-compliant?

- A. `az policy remediation create` with `ReEvaluateCompliance`
- B. Re-assigning the policy
- C. `az policy state trigger-scan`
- D. Waiting for the next evaluation cycle

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** lines **304–308**.

**Why the others fail**

- **B** — a fresh assignment still only acts on create and update events
- **C** — evaluates and reports; it corrects nothing
- **D** — the next cycle re-evaluates the same machines and finds them non-compliant again

**The underlying rule:** `DeployIfNotExists` and `Modify` effects trigger on resource **create or
update**. Everything that already existed when you assigned the policy needs a remediation task to
reach it.

</details>

---

## Q47

Which configuration meets the no-stored-credential requirement for scheduled remediation?

- A. An Automation runbook using `Connect-AzAccount -Identity`
- B. A runbook with a service principal secret in an Automation variable
- C. A pipeline using `AZURE_CREDENTIALS`
- D. A scheduled task on each VM using a stored password

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** line **332**.

```powershell
Connect-AzAccount -Identity
```

The Automation account's **managed identity** — no secret created, stored or rotated.

**Why the others fail**

- **B** — an encrypted Automation variable is better than plaintext and is still a stored secret that
  expires and must be rotated
- **C** — a stored service principal credential; the OIDC pattern from Challenge 31 is the pipeline
  equivalent of `-Identity`
- **D** — a password on 70 machines

**Third appearance of the same principle**, across three different services: OIDC for GitHub Actions,
`-Identity` for Automation, system-assigned identity for the policy assignment. **Identity over stored
secret** — the domain that has cost you marks in every mock.

</details>

---

## Q48

Six months on, compliance sits at 95%. The remaining 5% are Linux VMs showing `Pending`, and the team
has verified they have managed identities and the extension installed.

What is the most likely cause?

- A. The policy targets `-Platform Windows`, so Linux machines are never evaluated
- B. Linux VMs do not support Machine Configuration
- C. The SAS token expired
- D. The Linux VMs need a reboot

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-32.md`:** line **190**.

```powershell
New-GuestConfigurationPolicy `
    -Platform Windows `
```

**The baseline was authored for Windows only**, and the configuration proves it: `Registry`, a
`WindowsFeature`, and the `MpsSvc` service (lines 88–132) have no Linux equivalent.

**What Contoso actually needs** is a second package and policy: `-Platform Linux`, a configuration
built from Linux DSC resources, and the `AzurePolicyforLinux` extension. The **requirements** are the
same — TLS, firewall, password policy — but the **implementation** is entirely different.

**Why the others fail**

- **B** — Machine Configuration supports Linux. The scenario at line 20 has 20 Linux VMs for exactly
  this reason
- **C** — an expired SAS would break Windows machines too
- **D** — reboots do not change platform targeting

**The wider lesson:** a "fleet-wide baseline" in a mixed estate is at least two baselines. The gap
hides because Windows machines report healthily and the 5% looks like a rounding error rather than an
entire unprotected platform.

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **`Audit` used where enforcement is required** | Q3, Q18, Q25, Q43 | `AuditAndSet` package **and** `ApplyAndAutoCorrect` policy |
| **`ApplyAndMonitor` mistaken for auto-correct** | Q4, Q28, Q43 | It applies once, then only reports |
| **Assignment with no managed identity** | Q6, Q19, Q26 | No identity means audit-only and failed remediation |
| **Identity created but no RBAC role** | Q19, Q29 | The grant is a separate step |
| **"Pending" read as a failure** | Q2, Q15, Q45 | Pending means never reported — check identity and extension |
| **Dependent module not bundled** | Q16, Q37 | `-FilesToInclude` with the module base path |
| **`trigger-scan` confused with remediation** | Q7, Q13, Q33, Q46 | Scan evaluates. Remediation fixes |
| **Bicep offered for guest OS settings** | Q42 | Bicep configures the resource, not inside the VM |
| **Automation State Configuration as current** | Q11, Q21, Q30 | Legacy, superseded by Machine Configuration |
| **Inbound network access assumed** | Q17, Q27 | The agent connects outbound only |
| **`-Platform Windows` on a mixed fleet** | Q48 | Linux needs its own package and policy |
| **SAS expiry ignored** | Q5, Q35 | Every assignment breaks the day it expires |

---

# The blocks to memorise

Line numbers are in `challenge-32.md`.

```bash
# 1. Prerequisites - all three, or the machine sits at "Pending"  (lines 37-61)
az provider register --namespace Microsoft.GuestConfiguration
az vm identity assign --resource-group rg-contoso-vms --name vm-contoso-web-01
az vm extension set --name AzurePolicyforWindows \
  --publisher Microsoft.GuestConfiguration --version 1.1
az policy state trigger-scan --resource-group rg-contoso-vms --no-wait
```

```powershell
# 2. Package - capability  (lines 140-144, 566-573)
New-GuestConfigurationPackage `
    -Configuration './compiled/ContosoWindows.mof' `
    -Type AuditAndSet `
    -FilesToInclude @( (Get-Module PSDscResources -ListAvailable).ModuleBase )

# 3. Test locally BEFORE publishing  (lines 151-154)
Test-GuestConfigurationPackage -Path './...zip'
$result.resources | Format-Table ResourceName, InDesiredState, Reasons

# 4. Policy - permission  (lines 185-206)
New-GuestConfigurationPolicy -ContentUri $packageUri -Platform Windows `
    -Mode ApplyAndAutoCorrect -Version '1.0.0'
New-AzPolicyAssignment -PolicyDefinition $definition -Scope <scope> `
    -IdentityType SystemAssigned -Location 'eastus2'
# then grant that identity an RBAC role at the scope

# 5. DSC resources used  (lines 88-132)
Registry  { Ensure='Present'; ValueName='Enabled'; ValueType='DWord'; ValueData='0' }  # TLS off
Service   { Name='MpsSvc'; State='Running'; StartupType='Automatic' }                  # firewall
WindowsFeature { Ensure='Present'; Name='...' }
```

```bash
# 6. Compliance and remediation  (lines 215-223, 304-308)
az policy state summarize --filter "policyDefinitionName eq '...'"
az policy state list --filter "... and complianceState eq 'NonCompliant'"
az policy remediation create --policy-assignment <name> \
  --resource-discovery-mode ReEvaluateCompliance
```

```kusto
// 7. Fleet compliance - three buckets, not two  (lines 235-246)
GuestConfigurationResources
| summarize Compliant = countif(complianceStatus == "Compliant"),
            NonCompliant = countif(complianceStatus == "NonCompliant"),
            Pending = countif(complianceStatus == "Pending") by configName
| extend ComplianceRate = round(todouble(Compliant) / todouble(Total) * 100, 1)
```

```powershell
# 8. Automation State Configuration - LEGACY  (lines 276-286)
Register-AzAutomationDscNode `
    -ConfigurationMode 'ApplyAndAutoCorrect' `
    -RefreshFrequencyMins 30 `             # how often it DOWNLOADS
    -ConfigurationModeFrequencyMins 15     # how often it APPLIES
```

**Enforcement needs both halves:** `AuditAndSet` (package) + `ApplyAndAutoCorrect` (policy) +
an assignment identity with an RBAC role.

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Challenge 32 is exam-ready. Move to Challenge 33 |
| 38–43 | Solid. Re-read the trap index, then move on |
| 30–37 | Redo Tasks 2 and 3, then retake this |
| Below 30 | Redo the challenge, writing the package-type and policy-mode pairing from memory |

Record your result in `AZ-400-Learning-Log.md` under Challenge 32.

:::tip The one thing

**Capability comes from the package; permission comes from the policy; the ability to act comes from
the identity.**

`AuditAndSet` + `ApplyAndAutoCorrect` + a managed identity with an RBAC role. Miss any one and you get
a dashboard instead of enforcement.

:::
