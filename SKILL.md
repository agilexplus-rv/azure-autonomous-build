---
name: azure-autonomous-build
description: Use when building or operating a production web application autonomously on Azure — Container Apps, PostgreSQL, Blob Storage with malware scanning, Key Vault, Event Grid — with Cloudflare DNS and Stripe payments. Covers the orchestration model (passes/phases/sprints/slices), the adversarial gate system, the CI contract that makes "no stubs, no stales" mechanical, and the deployment, security and integration traps that each cost a day to learn.
---

# Building software autonomously on Azure

This is a working method, not a tutorial. Every rule here was bought with a failure on a real
build: a government compliance portal (Next.js, Azure, Stripe) built by parallel agents over
~30 hours, against ~200 traceable requirements.

**The failure mode this exists to prevent** is software that demos well and is hollow — every
check green, every PR merged, and the feature unreachable. That happened four times on the
reference build before the guards in §3 existed. Assume it will happen to you.

---

## 1. The shape of the work

```
PASS    a half of the project, defined by the SHAPE of the work, not its size.
        Pass 1 = spec-fidelity work: many interlocking requirements, held in context,
                 built outward against a CI gate.
        Pass 2 = external-signal work: pen test, axe, load profile, translator — each
                 hands back findings with an oracle OUTSIDE the model.
PHASE   a coherent capability. Ends with an adversarial gate, not with its last merge.
SPRINT  one sitting of work with ONE falsifiable Sprint Goal.
SLICE   3–5 per sprint, each in its own worktree, each owning declared files.
```

### Every sprint has ONE Sprint Goal, written as an exit test

The single most load-bearing convention. A goal must be able to FAIL.

> ✗ "Payments are implemented with Stripe Checkout."
> ✓ "A replayed webhook produces exactly ONE transition and ONE payment row; a dropped webhook
>    is recovered by the nightly job AND the customer is emailed; withdraw-then-pay lands in
>    `PAID_IN_ERROR` and never enters the queue; an abandoned checkout expires with its blobs
>    deleted."

The second cannot be satisfied by a plausible-looking implementation. The first can.

**The loop's exit condition is the Sprint Goal green on a squash-merged PR** — not "the code is
written", not "tests pass locally". That one rule is what stops a sprint becoming a fortnight.

### Slices run in parallel, in isolated worktrees, with declared file ownership

A shared working tree corrupted an agent's in-flight work twice. Worktrees are not optional.
Parallelism then moves the collision surface from the tree to shared *files*, so declare
ownership up front:

| Shared file | Rule |
|---|---|
| migrations | numbers **pre-assigned per slice in the sprint script** — two agents can never pick the same one |
| `package.json` | exactly **one** slice per sprint may edit it, named in its prompt |
| the progress/state file | written by the **gate only** |
| CI workflow | one slice per sprint, named explicitly |

Default to concurrent; serialise only on a real dependency. One reference sprint was written as
six sequential stages when its dependency graph was three — the measured cost was 2h27m versus
1h35m on comparable work. But do not fake independence: two slices editing one file is a merge
conflict with extra steps.

### ONE slice must own the integration point

**This is the rule bought most expensively.** Strict file ownership stops two agents fighting
over a file; it also means *a file nobody claims is a file nobody writes*.

Four times on the reference build, a feature shipped complete, CI-green, and entirely
unreachable: a logo component never mounted; a webhook route never registered in the release
gate, so the cloud provider's validation handshake 404'd and no event could ever arrive; a
signed-URL issuer with no caller, so no uploaded document could be opened; and the primary
call-to-action pointing at a route with no page behind it.

So every sprint that splits a feature **names the slice that owns the route/mount/wiring**, that
slice **runs last against merged main**, and its exit test **walks the real journey with no
hand-constructed URL**. A component with no caller is the same class of defect as a stub: it
passes every check and does nothing.

### Model pinning — by the COST OF A SUBTLE MISTAKE, not difficulty

