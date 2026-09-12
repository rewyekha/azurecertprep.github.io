---
sidebar_position: 6.5
toc_max_heading_level: 2
title: "Challenge 06: exam questions"
sidebar_label: "Exam questions (48 Q)"
---

# Challenge 06 — AZ-400 exam questions

**48 questions** built only from what Challenge 06 covers, in the seven shapes the live exam uses.

Every question is followed by its answer, **why each wrong option is wrong**, and the **line numbers
in `challenge-06.md`**.

:::danger Read this before you start

Every integration question is answered by **direction** and **trigger**. Ask: *who initiates, and on
what event?*

**Outbound** — a **webhook**. GitHub or Azure DevOps sends a POST to a URL you own when something
happens. You receive.
**Inbound** — **`repository_dispatch`**. An external system POSTs to the GitHub API to start a
workflow, optionally carrying a `client_payload`. You are called.
**Bidirectional** — the **Azure Boards ↔ GitHub** integration. A commit or PR references `AB#`, and the
work item links back.

Two facts carry most of the marks.

**A webhook's authenticity comes from a signature, not from the URL.** GitHub HMACs the payload with
your secret and sends `X-Hub-Signature-256`. A receiver that does not verify it will accept anything
anyone POSTs to that address.

**`AB#` needs two halves.** The Azure Boards GitHub App installed on the repository, **and** the
repository connected in Azure DevOps under Boards > GitHub connections. One without the other fails
silently.

The scenario at line 22 is three tools with nothing between them: PMs checking Boards by hand, on-call
missing deployment failures because alerts go to unread email, and developers context-switching with no
cross-links.

:::

---

# Section A — Multiple choice

---

## Q1

What is the purpose of the `X-Hub-Signature-256` header?

- A. It identifies which user account triggered the event
- B. It encrypts the payload so it cannot be read in transit
- C. It carries an HMAC-SHA256 of the payload body
- D. It specifies the webhook API version the sender used

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-06.md`:** lines **588–592** and **599**.

```javascript
function verifySignature(payload, signature, secret) {
  const hmac = crypto.createHmac('sha256', secret);
  const digest = 'sha256=' + hmac.update(payload).digest('hex');
  return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(digest));
}
```

**Sign, do not encrypt — and the distinction is the question.** The payload travels in clear text over
TLS; the signature proves it came from someone holding the shared secret and was not altered.

**Why the comparison uses `timingSafeEqual` and not `===`.** A normal string comparison returns as soon
as it finds a difference, so an attacker can measure response times to discover the correct digest byte
by byte. A constant-time comparison removes that channel.

**A webhook endpoint is a public URL.** Without this check, anyone who learns the address can POST a
fake "deployment failed" event.

</details>

---

## Q2

What is the difference between a webhook and a `repository_dispatch` event?

- A. They are the same mechanism under two different names
- B. Webhooks are faster than a dispatch round trip call
- C. Webhooks support all events; dispatch supports only push
- D. Webhooks send data out; dispatch triggers workflows in

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-06.md`:** lines **143–148** and **194**.

```bash
gh api repos/{owner}/contoso-webapp/hooks --method POST \
  --field config='{"url":"https://contoso-webhook-receiver.azurewebsites.net/api/github", ... }'
```

```bash
gh api repos/{owner}/contoso-webapp/dispatches --method POST \
  --field event_type="deploy-staging" \
  --field client_payload='{"ref":"feature/payment-retry", ... }'
```

**Read the two commands: one registers a URL for GitHub to call, the other calls GitHub.** Outbound
versus inbound.

**And `client_payload` is what makes the inbound direction useful** (line 260). The external system can
pass a ref, a version, a reason — data the workflow reads as
`github.event.client_payload.*` (line 216).

</details>

---

## Q3

In Azure DevOps service hooks, what determines which events fire the hook?

- A. Publisher inputs that filter which events fire
- B. The repository's branch configuration in Azure Repos
- C. The consumer endpoint's declared capabilities
- D. The Azure subscription tier the project sits on

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-06.md`:** lines **472–476** and **498–501**.

```json
  "publisherInputs": {
    "pipelineId": "",
    "runStateId": "4",
    "runResultId": "2"
  }
```

```json
  "publisherInputs": {
    "areaPath": "Contoso Web Platform",
    "workItemType": "Bug"
  }
```

**Publisher = the source and its filters. Consumer = the destination.** The subscription is built from
four pieces: `publisherId`, `eventType`, `consumerId` and `consumerActionId` (lines 468–471).

**And an empty filter means "all".** `"pipelineId": ""` at line 473 subscribes to **every** pipeline —
which is either exactly what you want or the reason a channel becomes unreadable.

</details>

---

## Q4

A commit contains `AB#5678` but the work item shows no link. What is the most likely cause?

- A. The commit message format is subtly wrong
- B. The work item must be in the Active state
- C. The Boards app or repo link is missing
- D. Azure Boards links only from PR descriptions

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-06.md`:** lines **898–906**.

```text
**Fix:** The Azure Boards GitHub App may have lost access due to organization permission changes.
Re-authorize the app in GitHub organization settings, and verify the repository is still linked in
Azure DevOps Project Settings > GitHub connections.
```

**Two halves of one handshake** — and the exam tests this in every challenge that touches Boards.

**Why D is refuted at line 109**, where the commit message itself carries `AB#${WORK_ITEM_ID}`. Commits
and PR descriptions both work.

**And note the failure is silent.** The commit is accepted, the syntax looks right, and nothing appears
— which is why Break scenario 3 exists and why the diagnosis at lines 890–899 checks both sides.

</details>

---

## Q5

Webhook deliveries return 401. What is the cause?

- A. The secret in GitHub differs from the receiver's
- B. The Function App receiving the webhook is stopped
- C. The event type sent is not supported by the receiver
- D. TLS certificate verification failed on the connection

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-06.md`:** lines **836** and **604–609**.

```javascript
    const secret = process.env.GITHUB_WEBHOOK_SECRET;
    if (secret && signature) {
      if (!verifySignature(body, signature, secret)) {
        context.log('Webhook signature verification failed');
        return { status: 401, body: 'Invalid signature' };
      }
    }
```

**The 401 comes from your own code**, which is what makes the diagnosis straightforward: the receiver
computed a different digest, so the secrets differ.

**And the fix must update both sides atomically** (lines 840–851): generate a new secret, PATCH the
GitHub webhook config, then set the Function App setting. Change one and every delivery fails.

**Note the condition at line 605: `if (secret && signature)`.** If the environment variable is unset,
verification is **skipped entirely** — the endpoint silently accepts unsigned requests. That is a
deployment-time failure mode worth spotting.

</details>

---

## Q6

Teams notifications stop arriving although the receiver logs success. What is the cause?

- A. The Function App has no outbound network access
- B. The adaptive card schema in the payload is wrong
- C. The Teams channel has been archived by an owner
- D. The Teams incoming webhook URL is invalid

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-06.md`:** line **878**.

```text
**Fix:** Teams incoming webhook URLs expire when the connector is removed or the channel is deleted.
Recreate the connector in Teams and update the `TEAMS_WEBHOOK_URL` app setting.
```