A wrong constraint in a migration is found weeks later by a user. A wrong sentence in a doc is
found by reading it.

| Tier | Use for |
|---|---|
| **Strongest** | schema and DB constraints · state machines · authn/authz · **money** · publication and anything irreversible · security controls |
| **Middle** | orchestration and prompt-writing · **gates and verifiers** · triage · CRUD over an already-modelled domain · reporting that reconciles against fixtures |
| **Cheapest** | mechanical, single-file, no debugging loop — doc sections, string catalogues, config keys, lint cleanups |

**Hard limit:** never give the cheapest tier a task whose exit condition is "make CI green", or
anything needing a diagnostic loop. One wrote correct code, correctly refused to merge on red,
then reported *"cannot diagnose without access to CI logs"* — it never fetched them.

---

## 2. The gate system

A sprint is not complete because its slices merged. Independent agents that **did not write the
code** hunt for failure.

### Fan out, then synthesise

Measured on a single-agent gate: **151 sequential tool calls, only one a production build,
~54 minutes** — the cost was round trips, not compute, and four consecutive calls retried one
probe because the agent worked in the main checkout and kept finding the wrong dependencies.

```
N parallel verifiers over DISJOINT areas  →  one synthesiser
        every agent worktree-isolated, each running its own dependency install
```

Verifiers **find; they do not fix and do not open PRs.** A separate agent writes the fix prompts,
so a shallow diagnosis is not quietly patched over by whoever made it.

### What the synthesiser must do

1. De-duplicate, keeping the strongest evidence.
2. **Spot-check any blocker whose evidence looks thin** — a wrong blocker costs a whole cycle.
3. **Downgrade any claimed PASS whose evidence could not have failed.** This is the highest-value
   instruction in the whole method. It catches: a dev-mode check for a production-only defect; a
   test asserting the *absence* of an error string; a permission check run as the database owner.
   That class of false pass cost two entire fix cycles on one sprint.
4. Return a **structured verdict** (`goalMet`, `journeyReachable`, `findings[]`) so "done" is a
   value, not a vibe.

### A missing report is UNVERIFIED, never a pass

An agent that dies on a transient error otherwise reads as silence, and silence reads as
success. This happened: a synthesiser died after pushing its findings and returned `null`. The
script must detect it and say so.

### Four rules for every gate prompt

1. **Reproduce on a production build.** Dev mode hid a CSP defect that broke *every* client-side
   navigation in the application.
2. **Reproduce permission defects as the least-privileged principal**, against a real database
   with every migration applied. Running as the owner proves nothing.
3. **Assert positive evidence.** A test asserting the absence of an error string is worse than no
   test, because it counts as coverage. One passed while a journey was broken end to end: it
   checked for `"This page could not be found"` while the real failure rendered
   `"This page couldn't load"`.
4. **Plant violations to prove each guard can fail.** If a guard cannot be made to fail, that is a
   defect *in the guard*, and it is a finding.

### Risk-proportional gating

Full gates cost ~9% of total run time and repeatedly earned it. Spend them where a defect would
be *dangerous* rather than merely annoying: money, security controls, irreversible operations,
anything published to third parties. But note — on the reference build the two longest sprints
had **no** gate, and the ungated sprints are where the unreachable features hid.

---

## 3. The contract: making "no stubs, no stales" mechanical

A `verify` script that CI runs, composed of static checks. The ones that earned their place:

| Check | Fails on |
|---|---|
| `no-stubs` | TODO/FIXME/stub/mock/placeholder/lorem in source; a handler returning a literal never sourced from the database |
| `trace` | **bidirectional** — a requirement ID with no test (nothing left behind) AND an ID in code the spec doesn't contain (nothing stale) |
| `portability` | any subscription id, tenant id, resource group, region literal, globally-unique name or own hostname outside the parameter files; any credential read from env where a managed identity exists |
| `grants` | every DB write compared **column by column** against every GRANT (see §5) |
| `keys` | live payment-provider keys in the repo or a resolved environment |
| `route-coverage` | a route on disk with no deliberate gate decision, **and every internal link resolving to a route that exists** |
| `i18n` | key parity across locales; a hardcoded user-facing string in a rendered component |
| `design-rules` | product constraints where being wrong is a legal or safety failure — **never re-baselined** |

### Three ways a guard silently stops guarding

**It never runs in CI.** Four checks lived in the `verify` script and had never been listed in
the workflow, so they gated a local run and nothing else. All four passed once wired in — but
nothing would have stopped them failing. *A guard that never runs is the defect it exists to catch.*

**It can't see the file.** A source file containing a `0x00` byte was treated as binary by the
shared file reader, which returned `null`; every check then did `if (text === null) continue`.
**Thirteen guards silently skipped that file.** The NUL was a legitimate composite-key delimiter.
Fix the *reader*, not the file — and make a skip **loud**: a guard that cannot see a file must
never look the same as one that saw it and found nothing.

**It asks the wrong question.** A grant check asked whether UPDATE was granted on the *table*; it
was — on ten other columns — so it passed a tree whose main form could not save. It was also
line-oriented, so a multi-line `update X\n set …` never matched at all.

### Verify state, never output

Do not judge a command by a piped exit status — **a pipeline's status is the last command's**, so
`deploy.sh … | tail` reports `0` on a failed deploy. This produced three wrong conclusions in one
evening, and nearly a PR "fixing" tooling that had failed correctly and loudly.

Verify the *effect*: the cloud control plane for a deployment, the database for a migration, the
endpoint for a route.

---

## 4. Azure

See `reference/azure.md` for the full estate and parameterisation. The traps:

- **One supported deploy path, and use it.** A raw `az deployment group create` against a template
  whose app resources are gated on image parameters **silently skips them and reports
  `Succeeded`**. Wrap deployment in a script and make the script the only way.
- **Always pass the image parameter.** Deploying without it left the app-name deployment *outputs
  empty*, which the publish script and live test suites both read.
- **Event Grid validates against the RUNNING app.** A deployment that both ships a new webhook
  route and creates the subscription consuming it **cannot succeed in one pass** — the handshake
  hits the old image and 404s. The apps update before that step fails, so re-run the same deploy.
  On a clean estate this bites on the very first install.
- **Never tear down the working estate to test another.** Deploy alongside, verify, then tear down
  the second. There must never be a moment with zero estates. Container Apps environment deletion
  takes ~2 hours — never block on it.
- **Never deploy into an in-flight CI run** if the browser suites test the live estate. It turns
  `main` red on a change that was green.
- **Deploy once per phase, before each gate** — not after every sprint. Deploys measured ~20% of
  run time and mostly re-proved what CI had. But never defer them *past* a gate: a whole class of
  defect is invisible to CI and appears only against the live estate.
- **Region is a parameter with an allowed-values list**, and regional capability genuinely differs
  (zone-redundant HA vs geo-redundant backup vary by region). A preflight must print what the
  chosen region can and cannot do *before* anything is created.
- **Develop in a different region from the one you recommend**, so portability is exercised by
  every deployment rather than tested once at the end.

---

## 5. Security precautions

Full detail in `reference/security.md`. The non-negotiables:

**Malware scanning on every upload.** Enable the cloud provider's own on-upload scanning
(Defender for Storage) rather than hand-rolling one, cap the monthly scan volume so a runaway
upload loop cannot run away with the bill, and use the provider's built-in quarantine. **Prove it
with a real EICAR file** through the real pipeline — an infected file must be quarantined, never
rendered to a reviewer, and the uploader told. Note the scan-result webhook must be reachable
(see §4's Event Grid trap) or verdicts never arrive.

**Accept files by content sniffing, never by extension or client Content-Type.** Use a maintained
library. The decisive test is a PNG renamed `.pdf` being rejected.

**Uploads never served from the application origin.** Short-lived, permission-checked signed URLs
with `Content-Disposition: attachment`, from a private container namespaced per record. Test that
a bare URL 403s, an expired signature 403s, a tampered signature 403s, and another record's
prefix 404s.

**Managed identity everywhere; no secrets in environment variables.** The environment carries
endpoints and names, never credentials. Third-party secrets go into the vault out of band and are
referenced by URI — **never** as template parameters, outputs or deployment arguments, because the
deployment history retains them and discloses them to anyone with reader access long afterwards.

**Least privilege at the database grant level, column-scoped.** Default privileges grant SELECT
and nothing else; every write privilege arrives with the migration that introduces the write path,
and UPDATE grants name **columns**, so the app can move a record's lifecycle but not rewrite what
the user submitted. Test as a role holding *only* the app's group membership.

**Content-Security-Policy: use `'strict-dynamic'` with a nonce.** Without it, modern framework
chunk loaders are blocked and every client-side navigation dies. Do **not** "fix" that with
`'self'` (trusts any same-origin script on a service accepting uploads) or `'unsafe-eval'`.

**Append-only audit at the grant level**, not just in code — prove the app role *cannot* UPDATE or
DELETE the audit table.

---

## 6. Cloudflare and Stripe

**Cloudflare.** Resolve the zone **by name first and hard-stop on an empty result**, so a token
for the wrong account cannot silently write DNS somewhere else. Two accounts on one machine is
normal and the failure is silent.

**Stripe.** Enforce webhook idempotency with a **database constraint** on the session id, not
application code — two concurrent deliveries must not both win. Verify signatures with the
provider's own library, never hand-rolled HMAC. Build and test the four failure modes explicitly:
replayed webhook, **lost webhook recovered by a nightly job *and the customer emailed*** (a
recovery nobody is told about is not a recovery), withdraw-then-pay landing in an error state that
never enters the queue, and an abandoned checkout expiring with its blobs deleted. Keep the fee in
configuration, never a literal, and fail the build on any live key reaching the repo.

---

## 7. Operational contract

- Branch → commit → push → PR → **wait for CI** → squash merge on green → delete branch. Never
  commit to main. Never merge on red or pending.
- **Make the required status check an aggregating job** that depends on the real (possibly
  sharded, possibly conditional) jobs. Branch protection can only require a fixed name.
  **Renaming it blocks every PR in the repository.**
- **A skipped job counts as passing** in that aggregation. So a "docs-only" fast path that skips
  the browser suites is a trap the moment a PR touches anything else.
- **Require branches to be up to date** (this prevents merging on stale-green CI) — and understand
  the consequence: **every merge puts every open PR BEHIND**, costing a rebase and a full CI pass.
  So **don't merge while parallel agents hold open PRs**, and **split fix rounds by area of code,
  not by finding**.
- Record decisions **with their reasoning, at the moment they are taken**. A decision whose *why*
  is lost gets relitigated or silently reversed.
- Write the state file for a **cold start** — assume the reader has no memory of the session.
- Handing the system to its owners is a separate exercise from a cold start, with its own exit
  test and its own off-boarding. See `reference/handover.md` — and settle domain, payment-account
  and repository ownership at the START of the build, not at the end.

---

## 8. Reference

- `reference/orchestration.md` — sprint script skeleton, parallelism, worktrees
- `reference/gates.md` — verifier area splits, synthesiser prompt, schemas
- `reference/azure.md` — estate, parameterisation, preflight, deploy, client-tenant access
- `reference/security.md` — malware scanning, SAS, grants, CSP, secrets
- `reference/failure-signatures.md` — symptom → real cause index
- `reference/handover.md` — custody handover, access off-boarding, restore drill, teardown
- `adapters/` — how to load this in Claude Code, Cursor and Hermes