**"Logs show successful processing" is the diagnostic detail.** The receiver did its job; the outbound
call is what failed.

**And look at `notifyTeams` at lines 740–744** — the `fetch` result is never checked. A dead webhook
returns an error the function never inspects, so the log line says success either way.

**The direct test at lines 869–871 is the right first move**: POST a minimal card to the URL and see
what comes back. It isolates Teams from everything else in one command.

</details>

---

## Q7

Which events does the initial webhook subscribe to?

- A. `push`, `pull_request`, `issues`, `deployment_status`
- B. All events, using the wildcard subscription form instead
- C. `push` only, filtered to the repository default branch
- D. `workflow_run` and `release` events only

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-06.md`:** line **147**.

```bash
  --field events='["push","pull_request","issues","deployment_status"]' \
```

**Subscribing to named events rather than everything is the design choice.** A wildcard subscription
delivers hundreds of event types you will never handle, and every one costs a delivery your receiver
must parse and discard.

**Why D is what gets *added* later** (line 165) with `add_events` — and note the API has a matching
`remove_events` at line 170. Both are PATCH operations on the existing hook rather than a recreate.

</details>

---

## Q8

What does the `pings` endpoint do?

- A. It measures the network latency to the receiver
- B. It re-sends the most recent webhook delivery
- C. It sends a synthetic `ping` event to the receiver
- D. It validates the configured webhook secret value

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-06.md`:** lines **154–156** and **629–631**.

```bash
gh api repos/{owner}/contoso-webapp/hooks/$HOOK_ID/pings --method POST
```

```javascript
      case 'ping':
        context.log('Ping received - webhook is configured correctly');
        break;
```

**The receiver handles `ping` explicitly**, which matters: an unhandled event falls to `default` and
logs "Unhandled event type" (line 633) — so without that case the test looks like a failure.

**Why B is the adjacent operation.** Re-sending a specific delivery is the `attempts` endpoint at line
186 — that is redelivery; this is a fresh synthetic event.

</details>

---

## Q9

How do you inspect why a webhook delivery failed?

- A. Read the repository's organisation audit log entries
- B. Query `deliveries`, then fetch that delivery's body
- C. Check the Actions run log for the workflow
- D. Enable debug logging on the webhook itself

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-06.md`:** lines **177–183** and **824–829**.

```bash
gh api repos/{owner}/contoso-webapp/hooks/$HOOK_ID/deliveries \
  --jq '.[] | {id: .id, event: .event, status_code: .status_code, delivered_at: .delivered_at}'
```

```bash
gh api repos/{owner}/contoso-webapp/hooks/$HOOK_ID/deliveries/{delivery_id} \
  --jq '.response.body'
```

**The list gives you status codes; the individual delivery gives you the request *and* the response.**
That pairing is what makes webhook debugging tractable — you can see exactly what was sent and exactly
what came back.

**And `attempts` re-delivers the same payload** (line 186), so once you fix the receiver you can replay
the failure rather than waiting for the event to happen again.

</details>

---

## Q10

What does `repository_dispatch` use to route work to the right job?

- A. The name of the branch the dispatch targets
- B. The size of the `client_payload` object sent
- C. The filename of the workflow being dispatched
- D. `github.event.action` matched against `types`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-06.md`:** lines **202–207**.

```yaml
on:
  repository_dispatch:
    types: [deploy-staging, deploy-production, run-tests]

jobs:
  deploy-staging:
    if: github.event.action == 'deploy-staging'
```

**`types:` declares which event types this workflow accepts; the `if:` on each job selects one.** The
`event_type` sent by the caller (line 259) becomes `github.event.action`.

**Which is worth noticing: the field is called `event_type` on the way in and `action` on the way out.**
The exam uses that mismatch.

**Omit a type from the `types:` list and the dispatch is accepted by the API and runs nothing** — no
error to the caller.

</details>

---

## Q11

What does `${{ github.event.client_payload.ref || 'main' }}` accomplish?

- A. It merges the supplied ref into the `main` branch
- B. It checks out the caller's ref, or `main`
- C. It validates that the supplied ref actually exists
- D. It creates a new branch from the supplied ref name

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-06.md`:** lines **211–212**.

```yaml
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.client_payload.ref || 'main' }}
```

**A default for an optional payload field.** The caller may or may not send `ref`; without the fallback,
an omitted value produces an empty string and the checkout fails.

**And this is the security surface of `repository_dispatch` worth thinking about.** Whoever can call the
API chooses what gets checked out and deployed — which is why the production job at line 224 declares
`environment: production`, so the environment's protection rules still apply.

</details>

---

## Q12

Which job in the Teams workflow fires only on a failed deployment?

- A. `notify-deployment-failure`, on `state == 'failure'`
- B. `notify-ci-failure`, gated on the workflow run
- C. `notify-pr`, gated on the pull request event type fired
- D. `notify-release`, gated on a published release

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-06.md`:** line **352**.

```yaml
    if: github.event_name == 'deployment_status' && github.event.deployment_status.state == 'failure'
```

**Two clauses: which event, and which state within it.** The workflow has three triggers (lines
287–293), so every job must first establish that its own event fired.

**And compare the CI job at line 400**, which checks `workflow_run.conclusion == 'failure'`. **Deployment
statuses have a `state`; workflow runs have a `conclusion`** — the same two-vocabulary trap as Challenge
04.

</details>

---

## Q13

What is the Teams message format used throughout this challenge?

- A. A legacy `MessageCard` payload object
- B. Plain text supplied in the request body
- C. Markdown rendered by the connector
- D. An Adaptive Card in a message attachment

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-06.md`:** lines **535–542**.

```json
{
  "type": "message",
  "attachments": [
    {
      "contentType": "application/vnd.microsoft.card.adaptive",
      "content": {
        "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
        "type": "AdaptiveCard",
        "version": "1.4",
```

**Adaptive Cards are the current format; `MessageCard` is the legacy one.** The exam offers both because
older documentation is full of `MessageCard`.

**And the structure is worth recognising on sight:** a `body` of `TextBlock` and `FactSet` elements, then
`actions` containing an `Action.OpenUrl` — a title, a set of facts, and a link (lines 322–344).

</details>

---

## Q14

What makes an alert card visually urgent?

- A. `"color": "Attention"` on the TextBlock
- B. A red background colour on the card body
- C. `"priority": "high"` on the message envelope
- D. An exclamation mark at the start of the title

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-06.md`:** lines **372–375** and **724**.

```json
                      "text": "DEPLOYMENT FAILED",
                      "weight": "Bolder",
                      "size": "Medium",
                      "color": "Attention"
```

```javascript
            color: urgent ? 'Attention' : 'Default'
```

**The Function generalises it into a parameter** (line 724) so one card builder serves both routine and
urgent notifications.

**Which is the design point for the on-call problem at line 22.** Alerts that arrive in email nobody
reads after hours are invisible; a card in the channel, visually distinct, is not.

</details>

---

## Q15

How does the Function decide whether to forward a push to Teams?

- A. On every push the webhook delivers to it
- B. Only when more than five commits are pushed
- C. Only when the branch the push targets is `main`
- D. Only on pushes that create or update a tag

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-06.md`:** lines **641–658**.

```javascript
  const branch = payload.ref.replace('refs/heads/', '');
  ...
  if (branch === 'main') {
    await notifyTeams({ ... });
  }
```

**Filtering at the receiver, not at the subscription.** The webhook is subscribed to all pushes (line
147) and the Function decides which are worth a notification.

**And note `payload.ref.replace('refs/heads/', '')`** — the payload carries the **full ref**, not a bare
branch name. That is the same detail as Challenge 39's federated credential subjects.

</details>

---

## Q16

Which header tells the receiver what kind of event it received?

- A. `X-GitHub-Delivery`, unique per delivery
- B. `X-GitHub-Event`, naming the event type
- C. `X-Hub-Signature-256`, the payload HMAC
- D. `Content-Type`, describing the body format

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-06.md`:** lines **599–601**.

```javascript
    const signature = request.headers.get('x-hub-signature-256');
    const event = request.headers.get('x-github-event');
    const deliveryId = request.headers.get('x-github-delivery');
```

**Three headers, three jobs.** `X-GitHub-Event` is the type and drives the `switch` at line 616.
`X-GitHub-Delivery` is a unique ID — useful for correlating a log line with a delivery in the API (Q9),
and for detecting a redelivery of the same event.

**Why C is the authenticity header** (Q1), not the routing one.

</details>

---

# Section B — Multiple answer

---

## Q17

Which **three** are required for `AB#` linking to work? (Choose three.)

- A. A PAT for Azure DevOps stored in repository secrets
- B. The Azure Boards GitHub App installed on the repository
- C. Branch protection enabled on the default branch
- D. The repository linked under Boards > GitHub connections
- E. The work item sitting in the Active state already
- F. The app's authorisation still valid for the organisation

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-06.md`:** lines **45–46**, **62**, **906**.

**B and D are the two halves of the handshake; F is the one that breaks *later*.** Break scenario 3
exists because an installation that worked can lose access when organisation permissions change — the
integration was correct on the day it was built and is not correct now.

**Which is why the diagnosis checks both sides** (lines 890–899): the connection in Azure DevOps, and
the app installation in GitHub.

</details>

---

## Q18

Which **three** does the webhook `deliveries` API let you do? (Choose three.)

- A. List deliveries together with their status codes
- B. Edit the payload before redelivering the event
- C. Inspect one delivery's request and response
- D. Delete a delivery record from the history
- E. Redeliver a delivery that previously failed
- F. Change the webhook's shared secret value

<details>
<summary>Show answer</summary>

### Answer: A, C, E

**In `challenge-06.md`:** lines **177–187**.

```bash
gh api .../hooks/$HOOK_ID/deliveries                              # list
gh api .../hooks/$HOOK_ID/deliveries/$DELIVERY_ID                 # inspect
gh api .../hooks/$HOOK_ID/deliveries/$DELIVERY_ID/attempts --method POST   # redeliver
```

**Redelivery is what makes a receiver outage recoverable.** Fix the endpoint, replay the events you
missed, and no data is lost.

**Why B is deliberately impossible.** The payload is signed (Q1) — an editable redelivery would either
break the signature or let anyone forge events through GitHub's own API.

</details>

---

## Q19

Which **three** appear in an Adaptive Card used here? (Choose three.)

- A. An embedded image in the card body
- B. A `TextBlock` holding the card title
- C. An input form for a reply
- D. A `FactSet` of name and value pairs
- E. A chart rendered from metrics
- F. An `Action.OpenUrl` link to the run

<details>
<summary>Show answer</summary>

### Answer: B, D, F

**In `challenge-06.md`:** lines **322–344**.

**Title, facts, link — that is the whole anatomy of a useful alert.** Enough to triage from a phone
without opening anything, and one tap to the detail.

**Compare with the same three ideas in Challenge 48's Slack payload**: a header, a fields block and a
button. **Different vendor, identical structure**, because the requirement is the same.

</details>

---

## Q20

Which **two** distinguish outbound from inbound integration? (Choose two.)

- A. Both directions are initiated by the platform
- B. A webhook is outbound — the platform POSTs to your URL
- C. Both directions are initiated by the external system
- D. `repository_dispatch` is inbound — the caller POSTs in
- E. Webhooks always require a payload body

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-06.md`:** lines **143–148**, **257–260**, **927**.

**"Who initiates?" is the whole distinction**, and it decides which credential is involved: an outbound
webhook proves itself with a **signature**; an inbound dispatch proves itself with a **token** (line
270).

**Two directions, two authentication models** — which is worth holding onto, because the exam asks
"how do you secure this integration" and the answer depends entirely on the direction.

</details>

---

## Q21

Which **two** are true of Azure DevOps service hooks? (Choose two.)

- A. They can only target Microsoft Teams as a consumer
- B. Publisher inputs filter which events fire the hook
- C. They are configured per repository branch in Repos
- D. They require an Azure subscription to be present
- E. A subscription names publisher, event, consumer, action

<details>
<summary>Show answer</summary>

### Answer: B, E

**In `challenge-06.md`:** lines **468–482**.

```json
  "publisherId": "pipelines",
  "eventType": "ms.vss-pipelines.run-state-changed-event",
  "consumerId": "webHooks",
  "consumerActionId": "httpRequest",
```

**Four fields describe the whole subscription: what emits, which event, what receives, and how.**

**Why A is wrong and useful to know.** `consumerId: "webHooks"` with `consumerActionId: "httpRequest"`
sends to **any** HTTP endpoint — which is why this challenge points it at the same Azure Function that
receives GitHub's webhooks (line 478). One receiver, two sources.

</details>

---

## Q22

Which **two** protect a webhook receiver? (Choose two.)

- A. Verifying `X-Hub-Signature-256` with the shared secret
- B. Restricting the endpoint to a range of source IPs
- C. Requiring a password passed in the query string
- D. A constant-time comparison of the two digests
- E. Setting the function to `authLevel: 'function'`

<details>
<summary>Show answer</summary>

### Answer: A, D

**In `challenge-06.md`:** lines **588–591** and **606–608**.

```javascript
  return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(digest));
```

**A proves the sender; D stops the proof being reverse-engineered by timing** (Q1).

**Why E is a real and different control worth understanding.** The functions are created with
`--authlevel anonymous` (lines 575–578) **on purpose** — GitHub cannot add a function key to its
requests, so the endpoint must be anonymous and the **signature** is what secures it.

**Why D is what people build instead and why it is weaker.** A secret in the query string lands in
access logs and proxy logs; the HMAC never travels.

</details>

---

## Q23

Which **two** problems from the scenario do these integrations solve? (Choose two.)

- A. Slow CI builds on every pull request
- B. PMs manually checking Boards for status
- C. Flaky tests failing at random in CI
- D. On-call missing deploy failures in email
- E. Large repository clone times for new joiners

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-06.md`:** line **22**, with **114–133** and **351–397**.

**B is solved by `Fixes AB#`, which transitions the work item on merge** — the PM sees the state change
without asking anyone.

**D is solved by routing `deployment_status` failures to a Teams channel** rather than to email. The
alert arrives where the team already is.

**Why A, C and E belong to Challenges 35, 34 and 10.** The exam mixes symptoms across domains; attribute
each to its own mechanism.

</details>

---

# Section C — Repeated scenario

**Scenario:** Contoso must connect GitHub, Azure Boards and Teams so work items transition automatically,
deployment failures reach the on-call channel immediately, external systems can trigger deployments, and
every integration is authenticated.

---

## Q24

**Proposed solution:** Install the Azure Boards GitHub App and link the repository in Azure DevOps, then
use `Fixes AB#` in commits and PRs. Create a GitHub webhook for push, pull_request, issues and
deployment_status pointing at an Azure Function, with a shared secret, and verify
`X-Hub-Signature-256` using a constant-time comparison. Add a `repository_dispatch` workflow with typed
events and `client_payload`, keeping the production job on a protected environment. Post Adaptive Cards
to a Teams incoming webhook on deployment failure and CI failure. Add Azure DevOps service hooks
filtered by publisher inputs, targeting the same Function.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: Yes

**In `challenge-06.md`:** lines **45–133**, **143–148**, **588–609**, **202–228**, **351–397**,
**461–508**.

| Requirement | Mechanism |
|---|---|
| Work items transition automatically | Boards app + connection + `Fixes AB#` |
| Failures reach the channel | `deployment_status` → Adaptive Card in Teams |
| External systems can deploy | `repository_dispatch` with typed events |
| Production still gated | `environment: production` on that job |
| Every integration authenticated | HMAC signature in, token out |

**The `environment: production` clause matters more than it looks.** `repository_dispatch` lets an
external caller start a production deployment — naming the environment means the protection rules still
apply (Q11).

</details>

---

## Q25

**Proposed solution:** Install the Azure Boards GitHub App. Create a webhook with no secret, since the
Function URL is unguessable. Have the Function post to Teams. Use `workflow_dispatch` so external
systems can trigger deployments. Send every GitHub event to the channel so nothing is missed.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

**Four failures.**

**Installing the app without the Azure DevOps connection leaves `AB#` inert** (Q4, Q17) — and it fails
silently, so the PM team will believe it works until someone checks a work item.

**"The URL is unguessable" is not a security control.** It appears in the webhook configuration, in
delivery logs, in anyone's shell history, and in the Function's own settings. Without the secret, the
receiver accepts any POST — including a forged "deployment succeeded".

**`workflow_dispatch` is the manual trigger**, designed for a human clicking Run workflow.
`repository_dispatch` is the external-system trigger and the one that carries `client_payload` (Q2).

**And "every event to the channel" recreates the unread-email problem in a new medium.** The reason
on-call misses failures at line 22 is volume, not channel — a Teams channel receiving every push,
issue comment and label change is muted within a week.

</details>

---

## Q26

**Proposed solution:** Install the Azure Boards GitHub App and link the repository in Azure DevOps, and
use `Fixes AB#`. Create a GitHub webhook for the four named events with a shared secret. Verify the
signature in the Function. Add a `repository_dispatch` workflow with typed events and `client_payload`.
Post Adaptive Cards to Teams on deployment and CI failures. Add filtered Azure DevOps service hooks.
Compare the computed digest with the header using a standard string equality check, which is simpler and
produces the same result.

Does this meet the goal?

<details>
<summary>Show answer</summary>

### Answer: No

Everything is right except the last sentence — and compare it with Q24, which is otherwise identical.

**A standard `===` comparison returns as soon as it finds a mismatched byte**, so how long it takes
reveals **how much of the digest was correct**. An attacker POSTing repeatedly to the endpoint can
measure those differences and reconstruct a valid signature one byte at a time, without ever knowing the
secret.

**And the claim that it "produces the same result" is true for the *answer* and false for the
*information leaked*.** Both comparisons return the same boolean; only one of them takes the same time
regardless of input.

**Line 591 uses the right primitive**, and it is not an accident:

```javascript
  return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(digest));
```

**This is the exam's favourite shape of wrong answer** — a simplification that is *functionally*
equivalent and *not* equivalent in the property that mattered. The signature check exists to keep
forged events out; a timing side channel is a way in.

</details>

---

# Section D — Yes/No statement grid

---

## Q27 — direction

| # | Statement | Answer |
|---|---|---|
| 1 | A webhook is an outbound POST from the platform to your URL |  |
| 2 | `repository_dispatch` is triggered by an external system |  |
| 3 | `workflow_dispatch` is the external-system trigger |  |
| 4 | `client_payload` carries caller-supplied data |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | A webhook is an outbound POST from the platform to your URL | **Yes** |
| 2 | `repository_dispatch` is triggered by an external system | **Yes** |
| 3 | `workflow_dispatch` is the external-system trigger | **No** |
| 4 | `client_payload` carries caller-supplied data | **Yes** |

**In `challenge-06.md`:** lines **143**, **194**, **202**, **260**.

Row 3 is the pair the exam swaps every time. **`workflow_dispatch` is the manual button;
`repository_dispatch` is the API call.**

</details>

---

## Q28 — webhook security

| # | Statement | Answer |
|---|---|---|
| 1 | `X-Hub-Signature-256` is an HMAC of the payload |  |
| 2 | The payload is encrypted |  |
| 3 | The digest comparison should be constant-time |  |
| 4 | Skipping verification when the secret is unset is safe |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `X-Hub-Signature-256` is an HMAC of the payload | **Yes** |
| 2 | The payload is encrypted | **No** |
| 3 | The digest comparison should be constant-time | **Yes** |
| 4 | Skipping verification when the secret is unset is safe | **No** |

**In `challenge-06.md`:** lines **588–591**, **605**.

Row 4 is the condition at line 605: `if (secret && signature)`. **An unset environment variable turns
the check off silently**, which is a deployment failure that looks like a working system.

</details>

---

## Q29 — service hooks and Teams

| # | Statement | Answer |
|---|---|---|
| 1 | Publisher inputs filter which events fire a service hook |  |
| 2 | An empty `pipelineId` means all pipelines |  |
| 3 | Teams incoming webhook URLs can stop working |  |
| 4 | Adaptive Cards are the legacy format |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | Publisher inputs filter which events fire a service hook | **Yes** |
| 2 | An empty `pipelineId` means all pipelines | **Yes** |
| 3 | Teams incoming webhook URLs can stop working | **Yes** |
| 4 | Adaptive Cards are the legacy format | **No** |

**In `challenge-06.md`:** lines **472–476**, **473**, **878**, **538–541**.

Row 4 is inverted: **`MessageCard` is legacy, Adaptive Card is current.**

Row 3 is Break scenario 2 — and the reason the outbound call should be checked rather than assumed
(Q6).

</details>

---

## Q30 — payloads

| # | Statement | Answer |
|---|---|---|
| 1 | `X-GitHub-Event` names the event type |  |
| 2 | `X-GitHub-Delivery` uniquely identifies the delivery |  |
| 3 | A deployment status uses `conclusion` |  |
| 4 | A push payload's `ref` includes `refs/heads/` |  |

<details>
<summary>Show answer</summary>

| # | Statement | Answer |
|---|---|---|
| 1 | `X-GitHub-Event` names the event type | **Yes** |
| 2 | `X-GitHub-Delivery` uniquely identifies the delivery | **Yes** |
| 3 | A deployment status uses `conclusion` | **No** |
| 4 | A push payload's `ref` includes `refs/heads/` | **Yes** |

**In `challenge-06.md`:** lines **600**, **601**, **352** with **400**, **641**.

Row 3 is the two-vocabulary trap: **deployment statuses have a `state`, workflow runs have a
`conclusion`** (Q12).

Row 4 is why the Function strips the prefix before comparing to `main`.

</details>

---

# Section E — Drag and drop

---

## Q31

Match each requirement to its mechanism.

| Requirement | Mechanism |
|---|---|
| Notify an external system when a PR opens |  |
| Let an external system start a deployment |  |
| Transition a work item when a PR merges |  |
| Alert the on-call channel on deployment failure |  |
| React to an Azure Pipelines run finishing |  |
| Let a human trigger a workflow by hand |  |

**Options:** Adaptive Card to a Teams incoming webhook · Azure DevOps service hook · `Fixes AB#` + Boards app + connection · GitHub webhook · `repository_dispatch` · `workflow_dispatch`

<details>
<summary>Show answer</summary>

| Requirement | Mechanism |
|---|---|
| Notify an external system when a PR opens | **GitHub webhook** |
| Let an external system start a deployment | **`repository_dispatch`** |
| Transition a work item when a PR merges | **`Fixes AB#` + Boards app + connection** |
| Alert the on-call channel on deployment failure | **Adaptive Card to a Teams incoming webhook** |
| React to an Azure Pipelines run finishing | **Azure DevOps service hook** |
| Let a human trigger a workflow by hand | **`workflow_dispatch`** |

**In `challenge-06.md`:** lines **143–148**, **257–260**, **114–133**, **357–397**, **461–483**,
and Q2.

**Rows 2 and 6 are the pair worth over-learning.** Both start a workflow; one is called by a **system**
with a payload, the other is clicked by a **person**.

</details>

---

## Q32

Match each header or field to what it carries.

| Field | Carries |
|---|---|
| `X-GitHub-Event` |  |
| `X-GitHub-Delivery` |  |
| `X-Hub-Signature-256` |  |
| `client_payload` |  |
| `publisherInputs` |  |
| `consumerInputs` |  |

**Options:** A unique delivery ID for correlation · Caller-supplied data on a dispatch · Service hook event filters · The event type, used to route · The HMAC proving authenticity · Where the service hook sends, and what

<details>
<summary>Show answer</summary>

| Field | Carries |
|---|---|
| `X-GitHub-Event` | **The event type, used to route** |
| `X-GitHub-Delivery` | **A unique delivery ID for correlation** |
| `X-Hub-Signature-256` | **The HMAC proving authenticity** |
| `client_payload` | **Caller-supplied data on a dispatch** |
| `publisherInputs` | **Service hook event filters** |
| `consumerInputs` | **Where the service hook sends, and what** |

**In `challenge-06.md`:** lines **600**, **601**, **599**, **260**, **472**, **477–482**.

**The last two are the halves of an Azure DevOps subscription**: publisher inputs decide *whether* it
fires, consumer inputs decide *where it goes* and how much detail it carries
(`resourceDetailsToSend`, line 480).

</details>

---

## Q33

Arrange the steps to stand up a secured GitHub webhook receiver.

**Items:** Point the GitHub webhook at the Function URL · Deploy the Function to Azure · Create the
webhook with a secret · Set `GITHUB_WEBHOOK_SECRET` and `TEAMS_WEBHOOK_URL` on the Function App · Send a
ping and check the delivery

<details>
<summary>Show answer</summary>

### Answer

1. Create the webhook with a secret — lines **143–148**
2. Deploy the Function to Azure — lines **758–784**
3. Set `GITHUB_WEBHOOK_SECRET` and `TEAMS_WEBHOOK_URL` on the Function App — lines **776–781**
4. Point the GitHub webhook at the Function URL — lines **807–809**
5. Send a ping and check the delivery — lines **154–156**, **177–178**

**Step 3 before step 4 is the ordering that matters.** Point GitHub at a Function whose secret is not
yet set and the receiver's `if (secret && signature)` guard (line 605) is false — so it accepts
**unverified** payloads for however long that window lasts.

**And step 5 is not optional.** A ping plus a delivery check is the only way to know the round trip
works before a real event depends on it — and the receiver handles `ping` explicitly (line 629) so the
test is unambiguous.

</details>

---

## Q34

Match each symptom to its cause.

| Symptom | Cause |
|---|---|
| Deliveries return 401 |  |
| Receiver logs success, no Teams message |  |
| `AB#` creates no link |  |
| A dispatch is accepted but nothing runs |  |
| The receiver accepts unsigned requests |  |
| The channel is muted within a week |  |

**Options:** App not installed, or repo not connected in Azure DevOps · `event_type` missing from the workflow's `types:` · `GITHUB_WEBHOOK_SECRET` unset, so verification is skipped · Secret mismatch between GitHub and the Function · Subscribed to every event with no filtering · Teams incoming webhook URL no longer valid

<details>
<summary>Show answer</summary>

| Symptom | Cause |
|---|---|
| Deliveries return 401 | **Secret mismatch between GitHub and the Function** |
| Receiver logs success, no Teams message | **Teams incoming webhook URL no longer valid** |
| `AB#` creates no link | **App not installed, or repo not connected in Azure DevOps** |
| A dispatch is accepted but nothing runs | **`event_type` missing from the workflow's `types:`** |
| The receiver accepts unsigned requests | **`GITHUB_WEBHOOK_SECRET` unset, so verification is skipped** |
| The channel is muted within a week | **Subscribed to every event with no filtering** |

**In `challenge-06.md`:** lines **836**, **878**, **906**, **203**, **605**, **147**.

**Three of these six report success.** The 401 is the only one that announces itself — which is why the
deliveries API (Q9) is the first tool to reach for.

</details>

---

## Q35

Match each integration to how it authenticates.

| Integration | Authenticates with |
|---|---|
| GitHub webhook → your receiver |  |
| External system → GitHub dispatch |  |
| Azure DevOps service hook → your receiver |  |
| Your receiver → Teams |  |
| Azure Boards ↔ GitHub |  |

**Options:** A custom header, e.g. `X-Custom-Auth` · A token in the `Authorization` header · HMAC signature in `X-Hub-Signature-256` · The installed GitHub App · The secret in the incoming webhook URL itself

<details>
<summary>Show answer</summary>

| Integration | Authenticates with |
|---|---|
| GitHub webhook → your receiver | **HMAC signature in `X-Hub-Signature-256`** |
| External system → GitHub dispatch | **A token in the `Authorization` header** |
| Azure DevOps service hook → your receiver | **A custom header, e.g. `X-Custom-Auth`** |
| Your receiver → Teams | **The secret in the incoming webhook URL itself** |
| Azure Boards ↔ GitHub | **The installed GitHub App** |

**In `challenge-06.md`:** lines **599**, **270**, **479**, **530**, **45–46**.

**Four different models, and the direction chooses which.** Inbound-to-you needs a signature or a shared
header; outbound-from-you needs a token or a secret URL.

**Note the Teams row is the weakest of the five.** The incoming webhook URL *is* the credential — anyone
with it can post to the channel, which is why it lives in `secrets` (line 310) and why its rotation
breaks notifications (Q6).

</details>

---

# Section F — Hot area

---

## Q36

```javascript
function verifySignature(payload, signature, secret) {
  const hmac = crypto.createHmac('[BLANK 1]', secret);
  const digest = '[BLANK 2]=' + hmac.update(payload).digest('hex');
  return crypto.[BLANK 3](Buffer.from(signature), Buffer.from(digest));
}
```

- **BLANK 1:** `md5` / `sha1` / `sha256` / `aes-256`
- **BLANK 2:** `hmac` / `sha256` / `github` / `sig`
- **BLANK 3:** `equals` / `compare` / `isEqual` / `timingSafeEqual`

<details>
<summary>Show answer</summary>

### Answer: `sha256`, `sha256`, `timingSafeEqual`

**In `challenge-06.md`:** lines **589–591**.

**The digest is prefixed with the algorithm name** — the header value is literally `sha256=<hex>`, which
is why the prefix is concatenated before comparing.

**`sha1` is the deprecated `X-Hub-Signature` header** without the `-256` suffix, and it is the distractor
that catches people working from older documentation.

**And `timingSafeEqual` is the one the exam grades** (Q26).

</details>

---

## Q37

```bash
gh api repos/OWNER/REPO/hooks --method POST \
  --field events='["push","pull_request","issues","deployment_status"]' \
  --field config='{"url":"https://...","content_type":"[BLANK 1]","secret":"...","insecure_ssl":"[BLANK 2]"}'
```

- **BLANK 1:** `form` / `json` / `xml` / `text`
- **BLANK 2:** `1` / `0`

<details>
<summary>Show answer</summary>

### Answer: `json`, `0`

**In `challenge-06.md`:** line **148**.

**`insecure_ssl: "0"` means verify the TLS certificate** — the value is inverted from how it reads, so
`"1"` would *disable* verification.

**And `content_type: "json"` matters for the signature.** The HMAC is computed over the raw body; a
`form` content type wraps the payload differently, so the receiver's `request.text()` (line 598) would
be hashing something other than what GitHub signed.

</details>

---

## Q38

```yaml
on:
  [BLANK 1]:
    types: [deploy-staging, deploy-production, run-tests]

jobs:
  deploy-production:
    if: github.event.[BLANK 2] == 'deploy-production'
    [BLANK 3]: production
```

- **BLANK 1:** `workflow_dispatch` / `workflow_run` / `repository_dispatch` / `deployment`
- **BLANK 2:** `event_type` / `action` / `type` / `client_payload.type`
- **BLANK 3:** `runs-on` / `concurrency` / `permissions` / `environment`

<details>
<summary>Show answer</summary>

### Answer: `repository_dispatch`, `action`, `environment`

**In `challenge-06.md`:** lines **202–224**.

**BLANK 2 is the naming mismatch worth memorising.** The caller sends `event_type` (line 259); the
workflow reads it as `github.event.action`. Same value, two names.

**And `environment: production` is what keeps the protection rules in play** when an external system can
start the deployment (Q11) — without it, anyone who can call the dispatch API deploys straight to
production.

</details>

---

## Q39

```json
{
  "publisherId": "[BLANK 1]",
  "eventType": "workitem.updated",
  "consumerId": "[BLANK 2]",
  "consumerActionId": "httpRequest",
  "publisherInputs": {
    "areaPath": "Contoso Web Platform",
    "workItemType": "Bug"
  }
}
```

- **BLANK 1:** `pipelines` / `boards` / `tfs` / `git`
- **BLANK 2:** `teams` / `webHooks` / `slack` / `email`

<details>
<summary>Show answer</summary>

### Answer: `tfs`, `webHooks`

**In `challenge-06.md`:** lines **494–496**.

**`tfs` is the publisher for work item events; `pipelines` is the publisher for run events** (line 468).
The publisher IDs carry historical names and the exam quotes them literally.

**And `webHooks` with `httpRequest` is the generic consumer** — it can target any HTTP endpoint, which
is how one Azure Function serves both GitHub and Azure DevOps in this challenge (Q21).

</details>

---

## Q40

```javascript
const event = request.headers.get('[BLANK 1]');
...
switch (event) {
  case 'push': ...
  case '[BLANK 2]': context.log('Ping received - webhook is configured correctly'); break;
  default: context.log(`Unhandled event type: ${event}`);
}
```

- **BLANK 1:** `x-github-delivery` / `x-hub-signature-256` / `x-github-event` / `content-type`
- **BLANK 2:** `test` / `hello` / `check` / `ping`

<details>
<summary>Show answer</summary>

### Answer: `x-github-event`, `ping`

**In `challenge-06.md`:** lines **600** and **629–631**.

**Handling `ping` explicitly is what makes the configuration test meaningful.** Without that case it
falls through to `default` and logs "Unhandled event type" — which reads like a failure when the
webhook is actually fine.

</details>

---

## Q41

```bash
# Rotate a compromised webhook secret
NEW_SECRET=$(openssl rand -hex 32)

gh api repos/OWNER/REPO/hooks/$HOOK_ID --method [BLANK 1] \
  --field config="{\"url\":\"$FUNCTION_URL\",\"content_type\":\"json\",\"secret\":\"$NEW_SECRET\"}"

az functionapp config appsettings set --name $FUNCTION_APP \
  --resource-group $RESOURCE_GROUP \
  --settings [BLANK 2]="$NEW_SECRET"
```

- **BLANK 1:** `POST` / `PATCH` / `PUT` / `DELETE`
- **BLANK 2:** `WEBHOOK_SECRET` / `GH_SECRET` / `GITHUB_WEBHOOK_SECRET` / `TEAMS_WEBHOOK_URL`

<details>
<summary>Show answer</summary>

### Answer: `PATCH`, `GITHUB_WEBHOOK_SECRET`

**In `challenge-06.md`:** lines **843–851** and **604**.

**`PATCH` updates the existing hook** — `POST` would create a second one, leaving the old secret live on
the original.

**And the environment variable name must match what the code reads** (line 604). Set
`WEBHOOK_SECRET` instead and `process.env.GITHUB_WEBHOOK_SECRET` is undefined, so line 605's guard is
false and **verification is skipped entirely** — a rotation that silently disables the check it was
meant to strengthen.

</details>

---

# Section G — Case study

## Case study: Contoso three-tool integration

### Background

Contoso Ltd uses **GitHub** for source control and CI/CD, **Azure Boards** for project management, and
**Microsoft Teams** for internal communication. **Nothing is connected.** The PM team **manually checks
Azure Boards** because they get no notification when PRs close work items. The on-call team **misses
deployment failures** because alerts go to email nobody reads after hours. Developers context-switch
between three tools **with no cross-linking**.

### Requirements

**Linking**

- Merging a pull request must transition the referenced work item with no manual step
- A work item must show the commits and pull requests that touched it

**Alerting**

- Deployment failures must reach the on-call Teams channel immediately, visually distinct from routine
  messages
- Routine activity must not flood the channel

**Automation**

- An external system must be able to trigger a staging deployment, passing a ref and a reason
- Production deployments triggered this way must still be gated

**Security**

- Every inbound integration must verify the caller
- A compromised secret must be rotatable without leaving a gap

---

## Q42

How is automatic work item transition achieved?

- A. Install the app and use a bare `AB#` in the PR body
- B. A workflow that calls the Azure Boards API on merge
- C. Install and connect the app, then use `Fixes AB#`
- D. A nightly job that synchronises board state

<details>
<summary>Show answer</summary>

### Answer: C

**In `challenge-06.md`:** lines **45–46**, **62**, **114–132**.

```bash
gh pr create --body "Adds retry handling for transient payment failures.

Fixes AB#${WORK_ITEM_ID}" --base main
```

**Both requirements are met by the same integration**: the `Fixes` keyword transitions, and the work item
gains `GitHub Commit` and `GitHub Pull Request` relations (line 127) so it shows what touched it.

**Why A is the near-miss that appears in every Boards challenge.** Bare `AB#` links without
transitioning — the PM still has to look.

**Why B is real, unnecessary and worse.** You would maintain code and a credential to reproduce
behaviour the integration already provides.

</details>

---

## Q43

How should deployment failures reach the channel without flooding it?

- A. Subscribe the channel to all repository events
- B. Post every deployment status to the channel
- C. Email the on-call rota as changes deploy
- D. Gate on failure state, card in `Attention`

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-06.md`:** lines **351–375**.

**Two requirements, two parts of the answer.** The `if:` filters to failures only; the `Attention` colour
makes the card visually distinct from routine messages (Q14).

**Why A and B recreate the problem in a new medium.** The scenario's failure at line 22 is not that email
is the wrong tool — it is that the signal is buried. **A Teams channel receiving everything is unread
email with better formatting.**

**Why C is what they do today.**

</details>

---

## Q44

How should an external system trigger a staging deployment with a ref and a reason?

- A. `workflow_dispatch` with declared inputs for each field
- B. `repository_dispatch` with a `client_payload` for both
- C. A webhook from the external system into GitHub
- D. A scheduled workflow that polls for pending requests

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-06.md`:** lines **257–260** and **216–218**.

```bash
  --field event_type="deploy-staging" \
  --field client_payload='{"ref":"feature/payment-retry","triggered_by":"platform-bot","reason":"QA requested staging deploy"}'
```

**`client_payload` is the only mechanism here that carries arbitrary caller data**, and the workflow
reads it at lines 216–218.

**Why A is the trap** (Q25). `workflow_dispatch` is the manual trigger; it takes declared `inputs`, not
a free-form payload, and it is designed for a person.

**Why C inverts the direction.** A webhook is GitHub calling you.

</details>

---

## Q45

Which **two** keep an external-triggered production deployment gated? (Choose two.)

- A. A separate `event_type` for production deploys
- B. `environment: production` declared on the job
- C. A flag set inside the `client_payload` object
- D. Protection rules configured on that environment
- E. Restricting who holds the API token that calls

<details>
<summary>Show answer</summary>

### Answer: B, D

**In `challenge-06.md`:** line **224**, with Challenge 41's environment checks.

**B names the environment; D is where the approval actually lives.** The YAML declares which environment
the job targets, and the rules on that environment decide whether it proceeds — the same separation as
Challenge 41 and Challenge 45.

**Why E is genuinely important and not a gate.** Controlling the token limits *who can ask*; it does not
review *what was asked for*. **A caller with a valid token would otherwise deploy straight to
production.**

**Why A and C are caller-controlled** — anything in the payload can be set by whoever is calling.

</details>

---

## Q46

How should a compromised webhook secret be rotated without a gap?

- A. Generate a new secret, PATCH the hook, then the setting
- B. Remove the secret so verification is skipped meanwhile
- C. Delete the webhook and create a replacement hook
- D. Rotate the Teams incoming webhook URL instead

<details>
<summary>Show answer</summary>

### Answer: A

**In `challenge-06.md`:** lines **840–851**.

**PATCH rather than recreate** (Q41), so the hook ID, the event subscription and the delivery history
survive.

**Why B is the option that turns a rotation into an incident.** With `GITHUB_WEBHOOK_SECRET` unset, the
guard at line 605 is false and the receiver accepts **anything** — precisely while you know the old
secret is compromised.

**And the honest caveat on "without a gap":** between the two commands there is a brief window where
GitHub signs with the new secret and the Function still holds the old one, so those deliveries 401. They
are recoverable — replay them with the `attempts` endpoint (line 186) once both sides agree.

</details>

---

## Q47

Seven months after the integrations go live, the PM team reports that work items stopped transitioning
about three weeks ago. Commits still contain `Fixes AB#`, the Teams notifications still arrive, and the
webhook deliveries are all 200. GitHub organisation administrators recently tightened third-party
application access.

What is the most likely cause?

- A. The webhook secret was rotated on only one side
- B. The Boards app lost its authorisation
- C. The `Fixes` keyword was deprecated by Azure Boards
- D. The Teams connector expired and was not recreated

<details>
<summary>Show answer</summary>

### Answer: B

**In `challenge-06.md`:** lines **897–906**.

```text
**Fix:** The Azure Boards GitHub App may have lost access due to organization permission changes.
Re-authorize the app in GitHub organization settings, and verify the repository is still linked in
Azure DevOps Project Settings > GitHub connections.
```

**Every other integration still working is the detail that localises it.** The webhook is a **repository
hook** with its own secret; the Teams post is an **outbound call**. Neither depends on the Boards app,
so a change that revokes app authorisation breaks the Boards link alone.

**And this is the integration most likely to break from a change nobody associates with it.** Tightening
third-party access is a security improvement made by administrators who are not thinking about Azure
Boards — the app quietly loses access, and the only symptom is work items that stop moving.

**The durable lesson: a GitHub App installation is a standing grant that someone else can revoke.** Add
the diagnosis at lines 898–899 to your runbook, and treat "work items stopped transitioning" as an
authorisation question before a syntax one.

</details>

---

## Q48

A year on, the PM team never opens Boards to check status, on-call finds out about a failed deployment
before a customer does, and a QA request can deploy to staging without anyone in the room.

Which explanation best accounts for the change?

- A. The teams learned to check the other tools more often
- B. More people were added to the on-call email alias
- C. A daily stand-up was added to share status widely
- D. Every link is a by-product of work already being done

<details>
<summary>Show answer</summary>

### Answer: D

**In `challenge-06.md`:** lines **114–133**, **588–638**, **202–228**, **351–397**.

**Take the three complaints at line 22 in turn.**

*PMs manually checking Boards* — the work item now moves itself when the PR merges, and it carries links
to the commit and the pull request. **The PM's question is answered before they ask it.**

*On-call missing deployment failures* — the failure arrives as a visually distinct card in the channel
the team already has open, filtered so it is not competing with routine noise.

*Context-switching with no cross-links* — the work item links to the PR, the PR references the work
item, and the notification links to the deployment. **Each tool now points at the others.**

**And the receiver design is the quiet win.** One Azure Function accepts GitHub webhooks and Azure DevOps
service hooks (lines 478, 503) and forwards to Teams — so the integration surface is one endpoint to
secure, monitor and rotate rather than three.

**What actually changed is not attention.** Nobody at Contoso was refusing to check Azure Boards; they
were being asked to **maintain synchronisation by hand between three systems that had no idea the others
existed**. That is a task that fails whenever anyone is busy.

**The graded idea is the one this whole domain keeps returning to: the integration must be a by-product
of the work.** The commit already exists, the merge already happens, the deployment already reports its
state. **Connecting those events is cheaper and more reliable than asking three teams to keep three
tools in agreement.**

</details>

---
---

# Trap index

| Trap | Questions | Defence |
|---|---|---|
| **`workflow_dispatch` for external triggers** | Q25, Q27, Q31, Q44 | `repository_dispatch` carries `client_payload` |
| **`event_type` vs `github.event.action`** | Q10, Q38 | Sent as `event_type`, read as `action` |
| **App installed but repo not connected** | Q4, Q17, Q25, Q42 | Two halves. Silent failure |
| **Bare `AB#` expected to transition** | Q42 | `Fixes AB#` transitions; `AB#` only links |
| **"Unguessable URL" as security** | Q25 | The URL is in logs and config. Verify the signature |
| **`===` instead of `timingSafeEqual`** | Q26, Q36 | Leaks the digest byte by byte through timing |
| **Verification skipped when the secret is unset** | Q28, Q41, Q46 | `if (secret && signature)` fails open |
| **Wrong env var name after rotation** | Q41 | Silently disables the check |
| **`sha1` / `X-Hub-Signature`** | Q36 | Deprecated. Use the `-256` header |
| **`insecure_ssl: "1"`** | Q37 | Inverted. `"0"` means verify |
| **Subscribing to every event** | Q7, Q25, Q34, Q43 | Volume is why the current alerts are missed |
| **`state` vs `conclusion`** | Q12, Q30 | Deployment status has `state`; workflow run has `conclusion` |
| **`MessageCard` assumed current** | Q13, Q29 | Adaptive Card is current |
| **Teams webhook assumed permanent** | Q6, Q29, Q34 | Expires when the connector or channel goes |
| **`POST` instead of `PATCH` on a hook** | Q41, Q46 | Creates a second hook, old secret still live |

---

# What to memorise

**In `challenge-06.md`:** lines **143–188**, **194–274**, **461–508**, **584–638**.

```text
DIRECTION DECIDES EVERYTHING - who initiates, and on what event?
  OUTBOUND   webhook              platform POSTs to YOUR url        auth: HMAC SIGNATURE
  INBOUND    repository_dispatch  external system POSTs to GitHub   auth: TOKEN
  MANUAL     workflow_dispatch    a PERSON clicks Run workflow      (declared inputs, no payload)
  BOTH WAYS  Azure Boards <-> GitHub                                auth: the installed GitHub App
  ADO OUT    service hook         ADO POSTs to any HTTP endpoint    auth: custom header

AZURE BOARDS LINKING needs BOTH halves
  1. Azure Boards GitHub App installed on the repo
  2. repo connected in ADO > Project Settings > Boards > GitHub connections
  AB#1234        links only          Fixes AB#1234   links AND transitions
  breaks later when org app access is tightened -> re-authorise (Break scenario 3)
```

```bash
# GitHub webhook                                    (lines 143-188)
gh api repos/O/R/hooks --method POST \
  --field events='["push","pull_request","issues","deployment_status"]' \
  --field config='{"url":"...","content_type":"json","secret":"...","insecure_ssl":"0"}'
#   insecure_ssl "0" = VERIFY TLS (inverted from how it reads)
#   content_type json - the HMAC is over the RAW BODY

.../hooks/$ID/pings      --method POST      send a test ping
.../hooks/$ID            --method PATCH     add_events / remove_events / rotate secret
.../hooks/$ID/deliveries                    list: id, event, status_code
.../hooks/$ID/deliveries/$DID               inspect request AND response body
.../hooks/$ID/deliveries/$DID/attempts POST REDELIVER - how you recover a receiver outage
```

```javascript
// Signature verification - the exam grades this     (lines 588-609)
const hmac = crypto.createHmac('sha256', secret);
const digest = 'sha256=' + hmac.update(payload).digest('hex');
return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(digest));
//     ^^^^^^^^^^^^^^^ NOT ===  - a normal compare leaks the digest byte by byte via timing

if (secret && signature) { ... }   // <- FAILS OPEN if the env var is unset

// headers
x-github-event      the type -> routes the switch     x-github-delivery   unique id
x-hub-signature-256 the HMAC                          (x-hub-signature = deprecated sha1)
// push payload ref is the FULL ref: payload.ref.replace('refs/heads/','')
```

```yaml
# repository_dispatch                                (lines 202-228, 257-266)
on:
  repository_dispatch:
    types: [deploy-staging, deploy-production, run-tests]   # omit a type -> dispatch runs NOTHING
jobs:
  deploy-production:
    if: github.event.action == 'deploy-production'   # sent as event_type, READ as action
    environment: production                          # external caller, but the GATE still applies
    steps:
      - uses: actions/checkout@v4
        with: {ref: "${{ github.event.client_payload.ref || 'main' }}"}

# caller
gh api repos/O/R/dispatches --method POST \
  --field event_type="deploy-staging" \
  --field client_payload='{"ref":"...","triggered_by":"...","reason":"..."}'
```

```text
AZURE DEVOPS SERVICE HOOK - 4 fields                (lines 461-508)
  publisherId       pipelines  (run events)   |  tfs  (work item events)
  eventType         ms.vss-pipelines.run-state-changed-event  |  workitem.updated
  consumerId        webHooks        consumerActionId  httpRequest   -> ANY http endpoint
  publisherInputs   the FILTERS: pipelineId / areaPath / workItemType    ("" = all)
  consumerInputs    url, httpHeaders (X-Custom-Auth), resourceDetailsToSend, messagesToSend

TEAMS - Adaptive Card (MessageCard is LEGACY)       (lines 535-557)
  {"type":"message","attachments":[{"contentType":"application/vnd.microsoft.card.adaptive",
    "content":{"type":"AdaptiveCard","version":"1.4",
      "body":[{TextBlock, color:"Attention" for urgent}, {FactSet}],
      "actions":[{"type":"Action.OpenUrl"}]}}]}
  the incoming webhook URL IS the credential -> keep it in secrets, it EXPIRES with the connector
```

---

# Scoring

| Score | Meaning |
|---|---|
| 44–48 | Domain 1 complete. Move to Challenge 07 |
| 38–43 | Re-read the trap index and the direction table, then move on |
| 30–37 | Rewrite the direction table and the signature check from memory, then retake |
| Below 30 | Redo Tasks 1, 2 and 6 hands-on before retaking |

Record your result in `AZ-400-Learning-Log.md` under Challenge 06.

:::danger The two questions

**Who initiates, and on what event?** Outbound is a webhook. Inbound is `repository_dispatch`. Manual is
`workflow_dispatch`. That single question routes most of this domain.

**How does the receiver know it is really them?** A signature it verifies in constant time — never an
unguessable URL, and never a check that is skipped when a variable is unset.

Contoso's teams were not ignoring each other's tools. They were being asked to keep three systems in
agreement by hand.

:::
